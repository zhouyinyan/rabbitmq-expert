[原文链接](http://www.rabbitmq.com/monitoring.html)

# 监控

## 基础设施和核心指标
监控系统的第一步是监控基础设施核心指标。应该在所有的rabbitmq节点，甚至于所有应用程序的节点上监控下述指标：

CPU(idle，user，system， iowait)
Memory(free，cached，buffered)
Disk I/O(reads & writes per unit time，I/O wait percentages)
Free Disk Space
File descriptors used by `beam.smp` vs. mas system limit
Network throughput(bytes received,bytes send) vs. maximum network link throughput
vm statistic(dirty page flushes, writeback volume)
system load average(/proc/loadavg)

不缺乏这类工具(如Graphite或者Datadog)来周期的收集，存储、可视化这些核心指标。 

## rabbitmq指标
rabbitmq management 插件提供了监控rabbitmq指标的能力，然而management插件本身有限制，只会存储一天的指标数据。 保存历史的指标数据有助于判断问题的根因，和规划未来的容量。

management 插件的HTTP API通过api/queues/vhost/qname 端点收集rabbitmq指标。建议60s的间隔时间收集指标，因为更快的频率收集可能引起rabbitmq server的负载变高，影响性能。

| 指标                    | json字段名                            |
| --------------------- | ---------------------------------- |
| memory                | memory                             |
| queued messages       | messages                           |
| un-acked messsages    | messages_unacknowledged            |
| messages published    | message_stats.publish              |
| message publish rate  | message_stats.publish_details.rate |
| message delivered     | message_stats.deliver_get          |
| message delivery rate | message_stats.deliver_get.rate     |
| orther message stats  | message_stats.*                    |


## 三方监控工具

下面列表列出了三方收集rabbitmq指标的工具。

AppDynamics  
collectd  
datadog  
ganglia  
munin  
nagios  
new relic  
prometheus  
zabbix  
zenoss  

## 应用级别监控  

使用rabbitmq，或者基于消息的系统，通常是分布式的，分布式的系统常常无法明显的发现那个组件出现问题。 因此分布式系统中的各个组件，包括应用程序，应该被监控。  

一些基础级别指标和rabbitmq指标可以表明系统不正常，但不能直接指明问题根因。比如，非常容易知道一个节点的磁盘空间不够了，但不清楚为什么。这就是应用级别监控指标用到的地方：他们能够帮助识别一个下线的生产者，一个重复失败的消费者，一个不能跟上速率的消费者，甚至一个低速处理的下游服务(比如，一个没有使用数据库索引的消费者)  

一些客户端库(比如java客户端)和框架(比如 spring amqp) 提供了注册指标收集器功能，也有开箱即用的指标收集器。 

## 日志聚合  

考虑从所有rabbitmq节点收集日志，如果可能甚至包括所有的应用上也收集。如同指标，日志能够提供重要的线索，帮组识别问题根因。


