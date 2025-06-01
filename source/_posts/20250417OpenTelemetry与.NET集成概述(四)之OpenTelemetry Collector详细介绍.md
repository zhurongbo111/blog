---
title: OpenTelemetry与.NET集成概述(四)之OpenTelemetry Collector详细介绍
date:  2025-04-17 19:37:33
categories:
 - 服务端
tags:
 - C#/.Net
 - OpenTelemetry
 - 链路追踪
 - 性能监控
description: 详细介绍OpenTelemetry Collector
permalink: /posts/13.html
---

# OpenTelemetry Collector的三种模式

在OpenTelemetry 3种收集可观测数据的方式，分别是No Collector，Agent和Gateway，接下来我们会着重介绍前2种方式

## No Collector 模式

这种模式是指应用程序通过集成 OpenTelemetry SDK，将遥测信号（追踪数据、指标、日志）直接导出到后端，无需经过 Collector 组件，是最简单的部署模式。
![](/images/otel5.png)

### 优缺点

- 优点

  - 使用简单，尤其适用于开发和测试环境。
  - 在生产环境中，无需操作额外组件。

- 缺点

  - 当收集、处理或摄入方式发生变化时，需要修改代码。
  - 应用代码与后端紧密耦合。
  - 每种语言实现的导出器数量有限。当前.Net语言还支持的导出器有如下几个：
    - OpenTelemetry.Exporter.Zipkin
    - OpenTelemetry.Exporter.Geneva
    - OpenTelemetry.Exporter.InfluxDB
    - OpenTelemetry.Exporter.Instana
    - OpenTelemetry.Exporter.OneCollector
    - OpenTelemetry.Exporter.Stackdriver

### 适用场景

该模式适合对架构复杂度要求较低、需要快速验证功能的场景，如开发测试阶段或简单的生产环境。但在需要灵活处理遥测数据、支持多后端或复杂路由的场景中，可能存在局限性。

### 示例

现在我们先使用docker安装Zipkin：

```bash
docker run -d -p 9411:9411 --name zipkin openzipkin/zipkin
```

为我们之前的QuickStart项目安装一个新的Nuget Package：

```
OpenTelemetry.Exporter.Zipkin
```

修改我们之前的项目，改成如下：

```c#
    static void ConfigureServices(IServiceCollection services)
    {
        // Add OpenTelemetry services here
        services.AddOpenTelemetry()
            .ConfigureResource(resourceBuilder => resourceBuilder.AddService("QuickStart")) // Replace with your application name
            .WithTracing(builder =>
                builder.AddAspNetCoreInstrumentation().AddConsoleExporter().AddZipkinExporter())
            .WithMetrics(builder =>
                builder.AddAspNetCoreInstrumentation().AddConsoleExporter())
            .WithLogging(builder =>
                builder.AddConsoleExporter()); 

    }
```

跟之前的代码比较，我们只是在WithTracing的配置方法中添加了一个`.AddZipkinExporter()`，改动非常小。这样的改动会把对应的链路追踪数据（Traces）发送到Zipkin中，可能你也已经注意到了Trace里，我们并没有删除.AddConsoleExporter()，这也就意味着Traces数据会同时发送到两个地方：控制台和Zipkin服务端。另外我们只修改了Trace，并没有修改Metrics和Logging，这也说明了这3部分数据是互不影响的，各自有各自的配置。让我们运行这个项目，稍等一会，我们就会发现Zipkin就有数据了。
![](/images/otel4.png)

## Agent 模式

Agent 模式指应用程序通过**OpenTelemetry SDK**（基于 OpenTelemetry 协议 OTLP），将遥测信号（追踪数据、指标、日志）发送至**与应用同机部署的 Collector 实例**（如 Sidecar 容器或守护进程）。核心流程：

```js
应用 / 下游 Collector → OTLP 协议 → Agent Collector → 处理后转发至后端(例如：Zipkin)
```

关键组件交互：

- **应用端**：SDK 配置为将 OTLP 数据发送至 Collector 地址（如环境变量指定端点）。
- **Collector 端**：接收数据后，通过配置的处理器（如批量处理）和导出器（如 Jaeger、Prometheus Remote Write）转发至一个或多个后端。

### 优缺点

- 优点

  - **入门简单**：架构清晰，适合快速搭建监控链路。

  - **1:1 映射关系**：应用与 Collector 一一对应，便于调试和单节点管理。

- 缺点

  - 可扩展性受限：

    - **人力成本**：当应用节点增多时，需逐一管理每个节点的 Collector 配置。

    - **负载压力**：单节点 Collector 可能成为性能瓶颈（如高并发数据摄入）。

  - 灵活性不足：难以支持复杂的数据路由、转换或多后端集成需求。

### **适用场景**

- **中小型系统**或**开发 / 测试环境**：适合需要快速验证监控功能、节点规模较小的场景。
- **同机数据处理**：如 Sidecar 模式下，Collector 与应用共享资源，降低跨节点通信开销。
- **局限性场景**：不适合大规模集群或需要集中式数据治理、灵活处理逻辑的生产环境。
