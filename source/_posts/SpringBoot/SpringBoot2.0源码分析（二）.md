---
title: SpringBoot2.0源码分析（二）：整合ActiveMQ分析
categories: SpringBoot
tags:
  - SpringBoot
  - 源码分析
comments: true
abbrlink: 20003
date: 2018-04-06 15:51:30
---
### ActiveMQ自动注入
当项目中存在`javax.jms.Message`和`org.springframework.jms.core.JmsTemplate`着两个类时，SpringBoot将ActiveMQ需要使用到的对象注册为Bean，供项目注入使用。一起看一下`JmsAutoConfiguration`类。
```
@Configuration
@ConditionalOnClass({ Message.class, JmsTemplate.class })
@ConditionalOnBean(ConnectionFactory.class)
@EnableConfigurationProperties(JmsProperties.class)
@Import(JmsAnnotationDrivenConfiguration.class)
public class JmsAutoConfiguration {

	@Configuration
	protected static class JmsTemplateConfiguration {
	    ......
		@Bean
		@ConditionalOnMissingBean
		@ConditionalOnSingleCandidate(ConnectionFactory.class)
		public JmsTemplate jmsTemplate(ConnectionFactory connectionFactory) {
			PropertyMapper map = PropertyMapper.get();
			JmsTemplate template = new JmsTemplate(connectionFactory);
			template.setPubSubDomain(this.properties.isPubSubDomain());
			map.from(this.destinationResolver::getIfUnique).whenNonNull()
					.to(template::setDestinationResolver);
			map.from(this.messageConverter::getIfUnique).whenNonNull()
					.to(template::setMessageConverter);
			mapTemplateProperties(this.properties.getTemplate(), template);
			return template;
		}
            ......
	}

	@Configuration
	@ConditionalOnClass(JmsMessagingTemplate.class)
	@Import(JmsTemplateConfiguration.class)
	protected static class MessagingTemplateConfiguration {

		@Bean
		@ConditionalOnMissingBean
		@ConditionalOnSingleCandidate(JmsTemplate.class)
		public JmsMessagingTemplate jmsMessagingTemplate(JmsTemplate jmsTemplate) {
			return new JmsMessagingTemplate(jmsTemplate);
		}

	}

}
```
`RabbitAutoConfiguration`主要注入了`jmsMessagingTemplate`和`jmsTemplate`。
`RabbitAutoConfiguration`同时导入了`RabbitAnnotationDrivenConfiguration`，注入了`jmsListenerContainerFactory`。
### 消息发送
以下面的发送为例：
```
    jmsMessagingTemplate.convertAndSend(this.queue, msg);
```
这个方法会先对消息进行转换，预处理，最终通过调用`JmsTemplate`的doSend实现消息发送的。
```
	protected void doSend(Session session, Destination destination, MessageCreator messageCreator)
			throws JMSException {
		Assert.notNull(messageCreator, "MessageCreator must not be null");
		MessageProducer producer = createProducer(session, destination);
		try {
			Message message = messageCreator.createMessage(session);
			doSend(producer, message);
			if (session.getTransacted() && isSessionLocallyTransacted(session)) {
				JmsUtils.commitIfNecessary(session);
			}
		}
		finally {
			JmsUtils.closeMessageProducer(producer);
		}
	}
```
首先创建一个MessageProducer的实例，接着将最初的`org.springframework.messaging.Message`转换成`javax.jms.Message`，再将消息委托给producer进行发送。
### 消息接收
先看一个消费的事例：
```
@Component
public class Consumer {
	@JmsListener(destination = "sample.queue")
	public void receiveQueue(String text) {
		System.out.println(text);
	}
}
```
SpringBoot会去解析`@JmsListener`，具体实现在`JmsListenerAnnotationBeanPostProcessor`的`postProcessAfterInitialization`方法。
```
	public Object postProcessAfterInitialization(final Object bean, String beanName) throws BeansException {
		if (!this.nonAnnotatedClasses.contains(bean.getClass())) {
			Class<?> targetClass = AopProxyUtils.ultimateTargetClass(bean);
			Map<Method, Set<JmsListener>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
					(MethodIntrospector.MetadataLookup<Set<JmsListener>>) method -> {
						Set<JmsListener> listenerMethods = AnnotatedElementUtils.getMergedRepeatableAnnotations(
								method, JmsListener.class, JmsListeners.class);
						return (!listenerMethods.isEmpty() ? listenerMethods : null);
					});
			if (annotatedMethods.isEmpty()) {
				this.nonAnnotatedClasses.add(bean.getClass());
			}
			else {
				annotatedMethods.forEach((method, listeners) ->
						listeners.forEach(listener ->
								processJmsListener(listener, method, bean)));
			}
		}
		return bean;
	}
```
SpringBoot根据注解找到了使用了`@JmsListener`注解的方法，当监听到ActiveMQ收到的消息时，会调用对应的方法。来看一下具体怎么进行listener和method的绑定的。
```
	protected void processJmsListener(JmsListener jmsListener, Method mostSpecificMethod, Object bean) {
		Method invocableMethod = AopUtils.selectInvocableMethod(mostSpecificMethod, bean.getClass());
		MethodJmsListenerEndpoint endpoint = createMethodJmsListenerEndpoint();
		endpoint.setBean(bean);
		endpoint.setMethod(invocableMethod);
		endpoint.setMostSpecificMethod(mostSpecificMethod);
		endpoint.setMessageHandlerMethodFactory(this.messageHandlerMethodFactory);
		endpoint.setEmbeddedValueResolver(this.embeddedValueResolver);
		endpoint.setBeanFactory(this.beanFactory);
		endpoint.setId(getEndpointId(jmsListener));
		endpoint.setDestination(resolve(jmsListener.destination()));
		if (StringUtils.hasText(jmsListener.selector())) {
			endpoint.setSelector(resolve(jmsListener.selector()));
		}
		if (StringUtils.hasText(jmsListener.subscription())) {
			endpoint.setSubscription(resolve(jmsListener.subscription()));
		}
		if (StringUtils.hasText(jmsListener.concurrency())) {
			endpoint.setConcurrency(resolve(jmsListener.concurrency()));
		}

		JmsListenerContainerFactory<?> factory = null;
		String containerFactoryBeanName = resolve(jmsListener.containerFactory());
		if (StringUtils.hasText(containerFactoryBeanName)) {
			Assert.state(this.beanFactory != null, "BeanFactory must be set to obtain container factory by bean name");
			try {
				factory = this.beanFactory.getBean(containerFactoryBeanName, JmsListenerContainerFactory.class);
			}
			catch (NoSuchBeanDefinitionException ex) {
				throw new BeanInitializationException("Could not register JMS listener endpoint on [" +
						mostSpecificMethod + "], no " + JmsListenerContainerFactory.class.getSimpleName() +
						" with id '" + containerFactoryBeanName + "' was found in the application context", ex);
			}
		}

		this.registrar.registerEndpoint(endpoint, factory);
	}
```
先设置`endpoint`的相关属性，再获取`jmsListenerContainerFactory`,最后将`endpoint`注册到`jmsListenerContainerFactory`。



