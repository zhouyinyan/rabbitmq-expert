[原文链接](http://www.rabbitmq.com/persistence-conf.html)

# 持久化配置  

大多数情况下，默认的配置可以让rabbitmq持久化层工作的很好；尽管如此，有些配置有时候很有用，本文将说明如何配置。建议在做任何配置之前，先读下本文。  

## 持久化层如何工作

首先，先明白一些背景：持久化消息和瞬时消息都能够写入磁盘. 持久化消息是一旦消息到达队列，尽可能快的写入磁盘，而瞬时消息仅在当内存吃紧时被逐出内存后写入磁盘。 持久化消息也会常驻内存中，仅当内存吃紧后被逐出内存。 “持久化层”指保存这两种类型消息到磁盘的机制。   

本文中的“队列”指非镜像队列或者主队列或者从队列，镜像队列的持久化机制会有所不同。   

持久化层有两个组件：队列索引(queue index)和消息存储(message store)。 队列索引负责维护一个消息在队列中的哪儿，一直到消息是否已经投递或者ack。 因此每个队列都有一个队列索引。 消息存储是server中所有队列共享的一个k-v存储，消息(包括body，properties，headers)可以直接存储在队列索引中，或者写入消息存储中。 技术上存在两种消息存储(一个为瞬时消息，一个为持久化消息)，但通常都称为“消息存储”。   

## 内存消耗  

内存吃紧时，持久化层尝试尽可能多的将消息写入磁盘，尽可能多的从内存中删除消息。尽管如此，还是有一些信息必须保留在内存中：  
  每个队列为每个unacknowledged消息维护的一些元信息。如果消息是要保存到消息存储中，消息本身可以从内存中删除。  
  消息存储需要索引。 默认的消息存储索引为存储中的每个消息使用了小量的内存。 

## 队列索引中的消息  

将消息写入队列索引存在一些好处和坏处：  
  好处：  
   只需一次操作即可将消息写入磁盘，而不需要两次操作；for tiny messages this can be a substantial gain（对小体积消息来说，这样很好）  
   消息写入队列索引不需要在消息存储索引中记录，thus do not have a memory cost when paged out（因此当置换内存页时没有内存消耗）。 

  坏处：  
  队列索引会在内存中保持一定数量的记录，如果写入队列索引中的不是小体积消息，将会对内存消耗有较大影响。   
  如果消息被路由到多个队列，那么消息就会被爱多次写入到多个队列索引中。但如果是保存到消息存储中，只需要写一次。   
  队列索引中未ACK的消息会始终驻留在内存中。 

比较好的做法是，将非常小体积的消息保存到队列索引中，而将其他消息写入到消息存储中。 这个可以通过配置queue_index_embed_msgs_below来控制。 默认，消息序列化后体积小于4096(包括属性和头)将会保存在队列索引中。   

当从磁盘中读取消息时，每个队列索引需要至少在内存中保留一段文件，该段文件包含了16384条消息记录。 因此如果要增加 queue_index_embed_msgs_below值时要非常小心，一点点的增加会导致大量的内存消耗。   

## 限制持久化性能的次要方面

持久化可能性能不佳，是因为被文件描述符或者工作的异步线程数量限制。 当你有大量的队列需要同时访问磁盘时，就会发生这两种场景。  

### 太少的文件描述符(file handlers)

The RabbitMQ server is typically limited in the number of file handles it can open (on Unix, anyway). Every running network connection requires one file handle, and the rest are available for queues to use. If there are more disk-accessing queues than file handles after network connections have been taken into account, then the disk-accessing queues will share the file handles among themselves; each gets to use a file handle for a while before it is taken back and given to another queue.

This prevents the server from crashing due to there being too many disk-accessing queues, but it can become expensive. The management plugin can show I/O statistics for each node in the cluster; as well as showing rates of reads, writes, seeks and so on it will also show a rate of reopens - the rate at which file handles are recycled in this way. A busy server with too few file handles might be doing hundreds of reopens per second - in which case its performance is likely to increase notably if given more file handles.

### 太少的异步线程(async threads)

The Erlang virtual machine creates a pool of async threads to handle long-running file I/O operations. These are shared among all queues. Every active file I/O operation uses one async thread while it is occurring. Having too few async threads can therefore hurt performance.

Note that the situation with async threads is not exactly analogous to the situation with file handles. If a queue executes a number of I/O operations in sequence it will perform best if it holds onto a file handle for all the operations; otherwise we may flush and seek too much and use additional CPU orchestrating it. However, queues do not benefit from holding an async thread across a sequence of operations (in fact they cannot do so).

Therefore there should ideally be enough file handles for all the queues that are executing streams of I/O operations, and enough async threads for the number of simultaneous I/O operations your storage layer can plausibly execute.

It's less obvious when a lack of async threads is causing performance problems. (It's also less likely in general; check for other things first!) Typical symptoms of too few async threads include the number of I/O operations per second dropping to zero (as reported by the management plugin) for brief periods when the server should be busy with persistence, while the reported time per I/O operation increases.

The number of async threads is configured by the +A argument to the Erlang virtual machine as described here, and is typically configured through the envirnment variable RABBITMQ_SERVER_ERL_ARGS. The default value is +A 64. It is likely to be a good idea to experiment with several different values before changing this.

## Alternate message store index implementations

As mentioned above, each message which is written to the message store uses a small amount of memory for its index entry. The message store index is pluggable in RabbitMQ, and other implementations are available as plugins which can remove this limitation. (The reason we do not ship any with the server is that they all use native code.) Note that such plugins typically make the message store run more slowly.

  

  

  

  

  

  
