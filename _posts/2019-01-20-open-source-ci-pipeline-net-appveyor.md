---
layout: post
title: Open Source CI pipeline for .NET with AppVeyor
tags: ci/cd
---

Plenty of tools offer **free** licenses for open source projects. Often they work quite nicely with eachother. In this post I'll show how GitHub, AppVeyor, MyGet and CodeCov can work together in a complete CI/CD pipeline.

This is the setup I use for [Firestorm](https://github.com/connellw/Firestorm) - a solution with 50+ projects, some multi-targetting .NET Standard and .NET Framework, some integration tests communicating with SQL Server.

The majority of this I learned from [Andrew Lock](https://andrewlock.net/publishing-your-first-nuget-package-with-appveyor-and-myget/) and [Jimmy Bogard](https://lostechies.com/jimmybogard/2016/05/24/my-oss-cicd-pipeline/) and recommend reading their posts for full details. As with their posts, we're going to start assuming your .NET source is already hosted in GitHub.

# Versioning

## No pushing to master

We should setup a **Branch protection rule** to prevent us pushing commits straight to the master branch. This forces us to create a new branch and create a Pull Request for new changes.

In your GitHub project, go to **Settings -> Branches** and add a new rule for the `master` branch.

## Version number

All the packages should share the same version number, author, and a few other properties. We're going to use [Directory.build.props](https://github.com/jbogard/MediatR/blob/master/Directory.Build.props) to define these shared properties in the root directory. Here an example based on [the one I use](https://github.com/connellw/Firestorm/blob/master/Directory.Build.props).

```xml
<Project>
  <PropertyGroup>
    <VersionPrefix>0.9.4</VersionPrefix>
    <VersionSuffix>alpha-00001</VersionSuffix>
    <Authors>Connell Watkins</Authors>
    <Copyright>Copyright © Connell Watkins 2017</Copyright>
    <PackageProjectUrl>https://github.com/connellw/Firestorm</PackageProjectUrl>
  </PropertyGroup>
</Project>
```

Note that we have defined the `VersionSuffix` separately. This is because our build script will replace the suffix in most circumstances.

- If project is built locally, the suffix will be the checked-out branch name followed by the last commit hash.
- When built from a PR, it'll be the source branch name followed by AppVeyor's auto-incrementing build number.
- When merging a PR into master, it'll be whatever is defined above.
- If that is blank, the version won't include any pre-release suffix.

# Build

It's almost too easy to install AppVeyor as a GitHub App. Sign up at https://www.appveyor.com/ using your GitHub OAuth login. From the *'New Project'* screen you are automatically taken to, select *GitHub*, select your repository and complete the authorisation process. This configures a WebHook so AppVeyor is notified when commits are pushed to your repository.

## appveyor.yml

AppVeyor needs to know how to build your source. The best way to do this is to add an [`appveyor.yml`](https://www.appveyor.com/docs/appveyor-yml/) file to the root of your repository. I'll take you through the first bits of [the one I use](https://github.com/connellw/Firestorm/blob/master/appveyor.yml).

```yml
version: '{build}'
```
This uses the auto-incrementing build number as the version *at the start* of the build. We replace this later.

```yml
branches:
  only:
  - master
```
Only build changes to the master branch. This includes opening a Pull Request to merge into the master branch.

```yml
build_script:
- ps: .\build.ps1
```
Use the given powershell script to build the project.

## build.ps1

[My build script](https://github.com/connellw/Firestorm/blob/master/build.ps1) is largely taken from [Jimmy Bogard's](https://github.com/jbogard/MediatR/blob/master/Build.ps1) mentioned in the articles linked above.

We use the same `Exec` function to throw exceptions if an exit code is not `0` and use a similar method to calculate the full version number.

However, because we are only building the `master` branch, the script checks the `APPVEYOR_PULL_REQUEST_HEAD_REPO_BRANCH` environment variable first. Otherwise for PR builds, the `VersionSuffix` would be for example `master-00001`.

We've also added the `Update-AppveyorBuild` cmdlet to replace the version number displayed on the AppVeyor build.

```powershell
if (Test-Path env:APPVEYOR) {
    $props = [xml](Get-Content Directory.Build.props)
    $prefix = $props.Project.PropertyGroup.VersionPrefix
    
    $avSuffix = @{ $true = $($suffix); $false = $props.Project.PropertyGroup.VersionSuffix }[$suffix -ne ""]
    $full = @{ $true = "$($prefix)-$($avSuffix)"; $false = $($prefix) }[-not ([string]::IsNullOrEmpty($avSuffix))]
    
    echo "Build: Full version is $full"
    Update-AppveyorBuild -Version $full
}
```

# Test

Test coverage reports are made easy with [coverlet](https://github.com/tonerdo/coverlet) and [CodeCov](https://codecov.io).

## appveyor.yml

We're going to run our tests within our build script, so tell appveyor to turn the tests off.

```yml
test: off
```

AppVeyor still recognises the test output and displays results in the **Tests** tab.

## build.ps1

There are many test projects, so we iterate through them and run them individually.

We apply a logical order for these test runs:
  - **Unit tests** first. These execute quickly and give more specific error messages. If these fail, we stop the build as there is no point executing the other tests.
  - **Integration tests** next. These are slower as they spin-up real HTTP servers or communicate with real SQL servers.
  - **Functional tests** last. These can take a lot longer to run.

We need to install the coverlet Global Tool then pass command args parameters into our `dotnet test` calls. This generate a `coverage.json` file that we merge into each test run. However, the last test must output in the `opencover.xml` format so we can push to CodeCov.

```powershell
exec { & dotnet tool install --global coverlet.console }

$testDirs  = @(Get-ChildItem -Path tests -Include "*.Tests" -Directory -Recurse)
$testDirs += @(Get-ChildItem -Path tests -Include "*.IntegrationTests" -Directory -Recurse)
$testDirs += @(Get-ChildItem -Path tests -Include "*FunctionalTests" -Directory -Recurse)

$i = 0
ForEach ($folder in $testDirs) { 
    echo "Testing $folder"

    $i++
    $format = @{ $true = "/p:CoverletOutputFormat=opencover"; $false = ""}[$i -eq $testDirs.Length ]

    exec { & dotnet test $folder.FullName -c Release --no-build --no-restore /p:CollectCoverage=true /p:CoverletOutput=$root\coverage /p:MergeWith=$root\coverage.json /p:Include="[*]Firestorm.*" /p:Exclude="[*]Firestorm.Testing.*" $format }
}
```

Next we want to upload our coverage report to CodeCov. Sign up using GitHub OAuth and you don't even need to use any API keys. Just install CodeCov via chocolatey and run the tool.

```powershell
choco install codecov --no-progress
exec { & codecov -f "$root\coverage.opencover.xml" }
```

Coverage reports will now appear in CodeCov.

# Pack

There are many projects, so we use a shared `artifacts` directory in the project root.

## appveyor.yml

```yml
artifacts:
- path: .\artifacts\**\*.nupkg
  name: NuGet
```
This tells AppVeyor that our NuGet packages will be in this directory.

## build.ps1

At the top of our build script, we define the full artifacts path and clear its current contents if there are any.

```powershell
$artifactsPath = (Get-Item -Path ".\").FullName + "\artifacts"
if(Test-Path $artifactsPath) { Remove-Item $artifactsPath -Force -Recurse }
```

After building and testing, we pack the full solution, specifying the full output directory.

```powershell
exec { & dotnet pack -c Release -o $artifactsPath --include-symbols --no-build $versionSuffix }
```

Now the resulting `.nupkg` files will be picked up by AppVeyor and appear in the **Artifacts** tab.

# Deploy

Now we just need to deploy our packages. Following the posts linked above, we will:

- Deploy to our MyGet feed when the PR is merged to master.
- Publish to nuget.org when a tag is created for the master branch.

## appveyor.yml

```yml
deploy:
- provider: NuGet
  server: https://www.myget.org/F/firestorm/api/v2/package
  api_key:
    secure: asdfasdfasdfasdfasdfasdfasdf
  skip_symbols: true
  on:
    branch: master
- provider: NuGet
  name: production
  api_key:
    secure: fdsafdsafdsafdsafdsafdsafdsa
  on:
    branch: master
    appveyor_repo_tag: true
```

The secure variables are our **API keys,** found in MyGet and NuGet's settings pages.

You don't want to paste the keys directly. Encrypt them using [AppVeyor's Encrypt page](https://ci.appveyor.com/tools/encrypt) and paste them in as **secure variables**.

AppVeyor does not give PR builds access to secure variables in public projects. The artifacts are still created in AppVeyor, but not deployed to MyGet automatically. If desired, you can manually Deploy from AppVeyor after a successful PR build.

# Badges

As a final touch, add these to the top of your `README.md` file.

[![Build status](https://ci.appveyor.com/api/projects/status/1bo4yw50e7m7m2cm?svg=true)](https://ci.appveyor.com/project/connellw/firestorm) [![codecov](https://codecov.io/gh/connellw/Firestorm/branch/master/graph/badge.svg)](https://codecov.io/gh/connellw/Firestorm) [![MyGet](https://img.shields.io/myget/firestorm/vpre/Firestorm.Endpoints.svg?label=myget)](https://myget.org/gallery/firestorm) [![NuGet](https://img.shields.io/nuget/v/Firestorm.Endpoints.svg)](https://www.nuget.org/packages?q=firestorm)

AppVeyor provides *sample markdown code* you can copy under the project's **Settings** tab.

```md
[![Build status](https://ci.appveyor.com/api/projects/status/1bo4yw50e7m7m2cm?svg=true)](https://ci.appveyor.com/project/connellw/firestorm)
```

Similarly, CodeCov provides this under the project's **Settings** tab.

```md
[![codecov](https://codecov.io/gh/connellw/Firestorm/branch/master/graph/badge.svg)](https://codecov.io/gh/connellw/Firestorm)
```

For the other badges I use [shields.io](https://shields.io/). You can get a badge for pretty much anything here. And even if you can't, you can make them.

For the NuGet badge, its simple. `https://img.shields.io/nuget/v/Firestorm.svg`.

The link I've just set to search NuGet.org, rather than linking to one of the package pages.

```md
[![NuGet](https://img.shields.io/nuget/v/Firestorm.svg)](https://www.nuget.org/packages?q=firestorm)
```

For MyGet, you have to add the feed name to the URL too. `https://img.shields.io/myget/firestorm/vpre/Firestorm.svg`. I've also specified `vpre` to display the latest pre-release version, and I've used the querystring to change the text to 'myget'.

For the link, you can use the public gallery page. You have to enable this manually in your MyGet feed settings.

```md
[![MyGet](https://img.shields.io/myget/firestorm/v/Firestorm.svg?label=myget)](https://myget.org/gallery/firestorm)
```