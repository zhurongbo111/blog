---
title: OpenTelemetry与.NET集成概述(四)之OpenTelemetry Collector配置详解
date:  2025-04-18 20:03:28
categories:
 - 服务端
tags:
 - C#/.Net
 - OpenTelemetry
 - 链路追踪
 - 性能监控
description: 详细介绍OpenTelemetry Collector配置文件
permalink: /posts/14.html
---

# OpenTelemetry Collector的配置文件详细介绍
   ![](/images/otel0.png)
### **配置文件核心结构**

Collector 配置由**管道组件**和**扩展组件**组成，通过`service`部分启用组件并定义数据流向。

#### **管道组件（Pipeline Components）**

- ​	负责处理遥测数据的全生命周期，包括：

  - Receivers（接收器）：从数据源收集数据（如 OTLP、Prometheus、Jaeger 等），支持推 / 拉模式。可以同时有多个Receiver，但是目前主推OTLP协议的接收器。

    ```yaml
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317  # 监听GRPC端口接收数据
          http:
            endpoint: 0.0.0.0:4318  # 监听Http端口接收数据
    ```

  - Processors（处理器）：对数据进行转换（如过滤、采样、批量处理）。

    ```yaml
    processors:
      batch:  # 批量处理以减少网络开销
      filter:
        traces:
          - 'attributes["container.name"] == "app_container_1"'  # 按属性过滤追踪数据
    ```

  - Exporters（导出器）：将数据发送至后端（如文件、Prometheus、Jaeger）。可以有多个Exporter。

    ```yaml
    exporters:
      otlp/jaeger:
        endpoint: jaeger-server:4317  # 导出至Jaeger后端
      zipkin/nontls:
        endpoint: "http://some.url:9411/api/v2/spans"
    ```

  - Connectors（连接器）（不常用）：连接两个管道，兼具导出器和接收器功能（如将追踪数据转换为指标）。

    ```yaml
    connectors:
      count:  # 统计符合条件的跨度事件并转为指标
        spanevents:
          my.prod.event.count:
            conditions: ['attributes["env"] == "prod"']
    ```

#### **扩展组件（Extensions）**

- ​	提供非数据处理功能（如健康检查、性能分析），需在`service.extensions`中启用

  ```yaml
  extensions:
    health_check:  # 健康检查端点
    pprof:         # 性能分析接口
    zpages:        # 调试页面
  service:
    extensions: [health_check, pprof, zpages]
  ```

### **Service 部分**

- 定义组件的启用和管道逻辑：

  ```yaml
  service:
    pipelines:
      traces:
        receivers: [otlp]         # 使用OTLP接收器
        processors: [batch]       # 批量处理
        exporters: [otlp/jaeger]  # 导出至Jaeger
      metrics:
        receivers: [prometheus]   # 使用Prometheus接收器
        exporters: [prometheusremotewrite]  # 远程写入Prometheus
    telemetry:
      logs:                       # 配置Collector自身日志
        level: debug
      metrics:                    # 配置Collector自身指标
        level: basic
  ```

  这个配置非常重要，这里组装配置的地方来拼接：receiver,processor,exporter三个部分。

### **组件命名与复用**

- 同一类型组件可通过`type/name`格式创建多个实例（如`otlp/2`、`batch/test`），并在不同管道中复用。
  - 组件的类型有`type`来确定，名称规则：`type[/name]`
- 管道支持多接收器、多处理器和多导出器组合，处理器顺序决定数据处理流程。

### 示例

- 继续我们之前的QuickStart项目，在之前的文章中我们已经启动了一个OpenTelemetry Collector容器，现在我们去修改这个容器的配置文件，改成如下配置：

  ```yaml
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
  
  processors:
    batch:
  
  exporters:
    debug:
      verbosity: detailed
    elasticsearch:
      endpoint: http://<your-local-address>:9200
  
  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [batch]
        exporters: [debug,elasticsearch]
      metrics:
        receivers: [otlp]
        processors: [batch]
        exporters: [debug,elasticsearch]
      logs:
        receivers: [otlp]
        processors: [batch]
        exporters: [debug,elasticsearch]
  ```

- 创建一个docker-compose.yml文件，内容如下：

  ```yaml
  version: '3.8'
  
  services:
    elasticsearch:
      image: docker.elastic.co/elasticsearch/elasticsearch:8.18.2
      container_name: elasticsearch
      environment:
        - xpack.security.enabled=false
        - discovery.type=single-node
        - ES_JAVA_OPTS=-Xms512m -Xmx512m
      volumes:
        - elasticsearch-data:/usr/share/elasticsearch/data
      ports:
        - "9200:9200"
      networks:
        - elastic
  
    kibana:
      image: docker.elastic.co/kibana/kibana:8.18.2
      container_name: kibana
      environment:
        - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      ports:
        - "5601:5601"
      depends_on:
        - elasticsearch
      networks:
        - elastic
  
  volumes:
    elasticsearch-data:
  
  networks:
    elastic:
      driver: bridge
  ```

- 再次启动QuickStart项目

- 访问http://localhost:5601/

  - 在首页选择**Elastic Search**
    ![](/images/es1.png)
  - 打开后在右侧点击**Manage**按钮
    ![](/images/es.png)
  - 在左侧的侧边栏找找到**Kibana -> Data Views**
  - 点击右上角的**Create data view**按钮
  - 就可以看到对应数据的Index，输入合适参数后创建
    - 这里建议分别创建trace, log, metric这3个与之对应index的data view
    ![](/images/es3.png)
  - 首页选择Analytics，再选择Discover
    ![](/images/es4.png)
    ![](/images/es6.png)
  - 在Data View下拉框中选择你刚刚创建的Data View，就可以显示相应的数据了。
    ![](/images/es5.png)

请注意：因为博主对elasticsearch和kibana都不甚熟悉，所以数据展现的时候不够优化。但是本篇的目的是为了说明，在进入OpenTelemetry Collector之后，可以通过exporter来配置输出数据的后端服务。这样的好处就是当我们想把数据切换到别的平台时候，就不需要更改我们的Application Code。另外在例子中，我把3种都输出到了debug和elasticsearch，希望大家不要被误导，3中数据的输出目标是可以不一致的，种类和数量都可以不一样，互不影响。

