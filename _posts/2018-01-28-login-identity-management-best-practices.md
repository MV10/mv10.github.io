---
title: Login and Identity Management Best-Practices
tags: c# .net asp.net.core security
header:
  image: "/assets/2018/01-28/header1280.png"
  teaser: "/assets/2018/01-28/header1280.png"

# image paths:
#   publish:      (/assets/2018/mm-dd/pic.jpg)
#   edit _drafts: (assets/2018/mm-dd/pic.jpg)
#   edit _post:   (../assets/2018/mm-dd/pic.jpg)
---

There are many interesting and tricky concerns surrounding user login, logout, and identity management. We'll take a look at common and uncommon scenarios and discuss techniques to handle them.

<!--more-->

We're using Identity Server 4, but many of these concerns apply to any web-based user identity subsystem. A few only become complicated when the identity management system and/or identity provider is separate from the client application requesting user authorization. These concerns are valid whether the identity system is Identity Server, or a service like Okta or oauth.net, or even a roll-your-own OAuth2 client using third-party providers like Google. This article is written with the assumption that a stand-alone OpenId Connect (OIDC) compliant system such as Identity Server 4 is in use.

Another assumption is that we want to keep as many of these concerns isolated in the identity management system as possible. With OIDC in particular, this is not always possible since OIDC dictates that the client application initiates certain processes, but it is certainly possible to significantly minimize how much the client app needs to know about authentication.

Here are the topics we'll consider:

* Account data versus identity
* Prompting anonymyous users
* Handling of new accounts
* Email-based account verification
* Password-reset requests
* Persistent login
* Users who bookmark the login page
* Allowing users to change their IdP
* Reminding users which IdP they chose
* Failed-login lockouts
* Translating the default Windows `Claim.Issuer`
* `POST` vs `GET` endpoints

By the time we're finished, you'll understand why the ASP.NET Core Identity template that comes with Visual Studio, or the Identity Server Quickstart UI projects are only partial solutions to identity and account management, and why both Microsoft and the Identity Server team warn that those samples are not production-ready implementations. The majority of the issues we'll discuss simply aren't addressed by those demo-grade code samples.

## The Identity is Not the User

You should consider your IdPs to be sources of only the most basic information about your users. Apart from very specialized use-cases, normally you should only expect an external IdP to send a few claims about the user's name, and hopefully the user's email address.

Thus, the first lesson is that you should plan to treat IdP claims as separate from the user account information stored by your site. You may want to let the user alter their username used for your site, or to choose some kind of avatar for their account. If you're building a paid service, you'll want to associate subscription data with the user account, and so on.

Generally speaking, you should expect to store a lot more data about your users than you're going to receive from the IdP.

## Prompting Anonymous Users

When a user first arrives at your client site, they will be anonymous. If the user is new, they'll want to create an account on your site, either through a third-party IdP or by registering a local account. If the user already has an account, they'll want to login.

Are these two separate use-cases? Yes and no. The Identity Server Quickstart UI doesn't address new user registration at all, and the ASP.NET Core Identity template sample-code treats them as separate issues.

Login with OIDC requires the client application to start the flow, and since you can't know in advance if a new user wants a third-party account or will register a local account, the easiest and best UX is to combine the two into a single "Register or Login" link. On the login page at the identity service, explain the options for new users and returning users. It makes for a very easy and natural experience with little development effort.

## Creating New Third-Party Accounts

The need to store your own user data implies a need to recognize when a third-party login represents a new user. In theory, if a user signs in via Google (for example), you should check your database for a user record whose IdP is Google and the IdP Subject Id on that same user record matches the Subject Id claim received from the IdP.

Logically, if there isn't a user account that matches the IdP Subject Id from that IdP, then it's a new user. Or is it? What if the user originally joined your site with a Microsoft login, but later forgot that fact and tried to login via a Google account, instead?

In our system, we require all users to have a valid email address on file. When the scenario above takes place, we don't immediately assume the login is a new account. We look for an email claim, and we search for that in the database. If we find it, we assume this user simply chose the wrong login provider. We send a reminder email, and we route the user to a message asking them to check their email.

It should be noted that the various organizations who define the standard names for claims usually caution against treating email addresses as unique identifiers, but we're of the opinion that if an email address is safe enough for a password-reset, then it's safe enough to identify the user, too. (Obviously you probably wan't to exercise quite a bit more caution if you're writing a banking application, for example.)

A typical site will also require users to have unique usernames and email addresses. While IdPs _usually_ supply claims containg various pieces of name data (often, a display name, and first and last names) and the user's email address, it's entirely possible that none of that information will be provided, or that whatever username you derive from the name claims is not unique. Those are edge-cases that your system will have to address.

## Creating New Local Accounts

By contrast, registering a new local account is relatively easy. You can verify the username and email addresses are unique before you store the data. Most sites lock down the user until the email address has been verified, or limit what the user can do until their address is verified.

Your greatest risk with local accounts is secure handling of user passwords. As long as you're [using SSL]({{ site.baseurl }}{% post_url 2018-01-04-localhost-ssl-identityserver-certificates %}), you can safely transmit the password on the wire between the client and among your servers, but you should _never_ persist the plaintext password. Instead, you should generate a random "salt" value and hash the password with that salt (using cryptography-grade randomization and hashing). Store those in the database and use them to validate the user's password during login. You can read more about this in the "Data Persistence Classes" section of my earlier article, [IdentityServer4 Without Entity Framework]({{ site.baseurl }}{% post_url 2018-01-02-identityserver4-without-entity-framework %}).

## Email or Account Verification

Creating new accounts potentially involves checking that a newly-entered email address is valid. An external IdP may not provide an email claim, and a locally-registered account will use whatever the new user has submitted with their other information. We've all been through this before -- the site parks you on a message telling you to check your Inbox, and you receive an email with a validation link of some kind.

We mentioned earlier that we'd prefer to isolate identity-related concerns in the identity management system to the greatest extent possible, so this implies that system should handle email verification with minimal impact on the client application. A complicating factor is that OIDC requires the client application to initiate login requests.

This is potentially a problem for email verification because you're now sending the user out of the login-flow. When they click that verification link in the email, they're starting a brand new browsing session -- a session only the identity service knows about. The user is no longer in the middle of an OIDC login flow initated by the client site.

We resolve this by creating a simple table used to authorize accounts to perform various activites -- email verification, password reset, and IdP migration are examples (we'll talk about the last two scenarios in a bit). The table contains the user id (you could use the locally-assigned Subject Id, it serves the same purpose, but we prefer auto-incremented identity-column integers for database-level foreign keys), a token generated with crypto-grade randomization, a flag to indicate the authorization type, an optional expiration timestamp, and crucially, the `ClientId` that Identity Server uses in the `Client` configuration data.

The token is a parameter in the verification link in the email sent to the user. When the user arrives, we validate the token against the database and flag the user's email account as verified. 

The final trick is to get the user back into the OIDC login flow. Since we're using Identity Server 4, we add a custom key/value-pair to the `Properties` list for each client app which defines a URI fragment pointing to a controller action or handler in the client app that triggers the OIDC login process. Combined with the `ClientUri`, the fragment produces a callback to the client app. This is explained in detail as part of [this]({{ site.baseurl }}{% post_url 2018-01-26-signoutasync-and-identity-server-cookies %}) earlier article.

## Password Reset

A secure plan for handling passwords should never include reversible encryption, so password recovery should never be an option. Instead, users should always be directed to reset their passwords. There isn't much to say about this process -- authorize the account to perform a reset, send an email with a link, and update the password on their return-trip.

We use the same authorization token process described in the previous section for email verification, and like that process, the final step is to re-start the OIDC login flow with a callback to the client application.

A common alternative is to send a cut-and-paste code in the email, allowing the user to enter that code to proceed, but that actually requires more work both from the development standpoint and the user-effort standpoint.

## Persistent Logins

Most modern-day websites support persistent login, and you should probably consider supporting it for your site. Since we wrote about this extensively in an [earlier article]({{ site.baseurl }}{% post_url 2018-01-12-persistent-login-with-identityserver %}), I won't rehash those details, but it should definitely be on your to-do list.

## Bookmarks to the Login Page

There are frequent StackOverflow and Github questions around dealing with this problem. By now, you're familiar with the requirement that OIDC login flows start from the client application. So what happens if the user browses directly to your identity service's login page?

If your identity service only supports a single client site, the answer is simple: redirect to the client's base URI. On the other hand, if your identity service manages identity for multiple client sites (and this is really the primary use-case for a stand-alone service: single sign-on, or SSO), there isn't really a _great_ answer for this.

When a user initially joins the site, we store the `ClientId` that started the OIDC login flow as part of the user's account data. Should the user find themselves at the identity service without a valid OIDC context, we fall back to that original `ClientId` as a default. You could also present the user with a list of client apps after login, or allow the user to choose their preferred default site.

## Changing Identity Providers

Users may change their mind about how they log into your sites, and fortunately this is easy to allow, which makes it a little strange that it's such a rare feature. I call this IdP migration. In the [article](post_url 2018-01-26-signoutasync-and-identity-server-cookie) mentioned in the email verification topic earlier, this is actually the reason for presenting the account verification and OIDC-restart concepts. If you want more information about how we achieve this, that's the place to start.

Incidentally, just a few hours after posting that article, I did find the first example of IdP migration I've ever seen "in the wild" on GitLab. Unfortunately, if you want to actually _use_ GitLab for source control, they force you to create a local username/password, so I'm not entirely sure why they support third-party logins at all, but at least someone recognizes that IdP migration is desirable.

## Identity Provider Reminder

This is another nice-to-have edge-case feature that is easy to implement. Our system currently supports six IdPs, and while developing and testing the IdP migration feature, I caught myself occasionally referring to the database to remind myself which IdP my test account was using.

I realized I've occasionally had this problem in the real world. I'll hear of some useful-sounding site, only to find that I'd registered there before, at some time in the distant past. Usually this is a username/password local login that either rejects my username or email address as already-in-use, but I've also had it happen at least once with a third-party login. (Yes, I'm getting old, sometimes I forget things.)

There are two opportunities to improve the UX around this issue.

In the process of allowing IdP migration, we look for a new-account IdP login and we compare the user's new IdP-email address to the database. If the email address is already known _and_ the account is authorized for IdP migration, we process the migration. However, if the account is _not_ authorized for IdP migration, we can interrupt the login and send the user a helpful email remider along the lines of, "You tried to log into our site with Google, but your email address is already registered with your Microsoft account. Please try again. You can change login providers later from the Manage Account page."

Similarly, a user may recognize up front that they don't remember their IdP, so it's a simple process to add a "remind me" button or link to your login page.

You could also just transparently migrate the user (that's what GitLab does), but I feel as if user account changes should always be initiated by the user.

## Failed Login Lockout

This is a pretty typical feature, so I won't say much about it except to ensure it's on your to-do list. Obviously, it only applies to local username/password accounts.

One aspect to consider is whether expired tokens for email verification, IdP migration, and password reset authorizations should count against the failed login attempts.

We also asked ourselves what to do once the account was locked. The final decision was to send another type of email, this time with two links (again based on the same authorization tokens used for password reset and so on). The first link allows the user to unlock the account to keep trying. The second link allows the user to initiate a password reset.

In theory, this means if the user's email is compromised but their account in our system is not, this will also compromise the account. However, the standard reset password feature carries the same risk. This is just a shortcut to the same functionality already available from the login page.

## Respect My Authority

All the properties of a `Claim` object are read-only. When your code creates a new `Claim`, if you don't provide an `Issuer` name in the constructor Windows will default to the user-unfriendly string `LOCAL AUTHORITY`. If you follow the Identity Server 4 Quickstart.UI or any of the related articles I've written, you'll have a few of these for local accounts, but you'll also have at least one and probably two (the local Subject Id and the username) even for third-party logins.

For all end-user communications, we translate this to "local username/password", which means email and message templates work equally well for third-party providers and local accounts.

## Use `HTTP POST` Whenever Possible

This is Web App Programming 101, but since we're talking about security, it bears repeating. Any endpoint you create should require an `HTTP POST` over `HTTP GET` because only a `POST` can be secured against various types of cross-site and impersonation attacks.

With the maze of redirection involved with OIDC, this sometimes requires careful planning. In general, a controller action or a page model handler that uses `return` to issue redirection is using `GET`, so the target of that redirection shouldn't ever be doing anything account-sensitive.

If you find yourself backed into a corner on this, you can always treat the endpoint as an API and use `GetAsync` to `POST` the data, then redirect to some safe target.

## Identity-Agnostic Clients

Notice that the preceding topics concentrate identity management in the identity service. Client applications only need to provide the following five capabilities:

* "Register or Login" link to start an OIDC signin flow
* "Logout" link to start an OIDC signout flow
* Redirect endpoint to start an OIDC signin flow
* UI to request email, password, and IdP changes
* Query user data using the authorized ASP.NET `User` Subject Id

Most of these can be implemented with just a few lines of code, then secure the rest of your site with standard `[Authorize]` attributes.

## Conclusion

Handling all of the edge cases and quirks of identity management requires real time and effort, and no small amount of careful planning, but it makes for a great user experience. Since this is usually the second thing a new user sees after your site's landing page, getting this right is a worthwhile investment. Your returning users will also appreciate the effort should they find they've forgotten the details of their account.
