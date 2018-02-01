[原文链接](http://www.rabbitmq.com/management.html)

# 管理插件(management plugin)

### 介绍
management插件提供基于http的api用于管理和监控你的rabbitmq server，通过浏览器UI和一个命令行工具（rabbitmqadmin）的方式。 包含的特性：

	声明、列出和删除交换机、队列、绑定、用户、虚拟主机和权限。  
	监控队列长度、全局消息速率和每个channel、每个连接的数据速率等  
	监控资源使用、比如文件描述符(file descriptors)、内存使用、可用磁盘等  
	管理用户  
	导出和导入对象定义(vhost，users，permissions，queues，exchanges，bindings，parameters，policies)到json  
	关闭连接，清空队列  
	发送和接受消息(仅用于开发环境和调试)  

### 入门

management插件包含在rabbitmq的发行包里，使用rabbitmq-plugins工具启用:
```java
rabbitmq-plugins enable rabbitmq-management
```
 web ui地址为http://server-name:15672/
 http api地址为http://server-name:15672/api
 rabbitmqadmin下载：http://server-name:15672/cli/

 使用web ui，需要用rabbitmq user授权登录(全新安装时，默认用户为guest/guest)。从这儿你可以管理交换机、队列、vhost、用户和权限。

 ### 权限
 management插件继承了一些rabbitmq权限模型，可以给用户设置任意的标签。management插件的用户标签称为"management","policymaker","monitoring","administrator". 下表说明不同的类型可以做的事情：

| TAG           | capability                               |
| ------------- | ---------------------------------------- |
| （none）        | No access to the management plugin       |
| management    | Anything the user could do via AMQP plus:<br />  <br />  List virtual hosts to which they can log in via AMQP <br />  View all queues, exchanges and bindings in "their" virtual hosts <br />  View and close their own channels and connections  <br />  View "global" statistics covering all their virtual hosts, including activity by other users within them |
| policymaker   | Everything "management" can plus:<br />     View, create and delete policies and parameters for virtual hosts to which they can log in via AMQP |
| monitoring    | Everything "management" can plus:<br />     List all virtual hosts, including ones they could not log in to via AMQP<br />     View other users's connections and channels<br />     View node-level data such as memory use and clustering<br />     View truly global statistics for all virtual hosts |
| administrator | Everything "policymaker" and "monitoring" can plus:<br />     Create and delete virtual hosts<br />     View, create and delete users<br />     View, create and delete permissions<br />     Close other users's connections |

 因为"administrator"可以做所有"monitoring" 能做的事情， "monitoring" 能做所有“management”能做的事情，因此通常只需要指定最大权限的那个tag给到用户即可。   

 Normal RabbitMQ permissions still apply to monitors and administrators; just because a user is a monitor or administrator does not give them full access to exchanges, queues and bindings through either AMQP or the management plugin.(ps://没有理解透这段话想表达啥意思？)

 所有的用户只能访问它拥有权限的那个vhost下的对象。

 如果由于没有管理员用户，甚至没有用户，而不能登录web ui，可以使用rabbitmqctl创建用户，并设置标签解决。

 ### HTTP API

 management插件提供了基于HTTP的API，地址为http://server-name:15672/api/. 浏览这个地址可以获取API的详细信息，或者更方便的访问在github的地址[latest HTTP API documentation](https://cdn.rawgit.com/rabbitmq/rabbitmq-management/v3.7.3/priv/www/api/index.html) 。  

这套API用于监控和告警目的。它提供了可以访问节点状态、连接、channel，队列、消费者等详细信息。  

在任意启用了rabbitmq-management插件的节点上能够使用API，然后它就可以提供任意（或者所有）集群节点的信息。当监控一个集群时，没有必要通过api单独的连接每个节点，连接任意一个节点即可（或者使用load balancer）。  

有些API端点返回大量的信息，可以通过“http get”请求时过滤那些是想要的列来减少返回的数据体积。 查看latest HTTP API documentation](https://cdn.rawgit.com/rabbitmq/rabbitmq-management/v3.7.3/priv/www/api/index.html)了解细节。  

rabbitmqadmin 是一个python命令行工具，通过api交互。  


### 配置
有几个配置选项会影响到management插件，他们通过rabbitmq主配置文件管理。查看[configuration file](http://www.rabbitmq.com/configure.html#configuration-file).

### HTTP请求日志
创建简单的请求访问日志，通过在rabbitmq_management应用中设置http_log_dir变量，指明日志目录，然后重启rabbitmq生效。注意，仅访问/api的请求会记录日志。

### 统计间隔  
默认，server将会每5秒发出一个统计事件。management插件展示的消息速率通过该周期来计算。因此你可能想要提高这个间隔时间，做一个长周期的速率计算，或者在一个非常多队列和channel的server上减少统计负载。  
在rabbit 应用中设置collect_statistics_interval变量为想要的毫秒数，重启rabbitmq生效。  

### 消息速率  
默认management插件显示全局的消息速率，以及每个队列，channel，交换机和vhost。 默认basic消息速率模式。  

也可以显示所有的channel到exchange，exchange到queue，queue到channel的组合的消息速率，使用detailed消息速率模式。默认detailed模式是禁用的，因为当有大量的channel，queues和exchanges组合时，该模式会导致大量的内存消耗。   

另一方面，可以完全禁用消息速率，这样可以在cpu受限的server上提高些性能。   

消息速率模式通过在rabbitmq_management中配置rates_mode参数，值为basic(默认)，detailed和none。  

### 启动时加载定义文件 
management插件可以导出包含所有定义对象（queues，exchanges，bindings，users，vhosts，permissions，patameters）的json格式的文件。 在某些场景，它可以确保在每次启动时所有的这些定义都会存在。   

设置management.load_definitions(经典配置格式下使用rabbitmq_management.load_definitions)为之前导出的json文件路径。 
```java
management.load_definitions = /path/to/definitions/file.json
```
或经典配置：
```java
[
  {rabbitmq_management, [
    {load_definitions, "/path/to/definitions/file.json"}
  ]}
].
```
注：在定义文件会重写broker上的对象，但不会删除已经存在的对象。尽管如此， 如果你启动一个完全的重置(reset)的broker， 此时将阻止常规默认的用户/vhost/permission的创建。   


### 事件backlog
在高负载下，处理统计事件会增加内存消息，可以配置channel和queue统计收集器的最大的backlog size来减少。在rabbitmq_management中设置stats_event_max_backlog变量，同时影响channel和queue的backlog size，默认该值为250.  

### 配置HTTP监听  
可以通过配置rabbitmq-web-dispatch使得management插件监听在不同的网卡或者端口上，同时配置SSL等。 例如配置监听端口
```java
management.listener.port = 12345
```
或经典格式：
```java
[
  {rabbitmq_management, [
    {listener, [{port, 12345}]}
  ]}
].
```
再例如配置使用https
```java
management.listener.port = 15671
management.listener.ssl = true
management.listener.ssl_opts.cacertfile = /path/to/cacert.pem
management.listener.ssl_opts.certfile = /path/to/cert.pem
management.listener.ssl_opts.keyfile = /path/to/key.pem
```
或经典格式:
```java
[{rabbitmq_management,
  [{listener, [{port,     15671},
               {ssl,      true},
               {ssl_opts, [{cacertfile, "/path/to/cacert.pem"},
                           {certfile,   "/path/to/cert.pem"},
                           {keyfile,    "/path/to/key.pem"}]}
              ]}
  ]}
].
```
查看 [rabbitmq-web-dispatch](http://www.rabbitmq.com/web-dispatch.html)获取详细信息。 

### 采样（样本）保留策略

management插件会保留比如消息速率和队列长度等数据样本。可以配置保留这些数据保留多长时间。
```java
management.sample_retention_policies.global.minute  = 5
management.sample_retention_policies.global.hour    = 60
management.sample_retention_policies.global.day = 1200

management.sample_retention_policies.basic.minute = 5
management.sample_retention_policies.basic.hour   = 60

management.sample_retention_policies.detailed.10 = 5
```
有三种策略：  
	global - 保留概览（overview）和vhost页数据时间
	basic - 保留连接，channel，exchange和queues时间
	detailed - how long to retain data for message rates between pairs of connections, channels, exchanges and queues (as shown under "Message rates breakdown")(ps://没理解到)

This configuration (which is the default) retains global data at a 5 second resolution (sampling happens every 5 seconds) for 10 minutes and 5 seconds, then at a 1 minute resolution for 1 hour and 1 minute, then at a 10 minute resolution for about 8 hours. It retains basic data at a 5 second resolution for 1 minute and 5 seconds, then at a 1 minute resolution for 1 hour, and detailed data only for 10 seconds. All three policies are mandatory, and must contain at least one retention pair {MaxAgeInSeconds, SampleEveryNSeconds}.  (ps:// 我的理解是采样数据的聚合，比如如果视图上是10分钟，那么采样点是每5秒一次；如果是1小时，采样点是1分钟；如果是8小时，采样点是10分钟。)

### CORS(cross-origin resource sharing) 跨域资源共享

management api默认不允许跨域访问,必须通过显式的白名单配置才能允许访问。比如：
```java
[
  {rabbitmq_management,
    [{cors_allow_origins, ["http://rabbitmq.com", "http://example.org"]}]},
].
```
可能够配置允许所有的域能够访问API，但是不建议外部网络可访问的情况下开启。 
```java
[
  {rabbitmq_management,
    [{cors_allow_origins, ["*"]}]},
].
```
CORS请求会被浏览器缓存，management插件默认定义的超时时间是30分，可以修改配置:
```java
[
  {rabbitmq_management,
    [{cors_allow_origins, ["http://rabbitmq.com", "http://example.org"]},
     {cors_max_age, 3600}]},
].
```

### 路径前缀

某些环境要求为所有http请求自定义的前缀。 management插件通过配置 path_prefix来实现。 设置path_prefix为 /my-prefix 后，所有的请求应该为 host:prot/my-prefix/api/[....] ， 同时登录页面url也会变为 host:prot/my-prefix/ (注意最后一个/是需要的) 
```java
[
  ...
  {rabbitmq_management,
    [{path_prefix, "/my-prefix"}]},
  ...
].
```

### 示例

```java
listeners.tcp.default = 5672

collect_statistics_interval = 10000

# management.load_definitions = /path/to/exported/definitions.json

management.listener.port = 15672
management.listener.ip   = 0.0.0.0
management.listener.ssl  = true

management.listener.ssl_opts.cacertfile = /path/to/cacert.pem
management.listener.ssl_opts.certfile   = /path/to/cert.pem
management.listener.ssl_opts.keyfile    = /path/to/key.pem

management.http_log_dir = /path/to/rabbit/logs/http

management.rates_mode = basic

# Configure how long aggregated data (such as message rates and queue
# lengths) is retained.
# Your can use 'minute', 'hour' and 'day' keys or integer key (in seconds)
management.sample_retention_policies.global.minute    = 5
management.sample_retention_policies.global.hour  = 60
management.sample_retention_policies.global.day = 1200

management.sample_retention_policies.basic.minute   = 5
management.sample_retention_policies.basic.hour = 60

management.sample_retention_policies.detailed.10 = 5
```
或经典格式：
```java
[
{rabbit, [{tcp_listeners,               [5672]},
          {collect_statistics_interval, 10000}]},

{rabbitmq_management,
  [
   %% Pre-Load schema definitions from the following JSON file.
   %%
   %% {load_definitions, "/path/to/definitions.json"},

   %% Log all requests to the management HTTP API to a directory.
   %%
   {http_log_dir, "/path/to/rabbit/logs/http"},

   %% Change the port on which the HTTP listener listens,
   %% specifying an interface for the HTTP server to bind to.
   %% Also set the listener to use TLS and provide TLS options.
   %%
   %% {listener, [{port,     15672},
   %%             {ip,       "0.0.0.0"},
   %%             {ssl,      true},
   %%             {ssl_opts, [{cacertfile, "/path/to/cacert.pem"},
   %%                         {certfile,   "/path/to/cert.pem"},
   %%                         {keyfile,    "/path/to/key.pem"}]}]},

   %% One of 'basic', 'detailed' or 'none'.
   {rates_mode, basic},

   %% increasing this parameter will make HTTP API cache data retrieved
   %% from other cluster peers more aggressively
   %% {management_db_cache_multiplier, 5},

   %% If event collection falls back behind stats emission,
   %% up to this many events will be kept in the backlog, the rest
   %% will be dropped to avoid runaway memory consumption growth.
   %% This setting is per-node. Unless there is evidence of
   %% a stats collector backlog, you don't need to change this value.
   %% {stats_event_max_backlog, 250},

   %% CORS settings for HTTP API
   %% {cors_allow_origins, ["https://rabbitmq.eng.megacorp.local", "https://monitoring.eng.megacorp.local"]},
   %% {cors_max_age, 1800},

   %% Configure how long aggregated data (such as message rates and queue
   %% lengths) is retained.
   %%
   %% {sample_retention_policies,
   %%  [{global,   [{60, 5}, {3600, 60}, {86400, 1200}]},
   %%   {basic,    [{60, 5}, {3600, 60}]},
   %%   {detailed, [{10, 5}]}]}
  ]}
].
```

### 内存使用分析
management ui可以检查节点的内存使用，查看[Memory Use Analysis](http://www.rabbitmq.com/memory-use.html)了解详情。 


### 集群注意

management插件能够感知集群。可以在集群中的一个或者多个节点启用插件，然后就可以知道整个集群的信息，不管是连接到哪个节点。   

如果部署的集群节点没有完全启用management插件，至少要启用rabbitmq-management-agent插件。 

当使用management插件执行集群范围的查询时，它会受到集群中的各种网络事件的影响，比如partitions。 

### 反向代理设置

It is possible to make the web UI available via any proxy that conforms with RFC 1738. The following sample Apache configuration illustrates the minimum necessary directives to coax Apache into conformance. It assumes a management web UI on the default port of 15672:
```java
AllowEncodedSlashes On
ProxyPass        /api http://localhost:15672/api nocanon
ProxyPass        /    http://localhost:15672/
ProxyPassReverse /    http://localhost:15672/
```

### 重启统计数据库

统计数据库是完全保存在内存中的，所有的内容是瞬时的。 在3.6.7之前，统计数据库保存在单个节点，从3.6.7开始，每个节点保存了自己节点记录的部分统计数据。可以重新开始统计数据。   

3.6.2之前使用
```java
rabbitmqctl eval 'exit(erlang:whereis(rabbit_mgmt_db), please_terminate).'
```
命令重启， 3.6.2到3.6.5 使用
```java
rabbitmqctl eval 'supervisor2:terminate_child(rabbit_mgmt_sup_sup, rabbit_mgmt_sup),
                  rabbit_mgmt_sup_sup:start_child().'
```
这些命令必须在拥有数据库的节点上执行。 而在3.6.7开始， 使用
```java
rabbitmqctl eval 'rabbit_mgmt_storage:reset().'
```
也可以重启所有节点整个management数据库， 
```java
rabbitmqctl eval 'rabbit_mgmt_storage:reset_all().'
```
可以使用HTTP API端点来重启整个数据库：
```java
delete /api/reset
```
或者单个节点:
```java
DELETE /api/reset/:node
```

### 内存管理

management的数据库使用的内存可以使用rabbitmqctl获取
```java
rabbitmqctl status
```
或者通过GET请求/api/nodes/name。 

统计数据库消耗的内存总量依赖于统计间隔时间、实际的速率模式 和 保留策略。

增加rabbit.collect_statistics_interval值到30-60s可以减少内存消耗。 调整保留策略来保留更少的数据也有一定帮助。

channel和queue的统计收集器占用的内存可以通过backlog队列长度来限制，使用stats_event_max_backlog参数。 如果backlog 队列满了， 那么新的channel和queue的统计数据会被丢弃，直到队列中前一个的被处理完成。 

统计间隔时间可以在运行时调整，但是不会影响到已经存在的connection，channel和queue。 只对新创建的对象有效。 
```java
rabbitmqctl eval 'application:set_env(rabbit, collect_statistics_interval, 60000).'
```
统计数据库可以重启，因此强制释放所有的内存。 