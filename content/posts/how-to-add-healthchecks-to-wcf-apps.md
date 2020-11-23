---
title: "Why and how you should add a healthcheck endpoint to your Wcf legacy applications"
date: 2020-07-04T17:30:06+02:00
tags: ["dotnet", ".NET", "wcf", "legacy", "csharp", "azure", "appservices"]
draft: false
---

Nowadays no one on his right mind is going to create a WCF from scratch. It really makes no sense to do it. It's a deprecated technology, it does not work with .NET Core, only works on Windows OS and personally I found the configuration confusing as hell, every time I need to modify an existing one I have to spend a good amount of time trying to figure out what everything means.

If you are not old enough, probably you're asking yourself: _"What the hell is WCF?"_.   
WCF stands for "Windows Communication Foundation", and if you want to know more about read the official Microsoft docs: [here](https://docs.microsoft.com/es-es/dotnet/framework/wcf/whats-wcf)

If you want to build a .NET application from scratch right now, most likely you are going to build an app using .NET Core, probably is going to use REST over HTTP or even GRPC and it is also pretty likely that you are going to host it in a container platform, a serverless function or a Paas platform like Azure App Services.   
But let's be honest if you're working for an old enough company and a company that uses .NET as its main stack, it's pretty likely that you are going to stumble with a bunch of legacy applications in some lost server and I'm pretty sure you are going to find some WCF applications among them.  
These past few years I worked as a freelance quite few times and I know as a fact that a lot of .NET shops are still running WCFs nowadays.

What I wanted to talk in these post is how easy is to add a healthcheck endpoint in a WCF application, and how you can use it in case you want to move the application into the Cloud in the near future.

> With the expansion of cloud computing and also with the huge adoption of Kubernetes almost everyone knows what a healthcheck endpoint is and why is good to have one in your application, so I'm gonna skip that part.

I'm also aware that in .NET 4.8 Microsoft added the possibility to add a healthcheck endpoint in your WCF application using a **ServiceHealthBehavior**   
But I'm going to give another option, that option is to build a healthcheck REST endpoint inside your WCF application.   
Having multiple options is always nice and maybe in your company they cannot update the .NET framework version to the latest version.


# Build the healthcheck library for wcf


>  I push the code into my personal github if you want to check it out:  [wcf-healthcheck-code](https://github.com/karlospn/healthchecks-wcf)

I'm going to create a class library so I can pack it and use it as a nuget.

- First of all I define the response object that the healthcheck is going to return.   
The healthcheck is going to return a _"HealthCheckResponse"_ that contains a descriptive message and the status of our service: _Healthy_, _Unhealthy_ or _Degraded_

```csharp
using System.Runtime.Serialization;

namespace HealthChecks.Wcf
{
    [DataContract]
    public class HealthCheckResponse
    {

        [DataMember]
        public string Status { get; set; }

        [DataMember]
        public string Message { get; set; }

    }
}
```

- Next step is to define the contract.   
The endpoint created is going to be located under the _/check_ uri

```csharp
using System.ServiceModel;
using System.ServiceModel.Web;

namespace HealthChecks.Wcf
{
    [ServiceContract(Name ="health")]
    public interface IHealthCheckService
    {
        [OperationContract]
        [WebInvoke(
            Method = "GET", 
            UriTemplate = "/check", 
            BodyStyle = WebMessageBodyStyle.Bare,
            RequestFormat = WebMessageFormat.Json, 
            ResponseFormat = WebMessageFormat.Json)]
        HealthCheckResponse CheckHealth();
    }
}

```

- Next step is to implement the _CheckHealth_ method that I defined in the _IHealthCheckService_ interface.   
  
Notice that I  created an abstract class that defines an abstract method: _ExecuteHealthCheck()_ , that's because the _ExecuteHealthcheck_ method needs to be implemented by the client app and it will contain the healthcheck logic that the client want to check.

The _HealthCheckBase_ is going to fetch the result of the _ExecuteHealthCheck_ method, map from the method response to the response object and return a result.   
In case the healthcheck fails it will return a WebFaultException with a 500 status code, and if the healthcheck succeeds with a healthy or degraded status it will return a 200 OK.


```csharp
using System.Net;
using System.ServiceModel.Web;
using HealthChecks.Wcf.Enums;

namespace HealthChecks.Wcf
{
    public abstract class HealthCheckBase : IHealthCheckService
    {
        public HealthCheckResponse CheckHealth()
        {
            var result = EvaluateHealthCheck();
            return MappedResultToResponse(result);

        }

        private HealthCheckResult EvaluateHealthCheck()
        {
            var healthCheckResult = ExecuteHealthCheck();

            if (healthCheckResult == null || 
                string.IsNullOrEmpty(healthCheckResult.Message))
            {
                throw new WebFaultException<string>(
                    "Something when wrong on your healthcheck. Check your implementation", 
                    HttpStatusCode.InternalServerError);

            }
            if (healthCheckResult.Status == HealthStatus.Unhealthy)
            {
                throw new WebFaultException<HealthCheckResult>(
                    healthCheckResult, 
                    HttpStatusCode.InternalServerError);
            }

            return healthCheckResult;
        }

        private HealthCheckResponse MappedResultToResponse(HealthCheckResult result)
        {
            return new HealthCheckResponse
            {
                Status = result.Status.ToString(),
                Message = result.Message
            };
        }

        protected abstract HealthCheckResult ExecuteHealthCheck();
    }

}


using HealthChecks.Wcf.Enums;

namespace HealthChecks.Wcf
{
    public class HealthCheckResult
    {

        public HealthStatus Status  {get;set;}

        public string Message { get; set; }

    }
}

namespace HealthChecks.Wcf.Enums
{
    public enum HealthStatus
    {
        Healthy,
        Unhealthy,
        Degraded
    }
}

```

- Last step is to wire everything up.   
When the serviceHost is opening the connection we use it to create a ServiceEndpoint with a WebHttpBinding.   
Basically that will allow us to create a new REST endpoint in our WCF application.


```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;
using System.ServiceModel;
using System.ServiceModel.Description;

namespace HealthChecks.Wcf
{
    public static class HealthCheckExtensions
    {
        public static Action<ServiceHostBase> AddHealthCheckEndpoint()
        {
            return serviceHost =>
            {
                serviceHost.Opening += (snd, args) =>
                     {
                         GenerateHealthEndpoint(serviceHost);
                     };
            };
        }

        private static void GenerateHealthEndpoint(ServiceHostBase serviceHost)
        {
            var implementedContracts = serviceHost
                .GetType()
                .GetProperty("ImplementedContracts", BindingFlags.Instance | BindingFlags.NonPublic | BindingFlags.GetField);

            if (implementedContracts != null)
            {
                var contractDic = (Dictionary<string, ContractDescription>)implementedContracts.GetValue(serviceHost, null);
                foreach (var contract in contractDic.Values)
                {
                    if (serviceHost.Description.Endpoints.Any())
                    {
                        continue;
                    }

                    if (contract.ConfigurationName == typeof(IHealthCheckService).FullName)
                    {
                        var serviceEndpoint = new ServiceEndpoint(contract, 
                            new WebHttpBinding(), 
                            new EndpointAddress(serviceHost.BaseAddresses[0]));

                        serviceEndpoint.Behaviors.Add(new WebHttpBehavior());
                        serviceHost.AddServiceEndpoint(serviceEndpoint);
                    }
                }
            }

        }
    }
}

```

And that's it, we have built a library capable of adding a healthcheck REST endpoint into our WCF application in less than 200 lines of code.

# Test the library

Begin by creating a new WCF from scratch and adding a new csharp class that inherits from _HealthCheckBase_.   
The class needs to implement the _ExecuteHealthCheck_ method. The method will contain our health checking code. In my case I'm returning directly an OK result.

```csharp
using HealthChecks.Wcf.Enums;

namespace HealthChecks.Wcf.IntegrationTest
{
    public class MyHealthCheckService : HealthCheckBase 
    {

        protected override HealthCheckResult ExecuteHealthCheck()
        {
            return new HealthCheckResult
            {
                Message = "Everything runs smoothly",
                Status = HealthStatus.Healthy
            };
        
        }

    }
}

```

Next we need to register everything on our IoC container. 
I'm using Autofac, so I need to follow the steps that Autofac dictates when you're registering the dependencies in a WCF.   
You can use the IoC container that best suits your necessities, just look it up at the documentation what steps you need to follow.


```csharp
  protected void Application_Start(object sender, EventArgs e)
        {
            var builder = new ContainerBuilder();
            builder.RegisterType<Service1>().As<IService1>();
            builder.RegisterType<MyHealthCheckService>();
            var container = builder.Build();

            AutofacHostFactory.Container = container;
            AutofacHostFactory.HostConfigurationAction = HealthCheckExtensions.AddHealthCheckEndpoint();
        }

```

We are creating a service without a .svc file, to do it with Autofac you need to add the following lines in the web.config.

If you want more information about how to add a svc-less service using Autofac read it from the offical documentation [here](https://autofaccn.readthedocs.io/en/latest/integration/wcf.html#svc-less-services)

As I said before if you're using another IoC container just search in the documentation how to create a svc-less service.

```xml
  <serviceHostingEnvironment aspNetCompatibilityEnabled="true" multipleSiteBindingsEnabled="true" >
      <serviceActivations>
       <add relativeAddress="health.svc" service="HealthChecks.Wcf.IntegrationTest.MyHealthCheckService, HealthChecks.Wcf.IntegrationTest" factory="Autofac.Integration.Wcf.AutofacServiceHostFactory, Autofac.Integration.Wcf"/>
      </serviceActivations>
    </serviceHostingEnvironment>
```

The REST endpoint is going to be located in the uri : _**/health.svc/check**_


Now run the application and test the REST endpoint:

```bash
curl -i http://localhost:39150/health.svc/check
HTTP/1.1 200 OK
Cache-Control: private
Content-Type: application/json; charset=utf-8
Server: Microsoft-IIS/10.0
Set-Cookie: ASP.NET_SessionId=04nkjaxw1h32o0jwybcpzgil; path=/; HttpOnly; SameSite=Lax
X-AspNet-Version: 4.0.30319
X-SourceFiles: =?UTF-8?B?RDpcTkVUIFByb2plY3RzXGhlYWx0aGNoZWNrcy13Y2ZcdGVzdFxBVEMuSGVhbHRoQ2hlY2tzLldjZi5JbnRlZ3JhdGlvblRlc3RcaGVhbHRoLnN2Y1xjaGVjaw==?=
X-Powered-By: ASP.NET
Date: Sat, 04 Jul 2020 20:39:39 GMT
Content-Length: 57

```

Now modify the healthcheck class to make it fail:

```csharp
using HealthChecks.Wcf.Enums;

namespace HealthChecks.Wcf.IntegrationTest
{
    public class MyHealthCheckService : HealthCheckBase 
    {

        protected override HealthCheckResult ExecuteHealthCheck()
        {
            return new HealthCheckResult
            {
                Message = "Everything is failing smoothly",
                Status = HealthStatus.Unhealthy
            };
        
        }

    }
}
```

And test it again:

```bash
HTTP/1.1 500 Internal Server Error
Cache-Control: private
Content-Type: application/json; charset=utf-8
Server: Microsoft-IIS/10.0
X-AspNet-Version: 4.0.30319
Set-Cookie: ASP.NET_SessionId=vktoksbu5nuzjovhhqryt1iq; path=/; HttpOnly; SameSite=Lax
X-SourceFiles: =?UTF-8?B?RDpcTkVUIFByb2plY3RzXGhlYWx0aGNoZWNrcy13Y2ZcdGVzdFxBVEMuSGVhbHRoQ2hlY2tzLldjZi5JbnRlZ3JhdGlvblRlc3RcaGVhbHRoLnN2Y1xjaGVjaw==?=
X-Powered-By: ASP.NET
Date: Sat, 04 Jul 2020 20:41:56 GMT
Content-Length: 46

```

# Azure

I said that adding a healthcheck into your WCF could be a first step towards a lift-and-shift migration into Azure, but why I said that?

Well that's because if you're going to move an existing WCF into Azure hands down the best option is to use Azure App Services. And if you have implemented a healthcheck endpoint in your WCF you can totally put it to good use.

Azure Web Apps contains a Health Check feature and is used to prevent unhealthy instance(s) from serving requests, thus improving availability. The feature will ping the specified health check path on all instances of your webapp every 2 minutes.  
Just open the Resource Explorer find and element named "healthCheckPath" and set its value to the health path of your Wcf application: **_/health.svc/check_**

![healthcheck](/img/healthcheck-web-app.png)

If you want to know more information about the healthcheck feature in Azure Web App Services you can read it from the Project Kudu github page: [here](https://github.com/projectkudu/kudu/wiki/Health-Check-(Preview))






