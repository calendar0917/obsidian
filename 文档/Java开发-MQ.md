---
title: "MQ"
subtitle: "MQ基础"
summary: "记录MQ的学习"
description: "记录MQ的学习"
date: 2025-10-02
lastmod: 2025-10-02
draft: false
toc:
 enable: true
weight: false
categories: ["开发"]
tags: ["技术"]
---

## 初识 MQ

### 服务调用类型

#### 同步调用

服务 A 同时请求多个服务，导致服务种类的不断扩增，服务的等待耗时增加。

缺点：

- 扩展性差

- 性能下降
- 级联失败问题

#### 异步调用

基于消息通知的方式，包含：

- 消息发送者：即原来的调用者
- 消息接收者：接收和处理消息的人
- 消息代理：管理、暂存转发消息

不再同步调用业务关联度第的服务，而是分别发送消息到 Broker

- 接触耦合，扩展性强

- 无需等待，性能好
- 故障隔离
- 缓存消息，流量削锋填谷

#### 技术选型

RabbitMQ、ActiveMQ、RocketMQ、Kafka

### 部署

Docker安装即可

整体架构：

- publisher：消息发送者
- consumer：消息消费者
- queue：队列，存储消息
- exchange：交换机，负责路由消息
- vertual-host：虚拟主机，用于数据隔离

消息发送给交换机，再由交换机分发给对应的 queue，交换机没有消息存贮的能力。

## Java客户端

AMQP：无协议传输

- 封装为 Spring AMQP
- 再封装为 SpringRabbit，`RabbitTemplate`包装类

### 收发消息

发送：

```java
@Autowired
private RabbitTemplate rabbitTemplate;

@Test
public void testSimpleQueue() {
    // 队列名称
    String queueName = "simple.queue";
    // 消息
    String message = "hello, spring amqp!";
    // 发送消息
    rabbitTemplate.convertAndSend(queueName, message);
}
```

接收：

```Java
@Slf4j
@Component
public class SpringRabbitListener {

    @RabbitListener(queues = "simple.queue")
    public void listenSimpleQueueMessage(String msg) throws InterruptedException {
        log.info("spring 消费者接收到消息: [" + msg + "] ");
    }
}
```

### Work Queues

多个消费者绑定到一个队列

- 一条消息只能被一个消费者处理

- 多条消息，默认轮流接收
- 通过添加消费者来处理超量数据

修改配置

- 修改 prefetch 来控制消费者预取的消息数量，使性能高的服务器多处理

### 交换机

#### Fanout

广播模式，将接收到的消息路由到每一个与其绑定的 queue

#### Direct

定向路由，根据规则路由到指定的 queue

- 设置 BindingKey 和 RoutingKey

#### Topic

基于 RoutingKey，但其通常是多个单词的组合，且以`.`分割，可以使用通配符`# *`

### 声明队列交换机

用代码自动完成队列、交换机的创建

#### 基于Bean声明

在消费者端声明：

```Java
@Configuration
public class FanoutConfig {
    // 声明FanoutExchange交换机
    @Bean
    public FanoutExchange fanoutExchange(){
        return new FanoutExchange("hmall.fanout");
    }
    // 声明第1个队列
    @Bean
    public Queue fanoutQueue1(){
        return new Queue("fanout.queue1");
    }
    // 绑定队列和交换机
    @Bean
    public Binding bindingQueue1(Queue fanoutQueue1, FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue1).to(fanoutExchange);
    }
    // ... 略，以相同方式声明第2个队列，并完成绑定
}
```

#### 基于注解声明

优化 Bean 声明中绑定 Key 的冗余代码。

```java
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(name = "direct.queue1"),
        exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
        key = {"red", "blue"}
))
public void listenDirectQueue1(String msg){
    System.out.println("消费者1接收到Direct消息: ["+msg+"] ");
}
```

### 消息转换器

负责将对象转换为字节格式传输

问题：

- 默认序列化由安全风险
- 信息体积变大
- 可读性差

解决：

- 用 Jackson 序列转换器

- 注意收发一致

## 进阶

改进消息可靠性问题

### 发送者可靠性

#### 重连

由于网络波动，可能出现发送者连接 MQ 失败。

配置中开启重连机制即可

- 注意性能损耗
- 使用合理的重连机制

#### 确认

MQ 接收到消息后，返回 ACK 给发送者。（对性能影响较大）

- 不同的返回情况、强度
- 其他情况返回 NACK，告知投递失败

使用：

- 先开启配置 `public confirm type`

- 配置类

```Java
@Slf4j
@AllArgsConstructor
@Configuration
public class MqConfig {
    private final RabbitTemplate rabbitTemplate;

    @PostConstruct //只在启动时初始化依次
    public void init(){
        rabbitTemplate.setReturnsCallback(new RabbitTemplate.ReturnsCallback() {
            @Override
            public void returnedMessage(ReturnedMessage returned) {
                log.error("触发return callback,");
                log.debug("exchange: {}", returned.getExchange());
                log.debug("routingKey: {}", returned.getRoutingKey());
                log.debug("message: {}", returned.getMessage());
                log.debug("replyCode: {}", returned.getReplyCode());
                log.debug("replyText: {}", returned.getReplyText());
            }
        });
    }
}
```

- 发送方：

```Java
@Test
void testPublisherConfirm() throws InterruptedException {
    // 1.创建CorrelationData
    CorrelationData cd = new CorrelationData();
    // 2.给Future添加ConfirmCallback
    cd.getFuture().addCallback(new ListenableFutureCallback<CorrelationData.Confirm>() {
        @Override
        public void onFailure(Throwable ex) {
            // 2.1.Future发生异常时的处理逻辑，基本不会触发
            log.error("handle message ack fail", ex);
        }

        @Override
        public void onSuccess(CorrelationData.Confirm result) {
            // 2.2.Future接收到回执的处理逻辑，参数中的result就是回执内容
            if(result.isAck()){ // result.isAck()是boolean类型，true代表ack回执，false代表nack回执
                log.debug("发送消息成功，收到 ack!");
            }else{ // result.getReason()是String类型，返回nack时的异常描述
                log.error("发送消息失败，收到 nack，reason：{}", result.getReason());
            }
        }
    });
    // 3.发送消息
    rabbitTemplate.convertAndSend("hmall.direct", "red1", "hello", cd);
}
```

若发送失败，则尝试重发

### MQ可靠性

问题：

- 数据丢失

- 内存空间有限，可能导致消息阻塞、堆积

#### 数据持久化

交换机、队列、消息

- 消息：内存到上限后，才写出到磁盘，阻塞
  - 一直写出到磁盘

#### Lazy Queue

惰性队列

- 接到消息不再写到内存，直接存入磁盘
- 消费消息时，从磁盘中读取并加载到内存

### 消费者可靠性

#### 确认

消费者处理消息结束后，向 MQ 发送回执，告知消息处理状态

- ack：成功处理
  - 配置`acknowledge-mode
  - none 接到后直接返回。不安全
  - manual 手动编写返回逻辑
  - auto
- nack：处理失败，需要重发
- reject：处理失败并拒绝，MQ 从队列中删除该消息

#### 失败重试

问题：消费者反复调用 MQ 导致性能损耗

解决：消费者出现异常时利用本地调试机制，无需调用 queue

重试耗尽后的策略

- 直接 reject（默认）
- 返回 nack，重新入队
- 将失败消息投递到指定的交换机

#### 业务幂等

程序开发时，同一个业务执行一次和多次对业务状态的影响是一致的。用于确保消息不被多次执行。

解决方案：

1. 给每个消息设置唯一 id ，配置`SetMessageId`，然后将 id 写入数据库

2. 业务判断：基于业务本身
   - 保证服务间一致性

### 延迟消息

实现一致性的兜底方案。

发送者发送消息时指定时间，消费者在指定时间后才收到消息

- 如支付超时取消

#### 死信交换机

死信：

- requeue = false
- 消息无人消费、过期 --> 用于实现延迟消息
- 消息堆积满了，最早的消息成为死信

死信交换机：

- 接收死信

#### 消息延迟插件

RabbitMQ的插件，docker部署

- 计时需要占用 CPU，产生资源消耗
- 尽可能延时缩短