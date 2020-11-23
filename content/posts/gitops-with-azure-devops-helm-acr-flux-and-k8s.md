---
title: "A practical example of GitOps using Azure DevOps, Azure Container Registry, Helm, Flux and Kubernetes"
date: 2020-07-29T22:10:21+02:00
tags: ["azure", "acr", "kubernetes", "gitops", "helm"]
draft: false
---


GitOps is a term introduced by WeaveWorks a few years ago (https://www.weave.works/blog/gitops-operations-by-pull-request)

GitOps is a way of implementing Continuous Deployment. The core idea of GitOps is having a Git repository that always contains declarative descriptions of the infrastructure currently desired in the production environment and an automated process to make the production environment match the described state in the repository.   
If you want to deploy a new application or update an existing one, you only need to update the repository - the automated process handles everything else.

In these post I want to build a practical example of GitOps, it's not going to be a production-ready example but nonetheless it's going to be an interesting proof of concept, so let's try it.

## Components
---

I'm going to use the following components:

- **Azure DevOps** as my VCS (version control system) 
- **Azure Container Registry** as my container registry and also as my helm chart repository.
- **Kubernetes** on-premise for hosting the applications.
- **Helm** to manage the application kubernetes files.
- **Flux** and **Helm-Operator** to deploy the applications in an automatic way into the cluster.
  
That's a diagram of how the components are going to interact.

![diagram](/img/gitops-ci.png)


Let me explain a little bit about how the workflow works:

1- The **developer** creates a new application an pushes the source code into an Azure DevOps repository. 

2- The **code pushed** triggers an Azure Pipeline that creates a container image and a Helm chart and pushes everything into an existing Azure Container Registry.  

3- The **DevOps Team** has a **centralized repository** where they maintain a declarative description of all the components deployed into the Kubernetes cluster.    

4- The **DevOps Team** pushes a new file into the **centralized repository**, that file contains the description of the new application that the **developer** has created on **step 1**.  
**Flux** is keeping watch for changes in the centralized repository and when it sees a new file, it picks it up and deploys the new application using the Helm chart that we created in **step 2.**

_**In my example I'm using 2 different actors: a developer and a devops team, but there are tons of viable combinations possible, for example:_    
_You might not need two actors, it could be the same actor that does all the steps by himself. Or maybe you want even more actors because somebody needs to test and validate the image after being pushed into ACR._

Anyways, let's begin building the workflow.

## Step 0 - Initial setup
---

1- On my Azure DevOps account I'm just going to create:   

- An Azure Service connection with permissions to push and pulll images from ACR. 
- An Azure DevOps Git repository named _**"AppA"**_: 
  - That's where the **application source code** is going to be. 
- An Azure DevOps Git repository named _**"Manifests"**_: 
  - That's the **centralized repository** that Flux is going to monitor.

2 - On my Azure subscription I'm going to create an Azure Container Registry called "_**acrgitopsdev**_"


## Step 1 - Creating an application
---

It's going to be a .NET Core 3.1 WebAPI.  The application per se it's not important, and that's why I'm not going to explain how to build it. But there are a couple of things I want to remark: 
- The app needs to have a _DockerFile_, because I'm using Kubernetes and Kubernetes likes containers...
- The app also needs to have a _/chart_ folder. That folder is going to contain all the Kubernetes definition files that the application needs. 

An example of the application structure would be something like this:

```bash
.
│   .gitignore
│   azure-pipelines.yml
│   README.md
│
└───AppA.WebApi
    │   .dockerignore
    │   AppA.WebApi.csproj
    │   AppA.WebApi.sln
    │   appsettings.Development.json
    │   appsettings.json
    │   Dockerfile
    │   Dockerfile.develop
    │   Program.cs
    │   Startup.cs
    │   WeatherForecast.cs
    │
    ├───chart
    │   │   .helmignore
    │   │   Chart.yaml
    │   │   values.yaml
    │   │
    │   └───templates
    │           deployment.yaml
    │           ingress.yaml
    │           secrets.yaml
    │           service.yaml
    │           _helpers.tpl
    │
    ├───Controllers
    │       WeatherForecastController.cs
    │
    └───Properties
            launchSettings.json

```

As I pointed out the app contains a Dockerfile and also a _/chart_ folder.

After seeing that you could argue that maybe I should not place the Kubernetes files / Chart files within the application code and maybe I should place it directly in the centralized repository, and that's totally legit, but I prefer to have the application code and the application kubernetes definitions all in the same place.


## Step 2 - Building an Azure Pipeline
---

We're going to build an Azure Pipeline that is going to do those 4 steps:

- Build the docker image
- Push the docker image into an existing ACR
- Package the helm chart 
- Push the helm chart into an existing ACR


That's the pipeline:

```YAML
trigger:
- master

resources:
- repo: self

variables:
  HELM_EXPERIMENTAL_OCI: 1
  registryName: 'acrgitopsdev'
  imageName: 'appawebapi'
  BuildId: '$(Build.BuildId)'
  tag: '$(Build.BuildId)'

steps:
- task: AzureCLI@2
  inputs:
    azureSubscription: 'azure-dev-subscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: 'az acr login --name $(registryName)'
    
- task: AzureCLI@2
  inputs:
    azureSubscription: 'azure-dev-subscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: 'az acr build --image $(imageName):$(BuildId) --registry $(registryName) --file Dockerfile .'
    useGlobalConfig: true
    workingDirectory: '$(Build.SourcesDirectory)/AppA.WebApi'

- task: HelmInstaller@1
  inputs:
    helmVersionToInstall: 'latest'

- task: replacetokens@3
  inputs:
    targetFiles: '**/*.yaml'
    encoding: 'auto'
    writeBOM: true
    actionOnMissing: 'warn'
    keepToken: false
    tokenPrefix: '#{'
    tokenSuffix: '}#'
    useLegacyPattern: false
    enableTelemetry: true
       
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |
      helm package --destination $(Build.ArtifactStagingDirectory)/ --version $(BuildId).0.0 ./chart/ 
    workingDirectory: '$(Build.SourcesDirectory)/AppA.WebApi'

- task: AzureCLI@2
  inputs:
    azureSubscription: 'azure-dev-subscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: 'az acr helm push -n $(registryName) $(Build.ArtifactStagingDirectory)/*.tgz'
    useGlobalConfig: true

```

Helm 3 needs the environment variable "HELM_EXPERIMENTAL_OCI: 1" defined or it won't work, so just put it there...

I'm using the Azure Pipeline **BuildId** to tag the docker image and also to set the Helm Chart version.   
That's a simple Helm versioning strategy, using a 1-1 versioning just keeps the chart version in sync with the application. That approach makes version bumping very easy (you bump everything up) and also allows you to quickly track what application version is deployed on your cluster (same as chart version).   
If you're interested in helm versioning strategies you can read it more here: https://codefresh.io/docs/docs/new-helm/helm-best-practices/

Let me detail a little bit what are the steps that I'm doing inside the pipeline:

- Login into ACR.
- Build and push the image into ACR. I'm doing both things with just a single command: _az acr build_. 
- Install Helm 3. 
- Replace tokens. 
- Create the Helm Chart from the _/chart_ folder using the Helm CLI.
- Push the Helm Chart into ACR using the AZ CLI.


> _Why I'm using the Replace Tokens task in my pipeline?_ (https://marketplace.visualstudio.com/items?itemName=qetza.replacetokens)   
> That's because in the Kubernetes Deployment YAML file I defined the image i'm using like this: 
```yaml
    containers:
    - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:#{tag}#" 
```
> As I previously said I'm using the Azure Pipelines **BuildId** as my image tag and I only know the value **after the pipeline has started**, so I'm using the replace tokens task to replace the _#{tag}#_ placeholder with the BuildId while the Azure Pipeline is running. 


## Step 3 - Installing Flux and the Helm Operator
---

What is Flux? Flux is a tool that automatically ensures that the state of your Kubernetes cluster matches the configuration you’ve supplied in Git. It uses an operator in the cluster to trigger deployments inside Kubernetes, which means that you don’t need a separate continuous delivery tool.  

What is the Helm Operator?  The Helm Operator is a Kubernetes operator, allowing one to declaratively manage Helm chart releases. Combined with Flux this can be utilized to automate helm charts releases in a GitOps manner.

You can read more about Flux and the Helm Operator here: https://docs.fluxcd.io/en/1.20.0/

That's how it works:

![flux-helm-diagram](/img/fluxcd-helm-operator-diagram.png)


I'm going to enumerate the steps required to install both Flux and the Helm Operator into the Kubernetes cluster.

1 - Create the Flux namespace:

```bash
kubectl create ns flux
```

2 - Add the Flux repository

```bash
helm repo add fluxcd https://charts.fluxcd.io
```

3 - Apply the Helm Operator CRD

```bash
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml
```

4 - Install Flux

```bash
helm upgrade -i flux fluxcd/flux --set git.url=git@ssh.dev.azure.com:v3/cpn/Provisioning/Manifests  --set registry.acr.enabled=true --namespace flux
```
- The **git.url** attribute needs to be the **Azure DevOps Repository URL** that we want Flux to monitor. Flux is going to connect to the Git repo through **SSH**.   

- The **registry.acr.enabled** attribute is needed because we are using ACR as our image registry.

5 - Giving Flux read and write access to the Azure DevOps repository

In the previous step we told Flux which Azure DevOps git repository should monitor. Now we need to give Flux access to the git repository.   
This is pretty straight-forward as Flux generates a SSH key at startup.   

To get the SSH public key generated by Flux you need to run:
```bash
kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
```
And in order to sync Flux with your Git repository you need to add the key as an SSH public key in your Azure DevOps account settings.

![Azure DevOps flux SSH key](/img/flux-ados-ssh.png)

6 - Install the Flux Operator

That step is the most problematic one, that's because there are some authentication problems between the Flux Operator and ACR.

I'm not the only one facing the same problem, so I have stolen the solution found here:
https://gaunacode.com/configuring-flux-to-use-helm-charts-from-azure-container-registry 

Basically when we install the Helm-Operator we have to provide the following attributes:

- The ACR name.
- The client id of a service principal that has at least AcrPull rights to your ACR.
- The client secret of that service principal.
- The url of your ACR Helm endpoint. It must end with a slash.

So the command is going to be something like:

```bash
helm upgrade -i helm-operator fluxcd/helm-operator --set git.ssh.secretName=flux-git-deploy --set helm.versions=v3 --namespace flux --set configureRepositories.enable=true --set configureRepositories.repositories[0].name=acrgitopsdev,configureRepositories.repositories[0].url=https://acrgitopsdev.azurecr.io/helm/v1/repo,configureRepositories.repositories[0].username=put-your-sp-id-here,configureRepositories.repositories[0].password=put-your-sp-password-here
```

> You can do all those steps manually or create an Azure Pipelines to orchestrate all those commands.   
To be honest, it's really easy to build the pipeline once you know all the commands you need to execute, so i'm not going to waste time doing it.


## Step 4 - Creating the application config file
---

The last step is to create the application config file, push it into the "**Manifests**" repository and see if Flux is capable of deploying our application in our Kubernetes cluster.


```yaml

apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: appa
spec:
  chart:
    repository: https://acrgitopsdev.azurecr.io/helm/v1/repo/
    name: appawebapi
    version: 158.0.0
  values:
    imagePullSecrets:
      - name: acr-gitdevops

```

The only thing you need to specify in the  file is the Chart repository URL, the Chart name and the Chart version to deploy.   
There are more options available when creating the config file, if you're interested you can read about it here: https://docs.fluxcd.io/projects/helm-operator/en/1.0.0-rc9/references/helmrelease-custom-resource.html

In my case I need to define a secret and that's because Kubernetes uses an image pull secret to store information needed to authenticate to the container registry. **If you're using AKS you don't need to do it.**   
If you want to read more about it: https://docs.microsoft.com/es-es/azure/container-registry/container-registry-auth-kubernetes

And that's it, we are done! Flux has deployed the app into our cluster.   
Let's test it a little bit. If we change the chart version and commit the file again:

```yaml

apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: appa
spec:
  chart:
    repository: https://acrgitopsdev.azurecr.io/helm/v1/repo/
    name: appawebapi
    version: 160.0.0
  values:
    imagePullSecrets:
      - name: acr-gitdevops

```

Flux automatically picks the change (with a 5 minutes delay) and replaces the application with the old version with the new one.

Next we create a new application and push a new config file into the centralized repository, like this:


```yaml

apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: appb
spec:
  chart:
    repository: https://acrgitopsdev.azurecr.io/helm/v1/repo/
    name: appbwebapi
    version: 164.0.0
  values:
    imagePullSecrets:
      - name: acr-gitdevops
```

Flux automatically picks the change (with a 5 minutes delay) and deploy the new application.   

So overall our process is working quite well, any new or modified config file we push inside our centralized repository it is going to be reflected in the Kubernetes cluster. 


## Conclusion
---

There are a lot of cool features that you can do with Flux that I haven't tested yet, for example you can configure Flux to update the config files by itself when you push a new version of the application into ACR.   

Keep in mind that not only the application files should go into that centralized repository. **Everything** you deploy inside the cluster should be put it there.     
For example, if you're deploying a fluentd daemonset or an nginx ingress controller just put the files in there at let Flux deploy it for you. Also if you don't want to use Helm Charts to deploy you can deploy plain kubernetes files and use tools like **Kustomize**.    
The main point I'm trying to make here is that if you want to use a GitOps approach the centralized repository needs be the **single source of truth** for all the code that goes into the cluster.

Also take that example with a grain of salt, it is nothing more than a proof of concept I wanted to build. It's not a production ready GitOps process.  


