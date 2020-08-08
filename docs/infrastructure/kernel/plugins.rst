插件模式 (Plug-ins)
=====================

插件加载的行为由第三方来控制，Kernel只提供抽象，Kernel本身不限制是何种插件才能注册到插件生命周期中。

在整个EmberTools中，插件的加载由EmberCore实现，并有如下要求：

- 继承了 ``EmberKernel.Plugins.Plugin`` 的访问标识符味 ``public`` 的类
- 含有 ``EmberPlugin`` Attribute的类

Core会使用Kernel中定义的Plugin、PluginDescriptor来进行插件的加载，详情可以参考 **EmberCore** 中的插件加载。

大部分情况，插件可以只依赖 ``EmberKernel`` 。

对于插件入门，可以参考 :doc:`开始你的第一个插件 <../../getting-started/index>` 。
