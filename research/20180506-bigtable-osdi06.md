---
title: A Distributed Storage System for Structured Data
date: 2018-05-06 16:49:20
tags: [bigtable]
category: BigData
---

## 简介

BigTable是Google提出的一个分布式的海量数据存储系统。Google将其运用在一些数据量较大的应用中。从分布式系统CAP角度理解，BigTable是牺牲高可用性，主要加强数据一致性和可扩展性。

早期BigTable设计主要解决，海量的网页数据存储。搜索引擎抓取的海量网页数据，需要做存储和分析，而且是大批量的数据写入，需要一个高可扩展和高速写入性能的一个数据库系统，同时能提供简单的快速过滤返回符合条件的数据。

## 特点

设计简单

> 基于分布式文件系统，把数据库和底层存储分离，让两者实现起来容易很多，各自实现好自己的领域。坏处是，任何一个tablet故障，恢复数据需要通过读取其他机器log恢复，造成可用性问题。 

Row Key有序

> 行键上顺序存储，因为有中心设计，很方便做split/move操作，让row key有序的前提下尽可能的balance数据，分布式查询利用多机资源，响应快，对于按顺序读取的应用来说是非常有好的。

Data Mode

> 多数据版本实现，timestamp对于很多数据应用非常方便。

高压缩比

> 典型的重复内容快速压缩， 以CPU带宽换硬盘带宽。属于同类型ROWS临近存储，高度压缩。

数据模型

![](https://github.com/jikelab/paper/raw/master/research/img/bigtable-data-model.png)

* row_key 反转的URL，为了是属于同一类型数据临近存储。

* column_key 引用的row_key的网站URL，灵活变更column。

* timestamp 一个单元格不同版本跟进时间戳降序顺序存储，灵活访问任意版本，无疑增加维护成本。

API 

> 支持数据的增删改查，提供单行事物，不支持跨行事物，跨行事物代价高昂，不利于高并发写入。中高性能写入，快速过滤返回数据，一般只能支持简单数据分析，很难做复杂聚合与关联查询，也可配合MapReduce系统实现，需要忍受性能低下。

## Tablet实现

有一个Master Server，一个支持多语言客户端，多个Tablet Server。通过添加或删除支持集群的动态扩展。每个Tablet Server管理着一个tablet集合。在每个tablet server上，存放100-1000个tablet。当存储的tablet超过一个默认的大小就会自动进行拆分为多个tablet，尽可能负载到更多机器。每个tablet大小为100-200M。每个Tablet包含某个区域内所有的数据，如果没有尽可能拆分分散到多台机器，可能会导致查询特别缓慢，造成数据热点问题。

* Tablet位置信息

存储tablet位置信息使用类似三层b+树架构。

![](https://github.com/jikelab/paper/raw/master/research/img/three-bplus-design.png)

如上引入一个第三方系统，用来保障Root Table位置信息的一致性与可获得性。

* Tablet worker

通过chubby锁，发现其他Table Server；记录redo，持久化存储于GFS，分组提交redo，提升性能。

存储使用SSTable与memtable，按照字典排序的数据结构，sstable与memtable合并视图执行非常高效。

* compactions

是一个很大的坑，由于支持tablet拆分，并且基于sstable与memtable合并生成，支持数据多版本，数据变更清理，数据多版本对业务应用友好，带来如下两个印象系统性能的操作：

  + minor compation
  + major compation

随着写操作的执行，memtable 的尺寸逐渐增加。当 memtable 的尺寸到达一个门槛值的 时候，memtable 就被冻结，就创建一个新的 memtable，被冻结的 memtable 就转化成一个 SSTable，并被写入到 GFS。这个“次压缩”(minor compaction)过程有两个目标:(1)它 缩减了 tablet 服务器的内存使用率;(2)当发生服务器死亡需要恢复时，它减少了需要从重 做日志中读取的数据量。当压缩过程正在进行时，正在到达的读写操作仍然可以继续进行。

个合并压缩程序，把所有的 SSTable 的数据重写到一个 SSTable，这个合并压缩被称 为“主压缩”(major compaction)。主要作用是回收被删除数据占用的资源，数据合并压缩写新的SSTable意味着需要大量的磁盘IO和网络IO，如果大量Major Compaction，会对在线业务造成访问影响。

* 实现细节

locality group 的 SSTable 进行压缩算法可由客户端指定。压缩使用“两段自定义压缩模式”，第一遍使用 Bentley and McIlroy[模式，它对一个大窗口内的长公共字符串进行压缩。第二遍使用一个快速的压缩算法，这个 压缩算法在一个 16KB 数据量的窗口内寻找重复数据。

缓存读取性能，scan缓存和block缓存，对于连续读取同一批数据，连续执行同一scan将可快速从内存读取数据，避免磁盘IO，获得加大性能提升。

Bloom Filters，建立于tablet只是，可以快速跳过大量不符合数据区域的table，减少磁盘访问，提升性能。

## 可扩展性

scaling，随着tablet server从几个扩展到几十个，每个服务器的吞吐量显著的降低了，这是由于多服务器负载不均衡引起的，常由于有其他进程争抢 CPU 资源和网络带宽。而忘了饱和也是重大影响因素之一。

## 应用实践

[1] google Analytics  网站流量分析应用，非结构化网页数据存储，全世界网站，数量庞大。

[2] Google Earth    位置数据存储+不同级别的高清图片街景。

[3] 个性化搜索      根据用户搜索历史记录，通过MapReduce来计算数据，给用户提供个性化推荐。

## 总结

通过系统的拆分，简化系统设计和实现的复杂度，这是工程级别的实践经验。系统设计有很多权衡，为了获得极大的吞吐量与可扩展性，就需要牺牲数据可靠性；分布式系统数据负载均衡，造成大量资源的占用，极大影响业务；SSTable + LSM Tree 开源实现LevenDB，就是BigTable核心功能。分布式系统的开发与测试是非常困难的，问题很难追踪与复现，所以如果没有足够的业务场景检验与实践，很难写出非常易用的分布式系统，需要足够强的驱动力来帮助塑造健壮的分布式系统。工业界经过BigTable的启发已经实现了很多类BigTable分布式系统，设计与实现大体类似，只有细微差别，主要在解决问题的方向权衡，导致了不同的实现。

今天大量互联网公司使用BigTable开源版实现Hbase，但是我们发现很多企业级用户很难把Hbase使用起来，究其原因，使用门槛与应用场景的局限。随着时间的发展Google内部也在不断迭代更新，因为BigTable未能实现跨行事物，仅支持单行事物，而业务需求肯定需要跨行事物的支持。于是新的系统诞生了，基于Spanner+F1的全球分布式一致性数据库，一大波工业界的极客们又开始无私的基于spanner+f1的论文写具体实现，并且完全开源，以此来解决分布式数据库跨行事物和横向扩展问题，解决困扰多年的单机数据库分库分表的痛苦。

在NoSQL之后，给它取了一个新名字叫NewSQL。

对于BigTable系统，实现起来应该相对复杂，论文很多地方也未详述，希望通过以上内容帮助了解一个分布式系统设计涉及的关键技术与权衡，而不是过多关注code细节，我也无实现经验，主要还是工作中很难有这样的机会吧。

关于NewSQL系统，我关注良久，有很多创新，在未来的文章中我们会更多谈论NewSQL。

**推荐阅读：**

[1] [分布式文件系统设计与实现](http://itweet.cn/blog/2018/04/20/distributed-file-system-design)

[2] [MapReduce设计与实现](http://itweet.cn/blog/2018/04/23/mapreduce-osdi04)

[3] [BigTable设计与实现](http://itweet.cn/blog/2018/04/23/bigtable-osdi06)

欢迎关注微信公众号[Whoami]，阅读更多内容。

![Whoami公众号](https://github.com/itweet/labs/raw/master/common/img/weixin_public.gif)

原创文章，转载请注明： 转载自[Itweet](http://www.itweet.cn)的博客
`本博客的文章集合:` http://www.itweet.cn/blog/archive




