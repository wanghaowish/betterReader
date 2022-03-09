# kafaka
![](https://img2018.cnblogs.com/blog/1515111/201911/1515111-20191128124853636-833212813.png "kafka")
ApacheKafka®是一个分布式流媒体平台。它是一个分布式的，支持多分区、多副本，基于 Zookeeper 的分布式消息流平台，它同时也是一款开源的基于发布订阅模式的消息引擎系统。

## kafka基本术语

**消息**：Kafka 中的数据单元被称为`消息`，也被称为记录，可以把它看作数据库表中某一行的记录。

**批次**：为了提高效率， 消息会`分批次`写入 Kafka，批次就代指的是一组消息。

**主题**：消息的种类称为 `主题`（Topic）,可以说一个主题代表了一类消息。相当于是对消息进行分类。主题就像是数据库中的表。

**分区**：主题可以被分为若干个分区（partition），同一个主题中的分区可以不在一个机器上，有可能会部署在多个机器上，由此来实现 kafka 的`伸缩性`，单一主题中的分区有序，但是无法保证主题中所有的分区有序

![img](https://raw.githubusercontent.com/wanghaowish/picGo/main/img/202202171552711.png)

**生产者**： 向主题发布消息的客户端应用程序称为`生产者`（Producer），生产者用于持续不断的向某个主题发送消息。

**消费者**：订阅主题消息的客户端程序称为`消费者`（Consumer），消费者用于处理生产者产生的消息。

**消费者群组**：生产者与消费者的关系就如同餐厅中的厨师和顾客之间的关系一样，一个厨师对应多个顾客，也就是一个生产者对应多个消费者，`消费者群组`（Consumer Group）指的就是由一个或多个消费者组成的群体。

![img](https://raw.githubusercontent.com/wanghaowish/picGo/main/img/202202171552724.png)

**偏移量**：`偏移量`（Consumer Offset）是一种元数据，它是一个不断递增的整数值，用来记录消费者发生重平衡时的位置，以便用来恢复数据。

**broker**: 一个独立的 Kafka 服务器就被称为 `broker`，broker 接收来自生产者的消息，为消息设置偏移量，并提交消息到磁盘保存。

**broker 集群**：broker 是`集群` 的组成部分，broker 集群由一个或多个 broker 组成，每个集群都有一个 broker 同时充当了`集群控制器`的角色（自动从集群的活跃成员中选举出来）。

**副本**：Kafka 中消息的备份又叫做 `副本`（Replica），副本的数量是可以配置的，Kafka 定义了两类副本：领导者副本（Leader Replica） 和 追随者副本（Follower Replica），前者对外提供服务，后者只是被动跟随。

**重平衡**：Rebalance。消费者组内某个消费者实例挂掉后，其他消费者实例自动重新分配订阅主题分区的过程。Rebalance 是 Kafka 消费者端实现高可用的重要手段。

## Kafka 的特性（设计原则）

* `高吞吐、低延迟`：kakfa 最大的特点就是收发消息非常快，kafka 每秒可以处理几十万条消息，它的最低延迟只有几毫秒。

* `高伸缩性`： 每个主题(topic) 包含多个分区(partition)，主题中的分区可以分布在不同的主机(broker)中。

* `持久性、可靠性`： Kafka 能够允许数据的持久化存储，消息被持久化到磁盘，并支持数据备份防止数据丢失，Kafka 底层的数据存储是基于 Zookeeper 存储的，Zookeeper 我们知道它的数据能够持久存储。

* `容错性`： 允许集群中的节点失败，某个节点宕机，Kafka 集群能够正常工作

* `高并发`： 支持数千个客户端同时读写 
## kafka系统架构

![img](https://raw.githubusercontent.com/wanghaowish/picGo/main/img/202202181357369.png)

一个典型的 Kafka 集群中包含若干Producer（可以是web前端产生的Page View，或者是服务器日志，系统CPU、Memory等），若干broker（Kafka支持水平扩展，一般broker数量越多，集群吞吐率越高），若干Consumer Group，以及一个Zookeeper集群。Kafka通过Zookeeper管理集群配置，选举leader，以及在Consumer Group发生变化时进行rebalance。Producer使用push模式将消息发布到broker，Consumer使用pull模式从broker订阅并消费消息。

## kafka 为何如此之快

Kafka 实现了`零拷贝`原理来快速移动数据，避免了内核之间的切换。Kafka 可以将数据记录分批发送，从生产者到文件系统（Kafka 主题日志）到消费者，可以端到端的查看这些批次的数据。

批处理能够进行更有效的数据压缩并减少 I/O 延迟，Kafka 采取顺序写入磁盘的方式，避免了随机磁盘寻址的浪费

## kafka重要参数配置

### broker 端配置

- broker.id

每个 kafka broker 都有一个唯一的标识来表示，这个唯一的标识符即是 broker.id，它的默认值是 0。这个值在 kafka 集群中必须是唯一的，这个值可以任意设定，

- port

如果使用配置样本来启动 kafka，它会监听 9092 端口。修改 port 配置参数可以把它设置成任意的端口。要注意，如果使用 1024 以下的端口，需要使用 root 权限启动 kakfa。

- zookeeper.connect

用于保存 broker 元数据的 Zookeeper 地址是通过 zookeeper.connect 来指定的。比如我可以这么指定 `localhost:2181` 表示这个 Zookeeper 是运行在本地 2181 端口上的。我们也可以通过 比如我们可以通过 `zk1:2181,zk2:2181,zk3:2181` 来指定 zookeeper.connect 的多个参数值。该配置参数是用冒号分割的一组 `hostname:port/path` 列表，其含义如下

hostname 是 Zookeeper 服务器的机器名或者 ip 地址。

port 是 Zookeeper 客户端的端口号

/path 是可选择的 Zookeeper 路径，Kafka 路径是使用了 `chroot` 环境，如果不指定默认使用跟路径。

> 如果你有两套 Kafka 集群，假设分别叫它们 kafka1 和 kafka2，那么两套集群的`zookeeper.connect`参数可以这样指定：`zk1:2181,zk2:2181,zk3:2181/kafka1`和`zk1:2181,zk2:2181,zk3:2181/kafka2`

- log.dirs

Kafka 把所有的消息都保存到磁盘上，存放这些日志片段的目录是通过 `log.dirs` 来制定的，它是用一组逗号来分割的本地系统路径，log.dirs 是没有默认值的，**你必须手动指定他的默认值**。其实还有一个参数是 `log.dir`，如你所知，这个配置是没有 `s` 的，默认情况下只用配置 log.dirs 就好了，比如你可以通过 `/home/kafka1,/home/kafka2,/home/kafka3` 这样来配置这个参数的值。

- num.recovery.threads.per.data.dir

对于如下3种情况，Kafka 会使用`可配置的线程池`来处理日志片段。

服务器正常启动，用于打开每个分区的日志片段；

服务器崩溃后重启，用于检查和截断每个分区的日志片段；

服务器正常关闭，用于关闭日志片段。

默认情况下，每个日志目录只使用一个线程。因为这些线程只是在服务器启动和关闭时会用到，所以完全可以设置大量的线程来达到井行操作的目的。特别是对于包含大量分区的服务器来说，一旦发生崩愤，在进行恢复时使用井行操作可能会省下数小时的时间。设置此参数时需要注意，所配置的数字对应的是 log.dirs 指定的单个日志目录。也就是说，如果 num.recovery.threads.per.data.dir 被设为 8，并且 log.dir 指定了 3 个路径，那么总共需要 24 个线程。

- auto.create.topics.enable

默认情况下，kafka 会使用三种方式来自动创建主题，下面是三种情况：

当一个生产者开始往主题写入消息时

当一个消费者开始从主题读取消息时

当任意一个客户端向主题发送元数据请求时

`auto.create.topics.enable`参数我建议最好设置成 false，即不允许自动创建 Topic。在我们的线上环境里面有很多名字稀奇古怪的 Topic，我想大概都是因为该参数被设置成了 true 的缘故。

* listeners

  ```shell
  listeners: Listener List - Comma-separated list of URIs we will listen on and the listener names. If the listener name is not a security protocol, listener.security.protocol.map must also be set. Specify hostname as 0.0.0.0 to bind to all interfaces. Leave hostname empty to bind to default interface. Examples of legal listener lists: PLAINTEXT://myhost:9092,SSL://:9091 CLIENT://0.0.0.0:9092,REPLICATION://localhost:9093
  Type: stringDefault: nullValid Values: Importance: highUpdate Mode: per-broker
  # The address the socket server listens on. It will get the value returned from
  # java.net.InetAddress.getCanonicalHostName() if not configured.
  #   FORMAT:
  #     listeners = listener_name://host_name:port
  #   EXAMPLE:
  #     listeners = PLAINTEXT://your.host.name:9092
  ```

  学名叫监听器，其实就是告诉外部连接者要通过什么协议访问指定主机名和端口开放的 `Kafka` 服务。如果侦听协议不是安全协议，则还必须设置 listener.security.protocol.map。监听的协议名称和端口号必须是唯一的。

* advertisied.listeners

  ```shell
  advertised.listeners: Listeners to publish to ZooKeeper for clients to use, if different than the listeners config property. In IaaS environments, this may need to be different from the interface to which the broker binds. If this is not set, the value for listeners will be used. Unlike listeners it is not valid to advertise the 0.0.0.0 meta-address.
  Type: stringDefault: nullValid Values: Importance: highUpdate Mode: per-broker
  # Hostname and port the broker will advertise to producers and consumers. If not set,
  # it uses the value for "listeners" if configured.  Otherwise, it will use the value
  # returned from java.net.InetAddress.getCanonicalHostName().
  ```

  和 `listeners` 相比多了个 `advertised`。`Advertised` 的含义表示宣称的、公布的，就是说这组监听器是 `Broker` 用于对外发布的。

  advertised_listeners 是对外暴露的服务端口，真正建立连接用的是 listeners。
  
  ~

### 主题默认配置

Kafka 为新创建的主题提供了很多默认配置参数，下面就来一起认识一下这些参数

- num.partitions

num.partitions 参数指定了新创建的主题需要包含多少个分区。如果启用了主题自动创建功能（该功能是默认启用的），主题分区的个数就是该参数指定的值。该参数的默认值是 1。要注意，我们可以增加主题分区的个数，但不能减少分区的个数。

- default.replication.factor

这个参数比较简单，它表示 kafka保存消息的副本数，如果一个副本失效了，另一个还可以继续提供服务default.replication.factor 的默认值为1，这个参数在你启用了主题自动创建功能后有效。

- log.retention.ms

Kafka 通常根据时间来决定数据可以保留多久。默认使用 log.retention.hours 参数来配置时间，默认是 168 个小时，也就是一周。除此之外，还有两个参数 log.retention.minutes 和 log.retentiion.ms 。这三个参数作用是一样的，都是决定消息多久以后被删除，推荐使用 log.retention.ms。

- log.retention.bytes

另一种保留消息的方式是判断消息是否过期。它的值通过参数 `log.retention.bytes` 来指定，作用在每一个分区上。也就是说，如果有一个包含 8 个分区的主题，并且 log.retention.bytes 被设置为 1GB，那么这个主题最多可以保留 8GB 数据。所以，当主题的分区个数增加时，整个主题可以保留的数据也随之增加。

- log.segment.bytes

上述的日志都是作用在日志片段上，而不是作用在单个消息上。当消息到达 broker 时，它们被追加到分区的当前日志片段上，当日志片段大小到达 log.segment.bytes 指定上限（默认为 1GB）时，当前日志片段就会被关闭，一个新的日志片段被打开。如果一个日志片段被关闭，就开始等待过期。这个参数的值越小，就越会频繁的关闭和分配新文件，从而降低磁盘写入的整体效率。

- log.segment.ms

上面提到日志片段经关闭后需等待过期，那么 `log.segment.ms` 这个参数就是指定日志多长时间被关闭的参数和，log.segment.ms 和 log.retention.bytes 也不存在互斥问题。日志片段会在大小或时间到达上限时被关闭，就看哪个条件先得到满足。

- message.max.bytes

broker 通过设置 `message.max.bytes` 参数来限制单个消息的大小，默认是 1000 000， 也就是 1MB，如果生产者尝试发送的消息超过这个大小，不仅消息不会被接收，还会收到 broker 返回的错误消息。跟其他与字节相关的配置参数一样，该参数指的是压缩后的消息大小，也就是说，只要压缩后的消息小于 mesage.max.bytes，那么消息的实际大小可以大于这个值

这个值对性能有显著的影响。值越大，那么负责处理网络连接和请求的线程就需要花越多的时间来处理这些请求。它还会增加磁盘写入块的大小，从而影响 IO 吞吐量。

- retention.ms

规定了该主题消息被保存的时常，默认是7天，即该主题只能保存7天的消息，一旦设置了这个值，它会覆盖掉 Broker 端的全局参数值。

- retention.bytes

`retention.bytes`：规定了要为该 Topic 预留多大的磁盘空间。和全局参数作用相似，这个值通常在多租户的 Kafka 集群中会有用武之地。当前默认值是 -1，表示可以无限使用磁盘空间。

### JVM 参数配置

JDK 版本一般推荐直接使用 JDK1.8，这个版本也是现在中国大部分程序员的首选版本。JVM参数可以直接修改启动脚本bin/kafka-server-start.sh 中的变量值。

说到 JVM 端设置，就绕不开`堆`这个话题，业界最推崇的一种设置方式就是直接将 JVM 堆大小设置为 6GB，这样会避免很多 Bug 出现。

JVM 端配置的另一个重要参数就是垃圾回收器的设置，也就是平时常说的 `GC` 设置。如果你依然在使用 Java 7，那么可以根据以下法则选择合适的垃圾回收器：

- 如果 Broker 所在机器的 CPU 资源非常充裕，建议使用 CMS 收集器。启用方法是指定`-XX:+UseCurrentMarkSweepGC`。
- 否则，使用吞吐量收集器。开启方法是指定`-XX:+UseParallelGC`。

当然了，如果你已经在使用 Java 8 了，那么就用默认的 G1 收集器就好了。在没有任何调优的情况下，G1 表现得要比 CMS 出色，主要体现在更少的 Full GC，需要调整的参数更少等，所以使用 G1 就好了。

一般 G1 的调整只需要这两个参数即可

- ​	MaxGCPauseMillis

该参数指定每次垃圾回收默认的停顿时间。该值不是固定的，G1可以根据需要使用更长的时间。它的默认值是 200ms，也就是说，每一轮垃圾回收大概需要200 ms 的时间。

- InitiatingHeapOccupancyPercent

该参数指定了 G1 启动新一轮垃圾回收之前可以使用的堆内存百分比，默认值是45，这就表明G1在堆使用率到达45之前不会启用垃圾回收。这个百分比包括新生代和老年代。

## Kafka Producer

![img](https://raw.githubusercontent.com/wanghaowish/picGo/main/img/202202181610219.png)

我们从创建一个`ProducerRecord` 对象开始，ProducerRecord 是 Kafka 中的一个核心类，它代表了一组 Kafka 需要发送的 `key/value` 键值对，它由记录要发送到的主题名称（Topic Name），可选的分区号（Partition Number）以及可选的键值对构成。

在发送 ProducerRecord 时，我们需要将键值对对象由序列化器转换为字节数组，这样它们才能够在网络上传输。然后消息到达了分区器。

如果发送过程中指定了有效的分区号，那么在发送记录时将使用该分区。如果发送过程中未指定分区，则将使用key 的 hash 函数映射指定一个分区。如果发送的过程中既没有分区号也没有，则将以循环的方式分配一个分区。选好分区后，生产者就知道向哪个主题和分区发送数据了。

ProducerRecord 还有关联的时间戳，如果用户没有提供时间戳，那么生产者将会在记录中使用当前的时间作为时间戳。Kafka 最终使用的时间戳取决于 topic 主题配置的时间戳类型。

- 如果将主题配置为使用 `CreateTime`，则生产者记录中的时间戳将由 broker 使用。
- 如果将主题配置为使用`LogAppendTime`，则生产者记录中的时间戳在将消息添加到其日志中时，将由 broker 重写。

然后，这条消息被存放在一个记录批次里，这个批次里的所有消息会被发送到相同的主题和分区上。由一个独立的线程负责把它们发到 Kafka Broker 上。

Kafka Broker 在收到消息时会返回一个响应，如果写入成功，会返回一个 RecordMetaData 对象，**它包含了主题和分区信息，以及记录在分区里的偏移量，上面两种的时间戳类型也会返回给用户**。如果写入失败，会返回一个错误。生产者在收到错误之后会尝试重新发送消息，几次之后如果还是失败的话，就返回错误消息。

### golang构建kafka producer

使用sarama包

```go get github.com/Shopify/sarama```

![Sarama Producer 流程.png](https://raw.githubusercontent.com/wanghaowish/picGo/main/img/202202181705530.png)

```go
func Producer(topic string, limit int) {
	config := sarama.NewConfig()
	// 异步生产者不建议把 Errors 和 Successes 都开启，一般开启 Errors 就行
	// 同步生产者就必须都开启，因为会同步返回发送成功或者失败
	config.Producer.Return.Errors = true   // 设定需要返回错误信息
	config.Producer.Return.Successes = false // 设定需要返回成功信息
	producer, err := sarama.NewAsyncProducer([]string{kafka.HOST}, config)
	if err != nil {
		log.Fatal("NewSyncProducer err:", err)
	}
	defer producer.AsyncClose()
    go func() {
            // [!important] 异步生产者发送后必须把返回值从 Errors 或者 Successes 中读出来 不然会阻塞 sarama 内部处理逻辑 导致只能发出去一条消息
            for {
                select {
                case s := <-producer.Successes():
                    if s != nil {
                        log.Printf("[Producer] Success: key:%v msg:%+v \n", s.Key, s.Value)
                    }
                case e := <-producer.Errors():
                    if e != nil {
                        log.Printf("[Producer] Errors：err:%v msg:%+v \n", e.Msg, e.Err)
                    }
                }
            }
        }()
	// 异步发送
	for i := 0; i < limit; i++ {
		str := strconv.Itoa(int(time.Now().UnixNano()))
		msg := &sarama.ProducerMessage{Topic: topic, Key: nil, Value: sarama.StringEncoder(str)}
		// 异步发送只是写入内存了就返回了，并没有真正发送出去
		// sarama 库中用的是一个 channel 来接收，后台 goroutine 异步从该 channel 中取出消息并真正发送
		producer.Input() <- msg
		atomic.AddInt64(&count, 1)
		if atomic.LoadInt64(&count)%1000 == 0 {
			log.Printf("已发送消息数:%v\n", count)
		}

	}
	log.Printf("发送完毕 总发送消息数:%v\n", limit)
}
```

* NewAsyncProducer() ：创建 一个 producer 对象

* producer.Input() <- msg ：发送消息

* s = <-producer.Successes()，e := <-producer.Errors() ：异步获取成功或失败信息

异步生产者使用`channel`接收（生产成功或失败）的消息，并且也通过`channel`来发送消息，这样做通常是性能最高的。而同步生产者需要阻塞，直到收到了`acks`。但是这也带来了两个问题，一是性能变得更差了，而是可靠性是依靠参数`acks`来保证的。同步生产者可直接调用sendMessage方法，返回值为partition，offset以及error。同步生产者和异步生产者逻辑是一致的，只是在异步生产者基础上封装了一层。

sarama的生产者config参数

* MaxMessageBytes int 这个参数影响了一条消息的最大字节数，默认是1000000。但是注意，这个参数必须要小于broker中的 `message.max.bytes`。

* RequiredAcks RequiredAcks 这个参数影响了消息需要被多少broker写入之后才返回。取值可以是0、1、-1，分别代表了不需要等待broker确认才返回、需要分区的leader确认后才返回、以及需要分区的所有副本确认后返回。

* Partitioner PartitionerConstructor 这个是分区器。`Sarama`默认提供了几种分区器，如果不指定默认使用Hash分区器。

* Retry 这个参数代表了重试的次数，以及重试的时间，主要发生在一些可重试的错误中。

* Flush 用于设置将消息打包发送，简单来讲就是每次发送消息到broker的时候，不是生产一条消息就发送一条消息，而是等消息累积到一定的程度了，再打包发送。所以里面含有两个参数。一个是多少条消息触发打包发送，一个是累计的消息大小到了多少，然后发送。

* MaxOpenRequests int 这个参数代表了允许没有收到acks而可以同时发送的最大`batch`数。

* Idempotent bool 用于幂等生产者，当这一项设置为`true`的时候，生产者将保证生产的消息一定是有序且精确一次的。

当MaxOpenRequests这个参数配置大于1的时候，代表了允许有多个请求发送了还没有收到回应。假设此时的重试次数也设置为了大于1，当同时发送了2个请求，如果第一个请求发送到broker中，broker写入失败了，但是第二个请求写入成功了，那么客户端将重新发送第一个消息的请求，这个时候会造成乱序。消息的有序可以通过MaxOpenRequests设置为1来保证，这个时候每个消息必须收到了acks才能发送下一条，所以一定是有序的，但是不能够保证不重复。而且当MaxOpenRequests设置为1的时候，吞吐量不高。注意，当启动幂等生产者的时候，Retry次数必须要大于0，ack必须为all。

### producer源码分析

#### NewAsyncProducer

`async_producer.go`

```go
// NewAsyncProducer creates a new AsyncProducer using the given broker addresses and configuration.
func NewAsyncProducer(addrs []string, conf *Config) (AsyncProducer, error) {
	client, err := NewClient(addrs, conf)
	if err != nil {
		return nil, err
	}
	return newAsyncProducer(client)
}

func newAsyncProducer(client Client) (AsyncProducer, error) {
	// Check that we are not dealing with a closed Client before processing any other arguments
	if client.Closed() {
		return nil, ErrClosedClient
	}

	txnmgr, err := newTransactionManager(client.Config(), client)
	if err != nil {
		return nil, err
	}

	p := &asyncProducer{
		client:     client,
		conf:       client.Config(),
		errors:     make(chan *ProducerError),
		input:      make(chan *ProducerMessage),
		successes:  make(chan *ProducerMessage),
		retries:    make(chan *ProducerMessage),
		brokers:    make(map[*Broker]*brokerProducer),
		brokerRefs: make(map[*brokerProducer]int),
		txnmgr:     txnmgr,
	}

	// launch our singleton dispatchers
	go withRecover(p.dispatcher)
	go withRecover(p.retryHandler)

	return p, nil
}
```

`utils.go`

```go
func withRecover(fn func()) {
	defer func() {
		handler := PanicHandler
		if handler != nil {
			if err := recover(); err != nil {
				handler(err)
			}
		}
	}()

	fn()
}
```



可以看到在 `newAsyncProducer` 最后开启了两个 goroutine，一个为 `dispatcher`，一个为 `retryHandler `。

#### dispatcher

主要根据 `topic` 将消息分发到对应的 channel。

```go
// singleton
// dispatches messages by topic
func (p *asyncProducer) dispatcher() {
	handlers := make(map[string]chan<- *ProducerMessage)
	shuttingDown := false

	for msg := range p.input {
		if msg == nil {
			Logger.Println("Something tried to send a nil message, it was ignored.")
			continue
		}

		if msg.flags&shutdown != 0 {
			shuttingDown = true
			p.inFlight.Done()
			continue
		} else if msg.retries == 0 {
			if shuttingDown {
				// we can't just call returnError here because that decrements the wait group,
				// which hasn't been incremented yet for this message, and shouldn't be
				pErr := &ProducerError{Msg: msg, Err: ErrShuttingDown}
				if p.conf.Producer.Return.Errors {
					p.errors <- pErr
				} else {
					Logger.Println(pErr)
				}
				continue
			}
			p.inFlight.Add(1)
		}
		//拦截器逻辑
		for _, interceptor := range p.conf.Producer.Interceptors {
			msg.safelyApplyInterceptor(interceptor)
		}

		version := 1
		if p.conf.Version.IsAtLeast(V0_11_0_0) {
			version = 2
		} else if msg.Headers != nil {
			p.returnError(msg, ConfigurationError("Producing headers requires Kafka at least v0.11"))
			continue
		}
		if msg.byteSize(version) > p.conf.Producer.MaxMessageBytes {
			p.returnError(msg, ErrMessageSizeTooLarge)
			continue
		}
		// 找到这个Topic对应的Handler
		handler := handlers[msg.Topic]
		if handler == nil {
			handler = p.newTopicProducer(msg.Topic)
			handlers[msg.Topic] = handler
		}
		// 把消息写进这个Handler中
		handler <- msg
	}

	for _, handler := range handlers {
		close(handler)
	}
}
```

具体逻辑：从 `p.input` 中取出消息并写入到 `handler` 中，如果 `topic` 对应的 `handler` 不存在，则调用 `newTopicProducer()` 创建。这里的 handler 是一个 buffered channel。

然后再来看下`handler = p.newTopicProducer(msg.Topic)`这一行的代码。

```go
func (p *asyncProducer) newTopicProducer(topic string) chan<- *ProducerMessage {
	input := make(chan *ProducerMessage, p.conf.ChannelBufferSize)
	tp := &topicProducer{
		parent:      p,
		topic:       topic,
		input:       input,
		breaker:     breaker.New(3, 1, 10*time.Second),
		handlers:    make(map[int32]chan<- *ProducerMessage),
		partitioner: p.conf.Producer.Partitioner(topic),
	}
	go withRecover(tp.dispatch)
	return input
}
```

在这里创建了一个缓冲大小为`ChannelBufferSize`的channel，用于存放发送到这个主题的消息，然后创建了一个 `topicProducer`。一个需要注意的是`newTopicProducer` 的这种写法，内部创建一个 chan 返回到外层，然后通过在内部新开一个 goroutine 来处理该 chan 里的消息，这种写法在后面还会遇到好几次。在这个时候可以认为消息已经交付给各个 topic 对应的 topicProducer 了。

```topicDispatch```

`newTopicProducer`的最后一行`go withRecover(tp.dispatch)`又启动了一个 goroutine 用于处理消息。也就是说，到了这一步，对于每一个Topic，都有一个协程来处理消息。

```go
func (tp *topicProducer) dispatch() {
	for msg := range tp.input {
		if msg.retries == 0 {
			if err := tp.partitionMessage(msg); err != nil {
				tp.parent.returnError(msg, err)
				continue
			}
		}

		handler := tp.handlers[msg.Partition]
		if handler == nil {
			handler = tp.parent.newPartitionProducer(msg.Topic, msg.Partition)
			tp.handlers[msg.Partition] = handler
		}

		handler <- msg
	}

	for _, handler := range tp.handlers {
		close(handler)
	}
}
```

可以看到又是同样的套路：

- 1）找到这条消息所在的分区对应的 channel，然后把消息丢进去

- 2）如果不存在则新建 chan

#### PartitionDispatch

新建的 chan 是通过 `newPartitionProducer` 返回的，和之前的`newTopicProducer`又是同样的套路,点进去看一下：

```go
func (p *asyncProducer) newPartitionProducer(topic string, partition int32) chan<- *ProducerMessage {
	input := make(chan *ProducerMessage, p.conf.ChannelBufferSize)
	pp := &partitionProducer{
		parent:    p,
		topic:     topic,
		partition: partition,
		input:     input,

		breaker:    breaker.New(3, 1, 10*time.Second),
		retryState: make([]partitionRetryState, p.conf.Producer.Retry.Max+1),
	}
	go withRecover(pp.dispatch)
	return input
}
```

`TopicProducer`是按照 `Topic` 进行分发，这里的 `PartitionProducer` 则是按照 `partition` 进行分发。到这里可以认为消息已经交付给对应 topic 下的对应 partition 了，每个 partition 都会有一个 goroutine 来处理分发给自己的消息。

```go
func (pp *partitionProducer) dispatch() {
	// try to prefetch the leader; if this doesn't work, we'll do a proper call to `updateLeader`
	// on the first message
	pp.leader, _ = pp.parent.client.Leader(pp.topic, pp.partition)
	if pp.leader != nil {
		pp.brokerProducer = pp.parent.getBrokerProducer(pp.leader)
		pp.parent.inFlight.Add(1) // we're generating a syn message; track it so we don't shut down while it's still inflight
		pp.brokerProducer.input <- &ProducerMessage{Topic: pp.topic, Partition: pp.partition, flags: syn}
	}

	defer func() {
		if pp.brokerProducer != nil {
			pp.parent.unrefBrokerProducer(pp.leader, pp.brokerProducer)
		}
	}()

	for msg := range pp.input {
		if pp.brokerProducer != nil && pp.brokerProducer.abandoned != nil {
			select {
			case <-pp.brokerProducer.abandoned:
				// a message on the abandoned channel means that our current broker selection is out of date
				Logger.Printf("producer/leader/%s/%d abandoning broker %d\n", pp.topic, pp.partition, pp.leader.ID())
				pp.parent.unrefBrokerProducer(pp.leader, pp.brokerProducer)
				pp.brokerProducer = nil
				time.Sleep(pp.parent.conf.Producer.Retry.Backoff)
			default:
				// producer connection is still open.
			}
		}

		if msg.retries > pp.highWatermark {
			// a new, higher, retry level; handle it and then back off
			pp.newHighWatermark(msg.retries)
			pp.backoff(msg.retries)
		} else if pp.highWatermark > 0 {
			// we are retrying something (else highWatermark would be 0) but this message is not a *new* retry level
			if msg.retries < pp.highWatermark {
				// in fact this message is not even the current retry level, so buffer it for now (unless it's a just a fin)
				if msg.flags&fin == fin {
					pp.retryState[msg.retries].expectChaser = false
					pp.parent.inFlight.Done() // this fin is now handled and will be garbage collected
				} else {
					pp.retryState[msg.retries].buf = append(pp.retryState[msg.retries].buf, msg)
				}
				continue
			} else if msg.flags&fin == fin {
				// this message is of the current retry level (msg.retries == highWatermark) and the fin flag is set,
				// meaning this retry level is done and we can go down (at least) one level and flush that
				pp.retryState[pp.highWatermark].expectChaser = false
				pp.flushRetryBuffers()
				pp.parent.inFlight.Done() // this fin is now handled and will be garbage collected
				continue
			}
		}

		// if we made it this far then the current msg contains real data, and can be sent to the next goroutine
		// without breaking any of our ordering guarantees

		if pp.brokerProducer == nil {
			if err := pp.updateLeader(); err != nil {
				pp.parent.returnError(msg, err)
				pp.backoff(msg.retries)
				continue
			}
			Logger.Printf("producer/leader/%s/%d selected broker %d\n", pp.topic, pp.partition, pp.leader.ID())
		}

		// Now that we know we have a broker to actually try and send this message to, generate the sequence
		// number for it.
		// All messages being retried (sent or not) have already had their retry count updated
		// Also, ignore "special" syn/fin messages used to sync the brokerProducer and the topicProducer.
		if pp.parent.conf.Producer.Idempotent && msg.retries == 0 && msg.flags == 0 {
			msg.sequenceNumber, msg.producerEpoch = pp.parent.txnmgr.getAndIncrementSequenceNumber(msg.Topic, msg.Partition)
			msg.hasSequence = true
		}

		pp.brokerProducer.input <- msg
	}
}
```

##### BrokerProducer

真正的逻辑肯定在`pp.parent.getBrokerProducer(pp.leader)` 这个方法里面。我们继续跟进`pp.parent.getBrokerProducer(pp.leader)`这行代码里面的内容。其实就是找到`asyncProducer`中的`brokerProducer`，如果不存在，则创建一个。

```go
func (p *asyncProducer) getBrokerProducer(broker *Broker) *brokerProducer {
	p.brokerLock.Lock()
	defer p.brokerLock.Unlock()

	bp := p.brokers[broker]

	if bp == nil {
		bp = p.newBrokerProducer(broker)
		p.brokers[broker] = bp
		p.brokerRefs[bp] = 0
	}

	p.brokerRefs[bp]++

	return bp
}
```

又调用了`newBrokerProducer()`，继续追踪下去：

```go
// one per broker; also constructs an associated flusher
func (p *asyncProducer) newBrokerProducer(broker *Broker) *brokerProducer {
	var (
		input     = make(chan *ProducerMessage)
		bridge    = make(chan *produceSet)
		pending   = make(chan *brokerProducerResponse)
		responses = make(chan *brokerProducerResponse)
	)

	bp := &brokerProducer{
		parent:         p,
		broker:         broker,
		input:          input,
		output:         bridge,
		responses:      responses,
		buffer:         newProduceSet(p),
		currentRetries: make(map[string]map[int32]error),
	}
	go withRecover(bp.run)

	// minimal bridge to make the network response `select`able
	go withRecover(func() {
		// Use a wait group to know if we still have in flight requests
		var wg sync.WaitGroup

		for set := range bridge {
			request := set.buildRequest()

			// Count the in flight requests to know when we can close the pending channel safely
			wg.Add(1)
			// Capture the current set to forward in the callback
			sendResponse := func(set *produceSet) ProduceCallback {
				return func(response *ProduceResponse, err error) {
					// Forward the response to make sure we do not block the responseReceiver
					pending <- &brokerProducerResponse{
						set: set,
						err: err,
						res: response,
					}
					wg.Done()
				}
			}(set)

			// Use AsyncProduce vs Produce to not block waiting for the response
			// so that we can pipeline multiple produce requests and achieve higher throughput, see:
			// https://kafka.apache.org/protocol#protocol_network
			err := broker.AsyncProduce(request, sendResponse)
			if err != nil {
				// Request failed to be sent
				sendResponse(nil, err)
				continue
			}
			// Callback is not called when using NoResponse
			if p.conf.Producer.RequiredAcks == NoResponse {
				// Provide the expected nil response
				sendResponse(nil, nil)
			}
		}
		// Wait for all in flight requests to close the pending channel safely
		wg.Wait()
		close(pending)
	})

	// In order to avoid a deadlock when closing the broker on network or malformed response error
	// we use an intermediate channel to buffer and send pending responses in order
	// This is because the AsyncProduce callback inside the bridge is invoked from the broker
	// responseReceiver goroutine and closing the broker requires such goroutine to be finished
	go withRecover(func() {
		buf := queue.New()
		for {
			if buf.Length() == 0 {
				res, ok := <-pending
				if !ok {
					// We are done forwarding the last pending response
					close(responses)
					return
				}
				buf.Add(res)
			}
			// Send the head pending response or buffer another one
			// so that we never block the callback
			headRes := buf.Peek().(*brokerProducerResponse)
			select {
			case res, ok := <-pending:
				if !ok {
					continue
				}
				buf.Add(res)
				continue
			case responses <- headRes:
				buf.Remove()
				continue
			}
		}
	})

	if p.conf.Producer.Retry.Max <= 0 {
		bp.abandoned = make(chan struct{})
	}

	return bp
}
```

这里又启动了两个 goroutine，一个为 run，一个是匿名函数姑且称为 bridge。

看这个方法中启动的第二个协程，可以推测`bridge`这个`channel`收到消息后，会把收到的消息打包成一个request，然后调用`AsyncProduce()`方法。

并且，将返回的结果的指针地址，写进response中。

然后构造好brokerProducerResponse，并且写入responses中。

##### buildRequest

buildRequest 方法主要是构建一个标准的 Kafka Request 消息。

根据不同版本、是否配置压缩信息做了额外处理：

```go

func (ps *produceSet) buildRequest() *ProduceRequest {
	req := &ProduceRequest{
		RequiredAcks: ps.parent.conf.Producer.RequiredAcks,
		Timeout:      int32(ps.parent.conf.Producer.Timeout / time.Millisecond),
	}
	if ps.parent.conf.Version.IsAtLeast(V0_10_0_0) {
		req.Version = 2
	}
	if ps.parent.conf.Version.IsAtLeast(V0_11_0_0) {
		req.Version = 3
	}

	if ps.parent.conf.Producer.Compression == CompressionZSTD && ps.parent.conf.Version.IsAtLeast(V2_1_0_0) {
		req.Version = 7
	}

	for topic, partitionSets := range ps.msgs {
		for partition, set := range partitionSets {
			if req.Version >= 3 {
				// If the API version we're hitting is 3 or greater, we need to calculate
				// offsets for each record in the batch relative to FirstOffset.
				// Additionally, we must set LastOffsetDelta to the value of the last offset
				// in the batch. Since the OffsetDelta of the first record is 0, we know that the
				// final record of any batch will have an offset of (# of records in batch) - 1.
				// (See https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-Messagesets
				//  under the RecordBatch section for details.)
				rb := set.recordsToSend.RecordBatch
				if len(rb.Records) > 0 {
					rb.LastOffsetDelta = int32(len(rb.Records) - 1)
					for i, record := range rb.Records {
						record.OffsetDelta = int64(i)
					}
				}
				req.AddBatch(topic, partition, rb)
				continue
			}
			if ps.parent.conf.Producer.Compression == CompressionNone {
				req.AddSet(topic, partition, set.recordsToSend.MsgSet)
			} else {
				// When compression is enabled, the entire set for each partition is compressed
				// and sent as the payload of a single fake "message" with the appropriate codec
				// set and no key. When the server sees a message with a compression codec, it
				// decompresses the payload and treats the result as its message set.

				if ps.parent.conf.Version.IsAtLeast(V0_10_0_0) {
					// If our version is 0.10 or later, assign relative offsets
					// to the inner messages. This lets the broker avoid
					// recompressing the message set.
					// (See https://cwiki.apache.org/confluence/display/KAFKA/KIP-31+-+Move+to+relative+offsets+in+compressed+message+sets
					// for details on relative offsets.)
					for i, msg := range set.recordsToSend.MsgSet.Messages {
						msg.Offset = int64(i)
					}
				}
				payload, err := encode(set.recordsToSend.MsgSet, ps.parent.conf.MetricRegistry)
				if err != nil {
					Logger.Println(err) // if this happens, it's basically our fault.
					panic(err)
				}
				compMsg := &Message{
					Codec:            ps.parent.conf.Producer.Compression,
					CompressionLevel: ps.parent.conf.Producer.CompressionLevel,
					Key:              nil,
					Value:            payload,
					Set:              set.recordsToSend.MsgSet, // Provide the underlying message set for accurate metrics
				}
				if ps.parent.conf.Version.IsAtLeast(V0_10_0_0) {
					compMsg.Version = 1
					compMsg.Timestamp = set.recordsToSend.MsgSet.Messages[0].Msg.Timestamp
				}
				req.AddMessage(topic, partition, compMsg)
			}
		}
	}

	return req
}
```

首先是构建一个 req 对象，然后遍历 ps.msg 中的消息，根据 topic 和 partition 分别写入到 req 中。

回到newBrokerProducer()方法中，打包好request后，会调用`AsyncProduce()`方法。

##### AsyncProduce

```go
// AsyncProduce sends a produce request and eventually call the provided callback
// with a produce response or an error.
//
// Waiting for the response is generally not blocking on the contrary to using Produce.
// If the maximum number of in flight request configured is reached then
// the request will be blocked till a previous response is received.
//
// When configured with RequiredAcks == NoResponse, the callback will not be invoked.
// If an error is returned because the request could not be sent then the callback
// will not be invoked either.
//
// Make sure not to Close the broker in the callback as it will lead to a deadlock.
func (b *Broker) AsyncProduce(request *ProduceRequest, cb ProduceCallback) error {
	needAcks := request.RequiredAcks != NoResponse
	// Use a nil promise when no acks is required
	var promise *responsePromise

	if needAcks {
		// Create ProduceResponse early to provide the header version
		res := new(ProduceResponse)
		promise = &responsePromise{
			headerVersion: res.headerVersion(),
			// Packets will be converted to a ProduceResponse in the responseReceiver goroutine
			handler: func(packets []byte, err error) {
				if err != nil {
					// Failed request
					cb(nil, err)
					return
				}

				if err := versionedDecode(packets, res, request.version()); err != nil {
					// Malformed response
					cb(nil, err)
					return
				}

				// Wellformed response
				b.updateThrottleMetric(res.ThrottleTime)
				cb(res, nil)
			},
		}
	}

	return b.sendWithPromise(request, promise)
}
```

##### sendWithPromise

```go
func (b *Broker) sendWithPromise(rb protocolBody, promise *responsePromise) error {
	b.lock.Lock()
	defer b.lock.Unlock()

	if b.conn == nil {
		if b.connErr != nil {
			return b.connErr
		}
		return ErrNotConnected
	}

	if !b.conf.Version.IsAtLeast(rb.requiredVersion()) {
		return ErrUnsupportedVersion
	}

	req := &request{correlationID: b.correlationID, clientID: b.conf.ClientID, body: rb}
	buf, err := encode(req, b.conf.MetricRegistry)
	if err != nil {
		return err
	}

	requestTime := time.Now()
	// Will be decremented in responseReceiver (except error or request with NoResponse)
	b.addRequestInFlightMetrics(1)
	bytes, err := b.write(buf)
	b.updateOutgoingCommunicationMetrics(bytes)
	if err != nil {
		b.addRequestInFlightMetrics(-1)
		return err
	}
	b.correlationID++

	if promise == nil {
		// Record request latency without the response
		b.updateRequestLatencyAndInFlightMetrics(time.Since(requestTime))
		return nil
	}

	promise.requestTime = requestTime
	promise.correlationID = req.correlationID
	b.responses <- promise

	return nil
}
```

最终通过`bytes, err := b.write(buf)` 发送出去。

#### retryHandler
