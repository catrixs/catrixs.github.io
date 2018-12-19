---
layout: post
title: 图数据库
---

本文将介绍Graph数据库底层实现机制，与SQL，NoSQL有什么不同。Graph数据库概念算是与SQL同样古老了，但一直缺少大规模实践案例。但随着互联网的发展，公司能够收集的数据种类与规模越来越多，通过SQL（NoSQL）对数据建模工作量越来越大，同时挖掘数据关联的SQL Join爆炸，难以维护。所以Graph数据库开始受到重视。

Graph数据库开源实现，以及服务提供商有很多，例如：Neo4j、Titan、Amazon Neptune、AnzoGraph、TigerGraph、JanusGraph、Arangodb，本文主要分析Neo4j、Titan的实现机制。

## 数据模型

下面以微博、Twitter常见的好友关系，来看一下SQL、NoSQL、TiTan、Neo4j的实现有什么不同。

下图是好友关系的示意图，我们希望数据库的实现能够高效的完成以下任务：

1. 获取用户A的Follows（出边）
2. 获取用户A的Followers（入边）
3. 判断用户A是否Follows用户B，用户A是否被用户B Follows
4. 获取A的好友（A Follows，且Follows A的用户）
5. 获取A的Follows中所在地是北京的用户

![](/images/graph.png)

### SQL

SQL Schema要对User和User之间的关系建模，User表比较简单：

| UID  | Name | Location |
| ----:| -----:| -------:|
| 1 | Ruth | 北京 |
| 2 | Harry | 北京 |
| 2 | Billy | 上海 |

Follows表要表示User之间的关系：

| From | To | Type |
| ----:| -----:| -------:|
| 1 | 2 | Follows |
| 1 | 3 | Follows |
| 2 | 1 | Follows |
| 2 | 3 | Follows |
| 3 | 2 | Follows |

在数据量较小的情况下，这种表结构能够能够很好的完成任务1、2、3、5，但是任务4会稍微麻烦一点：

```sql
select follows.from_id from follows
	inner join 
		(select a.from_id, a.to_id from follows as a where a.from_id = ?) as a_follows
	on follows.from_id = a_follows.to_id
	where follows.to_id = ?
```

在微博、Twitter数据量级场景下，分库分表扩展方式会导致任务2、4业务实现难度大幅增加。一种解决方案是对任务2、4分别建表，而任务2建表还面临数据倾斜问题，在微博场景下，一个大V的粉丝都会在一张表中，无法彻底解决Partition问题。Graph Partition算法可以缓解这个问题，不过已经超出本文讨论范围，后续可以专门写一篇讨论Graph Partition问题。

为了避免分库分表带来的业务实现复杂度问题，可以引入SQL Cluster（像TiDB）来解决扩展性问题，join时尽可能的将计算推到存储节点来提升性能。

### NoSQL

Google当年提出BigTable来解决SQL的扩展问题，来表示Web之间的关系。上面提到的出边SQL表数据倾斜问题，可以通过HBase来解决，俗称“宽表”。Column在HBase底层是有序存储，可以利用这个有序性实现：“获取最近5000个粉丝”的需求。同时一个Row Key的所有Column又都分布在一个或相邻的几个Region内，这样查询复杂度可以保持在O(1)以内。

不过受限HBase底层实现机制，宽表也不能无限宽。Column的遍历，是随宽度线性增长的。当宽度超过几千万甚至过亿时，就无法满足“获取最早5000个粉丝”的需求了。

| Row Key | Column1 | Column2 | ... | Columnn |
|:----- |:----- |:----- |:----- |:----- |
| 1 | 2|
| 2 | 1 | 3 |
| 3 | 1 | 2 |

#### BigTable方式（Titan）

下面终于可以看看神秘的Graph数据库底层是如何实现的，怎么解决上面提到的问题。Titan的实现，与上面提到的NoSQL方式很相似（实际上Titan没有实现存储引擎，^_^）。Titan对关系的建模，是把一个顶点的出+入边都以Column形式存储到顶点的Row Key下面。这样做出入的遍历都复杂度都可以做到O(1)（这里理解，应该是为了便面散出到其他Partition），但代价是每条边都要存储两遍。

![](http://s3.thinkaurelius.com/docs/titan/1.0.0/images/titanstoragelayout.png)

另外这里还需要提到Titan实现上的一个小优化。Titan在存储边的Column时，`adjacent vertext id`并不是实际的`vertext id`，而是`dest vertext id - src vertext id`，这样对于顶点量特别大的情况，可以节省很多存储空间（小值可以更好的被压缩）。

![](http://s3.thinkaurelius.com/docs/titan/1.0.0/images/relationlayout.png)

在这种模型下，可以看出任务1、2、3、5实现复杂度比较简单。但是任务4实现仍有难度，散出的Row Key数量会比较大。

#### Doubly Linked Lists方式（Neo4j）

因为Titan没有实现自己的存储引起，所以它的数据模型很大程度上是受BigTable模型限制的。而Neo4j实现了自己的存储引擎，接下来看一看它是如何做的。

上面提到SQL的follows表在实现任务4时，复杂度比较高。主要原因是因为SQL表是单向的，可以抽象理解成单向链表（只能从From指向To）。Titan的解决方案是把边多存一份，出和入边都存。Neo4j实现思路是直接把follows表改成双向链表Relationship Chain（参考代码：[RelationshipCreator.relationshipCreate](https://github.com/neo4j/neo4j/blob/f7b3d2f991a3aa3f5b25d1add913e8d65f9d4994/community/kernel/src/main/java/org/neo4j/kernel/impl/storageengine/impl/recordstorage/RelationshipCreator.java#L57)），Neo4j存储结构示意图：

![](/images/neo4j-rel-chain.png)

因为Neo4j的Node、Relation存储都是定长的（每个Node占用9B，每条边占用33B），检索速度就变成了在一个“大数组”内通过指针寻址。例如遍历节点的边，只需要用边的ID乘以33B就是边所在存储文件中位置。但是这种设计也以为着磁盘的随机读会比较严重，所以最好使用SSD，或者留出比较大的Page Cache给系统。而且这种存储结构设计，依然解决不了Partition的问题，虽然Neo4j的商业版本支持集群模式，但是估计无法承载互联网的数据规模。

## 总结

综合看，SQL Cluster和Titan是比较有前途的图数据库方案。

## 参考

[1. Titan Documentation](http://s3.thinkaurelius.com/docs/titan/1.0.0/index.html)

[2. 《Graph Databases NEW OPPORTUNITIES FOR CONNECTED DATA》](http://graphdatabases.com/)