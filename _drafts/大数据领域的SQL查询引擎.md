---
title: 大数据领域的SQL查询引擎
categories:
- 其他
tags: [其他]
draft: true
layout: post
---

* 目录
{:toc}

# Presto
Presto是由Facebook开发的一个分布式SQL查询引擎，它被设计为用来专门进行高速、实时的数据分析。它的产生是为了解决Hive的MapReduce模型太慢以及不能通过BI或Dashboards直接展现HDFS数据等问题。Presto是一个纯粹的计算引擎，它不存储数据，其通过Connector获取第三方Storage服务的数据。

## Presto功能及优点包括
1. Ad-hoc，期望查询时间秒级或几分钟；
2. 比Hive快10倍；
3. 支持多数据源，如Hive、Kafka、MySQL、MonogoDB、Redis、JMX等，也可自己实现Connector；
4. Client Protocol: HTTP+JSON, support various languages(Python, Ruby, PHP, Node.js Java)；
5. 支持JDBC/ODBC连接；
6. ANSI SQL，支持窗口函数，join，聚合，复杂查询等。

## Presto的缺点包括
1. No fault tolerance；当一个Query分发到多个Worker去执行时，当有一个Worker因为各种原因查询失败，那么Master会感知到，整个Query也就查询失败了，而Presto并没有重试机制，所以需要用户方实现重试机制。
2. Memory Limitations for aggregations, huge joins；比如多表join需要很大的内存，由于Presto是纯内存计算，所以当内存不够时，Presto并不会将结果dump到磁盘上，所以查询也就失败了，但最新版本的Presto已支持写磁盘操作，这个待后续测试和调研。
3. MPP(Massively Parallel Processing )架构；这个并不能说其是一个缺点，因为MPP架构就是解决大量数据分析而产生的，但是其缺点也很明显，假如我们访问的是Hive数据源，如果其中一台Worker由于load问题，数据处理很慢，那么整个查询都会受到影响，因为上游需要等待上游结果。

# Apache Drill
Apache Drill是一个低延迟的分布式海量数据（涵盖结构化、半结构化以及嵌套数据）交互式查询引擎，使用ANSI SQL兼容语法，支持本地文件、HDFS、HBase、MongoDB等后端存储，支持Parquet、JSON、CSV、TSV、PSV等数据格式。本质上Apache Drill是一个分布式的mpp（大规模并行处理）查询层。Drill的目的在于支持更广泛的数据源，数据格式，以及查询语言。受Google的Dremel启发，Drill满足上千节点的PB级别数据的交互式商业智能分析场景。

## Apache Drill功能及优点包括
1. 支持自定义的嵌套数据集，数据灵活。
2. Hive一体化，完全支持hive包括hive的udf，并且支持自定义udf
3. 低延迟的sql查询，与常用sql有些不同，不过学习成本较低
4. 支持多数据源
5. 性能较高。

## Apache Drill缺点包括
1. drill语法和常规sql有区别,一般是如“select * from 插件名.表名”的形式。主要是因为drill查询不同数据源时需要切换不同的插件。下面分别是hive，hbase，文件系统三种数据源时候的查询sql。
    - hvie：select * from hive.test01;
    - hbase：select * from hbase.test01;
    - dfs：select * from dfs./home/liking/test.csv;
查出来之后是一列数据，若想对数据查询不同的列需要用columns[0]的形式来标记。
Select columns[0] from dfs./home/liking/test.csv。另外匹配的文件类型需要在插件的工作空间来配置，具体的配置请参考官方文档。
2. 技术线太长，不容易切合到实际生产线上去。
3. 国内使用较少，没有大型成功案例，不够大众化，出现问题可能维护起来比较困难。资料比较少。

# Apache Impala
Apache Impala的定位是一种新型的MPP查询引擎，但是它又不是典型的MPP类型的SQL引擎，提到MPP数据库首先想到的可能是GreenPlum，它的每一个节点完全独立，节点直接不共享数据，节点之间的信息传递全都通过网络实现。而Impala可以说是一个MPP计算引擎，它需要处理的数据存储在HDFS、Hbase或者Kudu之上，这些存储引擎都是独立于Impala的，可以称之为第三方存储引擎，Impala使用MPP的思想实现了计算。

## Apache Impala功能及优点
1. 支持SQL查询，快速查询大数据。
2. 可以对已有数据进行查询，减少数据的加载，转换。
3. 多种存储格式可以选择（Parquet, Text, Avro, RCFile, SequeenceFile）。
4. 可以与Hive配合使用。

## Apache Impala 缺点
1. 不支持用户定义函数UDF。
2. 不支持text域的全文搜索。
3. 不支持Transforms。
4. 不支持查询期的容错。
5. 对内存要求高。

# 统计汇总
以上调研的SQL查询引擎均多支持多数据源的查询，其中presto更是支持跨库和跨存储查询。不过总体上对机器的内存较高，不符合具体的应用场景，比如presto需要维护集群同时还得维护一套元数据；而drill本身的sql查询引起来比较复杂，不同的查询需要切换插件，用起来比较麻烦，且目前在国内应用的场景也比较少；Apache Impala则不支持用户自定义函数，查询期间的容错效果也不好，另外未见到支持跨库跨存储的计算功能。