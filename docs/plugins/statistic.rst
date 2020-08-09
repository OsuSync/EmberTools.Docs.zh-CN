数据统计
============

数据统计插件是在Kernel中数据统计相关基础设施之上开发的插件，实现了导出数据、持久化 Format、用户界面等。

MMF
---------------------------

MMF (MemoryMappedFile) 只输出 Format ，只需要用 Format 名称进行访问即可。

HTTP
-------

在程序运行之后即可通过HTTP接口访问相关数据

.. sourcecode:: sh

  # 获得当前所有变量和值
  curl --location --request GET 'http://localhost:11111/api/variable'

  # 获得当前所有format和值
  curl --location --request GET 'http://localhost:11111/api/format'

目前只支持获取。之后还会开放直接使用HTTP接口操作format的接口。

Websocket
-----------

Websocket方式可以进行变更通知

下列对应Endpoints

.. sourcecode:: sh

  ws://localhost:11111/ws/format
  ws://localhost:11111/ws/variable

在每个format更新时，都会推送更新的值，variable也相同。
