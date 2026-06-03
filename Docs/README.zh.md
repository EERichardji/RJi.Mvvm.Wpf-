# RJi.Mvvm.Wpf

基于 `CommunityToolkit.Mvvm` 和 `Microsoft.Extensions.DependencyInjection` 的轻量级 WPF MVVM 框架，支持 `.NET 8.0` 和 `.NET Framework 4.8`。

> **⚠️ 目标 .NET Framework 4.8 必须使用 SDK 风格项目**
>
> 本文档介绍的**所有源码生成器**（包括 CT.Mvvm 的 `[ObservableProperty]`、`[RelayCommand]` 和框架自身的 `[NavigationTarget]`、`[DialogTarget]`、DI 特性等）均依赖 **C# 8.0+** 及 NuGet 分析器自动加载机制。传统 `packages.config` + 旧式 csproj 无法满足此要求，必须手动转为 SDK 风格。
>
> **操作步骤：**
>
> 1. 找到 `.csproj` 文件，右键 → **打开并编辑 .csproj**，**删除全部内容**
> 2. **粘贴以下内容**，替换原内容：
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
> 3. **删除 `packages.config`** 文件（如存在）
> 4. 重新生成解决方案
>
> > .NET 8.0 项目无此限制，SDK 风格已是默认格式。

## 核心特性

- **MVVM 集成**: 基于 `CommunityToolkit.Mvvm`，内置信使服务支持。
- **零反射**: 所有 View-ViewModel 绑定通过泛型键值方法在启动时显式注册。
- **导航系统**: 视图区域、生命周期管理、历史记录和视图缓存。
- **对话框窗口**: 内置 `DialogWindow`，支持生命周期钩子和自定义窗口。
- **源码生成器**: 集成 CT.Mvvm 源码生成器（`[ObservableProperty]`、`[RelayCommand]` 等），并额外提供 `[NavigationTarget]`、`[DialogTarget]`、DI 特性等框架级生成器。

## 快速开始

### 引用命名空间

**App.xaml 使用 RJiApplication 作为根元素：**

```xml
<rj:RJiApplication x:Class="MyApp.App"
                   xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                   xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                   xmlns:rj="http://rji.mvvm.wpf/views"
                   xmlns:local="clr-namespace:MyApp">

    <rj:RJiApplication.Resources>
        <!-- 应用资源 -->
    </rj:RJiApplication.Resources>
</rj:RJiApplication>
```

> **重要**：必须**删除 `StartupUri` 属性**，否则会导致应用启动两个窗口（一个来自 `StartupUri`，一个来自 `CreateShell()`）。框架通过 `CreateShell()` 方法创建主窗口。

**命名空间说明：**

框架已通过 `AssemblyInfo.cs` 将所有相关 CLR 命名空间统一映射到单个 XML 命名空间，因此只需要在 XAML 中引用一次：

```xml
xmlns:rj="http://rji.mvvm.wpf/views"
```

**C# 代码中需要的 using 语句：**

```csharp
using RJi.Mvvm.Wpf;              // 核心应用类 RJiApplication
using RJi.Mvvm.Wpf.IoC;          // 服务注册扩展方法
using RJi.Mvvm.Wpf.Navigation;   // 导航相关接口和类
using RJi.Mvvm.Wpf.Dialogs;      // 对话框相关接口和类
```

## 创建应用

```csharp
using RJi.Mvvm.Wpf;
using RJi.Mvvm.Wpf.IoC;
using Microsoft.Extensions.DependencyInjection;

public class App : RJiApplication
{
    protected override void ConfigureServices(IServiceCollection services)
    {
        // 注册导航视图及其 ViewModel
        services.AddViewToNavigation<HomeView, HomeViewModel>();
        services.AddViewToNavigation<SettingsView, SettingsViewModel>();

        // 注册对话框视图
        services.AddViewToDialog<ConfirmDialogView, ConfirmDialogViewModel>();
        services.AddViewToDialog<AboutDialogView, AboutDialogViewModel>();

        // 注册自定义对话框窗口（可选）
        services.AddDialogWindow<CustomDialogWindow>();
    }

    /// <summary>
    /// 创建应用主窗口（启动画面）
    /// </summary>
    /// <returns>返回一个 Window 实例作为应用的主窗口</returns>
    protected override Window CreateShell() => new MainWindow();
}
```

### 容器默认服务

框架基于 [`Microsoft.Extensions.DependencyInjection`](https://learn.microsoft.com/zh-cn/dotnet/core/extensions/dependency-injection/usage) 构建依赖注入容器。在 `RJiApplication` 初始化时自动注册以下服务：

| 服务接口             | 说明                                        |
| -------------------- | ------------------------------------------- |
| `INavigationService` | 导航服务（Singleton）                       |
| `IDialogService`     | 对话框服务（Singleton）                     |
| `IDispatcherMessenger` | CommunityToolkit.Mvvm 信使（带 UI 线程调度 + DispatchMode 支持）（Singleton） |
| `IUIDispatcher`      | UI 线程调度器（Singleton）                  |
| `IServiceProvider`   | 依赖注入容器                                |

> **注意**：`IDispatcherMessenger` 已预注册（继承 `IMessenger`），可直接在 ViewModel 中通过构造函数注入使用。详见[信使与 UI 线程调度](#信使与-ui-线程调度)。

### CommunityToolkit.Mvvm 用法

框架基于 [`CommunityToolkit.Mvvm`](https://learn.microsoft.com/zh-cn/dotnet/communitytoolkit/mvvm/) 构建，核心用法：

> **前提**：目标 net48 需先按文档开头的说明配置 SDK 风格项目，否则以下 CT.Mvvm 源码生成器无效。

```csharp
public partial class HomeViewModel : ObservableObject
{
    // [ObservableProperty] → 自动生成 INotifyPropertyChanged
    [ObservableProperty]
    private string _title = "Home";

    partial void OnTitleChanged(string value) => /* 响应变化 */;

    // [RelayCommand] → 自动生成 ICommand 属性（方法名 + Command）
    [RelayCommand]
    private void Navigate() { }                         // 无参

    [RelayCommand]
    private void Open(string path) { }                  // 带参

    [RelayCommand(CanExecute = nameof(CanSave))]
    private void Save() { }                             // 带 CanExecute

    private bool CanSave => true;
}

// 不使用源码生成器时，无需 partial 关键字，也无分部方法：
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

**信使**（支持自动 UI 线程调度）：

> **⚠️ 必须使用注入的 `IDispatcherMessenger`**（而非直接使用 `WeakReferenceMessenger.Default`），否则 handler 不会自动切到 UI 线程。`IDispatcherMessenger` 提供了配套的 `Send<TMessage>()` 扩展方法，确保 token 一致、消息正确送达。

框架的 `IDispatcherMessenger` 包装了 `WeakReferenceMessenger.Default`，从后台线程发消息时，**handler 自动在 UI 线程执行**，无需手动 `Dispatcher.Invoke`。

```csharp
public class DataViewModel
{
    private readonly IDispatcherMessenger _messenger;
    public DataViewModel(IDispatcherMessenger messenger) => _messenger = messenger;

    // 后台线程发 → handler 自动在 UI 线程执行
    public void OnDataReceived(float value) =>
        _messenger.Send(new DataUpdatedMessage(value));
}

// 收方 — 更新 binding 安全
public class DisplayViewModel
{
    public DisplayViewModel(IDispatcherMessenger messenger) =>
        messenger.Register<DataUpdatedMessage>(this, (r, m) =>
        {
            DataValue = m.Value; // 不用 Dispatcher.Invoke
        });
}

public class DataUpdatedMessage(float value)
{
    public float Value { get; } = value;
}
```

**指定线程调度模式**：注册时可通过 `DispatchMode` 控制 handler 是否自动切到 UI 线程：

```csharp
// 默认模式：自动切 UI 线程（和上述示例相同）
messenger.Register<Msg>(this, handler);
messenger.Register<Msg>(this, handler, DispatchMode.UIThread);

// 发布者线程执行：handler 在 Send 所在的线程直跑，不做任何调度
messenger.Register<Msg>(this, handler, DispatchMode.PublisherThread);
```

> `DispatchMode.PublisherThread` 适用于日志记录、缓存更新等不需要碰 UI 的场景——handler 直接执行，没有调度开销。

> **手动调度**：`IUIDispatcher` 也已注册到容器，封装了 `Application.Current.Dispatcher`，需要时可注入使用。

> **完整文档**：[CommunityToolkit.Mvvm](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/)

### 定义视图区域

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

## 导航 API

### 基础导航

```csharp
// 通过依赖注入获取导航服务
INavigationService navigationService = serviceProvider.GetRequiredService<INavigationService>();

// 导航到视图并处理回调
navigationService.NavigateTo(
    viewZoneName: "Main",
    viewKey: "Home",
    parameters: new NavigationParameters { { "userId", 123 } },
    navigationCallback: result => {
        if (result.Status == NavigationStatus.Success) {
            // 导航成功
        } else if (result.Status == NavigationStatus.Canceled) {
            // 用户取消导航
        }
    });

// 设置初始视图（跳过确认和历史记录）
navigationService.SetInitialView("Main", "Home", null);
```

### 导航上下文和参数

```csharp
// 创建导航参数
var parameters = new NavigationParameters {
    { "id", 1 },
    { "name", "Test" }
};

// 或从查询字符串创建
var paramsFromQuery = new NavigationParameters("id=1&name=Test");

// 在 ViewModel 中获取参数
public void OnNavigationEnter(NavigationContext context) {
    int? userId = context.Parameters?.GetValue<int>("userId");
    string? name = context.Parameters?.GetValue<string>("name");
}
```

### 导航生命周期

```csharp
public class HomeViewModel : ObservableObject, INavigationLifecycle
{
    // 导航离开后保持视图实例存活（默认：false）
    public bool KeepAlive => true;

    // 检查是否应重用现有实例
    public bool ShouldReuseInstance(NavigationContext context) => true;

    // 进入视图时调用
    public void OnNavigationEnter(NavigationContext context) {
        // 初始化数据
    }

    // 离开视图时调用
    public void OnNavigationExit(NavigationContext context) {
        // 清理资源
    }
}
```

> **提示**：如果 ViewModel 实现了 `IDisposable`，当视图被导航缓存驱逐（如 `KeepAlive=false` 切走、LRU 缓存满、ViewZone 卸载）时，框架会自动调用 `Dispose()` 方法，方便释放信使订阅、定时器等托管资源：

> ```csharp
> public class MyViewModel : ObservableObject, INavigationLifecycle, IDisposable
> {
>     public void Dispose()
>     {
>         WeakReferenceMessenger.Default.UnregisterAll(this);
>     }
> }
> ```

> **注意**：回调模式支持异步确认场景（如对话框确认）。如果视图未实现 `IConfirmNavigationRequest`，则默认允许导航。

### 导航历史

实现 `IHistoryRecordable` 接口控制视图是否记录到导航历史：

```csharp
public class MyViewModel : ObservableObject, IHistoryRecordable
{
    /// <summary>
    /// true 表示此次导航会被记录到历史，支持 GoBack/GoForward；
    /// false 表示跳过历史记录。
    /// </summary>
    public bool RecordInHistory => true;
}
```

导航完成后通过 `NavigationResult` 获取 `INavigationHistory` 实例进行前进/后退操作：

```csharp
navigationService.NavigateTo("Main", "Views.MyView", parameters, result =>
{
    if (result.Status == NavigationStatus.SuccessWithHistory)
    {
        var history = result.Context.History;
        if (history != null)
        {
            // 后退/前进
            history.GoBack();
            history.GoForward();

            // 检查是否可操作
            bool canBack = history.CanGoBack;
            bool canForward = history.CanGoForward;
        }
    }
});
```

## 对话框 API

### 显示对话框

```csharp
IDialogService dialogService = serviceProvider.GetRequiredService<IDialogService>();

// 显示非模态对话框
dialogService.Show(
    viewKey: "Settings",
    parameters: new DialogParameters { { "config", configData } },
    dialogCallback: result => {
        if (result.Result == ButtonResult.OK) {
            // 处理确定
        }
    });

// 显示模态对话框（阻塞UI）
dialogService.ShowDialog(
    viewKey: "Confirm",
    parameters: new DialogParameters { { "message", "确定要删除吗？" } },
    dialogCallback: result => {
        if (result.Result == ButtonResult.Yes) {
            // 用户确认
        }
    });
```

### 对话框生命周期

> **重要**：对话框 ViewModel **必须实现 `IDialogLifecycle` 接口**，否则在显示对话框时会抛出 `DialogLifecycleException` 异常。

```csharp
public class ConfirmDialogViewModel : ObservableObject, IDialogLifecycle
{
    // 请求关闭对话框的事件
    public event Action<DialogResult> RequestClose = delegate { };

    // 检查对话框是否可以关闭
    public bool CanCloseDialog() => true;

    // 对话框打开时调用
    public void OnDialogOpening(DialogParameters parameters) {
        // 使用参数初始化
        Message = parameters.GetValue<string>("message");
    }

    // 对话框关闭后调用
    public void OnDialogClosed() {
        // 清理资源
    }

    // 以编程方式关闭对话框
    private void CloseWithResult(ButtonResult result) {
        RequestClose.Invoke(new DialogResult(result));
    }
}
```

### 对话框窗口附加属性

框架提供了 `Dialog` 附加属性，**设置在对话框内容视图（UserControl）上**，用于配置对话框窗口的行为：

```xml
<UserControl x:Class="MyApp.Dialogs.ConfirmDialogView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:rj="http://rji.mvvm.wpf/views"
             rj:Dialog.WindowStartupLocation="CenterOwner">

    <!-- 内联定义对话框窗口样式 -->
    <rj:Dialog.WindowStyle>
        <Style TargetType="Window">
            <Setter Property="Background" Value="Bisque" />
            <Setter Property="Width" Value="400" />
            <Setter Property="Height" Value="250" />
        </Style>
    </rj:Dialog.WindowStyle>

    <StackPanel Margin="20">
        <TextBlock FontSize="20" Text="对话框内容" />
        <StackPanel Orientation="Horizontal" HorizontalAlignment="Center">
            <Button Content="确定" Command="{Binding OkCommand}" />
            <Button Content="取消" Command="{Binding CancelCommand}" />
        </StackPanel>
    </StackPanel>
</UserControl>
```

**附加属性说明：**

| 属性                              | 类型                    | 说明                                 |
| --------------------------------- | ----------------------- | ------------------------------------ |
| `rj:Dialog.WindowStyle`           | `Style`                 | 设置对话框窗口的样式（支持内联定义） |
| `rj:Dialog.WindowStartupLocation` | `WindowStartupLocation` | 设置窗口启动位置（默认 CenterOwner） |

**工作原理：**

`DialogService` 在显示对话框时，会从 UserControl 上读取这些附加属性值，然后**显式设置**到对话框窗口对象上。这意味着：

- 使用**默认对话框窗口**时，附加属性同样生效
- 使用**自定义对话框窗口**时，附加属性也会覆盖窗口的默认设置

**第三方库兼容性：**

附加属性使用标准的 WPF DependencyProperty 机制，理论上与所有基于 WPF 的 UI 库兼容，包括以下免费开源库：

- MaterialDesignThemes
- MahApps.Metro
- HandyControl

> **注意**：建议在实际使用前进行测试验证。

### 自定义对话框窗口

框架**默认已提供 `DialogWindow` 实现**，通常情况下无需自定义，可直接使用。仅当需要特殊样式或行为时才需要自定义对话框窗口。

> **注意**：自定义对话框窗口**必须同时继承 `Window` 和实现 `IDialogWindow` 接口**，否则无法注册和使用。

#### 步骤 1：创建自定义窗口

```csharp
// CustomDialogWindow.xaml.cs
public partial class CustomDialogWindow : Window, IDialogWindow
{
    public CustomDialogWindow()
    {
        InitializeComponent();
    }

    // 必须实现 IDialogWindow 接口
    public DialogResult? Result { get; set; }
}
```

#### 步骤 2：注册自定义窗口

```csharp
// 在 ConfigureServices 中注册  （需要 using RJi.Mvvm.Wpf.IoC;）
protected override void ConfigureServices(IServiceCollection services)
{
    // 注册默认自定义对话框窗口
    services.AddDialogWindow<CustomDialogWindow>();

    // 或使用键注册多个窗口类型
    services.AddDialogWindow<CustomDialogWindow>("CustomKey");
    services.AddDialogWindow<AnotherDialogWindow>("AnotherKey");
}
```

#### 步骤 3：使用指定窗口显示对话框

```csharp
// 使用默认窗口
dialogService.ShowDialog("Confirm", parameters, callback);

// 使用指定键的窗口
dialogService.ShowDialog(
    viewKey: "Settings",
    parameters: parameters,
    dialogCallback: callback,
    windowKey: "CustomKey"); // 指定使用哪个窗口
```

### 按钮结果

```csharp
// 常见按钮结果枚举
public enum ButtonResult {
    None,    // 默认，无结果
    OK,      // 确定按钮
    Cancel,  // 取消按钮
    Yes,     // 是按钮
    No       // 否按钮
}
```

### 对话框结果

```csharp
public class DialogResult {
    public ButtonResult Result { get; }
    public DialogParameters? Parameters { get; }
}
```

### 导航结果

```csharp
public class NavigationResult {
    public NavigationStatus Status { get; set; }
    public NavigationContext Context { get; }
    public Exception? Exception { get; set; }
    public string? ErrorMessage { get; set; }
}

public enum NavigationStatus {
    Success,              // 导航成功（不入历史）
    SuccessWithHistory,   // 导航成功（记录到历史）
    ViewNotFound,
    ViewZoneNotFound,
    Canceled,
    InvalidContentHost,
    ViewInitializationFailed
}
```

## 异常处理

### 自定义异常

```csharp
try {
    navigationService.NavigateTo("Main", "NonExistentView", null, null);
}
catch (ViewNotFoundException ex) {
    // 视图未注册
    Console.WriteLine($"视图 '{ex.ViewKey}' 未找到");
}
catch (ViewZoneNotFoundException ex) {
    // 视图区域未找到
    Console.WriteLine($"视图区域 '{ex.ViewZoneName}' 未找到");
}
catch (NavigationCanceledException) {
    // 用户取消导航
}
catch (NavigationException ex) {
    // 通用导航错误
    Console.WriteLine($"导航失败: {ex.Message}");
}

// 对话框异常
try {
    dialogService.ShowDialog("MissingDialog", null, null);
}
catch (DialogViewNotFoundException ex) {
    // 对话框视图未注册
    Console.WriteLine($"对话框视图 '{ex.ViewKey}' 未注册");
}
catch (DialogWindowNotFoundException ex) {
    // 自定义对话框窗口未注册
    Console.WriteLine($"对话框窗口 '{ex.WindowKey}' 未注册");
}
catch (DialogWindowContentException ex) {
    // 视图类型为 Window，不能作为对话框内容
    Console.WriteLine($"对话框内容无效: {ex.ViewKey}");
}
catch (DialogViewTypeNotSupportedException ex) {
    // 视图类型不是 FrameworkElement
    Console.WriteLine($"视图类型不支持: {ex.ViewType}");
}
catch (DialogLifecycleException ex) {
    // ViewModel 未实现 IDialogLifecycle 接口
    Console.WriteLine($"ViewModel 缺少 IDialogLifecycle: {ex.DataContextType}");
}
catch (DialogException ex) {
    // 通用对话框错误
    Console.WriteLine($"对话框操作失败: {ex.Message}");
}
```

## 注册扩展方法

框架提供了三个核心扩展方法（命名空间 `RJi.Mvvm.Wpf.IoC`），用于注册导航视图、对话框视图和自定义对话框窗口：

```csharp
public static class IServiceCollectionExtension
{
    // 注册导航视图及其 ViewModel
    public static IServiceCollection AddViewToNavigation<TView, TViewModel>(
        this IServiceCollection services, string? key = null)
        where TView : UserControl where TViewModel : class;

    // 注册对话框视图及其 ViewModel
    public static IServiceCollection AddViewToDialog<TView, TViewModel>(
        this IServiceCollection services, string? key = null)
        where TView : UserControl where TViewModel : class, IDialogLifecycle;

    // 注册自定义对话框窗口
    public static IServiceCollection AddDialogWindow<TDialogWindow>(
        this IServiceCollection services, string? key = null)
        where TDialogWindow : Window, IDialogWindow;
}
```

### 注册说明

> **注意**：
>
> - 导航视图注册仅支持 `UserControl` 和 `ContentControl` 类型。其他控件类型（如 `Window`、`Page` 等）不支持作为导航视图使用。对话框视图同样只支持 `UserControl` 和 `ContentControl`。
> - **泛型约束**：`AddViewToNavigation<TView, TViewModel>` 要求 `TView : UserControl` 且 `TViewModel : class`；`AddViewToDialog<TView, TViewModel>` 要求 `TView : UserControl` 且 `TViewModel : class, IDialogLifecycle`；`AddDialogWindow<TDialogWindow>` 要求 `TDialogWindow : Window, IDialogWindow`。
> - **Key 冲突处理**：当使用相同的 key 注册不同的视图类型时，会抛出 `InvalidOperationException`；注册相同视图类型则忽略重复注册。
> - **DataContext 优先级**：如果视图（View）已有 `DataContext`（如手动设置或通过 XAML 绑定），则不会使用注册时指定的 ViewModel，支持 View-First 模式。
> - **默认生命周期**：通过扩展方法注册的视图默认均为**瞬态（Transient）**。如需 Singleton 或 Scoped 等其他生命周期，直接在容器中手动注册即可，**无需关注注册顺序**。

## 源码生成器

框架包含一个 Roslyn 源码生成器，消除手动注册的样板代码。

> **前提**：目标 net48 需先按文档开头的说明配置 SDK 风格项目，否则以下框架源码生成器无效。

### 使用方式

在 `ConfigureServices` 中用单行调用替代手动注册：

```csharp
protected override void ConfigureServices(IServiceCollection services)
{
    services.AddGeneratedRegistrations();  // 需要 using RJi.Mvvm.Wpf.IoC;
}
```

### 特性

#### 视图注册

| 特性 | 参数 | 默认值 |
|------|------|--------|
| `[NavigationTarget]` | `viewType`（可选）：显式指定 View 类型<br>`Key`（可选）：自定义注册键 | `viewType` = 按约定查找<br>`Key` = `{命名空间末段}.{View类型名}` |
| `[DialogTarget]` | `viewType`（可选）：显式指定 View 类型<br>`Key`（可选）：自定义注册键 | 同上 |
| `[DialogWindow]` | `Key`（可选）：自定义注册键 | `Key` = `{命名空间末段}.{Window类型名}` |

**参数组合示例：**
```csharp
[DialogTarget]                                   // viewType=约定, Key=默认
[DialogTarget(typeof(MyDialogView))]             // viewType=显式, Key=默认
[DialogTarget(Key = "custom")]                   // viewType=约定, Key=自定义
[DialogTarget(typeof(MyDialogView), Key = "custom")] // 都显式指定
```

**命名约定**：未指定视图类型时，生成器按约定搜索——去掉 ViewModel 名称的 `ViewModel` 后缀，在 `.Views` 命名空间中查找匹配类。例如 `ViewModels.MyView1ViewModel` → `Views.MyView1`。

**源代码 — 你的类声明：**
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

**生成的等价代码（在 `AddGeneratedRegistrations` 内部）：**
```csharp
services.AddViewToNavigation<Views.MyView1, ViewModels.MyView1ViewModel>();
services.AddViewToNavigation<Views.MyView2, ViewModels.MyView2ViewModel>("home");
services.AddViewToNavigation<Views.MyView3, ViewModels.MyView3ViewModel>();
services.AddViewToDialog<Views.MyDlg1, ViewModels.MyDlg1ViewModel>();
services.AddViewToDialog<Views.MyDlg2, ViewModels.MyDlg2ViewModel>("edit");
services.AddDialogWindow<Views.MyDialogWindow>();
services.AddDialogWindow<Views.MyPopupWindow>("popup");
```

#### DI 注册

| 特性 | 参数 | 默认值 |
|------|------|--------|
| `[AddSingleton]` | `serviceType`（可选）：接口类型 | `serviceType` = 类自身 |
| `[AddSingleton(typeof(IService))]` | `serviceType` + `implementationType`（可选） | `implementationType` = 类自身 |
| `[AddScoped]` / `[AddTransient]` | 同模式 | — |
| `[TryAddSingleton]` | 同 `[AddSingleton]`，已注册时跳过 | — |
| `[AddKeyedSingleton("key")]` | `serviceType`（可选）+ `key` | `serviceType` = 类自身 |
| `[AddKeyedTransient("key")]` | 同模式 | — |

```csharp
[AddSingleton(typeof(IMyService))]
internal class MyService : IMyService { }

[AddKeyedSingleton(typeof(IMyService), "backup")]
internal class MyBackupService : IMyService { }

[AddSingleton]
internal partial class MainWindowViewModel : ObservableObject { }
```

**生成的等价代码（在 `AddGeneratedRegistrations` 内部）：**
```csharp
services.AddSingleton<IMyService, MyService>();
services.AddKeyedSingleton<IMyService, MyBackupService>("backup");
services.AddSingleton<MainWindowViewModel>();
```

所有 12 种 DI 特性均可用：`Add`/`TryAdd` 对应 `Singleton`/`Scoped`/`Transient`，以及 `AddKeyed`/`TryAddKeyed` 变体。

### ViewModel 定位器

用 `[ViewModelLocator]` 标记 `partial class`，为所有已注册的 ViewModel 生成强类型访问属性。生成器从 MVVM 注册和 DI 注册（仅类名以 `ViewModel` 结尾）中收集 ViewModel 类型，去重后每个类型生成一个属性：

```csharp
[ViewModelLocator]
internal partial class VmLocator { }
```

基于上述示例中的注册，生成器产出：

```csharp
internal partial class VmLocator
{
    private VmLocator() { }

    public static VmLocator Instance { get; } = new VmLocator();

    // 来自 [NavigationTarget]、[DialogTarget] 以及 DI 特性（仅 ViewModel 后缀类）
    public MyView1ViewModel MyView1ViewModel => Ioc.Default.GetRequiredService<MyView1ViewModel>();
    public MyView2ViewModel MyView2ViewModel => Ioc.Default.GetRequiredService<MyView2ViewModel>();
    public MyView3ViewModel MyView3ViewModel => Ioc.Default.GetRequiredService<MyView3ViewModel>();
    public MyDlg1ViewModel MyDlg1ViewModel => Ioc.Default.GetRequiredService<MyDlg1ViewModel>();
    public MyDlg2ViewModel MyDlg2ViewModel => Ioc.Default.GetRequiredService<MyDlg2ViewModel>();
    // 注：类名不以 ViewModel 结尾的 DI 注册（如 MyBackupService）不会生成本地属性。
    // 带 Key 的 ViewModel 注册会附加后缀属性，例如 MyView2ViewModel_home。
}
```

**用法 — 代码解析：**
```csharp
var vm = VmLocator.Instance.MyView1ViewModel;
```

**用法 — XAML 绑定（code-behind 设 DataContext）：**
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

**用法 — 纯 XAML 绑定（通过 `x:Static`）：**
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

## API 参考

### 导航

| API                                   | 说明                   |
| ------------------------------------- | ---------------------- |
| `INavigationService.NavigateTo()`     | 导航到视图             |
| `INavigationService.SetInitialView()` | 设置初始视图（无确认） |
| `INavigationHistory`                  | 前进/后退导航历史      |
| `INavigationLifecycle`                | 导航生命周期接口       |
| `IConfirmNavigationRequest`           | 导航确认接口           |
| `IHistoryRecordable`                  | 历史记录接口           |
| `NavigationContext`                   | 导航上下文数据         |
| `NavigationParameters`                | 导航参数集合           |
| `NavigationResult`                    | 导航结果（含状态）     |
| `NavigationStatus`                    | 导航状态枚举           |

### 对话框

| API                           | 说明               |
| ----------------------------- | ------------------ |
| `IDialogService.Show()`       | 显示非模态对话框   |
| `IDialogService.ShowDialog()` | 显示模态对话框     |
| `IDialogLifecycle`            | 对话框生命周期接口 |
| `DialogResult`                | 对话框结果对象     |
| `DialogParameters`            | 对话框参数集合     |
| `ButtonResult`                | 按钮结果枚举       |

### 线程调度

| API                          | 说明                            |
| ---------------------------- | ------------------------------- |
| `IUIDispatcher`              | UI 线程调度器接口               |
| `IUIDispatcher.Invoke()`     | 同步调度操作到 UI 线程          |
| `IUIDispatcher.InvokeAsync()`| 异步调度操作到 UI 线程          |
| `IUIDispatcher.CheckAccess()`| 检查当前线程是否为 UI 线程      |
| `IDispatcherMessenger`       | 带 DispatchMode 支持的 IMessenger 接口                  |
| `DispatchMode`               | 消息分发模式枚举（`UIThread` / `PublisherThread`）       |
| `IDispatcherMessengerExtensions.Register()` | 指定 DispatchMode 的注册扩展方法 |
| `IDispatcherMessengerExtensions.Send()`     | 统一默认 token 的发送扩展方法     |

### 源码生成器

| API                              | 说明                           |
| -------------------------------- | ------------------------------ |
| `AddGeneratedRegistrations()`    | 所有生成注册的统一入口（命名空间 `RJi.Mvvm.Wpf.IoC`） |
| `[NavigationTarget]`             | 注册导航 ViewModel             |
| `[DialogTarget]`                 | 注册对话框 ViewModel           |
| `[DialogWindow]`                 | 注册自定义对话框窗口           |
| `[ViewModelLocator]`             | 生成强类型 ViewModel 定位器    |
| `[AddSingleton]` / `[AddScoped]` / `[AddTransient]` | DI 生命周期特性   |
| `[AddKeyedSingleton]` / `[AddKeyedTransient]` | 键控 DI 生命周期特性 |

### 异常类型

| 异常                              | 说明                           |
| --------------------------------- | ------------------------------ |
| `NavigationException`             | 导航异常基类                   |
| `ViewNotFoundException`           | 视图未注册                     |
| `ViewZoneNotFoundException`       | 视图区域未找到                 |
| `ViewZoneAlreadyRegisteredException` | 视图区域重复注册           |
| `ViewModelNotFoundException`      | ViewModel 未注册               |
| `ViewInitializationException`     | 视图初始化失败                 |
| `NavigationCanceledException`     | 导航被取消                     |
| `DialogException`                 | 对话框异常基类                 |
| `DialogViewNotFoundException`     | 对话框视图未注册               |
| `DialogWindowNotFoundException`   | 自定义对话框窗口未注册         |
| `DialogWindowContentException`    | 视图类型为 Window，不能作为内容|
| `DialogViewTypeNotSupportedException` | 视图类型不是 FrameworkElement|
| `DialogLifecycleException`        | 缺少 IDialogLifecycle 接口     |

## 许可证

MIT License
