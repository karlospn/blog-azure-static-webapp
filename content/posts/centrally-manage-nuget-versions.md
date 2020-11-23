---
title: "How to centrally manage NuGet package versions within your solution"
date: 2020-09-02T10:01:20+02:00
tags: ["csharp", "dotnet", ".NET", "nuget"]
draft: false
---

A pretty common scenario in complex .NET applications is having them splitted in multiple csharp projects inside a single solution, even with the proliferation of microservices is pretty common seeing the client and the business logic decoupled in multiple projects inside a single solution.   
And that behaviour is even more common if you're using some kind of software architecture like _"Domain Driven Design"_, "_Hexagonal Architecture"_ or _"Clean Architecture"_.   

A common problem that usually appears when the number of projects begins to pile up inside a single solution is mantaining consistency between nuget versions, and quite some times you end up with every project using a different version of the same package.   
And that problem is even worse if there are multiple applications inside a single solution and those applications are sharing projects between them, at that point it becomes a nightmare trying to maintain a consistent package version across all projects because when you try to update a package you might be affecting multiple applications without even knowing.

That's why in most cases I like to manage the package versions in a central point inside my solution and if the solution contains multiple applications then **every application uses the same nuget version** and every application is affected when I modify the package version but at least I don't have to navigate between the dependency mesh of multiples projects being referenced between them.   
Also in case that I have multiple applications inside a single solution and I want to update a package version of only one of them I tend to move the application that I want to modify and place it in a new solution just by itself.

Anyways, in these post I want to show you some of the different options available when you want to centrally manage nuget versions within a solution.


# Option 1 : Using the Microsoft.Build.CentralPackageVersions SDK project

_**Source code**_: https://github.com/microsoft/MSBuildSdks/tree/master/src/CentralPackageVersions


**Microsoft.Build.CentralPackageVersions** it's a project SDK built by Microsoft themself.   
MSBuild 15.0 introduced the concept of the _"project SDK"_, which simplifies using software development kits that require properties and targets to be imported. 

> - If you want to know more about project SDK you can read  more about it here: https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-use-project-sdk?view=vs-2019   
> 
> - If you don't know what it's a project SDK you can read more about it here: https://docs.microsoft.com/en-us/dotnet/core/project-sdk/overview


To get started, you need to create an MSBuild project at the root of your solution named _"Packages.props"_ that declares the version of the packages you're going to use.

```csharp

<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">  
  <ItemGroup Label="Microsoft Nugets">
    <PackageReference Update="Microsoft.Extensions.Configuration" Version="2.1.1" />
    <PackageReference Update="Microsoft.Extensions.DependencyInjection.Abstractions" Version="2.1.1" />
    <PackageReference Update="Microsoft.Extensions.Http" Version="2.1.1" />
    <PackageReference Update="Microsoft.Extensions.Options.ConfigurationExtensions" Version="2.1.1" />
    <PackageReference Update="Microsoft.Extensions.Http.Polly" Version="2.1.1" />
  </ItemGroup>
</Project>

```

In each project (csproj) you must add the Sdk and also you must add the packages you're going to use but without specifying the versions.

```csharp
<Project Sdk="Microsoft.NET.Sdk">
  <Sdk Name="Microsoft.Build.CentralPackageVersions" Version="2.0.1" />
  
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Configuration" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
    <PackageReference Include="Microsoft.Extensions.Http" />
    <PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" />
    <PackageReference Include="Microsoft.Extensions.Http.Polly" />
  </ItemGroup>
</Project>

```

If someone attempts to add a version inside a csproj, he will get a build error:  

```csharp
The package reference 'Microsoft.Extensions.Configuration' should not specify a version.  Please specify the version in 'C:\git\mysolution\Packages.props' or set VersionOverride to override the centrally defined version.

```

I have very few complaints about that solution and that's what I'm using nowadays in most of my projects.      
The only "minor" issue I have found is when trying to add or update a nuget package using the dotnet CLI or the Visual Studio Package Manager.   
Visual Studio Package Manager is capable of showing me which nuget version is using every csproj but when I try to update or add a new package using the Package Manager it installs the nuget into the target csproj instead of the props file.   

The same behaviour happens if you try to add a nuget using the dotnet CLI.   

If you want to use the **CentralPackageVersions SDK** and also use the **Visual Studio Package Manager** or **dotnet CLI** for adding or updating the package versions then after you install the package you will have to remove the version from the csproj and place it into the .props file **manually**.      


# Option 2 - Using the Directory.Build.targets MSBuild file
 
That option relies on using the **Directory.Build.Targets** file to define the nuget package versions in a centralized way.

MSBuild projects that use the standard build process (importing **Microsoft.Common.props** and **Microsoft.Common.targets**) have several extensibility hooks that you can use to customize your build process.   

MSBuild 15 introduced the possibility to add new properties to every project in one step by defining it in files called **Directory.Build.props** and **Directory.Build.targets**  and place those files in the solution root folder.   
When MSBuild runs, **Microsoft.Common.props** searches your directory structure for the **Directory.Build.props** file and **Microsoft.Common.targets** searches your directory structure for the **Directory.Build.targets**. If it finds one of them, it imports the properties.   

> If you want to know more about **Directory.Build.Targets** and **Directory.Build.props**, read here: https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2017


To get started, you need to create a file named **Directory.Build.targets** at the root of your solution that declares which packages and which version you're going to use.

```csharp

<Project>
  <ItemGroup>
    <PackageReference Update="Microsoft.Extensions.Configuration" Version="2.1.1" />
    <PackageReference Update="Microsoft.Extensions.DependencyInjection.Abstractions" Version="2.1.1"/>
    <PackageReference Update="Microsoft.Extensions.Http" Version="2.1.1"/>
    <PackageReference Update="Microsoft.Extensions.Options.ConfigurationExtensions" Version="2.1.1"/>
    <PackageReference Update="Microsoft.Extensions.Http.Polly" Version="2.1.1"/>
  </ItemGroup>
</Project>

```

In each project (csproj) you must add the packages you're going to use but without specifying the versions.

```csharp
<Project Sdk="Microsoft.NET.Sdk">
  
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Configuration" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
    <PackageReference Include="Microsoft.Extensions.Http" />
    <PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" />
    <PackageReference Include="Microsoft.Extensions.Http.Polly" />
  </ItemGroup>
</Project>

```

As you can see both option 1 and option 2 seems almost identical, even shares the same issue  when you try to use the .targets file in conjunction with the Visual Studio Package Manager or dotnet CLI.

Nonetheless I prefer using the CentralPackageVersions SDK project because you have nuget validations out-of-the-box. If someone tries to add the version of a package in the csproj file instead of the .targets file the compilation won't throw an error.


# Option 3 - Using the ManagePackageVersionsCentrally project attribute and the Directory.Packages.props file

That's the most recent iteration about how to manage centrally nuget package versions.

It works almost exactly the same way as option 1 and option 2. You have to create a **Directory.Packages.props**  file at the root of the solution where you define the packages versions.


```csharp

<Project>
  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="2.1.1" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="2.1.1"/>
    <PackageReference Include="Microsoft.Extensions.Http" Version="2.1.1"/>
    <PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="2.1.1"/>
    <PackageReference Include="Microsoft.Extensions.Http.Polly" Version="2.1.1"/>
  </ItemGroup>
</Project>

```

In each project (csproj) you must add the packages you're going to use but without specifying the nuget versions. Also you need to add the attribute **_"ManagePackageVersionsCentrally"_**

```csharp
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Configuration" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" />
    <PackageReference Include="Microsoft.Extensions.Http" />
    <PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" />
    <PackageReference Include="Microsoft.Extensions.Http.Polly" />
  </ItemGroup>
</Project>

```

When I tried that feature in my machine at first it didn't work, that's because the feature seems to be still **in preview** and you need to use **.NET Core SDK 3.1.300** and **VS 2019.6** (https://github.com/NuGet/Home/issues/6764)

Reading through the documentation (https://github.com/NuGet/Home/wiki/Centrally-managing-NuGet-package-versions) I can see that they are planning to **add support with Visual Studio Package Manager and also dotnet CLI**, but as of today it still doesn't work.


# Option 4 - Using Visual Studio Package Manager

That's the most simple option. If you're using Visual Studio you're good to go.
You don't need to do any extra step. 

> Just open your solution in Visual Studio > Go To Tools >  Nuget Package Manager > Manage NuGet Packages for Solution

You can manage all the nuget versions inside your solution from there.   
To me it's the most cumbersome option and there is not a centralized file that contains the package versions used, but nonetheless is a viable, out-of-the-box option if you want to manage the nuget versions in _"one place"_.


# Conclusion 

As you can see option 1, 2 and 3 looked almost identical.   
Right now if I have to choose one I prefer to use the **CentralPackageVersions SDK** over the .targets file,  but it seems the way to go in a near future will be using the **ManagePackageVersionsCentrally** attribute and the **Directory.Packages.props** file, so just keep an eye for when the feature gets out of preview mode.