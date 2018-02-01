[原文链接](http://www.rabbitmq.com/clustering.html)

# 概览
一个rabbitmq broker是一个包含一个或者多个erlang节点（每个节点运行着rabbitmq应用程序），共享用户, 虚拟主机, 队列, 交换机, 绑定和运行时参数的逻辑组，有时我们说节点的集合就是一个集群。  

### 什么是复制

rabbitmq操作的所有数据/状态需要复制到所有集群节点上，除了队列，默认队列只存在于一个节点，但是所有集群节点都是可以看到并且可达的。 如果要复制队列到集群的其他节点，查看 [high availability](http://www.rabbitmq.com/ha.html)文档（注：前提是必须工作在集群）。 

### 要求主机解析

rabbitmq节点地址使用域名来识别，要么是短域名，要么是全域名(FQDNs). 因此集群中的所有成员的主机名都必须能被其他所有节点解析，也包括使用如rabbitmqctl命令行工具的主机。  

主机名解析可以使用标准的操作系统提供的方法：  
	DNS记录  
	本地host文件(比如/etc/hosts)  
在某些要求严格的环境下，对DNS记录或者hosts的修改是被限制的，也可以使用erlang vm配置来替代标准os方法（具体查看文档[Erlang VM can be configured to use alternative hostname resolution methods](http://erlang.org/doc/apps/erts/inet_cfg.html)）。  
使用FQDNs，在查看[Configuration guide](http://www.rabbitmq.com/configure.html#define-environment-variables)中查看RABBITMQ_USE_LONGNAME配置。  

### 集群构建

集群可以通过多种方式来构建：
 使用配置文件声明  
 基于dns发现声明  
 基于AWS(EC2)实例发现声明  
 基于Kubernets发现声明  
 基于Consul发现声明  
 基于etcd发现声明  
 使用rabbitmqctl手工配置  

集群的成员可以动态的修改，所有的rabbitmq broker开始运行时都是单节点，他们可以加入到集群中，集群中的节点也可以从集群中分离。

### 故障处理  
rabbitmq broker允许个别的节点故障。 rabbitmq集群有多种方式来处理网络分区([network partitions](http://www.rabbitmq.com/partitions.html))  。集群意味者在LAN上构建，而不建议在WAN上。Shovel或者Federation插件是在WAN上连接broker更好的解决方案。  

###  磁盘或者内存节点  

节点可以是磁盘节点或者内存节点。内存节点存储内部的数据表在内存(RAM)中，但是不包括消息，消息索引，队列索引和其他节点状态。  

在90%的情况下你都应该使用磁盘节点，内存节点仅在特殊的情况下用来提高性能(非常高的队列，交换机，绑定的创建和销毁)。内存节点不会提高消息吞吐率。  

因为内存节点存储内部数据表在内存中，因此启动的时候它必须从其他节点同步信息。这意味着一个集群中至少包含一个磁盘节点，所以不可能从集群中移除最后一个磁盘节点。  

### 集群间的节点(包括CLI 节点)认证：erlang cookie








