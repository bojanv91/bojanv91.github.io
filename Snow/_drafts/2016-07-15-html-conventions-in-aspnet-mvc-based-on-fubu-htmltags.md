---
layout: post
title: Metadata driven UIs with Html Conventions in ASP.NET MVC (based on Fubu's HtmlTags)
---

Html conventions are opinionated input and html builders based on metadata. Instead of explicitly declaring what types of inputs and outputs will be rendered, you simply state that there will be an input or output rendered, and the conventions mechanism kick in to determine the actual types to be used based on the models' metadata.

I use Html conventions in my apps because they provide a single point where I can change or extend appearance or functionality, and also to standardize my views. I don't need to change every view if I decide to change one datepicker with another, or change the way how form validation messages are displayed, or how required fields are marked with asterix, or how to provide additional help page links for each field, or... you get the point.  

In my web apps,<!--excerpt--> I use the model to define what data will be available on the view. On the model I use DataAnnotations to provide the view with metadata about how to render certain controls. But also, besides the DataAnnotations, in few cases I build metadata from FluentValidation rules, e.g. NotEmpty(), GreaterThanOrEqual(...) and similar. For instance I have ColorAttribute which renders a color picker; a DateTime property which renders a date picker; a decimal property which renders a decimal masked input field; and similar.

[TOC]      

## Introducing Fubu's HtmlTags

Fubu's HtmlTags is a library for generating HTML elements based on object models. It has fluent API which is easy to chain with methods, which are modeled after jQuery. Given HtmlTags flexibility you can build whatever tags you want with great ease, which is not the case with the default MVC HtmlHelpers.

Here is an example how you can build a button link tag with statically typed URLs to your controller's action methods:

    namespace System.Web.Mvc
    {
        public static class HtmlHelperExtensions
        {
            public static HtmlTag LinkButton<TController>(this HtmlHelper helper, Expression<Action<TController>> action, string linkText)
                where TController : Controller
            {
                var url = WebUtil.BuildLink(action, helper.RouteCollection, helper.ViewContext.RequestContext);
                return new HtmlTag("a")
                    .Text(linkText)
                    .Attr("href", url)
                    .AddClass("btn btn-default");
            }
        }
    }

You can use it in your views like this:

    @(Html.Link<CoursesController>(x => x.Index(), "List of courses"))

Which will generate an HTML like this:

    <a href="http://YOURDOMAIN/Courses/Index">List of courses</a>

Okay, this looks nice but let's focus back on the primary topic. The area of great value I want to dive in, is in our forms. Namely, one form can have many different types of input fields, and I want my input fields to be generated automatically based on the metadata which is provided. Let's dig in.

## Configuring Html Conventions

In your IoC container you must register a class that provides the convention configuration. This class defines tag builders and modifiers that get executed based on whether they match the property being rendered. The class finds the first tag builder that matches the property and executes it. Then it finds all the modifiers which match the property and executes them on the tag. Lastly on this tag is called .ToString() method and is rendered to the page markup.

### Sample conventions

    public class MyHtmlConventions : DefaultHtmlConventions
    {
        public MyHtmlConventions()
        {
            Labels.Always.AddClass("control-label");
            Labels.ModifyForAttribute<DisplayAttribute>((t, a) => t.Text(a.GetName()));

            Editors.Always.AddClass("form-control");
            Editors.IfPropertyIs<bool>().ModifyWith(m => m.CurrentTag.Attr("type", "checkbox").RemoveClass("form-control"));
            Editors.IfPropertyIs<int>().ModifyWith(m => m.CurrentTag.Data("type", "integer"));
            Editors.Modifier<EnumModifier>();   // Generates <option> element with list of all available enums.
            Editors.ModifyForAttribute<ReadOnlyAttribute>((tag, attr) =>
            {
                if (attr.IsReadOnly)
                    tag.Attr("readonly", "readonly");
            });
        }
    }

### Registering MyHtmlConventions in StructureMap IoC

    public class Bootstrapper
    {
        public Container Container { get; private set; }

        public Bootstrapper()
        {
            Container = new Container(cfg =>
            {
                // HtmlConventions
                var htmlConventionLibrary = new HtmlConventionLibrary();
                var conv = new MyHtmlConventions();
                conv.Apply(htmlConventionLibrary);
                cfg.For<HtmlConventionLibrary>().Use(htmlConventionLibrary);
            });
        }
    }

## Using Html Conventions

### HtmlExtensions which provide access to the conventions from the views

    namespace System.Web.Mvc
    {
        public static class HtmlHelperExtensions
        {
            public static HtmlTag InputFor<TModel>(this HtmlHelper<TModel> helper, Expression<Func<TModel, object>> expression)
                where TModel : class
            {
                return GetGenerator().InputFor(expression, model: helper.ViewData.Model);
            }

            public static HtmlTag LabelFor<TModel>(this HtmlHelper<TModel> helper, Expression<Func<TModel, object>> expression)
                where TModel : class
            {
                return GetGenerator().LabelFor(expression, model: helper.ViewData.Model);
            }

            internal IElementGenerator<TModel> GetGenerator()
                where TModel : class
            {
                var library = DependencyResolver.Current.GetService<HtmlConventionLibrary>();
                return ElementGenerator<TModel>.For(library);
            }
        }
    }

### Using Html Conventions in the views

    <form>
        <div class="form-group">
            @Html.LabelFor(x => x.Title)
            @Html.InputFor(x => x.Title)
        </div>
    </form>

References:

https://github.com/HtmlTags/htmltags
https://medium.com/@ZombieCodeKill/advanced-web-forms-a98988996c02#.d1qee5fs4
https://code.msdn.microsoft.com/ASPNET-MVC-Application-b01a9fe8
https://lostechies.com/jimmybogard/2013/08/13/conventional-html-in-asp-net-mvc-building-tags/
http://codebetter.com/jeremymiller/2010/01/29/shrink-your-views-with-fubumvc-html-conventions/
http://geekswithblogs.net/ryanohs/archive/2010/02/24/using-fubumvcrsquos-html-conventions-in-microsoft-mvc.aspx
http://blog.ntcoding.com/2011/11/fun-with-fubus-html-conventions.html 
https://www.benefitfocus.com/blogs/shawn-jenkins/metadata-driven-architecture
http://www.evolutility.org/product/Product.aspx
https://www.infoq.com/articles/mdd-webapi-for-mobile-apps
http://www.dotnet-tricks.com/Tutorial/mvc/N50P050314-Understanding-HTML-Helpers-in-ASP.NET-MVC.html
http://demos.telerik.com/aspnet-mvc/