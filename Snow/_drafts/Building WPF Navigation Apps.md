Building WPF Navigation Application

Few months ago I had the "privilege" to inherit WPF codebase. It has custom built navigation, instad of incorporating the WPF features.

Navigation applications are applications composed of many incorporated pages inside a single window and they feel like web applications.
Navigation comprises many functions: going back, going forward, going to a new page, refreshing a page, and so on.

The pages of a navigation app can be *standalone* or *pages that make step in a workflow*.

While working on production application, I found this blog post extremelly useful in order to setup the project - http://paulstovell.com/blog/wpf-navigation

## The Shell - Startup window that contains all pages and navigation bar

The Shell is the main window of our application and is responsible for hosting our pages.

    <Window x:Class="HaldaTest.Shell"
            xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
            xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
            xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
            xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
            xmlns:local="clr-namespace:HaldaTest"
            mc:Ignorable="d"
            Title="Shell" Height="300" Width="300">
        <DockPanel>
            <Frame x:Name="MainFrame" />
        </DockPanel>
    </Window>

In code we can navigate in the *MainFrame* like this:

    MainFrame.Navigate(new Page1());
    MainFrame.GoBack();
    MainFrame.Refresh();

## Customizing the navigation UI

## Navigation between pages and passing data

## Page lifecycle in navigation apps

## PageFunction<> for "de-tour" navigation

## Types of "Windows"

### Window

### Page

### PageFunction

### UserControl