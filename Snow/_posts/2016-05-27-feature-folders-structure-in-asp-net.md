---
layout: post
title: "Feature Folders" structure in ASP.NET MVC
---

Structuring files around **business concerns** is more convenient and natural way of handling projects than structuring them around **technical concerns**. The [Separation of Concerns](http://deviq.com/separation-of-concerns/) is applied in both approaches, but not both of them gives the same desired clarity and ease of handling a project. This article focuses on which structure gives better results.<!--excerpt-->

Let's think about how do project managers usually request changes in requirements ?

- Change these fields in all these views; all these models; all these controllers, or
- Change these fields in customer registration; shopping cart payment; ...

Most of the time developers make modifications related to a single feature (e.g. adding new field). Structuring folders around interrelated files can make modification process simpler. The default MVC folder structure violates the rule of *"Files that change together should be structured close together"*. Structuring by business concerns embraces this important rule.

Let's see both approaches in examples. 

# Structuring files by technical concerns (horizontal)

    Controllers
        CustomersController.cs
        OrdersController.cs
        ShoppingCartController.cs
        ProductsController.cs
        ...
    Models
        CustomersEditModel.cs
        CustomersIndexModel.cs
        CustomersLoginModel.cs
        CustomersRegisterModel.cs
        OrdersDetailsModel.cs
        OrdersIndexModel.cs
        ShoppingCartCustomerDetails.cs
        ShoppingCartIndex.cs
        ShoppingCartOrdersList.cs
        ShoppingCartPaymentDetails.cs
        ...
    Views
        Customers
            Edit.cshtml
            Index.cshtml
            Login.cshtml
            Register.cshtml
        Orders
            Details.cshtml
            Index.cshtml
        ShoppingCart
            _MiniCart.cshtml
            CustomerDetails.cshtml
            Index.cshtml
            OrdersList.cshtml
            PaymentDetails.cshtml
        Products
            ...

Now, imagine you have many-many controllers, in addition to the standard N-Layer stuff like repositories, services, DTOs, etc... You will soon notice that things are starting to get messy.

# Structuring files by business concerns (vertical)

    Features
        Customers
            CustomersController.cs
            Edit.cs
            Edit.cshtml
            Index.cs
            Index.cshtml
            Login.cs
            Login.cshtml       
            Register.cs         
            Register.cshtml
        Orders
            Details.cs
            Details.cshtml
            Index.cs
            Index.cshtml
            OrdersController.cs
        ShoppingCart
            _MiniCart.cshtml
            CustomerDetails.cs
            CustomerDetails.cshtml
            Index.cs
            Index.cshtml
            OrdersList.cs
            OrdersList.cshtml
            PaymentDetails.cs
            PaymentDetails.cshtml
            ShoppingCartController.cs
        Products
            ...

In Visual Studio this looks like following:

![Feature Folders example in ASP.NET MVC](/images/2016-05-27-feature-folders-structure-in-asp-net/image01.png)

Food for thought:
 
- What if we put our JavaScript files also in these feature folders?
- What if one feature folder becomes so demanding on the UI that needs to be a full SPA view/module - can we structure it to use Angular (or React, or whatever)?

Example of single feature evolved as Angular application:

    ShoppingCart
        Components
            CartComponent.js
            CartComponent.css
            PaymentComponent.js
            PaymentComponent.css
            CartContainer.js
        App.js
        App.css
        Index.cshtml
        ShoppingCartController.cs

# Configuration

To make this work in ASP.NET MVC, the default Razor view engine should be replaced with one that makes the distincion of feature folders.

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
        public FeatureFoldersRazorViewEngine()
        {
            var featureFolderViewLocationFormats = new[]
            {
                "~/Features/{1}/{0}.cshtml",
                "~/Features/{1}/{0}.vbhtml",
                "~/Features/Shared/{0}.cshtml",
                "~/Features/Shared/{0}.vbhtml",
            };

            ViewLocationFormats = featureFolderViewLocationFormats;
            MasterLocationFormats = featureFolderViewLocationFormats;
            PartialViewLocationFormats = featureFolderViewLocationFormats;
        }
    }

# Summary

Structuring files by features (business concerns) makes things easier to find and manage. 

- You don't step over each other toes with your peers, thus, avoid spending time on fixing merge conflicts. 
- You can scale and modify each feature on its own, independently from other features.
- You immediately understand what an application does and where to find necessary files for your given requirement.
- You can easily reuse similar features across projects by simply copying a single folder.
- Time spent on navigation through Solution Explorer to locate interdependent files is drastically reduced since they are all in a single folder. 

At our company, we have been using structuring around business concerns on over dozens of projects for more than a year, and due to the high success and productivity boost in our teams, it became our default structure on the presentation layer. But, you may ask - how do we structure our application services and data access? Stay tuned...

Happy coding!  