本文来自11月举办的 [Data + AI Summit](https://www.iteblog.com/archives/tag/data-ai-summit/) 2020 （原 [Spark](https://www.iteblog.com/archives/tag/spark/)+AI Summit），主题为《Materialized Column- An Efficient Way to Optimize Queries on Nested Columns》的分享，作者为字节跳动的郭俊。本文相关 PPT 可以关注 **Java与大数据架构** 公众号并回复  **9910** 获取。

在数据仓库领域，使用复杂类型（如map）中的一列或多列，或者将许多子字段放入其中的场景是非常常见的。这些操作可能会极大地影响查询性能，因为:

* 这些操作非常废 IO。比如我们有一个字段类型为 Map，其包含数十个子字段那么在读取这列时需要把整个列都读出来。并且 [Spark](https://www.iteblog.com/archives/tag/spark/) 将遍历整个 map 获得目标键的值。
* 在读取嵌套类型列时不能利用向量化读取。
* 读取嵌套列时不能使用算子下推（Filter pushdown）。

在过去的一年里，字节的数据引擎团队在 Apache Spark 中添加了一系列优化来解决 Parquet 格式中的上述问题。

其中包括支持对 Parquet 中复杂数据类型的向量化读取，允许对 Parquet 中的结构列进行子字段修剪，等等。此外，该团队还开发了一个名为物化列（materialized column）的新特性，透明地解决了任意列式存储（不仅仅是Parquet）的上述问题。在过去的一年里，物化列在字节跳动数据仓库中工作得很好。以一个典型的表为例，每天增量的数据量约为 200 TB。在它上面创建 15 个物化列可以提高超过110%的查询性能，而存储开销不到7%。在这次讨论中，我们将深入介绍 Spark SQL 中物化列的内部原理，描述物化列的使用场景。

今天的主题主要分为：

* 简单介绍字节跳动的 Spark SQL 使用情况
* 为什么嵌套类型被广泛使用
* 嵌套数据类型的主要问题
* 优化解决方案
* 物化列是如何解决这些问题的

## 字节跳动的 Spark SQL 使用情况

[![image](https://cdn.nlark.com/yuque/0/2020/png/938149/1607867739233-f17db136-0b6d-4a78-88dc-5b80b6cbdb2a.png "image")](https://www.iteblog.com/pic/spark/spark_at_bytedance1-iteblog.png)

如果想及时了解Spark、Hadoop或者HBase相关的文章，欢迎关注微信公众号：**iteblog_hadoop**

下面是 Spark SQL 在字节跳动的应用。

* 2016年主要是小规模的测试阶段
* 2017年用于处理 Ad-hoc 工作负载
* 2018年在生产环境下处理少量的 ETL 管道工作；
* 2019年在生产环境下全面部署；
* 2020年成为 DW 领域的主要计算引擎。

## 为什么嵌套类型被广泛使用

字节跳动主要有两种类型的数据。第一类也是最大的一类也就是事件日志（Event log），通过事件日志可以进行一些事件跟踪。在字节每天都有大量的事件日志，而且不同的事件日志有很多不同的信息。为事件日志的每个字段都创建一个列存储是非常不明智的，所以在字节事件日志的存储就两列 Event type（String类型）和 Message（Map类型）。所有的事件信息都是存放在 Message 中的。

另外一种日志是维度数据（Dimension data），维度表里面的数据是从 MySQL dump 过来的。MySQL 中对应表的字段可能会增加，也可能会减少。在我们的维度表里面我们不能简单的删除对应的列，因为如果那样我们的数据可能会丢失。所以对于维度数据，一些常用的列是以一列一列单独存储的；其他一些列是存储在一个 map 类型的列中。

## 嵌套数据类型的主要问题

[![image](https://cdn.nlark.com/yuque/0/2020/png/938149/1607867739199-0e2022c6-a849-49b9-81c0-0336cfd93a48.png "image")](https://www.iteblog.com/pic/spark/spark_at_bytedance2-iteblog.png)

如果想及时了解Spark、Hadoop或者HBase相关的文章，欢迎关注微信公众号：**iteblog_hadoop**

嵌套类型的数据主要有以下四个问题：

* 这些操作非常废 IO。比如我们有一个字段类型为 Map，其包含数十个子字段那么在读取这列时需要把整个列都读出来。并且 Spark 将遍历整个 map 获得目标键的值。
* 在读取嵌套类型列时不能利用向量化读取。
* 读取嵌套列时不能使用算子下推（Filter pushdown）。
* 大量的重复计算

## 优化解决方案

假设我们有一张名为 base_table 的表，里面有三个字段：item、count 以及 people，其中 people 字段是 map 类型的。现在我们要查询 people 字段里面的 age 字段。为了解决嵌套类型的查询慢问题，主要有两种方法来解决这个问题。

[![image](https://cdn.nlark.com/yuque/0/2020/png/938149/1607867739208-44ec5c78-201c-46f3-b30f-7b9f236b84ad.png "image")](https://www.iteblog.com/pic/spark/spark_at_bytedance3-iteblog.png)

如果想及时了解Spark、Hadoop或者HBase相关的文章，欢迎关注微信公众号：**iteblog_hadoop**

第一种方法就是创建一个名为 sub_table 的表。sub_table 表除了包含 base_table 的所有字段之外还新增了一个新字段 age，这个字段其实是 people 中 age 字段的值。所以如果我们要查询 people 里面的 age 字段，那么我们可以直接查询 sub_table 表里面的 age 字段，这样查询性能会比直接查询 base_table 表中 people 里面的 age 字段快。事实上，很多公司都在使用这种方法， 这种方法优点是由于嵌套列里面的字段被单独抽出来存储，查询直接在新的列里面进行，所以查询性能自然提升了。

但是这种方法的缺点也很明显：

* 需要告诉下游所有使用到 base_table 的用户来使用 sub_table；
* sub_table 明显占用了重复的存储
* 不能处理频繁的子字段更改

[![image](https://cdn.nlark.com/yuque/0/2020/png/938149/1607867739246-ebffcbdd-2edc-43a6-a44b-1a8a8c4b2952.png "image")](https://www.iteblog.com/pic/spark/spark_at_bytedance4-iteblog.png)

如果想及时了解Spark、Hadoop或者HBase相关的文章，欢迎关注微信公众号：**iteblog_hadoop**

第二种方法是被称为 Parquet vectorized reader 的方法， 其可以向量化读取嵌套数据类型。并且支持对 struct 类型的过滤下推，但是 map 类型的数据并不支持。

这种方法的优点是不需要额外的存储。同时对于 struct 类型的数据性能提升 100%，map 类型的数据性能提升 163%。

但是这种方法也有缺点：

* 需要分别重构 Parquet 和 ORC 的向量化读取逻辑；
* Array/Map 数据类型的过滤下推目前还不支持；
* 嵌套类型数据的向量化读取性能没有简单数据类型的高

## 物化列是如何解决这些问题的

为了解决上面提到的问题，字节特意开发了物化列（Materialized Column）来解决这些问题。

[![image](https://cdn.nlark.com/yuque/0/2020/png/938149/1607867739243-5d5943b0-f0d9-4db0-b099-21b0492b90a1.png "image")](https://www.iteblog.com/pic/spark/spark_at_bytedance5-iteblog.png)

如果想及时了解Spark、Hadoop或者HBase相关的文章，欢迎关注微信公众号：**iteblog_hadoop**

假设我们的原表为 base_table，其中包含 item、count、people 以及 date 等字段，其中 date 是分区字段。

为了物化 age 这列，我们开发了 alter table xxx add columns 功能，直接使用 MATERIALIZED 关键字来创建物化列。从上面可以看出，其实是往表里面插入了一个名为 age 字段，这个字段是物化了 people 这列的 age 字段，并且是 int 类型。

[![image](https://cdn.nlark.com/yuque/0/2020/png/938149/1607867739225-fe411420-160f-49a4-896c-570a1cd54d02.png "image")](https://www.iteblog.com/pic/spark/spark_at_bytedance6-iteblog.png)

如果想及时了解Spark、Hadoop或者HBase相关的文章，欢迎关注微信公众号：**iteblog_hadoop**

当用户使用 select people.age from base_table 来查询数据，Spark 将会自动的把它 rewrite 成 select age from base_table，这些完全由 Spark SQL 自动完成。所以新的查询直接查询 base_table 里面的 age 字段，这样我们就无需把 people 字段里面的数据都读出来，而是直接读取 base_table 的 age 字段。因为 age 字段是 int 类型的，所以可以直接使用向量化读取，同时也支持过滤下推，性能自然上去了。

下面我们来介绍一下物化列的原理。

[![image](https://cdn.nlark.com/yuque/0/2020/png/938149/1607867739235-64e2a84c-ccab-4d89-928e-6bec5375a0df.png "image")](https://www.iteblog.com/pic/spark/spark_at_bytedance7-iteblog.png)

如果想及时了解Spark、Hadoop或者HBase相关的文章，欢迎关注微信公众号：**iteblog_hadoop**

我们往 base_table 表插入数据只需要填写三个字段：比如 insert into base_table partition(date='20201010') select 'appole', 1, map('age', '18', 'name', 'jack', 'gender', 'male')。不管我们有多少个物化列，插入数据的逻辑都不需要修改。

但是对于 Spark SQL 引擎而言，插入数据时需要考虑物化列，根据上面对 insert 的 explain 输出可以看出，age 那列被自动加上。

[![image](https://cdn.nlark.com/yuque/0/2020/png/938149/1607867739227-f5cf3fe5-b637-4255-ad1e-9283d7b0fa37.png "image")](https://www.iteblog.com/pic/spark/spark_at_bytedance8-iteblog.png)

如果想及时了解Spark、Hadoop或者HBase相关的文章，欢迎关注微信公众号：**iteblog_hadoop**

对于数据的读取，从上图的左边可以看出，对于非物化列的表，Spark 会读取 people 中读取 age 自动；对于物化列的表，Spark 会自动将 SQL 重写成对 age 自动的查询。

[![image](https://cdn.nlark.com/yuque/0/2020/png/938149/1607867739265-3ca58679-07c4-4f70-bae5-e8a9ded93cc2.png "image")](https://www.iteblog.com/pic/spark/spark_at_bytedance9-iteblog.png)

如果想及时了解Spark、Hadoop或者HBase相关的文章，欢迎关注微信公众号：**iteblog_hadoop**

上图是物化列的查询性能指标。可以看出，对于物化列的查询性能均有很大提升，查询读的数据量大大减少。