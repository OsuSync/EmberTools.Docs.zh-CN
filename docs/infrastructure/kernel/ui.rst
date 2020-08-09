用户界面相关 (UI)
==================

Kernel中提供了对窗口管理、View、ViewModel的抽象。同时还对组件(Component)做了统一抽象。

窗口管理器
------------
Kernel对窗体管理器的行为做了一些抽象，将窗口分为 ``WindowManager`` 和 ``HostedWindow`` ， WindowManager 代表 UI 生命周期中 HostedWindow 的管理者。同时窗口管理器还需要支持直接在UI线程上执行操作。

大部分场景下UI都是独立线程，跨线程的UI访问一般是被禁止的，我们可以通过 ``IWindowManager`` 的 ``BeginUIThreadScope`` 进行UI操作。

.. sourcecode:: csharp

    var controlType = ...
    var windowManager = scope.Resolve<IWindowManager>();
    var component = scope.Resolve<MyUIComponent>();
    await windowManager.BeginUIThreadScope(() =>
    {
        // 因为WPF组件只能在 UI 线程上 new ，所以此时需要 BeginUIThreadScope
        component.InputControl = (Control)Activator.CreateInstance(controlType);
        component.InputControl.DataContext = Value;
    });

EmberTools中的默认窗口管理器实现在 EmberCore 项目中，基于 WPF ，详情可移步 EmberCore 的相关基础设施介绍。

窗口
-----------

继承 IHostedWindow 的类即可满足 IWindowManager 的限制要求，IHostedWindow 接口将特定窗口的生命周期放入了 Kernel 中统一管理，即和插件一样有 ``Initialize`` 和 ``UnInitialize`` 方法来标志生命周期的开始和结束。

这里使用 EmberWpfCore 项目中的 Main 窗口作为例子：

.. sourcecode:: csharp

    partial class Main : Window, IHostedWindow
    {
        private Kernel Kernel { get; set; }

        // 生命周期开始，我们从scope中解析 kernel 的实例
        public ValueTask Initialize(ILifetimeScope scope)
        {
            Kernel = scope.Resolve<Kernel>();
            Show();
            return default;
        }

        // 生命周期结束，我们隐藏和关闭窗口
        public ValueTask Uninitialize(ILifetimeScope scope)
        {
            Hide();
            Close();
            return default;
        }

        // 界面上支持点击按钮后退出程序，code behind，直接调用 kernel 实例中的退出方法
        private void Button_Click(object sender, RoutedEventArgs e)
        {
            _ = Kernel.Exit();
        }
    }

ViewModel
------------

Kenrel中提供了 ``ViewModelManager`` 可以对实现了 ``INotifyPropertyChanged`` 接口的类实例进行管理的能力，同时提供了 ``DependencySet`` 来平展POCO，方便View进行相关数据处理。

ViewModelManager
^^^^^^^^^^^^^^^^^^

ViewModelManager 本身实现和使用都较为简单，这里不再过多阐述。Kernel基于 ViewModelManager 实现了 ConfigurationModelManager。可以参考其实现。

（题外话：ViewModelManager实现待重构为从生命周期中解析并绑定）

.. sourcecode:: csharp

    public class ConfigurationModelManager : ObservableCollection<object>, INotifyPropertyChanged, IConfigurationModelManager, IDisposable
    {
        private IViewModelManager Manager { get; }
        public ConfigurationModelManager(IViewModelManager manager)
        {
            this.Manager = manager;
            this.Manager.Register(this);
        }

        public void Dispose()
        {
            Manager.Unregister<ConfigurationModelManager>();
        }
    }

DependencySet
^^^^^^^^^^^^^^^

DependencySet 可将 POCO 转换为平展的 ``Dictionary<T, U>`` 形式，供View渲染使用，通过获得PropertyInfo来直接读写和访问源对象。

EmberCore 的WPF实现中实现了一系列 Converter 可供自动转换，也可以详细控制转换目标的组件类型，详见 EmberCore 相关设施的介绍。

在实际使用案例是与 `插件配置 <pluginconfig>`_ 配合做双向绑定。即配置文件的修改可以同步到 UI 上， UI的修改也能同步到配置文件中。

.. sourcecode:: csharp

    // 构建生命周期时，额外使用ConfigureUIModel注册一个实例
    public override void BuildComponents(IComponentBuilder builder)
    {
        builder.UsePluginOptionsModel<MyPlugin, MyPluginConfiguration>();
        builder.ConfigureUIModel<MyPlugin, MyPluginConfiguration>();
    }

    // 手动将实例注册到 UIModel 上
    public override ValueTask Initialize(ILifetimeScope scope)
    {
        scope.RegisterUIModel<MyPlugin, MyPluginConfiguration>();
    }

    // 同样也需要反注册
    public override ValueTask Uninitialize(ILifetimeScope scope)
    {
        scope.UnregisterUIModel<MyPlugin, MyPluginConfiguration>();
    }

注册之后 EmberWpfCore 中的配置界面就能自动展示出你的配置类映射到UI之后的界面了。如果对数据格式有特殊要求，还可以引入 EmberCore 来做更细粒度的控制。

引入EmberCore之后，RegisterUIModel将多出一个重载，可以对界面选项做出控制。

样例来自 BeatmapDownloader

.. sourcecode:: csharp

    public override async ValueTask Initialize(ILifetimeScope scope)
    {
        // 解析出生命周期中的所有 DownloadProvider
        var downloadProviderViewModel = scope.Resolve<DownloadProvidersViewModel>();

        // 让DownloadProvider这个配置项使用 UseComboList 并将 解析出来的 DownloadProvider 作为选项。
        scope.RegisterUIModel<MultiPlayerDownloaderUI, BeatmapDownloaderConfiguration>(wpf => wpf
            .UseComboList(f => f.DownloadProvider, downloadProviderViewModel));
    }
