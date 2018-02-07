---
title: Serilog Dependency Injection and Easy IP Logging
tags: c# .net asp.net.core serilog dependency.injection
header:
  image: "/assets/2018/02-07/header1280.png"
  teaser: "/assets/2018/02-07/header1280.png"
---

We demonstrate how to inject references to Serilog, the wildly popular .NET logging framework, and we show how to inject an IP-logging variation for website scenarios.

<!--more-->

Serilog, introduced in 2013, quickly gained popularity in the .NET world. With the venerable log4net already falling by the wayside, and with NLog starting to show its age, the stage was set for a new contender. However, Serilog is a `static`-style library which appears to make it very unfriendly to dependency injection. We'll show that isn't necessarily the case.

Then, building on the convenience of DI, we'll demonstrate a helper class that automatically adds the IP address to all log entries for websites that use Serilog.

The code discussed here can be found on GitHub: [Serilog.Dependency.Injection](https://github.com/MV10/Serilog.Dependency.Injection)

## Register the ILogger Factory

Serilog exposes many logging methods such as `Debug` and `Warn` through a `static` class named `Log`. Most of those methods are defined by the `ILogger` interface, but there is one important exception: `Log.Logger` which returns a new (already configured) instance of `ILogger`. In other words, it's a factory. (It's also an example of the Service Locator anti-pattern, the arch-nemesis of correct dependency injection.)

Once you realize a factory is available, registering Serilog as a DI-friendly service is easy. Notice also that we tie into `ProcessExit` to run Serilog's `CloseAndFlush` method.

```
public static IServiceCollection AddSerilogServices(
    this IServiceCollection services, 
    LoggerConfiguration configuration)
{
    Log.Logger = configuration.CreateLogger();
    AppDomain.CurrentDomain.ProcessExit += (s, e) => Log.CloseAndFlush();
    return services.AddSingleton(Log.Logger);
}
```

I'm using this today in a real system. In that implementation, I also use a few overrides. One takes a SQL connection string which is applied to a default configuration (which is then passed to the method shown above), and another applies my preferred default configuration for console-only logging (used by command-line tools and the like, where persistent logging isn't needed or must be piped).

## Demonstration

We'll take a look at using this from a console program. We'll also demo an external library to prove the logger is correctly injected across the entire system. This is borrowed from the Serilog getting-started documentation, where a divide-by-zero exception is logged.

```
public class DivideByZero : IDivideByZero
{
    readonly ILogger log;

    public DivideByZero(ILogger logger) 
        => log = logger.ForContext<DivideByZero>();

    public void CrashThyself(int dividend)
    {
        int divisor = 0;
        try
        {
            log.Debug("Dividing {dividend} by {divisor}", dividend, divisor);
            Console.WriteLine(dividend / divisor);
        }
        catch(Exception ex)
        {
            log.Error(ex, ex.Message);
        }
    }

}
```

You can see we're declaring a dependency on Serilog's `ILogger` in the constructor, and the `IDivideByZero` interface just lists the `CrashThyself` method. It will be registered with a transient lifespan.

Now the stage is set for our console program.

```
static void Main(string[] args)
{
    var services = new ServiceCollection();
    services.AddTransient<IDivideByZero, DivideByZero>;
    services.AddSerilogServices();
    var provider = services.BuildServiceProvider();

    var log = provider.GetService<ILogger>();
    log.Information("ILogger service reference obtained.");

    var divider = provider.GetService<IDivideByZero>();
    divider.CrashThyself(999);

    log.Information("Ready to exit.");
    Console.WriteLine("Press any key...");
    Console.ReadKey();
}
```

Running the code confirms that an `ILogger` was injected into our divide-by-zero service.

![Dividebyzero](/assets/2018/02-07/dividebyzero.png)

## IP Logging

Earlier I mentioned I'm doing this in a large, complicated system. I didn't want to duplicate my Serilog configuration code across several websites, utility programs, and Azure Function apps -- especially since it involves some non-obvious steps like obtaining connection strings stored in Azure Key Vault. I added a registration-helper class to one of the utility libraries in the system and began injecting `ILogger` all over the place. Life was good.

I turned to the task of converting a couple of websites to Serilog, and realized I'd like to add IP address logging to the picture. But by abstracting configuration away into an `Add`-helper, I'd also lost the option of tweaking the configuration. The obvious options were to add another registration-helper scenario specific to websites, use Serilog's `ForContext` method in each and every constructor to (somehow) include IP logging, or worst of all, manually incorporate the IP address into each call to write to the log.

I spent some time trying variations on those first two themes but they were all clumsy to use and generally inelegant. I had also looked at Serilog's "enrichers" -- extensions that add property information, usually during configuration, so at first they didn't seem to help. I didn't want to build a new, custom configuration.

Then I noticed `ForContext` could add an enricher to an already-configured `ILogger`, and thanks to dependency injection, a simple solution fell into place. I created a type of injectable factory class called `LoggerWithIP`. The end result was still an `ILogger` (after a fashion, I couldn't subclass Serilog's concrete implementation since it is sealed), but it is pre-configured with an IP-logging enricher. We also added a generic `<T>` to the class definition so that we could also add a Type property to each logged event.

```
public class LoggerWithIP<T> : ILoggerWithIP<T>
{
    private readonly ILogger logger;
    private readonly IHttpContextAccessor contextAccessor;
    public LoggerWithIP(ILogger logger, IHttpContextAccessor contextAccessor)
    {
        this.logger = logger.ForContext(typeof(T)).ForContext(this as ILogEventEnricher);
        this.contextAccessor = contextAccessor;
    }

    public ILogger Logger => logger;

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        string ip = $"({contextAccessor.HttpContext?.Connection.RemoteIpAddress.ToString() ?? "unknown"})";
        logEvent.AddPropertyIfAbsent(new LogEventProperty("IP", new ScalarValue(ip)));
    }
}
```

The factory declares a dependency on a real `ILogger` instance as well as ASP.NET Core's `IHttpContextAccessor` so that the enricher can obtain the remote IP address.

The interface requires implementing the Serilog interface `ILogEventEnricher`, which is where the `Enrich` method comes from.

```
public interface ILoggerWithIP<T> : ILogEventEnricher
{
    ILogger Logger { get; }
}
```

The sample project on GitHub doesn't include a web application, but usage is simple. Here is part of a controller in my live project. You can see the `ILoggerWithIP<AccountController>` dependency in the constructor. Notice the class-level field is just an `ILogger`, not `ILoggerWithIP`, and it is set by referring to the `ILoggerWithIP.Logger` property.

```
    public class AccountController : Controller
    {
        private readonly IAccountService acctsvc;
        private readonly ILogger log;
        public AccountController(
            IAccountService diAcctSvc, 
            ILoggerWithIP<AccountController> iplogger)
        {
            acctsvc = diAcctSvc;
            log = iplogger.Logger;
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task Logout()
        {
            log.Information("Logout");
            await acctsvc.SignOut();
        }

        // ... remainder omitted
    }
```

When logged to SQL Server, the Properties column shows an "IP" property with my localhost `127.0.0.1` address and a "SourceContext" property with my controller's namespace and type.

![Database](/assets/2018/02-07/database.png)

## Conclusion

Serilog has a rightfully-deserved reputation for high-performance and ease-of-use. With a very small amount of extra effort, it plays equally-well with any consumer that relies on dependency injection, whether it's an ASP.NET Core site, or a console program, or even an external library.
