[原文链接](http://www.rabbitmq.com/vhosts.html)

# 虚拟主机(vhost)

### 介绍

rabbitmq是一个多租户系统：connections、exchanges、queues、bingdings、user permissions、plicies 和 其他的一些对象都属于vhost，一个逻辑上的实体组。如果你熟悉apache的虚拟主机或者nginx的server blocks，他们的概念是相似的。但他们有一个重要的不同是：apache的虚拟主机是定义在配置文件中，而rabbitmq是通过rabbitmqctl或者HTTP API创建。 

### 逻辑和物理分离

vhost提供逻辑上的资源的分组和隔离。物理上的资源隔离不是vhost的目标。 

### vhost和client connection

vhost有一个名称，当客户端连接到rabbitmq时，需要指定vhost的名称。如果认证通过，并且用户名是在要连接的这个vhost已经授权了，连接才会建立。 

连接到某个vhost上之后，只能操作在这个vhost上的对象(exchanges，bindings，queues等等)。 要做跨vhost的操作，那么必须同时建立两个连接分别连接到不同的vhost，比如一个应用可以消费一个vhost中的消息，然后将消息发布到另一个vhost。 shovel插件就是这种应用的一个例子。 

### vhost和STOMP

同amqp 0-9-1一样，STOMP也包含了vhost的概念，查看[STOMP guide](http://www.rabbitmq.com/stomp.html)了解详情。 

### vhost和MQTT

MQTT没有vhost的概念。查看[MQTT guide](http://www.rabbitmq.com/mqtt.html) 了解详情。

### 限制

在某些场景下，需要限制vhost上允许的队列数量，客户端连接数等，在rabbitmq 3.7.0通过 per-vhost限制实现。限制配置可以通过rabbitmqctl或者HTTP API完成。   

例如使用rabbitmqctl配置最大连接数:
```java
rabbitmqctl set_vhost_limits -p vhost_name '{"max-connections": 256}'
```
可以通过设置限制为0，禁止连接
```java
rabbitmqctl set_vhost_limits -p vhost_name '{"max-connections": 0}'
```
设置为负数，不限制
```java
rabbitmqctl set_vhost_limits -p vhost_name '{"max-connections": -1}'
```
	
配置最大的队列数量:
```java
rabbitmqctl set_vhost_limits -p vhost_name '{"max-queues": 1024}'
````
设置为负数，不限制
```java
rabbitmqctl set_vhost_limits -p vhost_name '{"max-queues": -1}'
```