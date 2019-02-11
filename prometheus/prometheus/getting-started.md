---
title: "[文档翻译] prometheus入门"
date: 2018-11-16
tags: ["prometheus", "文档翻译"]
categories: ["文档翻译", "prometheus"]
draft: false
---

原文链接：[GETTING STARTED](https://prometheus.io/docs/prometheus/latest/getting_started/)

这是一个“hello-world”类型的教程，使用一个简单的例子来展示如何安装、配置和使用prometheus。你将会在示例中下载和本地运行prometheus，配置它监控自己，然后使用查询、规则和图来使用收集到的时间序列数据。

## 下载和运行prometheus

[Download the latest release](https://prometheus.io/download) 下载对应系统的安装包，然后解压和运行：

```bash
tar xvzf prometheus-*.tar.gz
cd prometheus-*
```

在运行prometheus前，我们需要配置它

## 配置prometheus监控自己

prometheus通过监控目标上HTTP指标接口来收集目标的指标数据。由于prometheus也使用相同的方式暴露它本身的指标数据，所以它也可以收集和监控它本身。

然而，prometheus服务如果只用了收集自身数据，并没什么作用，但可以用来作为一个简单的例子。将下面的prometheus的基本配置保存到prometheus.yml文件中：

```yaml
global:
  scrape_interval:     15s # 默认值，每15秒收集一次目标数据

  # 在与外部系统通信时，如其它集群、远程存储或alertmanager，将这些标签附加到时间序列或告警中
  external_labels:
    monitor: 'codelab-monitor'

# 监控配置，包括一个需要监控的端点:
# 下面是prometheus自身的监控端点.
scrape_configs:
  # job_name将会添加`job=<job_name>`的标签到该配置收集的所有时间序列中
  - job_name: 'prometheus'

    # 覆盖全局的scrape_interval，即每5秒收集一次目标数据
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
```

可参阅[configuration documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)获取完整的配置选项定义。

## 启动prometheus

进入包含prometheus二进制文件的目录，使用新创建的配置文件，启动prometheus：

```bash
# 启动prometheus
# prometheus默认使用./data存储数据(可使用参数 --storage.tsdb.path 指定）
./prometheus --config.file=prometheus.yml
```

prometheus已经启动，你可以通过 [localhost:9090](http://localhost:9090/)访问它的状态页面。给它几秒钟，等待它收集自己HTTP指标端点的数据。

你还可以访问[localhost:9090/metrics](http://localhost:9090/metrics)来检查prometheus提供的自身指标的服务是否在运行。

## 使用浏览器查询

让我们来看看prometheus收集到的关于自身的一些数据。通过访问 [localhost:9090/graph](http://localhost:9090/graph) 来访问prometheus内置浏览器服务，在graph标签中选择“console”。

通过 [localhost:9090/metrics](http://localhost:9090/metrics) 可知，prometheus关于自身的一个指标：prometheus_target_interval_length_seconds(收集目标数据的实际时间间隔)。在console的表达式框中输入这个指标：

```bash
prometheus_target_interval_length_seconds
```

这会返回一系列不同的时间序列(含有每个记录的最新值)，每个序列包含指标名称：prometheus_target_interval_length_seconds，但标签不同。这些标签指定不同的延迟百分位数和目标组间隔。

如果我们只对第99个延迟百分位感兴趣，我们可以使用下面的查询语句：

```pr
prometheus_target_interval_length_seconds{quantile="0.99"}
```

获得返回时间序列的计数：

```:nine:
count(prometheus_target_interval_length_seconds)
```

可以参考[expression language documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/)获取更多关于查询语言的详细信息

## 使用图形界面

访问来 [localhost:9090/graph](http://localhost:9090/graph) 来使用图查询，选择“Graph”标签。

例如，使用下面语句可以画出prometheus自身每秒申请块的速率：

```bash
rate(prometheus_tsdb_head_chunks_created_total[1m])
```

请尝试使用图形范围参数和其它选项

## 启动其它示例目标程序

接下来我们尝试启动更多的示例目标程序来供prometheus监控。

Go客户端库包含有个例子，它会暴露三个服务的虚拟RPC延迟不同分布。

确认你已经正确安装了Go编译器（[Go compiler installed](https://golang.org/doc/install)）和正确配置Go编译环境（[working Go build environment](https://golang.org/doc/code.html)）。

下载Prometheus的Go客户端库，然后运行三个该示例程序：

```bash
# Fetch the client library code and compile example.
git clone https://github.com/prometheus/client_golang.git
cd client_golang/examples/random
go get -d
go build

# Start 3 example targets in separate terminals:
./random -listen-address=:8080
./random -listen-address=:8081
./random -listen-address=:8082
```

现在已经有了三个示例程序监听在： <http://localhost:8080/metrics>, <http://localhost:8081/metrics>, 和 <http://localhost:8082/metrics>。

## 配置prometheus来监听示例程序

现在我们来配置prometheus来监听这些新的目标程序。我们把这个三个端点组合成一个叫 example-random 的Job。其中，我们把前两个程序作为生产环境端点，第三个作为一个金丝雀实例。要在prometheus中实现这样的建模，我们可以把多个组的端点添加到一个Job中，再给每个组的端点添加额外的标签。在这个例子中，我们添加 group="production" 标签到第一组端点中，添加 group="canary" 到第二组端点中。

为了实现上面的功能，在prometheus.yml文件中，添加下面的Job定义到 scrape_configs 块中，然后重启prometheus程序：

```yaml
scrape_configs:
  - job_name: 'example-random'

    # 覆盖全局默认值，每5秒收集一次该Job下的监控端点的数据.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

现在可以去浏览器中验证prometheus是否包含这些示例程序暴露的时间序列数据，如 rpc_durations_seconds 指标。

## 配置将收集到的数据聚合到新的时间序列的规则

在一个特定设备中汇总一个包含上千个时间序列的查询可能会变得很慢，虽然在示例程序中不成问题。但为了让查询更高效，prometheus允许你通过配置记录规则，将表达式预先记录到全新的持久时间序列中。假设我么对五分钟内所有实例,基于Job和Service纬度的平均每秒的RPCs (rpc_durations_seconds_count) 感兴趣，我们可以这样写：

```bash
avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

尝试绘制该表达式。

将此表达式中生成的时间序列结果集记录到一个新的指标中：*job_service:rpc_durations_seconds_count:avg_rate5m*，将下来规则写入到 *prometheus.rules.yml* 文件中：

```yaml
groups:
- name: example
  rules:
  - record: job_service:rpc_durations_seconds_count:avg_rate5m
    expr: avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

为了让prometheus执行这条规则，需要在 *prometheus.ym*l 文件的 *global* 配置块中添加 *rule_files* 语句。配置文件如下：

```yaml
global:
  scrape_interval:     15s #  默认值，每15秒收集一次目标数据
  evaluation_interval: 15s # 每15秒执行一次规则

  # 将这些额外标签附加到prometheus收集到的所有时间序列中
  external_labels:
    monitor: 'codelab-monitor'

rule_files:
  - 'prometheus.rules.yml'

scrape_configs:
  - job_name: 'prometheus'

    # 覆盖全局默认值，每5秒收集一次该Job下的监控端点的数据.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']

  - job_name:       'example-random'

    # 覆盖全局默认值，每5秒收集一次该Job下的监控端点的数据.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

使用新配置文件重启prometheus实例，验证新的指标中包含 *job_service:rpc_durations_seconds_count:avg_rate5m* ，且可以在浏览器中查询和画出。



