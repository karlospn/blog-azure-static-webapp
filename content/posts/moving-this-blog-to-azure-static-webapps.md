---
title: "Trying Azure Static WebApps. Trying to deploy this blog into an Azure Static Webapp"
date: 2020-11-22T17:14:40+01:00
tags: ["azure", "static", "webapps", "github"]
draft: true
---

I been wanted to try Azure Static WebApps for quite some time, so I have thought that instead of using some dummy static page I'm going to try to deploy this blog page.   

>If you only want to see the end result and not bother reading the post. I upload everything here: **[github-link](https://github.com/karlospn/dotnetramblings_source/tree/feature/static-web-app)**   
This branch contains a folder called **"infrastructure"**, that's where you can find the ARM template that creates the Azure Static WebApp.  
>And in the **.github** folder you can find both GitHub actions: 
>- The **infrastructure.yml** is a pipeline that creates the resource group, deploys the ARM template and sets the custom domain.
>- The **application.yml** is a pipeline that deploys the blog source code into the WebApp.


This blog it's just a bunch of static files, so it should be quite easy to deploy into an Azure Static WebApp.   
Right now it uses **Hugo** as a static site framework and **GitHub Pages** to host it.
The current workflow looks like this:

<FOTO>

- A GitHub repository hosts the source code. 
- When the code is pushed to master a **GitHub Action** builds the source code and pushes the artifact onto another GitHub repository.
- The second GitHub repository is configured as a GitHub Pages repository and have a custom domain associated.

As you can see it's a super straight-forward CI/CD process: I write a new post, I push it to master and it gets published. No need for anything more complicated.    
And I'm hoping to maintain that simplicity when I move it to Azure Static WebApps.

I have 3 goals in mind that I want to test with this move to Azure Static WebApps:
- Goal 1: Use Azure DevOps as my VCS (Version control system).
- Goal 2: Use an infrastructure-as-code (IaC) approach when creating the resources on Azure.
- Goal 3: Maintain the CI/CD simplicity that I have right now on GitHub.


## Goal 1. Moving the blog source code to Azure DevOps

I would like to move the blog source code from GitHub to Azure DevOps because that's where I'm hosting nowadays all my private projects. But it seems that you can't do that.

- **Azure Static WebApps doesn't support Azure DevOps** nowadays. 
- It **only support GitHub as a VCS**, you can't use any other version control system like Azure DevOps or GitLab.

But to be fair Azure Static WebApps it's still on preview and after poking around on the Azure GitHub repository it seems that they are working on supporting Azure DevOps: https://github.com/Azure/static-web-apps/issues/5   

So, I guess I'm staying on GitHub for now.


## Goal 2. Create the Azure Static WebApps blog using IaC

I'm going a little bit fancy here. For a project as simple as this I could create the WebApp directly on the Azure portal, but let's try to follow some good DevOps practices and provision all the resources using IaC.

My de facto IaC tool is **Terraform**, but it doesn't support Azure Static WebApps. 
They're working on it: https://github.com/terraform-providers/terraform-provider-azurerm/pull/7150   
Pulumi also doesn't support it.   
So it seems that my best option right now is using **ARM templates**.   

Let me show you the final result. That's the ARM template:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "type": "string"
        },
        "location": {
            "type": "string"
        },
        "sku": {
            "type": "string"
        },
        "skucode": {
            "type": "string"
        },
        "repositoryUrl": {
            "type": "string"
        },
        "branch": {
            "type": "string"
        },
        "repositoryToken": {
            "type": "securestring"
        },
        "appLocation": {
            "type": "string"
        },
        "apiLocation": {
            "type": "string"
        },
        "appArtifactLocation": {
            "type": "string"
        },
        "resourceTags": {
            "type": "object"
        }
    },
    "resources": [
        {
            "apiVersion": "2019-12-01-preview",
            "name": "[parameters('name')]",
            "type": "Microsoft.Web/staticSites",
            "location": "[parameters('location')]",
            "tags": "[parameters('resourceTags')]",
            "properties": {
                "repositoryUrl": "[parameters('repositoryUrl')]",
                "branch": "[parameters('branch')]",
                "repositoryToken": "[parameters('repositoryToken')]",
                "buildProperties": {
                    "appLocation": "[parameters('appLocation')]",
                    "apiLocation": "[parameters('apiLocation')]",
                    "appArtifactLocation": "[parameters('appArtifactLocation')]"
                }
            },
            "sku": {
                "Tier": "[parameters('sku')]",
                "Name": "[parameters('skuCode')]"
            }
        }
    ]
}
```
And that's the parameters file:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "value": "mytechramblings-blog"
        },
        "location": {
            "value": "westeurope"
        },
        "sku": {
            "value": "Free"
        },
        "skucode": {
            "value": "Free"
        },
        "repositoryUrl": {
            "value": "https://github.com/karlospn/dotnetramblings_source"
        },
        "branch": {
            "value": "feature/static-web-app"
        },
        "repositoryToken": {
            "value": "" 
        },
        "appLocation": {
            "value": "/"
        },
        "apiLocation": {
            "value": ""
        },
        "appArtifactLocation": {
            "value": "public"
        },
        "resourceTags": {
            "value": {
                "Environment": "Dev",
                "Project": "personal-blog",
                "ApplicationName": "mytechramblings-blog"
            }
        }
    }
}
```
The **"repositoryToken"** parameter must contain a GitHub PAT with admin/write access to the repository and the ability to interact with workflows.    
Right now I'm leaving it empty and I will inject the right value during the CI/CD pipeline.   
I'm also leaving the **apiLocation** parameter empty, you should include an api location only if your project uses an Azure Function.

I'm pushing these ARM template files into a folder called **"infrastructure"**.

<FOTO>

## Goal 3: Building the CI/CD pipeline

Not a big fan of what they have built here, and that's mainly because:

- An entire CI/CD pipeline is automatically created for you with Github Actions when you create your web app.
  
It sounds kind of weird to me. After provisioning a new web app if you browse into your github repository you're going to find out that a Github action has been pushed onto it. And also has already ran once.   
Maybe it can be turned off but I didn't find a way to do it.   
The bright side about that behaviour is that without doing absolutely nothing you have a working CI/CD pipeline already built on your GitHub repository.   
The bad thing about that behaviour is that without doing absolutely nothing you have a working CI/CD pipeline already built on your GitHub repository. Why it's a bad thing? Well because I want to build the pipeline myself and do whatever I want..
After the pipeline has ran once you can delete it or modify it, but I still don't like that approach.

That behaviour trumps my original idea of having a single pipeline that does everything: creates the needed infrastructure on Azure, compiles the blog source code and deploys it.   
If you try to do it you're going to end up with 2 pipelines that are doing exactly the same: the one that you created and the one that the service has created for you.

So instead of having a single pipeline I guess I should build 2 pipelines:

- The first pipeline will create the infrastructure using the ARM template. It will have a manual trigger.

- And for the second pipeline I'm going to use the pipeline that the services that the service has created automatically for me.

To be honest I find that deployment workflow looks a little bit weird:

- First of all I need to run manually the pipeline that will create the resources on Azure. 
  - That pipeline will create another pipeline that deploys the source code.
- Now I can use the auto-generated pipeline for later deployments. 


Those are the end results: 

- Pipeline 1:

```yaml

```

- Pipeline 2:

```yaml

```


## Final thoughts

My impressions after tinkering a little bit with Azure Static WebApps is that it's lacking a little bit in the deployment department and in the VCS department.   

I get the impression that the service is trying a little too hard to automate things for me, it's a nice feature that it creates a fully functional CI/CD pipeline for me but I want the ability to turn that behaviour off and use whatever I want to deploy the code.

Also I think that support for Azure DevOps is a must. GitHub is great and all, but seems a weird decision to support only GitHub.

Take everything I said here, with a grain of salt. The service it's still in preview so maybe in a couple a months they have fixed all the problems that I have encountered today. 
