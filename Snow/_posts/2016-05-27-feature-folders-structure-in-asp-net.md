---
layout: post
title: Feature Folders Project Structure in ASP.NET MVC5
---

Structuring your files around **business concerns** is more natural way of handling projects than structuring them around **technical concerns**. The [Separation of Concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) is applied in both approaches, but not both of them give the same desired clarity and ease for developers. This blogpost focuses on organizing MVC projects around **feature folders**, which represent the business concerns.<!--excerpt-->

Most of the time developers make modifications related to a single feature (e.g. adding new fields, changing business rules, adding validation...). Structuring folders around interrelated files can make modification process simpler. The common MVC folder structure violates the rule of *"Files that change together should be structured together"*. Structuring by business concerns embraces this very important rule.

> Files that change together should be structured together.


## Horizontal (Technical) vs. Vertical (Business) Folder Structure 

On the left side you can see the common MVC structure. On the right side you can see the feature folders structure of the very same project.

    Styles
        Shared.css
        Login.css
    Scripts
        Login.js
    Controllers
        CoursesController.cs
        DepartmentsController.cs
        EnrollmentsController.cs
        InstructorsController.cs
        StudentsController.cs
        UsersController.cs
    Models  
        CourseEditModel.cs
        CourseIndexModel.cs
        DepartmentEditModel.cs
        DepartmentIndexModel.cs
        EnrollmentEditModel.cs
        EnrollmentIndexModel.cs
        InstructorEditModel.cs
        InstructorIndexModel.cs
        StudentEditModel.cs
        StudentIndexModel.cs
        UserLoginModel.cs
        UserRegisterModel.cs
        UserForgotPasswordModel.cs
    Views
        Courses
            Edit.cshtml
            Index.cshtml
        Departments
            Edit.cshtml
            Index.cshtml
        Enrollments
            Edit.cshtml
            Index.cshtml
        Instructors
            Edit.cshtml
            Index.cshtml
        Shared
            _Layout.cshtml
        Students
            Edit.cshtml
            Index.cshtml
        Users
            ForgotPassword.cshtml
            Login.cshtml
            Register.cshtml
    _ViewStart.cshtml

<div style=""></div>

    Features
        Courses
            CoursesController.cs
            Edit.cs
            Edit.cshtml
            Index.cs
            Index.cshtml
        Departments
            DepartmentsController.cs
            Edit.cs
            Edit.cshtml
            Index.cs
            Index.cshtml
        Enrollments
            EnrollmentsController.cs
            Edit.cs
            Edit.cshtml
            Index.cs
            Index.cshtml
        Instructors
            InstructorsController.cs
            Edit.cs
            Edit.cshtml
            Index.cs
            Index.cshtml
        Shared
            _Layout.cshtml
            Shared.css
        Students
            StudentsController.cs
            Edit.cs
            Edit.cshtml
            Index.cs
            Index.cshtml
        Users
            UsersController.cs
            ForgotPassword.cs
            ForgotPassword.cshtml
            Login.cs
            Login.cshtml
            Login.css
            Login.js
            Register.cs
            Register.cshtml
    _ViewStart.cshtml


When you see this in your IDE (e.g. in Visual Studio), the distinction between the files is even greater, given that there is accompanied file type icon shown besides the filename.

Now, imagine you scale in amount of features, in addition to the standard N-Layer stuff like repositories, services, handlers, DTOs, etc... You will soon notice that things are starting to get messy in the technical folders organization. In the feature folders organization, each feature can scale on it's own, thus much easier to manage.

Food for thought:

- What if we put our CSS and JavaScript files also in these feature folders?
- What if one feature folder becomes so demanding on the UI that needs to be a full SPA view/module - can we structure it to use Angular?
- Can we develop one feature UI in Angular, another one in React, all other in classic server-side MVC, and stay sane with our overall project structure?

Example of single feature evolved as Angular application/module:


    Features
        ...
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
        ...


## Benefits of using Feature Folders (over technical folder structure)

Structuring your files by features (business concerns) makes things easier to find and manage. 

- You don't step over each other toes with your peers, thus, avoid spending time on fixing merge conflicts. 
- You can scale and modify each feature on its own, independently from other features and even use different UI technology.
- You immediately understand what an application does and where to find necessary files for your given requirement.
- You can easily reuse similar features across projects by simply copying just a single folder. 
- Time spent on navigation through Solution Explorer to locate interdependent files is drastically reduced since they are all in a single folder.
- You can reason much easier about each feature just by looking in a single (feature) folder.


## Implementing Feature Folders in ASP.NET MVC 5

To make this work in ASP.NET MVC 5, we should inherit the ``RazorViewEngine`` and change the view location parts to ones that fit our new structure.

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

Next, we have to add our newly created ``FeatureFoldersRazorViewEngine`` in our applcation.

    public class Global : HttpApplication
    {
        void Application_Start(object sender, EventArgs e)
        {
            // ...
            ViewEngines.Engines.Clear();
            ViewEngines.Engines.Add(new FeatureFoldersRazorViewEngine());
        }
    }


## Summary

Structuring our MVC projects following feature folders approach increases the productivity of our dev teams.

At our company, we have been using feature folders project structure on over dozens projects for over a year, and due to the high success and productivity boost, it became our default project structure on the presentation layer. 