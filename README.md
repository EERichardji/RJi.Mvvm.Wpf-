## RJi.Mvvm.Wpf

[简体中文](README.zh.md)

The documentation has been rewritten to adapt to the latest version, applicable for 1.2.7.

[简体中文](Docs/README.zh.md)

The documentation has been rewritten to adapt to the latest version, applicable for 2.0.0.
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
