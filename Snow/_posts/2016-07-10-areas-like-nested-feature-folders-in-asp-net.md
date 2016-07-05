---
layout: post
title: Areas-like Nested Feature Folders in ASP.NET MVC 
---

In the previous article I blogged about how to work with feature folders in ASP.NET MVC. In this article I focus on how to do the very same thing but support almost unlimited nesting of feature folders. This is needed in order to support Areas-like folder structure in your application.

# Here is the structure to be supported

![Areas-like Nested Feature Folders in ASP.NET MVC](/images/2016-07-10-areas-like-nested-feature-folders-in-asp-net/image01.png)

# Configuring your project

## First, to make this work in ASP.NET MVC, the default Razor view engine should be replaced with one that makes the distincion of feature folders.

    public class Global : HttpApplication
    {
        void Application_Start(object sender, EventArgs e)
        {
            // ...
            ViewEngines.Engines.Clear();
            ViewEngines.Engines.Add(new FeatureFoldersRazorViewEngine());
        }
    }
    
    public class FeatureFoldersRazorViewEngine : RazorViewEngine
    {
        //This needs to be initialized to the root namespace of your MVC project.
        private static readonly string RootNamespace = "VERT.WebApp";   // TODO: Change the namespace to your MVC project

        private static string GetPath(ControllerContext controllerContext, string viewName)
        {
            var controllerType = controllerContext.Controller.GetType();
            //TODO: Add caching
            var featureFolder = "~" + controllerType.Namespace.Replace(RootNamespace, string.Empty).Replace(".", "/");

            var path = featureFolder + "/" + viewName + ".cshtml";
            return path;
        }

        public override ViewEngineResult FindView(ControllerContext controllerContext, string viewName, string masterName, bool useCache)
        {
            var path = GetPath(controllerContext, viewName);
            return VirtualPathProvider.FileExists(path)
                ? new ViewEngineResult(CreateView(controllerContext, path, null), this)
                : new ViewEngineResult(new[] { path });
        }

        public override ViewEngineResult FindPartialView(ControllerContext controllerContext, string partialViewName, bool useCache)
        {
            var path = GetPath(controllerContext, partialViewName);
            return VirtualPathProvider.FileExists(path)
                ? new ViewEngineResult(CreateView(controllerContext, path, null), this)
                : new ViewEngineResult(new[] { path });
        }
    }

## Second, create base controllers for each big feature folder, so in the routing you can make distinction via the namespaces (similar to how Areas work by default)

For example, for *Administration* big feature folder, create new controller and put it inside the *Administration* folder. All other controllers should inherit this base controller.

    namespace VERT.WebApp.Features.Administration
    {
        public abstract class AdministrationMvcController : BaseMvcController
        {
        }
    }

For *Public* do the same thing.

    namespace VERT.WebApp.Features.Public
    {
        public abstract class PublicMvcController : BaseMvcController
        {

        }
    }

Keep an eye on the namespaces where these controllers are placed.

## Third, configure the routing

    public class RouteConfig
    {
        public static void RegisterRoutes(RouteCollection routes)
        {
            routes.IgnoreRoute("{resource}.axd/{*pathInfo}");

            var routeAdministration = routes.MapRoute(
                name: "Administration",
                url: "Administration/{controller}/{action}/{id}",
                defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional },
                namespaces: ReflectionUtil.GetNamespacesForInheritedClassesOf<AdministrationMvcController>()
            );

            var routePublic = routes.MapRoute(
                name: "Public",
                url: "{controller}/{action}/{id}",
                defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional },
                namespaces: ReflectionUtil.GetNamespacesForInheritedClassesOf<PublicMvcController>()
            );
        }
    }

    // A help hand with reflection
    public static class ReflectionUtil
    {
        public static string[] GetNamespacesForInheritedClassesOf<T>()
        {
            return Assembly.GetAssembly(typeof(T))
                .GetTypes()
                .Where(myType => myType.IsClass && !myType.IsAbstract && myType.IsSubclassOf(typeof(T)))
                .Select(type => type.Namespace)
                .ToArray();
        }
    }

## Finally, check how this works in your browser



--------
# Summary

In previous article you read the story behind feature folders and how to make it work in your ASP.NET MVC application.
In this article you read how to configure this approach to work with areas-like nested feature folders.

For us, this approach works like a charm.

Happy coding!  