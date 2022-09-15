# 可观测性

Observability lets us understand a system from the outside, by letting us ask questions about that system without knowing its inner workings. 



![2](2.png)



我们先看看一个典型服务问题排查过程是怎样的：

- 通过各式各样预设报警发现异常（Metrics/Logs）
- 打开监控大盘查找异常现象，并通过查询找到异常模块（Metrics）
- 对异常模块以及关联日志进行查询分析，找到核心的报错信息（Logs）
- 通过详细的调用链数据定位到引起问题的代码（Tracing）

为了能够获得更好的可观测性或快速解决上述问题，Tracing、Metrics、Logs缺一不可。



![1.png](1.png)

与此同时，行业中已经有了丰富的开源及商业方案，其中包括：

- **Metric：**Zabbix、Nagios、Prometheus、InfluxDB、OpenFalcon、OpenCensus
- **Tracing：**Jaeger、Zipkin、SkyWalking、OpenTracing、OpenCensus
- **Logs：**ELK、Splunk、SumoLogic、Loki、Loggly。

有着五花八门的方案同时，各个方案也有着五花八门的协议格式/数据类型。不同的方案之间很难兼容/互通。与此同时，实际的业务场景中也会将各种方案混用，开发人员只能自己开发各类 Adapter 去兼容，

# Opentelemetry是什么？

作为 CNCF 的孵化项目，OpenTelemetry 由 OpenTracing 和 OpenCensus 项目合并而成，是一组产商无关的SDK、API 接口、工具，可用来收集、转换、发送数据到开源或者商业的可观测性后端。同时为众多开发人员带来 Metrics、Tracing、Logs 的统一标准，三者都有相同的元数据结构，可以轻松实现互相关联。

## 能做啥？

- 每种语言都有产商无关的库来支持自动和手动的测量
- 可支持多种部署方式，且与产商无关的二进制收集器
- 一个端到端实现产生，发射，收集，处理和导出telemetry数据
-  可通过配置将数据并行发送到多个目的地。  
- 开放标准语义约定，以确保供应商无关的数据收集  

## 不是啥？

OpenTelemetry 不是可观测性的后端，如Prometheus、Jaeger，不提供与可观测性相关的后端服务。可根据用户需求将可观测类数据导出到存储、查询、可视化等不同后端，如 Prometheus、Jaeger 、云厂商服务中。提供了可插拔的架构。

# Opentelemetry设计

![Cross cutting concerns](architecture.png)

客户端设计

The SDK implementation should include the following exporters:

- logs, metrics, trace
  - OTLP (OpenTelemetry Protocol).
  - Standard output (or logging) to use for debugging and testing as well as an input for the various log proxy tools.
  - In-memory (mock) exporter that accumulates telemetry data in the local memory and allows to inspect it (useful for e.g. unit tests).
- metrics
  - Prometheus.
- trace
  - Jaeger.
  - Zipkin.

![OpenTelemetry client Design Diagram](library-design.png)

collection设计



![Separate Collection Diagram](separate-collection.png)

![Unified Collection Diagram](unified-collection.png)

# Opentelemetry数据类型-Signals

## Trace

DAG

For example, the following is an example **Trace** made up of 6 **Spans**:

```text
Causal relationships between Spans in a single Trace

        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `child` of Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F]
```

Sometimes it’s easier to visualize **Traces** with a time axis as in the diagram below:

```text
Temporal relationships between Spans in a single Trace

––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··········································]
      [Span D······································]
    [Span C····················································]
         [Span E·······]        [Span F··]
```



### Trace Context

We identify Span Context using four major components: a **`traceID`** and **`spanID`**, **Trace Flags**, and **Trace State**.

**`traceID`** - A unique 16-byte array to identify the trace that a span is associated with

**`spanID`** - Hex-encoded 8-byte array to identify the current span

**Trace Flags** - Provides more details about the trace, such as if it is sampled

**Trace State** - Provides more vendor-specific information for tracing across multiple distributed systems. Please refer to [W3C Trace Context](https://www.w3.org/TR/trace-context/#trace-flags) for further explanation.



### Span

```json
{
  "trace_id": "7bba9f33312b3dbb8b2c2c62bb7abe2d",
  "parent_id": "",
  "span_id": "086e83747d0e381e",
  "name": "/v1/sys/health",
  "start_time": "2021-10-22 16:04:01.209458162 +0000 UTC",
  "end_time": "2021-10-22 16:04:01.209514132 +0000 UTC",
  "status_code": "STATUS_CODE_OK",
  "status_message": "",
  "attributes": {
    "net.transport": "IP.TCP",
    "net.peer.ip": "172.17.0.1",
    "net.peer.port": "51820",
    "net.host.ip": "10.177.2.152",
    "net.host.port": "26040",
    "http.method": "GET",
    "http.target": "/v1/sys/health",
    "http.server_name": "mortar-gateway",
    "http.route": "/v1/sys/health",
    "http.user_agent": "Consul Health Check",
    "http.scheme": "http",
    "http.host": "10.177.2.152:26040",
    "http.flavor": "1.1"
  },
  "events": [
    {
      "name": "",
      "message": "OK",
      "timestamp": "2021-10-22 16:04:01.209512872 +0000 UTC"
    }
  ]
}
```

events打印一些日志，比如请求和响应体的内容；

links表示trace的关联，比如新启动一个异步的任务

## Metric

Metric 是关于一个服务的度量，在运行时捕获。从逻辑上讲，捕获其中一个量度的时刻称为 Metric event，它不仅包含量度本身，还包括获取它的时间和相关元数据。应用和请求指标是可用性和性能的重要指标。自定义指标可以深入了解可用性如何影响用户体验和业务。自定义 Metrics 可以深入理解可用性 Metrics 是如何影响用户体验或业务的。OpenTelemetry 目前定义了三个 Metric 工具：

- **counter:** 一个随时间求和的值，可以理解成汽车的里程表，它只会上升。
- **measure:** 随时间聚合的值。它表示某个定义范围内的值。
- **observer:** 捕捉特定时间点的一组当前值，如车辆中的燃油表。

https://opentelemetry.io/docs/reference/specification/metrics/datamodel/

## Log

日志是带有时间戳的文本记录，可以是带有元数据结构化的，也可以是非结构化的。虽然每个日志都是独立数据源，但可以附加到 Trace 的 Span 中。日常使用调用时，在进行节点分析时出伴随着也可看到日志。

https://opentelemetry.io/docs/reference/specification/logs/overview/#limitations-of-non-opentelemetry-solutions

## Baggage

OpenTelemetry 还提供了 Baggage 来传播键值对。Baggage 用于索引一个服务中的可观察事件，该服务包含同一事务中先前的服务提供的属性，有助于在事件之间建立因果关系。虽然 Baggage 可以用作其他横切关注点的原型，但这种机制主要是为了传递 OpenTelemetry 可观测性系统的值。这些值可以从 Baggage 中消费，并作为度量的附加维度，或日志和跟踪的附加上下文使用。

# Opentelemetry Collection

Collector提供了产商无关的数据接受、处理、导出的telemetry的数据能力，不需要运行多个Collector来支持多个开源的数据格式后端如prometheus和jaeger。

## 部署方式

提供了一个二进包两种部署方式

agent：与应用程序一起运行或与应用程序在同一主机上运行的Collector实例（二进制、sidecar、daemonset）

gateway：一个或多个Collector实例作为一个独立的服务(例如container or deployment)运行，通常为每个集群、数据中心或区域。

## 组件

收集器由以下组件组成:

receivers:如何将数据放入收集器;这些可以是推或拉的

processors:如何处理接收到的数据

exporters:将接收到的数据发送到哪里;这些可以是推或拉的

这些组件是通过pipeline启用的。可以通过YAML配置定义组件和管道的多个实例。

# 示例

https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/examples/demo

![demo-arch](demo-arch.png)

```shell
yum install docker
yum install docker-compose
git clone git@github.com:open-telemetry/opentelemetry-collector-contrib.git
cd examples/demo
docker-compose up -d #需要修改Dockerfile中的env变量为阿里的go镜像
```

访问地址

jaeger:  http://39.105.101.198:16686/

zipkin:  http://39.105.101.198:9411/

prometheus:  http://39.105.101.198:9090/
