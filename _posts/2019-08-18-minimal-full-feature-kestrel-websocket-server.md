---
title: A Minimal Full-Feature Kestrel WebSocket Server
tags: c# .net asp.net asp.net-core websocket
header:
  image: "/assets/2019/08-18/header1280.png"
  teaser: "/assets/2019/08-18/header1280.png"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2019/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2019/mm-dd/pic.jpg)
---

A Kestrel WebSocket server with realistic features using .NET Core 3.

<!--more-->

In my most recent [article]({{ site.baseurl }}{% post_url 2019-08-17-how-to-close-websocket-correctly %}) about my WebSocket sample projects, I noted that the venerable .NET `HttpListener` class is on minimal life support and destined to be [deprecated](https://github.com/dotnet/platform-compat/issues/88). I wasn't too happy about this, as the class is very easy to use and has little overhead -- especially compared to the ASP.NET Core and/or Kestrel stack, which is a bewildering maze of interdependent NuGet packages (and poorly documented from the standpoint of trying to do anything other than a run-of-the-mill website).

Originally (yesterday) I was hoping my "minimal" example would document the specific packages needed by a project of this type. That was an exercise in futility. Almost every line of code I wrote had to pull in one or two other packages, each of which pulled in other dependences. After a few hours of this, it became apparent that the ASP.NET Core world is so heavily oriented to Big Framework thinking that it isn't realistic to try to use much of it in a piecemeal fashion. That goal was basically a failure, but once I gave up and started from a new (but empty) ASP.NET Core project, the rest fell into place rather easily. There isn't a lot of code, but there is a lot of plumbing in the mix that probably isn't really necessary. On the other hand, I ultimately wrote less code, and the overall architecture is much more flexible and has a cleaner feel to it.

As a result, this article still demonstrates how to set up a very bare-bones Kestrel-based WebSocket server, even though the starting point ended up being a full-blown ASP.NET Core project template. I have added a new project to the WebSocketExample solution found in the same GitHub repository used by my previous posts about WebSockets, namely [WebSocketExample](https://github.com/MV10/WebSocketExample). Note that this currently requires Visual Studio 2019 16.3 Preview 2 and .NET Core 3.0 Preview 8, mainly because I use the new ASP.NET Core 3.0-specific hosting model.

## The Program and Startup Classes

Like my earlier articles, the `Program` class declares a couple of constants. In my `Main` method I handle the basic server setup. I've never understood why Microsoft's templates put the builder stuff in a separate method. It's literally a few lines of code. Nothing exciting here:

```c#
public static void Main(string[] args)
{
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseUrls(new string[] { @"http://localhost:8080/" });
            webBuilder.UseStartup<Startup>();
        })
        .Build()
        .Run();
}
```

The `Startup` class warrants discussion. It's pretty simple compared to your typical ASP.NET Core application, but there are several uncommon entries:

```c#
public class Startup
{

    public void ConfigureServices(IServiceCollection services)
    {
        // register our custom middleware since we use the IMiddleware factory approach
        services.AddTransient<WebSocketMiddleware>();

        // register the background process to periodically send a timestamp to clients
        services.AddHostedService<BroadcastTimestamp>();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // enable websocket support
        app.UseWebSockets(new WebSocketOptions
        {
            KeepAliveInterval = TimeSpan.FromSeconds(120),
            ReceiveBufferSize = 4 * 1024
        });

        // add our custom middleware to the pipeline
        app.UseMiddleware<WebSocketMiddleware>();
    }
}
```

The `ConfigureServices` method includes registering a `WebSocketMiddleware` class as a transient-scoped service and the registration of a hosted service called `BroadcastTimestamp`. It's unusual to register an ASP.NET Core request-processing-pipeline middleware class with the dependency-injection container, but this is required when using the interface-based `IMiddleware` factory model. I'm not a fan of the so-called "convention" (aka "magic methods") approach to writing software which has been creeping into ASP.NET over the years. As for the hosted service, `BroadcastTimestamp` is a background process which broadcasts a timestamp to all connected clients every 15 seconds. If you've read my earlier WebSocket articles, you've already seen a less elegant approach to this same feature.

The `Configure` method calls `app.UseWebSockets` which enables generic WebSocket support within ASP.NET Core, then registers a single middleware pipeline handler, our very own `WebSocketMiddleware`. There is nothing else handled by this server -- no MVC, no SignalR, no static file serving, no routing -- you get the idea. This is as basic as it gets for Kestrel while still performing real work.

## WebSocket Ping-Pong

The WebSocket protocol defines a ping-pong keepalive mechanism, whereby one party sends a "ping" frame, and the other party is supposed to respond with a "pong" frame. And in fact, the .NET Core implementation of WebSockets does automatically support this. In the `UseWebSockets` call in the `Startup` class, you see a `KeepAliveInterval` which you'd naturally expect keeps your server and clients connected. The [`KeepAliveInterval` documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/websockets?view=aspnetcore-3.0#configure-the-middleware) even says this option controls the ping-pong frequency. Unfortunately it doesn't work so well. There are some fairly serious problems with WebSockets in the 3.0 stack (and the basic Kestrel architecture) which prevents this from working -- to the point that the built-in support seems almost pointless.

For example, if your Kestrel server runs behind IIS, it turns out that [IIS intercepts the ping](https://github.com/aspnet/AspNetCore/issues/2464#issuecomment-484206910) and your WebSocket server never sees a client's pong response. One impact of this is that your server never really knows [if the client has disconnected](https://github.com/aspnet/AspNetCore.Docs/issues/9053#issuecomment-430358764). It also appears there are some [problems](https://github.com/aspnet/AspNetCore/issues/2464#issuecomment-390790434) with the default IIS configuration that may completely disable WebSocket ping-pong frames.

Worse, it will probably _never_ be fully fixed. One area that is completely out of the hands of Microsoft is proxy servers, which commonly fail to pass-through ping-pont frames. Unless you work in a relatively small organization with 100% internally-facing projects, you probably can't guarantee there won't be a non-compliant proxy server somewhere in the network.

The solution is beyond the scope of this article, but for any serious project, the bottoml ine is that you should expect to implement your own keep-alive logic. Your client should hit the server periodically, and the server should sweep the list of connected clients, dropping those which haven't phoned home within some reasonable threshold.

## The WebSocketMiddleware Class

The meat of this project is the `WebSocketMiddleware` class. Earlier you saw how it was registered for DI and added to the request pipeline. I chose to use the [optional](https://github.com/aspnet/AspNetCore.Docs/issues/11443#issuecomment-478017538) `IMiddleware` interface, which is a [factory-based](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/extensibility?view=aspnetcore-3.0) approach. The interface is simple, requiring just one method with the following signature:

`async Task InvokeAsync(HttpContext context, RequestDelegate next)`

This type of middleware is transient scoped, meaning the instance only exists for the duration of the request -- in theory. For the sake of simplifying the example, we still reply heavily on static fields and methods, but the class itself cannot be static. Also, Microsoft's own documentation about WebSockets states that the middleware pipeline must be kept alive for as long as the socket is in use, so in this case the instance lingers for much longer than would normally be the case for a transient scoped service.

There are serveral key differences between the Kestrel implementation and the WebSocket servers shown in the earlier articles.

Since we're running under the full-blown ASP.NET Core stack, dependency injection is available, and the constructor requires an instance of an `IHostApplicationLifetime` object. This interface is new in ASP.NET Core 3.0, although it's very similar to earlier hosting implementations such as `WebHost`. The interface exposes a few `CancellationToken` properties which, when cancelled, signal lifecycle events like startup, shutdown initiation, and shutdown completion. The thread-safe `CancellationToken` has become the de rigueur replacement for simple events in .NET Core of late. In this case, we `Register` a handler which is called at application shutdown to cleanly disconnect all clients. Other than setting up this handler, the shutdown is exactly like the `CloseAllSocketsAsync` method introduced in the previous article.

The big difference between the Kestrel version and previous version is the `InvokeAsync` method. In the ASP.NET middleware pipeline model, various middleware participants hand off `HttpContext` until one of the middleware classes can handle the request. The documentation usually shows a middleware class executing `await next(context)` as the final statement, which hands off processing to the next piece of middleware in the stack. In this case, the handoff only happens if we're ignoring the request (we send back the simple echo-client web page for `text/html` requests, or we upgrade HTTP to a WebSocket, or we ignore the request). In that case, the default framework handlers step in.

Other than that, the `WebSocketMiddleware` class is a simplified version of the code used in previous articles.

## The BroadcastTimestamp Hosted Service

ASP.NET Core has been moving towards more and more generic hosts -- a side-effect of features like WebJobs which have been available for years on Azure. In earlier articles, our sample servers demonstrated cross-thread interaction with a simple timer-delayed loop in `Program.Main`, but for this version, we'll take advantage of a hosted service that runs on a background thread.

As luck would have it, Microsoft's own documentation shows how to create a [timed background task](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-3.0&tabs=visual-studio#timed-background-tasks). For this example we don't use the logger and there are a few other very minor changes, but it's largely the same thing shown in the docs.

Like the earlier articles, it just pumps out timestamp messages to all connected WebSocket clients every 15 seconds.

## Secure Connections With SSL/TLS/WSS

Since `HttpListener` was relegated to legacy status before Microsoft was finished porting it to .NET Core, they didn't bother to add SSL/TLS support. Unless you want to invest in the dead-end .NET Framework stack, this means Kestrel is the _only_ viable option for a real-world project. These examples are easy to test with a secure connection. Just change the `UseUrls` entry in the Kestrel `Program` class to `https`, and change the `StartAsync` target URI to `wss` in the client's `Program` class.

Apart from that, they're 100% identical.

## Real-World Architecture

As I've emphasized in previous articles, these examples are not production-ready code. 

The thread-safe collections use the "Try" pattern, which means adding to, removing from, and reading from collections may not always succeed. Sometimes this is handled in the examples where it won't unnecessarily complicate the code, but in general this requires more effort in real projects.

Exception handling is crucial when working with WebSockets. Connections are practically guaranteed to fail in production usage. You should aggressively test your applications for unexpected disconnects. Literally pull the plug on your clients and your servers to ensure failure on the other end of the connection is handled gracefully. Figure out the client re-connect behavior your system requires. Clean up associated resources, including disposing WebSocket references. Check the special [`WebSocketError` enum](https://docs.microsoft.com/en-us/dotnet/api/system.net.websockets.websocketerror?view=netcore-3.0) exposed as the `WebSocketErrorCode` property when a `WebSocketException` is thrown.

Authentication is something the WebSocket protocol ignores completely. The protocol discusses support for tokens in the header, but doesn't go into any detail beyond that. There are many articles on the Internet about different ways to handle this, and the solution will depend heavily on the types of clients you support. Unfortunately the .NET world's move to Big Frameworks means it's difficult to find good information for server-side solutions that don't assume you're using SignalR, but it's not too hard to generalize from non-.NET approaches. Assume you'll be writing more pipeline middleware handlers to add headers to the response or read and validate them from requests.

For publicly-accessible sites, WebSockets are an attractive attack-surface. Good exception handling is a critical part of safe usage. Manage your buffers carefully. Authenticate every request. Common guidance is to simply ignore (but log) invalid requests. Don't even respond with an error, just drop the connection as quickly as you're able to identify it as invalid.

As explained earlier in the ping-pong topic, plan to build your own keepalive mechanism. The troubles using real ping-pong frames from Kestrel are several years old and are partly out of Microsoft's hands (particularly when it comes to proxy servers). I wouldn't expect ping-pong keepalive to suddenly start working any time soon.

Since you'll pay the cost of the overhead of the ASP.NET Core framework anyway, _use it_ ... Dependency injection is great. Extensible config is great. Any non-trivial WebSocket server will probably need admin features anyway ... put these capabilities to good use. (I'm dying to try a Blazor server-side app with WebSocket upgrade on a specific URI path...)

My examples make heavy use of static classes and methods to simplify the sample code, but real implementations shouldn't do this. It's pretty common for WebSocket clients to use more than one connection -- often many connections. Connections are cheap, it's easier to open a purpose-built connection that to design complex communications protocols. The server examples partly abstract connections into instantiable classes, but real projects can go further. A server could use the request path to deduce the intended use of a given connection. If your project is designed to use multiple concurrent connections, consider a way to assign correlation IDs to clients so that the server can correctly associate connections used for different purposes.

## Conclusion

When I realized `HttpListener` was being put to pasture, I was a bit concerned about having to switch to Kestrel. Whereas it's easy to use `HttpListener` with just a few lines of code, Kestrel was designed with the full, large, complex ASP.NET Core framework in mind. It requires considerably more setup "ceremony" to use. These sample projects are a good basis for comparison because the functional code (the socket and broadcast processing loops) is nearly cut-and-paste identical in both versions.

Size on disk isn't really something anybody worries about much now, but the individual exe/dll sizes are comparable. For Release builds, which is the same as publishing an installed-framework-dependent build, the Kestrel version produces 295K of files versus 190K for the `HttpListener` example (with broadcast support). Of course, even in .NET Core 3.0 with some tree-shaking support, published self-contained deployments are huge. The Kestrel version produces 75MB of files while the `HttpListener` version produces 59MB. (Core 2.x projects typically published about three times as much content, so progress is being made.)

More importantly, the `HttpListener` consumes about 5MB of memory at runtime, whereas the Kestrel version uses nearly 11MB. As clients connect, the small additional increase in memory usage is consistent over time. In server terms, an extra 6MB of memory is probably negligible, but it does illustrate how the .NET world is becoming increasingly reliant on more complex solutions, for better or for worse. 

Realistically, though, 5MB for such a simple application isn't exactly lightweight either. And for the purposes of creating more than just demo apps, that extra 6MB of overhead brings a lot of functionality -- exetensible config support, dependency injection, various baseline features of a full-blown web server stack, and so on -- all things you'll probably actually want when building a real-world solution.

So, at the end of the day, I have to say the Kestrel approach is probably a step forward.
