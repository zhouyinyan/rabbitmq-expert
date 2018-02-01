[原文链接](http://www.rabbitmq.com/plugins.html) 

# 插件(plugins)  

rabbitmq支持很多插件。

启用插件，使用rabbitmq-plugins工具:
```java
rabbitmq-plugins enable *plugin-name*
```
禁用插件：
```java
rabbitmq-plugins disable *plugin-name*
```
列出哪些插件启用：
```java
rabbitmq-plugins list
```

rabbitmq-plugins 工具可以启用和禁用插件，通过更新插件配置文件，同时它会联系运行中的server，告诉它根据要求启动和停止插件。可以使用-n来指定不同的节点，使用--offline来仅修改配置文件，而不立即生效。  更多关于rabbitmq-plugins工具的信息参考[the manual page](http://www.rabbitmq.com/man/rabbitmq-plugins.8.html).

支持的插件列表：

| rabbitmq_auth_backend_ldap        |
| --------------------------------- |
| rabbitmq_auth_backend_http        |
| rabbitmq_auth_mechanism_ssl       |
| rabbitmq_consistent_hash_exchange |
| rabbitmq_federation               |
| rabbitmq_federation_management    |
| rabbitmq_management               |
| rabbitmq_management_agent         |
| rabbitmq_mqtt                     |
| rabbitmq_shovel                   |
| rabbitmq_shovel_management        |
| rabbitmq_stomp                    |
| rabbitmq_tracing                  |
| rabbitmq_trust_store              |
| rabbitmq_web_stomp                |
| rabbitmq_web_mqtt                 |
| rabbitmq_web_stomp_examples       |
| rabbitmq_web_mqtt_examples        |





