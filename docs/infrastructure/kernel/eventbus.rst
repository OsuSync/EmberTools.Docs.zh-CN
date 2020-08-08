事件总线 (EventBus)
====================

事件总线旨在为插件提供一个关键事件的广播服务，即一个事件广播，多个收听者监听。例如osu!游戏进程的启动事件，由一个插件进行广播，之后由多个插件监听，并进行相关的工作。

事件的定义
----------

我们使用POCO定义一个事件

.. sourcecode:: csharp

    public class ExamplePluginPublishEvent : Event<ExamplePluginPublishEvent>
    {
        public int InputNumber { get; set; }
    }

注意，事件必须继承 ``EmberKernel.Services.EventBus<>`` 类。默认使用类名作为标识符，我们可以使用 ``[EventNamespace("...")]`` 属性来标志该事件属于不同的Namespace，以区别于同名事件。

事件的发布
-----------

只需要Resolve出 ``IEventBus`` 接口即可。

.. sourcecode:: csharp

  ...
    public override ValueTask Initialize(ILifetimeScope scope)
    {
        var eventBus = scope.Resolve<IEventBus>();
        eventBus.Publish(new ExamplePluginPublishEvent() { InputNumber = 114514 });
    }

此时 ``ExamplePluginPublishEvent`` 事件将被广播，任何监听该事件的监听者都会收到通知。

事件的监听
-----------

监听事件需要存在于生命周期的组件进行监听，需要实现 ``IEventListener<T>`` 类，其中 ``T`` 是 ``Event`` 的实现。

也可以插件自身进行监听，也可以某个组件进行监听，也可以是专门的监听类进行监听，这里只要求能在当前Scope中被Resolve出来即可。

例如，在插件自身实例上进行监听（不推荐）：

.. sourcecode:: csharp

    public class MyPlugin : Plugin, IEventListener<ExamplePluginPublishEvent>
    {
        public override ValueTask Initialize(ILifetimeScope scope)
        {
            // 监听该事件
            scope.Subscription<ExamplePluginPublishEvent, MyPlugin>();
        }

        public override ValueTask Uninitialize(ILifetimeScope scope)
        {
            // 取消监听该事件
            scope.Unsubscriptio<ExamplePluginPublishEvent, MyPlugin>();
        }

        public ValueTask Handle(ExamplePluginPublishEvent @event)
        {
            if (@event.InputNumber == 114514)
            {
                _logger.LogInformation("hum, hum, hum, Aaaaaaaaaaaaaaaaaaaa. ");
            }
            return default;
        }
    }

注意：有始有终，监听了一个事件之后，也需要在对应生命周期函数中取消监听。

例如，在服务类中进行监听：

.. sourcecode:: csharp

    public class MyPlugin : Plugin, IEventListener<ExamplePluginPublishEvent>
    {
        public override ValueTask Initialize(ILifetimeScope scope)
        {
            // 监听该事件
            scope.Subscription<OsuProcessMatchedEvent, BeatmapDownloadService>();
            scope.Subscription<BeatmapDownloadAddressPrepared, BeatmapDownloadService>();
        }

        public override ValueTask Uninitialize(ILifetimeScope scope)
        {
            // 取消监听该事件
            scope.Unsubscriptio<OsuProcessMatchedEvent, BeatmapDownloadService>();
            scope.Unsubscriptio<BeatmapDownloadAddressPrepared, BeatmapDownloadService>();
        }
    }

    // 监听 osu!启动 的事件 和 Beatmap下载地址准备完成 的事件
    public class BeatmapDownloadService : IComponent,
        IEventHandler<OsuProcessMatchedEvent>,
        IEventHandler<BeatmapDownloadAddressPrepared>
    {
        public async ValueTask Handle(BeatmapDownloadAddressPrepared @event)
        {
            // 实现下载，在OsuProcessMatchedEvent发生之前，暂时不允许下载
            if (OsuGamePath == null) { return; }
        }


        public ValueTask Handle(OsuProcessMatchedEvent @event)
        {
            // 设置目标下载路径
            OsuGamePath = @event.GameDirectory;
            return default;
        }
    }

事件的别名
------------

有时候事件的命名会有冲突，导致接收到意外的事件，此时可以在事件定义上加上别名

.. sourcecode:: csharp

    [EventNamespace("介系偶独一无二的事件别名")]

事件模式一些实现建议
----------------------

如果确定事件会被多个插件共享，建议抽象出 Abstract 项目，事件订阅方和发布方都依赖于这个项目进行事件的发布和订阅。

这样做可以给双方带来固定的调用、别名约定，且解耦，我们只需要保证向下兼容性，即可保证插件之间的交互大部分时间都能够工作。

这种模式还可以包括事件以外的其他公共使用的一些组件。可以参考如下实现

- `BeatmapDownloader <https://github.com/OsuSync/EmberTools/tree/master/src/plugins/osu>`_
- `EmberMemoryReader <https://github.com/OsuSync/EmberTools/tree/master/src/plugins/osu>`_
- `Statistic <https://github.com/OsuSync/EmberTools/tree/master/src/plugins/statistic>`_
