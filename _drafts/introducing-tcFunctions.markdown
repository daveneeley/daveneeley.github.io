---
layout: post
title:  "Introducing tcFunctions"
date:   2015-10-30 19:42:05 -0600
categories: tcFunctions
---
In my [day job](https://www.spillman.com), I manage a busy [TeamCity](https://jetbrains.com/teamcity) server with anywhere from 6,000-10,000 builds at any given time. We ship an application server with several components built with several different technologies and build it on several different operating systems. Propogating build system changes by hand with these constraints can be very time consuming.

TeamCity came out with the first version of their REST API a few months before my team started converting builds from CruiseControl to TeamCity. We knew that managing all of these from the web page would not be efficient. So, as a C# developer, I took it upon myself to learn Powershell, build a TeamCity client, and migrate build systems all at the same time. And thus [tcFunctions](https://github.com/SpillmanTech/tcFunctions) was born.

tcFunctions leverages the power of objects and pipelines in Powershell to let you do batch operations on all your builds. Since the API supports it, you can also new up projects and build types from scratch.

##Build a project from scratch##
Let's start a new project in TeamCity that will run the unit tests for tcFunctions on commit. First things first, download tcFunctions, and import it into your current session. By default, tcFunctions connects to a local teamcity installation at http://localhost:8111. If your server is somewhere else, edit tcFunctions.properties before importing. Also, it will assume the user running the shell has an account on the TeamCity server, and will ask for the password. Again, edit tcFunctions.properties if this is not the case for you.

{% highlight PowerShell %}
git clone https://github.com/SpillmanTech/tcFunctions.git
cd tcFunctions
import-module .\tcFunctions.psm1
{% endhighlight %}

Make a new project
{% highlight PowerShell %}
$project = New-TeamCityProject -name tcFunctions
Add-TeamCityProject -project $project
{% endhighlight %}

The result:
{% highlight PowerShell %}
id              : Tcfunctions
name            : tcFunctions
parentProjectId : \_Root
href            : /httpAuth/app/rest/projects/id:Tcfunctions
webUrl          : http://localhost:8111/project.html?projectId=Tcfunctions
parentProject   : parentProject
buildTypes      : buildTypes
templates       : templates
parameters      : parameters
vcsRoots        : vcsRoots
projects        : projects

{% endhighlight %}

This is the same output that would be returned by:
{% highlight PowerShell %}
Get-Project -name tcFunctions
{% endhighlight %}

Now that we've got a project, we'll add a VCS root, a build type and a step to run the tests. To save you time, I won't show all the output.
{% highlight PowerShell %}
$project = Get-Project -name tcFunctions
$buildType = Add-BuildType -name "Run Tests" -project $project.id

$newVcsRoot = $project | New-VcsRoot -name Pester -url https://github.com/pester/Pester.git
$vcsRoot = Add-VcsRoot -newobj $newVcsRoot
$newVcsRootEntry = New-VcsRootEntry -vcsRootId $vcsRoot.id -checkoutRules ". => Pester"
$vcsRootEntry = Add-VcsRootEntry -btid $buildType.id -newobj $newVcsRootEntry

$newVcsRoot = $project | New-VcsRoot -name tcFunctions -url https://github.com/SpillmanTech/tcFunctions.git
$vcsRoot = Add-VcsRoot -newobj $newVcsRoot
$newVcsRootEntry = New-VcsRootEntry -vcsRootId $vcsRoot.id -checkoutRules ". => tcFunctions"
$vcsRootEntry = Add-VcsRootEntry -btid $buildType.id -newobj $newVcsRootEntry

$newBuildStep = New-TeamCityElementWithProperties -elementName step -type simpleRunner -data @{'command.executable'='pester/bin/pester.bat';'teamcity.build.workingDir'='tcFunctions'}
$buildStep = Add-BuildStep -buildType $buildType -newobj $newBuildStep
{% endhighlight %}

You now have project in TeamCity called tcFunctions, complete with a 'Run Tests' build that checks out Pester and tcFunctions and runs the tcFunctions tests. However, this build will fail. Many of the tests require a live project in TeamCity. There is also the matter of authentication. I'll cover these in my next post.
