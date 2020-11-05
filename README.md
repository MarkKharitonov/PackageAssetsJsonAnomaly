# PackageAssetsJsonAnomaly
A minimal reproduction of a potential msbuild bug related to inconsistent generation of project.assets.json for non SDK projects using PackageReference.

## Introduction
We have a big Monolithic application consisting of a couple of 100+ projects strong solutions. Naturally, it is all .Net Framework, not .Net Core. Migrating to .Net Core is not an option, because of the enormous effort associated with it. We are working to break the Monolith, but it is a huge effort and takes a lot of time.

Up until recently we used the **packages.config** approach to managing NuGet packages. Recently we migrated to **PackageReference** approach (An ordeal in its own right, since there is no automation for multiple projects - https://stackoverflow.com/a/64200862/80002). And all hell broke loose - https://stackoverflow.com/questions/64576860/why-does-console-build-generate-radically-different-project-assets-json-than-tha and https://stackoverflow.com/questions/64689231/eliminating-assembly-binding-redirects-when-targeting-the-net-framework. My frenetic investigation into possible solutions makes me discover all kinds of things. This repository is the result of one of these discoveries. More may follow.

## The Anomaly

The project.assets.json file has several different sections. I will concentrate on the `libraries` section, which simply speaking lists all the project and package dependencies as embodied by the respective `ProjectReference` and `PackageReference` constructs in the respective csproj file. It also lists the transitive dependencies of these project and package dependencies, but this is immaterial for my issue. Also for the sake of this issue we are only interested in the project dependencies.

Let us observe a simple Powershell session, which does the following:
 1. Runs msbuild restore without building
 1. Parses the project.assets.json generated for one of the projects into a Powershel object
 1. Shows the names of the `libraries` project entries (as opposed to package entries, which are of no interest here)

Here we go:
```Powershell
C:\work\PackageAssetsJsonAnomaly [master]> git clean -qdfx
C:\work\PackageAssetsJsonAnomaly [master]> Test-Path .\DataSvc\obj
False
C:\work\PackageAssetsJsonAnomaly [master]> msbuild /t:restore /v:m
Microsoft (R) Build Engine version 16.7.0+b89cb5fde for .NET Framework
Copyright (C) Microsoft Corporation. All rights reserved.

  Determining projects to restore...
  Restored C:\work\PackageAssetsJsonAnomaly\DataSvc\DataSvc.csproj (in 282 ms).
  Restored C:\work\PackageAssetsJsonAnomaly\Common\Common.csproj (in 282 ms).
C:\work\PackageAssetsJsonAnomaly [master]> $ProjectAssets = cat .\DataSvc\obj\project.assets.json | ConvertFrom-Json
C:\work\PackageAssetsJsonAnomaly [master]> $ProjectAssets.libraries.PSObject.Properties |? { $_.Value.type -eq 'project' } |% { $_.Name }
RateEngine/1.0.0
xyz.Common/1.0.0
C:\work\PackageAssetsJsonAnomaly [master]>
```
Now the question is - what are these names? I can think of two options:
 1. Project name, i.e. the value of the `MSBuildProjectName` build property, which defaults to the name of the respective csproj file without the extension.
 1. Assembly name, i.e. the value of the `AssemblyName` build property.
 
Logic mandates that it should be either one of these. What happens in this example, is that one project dependency is named after the project name while another - after the assembly name. Please, observe:
```Powershell
C:\work\PackageAssetsJsonAnomaly [master]> cat .\RateEngine\RateEngine.csproj,.\Common\Common.csproj | sls AssemblyName                                                        
    <AssemblyName>RateEngine2</AssemblyName>
    <AssemblyName>xyz.Common</AssemblyName>

C:\work\PackageAssetsJsonAnomaly [master]> 
```
As one can see, the project reference `RateEngine/1.0.0` is named after the project name, but `xyz.Common/1.0.0` - after the assembly name.

Inspecting the csproj files reveals that they are very ordinary ones, so what is the root cause of this inconsistency?
