[English](#english-version) | [中文](#chinese-version)

# English Version

## RJi.Mvvm.Wpf

The documentation has been rewritten to adapt to the latest version, applicable for 1.2.7.

### Introduction

RJi.Mvvm.Wpf is a lightweight WPF MVVM framework for `.NET 6.0`, built on `CommunityToolkit.Mvvm`. This framework provides functionality similar to `Prism.Wpf` while integrating `Microsoft.Extensions.Hosting` for dependency injection management. It supports navigation between `ContentControl` and `UserControl`, and implements features such as navigation parameter passing, navigation history, navigation confirmation, dialog windows, and view caching, providing convenient development support for small WPF projects.

### Core Features

- **Automatic ViewModel Registration**: When the project starts, the framework automatically scans the project's ViewModels folder through reflection and registers all classes ending with "ViewModel" into the dependency injection container, defaulting to transient injection.
- **ViewModel Locator**: Similar to Prism.Wpf, it automatically associates contexts that follow the rules.
- **Navigation System**:
  - **View Zones**: Provides the attached property `ViewZone.Name` for marking `ContentControl` controls, enabling seamless navigation between `ContentControl` and `UserControl`.
  - **Navigation Interfaces**: Provides a series of interfaces that can be inherited by `ViewModel`, such as `INavigationService`, `INavigationLifecycle`, `IViewZoneLifetime`, `IHistoryRecordable`, `INavigationHistory`, `IConfirmNavigationRequest`, etc., supporting navigation parameter passing, forward/backward navigation, and view lifecycle management.
- **Dialog Windows**:
  - Built-in default `DialogWindow` control, used as a standard dialog window.
  - **Dialog Interfaces**: Supports custom dialog windows by implementing `IDialogService`, including style settings and lifecycle management.
- **Dependency Injection Container Extension Methods**: Provides convenient extension methods for `IServiceCollection`, including `AddViewToNavigation`, `AddViewToDialog`, `AddDialogWindow`, simplifying the configuration of navigation and dialog functionality.

### Design Goals

- **Low Intrusion**: Follows the "convention over configuration" principle, minimizing code modifications.
- **High Extensibility**: Supports dependency injection, custom navigation and dialogs, and complete lifecycle management.
- **Developer-Friendly**: Automatically implements `ViewModel` binding, simplifies navigation logic, and significantly improves WPF development efficiency.
- **Applicable Scenarios**: Specifically designed for small to medium WPF projects, enabling quick construction of modular MVVM applications.

### Development Roadmap

| Feature Module          | Current Status      | Priority | Target Version | Description / Optimization Direction                    |
| ----------------------- | ------------------- | -------- | -------------- | ------------------------------------------------------- |
| Modularization          | ⏳ Planning         |          |                | -                                                       |
| Continuous Optimization | ✅ Partially Stable | P1       | V1.2.7         | Optimize historical issues and improve system stability |

**Status Explanation**:

🔧 In Development | ⏳ Planning | 🐞 To be Optimized | ✅ Stable

- **Priority**: P0 (Highest) > P1 > P2

## Quick Start

Create a new WPF project, add the namespace `xmlns:rj="http://rji.mvvm.wpf/views"` to App.xaml, replace the root node `Application` with `rj:RJiApplication`, and remove the default `StartupUri` property.

```cs
<rj:RJiApplication x:Class="WpfAppDemo.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:WpfAppDemo"
             xmlns:rj="http://rji.mvvm.wpf/views">
    <rj:RJiApplication.Resources>

    </rj:RJiApplication.Resources>
</rj:RJiApplication>
```

In `App.xaml.cs`, reference the namespace `RJi.Mvvm.Wpf`, modify App to inherit from `RJiApplication`, override the `ConfigureServices` and `CreateShell` methods, and return the main window `MainWindow` as the application startup main page in `CreateShell`.

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using System.Windows;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

Create two folders in the project, `Views` and `ViewModels`, for storing corresponding classes.

The namespaces of the classes in the folders should include `.Views` / `.ViewModels` respectively, and the prefixes should be consistent. `ViewModel` classes must end with `ViewModel`.

Example: Project.Views.MainWindow → Project.ViewModels.MainWindowViewModel

The project can start and run normally, and the `MainWindow` returned by `CreateShell()` will automatically associate the corresponding `MainWindowViewModel` as `DataContext`.

This automatic association only takes effect when the following methods are called:

- `CreateShell()`
- Navigation services: `NavigateTo()`/`SetInitialView()`
- Dialog services: `ShowDialog()/Show()`

## MVVM Support

Uses `CommunityToolkit.Mvvm` as the MVVM support package.

[Introduction to MVVM Toolkit - Community Toolkits for .NET | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/)

## RJiApplication Base Class

### Automatic ViewModel Registration

This base class relies on the dependency injection container in `Microsoft.Extensions.Hosting` and **automatically registers all compliant ViewModels as transient** through reflection when the application starts.

Requirements: Located in the ViewModels folder, namespace contains `.ViewModels`, class name ends with `ViewModel`.

**Important Notes**:

- Microsoft DI does not support runtime dynamic registration; all services must be registered before application startup.
- When ViewModel is automatically registered, it will use the **constructor with the most parameters** for instantiation, and automatically register all recognizable parameter types in the project as transient through recursive methods.
- Custom interface type services must be manually configured with associated implementation types, otherwise an exception will be thrown.
- Framework built-in interfaces do not need to be manually registered.

Code example:

App.xaml.cs

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using System.Windows;
using WpfAppDemo.Services;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
            services.AddSingleton<IMyService, MyService>(); // Custom services must be manually registered
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

MainWindowViewModel.cs

```cs
using CommunityToolkit.Mvvm.Messaging;
using RJi.Mvvm.Wpf.Navigation;
using WpfAppDemo.Services;

namespace WpfAppDemo.ViewModels
{
    internal class MainWindowViewModel
    {
        private readonly INavigationService _navigationService; // Navigation service - RJi.Mvvm.Wpf.Navigation
        private readonly IMessenger _messenger; // Message service - CommunityToolkit.Mvvm.Messaging
        private readonly IMyService _myService; // Custom service

        public MainWindowViewModel(INavigationService navigationService, IMessenger messenger, IMyService myService)
        {
            _navigationService=navigationService;
            _messenger=messenger;
            _myService=myService;
        }
    }
}
```

### RJiApplication Virtual Methods

**InitializeAsync**: Application initialization, configures `Hosting`.

```cs
protected virtual async Task InitializeAsync()
{
    var hostBuilder = CreateAndConfigureHostBuilder();
    _host=hostBuilder.Build();
    Ioc.Default.ConfigureServices(_host.Services);
    await _host.StartAsync().ConfigureAwait(true);
    var shell = CreateShell();
    ArgumentNullException.ThrowIfNull(shell, nameof(shell));
    MvvmHelpers.EnsureAutoResolveViewModelEnabled(shell);
    InitializeShell(shell);
}

private IHostBuilder CreateAndConfigureHostBuilder()
{
    var hostBuilder = Host.CreateDefaultBuilder();
    hostBuilder.ConfigureServices(
        (context, services) =>
        {
            ConfigureDefaultServices(services);
            ConfigureServices(context, services);
        });
    ConfigureHostBuilder(hostBuilder);
    return hostBuilder;
}
```

**InitializeShell**: Initialize the startup page.

```cs
protected virtual void InitializeShell(Window shell)
{
  MainWindow=shell;
}
```

**OnInitialized**: Main window display.

```cs
protected virtual void OnInitialized()
{
  MainWindow?.Show();
}
```

**ConfigureViewModelResolver**: Configure ViewModel resolver, ViewModel locator will resolve instances from the dependency injection container; this framework is based on `CommunityToolkit.Mvvm.DependencyInjection.Ioc` implementation.

```cs
protected virtual void ConfigureViewModelResolver()
{
    ViewModelResolverProvider.ConfigureViewModelFactory(viewModelType => Ioc.Default.GetService(viewModelType)!);
}
```

**ConfigureDefaultServices**: Configure DI container default services, executed earlier. Subsequent services can be registered in the abstract method `ConfigureServices` (DI container will prioritize the last registered service).

```cs
protected virtual void ConfigureDefaultServices(IServiceCollection services)
{
  services.AddMessenger()
          .AddDialogService()
          .AddNavigationService()
          .AddViewModels();
  }
```

**ConfigureHostBuilder**: Default is empty, used as a hook method for configuring HostBuilder and custom services.

```cs
protected virtual void ConfigureHostBuilder(IHostBuilder hostBuilder)
{
}
```

### RJiApplication Properties

`InitializeAsync()` has already configured `CommunityToolkit.Mvvm.DependencyInjection.Ioc`, `CommunityToolkit.Mvvm` provides a singleton pattern `Ioc.Default` for obtaining the container. In `RJiApplication`, the dependency injection container can also be accessed through the `ServiceProvider` property.

```cs
 public IServiceProvider ServiceProvider
 {
     get { return Ioc.Default; }
 }
```

## Ioc

### Singleton Pattern

The namespace `CommunityToolkit.Mvvm.DependencyInjection` provides a singleton pattern `Ioc.Default` for obtaining the container. Other services need to be registered in App.xaml.

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using System.Windows;
using WpfAppDemo.Services;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
            services.AddSingleton<IMyService, MyService>(); // Custom services must be manually registered
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

### Messenger

The `CommunityToolkit.Mvvm` messenger has been registered to the IoC container. After the view model inherits `ObservableRecipient`, it will automatically obtain the `Messenger` property; **the container injected instance, singleton `WeakReferenceMessenger.Default`, and this property are the same instance**.

```cs
using CommunityToolkit.Mvvm.Messaging;
using RJi.Mvvm.Wpf.Navigation;
using WpfAppDemo.Services;
using CommunityToolkit.Mvvm.Messaging.Messages;

namespace WpfAppDemo.ViewModels
{
    internal class MainWindowViewModel: ObservableRecipient
    {
        private readonly IMessenger _messenger;

        public MainWindowViewModel(IMessenger messenger)
        {
            _messenger=messenger;
            var message = new ValueChangedMessage<string>("Hello,MainWindowViewModel");
            Messenger.Send(message);
            _messenger.Send(message);
            WeakReferenceMessenger.Default.Send(message);
        }
    }
}

```

## ViewModel Locator

Similar to Prism.Wpf, when executing `CreateShell()`, navigation services `NavigateTo()`/`SetInitialView()`, and dialog services `ShowDialog()`/`Show()` methods, it tries to obtain the class with the same name as `View` and ending with `ViewModel` from the `Ioc` container as the `DataContext` of the `View`.

### Attached Properties

The locator function is implemented from the dependency property `rj:ViewModelResolver.AutoResolveViewModel="True"`. When `CreateShell()`, navigation services `NavigateTo()`/`SetInitialView()`, and dialog services `ShowDialog()`/`Show()` are executed, this attached property will be automatically added to the control and set to true. In other cases, it needs to be manually added to take effect.

```cs
<UserControl x:Class="WpfAppDemo.Views.MyView1"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:rj="http://rji.mvvm.wpf/views"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             rj:ViewModelResolver.AutoResolveViewModel="True"
             mc:Ignorable="d"
             d:DesignHeight="450"
             d:DesignWidth="800">
    <Grid>

    </Grid>
</UserControl>
```

This function can also be implemented in the background code using `MvvmHelpers.EnsureAutoResolveViewModelEnabled(FrameworkElement)`.

```cs
public static void EnsureAutoResolveViewModelEnabled(object control)
{
if (control is FrameworkElement view&&view.DataContext is null&&ViewModelResolver.GetAutoResolveViewModel(view) is null)
    ViewModelResolver.SetAutoResolveViewModel(view, true);
}
```

## Navigation

The framework supports page navigation functionality based on `ContentControl` and `UserControl`.

### Attached Properties

Complete the `MainWindow` navigation area registration through the attached property `rj:ViewZone.Name="MainContentZone"`:

When the `ContentControl` triggers the `Loaded` event, the framework will automatically register the control into the navigation service's area dictionary.

**Constraint Rules**: Each navigation area only supports hosting **a single** `UserControl` instance, and does not support hosting collection-type content.

```cs
<Window x:Class="WpfAppDemo.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:rj="http://rji.mvvm.wpf/views"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        mc:Ignorable="d"
        Title="MainWindow"
        Height="450"
        Width="800">
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="100" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>
        <Button Content="Change View" Height="30" Command="{Binding ChangeViewCommand}"/>
        <ContentControl Grid.Column="1"
                            rj:ViewZone.Name="MainContentZone" />
    </Grid>
</Window>
```

### Registering Services

In `App.xaml.cs`, register the `UserControl` needed for navigation to the IoC container for navigation.

It is recommended to use the extension method `AddViewToNavigation(string key = null)` to complete the registration:

- This method includes an optional `key` parameter;
- If no parameter value is manually specified, `key` will automatically use the full name of the control type (`Type.FullName`).

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using RJi.Mvvm.Wpf.IoC;
using System.Windows;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
            services.AddViewToNavigation<MyView1>();
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

### Switching Pages

In `MainWindowViewModel`, obtain the `INavigationService` navigation service instance through **dependency injection** and call the `NavigateTo("viewZoneName", "viewKey")` method to complete page switching

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Navigation;

namespace WpfAppDemo.ViewModels
{
    internal partial class MainWindowViewModel : ObservableObject
    {
        private readonly INavigationService _navigationService;

        public MainWindowViewModel(INavigationService navigationService)
        {
            _navigationService=navigationService;
        }

        [RelayCommand]
        private void ChangeView()
        {
            _navigationService.NavigateTo("MainContentZone", "MyView1");
        }
    }
}
```

### Navigation Parameters

`NavigateTo()` provides **overloaded versions** that support parameter passing between pages. The framework has a built-in `NavigationParameters` class specifically designed to carry and pass navigation parameters.

```cs
[RelayCommand]
     private void ChangeView()
     {
         var pars = new NavigationParameters();
         pars.Add("key1", "Hello,Richard");
         pars.Add("key2", 12);
         pars.Add("key3", new DateTime(2024, 6, 1));
         _navigationService.NavigateTo("MainContentZone", "MyView1", pars);
     }
```

### Receiving Parameters

Taking the navigation view `UserControl-MyView1` as an example, create the corresponding view model class **`MyView1ViewModel`** in the `ViewModels` folder. The framework will implement **automatic binding of view and view model** according to the naming rules.

The view model needs to inherit the **`INavigationLifecycle`** navigation lifecycle interface and implement the following core methods:

- `OnNavigationEnter()`: Triggered when the page **enters/loads**, can receive parameters passed by navigation.
- `OnNavigationExit()`: Triggered when the page **leaves/unloads**, can perform resource release, state saving, etc.
- `ShouldReuseInstance`: Configure view instance reuse strategy. Views are cached by default; when the property is `true`, cached instances are reused, when `false`, new instances are created for each navigation and new instances are not stored in the cache. **This property takes effect after `OnNavigationExit()`**.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Navigation;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject, INavigationLifecycle
    {
        public void OnNavigationEnter(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public void OnNavigationExit(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public bool ShouldReuseInstance(NavigationContext navigationContext)
        {
            return true;
        }
    }
}
```

### Navigation Interception

The interface `IConfirmNavigationRequest` inherits from `INavigationLifecycle` and additionally implements the `ConfirmNavigationRequest()` method for navigation interception confirmation.

- Execution timing: When navigation is triggered, **executed prior to `OnNavigationExit()`**
- Interception logic: Must call `continuationCallback(true)` to allow navigation to continue; passing `false` will directly cancel the current navigation.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Navigation;
using System.Windows;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject, IConfirmNavigationRequest
    {
        public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
        {
            var result = MessageBox.Show("go back？", "tip", MessageBoxButton.YesNo)==MessageBoxResult.Yes;
            if (!result) return;
            continuationCallback(true);
        }

        public void OnNavigationEnter(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public void OnNavigationExit(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public bool ShouldReuseInstance(NavigationContext navigationContext)
        {
            return true;
        }
    }
}
```

### Navigation History

The view model implements the `IHistoryRecordable` interface, and controls whether to add to the navigation history through `ShouldPersistInHistory()`; it is only recorded when returning `true`, not recorded by default, and needs to be explicitly implemented to enable.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Navigation;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject, INavigationLifecycle, IHistoryRecordable
    {

        public void OnNavigationEnter(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public void OnNavigationExit(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public bool ShouldPersistInHistory()
        {
            return true;
        }

        public bool ShouldReuseInstance(NavigationContext navigationContext)
        {
            return true;
        }
    }
}
```

### Navigation State/History Back

`NavigateTo()` provides **overloaded versions** that support **navigation state callback**, **parameter passing**, and **back/forward**, returning navigation execution results through callback functions.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Navigation;

namespace WpfAppDemo.ViewModels
{
    internal partial class MainWindowViewModel : ObservableObject
    {
        private readonly INavigationService _navigationService;
        private INavigationHistory? _navigationHistory;

        public MainWindowViewModel(INavigationService navigationService)
        {
            _navigationService=navigationService;
        }

        [RelayCommand]
        private void ChangeView()
        {
            var pars = new NavigationParameters();
            pars.Add("key1", "Hello,Richard");
            pars.Add("key2", 12);
            pars.Add("key3", new DateTime(2024, 6, 1));
            _navigationService.NavigateTo("MainContentZone", "MyView1", pars, result =>
            {
                var pars = result.Context.Parameters;
                if (result.Status==NavigationStatus.Success)
                {
                    _navigationHistory=result.Context.History;
                    _navigationHistory?.GoBack();
                    _navigationHistory?.GoForward();
                }
            });
        }
    }
}
```

## View Caching

`INavigationLifecycle.ShouldReuseInstance()`: Only configures the view cache reuse strategy, views are cached by default.

Therefore, the view model class can optionally implement the `IViewZoneLifetime` interface to solve some limitations caused by the default behavior that views are automatically cached when inheriting `INavigationLifecycle`, by implementing the `KeepAlive` property:

- When set to `false`: After leaving the view area, the view will be immediately removed from the cache;
- When set to `true`: The behavior is exactly the same as not implementing this interface.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Navigation;
using RJi.Mvvm.Wpf.ViewZone;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject, INavigationLifecycle, IViewZoneLifetime
    {
        public bool KeepAlive => false;

        public void OnNavigationEnter(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public void OnNavigationExit(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public bool ShouldReuseInstance(NavigationContext navigationContext)
        {
            return true;
        }
    }
}
```

## Dialog Windows

Use `UserControl` as the `Content` of `DialogWindow`.

### Attached Properties

**Dialog.WindowStyle**:
This property needs to be set in `UserControl`, and the style will be applied when `UserControl` is filled into `DialogWindow` to load the view; by default, only `SizeToContent="WidthAndHeight"` is configured, custom styles will override the default values, **but this style will be overridden by third-party UI libraries**. If the UI library has basic window controls, it will override this effect, so it is recommended to customize the dialog window.
**Dialog.WindowStartupLocation**:
Set the startup location of the dialog window. The configuration is automatically applied when the dialog loads the view, and the default value is `WindowStartupLocation=CenterOwner`.

```xml
<UserControl x:Class="WpfAppDemo.Views.MyDialogView1"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:rj="http://rji.mvvm.wpf/views"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             mc:Ignorable="d"
             d:DesignHeight="450"
             rj:Dialog.WindowStartupLocation="CenterOwner"
             d:DesignWidth="800">
    <rj:Dialog.WindowStyle>
        <Style TargetType="Window">
            <Setter Property="Background"
                    Value="Bisque" />
            <Setter Property="Width"
                    Value="400" />
            <Setter Property="Height"
                    Value="300" />
        </Style>
    </rj:Dialog.WindowStyle>
    <Grid>
        <TextBlock>this is dialogview1</TextBlock>
    </Grid>
</UserControl>
```

### Custom Dialog Window

The framework provides a default `DialogWindow` control, but if you need a custom style, you need to create a custom dialog window control. Create a new Window control, and the background code must inherit and implement `IDialogWindow`.

```cs
using RJi.Mvvm.Wpf.Dialogs;
using System.Windows;

namespace WpfAppDemo.Views
{

    public partial class MyDialogWindow : Window, IDialogWindow
    {
        public MyDialogWindow()
        {
            InitializeComponent();
        }

       public DialogResult? Result { get; set; }
    }
}
```

### Registering Services

In `App.xaml.cs`, register the `UserControl` needed for the dialog window to the IoC container for use in the dialog window. The framework provides a default `DialogWindow`, so you only need to register the content controls displayed in the window.

It is recommended to use the extension method `AddViewToDialog(string key = null)` to complete the registration:

- This method includes an optional `key` parameter;
- If no parameter value is manually specified, `key` will automatically use the full name of the control type (`Type.FullName`).

If you need a custom `DialogWindow`, it is recommended to use the extension method `AddDialogWindow(string key = null)` to complete the registration:

- This method includes an optional `key` parameter;
- If no parameter value is manually specified, `key` will automatically use the full name of the control type (`Type.FullName`).

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using RJi.Mvvm.Wpf.IoC;
using System.Windows;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
            services.AddViewToNavigation<MyView1>();
            services.AddViewToDialog<MyDialogView1>();
            services.AddDialogWindow<MyDialogWindow>();
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

When using the dialog window function, **the view model must implement the `IDialogLifecycle` interface**, otherwise the framework will throw an exception.

Taking the view `UserControl-MyDialogView1` as an example, create the corresponding view model **`MyDialogView1ViewModel`** in the `ViewModels` folder. The framework will automatically complete the **binding of the view and view model** according to the naming convention.

- `DialogWindowTitle`：The title bar text of the dialog window
- `RequestClose`：An event for the view model to actively close the window, which can carry return results
- `CanCloseDialog()`：Determines whether the window is allowed to close, returns `true` to allow closing, `false` to prohibit closing
- `OnDialogOpening()`：Triggered when the window opens, used to receive incoming parameters
- `OnDialogClosed()`：Triggered after the window closes, used for resource release and state cleanup

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyDialogView1ViewModel : ObservableObject, IDialogLifecycle
    {
        public string DialogWindowTitle { get; set; } = "Hello,Dialog";

        public event Action<DialogResult>? RequestClose;

        public bool CanCloseDialog()
        {
            return true;
        }

        public void OnDialogClosed()
        {
        }

        public void OnDialogOpening(DialogParameters parameters)
        {
        }
    }
}
```

### Opening Dialog Window

In `MyView1ViewModel`, obtain the `IDialogService` dialog window service instance through **dependency injection** and call the `Show/ShowDialog(dialogViewKey, dialogWindowKey=null)` method to open the dialog window.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject
    {
        private readonly IDialogService _dialogService;

        public MyView1ViewModel(IDialogService dialogService)
        {
            _dialogService=dialogService;
        }

        [RelayCommand]
        private void OpenDialog()
        {
            _dialogService.Show("MyDialogView1","MyDialogWindow");
            _dialogService.ShowDialog("MyDialogView1");
        }
    }
}
```

### Parameter Passing

Similar to navigation parameters, `DialogParameters` is provided, and `ShowDialog/Show()` has overloaded versions to pass parameters.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject
    {
        private readonly IDialogService _dialogService;

        public MyView1ViewModel(IDialogService dialogService)
        {
            _dialogService=dialogService;
        }

        [RelayCommand]
        private void OpenDialog()
        {
            var pars = new DialogParameters();
            pars.Add("key1", DateTime.Now);
            _dialogService.ShowDialog("MyDialogView1", pars);
            _dialogService.Show("MyDialogView1", pars);
        }
    }
}
```

### Receiving Parameters

`OnDialogOpening()` in the implementation of `IDialogLifecycle` is used to receive parameters.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyDialogView1ViewModel : ObservableObject, IDialogLifecycle
    {
        public string DialogWindowTitle { get; set; } = "Hello,Dialog";

        public event Action<DialogResult>? RequestClose;

        public bool CanCloseDialog()
        {
            return true;
        }

        public void OnDialogClosed()
        {
        }

        public void OnDialogOpening(DialogParameters parameters)
        {
            var pars = parameters;

        }
    }
}
```

### Dialog Results

`RequestClose` in the implementation of `IDialogLifecycle` is used to return dialog results and dialog parameters.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyDialogView1ViewModel : ObservableObject, IDialogLifecycle
    {
        public string DialogWindowTitle { get; set; } = "Hello,Dialog";

        public event Action<DialogResult>? RequestClose;

        public bool CanCloseDialog()
        {
            return true;
        }

        public void OnDialogClosed()
        {
        }

        public void OnDialogOpening(DialogParameters parameters)
        {
            var pars = parameters;

        }
        [RelayCommand]
        private void Ok()
        {
            RequestClose?.Invoke(new DialogResult(ButtonResult.Ok, new DialogParameters()));
        }

        [RelayCommand]
        private void No()
        {
            RequestClose?.Invoke(new DialogResult(ButtonResult.No));
        }
    }
}

```

Accept results and parameters in the overloads of `IDialogService.Show/ShowDialog()`.

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject
    {
        private readonly IDialogService _dialogService;

        public MyView1ViewModel(IDialogService dialogService)
        {
            _dialogService=dialogService;
        }

        [RelayCommand]
        private void OpenDialog()
        {
            var pars = new DialogParameters();
            pars.Add("key1", DateTime.Now);
            _dialogService.ShowDialog("MyDialogView1", pars);
            _dialogService.Show("MyDialogView1", pars, result =>
            {
                if (result.Result==ButtonResult.Ok)
                {
                    var pars = result.Parameters;
                }
            });
        }
    }
}
```

# Chinese Version

## RJi.Mvvm.Wpf

文档已重新编写，适配最新版本，适用于 1.2.7。

### 介绍

RJi.Mvvm.Wpf 是一款适用于 `.NET 6.0` 的轻量级 WPF MVVM 框架，基于 `CommunityToolkit.Mvvm` 构建。该框架提供类似 `Prism.Wpf` 的功能，同时集成 `Microsoft.Extensions.Hosting` 实现依赖注入管理，支持 `ContentControl` 和 `UserControl` 的导航，并实现导航参数传递、导航历史记录、导航确认、会话窗口以及视图缓存等功能，为小型 WPF 项目提供便捷的开发支持。

### 核心功能

- ViewModel 自动注册 ：项目启动时，框架会通过反射自动扫描项目中名为 ViewModels 的文件夹，将所有以 "ViewModel" 结尾的类注册到依赖注入容器中，默认以瞬态（Transient）模式注入。
- ViewModel 定位器：与 Prism.Wpf 类似，自动关联符合规则的上下文。
- 导航系统 ：
  - 视图区域：提供附加属性 `ViewZone.Name` 用于标记 `ContentControl` 控件，实现 `ContentControl` 与 `UserControl` 之间的无缝导航。
  - 导航接口 ：提供一系列可被 `ViewModel` 继承的 `INavigationService` 、`INavigationLifecycle` 、`IViewZoneLifetime`、`IHistoryRecordable` 、`INavigationHistory`、`IConfirmNavigationRequest` 等接口，支持导航参数传递、前进后退导航以及视图生命周期管理。。
- 会话窗口
  - 内置默认 `DialogWindow` 控件，作为标准会话窗口使用。
  - 会话接口：支持通过实现 `IDialogService` 自定义会话窗口，包含样式设置与生命周期管理。
- 依赖注入容器扩展方法：为 `IServiceCollection` 提供了便捷的扩展方法，包括 `AddViewToNavigation`、`AddViewToDialog`、`AddDialogWindow` ，简化导航和会话功能的配置。

### 设计目标

- **低侵入性**：遵循 " 约定优于配置 " 原则，将代码修改量降至最低。
- **高可扩展性**：支持依赖注入、自定义导航与对话框，以及完整的生命周期管理。
- **开发者友好**：自动实现 `ViewModel` 绑定，简化导航逻辑，显著提升 WPF 开发效率。
- **适用场景**：专为中小型 WPF 项目设计，可快速构建模块化 MVVM 应用。

### 开发路线图

| 功能模块 | 当前状态        | 优先级 | 目标版本 | 说明 / 优化方向                  |
| -------- | --------------- | ------ | -------- | -------------------------------- |
| 模块化   | ⏳ 规划中       |        |          | -                                |
| 持续优化 | ✅ 部分功能稳定 | P1     | V1.2.7   | 优化历史遗留问题，提升系统稳定性 |

**状态说明**：

🔧 开发中 | ⏳ 规划中 | 🐞 待优化 | ✅ 已稳定

- **优先级**：P0（最高）> P1 > P2

## 快速开始

新建 WPF 项目，在 App.xaml 中添加命名空间 `xmlns:rj="http://rji.mvvm.wpf/views"`，将根节点 `Application` 替换为 `rj:RJiApplication`，并移除默认的 `StartupUri` 属性。

```cs
<rj:RJiApplication x:Class="WpfAppDemo.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:WpfAppDemo"
             xmlns:rj="http://rji.mvvm.wpf/views">
    <rj:RJiApplication.Resources>

    </rj:RJiApplication.Resources>
</rj:RJiApplication>
```

在 `App.xaml.cs` 中引用命名空间 `RJi.Mvvm.Wpf`，并修改 App 继承 `RJiApplication`，重写 `ConfigureServices` 与 `CreateShell` 方法，在 `CreateShell` 中返回主窗口 `MainWindow` 作为应用启动主页面。

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using System.Windows;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

在项目中创建 `Views` 和 ` ViewModels` 两个文件夹，分别用于存放对应类。

文件夹内的类命名空间需分别包含 `.Views` / `.ViewModels`，并保持前缀一致，`ViewModel` 类必须以 `ViewModel` 结尾。

示例：Project.Views.MainWindow → Project.ViewModels.MainWindowViewModel

项目即可正常启动运行，`CreateShell()` 返回的 `MainWindow` 会自动关联对应的 `MainWindowViewModel` 作为 `DataContext`。

该自动关联仅在以下方法调用时生效：

- `CreateShell()`
- 导航服务：`NavigateTo()`/`SetInitialView()`
- 会话服务： `ShowDialog()/Show()`

## MVVM 支持

使用 `CommunityToolkit.Mvvm` 作为 Mvvm 支持包。

[MVVM 工具包简介 - Community Toolkits for .NET | Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/communitytoolkit/mvvm/)

## RJiApplication 基类

### 自动注册 ViewModel

该基类依托 `Microsoft.Extensions.Hosting` 中的依赖注入容器，在应用启动时通过反射**自动以瞬态（Transient）注册所有符合规范的 ViewModel**。

要求：位于 ViewModels 文件夹、命名空间包含 `.ViewModels`、类名以 `ViewModel` 结尾。

**重要提示**：

- 微软 DI 不支持运行时动态注册，所有服务需在应用启动前完成注册。
- ViewModel 自动注册时，会选用**参数最多的构造函数**进行实例化，并通过递归方式自动注册项目中可识别的所有参数类型为瞬态（Transient）。
- 自定义接口类型服务必须手动配置关联实现类型，否则会抛出异常。
- 框架内置接口无需手动注册。

代码示例:

App.xaml.cs

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using System.Windows;
using WpfAppDemo.Services;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
            services.AddSingleton<IMyService, MyService>(); // 自定义服务必须手动注册
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

MainWindowViewModel.cs

```cs
using CommunityToolkit.Mvvm.Messaging;
using RJi.Mvvm.Wpf.Navigation;
using WpfAppDemo.Services;

namespace WpfAppDemo.ViewModels
{
    internal class MainWindowViewModel
    {
        private readonly INavigationService _navigationService; //导航服务 - RJi.Mvvm.Wpf.Navigation
        private readonly IMessenger _messenger;//消息服务 -CommunityToolkit.Mvvm.Messaging
        private readonly IMyService _myService;//自定义服务

        public MainWindowViewModel(INavigationService navigationService, IMessenger messenger, IMyService myService)
        {
            _navigationService=navigationService;
            _messenger=messenger;
            _myService=myService;
        }
    }
}
```

### RJiApplication 的虚方法

**InitializeAsync**：应用初始化，配置 `Hosting`。

```cs
protected virtual async Task InitializeAsync()
{
    var hostBuilder = CreateAndConfigureHostBuilder();
    _host=hostBuilder.Build();
    Ioc.Default.ConfigureServices(_host.Services);
    await _host.StartAsync().ConfigureAwait(true);
    var shell = CreateShell();
    ArgumentNullException.ThrowIfNull(shell, nameof(shell));
    MvvmHelpers.EnsureAutoResolveViewModelEnabled(shell);
    InitializeShell(shell);
}

private IHostBuilder CreateAndConfigureHostBuilder()
{
    var hostBuilder = Host.CreateDefaultBuilder();
    hostBuilder.ConfigureServices(
        (context, services) =>
        {
            ConfigureDefaultServices(services);
            ConfigureServices(context, services);
        });
    ConfigureHostBuilder(hostBuilder);
    return hostBuilder;
}
```

**InitializeShell**：初始化启动页面。

```cs
protected virtual void InitializeShell(Window shell)
{
  MainWindow=shell;
}
```

**OnInitialized**：主窗口显示。

```cs
protected virtual void OnInitialized()
{
  MainWindow?.Show();
}
```

**ConfigureViewModelResolver**：配置 ViewModel 解析器，ViewModel 定位器将从依赖注入容器中解析实例；本框架基于 `CommunityToolkit.Mvvm.DependencyInjection.Ioc` 实现。

```cs
protected virtual void ConfigureViewModelResolver()
{
    ViewModelResolverProvider.ConfigureViewModelFactory(viewModelType => Ioc.Default.GetService(viewModelType)!);
}
```

**ConfigureDefaultServices**：配置 DI 容器默认服务，执行顺序较早。可在抽象方法 `ConfigureServices` 中注册后续服务（DI 容器会优先使用最后注册的服务）。

```cs
protected virtual void ConfigureDefaultServices(IServiceCollection services)
{
  services.AddMessenger()
          .AddDialogService()
          .AddNavigationService()
          .AddViewModels();
  }
```

**ConfigureHostBuilder**：默认为空，用于配置 HostBuilder 及自定义服务的钩子方法。

```cs
protected virtual void ConfigureHostBuilder(IHostBuilder hostBuilder)
{
}
```

### RJiApplication 的属性

`InitializeAsync()` 中已经配置了 `CommunityToolkit.Mvvm.DependencyInjection.Ioc` ,`CommunityToolkit.Mvvm` 提供了单例模式 `Ioc.Default` 用于获取容器。在 `RJiApplication` 中，也可通过 `ServiceProvider` 属性访问依赖注入容器。

```cs
 public IServiceProvider ServiceProvider
 {
     get { return Ioc.Default; }
 }
```

## Ioc

### 单例模式

命名空间 `CommunityToolkit.Mvvm.DependencyInjection` 中提供了单例模式 `Ioc.Default` 用于获取容器。其他服务需要在 App.xaml 中注册。

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using System.Windows;
using WpfAppDemo.Services;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
            services.AddSingleton<IMyService, MyService>(); // 自定义服务必须手动注册
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

### 信使

`CommunityToolkit.Mvvm` 的信使已注册至 IoC 容器。视图模型继承 `ObservableRecipient` 后，将自动获取 `Messenger` 属性；**容器注入实例、单例 `WeakReferenceMessenger.Default` 与该属性为同一实例**。

```cs
using CommunityToolkit.Mvvm.Messaging;
using RJi.Mvvm.Wpf.Navigation;
using WpfAppDemo.Services;
using CommunityToolkit.Mvvm.Messaging.Messages;

namespace WpfAppDemo.ViewModels
{
    internal class MainWindowViewModel: ObservableRecipient
    {
        private readonly IMessenger _messenger;

        public MainWindowViewModel(IMessenger messenger)
        {
            _messenger=messenger;
            var message = new ValueChangedMessage<string>("Hello,MainWindowViewModel");
            Messenger.Send(message);
            _messenger.Send(message);
            WeakReferenceMessenger.Default.Send(message);
        }
    }
}

```

## ViewModel 定位器

与 Prism.Wpf 类似，在 `CreateShell()`、导航服务 `NavigateTo()`/`SetInitialView()` 以及会话服务 `ShowDialog()`/`Show()` 方法执行时，尝试从 `Ioc` 容器中获取与 `View` 命名相同，且以 `ViewModel` 结尾的类作为该 `View` 的 `DataContext`。

### 附加属性

定位器功能的实现源于依赖属性 `rj:ViewModelResolver.AutoResolveViewModel="True"`,在 `CreateShell()`、导航服务 `NavigateTo()`/`SetInitialView()` 以及会话服务 `ShowDialog()`/`Show()` 中，会自动为控件添加该附加属性并设为 true。其他情况需要手动添加生效。

```cs
<UserControl x:Class="WpfAppDemo.Views.MyView1"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:rj="http://rji.mvvm.wpf/views"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             rj:ViewModelResolver.AutoResolveViewModel="True"
             mc:Ignorable="d"
             d:DesignHeight="450"
             d:DesignWidth="800">
    <Grid>

    </Grid>
</UserControl>
```

也可用在后台代码中用 `MvvmHelpers.EnsureAutoResolveViewModelEnabled(FrameworkElement)` 实现此功能。

```cs
public static void EnsureAutoResolveViewModelEnabled(object control)
{
if (control is FrameworkElement view&&view.DataContext is null&&ViewModelResolver.GetAutoResolveViewModel(view) is null)
    ViewModelResolver.SetAutoResolveViewModel(view, true);
}
```

## 导航

框架支持基于 `ContentControl` 和 `UserControl` 的页面导航功能。

### 附加属性

通过附加属性 `rj:ViewZone.Name="MainContentZone"` 完成 `MainWindow` 导航区域注册：

当 `ContentControl` 触发 `Loaded` 事件时，框架会自动将该控件注册到导航服务的区域字典中。

**约束规则**：每个导航区域仅支持承载**单个** `UserControl` 实例，不支持承载集合类型内容。

```cs
<Window x:Class="WpfAppDemo.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:rj="http://rji.mvvm.wpf/views"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        mc:Ignorable="d"
        Title="MainWindow"
        Height="450"
        Width="800">
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="100" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>
        <Button Content="Change View" Height="30" Command="{Binding ChangeViewCommand}"/>
        <ContentControl Grid.Column="1"
                            rj:ViewZone.Name="MainContentZone" />
    </Grid>
</Window>
```

### 注册服务

在 `App.xaml.cs` 中，将需要用于导航的 `UserControl` 注册到 IoC 容器，用于导航。

推荐使用扩展方法 `AddViewToNavigation(string key = null)` 完成注册：

- 该方法包含一个可选的 `key` 参数；
- 若不手动指定参数值，`key` 将自动使用控件类型的完整名称（`Type.FullName`）。

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using RJi.Mvvm.Wpf.IoC;
using System.Windows;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
            services.AddViewToNavigation<MyView1>();
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

### 切换页面

在 `MainWindowViewModel` 中通过**依赖注入**获取 `INavigationService` 导航服务实例，调用 `NavigateTo("viewZoneName", "viewKey")` 方法即可完成页面切换

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Navigation;

namespace WpfAppDemo.ViewModels
{
    internal partial class MainWindowViewModel : ObservableObject
    {
        private readonly INavigationService _navigationService;

        public MainWindowViewModel(INavigationService navigationService)
        {
            _navigationService=navigationService;
        }

        [RelayCommand]
        private void ChangeView()
        {
            _navigationService.NavigateTo("MainContentZone", "MyView1");
        }
    }
}
```

### 导航参数

`NavigateTo()` 提供**重载版本**支持页面间参数传递，框架内置 `NavigationParameters` 类专门用于承载、传递导航参数。

```cs
[RelayCommand]
     private void ChangeView()
     {
         var pars = new NavigationParameters();
         pars.Add("key1", "Hello,Richard");
         pars.Add("key2", 12);
         pars.Add("key3", new DateTime(2024, 6, 1));
         _navigationService.NavigateTo("MainContentZone", "MyView1", pars);
     }
```

### 接收参数

以导航视图 `UserControl-MyView1` 为例，在 `ViewModels` 文件夹中创建对应视图模型类 **`MyView1ViewModel`**，框架将按命名规则实现**视图与视图模型自动绑定**。

视图模型需继承 **`INavigationLifecycle`** 导航生命周期接口，并实现以下核心方法：

- `OnNavigationEnter()`：页面**进入 / 加载**时触发，可接收导航传递的参数。
- `OnNavigationExit()`：页面**离开 / 卸载**时触发，可执行资源释放、状态保存等操作。
- `ShouldReuseInstance`：配置视图实例复用策略。视图默认会被缓存；属性为 `true` 时复用缓存实例，为 `false` 时每次导航创建新实例且新实例不存入缓存。**该属性在 `OnNavigationExit()` 之后生效**。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Navigation;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject, INavigationLifecycle
    {
        public void OnNavigationEnter(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public void OnNavigationExit(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public bool ShouldReuseInstance(NavigationContext navigationContext)
        {
            return true;
        }
    }
}
```

### 导航拦截

接口 `IConfirmNavigationRequest` 继承自 `INavigationLifecycle`，额外实现 `ConfirmNavigationRequest()` 方法，用于导航拦截确认。

- 执行时机：导航触发时，**优先于 `OnNavigationExit()` 执行**
- 拦截逻辑：必须调用 `continuationCallback(true)` 才能允许继续导航；传入 `false` 则直接取消当前导航。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Navigation;
using System.Windows;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject, IConfirmNavigationRequest
    {
        public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
        {
            var result = MessageBox.Show("go back？", "tip", MessageBoxButton.YesNo)==MessageBoxResult.Yes;
            if (!result) return;
            continuationCallback(true);
        }

        public void OnNavigationEnter(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public void OnNavigationExit(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public bool ShouldReuseInstance(NavigationContext navigationContext)
        {
            return true;
        }
    }
}
```

### 导航历史

视图模型实现 `IHistoryRecordable` 接口，通过 `ShouldPersistInHistory()` 控制是否加入导航历史；仅返回 `true` 时记录，默认不记录，需显式实现启用。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Navigation;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject, INavigationLifecycle, IHistoryRecordable
    {

        public void OnNavigationEnter(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public void OnNavigationExit(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public bool ShouldPersistInHistory()
        {
            return true;
        }

        public bool ShouldReuseInstance(NavigationContext navigationContext)
        {
            return true;
        }
    }
}
```

### 导航状态/历史回退

`NavigateTo()` 提供**重载版本**，支持**导航状态回调**、**参数传递**和**回退/前进**，通过回调函数返回导航执行结果。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Navigation;

namespace WpfAppDemo.ViewModels
{
    internal partial class MainWindowViewModel : ObservableObject
    {
        private readonly INavigationService _navigationService;
        private INavigationHistory? _navigationHistory;

        public MainWindowViewModel(INavigationService navigationService)
        {
            _navigationService=navigationService;
        }

        [RelayCommand]
        private void ChangeView()
        {
            var pars = new NavigationParameters();
            pars.Add("key1", "Hello,Richard");
            pars.Add("key2", 12);
            pars.Add("key3", new DateTime(2024, 6, 1));
            _navigationService.NavigateTo("MainContentZone", "MyView1", pars, result =>
            {
                var pars = result.Context.Parameters;
                if (result.Status==NavigationStatus.Success)
                {
                    _navigationHistory=result.Context.History;
                    _navigationHistory?.GoBack();
                    _navigationHistory?.GoForward();
                }
            });
        }
    }
}
```

## 视图缓存

`INavigationLifecycle.ShouldReuseInstance()`：只是配置复用视图缓存策略，视图默认会被缓存。

因此视图模型类可选择性实现 `IViewZoneLifetime` 接口，用于解决继承 `INavigationLifecycle` 时视图会被自动缓存这一默认行为带来的部分限制，通过实现 `KeepAlive` 属性：

- 设为 `false` 时：离开视图区域后，视图将立即从缓存中移除；
- 设为 `true` 时：行为与未实现该接口完全一致。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Navigation;
using RJi.Mvvm.Wpf.ViewZone;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject, INavigationLifecycle, IViewZoneLifetime
    {
        public bool KeepAlive => false;

        public void OnNavigationEnter(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public void OnNavigationExit(NavigationContext navigationContext)
        {
            var pars = navigationContext.Parameters;
        }

        public bool ShouldReuseInstance(NavigationContext navigationContext)
        {
            return true;
        }
    }
}
```

## 会话窗口

将 `UserControl` 作为 `DialogWindow` 的 `Content`。

### 附加属性

**Dialog.WindowStyle**：
此属性需要在 `UserControl` 中设置，`UserControl` 填充到 `DialogWindow` 中加载视图时将应用该样式；默认仅配置 `SizeToContent="WidthAndHeight"`，自定义样式会覆盖默认值，**但该样式会被第三方 UI 库覆盖**。如果 UI 库带基础窗口控制会覆盖此效果，推荐自定义会话窗口。
**Dialog.WindowStartupLocation**：
设置对话框窗口的启动位置。对话框加载视图时自动应用该配置，默认值为 `WindowStartupLocation=CenterOwner`。

```xml
<UserControl x:Class="WpfAppDemo.Views.MyDialogView1"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:rj="http://rji.mvvm.wpf/views"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             mc:Ignorable="d"
             d:DesignHeight="450"
             rj:Dialog.WindowStartupLocation="CenterOwner"
             d:DesignWidth="800">
    <rj:Dialog.WindowStyle>
        <Style TargetType="Window">
            <Setter Property="Background"
                    Value="Bisque" />
            <Setter Property="Width"
                    Value="400" />
            <Setter Property="Height"
                    Value="300" />
        </Style>
    </rj:Dialog.WindowStyle>
    <Grid>
        <TextBlock>this is dialogview1</TextBlock>
    </Grid>
</UserControl>
```

### 自定义会话窗口

框架提供默认的 `DialogWindow` 控件，但如果需要自定义样式则需要创建自定义会话窗口控件。新建 Window 控件，后台代码必须继承实现 `IDialogWindow`。

```cs
using RJi.Mvvm.Wpf.Dialogs;
using System.Windows;

namespace WpfAppDemo.Views
{

    public partial class MyDialogWindow : Window, IDialogWindow
    {
        public MyDialogWindow()
        {
            InitializeComponent();
        }

       public DialogResult? Result { get; set; }
    }
}
```

### 注册服务

在 `App.xaml.cs` 中，将需要用于会话窗口中的视图 `UserControl` 注册到 IoC 容器，用于会话窗口。框架提供了默认的 `DialogWindow`，只需要注册窗口中显示的内容控件即可。

推荐使用扩展方法 `AddViewToDialog(string key = null)` 完成注册：

- 该方法包含一个可选的 `key` 参数；
- 若不手动指定参数值，`key` 将自动使用控件类型的完整名称（`Type.FullName`）。

如果需要自定义 `DialogWindow`，推荐使用扩展方法 `AddDialogWindow(string key = null)` 完成注册：

- 该方法包含一个可选的 `key` 参数；
- 若不手动指定参数值，`key` 将自动使用控件类型的完整名称（`Type.FullName`）。

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using RJi.Mvvm.Wpf;
using RJi.Mvvm.Wpf.IoC;
using System.Windows;
using WpfAppDemo.Views;

namespace WpfAppDemo
{
    public partial class App : RJiApplication
    {
        protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
        {
            services.AddSingleton<MainWindow>();
            services.AddViewToNavigation<MyView1>();
            services.AddViewToDialog<MyDialogView1>();
            services.AddDialogWindow<MyDialogWindow>();
        }

        protected override Window CreateShell()
        {
            return ServiceProvider.GetRequiredService<MainWindow>();
        }
    }
}
```

使用会话窗口功能时，**视图模型必须实现 `IDialogLifecycle` 接口**，否则框架将抛出异常。

以视图 `UserControl-MyDialogView1` 为例，在 `ViewModels` 文件夹中创建对应视图模型 **`MyDialogView1ViewModel`**，框架将按照命名约定自动完成**视图与视图模型的绑定**。

- `DialogWindowTitle`：会话窗口的标题栏文本
- `RequestClose`：视图模型主动关闭窗口的事件，可携带返回结果
- `CanCloseDialog()`：判断窗口是否允许关闭，返回 `true` 可关闭，`false` 禁止关闭
- `OnDialogOpening()`：窗口打开时触发，用于接收传入的参数
- `OnDialogClosed()`：窗口关闭后触发，用于资源释放、状态清理

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyDialogView1ViewModel : ObservableObject, IDialogLifecycle
    {
        public string DialogWindowTitle { get; set; } = "Hello,Dialog";

        public event Action<DialogResult>? RequestClose;

        public bool CanCloseDialog()
        {
            return true;
        }

        public void OnDialogClosed()
        {
        }

        public void OnDialogOpening(DialogParameters parameters)
        {
        }
    }
}
```

### 打开会话窗体

在 `MyView1ViewModel` 中通过**依赖注入**获取 `IDialogService` 会话窗口服务实例，调用 `Show/ShowDialog(dialogViewKey, dialogWindowKey=null)` 方法即可打开会话窗体。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject
    {
        private readonly IDialogService _dialogService;

        public MyView1ViewModel(IDialogService dialogService)
        {
            _dialogService=dialogService;
        }

        [RelayCommand]
        private void OpenDialog()
        {
            _dialogService.Show("MyDialogView1","MyDialogWindow");
            _dialogService.ShowDialog("MyDialogView1");
        }
    }
}
```

### 参数传递

与导航参数雷同，提供 `DialogParameters`,`ShowDialog/Show()` 重载传递参数。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject
    {
        private readonly IDialogService _dialogService;

        public MyView1ViewModel(IDialogService dialogService)
        {
            _dialogService=dialogService;
        }

        [RelayCommand]
        private void OpenDialog()
        {
            var pars = new DialogParameters();
            pars.Add("key1", DateTime.Now);
            _dialogService.ShowDialog("MyDialogView1", pars);
            _dialogService.Show("MyDialogView1", pars);
        }
    }
}
```

### 接收参数

`IDialogLifecycle` 的实现中 `OnDialogOpening()` 用于接收参数。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyDialogView1ViewModel : ObservableObject, IDialogLifecycle
    {
        public string DialogWindowTitle { get; set; } = "Hello,Dialog";

        public event Action<DialogResult>? RequestClose;

        public bool CanCloseDialog()
        {
            return true;
        }

        public void OnDialogClosed()
        {
        }

        public void OnDialogOpening(DialogParameters parameters)
        {
            var pars = parameters;

        }
    }
}
```

### 会话结果

`IDialogLifecycle` 的实现中 `RequestClose` 用于传回会话结果和会话参数。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyDialogView1ViewModel : ObservableObject, IDialogLifecycle
    {
        public string DialogWindowTitle { get; set; } = "Hello,Dialog";

        public event Action<DialogResult>? RequestClose;

        public bool CanCloseDialog()
        {
            return true;
        }

        public void OnDialogClosed()
        {
        }

        public void OnDialogOpening(DialogParameters parameters)
        {
            var pars = parameters;

        }
        [RelayCommand]
        private void Ok()
        {
            RequestClose?.Invoke(new DialogResult(ButtonResult.Ok, new DialogParameters()));
        }

        [RelayCommand]
        private void No()
        {
            RequestClose?.Invoke(new DialogResult(ButtonResult.No));
        }
    }
}
}

```

在 `IDialogService.Show/ShowDialog()` 的重载中接受结果和参数。

```cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using RJi.Mvvm.Wpf.Dialogs;

namespace WpfAppDemo.ViewModels
{
    internal partial class MyView1ViewModel : ObservableObject
    {
        private readonly IDialogService _dialogService;

        public MyView1ViewModel(IDialogService dialogService)
        {
            _dialogService=dialogService;
        }

        [RelayCommand]
        private void OpenDialog()
        {
            var pars = new DialogParameters();
            pars.Add("key1", DateTime.Now);
            _dialogService.ShowDialog("MyDialogView1", pars);
            _dialogService.Show("MyDialogView1", pars, result =>
            {
                if (result.Result==ButtonResult.Ok)
                {
                    var pars = result.Parameters;
                }
            });
        }
    }
}
```
