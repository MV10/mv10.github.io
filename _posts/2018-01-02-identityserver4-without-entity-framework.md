---
title: IdentityServer4 Without Entity Framework
tags: c# .net asp.net.core security identityserver oauth2 openidconnect ado.net
header:
  image: "/assets/2018/01-02/headerpic.png"
  teaser: "/assets/2018/01-02/headerpic.png"
---

IdentityServer4 is arguably the most popular OpenID Connect server on the .NET platform, but like ASP.NET Core Identity, if you want persistence, you either have to accept considerable Entity Framework baggage or write it yourself. Fortunately the DIY route is easy: just three small tables and 13 SQL statements gets the job done. This post walks you through a basic IdentityServer setup with custom user-data and grant persistence using ADO.NET and SQL Server.

<!--more-->
## Persistence Without Entity Framework

I'm not a fan of Entity Framework or ORMs in general. I'm not here to argue the pros and cons, but I'm not exactly alone: a Google search will turn up _many_ blog posts, stackoverflow questions, and other instances of people trying to figure out how to use ASP.NET Core Identity and IdentityServer without EF. I don't think there are many programmers who actually like writing data access layers, but in most cases, my opinion is that old-school is still the best option for any serious project.

There are three categories of persisted data associated with IdentityServer4: configuration, grant tokens, and user data. If you go the Entity Framework route, IdentityServer addresses the first three, and you are directed to ASP.NET Core 2.0 Identity for persisting user data (also using Entity Framework). While this is certainly the easiest route, even if you're not opposed to Entity Framework, ASP.NET Core Identity is a flawed model since it doesn't differentiate between claims issued by different identity providers (among other arguably more subjective problems).

In this article, we'll cover persisting user data and grant tokens. I assume you're generally familiar with the concepts of OAuth2, OpenID Connect, ASP.NET Core 2.0, and what IdentityServer brings to the table. I'm going to move pretty quickly through setting up the projects so we can focus on the data problem. I won't cover persisting configuration data, but it is the easiest of the three categories to manage and it only needs to be stored in a database if your configuration changes often. Otherwise it's perfectly acceptable to hard-code the configuration and use the provided in-memory stores.

In later articles, I'll re-use these same projects to demonstrate how to quickly and easily switch your localhost development environments to SSL, and how to combine the IdentityServer and client web apps into a single Razor Pages project.

This article is based upon Visual Studio 2017 v15.5.2, ASP.NET Core 2 v2.0.3, IdentityServer4 v2.0.6, System.Data.SqlClient v4.4.2, and related current tooling available as of the end of December 2017. Fair warning, this content is not especially relevant to older versions of any of these products, they've all undergone significant architectural changes within the past six months.

The code is available [here](https://github.com/MV10/IdentityServer4.AdoPersistence) on GitHub.

## A Simple Client

The IdentityServer4 [documentation](https://identityserver4.readthedocs.io/en/release/index.html) is excellent -- probably some of the better documentation I've seen anywhere in recent years. They also provide quite a few Quickstart templates and a simple example of an in-memory user data store that makes a pretty good model to follow. Before we work on IdentityServer, we'll spend a few minutes creating a stand-alone client client website that can be used to test a complete OAuth2 flow.

First, create a new solution and project. Choose the "ASP.NET Core Web Application" project type, then the "Web Application" option, which is now a Razor Pages template. Do not enable authentication.

![Newclientproject](/assets/2018/01-02/newclientproject.png)

Right-click on the project name, choose Properties, then click the Build tab. Change the App URL to use port 5002.

![Clienturl](/assets/2018/01-02/clienturl.png)

Open NuGet and add the release version of the `IdentityServer4.AccessTokenValidation` package.

![Nugetclient](/assets/2018/01-02/nugetclient.png)

Now open `Startup.cs` and reference these assemblies.

```
using System.IdentityModel.Tokens.Jwt;
using Microsoft.ApplicationInsights.Extensibility;
```

You'll want to reference the Application Insights assembly so that you can disable the vast amount of junk that AI dumps into your logs. It's hard to imagine what Microsoft was thinking by forcing this upon us ([read more here](https://github.com/aspnet/Home/issues/2051)) but turning it off will make it a lot easier to find any debug content you may add for your own use. I don't use VS Code, but it is wrapped in a `try` block because (apparently) it may throw an exception in that environment.

Alter the startup configuration methods as follows. For OIDC purposes, our client application is named `mv10blog.client` and as recommended by the IdentityServer quickstart documentation, the identity application will run on port 5000.

```
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc();

            JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();

            services.AddAuthentication(options =>
            {
                options.DefaultScheme = "Cookies";
                options.DefaultChallengeScheme = "oidc";
            })
            .AddCookie("Cookies")

            .AddOpenIdConnect("oidc", options =>
            {
                options.SignInScheme = "Cookies";
                options.Authority = "http://localhost:5000";
                options.RequireHttpsMetadata = false;
                options.ClientId = "mv10blog.client";
                options.ClientSecret = "the_secret";
                options.ResponseType = "code id_token";
                options.SaveTokens = true;
                options.GetClaimsFromUserInfoEndpoint = true;
            });
        }

        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if(env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
                app.UseBrowserLink();
                try
                {
                    var configuration = app.ApplicationServices.GetService<TelemetryConfiguration>();
                    configuration.DisableTelemetry = true;
                }
                catch { }
            }
            else
            {
                app.UseExceptionHandler("/Error");
            }

            app.UseAuthentication();

            app.UseStaticFiles();
            app.UseMvc(routes =>
            {
                routes.MapRoute(
                    name: "default",
                    template: "{controller}/{action=Index}/{id?}");
            });
        }
```

Next, we need to actually secure something in the application to trigger the authorization flow. We'll secure the About page provided by the project template. In the `Pages` folder, expand `About.cshtml` and open the C# page model. Add the following `using` statements and the `[Authorize]` attribute to the class as shown in the code fragment below.

```
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Authentication;

namespace IdentityServer4.AdoPersistence.Pages
{
    [Authorize]
    public class AboutModel : PageModel
    {
    ...
    }
}
```

That is literally all that is required to secure a client site. Again, if you're new to Identity Server, you should read the documentation to better-understand what is happening. Even if you're familiar with the product, as I mentioned earlier, you should also read the docs if you're new to version 2.0 of ASP.NET Core or version 4 of Identity Server, both have undergone major architectural changes compared to their predecessors.

Finally, two simple additions are required to add a logoff feature. Add this method to the About page model.

```
public async Task OnGetLogoff()
{
    await HttpContext.SignOutAsync("Cookies");
    await HttpContext.SignOutAsync("oidc");
}
```

Now open the `About.cshtml` Razor template and replace everything with the following. Once you've logged in, this code will dump the details from your chosen identity-provider. The last line styles an anchor tag like a button which links back to the `OnGetLogoff` method added above.

```html
@page
@using Microsoft.AspNetCore.Authentication;
@model AboutModel
@{
    ViewData["Title"] = "About";
}
<h1>About Your Login</h1>

<dl>
    @foreach(var claim in User.Claims)
    {
        <dt>@claim.Type</dt>
        <dd>@claim.Value</dd>
    }
    <dt>access token</dt>
    <dd>@await ViewContext.HttpContext.GetTokenAsync("access_token")</dd>

    <dt>refresh token</dt>
    <dd>@await ViewContext.HttpContext.GetTokenAsync("refresh_token")</dd>
</dl>
<br />
<a class="btn btn-default" href="/About?handler=logoff">Logoff</a>
```

There are a couple of important security features this client is lacking that you'd want in a real-world project: SSL support and anti-forgery tokens. We'll cover those in a later article.

## The Identity Server Project

In the same solution, add a new ASP.NET Core 2.0 Web Application project, but this time choose the "Empty" template.

![Newidentproject](/assets/2018/01-02/newidentproject.png)

This project will be configured to run in the Kestrel host on port 5000. Right-click on the project name and choose Properties, then click on the Debug tab. Change the Profile drop-down to the entry that reflects the project name (not IIS Express), then change the App URL port to 5000. 

![Launchprofile](/assets/2018/01-02/launchprofile.png)

![Identurl](/assets/2018/01-02/identurl.png)

The IdentityServer documentation suggests starting these two projects independently. We'll stick with that recommendation. I find it easier to have them launch at the same time, but sometimes Visual Studio gets confused about whether it should be starting up IIS Express or Kestrel and we want to avoid such issues for this article.

Right-click on the solution and choose "Set startup projects". Choose the "Current selection" option. 

![Startupoptions](/assets/2018/01-02/startupoptions.png)

Open NuGet and add the release versions of the `IdentityServer4` and `System.Data.SqlClient` packages.

![Nugetidentityserver](/assets/2018/01-02/nugetidentityserver.png)

![Nugetsqlclient](/assets/2018/01-02/nugetsqlclient.png)

The IdentityServer team provides a very nice sample user interface in their [Quickstart UI repository](https://github.com/IdentityServer/IdentityServer4.Quickstart.UI/tree/release). Even better, the [documentation](https://identityserver4.readthedocs.io/en/release/quickstarts/3_interactive_login.html#adding-the-ui) explains how to run [this](https://github.com/IdentityServer/IdentityServer4.Quickstart.UI/blob/release/get.ps1) powershell command to add it to the project we just created. You may want to click that link, powershell is a powerful tool -- you should never blindly run a script without understanding it. This one just retrieves the Quickstart UI as a zip file, creates three folders in your project, and copies the contents into them.

Close Visual Studio, open powershell, change to the directory containing your IdentityServer project, then run the command:

```powershell
iex ((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/IdentityServer/IdentityServer4.Quickstart.UI/release/get.ps1'))
```

![Downloadquickstart](/assets/2018/01-02/downloadquickstart.png)

The Quickstart UI project adds quite a lot of functionality to your project: user login and logoff for both locally-created accounts and  third-party identity providers such as Google or Facebook, a consent page which tells users what information a third-party will provide to the client application, a grants page which lets users review that same information for a current login, and a diagnostics page which provides a more detailed look at what is happening inside IdentityServer. 

![Quickstartsln](/assets/2018/01-02/quickstartsln.png)

None of the Quickstart UI code should be considered production-ready. It is all branded with the IdentityServer name and logo, and the interface is clean but very basic. Also, no support is provided for registering new users (with one exception we'll discuss later), managing user accounts, or handling things like email verification or two-factor authentication. These are implementation details left to your client application.

It's also important to understand that IdentityServer is almost _completely_ uninterested in your user data. It is solely concerned with sign-in and sign-out and related things like cookies and OAuth2 flows. The Quickstart UI is where those concerns meld with your user data, and it's completely replaceable.

The last step in the basic project setup is to create a new folder for our persistence-related files.

![Persistencefolder](/assets/2018/01-02/persistencefolder.png)

## SQL Server

Before we get into the code, we'll prepare a database with three tables to store our user information, the claims received from the identity provider, and any grants associated with those claims. For this article, I'm using one of the local SQL Server instances that was installed alongside Visual Studio. Below, I have opened SQL Server Explorer and drilled down to the database level, then right-clicked "Add new datbase" to create a database named "Identity".

![Newdatabase](/assets/2018/01-02/newdatabase.png)

Next, right-click on the new database and choose "New query," then execute the following `CREATE TABLE` commands.

```sql
CREATE TABLE [dbo].[AppUser] (
    [id]                  INT                IDENTITY (1, 1) NOT NULL,
    [SubjectId]           NVARCHAR (MAX)     NOT NULL,
    [Username]            NVARCHAR (MAX)     NOT NULL,
    [PasswordSalt]        NVARCHAR (MAX)     NOT NULL,
    [PasswordHash]        NVARCHAR (MAX)     NOT NULL,
    [ProviderName]        NVARCHAR (MAX)     NOT NULL,
    [ProviderSubjectId]   NVARCHAR (MAX)     NOT NULL,
    PRIMARY KEY CLUSTERED ([id] ASC)
);

CREATE TABLE [dbo].[Claim] (
    [id]             INT            IDENTITY (1, 1) NOT NULL,
    [AppUser_id]     INT            NOT NULL,
    [Issuer]         NVARCHAR (MAX) DEFAULT ('') NOT NULL,
    [OriginalIssuer] NVARCHAR (MAX) DEFAULT ('') NOT NULL,
    [Subject]        NVARCHAR (MAX) DEFAULT ('') NOT NULL,
    [Type]           NVARCHAR (MAX) DEFAULT ('') NOT NULL,
    [Value]          NVARCHAR (MAX) DEFAULT ('') NOT NULL,
    [ValueType]      NVARCHAR (MAX) DEFAULT ('') NOT NULL,
    PRIMARY KEY CLUSTERED ([id] ASC)
);

CREATE TABLE [dbo].[Grant] (
    [id]           INT            IDENTITY (1, 1) NOT NULL,
    [Key]          NVARCHAR (200) NOT NULL,
    [ClientId]     NVARCHAR (200) NOT NULL,
    [CreationTime] DATETIME2 (7)  NOT NULL,
    [Data]         NVARCHAR (MAX) NOT NULL,
    [Expiration]   DATETIME2 (7)  NULL,
    [SubjectId]    NVARCHAR (200) NOT NULL,
    [Type]         NVARCHAR (50)  NOT NULL,
    PRIMARY KEY CLUSTERED ([id] ASC)
);
```

The user information we're storing is very basic. In a production application, you'd likely want to expand this to include things like account status, failed login attempts and temporary lockouts, email address and verification, and so on. Some of this information is even available from third-party identity providers (OpenID Connect defines "scopes" which are groups of claims, some of which are [standardized](https://developer.okta.com/blog/2017/07/25/oidc-primer-part-1#whats-a-scope)).

Each table uses an integer identity column as the clustered primary key. IdentityServer normally relies on the OAuth2 concept of a SubjectId to uniquely identify user identities, but there are SQL Server performance benefits to this PK arrangement, so this is just a convention I follow as a matter of routine. The `Claim.AppUser_id` column is a foreign key relating to the `AppUser.id` column, but the more I work with NoSQL document databases, the less I rely on database-enforced constraints. Obviously, you're free to set up the data any way you like.

ASP.NET Core has done away with conventions such as `web.config` but we need to store the connection string somewhere. Create a file named `appsettings.json` in the IdentityServer project folder. Next, click on the file and press `F4` to open the file properties. Change the "Copy to Output Directory" setting to "Copy always" to ensure the file can be found in the executable's location at runtime.

![Copyalways](/assets/2018/01-02/copyalways.png)

Add your connection string to the file (remember to escape any backslashes with a `\\` double-blackslash if your server name references a local SQL instance like I have):

```json
{
"ConnectionStrings": { "DefaultConnection": "Server=YOURSERVERNAME; Database=Identity; Trusted_Connection=True; MultipleActiveResultSets=true" } 
}
```

## IdentityServer Configuration

There are three types of configuration data used by IdentityServer: Identity Resources, API Resources, and Clients. Configuration could be persisted to a database, but in this article we'll just create lists and pass them to IdentityServer during startup. I won't go into a lot of detail about what is happening here since it is all covered in the IdentityServer documentation, and isn't particularly important to the data persistence issue. Create a class named `ConfigureIdentityServer.cs` in your project root and add the following code to it.

```
using System.Collections.Generic;
using IdentityServer4;
using IdentityServer4.Models;

namespace IdentityServer
{
    public class ConfigureIdentityServer
    {
        public static IEnumerable<IdentityResource> GetIdentityResources()
        {
            return new List<IdentityResource>
            {
                new IdentityResources.OpenId(),
                new IdentityResources.Email(),
                new IdentityResources.Profile(),
                new IdentityResources.Phone(),
                new IdentityResources.Address(),

                new IdentityResource(
                    name: "mv10blog.identity",
                    displayName: "MV10 Blog User Profile",
                    claimTypes: new[] { "mv10_accounttype" })
            };
        }

        public static IEnumerable<Client> GetClients()
        {
            return new List<Client>
            {
                new Client
                {
                    ClientId = "mv10blog.client",
                    ClientName = "McGuireV10.com",
                    ClientUri = "http://localhost:5002",
                    AllowedGrantTypes = GrantTypes.HybridAndClientCredentials,
                    ClientSecrets = {new Secret("the_secret".Sha256())},
                    AllowRememberConsent = true,
                    AllowOfflineAccess = true,
                    RedirectUris = { "http://localhost:5002/signin-oidc"}, // after login
                    PostLogoutRedirectUris = { "http://localhost:5002/signout-callback-oidc"}, // after logout
                    AllowedScopes = new List<string>
                    {
                        IdentityServerConstants.StandardScopes.OpenId,
                        IdentityServerConstants.StandardScopes.Profile,
                        IdentityServerConstants.StandardScopes.Email,
                        IdentityServerConstants.StandardScopes.Phone,
                        IdentityServerConstants.StandardScopes.Address,
                        "mv10blog.identity"
                    }
                }
            };
        }
    }
}
```

The `GetIdentityResources` method returns a single protected resource which is declared as having all of the standard OIDC scopes, plus a custom scope containing one custom claim. The `GetClients` method defines who can request identities from IdentityServer, what scopes they can request, the method of interaction ("grant types"), and other details.

Now open `Startup.cs` and add the following `using` statements.

```
using IdentityServer4;
using IdentityServer4.Stores;
using Microsoft.ApplicationInsights.Extensibility;
```

Replace the `ConfigureServices` and `Configure` methods with the code shown below. The IDE will flag three errors in the code. We'll fix them in the next section, they are references to classes that support data persistence.

```
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();

    services.AddIdentityServer()
        .AddDeveloperSigningCredential()
        .AddInMemoryClients(ConfigureIdentityServer.GetClients())
        .AddInMemoryIdentityResources(ConfigureIdentityServer.GetIdentityResources())
        .AddProfileService<UserProfileService>();

    services.AddSingleton<IUserStore, UserStore>();

    services.AddTransient<IPersistedGrantStore, PersistedGrantStore>();

    services.AddAuthentication()
        .AddGoogle("Google", options =>
        {
            options.SignInScheme = IdentityServerConstants.ExternalCookieAuthenticationScheme;
            options.ClientId = "434483408261-55tc8n0cs4ff1fe21ea8df2o443v2iuc.apps.googleusercontent.com";
            options.ClientSecret = "3gcoTrEDPPJ0ukn_aYYT6PWo";
        });
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if(env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();

        try
        {
            var configuration = app.ApplicationServices.GetService<TelemetryConfiguration>();
            configuration.DisableTelemetry = true;
        }
        catch { }
    }

    app.UseIdentityServer(); // includes a call to UseAuthentication

    app.UseStaticFiles();
    app.UseMvcWithDefaultRoute();
}
```

Once again, we'll refer you to the IdentityServer documentation to learn more about most of this. The `ClientId` and `ClientSecret` for Google sign-in are associated with `http://localhost:5000` under the IdentityServer team's account, as described in the [documentation](http://docs.identityserver.io/en/release/quickstarts/4_external_authentication.html#adding-google-support). Obviously they're only suited for development purposes, and the same caveats apply to this project as noted for the client -- in the real world you'll need SSL and anti-forgery tokens, as well as IdentityServer-specific concerns like real signing and key validation certificates (all of which is explained in their documentation).

## Data Persistence Classes

Finally, we're ready to begin tackling data persistence. Four new classes and one interface are all we need. In your Persistence folder, create classes named `AppUser`, `UserProfileService`, `PersistedGrantStore`, `UserStore`, and `IUserStore`. Since your project names may not match mine, for the sake of cut-and-paste convenience, we'll assign these to the global namespace.

The following code goes in the `AppUser` class, which matches the AppUser table we created in the database earlier, plus a `Claims` list associated with the user's current identity.

```
using Microsoft.AspNetCore.Cryptography.KeyDerivation;
using System;
using System.Collections.Generic;
using System.Security.Claims;
using System.Security.Cryptography;

// Global namespace

public class AppUser
{
    public int id = 0;
    public string SubjectId = string.Empty;
    public string Username = string.Empty;
    public string PasswordSalt = string.Empty;
    public string PasswordHash = string.Empty;
    public string ProviderName = string.Empty;
    public string ProviderSubjectId = string.Empty;

    public List<Claim> Claims = new List<Claim>();

    public static string PasswordSaltInBase64()
    {
        var salt = new byte[32]; // 256 bits
        using(var random = RandomNumberGenerator.Create())
        {
            random.GetBytes(salt);
        }
        return Convert.ToBase64String(salt);
    }

    public static string PasswordToHashBase64(string plaintextPassword, string storedPasswordSaltBase64)
    {
        var salt = Convert.FromBase64String(storedPasswordSaltBase64);
        var bytearray = KeyDerivation.Pbkdf2(plaintextPassword, salt, KeyDerivationPrf.HMACSHA512, 50000, 24);
        return Convert.ToBase64String(bytearray);
    }

    public static bool PasswordValidation(string storedPasswordHashBase64, string storedPasswordSaltBase64, string plaintextToValidate)
    {
        return storedPasswordHashBase64.Equals(PasswordToHashBase64(plaintextToValidate, storedPasswordSaltBase64));
    }
}
```

In a real application, you'd probably move some of the database code to a shared library and expand the information you keep about the user, as described at the beginning of the article. IdentityServer itself doesn't need anything but the information shown here (and in some cases, not even all of that) but by matching the SubjectId, the client application could retrieve the same data for other purposes.

The three static methods in our AppUser class relate to password security. Discussion about how this works is out of the scope of this article, but [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2) is a reasonably secure password hashing algorithm provided out-of-the-box by ASP.NET Core 2. We'll briefly touch on these methods again when we get to the Quickstart UI `AccountController` class, but this article relies on third-party login, whereas password salting and hashing is only a concern for local account registration.

Replace the `UserProfileService` template code with the following.

```
using IdentityServer4.Extensions;
using IdentityServer4.Models;
using IdentityServer4.Services;
using System.Linq;
using System.Threading.Tasks;

// Global namespace

public class UserProfileService : IProfileService
{
    protected readonly IUserStore userstore;

    public UserProfileService(IUserStore injectedUserStore)
    {
        userstore = injectedUserStore;
    }

    public virtual async Task GetProfileDataAsync(ProfileDataRequestContext context)
    {
        if(context.RequestedClaimTypes.Any())
        {
            var user = await _userstore.FindBySubjectId(context.Subject.GetSubjectId());
            if(user != null)
            {
                context.AddRequestedClaims(user.Claims);
            }
        }
        return;
    }

    public virtual async Task IsActiveAsync(IsActiveContext context)
    {
        var user = await _userstore.FindBySubjectId(context.Subject.GetSubjectId());
        context.IsActive = !(user is null); // TODO check indicators like account status
        return;
    }
}
```

The `IsActiveAsync` method is important. It is how IdentityServer verifies whether the user should be allowed to complete a login. You would therefore replace the code which sets `context.IsActive` with decisions based on your own application's requirements. For example, you may want to check for failed-login lockouts, or account suspension due to moderation or past-due billing. All of these could be represented as concrete properties and columns in `AppUser` and the database, or stored as claims with the local-account identity. As usual, IdentityServer doesn't care how you approach user-related problems.

Next, replace the `PersistedGrantStore` code with the following.

```
using IdentityServer4.Models;
using IdentityServer4.Stores;
using Microsoft.Extensions.Configuration;
using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SqlClient;
using System.Threading.Tasks;

// Global namespace

public class PersistedGrantStore : IPersistedGrantStore
{
    private string connectionString;
    public PersistedGrantStore(IConfiguration configuration)
    {
        connectionString = configuration.GetConnectionString("DefaultConnection");
    }

    public async Task<IEnumerable<PersistedGrant>> GetAllAsync(string subjectId)
    {
        var grants = new List<PersistedGrant>();
        using(var conn = new SqlConnection(connectionString))
        {
            await conn.OpenAsync();
            using(var cmd = new SqlCommand("SELECT * FROM [Grant] WHERE [SubjectId] = @sub;", conn))
            {
                cmd.Parameters.Add(new SqlParameter("@sub", subjectId));
                var reader = await cmd.ExecuteReaderAsync();
                if(reader.HasRows)
                {
                    var table = new DataTable();
                    table.Load(reader);
                    foreach(DataRow row in table.Rows)
                    {
                        grants.Add(DataToGrant(row));
                    }
                }
                reader.Close();
            }
        }
        return grants;
    }

    public async Task<PersistedGrant> GetAsync(string key)
    {
        PersistedGrant grant = null;
        using(var conn = new SqlConnection(connectionString))
        {
            await conn.OpenAsync();
            using(var cmd = new SqlCommand("SELECT * FROM [Grant] WHERE [Key] = @key", conn))
            {
                cmd.Parameters.Add(new SqlParameter("@key", key));
                var reader = await cmd.ExecuteReaderAsync();
                if(reader.HasRows)
                {
                    var table = new DataTable();
                    table.Load(reader);
                    grant = DataToGrant(table.Rows[0]);
                }
                reader.Close();
            }
        }
        return grant;
    }

    public async Task RemoveAllAsync(string subjectId, string clientId)
    {
        using(var conn = new SqlConnection(connectionString))
        {
            await conn.OpenAsync();
            using(var cmd = new SqlCommand("DELETE FROM [Grant] WHERE [SubjectId] = @sub AND [ClientId] = @client", conn))
            {
                cmd.Parameters.Add(new SqlParameter("@sub", subjectId));
                cmd.Parameters.Add(new SqlParameter("@client", clientId));
                await cmd.ExecuteNonQueryAsync();
            }
        }
    }

    public async Task RemoveAllAsync(string subjectId, string clientId, string type)
    {
        using(var conn = new SqlConnection(connectionString))
        {
            await conn.OpenAsync();
            using(var cmd = new SqlCommand("DELETE FROM [Grant] WHERE [SubjectId] = @sub AND [ClientId] = @client AND [Type] = @type", conn))
            {
                cmd.Parameters.Add(new SqlParameter("@sub", subjectId));
                cmd.Parameters.Add(new SqlParameter("@client", clientId));
                cmd.Parameters.Add(new SqlParameter("@type", type));
                await cmd.ExecuteNonQueryAsync();
            }
        }
    }

    public async Task RemoveAsync(string key)
    {
        using(var conn = new SqlConnection(connectionString))
        {
            await conn.OpenAsync();
            using(var cmd = new SqlCommand("DELETE FROM [Grant] WHERE [Key] = @key", conn))
            {
                cmd.Parameters.Add(new SqlParameter("@key", key));
                await cmd.ExecuteNonQueryAsync();
            }
        }
    }

    public async Task StoreAsync(PersistedGrant grant)
    {
        using(var conn = new SqlConnection(connectionString))
        {
            await conn.OpenAsync();
            string upsert =
                $"MERGE [Grant] WITH (ROWLOCK) AS [T] " +
                $"USING (SELECT '{grant.Key}' AS [Key]) AS [S] " +
                $"ON [T].[Key] = [S].[Key] " +
                $"WHEN MATCHED THEN UPDATE SET [ClientId]='{grant.ClientId}', [CreationTime]='{FormatDate(grant.CreationTime)}', [Data]='{grant.Data}', [Expiration]={NullOrDate(grant.Expiration)}, [SubjectId]='{grant.SubjectId}', [Type]='{grant.Type}' " +
                $"WHEN NOT MATCHED THEN INSERT ([Key], [ClientId], [CreationTime], [Data], [Expiration], [SubjectId], [Type]) " +
                $"VALUES ('{grant.Key}','{grant.ClientId}','{FormatDate(grant.CreationTime)}','{grant.Data}',{NullOrDate(grant.Expiration)},'{grant.SubjectId}','{grant.Type}'); ";
            using(var cmd = new SqlCommand(upsert, conn))
            {
                await cmd.ExecuteNonQueryAsync();
            }
        }
    }

    private string NullOrDate(DateTime? value)
    {
        return (value.HasValue) ? $"'{FormatDate(value.Value)}'" : "null";
    }

    private string FormatDate(DateTime value)
    {
        // UTC ISO 8601 format
        return ((DateTimeOffset)value).ToUniversalTime().ToString("o");
    }

    private PersistedGrant DataToGrant(DataRow row)
    {
        DateTime? expiration = (row["Expiration"] is DBNull) ? null : (DateTime?)row["Expiration"];
        return new PersistedGrant()
        {
            Key = (string)row["Key"],
            ClientId = (string)row["ClientId"],
            CreationTime = (DateTime)row["CreationTime"],
            Data = (string)row["Data"],
            Expiration = expiration,
            SubjectId = (string)row["SubjectId"],
            Type = (string)row["Type"]
        };
    }
}
```

Next, create the `IUserStore` interface so that we can inject configuration data into the concrete implementation.

```
using System.Collections.Generic;
using System.Security.Claims;
using System.Threading.Tasks;

// Global namespace

public interface IUserStore
{
    Task<bool> ValidateCredentials(string username, string password);
    Task<AppUser> FindBySubjectId(string subjectId);
    Task<AppUser> FindByUsername(string username);
    Task<AppUser> FindByExternalProvider(string provider, string subjectId);
    Task<AppUser> AutoProvisionUser(string provider, string subjectId, List<Claim> claims);
    Task<bool> SaveAppUser(AppUser user, string newPasswordToHash = null);
}
```

Finally, replace the `UserStore` class template with the code below.

```
using IdentityModel;
using Microsoft.Extensions.Configuration;
using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SqlClient;
using System.IdentityModel.Tokens.Jwt;
using System.Linq;
using System.Security.Claims;
using System.Threading.Tasks;

// Global namespace

public class UserStore : IUserStore
{
    private string connectionString;
    public UserStore(IConfiguration configuration)
    {
        connectionString = configuration.GetConnectionString("DefaultConnection");
    }

    public async Task<bool> ValidateCredentials(string username, string password)
    {
        string hash = null;
        string salt = null;
        using(var conn = new SqlConnection(connectionString))
        {
            await conn.OpenAsync();
            DataTable table = new DataTable();
            using(var cmd = new SqlCommand("SELECT [PasswordSalt], [PasswordHash] FROM [AppUser] WHERE [Username] = @username;", conn))
            {
                cmd.Parameters.Add(new SqlParameter("@username", username));
                var reader = await cmd.ExecuteReaderAsync();
                table.Load(reader);
                reader.Close();
            }
            if(table.Rows.Count > 0)
            {
                salt = (string)(table.Rows[0]["PasswordSalt"]);
                hash = (string)(table.Rows[0]["PasswordHash"]);
            }
        }
        return (String.IsNullOrEmpty(salt) || String.IsNullOrEmpty(hash)) ? false : AppUser.PasswordValidation(hash, salt, password);
    }

    public async Task<AppUser> FindBySubjectId(string subjectId)
    {
        AppUser user = null;
        using(SqlCommand cmd = new SqlCommand("SELECT * FROM [AppUser] WHERE [SubjectId] = @subjectid;"))
        {
            cmd.Parameters.Add(new SqlParameter("@subjectid", subjectId));
            user = await ExecuteFindCommand(cmd);
        }
        return user;
    }

    public async Task<AppUser> FindByUsername(string username)
    {
        AppUser user = null;
        using(SqlCommand cmd = new SqlCommand("SELECT * FROM [AppUser] WHERE [Username] = @username;"))
        {
            cmd.Parameters.Add(new SqlParameter("@username", username));
            user = await ExecuteFindCommand(cmd);
        }
        return user;
    }

    public async Task<AppUser> FindByExternalProvider(string provider, string subjectId)
    {
        AppUser user = null;
        using(SqlCommand cmd = new SqlCommand("SELECT * FROM [AppUser] WHERE [ProviderName] = @pname AND [ProviderSubjectId] = @psub;"))
        {
            cmd.Parameters.Add(new SqlParameter("@pname", provider));
            cmd.Parameters.Add(new SqlParameter("@psub", subjectId));
            user = await ExecuteFindCommand(cmd);
        }
        return user;
    }

    private async Task<AppUser> ExecuteFindCommand(SqlCommand cmd)
    {
        AppUser user = null;
        using(var conn = new SqlConnection(connectionString))
        {
            await conn.OpenAsync();
            cmd.Connection = conn;
            var reader = await cmd.ExecuteReaderAsync();
            if(reader.HasRows)
            {
                DataTable table = new DataTable();
                table.Load(reader);
                reader.Close();
                var userRow = table.Rows[0];
                user = new AppUser()
                {
                    id = (int)userRow["id"],
                    SubjectId = (string)userRow["SubjectId"],
                    Username = (string)userRow["Username"],
                    PasswordSalt = (string)userRow["PasswordSalt"],
                    PasswordHash = (string)userRow["PasswordHash"],
                    ProviderName = (string)userRow["ProviderName"],
                    ProviderSubjectId = (string)userRow["ProviderSubjectId"],
                };
                using(var claimcmd = new SqlCommand("SELECT * FROM [Claim] WHERE [AppUser_id] = @uid;", conn))
                {
                    claimcmd.Parameters.Add(new SqlParameter("@uid", user.id));
                    reader = await claimcmd.ExecuteReaderAsync();
                    if(reader.HasRows)
                    {
                        table = new DataTable();
                        table.Load(reader);
                        user.Claims = new List<Claim>(table.Rows.Count);
                        foreach(DataRow row in table.Rows)
                        {
                            user.Claims.Add(new Claim(
                                type: (string)row["Type"],
                                value: (string)row["Value"],
                                valueType: (string)row["ValueType"],
                                issuer: (string)row["Issuer"],
                                originalIssuer: (string)row["OriginalIssuer"]));
                        }
                    }
                    reader.Close();
                }
            }
            cmd.Connection = null;
        }
        return user;
    }

    public async Task<AppUser> AutoProvisionUser(string provider, string subjectId, List<Claim> claims)
    {
        // create a list of claims that we want to transfer into our store
        var filtered = new List<Claim>();

        foreach(var claim in claims)
        {
            // if the external system sends a display name - translate that to the standard OIDC name claim
            if(claim.Type == ClaimTypes.Name)
            {
                filtered.Add(new Claim(JwtClaimTypes.Name, claim.Value));
            }
            // if the JWT handler has an outbound mapping to an OIDC claim use that
            else if(JwtSecurityTokenHandler.DefaultOutboundClaimTypeMap.ContainsKey(claim.Type))
            {
                filtered.Add(new Claim(JwtSecurityTokenHandler.DefaultOutboundClaimTypeMap[claim.Type], claim.Value));
            }
            // copy the claim as-is
            else
            {
                filtered.Add(claim);
            }
        }

        // if no display name was provided, try to construct by first and/or last name
        if(!filtered.Any(x => x.Type == JwtClaimTypes.Name))
        {
            var first = filtered.FirstOrDefault(x => x.Type == JwtClaimTypes.GivenName)?.Value;
            var last = filtered.FirstOrDefault(x => x.Type == JwtClaimTypes.FamilyName)?.Value;
            if(first != null && last != null)
            {
                filtered.Add(new Claim(JwtClaimTypes.Name, first + " " + last));
            }
            else if(first != null)
            {
                filtered.Add(new Claim(JwtClaimTypes.Name, first));
            }
            else if(last != null)
            {
                filtered.Add(new Claim(JwtClaimTypes.Name, last));
            }
        }

        // create a new unique subject id
        var sub = CryptoRandom.CreateUniqueId();

        // check if a display name is available, otherwise fallback to subject id
        var name = filtered.FirstOrDefault(c => c.Type == JwtClaimTypes.Name)?.Value ?? sub;

        // create new user
        var user = new AppUser
        {
            SubjectId = sub,
            Username = name,
            ProviderName = provider,
            ProviderSubjectId = subjectId,
            Claims = filtered
        };

        // store it and give it back
        await SaveAppUser(user);
        return user;
    }

    public async Task<bool> SaveAppUser(AppUser user, string newPasswordToHash = null)
    {
        bool success = true;
        if(!String.IsNullOrEmpty(newPasswordToHash))
        {
            user.PasswordSalt = AppUser.PasswordSaltInBase64();
            user.PasswordHash = AppUser.PasswordToHashBase64(newPasswordToHash, user.PasswordSalt);
        }
        try
        {
            using(var conn = new SqlConnection(connectionString))
            {
                await conn.OpenAsync();
                string upsert =
                    $"MERGE [AppUser] WITH (ROWLOCK) AS [T] " +
                    $"USING (SELECT {user.id} AS [id]) AS [S] " +
                    $"ON [T].[id] = [S].[id] " +
                    $"WHEN MATCHED THEN UPDATE SET [SubjectId]='{user.SubjectId}', [Username]='{user.Username}', [PasswordHash]='{user.PasswordHash}', [PasswordSalt]='{user.PasswordSalt}', [ProviderName]='{user.ProviderName}', [ProviderSubjectId]='{user.ProviderSubjectId}' " +
                    $"WHEN NOT MATCHED THEN INSERT ([SubjectId],[Username],[PasswordHash],[PasswordSalt],[ProviderName],[ProviderSubjectId]) " +
                    $"VALUES ('{user.SubjectId}','{user.Username}','{user.PasswordHash}','{user.PasswordSalt}','{user.ProviderName}','{user.ProviderSubjectId}'); " +
                    $"SELECT SCOPE_IDENTITY();";
                object result = null;
                using(var cmd = new SqlCommand(upsert, conn))
                {
                    result = await cmd.ExecuteScalarAsync();
                }
                int newId = (result is null || result is DBNull) ? 0 : Convert.ToInt32(result); // SCOPE_IDENTITY returns a SQL numeric(38,0) type
                if(newId > 0) user.id = newId;
                if(user.id > 0 && user.Claims.Count > 0)
                {
                    foreach(Claim c in user.Claims)
                    {
                        string insertIfNew =
                            $"MERGE [Claim] AS [T] " +
                            $"USING (SELECT {user.id} AS [uid], '{c.Subject}' AS [sub], '{c.Type}' AS [type], '{c.Value}' as [val]) AS [S] " +
                            $"ON [T].[AppUser_id]=[S].[uid] AND [T].[Subject]=[S].[sub] AND [T].[Type]=[S].[type] AND [T].[Value]=[S].[val] " +
                            $"WHEN NOT MATCHED THEN INSERT ([AppUser_id],[Issuer],[OriginalIssuer],[Subject],[Type],[Value],[ValueType]) " +
                            $"VALUES ('{user.id}','{c.Issuer ?? string.Empty}','{c.OriginalIssuer ?? string.Empty}','{user.SubjectId}','{c.Type}','{c.Value}','{c.ValueType ?? string.Empty}');";
                        using(var cmd = new SqlCommand(insertIfNew, conn))
                        {
                            await cmd.ExecuteNonQueryAsync();
                        }
                    }
                }
            }
        }
        catch
        {
            success = false;
        }
        return success;
    }
}
```

The `AutoProvisionUser` method is of particular interest. We haven't modified it much from the examples provided by IdentityServer, but in a real application you'd want to do additional work here. For example, it is common to link identities by looking for common elements such as an email address, so that if a user decides to switch from a Google login to a Facebook login, your site will recognize it's still the same user.

Even though these classes are the important parts of data persistence, there isn't really much to say about the SQL, which is very simple. Obviously in a production application, you'd probably want to move a lot of this to stored procedures, and you may even abstract some of this into more utility classes, but this is literally all it takes to persist IdentityServer user and grant data.

## AccountController and the UserStore Class

IdentityServer itself communicates with the `UserProfileService` and the `PersistedGrantStore` classes using DI and the references added during Setup. However, the Quickstart UI's `AccountController` needs to be modified to communicate with our new UserStore.

First, a little housekeeping: the Quickstart UI has some demo code we won't need. Open the Quickstart folder and delete the `TestUsers.cs` file.

![Deletetestusers](/assets/2018/01-02/deletetestusers.png)

Now open the `AccountController` and take a look at the `using` statements. Near the top you'll see `using IdentityServer4.Test`. Delete that entry. The `Test` assembly contains IdentityServer's [sample user handlers](https://github.com/IdentityServer/IdentityServer4/tree/dev/src/IdentityServer4/Test) which are in-memory versions of all the database functionality we just created.

![Usingtest](/assets/2018/01-02/usingtest.png)

Removing that `using` statement produces three errors. The controller was using `TestUserStore` to manage user data. 

![Testuserstoreerrors](/assets/2018/01-02/testuserstoreerrors.png)

We'll inject our own `IUserStore` service instead. The change for the private field is a direct replacement. The reference in the constructor argument list has a `null` default that we won't need. The third reference in the body of the constructor checked for that `null` default to trigger a fallback to the `TestUsers.cs` class we deleted earlier (it created a pair of hard-coded `TestUser` objects to act as data records for demo purposes). Just change it to a direct reference to the `IUserStore` service that will be injected into the constructor. You can remove the comment, too. The completed fixes should look like the fragment below.

```
private readonly IUserStore _users;
private readonly IIdentityServerInteractionService _interaction;
private readonly IEventService _events;
private readonly AccountService _account;

public AccountController(
    IIdentityServerInteractionService interaction,
    IClientStore clientStore,
    IHttpContextAccessor httpContextAccessor,
    IAuthenticationSchemeProvider schemeProvider,
    IEventService events,
    IUserStore users)
{
    _users = users;
    _interaction = interaction;
    _events = events;
    _account = new AccountService(interaction, httpContextAccessor, schemeProvider, clientStore);
}
```

As soon as those changes are applied, five more errors appear. This is because our `UserStore` methods are async, whereas the IdentityServer example was not. (Keep in mind the screenshot line numbers may not exactly match your copy if you have more or fewer lines earlier in the code, or if the IdentityServer team has modified their Quickstart templates since this was written.)

![Asyncerror1](/assets/2018/01-02/asyncerror1.png)

The first one is easily corrected by adding the `await` keyword to both `UserStore` calls.

![Asyncfix1](/assets/2018/01-02/asyncfix1.png)

The source of the second pair of errors is a little less obvious.

![Asyncerror2](/assets/2018/01-02/asyncerror2.png)

You must add `await` about 30 lines earlier in the code where the `user` reference is obtained. You should also add `await` a few lines below that on the call to `AutoProvisionUsers`.

![Asyncfix2](/assets/2018/01-02/asyncfix2.png)

Those are the only changes relating to data persistence, but while we're here, I'll show you one more trick. For some reason, the IdentityServer team decided the version 4 rewrite would only support session-duration third-party logins. That means your users would have to re-authenticate every time they returned to the website, which is almost always horrible UX. Fortunately it's easy to fix and you're already looking at the relevant section of code.

Find this:

![Cookiepersistence1](/assets/2018/01-02/cookiepersistence1.png)

Replace that entire block with the following code. The `AccountOptions` class sets persistent login duration to 30 days, but that is part of the Quickstart UI, so you can change that to anything you like. You could even present a drop-down list in your UI and let the user decide.

```
// make external logins persistent rather than session-duration
AuthenticationProperties props = new AuthenticationProperties();
props.IsPersistent = true;
props.ExpiresUtc = DateTimeOffset.UtcNow.Add(AccountOptions.RememberMeLoginDuration);

// if the external provider issued an id_token, we'll keep it for signout
var id_token = result.Properties.GetTokenValue("id_token");
if(id_token != null)
{
    props.StoreTokens(new[] { new AuthenticationToken { Name = "id_token", Value = id_token } });
}
```

## Test Drive

Now it's time to spin up the servers and try it out!

Click on the IdentityServer project, then press `CTRL-F5`to run the project without debugging. The Kestrel console window should open after a few seconds.

![Kestrel](/assets/2018/01-02/kestrel.png)

Next, select the client web app project and start that. It should launch a browser showing the ASP.NET Core template site.

![Homepage](/assets/2018/01-02/homepage.png)

Recall that we added configured the About page as a resource that requires authorization. Click the "About" link in the header. After a few seconds you will be redirected to IdentityServer for login.

![Loginpage](/assets/2018/01-02/loginpage.png)

Since we don't have any local users in the database, click the Google button for third-party login. (Dominick's Demo is the application name the IdentityServer4 team registered for the Google API client ID and secret we're using to authenticate as the http://localhost:5000 URL.)

![Googleoauth2](/assets/2018/01-02/googleoauth2.png)

After signing in, IdentityServer displays the consent page so the user can review the information being exposed to the client app. Our client configuration has enabled the remember-consent option, so a subsequent sign off followed by a sign in would skip this step.

![Consent](/assets/2018/01-02/consent.png)

And finally we return to the client website, having been granted access to the page requiring authorization.

![Protectedpage](/assets/2018/01-02/protectedpage.png)

## Show Me the Data

![Trekdata](/assets/2018/01-02/trekdata.png)

So far, you could have completed the exact same login process using the in-memory test-user classes built into the default Quickstart UI. Let's take a quick look at the reason we're all here: the underlying data persisted during this process.

The AppUser table has the locally-generated Subject Id, the Username as my real name that was created by concatenating the FirstName and FamilyName claims, "Google" as the identity provider's name, and finally the Provider Subject Id.

![Dataappuser](/assets/2018/01-02/dataappuser.png)

The Claim table reflects the various claims received from Google and also those generated locally. Note that the Subject field on the Claim objects are never populated. Our data persistence layer applies the locally-assigned SubjectId. This way, claims from multiple Identity Providers will relate to the same user (the Principal, in security-speak).

![Dataclaim](/assets/2018/01-02/dataclaim.png)

Finally, the Grant table records my consent to share this information with the client site. For flows requesting authentication for API resources, you will also see access tokens and refresh tokens stored here.

![Datagrant](/assets/2018/01-02/datagrant.png)

## Conclusion

I'm seeing a troubling trend in software development. Free and open access to tens of thousands of libraries is an amazing thing, but too many developers are simply plugging in frameworks and tools without necessarily understanding how or why it works, or exactly how much baggage is hidden in the latest new magic black-box. For awhile I thought this was limited to the frenetic anarchy of the scripting-framework world, but as I'm spending more and more time working with Azure and .NET Core, and as Microsoft struggles to migrate to OSS using Agile-like release-early/release-often methodologies, the formerly-stable .NET world is starting to experience much of the same aggravating instabilities and confusion. The sudden uptick in Entity Framework usage is just one small symptom of this trend.

Creating a dependency on an ORM is a fairly major architectural decision, and in my experience the benefits (quick-starts and abstracting a bit of unpleasant coding) are rarely worth the long-term risks and costs for any serious project. Using IdentityServer4 without Entity Framework is relatively painless.

Like IdentityServer4's own Quickstart UI, the persistence code provided here isn't meant to be production-ready. Instead, it shows you how to get there. Hope it helps.
