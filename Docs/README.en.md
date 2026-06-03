# RJi.Mvvm.Wpf

A lightweight WPF MVVM framework for `.NET 8.0` and `.NET Framework 4.8`, built on `CommunityToolkit.Mvvm` and `Microsoft.Extensions.DependencyInjection`.

> **⚠️ .NET Framework 4.8 Requires SDK-Style Project**
>
> **All source generators** described in this document — including CT.Mvvm's `[ObservableProperty]`, `[RelayCommand]` and the framework's own `[NavigationTarget]`, `[DialogTarget]`, DI attributes — require **C# 8.0+ / LangVersion latest** and NuGet's automatic analyzer loading. The legacy `packages.config` + old csproj format does not support either. You must manually convert to SDK-style.
>
> **Steps:**
>
> 1. Locate the `.csproj` file, right-click → **Open with** → **edit the .csproj**, **delete all content**
> 2. **Paste the following** to replace the original content:
>
> ```xml
> <Project Sdk="Microsoft.NET.Sdk">
>   <PropertyGroup>
>     <OutputType>WinExe</OutputType>
>     <TargetFramework>net48</TargetFramework>
>     <LangVersion>latest</LangVersion>
>     <Nullable>enable</Nullable>
>     <UseWPF>true</UseWPF>
>     <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
>   </PropertyGroup>
>   <ItemGroup>
>     <PackageReference Include="RJi.Mvvm.Wpf" Version="..." />
>   </ItemGroup>
> </Project>
> ```
>
> 3. **Delete `packages.config`** if present
> 4. Rebuild the solution
>
> > .NET 8.0 projects are unaffected — SDK-style is the default format.

## Core Features

- **Zero Reflection**: All View-ViewModel bindings are registered explicitly via generic keyed methods at startup.
- **Navigation System**: View zones, lifecycle management, history recording, and view caching.
- **Dialog Windows**: Built-in `DialogWindow` with lifecycle hooks and custom window support.
- **MVVM Integration**: Built on `CommunityToolkit.Mvvm` with built-in Messenger support.
- **Source Generators**: CT.Mvvm source generators (`[ObservableProperty]`, `[RelayCommand]`, etc.) plus framework-level generators (`[NavigationTarget]`, `[DialogTarget]`, DI attributes).

## Quick Start

### Reference Namespaces

**Use RJiApplication as root element in App.xaml:**

```xml
<rj:RJiApplication x:Class="MyApp.App"
                   xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                   xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                   xmlns:rj="http://rji.mvvm.wpf/views"
                   xmlns:local="clr-namespace:MyApp">

    <rj:RJiApplication.Resources>
        <!-- Application resources -->
    </rj:RJiApplication.Resources>
</rj:RJiApplication>
```

> **Important**: You **must remove the `StartupUri` property** from App.xaml. Otherwise, two windows will be created at startup (one from `StartupUri` and one from `CreateShell()`). The framework creates the main window via the `CreateShell()` method.

**Namespace Notes:**

The framework has mapped all related CLR namespaces to a single XML namespace via `AssemblyInfo.cs`, so you only need to reference it once in XAML:

```xml
xmlns:rj="http://rji.mvvm.wpf/views"
```

**Required using statements in C# code:**

```csharp
using RJi.Mvvm.Wpf;              // Core application class RJiApplication
using RJi.Mvvm.Wpf.IoC;          // Service registration extension methods
using RJi.Mvvm.Wpf.Navigation;   // Navigation interfaces and classes
using RJi.Mvvm.Wpf.Dialogs;      // Dialog interfaces and classes
```

## Create Application

```csharp
using RJi.Mvvm.Wpf;
using RJi.Mvvm.Wpf.IoC;
using Microsoft.Extensions.DependencyInjection;

public class App : RJiApplication
{
    protected override void ConfigureServices(IServiceCollection services)
    {
        // Register navigation views with their ViewModels
        services.AddViewToNavigation<HomeView, HomeViewModel>();
        services.AddViewToNavigation<SettingsView, SettingsViewModel>();

        // Register dialog views
        services.AddViewToDialog<ConfirmDialogView, ConfirmDialogViewModel>();
        services.AddViewToDialog<AboutDialogView, AboutDialogViewModel>();

        // Register custom dialog window (optional)
        services.AddDialogWindow<CustomDialogWindow>();
    }

    /// <summary>
    /// Creates the main application window (shell)
    /// </summary>
    /// <returns>A Window instance to use as the application's main window</returns>
    protected override Window CreateShell() => new MainWindow();
}
```

### Default Container Services

The framework is built on [`Microsoft.Extensions.DependencyInjection`](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/usage) for its dependency injection container. The following services are automatically registered during `RJiApplication` initialization:

| Service Interface      | Description                                         |
| ---------------------- | --------------------------------------------------- |
| `INavigationService`   | Navigation service (Singleton)                      |
| `IDialogService`       | Dialog service (Singleton)                          |
| `IDispatcherMessenger` | CommunityToolkit.Mvvm Messenger with UI thread dispatch + DispatchMode (Singleton) |
| `IUIDispatcher`        | UI thread dispatcher wrapper (Singleton)            |
| `IServiceProvider`     | Dependency injection container                      |

> **Note**: `IDispatcherMessenger` is pre-registered (inherits `IMessenger`) and can be used via constructor injection. See [Messenger with UI Thread Dispatch](#messenger-with-ui-thread-dispatch) for details on automatic UI thread dispatching.

### CommunityToolkit.Mvvm Usage

The framework is built on [`CommunityToolkit.Mvvm`](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/). Key patterns:

> **Prerequisite**: For net48 targets, configure the SDK-style project as shown at the top of this document; otherwise the CT.Mvvm source generators below won't work.

```csharp
public partial class HomeViewModel : ObservableObject
{
    // ObservableProperty → auto-generates INotifyPropertyChanged
    [ObservableProperty]
    private string _title = "Home";

    partial void OnTitleChanged(string value) => /* react to change */;

    // [RelayCommand] → auto-generates ICommand property (MethodName + Command)
    [RelayCommand]
    private void Navigate() { }                         // no parameter

    [RelayCommand]
    private void Open(string path) { }                  // with parameter

    [RelayCommand(CanExecute = nameof(CanSave))]
    private void Save() { }                             // with CanExecute

    private bool CanSave => true;
}

// Without source generator — no `partial` keyword, no partial methods:
public class ManualViewModel : ObservableObject
{
    private string _title = "";
    public string Title
    {
        get => _title;
        set => SetProperty(ref _title, value);
    }

    public ICommand NavigateCommand { get; }
    public ICommand OpenCommand { get; }
    public ICommand SaveCommand { get; }

    public ManualViewModel()
    {
        NavigateCommand = new RelayCommand(OnNavigate);
        OpenCommand = new RelayCommand<string>(OnOpen);
        SaveCommand = new RelayCommand(OnSave, CanSave);
    }

    private void OnNavigate() { }
    private void OnOpen(string path) { }
    private void OnSave() { }
    private bool CanSave() => true;
}
```

**Messenger** (with automatic UI thread dispatch):

> **⚠️ Always use the injected `IDispatcherMessenger`** (not `WeakReferenceMessenger.Default`), otherwise handlers will NOT be dispatched to the UI thread. `IDispatcherMessenger` also provides a matching `Send<TMessage>()` extension method to ensure consistent token behavior and correct message delivery.

The `IDispatcherMessenger` wraps `WeakReferenceMessenger.Default`, and automatically dispatches the handler to the UI thread when sending from a background thread — no manual `Dispatcher.Invoke` needed.

```csharp
public class DataViewModel
{
    private readonly IDispatcherMessenger _messenger;
    public DataViewModel(IDispatcherMessenger messenger) => _messenger = messenger;

    // Background thread sends → handler executes on UI thread
    public void OnDataReceived(float value) =>
        _messenger.Send(new DataUpdatedMessage(value));
}

// Receiver — safe to update bindings
public class DisplayViewModel
{
    public DisplayViewModel(IDispatcherMessenger messenger) =>
        messenger.Register<DataUpdatedMessage>(this, (r, m) =>
        {
            DataValue = m.Value; // no Dispatcher.Invoke needed
        });
}

public class DataUpdatedMessage(float value)
{
    public float Value { get; } = value;
}
```

**Specify dispatch mode**: control whether the handler is dispatched to the UI thread via `DispatchMode`:

```csharp
// Default: dispatch to UI thread (same as above)
messenger.Register<Msg>(this, handler);
messenger.Register<Msg>(this, handler, DispatchMode.UIThread);

// Publisher thread: handler runs on the same thread as Send — no dispatch, no try-catch
messenger.Register<Msg>(this, handler, DispatchMode.PublisherThread);
```

> `DispatchMode.PublisherThread` is useful for logging, cache updates, or other non-UI work — the handler runs inline with no dispatch overhead.

> **Manual dispatch**: `IUIDispatcher` is also registered in the container, wrapping `Application.Current.Dispatcher` for when you need it.

> **Full docs**: [CommunityToolkit.Mvvm](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/)

### Define View Zone

```xml
<!-- MainWindow.xaml -->
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:rj="http://rji.mvvm.wpf/views">

    <Grid>
        <ContentControl rj:ViewZone.Name="Main" />
        <ContentControl rj:ViewZone.Name="Sidebar" />
    </Grid>
</Window>
```

## Navigation API

### Basic Navigation

```csharp
// Get navigation service via DI
INavigationService navigationService = serviceProvider.GetRequiredService<INavigationService>();

// Navigate to view with callback
navigationService.NavigateTo(
    viewZoneName: "Main",
    viewKey: "Home",
    parameters: new NavigationParameters { { "userId", 123 } },
    navigationCallback: result => {
        if (result.Status == NavigationStatus.Success) {
            // Navigation succeeded
        } else if (result.Status == NavigationStatus.Canceled) {
            // User canceled navigation
        }
    });

// Set initial view (skips confirmation and history)
navigationService.SetInitialView("Main", "Home", null);
```

### Navigation Context and Parameters

```csharp
// Create navigation parameters
var parameters = new NavigationParameters {
    { "id", 1 },
    { "name", "Test" }
};

// Or from query string
var paramsFromQuery = new NavigationParameters("id=1&name=Test");

// Retrieve parameters in ViewModel
public void OnNavigationEnter(NavigationContext context) {
    int? userId = context.Parameters?.GetValue<int>("userId");
    string? name = context.Parameters?.GetValue<string>("name");
}
```

### Navigation Lifecycle

```csharp
public class HomeViewModel : ObservableObject, INavigationLifecycle
{
    // Keep view instance alive after navigation (default: false)
    public bool KeepAlive => true;

    // Check if existing instance should be reused
    public bool ShouldReuseInstance(NavigationContext context) => true;

    // Called when entering the view
    public void OnNavigationEnter(NavigationContext context) {
        // Initialize data
    }

    // Called when leaving the view
    public void OnNavigationExit(NavigationContext context) {
        // Cleanup resources
    }
}
```

> **Tip**: If your ViewModel implements `IDisposable`, the framework will automatically call `Dispose()` when the view is evicted from the navigation cache (e.g., `KeepAlive=false` on navigation away, LRU cache full, or ViewZone unloaded). This is useful for releasing messenger subscriptions, timers, and other managed resources:

> ```csharp
> public class MyViewModel : ObservableObject, INavigationLifecycle, IDisposable
> {
>     public void Dispose()
>     {
>         WeakReferenceMessenger.Default.UnregisterAll(this);
>     }
> }
> ```

> **Note**: The callback pattern supports async confirmation scenarios (e.g., dialog-based confirmation). If the View does not implement `IConfirmNavigationRequest`, navigation proceeds automatically.

### Navigation History

Implement `IHistoryRecordable` to control whether a view is recorded in navigation history:

```csharp
public class MyViewModel : ObservableObject, IHistoryRecordable
{
    /// <summary>
    /// true means this navigation will be recorded (supports GoBack/GoForward);
    /// false means skip history recording.
    /// </summary>
    public bool RecordInHistory => true;
}
```

After navigation completes, obtain `INavigationHistory` from `NavigationResult` for back/forward operations:

```csharp
navigationService.NavigateTo("Main", "Views.MyView", parameters, result =>
{
    if (result.Status == NavigationStatus.SuccessWithHistory)
    {
        var history = result.Context.History;
        if (history != null)
        {
            // Back/Forward navigation
            history.GoBack();
            history.GoForward();

            // Check availability
            bool canBack = history.CanGoBack;
            bool canForward = history.CanGoForward;
        }
    }
});
```

## Dialog API

### Show Dialog

```csharp
IDialogService dialogService = serviceProvider.GetRequiredService<IDialogService>();

// Show non-modal dialog
dialogService.Show(
    viewKey: "Settings",
    parameters: new DialogParameters { { "config", configData } },
    dialogCallback: result => {
        if (result.Result == ButtonResult.OK) {
            // Handle OK
        }
    });

// Show modal dialog (blocks UI)
dialogService.ShowDialog(
    viewKey: "Confirm",
    parameters: new DialogParameters { { "message", "Are you sure?" } },
    dialogCallback: result => {
        if (result.Result == ButtonResult.Yes) {
            // User confirmed
        }
    });
```

### Dialog Lifecycle

> **Important**: Dialog ViewModels **must implement `IDialogLifecycle` interface**, otherwise a `DialogLifecycleException` will be thrown when showing the dialog.

```csharp
public class ConfirmDialogViewModel : ObservableObject, IDialogLifecycle
{
    // Event to request dialog close
    public event Action<DialogResult> RequestClose = delegate { };

    // Check if dialog can be closed
    public bool CanCloseDialog() => true;

    // Called when dialog is opening
    public void OnDialogOpening(DialogParameters parameters) {
        // Initialize with parameters
        Message = parameters.GetValue<string>("message");
    }

    // Called after dialog is closed
    public void OnDialogClosed() {
        // Cleanup
    }

    // Close dialog programmatically
    private void CloseWithResult(ButtonResult result) {
        RequestClose.Invoke(new DialogResult(result));
    }
}
```

### Dialog Window Attached Properties

The framework provides `Dialog` attached properties, **set on the dialog content view (UserControl)**, for configuring dialog window behavior:

```xml
<UserControl x:Class="MyApp.Dialogs.ConfirmDialogView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:rj="http://rji.mvvm.wpf/views"
             rj:Dialog.WindowStartupLocation="CenterOwner">

    <!-- Define window style inline -->
    <rj:Dialog.WindowStyle>
        <Style TargetType="Window">
            <Setter Property="Background" Value="Bisque" />
            <Setter Property="Width" Value="400" />
            <Setter Property="Height" Value="250" />
        </Style>
    </rj:Dialog.WindowStyle>

    <StackPanel Margin="20">
        <TextBlock FontSize="20" Text="Dialog Content" />
        <StackPanel Orientation="Horizontal" HorizontalAlignment="Center">
            <Button Content="OK" Command="{Binding OkCommand}" />
            <Button Content="Cancel" Command="{Binding CancelCommand}" />
        </StackPanel>
    </StackPanel>
</UserControl>
```

**Attached Properties:**

| Property                          | Type                    | Description                                               |
| --------------------------------- | ----------------------- | --------------------------------------------------------- |
| `rj:Dialog.WindowStyle`           | `Style`                 | Sets the dialog window style (supports inline definition) |
| `rj:Dialog.WindowStartupLocation` | `WindowStartupLocation` | Sets window startup position (default: CenterOwner)       |

**How It Works:**

When `DialogService` shows a dialog, it reads these attached property values from the UserControl and then **explicitly sets** them on the dialog window object. This means:

- Properties work with the **default dialog window**
- Properties also override default settings when using **custom dialog windows**

**Third-Party Library Compatibility:**

Attached properties use standard WPF DependencyProperty mechanism and are theoretically compatible with all WPF-based UI libraries, including the following free and open-source libraries:

- MaterialDesignThemes
- MahApps.Metro
- HandyControl

> **Note**: Testing is recommended before production use.

### Custom Dialog Window

The framework **provides a default `DialogWindow` implementation** that works out of the box. Custom dialog windows are only needed when you require special styling or behavior.

> **Note**: Custom dialog windows **must inherit from `Window` and implement `IDialogWindow` interface** to be registered and used.

#### Step 1: Create Custom Window

```csharp
// CustomDialogWindow.xaml.cs
public partial class CustomDialogWindow : Window, IDialogWindow
{
    public CustomDialogWindow()
    {
        InitializeComponent();
    }

    // Must implement IDialogWindow interface
    public DialogResult? Result { get; set; }
}
```

#### Step 2: Register Custom Window

```csharp
// In ConfigureServices  (requires using RJi.Mvvm.Wpf.IoC;)
protected override void ConfigureServices(IServiceCollection services)
{
    // Register default custom dialog window
    services.AddDialogWindow<CustomDialogWindow>();

    // Or register multiple window types with keys
    services.AddDialogWindow<CustomDialogWindow>("CustomKey");
    services.AddDialogWindow<AnotherDialogWindow>("AnotherKey");
}
```

#### Step 3: Show Dialog with Specific Window

```csharp
// Use default window
dialogService.ShowDialog("Confirm", parameters, callback);

// Use specific window by key
dialogService.ShowDialog(
    viewKey: "Settings",
    parameters: parameters,
    dialogCallback: callback,
    windowKey: "CustomKey"); // Specify which window to use
```

### Button Results

```csharp
// Common button results
public enum ButtonResult {
    None,    // Default, no result
    OK,      // OK button clicked
    Cancel,  // Cancel button clicked
    Yes,     // Yes button clicked
    No       // No button clicked
}
```

### Dialog Result

```csharp
public class DialogResult {
    public ButtonResult Result { get; }
    public DialogParameters? Parameters { get; }
}
```

### Navigation Result

```csharp
public class NavigationResult {
    public NavigationStatus Status { get; set; }
    public NavigationContext Context { get; }
    public Exception? Exception { get; set; }
    public string? ErrorMessage { get; set; }
}

public enum NavigationStatus {
    Success,              // Navigated (no history)
    SuccessWithHistory,   // Navigated (recorded in history)
    ViewNotFound,
    ViewZoneNotFound,
    Canceled,
    InvalidContentHost,
    ViewInitializationFailed
}
```

## Exception Handling

### Custom Exceptions

```csharp
try {
    navigationService.NavigateTo("Main", "NonExistentView", null, null);
}
catch (ViewNotFoundException ex) {
    // View not registered
    Console.WriteLine($"View '{ex.ViewKey}' not found");
}
catch (ViewZoneNotFoundException ex) {
    // View zone not found
    Console.WriteLine($"View zone '{ex.ViewZoneName}' not found");
}
catch (NavigationCanceledException) {
    // User canceled navigation
}
catch (NavigationException ex) {
    // Generic navigation error
    Console.WriteLine($"Navigation failed: {ex.Message}");
}

// Dialog exceptions
try {
    dialogService.ShowDialog("MissingDialog", null, null);
}
catch (DialogViewNotFoundException ex) {
    // Dialog view not registered
    Console.WriteLine($"Dialog view '{ex.ViewKey}' not registered");
}
catch (DialogWindowNotFoundException ex) {
    // Custom dialog window not registered
    Console.WriteLine($"Dialog window '{ex.WindowKey}' not registered");
}
catch (DialogWindowContentException ex) {
    // View type is Window, cannot be used as dialog content
    Console.WriteLine($"Invalid dialog content: {ex.ViewKey}");
}
catch (DialogViewTypeNotSupportedException ex) {
    // View type is not FrameworkElement
    Console.WriteLine($"Unsupported view type: {ex.ViewType}");
}
catch (DialogLifecycleException ex) {
    // ViewModel does not implement IDialogLifecycle interface
    Console.WriteLine($"ViewModel missing IDialogLifecycle: {ex.DataContextType}");
}
catch (DialogException ex) {
    // Generic dialog error
    Console.WriteLine($"Dialog operation failed: {ex.Message}");
}
```

## Registration Extensions

The framework provides three core extension methods (namespace `RJi.Mvvm.Wpf.IoC`) for registering navigation views, dialog views, and custom dialog windows:

```csharp
public static class IServiceCollectionExtension
{
    // Register navigation view with its ViewModel
    public static IServiceCollection AddViewToNavigation<TView, TViewModel>(
        this IServiceCollection services, string? key = null)
        where TView : UserControl where TViewModel : class;

    // Register dialog view with its ViewModel
    public static IServiceCollection AddViewToDialog<TView, TViewModel>(
        this IServiceCollection services, string? key = null)
        where TView : UserControl where TViewModel : class, IDialogLifecycle;

    // Register custom dialog window
    public static IServiceCollection AddDialogWindow<TDialogWindow>(
        this IServiceCollection services, string? key = null)
        where TDialogWindow : Window, IDialogWindow;
}
```

### Registration Notes

> **Note**:
>
> - Navigation view registration only supports `UserControl` and `ContentControl` types. Other control types (such as `Window`, `Page`, etc.) are not supported as navigation views. Dialog views also only support `UserControl` and `ContentControl`.
> - **Generic Constraints**: `AddViewToNavigation<TView, TViewModel>` requires `TView : UserControl` and `TViewModel : class`; `AddViewToDialog<TView, TViewModel>` requires `TView : UserControl` and `TViewModel : class, IDialogLifecycle`; `AddDialogWindow<TDialogWindow>` requires `TDialogWindow : Window, IDialogWindow`.
> - **Key Conflict Handling**: When registering a **different** view type with the same key, an `InvalidOperationException` is thrown. Registering the same view type again is silently ignored.
> - **DataContext Priority**: If a view already has a `DataContext` (e.g., set manually or via XAML binding), the registered ViewModel will not be used, supporting View-First patterns.
> - **Default Lifetime**: Views registered via extension methods are **Transient** by default. For other lifetimes (Singleton, Scoped), simply register them manually in the container — **registration order does not matter**.

## Source Generator

The framework includes a Roslyn source generator that eliminates manual registration boilerplate.

> **Prerequisite**: For net48 targets, configure the SDK-style project as shown at the top of this document; otherwise the framework source generators below won't work.

### Usage

Replace manual registration with a single call in `ConfigureServices`:

```csharp
protected override void ConfigureServices(IServiceCollection services)
{
    services.AddGeneratedRegistrations();  // requires using RJi.Mvvm.Wpf.IoC;
}
```

### Attributes

#### View Registration

| Attribute | Parameters | Default |
|-----------|-----------|---------|
| `[NavigationTarget]` | `viewType` (optional): explicit View type<br>`Key` (optional): custom registration key | `viewType` = found by convention<br>`Key` = `{Namespace.LastSegment}.{ViewType.Name}` |
| `[DialogTarget]` | `viewType` (optional): explicit View type<br>`Key` (optional): custom registration key | same as above |
| `[DialogWindow]` | `Key` (optional): custom registration key | `Key` = `{Namespace.LastSegment}.{WindowType.Name}` |

**Parameter variations:**
```csharp
[DialogTarget]                                   // viewType=convention, Key=default
[DialogTarget(typeof(MyDialogView))]             // viewType=explicit,  Key=default
[DialogTarget(Key = "custom")]                   // viewType=convention, Key=custom
[DialogTarget(typeof(MyDialogView), Key = "custom")] // both explicit
```

**Naming Convention**: When view type is not specified, the generator searches by convention — strip `ViewModel` suffix and look for the matching class in the `.Views` namespace. e.g., `ViewModels.MyView1ViewModel` → `Views.MyView1`.

**Source — your code:**
```csharp
[NavigationTarget]
public partial class MyView1ViewModel { }

[NavigationTarget(Key = "home")]
public partial class MyView2ViewModel { }

[NavigationTarget(typeof(MyView3))]
public partial class MyView3ViewModel { }

[DialogTarget]
public partial class MyDlg1ViewModel : IDialogLifecycle { }

[DialogTarget(Key = "edit")]
public partial class MyDlg2ViewModel : IDialogLifecycle { }

[DialogWindow]
public partial class MyDialogWindow { }

[DialogWindow(Key = "popup")]
public partial class MyPopupWindow { }
```

**Generated equivalent (inside `AddGeneratedRegistrations`):**
```csharp
services.AddViewToNavigation<Views.MyView1, ViewModels.MyView1ViewModel>();
services.AddViewToNavigation<Views.MyView2, ViewModels.MyView2ViewModel>("home");
services.AddViewToNavigation<Views.MyView3, ViewModels.MyView3ViewModel>();
services.AddViewToDialog<Views.MyDlg1, ViewModels.MyDlg1ViewModel>();
services.AddViewToDialog<Views.MyDlg2, ViewModels.MyDlg2ViewModel>("edit");
services.AddDialogWindow<Views.MyDialogWindow>();
services.AddDialogWindow<Views.MyPopupWindow>("popup");
```

#### DI Registration

| Attribute | Parameters | Default |
|-----------|-----------|---------|
| `[AddSingleton]` | `serviceType` (optional): interface type | `serviceType` = class itself |
| `[AddSingleton(typeof(IService))]` | `serviceType` + `implementationType` (optional) | `implementationType` = class itself |
| `[AddScoped]` / `[AddTransient]` | same pattern | — |
| `[TryAddSingleton]` | same as `[AddSingleton]`, skips if registered | — |
| `[AddKeyedSingleton("key")]` | `serviceType` (optional) + `key` | `serviceType` = class itself |
| `[AddKeyedTransient("key")]` | same pattern | — |

```csharp
[AddSingleton(typeof(IMyService))]
internal class MyService : IMyService { }

[AddKeyedSingleton(typeof(IMyService), "backup")]
internal class MyBackupService : IMyService { }

[AddSingleton]
internal partial class MainWindowViewModel : ObservableObject { }
```

**Generated equivalent (inside `AddGeneratedRegistrations`):**
```csharp
services.AddSingleton<IMyService, MyService>();
services.AddKeyedSingleton<IMyService, MyBackupService>("backup");
services.AddSingleton<MainWindowViewModel>();
```

All 12 DI attributes are available: `Add`/`TryAdd` variants for `Singleton`/`Scoped`/`Transient`, plus `AddKeyed`/`TryAddKeyed` variants.

### ViewModel Locator

Mark a `partial class` with `[ViewModelLocator]` to generate strongly-typed accessor properties for all registered ViewModels. The generator collects ViewModel types from both MVVM and DI registrations (DI only when class name ends with `ViewModel`), deduplicates, and emits one property per type:

```csharp
[ViewModelLocator]
internal partial class VmLocator { }
```

Given the registrations from the examples above, the generator produces:

```csharp
internal partial class VmLocator
{
    private VmLocator() { }

    public static VmLocator Instance { get; } = new VmLocator();

    // From [NavigationTarget], [DialogTarget], and DI attributes on ViewModel-ending classes
    public MyView1ViewModel MyView1ViewModel => Ioc.Default.GetRequiredService<MyView1ViewModel>();
    public MyView2ViewModel MyView2ViewModel => Ioc.Default.GetRequiredService<MyView2ViewModel>();
    public MyView3ViewModel MyView3ViewModel => Ioc.Default.GetRequiredService<MyView3ViewModel>();
    public MyDlg1ViewModel MyDlg1ViewModel => Ioc.Default.GetRequiredService<MyDlg1ViewModel>();
    public MyDlg2ViewModel MyDlg2ViewModel => Ioc.Default.GetRequiredService<MyDlg2ViewModel>();
    // Note: DI registrations whose class name does not end with "ViewModel"
    // (e.g., MyBackupService) are excluded from the locator.
    // Keyed ViewModel registrations get a suffixed property, e.g., MyView2ViewModel_home.
}
```

**Usage — resolve from code:**
```csharp
var vm = VmLocator.Instance.MyView1ViewModel;
```

**Usage — XAML binding (code-behind):**
```csharp
public partial class MyView1 : UserControl
{
    public MyView1()
    {
        InitializeComponent();
        DataContext = VmLocator.Instance.MyView1ViewModel;
    }
}
```

**Usage — XAML binding (pure XAML via `x:Static`):**
```xaml
<UserControl x:Class="YourApp.Views.MyView1"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:YourApp">
    <UserControl.DataContext>
        <Binding Source="{x:Static local:VmLocator.Instance}" Path="MyView1ViewModel"/>
    </UserControl.DataContext>
    <Grid>
        <TextBlock Text="{Binding Title}"/>
    </Grid>
</UserControl>
```

## API Reference

### Navigation

| API                                   | Description                        |
| ------------------------------------- | ---------------------------------- |
| `INavigationService.NavigateTo()`     | Navigate to a view                 |
| `INavigationService.SetInitialView()` | Set initial view (no confirmation) |
| `INavigationHistory`                  | Back/forward navigation history    |
| `INavigationLifecycle`                | Navigation lifecycle interface     |
| `IConfirmNavigationRequest`           | Navigation confirmation interface  |
| `IHistoryRecordable`                  | History recording interface        |
| `NavigationContext`                   | Navigation context data            |
| `NavigationParameters`                | Navigation parameters collection   |
| `NavigationResult`                    | Navigation result with status      |
| `NavigationStatus`                    | Navigation status enum             |

### Dialog

| API                           | Description                  |
| ----------------------------- | ---------------------------- |
| `IDialogService.Show()`       | Show non-modal dialog        |
| `IDialogService.ShowDialog()` | Show modal dialog            |
| `IDialogLifecycle`            | Dialog lifecycle interface   |
| `DialogResult`                | Dialog result object         |
| `DialogParameters`            | Dialog parameters collection |
| `ButtonResult`                | Button result enum           |

### Threading

| API                          | Description                                    |
| ---------------------------- | ---------------------------------------------- |
| `IUIDispatcher`              | UI thread dispatcher interface                 |
| `IUIDispatcher.Invoke()`     | Synchronously dispatch action to UI thread     |
| `IUIDispatcher.InvokeAsync()`| Asynchronously dispatch action to UI thread    |
| `IUIDispatcher.CheckAccess()`| Check if current thread is the UI thread       |
| `IDispatcherMessenger`       | IMessenger interface with DispatchMode support                                  |
| `DispatchMode`               | Dispatch mode enum (`UIThread` / `PublisherThread`)                             |
| `IDispatcherMessengerExtensions.Register()` | Extension method to register with a specific DispatchMode |
| `IDispatcherMessengerExtensions.Send()`     | Extension method to send with consistent default token   |

### Source Generator

| API                              | Description                           |
| -------------------------------- | ------------------------------------- |
| `AddGeneratedRegistrations()`    | Single entry point for all generated registrations (namespace `RJi.Mvvm.Wpf.IoC`) |
| `[NavigationTarget]`             | Register ViewModel for navigation     |
| `[DialogTarget]`                 | Register ViewModel for dialog         |
| `[DialogWindow]`                 | Register custom dialog window         |
| `[ViewModelLocator]`             | Generate strongly-typed ViewModel locator |
| `[AddSingleton]` / `[AddScoped]` / `[AddTransient]` | DI lifetime attributes |
| `[AddKeyedSingleton]` / `[AddKeyedTransient]` | Keyed DI lifetime attributes |

### Exceptions

| Exception                         | Description                           |
| --------------------------------- | ------------------------------------- |
| `NavigationException`             | Base navigation exception             |
| `ViewNotFoundException`           | View not registered                   |
| `ViewZoneNotFoundException`       | View zone not found                   |
| `ViewZoneAlreadyRegisteredException` | View zone already registered      |
| `ViewModelNotFoundException`      | ViewModel not registered              |
| `ViewInitializationException`     | View initialization failed            |
| `NavigationCanceledException`     | Navigation canceled                   |
| `DialogException`                 | Base dialog exception                 |
| `DialogViewNotFoundException`     | Dialog view not registered            |
| `DialogWindowNotFoundException`   | Custom dialog window not registered   |
| `DialogWindowContentException`    | View is a Window, can't be content    |
| `DialogViewTypeNotSupportedException` | View is not a FrameworkElement    |
| `DialogLifecycleException`        | Missing IDialogLifecycle interface    |

## License

MIT License
