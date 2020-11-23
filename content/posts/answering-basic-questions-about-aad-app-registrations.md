---
title: "Answering some basic questions about registering applications on Azure Active Directory"
date: 2020-10-14T10:00:50+02:00
tags: ["azure","aad", "msal", "adal"]
draft: false
---

These past years I worked with a few companies that uses Azure Active Directory to perform authentication and authorization on their line-of-business (LOB) applications and from time to time I get questions about how to register apps on AAD.   
Those questions are always pretty similar and most of the time the problem lies in some misconception, so I thought it might be a good idea to try to answer some of them and at the same time write them down in this post.           
Most of the questions I'm going to answer are about some pretty basic stuff so if you're an _"AAD veteran"_ this post is probably not for you.   

Maybe in the near future I'll try to update this post with more questions or maybe I write another post with another batch of qüestions.   
Without further ado, I'm going to start with a first batch of 10 questions. 

# Q1: What's the difference between "App Registrations" and "Enterprise Applications"?

If you have developed a new application and you want to integrate it with AAD you need to register it in the **App Registrations** section.       
Registering an app establishes a trust relationship between the application and the identity provider, it doesn't matter if it's a client application like a web or mobile app or it's an API that backs a client app.  

Some people might get confused between the **Enterprise Applications** section and the **App Registrations** section and that's because when you create an app in the **App Registrations** section another one with the same name is also created as an **Enterprise Application** in your AAD.

In **App Registrations** you are defining how your application is going to behave across all tenants, meanwhile in the **Enterprise Applications** section what it gets created is an instantiation of your application for that specific tenant (Service Principal).

You can think that the **App Registration** step is like creating the definition of how you want your application to behave and in the **Enterprise Applications** section you're seeing the instantiation of that definition for that specific tenant.  

Also in the **Enterprise Application** section you can find applications published by other companies that can be used within your organization. For example, if you want to federate AWS and manage SSO within your organization, you can integrate it from the **Enterprise Applications** section.

If you're registering a new app on AAD I'm pretty confident that almost all your time is going to be spent on the **App Registrations** section, that's where you're going to define the app branding, how the authentication is going to work, which permissions your application is going to expose and which permissions is going to consume, what claims you want to add in your tokens and how the service will issue tokens.   
Meanwhile on the **Enterprise Applications** section you are going to configure who can access the application and under which conditions.   

> I tried to explain it using my own words, but I think that the best explanation possible to this qüestion lies right here:  https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals


# Q2: What's the difference between AppRoles and Scopes? And how can I add a new Scope / AppRole in my application?

When talking about **scopes** on AAD we're refering to the collection of permissions that an API (resource) exposes to client apps.   
These permission scopes need to be granted to client apps via consent grant.   

> If you take a look at the application manifest the scopes are being called called "oauth2permissions", 

![manifest-oauth2permissions](/img/scopes-ouath2permissions.png)

You can add new scopes to an app from the "Expose API" section

![add-scopes](/img/add-scopes.png)

If your app needs to consume another app already registered on AAD it can request which scopes it needs from the "API permissions" section

![api-permissions](/img/api-permissions.png)

When talking about **appRoles** we should first talk about RBAC. Role-based access control or RBAC is a popular mechanism to enforce authorization in applications. When using RBAC, an administrator grants permissions to roles, and not to individual users or groups. The administrator can then assign roles to different users and groups to control who has access to what content and functionality.    
AppRoles are the collection of roles that an app may declare. These roles can be assigned to users, groups, or service principals.   

If you need to add a need AppRole onto your app you need to add it directly into the app manifest, **there is not a UI for it**.

![approles-manifest](/img/aad-approles-manifest.png)

If you want to assign users or groups onto an AppRole you need to do it from the "Enterprise Application" blade. You can't do it from the "App Registration" blade.

![add-users-to-app](/img/aad-add-users-to-app.png)

# Q3: What's the difference between delegated permissions and application permissions?

**Delegated permissions** are used when you want to authenticate to a WebAPI or other services with the currently logged-on user.   
This involves a physical user and a user interface.

**Application permission** are used when there is no user present. Mostly used for API to API calls. This is also used for background and daemon services.   
Unlike delegated permissions, application permissions, however, uses the app id and secret to login and always has the given permissions of the application.


# Q4: Can I add a delegated permission and also an application permission in my application?

Absolutely. "Application Permissions" and "Delegated Permissions" are completely independent of one another.    
For example,  your application might have a section that requires user interaction and needs that a user is logged-in and another section that is being accesed programmatically by another application.

Here are a couple of tidbits when working with delegated and application permissions:
- In application permissions the "scp" claim is not present in the access token. You need to use the "roles" attribute.
Meanwhile for delegated permissions if the user logged has a role assigned the token will contains both claims: "scp" and "roles".
- If your working with the Graph API you might need different permissions depending if you're using delegated or application permissions. Take a quick look into the Microsoft documentation before assuming that the same permissions will work in both scenarios. For example: https://docs.microsoft.com/en-us/graph/api/drive-get?view=graph-rest-beta&tabs=http


# Q5: My application needs permission to call Microsoft Graph API, what I need to do?

That's probably one of the simplest things to do on App Registrations.   
Just register your application and after that navigate to the "API permissions" section, select "Add permission" > "Microsoft Graph" > select between delegated or application permissions >  select which permissions you need from the Microsoft Graph API.

But be careful and don't get confused between the "Microsoft Graph (https://graph.microsoft.com/)" Api and the "Azure Active Directory Graph" Api (https://graph.windows.net/).    
The AAD Graph is the old version and is currently deprecated, you should always use the Microsoft Graph API.


# Q6: What's the use of the .default scope?

The .default scope is a built-in scope for every application and it refers to the static list of permissions configured on the application registration.
If we request an access token using the .default scope our users will be asked to consent to all of the configured permissions present on our Azure AD App Registration for that specific resource and in return they will get an access token containing all the permissions present.

Let's see a couple of quick examples:

- ## Example 1
  
I have register an app with the following permissions:

![default-scope-permissions-1](/img/default-scope-permissions-1.png)

And if I request an access token using the .default scope

```javascript
https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/oauth2/v2.0/authorize
?client_id=5833d6c7-1ba5-4184-9b14-050a6b5ed212
&redirect_uri=https://oidcdebugger.com/debug
&scope=https://graph.microsoft.com/.default
&response_type=token
&response_mode=form_post
&nonce=yj8o2c250eq
```
I get an access token containing all the scopes that where present on the permissions section

```javascript
{
   ...
   "scp": "AccessReview.Read.All Agreement.Read.All Calendars.Read Device.Read.All profile openid email",
   ...
}

```

- ## Example 2:

The payment app exposes these 3 scopes:

![default-scope-permissions-2](/img/default-scope-permissions-2.png)

The Booking app has the following permissions for the payment app:

![default-scope-permissions-3](/img/default-scope-permissions-3.png)

If I request an access token using the .default scope

```javascript
https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/oauth2/v2.0/authorize
?client_id=c3b8b95f-922b-4384-8ab2-a1215a9fc295
&redirect_uri=https://oidcdebugger.com/debug
&scope=api://payment/.default
&response_type=token
&response_mode=form_post
&nonce=wq2vzi24b5
```

I get an access token containing all the scopes for the payment api that where present on the permissions section

```javascript
{
   ...
   "scp": "payment.read payment.write",
   ...
}
```

> You can find a more flesh out explanation to this question right here: https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent


# Q7: I have built a WebApi using csharp and I want to add authentication and authorization using AAD. What library should I use? ADAL.NET, MSAL.NET or Microsoft.Identity.Web? 

The question between ADAL.NET and MSAL.NET is a pretty common question and to make things worse now we have another new library...  

The main difference between ADAL and MSAL is that ADAL.NET integrates with the Azure AD for developers (v1.0) endpoint, where MSAL.NET integrates with the Microsoft identity platform (v2.0) endpoint. 

![adal-vs-msal](/img/adal-vs-msal.png)


If you can choose between ADAL.NET and MSAL.NET just use MSAL.NET.   

MSAL.NET is the newer version of Microsoft authentication libraries and it will allow you to acquire tokens for users signing-in to your application with Azure AD, Microsoft accounts or Azure AD B2C.   

If you have an old app lying around that is using the AAD for authentication probably is using the ADAL.NET library, but if you're creating a new application right now it doesn't make sense to use ADAL.NET.

But what about Microsoft.Identity.Web? 

> Microsoft Identity Web provides the glue between the ASP.NET Core middleware and MSAL .NET to bring a clearer, more robust developer experience, which also leverages the power of the Microsoft identity platform (formerly Azure AD v2.0 endpoint), and leverages OpenId Connect middleware, which means developers can develop applications which allow several identity providers, including integration with Azure AD B2C.

![microsoft-web-identity-fits](/img/microsoft-web-identity-fits.png)

That's how Microsoft describes this new library.

Microsoft.Identity.Web is the most recent library for authentication (version 1.0.0 was released 15 days ago) and from my standpoint it seems that with this library Microsoft is trying to simplify the multiple steps you needed to perform in your application when using MSAL.NET.     
Nonetheless if you want to use the library there is catch: **it only works with .NET Core 3.1 and .NET 5**

If you want to know more about this library, go here: https://github.com/AzureAD/microsoft-identity-web/wiki

Just to make it easy:
- ADAL.NET is obsolete, do not use it. If you find and old application using ADAL.NET you should try to migrate it to MSAL.NET, there is even a page explaining with detail how to do it: https://docs.microsoft.com/es-es/azure/active-directory/develop/msal-net-migration   
- If you're working with .NET Core 3.1 and you don't mind using a super recent library use Microsoft.Web.Identity.   
But if you plan to install the library in your app and forget about it, that might not be a good fit for you. The library is so recent that probably is going to bump some versions pretty quick. In fact version 1.1.0 is already out.   
- For everything else just use MSAL.NET


# Q8: I have a SPA (Single Page Application) that makes a call to a protected WebAPI, this protected API needs to call another protected WebAPI. What authentication protocol flow should I use?   

In this case you need to use the OAuth 2.0 On-Behalf-Of flow (OBO Flow).   
The idea is to propagate the delegated user identity and permissions through the request chain.
The SPA has to make authenticated requests to the WebApi and also the middle-tier API need to make authenticated requests to the downstream services.

Let me try to build a quick example to see how the OBO Flow works. I'm going to build the following example:

![obo-flow](/img/obo-flow.png)

1 - The SPA signs a user using MSAL.js and in response acquires an access token for the WebApi 1.   
2 - The SPA calls the WebApi 1 passing the access token.   
3 - The WebApi 1 authorizes the caller and uses the access token that has received to request another access token for WebApi 2.   
4 - WebApi 1 uses the new access token that has acquired to call WebApi 2.   

For that example I'm going to use the **Microsoft.Identity.Web** library
-  If you want to take a look at the code, it's on my GitHub: https://github.com/karlospn/testing-obo-flow-with-microsoft-identity-web-library

Let me comment the most interesting parts of the demo:

- On the WebApi 1 project: 
  
```csharp
using Microsoft.Identity.Web;

public class Startup
{
   public void ConfigureServices(IServiceCollection services)
   {
      services.AddMicrosoftIdentityWebApiAuthentication(Configuration)
            .EnableTokenAcquisitionToCallDownstreamApi()
            .AddDownstreamWebApi("WebApi2", Configuration.GetSection("WebApi2"))
            .AddInMemoryTokenCaches();

      services.AddControllers();
   }
}

```

WebApi 1 needs to acquire a token for the downstream API (WebApi 2). You specify it by adding the _.EnableTokenAcquisitionToCallDownstreamApi()_ line after _.AddMicrosoftIdentityWebApi(Configuration)_.   
You can call the downstream API without adding the extension _.AddDownstreamWebApi()_ but it's really useful because now I can directly inject an _IDownstreamWebApi_ service in my controller and use it to call the WebApi 2 app without the need to build an HttpClient.

```csharp

[ApiController]
[Authorize]
[Route("api/invoke")]
public class InvokeController : ControllerBase
{

   private readonly ILogger<InvokeController> _logger;
   private IDownstreamWebApi _downstreamWebApi;

   public InvokeController(ILogger<InvokeController> logger, 
      IDownstreamWebApi downstreamWebApi)
   {
      _logger = logger;
      _downstreamWebApi = downstreamWebApi;
   }

   [HttpGet]
   public async Task<ActionResult> Get()
   {
      _logger.LogInformation("Trying to call a downstream Api");
      var value = await _downstreamWebApi.CallWebApiForUserAsync(
            "WebApi2",
            options =>
            {
               options.HttpMethod = HttpMethod.Get;
               options.RelativePath = "api/downstream";
            });

            return Ok(value);
        }
    }

```

- On the WebApi 2 project:

We simply need to add _.AddMicrosoftIdentityWebApiAuthentication()_ on the _Startup.cs_.   
This method enables your WebAPI to be protected using the Microsoft identity platform. This includes validating the token in all scenarios (single- and multi-tenant applications).   

```csharp
public void ConfigureServices(IServiceCollection services)
{
   services.AddMicrosoftIdentityWebApiAuthentication(Configuration);
   services.AddControllers();
}
```

# Q9: Can I define custom scopes and have them returned when using the client credential flow in Azure AD?

No, you can't.
What you can do is define appRoles and have them returned via the client credential flow. Also you need to specify the .default scope in a client credentials flow, you cannot specify a concrete scope.

Just like this:

```javascript
curl -X POST https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/oauth2/v2.0/token 
-H 'Cache-Control: no-cache' 
-H 'Content-Type: application/x-www-form-urlencoded' 
-d 'grant_type=client_credentials&client_id=c3b8b9 5f-922b-4384-8ab2-a1215a9fc295&client_secret=e-M2va9la3O1pl~x1HPQNVXKSrK-a9_iRK&scope=api://payment/.default'
```

# Q10: I have registered 2 apps on my AAD and I'm having some trouble validating the issuer attribute from the access tokens. The value of the issuer attribute in an access token issued for the app1 is "login.microsoftonline.com" and the value of the issuer attribute in an access token issued for the app2 is "sts.windows.net". What's happening?

Imagine that you are building a protected WebApi and you want to validate the access tokens.    
Two of the attributes you could validate are the "Issuer" and the "Audience".   
The issuer attribute (iss) identifies the security token service (STS) that constructs and returns the token and also the Azure AD tenant in which the user was authenticated.   
The audience attribute (aud) identifies the intended recipient of the token.
  
But you need to know that the issuer and the audience attributes have **different values** depending if the access token is issued by the Azure Active Directory V1 endpoint or the Azure Active Directory V2 endpoint.

And probably the first thing that comes into your mind after reading up to this point:
- _"Nowadays no one is using the AAD V1 endpoint anymore, why should I care about it?"_

But here comes the catch: **The AAD V2 endpoint is capable of issuing tokens in V1 and V2 format. And by default is issuing tokens in V1 format.**   

The parameter that allows you to switch between tokens formats is called **"accessTokenAcceptedVersion"** and you can find it in the application manifest (https://docs.microsoft.com/es-es/azure/active-directory/develop/reference-app-manifest)   
The possible values for accesstokenAcceptedVersion are 1, 2, or null. If the value is null, this parameter defaults to 1, which corresponds to the v1.0 endpoint.

![aad-accesstokenacceptedversion-attribute](/img/aad-accesstokenacceptedversion-attribute.png)

A common issue I'm seeing around in a few places is people thinking that they can validate exactly the same way all the access token issued from the V2 endpoint.   
You're going to run in some troubles if you're try to validate the issuer and the audience from an access token **issued** by the AAD V2 endpoint but in the **V1** format.   

- Let me show you what I'm trying to explain with an example: 

Here's an access token that has been issued when the attribute "accesstokenAcceptedVersion" is set to **null** or **1**:

```javascript
{
   "aud": "api://42216d82-c01b-45c6-8bcf-511d472900b1",
   "iss": "https://sts.windows.net/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/",
   "iat": 1602591474,
   "nbf": 1602591474,
   "exp": 1602595374,
   "acr": "1",
   "aio": "AVQAq/8RAAAA2p8+WYcyz9UuytxuBb6mjh6aVBnvgFIZgPeLy3qshHe2H6ffBuyTZ7KLLnG125Tpc9qtO9fvGy2hAVQKa4c1qoe1VcYYWlu+V43D3m5Q5OI=",
   "amr": [
      "pwd"
   ],
   "appid": "42216d82-c01b-45c6-8bcf-511d472900b1",
   "appidacr": "0",
   "email": "cpn@outlook.com",
   "family_name": "p",
   "given_name": "c",
   "idp": "live.com",
   "ipaddr": "10.10.10.10",
   "name": "cpn",
   "oid": "221f4ba1-8bf4-4083-bd22-ba078fd56580",
   "rh": "0.AR8A4nEGijA6ME2cua1wm5x0SoJtIUIbwMZFi89RHUcpALEfALk.",
   "scp": "api1.read",
   "sub": "LFctpniExhuoNWlwh52C_SRnQN4Z74hkUZu1AhceU6s",
   "tid": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
   "unique_name": "live.com#cpn@outlook.com",
   "uti": "hx1a4jDUAkyucKxfwaE-AA",
   "ver": "1.0"
}
```

And here's an access token that has been issued when the attribute "accesstokenAcceptedVersion" is set to **2**:

```javascript
{
   "aud": "42216d82-c01b-45c6-8bcf-511d472900b1",
   "iss": "https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/v2.0",
   "iat": 1602594547,
   "nbf": 1602594547,
   "exp": 1602598447,
   "aio": "AWQAm/8RAAAACGs+Od0VxVRWvsNt4b463+iPnuLt0uHgEKFB+Zh6BCIp4RooCkDrxdnGKIp+ku+QahSYbBTHCXnOzk+gO6rj4DIpWdsMSULPmyBdBJKpHzhqdpX8Xcmexa9axdA3CRj7",
   "azp": "42216d82-c01b-45c6-8bcf-511d472900b1",
   "azpacr": "0",
   "idp": "live.com",
   "name": "cpn",
   "oid": "221f4ba1-8bf4-4083-bd22-ba078fd56580",
   "preferred_username": "cpn@outlook.com",
   "rh": "0.AR8A4nEGijA6ME2cua1wm5x0SoJtIUIbwMZFi89RHUcpALEfALk.",
   "scp": "api1.read",
   "sub": "LFctpniExhuoNWlwh52C_SRnQN4Z74hkUZu1AhceU6s",
   "tid": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
   "uti": "uAv96bNnUkqogjeo0nFuAA",
   "ver": "2.0"
}
```

As you can see both the issuer and the audience values are completely different, so be mindful if you do some kind of token validations like this one:

```csharp
app.UseWindowsAzureActiveDirectoryBearerAuthentication(
   new WindowsAzureActiveDirectoryBearerAuthenticationOptions
   {
         Audience = "https://sts.windows.net/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/"
   });
```
Because it only would work if the attribute "accessTokenAcceptedVersion" is set to 1 or null.