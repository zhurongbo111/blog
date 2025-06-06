---
title: OpenTelemetry与.NET集成概述(五)之OpenTelemetry Collector配置详解
date:  2025-02-18 20:03:28
categories:
 - 服务端
tags:
 - C#/.Net
 - OpenTelemetry
 - 链路追踪
 - 性能监控
 - ElasticSearch
 - Kibana
 - Jaeger
 - Zipkin
 - prometheus
 - grafana
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
      zipkin/nontls:
        endpoint: http://your-local-ip:9411/api/v2/spans # 导出至Zipkin后端
      elasticsearch:
        endpoint: http://your-local-ip:9200 # 导出至ElasticSearch后端
      otlphttp/jaeger:
        endpoint: http://your-local-ip:4328 # 导出至Jaeger后端
      otlphttp/prometheus:
        endpoint: http://your-local-ip/api/v1/otlp # 导出至prometheus后端
        tls:
          insecure: true
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
        exporters: [otlphttp/jaeger, zipkin/nontls]  # 导出至Jaeger和Zipkin
      metrics:
        receivers: [otlp]   		# 使用OTLP接收器
        processors: [batch]       # 批量处理
        exporters: [otlphttp/prometheus]  # 导出至prometheus
      logs:
        receivers: [otlp]			# 使用OTLP接收器
        processors: [batch]		# 使用OTLP接收器
        exporters: [elasticsearch]	# 导出至ElasticSearch
    telemetry:
      logs:                       # 配置Collector自身日志
        level: debug
      metrics:                    # 配置Collector自身指标
        level: basic
  ```
  
  这个配置非常重要，这里组装配置的地方来拼接：receiver,processor,exporter三个部分。

### **组件命名与复用**

- 同一类型组件可通过`type/name`格式创建多个实例（如`otlphttp/prometheus`、`otlphttp/jaeger`），并在不同管道中复用。
  - 组件的类型有`type`来确定，名称规则：`type[/name]`
- 管道支持多接收器、多处理器和多导出器组合，处理器顺序决定数据处理流程。

### 示例

接下来我们使用一个实际的配置来进行演示：

- 安装必要的组件

  - 使用docker安装Zipkin

    ```bash
    docker run --name zipkin -d -p 9411:9411  openzipkin/zipkin
    ```

  - 使用docker安装Jaeger

    ```bash
    docker run  --name jaeger -d -p 16686:16686 -p 4328:4318 jaegertracing/jaeger:2.6.0
    ```

    注意这里我把默认端口**4318**映射成了**4328**，为了和OpenTelemetry Collector端口避免冲突

    

  - 使用docker安装OpenTelemetry Collector，如果看过之前的系列文章已经安装的话，可以忽略

    ```bash
    docker run --name otel-collector -d -p 4317:4317 -p 4318:4318 -v C:\Users\yourname\config.yaml:/etc/otel-config.yaml otel/opentelemetry-collector-contrib:latest --config=/etc/otel-config.yaml
    ```

    对应的config.yaml文件的内容如下：

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
      transform:
        metric_statements:
            - replace_pattern(metric.name, "\\.", "_")
            - replace_all_patterns(resource.attributes, "key", "\\.", "_")
            - replace_all_patterns(datapoint.attributes, "key", "\\.", "_")
    
    exporters:
      debug:
        verbosity: detailed
      zipkin/nontls:
        endpoint: http://your-local-ip:9411/api/v2/spans
      elasticsearch:
        endpoint: http://your-local-ip:9200
      otlphttp/jaeger:
        endpoint: http://your-local-ip:4328
      otlphttp/prometheus:
        endpoint: http://your-local-ip:9090/api/v1/otlp
        tls:
          insecure: true
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlphttp/jaeger, zipkin/nontls]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlphttp/prometheus]
        logs:
          receivers: [otlp]
          processors: [batch]
          exporters: [elasticsearch]
      telemetry:
        logs:
          level: debug
        metrics:
          level: basic
    ```

  - 使用docker-compose安装ElasticSearch和Kibana

    - 创建docker-compose.yml文件，内容如下并运行：`docker-compose up -d`：

      ```yml
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

  - 使用docker安装prometheus

    ```bash
    docker run --name prometheus -d -p 9090:9090 prom/prometheus --config.file=/etc/prometheus/prometheus.yml --web.enable-otlp-receiver
    ```

    这里指定了参数`--web.enable-otlp-receiver`，这样prometheus才能接收otlp格式的数据

  - 使用docker安装grafana

    ```bash
    docker run -d --name grafana -p 3000:3000 grafana/grafana
    ```

    记得在grafana启动后配置prometheus的数据源

- 创建项目并安装必要依赖

  - 在Visual Studio中创建一个Asp.Net Core Web API项目，也可以使用我们之前系列文章中的QuickStart项目

  - 安装如下Nuget package

    - OpenTelemetry.Extensions.Hosting
    - OpenTelemetry.Instrumentation.AspNetCore
    - OpenTelemetry.Exporter.OpenTelemetryProtocol

  - 修改项目启动代码，往容器中注入必要的服务，类似这样：

    ```c#
            // Add OpenTelemetry services here
            services.AddOpenTelemetry()
                .ConfigureResource(resourceBuilder => resourceBuilder.AddService("QuickStart")) 
                .WithTracing(builder => builder.AddAspNetCoreInstrumentation().AddOtlpExporter())
                .WithMetrics(builder => builder.AddAspNetCoreInstrumentation().AddOtlpExporter())
                .WithLogging(builder => builder.AddOtlpExporter());
    ```

- 启动QuickStart项目，等待1分钟左右，依次打开遥测数据后端，就可以看到数据了。

### 查看遥测数据

#### 查看链路追踪数据

因为在配置中，我们把数据同时导出到了Jaeger和Zipkin，所以我们可以同时在这2个工具中查看到数据

- 在Jaeger中查看链路追踪（Traces）数据

  访问http://localhost:16686/ 就可以看到数据了

  ![](/images/jaeger1.png)

- 在Zipkin中查看链路追踪数据

  访问http://localhost:9411/

  ![](/images/zipkin1.png)

#### 查看指标数据（Metric）数据

- 访问[http://localhost:9090/](http://localhost:9090/)， 确保prometheus已经正常运行。

- 访问grafana，地址：[http://localhost:3000/](http://localhost:3000/)，默认用户名/密码：admin/admin

  - 在左边侧边栏依次访问：Connections->Data Sources，然后点击Add new data source

  - 选择Prometheus，在`Prometheus server UR`L中填入：[http://your-ip:9090](http://your-ip:9090) 不能是localhost:9090，因为2个docker容器不在一个网络中，不过可以使用本机的局域网地址

  - 最后点击Save & Test

  - 然后点击左边侧边栏的Explore，选择对应的选项，就可以看到数据了，如下图所示：

    ![](/images/grafana1.png)

#### 查看日志数据

- 访问kibana，地址：[http://localhost:5601/](http://localhost:5601/)，在首页选择访问Analytics

  ![](/images/es1.png)

- 在打开的页面中选择**Create a data view**

  ![](/images/es2.png)

- 在弹窗依次输入如下信息，点击Save data view to Kibana

  ![](/images/es3.png)

- 然后点击Discovery，输入相应的信息，就可以看到日志了

  ![](/images/es4.png)

请注意：本篇文章只是介绍了如何把遥测数据导入到不同的后端服务，并没有使用一些进阶用法。在实际的使用场景，还需要更多的配置来帮助更好的分析数据。

### 参考文章：

[Getting Started with Prometheus and Grafana](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/docs/metrics/getting-started-prometheus-grafana/README.md)

[Using Prometheus as your OpenTelemetry backend | Prometheus](https://prometheus.io/docs/guides/opentelemetry/#enable-the-otlp-receiver)

[Configuration — Jaeger documentation](https://www.jaegertracing.io/docs/2.6/configuration/)

[OpenTelemetry OTLP/HTTP Exporter](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/otlphttpexporter)

[OpenTelemetry Zipkin Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/zipkinexporter)

[OpenTelemetry Elasticsearch Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/elasticsearchexporter)
