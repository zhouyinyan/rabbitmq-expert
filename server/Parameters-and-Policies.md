[原文链接](http://www.rabbitmq.com/parameters.html)

# 参数和策略(parameters and policies)

## 介绍

虽然绝大多数配置是在配置文件中，但有些配置却合适放在配置文件中，比如：  
  如果需要在集群中所有节点都需要一样的配置  
  如果需要在运行时修改配置  

 rabbitmq称这些为参数(parameters)。参数可以通过rabbitmqctl或者management插件的HTTP API设置。有两种类型参数: vhost范围参数和全局参数。 vhost范围参数和vhost绑定，由组件名称，一个名字和一个值 三部分组成。 全局参数不会绑定到特定的vhost，只有一个名字和一个值两部分组成。   

 参数的一种特殊用法是策略(policies), 用来为一组队列或者交换机指定参数，亦可为比如federation和shovel等插件指定参数。 策略是vhost范围的。  

 ## 全局和per-vhost参数  

 如前所述，存在vhost范围的参数和全局参数。 An example of vhost-scoped parameter is a federation upstream: it targets a component (federation-upstream), it has a name that identifies it, it's tied to a virtual host (federation links will target some resources of this virtual host), and its value defines connection parameters to an upstream broker.(PS://还没有看federation插件的文档，所以这个例子还不能完全理解)， vhost范围的参数可以被设置，清除和列出：

| rabbitmqctl | rabbitmqctl set_parameter {-p *vhost*} *component_name* *name* *value*<br />rabbitmqctl clear_parameter {-p *vhost*} *component_name* *name*<br />rabbitmqctl list_parameters {-p *vhost*} |
| ----------- | ---------------------------------------- |
| HTTP API    | PUT /api/parameters/*component_name*/*vhost*/*name*<br />DELETE /api/parameters/*component_name*/*vhost*/*name*<br />GET /api/parameters |

全局参数是另一种参数类型，比如一个集群名字的全局参数。全局参数可以被设置，清除和列出：

| rabbitmqctl | rabbitmqctl set_global_parameter *name* *value*<br />rabbitmqctl clear_global_parameter *name*<br />rabbitmqctl list_global_parameters |
| ----------- | ---------------------------------------- |
| HTTP API    | PUT /api/global-parameters/*name*<br />DELETE /api/global-parameters/*name*<br />GET /api/global-parameters |

因为参数值是一个json文档，使用rabbitmqctl在设置的时候需要引号包起来。   

参数同vhost、交换机、队列、绑定、权限等定义一样，保存在rabbitmq的内部数据库中，可以通过management插件的导出功能将参数随同其他定义一起导出到定义文件。   

federation和shovel插件使用vhost范围参数， mqtt插件使用全局参数。

## 策略

- 为什么需要策略？

在解释什么是策略，以及怎么使用它们之前，有必要说明下为什么rebbitmq要引进策略。   

在rabbitmq中，队列和交换机除了必选属性(比如durable或者exclusive)之外，还有可选的属性（参数）。这种方式有时通过在客户端声明队列（交换机）时指定x-arguments，控制各种选项特性，比如队列长度限制或者TTL。 

客户端控制属性的方式通常可以工作的很好，但也有限制：更新TTL值或者镜像参数时，需要修改应用代码，重新部署和重新声明队列；除此之外，也没有其他办法控制一组队列或者的交换机的扩展参数。 策略就是来解决这些问题。   

一条策略通过名字匹配一个或者多个队列(使用正则表达式)，并且策略定义(一个可选参数的map)附加到匹配上的队列的x-arguments上。 换句话说，可以通过策略一次性配置多个队列的x-arguments属性，并且通过更新策略定义来同时更新它们。  

在rabbitmq新版本中，策略能够控制的特性集并不和客户端可以控制的特性集完全相同。 

- 策略怎么工作？

核心策略属性为：
  名字（name）  
  模式(pattern) ：匹配一个或者多个队列(交换机)名字的正则表达式。  
  定义(definition) ：键值对的集合(想象json文档)，将会注入到匹配的队列和交换机的可选参数的map中。   
  优先级(priority): see blow

  policies automatically match against exchanges and queues, and help determine how they behave.(ps://怎么表达?)。 每个交换机或者队列最多和一条策略匹配(阅读后面的联合策略定)，然后每个策略注入一组键值对到匹配的队列(交换机).   

  策略可仅匹配队列，或者仅匹配交换机，或者都匹配。在创建策略时，可通过使用apply-to标记指定。   

  策略可以在任何时候修改。 当一个策略更新了，该策略匹配的队列和交换机会应用更新。 通常，这种更新立即生效，但对于非常繁忙的队列，可能会花上一点时间(几秒钟)。   

  每当一个交换机或者队列创建时，策略都会匹配和应用，而不仅是策略创建时。   

  策略可用来配置federation插件，镜像队列，alternate交换机，死信队列，TTLs和队列最大长度等。   

  策略定义示例:

| abbitmqctl            | `rabbitmqctl set_policy federate-me "^amq\." '{"federation-upstream-set":"all"}' --priority 1 --apply-to exchanges` |
| --------------------- | ---------------------------------------- |
| rabbitmqctl (Windows) | `rabbitmqctl.bat set_policy federate-me "^amq\." "{""federation-upstream-set"":""all""}" --priority 1 --apply-to exchanges` |
| HTTP API              | `PUT /api/policies/%2f/federate-me                    {"pattern": "^amq\.",                     "definition": {"federation-upstream-set":"all"},                     "priority": 1,                    "apply-to": "exchanges"}` |
| Web UI                | Navigate to Admin > Policies > Add / update a policy.Enter "federate-me" next to Name, "^amq\." next to Pattern, and select "Exchanges" next to Apply to.Enter "federation-upstream-set" = "all" in the first line next to Policy.Click Add policy. |


这条策略会设置所有在“/”的vhost中，以“amq.”开头的交换机的“federation-upstream-set”参数设置为“all”.   

"pattern"参数就是匹配队列或者交换机名字的正则表达式。   

如果多条策略匹配某个交换机或者队列， 将通过优先级来确定应用哪个策略。   

“apply-to”参数可以取值为“exchanges”，“queues”或者“all”. “apply-to”和“priority”是可选的， 默认是“all”和“0”。 

- 联合策略定义(combining policy definitions)

  某些场景下，我们可能想要应用多条策略定义到一个资源上。 比如我们需要一个队列既是federated，又是mirrored。 虽然任何时候只能有一条策略应用到资源上，但是我们可以在一条策略中使用多条策略定义。 

  一个federation策略定义需要指定 upstream set，因此需要在策略定义中包含“federation-upstream-set”这个key， 另一方面需要定义队列是镜像的，还需要在策略定义中包含“ha-mode”这个key。 因为策略定义就是json对象，所以我们很容易在一条策略中包含两个key。 示例如下：

  | rabbitmqctl           | `rabbitmqctl set_policy ha-fed "^hf\." '{"federation-upstream-set":"all","ha-mode":"all"}' \--priority 1 --apply-to queues` |
  | --------------------- | ---------------------------------------- |
  | rabbitmqctl (Windows) | `rabbitmqctl set_policy ha-fed "^hf\." "{""federation-upstream-set"":""all"", ""ha-mode"":""all""}" ^--priority 1 --apply-to queues` |
  | HTTP API              | `PUT /api/policies/%2f/ha-fed{"pattern": "^hf\.", "definition": {"federation-upstream-set":"all", "ha-mode": "all"}, "priority": 1, "apply-to": "queues"}` |
  | Web UI                | Navigate to Admin > Policies > Add / update a policy.Enter "ha-fed" next to Name, "^hf\." next to Pattern, and select "Queues" next to Apply to.Enter "federation-upstream-set" = "all" in the first line next to Policy.Enter "ha-mode" = "all" on the next line.Click Add policy. |

- Operator Policies(ps://没搞懂)

  - 和常规策略的区别  
    Sometimes it is necessary for the operator to enforce certain policies. For example, it may be desirable to force queue TTL but still let other users manage policies. Operator policies allow for that.  
    Operator policies are much like regular ones but their definitions are used differently. They are merged with regular policy definitions before the result is applied to matching queues.  
    Because operator policies can unexpectedly change queue attributes and, in turn, application assumptions and semantics, they are limited only to a few arguments:  
    expires
    message-ttl
    max-length
    max-length-bytes  
    The arguments above are all numeric. Thie reason for that is explained in the following section

  - 和常规策略的冲突解决  
    An operator policy and a regular one can contain the same keys in their definitions. When it happens, the smaller value is chosen as effective. For example, if a matching operator policy definition sets max-length to 50 and a matching regular policy definition uses the value of 100, the value of 50 will be used. If, however, regular policy's value was 20, it would be used. Operator policies, therefore, don't just overwrite regular policy values. They enforce limits but try to not override user-provided policies where possible.  

  - 定义operator 策略 
     Operator policies defined in a way very similar to regular (user) policies. When rabbitmqctl is used, the command name is set_operator_policy instead of set_policy. In the HTTP API, /api/policies/ in request path becomes /api/operator-policies/:  


| rabbitmqctl           | `rabbitmqctl set_operator_policy transient-queue-ttl "^amq\." '{"expires":1800000}' --priority 1 --apply-to queues` |
| --------------------- | ---------------------------------------- |
| rabbitmqctl (Windows) | `rabbitmqctl.bat set_operator_policy transient-queue-ttl "^amq\." "{""expires"": 1800000}" --priority 1 --apply-to queues` |
| HTTP API              | `PUT /api/operator-policies/%2f/transient-queue-ttl                        {"pattern": "^amq\.",                         "definition": {"expires": 1800000},                         "priority": 1,                         "apply-to": "queues"}` |
| Web UI                | Navigate to Admin > Policies > Add / update an operator policy.Enter "transient-queue-ttl" next to Name, "^amq\." next to Pattern, and select "Queues" next to Apply to.Enter "expires" = 1800000 in the first line next to Policy.Click Add policy. |