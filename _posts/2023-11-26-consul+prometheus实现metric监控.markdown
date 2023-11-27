---
title: 图文分享系统5：可观测性——consul+prometheus实现metrics监控
date: 2023-11-26 22:00:00 +0800
categories: [系统设计]
tags: [consul, prometheus, grafana]
author: stephan
mermaid: true
---

## 背景
在这之前，服务注册用的nacos，当初选用nacos主要因为：
1. 服务注册和配置中心在一块，使用起来方便
2. 官方提供docker-compose，开箱即用

之所以在这里摒弃nacos，选用consul做服务注册与发现，有以下几个因素考虑：
1. nacos需要额外引入存储如mysql，consul不需要，存储集成在server agent
2. nacos java开发，consul go开发，consul更贴合项目技术栈
3. consul除服务注册发现、配置中心外，还具有服务治理能力，可扩展性更强
4. consul集群部署，更云原生，k8s、prometheus等都支持

consul官网：[consul](https://developer.hashicorp.com/consul/docs/intro)，prometheus支持使用consul来做服务发现，只需要在prometheus中添加consul服务发现相关的配置，无需再填写指标收集的服务地址即可自动收集metrics，在某些场景比较适用。

这一章节重点来聊聊服务可观测性的另一块：metrics，metrics一般是一些服务的关键性指标如：请求总数，error总数，接口延时等等，通过这些指标，搭配一些可视化工具，可以对服务性能有一个直观的了解。prometheus是一个优秀的开源组件，通过prometheus可以轻松实现系统监控和告警，应用系统收集metrics，prometheus周期性从服务端pull（或者主动推送metrics到prometheus）metrics并存储，通过grafana来对指标进行可视化展示，prometheus整体系统架构如下：
![avatar](/assets/img/nov/prometheus.jpg)

下面先简单介绍服务如何用consul来做服务注册并和apisix打通
## consul服务注册与发现
1. 进入项目目录，执行docker compose，启动服务运行所需要的组件和consul server：
    ```
    cd example && docker compose up -d
    ```
    > 注意：apisix要想发现consul上的服务，需要在apisix启动前加上consul服务地址，在apisix配置文件：example/conf/apisix/apisix.yaml增加如下配置：
    {: .prompt-danger }
    ```
    discovery:
      consul:
        servers:
          - "http://consul-server:8500"
    ```
    具体配置参考：[apisix通过consul服务发现](https://apisix.apache.org/zh/docs/apisix/discovery/consul/)

    另外apisix-dashboard docker暂时没支持consul，只支持了consul_kv，可以配置的时候选择consul_kv, 配置完成后手动更改成consul:
    ![avatar](/assets/img/nov/apisix-consul.jpg)
2. 浏览器输入`http://127.0.0.1:8500/`，检查consul正常启动
3. 在consul控制台，增加程序启动所需的k/v配置，具体配置项参考github readme：[readme](https://github.com/Monstergogo/beauty-share)
4. **代码层改动**，consul支持api和cli两种方式注册服务，这里在代码中通过请求api的方式来注册和deregister服务：在服务正常启动后，请求consul服务注册api，将服务注册到consul，主要代码如下：
    ```go
    type MicroServer struct {
        GinServer    *gin.Engine
        ConsulServer *app.ConsulServiceImpl
    }

    func (m MicroServer) registerService() error {
        ip, err := util.GetOutboundIP()
        if err != nil {
            return err
        }
        grpcServicePayload := app.RegisterPayload{
            Address: ip.String(),
            Name:    util.GrpcServiceName,
            Port:    util.GrpcServerPort,
            Tags:    []string{"share", "v1"},
            Meta:    map[string]string{"version": "0.1.1", "service_type": "grpc"},
            Check: map[string]interface{}{
                "DeregisterCriticalServiceAfter": "90m",
                "HTTP":                           fmt.Sprintf("http://%s:%d/v1/ping", ip.String(), util.GinServerPort),
                "Interval":                       "10s",
                "Timeout":                        "5s",
            },
        }
        metricHttpServicePayload := app.RegisterPayload{
            Address: ip.String(),
            Name:    util.HttpServiceName,
            Port:    util.GinServerPort,
            Tags:    []string{"share-http", "v1"},
            Meta:    map[string]string{"version": "0.1.1", "service_type": "gin"},
        }

        var eg errgroup.Group
        eg.Go(func() error {
            err := m.ConsulServer.RegisterService(grpcServicePayload)
            return err
        })

        eg.Go(func() error {
            err := m.ConsulServer.RegisterService(metricHttpServicePayload)
            return err
        })
        if err = eg.Wait(); err != nil {
            return err
        }
        return nil
    }
    ```
    > 正常来说，注册grpc服务就行，上面额外注册了一个http服务，主要是给prometheus自动发现服务pull metrics数据的，这个后续详细说
    {: .prompt-warning }

   程序退出时，取消注册：
   ```go
    func (m MicroServer) deregisterService() error {
        var eg errgroup.Group
        eg.Go(func() error {
            err := m.ConsulServer.DeregisterService(util.GrpcServiceName)
            return err
        })

        eg.Go(func() error {
            err := m.ConsulServer.DeregisterService(util.HttpServiceName)
            return err
        })
        if err := eg.Wait(); err != nil {
            return err
        }
        return nil
    }
   ```
5. 本地启动服务，打开consul ui：http://127.0.0.1:8500/ui/dc1/services, 查看服务share.ShareService和share-service-http-metric两个服务是否正常注册：
![avatar](/assets/img/nov/consul-services.jpg)
到此，服务正常注册到consul，可在apisix ui: 浏览器输入: http://127.0.0.1:9005/, username: admin, password: admin 新建路由，upstream选consul测试接口，我的测试路由配置如下：

    ```json
    {
    "uri": "/grpc/test",
    "name": "http-grpc-test",
    "methods": [
        "GET"
    ],
    "plugins": {
        "grpc-transcode": {
        "_meta": {
            "disable": false
        },
        "method": "GetShareByPage",
        "proto_id": "1",
        "service": "share.ShareService"
        },
        "request-id": {
        "_meta": {
            "disable": true
        },
        "header_name": "traceId",
        "include_in_response": true
        }
    },
    "upstream": {
        "timeout": {
        "connect": 5,
        "send": 5,
        "read": 5
        },
        "type": "roundrobin",
        "scheme": "grpc",
        "discovery_type": "consul",
        "pass_host": "pass",
        "service_name": "share.ShareService",
        "keepalive_pool": {
        "idle_timeout": 60,
        "requests": 1000,
        "size": 320
        }
    },
    "status": 1
    }
    ```
   测试命令：`curl http://127.0.0.1:9080/grpc/test\?page_size\=10`

## 生成telemetry metrics数据到prometheus
opentelemetry可以生成metrics数据并将数据导入到prometheus，prometheus可以通过consul自动发现需要收集metics的服务，需要在prometheus增加服务发现配置，以本服务为例：
```yaml
scrape_configs:
  - job_name: share-service-exporter
    metrics_path: /metrics
    scheme: http
    consul_sd_configs:
      - server: consul-server:8500    # consul server地址
        services:
          - share-service-http-metric
```
在consul_sd_configs中，配置consul服务器地址和上报metics的services名称（名称和consul ui上的名称一致），这样，prometheus可以自动发现要pull的metrics服务地址，周期性从服务拉取metrics。

自定义metrics数据上报到prometheus主要步骤：
1. 通过opentelemetry sdk 生成metrics数据，第一步需要初始化MeterProvider：
   ```go
   var (
	Shutdowns []func(context.Context) error
	Meter     otelmetric.Meter
   )

   func initMeterProvider(res *resource.Resource) {
	exporter, err := prometheus.New()
	if err != nil {
		panic(err)
	}

	meterProvider := metric.NewMeterProvider(metric.WithResource(res),
		metric.WithReader(exporter))
	otel.SetMeterProvider(meterProvider)
	Meter = otel.Meter("share")
	Shutdowns = append(Shutdowns, meterProvider.Shutdown)
   }
   ```
    这里通过prometheus.New()来初始化一个exporter，具体参考[prometheus exporter](https://opentelemetry.io/docs/instrumentation/go/exporters/#prometheus-experimental)


2. 自定义两个metric: 
    - gin api的请求总数：apiCounter
    - gin api接口的请求耗时直方图：histogram

   在代码中初始化如下：
   ```go
   var (
	apiCounter metric.Int64Counter
	histogram  metric.Int64Histogram
   )

   func initMetricKind() {
	var err error
	apiCounter, err = tracing.Meter.Int64Counter(
		"gin_api_counter_total",
		metric.WithDescription("Number of API calls."),
		metric.WithUnit("{call}"),
	)
	if err != nil {
		panic(err)
	}
	histogram, err = tracing.Meter.Int64Histogram(
		"gin_api_duration",
		metric.WithDescription("The duration of api execution."),
		metric.WithUnit("ms"),
	)
	if err != nil {
		panic(err)
	}
   }
   ```

3. 生成一个gin拦截器，每次接口请求apiCount + 1并记录接口耗时：
   ```go
   func metricMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		apiCounter.Add(c, 1)
		start := time.Now()
		c.Next()
		duration := time.Since(start)
		histogram.Record(c, duration.Milliseconds())
	}
   }
   ```
4. gin加入拦截器，并开放一个/metrics接口，让prometheus pull metrics:
   ```go
   var ginServer  *gin.Engine

   ginServer = gin.Default()
   ginServer.Use(metricMiddleware())

   // prometheus metrics
   ginServer.GET("/metrics", gin.WrapH(promhttp.Handler()))
   ```

5. docker compose 启动prometheus、jaeger等:
   ```
   docker compose -f observability/docker-compose-all-in-one.yml up -d
   ```
   这块主要是启动可观测性的一些组件，不熟悉的可以参考之前文章有详细介绍

5. 在本地启动服务，一段时间后(consul健康检查会周期请求gin ping接口，prometheus周期性请求/metrics接口)，可在grafana浏览器打开: http://127.0.0.1:3000/explore, 查看自定义的两个metric（还有许多自动生成的metrics，这个是使用开源的一些组件自动生成的，可以先不管）：
![avatar](/assets/img/nov/metrics-manual.jpg)


## 将上报的两个指标通过grafana生成图表：
1. 浏览器输入：http://127.0.0.1:3000/, 打开grafana
2. 点击Dashboards, 新建一个dashboard
3. 新建一个gauge panel，用来统计gin api的总请求量
4. 新建一个Time series panel，统计接口的平均耗时,主要配置如下：
![avatar](/assets/img/nov/grafana-config.png)
平均请求耗时计算公示：
   ```
   rate(gin_api_duration_milliseconds_sum{job="share-service-exporter"}[1m])/rate(gin_api_duration_milliseconds_count{job="share-service-exporter"}[1m])
   ```
5. 最终dashboard:
   ![avatar](/assets/img/nov/grafana-dashboard.jpg)
   耗时主要是两个接口：v1/ping和/metrics，耗时都比较低，单位ms

### 本博客对应代码仓库：[github 地址](https://github.com/Monstergogo/beauty-share), **分支：feature/observer-consul**