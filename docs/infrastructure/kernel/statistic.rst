数据统计
===========

数据统计可以按照变量进行数据的收集和发布更新，变量持有方通知DataSource，DataSource比较变量的值，通知Formatter，Formatter按照变量变动进行统计数据的格式化。

DataSource注册变量
---------------------

我们使用 ``Variable`` 来存储我们的变量。，这个类有两个子类分别是 ``StringVariable`` 和 ``NumberVariable``。我们以 ``StringVariable`` 为例：

.. sourcecode:: csharp

    // 定义变量
    var myValue = string.Empty;
    var myVar = new Variable()
    {
        Id = "我的变量标识符，这个用来代表我的变量",
        Name = "我的变量名字，用于展示",
        Description = "我的变量介绍，简单介绍一下你的变量",
        Value = Variable.ConvertFrom(myValue),
    };

    // 注册变量
    var dataSource = scope.Resolve<IDataSource>();
    dataSource.Register(myVar, StringValue.Default);

    // 发布变量更新
    myVar.Value = Variable.ConvertFrom("变量更新辣!");
    dataSource.Publish(myVar);


监听DataSource变量更改
-----------------------

Variable实现了 ``INotifyPropertyChanged`` 接口，我们在注册后可以直接监听更改：

.. sourcecode:: csharp

  myVar += PropertyChanged;

  private void PropertyChanged(object sender, PropertyChangedEventArgs e)
  {
      var myVar = sender as Variable;
      // do something
  }

如果拿不到Variable实例，我们可以监听 ``DataSource`` 的事件

.. sourcecode:: csharp

    var logger = scope.Resolve<ILogger<MyPlugin>>();
    var dataSource = scope.Resolve<IDataSource>();
    dataSource.OnMultiDataChanged += (variables) =>
    {
        for (var variable in variables)
        {
            logger.LogInformation("变量{variable.Name}更新辣，他的值是{variable.Value}");
        }
    }

Formatter的注册和监听
---------------------

Formatter负责注册Format，Format是数个变量组合起来的一个表达式，在变量更新时同时对表达式进行更新。

.. sourcecode:: text

  当前谱面 ${Artist}-${Title} PP: ${PP}

例如上述表达式，可以被Formatter进行格式化，结果可能是 ``当前谱面 jubeat(Ryu\*)-I'm so happy PP: 100.2``

要使用Formatter我们首先要注册Format，首先，我们需要一个实现了 ``IFormatContainer`` 接口的类。

.. sourcecode:: csharp

    using EmberKernel.Plugins.Components;
    public MyFormatterContainer : IComponent, IFormatContainer 
    {
        public ValueTask FormatUpdated(string format, string value)
        {
            // 如果注册的format有更新，这个函数会被调用
        }
    }

接下来我们注册Format和FormatContainer

.. sourcecode:: csharp

    public class MyPlugin : Plugin
    {
        public override void BuildComponents(IComponentBuilder builder)
        {
            // 在生命周期中添加实现了IFormatContainer的类
            builder.ConfigureComponent<MyFormatterContainer>();
        }

        public override ValueTask Initialize(ILifetimeScope scope)
        {
            var formatter = scope.Resolve<IFormatter>();
            formatter.Register<MyFormatterContainer>(scope, "1", "当前谱面 ${Artist}-${Title} PP: ${PP}");
        }

        public override ValueTask Uninitialize(ILifetimeScope scope)
        {
            var formatter = scope.Resolve<IFormatter>();
            formatter.Unregister<MyFormatterContainer>("1");
        }
    }

直接使用DataSource和Formatter还是比较麻烦的，我们可以配合一些工具来使用。

DataSource集成事件
---------------------

我们可以通过 :doc:`事件总线 (EventBus) <eventbus>` 对单个事件进行监听和变量管理。

举例一个非常实用的场景，游戏内存数据的抓取可以通过事件进行广播，此时可以用这个事件来注册DataSource。使用 ``ConfigureEventStatistic`` 进行注册：

.. sourcecode:: csharp

    public class BeatmapInfo : Event<BeatmapInfo>
    {
        [DataSourceVariable]
        public int BeatmapId { get; set; }

        [DataSourceVariable]
        public int SetId { get; set; }

        [DataSourceVariable(Id = "谱面文件名", Name = "谱面文件名", Description = "谱面文件名")]
        public string BeatmapFile { get; set; }

        [DataSourceVariable]
        public string BeatmapFolder { get; set; }
    }

    public class MyPlugin : Plugin
    {
        public override void BuildComponents(IComponentBuilder builder)
        {
            // 在生命周期中添加相关的帮助类
            builder.ConfigureEventStatistic<BeatmapInfo>();
        }

        public override ValueTask Initialize(ILifetimeScope scope)
        {
            // 我们使用快捷注册函数，来启动帮助类
            // 这里会把所有BeatmapInfo中标有[DataSourceVariable]的属性注册为变量
            scope.Track<BeatmapInfo>();
        }

        public override ValueTask Uninitialize(ILifetimeScope scope)
        {
            // 这里将所有变量反注册，从DataSource中移除
            scope.Untrack<BeatmapInfo>();
        }
    }

StatisticHub 的使用
-----------------------

``StatisticHub`` 自身作为一个FormatContainer的实现，可以直接注册和管理Format，并提对外提供当前Variable的访问。该服务设计主旨是为了对 DataSource 和 Formatter 做一层封装，外部使用者大部分情况直接使用这个类即可。

但要注意， StatisticHub 的主要目标还是 Format 的管理。


.. sourcecode:: csharp

    public class MyPlugin : Plugin
    {
      ...
        private void Hub_OnFormatUpdated(string name, string format, string value)
        {
        }
        public override ValueTask Initialize(ILifetimeScope scope)
        {
            // 直接使用 StatisticHub 
            var hub = scope.Resolve<IStatisticHub>();
            hub.OnFormatUpdated += Hub_OnFormatUpdated;
            hub.Register("1", "当前谱面 ${Artist}-${Title} PP: ${PP}");
        }

        public override ValueTask Uninitialize(ILifetimeScope scope)
        {
            // Uninitialize 时也需要反注册
            var hub = scope.Resolve<IStatisticHub>();
            hub.Unregister("1");
            hub.OnFormatUpdated -= Hub_OnFormatUpdated();
        }
    }

