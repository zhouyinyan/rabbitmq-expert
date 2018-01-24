[原文链接](http://www.rabbitmq.com/reliability.html)

# 可靠性

本页说明怎么使用amqp和rabbitmq的各种特性来实现可靠投递 -- 确保消息始终投递，即使系统中的各个部分遭遇到故障。  

### 什么会导致故障?

网络问题是最常见的故障。不仅仅是网络故障，防火墙也可以中断空闲的连接，并且网络故障也不总是能立即被监测到。  
除网络故障外，broker和客户端应用也可能随时出现硬件故障(或者软件崩溃)，此外，就算客户端应用一直保持运行，但逻辑错误也可能导致channel或者connection错误，这会导致客户端重新连接并修复问题。 

### 网络故障

如果发生网络故障，客户端需要和broker重新建立新的连接，因之前的connection和从connection开启的channel会自动关闭，因此它们也需要重新开启。  
通常，当网络故障时，客户端会收到connection抛出的异常，官方的java和.net客户端提供了回调方法让你可以监听connection故障。 java在Connection类和Channel类提供了ShutdownListener回调，.net提供了IConnection.ConnectionShutdown和IModel.ModelShutdown事件。  

### 回执和确认(Acknowledgements and Confirms)

当网络发送故障时， 客户端和服务端可能正在传输消息 - 消息可能处在操作系统的缓冲区，或者网线上，处于解析或者生成阶段等等，而传输中的消息会丢失 ，因此应该重新传输这些消息。 回执可以让服务端和客户端知道什么时候重新传输。  

回执可以用于两个方向 - 让消费者向服务端表明它已经成功接收/处理消息，也可以让服务端向生产者表明它已经成功接收/处理消息， rabbitmq称第二种为确认(confirm)。 

Of course, TCP ensures that packets have been received, and will retransmit until they are - but that's just the network layer. Acknowledgements and confirms indicate that messages have been received and acted upon. An acknowledgement signals both the receipt of a message, and a transfer of ownership where the receiver assumes full responsibility for it.(//PS:还是在说明回执的概念)

消息回执语义：一个消费者应用收到消息后，应该处理完需要它需要处理的工作(比如记录到数据库，转发消息，打印等等)之后，才发送回执给broker。一旦回执，broker将会删除消息。  

同样的，broker一旦收到消息并作出了相应的工作之后（//其他章节会细讲），会发送消息确认(confirm)给生产者应用。

使用回执和确认保证至少一次投递。不使用回执和确认，消息可能在发送和消费过程中丢失，仅保证最多一次投递。

### 使用心跳(heartbeat)监测死tcp连接

操作系统会监测并中断占用时间较长的TCP连接（比如linux系统默认配置的TCP连接时长是11分钟）。AMQP 0-9-1提供了心跳(heartbeat)特性使得应用层可以快速的发现被断掉的连接。心跳也可以保护因某些网络设备可能终止空闲的TCP连接。细节请查看[Heartbeats](http://www.rabbitmq.com/heartbeats.html) 章节。 

### broker端

为了避免broker丢消息，我们需要处理broker重启，broker硬件故障和甚至broker崩溃的情况。  
为了确保消息以及broker定义（exchange，queue，bingding）在重启后幸存，我们需要确保他们被写入磁盘。AMQP标准存在持久化exchange、queue以及持久化消息的概念，要求持久化的对象或者消息在重启后能够幸存。更多信息参考[AMQP Concepts Guide](http://www.rabbitmq.com/tutorials/amqp-concepts.html).

### 集群和高可用

如果我们需要确保broker幸免于硬件故障，那么使用rabbitmq集群。在rabbitmq的集群中，所有的定义(exchanges，bingdings，users等等)会镜像到整个集群。但queues不一样，默认情况下，队列只存在于声明的那个节点(单个节点)，但是也可以镜像到多个或者全部节点。不管队列存在哪儿，队列对所有节点都是可见且可达的。  

镜像队列复制他们的内容给所有配置的节点上，可以容忍节点故障而不会丢失消息。不管怎样， 消费者应用需要知道当他们消费的队列故障时，应该取消，然后重新消费。 查看[the documentation](http://www.rabbitmq.com/ha.html#behaviour)了解细节。  

### 生产者端

当使用消息确认，生产者可以从channel或者connection故障中恢复，通过重新传输哪些没有从broker返回确认的消息。这可能导致消息重复发布，因为broker可能给生产者发送了一个确认，但没有到达生产者(因为网络故障等等)。因此，消费者应用需要以幂等的方式处理重复消息。  

### 确保消息成功路由

在某些情况，生产者需要确保它发布的消息已经路由到队列了(当然在pub-sub系统中，生产者仅仅发布消息，如果没有消费者订阅，消息被丢弃也是正常的)。   

为了确保消息路由到了一个已知的单个队列，生产者仅仅需要定义目标队列，并直接发布到该队列即可。如果消息路由更复杂，并且生产者依然需要知道它发布的消息是否到达至少一个队列，就需要在发布消息的时候使用mandatory标记，保证在没有合适的队列绑定时，一个basic.return（包含应答码和一些解释文本）将会发送回客户端。  

生产者应该意识到它是发布消息到集群节点， 如果一个或者多个绑定到交换机的队列存在镜像，可能因为网络故障导致节点间延迟。查看  [here](http://www.rabbitmq.com/nettick.html)获取详细信息。  

### 消费者端

当发生网络故障(或者节点崩溃)时，消息可能重复，消费者必须准备好处理重复消息。 最简单有效的是确保消费端处理消息是幂等的而不是明确的处理重复消息。   

如果一个消息已经投递给消费者，然后又重入队列(比如因为网络断了导致broker没有收到ack)， 当重新投递消息时，rabbitmq会给消息打上redelivered标记。 这个标记是一个提示：消费者可能之前收到过该消息(但这不是完全保证的，因为消息可能从broker发出后就断线了，还没有到达消费者)，相反的，如果消息中没有redelivered标记，那么可以肯定消费者是第一次收到该消息。因此如果一个消费者发现删除重复消息或者幂等操作太昂贵，it can do this only for messages with the redelivered flag set.(ps://没搞清楚这句具体是让做什么，丢弃消息？还是怎么做)  

### 消费者取消通知

在某些情况下，server需要取消消费者 - 因为消费者消费的队列被删除，或者发生故障转移(在这种情况下，消费者会重新消费，但是应该意识到它可能会消费之前已经消费过的消息)。   

注：消费者取消通知是rabbitmq对amqp的扩展，并不是所有客户端都支持(ps://当然java肯定是支持的)

### 不能被处理的消息

如果消费者监测到它不能处理消息，那么它可以通过使用basic.reject (或者 basic.nack)拒绝(reject)消息，要么要求server重入队列投递，要么不重入队列(这种情况下，server可能配置死信队列)。   

### 分布式rabbitmq

rabbit提供两个插件来帮助在不可靠网络上组建分布式节点：federation 和 shovel， 他们都作为AMQP客户端实现，因此你可以配置他们使用消息回执和消息确认，默认这两个特性是开启的。 

When connecting clusters with federation or the shovel, it is desirable to ensure that the federation links and shovels tolerate node failures. Federation will automatically distribute links across the downstream cluster and fail them over on failure of a downstream node. In order to connect to a new upstream when an upstream node fails you can specify multiple redundant URIs for an upstream, or connect via a TCP load balancer.

When using the shovel, it is possible to specify redundant brokers in a source or destination clause; however it is not currently possible to make the shovel itself redundant. We hope to improve this situation in the future; in the mean time a new node can be brought up manually to run a shovel if the node it was originally running on fails.









