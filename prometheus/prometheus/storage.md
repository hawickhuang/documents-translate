---
title: "[文档翻译] prometheus 存储"
date: 2018-11-21
tags: ["prometheus", "文档翻译"]
categories: ["文档翻译", "prometheus"]
draft: false
---

Prometheus可以使用本地磁盘作为时间序列数据库，也可以选择与远程存储系统集成。

## 本地存储

Prometheus将时间序列数据以自定义的格式存储在本地磁盘的时间序列数据库中。

### 磁盘布局

提取的样本将会被分为2小时块，其中每个2小时块中有一个或多个块文件，块文件包含该窗口所有的时间序列样本、元数据文件和索引文件（通过度量标准和标签索引到块文件中的时间序列）。当通过API删除序列，被删除的记录会被存储到逻辑删除文件中，而不是立即从块文件中删除。

新传入样本的块保存在内存中，并未完全持久化。它通过预写日志来防止崩溃，当Prometheus服务崩溃时，可以进行重放。预写日志文件存储在以128MB为段的 `wal` 目录中。这些文件存储尚未压缩的原始数据，所以他们会比常规的块文件大得多。Prometheus至少会保存3个预写日志文件，但对于高流量服务，你可能会看到多余3个预写日志文件，因为需要保存至少2小时的原始数据。

一个Prometheus服务器的数据目录结构看起来会像这样：

```
./data/01BKGV7JBM69T2G1BGBGM6KB12
./data/01BKGV7JBM69T2G1BGBGM6KB12/meta.json
./data/01BKGTZQ1SYQJTR4PB43C8PD98
./data/01BKGTZQ1SYQJTR4PB43C8PD98/meta.json
./data/01BKGTZQ1SYQJTR4PB43C8PD98/index
./data/01BKGTZQ1SYQJTR4PB43C8PD98/chunks
./data/01BKGTZQ1SYQJTR4PB43C8PD98/chunks/000001
./data/01BKGTZQ1SYQJTR4PB43C8PD98/tombstones
./data/01BKGTZQ1HHWHV8FBJXW1Y3W0K
./data/01BKGTZQ1HHWHV8FBJXW1Y3W0K/meta.json
./data/01BKGV7JC0RY8A6MACW02A2PJD
./data/01BKGV7JC0RY8A6MACW02A2PJD/meta.json
./data/01BKGV7JC0RY8A6MACW02A2PJD/index
./data/01BKGV7JC0RY8A6MACW02A2PJD/chunks
./data/01BKGV7JC0RY8A6MACW02A2PJD/chunks/000001
./data/01BKGV7JC0RY8A6MACW02A2PJD/tombstones
./data/wal/00000000
./data/wal/00000001
./data/wal/00000002
```

最初的2小时块会在后台被压缩成大的块。

请注意，本地存储的限制是它不是集群或复制的。所以，面对磁盘或节点中断时，它不是任意可扩展或持久的。因此应该被视为更近期数据的短暂滑动窗口。但是，若你的耐久性要求不高，仍然可以在本地成功存储多达数年的数据。

想了解更多关于文件格式的细节，请参考[TSDB Format]()

## 运营维护

Prometheus有几个配置参数用来配置本地存储，下面是最终呀的几个：

* `--storage.tsdb.path`：Prometheus时间序列数据库路径。默认`data/`;
* `--storage.tsdb.retention.time`：数据保存时间。默认`15d`，会覆盖`storage.tsdb.retention`如果该值被设置为非默认值。
* `--storage.tsdb.retention.size`: 存储块可以使用的最大字节数（请注意，这不包括WAL的大小）。最早的数据被删除，默认为0或禁用。此设置是实验性的，可以在将来的版本中进行更改。支持的单位：KB，MB，GB，PB。例如：512MB；
* `--storage.tsdb.retention`: 不推荐使用此参数，请改用`storage.tsdb.retention.time`;

平均而言，Prometheus每个样本仅使用1-2字节，因此，要规划Prometheus服务器的容量，你可以使用粗略的公式：

```
needed_disk_space = retention_time_seconds * ingested_samples_per_second * bytes_per_sample
```

要调整每秒获取样本的速率，可以通过减少提取的时间序列数（更少的提取目标或者每个模板更少的序列数），或者可以增加提取的间隔。然而，因为样本系列的压缩，所以，减少系列数通常更有效。

如果你的本地存储因任何原原因而损坏，最好的办法就是关闭Prometheus并删除整个存储目录。但是，也可以尝试删除单个块目录以解决此问题。这意味着每个块目录会丢失大约2小时的数据时间窗口。同样，Prometheus的本地存储并不意味着持久的长期存储。

如果同时指定了时间和大小保留策略，则将会使用率先触发的策略。

## 远程存储集成

Prometheus的本地存储在扩展性和持久性方面受到单个节点的限制。Prometheus没有尝试解决自身的集群存储问题，而是提供了一组允许与远程存储系统集成的接口。

### 概览

Prometheus通过两个步骤与远程存储系统进行集成：

* Prometheus可以以标准格式将提取的样本写入远程URL
* Prometheus可以以标准格式从远程URL中读取样本数据

![](./remote_integrations.png)

读写协议都使用基于HTTP的snappy压缩协议缓冲区编码，这些协议尚未被视为稳定的API，并且可能会在将来更改为使用基于HTTP2的gRPC，此时可以安全地假设Prometheus和远程存储之间的所有跃点都支持HTTP2.

有关Prometheus中配置远程存储集成的细节，请参考Prometheus配置文档的[远程写入]()和[远程读取]()

有关请求和响应消息的细节，请参考[远程存储协议缓冲区定义]()

请注意，在读取路径上，Prometheus仅从远程端获取一组标签选择器和时间范围的原始系列数据。对原始数据的所有PromQL评估仍然发生在Prometheus本身，这意味着远程读取查询具有一定的可伸缩性限制，因为需要首先将所有必须的数据加载到查询Prometheus服务器中，然后在那里进行处理。但是，支持PromQL的完全分布式评估暂时被认为是不可行的。

### 现有的集成

想要学习更多现有的远程存储系统集成，请参考[集成文档]()