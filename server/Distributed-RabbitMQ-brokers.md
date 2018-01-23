[原文链接](http://www.rabbitmq.com/distributed.html)

# 分布式rabbitmq

rabbitmq支持的amqp以及由rabbitmq插件支持的其其他消息协议(比如STOMP)，天生就是分布式的。- 从互联网上多个机器的应用程序连接到单独broker是非常常见的。  

让broker自己分布式无论何时都是有必要和值得的。有三种方式可以让broker分布式：clustering，federation， shovel。   

注：你无需只选择一种方式，可以是三种方式的组合。  

## clustering(集群)

clustering连接多个机器到一起形成一个逻辑的broker，通过erlang消息传递来通讯。因此集群中的所有节点都必须拥有相同的erlang cookie，节点间的网络必须是可靠的，节点上的rabbitmq和erlang版本必须一致。  
集群中的所有节点会自动同步vhost，exchanges，users，和permissions信息，但queue可能只存在于单个节点，或者多个节点存在镜像队列。 一个连接到集群任意一个节点的客户端可以看到所有队列，尽管队列不能不在这个节点上。  
通常，使用集群来提高可用性和提高吞吐量。  

## Federation(联邦插件) 

Federation允许一个broker上的交换机或者队列接受另一个broker上exchange或者队列中的消息，通过AMQP协议通讯。因此两个交换机或者队列需要同样的用户和权限。  

Federation交换机通过单向的点对点连接，默认，消息只能在Federation连接上传递一次，但是可以增强来适应更复杂的路由拓扑。有些消息可能不会在链路上传递；如果一个消息在被Federation交换机收到后不能路由到队列，那么它在第一个地方都不会传递(//ps:需要看看插件文档才明白是啥意思了)   

Federation队列同样的是点对点单向连接。消息会根据消费者情况在Federation队列中移动任意多次。 

通常，使用Federation来连接跨公网的brokers，提供pub/sub模式和工作队列模式的服务。

## shovel(铲子插件^_^)

使用shovel来连接brokers和使用Federation连接在概念上非常相似，但是shovel工作在更底层。  
鉴于Federation的目标是提供分布式的交换机和队列，shovel则是简单的从一个broker的队列中消费消息，并转发消息给另一个broker的exchange。

通常，使用shovel来连接跨公网的brokders，并且要使用比Federation更强的控制。

## 概要

| Federation / Shovel               | Clustering                   |
| --------------------------------- | ---------------------------- |
| brokers是逻辑分开的，可能所属者都不同            | 一个单独逻辑broker                 |
| brokers可以是不同的rabbitmq和erlang版本    | 所有节点的rabbitmq和erlang版本必须一样   |
| brokers可以通过不可靠的公网连接               | 节点必须通过可靠的本地网络连接              |
| 通过amqp协议通讯                        | 通过erlang内部节点通讯               |
| 需要分配用户和权限                         | 需要共享erlang cookie            |
| 可用使用任意的拓扑结构连接brokers，连接可以是单向或者双向  | 所有节点可以连接到所有其他节点（星型拓扑），都是双向连接 |
| 在CAP理论中选择的是高可用和分区容错性（AP)          | 在CAP理论中选择的是一致性和分区容错性（CP）     |
| 一个连接到任何broker的客户端，只能看到该broker上的队列 | 一个连接到任意节点的客户端，可以看到所有节点的队列    |




