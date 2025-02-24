## Redis与MySQL双写一致性如何保证？

![1646475282937](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/181444-181355.png)

### 什么是一致性

![1646475308142](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/181509-305428.png)

一致性就是数据保持一致，在分布式系统中，可以理解为多个节点中数据的值是一致的。

- **强一致性**：这种一致性级别是最符合用户直觉的，它要求系统写入什么，读出来的也会是什么，用户体验好，但实现起来往往对系统的性能影响大
- **弱一致性**：这种一致性级别约束了系统在写入成功后，不承诺立即可以读到写入的值，也不承诺多久之后数据能够达到一致，但会尽可能地保证到某个时间级别（比如秒级别）后，数据能够达到一致状态
- **最终一致性**：最终一致性是弱一致性的一个特例，系统会保证在一定时间内，能够达到一个数据一致的状态。这里之所以将最终一致性单独提出来，是因为它是弱一致性中非常推崇的一种一致性模型，也是业界在大型分布式系统的数据一致性上比较推崇的模型

### **三个经典的缓存模式**

缓存可以提升性能、缓解数据库压力，但是使用缓存也会导致数据**不一致性**的问题。一般我们是如何使用缓存呢？有三种经典的缓存使用模式：

- Cache-Aside Pattern
- Read-Through/Write-through
- Write-behind

### Cache-Aside Pattern

Cache-Aside Pattern，即**旁路缓存模式**，它的提出是为了尽可能地解决缓存与数据库的数据不一致问题。

#### Cache-Aside读流程

**Cache-Aside Pattern**的读请求流程如下：

![1646475380193](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/181621-656056.png)

1. 读的时候，先读缓存，缓存命中的话，直接返回数据
2. 缓存没有命中的话，就去读数据库，从数据库取出数据，放入缓存后，同时返回响应。

#### Cache-Aside 写流程

**Cache-Aside Pattern**的写请求流程如下：

![1646475399729](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/181640-253183.png)

更新的时候，先**更新数据库，然后再删除缓存**。

### **操作缓存的时候，到底是删除缓存呢，还是更新缓存？**

日常开发中，我们一般使用的就是**Cache-Aside**模式。 **Cache-Aside**在写入请求的时候，为什么是**删除缓存而不是更新缓存**呢？

![1646475476225](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/181757-604340.png)

我们在操作缓存的时候，到底应该删除缓存还是更新缓存呢？我们先来看个例子：

![1646475506060](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/181826-994079.png)

1. 线程A先发起一个写操作，第一步先更新数据库
2. 线程B再发起一个写操作，第二步更新了数据库
3. 由于网络等原因，线程B先更新了缓存
4. 线程A更新缓存。

这时候，缓存保存的是A的数据（老数据），数据库保存的是B的数据（新数据），数据**不一致**了，脏数据出现啦。如果是**删除缓存取代更新缓存**则不会出现这个脏数据问题。

**更新缓存相对于删除缓存**，还有两点劣势：

- 如果你写入的缓存值，是经过复杂计算才得到的话。更新缓存频率高的话，就浪费性能啦。
- 在写数据库场景多，读数据场景少的情况下，数据很多时候还没被读取到，又被更新了，这也浪费了性能呢(实际上，写多的场景，用缓存也不是很划算的,哈哈)

### **双写的情况下，先操作数据库还是先操作缓存？**

`Cache-Aside`缓存模式中，有些小伙伴还是会有疑问，在写请求过来的时候，为什么是**先操作数据库呢**？为什么**不先操作缓存**呢？

假设有A、B两个请求，请求A做更新操作，请求B做查询读取操作。

![1646475627752](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/182028-207219.png)

1. 线程A发起一个写操作，第一步del cache
2. 此时线程B发起一个读操作，cache miss
3. 线程B继续读DB，读出来一个老数据
4. 然后线程B把老数据设置入cache
5. 线程A写入DB最新的数据

酱紫就有问题啦，**缓存和数据库的数据不一致了。缓存保存的是老数据，数据库保存的是新数据**。因此，Cache-Aside缓存模式，选择了先操作数据库而不是先操作缓存。

- 个别小伙伴可能会问，先操作数据库再操作缓存，不一样也会导致数据不一致嘛？它俩又不是原子性操作的。这个是**会的**，但是这种方式，一般因为删除缓存失败等原因，才会导致脏数据，这个概率就很低。小伙伴们可以画下操作流程图，自己先分析下哈。接下来我们再来分析这种**删除缓存失败**的情况，**如何保证一致性**。

### **数据库和缓存数据保持强一致，可以嘛？**

实际上，没办法做到数据库与缓存**绝对的一致性**。

- 加锁可以嘛？并发写期间加锁，任何读操作不写入缓存？
- 缓存及数据库封装CAS乐观锁，更新缓存时通过lua脚本？
- 分布式事务，3PC？TCC？

其实，这是由**CAP理论**决定的。缓存系统适用的场景就是非强一致性的场景，它属于CAP中的AP。**个人觉得，追求绝对一致性的业务场景，不适合引入缓存**。

> CAP理论，指的是在一个分布式系统中， Consistency（一致性）、 Availability（可用性）、Partition tolerance（分区容错性），三者不可得兼。

但是，通过一些方案优化处理，是可以**保证弱一致性，最终一致性**的。

### **3种方案保证数据库与缓存的一致性**

#### 缓存延时双删

有些小伙伴可能会说，并不一定要先操作数据库呀，采用**缓存延时双删**策略，就可以保证数据的一致性啦。什么是延时双删呢？

![1646475829708](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/182351-101915.png)

1. 先删除缓存
2. 再更新数据库
3. 休眠一会（比如1秒），再次删除缓存。

这个休眠一会，一般多久呢？都是1秒？

> 这个休眠时间 =  读业务逻辑数据的耗时 + 几百毫秒。为了确保读请求结束，写请求可以删除读请求可能带来的缓存脏数据。

这种方案还算可以，只有休眠那一会（比如就那1秒），可能有脏数据，一般业务也会接受的。但是如果**第二次删除缓存失败**呢？缓存和数据库的数据还是可能不一致，对吧？给Key设置一个自然的expire过期时间，让它自动过期怎样？那业务要接受**过期时间**内，数据的不一致咯？还是有其他更佳方案呢？

#### 删除缓存重试机制

不管是**延时双删**还是**Cache-Aside的先操作数据库再删除缓存**，都可能会存在第二步的删除缓存失败，导致的数据不一致问题。可以使用这个方案优化：删除失败就多删除几次呀,保证删除缓存成功就可以了呀~ 所以可以引入**删除缓存重试机制**

![1646475896458](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/182457-505891.png)

1. 写请求更新数据库
2. 缓存因为某些原因，删除失败
3. 把删除失败的key放到消息队列
4. 消费消息队列的消息，获取要删除的key
5. 重试删除缓存操作

#### 读取biglog异步删除缓存

重试删除缓存机制还可以吧，就是会造成好多**业务代码入侵**。其实，还可以这样优化：通过数据库的**binlog来异步淘汰key**。

![1646475926313](https://tprzfbucket.oss-cn-beijing.aliyuncs.com/hadoop/202203/05/182530-707676.png)

以mysql为例吧

可以使用阿里的canal将binlog日志采集发送到MQ队列里面

然后通过ACK机制确认处理这条更新消息，删除缓存，保证数据缓存一致性


  