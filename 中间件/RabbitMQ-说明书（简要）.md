[TOC]

## RabbitMQ介绍
> 一个开源的消息代理和队列服务器，（通过elang语言来开发）通过普通的协议(AMQP协议)来完成不同应用之间的数据共享。文档完备。

场景使用：
- 秒杀；
- 短信发送；

## 安装与使用
```bash
# 依赖包
yum install build-essential openssl openssl-devel unixODBC unixODBC-devel make gcc gcc- c++ kernel-devel m4 ncurses-devel tk tc xz

# 需要elang
wget www.rabbitmq.com/releases/erlang/erlang-18.3-1.el7.centos.x86_64.rpm
wget http://repo.iotti.biz/CentOS/7/x86_64/socat-1.7.3.2-5.el7.lux.x86_64.rpm
wget www.rabbitmq.com/releases/rabbitmq-server/v3.6.5/rabbitmq-server-3.6.5-1.noarch.rpm

```

#### 用户创建
###### 命令
###### 后台

## AMQP
> 二进制协议，是一个应用层协议的规范,有很多不同的消息中间件产品遵循该规范

#### 结构
Server：是消息队列节点
Virtual Host：虚拟主机
Exchange：交换机(消息投递到交换机上)，有多种，根据对Route Key不同的匹配类型分类
Message Queue：被消费者监听消费，用于绑定Exchange上，可以绑定多个

一个Server有多个Virtual Host（建议一个项目对应一个），一个Virtual Host有多个Exchange，多个Message Queue可以绑定到多个Exchange，消息投递到Exchang时，根据自身的匹配规则，根据Route Key匹配分发到各Message Queue上。等待消费者消费这些消息。

#### 交换机
###### Direct Exchange
这里的消息都会被投递到与Route Key名称(与队列名称)相同的Queue上

###### Topic Exchange
通过通配符来匹配。通配符的规则是：`*`符号可以匹配1个单词，`#`符号可以匹配1到多个单词
例子：
log.* 可以匹配log.a 但是不可以匹配log.a.b
log.# 可以匹配log.a log.a.b log.a.b；

###### Fanout Exchange
只要队列和交换机绑定的那么消息就会被分发到队列上。不需要经过Route Key，速度最快

#### Spring例子说明
```xml
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>ampq-client</artifactId>
    <version>3.6.5</version>
</dependency>
```

```java
// 生产者
public static void main(String[] args) throws IOException, TimeoutException {

// 1. 连接工厂ConnectionFactory ip + port
// virtual host
// username password
// connetion timeout

// 2. 创建连接
// 创建信道 channel
// 发布消息：指定交换机 + routeKey +消息

// 3. 关闭信道 连接
}
```

```java
// 消费者
public static void main(String[] args) throws IOException, TimeoutException {

// 1. 连接工厂ConnectionFactory ip + port
// virtual host
// username password
// connetion timeout

// 2. 创建连接
// 创建信道 channel；
// 设置信道：交换机，名称 + 类型；队列，名称 + 是否持久化 + 否独占 ...
// 交换机 队列绑定

// 3. 创建消费者 
// 3. 或者新写消费者类，继承DefaultConsumer，实现handleDelivery
// 将消费者传入信道channel.basicComsume(...)，配置是否自动ack等功能

// 4. 消费者开始消费
}
```

```java
// 属性，生产者发送消息可以带上属性AMQP.BasicProperties，其中header可以带上map键值对
AMQP.BasicProperties basicProperties = new AMQP.BasicProperties.Builder().
deliveryMode(2). // 2为持久化,1 不是持久化 
expiration("10000"). // 消息10s后过期
appId("测试appid").
clusterId("测试集群id").
contentType("application/json").
contentEncoding("UTF-8").
headers(extraMap).
build();
```
注：生产者、消费者端均可声明队列、交换机

## 功能
#### 开启消息投递模式（生产者）
场景：可以解决生产者到Mq的消息丢失问题，使用异步confirm效果较好。
`channel.confirmSelect()`，即ConfirmListener。并添加自定义监听器（继承ConfirmListener），实现handleAck、handleNack。
当回调handleAck表示MQ服务接收到了消息。

#### 手动、自动签收（消费者）
场景：消费者没有启动，MQ堆积了很多消息。手动签收可以解决生产者不丢失消息。
自动签收会导致消费者启动会一次性接收完，出现资源告急。手动签收则会处理完一条再收下一条。
`channel.basicNack(...)`：不确认消息的消费。入参有requeue的选择，如果是true，则会重回队列，容易死循环，建议不重回队列。

#### 限流QoS（消费者）
`channel.basicQos(0, 1, false)`
第一个入参：每条消息的大小
第二个入参：每次推送多少条消息
第三个入参：false表示channel级别，true为consume级别

此时，自动签收需要关闭。继承DefaultConsumer的消费完成后需要ack回应，否则不会消费接下来的消息。

#### 不可达消息（生产者）
用于接收不可达消息的通知，比如Exchange写错了。
`channel.basicPublish(...)`：发布消息时，若设置mandatory为true，则回调ReturnListener做处理，若设置false，则MQ会自动删除消息

#### 死信队列（DLX）
会转到死信交换机上的情况：
- 消费者拒绝消费并且不重入队列（nack、reject）；
- 没有消费者消费；
- 消息过期；
- 队列超过最大长度

如何设置：
1. 普通队列绑定上死信交换机：`channel.queueDeclare(...)`，x-dead-letter-exchange=死信队列交换机名称，x-max-length=普通队列长度；
2. 声明死信队列交换机、队列，并将其绑定；

###### 延迟队列
场景：订单在限定时间内做支付
建立一个队列，不能被监听，x min后过期，就可以转到死信队列上。消费者监听这个死信队列。

可以基于死信队列做延迟队列：TTL + DLX

延迟队列：
自定义一个延迟交换机
```java
    @Bean
    public CustomExchange delayExchange() {
        Map<String, Object> args = new HashMap<>();
        arg.put("x-delayed-type", "direct");
        return new CustomExchange("delayExchange", "x-delayed-message", true, false, args);
    }
```
```java
    // product端， MessagePostProccesor是额外的处理
    rabbitTemplate.convertAndSend("交换机", "route key", 消息, new MessagePostProccesor() {
        @override
        public Message postProcessMessage() throws AmqpException{
            // 延迟的时间
            message.getMessageProperties().setHeader("x-delay", 10000);
        }
        return message;
    })
```

## 消息的可靠性投递
> 生产者 -> broker -> 消费者，前部分和后部分均ack

#### 方案1：定时任务
使用一个消息表记录所有发送的消息，并记录每条消息的状态(`status  0=初始状态 1=mq服务签收成功 2=mq服务签收失败 3=消费端消费成功 4=消费端消费失败`)
定时任务扫描5分钟之内`status!=3`的，重试5次（自定义次数）。
1. 生产者发送消息时业务数据和消息数据均要入库成功，消息表初始化状态（业务数据和消息数据要放在不同的库中。此时涉及到跨库，分布式事务JTA的问题）（`status=0`）；
2. mq接收到消息后根据是否能被消费做回调；
    - mq未收到消息会保持`status=0`，依赖定时任务检查，超过5次再人工干预；
    - 可达消息会进入ConfirmListener，返回ack，true表示mq签收成功（`status=1`），false表示签收失败（`status=2`）；
    - 不可达消息会进入ReturnListener，只有写错交换机会出现这种情况（`status=2`）；
    - **如果回调失败怎么办？会重复发送消息，需要消费者端保证幂等性，即保证消费同一个消息，结果是一致的；**
3. 消费者业务处理，处理成功（`status=3`），处理失败（`status=4`），这里注意消费了消息都要从队列清掉数据，即返回ack，而生产者端会重复发送消费失败的消息；

#### 方案2：可靠性检查，延迟队列
延迟队列可以用插件，不需要自己实现
1. 上游业务数据入库，并发送消息；
2. 上游业务再发送一条延时消息；
3. 下游业务接收到消息，成功处理则发送成功的消息；
4. 回调检查服务监听下游业务处理消息成功，对消息入库，标记为消费成功；
5. 回调检查服务再监听到延时的消息，检查第三步的操作是否成功；
6. 如果检查是没有的，则回调上游业务重发消息；

这方案比第一个架构会更复杂，多了一个检查服务，两个消息队列。并且服务之间需要有一个messageId来标识每一条消息，并共同维护消息的状态

## Spring & Springboot
#### Spring
```xml
<dependency>
    <groupId>com.springframework.amqp</groupId>
    <artifactId>spring-rabbit</artifactId>
    <version>2.0.8.RELEASE</version>
</dependency>
```
将需要用到的连接工厂、RabbitAdmin（把setAutoStartup设置为true，才能在spring启动时进行加载）、交换机、队列、绑定、RabbitTemplate、消息监听对象等交由Spring管理，即在一个类以@Configuration注解，每个方法@Bean。

创建消费者
```java
    @Bean
    public SimpleMessageListenerContainer simpleMessageListenerContainer() {
        SimpleMessageListenerContainer simpleMessageListenerContainer = new SimpleMessageListenerContainer();
        simpleMessageListenerContainer.setqueues(队列对象1, 队列对象1, ...);
        // 监听的并发量
        simpleMessageListenerContainer.setconcurrentComsumers(5);
        // 监听的最大并发量
        simpleMessageListenerContainer.setMaxconcurrentComsumers(10);
        // 自动签收
        simpleMessageListenerContainer.setAcknowledgeMode(AcknowledgeMode.AUTO);
        // 不重回队列
        simpleMessageListenerContainer.setDefaultRequeueRejected(false);

        // MessageListenerAdapter设置回调消费者的方法
        // 1. 在收到消息会固定调用方法名为handleMessage的方法。SelfCustomMsgDelegate是自定义的，只要里面有方法名handleMessage即可
        MessageListenerAdapter messageListenerAdapter = new MessageListenerAdapter(new SelfCustomMsgDelegate());
        // 2. 如果不希望用MessageListenerAdapter固定的回调方法，可以设置messageListenerAdapter.setDefaultLitenerMethod("自己指定方法名")
        // 3. 可以通过Map，键值对绑定队列 方法名，将map设置messageListenerAdapter.setQueueOrTagToMethodName(map);

        // MessageListenerAdapter设置处理不同格式的入参
        // 1. Json，设置转换器，而消费者对应的处理方法对参数用Map来接收
        Jackson2JsonMessageConverter jackson2JsonMessageConverter = new Jackson2JsonMessageConverter();
        messageListenerAdapter.setMessageConverter(jackson2JsonMessageConverter);
        // 2. Java Object，此时生产者发送时消息对象Message需要带上MessageProperties；
        // 并设置Header：messageProperties.getHeaders().put("__TypeId__", "类的路径，如com.xx.xx.xxx")
        Jackson2JsonMessageConverter jackson2JsonMessageConverter = new Jackson2JsonMessageConverter();
        DefaultJackson2JavaTypeMapper javaTypeMapper = new DefaultJackson2JavaTypeMapper();
        javaTypeMapper.setTrustedPackages("类的路径，如com.xx.xx.xxx");
        jackson2JsonMessageConverter.setJavaTypeMapper(javaTypeMapper);
        messageListenerAdapter.setMessageConverter(jackson2JsonMessageConverter);

        // 3. 图片、文件，而消费者对应的处理方法对参数用File来接收
        ContentTypeDelegatingMessageConverter messageConverter = new ContentTypeDelegatingMessageConverter();
        messageConverter.addDelegate("img/png", new SelfConverter());
        messageConverter.addDelegate("img/jpg", new SelfConverter());
        messageConverter.addDelegate("application/word", new SelfConverter());
        messageConverter.addDelegate("word", new SelfConverter());
        messageListenerAdapter.setMessageConverter(messageConverter);

        simpleMessageListenerContainer.setMessageListener(messageListenerAdapter);
        return simpleMessageListenerContainer;
    }
/**
 * 自己实现图片 文件的转换器
 */
public class SelfConverter implements MessageConverter{
    @Override
    public Object fromMessage(Message message) throws MessageConversionException {
        // 原理：从message获取的是二进制流，做什么转换需要生产者在发送的时候在ContentType带上格式，那么这里可以根据格式做不同的转换
    }

}
```

#### Springboot
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
```yaml
# 生产者端application.properties
# host、port、virtual host、username、password的设置
# 开启confirmListener
# 开启returnListener
# 设置mandatory
# 设置连接超时
```
```yaml
# 消费者端application.properties
# host、port、virtual host、username、password的设置
# 开启手动签收
# 线程并发数、最大并发数

```
`@RabbitListener(queues = {"queueName1"})`

## 集群
1. 先按照一台的方式配置多台；
2. host文件修改，本地DNS寻址；
3. 选择一台做主节点，进入/var/lib/rabbitmq目录下，把/var/lib/rabbitmq/.erlang.cookie文件的权 限修改为777`chmod 777 /var/lib/rabbitmq/.erlang.cookie`，把.erlang.cookie文件copy到各个节点下,最后把所有cookie文件权限还原为400即可；
4. 每台执行集群启动命令：`rabbitmq-server -detached`；
5. 从节点执行`rabbitmqctl stop_app`、`rabbitmqctl join_cluster user@192.168.1.100`、`rabbitmqctl start_app`；
6. 修改集群名称: 在主节点上执行命令( /usr/local) `rabbitmqctl set_cluster_name rabbitmq_cluster_dx`；
7. 查看集群状态：`rabbitmqctl cluster_status`；
8. 配置镜像队列，在任意节点上执行 `rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'`；

其他：剔除节点命令`rabbitmqctl forget_cluster_node user@192.168.1.111`

此时rabbitMQ集群搭建完成，但不会自动切换，需要使用HaProxy

#### HAProxy
是一款提供高可用性、负载均衡以及基于TCP和HTTP应用的代理软件，HAProxy是完全免费的、借助HAProxy可以快速并且可靠的提供基于TCP和HTTP应用的代理解决方案。 HAProxy适用于那些负载较大的web站点，这些站点通常又需要会话保持或七层处理。
HAProxy可以支持数以万计的并发连接,并且HAProxy的运行模式使得它可以很 简单安全的整合进架构中，同时可以保护web服务器不被暴露到网络上。

**注意**：如果需要HAProxy高可用，则需要用到KeepAlived

```bash
# 安装&解压
yum install gcc vim wget
wget http://www.haproxy.org/download/1.6/src/haproxy-1.6.5.tar.gz
tar -zxvf haproxy-1.6.5.tar.gz -C /usr/local

# 编译
cd /usr/local/haproxy-1.6.5
make TARGET=linux31 PREFIX=/usr/local/haproxy

# 安装
make install PREFIX=/usr/local/haproxy

# 创建一个haproxy的目录 用于存放配置文件
mkdir /etc/haproxy
# 赋权
groupadd -r -g 149 haproxy
useradd -g haproxy -r -s /sbin/nologin -u 149 haproxy
# 创建配置文件
touch /etc/haproxy/haproxy.cfg
```

haproxy.cfg配置文件详解:
```
#lgging options
global
    log 127.0.0.1 local0 info
    maxconn 5120
    chroot /usr/local/haproxy
    uid 99
    gid 99
    daemon
    quiet
    nbproc 20
    pidfile /var/run/haproxy.pid
    
defaults
    log global
    # 使用4层代理模式，”mode http”为7层代理模式
    mode tcp
    # if you set mode to tcp,then you nust change tcplog into httplog
    option tcplog
    option dontlognull
    retries 3
    option redispatch
    maxconn 2000
    contimeout 5s
    # 客户端空闲超时时间为 60秒 则HA 发起重连机制
    clitimeout 60s
    # 服务器端链接超时时间为 15秒 则HA 发起重连机制
    srvtimeout 15s
    #front-end IP for consumers and producters
    
listen rabbitmq_cluster
    # 监听的端口
    bind 0.0.0.0:5672
    #配置TCP模式
    mode tcp
    # 简单的轮询
    balance roundrobin
    # 重要 防止应用程序连接关闭
    timeout client 3h
    timeout server 3h
    # rabbitmq集群节点配置
    # inter 每隔五秒对mq集群做健康检查，2次正确证明服务器可用，2次失败证明服务器不可用，并且配置主备机制
    server dx100 192.168.1.100:5672 check inter 5000 rise 2 fall 2
    server dx101 192.168.1.101:5672 check inter 5000 rise 2 fall 2
    server dx102 192.168.1.102:5672 check inter 5000 rise 2 fall 2

#配置haproxy web监控，查看统计信息
listen stats
    bind 192.168.1.100:8100
    mode http
    option httplog
    stats enable
    #设置haproxy监控地址为http://192.168.1.100:8100/rabbitmq-stats
    stats uri /rabbitmq-stats
    stats refresh 5s
```