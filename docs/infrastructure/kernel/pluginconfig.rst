插件配置 (Plug-ins Configuration)
===================================

Kernel提供了配置文件的直接读写的组件，可以在插件生命周期开始时注册。

定义配置类
--------------

我们可以直接写一个POCO来定义我们的配置文件映射类型。

.. sourcecode:: csharp 

    public class MyPluginConfiguration
    {
        public int MyIntValue { get; set; }
        public string MyStringValue { get; set; }
        public string LatestBeatmapFile { get; set; }
    }

配置文件和配置文件类虽然支持多层嵌套，但是我们推荐只用一层。

注册到生命周期
---------------

定义好配置类，我们就可以在 ``BuildComponents`` 中将其注册到生命周期里。

.. sourcecode:: csharp

    ...
    public override void BuildComponents(IComponentBuilder builder)
    {
        builder.UsePluginOptionsModel<MyPlugin, MyPluginConfiguration>();
    }

其中

- ``MyPlugin`` 是当前的插件类，用于将该Namespace下这个配置文件的写权限绑定到当前插件生命周期中。
- ``MyPluginConfiguration`` 则为刚才定义的配置类。

读取当前配置
--------------

在注册到生命周期之后，即可进行读取配置操作了，先从生命周期中解析出配置类

.. sourcecode:: csharp

  ...
    public override ValueTask Initialize(ILifetimeScope scope)
    {
        var logger = scope.Resolve<ILogger<MyPlugin>>();
        var options = scope.Resolve<IPluginOptions<MyPlugin, MyPluginConfiguration>>();
        var config = options.Create();
        logger.LogInformation($"当前设置的值为：MyIntValue={config.MyIntValue} MyStringValue={config.MyStringValue}");
    }

更新当前配置
--------------

有些时候我们需要进行配置的更新写入，此时可以继续用我们上面解析的 ``IPluginOptions<>`` 进行操作

.. sourcecode:: csharp

    var currentValue = options.Create();
    currentValue.MyIntValue += 1;
    // 将新值写入配置文件中
    await options.SaveAsync(currentValue)

读取其他命名控件的配置
-----------------------

自读自写基本上能够覆盖大部分场景了，有些情况下需要读取其他插件注册的配置文件，此时使用 ``UseConfigurationModel`` 来将要读取的配置注册到生命周期中

.. sourcecode:: csharp

    ...
    public override void BuildComponents(IComponentBuilder builder)
    {
        builder.UseConfigurationModel<MyPluginConfiguration>("MyPlugin");
    }

注册完成之后可以解析 ``IReadOnlyPluginOptions<T>`` 进行配置的读取

.. sourcecode:: csharp

    var options = scope.Resolve<IReadOnlyPluginOptions<MyPluginConfiguration>>();
    var currentValue = options.Create();

    // do someting


