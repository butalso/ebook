# MQ-kafka

## 什么是kafka
Kafka起初是由Linkedin公司采用Scala语言开发的一个多分区、多副本且基于ZooKeeper协调的分布式消息系统，现己被捐献给Apache基金会。目前Kafka已经定位为一个分布式流式处理平台，它以高吞吐、可持久化、可水平扩展、支持流数据处理等多种特性而被广泛使用。

## kafka的用途有那些，使用场景如何
### 用途
* 消息系统：kafka与其他消息中间件都具备系统解耦、流量削峰、异步处理、弹性伸缩等功能。同时，kafka还提供其他大多数消息中间件难以实现的消息顺序性保障和回溯消费的功能。
* 存储系统：kafka可将消息持久化到磁盘，且可以多副本存储，相比于其他基于内存存储的系统而言，有效的降低了数据丢失的风险。
* 流式处理平台
### 使用场景
* 异步处理：非核心流程异步化，减少系统响应时间，提高吞吐量。例如：短信通知、终端状态推送、App推送、用户注册等。
* 系统解耦：系统之间不是强耦合的，消息接受者可以随意增加，而不需要修改消息发送者的代码。消息发送者的成功不依赖消息接受者（比如：有些银行接口不稳定，但调用方并不需要依赖这些接口）。
* 最终一致性：最终一致性不是消息队列的必备特性，但确实可以依靠消息队列来做最终一致性的事情。先写消息再操作，确保操作完成后再修改消息状态。定时任务补偿机制实现消息可靠发送接收、业务操作的可靠执行，要注意消息重复与幂等设计。所有不保证 100% 不丢消息 的消息队列，理论上无法实现 最终一致性。
* 流量削峰：当上下游系统处理能力存在差距的时候，利用消息队列做一个通用的“漏斗”，进行限流控制。在下游有能力处理的时候，再进行分发。
* 日志处理：将消息队列用在日志处理中，比如Kafka的应用，解决海量日志传输和缓冲的问题。

## Kafka中的ISR、AR又代表什么？ISR的伸缩又指什么
`AR=ISR+OSR`
* ISR: in-sync replicas，和leader副本保持在可接受滞后范围的副本列表
* OSR：out-of-sync replicas，与leader副本同步滞后过多的副本列表
* AR: asigned replicas，所有副本列表
### ISR伸缩
ISR伸缩指的是ISR集合副本数量的扩张和收缩。kafka在启动时会开启两个和ISR相关的任务，`isr-expiration`和`isr-change-propagation`，`isr-expiration`任务周期性的检测每个分区是否需要缩减其ISR集合，当检测到ISR集合中有失效副本时，就会收缩ISR集合。同样，随着follower副本不断与leader副本进行消息同步，follower副本的LEO会逐渐后移，并最终赶上leader副本（LEO不小于leader副本的HW），此时，follower副本可以加入ISR集合。当ISR集合发生增减、或者ISR集合的任一副本的LEO发生变化时，都会影响整个分区的HW。`isr-change-propagation`任务周期性检查`isrChangeSet`，当发现有ISR集合变更记录，就会在Zookeeper的`/isr_change_notification`路径下创建以`_isr_change_`开头的持久顺序节点，Kafka控制器监听该节点并向它管理的broker节点发送变更元数据的请求。
#### 失效副本
失效副本判定两个参数：
* replica.lag.max.messages：当follower副本滞后leader副本消息数超过该参数时，则判定为失效
* replica.lag.time.max.ms：当follower副本上一次将leader副本LEO之前的日志全部同步时的lastCaughtUpTimeMs值大于该参数时，则判定为失效

一般产生原因有：
* follower副本进程卡住，在一段时间内根本没有向leader副本发起同步请求，比如频繁的FullGC
* follower副本进程同步过慢，在一段时间内都无法追赶上leader副本，比如IO开销过大

## Kafka中的HW、LEO、LSO、LW等分别代表什么？
* HW：High Watermark，高水位，HW之前的消息均可被消费者消费。`HW=min(ISR.LEO)`
* LEO：Log End Offset，日志末端偏移，是下一条消息写入的位置
* LSO：Last Stable Offset，对未完成的事务而言，LSO的值等于事务中第一条消息的位置(firstUnstableOffset)，对已完成的事务而言，它的值同HW相同。`LSO<=HW<=LEO`
* LW：Low Watermark，低水位，代表AR集合中最小的LogStartOffset值

## Kafka中是怎么体现消息顺序性的？
kafka每个partition中的消息在写入时都是有序的，消费时，每个partition只能被每一个group中的一个消费者消费，保证了消费时也是有序的。
整个topic不保证有序。如果为了保证topic整个有序，那么将partition调整为1。值得一提的是，当acks参数为非0值，且max.in.flight.requests.per.connection参数为配置大于1的值时，那就有可能出现错序现象，比如第一批消息写入失败，第二批消息写入成功，生产者重试发送第一批消息写入成功，那么这两个批次的消息就出现了错序。

## Kafka中分区器、序列化器、拦截器是否了解？它们之间的处理顺序是什么？
* 生产者顺序：拦截器->序列化器->分区器
* 消费者：反序列化器->拦截器
### 拦截器
#### 生产者拦截器
拦截器可以用来在消息发送前做一些准备工作，比如按照某个规则过滤不符合要求的消息、修改消息的内容等，也可以用来在发送回调逻辑前做一些定制化的需求，比如统计类工作。包含三个方法`onSend,onAcknowledgement,close`
* onSend：在消息序列化和计算分区之前调用
* onAcknowledgement：消息被应答之前或消息发送失败时调用
* close：关闭拦截器时执行一些资源的清理工作
#### 消费者拦截器

### 序列化器&反序列化器
生产者需要用序列化器把对象转换成字节数组才能通过网络发送给Kafka，而在对侧，消费者需要用反序列化器把从Kafka中收到的字节数组转换成相应的对象。生产者使用的序列化器和消费者使用的反序列化器是需要一一对应的。序列化器包含三个方法`configure,serialize,close`，反序列化器包含三个方法`configure,deserialize,close`
### 分区器
消息经过序列化之后就需要确定它发往的分区，如果消息ProducerRecord中指定了partition字段，那么就不需要分区器的作用，因为partition代表的就是所要发往的分区号。如果消息ProducerRecord中没有指定partition字段，那么就需要依赖分区器。分区器根据key字段来计算partition的值，为消息分配分区。分区器包含三个方法`configure,partition,close`


## Kafka生产者客户端的整体结构是什么样子的？

Kafka生产者客户端中使用了几个线程来处理？分别是什么？

Kafka的旧版Scala的消费者客户端的设计有什么缺陷？

“消费组中的消费者个数如果超过topic的分区，那么就会有消费者消费不到数据”这句话是否正确？如果不正确，那么有没有什么hack的手段？

消费者提交消费位移时提交的是当前消费到的最新消息的offset还是offset+1?

有哪些情形会造成重复消费？

那些情景下会造成消息漏消费？

KafkaConsumer是非线程安全的，那么怎么样实现多线程消费？

简述消费者与消费组之间的关系

当你使用kafka-topics.sh创建（删除）了一个topic之后，Kafka背后会执行什么逻辑？

topic的分区数可不可以增加？如果可以怎么增加？如果不可以，那又是为什么？

topic的分区数可不可以减少？如果可以怎么减少？如果不可以，那又是为什么？

创建topic时如何选择合适的分区数？

Kafka目前有那些内部topic，它们都有什么特征？各自的作用又是什么？

优先副本是什么？它有什么特殊的作用？

Kafka有哪几处地方有分区分配的概念？简述大致的过程及原理

简述Kafka的日志目录结构

Kafka中有那些索引文件？

如果我指定了一个offset，Kafka怎么查找到对应的消息？

如果我指定了一个timestamp，Kafka怎么查找到对应的消息？

聊一聊你对Kafka的Log Retention的理解

聊一聊你对Kafka的Log Compaction的理解

聊一聊你对Kafka底层存储的理解（页缓存、内核层、块层、设备层）

聊一聊Kafka的延时操作的原理

聊一聊Kafka控制器的作用

消费再均衡的原理是什么？（提示：消费者协调器和消费组协调器）

Kafka中的幂等是怎么实现的

Kafka中的事务是怎么实现的（这题我去面试6家被问4次，照着答案念也要念十几分钟，面试官简直凑不要脸。实在记不住的话...只要简历上不写精通Kafka一般不会问到，我简历上写的是“熟悉Kafka，了解RabbitMQ....”）

Kafka中有那些地方需要选举？这些地方的选举策略又有哪些？

失效副本是指什么？有那些应对措施？

多副本下，各个副本中的HW和LEO的演变过程

为什么Kafka不支持读写分离？

Kafka在可靠性方面做了哪些改进？（HW, LeaderEpoch）

Kafka中怎么实现死信队列和重试队列？

Kafka中的延迟队列怎么实现（这题被问的比事务那题还要多！！！听说你会Kafka，那你说说延迟队列怎么实现？）

Kafka中怎么做消息审计？

Kafka中怎么做消息轨迹？

Kafka中有那些配置参数比较有意思？聊一聊你的看法

Kafka中有那些命名比较有意思？聊一聊你的看法

Kafka有哪些指标需要着重关注？

怎么计算Lag？(注意read_uncommitted和read_committed状态下的不同)

Kafka的那些设计让它有如此高的性能？

Kafka有什么优缺点？

还用过什么同质类的其它产品，与Kafka相比有什么优缺点？

为什么选择Kafka?

在使用Kafka的过程中遇到过什么困难？怎么解决的？

怎么样才能确保Kafka极大程度上的可靠性？

聊一聊你对Kafka生态的理解

## 参考链接
<https://mp.weixin.qq.com/s/I-YsRKcjcBv4eOop-jjO0A>
<https://juejin.im/post/5b41fe36e51d45191252e79e>