---
layout: post
title: Areas-like Nested Feature Folders in ASP.NET MVC 
---

In my [previous article](/2016/05/feature-folders-structure-in-asp-net/) I blogged about how to work with feature folders in ASP.NET MVC. In this article I focus on how to do the very same thing but with Areas support by nesting the feature folders. This is also a nice option if you want to do plugin-based work. 

## This is the Web MVC structure I want to support

![Areas-like Nested Feature Folders in ASP.NET MVC](/images/2016-07-10-areas-like-nested-feature-folders-in-asp-net/image01.png)

## Configuring your project

### First, you must replace the default Razor view engine with one that can distinguish nested feature folders.

    public class Global : HttpApplication
    {
        void Application_Start(object sender, EventArgs e)
        {
            // Code omitted for clarity

            ViewEngines.Engines.Clear();
            ViewEngines.Engines.Add(new FeatureFoldersRazorViewEngine(rootNamespace: typeof(Global).Namespace));
        }
    }
    
    public class FeatureFoldersRazorViewEngine : RazorViewEngine
    {
        // This needs to be initialized to the root namespace of your MVC project.
        private static string _rootNamespace; // e.g. BooUniversity.Delivery.WebMvc

        public FeatureFoldersRazorViewEngine(string rootNamespace)
        {
            _rootNamespace = rootNamespace;
        }

        public override ViewEngineResult FindView(ControllerContext controllerContext, string viewName, string masterName, bool useCache)
        {
            var path = GetPath(controllerContext, viewName);
            return VirtualPathProvider.FileExists(path)
                ? new ViewEngineResult(CreateView(controllerContext, path, null), this)
                : new ViewEngineResult(new[] { path });
        }

        private static string GetPath(ControllerContext controllerContext, string viewName)
        {
            var controllerType = controllerContext.Controller.GetType();
            var featureFolder = controllerType.Namespace
                .Replace(_rootNamespace, string.Empty)
                .Replace(".", "/");
            return $"~{featureFolder}/{viewName}.cshtml";
        }

        public override ViewEngineResult FindPartialView(ControllerContext controllerContext, string partialViewName, bool useCache)
        {
            var path = GetPath(controllerContext, partialViewName);
            return VirtualPathProvider.FileExists(path)
                ? new ViewEngineResult(CreateView(controllerContext, path, null), this)
                : new ViewEngineResult(new[] { path });
        }
    }

### Second, create base controllers for each area-like-feature, so later you can specify which route configuration is bound to which namespace

For example, for *Admin* area feature folder, create new base controller and put it in the *Features/Admin* folder. All controllers in the Admin big-feature folder must inherit from this base controller.

    namespace BooUniversity.Delivery.WebMvc.Features.Admin
    {
        public abstract class AdminMvcController : BaseMvcController
        {
        }
    }

For *Public* do the same thing.

    namespace BooUniversity.Delivery.WebMvc.Features.Public
    {
        public abstract class PublicMvcController : BaseMvcController
        {
        }
    }

Keep an eye on the namespaces where these controllers are placed.

    // Features/Admin/Home/
    namespace BooUniversity.Delivery.WebMvc.Features.Admin.Home
    {
        public class HomeController : AdminMvcController
        {
            public ActionResult Index() => View();  // Features/Admin/Home/Index.cshtml
        }
    }

    // Features/Public/Home/
    namespace BooUniversity.Delivery.WebMvc.Features.Public.Home
    {
        public class HomeController : PublicMvcController
        {
            public ActionResult Index() => View();  // Features/Public/Home/Index.cshtml
        }
    }


### Third, create routes for each area-feature

    public class RouteConfig
    {
        public static void RegisterRoutes(RouteCollection routes)
        {
            routes.IgnoreRoute("{resource}.axd/{*pathInfo}");

            var routeAdmin = routes.MapRoute(
                name: "Admin",
                url: "Admin/{controller}/{action}/{id}",
                defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional },
                namespaces: ReflectionUtil.GetNamespacesForInheritedClassesOf<AdminMvcController>()
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

### Finally, run your app and try out some routes to see how it works

    http://localhost:port/                 -> Public/Home/Index.cshtml
    http://localhost:port/Home             -> Public/Home/Index.cshtml
    http://localhost:port/Home/Index       -> Public/Home/Index.cshtml
    http://localhost:port/Admin            -> Admin/Home/Index.cshtml
    http://localhost:port/Admin/Home       -> Admin/Home/Index.cshtml
    http://localhost:port/Admin/Home/Index -> Admin/Home/Index.cshtml

## Summary

TODO

Happy coding!  