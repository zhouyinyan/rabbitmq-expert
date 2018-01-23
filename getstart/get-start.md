[原文链接](http://www.rabbitmq.com/getstarted.html)

//TODO ack 翻译又*确认*修改为**回执**

ps:使用java客户端，如果使用其他客户端，参考原文链接。

准备：需要安装rabbitmq，并且运行在本机（localhost）的默认端口5672上，如果主机和端口不一样，需要调整连接配置。

添加maven依赖：
```java
<dependency>
  <groupId>com.rabbitmq</groupId>
  <artifactId>amqp-client</artifactId>
  <version>5.1.1</version>
</dependency>
```
说明：
  producer 使用下面图表标表示：  
   ![](../images/producer.png)  
  queue 使用下面图标表示：  
   ![](../images/queue.png)  
  consumer 使用下面图标表示：  
   ![](../images/consumer.png)

- "hello world"  
  在这部分，我们要写两个java程序；一个生产者发送消息，一个消费者接受消息，并打印。不会讨论java api的细节，这是最简单的起步例子。拓扑图：  
  ![](../images/java-helloworld.png)

  - 发送
  ```java
  public class Send {

    private final static String QUEUE_NAME = "hello";  //队列名称

    public static void main(String[] argv) throws Exception {
    
      /*
      *  连接到rabbitmq。
      *  Connection是socket连接的抽象，关注协议版本协商，验证等工作。这儿我们连接到本机，如果要连接到其他rabbitmq，修改host属性即可。
      *  接下来创建channel，channel是客户端中操作最多的Api
      */
      ConnectionFactory factory = new ConnectionFactory();
      factory.setHost("localhost");
      Connection connection = factory.newConnection();
      Channel channel = connection.createChannel();

    /*
    * 定义队列，(队列会自动绑定到默认交换机""上)，然后发送消息。
    */
      channel.queueDeclare(QUEUE_NAME, false, false, false, null);
      String message = "Hello World!";
      channel.basicPublish("", QUEUE_NAME, null, message.getBytes("UTF-8"));
      System.out.println(" [x] Sent '" + message + "'");

      /*
      * 最后关闭channel和connection
      */
      channel.close();
      connection.close();
    }
  }
  ```

  - 接受
  ```java
  public class Recv {

    private final static String QUEUE_NAME = "hello";  //队列名称

    public static void main(String[] argv) throws Exception {
        /**
         * 同生产者一样， 先创建链接，获取channel
         */
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        /**
         * 定义队列，该队列会被接受者消费
         * 要注意的是，定义队列的参数要和生产者一样，否则会rabbitmq会返回错误。
         */
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        /**
         * 消息回调.
         */
        Consumer consumer = new DefaultConsumer(channel){

            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + message + "'");
            }
        };

        /**
         * 订阅队列
         */
        channel.basicConsume(QUEUE_NAME, true, consumer);

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    }
  }

  ```

当运行着两个小程序后， 生产者程序会成功投递消息到rabbitmq，消费者程序会获取到消息并打印到控制台。消费者程序会一直运行着，除非使用Ctrl-c停止。可以同时运行多个生产者和多个消费者，并观察消息消费情况(比如2个生产者，3个消费者)

我们可以通过使用rabbitmqctl管理命令来查看服务器中有哪些队列 ` rabbitmqctl list_queues `


- work queue

在第一个例子中我们写了两个小程序通过命名的队列来发送和接受消息。 在这个例子中，我们会创建一个"工作队列"使得多个分布耗时的工作者来消费工作(消息)，示例图：
![](../images/java-work-queues-1.png)  

工作队列(也叫任务队列）的核心思想是，避免等待立即处理资源密集型任务带来的耗时，而是通过安排任务在后台异步处理完成。我们封装任务作为一个消息发送到队列中，工作者程序会在后台取出消息并最终执行任务，当运行多个工作者程序时，这些工作者会共享任务(//原文是share，感觉好别扭)

生产者：
```
public class NewTesk {

    private final static String QUEUE_NAME = "work-queue";  //队列名称

    public static void main(String[] args) throws IOException, TimeoutException {

        /**
         * 获取连接，创建channel
         */
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        /**
         * 根据命令行参数封装消息
         */
        String message = getMessage(args);
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());

        System.out.println(" [x] send '" + message + "'"  );

        channel.close();
        connection.close();

    }

    private static String getMessage(String[] args) {
        if(args.length < 1){
            return "hello.."; //需要处理两秒
        }
        return joinStrings(args, " ");
    }

    private static String joinStrings(String[] strings, String delimiter) {
        int length = strings.length;
        if (length == 0) return "";
        StringBuilder words = new StringBuilder(strings[0]);
        for (int i = 1; i < length; i++) {
            words.append(delimiter).append(strings[i]);
        }
        return words.toString();
    }
}
```

工作者
```java
public class Worker {

    private final static String QUEUE_NAME = "work-queue";  //队列名称

    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        final Consumer consumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);

                String message = new String(body, "UTF-8");

                System.out.println(" [x] Received '" + message + "'");

                try {
                    //模拟耗时操作
                    doWork(message);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    System.out.println(" [x] Done");
                }

            }
        };

        /**
         * 自动ACK（ack的作用在后续会介绍）
         */
        boolean autoAck = true;
        channel.basicConsume(QUEUE_NAME, autoAck, consumer);

        System.out.println(" [*] Waiting for task. To exit press CTRL+C");

    }

    private static void doWork(String task) throws InterruptedException {
        for (char ch: task.toCharArray()) {
            if (ch == '.') Thread.sleep(1000);
        }
    }

}
```


**轮询分发**  
使用任务队列最大的好处之一就是工作者可以并行处理任务。如果任务队列积压，我们很容易通过添加更多的工作者来增强处理能力，这种方式成为水平扩展。  
我们可以启动多个工作者程序来观察轮询效果。默认rabbitmq会发送消息给按顺序下一个消费者，每个消费者会获取到平均数量（消息总量/消费者数量）的消息。  

**消息确认（message acknowledgment）**  
完成一个任务可能要花费数秒钟，在这个过程中，你不知道消费者是开启一个长时间任务还是处理一半任务就挂了。上面的例子中，一单rabbitmq投递一个消息给消费者后，它立即就将该消息删除。这样一来，如果你杀死一个工作者将会丢掉它正在处理的消息，也会丢掉将要派发给这个工作者的所有消息（//说法应该有点问题，杀死一个消息之后，连接断了就不会再派发了），但是，这些消息是没有处理过的。  

当然，我们不想丢掉任何未处理的任务，如果一个工作者挂了，我们希望消息会派发给其他的正常工作者。  

为了确保消息不被丢失，rabbitmq支持”**消息确认**“， 消费者发送一个ack回给rabbitmq来告诉它，一个具体的消息被接受了，并且被处理完成，rabbitmq可以安全的删除它。  

如果一个消费者挂了(它的channel关闭了， connection关闭了， 或者TCP连接断了)，并且没有返回ack给rabbitmq， rabbitmq就会明白消息没有完全处理成功，并将该消息重入队列。如果此时有其他消费者在线，rabbitmq会重新投递该消息给其他消费者。这种机制可以确保消息不会因为消费者挂了而丢失。  

消息没有超时时间，rabbitmq仅仅在消费者挂了之后才会重派消息，所以就算一个消息被消费者处理非常长的时间也没有关系。   

默认是开启的手动消息确认，上面的例子中我们显示的通过`auto-ack=true`来关闭手动确认消息。我们来修改成手动方式：  
```java
public class Worker {

    private final static String QUEUE_NAME = "work-queue";  //队列名称

    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        channel.basicQos(1); //消息预取（后面会说明），简单来说表示在没有ack之前可以预取多少条消息
        final Consumer consumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);

                String message = new String(body, "UTF-8");

                System.out.println(" [x] Received '" + message + "'");

                try {
                    //模拟耗时操作
                    doWork(message);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    System.out.println(" [x] Done");
                    //消息处理完成后，手动ack给rabbitmq
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }

            }
        };

        /**
         * 自动ACK关闭，开启手动ack
         */
        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME, autoAck, consumer);

        System.out.println(" [*] Waiting for task. To exit press CTRL+C");

    }

    private static void doWork(String task) throws InterruptedException {
        for (char ch: task.toCharArray()) {
            if (ch == '.') Thread.sleep(1000);
        }
    }

}
```

消费者使用手动确认可以确保就算你使用ctrl-c来杀掉工作者，该工作者正在处理的消息也不会丢（不信你大可试试）。

**忘记ack（手动确认）**  
一种常见的错误就是忘记ack，这将会导致非常严重的后果。消息会不断的重新派发，不断的堆积，rabbitmq会吃掉越来越多的内存，最终rabbitmq宕机。可以使用`rabbitmqctl list_queues messages_unacknowledged`来查看未ack的消息数量。

**消息持久化**  
上面我们聊了在消费者挂了的情况下，如果确保任务不丢。但是如果rabbitmq服务挂了，任务还是会丢掉。  
当rabbitmq server崩溃后，它会忘记之前的队列和消息，除非你告诉他别忘记。需要做两件事来确保消息不丢：我们需要对队列和消息都持久化。

首先，来定义持久化话队列
```java
boolean durable = true;
channel.queueDeclare("hello", durable, false, false, null);
```
注意，如果已经存在hello这个名字的队列（如果你按照上面的步骤来做的实验，那么hello队列是存在的），这个定义队列的命令会出现异常。这是因为rabbitmq不允许你使用不同的参数来重新定义一个存在的队列，变通一些，换个名字定义吧
```java
boolean durable = true;
channel.queueDeclare("task_queue", durable, false, false, null);
```
到此可以确保这个“task_queue”的队列在rabbitmq server重启后不丢。现在需要让消息持久化，通过设置MessageProperties的值为PERSISTENT_TEXT_PLAIN
```java
channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

消息持久化注意点：  
使消息持久化并不能完全100%的保证消息不丢，虽然rabbitmq serve会保存消息到磁盘，但当rabbitmq已经接收到消息，还未保存到磁盘中间，存在一个非常短的时间窗口会丢消息。rabbitmq不会为每个消息使用`fsync(2)`刷盘，它可能先保存到缓存中。这种持久化保证不是最强，如果需要更强的，请使用发布确认(publisher comfirms)

**公平派发**  
你可能注意到默认的派发方式不是我们想要的，比如，有两个工作者，当所有的奇数消息很重，所有的偶数消息很轻，将会到知道一个工作忙死，另一个工作闲死。  
为了防止这种情况，我们可以使用basicQos方法，设置prefetchCount = 1。该方法告诉rabbitmq不要同时给同一个工作者超过一个的消息，或者说，在一个工作者处理完上一个消息并ack之前，不要给该工作者派发消息。
```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

注意：如果所有工作者都繁忙，队列就会堆积，你应该时刻监控队列长度，并在队列堆积是做出相应的处理（比如增加工作者，或者其他措施）。

- publish/subscribe

上一个示例中我们创建了工作队列，每个任务只投递给某一个工作者。在本示例中，我们会投递消息给多个消费者，即发布-订阅模式。  

为了说明发布-订阅模式，我们将要构建一个简单的日志系统，其包含两个程序，一个发出日志消息，一个接受消息。 在我们的日志系统中，每个运行中的接受者程序都会受到消息。我们将运行一个接受者用来将日志记录到磁盘，运行另一个接受者将日志打印到屏幕上。  

**交换机（exchanges）**  
在前面示例中，我们发送和接收看起来都是直接和队列打交道。现在，是时候介绍一下rabbitmq 的完整模型了，让我们快速的回顾一下前两个示例中的概念：  
	*生产者*是发送消息的用户应用程序
	*队列*是存储消息的缓冲区
	*消费者*是接收消息的用户应用程序  

rabbitmq中核心的概念是 生产者 绝不会直接发送消息到队列。事实上，生产者并不知道消息是否被投递到哪个队列中。 相反，生产者只能发送消息给交换机。交换机非常简单，她的一头从生产者接收消息，另一头推送消息给队列。交换机当接收到消息之后必须知道如何处理，是否应该将该消息附加到一个特定队列？是否应该讲该消息附加到多个队列？或者是否应该丢弃？这些规则通过交换机类型定义。交换机示意：
![](../images/exchanges.png)
有几种交换机类型：直接(direct)，主题(topic)， 头(header)和散出(fanout)。本示例我们使用最后一种，即fanout。让我们来定义一个叫logs的fanout类型交换机：
```java
channel.exchangeDeclare("logs", "fanout");
```
fanout交换机非常简单，通过名字你可以猜出，它仅仅广播所有它收到的消息给它知道的所有队列。

注：**列出交换机**  
可以使用`rabbitmqctl list_exchanges`列出交换机列表。默认会存在amq.*和默认交换机(没名字或者名字是空字符串),这些是默认创建的，现在还不是用他们的时候。 

在之前我们根本都不知道交换机这个东东，但是我们同样能够发送消息，那是因为我们是使用的默认交换机，并且使用空字符串标识。看看之前发送消息的代码`channel.basicPublish("", "hello", null, message.getBytes());`, 第一个参数就是交换机名字，空字符串表示发送给默认交换机(默认交换机有个非常特殊的地方：所有队列定义后，都会使用队列名字作为路由键绑定到默认交换机)。

现在，我们通过交换机名字来发送消息给指定交换机：
```java
channel.basicPublish( "logs", "", null, message.getBytes());
```

**临时队列（temporary queues）**  
还记得之前我们使用队列时是指定了队列名字的(hello队列和task_queue队列)。当需要在生产者和消费者之间共享队列时，给队列命名还是很重要的。  
但是对我们的日志系统来说，不需要给队列命名。我们需要收集所有的日志，而不是一部分。我们关注于当前最新的日志消息流而不是旧的。  
首先，无论何时当我们连接到rabbitmq server时，我们需要一个全新的，空的队列。我们可以使用随机的名字来创建队列，甚至让rabbitmq server来生成随机的队列名字。  
其次， 一旦消费者断开连接，队列应该自动被删除。  
在java客户端中，当我们使用无参的queueDeclare()方法，我们就创建了一个非持久化、排他的、自动删除的、随机名字的队列。  
```java
String queueName = channel.queueDeclare().getQueue();
```
`queueName`就是rabbitmq server为我们随机生成的队列名字，看起来像`amq.gen-JzTY20BRgKO-HjmUJj0wLg`  

**绑定（bindings）**
![](../images/bindings.png)
我们已经创建了一个fanout交换机和一个临时队列，现在我们需要告诉交换机路由消息给队列。交换机和队列之间的关系称之为绑定。  
```java
channel.queueBind(queueName, "logs", "");
```
注：使用`rabbitmqctl list_bindings`列出绑定。

发送日志程序:
```java
public class Emitlog {

    private final static String EXCHANGE_NAME = "logs";

    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT);
        /**
         * 如果没有队列绑定到交换机，那么发往该交换机的消息就会被丢弃。
         * 对于该日志系统来说，定义好了交换机就可以发送消息了。我们的场景下日志消息可以没有消费者消费，消息可以丢。
         */
        sendMessage(channel);

        channel.close();
        connection.close();


    }

    private static void sendMessage(final Channel channel) throws IOException, InterruptedException {

        for (int i=0; i< 30; i++) {
            String loglevel = i % 2 == 0 ? "info" : "warn" ;
            String message = "[" + loglevel + "] hello - " + i;
            channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
            System.out.println(" [x] Sent '" + message + "'");
            TimeUnit.SECONDS.sleep(1);
        }
    }
}
```
日志接收者：
```java
public class ReceiveLogs {
    private final static String EXCHANGE_NAME = "logs";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT);

        String queueName = channel.queueDeclare().getQueue();
        channel.queueBind(queueName, EXCHANGE_NAME, "");

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);

                String message = new String(body, "UTF-8");

                channel.basicAck(envelope.getDeliveryTag(), false);

                System.out.println(" [x] Received '" + message + "'");
            }
        };

        channel.basicConsume(queueName, false, consumer);

    }
}
```
启动多个日志接收者程序，可以观察到，日志发布的所有消息，所有日志接受者都会收到。

- routing

在上一个示例中，我们构建了一个简单的日志系统，我们能够广播日志消息给多个接收者。  
在本示例中，我们将要订阅一部分消息，而不是全部。比如我们需要指定仅仅是严重级别的错误消息写到日志文件，同时也能够打印所有日志消息到控制台。  

**直接交换机（direct exchange）**  
直接交换机的路由算法非常简单 - 一个消息的路由键严格匹配队列绑定到交换机的绑定键(很多材料也叫路由键，为了区别消息的路由键和队列绑定到交换机的路由键，我们称为绑定键）。如下图：
![](../images/direct-exchange.png)
我们可以看到直接交换机x有两个队列绑定到它上面，第一个队列使用绑定键 orange，第二个有两个绑定，绑定键分别是black和green。 一个消息发送给X交换机， 如果路由键是orange，它将会路由到Q1中，如果路由键是black或者green， 将会路由到Q2中，其他消息将会丢弃。  

**多重绑定（multiple bindings）**  
![](../images/direct-exchange-multiple.png)
使用相同的绑定键绑定多个队列到同一个交换机也是OK的。比如上图中的例子。

日志发布者：
```java
public class Emitlog {

    private final static String EXCHANGE_NAME = "direct_logs";
    public final static String LOG_LEVEL_ERROR = "ERROR";
    public final static String LOG_LEVEL_WARN = "WARN";
    public final static String LOG_LEVEL_INFO = "INFO";

    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);

        sendMessage(channel);

        channel.close();
        connection.close();


    }

    private static void sendMessage(final Channel channel) throws IOException, InterruptedException {

        for (int i=0; i< 30; i++) {
            String loglevel = (i % 3 == 0 ? LOG_LEVEL_ERROR : (i % 3 == 1 ? LOG_LEVEL_WARN : LOG_LEVEL_INFO));
            String message = "[" + loglevel + "] hello - " + i;
            channel.basicPublish(EXCHANGE_NAME, loglevel, null, message.getBytes());
            System.out.println(" [x] Sent '" + message + "'");
            TimeUnit.SECONDS.sleep(1);
        }
    }
}
```
日志接收者:
```java
public class ReceiveErrorLogs {
    private final static String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);

        String queueName = channel.queueDeclare().getQueue();
        //使用绑定键ERROR绑定
        channel.queueBind(queueName, EXCHANGE_NAME, Emitlog.LOG_LEVEL_ERROR);

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);

                String message = new String(body, "UTF-8");

                channel.basicAck(envelope.getDeliveryTag(), false);

                System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
            }
        };

        channel.basicConsume(queueName, false, consumer);

    }
}
```
```java
public class ReceiveWarnAndInfoLogs {
    private final static String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);

        String queueName = channel.queueDeclare().getQueue();
        //使用绑定键WARN和INFO绑定
        channel.queueBind(queueName, EXCHANGE_NAME, Emitlog.LOG_LEVEL_WARN);
        channel.queueBind(queueName, EXCHANGE_NAME, Emitlog.LOG_LEVEL_INFO);

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);

                String message = new String(body, "UTF-8");

                channel.basicAck(envelope.getDeliveryTag(), false);

                System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
            }
        };

        channel.basicConsume(queueName, false, consumer);

    }
}
```


- topic

在上一个例子中我们改进了日志系统，使用direct交换机代替fanout交换机，但是direct交换机还是存在限制-它不能基于多个条件理由消息。  
我们希望在日志系统中不仅仅只根据日志的严重级别来订阅消息，还希望根据日志的发出源来订阅。为了实现这个目标，我们需要用到更复杂的topic交换机。  
**topic exchange**  
![](../images/topic-exchange.png)  
*(star)-匹配任意一个单词  
#(hash)-匹配任意0个或者多个单词  

在上图的例子中，我们通过发送描述所有动物的消息给交换机，消息路由键将会由三部分组成，使用“.“分割。第一个词描述速度，第二个描述颜色，第三个描述种类， "<speed>.<colour>.<species>".  

我们创建了3个绑定：Q1使用绑定键”*orange*“， Q2使用绑定键”*.*.rabbit“ 和 ”lazy.#“ 绑定到topic交换机上。这三个绑定可以概述为：  
	Q1对所有的orange颜色的动物感兴趣
	Q2对所有的兔子以及所有的速度慢的动物感兴趣。  
使用路由键为”quick.orange.rabbit“的消息将会投递给Q1和Q2，路由键为"lazy.orange.elephant"也会投递给Q1和Q2，”quick.orange.fox“只会投递给Q1， ”lazy.brown.fox“只会投递给Q2， ”lazy.pink.rabbit“虽然匹配Q2的两个绑定键，但是只会投递一次， ”quick.brown.fox“不匹配任何绑定，所以它会被丢弃。  如果路由键不是3个词，比如"orange" 或者 "quick.orange.male.rabbit"， 因为不匹配任何绑定，所以会丢弃。另一方面，路由键”lazy.orange.male.rabbit“尽管有4个词，但是匹配”lazy.#“这个绑定键，所有会投递给Q2.  

日志发送者：
```java
public class EmitlogTopic {

    private final static String EXCHANGE_NAME = "topic_logs";
    public final static String LOG_LEVEL_ERROR = "ERROR";
    public final static String LOG_LEVEL_WARN = "WARN";
    public final static String LOG_LEVEL_INFO = "INFO";
    public final static String LOG_SOURCE_CORE = "CORE";
    public final static String LOG_SOURCE_USER = "USER";

    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC);

        sendMessage(channel);

        channel.close();
        connection.close();


    }

    private static void sendMessage(final Channel channel) throws IOException, InterruptedException {

        for (int i=0; i< 30; i++) {
            String loglevel = (i % 3 == 0 ? LOG_LEVEL_ERROR : (i % 3 == 1 ? LOG_LEVEL_WARN : LOG_LEVEL_INFO));
            String source  = i % 2 == 0 ? LOG_SOURCE_CORE : LOG_SOURCE_USER ;
            String routingkey = loglevel + "." + source;
            String message = "[" + routingkey + "] hello - " + i;
            channel.basicPublish(EXCHANGE_NAME, routingkey, null, message.getBytes());
            System.out.println(" [x] Sent '" + message + "'");
            TimeUnit.SECONDS.sleep(1);
        }
    }
}
```

日志接收者：
```java
public class ReceiveLogsTopic {
    private final static String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC);

        String queueName = channel.queueDeclare().getQueue();
        String bingdingkey = EmitlogTopic.LOG_LEVEL_ERROR + ".*" ; //ERROR.*,关注所有源中错误的消息
        channel.queueBind(queueName, EXCHANGE_NAME, bingdingkey);

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);

                String message = new String(body, "UTF-8");

                channel.basicAck(envelope.getDeliveryTag(), false);

                System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
            }
        };

        channel.basicConsume(queueName, false, consumer);

    }
}
```
只写了一个接收所有错误消息的，你可以继续接收你想要的消息。

- rpc

在本示例中，我们使用rabbitmq来构建一个RPC系统：一个client和一个可水平扩展的rpc srver。

**客户端接口**  
为了说明怎么使用RPC服务，我们创建一个简单的client类，它暴露一个call方法来，这个方法用来发送rpc请求，并阻塞到返回结果：
```java
FibonacciRpcClient fibonacciRpc = new FibonacciRpcClient();
String result = fibonacciRpc.call("4");
System.out.println( "fib(4) is " + result);
```
**回调队列（callback queue)**  
rabbitmq实现rpc非常容易，客户端发送消息到一个队列，服务端消费消息并返回结果到另一个队列(回调队列)，客户单消费回调队列中的结果消息。
```java
callbackQueueName = channel.queueDeclare().getQueue();

BasicProperties props = new BasicProperties
                            .Builder()
                            .replyTo(callbackQueueName)
                            .build();

channel.basicPublish("", "rpc_queue", props, message.getBytes());
```
**消息属性（message properties）**  
AMQP 0-9-1协议定义了14个属性，大部分属性极少使用，但是下面几个属性用得较多：  
deliveryMode ： 投递模式，如果为2表示持久化消息，其他值为临时消息。  
contentType ：mime-type。  
replyTo ： 用于回调队列名称
correlationId ： 用于关联RPC的响应和请求。  

**关联ID（Correlation Id）**  
当回调队列中有多个响应消息时，如果对应每个响应消息到相应的请求上，这就是关联ID的作用。客户端为每个RPC请求生成唯一的id， 然后，当从回调队列中收到响应消息时，检查消息属性中的关联id,同请求的ID匹配就知道这个响应消息是哪个请求的了。当然如果是一个不清楚的关联ID，仅仅丢弃消息即可（这是因为server可能收到请求后，处理并返回响应消息给到了回调队列，但是还没有来得及ack就挂了，这种情况下rabbitmq server会重发消息给另外的server，所以同一个请求可能被处理两次，client会收到两个响应消息，所以client和server都需要幂等）。
![](../images/java-rpc.png)

当client启动后，它会创建一个排他的callback queue；构建一个rpc请求消息时，客户端使用两个消息属性：replyTo(设置回调队列名字)和correlationId(每个请求都是唯一值)。客户端将请求消息发送给 rpc_queue 队列。  
rpc server消费 rpc_queue 队列中的请求消息，当收到消息后， server处理请求，并发送响应消息给replyTo指定的回调队列。  
client收到响应队列后，它确认correlationId属性，如果匹配请求的ID，这返回结果给上层应用。  

RPCServer:
```java
public class RPCServer {

    private static final String RPC_QUEUE_NAME = "rpc_queue";

    public static void main(String[] args) throws IOException, TimeoutException {

        System.out.println("===="+fib(30));

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        final Channel channel = connection.createChannel();

        channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);

        channel.basicQos(1);

        System.out.println(" [x] Awaiting RPC requests");


        Consumer consumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                super.handleDelivery(consumerTag, envelope, properties, body);

                String response =  "";
                try {
                    String message = new String(body,"UTF-8");
                    int n = Integer.parseInt(message);

                    System.out.println(" [.] fib(" + message + ")");

                    response += fib(n);
                } finally {
                    /**
                     * 构建消息属性
                     */
                    AMQP.BasicProperties replyProps = new AMQP.BasicProperties.Builder()
                                                            .correlationId(properties.getCorrelationId())
                                                            .build();

                    /**
                     * 发送响应消息到回调队列
                     */
                    channel.basicPublish("", properties.getReplyTo(), replyProps, response.getBytes("UTF-8"));

                    //ack
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }


            }
        };

        channel.basicConsume(RPC_QUEUE_NAME, false, consumer);

    }

    private static int fib(int n) {
        if (n == 0) return 0;
        if (n == 1) return 1;
        return fib(n-1) + fib(n-2);
    }
    
}
```

RPCClient:
```java
public class RPCClient {

    private Connection connection;
    private Channel channel;
    private String requestQueueName = "rpc_queue";
    private String replyQueueName;

    public RPCClient() throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        connection = factory.newConnection();
        channel = connection.createChannel();

        //声明回调队列
        replyQueueName = channel.queueDeclare().getQueue();
    }

    public String call(String message) throws IOException, InterruptedException {
        String corrId = UUID.randomUUID().toString();


        AMQP.BasicProperties props = new AMQP.BasicProperties
                .Builder()
                .correlationId(corrId)  //关联ID
                .replyTo(replyQueueName) //回调队列
                .build();

        channel.basicPublish("", requestQueueName, props, message.getBytes("UTF-8"));

        final BlockingQueue<String> response = new ArrayBlockingQueue<String>(1);

        channel.basicConsume(replyQueueName, true, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                if (properties.getCorrelationId().equals(corrId)) {
                    response.offer(new String(body, "UTF-8"));
                }
            }
        });

        return response.take();
    }

    public void close() throws IOException {
        connection.close();
    }

}
```

ClientTest:
```java
public class RPCClientTest {

    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        RPCClient client = new RPCClient();
        System.out.println(" [x] Requesting fib(30)");
        String response = client.call("30");
        System.out.println(" [.] Got '" + response + "'");
        client.close();
    }
}
```

上面只是RPC示例，离真正的使用的RPC框架还有很长的距离，比如如果没有server再运行，client应该怎么处理？client的请求超时？server产生异常，是否要传递给client？异步？并行？等等问题。
