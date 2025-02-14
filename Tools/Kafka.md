# Kafka
- kafka是一个分布式的基于发布/订阅模式的消息队列，主要应用于大数据实时处理领域。
- 不保证全局有序，只保证分区内有序。
### 参考视频
- [2019尚硅谷kafka](https://www.bilibili.com/video/BV1a4411B7V9?p=11&vd_source=e9f1ced96b267a4bc02ec41ca31d850a)
### 特点
- 读写
- 无删除
- 无更新
- 采用了分片和索引机制
### 概念
1. Producer
2. Consumer
3. Broker
4. Topic
- topic是逻辑概念。
- partition是物理概念，每个分区从0到n维护一个offset。
  - segment1
    - xxx.index只存索引
    - yyy.log只存数据
  - segment2
  - segment3
- log：默认存7天。达到log的最大size（1G）以后，重新创建一个新文件。
### 消费效率高的原因
- 索引和数据分离
- 索引大小固定
- 二分查找找到对应的index
### 大小关系
- Topic(m)
  - Broker(n)
    - partition(k)（master/slave）
- Broker为Kafka的消息服务端，负责消息的存储和转发。
- Topic为消息类别，Kafka按照Topic来对消息分类。


### 数据存储

- 每个partition分为多个segment，每个segment对应两个文件.index和.log

1. xxx.index只存索引
2. xxx.log只存数据

- 采用稀疏索引和二分查找的方式快速定位到消息，这就是为什么kafka数据存储在磁盘上，但是速度还是很快的原因。
- 顺序写磁盘比内存随机写还快。

- Partition中的每条Message都包含3个属性：
1. offset
2. MessageSize
3. Data
- 其中，Offset表示Message在这个Partition中的偏移量，它在逻辑上是一个值，唯一确定了Partition中的一条Message；MessageSize表示消息内容Data的大小；Data为Message的具体内容。
- 同一个partition内的消息是有序的。

### 消费者
- Consumer通过Pull方式消费消息。
- Kafka不删除已消费的消息。
- 同一个Consumer Group中的多个Consumer线程可以并发地消费Topic上的消息，Consumer的线程并发数一般等于Partition的个数。
- 同一个Consumer Group中的多个Consumer不能同时消费同一个Partition上的数据。

### 分区的原因

1. 方便在集群中扩展。每个partition可以通过调整大小以适应它所在的机器，而一个topic又可以由多个partition组成，因此整个集群就可以适应任意大小的数据了。
2. 可以提高并发。可以以partition为单位读写。
- 分区的规则：hash或者round-robin
### 数据可靠性
1. ISR：in-sync replicas 副本同步队列

2. AR：assigned replicas 所有副本

- 消费者采用拉取pull和长轮询的方式

- ZooKeeper为Kafka提供集群的管理。
- 它保存着集群的Broker、Topic、Partition等元数据，还负责Broker故障发现、Leader选举、负载均衡等。
### 副本

- leader 具有通信的话语权
- follower

1. producer和consumer只和leader进行交互。
2. follower不停地从leader同步数据。

- HW：high watermark 高水平位

- LEO：log end offset 最新消息

- 全同步后才对HW+1

- ISR：leader维护了一个动态的ISR，表示和leader保持同步的follower集合。当ISR中的follower完成数据的同步之后，leader就会给producer发送ack。如果follower长时间未向leader同步数据，则该follower将被踢出ISR。leader发生故障以后，就会从ISR中选举新的leader。

### ACK应答机制

- ack=1：producer等待broker的ack，partition的leader落盘成功后返回ack，如果在follower同步成功之前leader故障，那么将会丢失数据。
- ack=0：producer不等待broker的ack，低延迟，broker一旦接收到数据还没有写入磁盘就已经返回了，当broker故障时有可能丢失数据。
- ack=-1(all)：producer等待broker的ack，partition的leader和follower(指的是ISR中的follower)全部落盘成功后才返回ack。但是如果在follower同步完成后，broker发送ack之前，leader发生故障，那么会造成**数据重复**

### Exactly Once

- 精准一次性：数据不重复不丢失

1. at most once：ack = 0 数据最多发送一次

2. at least once：ack = -1 数据至少发送一次

3. exactly once = 幂等性 + at least once

- 幂等性：Producer不论向broker发送多少次重复数据，broker都只会持久化一条。

- 启用幂等性，只需要将Producer的参数中enable.idompotence设置为true。

- 开启幂等性的producer在初始化的时候会被分配一个PID，发往同一个partition的消息会附带sequence number。而Broker端会对<PID，partition，seqNumber>做缓存，当具有相同主键的消息提交时，Broker只会持久化一条。




