---
title: HttpClient Web Proxy Authentication
tags: c# .net
header:
  image: "/assets/headers/post_it_notes.png"
  teaser: "/assets/headers/post_it_notes.png"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2019/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2019/mm-dd/pic.jpg)
---

A short article documenting proxy auth configuration.

<!--more-->

From time to time, I post short articles that are more about reminders to myself when I figure out something that wasn't obvious or easy to get working. In this case, I was implementing techniques from some of my recent Blazor and Orleans OIDC articles in our real applications at work. Our devops team is still setting up our OIDC infrastructure so I decided to temporarily wire it up to the publicly-accessible IdentityServer demo authority. Like most enterprise users, we're behind a proxy server, and for whatever reason, it requires authentication. It took awhile to figure out how to get the `HttpClient`, the `IdentityModel` OIDC helper package, and ASP.NET Core OIDC authentication configured properly for this.

## The Non-Proxy Scenario

This is the non-proxy version of `ConfigureServices` code copied directly from a the `Startup.cs` for a recent article about JWT access token validation for securing Microsoft Orleans services:

```csharp
services.AddHttpClient();

services.AddSingleton<IDiscoveryCache>(sp =>
{
    var factory = sp.GetRequiredService<IHttpClientFactory>();
    return new DiscoveryCache(
        "https://demo.identityserver.io",
        () => factory.CreateClient());
});

services.AddAuthentication(options =>
{
    options.DefaultScheme = "Cookies";
    options.DefaultChallengeScheme = "oidc";
})
.AddCookie("Cookies")
.AddOpenIdConnect("oidc", options =>
{
    options.Authority = "https://demo.identityserver.io/";
    options.ClientId = "interactive.confidential";
    options.ClientSecret = "secret";
    options.ResponseType = "code";
    options.SaveTokens = true;
    options.GetClaimsFromUserInfoEndpoint = true;
    options.Scope.Add("openid");
    options.Scope.Add("api");
    options.Scope.Add("offline_access");
});
```

## Problem Diagnosis

Attempting to use the `DiscoveryCache` feature of `IdentityModel` with a proxy that requires authetication results in an error message which says: 

> Unable to retrieve document from: [PII is hidden]

PII stands for Personally Identifiable Information. In order to expose this information in the error message, you have to add another line of code to `ConfigureServices`:

```csharp
IdentityModelEventSource.ShowPII
```

After that you'll get an error message with useful information. I can't show the error from our internal proxy server, and it will vary according to the proxy server product in use, but if it says it requires credentials, you're in the right place.

Once you've confirmed the problem, you can remove the `ShowPII` setting.

## Proxy-Aware Handler

In `ConfigureServices` create a new handler with proxy server details:

```csharp
var proxy = new HttyClientHandler
{
    UseProxy = true,
    Proxy = null, // use system proxy
    DefaultProxyCredentials = CredentialsCache.DefaultNetworkCredentials
}
```

As the comment indicates, setting `Proxy` to null causes the application to use the default system proxy (currently still defined in IE settings on the Connections tab). As long as that is the case, `DefaultProxyCredentials` are applied. However, if `Proxy` points to an actual `Proxy` object, `DefaultProxyCredentials` are ignored, and the credentials (and other important properties like `Address`) should be configured within the `Proxy` object.

## Register Http Client Factory

Earlier this line was used to register a simple default factory for `HttpClient`:

```csharp
services.AddHttpClient();
```

Unfortunately, using a proxy means you must (at a minimum) define a named client, then a new configuration extension becomes available. The registration becomes:

```csharp
services.AddHttpClient("with_proxy_auth")
    .ConfigurePrimaryHttpMessageHandler(() => proxy);
```

## Using Http Client Factory

Anywhere your code injects `IHttpClientFactory`, it must reference that named client in order to use proxy authentication:

```csharp
var client = httpClientFactory.CreateClient("with_proxy_auth");
```

That requirement includes the factory lambda we set up for the `IDiscoveryCache` service:

```csharp
services.AddSingleton<IDiscoveryCache>(sp =>
{
    var factory = sp.GetRequiredService<IHttpClientFactory>();
    return new DiscoveryCache(
        "https://demo.identityserver.io",
        () => factory.CreateClient("with_proxy_auth"));
});
```

## Fixing OIDC Middleware

When you register for OIDC authentication by calling `AddOpenIdConnect`, parts of the system will sometimes communicate with the OIDC authority behind the scenes. In OIDC terminology this is called "back-channel" communication. You must also tell the OIDC middleware to use proxy authentication:

```csharp
services.AddAuthentication(...)
.AddOpenIdConnect("oidc", options =>
{
    // omitted: Authority, ClientId, etc.
    options.BackchannelHttpHandler = proxy;
});
```

This is the reason I stored the `HttpClientHandler` in a variable (in most examples online, it's in a lambda with the `AddHttpClient` call).

## Conclusion

It's pretty simple stuff, but the exact syntax wouldn't be easy to recall, and I couldn't find all of the information expressed clearly and correctly in one place (I wasted close to two hours tracking this down). Maybe it will help others, too!
