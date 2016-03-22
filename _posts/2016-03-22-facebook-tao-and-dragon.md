---
layout: post
title: Facebook TAO and Dragon
---

## TAO

### 要解决的问题

- “data set is not easily partitionable”
- “request rates that can spike significantly”


#### Memcache and MySQL解决方案的问题

> the code that product engineers had to write for storing and retrieving their data became quite complex

- "work with two data stores and very different data models"
- "some made mistakes that led to bugs"
- "changing table schemas as products evolved required coordination"

### TAO的设计

> The Objects and Associations API that they created was based on the graph data model and was initially implemented in PHP and ran on Facebook's web servers. It represented data items as nodes (objects), and relationships between them as edges (associations). The API was an immediate success, with several high-profile features, such as likes, pages, and events implemented entirely on objects and associations, with no direct memcache or MySQL calls.

最初一版的实现是通过PHP在前端机上实现的，不过也存在以下问题：

- "small incremental updates to a list of edges required invalidation of the entire item that stored the list in cache, reducing hit rate"：命中率问题
- "requests operating on a list of edges had to always transfer the entire list from memcache servers over to the web servers"：传输成本问题
- "avoiding thundering herds in a purely client-side implementation required a form of distributed coordination that was not available for memcache-backed data at the time"：```thundering herds```问题

2009年实现TAO服务：

![](https://scontent.xx.fbcdn.net/hphotos-xfa1/t39.2365-6/851564_432068826912694_1902555099_n.png)

- 倒排索引，automatically create or delete
- 对象操作，create / set-fields / get / delete 
- 时间属性，TAO uses the association time value to optimize the working set in cache and to improve hit rate

TAO支持的操作：

- GET，例如判断某条关系是否存在，获取一个对象数据等
- RANGE，类似列表查询
- COUNT，计数

### TAO的实现

- RAM and Flash memory
- MySQL to manage persistent storage
- Partitioned: collocation, reducing communication overhead and avoiding hot spots

![](https://scontent.xx.fbcdn.net/hphotos-xfa1/l/t39.2365-6/851557_1407238129490405_234022031_n.png)

#### 模型

数据模型：

```
Object: (id) -> (otype, (key -> value)*)
Assoc: (id1, atype, id2) -> (time, (key -> value)*)
```

API：

```
assoc_add(id1, atype, id2, time, (k→v)*)
assoc_delete(id1, atype, id2)
assoc_change type(id1, atype, id2, newtype)

assoc_get(id1, atype, id2set, high?, low?)
assoc_count(id1, atype)
assoc_range(id1, atype, pos, limit)
assoc_time_range(id1, atype, high, low, limit)
```

#### 存储的实现

- default all object types are stored in one table, and all association types in another: 逻辑上就两张大表
- that every association query can be served from a single server: 以一个大List形式存在一台服务器上
- Two ids are unlikely to map to the same server unless they were explicitly colocated at creation time: 这个比较奇怪，难道Hash不是根据oid决定的？
- additional index based on id1, atype, and time
- association counts are stored in a separate table

#### 缓存的实现

- in-memory cache contains objects, associ- ation lists, and association counts，计数也在缓存里
- association with an inverse may involve two shards: 也不提供强一致性，有异步任务进行修复
- 基于Memcached实现，thread-safe hash table, LRU eviction among items of equal size, and a dynamic slab rebalancer
- 缓存隔离，RAM into arenas: 保证重要```type```的命中率, 避免相互影响
- 计数类缓存指针浪费空间，using direct-mapped 8-way associative caches that require no pointers
- 存储效率优化，map (id1, atype) to a 32-bit count in 14 bytes; records the absence of any id2 for an (id1, atype), takes only 10 bytes

#### Sharding and Hot Spots

- consistent hashing
- rebalances
- caches the data and version

#### 超长关系链

- 超过一定长度的关系链，就不会全部放进缓存
- 导致很多判断请求都会穿透到DB
- 解决方案：一、借助```assoc_count```来进行反向判断；二、借助```create_time```来缩小判断范围。都是治标。

## Dragon

### 要解决的问题

>  This makes it even more challenging to show people the pieces of information that are most relevant to them

TAO关注如何高效的解决关系的存储和读取，而Dragon要解决读取有价值的关系。

> Dragon, a distributed graph query engine, is the next step in our evolution toward efficient query plans for more complex queries

重点是在```complex queries```：

- monitors real-time updates，实时反馈
- creates several different types of indices，生成推荐索引
- far less data is transferred over the network to the web tier，减少数据传输

### TAO的问题

> TAO can fetch several thousand associations at the same time, which is acceptable for most queries but can cause increased latency when the amount of data queried is large

从描述看，之前的实现是通过TAO硬查询，由于计算量的问题，导致性能下降。例如从夏奇拉的评论中找到几条热门的中文评论。从下面这张图，左边是之前基于TAO的实现需要拉取两条关系链：评论关系，语言关系；右边这张是```Dragon```优化后的，简单理解就是把两条关系链合并了。

![](https://scontent.xx.fbcdn.net/hphotos-xta1/t39.2365-6/12056977_1581065718881783_289342919_n.jpg)

- Fanout: 减少拉取次数
- Indices: 解决长尾查询问题
- a richer query language that supports a filter/orderby operator

> A typical photo upload on Facebook results in about 20 edges being written to MySQL and cached in RAM via TAO

> Data size grew 20x over six years; about half the storage requirement was for data about edges — but only a small fraction of it described the primary relationship between two entities (for example, Alice → [uploaded] → PhotoID and PhotoID → [uploaded by] → Alice).

### Dragon的设计

核心：建索引

> it’s possible to compute a key involving (PostID, Language) and seek directly to the posts of interest. We can also do more complex sorting on persistent storage — for example, (Language, Score, CommentID) — to reduce the cost of the query

下面这三张图是实现的对比：

- 第一张是TAO的机制，注意Shakira的```LikedBy```关系都在同一台机器上，这时候要快速的获取应该哪些应该推荐给```Alice```，性能就不是很好了
- 所以第二张图，Dragon把倒排索引分散到了多台机器上
- 第三张图更进一步，Dragon可以根据社交属性，把同一个圈子的用户分配到一台```LikedBy```索引服务上

重中之重应该是这个**offline graph partitioning techniques**

![](https://scontent.xx.fbcdn.net/hphotos-xpt1/t39.2365-6/12679505_245485612456373_1682089351_n.jpg)
![](https://scontent.xx.fbcdn.net/hphotos-xpf1/t39.2365-6/10734323_1051857198204592_777962040_n.jpg)
![](https://scontent.xx.fbcdn.net/hphotos-xtp1/t39.2365-6/12532939_1744819099066780_1085026126_n.jpg)

### 函数式语言

> Dragon supports 20 or so easy-to-use functional programming primitives to express filtering/ordering on edges in the graph, as well as a single graph navigation primitive called assoc

这应该是另外一个核心竞争力

```
(->> ($alice) (assoc $friends) (assoc $friends) (filter (> age 20)) (count))
```

### 竞争力

> Dragon is backed by a demand-filled, in-process key-value store, updated in real time and eventually consistent. It uses a number of optimization techniques to conserve storage, improve locality, and execute queries in 1 ms or 2 ms with high availability and consistency.

1 ms or 2ms，醉了......

> Dragon combines them at scale and in novel ways to push down many complex queries closer to storage

关键是怎么实现的......不明觉厉

## Reference:

[TAO: The power of the graph](https://code.facebook.com/posts/227813470702374/tao-the-power-of-the-graph/)

[TAO: Facebook’s Distributed Data Store for the Social Graph](https://www.usenix.org/conference/atc13/technical-sessions/presentation/bronson)

[Dragon: A distributed graph query engine](https://code.facebook.com/posts/1737605303120405/dragon-a-distributed-graph-query-engine/)
