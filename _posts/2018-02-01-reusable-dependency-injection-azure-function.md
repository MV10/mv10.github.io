---
title: Reusable Dependency Injection for Azure Function Apps
tags: personal c# .net azure ai odm java unity
header:
  image: "/assets/2018/02-01/header1280.png"
  teaser: "/assets/2018/02-01/header1280.png"
---

We show how to turn an Azure Function dependency injection experiment into a reusable library for any Azure Function project.

<!--more-->

Code for this article can be found [here](https://github.com/MV10/Azure.Functions.Dependency.Injection).

Dependency injection has become a defacto standard technique for writing testable, loosely-coupled applications and libraries. And yet, Azure Function users have been waiting for almost two years for DI support. The original feedback was posted in [2016](https://feedback.azure.com/forums/355860-azure-functions/suggestions/15642447-enable-dependency-injection-in-c-functions), and it wasn't until a few days ago that Microsoft linked it to an open GitHub [issue](https://github.com/Azure/Azure-Functions/issues/299) that has been open since 2017.

I have a project with a large, complex DI-driven library which must be shared between several web apps and some utilities that will be deployed as a handful of Azure Function apps. I knew DI was going to be a challenge. Since Azure Functions run on top of the older WebJob hosting system, I spent the better part of two days trying to figure out how to achieve DI by wiring up `IServiceCollection` as a WebJob `IJobActivator`. However, it doesn't appear that a Function can reach the underlying WebJob host, which means the activator can't be connected to the `JobHostConfiguration`.

Shortly after concluding the WebJob activator approach was a dead end, I happened to find the GitHub discussion. In that discussion, two users contributed their interesting experiments in Function DI support. Last October, basic DI was accomplished by [BorisWilhelms](https://github.com/BorisWilhelms/azure-function-dependency-injection), but that implementation tightly tied the injection system to service registration. Just a few days ago, user [yuka1984](https://github.com/yuka1984/azure-function-dependency-injection) forked that project to add an inspired custom Function trigger that moves DI service registration out of the injection system and into the Function app.

I spent some time reviewing their work and realized they were very close to a reusable library. I've done quite a bit of cleanup and refactoring of their code, and I made quite a few changes to the code that demonstrates how the different use-cases work, but the basic concept is unchanged and they deserve all the credit for their insights.

## Using the Attributes

Azure Functions are `static` classes, which makes them incompatible with the normal constructor-based DI. BorisWilhelms' solution was to create an `[Inject]` attribute which can be applied to Function parameters to declare dependencies. From there, yuka1984 created an injection-configuration custom Function trigger and changed the `[Inject]` attribute to also reference the function that registers the DI services. 

The result is very easy to use.

```
public static class DemoFunction
{
        [FunctionName("QueryUsefulData")]
        public static async Task<HttpResponseMessage> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpRequestMessage req,
            [Inject("Registration")] IUsefulDataProvider data)
        {
            return req.CreateResponse(data.GetUsefulData());
        }

        [FunctionName("Registration")]
        public static void Config([InjectorConfigTrigger] IServiceCollection services)
        {
            services.AddSingleton<IUsefulDataProvider, UsefulDataProvider>();
        }
}
```

Although it isn't obvious from looking at the code above, one of the really interesting ideas in yuka1984's solution is to use a `Lazy<>` approach to trigger activation. Thus, the registration is not actually processed until it is used, and the registration is only processed once.

Since Azure Function apps are really just C# class libraries as far as Visual Studio is concerned, treating the end-result as a library was trivial. I just created a new Function project to act as the "main" project (the one that actually runs Functions) and referenced the DI project. After a recompile, the attributes just work.

## Verifying Full-Graph Injection

Since my real-world use-case is a large, complex, DI-dependent utility library, I modified their demo code to demonstrate to my own satisfaction that dependencies declared by the injected objects would also be correctly resolved. I also made a change I felt better demonstrated the different service lifetimes.

In yuka1984's version of the demo `IGreeter` service, he thoughtfully added a counter to demonstrate that the `AddSingleton` registration was really exhibiting singleton behavior. My modification was to simply create an `ICounter` which gets injected into the concrete `CountUpGreeter` class, and sure enough, at runtime the correct dependencies are injected even in the separate class library.

```
public class Counter : ICounter
{
    public int count { get; set; } = 0;
}


public class CountUpGreeter : AGreeter, IGreeter
{
    private readonly ICounter counter;
    public CountUpGreeter(ICounter counter) : base()
    {
        this.counter = counter;
    }

    public new string Greet()
    {
        return $"Hello World, counter {counter.count++}, created {constructed}";
    }
}
```

## Demonstrating Singleton Services

When you run the demo locally using the Azure Functions CLI, the console window dumps the URLs for any HTTP-triggered Functions defined in the application. Executing those functions is as easy as pointing your browser at them.

![Functionurls](/assets/2018/02-01/functionurls.png)

You can see there are two singleton URLs. Calling each of these proves they do indeed reference a singleton instance of the service thanks to yuka1984's addition of an incrementing counter.

![Singleton](/assets/2018/02-01/singleton.png)

## Demonstrating Transient Services

Services with a transient lifetime are even defined by the [documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection#service-lifetimes-and-registration-options) as great choices for the kind of services Function apps are meant to provide.

> Transient lifetime services are created each time they're requested. This lifetime works best for lightweight, stateless services.

I added a `constructed` property to the `AGreeter` base class. It is set to the date and time when the constructor is executed, and is emitted at the end of the "greeting" string as `ms:xxx`. You can see that multiple injections of `IGreeter` registered with `AddTransient` show a different creation time on every call.

![Transient](/assets/2018/02-01/transient.png)

## Demonstrating Scoped Services

The documentation says this about registering with `AddScoped`:

> Scoped lifetime services are created once per request.

Unfortunately yuka1984's scoped demonstration didn't actually demonstrate scoped behavior. It did register `IGreeter` using `AddScoped` but that alone won't prove that the service is created once per request. To do this, some other service would have to be registered which also injected `IGreeter` during the same request. When `IGreeter` is registered as scoped, both injected services will show the same `constructed` millisecond timestamp from their references to `IGreeter`, but should `IGreeter` be registered as transient, each service will receive a new copy of `IGreeter` and exhibit different timestamps.

To show this, I created scoped and non-scoped variants along with an `IGreeterConsumer` service (which is always registered as transient; its lifecycle isn't relevant, only the lifecycle of the `IGreeter` upon which it depends). You may have to run the non-scoped version a few times before you get different timestamps, it's possible for both instances to be created within the same millisecond.

![Scoped](/assets/2018/02-01/scoped.png)

## Service Registration Helpers

These demonstrations are relatively simple, but in a real-world class library, the dependencies may be extensive and difficult for library-consumers to understand and register. Because of this, I prefer to create registration extensions that service-consumers can rely upon to ensure all the necessary interfaces are registered for injection. Anyone who uses ASP.NET Core will recognize the many `services.Add` commands, which are an example of this same approach.

For the Function DI demonstration project, I've created a `GreeterInjectionExtensions` class which provides three methods to register `IGreeter` as singleton, transient, or scoped lifetimes. The `IGreeterConsumer` registration extension only supports transient registration since that's all the demonstration requires.

For my real-world class library, I have to consider different use-cases. Web applications have broad coverage and a relatively long lifecycle, and so it is appropriate to provide a registration extension which literally registers the entire library. However, Azure Function apps have very different needs, so it was expedient to create additional extensions specific to each use-case, pulling in only the library interfaces required by those scenarios.

## The Right Place to Register Services

Earlier I mentioned that I didn't like the way service registration was tightly tied to the rest of the dependency code in the original example from BorisWilhelms. When I first reviewed yuka1984's changes, I also initially questioned service registration within the Function class. 

I spent a few hours considering ways to move registration to a separate class within the Function app, similar to the way ASP.NET Core relies upon `Startup.cs` to register services. However, executing a Function in another class turns out to be very complicated (and maybe impossible) in a Function app. The class `InjectorConfigTrigger` has a method called `AddConfigExecutor` which is defines the `Lazy<>` reference to the configuration Function. In the source, you'll find the following line of code:

```
executor.TryExecuteAsync(
    new TriggeredFunctionData() {TriggerValue = services}, 
    CancellationToken.None).GetAwaiter().GetResult();
```

That causes the WebJob library to execute a Function. `TriggeredFunctionData` optionally allows for a custom invoker to execute the target, but an invoker is fairly complex (take a look at the [code](https://github.com/Azure/azure-webjobs-sdk/blob/master/src/Microsoft.Azure.WebJobs.Host/Executors/FunctionInvoker.cs) for the default invoker), and more importantly, right now Microsoft has the `IFunctionInvoker` interface declared as `internal`, so I couldn't roll my own even if I really wanted to tackle that headache.

Fortunately, after thinking about it for awhile, I've concluded that service registration within each Function class is a good fit for the Function application model.

As mentioned in the previous section, when a web application uses my utility library, there is a good chance most of the library will be used at some point. It is reasonable to just register everything in one shot. The web application runtime model is relatively loose about memory usage and in comparison to the default five-minute maximum Function lifecycle, a web application hangs around for a very long time.

On the other hand, Functions are meant to be very self-contained and very memory- and processing-efficient. These concerns should be reflected in the dependencies the Functions require and how related services are registered. Therefore, it makes sense to me that service registration should also be relatively localized to the Function itself. Yes, it is still important for the library to provide helpers so that the Function isn't too tightly tied to internal library dependency requirements, but registrations should be kept as streamlined as possible.

## Multiple Composition Roots

There is one unusual side-effect of this trigger-based injection configuration: the application effectively has multiple [composition roots](http://blog.ploeh.dk/2011/07/28/CompositionRoot/). Normally, an application registers services once at startup. But the nano-service orientation of a Function application means that each invocation of the application potentially has different, very tightly-constrained requirements. As a result, a Function app which provides several logically-related services may have very different runtime requirements.

The implementation of service registration as separate `Lazy<>` instances means each registration Function acts as a completely unique composition root. In fact, a single Function can reference more than one registration trigger Function (or if you prefer, more than one composition root), even providing references to the same interface but registered with different lifetimes from other injected references on the same Function. It's hard to imagine a use for this last bit, but the flexibility is certainly interesting.

## Conclusion

Thanks to the efforts of BorisWilhelms and yuka1984, we have an easy-to-use, highly flexible, reusable dependency injection library for Azure Function apps. In my opinion, they've handed Function DI to Microsoft on a platter. Here's hoping we see official support soon.
