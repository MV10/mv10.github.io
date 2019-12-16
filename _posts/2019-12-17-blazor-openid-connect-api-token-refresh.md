---
title: Blazor OpenID Connect API Token Refresh
tags: c# .net asp.net.core security identityserver blazor
header:
  image: "/assets/2019/12-15/header1280.png"
  teaser: "/assets/2019/12-15/header1280.png"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2019/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2019/mm-dd/pic.jpg)
---

Correctly refreshing OIDC access tokens for Blazor server-side apps

<!--more-->

This is the third in a series about using OpenID Connect authentication with Blazor server-side apps. The earlier two articles were [Blazor Authentication with OpenID Connect]({{ site.baseurl }}{% post_url 2019-12-15-blazor-authentication-with-openid-connect %}) and [Blazor Login Expiration with OpenID Connect]({{ site.baseurl }}{% post_url 2019-12-16-blazor-login-expiration-with-openid-connect %}).

The other feature supported by OIDC are bearer tokens used to access non-interactive APIs. In those scenarios, the access token is provided as part of the API call (typically in an HTTP header, but later I'll explore using them to secure Microsoft Orleans-based microservices, which are not HTTP-based). The API validates the token before executing the requested operation. The token simply acts as proof that the caller -- the bearer -- has actually been authenticated with an authority that both the bearer and API recognize.

In the second article, I mentioned that I would handle token refresh in the validator, but that I wasn't sure if this was architecturally correct. That's the approach I still intend to take in this article, and it seems to work well. The same general problem applies that we had to resolve for login expiration: under MVC there is an HTTP pipeline which has periodic opportunities to perform the refresh, but no such concept applies in the world of Blazor over SignalR. The timer-based validator is as close as we can get to this.

Like the earlier article, it is based on ASP.NET Core 3.1, and I have updated the same repository, MV10/BlazorOIDC. However, because this article covers a different, optional aspect of the Blazor / OIDC relationship, these changes are in a separate branch called **_[access_token_refresh](https://github.com/MV10/BlazorOIDC/tree/access_token_refresh)_**.

## IdentityModel and Startup

The first step is to add a reference to the [`IdentityModel`](https://identitymodel.readthedocs.io/en/latest/) NuGet package. There is a bit of overlap between this package some of the Microsoft OIDC support, but this has built-in support for the actual refresh flow and it didn't make sense to reinvent that wheel.

In the `ConfigureServices` section of `Startup.cs`, register two more services before the call to `AddAuthentication`:

```csharp
services.AddHttpClient();

services.AddSingleton<IDiscoveryCache>(sp =>
{
    var factory = sp.GetRequiredService<IHttpClientFactory>();
    return new DiscoveryCache(
        "https://demo.identityserver.io/",
        () => factory.CreateClient());
});
```

The first one makes the new-style [`IHttpClientFactory`](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-3.1) available for injection.

The second registration sets up the IdentityModel `DiscoveryCache` service, which is used to identify endpoint URLs on a given OIDC authority. It isn't strictly necessary for the purposes of this sample code, but it's new and easy to do, and makes the sample "more correct".

Finally, we have to alter our OIDC configuration. In the `AddOpenIdConnect` call, add these:

```csharp
options.Scope.Add("openid");
options.Scope.Add("api");
options.Scope.Add("offline_access");
```

The "openid" and "offline_access" scopes are standard OIDC scopes. You must request "offline_access" in order to retrieve a refresh token. The "api" scope is a custom scope defined by the IdentityServer demo. It allows access to a test API URL also provided by the demo server.

![Api Scope](/assets/2019/12-17/api_scope.png)

## Caching Additional Data

It turns out we need to cache more auth data than I expected when I wrote the original article. I've added `IdToken` and a `RefreshAt` value which, not surprisingly, is the time at or after which the validator will attempt to refresh the access token.

```csharp
public class BlazorServerAuthData
{
    public string SubjectId;
    public DateTimeOffset Expiration;
    public string IdToken;
    public string AccessToken;
    public string RefreshToken;
    public DateTimeOffset RefreshAt;
}
```

The signature of the `Add` method in `BlazorServerAuthStateCache` has matching changes:

```csharp
public void Add(string subjectId, DateTimeOffset expiration, string idToken, string accessToken, string refreshToken, DateTimeOffset refreshAt)
{
    System.Diagnostics.Debug.WriteLine($"Caching sid: {subjectId}");

    var data = new BlazorServerAuthData
    {
        SubjectId = subjectId,
        Expiration = expiration,
        IdToken = idToken,
        AccessToken = accessToken,
        RefreshToken = refreshToken,
        RefreshAt = refreshAt
    };
    Cache.AddOrUpdate(subjectId, data, (k, v) => data);
}
```

The `OnGet` handler for `_HostAuthModel` is how the cache is populated, so we need to provide the id token and the refresh target. The id token is another simple one-liner:

```csharp
string idToken = await HttpContext.GetTokenAsync("id_token");
```

Deciding when to refresh the access token requires a bit more code. The "expires_at" claim is a UTC timestamp which reflects the expiration of the access token. Obviously you want to refresh it before that happens -- that's the whole point of this article. ASP.NET tries to refresh it at about halfway through the expiration period. We do this below, but since the cache-population code may also be running against an older persistent login, it's possible the token has already expired, so we perform a few more checks before setting the refresh target.

```csharp
string expiresAtToken = await HttpContext.GetTokenAsync("expires_at");
if (!DateTimeOffset.TryParse(expiresAtToken, out var refreshAt) 
    || refreshAt < DateTimeOffset.UtcNow 
    || string.IsNullOrWhiteSpace(refreshToken))
{
    refreshAt = DateTimeOffset.MaxValue;
}
else
{
    var seconds = refreshAt.Subtract(DateTimeOffset.UtcNow).TotalSeconds;
    refreshAt = DateTimeOffset.UtcNow.AddSeconds(seconds / 2);
}
```

## Calling the API

At this point, login will return enough information to execute the API call, as long as we test within the 75-second timeout window provided by the "short" client definition provided by the IdentityServer demo implementation.

Originally `Index.razor` only injected an instance of the standard `AuthenticationStateProvider`. Now the head of the file looks like this:

```csharp
@page "/"
@using IdentityModel.Client
@inject IHttpClientFactory httpClientFactory
@inject BlazorServerAuthStateCache AuthStateCache
@inject AuthenticationStateProvider AuthState
```

We've also added a button and some API-response output to the body of the file:

```html
<h1>Hello, @Username</h1>

<p>Welcome to your new app.</p>

<p>
    <a href="/Logout">Logout</a>
</p>

<p>
    <button @onclick="ApiCall">Call API</button>
</p>

<p>
    API Result:<br/>
    @ApiResult
</p>
```

During initialization, we also grab and store the subject ID. This allows us to retrieve the access token from cache when we make the API call later on.

```csharp
private string Username = "Anonymous User";
private string ApiResult = "(no result available)";

private string SubjectId;

protected override async Task OnInitializedAsync()
{
    var state = await AuthState.GetAuthenticationStateAsync();

    SubjectId =
        state.User.Claims
        .Where(c => c.Type.Equals("sid"))
        .Select(c => c.Value)
        .FirstOrDefault() ?? string.Empty;

    Username =
        state.User.Claims
        .Where(c => c.Type.Equals("name"))
        .Select(c => c.Value)
        .FirstOrDefault() ?? string.Empty;

    await base.OnInitializedAsync();
}
```

Finally, we add the code which actually calls the demo server API endpoint:

```csharp
private async Task ApiCall()
{
    if(string.IsNullOrWhiteSpace(SubjectId))
    {
        ApiResult = "(no SubjectId)";
        StateHasChanged();
        return;
    }

    var data = AuthStateCache.Get(SubjectId);
    if(data == null)
    {
        ApiResult = "(SubjectId not in cache)";
        StateHasChanged();
        return;
    }

    var client = httpClientFactory.CreateClient();
    client.SetBearerToken(data.AccessToken);
    var request = new HttpRequestMessage(HttpMethod.Get, "https://demo.identityserver.io/api/test");

    var response = await client.SendAsync(request);
    if (response.IsSuccessStatusCode)
    {
        ApiResult = await response.Content.ReadAsStringAsync();
    }
    else
    {
        ApiResult = $"{response.StatusCode} ({response.ReasonPhrase})";
    }
    StateHasChanged();
}
```

There isn't much to say about this. We use the injected `HttpClientFactory` to create a new `HttpClient`, we use the IdentityModel extension `SetBearerToken` to store the access token in the header, and we execute the API call. The results are stored and the component is refreshed. Running the app, logging in, and clicking the "Call API" button results in the following output:

![Api Success](/assets/2019/12-17/api_success.png)

However, we still aren't refreshing the access token. About 75 seconds after login, the access token expires. In reality, the accuracy of the expiration doesn't seem to be that precise. Smetimes I could continue using the token for as much as two minutes after the expiration time -- there are probably differences between the server time and my local machine's clock.

![Api Unauthorized](/assets/2019/12-17/api_unauthorized.png)

## Token Refresh

Refreshing the token is relatively easy. In the validator, we inject two additional services -- an `HttpClientFactory` and the IdentityModel discovery service:

```csharp
private readonly BlazorServerAuthStateCache Cache;
private readonly IHttpClientFactory httpClientFactory;
private readonly IDiscoveryCache OidcDiscoveryCache;

public BlazorServerAuthState(
    ILoggerFactory loggerFactory,
    BlazorServerAuthStateCache cache,
    IHttpClientFactory httpClientFactory,
    IDiscoveryCache discoveryCache)
    : base(loggerFactory)
{
    Cache = cache;
    this.httpClientFactory = httpClientFactory;
    OidcDiscoveryCache = discoveryCache;
}
```

During validation, after we get the cached auth data using the subject ID, and after we check for login expiration, we add one more line of code:

```csharp
if (data.RefreshAt < DateTimeOffset.UtcNow) await RefreshAccessToken(data);
```

The new refresh method looks like this:

```csharp
private async Task RefreshAccessToken(BlazorServerAuthData data)
{
    var client = httpClientFactory.CreateClient();
    var disco = await OidcDiscoveryCache.GetAsync();
    if (disco.IsError) return;

    var tokenResponse = await client.RequestRefreshTokenAsync(
        new RefreshTokenRequest
        {
            Address = disco.TokenEndpoint,
            ClientId = "interactive.confidential.short",
            ClientSecret = "secret",
            RefreshToken = data.RefreshToken
        });
    if (tokenResponse.IsError) return;

    data.AccessToken = tokenResponse.AccessToken;
    data.RefreshToken = tokenResponse.RefreshToken;
    data.RefreshAt = DateTimeOffset.UtcNow + TimeSpan.FromSeconds(tokenResponse.ExpiresIn / 2);
}
```

Once again we're using the injected `HttpClientFactory` to create a new `HttpClient`. We hit the IdentityModel discovery service which will (among other things) retrieve the token refresh endpoint. IdentityModel's latest release implements most features as extensions to `HttpClient`, and in this case we use the `RequestRefreshTokenAsync` extension.

Assuming the call is successful, we update the tokens and the next refresh target stored in the cache. That's it!

A test run with some additional debug logging looks like this (with some logging noise from `HttpClient` removed):

![Refresh Test](/assets/2019/12-17/refresh_test.png)

## Stale Cookies

A problem with this approach is that the MVC-style auth cookie is no longer synchronized with the cached access token, refresh token, and token expiration. While the user is in the application, it doesn't matter, but if the user exits the app with a persistent login and returns later, the `OnGet` handler in `_HostAuthModel` will cache the incorrect, outdated tokens.

I'm not going to address that problem in this project, although I did have to figure it out for my real project at work. The solution is to set up a controller to handle the refresh operation, because updating the cookie requires access to `HttpContext` (seeing a pattern here?). The controller call just needs to know the subject ID so that it can also update the cached value.

The additional code to update the cookie values looks like this:

```csharp
var tokens = new List<AuthenticationToken>
{
    new AuthenticationToken
    {
        Name = OpenIdConnectParameterNames.IdToken,
        Value = data.IdToken
    },
    new AuthenticationToken
    {
        Name = OpenIdConnectParameterNames.AccessToken,
        Value = tokenResponse.AccessToken
    },
    new AuthenticationToken
    {
        Name = OpenIdConnectParameterNames.RefreshToken,
        Value = tokenResponse.RefreshToken
    }
};

var expiresAt = DateTimeOffset.UtcNow + TimeSpan.FromSeconds(tokenResponse.ExpiresIn);
tokens.Add(new AuthenticationToken
{
    Name = "expires_at",
    Value = expiresAt.ToString("o")
});

var cookieAuth = await HttpContext.AuthenticateAsync("Cookies");
cookieAuth.Properties.StoreTokens(tokens);
await HttpContext.SignInAsync("Cookies", cookieAuth.Principal, cookieAuth.Properties);
```

## Conclusion

While planning this series, I thought token refresh would be the most difficult part to figure out. It was actually the easiest. These three articles provide the answer to full-blown OIDC support for Blazor server side applications.

I hope you found them useful.