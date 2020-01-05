---
title: Lifecycle of Generic Host Background Services
tags: c# .net generichost backgroundservice
header:
  image: "/assets/2020/01-05/header1280.png"
  teaser: "/assets/2020/01-05/header1280.png"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2020/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2020/mm-dd/pic.jpg)
---

Cleaner startup by separating execution from initialization.

<!--more-->

.NET Core 2.1 introduced something called the Generic Host, which is a model for hosting `Task`-based asynchronous services side-by-side. For the .NET Core 3.0 release, ASP.NET was migrated off the old (but similar) `WebHost` model to Generic Host. Like so many other parts of .NET Core, there isn't anything about Generic Host which is inherently tied to ASP.NET Core, but for some reason the team still slots is under that repository.

I've started using Generic Host for practically everything, even quick-and-dirty one-off console test apps. It greatly simplifies the use of standardized configuration, dependency injection, and logging -- all things that you wouldn't normally associate with "quick and dirty" tests. However, it does create a bit of overhead in that your own code also must be a hosted service to take advantage of these features.

.NET Core provides two ways to create your own hosted service. You can implement the [`IHostedService`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostedservice?view=dotnet-plat-ext-3.1) interface, or you can derive from the [`BackgroundService`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.backgroundservice?view=dotnet-plat-ext-3.1) abstract class. However, the out-of-the-box experience allows for start-up race conditions if your services have dependencies. I will demonstrate an alternative abstract class which avoids this by providing discrete initialization, execution, and shutdown activities.

There is no repository associated with this article.

## BackgroundService

The `IHostedService` interface requires you to implement two methods: `StartAsync` and `StopAsync`. There are quite a few questions on StackOverflow about the correct way to interpret these, but we'll skip over DIY implementation of `IHostedService` since .NET's own `BackgroundService` abstract class already [implements](https://github.com/aspnet/Extensions/blob/master/src/Hosting/Abstractions/src/BackgroundService.cs) that interface in the most "obvious" way. If your service derives from this, you need only implement `ExecuteAsync`.

For our first example, we'll write a short console program that registers two hosted services:

```csharp
public static async Task Main(string[] args)
{
    await Host.CreateDefaultBuilder(args)
    .ConfigureLogging(builder => builder.SetMinimumLevel(LogLevel.Warning))
    .UseConsoleLifetime() // Ctrl+C support
    .ConfigureServices((ctx, svc) =>
    {
        svc.AddHostedService<Loop250ms>();
        svc.AddHostedService<Run5sec>();
    })
    .RunConsoleAsync();
}
```

The `ExecuteAsync` method accepts a `CancellationToken` which tells your code when the service is being terminated. Our `Loop250ms` class simply writes console output every 250ms until the token is cancelled:

```csharp
public class Loop250ms : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        Console.WriteLine("Loop250ms.ExecuteAsync");
        while(!stoppingToken.IsCancellationRequested)
        {
            Console.WriteLine("(loop)");
            await Task.Delay(250);
        }
        Console.WriteLine("Loop250ms.ExecuteAsync token cancelled");
    }
}
```

One of the things which isn't clearly documented (in my opinion) is how to end an application which is built around Generic Host. It turns out that the Generic Host registers a DI service called `IHostApplicationLifetime` which, among other things, provides a `StopApplication` method. Our `Run5sec` service provides a 5-second countdown at 1-second intervals, then ends the program:

```csharp
public class Run5sec : BackgroundService
{
    private readonly IHostApplicationLifetime appLifetime;

    public Run5sec(IHostApplicationLifetime appLifetime)
    {
        this.appLifetime = appLifetime;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        Console.WriteLine("Tick1000ms.ExecuteAsync");
        for (int i = 5; i > 0; i--)
        {
            if (stoppingToken.IsCancellationRequested) break;
            Console.WriteLine($"tick {i}");
            await Task.Delay(1000);
        }
        Console.WriteLine("Tick1000ms calling StopApplication");
        appLifetime.StopApplication();
    }
}
```

Easy, right? A test-run looks innocent enough:

```
Loop250ms.ExecuteAsync
(loop)
Run5sec.ExecuteAsync
tick 5
(loop)
(loop)
(loop)
tick 4
(loop)
(loop)
(loop)
(loop)
tick 3
(loop)
(loop)
(loop)
(loop)
tick 2
(loop)
(loop)
(loop)
(loop)
tick 1
(loop)
(loop)
(loop)
(loop)
Run5sec calling StopApplication
Loop250ms.ExecuteAsync token cancelled
```

## Off to the Races

The problem is exhibited in the first four lines of that console output. The `Loop250ms` service begins executing before the `Run5sec` service starts. It makes no difference in this trivial example, but I was recently writing a program that used several `BackgroundService` instances to monitor complex, long-running external processes. They also relied on another service to communicate with a different server. Occasionally that service would have problems starting, but this didn't always happen before the other services started running and tried to use it.

The problem was a classic race condition. There was no coordination between the services. I could have exposed a `Status` property or something along those lines, but this lack of start-up coordination felt wrong to me. I decided to investigate the interactions between the Generic Host, `IHostedService` instances, and `BackgroundService` in particular, and decided I could improve on the model by providing explicit calls to initialize, execute, and stop a hosted service.

## CoordinatedBackgroundService

I called the class `CoordinatedBackgroundService` because initialization is coordinated across all hosted services (those which derive from this class, at any rate). I believe this was probably the original intent of the `StartAsync` method for `IHostedService` -- perform quick initialization work, then wait for `Host` to signal application start-up. This suspicion is further supported by another `IHostApplicationLifetime` feature, the `ApplicationStarted` event. This isn't a "real" .NET event, it is actually a `CancellationToken`, but it's used like an event, which I'll explain shortly.

The `Host` startup process looks clear to me. First, it loops through all registered `IHostedService` instances and executes their `StartAsync` methods ([source](https://github.com/aspnet/Extensions/blob/master/src/Hosting/Hosting/src/Internal/Host.cs#L47)), then it fires the `ApplicationStarted` event by cancelling that token ([source](https://github.com/aspnet/Extensions/blob/master/src/Hosting/Hosting/src/Internal/Host.cs#L54)). The application can use the token's `Register` method to prepare a callback that is executed -- effectively, an event-handler. This is how I separate the initialization logic from the execution logic.

To my way of thinking, this is also a match for the rest of the Generic Host's use of the builder pattern. Microsoft doesn't provide access to services registered for dependency injection until the one-time call to `Build`, and to me, the `StartAsync` calls seem to be built around the same concept -- get the entire `Host`-owned infrastructure up and running before the application tries using any of it.

The resulting implementation is very similar to .NET's own `BackgroundService` with a few extra steps:

```csharp
public abstract class CoordinatedBackgroundService : IHostedService, IDisposable
{
    private readonly CancellationTokenSource appStoppingTokenSource = new CancellationTokenSource();

    protected readonly IHostApplicationLifetime appLifetime;

    public CoordinatedBackgroundService(IHostApplicationLifetime appLifetime)
    {
        this.appLifetime = appLifetime;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine($"IHostedService.StartAsync for {GetType().Name}");
        appLifetime.ApplicationStarted.Register(
            async () => 
            await ExecuteAsync(appStoppingTokenSource.Token).ConfigureAwait(false)
        );
        return InitializingAsync(cancellationToken);
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine($"IHostedService.StopAsync for {GetType().Name}");
        appStoppingTokenSource.Cancel();
        await StoppingAsync(cancellationToken).ConfigureAwait(false);
        Dispose();
    }

    protected virtual Task InitializingAsync(CancellationToken cancelInitToken)
        => Task.CompletedTask;

    protected abstract Task ExecuteAsync(CancellationToken appStoppingToken);

    protected virtual Task StoppingAsync(CancellationToken cancelStopToken)
        => Task.CompletedTask;

    public virtual void Dispose()
    { }
}
```

Readers familiar with `async/await` will probably question the lambda registered to execute upon token cancellation:

```csharp
async () => await ExecuteAsync(...)
```

This is the dreaded `async void` pattern. Microsoft tells us this is only acceptable for event-handlers. Although `Register` is technically a callback and not formally defined as a .NET event, under the hood they're the same thing (and certain older Microsoft docs sometimes say "event-handlers and callbacks" -- I suspect callbacks were dropped in response to `Func<>` becoming widely available).

A `Task` exists primarily to track the result of a given operation. The reason `async void` is acceptable for event-handlers and callbacks is that the _caller_ doesn't care about the result of the operation being executed. They're fire-and-forget. However, this usage pattern does introduce a serious (but easily addressed) issue that we'll discuss later.

## Coordinated Test

Modifying our demo services to use the new `CoordinatedBackgroundService` class is very straightforward. This simple demo doesn't require initialization or special shutdown handling, but we'll show console output just to demonstrate the lifecycle at work:

```csharp
public class Loop250ms : CoordinatedBackgroundService
{
    public Loop250ms(IHostApplicationLifetime appLifetime)
        : base(appLifetime)
    { }

    protected override Task InitializingAsync(CancellationToken cancelInitToken)
    {
        Console.WriteLine("Loop250ms.InitializingAsync");
        return Task.CompletedTask;
    }

    protected override async Task ExecuteAsync(CancellationToken appStoppingToken)
    {
        Console.WriteLine("Loop250ms.ExecuteAsync");
        while (!appStoppingToken.IsCancellationRequested)
        {
            Console.WriteLine("(loop)");
            await Task.Delay(250);
        }
        Console.WriteLine("Loop250ms.ExecuteAsync token cancelled");
    }

    protected override Task StoppingAsync(CancellationToken cancelStopToken)
    {
        Console.WriteLine("Loop250ms.StoppingAsync");
        return Task.CompletedTask;
    }
}
```

The `Run5sec` service is modified in a similar way. Although we must inject `IHostApplicationLifetime` to pass to the base class, we don't store a reference ourselves to call `StopApplication`, we now inherit the stored reference from the base class:

```csharp
public class Run5sec : CoordinatedBackgroundService
{
    public Run5sec(IHostApplicationLifetime appLifetime)
        :base(appLifetime)
    { }

    protected override Task InitializingAsync(CancellationToken cancelInitToken)
    {
        Console.WriteLine("Run5sec.InitializingAsync");
        return Task.CompletedTask;
    }

    protected override async Task ExecuteAsync(CancellationToken appStoppingToken)
    {
        Console.WriteLine("Run5sec.ExecuteAsync");
        for (int i = 5; i > 0; i--)
        {
            if (appStoppingToken.IsCancellationRequested) break;
            Console.WriteLine($"tick {i}");
            await Task.Delay(1000);
        }
        Console.WriteLine("Run5sec calling StopApplication");
        appLifetime.StopApplication(); // inherited
    }

    protected override Task StoppingAsync(CancellationToken cancelStopToken)
    {
        Console.WriteLine("Run5sec.StoppingAsync");
        return Task.CompletedTask;
    }
}
```

No changes to `Program.Main` are required. A quick test-run (with the intermediate tick/loop output removed) shows an orderly start-up sequence with discrete initialization actions running before execution begins:

```
IHostedService.StartAsync for Loop250ms
Loop250ms.InitializingAsync
IHostedService.StartAsync for Run5sec
Run5sec.InitializingAsync
Run5sec.ExecuteAsync
tick 5
Loop250ms.ExecuteAsync
(loop)
...
(loop)
Run5sec calling StopApplication
IHostedService.StopAsync for Run5sec
Run5sec.StoppingAsync
IHostedService.StopAsync for Loop250ms
Loop250ms.StoppingAsync
```

## Exception Handling

Those services regularly check the `IsCancellationRequested` property on the `appStoppingToken` passed to the `ExecuteAsync` method, which allows them to gracefully exit when the application is shutting down. However, in a real application that property usually creates a lot of nested `if` statements or complex combinations of `while` loops and `break` statements. To remedy this, .NET provides a [`ThrowIfCancellationRequested`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken.throwifcancellationrequested?view=netcore-3.1) method which, as the name suggests, throws an `OperationCancelledException` when the token is canceled. You should catch this exception, but what happens if we don't?

```csharp
protected override async Task ExecuteAsync(CancellationToken appStoppingToken)
{
    Console.WriteLine("Loop250ms.ExecuteAsync");
    while(true)
    {
        appStoppingToken.ThrowIfCancellationRequested();
        Console.WriteLine("(loop)");
        await Task.Delay(250);
    }
    //Console.WriteLine("Loop250ms.ExecuteAsync token cancelled");
}
```

Throwing an exception is the only way to exit this loop now. (The last line is commented out since `while(true)` made that code unreachable.) As you probably expect, running this version results in an unhandled exception:

![Unhandled Exception](/assets/2020/01-05/unhandled_exception.png)

Back in the .NET Framework 2.0 days, Microsoft changed how .NET responds to unhandled exceptions. The 1.0 release would silently ignore them, which meant the application could chug along with corrupted state, potentially causing other unnoticed problems. For the 2.0 release, they changed the runtime such that the application is immediately terminated when an unhandled exception is detected, and that behavior still exists today in .NET Core.

As mentioned earlier, the purpose of `Task` is to report back the result of some operation, and that includes exceptions. The `Task` actually detects whether the results are "observed" (meaning some other process queries result-oriented properties such as `Task.Exception`). With our use of `async void` to invoke our `ExecuteAsync` method, there is no `Task`, so there isn't any process that will ever receive the results of the `Task`.

`Task` is also the mechanism through which the runtime "bubbles" exceptions to potential parent `catch` blocks. This means you can't put `catch(OperationCancelledException)` in `Main` -- we're running the `async void` lambda, so there is no `Task`.

I thought about showing some extension-method tricks which can be performed to explicitly ignore `Task` exceptions, but they would have just been distractions -- bad practice for exactly the reason the .NET team changed the runtime behavior almost 18 years ago. 

The bottom line is that any real-world background processing service should gracefully handle exceptions internally. Normally `OperationCancelledException` would be caught and ignored since it's expected, but for demo purposes we'll simply log all exceptions to the console:

```csharp
protected override async Task ExecuteAsync(CancellationToken appStoppingToken)
{
    try
    {
        Console.WriteLine("Loop250ms.ExecuteAsync");
        while (true)
        {
            appStoppingToken.ThrowIfCancellationRequested();
            Console.WriteLine("(loop)");
            await Task.Delay(250);
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Loop250ms caught {ex.GetType().Name}");
    }
    finally
    {
        Console.WriteLine("Loop250ms exiting");
    }
}
```

The end of a test-run with the service exception handler in place looks like this:

```
...
(loop)
Run5sec calling StopApplication
IHostedService.StopAsync for Run5sec
Run5sec.StoppingAsync
IHostedService.StopAsync for Loop250ms
Loop250ms.StoppingAsync
Loop250ms caught OperationCanceledException
Loop250ms exiting
```

It was actually necessary to add a 250ms delay to the end of `Main`, otherwise most of the time the console service, which is also an `IHostedService` would exit before `Loop250ms` had time to write that last bit of output. But that's strictly a demo consideration.

## Conclusion

I've run into at least one person who disagrees with my interpretation (mostly he was upset by the use of `async void`), but as long as you have good exception handling practices in place, I can't see any issues with this approach, and it certainly eliminated real-world race conditions.

The system where I use this has been tested simultaneously handling tens of interdependent hosted services with no crashes or leaks, so I'm satisfied with the results.

If it helps you, too, drop me a note in the comments!
