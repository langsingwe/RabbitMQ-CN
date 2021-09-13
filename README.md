# 介绍

`RabbitMQ` 是一个消息中间件：它接受和转发消息。您可以将其视为邮局：当您将要投递的邮件放入邮箱时，您可以确定信件承运人最终会将邮件递送给您的收件人。在这个比喻中，`RabbitMQ` 是一个邮箱、一个邮局和一个信件载体。

* **生产**无非就是发送。发送消息的程序是**生产者**：

  ![img](https://tva1.sinaimg.cn/large/008i3skNly1gu8ypi6ekmj601z01fwe902.jpg)

* 一个**队列**是位于 `RabbitMQ` 中的邮箱的名称。尽管消息流经 `RabbitMQ` 和您的应用程序，但它们只能存储在**队列中**。一个**队列**仅由主机的存储器＆磁盘限制约束，它本质上是一个大的消息缓冲器。许多**生产者**可以将消息发送到一个**队列**，许多**消费者**可以尝试从一个**队列**接收数据。这是我们表示队列的方式：

  ![img](https://tva1.sinaimg.cn/large/008i3skNly1gu900nhgssj603m02jgle02.jpg)

* **消费**与接收具有相似的含义。一个**消费者**是一个程序，主要是等待接收信息：

  ![img](https://tva1.sinaimg.cn/large/008i3skNly1gu902g1bkej601z01ft8h02.jpg)

请注意，生产者、消费者和中间件不必处在同一主机上；实际上，在大多数应用程序中它们没有。应用程序也可以既是生产者又是消费者。

# HelloWorld

> 用最简单的方式做一些事

## "Hello World"

在本教程的这一部分中，我们将用 Java 编写两个程序；发送单个消息的生产者，和接收消息并将其打印出来的消费者。我们将忽略 Java API 中的一些细节，专注于这个非常简单的事情只是为了开始。这是消息传递的 “Hello World”。

在下图中，“P”是我们的生产者，“C”是我们的消费者。中间的盒子是一个队列——`RabbitMQ` 代表消费者保留的消息缓冲区。

![(P) -> [|||] -> (C)](https://tva1.sinaimg.cn/large/008i3skNly1gu90y6w9zfj60aw01n3yf02.jpg)

> **Java 客户端库**
>
> `RabbitMQ` 使用多种协议。本教程使用 `AMQP 0-9-1`，这是一种用于消息传递的开放式通用协议。`RabbitMQ` 有[许多不同语言](https://rabbitmq.com/devtools.html)的客户端。我们将使用 RabbitMQ 提供的 Java 客户端。
>
> 下载[客户端库](https://repo1.maven.org/maven2/com/rabbitmq/amqp-client/5.7.1/amqp-client-5.7.1.jar) 及其依赖项（[SLF4J API](https://repo1.maven.org/maven2/org/slf4j/slf4j-api/1.7.26/slf4j-api-1.7.26.jar)和 [SLF4J Simple](https://repo1.maven.org/maven2/org/slf4j/slf4j-simple/1.7.26/slf4j-simple-1.7.26.jar)）。将这些文件复制到您的工作目录中，以及教程 Java 文件中。
>
> 请注意，`SLF4J Simple` 对于教程来说已经足够了，但您应该在生产中使用成熟的日志记录库，如[Logback](https://logback.qos.ch/)。
>
> （`RabbitMQ` Java 客户端也在中央 `Maven` 存储库中，具有 `groupId com.rabbitmq`和 `artifactId amqp-client`。）

现在我们有了 Java 客户端及其依赖项，我们可以编写一些代码。

## 发送

![(P) -> [|||]](https://tva1.sinaimg.cn/large/008i3skNly1gu9140ruagj606002s3yc02.jpg)

我们将调用我们的消息发布者（发送者）`Send`和我们的消息使用者（接收者） `Recv`。发布者将连接到 `RabbitMQ`，发送一条消息，然后退出。

在 [`Send.java`](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)，我们需要导入一些类：

```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
```

设置类并命名队列：

```java
public class Send {
  private final static String QUEUE_NAME = "hello";
  public static void main(String[] argv) throws Exception {
      ...
  }
}
```

然后我们可以创建到服务器的连接：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
try (Connection connection = factory.newConnection();
     Channel channel = connection.createChannel()) {

}
```

连接抽象了套接字连接，并为我们处理协议版本协商和认证等。在这里，我们连接到本地机器上的 `RabbitMQ` 节点 - 因此是 *localhost*。如果我们想连接到另一台机器上的节点，我们只需在此处指定其主机名或 IP 地址。

接下来我们创建一个通道，这是大多数用于完成工作的 API 所在的位置。请注意，我们可以使用 `try-with-resources` 语句，因为`Connection`和`Channel` 都实现了`java.io.Closeable`。这样我们就不需要在我们的代码中明确关闭它们。

要发送，我们必须声明一个队列供我们发送；然后我们可以向队列发布消息，所有这些都在 `try-with-resources` 语句中：

```java
channel.queueDeclare(QUEUE_NAME, false , false , false , null );
String message = "Hello World!" ;
channel.basicPublish( "" , QUEUE_NAME, null , message.getBytes());
System.out.println( " [x] Sent '" + message + "'" );
```

声明队列是幂等的 —— 只有在它不存在时才会创建它。消息内容是一个字节数组，因此您可以在其中编码任何您喜欢的内容。

[这是整个 Send.java 类](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)

> **发送无效！**
>
> 如果这是您第一次使用 `RabbitMQ` 并且没有看到“已发送”消息，那么您可能会摸不着头脑，想知道可能出了什么问题。也许代理启动时没有足够的可用磁盘空间（默认情况下它至少需要 200 MB 可用空间），因此拒绝接受消息。检查代理日志文件以确认并在必要时减少限制。该[配置文件文档](https://www.rabbitmq.com/configure.html#config-items)会告诉你如何设置`disk_free_limit`。

## 接收

以上是我们的发布者。我们的消费者监听来自 `RabbitMQ` 的消息，因此与发布单个消息的发布者不同，我们将保持消费者运行以监听消息并将它们打印出来。

![[|||] -> (C)](https://tva1.sinaimg.cn/large/008i3skNly1gu97bgsxdsj606002s0sk02.jpg)

代码（在[`Recv.java`](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java)中）与`Send`具有几乎相同的导入：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;
```

我们将使用额外的`DeliverCallback`接口来缓冲服务器推送给我们的消息。

设置与发布者相同；我们打开一个连接和一个通道，并声明我们要从中消费的队列。请注意，这与`send`发布的队列匹配。

```java
public class Recv {

  private final static String QUEUE_NAME = "hello";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

  }
}
```

请注意，我们也在这里声明了队列。因为我们可能会在发布者之前启动消费者，所以我们希望在尝试使用来自队列的消息之前确保队列存在。

为什么不使用 `try-with-resource` 语句自动关闭通道和连接呢？通过这样做，我们只会让程序继续运行，关闭所有内容，然后退出！这会很尴尬，因为我们希望进程在消费者异步侦听消息到达时保持活动状态。

我们将要告诉服务器将队列中的消息传递给我们。由于它将异步推送消息，我们以对象的形式提供回调，该回调将缓冲消息，直到我们准备好使用它们。这就是`DeliverCallback`子类所做的。

```java
DeliverCallback deliveryCallback = (consumerTag, delivery) -> {
    String message = new String(delivery.getBody(), "UTF-8" );
    System.out.println( " [x] 收到 '" + message + "'" );
};
channel.basicConsume(QUEUE_NAME, true , deliveryCallback, consumerTag -> { });
```

[这是整个 Recv.java 类](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java)

## 总结

您可以仅使用类路径上的 `RabbitMQ` java 客户端来编译这两个：

```bash
javac -cp amqp-client-5.7.1.jar Send.java Recv.java
```

要运行它们，您需要`rabbitmq-client.jar`及其对类路径的依赖项。在终端中，运行消费者（接收者）：

```bash
java -cp .:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar Recv
```

然后，运行发布者（发送者）：

```bash
java -cp .:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar Send
```

在 Windows 上，使用分号而不是冒号来分隔类路径中的项目。

消费者将通过 `RabbitMQ` 打印它从发布者那里得到的消息。消费者将继续运行，等待消息（使用 `Ctrl-C` 停止它），因此尝试从另一个终端运行发布者

> **列出队列**
>
> 您可能希望查看 `RabbitMQ` 有哪些队列以及其中有多少消息。您可以使用`rabbitmqctl`工具（作为特权用户）执行此操作：
>
> ```bash
> sudo rabbitmqctl list_queues
> ```
>
> 在 Windows 上，省略 sudo：
>
> ```bash
> rabbitmqctl.bat list_queues
> ```

是时候进入 [第 2 部分](#工作队列) 并构建一个简单的*工作队列了*。

> **提示**
>
> 为了节省输入，您可以为类路径设置一个环境变量，例如
>
> ```bash
> export CP=.:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar
> java -cp $CP Send
> ```
>
> 或在 Windows 上：
>
> ```bash
> set CP=.;amqp-client-5.7.1.jar;slf4j-api-1.7.26.jar;slf4j-simple-1.7.26.jar
> java -cp %CP% Send
> ```

# 工作队列

> 在工人之间分配任务（竞争消费者模式）

![img](https://tva1.sinaimg.cn/large/008i3skNly1gu9aozwjo7j6098033aa002.jpg)

在 [第一个教程](#HelloWorld) 中，我们编写了从命名队列发送和接收消息的程序。在本节中，我们将创建一个*工作队列*，用于在多个工作人员之间分配耗时的任务。

工作队列（又名：*任务队列*）背后的主要思想是避免立即执行资源密集型任务而不得不等待它完成。相反，我们安排任务稍后完成。我们将*任务*封装 为消息并将其发送到队列。在后台运行的工作进程将弹出任务并最终执行作业。当您运行许多工人时，任务将在他们之间共享。

这个概念在 Web 应用程序中特别有用，因为在短的 HTTP 请求窗口期间不可能处理复杂的任务。

## 准备

在本教程的前一部分中，我们发送了一条包含“Hello World！”的消息。现在我们将发送代表复杂任务的字符串。我们没有现实世界的任务，比如要调整大小的图像或要渲染的 pdf 文件，所以让我们假设我们很忙来假装它 - 通过使用`Thread.sleep()`函数。我们将把字符串中的点数作为它的复杂度；每个点将占一秒钟的“工作”。例如，`Hello...`描述的假任务需要三秒钟。

我们将稍微修改前面示例中的`Send.java`代码，以允许从命令行发送任意消息。该程序会将任务安排到我们的工作队列中，因此我们将其命名为 `NewTask.java`：

```java
String message = String.join( " " , argv);

channel.basicPublish( "" , "hello" , null , message.getBytes());
System.out.println( " [x] Sent '" + message + "'" );
```

我们旧的`Recv.java`程序也需要一些更改：它需要为消息正文中的每个点伪造一秒钟的工作。它将处理传递的消息并执行任务，所以我们称之为`Worker.java`：

```java
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
  String message = new String(delivery.getBody(), "UTF-8");

  System.out.println(" [x] Received '" + message + "'");
  try {
    doWork(message);
  } finally {
    System.out.println(" [x] Done");
  }
};
boolean autoAck = true; // acknowledgment is covered below
channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, consumerTag -> { });
```

我们模拟执行时间的假任务：

```java
private  static  void  doWork (String task)抛出 InterruptedException {
    for ( char ch: task.toCharArray()) {
        if (ch == '.' ) Thread.sleep( 1000 );
    }
}
```

像教程1一样编译它们（使用工作目录中的 jar 文件和环境变量`CP`）：

```bash
javac -cp $CP NewTask.java Worker.java
```

## 轮询调度

使用任务队列的优势之一是能够轻松并行化工作。如果我们正在建立积压的工作，我们可以添加更多的工作人员，这样就可以轻松扩展。

首先，让我们尝试同时运行两个工作实例。他们都会从队列中获取消息，但具体如何实现？让我们来看看。

您需要打开三个控制台。两个将运行工作程序。这些控制台将成为我们的两个消费者——`C1` 和 `C2`。

```bash
# shell 1
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
```

```bash
# shell 2
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
```

在第三个控制台中，我们将发布新任务。启动消费者后，您可以发布一些消息：

```bash
# shell 3
java -cp $CP NewTask First message.
# => [x] Sent 'First message.'
java -cp $CP NewTask Second message..
# => [x] Sent 'Second message..'
java -cp $CP NewTask Third message...
# => [x] Sent 'Third message...'
java -cp $CP NewTask Fourth message....
# => [x] Sent 'Fourth message....'
java -cp $CP NewTask Fifth message.....
# => [x] Sent 'Fifth message.....'
```

让我们看看交付给我们的工人的是什么：

```bash
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

```bash
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

默认情况下，`RabbitMQ` 将按顺序将每条消息发送给下一个消费者。平均而言，每个消费者将获得相同数量的消息。这种分发消息的方式称为轮询。与三个或更多工人一起试试这个。

## 消息确认

完成一项任务可能需要几秒钟。您可能想知道，如果其中一个消费者开始了一项长期任务并且只完成了部分任务而死亡，会发生什么。使用我们当前的代码，一旦 `RabbitMQ` 将消息传递给消费者，它会立即将其标记为删除。在这种情况下，如果你杀死一个工人，我们将丢失它刚刚处理的消息。我们还将丢失所有已分派给该特定工作人员但尚未处理的消息。

但我们不想丢失任何任务。如果一个工人死了，我们希望将任务交付给另一个工人。

为了确保消息永远不会丢失，`RabbitMQ` 支持 [消息*确认*](https://www.rabbitmq.com/confirms.html)。消费者发回确认消息，告诉 RabbitMQ 特定消息已被接收、处理，并且 `RabbitMQ` 可以自由删除它。

如果消费者在没有发送 `ack` 的情况下死亡（其通道关闭、连接关闭或 TCP 连接丢失），`RabbitMQ` 将理解消息未完全处理并将重新排队。如果有其他消费者同时在线，它会迅速将其重新交付给另一个消费者。这样您就可以确保不会丢失任何消息，即使工作人员偶尔会死亡。

没有任何消息超时；`RabbitMQ` 会在消费者死亡时重新传递消息。即使处理一条消息需要非常非常长的时间也没关系。

默认情况下启用[手动消息确认](https://www.rabbitmq.com/confirms.html)。在前面的例子中，我们通过`autoAck=true`标志明确地关闭了它们。一旦我们完成了一项任务，是时候将此标志设置为false并让工作人员发送正确的确认。

```java
channel.basicQos(1); // 一次只接受一条未确认的消息（见下文）

DeliverCallback deliverCallback = (consumerTag, delivery) -> {
  String message = new String(delivery.getBody(), "UTF-8");

  System.out.println(" [x] Received '" + message + "'");
  try {
    doWork(message);
  } finally {
    System.out.println(" [x] Done");
    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
  }
};
boolean autoAck = false;
channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, consumerTag -> { });
```

使用此代码，我们可以确保即使您在处理消息时使用 `CTRL+C` 杀死工作人员，也不会丢失任何内容。工人死后不久，所有未确认的消息都将重新传递。

必须在接收交付的同一通道上发送确认。尝试使用不同的通道进行确认将导致通道级协议异常。请参阅[有关确认](https://www.rabbitmq.com/confirms.html)的[文档指南](https://www.rabbitmq.com/confirms.html) 以了解更多信息。

> **忘记确认**
>
> 缺少`basicAck`是一个常见的错误。这是一个简单的错误，但后果很严重。当您的客户端退出时，消息将被重新传送（这可能看起来像随机重新传送），但 `RabbitMQ` 将消耗越来越多的内存，因为它无法释放任何未确认的消息。
>
> 为了调试这种错误，您可以使用`rabbitmqctl` 打印`messages_unacknowledged`字段：
>
> ```bash
> sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
> ```
>
> 在 Windows 上，删除 `sudo`：
>
> ```bash
> rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
> ```

## 消息持久性

我们已经学会了如何确保即使消费者死亡，任务也不会丢失。但是如果`RabbitMQ`服务器停止，我们的任务仍然会丢失。

当 `RabbitMQ` 退出或崩溃时，它会忘记队列和消息，除非你告诉它不要。需要做两件事来确保消息不会丢失：我们需要将队列和消息都标记为持久的。

首先，我们需要确保队列能够在 `RabbitMQ` 节点重启后继续存在。为此，我们需要将其声明为*持久的*：

```java
boolean durable = true;
channel.queueDeclare("hello", durable, false, false, null);
```

虽然这个命令本身是正确的，但它在我们目前的设置中不起作用。那是因为我们已经定义了一个名为`hello`的队列 ，它不是持久的。`RabbitMQ` 不允许您使用不同的参数重新定义现有队列，并且会向任何尝试这样做的程序返回错误。但是有一个快速的解决方法 —— 让我们声明一个具有不同名称的队列，例如`task_queue`：

```java
boolean durable = true;
channel.queueDeclare("task_queue", durable, false, false, null);
```

此`queueDeclare`更改需要应用于生产者和消费者代码。

此时我们确信即使`RabbitMQ`重启，`task_queue`队列也不会丢失。现在我们需要将我们的消息标记为持久性 - 通过将`MessageProperties`（实现`BasicProperties`）设置为值`PERSISTENT_TEXT_PLAIN`。

```java
import com.rabbitmq.client.MessageProperties;

channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

>  消息持久化注意事项
>
>  将消息标记为持久并不能完全保证消息不会丢失。虽然它告诉 `RabbitMQ` 将消息保存到磁盘，但是当 `RabbitMQ` 已经接受一条消息并且还没有保存它时，仍然有很短的时间窗口。此外，`RabbitMQ` 不会对每条消息都执行`fsync(2)` —— 它可能只是保存到缓存中，而不是真正写入磁盘。持久性保证不强，但对于我们简单的任务队列来说已经足够了。如果您需要更强的保证，那么您可以使用 [发布者确认](https://www.rabbitmq.com/confirms.html)。

## 公平调度

您可能已经注意到调度仍然不能完全按照我们想要的方式工作。例如，在有两个 `worker` 的情况下，当所有奇数消息都很重，偶数消息很轻时，一个 `worker` 会一直很忙，而另一个 `worker` 几乎不做任何工作。好吧，`RabbitMQ` 对此一无所知，仍然会均匀地发送消息。

发生这种情况是因为 `RabbitMQ` 只是在消息进入队列时分派消息。它不考虑消费者未确认消息的数量。它只是盲目地将每条第 n 条消息分派给第 n 条消费者。

![img](https://tva1.sinaimg.cn/large/008i3skNly1guacuqfjh1j60b0033jre02.jpg)

为了解决这个问题，我们可以使用带有`prefetchCount = 1`设置的`basicQos`方法 。这告诉 `RabbitMQ` 一次不要给一个工人多个消息。或者，换句话说，在处理并确认前一条消息之前，不要向工作人员发送新消息。相反，它会将它分派给下一个不忙的工人。

```java
int prefetchCount = 1 ;
channel.basicQos(prefetchCount);
```

>  **关于队列大小的注意事项**
>
> 如果所有工作人员都很忙，您的队列可能会填满。你会想要密切关注这一点，也许会增加更多的工人，或者有一些其他的策略。

## 总结

我们的`NewTask.java`类的最终代码：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.MessageProperties;

public class NewTask {

  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    try (Connection connection = factory.newConnection();
        Channel channel = connection.createChannel()) {
        channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);

        String message = String.join(" ", argv);

        channel.basicPublish("", TASK_QUEUE_NAME, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes("UTF-8"));
        System.out.println(" [x] Sent '" + message + "'");
    }
  }

}
```

[(NewTask.java 源码)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/NewTask.java)

还有我们的`Worker.java`：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class Worker {

  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    final Connection connection = factory.newConnection();
    final Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    channel.basicQos(1);

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");

        System.out.println(" [x] Received '" + message + "'");
        try {
            doWork(message);
        } finally {
            System.out.println(" [x] Done");
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
    };
    channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> { });
  }

  private static void doWork(String task) {
    for (char ch : task.toCharArray()) {
        if (ch == '.') {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException _ignored) {
                Thread.currentThread().interrupt();
            }
        }
    }
  }
}
```

[(Worker.java 源码)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Worker.java)

使用消息确认和`prefetchCount`您可以设置一个工作队列。即使 `RabbitMQ` 重新启动，持久性选项也能让任务继续存在。

有关`Channel`方法和`MessageProperties` 的更多信息，您可以[在线](https://rabbitmq.github.io/rabbitmq-java-client/api/current/)浏览 [JavaDocs](https://rabbitmq.github.io/rabbitmq-java-client/api/current/)。

现在我们可以继续 [教程 3](#发布订阅) 并学习如何向许多消费者传递相同的消息。

# 发布订阅

> 一次向多个消费者发送消息

在[上一个教程中](#工作队列)，我们创建了一个工作队列。工作队列背后的假设是每个任务都被交付给一个工人。在这一部分，我们将做一些完全不同的事情——我们将向多个消费者传递一条消息。这种模式被称为“发布/订阅”。

为了说明该模式，我们将构建一个简单的日志系统。它将由两个程序组成——第一个将发出日志消息，第二个将接收并打印它们。

在我们的日志系统中，接收器程序的每个运行副本都会收到消息。这样我们就可以运行一个接收器并将日志定向到磁盘；同时我们将能够运行另一个接收器并在屏幕上查看日志。

本质上，发布的日志消息将被广播给所有接收者。

## 交换机

在本教程的前几部分中，我们向队列发送消息和从队列接收消息。现在是时候介绍 `Rabbit` 中的完整消息传递模型了。

让我们快速回顾一下我们在之前的教程中介绍的内容：

* 生产者是发送消息的用户程序
* 队列是一个缓冲区，用于存储消息
* 消费者是接收消息的用户程序

`RabbitMQ` 消息传递模型的核心思想是生产者从不直接向队列发送任何消息。实际上，生产者经常甚至根本不知道消息是否会被传送到任何队列。

相反，生产者只能将消息发送到*交换机*。交换是一件非常简单的事情。一方面它接收来自生产者的消息，另一方面将它们推送到队列中。交换机必须确切地知道如何处理它收到的消息。它应该附加到特定队列吗？它应该附加到许多队列中吗？或者它应该被丢弃。其规则由*交换类型*定义 。

![img](https://tva1.sinaimg.cn/large/008i3skNly1guadn3ctrjj6098033mx402.jpg)

有几种可用的交换机类型：`direct`、`topic`、`headers` 和`fanout`。我们将关注最后一个——`fanout（扇形交换机）`。让我们创建一个这种类型的交换机，并将其命名为`logs`：

```java
channel.exchangeDeclare("logs", "fanout");
```

扇形交换机非常简单。正如您可能从名称中猜到的那样，它只是将它收到的所有消息广播到它知道的所有队列。这正是我们日志所需要的。

> **列出交换机**
>
> 要列出服务器上的交换，您可以运行有用的`rabbitmqctl`：
>
> ```bash
> sudo rabbitmqctl list_exchanges
> ```
>
> 在此列表中将有一些`amq.*`交换机和默认（未命名）交换。这些是默认创建的，但您目前不太可能需要使用它们。
>
> **无名交换机**
>
> 在本教程的前几部分中，我们对交换机一无所知，但仍然能够将消息发送到队列。这是可能的，因为我们使用的是默认交换机，我们用空字符串 ( `""` )标识。
>
> 回想一下我们之前是如何发布消息的：
>
> ```java
> channel.basicPublish("", "hello", null, message.getBytes());
> ```
>
> 第一个参数是交换机的名称。空字符串表示默认或*无名*交换机：消息将路由到具有`routingKey`指定名称的队列（如果存在）。

现在，我们可以改为发布到我们的命名交换机：

```java
channel.basicPublish( "logs" , "" , null , message.getBytes());
```

## 临时队列

您可能还记得以前我们使用具有特定名称的队列（还记得`hello`和`task_queue`吗？）。能够命名队列对我们来说至关重要——我们需要将工作人员指向同一个队列。当您想在生产者和消费者之间共享队列时，为队列命名很重要。

但对于我们的日志系统而言，情况并非如此。我们希望了解所有日志消息，而不仅仅是其中的一部分。我们也只对当前流动的消息感兴趣，而不是旧消息。为了解决这个问题，我们需要做两件事。

首先，每当我们连接到 `Rabbit` 时，我们都需要一个新的空队列。为此，我们可以创建一个具有随机名称的队列，或者采取更好的方式 —— 让服务器为我们选择一个随机队列名称。

其次，一旦我们断开消费者连接，队列应该被自动删除。

在 Java 客户端中，当我们不向`queueDeclare()`提供参数时， 我们会创建一个具有生成名称的非持久、独占、自动删除队列：

```java
String queueName = channel.queueDeclare().getQueue();
```

您可以[在队列指南中](https://www.rabbitmq.com/queues.html)了解有关独占标志和其他队列属性的更多信息。

此时`queueName`包含一个随机队列名称。例如，它可能看起来像`amq.gen-JzTY20BRgKO-HjmUJj0wLg`。

## 绑定

![img](https://tva1.sinaimg.cn/large/008i3skNly1guaed7yccfj608y02j74802.jpg)

我们已经创建了一个扇形交换机和一个队列。现在我们需要告诉交换机向我们的队列发送消息。交换机和队列之间的这种关系称为*绑定*。

```java
channel.queueBind(queueName, "logs" , "" );
```

从现在开始，`logs`交换机会将消息附加到我们的队列中。

> **列出绑定**
>
> 你可以列出现有的绑定
>
> ```bash
> rabbitmqctl list_bindings
> ```

## 总结

![img](https://tva1.sinaimg.cn/large/008i3skNly1guaeh6aly1j609504gjre02.jpg)

发出日志消息的生产者程序与之前的教程看起来没有太大区别。最重要的变化是我们现在想要将消息发布到我们的`logs`交换机而不是无名的交换机。我们需要在发送时提供一个`routingKey`，但它的值在扇形交换时被忽略。下面是`EmitLog.java`程序的代码 ：

```java
public class EmitLog {

  private static final String EXCHANGE_NAME = "logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    try (Connection connection = factory.newConnection();
         Channel channel = connection.createChannel()) {
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        String message = argv.length < 1 ? "info: Hello World!" :
                            String.join(" ", argv);

        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes("UTF-8"));
        System.out.println(" [x] Sent '" + message + "'");
    }
  }
}
```

[(EmitLog.java 源码)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLog.java)

如您所见，在建立连接后，我们声明了交换机。这一步是必要的，因为禁止发布到不存在的交换机。

如果还没有队列绑定到交换机，消息将会丢失，但这对我们来说是可以的；如果还没有消费者在监听，我们可以安全地丢弃消息。

`ReceiveLogs.java`的代码：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class ReceiveLogs {
  private static final String EXCHANGE_NAME = "logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
    String queueName = channel.queueDeclare().getQueue();
    channel.queueBind(queueName, EXCHANGE_NAME, "");

    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");
        System.out.println(" [x] Received '" + message + "'");
    };
    channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
  }
}
```

[(ReceiveLogs.java 源码)](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogs.java)

像以前一样编译，我们就完成了。

```bash
javac -cp $CP EmitLog.java ReceiveLogs.java
```

如果要将日志保存到文件，只需打开控制台并键入：

```bash
java -cp $CP ReceiveLogs > logs_from_rabbit.log
```

如果您希望在屏幕上看到日志，请生成一个新终端并运行：

```bash
java -cp $CP ReceiveLogs
```

当然，要发出日志类型：

```bash
java -cp $CP EmitLog
```

使用`rabbitmqctl list_bindings`您可以验证代码是否确实按照我们的需要创建了绑定和队列。 运行两个`ReceiveLogs.java`程序时，您应该看到如下内容：

```bash
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```

结果的解释很简单：来自`logs`交换机的数据进入两个具有服务器分配名称的队列。而这正是我们的意图。

要了解如何侦听消息的子集，让我们继续学习 [教程 4](#路由)

# 路由

> 有选择地接收消息

在[之前的教程中](#发布订阅)我们构建了一个简单的日志系统。我们能够向许多接收器广播日志消息。

在本教程中，我们将向其添加一个功能 - 我们将使其仅订阅消息的子集成为可能。例如，我们将能够仅将关键错误消息定向到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有日志消息。

## 绑定

在前面的例子中，我们已经在创建绑定。你可能会想起这样的代码：

```java
channel.queueBind(queueName, EXCHANGE_NAME, "" );
```

绑定是交换机和队列之间的关系。这可以简单地理解为：队列对来自这个交换机的消息感兴趣。

绑定可以采用额外的`routingKey`参数。为了避免与`basic_publish`参数混淆，我们将称其为 `binding key`。这是我们如何使用键创建绑定：

```java
channel.queueBind(queueName, EXCHANGE_NAME, "black");
```

绑定键的含义取决于交换机的类型。我们之前使用的扇形交换机只是忽略了它的值。

## 直连交换机

我们上一教程中的日志系统将所有消息广播给所有消费者。我们希望扩展它以允许根据其严重性过滤消息。例如，我们可能希望将日志消息写入磁盘的程序只接收严重错误，而不是在警告或信息日志消息上浪费磁盘空间。

我们使用的是扇形交换机，它没有给我们很大的灵活性——它只能进行无意识的广播。

我们将改用直连交换机。直连交换机背后的路由算法很简单 —— 消息进入其`binding key`与消息的`routing key`完全匹配的队列 。

为了说明这一点，请考虑以下设置：

![img](https://tva1.sinaimg.cn/large/008i3skNly1guafwc8teqj60bc04r74c02.jpg)

在这个设置中，我们可以看到绑定了两个队列的直连交换机`X`。第一个队列绑定了绑定键`orange`，第二个队列有两个绑定，一个绑定键为`black`，另一个绑定为`green`。

在这样的设置中，使用路由键`orange`发布到交换机的消息 将被路由到队列`Q1`。路由键为`black` 或`green`的消息将转到`Q2`。所有其他消息将被丢弃。

## 多重绑定

![img](https://tva1.sinaimg.cn/large/008i3skNly1guag8xmq4nj60b204rq2z02.jpg)

使用相同的绑定键绑定多个队列是完全合法的。在我们的示例中，我们可以使用绑定键`black`在`X`和`Q1`之间添加一个绑定。在这种情况下，直连交换机的行为类似于扇出，并将消息广播到所有匹配的队列。路由键为`black` 的消息将同时发送到 `Q1`和`Q2`

## 发出日志

我们将在我们的日志系统中使用这个模型。我们将向直连交换机发送消息而不是扇形交换机。我们将提供日志严重性作为路由键。这样，接收程序将能够选择它想要接收的严重性。让我们首先专注于发出日志。

和往常一样，我们需要先创建一个交换机：

```java
channel.exchangeDeclare(EXCHANGE_NAME, "direct" );
```

我们准备发送消息：

```java
channel.basicPublish(EXCHANGE_NAME,severity, null , message.getBytes());
```

为了简化事情，我们假设“严重性”可以是“信息”、“警告”、“错误”之一。

## 订阅

接收消息的工作方式与上一教程一样，但有一个例外 —— 我们将为我们感兴趣的每个严重性创建一个新绑定。

```java
String queueName = channel.queueDeclare().getQueue();

for(String severity : argv){
  channel.queueBind(queueName, EXCHANGE_NAME, severity);
}
```

## 总结

![img](https://tva1.sinaimg.cn/large/008i3skNly1guagyp5ssdj60br04rdfy02.jpg)

`EmitLogDirect.java`类的代码：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class EmitLogDirect {

  private static final String EXCHANGE_NAME = "direct_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    try (Connection connection = factory.newConnection();
         Channel channel = connection.createChannel()) {
        channel.exchangeDeclare(EXCHANGE_NAME, "direct");

        String severity = getSeverity(argv);
        String message = getMessage(argv);

        channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes("UTF-8"));
        System.out.println(" [x] Sent '" + severity + "':'" + message + "'");
    }
  }
  
  private static String getSeverity(String[] strings) {
        if (strings.length < 1)
            return "info";
        return strings[0];
    }

  private static String getMessage(String[] strings) {
    if (strings.length < 2)
      return "Hello World!";
    return joinStrings(strings, " ", 1);
  }

  private static String joinStrings(String[] strings, String delimiter, int startIndex) {
    int length = strings.length;
    if (length == 0) return "";
    if (length <= startIndex) return "";
    StringBuilder words = new StringBuilder(strings[startIndex]);
    for (int i = startIndex + 1; i < length; i++) {
      words.append(delimiter).append(strings[i]);
    }
    return words.toString();
  }
}
```

`ReceiveLogsDirect.java`的代码：

```java
import com.rabbitmq.client.*;

public class ReceiveLogsDirect {

  private static final String EXCHANGE_NAME = "direct_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "direct");
    String queueName = channel.queueDeclare().getQueue();

    if (argv.length < 1) {
        System.err.println("Usage: ReceiveLogsDirect [info] [warning] [error]");
        System.exit(1);
    }

    for (String severity : argv) {
        channel.queueBind(queueName, EXCHANGE_NAME, severity);
    }
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");
        System.out.println(" [x] Received '" +
            delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
    };
    channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
  }
}
```

像往常一样编译（有关编译和类路径建议，请参阅[教程一](#HelloWorld)）。为方便起见，我们将在运行示例时为类路径使用环境变量 `$CP`（在 Windows 上为 `%CP%`）。

```bash
javac -cp $CP ReceiveLogsDirect.java EmitLogDirect.java
```

如果您只想将`warning`和`error`（而不是`info`）日志消息保存到文件中，只需打开控制台并键入：

```bash
java -cp $CP ReceiveLogsDirect warning error > logs_from_rabbit.log
```

如果您想在屏幕上查看所有日志消息，请打开一个新终端并执行以下操作：

```bash
java -cp $CP ReceiveLogsDirect info warning error
# => [*] Waiting for logs. To exit press CTRL+C
```

例如，要发出`error`级别日志消息，只需键入：

```bash
java -cp $CP EmitLogDirect error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
```

 [(EmitLogDirect.java 源码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLogDirect.java) 和 [(ReceiveLogsDirect.java 源码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogsDirect.java)

继续学习[教程 5](#主题)，了解如何根据模式监听消息。

# 主题

> 基于模式（主题）接收消息

在[之前的教程中，](#路由)我们改进了日志系统。我们没有使用只能进行虚拟广播的扇形交换机，而是使用直连交换机，并获得了选择性接收日志的可能性。

尽管使用直连交换机改进了我们的系统，但它仍然有局限性——它不能基于多个标准进行路由。

在我们的日志系统中，我们可能不仅希望根据严重性订阅日志，还希望根据发出日志的源订阅日志。您可能从 [syslog](http://en.wikipedia.org/wiki/Syslog) unix 工具中了解到这个概念，该 工具根据严重性 (info/warn/crit...) 和设施 (auth/cron/kern...) 路由日志。

这会给我们带来很大的灵活性——我们可能只想听来自“cron”的严重错误以及来自“kern”的所有日志。

为了在我们的日志系统中实现这一点，我们需要了解一个更复杂的主题交换。

## 主题交换机

发送到主题交换的消息不能具有任意的 `routing_key` —— 它必须是一个单词列表，由点分隔。这些词可以是任何东西，但通常它们会指定一些与消息相关的特征。一些有效的路由键示例：`“ stock.usd.nyse ”`、`“ nyse.vmw ”`、`“ quick.orange.rabbit ”`。路由键中可以有任意多个字，最多 255 个字节。

绑定建也必须采用相同的形式。主题交换机背后的逻辑 类似于直连交换机 —— 使用特定路由键发送的消息将被传递到使用匹配绑定键绑定的所有队列。然而，绑定键有两个重要的特殊情况：

* `*`（星号）用来表示一个单词。
* `#`（井号）用来表示任意数量（0个或多个）单词。

在一个例子中最容易解释这一点：

![img](https://tva1.sinaimg.cn/large/008i3skNly1gubbphdl09j60bs04rq3002.jpg)

在这个例子中，我们将发送所有描述动物的消息。消息将使用由三个字（两个点）组成的路由键发送。路由键中的第一个词将描述速度，第二个是颜色，第三个是物种：` <speed>.<colour>.<species> `。

我们创建了三个绑定：`Q1` 与绑定键 ` *.orange.*` 绑定，`Q2 `与 ` *.*.rabbit` 和 `lazy.# ` 绑定。

这些绑定可以总结为：

- `Q1` 对所有橙色动物都感兴趣。
- `Q2` 想听听关于兔子的一切，以及关于懒惰动物的一切。

路由键设置为 `quick.orange.rabbit` 的消息将发送到两个队列。消息 `lazy.orange.elephant` 也会发给他们两个。另一方面， `quick.orange.fox` 只会进入第一个队列，而 `lazy.brown.fox` 只会进入第二个队列。` lazy.pink.rabbit` 只会被传送到第二个队列一次，即使它匹配了两个绑定。`quick.brown.fox` 不匹配任何绑定，因此将被丢弃。

如果我们违反合同并发送一到四个字的消息，例如 `orange` 或 `quick.orange.male.rabbit` ，会发生什么？好吧，这些消息不会匹配任何绑定并且会丢失。

另一方面， `lazy.orange.male.rabbit` ，即使它有四个单词，也会匹配最后一个绑定，并将被传递到第二个队列。

> **主题交换机**
>
> 主题交换机功能强大，可以像其他交换机一样运行。
>
> 当队列与“ # ”（井号）绑定键绑定时——它将接收所有消息，而不管路由键——就像在扇形交换机中一样。
>
> 当绑定中不使用特殊字符“ * ”（星号）和“ # ”（井号）时，主题交换机的行为就像直连交换机一样。

## 总结

我们将在日志系统中使用主题交换机。我们将从一个可行的假设开始，即日志的路由键将有两个词：`<facility>.<severity>`。

代码与[上一教程中](#路由)的代码几乎相同 。

`EmitLogTopic.java`的代码：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class EmitLogTopic {

    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {

            channel.exchangeDeclare(EXCHANGE_NAME, "topic");

            String routingKey = getRouting(argv);
            String message = getMessage(argv);

            channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes("UTF-8"));
            System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");
        }
    }

    private static String getRouting(String[] strings) {
        if (strings.length < 1)
            return "anonymous.info";
        return strings[0];
    }

    private static String getMessage(String[] strings) {
        if (strings.length < 2)
            return "Hello World!";
        return joinStrings(strings, " ", 1);
    }

    private static String joinStrings(String[] strings, String delimiter, int startIndex) {
        int length = strings.length;
        if (length == 0) return "";
        if (length < startIndex) return "";
        StringBuilder words = new StringBuilder(strings[startIndex]);
        for (int i = startIndex + 1; i < length; i++) {
            words.append(delimiter).append(strings[i]);
        }
        return words.toString();
    }
}
```

`ReceiveLogsTopic.java`的代码：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class ReceiveLogsTopic {

    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "topic");
        String queueName = channel.queueDeclare().getQueue();

        if (argv.length < 1) {
            System.err.println("Usage: ReceiveLogsTopic [binding_key]...");
            System.exit(1);
        }

        for (String bindingKey : argv) {
            channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);
        }

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [x] Received '" + delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
        };
        channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
    }
}
```

编译并运行示例，包括[教程 1 中](#HelloWorld)的类路径- 在 Windows 上，使用 `%CP%`。

编译：

```bash
javac -cp $CP ReceiveLogsTopic.java EmitLogTopic.java
```

接收所有日志：

```bash
java -cp $CP ReceiveLogsTopic "#"
```

从工具“ kern ”接收所有日志：

```bash
java -cp $CP ReceiveLogsTopic "kern.*"
```

或者，如果您只想听到“critical”日志：

```bash
java -cp $CP ReceiveLogsTopic "*.critical"
```

您可以创建多个绑定：

```bash
java -cp $CP ReceiveLogsTopic "kern.*"  "*.critical"
```

并使用路由键“ kern.critical ”发出日志，请键入：

```bash
java -cp $CP EmitLogTopic "kern.critical"  "A critical kernel error"
```

使用这些程序代码尽情的测试。。请注意，该代码并未对路由或绑定键做出任何假设，您可能想要使用两个以上的路由键参数。

（[EmitLogTopic.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLogTopic.java) 和[ReceiveLogsTopic.java 的](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogsTopic.java)完整源代码）

接下来，在[教程 6 中](https://www.rabbitmq.com/tutorials/tutorial-six-java.html)找出如何将往返消息作为远程过程调用进行处理

# RPC

> 请求/应答模式示例

在[第二个教程中，](#工作队列)我们学习了如何使用*工作队列*在多个工作人员之间分配耗时的任务。

但是如果我们需要在远程计算机上运行一个函数并等待结果呢？嗯，这是一个不同的故事。这种模式通常称为*远程过程调用*或`RPC`。

在本教程中，我们将使用 `RabbitMQ` 构建一个 `RPC` 系统：一个客户端和一个可扩展的 `RPC` 服务器。由于我们没有任何值得分配的耗时任务，我们将创建一个返回斐波那契数列的虚拟 `RPC` 服务。

## 客户端界面

为了说明如何使用 `RPC` 服务，我们将创建一个简单的客户端类。它将公开一个名为`call`的方法，该方法 发送一个 `RPC` 请求并阻塞直到收到答案：

```java
FibonacciRpcClient fibonacciRpc = new FibonacciRpcClient();
String result = fibonacciRpc.call( "4" );
System.out.println( "fib(4) 是" + 结果);
```

> **关于 `RPC` 的说明**
>
> 尽管 `RPC` 是计算中非常常见的模式，但它经常受到批评。当程序员不知道函数调用是本地函数还是慢 `RPC` 时，就会出现问题。像这样的混乱会导致系统不可预测，并给调试增加不必要的复杂性。滥用 `RPC` 不仅不会简化软件，还会导致无法维护的意大利面条式代码。
>
> 牢记这一点，请考虑以下建议：
>
> - 确保哪个函数调用是本地的，哪个是远程的。
> - 记录您的系统。明确组件之间的依赖关系。
> - 处理错误情况。当`RPC`服务器长时间宕机时，客户端应该如何反应？
>
> 如有疑问，请避免使用 `RPC`。如果可以，您应该使用异步管道——而不是类似 `RPC` 的阻塞，结果被异步推送到下一个计算阶段。

## 回调队列

一般来说，通过 `RabbitMQ` 进行 `RPC` 很容易。客户端发送请求消息，服务器回复响应消息。为了接收响应，我们需要随请求一起发送“回调”队列地址。我们可以使用默认队列（在 Java 客户端中是独占的）。让我们试试看：

```java
callbackQueueName = channel.queueDeclare().getQueue();

BasicProperties props = new BasicProperties
                            .Builder()
                            .replyTo(callbackQueueName)
                            .build();

channel.basicPublish("", "rpc_queue", props, message.getBytes());

// ... 然后从 callback_queue 读取响应消息的代码 ...
```

> **消息属性**
>
> AMQP 0-9-1 协议预定义了一组 14 个与消息一起使用的属性。大多数属性很少使用，但以下情况除外：
>
> - `deliveryMode`：将消息标记为持久（值为`2`）或瞬态（任何其他值）。您可能还记得[第二个教程中的](#工作队列)这个属性。
> - `contentType`：用于描述编码的 MIME 类型。例如，对于经常使用的 JSON 编码，最好将此属性设置为：`application/json`。
> - `replyTo`：通常用于命名回调队列。
> - `correlationId`：用于将 `RPC` 响应与请求相关联

我们需要这个新的导入：

```java
import com.rabbitmq.client.AMQP.BasicProperties;
```

## 关联ID

在上面介绍的方法中，我们建议为每个 `RPC` 请求创建一个回调队列。这是非常低效的，但幸运的是有一个更好的方法 —— 让我们为每个客户端创建一个回调队列。

这引发了一个新问题，在该队列中收到响应后，不清楚该响应属于哪个请求。这就是使用`correlationId`属性的时候 。我们将为每个请求将其设置为唯一值。稍后，当我们在回调队列中收到一条消息时，我们将查看此属性，并基于此属性将响应与请求进行匹配。如果我们看到未知的 `correlationId`值，我们可以安全地丢弃该消息——它不属于我们的请求。

你可能会问，为什么我们要忽略回调队列中的未知消息，而不是失败并报错？这是由于服务器端可能存在竞争条件。虽然不太可能，但 `RPC` 服务器可能会在向我们发送答案之后，但在发送请求的确认消息之前就死了。如果发生这种情况，重新启动的 `RPC` 服务器将再次处理请求。这就是为什么在客户端我们必须优雅地处理重复响应，并且理想情况下 `RPC` 应该是幂等的。

## 摘要

![img](https://tva1.sinaimg.cn/large/008i3skNly1gubib5kooij60g005kdg202.jpg)

我们的 `RPC` 将像这样工作：

- 对于 `RPC` 请求，客户端发送具有两个属性的消息： `replyTo`，它被设置为仅为请求创建的匿名独占队列，以及`correlationId`，它被设置为每个请求的唯一值。
- 请求被发送到`rpc_queue`队列。
- `RPC` 工作者（又名：服务器）正在等待该队列上的请求。当一个请求出现时，它会执行工作并将带有结果的消息发送回客户端，使用来自`replyTo`字段的队列。
- 客户端等待回复队列中的数据。当出现一条消息时，它会检查`correlationId`属性。如果它与请求中的值匹配，则它将响应返回给应用程序。

## 总结

斐波那契任务：

```java
private static int fib(int n) {
    if (n == 0) return 0;
    if (n == 1) return 1;
    return fib(n-1) + fib(n-2);
}
```

我们声明我们的斐波那契函数。它只假设有效的正整数输入。（不要指望这适用于大数字，它可能是最慢的递归实现）。

我们的 RPC 服务器的代码可以在这里找到：[RPCServer.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCServer.java)。

服务器代码相当简单：

- 像往常一样，我们从建立连接、通道和声明队列开始。
- 我们可能想要运行多个服务器进程。为了在多个服务器上平均分配负载，我们需要在 `channel.basicQos` 中设置 `prefetchCount`设置。
- 我们使用`basicConsume`访问队列，在队列中我们以对象 ( `DeliverCallback` )的形式提供回调，该回调将完成工作并将响应发回。

我们的 RPC 客户端的代码可以在这里找到：[RPCClient.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCClient.java)。

客户端代码稍微复杂一点：

- 我们建立连接和通道。
- 我们的`call`方法发出实际的 `RPC` 请求。
- 在这里，我们首先生成一个唯一的`correlationId` 编号并保存它——我们的消费者回调将使用这个值来匹配适当的响应。
- 然后，我们为回复创建一个专用的排他队列并订阅它。
- 接下来，我们发布具有两个属性的请求消息： `replyTo`和`correlationId`。
- 在这一点上，我们可以坐下来等待正确的响应到来。
- 由于我们的消费者交付处理发生在一个单独的线程中，我们需要在响应到达之前暂停主线程。使用`BlockingQueue`是一种可能的解决方案。在这里，我们正在创建 容量设置为 `1` 的`ArrayBlockingQueue`，因为我们只需要等待一个响应。
- 消费者正在做一项非常简单的工作，对于每条消费的响应消息，它都会检查相关性`ID` 是否 是我们正在寻找的。如果是这样，它将响应放入`BlockingQueue`。
- 同时主线程正在等待响应从`BlockingQueue` 中获取它。
- 最后我们将响应返回给用户。

发出客户端请求：

```java
RPCClient fibonacciRpc = new RPCClient();

System.out.println(" [x] Requesting fib(30)");
String response = fibonacciRpc.call("30");
System.out.println(" [.] Got '" + response + "'");

fibonacciRpc.close();
```

现在是查看[RPCClient.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCClient.java)和[RPCServer.java 的](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCServer.java)完整示例源代码（包括基本异常处理）的好时机。

像往常一样编译和设置类路径（参见[教程一](#HelloWorld)）：

```bash
javac -cp $CP RPCClient.java RPCServer.java
```

我们的 RPC 服务现已准备就绪。我们可以启动服务器：

```bash
java -cp $CP RPCServer
# => [x] Awaiting RPC requests
```

要请求斐波那契数，请运行客户端：

```bash
java -cp $CP RPCClient
# => [x] Requesting fib(30)
```

这里介绍的设计并不是 `RPC` 服务的唯一可能实现，但它具有一些重要的优点：

- 如果 `RPC` 服务器太慢，您可以通过运行另一个服务器来扩展。尝试在新控制台中运行第二个`RPCServer`。
- 在客户端，`RPC` 只需要发送和接收一条消息。不需要像`queueDeclare`这样的同步调用 。因此，对于单个 `RPC` 请求，`RPC` 客户端只需要一次网络往返。

我们的代码仍然非常简单，并没有尝试解决更复杂（但重要）的问题，例如：

- 如果没有服务器在运行，客户端应该如何反应？
- 客户端是否应该为 `RPC` 设置某种超时？
- 如果服务器出现故障并引发异常，是否应该将其转发给客户端？
- 在处理之前防止无效的传入消息（例如检查边界、类型）。

> 如果您想进行试验，您可能会发现[管理 UI](https://www.rabbitmq.com/management.html)对查看队列很有用。

# 发布者确认

> 使用发布者确认进行可靠的发布

[发布者确认](https://www.rabbitmq.com/confirms.html#publisher-confirms) 是实现可靠发布的 RabbitMQ 扩展。当在通道上启用发布者确认时，客户端发布的消息由代理异步确认，这意味着它们已在服务器端得到处理。

## 概述

在本教程中，我们将使用发布者确认来确保发布的消息已安全到达代理。我们将介绍几种使用发布者确认的策略并解释它们的优缺点。

## 在管道上开启发布者确认

发布者确认是 `AMQP 0.9.1` 协议的 `RabbitMQ` 扩展，因此默认情况下不启用它们。使用`confirmSelect`方法在频道级别启用发布者确认：

```java
Channel channel = connection.createChannel();
channel.confirmSelect();
```

必须在您希望使用发布者确认的每个管道上调用此方法。确认应该只启用一次，而不是为每个发布的消息启用。

## 策略#1：单独发布消息

让我们从最简单的使用确认发布的方法开始，即发布消息并同步等待其确认：

```java
while (thereAreMessagesToPublish()) {
    byte[] body = ...;
    BasicProperties properties = ...;
    channel.basicPublish(exchange, queue, properties, body);
    // uses a 5 second timeout
    channel.waitForConfirmsOrDie(5_000);
}
```

在前面的示例中，我们像往常一样发布一条消息，并使用`Channel#waitForConfirmsOrDie(long)`方法等待其确认。一旦消息得到确认，该方法就会返回。如果消息在超时内没有得到确认或者它被 `nack-ed`（意味着中间件由于某种原因无法处理它），该方法将抛出异常。异常的处理通常包括记录错误消息或重试发送消息。

不同的客户端库有不同的方式同步处理发布者确认，所以一定要仔细阅读你正在使用的客户端的文档。

这种技术非常简单，但也有一个主要缺点：它**显著减慢了发布速度**，因为消息的确认会阻止所有后续消息的发布。这种方法不会提供超过每秒数百条已发布消息的吞吐量。尽管如此，这对于某些应用程序来说已经足够了。

> **发布者确认是异步的吗？**
>
> 我们在开头提到中间件异步确认已发布的消息，但在第一个示例中，代码同步等待直到消息被确认。客户端实际上异步接收确认并相应地解除对`waitForConfirmsOrDie`的调用 。将`waitForConfirmsOrDie`视为一个同步助手，它依赖于幕后的异步通知。

## 策略#2：批量发布消息

为了改进我们之前的示例，我们可以发布一批消息并等待整批消息得到确认。以下示例使用 100 个批次：

```java
nt batchSize = 100;
int outstandingMessageCount = 0;
while (thereAreMessagesToPublish()) {
    byte[] body = ...;
    BasicProperties properties = ...;
    channel.basicPublish(exchange, queue, properties, body);
    outstandingMessageCount++;
    if (outstandingMessageCount == batchSize) {
        ch.waitForConfirmsOrDie(5_000);
        outstandingMessageCount = 0;
    }
}
if (outstandingMessageCount > 0) {
    ch.waitForConfirmsOrDie(5_000);
}
```

与等待单个消息的确认相比，等待一批消息得到确认大大提高了吞吐量（使用远程 `RabbitMQ` 节点最多 20-30 次）。存在一个缺点是我们不知道在失败的情况下到底出了什么问题，所以我们可能不得不在内存中保留一整批来记录一些有意义的东西或重新发布消息。而且这个方案还是同步的，所以会阻塞消息的发布。

## 策略#3：异步处理发布者确认

中间件异步确认已发布的消息，只需要在客户端注册一个回调即可收到这些确认的通知：

```java
Channel channel = connection.createChannel();
channel.confirmSelect();
channel.addConfirmListener((sequenceNumber, multiple) -> {
    // code when message is confirmed
}, (sequenceNumber, multiple) -> {
    // code when message is nack-ed
});
```

有 2 个回调：一个用于确认消息，另一个用于未确认消息（可以被代理视为丢失的消息）。每个回调有2个参数：

- `sequenceNumber(序列号)`：标识已确认或未确认消息的编号。我们很快就会看到如何将它与发布的消息相关联。
- `multiple`：这是一个布尔值。如果为 `false`，则仅确认/未确认一条消息，如果为 `true`，则确认/未确认 所有具有较小或相等`sequenceNumber`的消息。

发布前可以通过`Channel#getNextPublishSeqNo()`获取序列号：

```java
int sequenceNumber = channel.getNextPublishSeqNo());
ch.basicPublish(exchange, queue, properties, body);
```

将消息与序列号相关联的一种简单方法是使用映射。假设我们要发布字符串，因为它们很容易变成用于发布的字节数组。下面是一个代码示例，它使用映射将发布序列号与消息的字符串正文相关联：

```java
ConcurrentNavigableMap<Long, String> outstandingConfirms = new ConcurrentSkipListMap<>();
// ... code for confirm callbacks will come later
String body = "...";
outstandingConfirms.put(channel.getNextPublishSeqNo(), body);
channel.basicPublish(exchange, queue, properties, body.getBytes());
```

发布代码现在使用地图跟踪出站消息。我们需要在确认到达时清理此映射，并在消息未确认时执行诸如记录警告之类的操作：

```java
ConcurrentNavigableMap<Long, String> outstandingConfirms = new ConcurrentSkipListMap<>();
ConfirmCallback cleanOutstandingConfirms = (sequenceNumber, multiple) -> {
    if (multiple) {
        ConcurrentNavigableMap<Long, String> confirmed = outstandingConfirms.headMap(
          sequenceNumber, true
        );
        confirmed.clear();
    } else {
        outstandingConfirms.remove(sequenceNumber);
    }
};

channel.addConfirmListener(cleanOutstandingConfirms, (sequenceNumber, multiple) -> {
    String body = outstandingConfirms.get(sequenceNumber);
    System.err.format(
      "Message with body %s has been nack-ed. Sequence number: %d, multiple: %b%n",
      body, sequenceNumber, multiple
    );
    cleanOutstandingConfirms.handle(sequenceNumber, multiple);
});
// ... publishing code
```

上一个示例包含一个回调，用于在确认到达时清理地图。请注意，此回调处理单个和多个确认。当确认到达时使用此回调（作为`Channel#addConfirmListener`的第一个参数 ）。未确认消息的回调检索消息正文并发出警告。然后它重新使用之前的回调来清理未完成确认的映射（无论消息是确认还是未确认，它们在映射中的相应条目都必须被删除。）

> **如何追踪未完成的确认？**
>
> 我们的示例使用`ConcurrentNavigableMap`来跟踪未完成的确认。出于多种原因，这种数据结构很方便。它允许轻松地将序列号与消息（无论消息数据是什么）相关联，并轻松地将条目清理到给定的序列 ID（以处理多个确认/未确认）。最后，它支持并发访问，因为在客户端库拥有的线程中调用确认回调，该线程应与发布线程保持不同。
>
> 除了使用复杂的映射实现之外，还有其他方法可以跟踪未完成的确认，例如使用简单的并发哈希映射和变量来跟踪发布序列的下限，但它们通常更复杂，不属于教程。

综上所述，异步处理发布者确认通常需要以下步骤：

- 提供一种将发布序列号与消息相关联的方法。
- 在通道上注册一个确认侦听器，以便在发布者确认/未确认到达时收到通知以执行适当的操作，例如记录或重新发布确认消息。在这个步骤中，序列号到消息的相关机制也可能需要一些清理。
- 在发布消息之前跟踪发布序列号。

> **重新发布未确认消息？**
>
> 从相应的回调中重新发布 未确认 消息可能很诱人，但应该避免这种情况，因为确认回调是在` I/O` 线程中调度的，而通道不应该在该线程中执行操作。更好的解决方案是将消息放入由发布线程轮询的内存队列中。像`ConcurrentLinkedQueue` 这样的类很适合在确认回调和发布线程之间传输消息。

## 摘要

在某些应用程序中，确保已发布的消息发送到中间件是必不可少的。发布者确认是 `RabbitMQ` 的一项功能，有助于满足此要求。发布者确认本质上是异步的，但也可以同步处理它们。没有明确的方法来实现发布者确认，这通常归结为应用程序和整个系统中的约束。典型的技术有：

- 单独发布消息，同步等待确认：简单，但吞吐量非常有限。
- 批量发布消息，批量同步等待确认：简单，合理的吞吐量，但很难推理出什么时候出现问题。
- 异步处理：最佳性能和资源使用，错误情况下的良好控制，但可以参与以正确实施。

## 总结

该[PublisherConfirms.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/PublisherConfirms.java) 类包含了我们所覆盖的技术代码。我们可以编译它，按原样执行它并查看它们各自的性能：

```bash
javac -cp $CP PublisherConfirms.java
java -cp $CP PublisherConfirms
```

输出将如下所示：

```bash
Published 50,000 messages individually in 5,549 ms
Published 50,000 messages in batch in 2,331 ms
Published 50,000 messages and handled confirms asynchronously in 4,054 ms
```

如果客户端和服务器位于同一台机器上，您计算机上的输出应该类似。单独发布消息的性能不如预期，但与批量发布相比，异步处理的结果有点令人失望。

发布者确认非常依赖网络，所以我们最好尝试使用远程节点，这更现实，因为客户端和服务器通常不在生产中的同一台机器上。 `PublisherConfirms.java`可以轻松更改为使用非本地节点：

```java
static Connection createConnection() throws Exception {
    ConnectionFactory cf = new ConnectionFactory();
    cf.setHost("remote-host");
    cf.setUsername("remote-user");
    cf.setPassword("remote-password");
    return cf.newConnection();
}
```

重新编译类，再次执行，等待结果：

```bash
Published 50,000 messages individually in 231,541 ms
Published 50,000 messages in batch in 7,232 ms
Published 50,000 messages and handled confirms asynchronously in 6,332 ms
```

我们看到单独发布现在表现得非常糟糕。但是在客户端和服务器之间的网络中，批量发布和异步处理现在执行类似，对于发布者的异步处理确认有一个小优势。

请记住，批量发布实施起来很简单，但在发布者否定确认的情况下，很难知道哪些消息无法到达中间件。异步处理发布者确认更需要实现，但提供更好的粒度和更好的控制，当发布的消息被 不确认 时要执行的操作。
