---
title: "[文档翻译] prometheus配置"
date: 2018-11-16
tags: ["prometheus", "文档翻译"]
categories: ["文档翻译", "prometheus"]
draft: false
---
原文链接：[CONFIGURATION](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

*prometheus* 通过命令行参数和配置文件来配置。其中命令行参数用于配置不可变的系统参数（如存储路径，数据被保存在磁盘还是内存中等等）；配置文件定义所有收集的 *Jobs* 和它的实例，还有需要加载的规则文件。

可以通过 *./prometheus -h* 来查阅所有支持的命令行参数。

*prometheus* 可以在运行时重载它的配置。如果心的配置格式不正确，新的修改将不会生效。可以通过发送一个 *SIGHUP* 到 *prometheus* *进程或发送一个 *HTTP POST* 请求到 */-/reload* 接口来触发重载配置，*HTTP POST*方式只有在 *--web.enable-lifecycle* 参数设置的时候才生效。这也会重载所有包含的规则文件。

## 配置文件

通过 *--config.file* 参数来指定使用哪个配置文件。

配置文件使用 *YAML* 格式，由下面描述的方案定义。括号表示参数是可选的。没有列出的参数将会使用默认值。

通用占位符定义如下：

* `<boolean>`：布尔值，`true` 或 `false`
* `<duration>`：一段时间，满足正则表达式 `[0-9]+(ms|[smhdwy])`
* `<labelname>`：字符串，满足正则表达式 `[a-zA-Z][a-zA-Z0-9_]*`
* `<labelvalue>`：字符串，满足正则表达式 `[a-zA-Z][a-zA-Z0-9_]*
* `<filename>`：一个有效的文件路径
* `<host>`：一个有效的域名或IP地址字符串，后面可选添加端口
* `<path>`：有效的URL路径
* `<scheme>`：协议字符串，`HTTP`或 `HTTPS`
* `<string>`：字符串
* `<secret>`：密码字符串
* `<tmpl_string>`: 使用前的模版扩展字符串

其它占位符需要特定指定。

示例文件地址：[https://github.com/prometheus/prometheus/blob/release-2.5/config/testdata/conf.good.yml](https://github.com/prometheus/prometheus/blob/release-2.5/config/testdata/conf.good.yml)

全局配置指定了在所有其它配置上下文中均有效的参数，它们也作为其它配置块中的默认值。

```yaml
global:
  # 默认收集目标数据的时间间隔
  [ scrape_interval: <duration> | default = 1m ]

  # 收集请求的timeout时间
  [ scrape_timeout: <duration> | default = 10s ]

  # 执行规则的时间间隔
  [ evaluation_interval: <duration> | default = 1m ]

  # 与扩展系统(集群，远程存储，Alertmanager等)通信时，添加到所有时间序列上的标签
  external_labels:
    [ <labelname>: <labelvalue> ... ]


# globs方式的规则文件列表。从所有匹配文件中读取规则和告警
rule_files:
  [ - <filepath_glob> ... ]

# 收集配置列表
scrape_configs:
  [ - <scrape_config> ... ]

# Alertmanager的告警配置
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

# 远程写入功能设置.
remote_write:
  [ - <remote_write> ... ]

# 远程读取功能设置.
remote_read:
  [ - <remote_read> ... ]
```

### `<scrape_config>`

`scrape_config` 块定义了一系列的目标和如何收集目标数据的参数。通常，一个收集配置项定义一个 `Job`，高级配置中，这可能会不一样。

目标可以通过 `static_configs`参数来静态配置，也可以通过支持的服务发现机制来动态发现。

另外，`relabel_configs`支持在收集前对目标和它的标签进行高级修改。

```yaml
# 收集指标的默认Job名称.
job_name: <job_name>

# 这个Job收集目标数据的时间间隔
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]

# 这个Job每个收集请求的timeout时间
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]

# 从目标收集指标的HTTP资源路径.
[ metrics_path: <path> | default = /metrics ]

# honor_labels控制如何处理已经存在于收集到的数据的标签
# 和prometheus即将附加到服务器端的标签之间的冲突。
# （“Job”和“instance”标签，手动配置的目标标签，和服务发现生成的标签）
#
# 如果 honor_labels 设置为 “true”，则保留已收集到的数据的标签，忽略冲突的服务器端的标签
#
# 如果 honor_labels 设置为 “false”，则重命名已收集到的数据的冲突标签为：“exported_<original-label>”，
# 例如“exported_instance”，“exported_job”。再附加服务器端标签。
# 这对于多集群方式很有用，目标中定义的所有标签都需要保留。
#
# 请注意，任何全局配置的“external_labels”都不受此设置的影响。当与外部系统通信时，只有当时间序列
# 美誉给定标签时才会应用它们，否则会被忽略
[ honor_labels: <boolean> | default = false ]

# 配置收集请求的协议类型.
[ scheme: <scheme> | default = http ]

# 可选的HTTP URL参数.
params:
  [ <string>: [<string>, ...] ]

# 根据配置的username和password，在每个收集请求中添加 `Authorization` 头
# password 和 password_file 是互斥的。
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# 根据设置的 bearer token，在每个收集请求中添加 `Authorization`头
# bearer_token 和 bearer_token_file 是互斥的
[ bearer_token: <secret> ]

# 根据文件中的 bearer token，在每个收集请求中添加 `Authorization`头
# bearer_token_file 和 bearer_token 是互斥的
[ bearer_token_file: /path/to/bearer/token/file ]

# 配置收集请求的TLS.
tls_config:
  [ <tls_config> ]

# 可选的代理URL.
[ proxy_url: <string> ]

# Azure服务发现配置列表
azure_sd_configs:
  [ - <azure_sd_config> ... ]

# Consul服务发现配置列表
consul_sd_configs:
  [ - <consul_sd_config> ... ]

# DNS服务发现配置列表
dns_sd_configs:
  [ - <dns_sd_config> ... ]

# EC2服务发现配置列表
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

# Openstack服务发现配置列表
openstack_sd_configs:
  [ - <openstack_sd_config> ... ]

# 文件服务发现配置列表
file_sd_configs:
  [ - <file_sd_config> ... ]

# GCE服务发现配置列表
gce_sd_configs:
  [ - <gce_sd_config> ... ]

# Kubernetes服务发现配置列表
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

# Marathon服务发现配置列表
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

# AirBnB的神经服务发现配置列表
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

# Zookeeper Serverset服务发现配置列表
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

# Triton服务发现配置列表
triton_sd_configs:
  [ - <triton_sd_config> ... ]

# 静态配置列表
static_configs:
  [ - <static_config> ... ]

# 目标重新标记配置列表
relabel_configs:
  [ - <relabel_config> ... ]

# 指标重新标记配置列表
metric_relabel_configs:
  [ - <relabel_config> ... ]

# 对目标进行收集时，每次收集的样本数的限制
# 如果在指标重新标记后，收集的样本数多余这个值，这这次收集将会被认为失败
# 0 表示没有限制
[ sample_limit: <int> | default = 0 ]
```

其中 `<job_name>`必须在所有收集配置项中唯一。

### `<tls_config>`

`tls_config` 运行配置 TLS 连接相关配置

```yaml
# 用于验证API服务证书的CA证书
[ ca_file: <filename> ]

# 用于客户端验证服务器端证书身份的证书和密钥文件
[ cert_file: <filename> ]
[ key_file: <filename> ]

# 扩展名，指定服务器名称
# http://tools.ietf.org/html/rfc4366#section-3.1
[ server_name: <string> ]

# 关闭服务器的证书验证
[ insecure_skip_verify: <boolean> ]
```

### `azure_sd_config`

Azure SD配置允许从Azure VMs中检索监控目标。

在重新标记时，目标可以使用以下元标签：

* `__meta_azure_machine_id`: 机器ID
* `__meta_azure_machine_location`: 机器所在区域
* `__meta_azure_machine_name`: 机器名称
* `__meta_azure_machine_os_type`: 机器操作系统
* `__meta_azure_machine_private_ip`: 机器的私有IP
* `__meta_azure_machine_resource_group`: 机器的资源组
* `__meta_azure_machine_tag_<tagname>`: 机器的每个tag值
* `__meta_azure_machine_scale_set`: vm所属的扩展集名称（只有在使用了扩展集时有效）

下面是Azure服务发现的配置选项

```yam
# 访问Azure API的信息
# Azure环境
[ environment: <string> | default = AzurePublicCloud ]
# The subscription ID
subscription_id: <string>
# The tenant ID.
tenant_id: <string>
# The client ID.
client_id: <string>
# The client secret.
client_secret: <secret>

# 读取实例列表的时间间隔
[ refresh_interval: <duration> | default = 300s ]

# 收集指标的端口。如果使用公共IP，使用重标记规则中的端口代替
[ port: <int> | default = 80 ]
```

### `<consul_sd_config>`

Consul服务发现配置允许从Consul的Catalog API中检索收集目标。

下面的元标签在重标记目标时允许访问

* `__meta_consul_address`: 目标地址
* `__meta_consul_dc`: `目标数据中心名
* `__meta_consul_metadata_<key>`: 目标中每个节点的metadata键
* `__meta_consul_node`:为目标定义的节点名称
* `__meta_consul_service_address`: 目标的服务地址
* `__meta_consul_service_id`: 目标的服务ID
* `__meta_consul_service_metadat_<key>`: 目标中每个服务的metadata键
* `__meta_consul_service_port`: 目标的服务的端口
* `__meta_consul_service`: 目标所属的服务的名称
* `__meta_consul_tags`: 由标签分隔符链接而成的目标标签列表

```yaml
# 访问consul API的信息。依照consul文档要求设置
[ server: <host> | default = "localhost:8500" ]
[ token: <secret> ]
[ datacenter: <string> ]
[ scheme: <string> | default = "http" ]
[ username: <string> ]
[ password: <secret> ]

tls_config:
  [ <tls_config> ]

# 检索目标的服务列表。如果省略，所有服务将会被检索
services:
  [ - <string> ]

# 查看https://www.consul.io/api/catalog.html#list-nodes-for-service 来获知更多可用的过滤器

# 可选的标签，来过滤给定服务的节点
[ tag: <string> ]

# 节点元信息，用来过滤给定服务的节点
[ node_meta:
  [ <name>: <value> ... ] ]

# 标签分隔符
[ tag_separator: <string> | default = , ]

# 可保留的过期Consul结果，以减轻Consul的负担。
# 参考 https://www.consul.io/api/index.html#consistency-modes
[ allow_stale: <bool> ]

# 名称的刷新时间
# 在大型系统中，增加此值是个好主意，因为Catalog一直在变化
[ refresh_interval: <duration> | default = 30s ]
```

请注意，收集目标的IP地址和端口组装成：`<__meta_consul_address>`:`<__meta_consul_service_port>`。然而，在有些consul设置中，相关地址在字段：`<__meta_consul_service_address>`中，在这种情况下，你可以使用重标记功能来替代`__address__`标签。

对基于任意标签的服务而言，重标记阶段时执行服务或节点过滤是优先也是更有效的手段。对拥有上千个服务的用户而言，直接使用Consul API更高效，因为它对过滤节点由基础支持。（当前是基于节点元数据和标签）

### `<dns_sd_config>`

基于DNS的服务发现配置允许通过定义一组DNS域名，这些域名会被定期查询来发现目标。DNS服务从`/etc/resolv.conf` 中读取。

这个服务发现方法只支持基本的DNS A、AAAA和SRV记录查询，其它在[RFC6763](https://tools.ietf.org/html/rfc6763)定义的高级DNS服务发现还不支持。

在重标记阶段，元标签 `__meta_dns_name` 在每个目标中均可被访问，且被用于记录生成的发现目标名称。

```yaml
# 要被查询的DNS域名列表
names:
  [ - <domain_name> ]

# DNS查询的类型
[ type: <query_type> | default = 'SRV' ]

# 当Type为非SRV时使用的端口
[ port: <number>]

# 名称刷新的时间间隔
[ refresh_interval: <duration> | default = 30s ]
```

其中 `<domain_name>` 时有效的DNS域名， `<query_type>` 是 `SRV`、`A` 或 `AAAA`。

### `<ec2_sd_config>`

EC2服务发现配置允许从AWS EC2实例中检索收集目标。默认使用私有IP，在重标记时可以更改为公共IP。

在重标记时，下面的元标签在目标中可以允许访问：

* `__meta_ec2_availability_zone`: 实例的可访问区
* `__meta_ec2_instance_id`: EC2实例ID
* `__meta_ec2_instance_state`: EC2实例的状态
* `__meta_ec2_instance_type`: EC2实例的类型
* `__meta_ec2_owner_id`: EC2实例所属账户的用户ID
* `__meta_ec2_primary_subnet_id`: 私有网络接口的子网ID，如果有
* `__meta_ec2_private_ip`: 实例的私有IP，如果有
* `__meta_ec2_public_dns_name`: 实例的公共DNS名称。如果有
* `__meta_ec2_public_ip`: 实例的公共IP，如果有
* `__meta_ec2_subnet_id`: 实例所属的逗号分隔的子网ID列表，如果有
* `__meta_ec2_tag_<tagkey>`: 实例的标签值
* `__meta_ec2_vpc_id`: 实例所属VPC的ID，如果有

下面是EC2服务发现的配置选项：

```yaml
# 访问EC2 API的信息

# AWS区域.
region: <string>

# 自定义端点.
[ endpoint: <string> ]

# AWS API访问key。如果为空，则环境变量 `AWS_ACCESS_KEY_ID`和`AWS_SECRET_ACCESS_KEY`将会被使用
[ access_key: <string> ]
[ secret_key: <secret> ]
# 连接AWS API的配置文件
[ profile: <string> ]

# AWS Role ARN，使用AWS API密钥的替代方法
[ role_arn: <string> ]

# 刷新实例列表的时间间隔
[ refresh_interval: <duration> | default = 60s ]

# 收集指标的端口. 如果使用公共IP，使用重标记规则的端口替代
[ port: <int> | default = 80 ]

# 可以选择使用过滤器按其他条件过滤实例列表
# 可在此处找到可用的过滤条件
# https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeInstances.html
# 过滤API文档：
# https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_Filter.html
filters:
  [ - name: <string>
      values: <string>, [...] ]
```

对基于任意标签的服务而言，重标记阶段时执行服务或节点过滤是优先也是更有效的手段。对拥有上千个服务的用户而言，直接使用EC2 API更高效，因为它对过滤实例由基础支持

### `<openstack_sd_config>`

Openstack 服务发现配置允许从Openstack Nova实例中检索收集目标。

下面的任意一种 `<openstack_role>`类型都可以用来配置发现目标：

#### `hypervisor`

hypervisor类型在每个Nova hypervisor节点发现一个目标。目标的地址默认为hypervisor的 `host_ip` 属性。

在重标记时，目标可以访问下列的元标签：

* `__meta_openstack_hypervisor_host_ip`: hypervisor 节点的IP地址
* `__meta_openstack_hypervisor_name`: hypervisor节点名称
* `__meta_openstack_hypervisor_state`: hypervisor节点区域
* `__meta_openstack_hypervisor_status` : hypervisor节点状态
* `__meta_openstack_hypervisor_type`: hypervisor节点类型

#### `instance`

instance类型在每个Nova实例的网络接口中发现一个目标，该目标的地址默认为网络接口的私有IP地址。

在重标记时，目标可以访问下列标签：

* `__meta_openstack_instance_id` : Openstack实例ID
* `__meta_openstack_instance_name`: Openstack实例名称
* `__meta_openstack_instance_status`: OpenStack实例状态
* `__meta_openstack_instance_flavor`: OpenStack实例个数
* `__meta_openstack_instance_public_ip`: OpenStack实例的公共IP
* `__meta_openstack_instance_private_ip`: OpenStack实例的私有IP
* `__meta_openstack_instance_address_pool`: OpenStack实例私有IP池
* `__meta_openstack_tag_<tagkey>`: 实例的标签对应值

下面是OpenStack intance类型的配置选项：

```yaml
# 访问OpenStack API的信息.

# 要被发现的OpenStack实例的角色.
role: <openstack_role>

# OpenStack区域.
region: <string>

# `identity_endpoint`指定与Identity API对应版本的HTTP端点.
# 所有身份服务都需要它，它通常有服务提供者提供.
[ identity_endpoint: <string> ]

# 如果使用Identity V2 API，则需要提供用户名。
# 请通过你的服务提供商的控制台面板，获取帐号的用户名。
# 在Identity V3中，需要userid或username和domain_id或domain_name的组合。
[ username: <string> ]
[ userid: <string> ]

# Identity V2和V3 API的密码。
# 请通过你的服务提供商的控制台面板，获取帐号验证的首选方法.
[ password: <secret> ]

# 如果使用Identity V3的用户名，则必须提供domain_id或domain_name中的一个。
# 如果没使用，该选项是可选的。
[ domain_name: <string> ]
[ domain_id: <string> ]

# project_id和project_name在Identity V2 API中是可选的。
# 一些提供商允许你使用project_name替代project_id.
# 一些则要求两者，你的提供商的认证策略决定了这些字段如何影响认证行为.
[ project_name: <string> ]
[ project_id: <string> ]

# 服务发现是否应该列出所有项目的所有实例.
# 它只在 `instance`类型时有效，而且要求管理员权限.
[ all_tenants: <boolean> | default: false ]

#刷新实例列表的时间间隔.
[ refresh_interval: <duration> | default = 60s ]

# 收集指标的端口. 
# 如果使用公共IP，使用重标记规则的端口替代.
[ port: <int> | default = 80 ]

# TLS 配置.
tls_config:
  [ <tls_config> ]
```

### `<file_sd_config>`

基于文件的服务发现提供了一种更通用的方法来配置静态目标，并用作插入自定义服务发现机制的接口。

它读取一组包含零个或多个 `<static_config>` 的文件。通过磁盘监测来检查所有文件的更改并立即使其生效。文件可以是YAML或JSON格式，只有格式正确的更改才会有效。

JSON文件必须包含静态配置的列表，格式如下：

```json
[
  {
    "targets": [ "<host>", ... ],
    "labels": {
      "<labelname>": "<labelvalue>", ...
    }
  },
  ...
]
```

作为后备，文件内容还会按指定的时间间隔重新读取。

在重标记阶段，每个目标都有一个元标签：`__meta_filepath`,它的值时目标提取时的文件路径。

有一个与此发现机制的集成列表。

```yaml
# 提取目标的文件名格式.
files:
  [ - <filename_pattern> ... ]

# 重新读取文件内容的时间间隔.
[ refresh_interval: <duration> | default = 5m ]
```

其中 `<filename_pattern>`可以是以 `.json`、`.yml`或 `.yaml`后缀结尾的文件路径。路径的最后一段可能包含一个 `*` ，表示匹配任何字符序列。如 `my/path/tg_*.json`.

### `<gce_sd_config>`

GCE服务发现配置允许从GCP GCE实例中检索监控目标。默认使用私有IP，但可以在重标记时改为公共IP。

下列元标签在重标记时可悲目标访问：

* `__meta_gce_instance_id`: 实例的ID
* `__meta_gce_instance_name`: 实例的名称
* `__meta_gce_label_<name>`:实例的标签
* `__meta_gce_machine_type`: 实例类型的全部或部分URL
* `__meta_gce_metadata_<name>`: 实例的源数据元素
* `__meta_gce_network`: 实例的网络URL
* `__meta_gce_private_ip`: 实例的私有IP
* `__meta_gce_project`: 实例所属项目
* `__meta_gce_public_ip`: 实例的公共IP
* `__meta_gce_subnetwork`: 实例的子网URL
* `__meta_gce_tags`: 逗号分隔的实例标签
* `__meta_gce_zone`: 实例所属的区域URL

下面是GCE服务发现的配置选项：

```yaml
# GCE API的访问信息.

# GCP 项目
project: <string>

# 收集目标所属区域. 如果你需要收集多个区域，请使用多个 `gce_sd_configs`.
zone: <string>

# 可以选择使用过滤器按条件过滤实例，过滤器字符串的语法在过滤器查询参数重描述：
# https://cloud.google.com/compute/docs/reference/latest/instances/list
[ filter: <string> ]

# 重新读取实例列表的时间间隔
[ refresh_interval: <duration> | default = 60s ]

# 收集指标的端口. 
# 如果使用公共IP，使用重标记规则的端口替代.
[ port: <int> | default = 80 ]

# 标签分隔符。
[ tag_separator: <string> | default = , ]
```

Google Cloud SDK默认客户端会在下列文件中寻找验证证书，按顺序表示查找的优先级：

1. 由环境变量 `GOOGLE_APPLICATION_CREDENTIALS` 指定的JSON文件
2. `$HOME/.config/gcloud/application_default_credentials.json` 文件
3. 从GCE元数据服务中获取

如果Prometheus在GCE中运行，则与其运行的实例关联的服务帐户应至少具有对计算资源的只读权限。如果在GCE之外运行时，请确保创建适当的服务帐户并将凭证文件放在其中一个预期位置。

### `<kubernetes_sd_config>`

kubernetes服务发现配置允许通过kubernetes REST API检索监控目标，而且会始终与群集状态保持同步。

可以配置以下 `role` 类型来发现监控目标：

#### `node`

node 类型通过kubelete的HTTP端口来发现每个集群阶段的监控目标。该目标的的地址默认为kubernetes节点对象中，下列地址中第一个存在的值：`NodeInternalIP`、`NodeExternalIP`、`NodeLegacyHostIP`和 `NodeHostName`。

可访问的元标签：

* `__meta_kubernetes_node_name`: 节点对象的名称
* `__meta_kubernetes_node_label_<labelname>`: 节点对象的标签值
* `__meta_kubernetes_node_annotation_<annotationname>`: 节点对象的注解
* `__meta_kubernetes_node_address_<address_type>`: 每个节点地址类型的第一个地址，如果存在

另外，节点的标签 `instance` 将设置为从API Server中检索到的节点名称。

#### `service`

service类型为每个服务中每个服务端口发现监控目标，通常这对服务的黑盒监控很有用。它的地址将会被设置为该服务的kubernetes的DNS名称和对应的服务端口。

可反问的元标签：

* `__meta_kubernetes_namespace`: 服务对象的命名空间
* `__meta_kubernetes_service_name`: 服务对象的名称
* `__meta_kubernetes_service_label_<labelname>`: 服务对象的标签
* `__meta_kubernetes_service_annotation_<annotationname>`: 服务对象的注解
* `__meta_kubernetes_service_port_name`： 目标服务端口名称
* `__meta_kubernetes_service_port_number`: 目标服务端口号
* `__meta_kubernetes_service_port_protocol`: 目标服务端口的协议

#### `pod`

pod类型发现所有pod并暴露其中所有的容器作为监控目标。对每个容器声明的端口，都将生成一个监控目标。如果容器没有定义端口，则每个容器将会创建一个五端口的监控目标，在重标记时会自动为容器添加端口。

可访问的元标签：

* `__meta_kubernetes_namespace`: pod对象的命名空间
* `__meta_kubernetes_pod_name`: pod对象的名称
* `__meta_kubernetes_pod_ip`: pod对象的pod IP
* `__meta_kubernetes_pod_label_<labelname>`: pod对象的标签
* `__meta_kubernetes_pod_annotation_<annotationname>`: pod对象的注解
* `__meta_kubernetes_pod_container_name`: 目标地址指向的容器名称
* `__meta_kubernetes_pod_container_port_name`: 容器端口的名称
* `__meta_kubernetes_pod_container_port_number`: 容器端口号
* `__meta_kubernetes_pod_container_port_protocol`: 容器端口的协议
* `__meta_kubernetes_pod_ready`: 设置pod ready状态为：`true`或 `false`
* `__meta_kubernetes_pod_node_name`: pod被调度所在节点的名称
* `__meta_kubernetes_pod_host_ip`: pod对象的host IP
* `__meta_kubernetes_pod_uid`: pod对象的UID
* `__meta_kubernetes_pod_controller_kind`: pod对象的控制器类型
* `__meta_kubernetes_pod_controller_name`: pod对象的控制器名称

#### `endpoints`

