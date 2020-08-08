==================
开始你的第一个插件
==================

开发要求
--------
- `.NET Core 3.1 <https://dotnet.microsoft.com/download/dotnet-core/3.1>`_ 
- `Ember Tools Repository <https://github.com/OsuSync/EmberTools>`_  

创建项目
--------
打开EmberTools项目文件夹 **EmberTools.sln** 文件即可看到代码结构一览。亦可使用 ``tree`` 等命令查看项目结构。

.. sourcecode:: text

  EmberTools
  ├─ build
  ├─ src
  │  ├─ EmberCore
  │  ├─ EmberKernel
  │  ├─ EmberWpfCore
  │  ├─ plugins
  │  │  ├─ ExamplePlugin
  │  │  ├─ osu
  │  │  │  ├─ EmberMemoryReader
  │  │  │  ├─ EmberMemoryReader.Abstract
  │  │  └─ statistic
  │  └─ share
  └─ test

项目架构大意如上，基本上“名副其实”，只需要按照字面意义去理解各个项目的功能即可。我们需要创建我们的插件，只需要在plugins文件夹找到合适位置即可。

新建工程
--------
在 **src/plugins** 或 **src/share** 文件夹下找到适合的路径，使用 ``dotnet new`` 命令或使用Visual Studio进行图形化操作。按照你的基础设施使用情况，分别可以新建：

- 基于.NET Core的Class library
- 基于.NET Core的WPF Class library
- 基于.NET Standard的Class libaray

需要再次强调的是，如果实现了任意图形界面则需要WPF Class library类型的工程。但是如果只是实现了ViewModel层，则仍能基于.NET Standard。

推荐新建完成之后，打开 **csproj** 文件，EmberKernel作为依赖，并增加如下编译Target：

.. sourcecode:: csproj

  <ItemGroup>
    <ProjectReference Include="..\..\..\EmberKernel\EmberKernel.csproj">
      <Private>false</Private>
      <CopyLocalSatelliteAssemblies>false</CopyLocalSatelliteAssemblies>
    </ProjectReference>
  </ItemGroup>
  <Target Name="PostBuild" AfterTargets="PostBuildEvent">
    <Exec Command="move $(OutDir)\$(TargetFileName) $(OutDir)\$(TargetName).dll&#xD;&#xA;mkdir $(SolutionDir)build\$(ConfigurationName)\plugins\$(ProjectName)&#xD;&#xA;copy $(OutDir)\*  $(SolutionDir)build\$(ConfigurationName)\plugins\$(ProjectName)" />
  </Target>

这个Target在大多数默认插件中都有，可以方便地将编译好的文件集成到 **build** 文件夹中

注意: 插件的加载是完全隔离的，如果携带了框架中的引用二进制文件，则在加载时插件不会被正确识别。所以在引用之后，需要将程序集的 ``private`` 属性设置为 ``true``。

实现插件类
----------
新建一个类，继承 ``EmberKernel.Plugins.Plugin`` 类，并增加 ``[EmberPlugin]`` Attribute。

.. sourcecode:: csharp

    namespace ExamplePlugin
    {
        [EmberPlugin(Author = "ZeroAsh", Name = "ExamplePlugin", Version = "1.0")]
        public class MyPlugin : Plugin
        {
            public override void BuildComponents(IComponentBuilder builder)
            {
            }

            public override ValueTask Initialize(ILifetimeScope scope)
            {
            }

            public override ValueTask Uninitialize(ILifetimeScope scope)
            {
            }
        }
    }

* ``BuildComponents`` 用来构建插件的生命周期，你可以使用 ``IComponentBuilder`` 类来向插件的生命周期注册服务。
* ``Initialize`` 用来调度插件开始工作，大多数服务应在此刻开始工作。在 ``BuildComponents`` 注册的组件在此时可以被Resolve。
* ``Uninitialize`` 用来调度插件停止工作，大多数用于工作的资源应在此刻释放。

注意：有始有终，在 ``Initialize`` 时注册和产生的资源，应在 ``Uninitialize`` 时回收。

使用全局的基础设施
-------------------

我们来尝试在插件的生命周期中解析一个日志类，并打印一条日志

.. sourcecode:: csharp

    using Microsoft.Extensions.Logging;
    ...
        public override ValueTask Initialize(ILifetimeScope scope)
        {
            var logger = scope.Resolve<ILogger<MyPlugin>>();
            logger.LogInformation("Hello world! Ember Tools");
        }

默认设置下，日志会自动持久化到硬盘中，也会在控制台中显式。我们可以修改 ``CoreAppSetting.json`` 文件中的格式化格式、持久化文件存储路径等。

所有基础设施可以通过 ``Initialize`` 的 ``ILifetimeScope`` 实例解析到，包括插件自身注册的相关服务。

注意：虽然可以拿到基础设施实例，但是无法拿到其他插件的实例。与其他插件的交互，需要通过其他设施来进行。

编译与运行
------------

由于在 **csproj** 文件中加入了 ``Target`` 选项，编译后会自动复制到 **build** 文件夹，之后运行 **EmberCore.exe** 或者使用 ``dotnet run`` 来运行。运行之后可以在控制台中发现我们刚才记录的Log。
