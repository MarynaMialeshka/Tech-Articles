# How I updated lib version in the entire solution and what problems it caused

This article is going to be a reflection on the task I completed (eventually) to pavoid such rather stupid mistakes in the future. Since it wasn't the last time I'm going to face such a challenge, I counted it worthy to note down my fails and make kind of a checklist for dealing with such issues. Yet we all wish we'd have a chance to avoid such unrewarding work :)

## Task description

I needed to upgrade versions of a few nuget packages used on my project in order to meet the security requirements, as the outdated versions of the packages in question were proved to be insecure. 

Everything was fine except for package A depending on package B (omitting the package names as insignificant, in fact).

## How I tackled the task

I decided to cut it short - via the built in NuGet Package Manager on Visual Studio. However, I decided I was more intelligent than it and removed some unnecessary (IMO) changes. After building the solution successfully and checking a few web applications were running on my machine, I moved the task to testing.

## Problem 1. Incorrect binding redirects

We all have seen such lines in config files:

        <dependentAssembly>
        <assemblyIdentity name="someAssembly"
            publicKeyToken="32ab4ba45e0a69a1"
            culture="en-us" />
        <bindingRedirect oldVersion="7.0.0.0" newVersion="8.0.0.0" />
        </dependentAssembly>


They handle the presence of multiple library versions during the runtime and choose a relevant one. E.g you have package A depending on package C.1.0.0, and package B depending on package C.1.0.1. Binding redirect will resolve this ambiguity (ususally, in favour for a higher version). 

It's important the version in question was downloaded during compile-time. Here is where I encountered first problems - manually removed binding redirect lines, I made the projects reference lib versions not downloaded on the environment.

## Problem 2. Multiple lib versions across solution 

Additionally, I upgraded package A versions for the projects menti0oned in the task description only (client and test applications). Which resulted in version mismatch during runtime - some projects referenced package versions not downloaded during on the environment. Pretty much the same problem as #1. However, accounted for my lack of attention to detail rather than unprofessionalism, if it may serve as an excuse :)

## Checklist to deal with upgrading lib versions

Finally, what it all was for, some actiion items to prevent the shutdowns of the whole environment in the future.

+ Trust NuGet Package Manager and include all its suggestions, however irrelevant they may seem.
+ Not only biuld and check some client apps locally, but also run all services which may be affected (in case of a huge solution and ubiquitous packages - literally, *each* service).
+ Check health status of all services (on Azure portal in my case) upon deplyment.
+ Make sure some side solutions which may depend on the packages downloaded during compile-time of the main one are updated as well (I missed one, because encoutered it second time only).

