---
title: OpenTelemetry与.NET集成概述(三)之OpenTelemetry的在.Net中的常用组件
date:  2024-12-13 20:23:18
categories:
 - 服务端
tags:
 - C#/.Net
 - OpenTelemetry
 - 链路追踪
 - 性能监控
description: 详细介绍OpenTelemetry在.Net中常用的一些组件包以及一些常用的github仓库
permalink: /posts/12.html
---

# OpenTelemetry 在.NET 中常用的组件包与 GitHub 仓库介绍

在.NET 开发中，OpenTelemetry 提供了一系列实用的组件包，助力开发者高效实现应用程序的可观测性，同时相关的 GitHub 仓库也为开发者提供了丰富的资源和协作平台。主要的包分为以下几类

## OpenTelemetry核心包

这一类的包主要都是官方团队根据OpenTelemetry标准进行开发的包，包括如下：

- **OpenTelemetry.API**

  定义了 OpenTelemetry 的核心接口和抽象类，是整个框架的契约层。它不包含具体的实现逻辑，仅提供标准化的 API 定义，确保不同实现（如官方 SDK、社区扩展）之间的兼容性。

  - **标准化接口**：定义了追踪、指标、日志的核心接口，例如：
    - `TracerProvider`/`Tracer`：用于创建追踪跨度（Span）。
    - `MeterProvider`/`Meter`：用于创建指标（Counter/Gauge/Histogram 等）。
    - `LoggerProvider`/`Logger`：用于记录日志并关联追踪上下文。
   - **上下文传播**： 
     - 提供 `Propagator` 接口，用于在分布式系统中传播追踪上下文（如 HTTP 头中的 Trace ID），确保跨服务的链路连续性。
   - **资源抽象**：
     - 通过 `Resource` 类定义应用的元数据（如服务名、实例 ID），供所有信号（Traces/Metrics/Logs）共享。在之前的快速应用文章中的`ConfigureResource`这个方法就是基于此。

- **OpenTelemetry**

  是 OpenTelemetry for .NET 的核心包，封装了实现可观测性（追踪、指标、日志）的基础框架和核心逻辑。它提供了一套标准化的 API 和 SDK 实现，用于生成、处理和导出遥测数据，是集成 OpenTelemetry 功能的基石。它依赖于OpenTelemetry.API。

  - **统一的可观测性支持**：
    整合了追踪（Traces）、指标（Metrics）、日志（Logs）三大核心信号，允许开发者在同一个框架下管理应用的全链路观测数据。
    
  - **灵活的组件扩展**：
    支持通过添加不同的 **处理器（Processors）**、**导出器（Exporters）** 和 **检测工具（Instrumentations）** 扩展功能。例如，通过 `OpenTelemetry.Exporter.OpenTelemetryProtocol` 导出数据到 OTLP 兼容的后端（如 Jaeger、Prometheus）。
    
  - **资源元数据管理**：
    支持配置应用的资源信息（如服务名称、版本、环境等），这些信息会附加到遥测数据中，便于后端分析和过滤。
    
    

## OpenTelemetry自动检测包

OpenTelmetry.Instrumentation.* 是 OpenTelemetry 生态中用于**自动检测**的组件包，主要作用是简化应用程序对遥测数据（追踪、指标、日志）的收集流程。这类包通过拦截框架、库或运行时的关键操作，自动生成标准化的观测数据，避免开发者手动编写大量检测代码，从而降低集成成本。常用的有如下几个包：

- **OpenTelemetry.Instrumentation.Http**

  对.NET 应用中使用HttpClient发起的 HTTP 请求进行检测。它会为每个HttpClient请求生成追踪跨度（span），并记录请求的关键信息，如请求地址、请求方法、响应状态码等，有助于了解应用与外部服务之间的 HTTP 交互情况。它与OpenTelemetry.Instrumentation.AspNetCore配合就可以形成微服务之间完整的链路追踪。主要产生Trace和Metric数据。

- **OpenTelemetry.Instrumentation.AspNetCore**

  专门用于对ASP.NET Core 应用进行自动检测的组件包。它会自动拦截ASP.NET Core 应用中的 HTTP 请求和响应，生成对应的追踪数据，记录请求的处理路径、响应时间、错误信息等。无需开发者手动在业务代码中大量添加追踪代码，就能轻松获取应用的 HTTP 请求链路信息。主要产生Trace和Metric数据。

- OpenTelemetry.Instrumentation.Runtime

  是 OpenTelemetry 针对 **.NET 运行时（Runtime）** 的自动检测组件包，主要用于收集和暴露 .NET 应用程序的运行时指标（如内存、CPU、线程等）和追踪数据。它通过挂钩 .NET 运行时的底层接口，无需手动编写检测代码即可获取关键性能数据，是监控 .NET 应用健康状态的核心工具之一。主要产生Metric数据。

- OpenTelemetry.Instrumentation.EntityFrameworkCore

  该组件包主要用于自动检测基于 Entity Framework Core 的数据库操作，它能够拦截 EF Core 执行的数据库查询、保存更改等操作，自动生成对应的追踪数据和性能指标。例如，记录 EF Core 生成的 SQL 查询语句、执行时间、数据库连接信息，以及操作涉及的实体类型和数量等，并且将这些数据关联到整个应用的追踪链路中，便于开发者了解数据库操作在应用流程中的表现。主要产生Trace数据。注意：这个包不能和具体的数据库驱动的检测包一起用，比如：OpenTelemetry.Instrumentation.SqlClient

  

## OpenTelemetry导出包

在 OpenTelemetry 体系中，`OpenTelemetry.Exporter.*` 这类包是数据导出器（Exporter）的核心实现，主要用于将 OpenTelemetry 收集到的追踪数据（Traces）、指标数据（Metrics）、日志数据（Logs）从应用程序导出到后端可观测性平台（如 Jaeger、Prometheus、Grafana、Elasticsearch 等），是连接应用与观测平台的关键桥梁。

- **OpenTelemetry.Exporter.OpenTelemetryProtocol**

  此导出包是**第一优先**导出选项。任意可观测后端平台都推荐使用此组件包。此组件将连接另一个OpenTelemetry的核心组件：OpenTelemetry Collector。后面的文章我们会单独介绍这个组件。

  作为数据导出器，该组件包用于将 OpenTelemetry 收集到的追踪、指标和日志数据，按照 OpenTelemetry 协议（OTLP）格式发送到后端可观测性平台，如 Jaeger、Grafana Tempo 等。它提供了稳定可靠的数据传输通道，确保数据能够准确无误地导出。

- **OpenTelemetry.Exporter.Console**

  将 OpenTelemetry 收集的追踪数据（Traces）、指标数据（Metrics）、日志数据（Logs）以文本形式输出到控制台（Console），用于本地调试和开发阶段验证。

- **OpenTelemetry.Exporter.Zipkin**（不推荐）

  将追踪数据导出到 **Zipkin** 分布式追踪系统，用于服务链路分析、性能瓶颈定位。基于 **Zipkin 原生协议**（JSON 格式），通过 HTTP 请求发送到 Zipkin Server。此包会在应用程序内把OpenTelemetry 格式的数据转换成Zipkin支持的数据格式导出，因此也不推荐。

- **OpenTelemetry.Exporter.Jaeder**（已弃用）

  将追踪数据导出到 **Jaeger** 分布式追踪系统，用于大规模微服务架构的链路追踪、故障排查和性能优化。可将 OpenTelemetry 的 `Span` 转换为 Jaeger 的 `Span`（如映射 Tag、Event、Span 关系等），因此也不推荐。

## 其他组件包

- **OpenTelemetry.Extensions.Hosting-常用**

  该组件包是在.NET 应用程序中集成 OpenTelemetry 的关键入口，通过与.NET 的通用主机（IHost）集成，简化了 OpenTelemetry 的配置和启动流程。开发者可以方便地在应用启动时添加 OpenTelemetry 相关服务，如添加不同类型的处理器、导出器等，从而实现对追踪、指标和日志数据的处理与导出。

- **OpenTelemetry.Resources.***

  通过标准化的资源定义，为可观测性数据提供了统一的 “身份标识” 系统。合理使用这些包可以：

  - 增强分布式系统中服务间的关联性
  - 简化问题定位和根因分析
  - 实现跨团队、跨环境的统一监控

- **OpenTelemetry.Extensions.***

  是 OpenTelemetry 生态中一类**扩展功能包**，用于增强核心功能的灵活性、兼容性或集成能力。这类包通常不直接提供核心的追踪、指标或日志功能，而是作为 插件或桥梁”，帮开发者更便捷地与其他框架、库或生态系统集成，或为核心功能添加额外特性。



## 常用 GitHub 仓库

### 1. [opentelemetry-dotnet](https://github.com/open-telemetry/opentelemetry-dotnet)

- **核心价值**：这是 OpenTelemetry 官方的.NET 项目仓库，是.NET 开发者使用 OpenTelemetry 的核心资源库。仓库中包含了所有.NET 相关的 SDK、工具包的源代码，开发者可以在这里查看最新的代码实现、提交 issue 反馈问题或贡献代码。同时，仓库中详细的文档和示例代码，也为开发者学习和使用 OpenTelemetry 提供了极大的帮助。

- **用途**：开发者可以通过该仓库了解 OpenTelemetry 在.NET 中的最新功能和特性，参与项目开发和维护，也可以参考示例代码快速上手使用 OpenTelemetry 组件包。

### 2. [open-telemetry/opentelemetry-dotnet-contrib](https://github.com/open-telemetry/opentelemetry-dotnet-contrib)

- **核心价值**：这是社区驱动的补充仓库，主要包含一些非核心但同样实用的组件和工具，如对更多第三方库和框架的检测支持。它扩展了 OpenTelemetry 在.NET 生态中的应用范围，为开发者提供了更多定制化的解决方案。

- **用途**：当开发者需要对特定的第三方库或框架进行检测，而官方仓库未提供支持时，可以在该仓库中查找相关的组件和代码示例，或者提交需求推动新功能的开发。

### 3. [open-telemetry/opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector)

- **核心价值**：消除了为支持多种开源遥测数据格式（如 Jaeger、Prometheus 等）而运行、操作和维护多个代理 / 收集器的需求，通过单一代码库实现对多种数据格式的支持。

- **用途**：在搭建 OpenTelemetry 数据收集和处理管道时，开发者可以从该仓库中选择合适的组件，满足特定的业务需求，如将数据导出到特定格式的存储或进行特定规则的过滤和转换。

### 4. [open-telemetry/opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)

- **核心价值**：该仓库包含了 OpenTelemetry Collector 的众多扩展组件，如各种数据接收器、处理器和导出器。这些组件可以帮助开发者更灵活地处理和导出从.NET 应用收集到的遥测数据。

- **用途**：当OpenTelemetry Collector 仓库中没有相应的扩展插件时，可以去这个仓库查找，这个仓库的扩展更多更全面。

### 5. [open-telemetry/semantic-conventions](https://github.com/open-telemetry/semantic-conventions)

- **核心价值**：定义了一组通用的语义属性，为不同来源的数据赋予明确的含义，使得不同的 OpenTelemetry 实现和工具能够以一致的方式理解和处理数据。

- **用途**：当开发者定义自定义的可观察数据的metadata时，可以优先去这个仓库查找是否有合适的契约来满足自己的 业务需求。