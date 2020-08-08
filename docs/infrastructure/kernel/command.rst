命令模式 (Command)
===================

命令模式允许你在控制台与插件进行交互，在无法使用图形界面的情况下，命令模式就能发挥作用了。

示例参考

- `MyCommandComponent <https://github.com/OsuSync/EmberTools/blob/master/src/plugins/ExamplePlugin/Components/MyCommandComponent.cs>`_ 
- `PluginControlCommand <https://github.com/OsuSync/EmberTools/blob/master/src/plugins/ExamplePlugin/Components/PluginControlCommand.cs>`_ 

定义命令容器
-------------

命令行模式的命令接收方需要实现 ``ICommandContainer`` 接口，然后注册到生命周期中：

.. sourcecode:: csharp

  using EmberKernel.Plugins.Components;
  using EmberKernel.Services.Command.Components;

  public class MyCommandContainer : IComponent, ICommandContainer
  {
      public bool TryAssignCommand(CommandArgument argument, out CommandArgument newArgument)
      {

      }
  }

使用命令容器
------------

在插件生命周期构建时，可以注册命令我们的命令容器到生命周期，随后在初始化时将命令容器的实例传递给命令服务。

.. sourcecode:: csharp

  [EmberPlugin(...)]
  public class MyPlugin : Plugin
  {
      public override void BuildComponents(IComponentBuilder builder)
      {
          // 将命令容器注册到生命周期中
          builder.ConfigureCommandContainer<MyCommandComponent>();
      }

      public override ValueTask Initialize(ILifetimeScope scope)
      {
          // 将命令容器注册到命令服务中
          scope.UseCommandContainer<MyCommandComponent>();
      } 

      public override ValueTask Uninitialize(ILifetimeScope scope)
      {
          // 插件卸载时，从命令服务中移除命令容器
          scope.RemoveCommandContainer<MyCommandComponent>();
      }
  }

使用容器别名
---------------

默认情况下，命令服务使用命令容器实现类的名字作为命令前缀，一般而言这个名字会很长，命令服务允许定义别名。

.. sourcecode:: csharp

    [CommandContainerNamespace("my")]
    [CommandContainerAlias("m")]
    public class MyCommandComponent : IComponent, ICommandContainer
    {
        ...
    }

这样则可以直接使用 ``my`` 命令访问命令容器了。也可以使用 ``m`` 的简写

``CommandContainerNamespace`` 定义了命令容器的Namespace，Namespace不允许冲突，如果出现冲突则会抛出异常。

``CommandContainerAlias`` 定义了别名，别名允许冲突，采用先来后到原则，先注册先得。


定义命令
--------------

定义命令在普通方法上使用 ``CommandHandler`` Attribute即可

.. sourcecode:: csharp

    ...
    [CommandContainerAlias("m")]
    public class MyCommandComponent : IComponent, ICommandContainer
    {
        [CommandHandler(Command = "command")]
        [CommandAlias("c")]
        public void MyCommand()
        {
            // do something
        }
    }

同样的，命令也支持定义别名，此时使用 ``m command`` 即可访问该方法，也可以直接使用别名 ``c`` 访问。

命令自定义参数解析
-------------------

命令服务支持使用 ``CommandParser`` 属性定义命令的解析类，解析类需要实现 ``IParser`` 接口。

.. sourcecode:: csharp

    public class CustomParser : IParser
    {
        public IEnumerable<object> ParseCommandArgument(CommandArgument args)
        {
            if (int.TryParse(args.Argument, out var parsedInt)) yield return parsedInt;
            else yield return 0;
        }
    }

示例中的 ``CustomParser`` 只进行第一个参数的解析，并尝试parse为数字，如果失败则返回0。在命令定义上，我们可以直接使用这个解析类：

.. sourcecode:: csharp

  ...
    [CommandHandler(Command = "command")]
    [CommandAlias("c")]
    [CommandParser(typeof(CustomParser))]
    public void MyCommand(int myArg)
    {
        // do something
    }

使用组件
------------------

因为命令容器时注入到生命周期中的，即可以解析生命周期中的所有组件。

.. sourcecode:: csharp

    public class MyCommandComponent : IComponent, ICommandContainer
    {
        private readonly ILogger<MyCommandComponent> _logger;
        public MyCommandComponent(ILogger<MyCommandComponent> logger)
        {
            _logger = logger;
        }

        public void MyCommand(int myArg)
        {
            _logger.LogInformation($"2*{myArg}={2 * myArg}");
        }
    }

命令模式不是非常推荐使用，但可以支持跨平台的场景，因此也提供了相关的实现。
