---
layout: post
title: Snippet: Fix Razor Views Compilation in ASP.NET MVC
---

Errors in Razor Views by default are detected and displayed during runtime. In the past this has caused us issues in production where end-clients were hit by these errors instead us detecting them earlier. There is an obvious but inconvenient fix to prevent this from happening. <!--excerpt-->

## How to fix the problem?

You need to compile the razor views, so such errors can be detected during compile time instead of runtime. 

To do that, in ``.csproj`` add following piece of code:

    <Project>
        ..
        <PropertyGroup>
            <MvcBuildViews>true</MvcBuildViews>
            ..
        </PropertyGroup>

You'll think this is enough and ASP.NET MVC is smart to do it's job, but nope, **it will not simply work**. Why? Read more [here](http://stackoverflow.com/questions/4725387/mvcbuildviews-not-working-correctly/4732019#4732019).
Short story to working view compilation functionality is to add the following piece of code too in your ``.csproj``:

    <Target Name="MvcBuildViews" AfterTargets="AfterBuild" Condition="'$(MvcBuildViews)'=='true'">
        <AspNetCompiler VirtualPath="temp" PhysicalPath="$(WebProjectOutputDir)" />
    </Target>

That's it. Now it works.

## But it slows down the build time?

Oh, just to mention that Razor Views compilation is slow process, so you might want to trigger it before making deployments or in CI (``Release``), instead of slowing down the whole build process while you are coding (``Debug``).

The target conditions are the solution:

    <PropertyGroup Condition="'$(Configuration)|$(Platform)' == 'Debug|AnyCPU'">
        ..
        <MvcBuildViews>false</MvcBuildViews>
    </PropertyGroup>
    <PropertyGroup Condition="'$(Configuration)|$(Platform)' == 'Release|AnyCPU'">
        ..
        <MvcBuildViews>true</MvcBuildViews>
    </PropertyGroup>
