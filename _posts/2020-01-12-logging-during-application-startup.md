---
title: Logging During Application Startup
tags: c# .net generichost logging serilog
header:
  image: "/assets/2020/01-12/header1280.png"
  teaser: "/assets/2020/01-12/header1280.png"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2020/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2020/mm-dd/pic.jpg)
---

They said it couldn't be done...

<!--more-->

Back in April of 2019, Microsoft [announced](https://github.com/dotnet/aspnetcore/issues/9337) a change to ASP.NET Core's `Startup` class as a result of migrating the platform to the Generic Host model. A few weeks later, somebody noticed this means you can't do logging during `Program.Main` or `Startup.cs`:

> Using the generic host builder, how do you log?...

The response wasn't encouraging:

> ...logging is built on dependency injection, and ConfigureLogging is just a wrapper for ConfigureServices. It's not possible to log until after all of the services have been registered and the container has been built.

The discussion is pretty long (and isn't entirely logging-focused), but suffice to say there isn't a good solution. Worse, like many things in today's .NET, this is bigger than just ASP.NET -- the limitation applies to any Generic Host -based application. On another project I'm doing a huge amount of work with hosted headless services, so I see this as a real problem. I decided to try doing something about it.

This is a short article to introduce my solution to this problem. The code is in my repository, [MV10/GenericHostBuilderLogger](https://github.com/MV10/GenericHostBuilderLogger), and the README has everything you need to know to use it in your own application. As I write this, it isn't packaged on NuGet yet, but that should be coming soon.

As a bonus, it optionally also supports [Serilog](https://github.com/serilog), because Serilog is too awesome to ignore. (This will be a separate package when I get around to setting up packaging.)

## The Basic Idea

Products like `Serilog` take the position that a static logger is better than a service-based logger. Quite frankly, this whole concern is a non-issue if you just use a static logger. While I do use Serilog myself, I strongly dislike configuration in code, and in production apps we don't use that approach. Instead we use the optional `IConfiguration`-based approach so that logger behaviors can be changed without re-building the entire application. This means Serilog suffers from the same problems as the vanilla .NET logger implementation.

My goals for this solution were:

* caching log messages prior to calling `IHostBuilder.Build`
* writing cached messages to the host-provided logger
* ensuring logging works when application startup fails
* ensuring logging works when logger configuration fails

Ironically, my solution is also basically a static logger, and since the general `ILogger` interface and extensions are so well-known, my solution replicates a subset of that functionality, supporting the basic `Log()` methods and related wrapper methods like `LogInformation()` and `LogWarning()`. It uses a thread-safe queue as the message cache, and adds some extension methods to make it easy to set up.

But the logger implementation isn't the interesting part here, it's what we do with the log messages that is important. The project defines a couple of Generic Host background services to ensure the cache is flushed either at application startup or during a terminal failure event. You can read more about that at the end of the repository README.

## The Sample Code

The `ConsoleTest` project in the repository has five variations on `Program.Main` demonstrating various scenarios, and the same five scenarios are also available in the `ConsoleWithSerilog` project.

### The "Successful Run" Sample

As the name suggests, this is the simplest example when everything goes right. Here we use the most basic registration extension, `AddHostBuilderLogger`. The program just writes one message to the startup log, then continues normally. After `IHostBuilder.Build` is called (which is one of the things that happens automatically when you execute `RunConsoleAsync`), the host fires `ApplicationStarted` and the background service uses the host's logger to dump the cached log message.

The console output looks like this. The highlighted entries were cached before the host's logger existed and were emitted later.

![Successfulrun Default](/assets/2020/01-12/successfulrun_default.png)

The same output from Serilog (with output produced by the _Serilog.Sinks.Console_ package) looks like this. Instead of calling `AddHostBuilderLogger`, it uses Serilog's static configure-in-code approach and calls `UseSerilogWithHostBuilderLogger`.

![Successfulrun Serilog](/assets/2020/01-12/successfulrun_serilog.png)

### The "Basic Exception" Sample

This sample throws an exception within `Program.Main` itself. The vanilla .NET version and the Serilog versions are configured the same way as above.

![Basicexception](/assets/2020/01-12/basicexception.png)

This time the log begins with a `Warning` message. This is emitted by our system when the application calls `TerminalEmitCachedMessages`, which is a request to flush the cache then terminate the application. Further down, you can see the `Error`-level log reporting the exception message and the stack trace.

### The "Successful Run With Config" Sample

The output of this sample is identical to the earlier "Successful Run" example. The difference is in how the Host Builder Logger is set up.

For the vanilla .NET version, instead of calling `AddHostBuilderLogger`, we call `ConfigureLoggerWithHostBuilderLogger`, which works the same way as the standard `ConfigureLogger`. 

```csharp
host.ConfigureLoggingWithHostBuilderLogger(log =>
{
    log.AddFilter("Microsoft", LogLevel.Information);
});
```

For Serilog, we move away from the static configure-in-code approach and use the optional [inline configuration syntax](https://github.com/serilog/serilog-extensions-hosting#inline-initialization). In a real program, this is where we would apply `IConfiguration` to set up Serilog with external configuration providers, but to keep the demo simple, we simply repeat what was done earlier in code:

```csharp
host.UseSerilogWithHostBuilderLogger((ctx, log) =>
{
    log
        .MinimumLevel.Debug()
        .MinimumLevel.Override("Microsoft", Serilog.Events.LogEventLevel.Information)
        .Enrich.FromLogContext()
        .WriteTo.Console();
});
```

In these cases we add a filter (or "Override" in Serilog terms) that restricts "Microsoft" entries to the `Information` level or higher. When the application is crashing and the log is being dumped as a result of calling `TerminalEmitCachedMessages`, the system sets the default log-level to `Trace` to provide you with maximum details. The Generic Host can output `Debug` level messages, so applying this filter demonstrates that the intercepted logger configuration is working. We'll look at this more in the next sample.

### The "Basic Exception With Config" Sample

This is similar to the "Basic Exception" sample shown earlier where an exception is thrown from `Program.Main`. Since this leads to app termination, the logging system changes the minimum log level to `Trace`. Without the filter added in the previous section, you'd see host `Debug` messages:

![Filteroff](/assets/2020/01-12/filteroff.png)

By filtering "Microsoft" messages to `Information` or higher, we can demonstrate the system was able to re-apply the Serilog configuration delegate on our simplified Generic Host -- the `Debug` messages are suppressed:

![Filteron](/assets/2020/01-12/filteron.png)

### The "Exception During Config" Sample

The final scenario is a failure during the configuration of the logger itself. We've added a log entry at the start of the logger configuration delegate, and we throw an exception before it finishes:

```csharp
host.UseSerilogWithHostBuilderLogger((ctx, log) =>
{
    HostBuilderLogger.Logger.LogInformation("Preparing to configure Serilog.");
    log
        .MinimumLevel.Debug()
        .MinimumLevel.Override("Microsoft", Serilog.Events.LogEventLevel.Information)
        .Enrich.FromLogContext()
        .WriteTo.Console();
    throw new Exception("Logger configuration failure");
});
```

The result looks like this:

![Exceptionduringconfig](/assets/2020/01-12/exceptionduringconfig.png)

There are several interesting things about this output.

First of all, even though we ran a Serilog-based example, the output is from the vanilla .NET logger. Since Serilog configuration failed, the system uses the .NET logger as a fallback. The .NET logger doesn't offer many providers, but under Windows, these messages will end up in the Windows Application Event Log, at a minimum:

![Eventviewer1](/assets/2020/01-12/eventviewer1.png)

![Eventviewer2](/assets/2020/01-12/eventviewer2.png)

![Eventviewer3](/assets/2020/01-12/eventviewer3.png)

Second, notice the "Microsoft"-prefixed `Debug`-level messages are back. Again, this is because Serilog configuration failed, meaning our Override setting wasn't applied to filter out messages lower than the `Information` level.

Finally, notice the "Preparing to configure Serilog" message is output _twice_ -- once just above the red "fail" entry, and again at the bottom of the highlighted area. This is because that log entry is part of our configuration delegate. The first one is the "real" execution while `IHostBuilder.Build` is running. The second one is when our terminal log-writer is trying to re-apply the intercepted log configuration.

That last bit could be a little confusing, but it sure beats no log output at all!

## Usage with ASP.NET Core

Ever since MVC rolled around more than ten years ago, .NET developers have been using `Startup.cs` to configure web-based applications. One of the many reasons I'm not a fan of "conventions" is that they often become defacto "requirements" because few people are aware that they're just a convention and not mandatory in any way. This is true of `Startup.cs` -- you don't really need it at all. But virtually everyone uses it, and at first glance, that poses a problem with this implementation.

If you take a look at the `catch` block in the samples' `Program.Main` you'll find this:

```csharp
catch (Exception ex)
{
    HostBuilderLogger.Logger.LogError("Program.Main caught exception", ex);
    await HostBuilderLogger.Logger.TerminalEmitCachedMessages(args);
}
```

The problem is the `await` keyword -- our venerable `Startup.cs` is not async-friendly. At one time Microsoft seemed to be planning an async option, but I can no longer locate the issue and that plan seems to have fallen by the wayside.

However, the solution is simple enough -- simply `throw` an exception from `Startup` and let `Program.Main` catch it.

## Conclusion

Application startup is a pretty important process, and as anyone who has supported real-world applications knows, when startup fails, having good logs available is usually a critical part of troubleshooting the failure. I understand where the .NET team is coming from, but I'm of the opinion that logging is a special case and I'm surprised they aren't backing down on this one.

This solution is fairly lightweight and very easy to use. Hopefully you'll find it useful.

P.S. My wife insisted I use the line, "They said it couldn't be done..." :)
