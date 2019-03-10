---
title: SpringBoot2.0源码分析（三）：整合RabbitMQ分析
categories: SpringBoot
tags:
  - SpringBoot
  - 源码分析
comments: true
abbrlink: 20005
date: 2018-04-12 15:51:30
---

### RabbitMQ自动注入
当项目中存在`org.springframework.amqp.rabbit.core.RabbitTemplate`和`com.rabbitmq.client.Channel`着两个类时，SpringBoot将RabbitMQ需要使用到的对象注册为Bean，供项目注入使用。一起看一下`RabbitAutoConfiguration`类。
```
@Configuration
@ConditionalOnClass({ RabbitTemplate.class, Channel.class })
@EnableConfigurationProperties(RabbitProperties.class)
@Import(RabbitAnnotationDrivenConfiguration.class)
public class RabbitAutoConfiguration {

	@Configuration
	@ConditionalOnMissingBean(ConnectionFactory.class)
	protected static class RabbitConnectionFactoryCreator {
		@Bean
		public CachingConnectionFactory rabbitConnectionFactory(
				RabbitProperties properties,
				ObjectProvider<ConnectionNameStrategy> connectionNameStrategy)
				throws Exception {
			PropertyMapper map = PropertyMapper.get();
			CachingConnectionFactory factory = new CachingConnectionFactory(
					getRabbitConnectionFactoryBean(properties).getObject());
			......
			return factory;
		}
	}

	@Configuration
	@Import(RabbitConnectionFactoryCreator.class)
	protected static class RabbitTemplateConfiguration {

		private final ObjectProvider<MessageConverter> messageConverter;

		private final RabbitProperties properties;

        ......

		@Bean
		@ConditionalOnSingleCandidate(ConnectionFactory.class)
		@ConditionalOnMissingBean
		public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
			PropertyMapper map = PropertyMapper.get();
			RabbitTemplate template = new RabbitTemplate(connectionFactory);
			......
			return template;
		}
        ......
```
`RabbitAutoConfiguration`使和RabbitMQ相关的配置生效，并根据相关属性创建和RabbitMQ的连接，并依赖连接工厂创建`rabbitTemplate`。
`RabbitAutoConfiguration`同时导入了`RabbitAnnotationDrivenConfiguration`，注入了`rabbitListenerContainerFactory`。
### RabbitMQ的Queue，Exchange和Routing Key关系创建
先看一个例子：
```
    @Bean
    Binding bindingExchangeMessage(Queue queue, TopicExchange exchange) {
        // 将队列1绑定到名为topicKey.A的routingKey
        return BindingBuilder.bind(queue).to(exchange).with("routingKey");
    }
```
将queue通过"routingKey"绑定到exchange。
`RabbitAdmin`类的initialize方法会拿出注入的Exchange，Queue和Binding进行声明。
```
	public void initialize() {
	    ......
		Collection<Exchange> contextExchanges = new LinkedList<Exchange>(
				this.applicationContext.getBeansOfType(Exchange.class).values());
		Collection<Queue> contextQueues = new LinkedList<Queue>(
				this.applicationContext.getBeansOfType(Queue.class).values());
		Collection<Binding> contextBindings = new LinkedList<Binding>(
				this.applicationContext.getBeansOfType(Binding.class).values());
	    ......
		final Collection<Exchange> exchanges = filterDeclarables(contextExchanges);
		final Collection<Queue> queues = filterDeclarables(contextQueues);
		final Collection<Binding> bindings = filterDeclarables(contextBindings);
        ......
		this.rabbitTemplate.execute(channel -> {
			declareExchanges(channel, exchanges.toArray(new Exchange[exchanges.size()]));
			declareQueues(channel, queues.toArray(new Queue[queues.size()]));
			declareBindings(channel, bindings.toArray(new Binding[bindings.size()]));
			return null;
		});
	}
```
### 消息发送
以下面的发送为例：
```
    rabbitTemplate.convertAndSend(MQConst.TOPIC_EXCHANGE, MQConst.TOPIC_KEY1, "from key1");
```
这个方法会先对消息进行转换，预处理，最终通过调用`ChannelN`的basicPublish方法提交一个AMQCommand，由AMQCommand完成最终的消息发送。
```
    public void transmit(AMQChannel channel) throws IOException {
        int channelNumber = channel.getChannelNumber();
        AMQConnection connection = channel.getConnection();

        synchronized (assembler) {
            Method m = this.assembler.getMethod();
            connection.writeFrame(m.toFrame(channelNumber));
            if (m.hasContent()) {
                byte[] body = this.assembler.getContentBody();
                connection.writeFrame(this.assembler.getContentHeader()
                        .toFrame(channelNumber, body.length));
                int frameMax = connection.getFrameMax();
                int bodyPayloadMax = (frameMax == 0) ? body.length : frameMax
                        - EMPTY_FRAME_SIZE;

                for (int offset = 0; offset < body.length; offset += bodyPayloadMax) {
                    int remaining = body.length - offset;
                    int fragmentLength = (remaining < bodyPayloadMax) ? remaining
                            : bodyPayloadMax;
                    Frame frame = Frame.fromBodyFragment(channelNumber, body,
                            offset, fragmentLength);
                    connection.writeFrame(frame);
                }
            }
        }
        connection.flush();
    }
```
### 消息接收
先看一个消费的事例：
```
@Component
public class TopicConsumer {

    @RabbitListener(queues = MQConst.TOPIC_QUEUENAME1)
    @RabbitHandler
    public void process1(String message) {
        System.out.println("queue:topic.message1,message:" + message);
    }
}
```
SpringBoot解析`@RabbitListener`和`@RabbitHandler`两个注解的是`RabbitListenerAnnotationBeanPostProcessor`中的buildMetadata方法：
```
	private TypeMetadata buildMetadata(Class<?> targetClass) {
		Collection<RabbitListener> classLevelListeners = findListenerAnnotations(targetClass);
		final boolean hasClassLevelListeners = classLevelListeners.size() > 0;
		final List<ListenerMethod> methods = new ArrayList<>();
		final List<Method> multiMethods = new ArrayList<>();
		ReflectionUtils.doWithMethods(targetClass, method -> {
			Collection<RabbitListener> listenerAnnotations = findListenerAnnotations(method);
			if (listenerAnnotations.size() > 0) {
				methods.add(new ListenerMethod(method,
						listenerAnnotations.toArray(new RabbitListener[listenerAnnotations.size()])));
			}
			if (hasClassLevelListeners) {
				RabbitHandler rabbitHandler = AnnotationUtils.findAnnotation(method, RabbitHandler.class);
				if (rabbitHandler != null) {
					multiMethods.add(method);
				}
			}
		}, ReflectionUtils.USER_DECLARED_METHODS);
		if (methods.isEmpty() && multiMethods.isEmpty()) {
			return TypeMetadata.EMPTY;
		}
		return new TypeMetadata(
				methods.toArray(new ListenerMethod[methods.size()]),
				multiMethods.toArray(new Method[multiMethods.size()]),
				classLevelListeners.toArray(new RabbitListener[classLevelListeners.size()]));
	}
```
@RabbitListener可以放在方法上，也可以放在类上。对于Class级别的Listener，会配合RabbitHandle进行绑定。对于method级别的Listener则不会。
解析完成后，由processListener方法对处理Listener。
```
	protected void processListener(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener, Object bean,
			Object adminTarget, String beanName) {
		endpoint.setBean(bean);
		endpoint.setMessageHandlerMethodFactory(this.messageHandlerMethodFactory);
		endpoint.setId(getEndpointId(rabbitListener));
		endpoint.setQueueNames(resolveQueues(rabbitListener));
		endpoint.setConcurrency(resolveExpressionAsStringOrInteger(rabbitListener.concurrency(), "concurrency"));
		endpoint.setBeanFactory(this.beanFactory);
		endpoint.setReturnExceptions(resolveExpressionAsBoolean(rabbitListener.returnExceptions()));
		String errorHandlerBeanName = resolveExpressionAsString(rabbitListener.errorHandler(), "errorHandler");
		if (StringUtils.hasText(errorHandlerBeanName)) {
			endpoint.setErrorHandler(this.beanFactory.getBean(errorHandlerBeanName, RabbitListenerErrorHandler.class));
		}
		String group = rabbitListener.group();
		if (StringUtils.hasText(group)) {
			Object resolvedGroup = resolveExpression(group);
			if (resolvedGroup instanceof String) {
				endpoint.setGroup((String) resolvedGroup);
			}
		}
		String autoStartup = rabbitListener.autoStartup();
		if (StringUtils.hasText(autoStartup)) {
			endpoint.setAutoStartup(resolveExpressionAsBoolean(autoStartup));
		}

		endpoint.setExclusive(rabbitListener.exclusive());
		String priority = resolve(rabbitListener.priority());
		if (StringUtils.hasText(priority)) {
			try {
				endpoint.setPriority(Integer.valueOf(priority));
			}
			catch (NumberFormatException ex) {
				throw new BeanInitializationException("Invalid priority value for " +
						rabbitListener + " (must be an integer)", ex);
			}
		}

		String rabbitAdmin = resolve(rabbitListener.admin());
		if (StringUtils.hasText(rabbitAdmin)) {
			Assert.state(this.beanFactory != null, "BeanFactory must be set to resolve RabbitAdmin by bean name");
			try {
				endpoint.setAdmin(this.beanFactory.getBean(rabbitAdmin, RabbitAdmin.class));
			}
			catch (NoSuchBeanDefinitionException ex) {
				throw new BeanInitializationException("Could not register rabbit listener endpoint on [" +
						adminTarget + "], no " + RabbitAdmin.class.getSimpleName() + " with id '" +
						rabbitAdmin + "' was found in the application context", ex);
			}
		}


		RabbitListenerContainerFactory<?> factory = null;
		String containerFactoryBeanName = resolve(rabbitListener.containerFactory());
		if (StringUtils.hasText(containerFactoryBeanName)) {
			Assert.state(this.beanFactory != null, "BeanFactory must be set to obtain container factory by bean name");
			try {
				factory = this.beanFactory.getBean(containerFactoryBeanName, RabbitListenerContainerFactory.class);
			}
			catch (NoSuchBeanDefinitionException ex) {
				throw new BeanInitializationException("Could not register rabbit listener endpoint on [" +
						adminTarget + "] for bean " + beanName + ", no " + RabbitListenerContainerFactory.class.getSimpleName() + " with id '" +
						containerFactoryBeanName + "' was found in the application context", ex);
			}
		}

		this.registrar.registerEndpoint(endpoint, factory);
	}
```
先设置`endpoint`的相关属性，再获取`rabbitListenerContainerFactory`,根据endpoint和rabbitListenerContainerFactory创建一个`messageListenerContainer`，并创建endpoint的id到messageListenerContainer的映射。
当收到消息时，SpringBoot会调用绑定好的方法。



