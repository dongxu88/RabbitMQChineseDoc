## 概览

rabbitmq使用ConnectionFactory来配置connection的各种参数信息，比如用户名、密码等信息；使用ConnectionFactory

rabbitmq使用channel来操作消息传递协议，使用Connection来打开channels。在Connection上面注册可以响应生命周期的事件处理器，在永不需要时关闭Connection。使用ConnectionFactory来配置Connection的各种参数并初始化Connection。

## Connections 和 Channels

Connection和Channel是核心类，分别代表了AMQP 0-9-1 Connection和channel。

## 连接到RabbitMQ服务端

下面的代码实例了怎么连接到RabbitMQ节点。

```java
ConnectionFactory factory = new ConnectionFactory();
// "guest"/"guest" by default, limited to localhost connections
factory.setUsername(userName);
factory.setPassword(password);
factory.setVirtualHost(virtualHost);
factory.setHost(hostName);
factory.setPort(portNumber);

Connection conn = factory.newConnection();
```

所有的参数都有合理的默认值来让RabbitMQ在本地跑起来，如果没有设置值，如下的参数会被设置为默认值：

| Property     | Default Value                                                |
| :----------- | ------------------------------------------------------------ |
| Username     | "guest" //该用户只能用于建立本地连接                         |
| Password     | "guest"                                                      |
| Virtual host | "/"                                                          |
| Hostname     | "localhost"                                                  |
| port         | 5672 for regular connections, 5671 for [[connections that use TLS](https://www.rabbitmq.com/ssl.html)] |

## 使用URI建立Connection

也可以使用URIS来建立连接：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://userName:password@hostName:portNumber/virtualHost");
Connection conn = factory.newConnection();
```

应用也可以为自己的客户端连接设置一个自定义名字，并且该名字会被展示在管理面UI上。

之后就能使用Connection来创建channel了，Channel可以用于发送和接收消息：

```java
Channel channel = conn.createChannel();
```

## 使用端点列表

在建立连接时可以指定一个服务端列表，如果前面的服务端节点连接失败，程序能够自动的去连接不同的服务端节点。

参考如下代码，提供一个Address数组给ConnectionFactory.newConnection函数：

```java
Address[] addrArr = new Address[]{ new Address(hostname1, portnumber1)
                                 , new Address(hostname2, portnumber2)};
Connection conn = factory.newConnection(addrArr);
```

如果提供了一个ExecutorService，比如factory.newConnection(executorService, addrArr)，那么这个线程池会自动和第一个成功返回的Connection绑定。

## 与RabbitMQ服务端断连

```java
channel.close();
conn.close();
```

**注意**：连接关闭是一个好的实践，但是并不是必须的。在任何情况下，当底层的连接关闭时，channel都会被自动关闭。

## Connection和Channel的生命周期

客户端连接是长久有效的。底层的协议旨在设计和优化一个长时间有效的Connection。这意味着不需要耗费大量的性能开销来为每个消息创建一个新的Connection。

Channel也是设计为长久有效的。但是由于一些自动恢复协议的错误导致channel关闭，channel的生命周期可能比Connection短一点。为每个新的操作来关闭和打开新的Channel虽然可行，但是并不必要。当有犹豫时，请先考虑复用channel。

channel级别的错误，比如从一个不存在的队列中消费消息，会导致channel关闭。一个被关闭的channel就永远不会接收任何消息。channel级别的错误会被记录在RabbitMQ并为该channel生成一个关闭的序列号。

## 客户端提供连接名称

RabbitMQ节点对于客户端信息数量是有数量上限的：

- 客户端的TCP端点

- 客户端证

客户端可以指定一个连接名称：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://userName:password@hostName:portNumber/virtualHost");
// provides a custom connection name
Connection conn = factory.newConnection("app:audit component:event-consumer");
```

## 使用交换器和队列

交换器和队列是RabbitMQ高层协议的基石，客户端应用都需要和他们协同工作。交换器和队列在使用前必须被声明和创建好。

下面示例声明一个交换器和一个服务端命名的队列，并将他们按照指定的路由键绑定到一起：

```java
channel.exchangeDeclare(exchangeName, "direct", true);
String queueName = channel.queueDeclare().getQueue();
channel.queueBind(queueName, exchangeName, routingKey);
```

这会声明如下的两个对象：

- 一个持久的、不会自动删除的、direct类型交换器
- 一个非持久化的、独享的、自动删除的、名字是自动生成的队列

声明一个自定义名称的队列：

```java
channel.exchangeDeclare(exchangeName, "direct", true);
channel.queueDeclare(queueName, true, false, false, null);
channel.queueBind(queueName, exchangeName, routingKey);
```

Channel有很多声明方法都是重载的。这些形式较短的方法都使用了合理的默认值。同时也提供了很多较长形式的函数，这些函数参数较多，可以在必须情况下用来覆盖默认值。

