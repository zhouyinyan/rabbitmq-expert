[原文链接](http://www.rabbitmq.com/management-cli.html)

# rabbitmqadmin

management插件自带一个命令行工具 rabbitmqadmin， 它能够执行和web-base ui差不多同样的事情，并且它可能在自动化任务时更加方便。 rabbitmqadmin仅仅是个HTTP 客户端，在应用程序中应该使用HTTP API而不是去调用rabbitmqadmin。  

### 获得rabbitmqadmin

浏览http://{hostname}:15672/cli/rabbitmqadmin 下载，或从github下载。 

### 入门

调用 rabbitmqadmin --help 查看用户指引，能够:
	列出exchanges，queues，bindings，vhosts，users，permissions， connections 和channels
	展示概览信息  
	声明和删除exchanges，queues，bindings，vhosts， users permissions
	发布和接受消息
	关闭连接和清空队列
	导入和导出配置  

### 示例
获取exchanges列表
```java
rabbitmqadmin -v test list exchanges
```
获取队列列表，并指定返回的列：
```java
rabbitmqadmin list queues vhost name node messages messages_stats.publish_details.rate
```
获取队列列表，并返回所有细节
```java
rabbitmqadmin -f long -d 3 list queues
```
使用其他用户连接到其他主机
```java
rabbitmqadmin -H server -u user -p passwd list vhosts
```

声明exchange::
```java
rabbitmqadmin declare exchange name=my-new-exchange type=fanout
```
声明queue,并指定参数:
```java
rabbitmqadmin declare queue name=my-new-queue durable=false
```
发布消息:
```java
rabbitmqadmin publish exchange=amq.default routing_key=test payload="hello, world"
```

获取消息：
```java
rabbitmqadmin get queue=test requeue=false
```
导出配置：
```java
rabbitmqadmin export rabbit.definitions.json
```
导入配置
```java
rabbitmqadmin -q import rabbit.definitions.json
```
关闭所有连接：
```java
rabbitmqadmin -f tsv -q list connections name | while read conn ; do rabbitmqadmin -q close connection name="${conn}" ; done
```
