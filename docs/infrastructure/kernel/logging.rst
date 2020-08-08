
日志记录 (Logging)
====================

在任意Scope均可使用日志服务，无需注册，只需在Scope中引入日志接口 ``ILogger<T>`` 即可。

.. sourcecode:: csharp

  ...
  public class MyPlugin : Plugin
  {
      public override ValueTask Initialize(ILifetimeScope scope)
      {
          var logger = scope.Resolve<ILogger<MyPlugin>>();
          logger.LogInformation("Hello world!");
      }
  }
  ...

日志由 ``Microsoft.Extensions.Logging`` 库统一封装，日志具体输出取决于构建 ``Kernel`` 时的基础设施。可以参考 `EmberCore 中的日志设施的注入 <https://github.com/OsuSync/EmberTools/blob/master/src/EmberCore/Program.cs>`_ ：

.. sourcecode:: csharp 

    new KernelBuild()
    .UseLogger((context, logger) =>
    {
        var config = context.Configuration.GetSection("Logging");
        var loggerFile = GetLoggerFilePath(config);
        logger
        .AddConfiguration(config)
        .AddSerilog((builder) => builder
            .Enrich.FromLogContext()
            .WriteTo.Console(
                theme: SystemConsoleTheme.Colored,
                outputTemplate: GetLoggerConsoleLogFormat(config))
            .WriteTo.File(
                path: loggerFile,
                rollingInterval: RollingInterval.Day,
                outputTemplate: GetFileLogFormat(config)))
        .AddDebug();
    })

**EmberCore** 使用了 **Serilog** 库作为扩展输出，将日志输出到控制台并持久化到文件。

