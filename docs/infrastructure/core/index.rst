==================
Core 基础设施
==================

Core 为部分平台有关的基础设施（例如 WPF 支持），同时也是整个程序运行的入口。Core负责选用基础设施，并进行配置和调度。同时也负责插件的载入和解析。

.. sourcecode:: csharp

    new KernelBuilder()
    .UseConfiguration(....) // 配置文件
    .UsePluginOptions(Path.Combine(Directory.GetCurrentDirectory(), "CoreAppSetting.json")) // Options
    .UseConfigurationModel<CoreAppSetting>() // 配置类
    .UseLogger(....) // 配置日志
    .UseEventBus()  // EventBus服务
    .UseCommandService() // Command服务
    .UseKernelService<CorePluginResolver>() // CorePluginResolver用于插件解析
    .UsePlugins<PluginsManager>() // 插件管理
    .UseWindowManager<EmberWpfUIService, Window>() // 窗口管理
    .UseMvvmInterface((mvvm) => mvvm.UseConfigurationModel()) // Kernel MVVM Model管理
    .UseEFSqlite() // 使用基于EntityFramework的Sqlite
    .UseStatistic(statistic => statistic // 使用数据统计
        .ConfigureEventSourceManager()
        .ConfigureDefaultDataSource()
        .ConfigureDefaultFormatter()
        .ConfigureDefaultHub())
    .Build()
    .Run();

.. toctree::
  :maxdepth: 1
  :glob:


  