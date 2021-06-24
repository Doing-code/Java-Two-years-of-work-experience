---
typora-copy-images-to: 图床-图片
---

## 定义 

> kafka 是一个分布式的基于 发布/订阅模式的消息队列（Message Queue），流媒体。



## 消息队列 

### 传统的消息队列的应用场景

![01.png](https://gitee.com/jallenkwong/LearnKafka/raw/master/image/01.png)

### 使用消息队列的好处 

- 1、解耦 （类似 Spring 的 IOC）
	- 允许你独立的扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束即可。
- 2、可恢复性
	- 系统的一部分组件失效时，不会影响到整个系统。消息队列降低了进程间的耦合度，即使一个处理消息的进程挂掉，加入队列中的消息仍然可以在系统恢复后被处理 。
- 3、缓冲 
	- 有助于控制和优化数据流经过系统的速度。解决生产消息和消费消息的处理速度不一致的情况 。
- 4、灵活性 & 峰值处理能力 （削峰）
	- 在访问量剧增的情况下，应用仍然需要继续发挥作用。但是这样的突发流量并不常见。如果为以能处理这类峰值访问为标准来投入资源随时待命无疑是巨大的浪费。而使用消息队列能够使关键组件顶住突发的访问压力，不会因为突发的超负荷的请求而完全崩溃 。
- 5、异步通信
	- 很多时候，用户不想也不需要立即处理消息。消息队列提供了异步处理机制，允许用户把一个消息放入队列，但不立即处理它。然后在需要的时候再去处理它们。



## 消费模式

> 消息队列的两种模式

### 点对点模式

> 一对一，消费者主动拉取数据，**消息消费后消息清除**

消息生产者生产消息发送到 Queue 当中，然后消息消费者从 Queue 中取出并消费消息。消息被消费后 Queue 中不再有存储消费过的消息，所以消息消费者不可能消费到已经被消费过的消息。Queue 支持存在多个消费者，但是对一个消息而言，只会有一个消费者可以消费。

![02.png](https://gitee.com/jallenkwong/LearnKafka/raw/master/image/02.png)

### 发布 / 订阅模式

> 一对多 ，**消费者消费数据之后不会清除消息** 

消息生产者（发布）将消息发布到 topic（主题） 中，同时有多个消息消费者（订阅）消费该消息。和点对点方式不同。发布到 topic 的消息会被所有订阅者消费 。

![03.png](https://gitee.com/jallenkwong/LearnKafka/raw/master/image/03.png)



## 基础架构

![04.png](https://gitee.com/jallenkwong/LearnKafka/raw/master/image/04.png)

> 名词术语定义解释 ：

- 1、`Producer` ：消息生产者，就是向 `kafka broker` 发送消息的客户端 。
- 2、`Cunsumer` ：消息消费者，就是向 `kafka broker` 取出消息的客户端 。
- 3、`Consumer Group` ：消费者组，有多个 consumer 组成。消费者组内每个消费者负责消费不同分区的数据。一个分区只能由一个组内消费者消费。消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者 。
- 4、`Broker `：缓存代理，Kafka 集群中的一台或多台服务器统称broker，一台 kafka 服务器就是一个 broker，一个集群由多个 broker 组成 。一个 broker 可以容纳多个 topic（主题）。
- 5、`Topic` ：主题。可以理解为一个队列，生产者和消费者面向的都是一个 topic 。
- 6、`Partition` ：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker （即服务器）上。**一个 topic 可以分为多个 partition ，每一个 partition 是一个有序的队列 。**
- 7、`Replica` ：副本（Replication），为保证集群中的某个节点发生故障时，该节点上的 partition 数据不会丢失，并且 kafka 仍然能够继续工作。kafka 提供了副本机制，**一个 topic 的每一个分区都有若干个副本，一个 leader 和 若干个 follower 。**
- 8、`Leader` ：每个分区多个副本的 "主" ，生产发送数据的对象，以及消费者消费数据的对象都是 leader 。
- 9、`Follower` ：每个分区多个副本的 "从"，实时从 leader 中同步数据，保持和 leader 数据的同步，一致性。leader 发生故障时，某个 Follower 会成为新的 leader 。？？？**是不是有点类似 redis 主从节点？？主节点挂了，从节点之中选一个做主节点。**







## kafka 实操

> 本次安装学习在Windows操作系统进行。（Linux版本的差别不大，运行脚本文件后缀从`bat`改为`sh`，配置路径改用Unix风格的）

### Download the code

> [下载代码](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fkafka.apache.org%2Fdownloads)并解压
>
> 下载[kafka 0.11.0.0](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Farchive.apache.org%2Fdist%2Fkafka%2F0.11.0.0%2Fkafka_2.11-0.11.0.0.tgz)版本，解压到`D:\Kafka\`路径下，Kafka主目录文件为`C:\Kafka\kafka_2.11-0.11.0.0`（**下文用KAFKA_HOME表示**）。



### Start the server

> Kafka 用到 ZooKeeper 功能，所以要预先运行ZooKeeper。

- 首先、修改 `%KAFKA_HOME%\conf\zookeeper.properties` 中的 `dataDir=/tmp/zookeeper `，改为 `dataDir=D:\\Kafka\\data\\zookeeper` 。
- 创建目录 `D:\\Kafka\\data\\zookeeper`。
- 启动 cmd ，工作目录切换到 `%KAFKA_HOME%` 。执行如下命令 :

```bash
start bin\windows\zookeeper-server-start.bat config\zookeeper.properties
```

- 修改 `%KAFKA_HOME%\conf\server.properties` 中的 `log.dirs=/tmp/kafka-logs`，改为 `log.dirs=D:\\Kafka\\data\\kafka-logs`。
- 创建目录  `D:\\Kafka\\data\\kafka-logs`。 盘符可根据自己心意去切换将其放在哪个位置。
- 另启动 cmd ，工作目录切换到 `%KAFKA_HOME%` ，执行命令行：

```bash
start bin\windows\kafka-server-start.bat config\server.properties
```





**==TODO==** 会出现的问题 ：通过 `kafka-server-stop.bat` 或右上角关闭按钮来关闭Kafka服务后，马上下次再启动Kafka，抛出异常，说某文件被占用，需清空 `log.dirs` 目录下文件，才能重启Kafka。

> [2020-07-21 21:43:26,755] ERROR There was an error in one of the threads during logs loading: java.nio.file.FileSystemException: D:\Kafka\data\kafka-logs-0\my-replicated-topic-0\00000000000000000000.timeindex: 另一个程序正在使用此文件，进程无法访问。 (kafka.log.LogManager) ...



`注意`  : 当你成功启动kafka，然后在对应的命令行窗口用`Ctrl + C`结束Kakfa，下次不用清理kafka日志，也能正常启动。



### Create a topic

- 用单一 `partition` 和单一 `replica` 创建一个名为 `test` 的 topic :

```bash
bin\windows\kafka-topics.bat --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```

- 查看已创建的 topic ，也就是刚才创建名为 `test` 的topic ：

```bash
bin\windows\kafka-topics.bat --list --zookeeper localhost:2181
```

也可以 配置你的broker去自动创建未曾发布过的topic，代替手动创建topic







### Send some messages

运行 producer，然后自定义几行文本。发至服务器 ：

```bash
bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic test
>hello, kafka.
>what a nice day!
>to be or not to be. that' s a question.
```

请勿关闭窗口，下面步骤需要用到 。



### Start a consumer

运行 consumer ，会将 上一步输入的文本，标准输出 。

```bash
bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test --from-beginning
hello, kafka.
what a nice day!
to be or not to be. that' s a question.
```

若你另启cmd，执行命令行`bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test --from-beginning`来运行consumer，然后在第四步中 producer 窗口输入一行句子，如`I must admit, I can't help but feel a twinge of envy.`，两个consumer也会同时输出`I must admit, I can't help but feel a twinge of envy.`





### Setting up a multi-broker cluster

> 设置多代理群集

- 首先，在 `%KAFKA%\config\server.properties` 的基础上创建两个副本 `server-1.properties` 和 `server-2.properties` 。

```bash
copy config\server.properties config\server-1.properties
copy config\server.properties config\server-2.properties
```

- 打开副本，编辑如下属性 。

```bash
#config/server-1.properties:
broker.id=1
listeners=PLAINTEXT://127.0.0.1:9093
log.dir=D:\\Kafka\\data\\kafka-logs-1
 
#config/server-2.properties:
broker.id=2
listeners=PLAINTEXT://127.0.0.1:9094
log.dir=D:\\Kafka\\data\\kafka-logs-2
```

`注意` ：这个 `broker.id` 属性是集群中每个节点的唯一永久的名词 。所以不可用重复 。

我们必须重写 端口 和 日志记录，因为我们在本地运行是在同一台及其上运行，并且我们希望阻止 `brokers` 试图在同一个端口上注册 或 覆盖彼此的数据 。

- 重启启动两个新节点 。已存在 zookeeper 和 一个单节点 。

```bash
start bin\windows\kafka-server-start.bat config\server-1.properties
start bin\windows\kafka-server-start.bat config\server-2.properties
```

- 创建一个 replication-factor 为3 的 topic ：

```bash
bin\windows\kafka-topics.bat --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```

> `replication-factor` 用来设置主题的副本数。每个主题可以有多个副本，副本位于集群中不同的broker上，也就是说副本的数量不能超过broker的数量，否则创建主题时会失败。

- 此时，我们就有了一个集群，但是我们想知道哪个 `broker` 在做什么呢？ 运行命令 `describe topics` : 

```bash
bin\windows\kafka-topics.bat --describe --zookeeper localhost:2181 --topic my-replicated-topic

Topic:my-replicated-topic       PartitionCount:1        ReplicationFactor:3
Configs:
		Topic: my-replicated-topic      Partition: 0    Leader: 0       
		Replicas: 0,1,2        Isr: 0,1,2
```

> 输出的说明 ：第一行给出所有 Partition 的摘要 。每一行提供一个 Partition 的信息。因为这个 Topic 只有一个 Partition，所以只显示了一行 。

- "leader" 是负责给定 Partition 的所有读写的节点 。每个节点都可能成为 Partition 随机选择的 leader 。
- "replicas" 是复制此 Partition 日志的节点列表 。无论它们是 leader 还是当前处于活动状态 。
- "isr" 是一组 "in-sync" replicas。这是 replicas 列表的一个子集，它当前处于存活状态，并补充 leader 。



### server.properties 配置说明

```properties
#broker 的全局唯一编号，不能重复
broker.id=0
#删除 topic 功能使能
delete.topic.enable=true
#处理网络请求的线程数量
num.network.threads=3
#用来处理磁盘 IO 的现成数量
num.io.threads=8
#发送套接字的缓冲区大小
socket.send.buffer.bytes=102400
#接收套接字的缓冲区大小
socket.receive.buffer.bytes=102400
#请求套接字的缓冲区大小
socket.request.max.bytes=104857600
#kafka 运行日志存放的路径
log.dirs=/opt/module/kafka/logs
#topic 在当前 broker 上的分区个数
num.partitions=1
#用来恢复和清理 data 下数据的线程数量
num.recovery.threads.per.data.dir=1
#segment 文件保留的最长时间，超时将被删除
log.retention.hours=168
#配置连接 Zookeeper 集群地址
zookeeper.connect=hadoop102:2181,hadoop103:2181,hadoop104:2181
```





## 命令操作符-Topic 增删查

> `topic` ：可以理解为 主题 

### 查看当前服务器中的所有 topic 

```bash
bin\windows\kafka-topics.bat --list --zookeeper localhost:2181
```

### 创建 topic 

```bash
bin\windows\kafka-topics.bat --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```

参数说明：

- `--topic` ：定义 topic 名 
- `replication-factor` ：定义副本数
- `partitions` ：定义分区数

> 为了实现扩展性，一个非常大的 topic 可以分布在多个 broker（即服务器）上，一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列 。
>
> a broker = a kafka server ，a broker can contain N topic ，a topic can contain N partition ，
>
> a broker can contain a part of a topic (a broker can contain M(N>M) partition)

### 删除 topic 

```bash
bin\windows\kafka-topics.bat --zookeeper localhost:2181 --delete --topic my-replicated-topic
```

> 需要配置文件 `server.properties` 中设置参数 `delete.topic.enable=tue` ，否则只是标记删除。



### 查看某个topic的详情 

```bash
bin\windows\kafka-topics.bat --zookeeper localhost:2181 --describe --topic first
```



### 修改分区数

```bash
bin\windows\kafka-topics.bat --zookeeper localhost:2181 --alter --topic first --partitions 6
```





## 工作流程

![05.png](https://gitee.com/jallenkwong/LearnKafka/raw/master/image/05.png)

kafka 中消息是以 `topic` 进行分类的，`producer` 生产消息，`consumer` 消费消息，都是面向 topic 。（从命令行操作可以看出 。）

```bash
bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic test
bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test --from-beginning
```

- tpoic 是逻辑上的概念，而 partition 是物理上的概念。每一个 partition 对于一个 log 日志文件，该 log 文件中存储的就是 producer 生产的数据。（topic = N partition ，a partition = a log）
- Producer 生产的数据会被不断追加到该 log 文件末端，且每条数据都有自己的 offset 。consumer 组中的每个 consumer 都会实时记录自己消费到了哪个 offset 。以便于出错恢复时，从上次的位置继续消费 。（producer -> log with offset -> consumer(s)）







