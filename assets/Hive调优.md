## Explain查看执行计划

生产环境中的任务一般执行时间都是几十分钟到几个小时，不可能每次都通过任务执行完成时间来评估任务的执行效率，可以通过explain查看执行计划来分析任务的执行效率。

**基本语法**

`EXPLAIN [EXTENDED | DEPENDENCY | AUTHORIZATION] query`

* extended：显示详细执行计划
* dependency：查看依赖的输入表、分区等信息

**案例实操**

```sql
explain select * from emp;
--没有生成MR，使用了Fetch Operator

explain select deptno, avg(sal) avg_sal from emp group by deptno;
--生成了MR，使用了Map Operator Tree, Reduce Operator Tree, Select Operator, File Output Operator和Fetch Operator
```

## 1.建表优化

分区表、分桶表、合适的文件存储格式和压缩格式

## 2.对中间和最终结果数据启用压缩

```sql
set hive.exec.compress.intermediate=true;
set hive.intermediate.compression.codec=org.apache.hadoop.io.compress.SnappyCodec;
set hive.intermediate.compression.type=BLOCK;
set hive.exec.compress.output=true;
```

## 3.尽早过滤

- 空值去除与转换：空值可去则去，不可去除则转换为不影响结果的值。
- 分区裁剪和列裁剪：查询时只指定必要的列和分区，避免全表扫描读取无关数据，减少内存压力和计算耗时
- 先过滤再join：类似`a join b on condition where ... `的查询语句，会先join再过滤where，可以用where过滤后再进行join，优化join的执行效率。(现在即使不自己写子查询，谓词下推也会优化为where先过滤)

## 4.group by开启map预聚合和负载均衡

- 开启map端预聚合：`set hive.map.aggr=true;`
- 数据倾斜时：`set hive.groupby.skewindata=true;`相当于打随机前后缀，会多一个MR任务将之前进入同一个节点的数据打散到多个节点，预处理之后再发往同一个Reduce中进行最终的聚合，此法需注意权衡性能的得失

## 5.向量化执行

Hive中的向量化查询执行大大减少了典型查询操作（如scan, filter, aggregate和join)的CPU使用率。

向量化执行将单行运算转换为批量运算(默认1024行一批，即一组列向量)，充分利用CPU。

要使用向量化执行，必须以ORC格式存储数据，并设置：

```sql
set hive.vectorized.execution.enabled=true;
set hive.vectorized.execution.reduce.enabled=true;
```

最新版本的Hive默认开启向量化执行。

## 6.谓词下推

默认生成的执行计划会在可见的位置执行过滤器，但在某些情况下，某些过滤器表达式可以被推到更接近首次看到此特定数据的运算符的位置。

比如下面的查询：

```sql
select
    a.*,
    b.* 
from
    a join b on (a.col1 = b.col1)
where a.col1 > 15 and b.col2 > 16
```

如果没有谓词下推，则在完成JOIN处理之后将执行过滤条件**(a.col1> 15和b.col2> 16)**。因此，在这种情况下，JOIN将首先发生，并且可能产生更多的行，然后在进行过滤操作。

使用谓词下推，这两个谓词**(a.col1> 15和b.col2> 16)**将在JOIN之前被处理，因此它可能会从a和b中过滤掉连接中较早处理的大部分数据行，因此，建议启用谓词下推。

通过将hive.optimize.ppd设置为true可以启用谓词下推(默认开启)。

```text
SET hive.optimize.ppd=true
```

## 7.基于成本优化CBO

Hive在提交最终执行之前会优化每个查询的逻辑和物理执行计划。基于成本的优化会根据查询成本进行进一步的优化，从而可能产生不同的决策：比如如何决定JOIN的顺序，执行哪种类型的JOIN以及并行度等。

可以通过设置以下参数来启用基于成本的优化。

```text
set hive.cbo.enable=true;
set hive.compute.query.using.stats=true;
set hive.stats.fetch.column.stats=true;
set hive.stats.fetch.partition.stats=true;
```

可以使用统计信息来优化查询以提高性能。基于成本的优化器（CBO）还使用统计信息来比较查询计划并选择最佳计划。通过查看统计信息而不是运行查询，效率会很高。

收集表的列统计信息：

```text
ANALYZE TABLE mytable COMPUTE STATISTICS FOR COLUMNS;
```

查看my_db数据库中my_table中my_id列的列统计信息：

```text
DESCRIBE FORMATTED my_db.my_table my_id
```

## 8.join优化

### 小表join大表：使用Map Join

普通的join是在reduce端进行的，而在小表join大表时，可以启用Map Join**把小表加载到每个mapper的内存** ，在map端进行join，没有了reduce，避免了shuffle。

**map join 原理**：

1. 小表所在节点将小表数据转换为一个**HashTable** 文件，然后分发到各个map节点的内存中，形成**DistributeCache**
2. 各mapper扫描大表，每一条记录去和DistributeCache进行关联，并直接输出结果，总输出文件的个数为map task的个数。

![](https://yeluoqianqiu.github.io/assets/20210714005658.jpeg)

开启map join

```sql
set hive.auto.convert.join = true; //默认为true

--小表的阈值设置（默认25M）
set hive.mapjoin.smalltable.filesize=25000000;
```

早期版本中，需要将小表放在左边，大表放在右边，才会启用map join优化，新版hive中小表JOIN大表和大表JOIN小表已经没有区别了。

显式使用map join：

```sql
insert overwrite table res
select /*+mapjoin(s)*/ b.id, ...
from bigtable  b
join smalltable  s
on s.id = b.id;
```

### 大表join大表：使用sort merge bucket join

两个表join的时候，小表不足以放到内存中，但是又想用map side join这个时候就要用到bucket Map join。两个表根据**join key**创建为分桶表，**较小表的桶数应是较大表桶数的整数倍**。两表join时，小表依然复制到所有节点，Map join的时候，小表的每一组bucket加载成hashtable，与对应的一个大表bucket做局部join，这样每次只需要加载部分hashtable就可以了。

## 9.合理设置map和reduce个数

几个问题：

- 是不是map数越多越好？

不是。如果一个任务有很多小文件（远远小于块大小128m），则每个小文件也会被当做一个块，用一个map任务来完成，而一个map任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。

- 是不是保证每个map处理接近128m的文件块，就高枕无忧了？

不一定。比如有一个127m的文件，正常会用一个map去完成，但**这个文件只有一个或者两个小字段，却有几千万的记录** ，如果map处理的逻辑比较复杂，用一个map任务去做，肯定也比较耗时。

- reduce是不是越多越好？

过多的启动和初始化reduce也会消耗时间和资源；另外，**有多少个** **reduce** **，就会有多少个输出文件** ，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；

因此需要合理设置map和reduce的个数。

合理设置Map数：

map task的数量与参数`mapreduce.job.maps`无关(MR 2.x之后该参数无任何作用)，而是**与InputSplits大小有关**，计算公式为`computeSplitSize(Math.max(minSize,Math.min(maxSize,blocksize)))`，设置maxSize小于blocksize，可以增加map task的个数，而设置minSize大于blockSize则可以减少map task的个数。

案例实操：

```shell
hive (default)> select count(*) from emp;
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 1

# 设置最大切片值为100字节
hive (default)> set mapreduce.input.fileinputformat.split.maxsize=100;
hive (default)> select count(*) from emp;
Hadoop job information for Stage-1: number of mappers: 6; number of reducers: 1
```

合理设置Reduce数：

```shell
# 方法1
--（1）参数1：每个Reduce处理的数据量默认是256MB
hive.exec.reducers.bytes.per.reducer=256000000
--（2）参数2：每个任务最大的reduce数，默认为1009
hive.exec.reducers.max=1009
--（3）计算reducer数的公式
N=min(参数2，总输入数据量/参数1)

# 方法2
set mapreduce.job.reduces = 15; 
```

## 10.其他

- 避免使用count(distinct)：单独的distinct执行计划同group by，而count(distinct)会在**一个reducer**里执行去重，优化方式是先在子查询进行GROUP BY或distinct，再求COUNT(*)，但同样需要注意 group by 可能造成的数据倾斜问题。(spark中count(distinct)已经优化，hive未知)
- 避免笛卡尔积：join的时候不加on条件，或者on条件无效，或显式使用cross join，都会导致笛卡尔积，对笛卡尔积的计算只在**一个reducer**中完成，hive默认禁止笛卡尔积。
- 小文件开启JVM重用：每一个map或reduce都是一个Java进程，开启JVM需要一定的时空开销，特别是在处理包含大量task(如：大量小文件)的job时，开销是惊人的。`mapreduce.job.jvm.numtasks`可以设定JVM重用的次数，默认为1，设为-1时，表示无限制。JVM重用非常适合处理包含大量小文件的job，因为这些小task的执行时间可能还不及开启JVM的时间，一般可将参数设为10~20，而对于长时间运行的task，JVM重用的提升非常有限，应保证每个task开启新的JVM，因为JVM长时间运行产生的内存碎片反而会降低性能。
- 推测执行：`set mapred.map.tasks.speculative.execution = true;`默认是 true

