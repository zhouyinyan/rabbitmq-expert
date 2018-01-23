[官网链接](http://www.rabbitmq.com/configure.html)

# RabbitMQ 配置

### 概述
rabbitmq有内置配置，在某些环境(比如开发和测试)下完全足够使用，但在其他场景下，比如生产环境，需要做一些broker以及plugin的配置。

本手册包括了配置相关的一些主题：  
	配置broker和plugin的方式  
	配置文件  
	环境变量  
	有效的核心配置
	配错：如何确认配置文件路径和生效配合  

### 配置方式
	使用配置文件(configuration files)：配置文件可以配置几乎所有的配置，包括网络相关配置、TLS、资源限制(告警)、认证和授权、消息存储配置等等。  
	使用环境变量(environment variables) ：环境变量可以定义节点名称、文件目录路径、运行时标志等  
	使用rabbitmqctl命令行工具：管理虚拟主机、用户、权限等。  
	使用rabbitmq-plugins工具：管理插件  
	使用运行时参数和策略(Runtime Parameters and Policies)：用于集群范围内的运行时参数管理。  

大多数配置使用前两种方式，因此本手册我们主要介绍前两种。 

### 配置文件

**简介**  
虽然rabbitmq的有些配置可以通过环境变量设置，但绝大多数配置是使用配置文件（rabbitmq.conf)来设置。配置文件包含了核心服务配置和插件配置。

**配置文件路径**  
默认的配置文件路径随着操作系统和安装包类型不同而不同，具体的可以参考[默认配置文件路径](http://www.rabbitmq.com/configure.html#config-location)  
当你不知道配置文件路径时，可以通过日志文件或者图形管理界面(//ps:我找了一下，在3.7.2中没看到)查看到。 

**确认配置：查找配置文件路径**  
生效的配置文件路径可以通过查看rabbitmq的日志文件找到，比如：
```java
node           : rabbit@example
home dir       : /var/lib/rabbitmq
config file(s) : /etc/rabbitmq/advanced.config
               : /etc/rabbitmq/rabbitmq.conf
```
如果配置文件不存在，日志大概长这样：  
```erlang
node           : rabbit@example
home dir       : /var/lib/rabbitmq
config file(s) : /var/lib/rabbitmq/hare.conf (not found)
```
(//ps:也许你看到的不一样，比如我看没有配置文件时长` config file(s) : (none)`这样)。另外也可以通过管理界面（   [management UI](http://www.rabbitmq.com/management.html)）查看。  
当在对配置配错时，确认配置文件路径正确，并且已经被rabbitmq正确的读取是非常有用的。  

**确认配置: 如何检查生效配置**  
使用rabbitmqctl environment 命令可以打印生效的配置(用户提供的配置和默认配置合并结果)。在配错时比较有用。  

**新老配置文件格式**  
在rabbitmq3.7.0之前的版本，配置文件叫`rabbitmq.config`，并且使用标准erlang配置格式。虽然在3.7.0和之后的版本，这种格式依然兼容，但建议使用3.7.0和之后版本的同学使用新的叫`sysctl`（ [sysctl format](https://github.com/basho/cuttlefish/wiki/Cuttlefish-for-Application-Users)）格式。  
新格式非常容易使用像Chef、Puppet、或者BOSH等工具生成。比较下面新老格式文件:  
```java
ssl_options.cacertfile           = /path/to/testca/cacert.pem
ssl_options.certfile             = /path/to/server/cert.pem
ssl_options.keyfile              = /path/to/server/key.pem
ssl_options.verify               = verify_peer
ssl_options.fail_if_no_peer_cert = true
```
```java
[
    {rabbit, [{ssl_options, [{cacertfile,           "/path/to/testca/cacert.pem"},
                             {certfile,             "/path/to/server/cert.pem"},
                             {keyfile,              "/path/to/server/key.pem"},
                             {verify,               verify_peer},
                             {fail_if_no_peer_cert, true}]}]}
].
```
新格式的可读性，可维护性和可使用工具管理能力都比较方便。当然新格式也有一些限制，比如当配置LDAP支持时，可能需要配置深层次的嵌套结构，此时使用老格式会更好。rabbitmq支持`rabbitmq.config`的老格式配置文件，也可同时支持两种格式的配置 `rabbitmq.conf` 和 `advanced.config`.   

**rabbitmq.conf文件**  
前面说到，3.7.0开始，rabbitmq.conf配置文件的格式使用新格式(sysctl)，该格式可以使用3句话简单说明：  
	每一行表示一个单独的配置  
	每条配置使用 KEY = VALUE  
	使用#开头的行作为注释  

一个简单的配置例子：
```java
listeners.tcp.default = 5673
```
老格式:
```java
[
  {rabbit, [{tcp_listeners, [5673]}]}
].
```
这条配置会修改rabbitmq 监听的端口为5673.  
rabbitmq的源代码仓库包含了一个rabbitmq.conf配置文件的示例（[an example rabbitmq.conf file](https://github.com/rabbitmq/rabbitmq-server/blob/master/docs/rabbitmq.conf.example) ）， 该例子包含了大部分你可能配置的条目。  
可以使用RABBITMQ_CONFIG_FILE环境变量覆盖默认的rabbitmq.conf的配置文件路径。

**advanced.config配置文件**  
一些配置很难通过sysctl格式完成，这时，就可以使用一个额外的erlang格式配置文件(和rabbitmq.config一样的格式)，称之为advanced.config，这个配置文件中的配置将会和rabbitmq.conf的配置合并。  
rabbitmq的源代码仓库包含了一个advanced.config配置文件的示例（ [an example advanced.config file](https://github.com/rabbitmq/rabbitmq-server/blob/master/docs/advanced.config.example) ）。  
可以使用RABBITMQ_ADVANCED_CONFIG_FILE环境变量覆盖默认的advanced.config的配置文件路径。  

**rabbitmq.config配置文件**  
rabbitmq3.7.0和之后的版本依然支持erlang格式的配置文件rabbitmq.config， 使用方式和之前的版本一样。  
rabbitmq的源代码仓库包含了一个rabbitmq.config配置文件的示例([an example configuration file](https://github.com/rabbitmq/rabbitmq-server/blob/master/docs/rabbitmq.config.example))。  
使用erlang格式修改端口的示例：
```java
[
    {rabbit, [{tcp_listeners, [5673]}]}
  ].
```

**rabbitmq.conf和rabbitmq-env.conf的路径**  
默认情况下，这两个配置文件没有创建，但期望在各个平台上保存在下面这些位置：  
Generic UNIX - $RABBITMQ_HOME/etc/rabbitmq/
Debian - /etc/rabbitmq/
RPM - /etc/rabbitmq/
Mac OSX (Homebrew) - ${install_prefix}/etc/rabbitmq/, the Homebrew prefix is usually /usr/local
Windows - %APPDATA%\RabbitMQ\  

如果rabbitmq-env.conf文件不存在，可以手工创建，并通过RABBITMQ_CONF_ENV_FILE环境变量指定。当然，如果是window平台，名字是rabbitmq-env.bat.  
如果rabbitmq.conf文件不存在， 可以手工创建。如果要修改默认位置，使用RABBITMQ_CONFIG_FILE环境变量指定，注意的是rabbitmq自动的在该环境变量值的后面添加.conf。  

**rabbitmq.conf的核心配置**  
下面这些配置是最常用的，但不是全部配置.  

	- listeners： 配置AMQP连接， 默认listeners.tcp.default = 5672
	- num_acceptors.tcp：erlang进程可以接收的连接数，默认num_acceptors.tcp = 10
	- handshake_timeout: 连接握手的最大超时时间，单温毫秒，默认handshake_timeout = 10000
	- listeners.ssl：ssl配置 默认listeners.ssl = none 
	- num_acceptors.ssl ： erlag进程允许的最大TLS连接数 默认num_acceptors.ssl = 1 
	- ssl_options : TLS配置，默认 none
	- ssl_handshake_timeout ：TLS握手超时。默认ssl_handshake_timeout = 5000
	- vm_memory_high_watermark：内存限额（达到限制时触发流控），可以配置为绝对值或者物理内存比例，默认vm_memory_high_watermark.relative = 0.4  
	- vm_memory_calculation_strategy：内存使用计算策略，可以是allocated，rss， legacy， erlang 默认allocated(使用erlang内存分配统计)。
	- vm_memory_high_watermark_paging_ratio ： 达到内存限额的什么比例时，队列开始记录消息到磁盘，并释放内存。默认：vm_memory_high_watermark_paging_ratio = 0.5
	- total_memory_available_override_value ： 用来覆盖可用的总内存。默认undefined
	- disk_free_limit ： 空闲磁盘最小额度：默认disk_free_limit.absolute = 50M
	- log.file.level ： 日志级别，默认log.file.level = info
	- channel_max ： 最大的channel数，默认 channel_max = 0 (表示无限制)
	- channel_operation_timeout ： channel超时时间，默认15000 
	- heartbeat ： 心跳间隔时间，单位秒。0表示禁用心跳  heartbeat = 60
	- default_vhost ： 默认vhost， 默认 default_vhost = /
	- default_user : 默认用户，默认default_user = guest
	- default_pass ： 默认用户密码： 默认 default_pass = guest
	- default_user_tags：默认用户标签： 默认default_user_tags.administrator = true
	- default_permissions ： 默认用户权限，默认 default_permissions.configure=.* default_permissions.read.* default_permissions.write.* 
	- loopback_users : 只能通过loopback连接的用户列表， 默认loopback_users.guest = true 
	- cluster_nodes ：让节点第一次启动的时候自动组成集群。默认none
	- collect_statistics ： 统计模式，主要为management插件使用。默认 collect_statistics = none  
	- collect_statistics_interval ：统计间隔时间 默认 collect_statistics_interval = 5000
	- management_db_cache_multiplier ：影响management插件缓存耗时查询结果的时间，缓存时间=最后一次查询时间*该配置值。默认5.
	- auth_mechanisms ： 认证机制 默认auth_mechanisms.1 = PLAIN auth_mechanisms.2 = AMQPLAIN
	- auth_backends ： 认证后端，默认auth_backends.1 = internal
	- reverse_dns_lookups : 默认 reverse_dns_lookups = false
	- delegate_count ： 默认delegate_count = 16
	- tcp_listen_options ：socket选项，一般不会修改。默认 tcp_listen_options.backlog = 128 等
	- hipe_compile ：hipe编译开关。默认false
	- cluster_partition_handling ： 如何处理集群下网络分裂， cluster_partition_handling = ignore
	- cluster_keepalive_interval ： 节点以什么频率（毫秒）发送keepalive消息给其他节点，注意同net_ticktime不一样。keepalive消息丢失不会导致认为说节点挂了。
	- queue_index_embed_msgs_below ： 默认queue_index_embed_msgs_below = 4096
	- mnesia_table_loading_retry_timeout ： 默认mnesia_table_loading_retry_timeout = 30000
	- mnesia_table_loading_retry_limit ：默认：mnesia_table_loading_retry_limit = 10
	- queue_master_locator ：默认queue_master_locator = client-local
	- proxy_protocol 默认proxy_protocol = false

下面配置仅在advance.config的rabbit部分。
	- msg_store_index_module ： 默认rabbit_msg_store_ets_index
	- backing_queue_module ：默认rabbit_variable_queue
	- msg_store_file_size_limit ：默认16777216
	- trace_vhosts ：默认[]
	- msg_store_credit_disc_bound ：默认{4000, 800}
	- mnesia_table_loading_retry_limit：默认10
	- mnesia_table_loading_retry_timeout ：默认30000
	- queue_index_max_journal_entries ：默认32768
	- mirroring_sync_batch_size ：默认4096
	- lazy_queue_explicit_gc_run_operation_threshold ： 默认1000
	- queue_explicit_ gc_run_operation_threshold ： 默认1000

另外，配置文件中rabbitmq_plugin部分包含很多插件配置，插件配置参考对应插件文档。  

**配置条目加密**  
敏感的配置条目(比如密码，带凭证的URL)可以加密后配置，rabbitmq在自动时会自动解密。  
注：配置家吗并不会让系统更安全，只是让开发者可以遵守“在文本配置文件中不应该包含敏感数据”这条惯例。  

加密的值必须在{encrypted, ...}里面，比如配置默认用户密码：
```java
[
  {rabbit, [
      {default_user, <<"guest">>},
      {default_pass,
        {encrypted,
         <<"cPAymwqmMnbPXXRVqVzpxJdrS8mHEKuo2V+3vt1u/fymexD9oztQ2G/oJ4PAaSb2c5N/hRJ2aqP/X0VAfx8xOQ==">>
        }
      },
      {config_entry_decoder, [
             {passphrase, <<"mypassphrase">>}
         ]}
    ]}
].
```
rabbitmq使用config_entry_decoder中的passphrase来解密，passphrase可以不用硬编码在配置文件中，可以分开在单独的文件中，比如：
```java
[
  {rabbit, [
      ...
      {config_entry_decoder, [
             {passphrase, {file, "/path/to/passphrase/file"}}
         ]}
    ]}
].
```
使用rabbitmqctl和encode命令来加密，使用decode命令解密，比如：
```java
rabbitmqctl encode '<<"guest">>' mypassphrase
{encrypted,<<"... long encrypted value...">>}
rabbitmqctl encode '"amqp://fred:secret@host1.domain/my_vhost"' mypassphrase
{encrypted,<<"... long encrypted value...">>}
```
```java
rabbitmqctl decode '{encrypted, <<"...">>}' mypassphrase
<<"guest">>
rabbitmqctl decode '{encrypted, <<"...">>}' mypassphrase
"amqp://fred:secret@host1.domain/my_vhost"
```
默认的加密机制使用PBKDF2从passphrase导出key，默认的hash算法是sha512，默认的迭代次数是1000，默认的cipher是AES 256 CBC. 你可以配置修改默认方式：
```java
[
  {rabbit, [
      ...
      {config_entry_decoder, [
             {passphrase, "mypassphrase"},
             {cipher, blowfish_cfb64},
             {hash, sha256},
             {iterations, 10000}
         ]}
    ]}
].   
```
或者在命令行：
```java
rabbitmqctl encode --cipher blowfish_cfb64 --hash sha256 --iterations 10000 \
                     '<<"guest">>' mypassphrase
```

**自定义rabbitmq环境** 
许多系统参数可以通过环境变量配置：节点名、配置文件路径、节点通讯端口，erlang vm flags等等。  

在unix-base平台上，可以通过创建/编辑 rabbitmq-env.conf 来定义环境变量。rabbitmq-env.con路径使用RABBITMQ_CONF_ENV_FILE环境变量配置。  
使用标准的环境变量名字，但是不需要RABBITMQ\_前缀，比如RABBITMQ_CONFIG_FILE在配置文件中名字为CONFIG_FILE：
```java
example rabbitmq-env.conf file entries
#Rename the node
NODENAME=bunny@myhost
#Config file location and new filename bunnies.config
CONFIG_FILE=/etc/rabbitmq/testdir/bunnies
```

在windows平台上，导航到Start > Settings > Control Panel > System > Advanced > Environment Variables. 然后创建和修改环境变量。 除此之外，你也可以创建/编辑 rabbitmq-env-conf.bat 来定义，并使用RABBITMQ_CONF_ENV_FILE变量来指明rabbitmq-env-conf.bat文件的路径。
注：windows平台上修改环境变量后，服务必须重新安装才会完全生效，具体方法参考原文中这一节。

**rabbitmq 环境变量**  
所有rabbitmq环境变量都有RABBITMQ\_ 前缀。 环境变量遵从下述规则：  
如果一个shell级别的名叫 RABBITMQ_VAR_NAME的环境变量定义了，那么就是它。  
否者，如果在rabbitmq-env.conf中设置的了 var_name的环境变量，就用这里面的。  
否者，就用系统定义的。  

shell环境变量具有最高优先级，会覆盖rabbitmq-env.conf中的配置，而rabbitmq-env.conf又会覆盖rabbitmq默认的内置配置。  

正常情况下，你应该不太需要配置环境变量。如果你有特殊的需求，可以看看下面这些环境变量：  
	- RABBITMQ_NODE_IP_ADDRESS ：绑定的IP地址，默认空串，表示绑定所有网卡。
	- RABBITMQ_NODE_PORT ： 默认5672
	- RABBITMQ_DIST_PORT ： 节点间通讯端口和CLI工具通讯端口，默认RABBITMQ_NODE_PORT+20000
	- RABBITMQ_NODENAME ： 节点名，默认 Unix*: rabbit@$HOSTNAME Windows: rabbit@%COMPUTERNAME% 
	- RABBITMQ_CONFIG_FILE ： 主配置文件路径。
	- RABBITMQ_ADVANCED_CONFIG_FILE ：附加配置文件路径
	- RABBITMQ_CONF_ENV_FILE ： 环境变量配置文件路径
	- RABBITMQ_USE_LONGNAME ： 是否使用长名
	- RABBITMQ_SERVICENAME ： windows service：RabbitMQ
	- RABBITMQ_CONSOLE_LOG ： 
	- RABBITMQ_CTL_ERL_ARGS ：默认 none
	- RABBITMQ_SERVER_ERL_ARGS ： 
	- RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS ： 
	- RABBITMQ_SERVER_START_ARGS ： 默认 none

另外， 还有几个环境变量用来告诉rabbitmq 日志，插件，数据库等文件路径，在文件路径一文中详细介绍。  
rabbitmq依赖的其他环境变量：
	HOSTNAME  机器名
	COMPUTERNAME  window平台的机器名
	ERLANG_SERVICE_MANAGER_PATH ： erlang的可执行问文件路径
