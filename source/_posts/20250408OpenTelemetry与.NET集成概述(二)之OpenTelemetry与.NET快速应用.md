---
title: OpenTelemetry与.NET集成概述(二)之OpenTelemetry与.NET快速应用
date:  2025-04-08 20:38:55
categories:
 - 服务端
tags:
 - C#/.Net
 - OpenTelemetry
 - 链路追踪
 - 性能监控
description: 介绍.NET中快速集成OpenTelemetry的方式
permalink: /posts/11.html
---

### 创建一个Asp.Net Core API应用

本系列的的项目都是基于Asp.Net Core创建的应用，基于.Net Framework的传统项目基本也都是一样的，因为OpenTelemetry整个框架的构建其实都是基于`Micosoft.Extensions.DependencyInjection`这个扩展包展开，而这个包也是适用于.Net Framework的，所以这里就不单独展开介绍.Net Framework的使用方法。
打开Visual Studio去创建一个Asp.Net Core WebAPi，注意把Use OpenAPI勾选框勾上，方便可视化。创建完以后安装以下几个Nuget包：

```
OpenTelemetry.Extensions.Hosting
OpenTelemetry.Instrumentation.AspNetCore
OpenTelemetry.Exporter.Console
```

然后在容器中添加如下代码：

```c#
    static void ConfigureServices(IServiceCollection services)
    {
        // Add OpenTelemetry services here
        services.AddOpenTelemetry()
            .ConfigureResource(resourceBuilder => resourceBuilder.AddService("QuickStart")) // Replace with your application name
            .WithTracing(builder =>
                builder.AddAspNetCoreInstrumentation().AddConsoleExporter())// Export traces to console for demonstration purposes
            .WithMetrics(builder =>
                builder.AddAspNetCoreInstrumentation().AddConsoleExporter())
            .WithLogging(builder =>
                builder.AddConsoleExporter()); 

    }
```

启动项目。你就会看到在控制台中有类似以下的输出，俺就说明成功了。

日志输出：
![](/images/otel1.png)

链路追踪（Traces）输出：
![](/images/otel3.png)
指标（Metrics）输出：
![](/images/otel2.png)

当然在实际的项目当中我们并不会把信息输出到控制台，而是相应的后端服务。这里只是为了快速演示OpenTelemetry与.Net如何集成的罢了，后续的文章中我们会介绍如何把这些信息输出到相应的服务中去，会相当的简单。