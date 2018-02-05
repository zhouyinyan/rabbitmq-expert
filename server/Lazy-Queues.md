[原文链接](http://www.rabbitmq.com/lazy-queues.html)

# lazy queue(延迟队列?)

## 概述

从rabbitmq 3.6.0开始，实现了lazy queue的概念 - 队列尽可能早的移动它的内容到磁盘上，并且仅在消费者请求时加载内容到内存中，因此取名为lazy。  

lazy queue的主要目标是支持非常长的队列(数百万消息量级).队列因为如下原因可以变的非常长：

 - 消费者离线/崩溃/停止维护
 - 突然的消息涌入，生成速度大于消费速度
 - 消费者消费速度下降

默认， 发布到rabbitmq的消息会被队列缓存在内存中，这儿缓存的目的是能够尽快投递消息给消费者。 注意：持久化消息当进入到broker时会被写入磁盘，同时也会保留在RAM中。  

无论何时，broker觉得需要释放内存时，缓存中的消息将会被写入磁盘。批量写入消息到磁盘会花费时间，并且阻塞队列进程，导致在写磁盘时不能接受消息。尽管最近rabbitmq版本增强了写入磁盘算法，但也不合适当你拥有数百万消息要写入磁盘的场景。   

lazy queue尝试尽早将写入磁盘，意味着比常规情况下，在内存中保留非常少的消息，但同时带来更高的磁盘I/O。   

## 构建lazy queue

队列可以被构建运行在 default 模式或者 lazy 模式，通过：
 - 通过 queue.declare 参数设置模式
 - 应用队列策略  

当同时使用两种方式(策略和队列参数)指定模式时，队列参数拥有更高优先级。   

如果队列模式时通过队列参数方式指定的，那么只通过删除队列并且重新声明的方式来修改模式。   
    1. 声明队列时使用队列参数构建  
      通过提供 x-queue-mode参数指定想要的模式，有效的值为“default”,"lazy" .   
        如果为指定任何值，默认使用“default”.   
        使用java声明一个lazy模式的队列例子：
  ```java
   Map<String, Object> args = new HashMap<String, Object>();
   args.put("x-queue-mode", "lazy");
   channel.queueDeclare("myqueue", false, false, false, args);
  ```
    2. 使用策略  
   ```java
   rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"lazy"}' --apply-to queues
   ```
   名字为"lazy-queue"的队列将会运行lazy模式。 策略也可以通过management ui定义。 

  ## 运行时修改队列模式
  如果队列模式是通过策略配置的，那么可以在运行时修改模式，而不需要删除和重建队列(使用队列参数声明的需要删除和重建)。 比如名为lazy-queue的队列修改模式为default:
  ```java 
  rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"default"}' --apply-to queues
  ```

  ## lazy queue 的性能考虑

- 磁盘使用  

  lazy queue会尽快的将消息写入到磁盘中，尽管发布的消息类型是瞬时的(transient)，这通常会导致高的磁盘I/O使用。   

  而常规的队列会尽可能长的保留消息在内存中，这会导致较低的延迟的磁盘I/O(拥有更多抖动-尖刺)，因为一次需要往磁盘写入跟多的数据。   

 - 内存使用  

  做一个简单的测试来对比常规和lazy queue的内存占用. 

| Number of messages | Message body size | Message type | Producers | Consumers |
| ------------------ | ----------------- | ------------ | --------- | --------- |
| 1,000,000          | 1,000 bytes       | persistent   | 1         | 0         |

在常规和lazy 队列吃掉上面这样量级的消息之后，内存使用情况:

| Queue mode | Queue process memory | Messages in memory | Memory used by messages | Node memory |
| ---------- | -------------------- | ------------------ | ----------------------- | ----------- |
| default    | 257 MB               | 386,307            | 368 MB                  | 734 MB      |
| lazy       | 159 KB               | 0                  | 0                       | 117 MB      |

两种队列都会持久化1000000条消息和使用1.2G磁盘空间。   



- 测试细节

  下面是一个简单测试脚本，测试常规队列模式：
```java
# Start a temporary RabbitMQ node:
#
#       export RABBITMQ_NODENAME=default-queue-test
#       export RABBITMQ_MNESIA_BASE=/tmp
#       export RABBITMQ_LOG_BASE=/tmp
#       rabbitmq-server &
#
# (the last command will fail if there is another RabbitMQ node already running)

# In a https://github.com/rabbitmq/rabbitmq-perf-test clone, run:
make run ARGS="-y 0 -s 1000 -f persistent -C 1000000 -u default -ad false"
# Run gmake on OS X

# Queue stats:
rabbitmqctl list_queues name arguments memory messages_ram message_bytes_ram messages_persistent message_bytes_persistent
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
default []  417421592   386307  386307000   1000000 1000000000

# Node memory stats
rabbitmqctl status | grep rss,
      {total,[{erlang,1043205272},{rss,770306048},{allocated,1103822848}]}]},

# Stop our temporary RabbitMQ node & clean all persistent files
#
#       rabbitmqctl shutdown
#       rm -fr /tmp/{log,$RABBITMQ_NODENAME*}
```



测试lazy queue的脚本非常类似:

```java
# Use a different RABBITMQ_NODENAME
# All other variables remain the same as the previous example
#
#       export RABBITMQ_NODENAME=lazy-queue-test

# In a https://github.com/rabbitmq/rabbitmq-perf-test clone, run:
make run ARGS="-y 0 -s 1000 -f persistent -C 1000000 -u lazy -qa x-queue-mode=lazy -ad false"
# Run gmake on OS X
```

注：这是非常简单的测试，你应该为你的特定环境做基准测试。  


- 运行时切换模式  

当模式从default转换为lazy时，该操作会遭受和将消息写入磁盘一样的性能影响。  

当转换时(default->lazy), 队列首先将内存中的所有消息写入磁盘，操作在进行中时，不会接受发布过来的新的消息。在最初的写入磁盘操作完成后，队列开始接受消息，ack和其他命令。  

当队列从lazy转为default时，会执行和服务重启时队列恢复一样的过程：一批(16384)消息会被加载到内存中。   

## 警告和限制  

当需要优先保障低内存使用以及能够接受高的磁盘I/O，lazy queue是合适的，lazy queue的其他方面也需要考虑清楚。   

 - 节点启动

When a node is running and under normal operation, lazy queues will keep all messages on disk, the only exception being in-flight messages.

When a RabbitMQ node starts, all queues, including the lazy ones, will load up to **16,384**messages into RAM. If [queue index embedding](http://www.rabbitmq.com/persistence-conf.html) is enabled (the queue_index_embed_msgs_belowconfiguration parameter is greater than 0), the payloads of those messages will be loaded into RAM as well.

For example, a lazy queue with **20,000** messages of **4,000** bytes each, will load **16,384**messages into memory. These messages will use **63MB** of system memory. The queue process will use another **8.4MB** of system memory, bringing the total to just over **70MB**.

This is an important consideration for capacity planning if the RabbitMQ node is memory constrained, or if there are many lazy queues hosted on the node.

**It is important to remember that an under-provisioned RabbitMQ node in terms of memory or disk space will fail to start.**

Setting queue_index_embed_msgs_below to 0 will disable payload embedding in the queue index. As a result, lazy queues will not load message payloads into memory on node startup. See the [Persistence Configuration guide](http://www.rabbitmq.com/persistence-conf.html) for details.

When setting queue_index_embed_msgs_below to 0 all messages will be stored to the message store. With many messages across many lazy queues, that can lead to higher disk usage and also higher file descriptor usage.

Message store is append-oriented and uses a compaction mechanism to reclaim disk space. In extreme scenarios it can use two times more disk space compared to the sum of message payloads stored on disk. It is important to overprovision disk space to account for such peaks.

All messages in the message store are stored in 16MB files called segment files or segments. Each queue has its own file descriptor for each segment file it has to access. Foe example, if 100 queues store 10GB worth of messages, there will be 640 files in the message store and up to 64000 file descriptors. Make sure the nodes have a high enough [open file limit](http://www.rabbitmq.com/production-checklist.html#resource-limits-file-handle-limit) and overprovision it when in doubt (e.g. to 300K or 500K). For new installations it is possible to increase file size used by the message store using msg_store_file_size_limit configuration key. **Never change segment file size for existing installations** as that can result in a subset of messages being ignored by the node and can break segment file compaction.



#### Lazy Queues with Mixed Message Sizes

If all messages in the first **10,000** messages are below the queue_index_embed_msgs_below value, and the rest are above this value, only the first **10,000** will be loaded into memory on node startup.

#### Lazy Queues with Interleaved Message

Given the following interleaved message sizes:

| Position in queue | Message size in bytes |
| ----------------- | --------------------- |
| 1                 | 5,000                 |
| 2                 | 100                   |
| 3                 | 5,000                 |
| 4                 | 200                   |
| ...               | ...                   |
| 79                | 4,000                 |
| 80                | 5,000                 |

Only the first **20** messages below the queue_index_embed_msgs_below value will be loaded into memory on node startup. In this scenario, messages will use **21KB** of system memory, and queue process will use another **32KB** of system memory. The total system memory required for the queue process to finish starting is **53KB**.