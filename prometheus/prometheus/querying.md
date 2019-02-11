---
title: "[文档翻译] prometheus 查询"
date: 2018-11-21
tags: ["prometheus", "文档翻译"]
categories: ["文档翻译", "prometheus"]
draft: false
---

## 查询prometheus

原文链接： [Basic](https://prometheus.io/docs/prometheus/latest/querying/basics/)

Prometheus 提供了函数式的表达式语言，使得用户可以实时的查询和汇总时间序列数据。表达式语言获得的结果可以在Prometheus的web页面中以图形或表格数据的格式查看，也可以在外部系统中使用[HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)来调用。

### 例子

本文档仅供参考。对学习来说，通过一系列的例子（[examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)）来学习会更容易。

### 表达式语言的数据类型

在Prometheus表达式语言中，一个表达式或子表达式可以得到以下四个类型之一：

* `Instant vector`： 一组时间序列，包含每个时间序列的单个样本，所有时间序列共享同个时间戳；
* `Range vector`：一组时间序列，包含每个时间序列随时间变化的一系列数据点；
* `Scalar`：一个简单的数字浮点值；
* `String`：一个简单的字符串值，当前不可用

根据用例，按用户定义的语句，只有部分的类型才会时合法的（如，使用绘图来表现语句的输出时）。例如，一个只返回 `instant vector` 的语句是唯一可以用来直接画图的类型。

### 序列

#### `String literals`

字符串可以使用单引号、双引号或反引号来定义为序列。

PromQL遵循与Go一样的转义规则。在单引号或双引号内，反斜杠开始一个转义序列，后面可以跟 `a`, `b`,`f`,`n`,`r`,`t`,`v`或 `\`。

可以使用八进制或十六进制表示特殊字符： `\nnn`, `\xnn`, `\unnnn`, `\Unnnnnnnn`。

在反引号中不会处理转义。与Go不同，Prometheus不会丢弃反引号中的换行符。

例如：

```promql
"this is a string"
'these are unescaped: \n \\ \t'
`these are not unescaped: \n ' " \t`
```

#### `Float literals`

标量浮点值可以字面上写为：`[-](数字)[.(数字 )]`.

```promql
-2.43
```

### 时间序列选择器

#### `Instant vector`选择器

`Instant vector`选择器允许为每个给定的时间戳选择一组时间序列和一个样本值：最简单的方式，就是只指定一个指标的名称。返回的结果是一个包含该指标的所有时间序列的元素的 `Instant vector`。

下面这个例子将会选择所有包含 `http_request_total` 指标的所有时间序列：

```promql
http_requests_total
```

可以通过匹配花括号 ` ({ })` 来添加一系列的标签实现对时间序列结果的进一步过滤。

下面的例子只会选择包含 `http_requests_total`指标，且 `job`标签被设置为  `prometheus`，`group`标签被设置为 `canary`的时间序列：

 ```promql
http_requests_total{job="prometheus",group="canary"}
 ```

也可以对标签值进行反匹配，或者对标签值进行正则匹配。有以下几种标签匹配操作符：

* `=`：选择与所提供字符串完全匹配的标签值；
* `!=`：选择与所提供字符串不匹配的标签值；
* `=~`：选择与所提供字符串正则匹配的标签值；
* `!~`：选择与所提供字符串不正则匹配的标签值；

例如，下面的语句将会选择所有 `environment`标签值为 `staging`、`testing`和 `development`和 `method` 标签值为 `GET`的所以 `http_requests_total`的时间序列：

```promql
http_requests_total{environment=~"staging|testing|development",method!="GET"}
```

与空标签值匹配的标签匹配器也会选择所有没有该标签的所有时间序列，正则匹配则是完全限定的，同一个标签也可以有多个标签匹配器。

`Vector`选择器必须提供一个名称或一个与空字符串不匹配的标签匹配器。下面是一个非法的语句：

```promql
{job=~".*"} # Bad!
```

相反，下面的语句都是有效的，因为它们都拥有一个与空字符串标签值不匹配的选择器：

```promql
{job=~".+"}              # Good!
{job=~".*",method="get"} # Good!
```

标签匹配器也可以通过匹配内部 `__name__`标签应用与指标名称。例如，语句 `http_requests_total`与 `{__name__="http_requests_total"}` 有一样的效果。也可以使用 `=`、`!=`、`=~`、`!=`。下面的语句选择所有名称以 `job:` 开头的指标：

```promql
{__name__=~"job:.*"}
```

Prometheus中所有的正则表达式使用 [RE2 syntax](https://github.com/google/re2/wiki/Syntax)。

#### `Range Vector`选择器

`Range Vector`序列与 `Instant Vector`序列一样，除了 `Range Vector`会从当前时刻选择一个范围的标本值。语法上，一个范围时间值将会附加到 `Vector`选择器后面的方括号中，以指定为每个 `Range Vector`元素获取多长时间的值。

范围时间值是一个整数，后面紧跟下面其中之一的单位：

* `s` —— 秒
* `m` —— 分
* `h` —— 时
* `d` —— 天
* `w` —— 星期
* `y` —— 年

在下面例子中，我们选择指标名称为 `http_requests_total`，标签 `job`值为 `prometheus`，在过去5分钟内的所有记录：

```promql
http_requests_total{job="prometheus"}[5m]
```

#### `Offset`编辑器

`Offset`编辑器允许在 `Instant vector`和 `Range vector` 查询中改变时间的偏移值。

例如，下面的语句将会返回相对于当前查询时间，过去5分钟前 `http_requests_total`指标的所有时间序列：

 ```promql
http_requests_total offset 5m
 ```

注意，`Offset`编辑器必须紧跟在选择器后面，例如，下面的语句是正确的：

```promql
sum(http_requests_total{method="GET"} offset 5m) // GOOD.
```

然而，下面的语句是错误的：

```promql
sum(http_requests_total{method="GET"}) offset 5m // INVALID.
```

对于 `Range Vector`也是一样，下面的语句返回一周前 `http_requests_total`5分钟的速率：

```promql
rate(http_requests_total[5m] offset 1w)
```

## 操作符

### 二元操作符

Prometheus查询语言支持基本的逻辑和算术运算符。对于两个 `Instant vectors`之间的操作，可以修改匹配行为。

#### 二元算术运算符

Prometheus中支持以下的二元算术运算符：

* `+`（加）
* `-`（减）
* `*`（乘）
* `/`（除）
* `%`（取模）
* `^`（幂运算）

二元算术运算符可以在以下值对之间操作：

* 标量/标量：操作结果为另一个标量
* 标量/向量：操作将会对向量中的所有样本数据有效。例如，一个即时时间向量乘以2，会生成另一个向量，其值是原向量中的所有样本值乘以2
* 向量/向量：二进制算术运算符应用于左侧向量中的每个条目及其右侧向量中的匹配元素。操作结果传递到结果向量中，并删除度量标准名称。其中右侧向量中没有相应匹配条目的条目不会成为结果的一部分。

#### 二元比较操作符

Prometheus中支持以下二元比较操作符：

* `==`（等于）
* `!=`（不等于）
* `>`（大于）
* `<`（小于）
* `>=`（大于等于）
* `<=`（小于等于）

二元比较操作符可以在标量/标量、向量/标量和向量/向量之间操作。比较操作符得到的结果是 `bool`类型值，即 `0`和 `1`。

在 `标量`/`标量`中，需要提供 `bool`修饰符，操作结果将会是值为 `0`或 `1`的标量值，取决于比较结果；

在 `向量`/ `标量`中，操作将会应用于向量中的所有样本值，其中比较结果为 `false`的样本值会被丢弃，比较结果为 `true`的样本值会被保留。如果提供了 `bool`修饰符，则比较结果为 `true`的值为 `1`，比较结果为 `false`的值为 `0`；

在 `向量`/`向量`中，默认情况下，操走符类似于过滤器，作用于匹配的条目。左边条目中，操作结果为非 `true`或在后边找不到相匹配的条目将会被丢弃，其余的会把值保存到含有左边向量的对应指标的结果向量中。如果提供的 `bool`修饰符，则比较结果为 `true`的值改为1，其余的则为 `0`；

#### 二元逻辑/集合操作符

下列二元逻辑/集合操作符只能应用于两个 `Instant vectors`中：

* `and` （交集/与）
* `or` （并集/或）
* `unless`（差集）

`vector1 and vector2`会将`vector1`中的所有样本数据与 `vector2`中的所有样本数据进行标签匹配，匹配的保留，不匹配的丢弃。保留结果是左边的度量指标和值。

`vector1 or vector2`会将 `vector1`中的所有元素保留，并把 `vector2`中不匹配的值追加到结果向量中。

`vector1 unless vector2`会将 `vector1`中存在的元素，且 `vector2`中不存在的元素保留到结果变量中。

### 向量匹配

两个向量之间的操作，需要在右边向量中寻找与左边向量相匹配的元素。匹配方式有两个基本的方式：一对一和多对一/一对多。

#### 一对一向量匹配

一对一会从操作的两侧找到唯一的匹配条目。默认情况下，操作遵循该格式：`vector1 <operator> vector2`。如果条目中的标签集和对应值均一样，则认为条目相匹配。其中，关键字 `ignoring` 可以在匹配时指定忽略特定标签，而关键字 `on`可指定只匹配给定的标签列表：

```promql
<vector expr> <bin-op> ignoring(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) <vector expr>
```

样本数据：

```promql
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120
```

查询：

```promql
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

这将会返回一个结果向量，它是在过去5分钟之内，HTTP请求中code为500的各种请求方法各自占的比率。如果没有 `ignoring(code)`，因为它们没有相同的标签集，这两个向量将无法匹配。其中 `put`和 `del`两个方法的样本数据因为不匹配而不会出现在结果集中：

```promql
{method="get"}  0.04            //  24 / 600
{method="post"} 0.05            //   6 / 120
```

#### 多对一和一对多向量匹配

**多对一**和 **一对多**匹配，是指一侧的每个向量元素与多侧的向量元素相匹配的情况。这里需要明确指定两个修饰符：`group_left` 或 `group_right`，其中左或右决定了哪边的向量具有更高的基数。

```promql
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```

其中带 `group`标签修饰符的标签列表包含了 `一对多`中的“一”侧的标签，这些标签将会被包含在结果指标中。对于 `on`中的标签则只能出现在这些列表中的其中一个。结果向量中的每一个时间序列数据都必须是唯一的。

`group`修饰符只能被用在比较和算术操作符中。`and`、`unless`和 `or`操作默认会尽可能的匹配右边向量的条目。

查询例子：

  ```promql
method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
  ```

在这个例子中，左边向量每个方法标签值多于一个条目，所以我们使用 `group_left`。右边向量的元素匹配左边多个有相同 `method`标签的元素。

 ```promql
{method="get", code="500"}  0.04            //  24 / 600
{method="get", code="404"}  0.05            //  30 / 600
{method="post", code="500"} 0.05            //   6 / 120
{method="post", code="404"} 0.175           //  21 / 120
 ```

多对一和一对多匹配应该被谨慎使用。通常，使用 `ignoring(<labels>)`来输出相应的结果。

### 聚合运算符

Prometheus支持一下的内置聚合操作符，这些聚合操作符可以将单个 `Instant Vector`的多个元素，聚合成一个更少元素的新向量：

* `sum`（在维度上求和）
* `min`（在维度上求最小值）
* `max`（在维度上求最大值）
* `avg`（在维度上求平均值）
* `stddev`（求标准差）
* `stdvar`（求方差）
* `count`（统计向量元素个数）
* `count_values`（统计相同数值的元素个数）
* `bottomk`（样本中第k个最小值）
* `topk`（样本中第k个最大值）
* `quantile`（统计分位数）

这些标签可以用来聚合所有的标签维度，也可以使用 `without`或 `by`字句来保留不同的维度。

```promql
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

`parameter`参数只用在 `count_values`、`quantile`、`topk`和 `bottomk`中。`without`子句从结果向量中移除列表中的标签，保留其它标签。`by`字句则相反，它从结果中移除没有在列表中的标签，即时这些标签值在所有的向量元素中均有定义。

`count_values`对每个唯一的样本值输出一个时间序列。每个时间序列都会附加一个额外标签，标签的名称由聚合参数指定，标签值也是唯一的样本值。每一个时间序列的值是样本值出现的次数。

`topk`和 `bottomk`与其它聚合操作符不同，返回的结果中包含原始的标签。`by`和 `without`只能用在输入向量的桶中。

例如：

如果指标 `http_requests_total`中包含有用 `application`、`instance`和 `group`标签组合的时间序列，我们可以通过以下方式计算去除 `instance`标签的每个 `application`和 `group`的所有HTTP请求数：

```promql
sum(http_requests_total) without (instance)
```

等效于：

```promql
sum(http_requests_total) by (application, group)
```

如果我们对所有 `application`中的http请求总数感兴趣，可以这样：

```promql
sum(http_requests_total)
```

统计每个编译版本的二进制文件数量，可以这样：

```promql
count_values("version", build_version)
```

在所有实例中，查找HTTP请求数第五大的值，可以这样：

```promql
topk(5, http_requests_total)
```

### 二元操作符优先级

下面的列表展示了Prometheus中二元操作符的优先级，从高到低：

1. `^`
2. `*`, `/`, `%`
3. `+`, `-`
4. `==`, `!=`, `<=`, `<`, `>=`, `>`
5. `and`, `unless`
6. `or`

相同优先级的操作符，遵循左关联规则。例如，`2 * 3 % 2`与 `(2 * 3) % 2`效果一样。不过，`^`是右相关的，`2 ^ 3 ^ 2`与 `2 ^ (3 ^ 2)`等效。

## 函数

一些函数包含默认参数，例如，`year(v=vector(time()) instant-vector)`。这表示在函数里有一个即时向量 `v`，如果不提供，则默认为 `vector(time())`.

#### `abs()`

`abs(v instant-vector)`返回一个新的向量，它的所有样本值是输入向量的绝对值。

`absent()`



## 示例

## HTTP API

