[原文链接](http://www.rabbitmq.com/backup.html)

# 备份和恢复

### 概述

本文介绍rabbitmq节点中包含的各种类型数据的备份和恢复流程。

### 两种类型的数据

每个rabbitmq节点都有个数据目录保存着该节点的所有信息。 

数据目录中包含两种类型数据：定义(元数据)数据 和 消息存储数据。 

 - 定义(definitions) - 拓扑(topology) 数据

节点和集群的user、vhost、queues、exchanges、bingdings、runtime parameters等都属于definitions类别。 

definitions可通过http api，命令行工具，甚至使用客户端库的app来导出和导入。 

definitions保存在内部数据库，并且在集群所有节点间复制。每个节点都拥有全部的definitions数据的拷贝。当definitions发生变更时， 所有的节点都会在同一事务中执行变更。 所以，备份definitions数据可以从任意节点导出。  

 - 消息存储数据

消息存储在“消息存储“中，为了便于说明，这儿定义”消息存储“代表消息的内部存储。   

每个节点都有自己的数据目录，保存着主队列消息数据， 使用镜像队列能够让消息在节点间复制。 消息数据保存在数据目录的子目录中。 

 ### 数据生命周期

definitions数据通常是静态，而消息数据是在生产者和消费者之间不停的流动。  

当执行备份时，第一步要确定是只备份definitions数据还是包括消息数据也备份。 因为消息通常是短生存并且可能是瞬时的，非常不建议在一个运行着的节点备份消息，这样会导致不一致的数据快照。 相反，definitions数据仅仅只能在运行着的节点执行备份。 

### 备份和恢复definitions

definitions能够被导出为json文件或者手工备份。 大部分情况下，导出/导入是最好的方式。 如果节点名称或者主机名发生变更时，手工备份需要额外的操作。 

- 导出definitions

通过HTTP API可以将definitions导出为json文件：
	management插件的overview页面
	rabbitmqadmin提供命令导出
	通过GET /api/definitions 端点

definitions可以指定是导出具体的vhost还是整个集群。 当指定某个vhost时，一些信息(比如集群用户和权限)会被排除掉。 

- 导入definitions

一个definitions的json文件依然可以通过上面的三种方法导入：
	management插件的overview页面
	rabbitmqadmin提供命令导入
	通过POST /api/definitions 端点

也可以在节点启动时，通过指定load_definitions配置参数来加载本地的definitions文件。 

- 手工备份定义

definitions保存在内部数据库中，而数据库文件保存在节点的数据目录中。 为了获取目录路径，可以运行一下下面的命令:
```java
rabbitmqctl eval 'rabbit_mnesia:dir().'
```
如果节点没有运行，这个命令就无法执行，可以查看默认的数据目录文档[default data directories](http://www.rabbitmq.com/relocate.html).

这个目录包含了存储消息数据的子目录，如果不备份消息数据，跳过复制消息子目录。

- 从手工备份定义中恢复

内部节点数据库在一些记录中存储了节点的名字。 修改节点名称，数据库必须首先被更新以适应修改。 通过使用rabbitmqctl来更新:
```java
rabbitmqctl rename_cluster_node <oldnode> <newnode>
```
改命令可以接受多个oldname / newname 对，以便于同时修改集群中的多个节点名字。 

When a new node starts with a backed up directory and a matching node name, it should perform the upgrade steps as needed and proceed booting.(ps://节点名已经匹配了，还需要啥升级步骤？)

### 备份和恢复消息

备份消息时，节点必须首先要停止。 

在集群中使用镜像队列的场景下， 需要停止整个集群后再备份。 如果停止一个节点，可能丢失消息或者重复消息， 就像备份单节点一样。 

- 手工备份消息

目前这是唯一的备份消息的方法。消息数据存储在节点数据目录中(上面提到)。

从3.7.0开始，所有消息数据在msg_stores/vhosts目录中，并且每个vhost单独存在一个子目录中，每个vhost目录以vhost名称的hash值命名，并且在目录中包含一个.vhost文件记录着vhost的名称，因此，vhost消息数据分开备份。   

在3.7.0之前，消息存储在节点数据目录下的几个目录中：queues, msg_store_persistent 和 msg_store_transient。如果节点是优雅停机，还有一个recovery.dets保存着恢复的元数据。

- 从手工备份消息中恢复

当节点启动时，会计算数据目录路径并且恢复消息。为了恢复消息，broker应该已经存在所有的definitions。 不知道vhost和queue的消息数据不会被加载，并且会被节点删除。 因此当手工备份消息数据目录时（ps://原文中好像有笔误，应该是恢复），确保目标节点上的definitions是存在的，要么通过definitions文件导入，或者备份整个数据目录。  

If a node's data directory was backed up manually (copied), the node should start with all the definitions and messages. There is no need to import definitions first.