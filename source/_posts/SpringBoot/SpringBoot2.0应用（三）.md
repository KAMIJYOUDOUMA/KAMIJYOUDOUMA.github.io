---
title: SpringBoot2.0应用（三）：SpringBoot2.0整合RabbitMQ
categories: SpringBoot
tags:
  - SpringBoot
  - 应用
comments: true
abbrlink: 20004
date: 2018-04-11 15:51:30
---

### 如何整合RabbitMQ
#### 1、添加spring-boot-starter-amqp
```
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-amqp</artifactId>
		</dependency>
```
#### 2、添加配置
```
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.publisher-confirms=true
spring.rabbitmq.dynamic=true
spring.rabbitmq.cache.connection.mode=channel
```
#### 3、注入队列
```
@Configuration
public class RabbitConfig {
    @Bean
    public Queue Queue() {
        return new Queue("hello");
    }
}
```
#### 4、创建操作数据的Repository对象
```
interface CityRepository extends Repository<City, Long> {

	Page<City> findAll(Pageable pageable);

	Page<City> findByNameContainingAndCountryContainingAllIgnoringCase(String name,
			String country, Pageable pageable);

	City findByNameAndCountryAllIgnoringCase(String name, String country);

}
```
#### 5、创建消费者
```
@Component
public class RabbitConsumer {
    @RabbitHandler
    @RabbitListener(queues = "hello")
    public void process(@Payload String foo) {
        System.out.println(new Date() + ": " + foo);
    }
}
```
#### 6、启动主类
```
@SpringBootApplication
@EnableScheduling
public class AmqpApplication {
    public static void main(String[] args) {
        SpringApplication.run(AmqpApplication.class, args);
    }
}
```
控制台输出：
```
Sun Sep 30 16:30:35 CST 2018: hello
```
到此，一个简单的`SpringBoot2.0`集成`RabbitMQ`就完成了。
熟悉`RabbitMQ`的小伙伴们应该知道，`RabbitMQ`在一般的队列基础上，增加了`ExChange`的概念。`ExChange`有四种类型：Direct, Topic, Headers and Fanout。其中Headers实际很少使用，Direct较为简单。接下来将详细介绍如何使用topic和Fanout。
### Topic Exchange
#### 1、配置Topic规则
```
@Configuration
public class TopicRabbitConfig {

    @Bean
    public Queue queueMessage1() {
        return new Queue(MQConst.TOPIC_QUEUENAME1);
    }

    @Bean
    public Queue queueMessage2() {
        return new Queue(MQConst.TOPIC_QUEUENAME2);
    }

    @Bean
    TopicExchange exchange() {
        return new TopicExchange(MQConst.TOPIC_EXCHANGE);
    }

    @Bean
    Binding bindingExchangeMessage(Queue queueMessage1, TopicExchange exchange) {
        // 将队列1绑定到名为topicKey.A的routingKey
        return BindingBuilder.bind(queueMessage1).to(exchange).with(MQConst.TOPIC_KEY1);
    }

    @Bean
    Binding bindingExchangeMessages(Queue queueMessage2, TopicExchange exchange) {
        // 将队列2绑定到所有topicKey.开头的routingKey
        return BindingBuilder.bind(queueMessage2).to(exchange).with(MQConst.TOPIC_KEYS);
    }
}
```
#### 2、配置消费者
```
@Component
public class TopicConsumer {

    @RabbitListener(queues = MQConst.TOPIC_QUEUENAME1)
    @RabbitHandler
    public void process1(String message) {
        System.out.println("queue:topic.message1,message:" + message);
    }

    @RabbitListener(queues = MQConst.TOPIC_QUEUENAME2)
    @RabbitHandler
    public void process2(String message) {
        System.out.println("queue:topic.message2,message:" + message);
    }
}
```
#### 3、生产消息
在Producer类中添加：
```
        // Topic
        rabbitTemplate.convertAndSend(MQConst.TOPIC_EXCHANGE, MQConst.TOPIC_KEYS, "from keys");
        rabbitTemplate.convertAndSend(MQConst.TOPIC_EXCHANGE, MQConst.TOPIC_KEY1, "from key1");
```
再次启动主类，控制台输出：
```
queue:topic.message2,message:from keys
queue:topic.message1,message:from key1
queue:topic.message2,message:from key1
```
### Fanout Exchange
#### 1、配置Fanout规则
```
@Configuration
public class FanoutRabbitConfig {
    @Bean
    public Queue MessageA() {
        return new Queue(MQConst.FANOUT_QUEUENAME1);
    }

    @Bean
    public Queue MessageB() {
        return new Queue(MQConst.FANOUT_QUEUENAME2);
    }

    @Bean
    FanoutExchange fanoutExchange() {
        return new FanoutExchange(MQConst.FANOUT_EXCHANGE);
    }

    @Bean
    Binding bindingExchangeA(Queue MessageA, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(MessageA).to(fanoutExchange);
    }

    @Bean
    Binding bindingExchangeB(Queue MessageB, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(MessageB).to(fanoutExchange);
    }
}
```
#### 2.配置消费者
```
@Component
public class FanoutConsumer {

    @RabbitListener(queues = MQConst.FANOUT_QUEUENAME1)
    @RabbitHandler
    public void process1(String message) {
        System.out.println("queue:fanout.message1,message:" + message);
    }

    @RabbitListener(queues = MQConst.FANOUT_QUEUENAME2)
    @RabbitHandler
    public void process2(String message) {
        System.out.println("queue:fanout.message2,message:" + message);
    }
}
```
#### 3、生产消息
在Producer类中添加：
```
        // FanOut
        rabbitTemplate.convertAndSend(MQConst.FANOUT_EXCHANGE, "", "fanout"); 
```
再次启动主类，控制台输出：
```
queue:fanout.message2,message:fanout
queue:fanout.message1,message:fanout
```
源码地址：[GitHub](https://github.com/KAMIJYOUDOUMA/spring-boot-samples)



 

