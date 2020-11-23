---
title: "Trying to automate Azure Active Directory App Registration process using Terraform"
date: 2020-07-21T21:22:44+02:00
tags: ["azure", "terraform", "AAD"]
draft: false
---

Today I want to try to use Terraform to automate the app registration process in Azure Active Directory.

If you want to secure an application Azure Active Directory is a really good option, but I don't want to configure my application on AAD manually, what I really want is to add a step in my CI / CD pipeline that does that for me, and for that purpose Terraform might be a good option.   

Terraform already has an official Azure Active Directory provider written by Microsoft itself (https://www.terraform.io/docs/providers/azuread/index.html), so in today's post I'm going to focus on trying it out. 

I'm going to build a pretty common and straightforward scenario using the Terraform provider. The scenario is the following one:

- **Payment API**: That's going to be our resource server.
  - It exposes 2 scopes : **payment.write** and **payment.read**
  - It has 2 application roles: **Reader** and **Admin**

- **Booking API**: 
  - Consumes the Payment API using a Client Credentials flow.
    - Obtains an access_token from the AAD token endpoint and uses it to attain access to the Payment API.
  - It has the Payment API Reader Role assigned.

- **FrontEnd SPA**:
  - It's a _"Single Page Application"_ written in some crazy Javascript framework.
    - Uses an implicit flow to obtain an access_token and id_token and uses the access_token to attain access to the Payment API.
  - The FrontEnd SPA app has permission only to ask for the payment.read scope.

- **Users**:
  - I have created  **2 users** in my AAD: **John** and **Jane**
    - Jane has assigned a Reader role in the Payment API app
    - John has assigned an Admin role in the Payment API app



Let's start building it, I need to register 3 apps. But first of all I need to configure the azuread provider.   

## Step 1: Configure the Azure AD provider
---

The first step is to configure the AzureAD Provider.   
To authenticate against my AAD I'm going to create a new Application and a Service Principal and use its client id and a client secret.   
There are other options available to authenticate against the AAD using the provider, you can read it here :https://www.terraform.io/docs/providers/azuread/guides/service_principal_client_secret.html

Basically what I'm going to do is create a _"master app"_ in my AAD, a "master app" is nothing more than an app with permissions to create another apps. Here is a detailed walkthrough about how to do it: https://www.terraform.io/docs/providers/azuread/guides/service_principal_configuration.html  

The first weird thing that you're going to find while creating the "master app" is the fact that the provider uses the **Legacy Azure Active Directory API (Azure Active Directory Graph)** instead of the newer MS Graph API.   
That's a bad sign to begin with, it means that all the most recent features probably are not doable with the provider.   
But let's going forward, that's the final look after registering in my AAD the master app and giving it the proper permissions:

![master-app-config](/img/tf-aad-master-app.png)

Now we can configure the Terraform provider using the master app client_id and client_secret.

```yaml

provider "azuread" {

  client_id         = "ba4d0620-0522-4ada-b0b6-0cdd8cfaeae7"
  client_secret     = "my_secret_goes_here"
  tenant_id         = "my_tenant_goes_here"
  subscription_id   = "my_subscription_id_goes_here"
}

```

## Step 2: Register the payment API
---

Next step is to create the payment API using Terraform.
The payment API has the following configuration:

  - It exposes 2 scopes : payment.write and payment.read.
  - It has 2 application roles: Reader and Writer.
  
```yaml

resource "azuread_application" "payments_api" {
    name                       = "payments_api"
    available_to_other_tenants = false
    oauth2_allow_implicit_flow = false
    type                       = "webapp/api"
    identifier_uris            = ["api://payment"]

  
    oauth2_permissions {
        admin_consent_description  = "Allow the application to access the commit payment methods"
        admin_consent_display_name = "payment.write"
        is_enabled                 = true
        type                       = "Admin"
        value                      = "payment.write"
    }

    oauth2_permissions {
        admin_consent_description  = "Allow the application to access the read payment methods"
        admin_consent_display_name = "payment.read"
        is_enabled                 = true
        type                       = "User"
        value                      = "payment.read"
        user_consent_description  = "Allow the application to access the read payment methods"
        user_consent_display_name = "payment.read"    
    }

    app_role {
        allowed_member_types = [
        "User",
        "Application",
        ]
        description  = "Can read and make payments"
        display_name = "Admin"
        is_enabled   = true
        value        = "Admin"
    }

    app_role {
        allowed_member_types = [
        "User",
        "Application",
        ]

        description  = "Can only read payments"
        display_name = "Reader"
        is_enabled   = true
        value        = "Reader"
    } 
}

resource "azuread_service_principal" "payment_sp" {
  application_id               = azuread_application.payments_api.application_id
  app_role_assignment_required = false
  tags = [ "payment", "api"]
}

```

It's a pretty straightforward config file but I have encountered some issues while building it.   

The first issue:
   
I can create the application roles but **there is NO way to assign users or groups to those roles**.   
I can go into the  _"Enterprise applications"_ blade in the Azure Active Directory Portal, select my payment app and  assign users and groups manually, but that's a bummer...

Poking around Github (https://github.com/terraform-providers/terraform-provider-azuread) I found that it's an already know issue  ( https://github.com/terraform-providers/terraform-provider-azuread/issues/230) and it seems that the issue is because the provider is using the legacy AAD api...

The other issue I found is that not all the Manifest attributes are present.   
For example, I like to change the _"accessTokenAcceptedVersion"_ attribute so the token endpoint only generates tokens in the V2 format _(I will talk about that nonsensical behaviour in a future post...)_ but I cannot do it with the provider, I have to change it manually again...


## Step 3: Register the Booking API
---

The Booking API has the following configuration:
  - Consumes the Payment API using a Client Credentials flow.
    - Obtains an access_token from AAD and uses it to attain access to the Payment API.
  - The Booking API has the Payment API Reader Role assigned.

```yaml

resource "azuread_application" "booking_api" {
    name                       = "booking_api"
    available_to_other_tenants = false
    oauth2_allow_implicit_flow = false
    type                       = "webapp/api"
    identifier_uris            = ["api://booking"]

    required_resource_access {
        resource_app_id = azuread_application.payments_api.application_id
        resource_access {
            id   = tolist(azuread_application.payments_api.app_role)[0].id
            type = "Role"
        }
    }
}


resource "azuread_service_principal" "booking_sp" {
  application_id               = azuread_application.booking_api.application_id
  app_role_assignment_required = false
  tags = [ "payment", "api"]
}

resource "azuread_application_password" "booking_api_pwd" {
  application_object_id = azuread_application.booking_api.id
  description           = "My managed password"
  value                 = "VT=uSgbTanZhyz@%nL9Hpd+Tfay_MRV#"
  end_date              = "2099-01-01T01:02:03Z"
}

```

Apart from creating the application I'm also creating a client secret to test the client credentials flow. Nonetheless there is another issue I found in the process.   
The issue is that I cannot grant consent to use the role in an automatically way, so I need to do it manually...

![booking-app-grant-admin-consent](/img/aad-api-grant-admin-consent.png)


After doing that, let's test it and see if it works. I'm going to request an access token using the Booking API client id and client secret.

```bash

curl -X POST \
  https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/oauth2/v2.0/token \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials&client_id=5cd49945-086c-4605-9f86-00fe08134dab&client_secret=VT%3DuSgbTanZhyz%40%25nL9Hpd%2BTfay_MRV%23&scope=api%3A%2F%2Fpayment%2F.default'

```

And it returns an access_token with the following attributes:

```json
{
  "aud": "api://payment",
  "iss": "https://sts.windows.net/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/",
  "iat": 1595422827,
  "nbf": 1595422827,
  "exp": 1595426727,
  "aio": "E2BgYGAtTs258t5NtmeuveDN2979AA==",
  "appid": "5cd49945-086c-4605-9f86-00fe08134dab",
  "appidacr": "1",
  "idp": "https://sts.windows.net/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/",
  "oid": "3b78fe9b-7641-4a73-9b0c-bd05383a27bc",
  "rh": "0.AR8A4nEGijA6ME2cua1wm5x0SkWZ1FxsCAVGn4YA_ggTTasfALk.",
  "roles": [
    "Reader"
  ],
  "sub": "3b78fe9b-7641-4a73-9b0c-bd05383a27bc",
  "tid": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
  "uti": "41oex9rUzU28oibY_D0hAA",
  "ver": "1.0"
}
```

So far so good, the issuer and the audience are both correct and it also contains the Reader application Role.


## Step 4: Register the FrontEnd SPA
---

The FrontEnd SPA has the following configuration:
  - It's a public _"Single Page Application"_ written in Angular. Uses an implicit flow to obtain an access_token and a id_token and uses the access_token to attain access to the Payment API
  - The FrontEnd SPA app has permission only to ask for the payment.read 
  scope

```yaml

resource "azuread_application" "frontend_spa" {
    name                       = "frontend_spa"
    available_to_other_tenants = false
    oauth2_allow_implicit_flow = true
    type                       = "webapp/api"
    reply_urls                 = ["http://localhost:4200"]
    logout_url                 = "https://localhost:4200/logout"
    public_client              = true

    required_resource_access {
        resource_app_id = azuread_application.payments_api.application_id
        resource_access {
            id   = tolist(azuread_application.payments_api.oauth2_permissions)[1].id
            type = "Scope"
        }
    }
}


resource "azuread_service_principal" "frontend_spa" {
  application_id               = azuread_application.frontend_spa.application_id
  app_role_assignment_required = false
  tags = [ "frontend", "spa"]
}
```

I have found the same problem that I mention in the step 3: I cannot grant admin content to use the payment API scopes in a programmatic way.

![frontend-spa-grant-admin-consent](/img/aad-spa-grant-admin-consent.png)

And there is another issue that bugs me, the issue is that if you're using the Azure Portal to register an SPA you can specify that the application type is "SPA" and use the grant type auth code flow with PKCE, but that option is missing here, so I'm forced to use and implicit flow.   
It seems that someone else opened an issue about it in Github (https://github.com/terraform-providers/terraform-provider-azuread/issues/286)

But anyway, let's test it.

I'm going to begin an implicit flow and login into the AAD as **Jane**.   
Remember from the step 2 that I have **manually** assigned a Reader role in the Payment API to Jane.   
The fastest way to begin an implicit flow is by building the URI by myself.


```bash

https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/oauth2/v2.0/authorize
  ?client_id=c6b731f2-785b-4206-974a-db8d34eb4397
  &redirect_uri=https%3A%2F%2Foidcdebugger.com%2Fdebug
  &scope=api%3A%2F%2Fpayment%2Fpayment.read%20openid
  &response_type=token%20id_token
  &response_mode=form_post
  &nonce=iaiqs1h3pa

```

And it returns an access_token with the following attributes:

```json
{
   "aud": "api://payment",
   "iss": "https://sts.windows.net/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/",
   "iat": 1595425611,
   "nbf": 1595425611,
   "exp": 1595429511,
   "acr": "1",
   "aio": "ATQAy/8QAAAAOe3HCSYBGo663Mt+8XSEK/yY+P8Ao4qLGurtTMz5S9VtG7FBYdfpCiPb3qP59gHO",
   "amr": [
      "pwd"
   ],
   "appid": "c6b731f2-785b-4206-974a-db8d34eb4397",
   "appidacr": "0",
   "given_name": "doe",
   "ipaddr": "00.00.000.0",
   "name": "jane",
   "oid": "0a92ffc5-9551-47a0-baf0-7f3352eac015",
   "rh": "0.AR8A4nEGijA6ME2cua1wm5x0SvIxt8ZbeAZCl0rbjTTrQ5cfAAc.",
   "roles": [
      "Reader"
   ],
   "scp": "payment.read",
   "sub": "ODPx3tnkeekXKN1Olvx8pD5e5PcXJMCg0LoaHz3F14g",
   "tid": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
   "unique_name": "jane@cpnoutlook.onmicrosoft.com",
   "upn": "jane@cpnoutlook.onmicrosoft.com",
   "uti": "wYVNdVa5Bk6FE7iZjNkjAA",
   "ver": "1.0"
}
```

Everything looks alright: issuer, audience, scopes, upn, roles.


## Conclusion
---

I tried to build a pretty common scenario and as you can see it's pretty straightforward to register an app on AAD using Terraform. If you already have a knowledge about how AAD works it's going to be easy work switching to Terraform.   
But be aware that some features are missing, just tinkering with the provider for a very brief period of time I have already found some missing features:

- You cannot assign users or groups into an app
- You cannot grant admin consent programatically
- It's missing the grant type auth code flow with PKCE

Those issues can be resolved using Powershell or the AZ CLI and building some kind of script that fills in those missing steps, but I don't want to manage both files.    

I'm also surprised that the provider is using the Legacy Azure Active Directory API (Azure Active Directory Graph) instead of the newer MS Graph API, that raises some doubts about the adoption of the new features that are only possible using the newer Graph API, so be aware of it.




