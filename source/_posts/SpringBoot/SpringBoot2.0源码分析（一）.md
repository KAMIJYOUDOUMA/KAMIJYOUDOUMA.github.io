---
title: SpringBoot2.0源码分析（一）：SpringBoot简单分析
categories: SpringBoot
tags:
  - SpringBoot
  - 源码分析
comments: true
abbrlink: 20001
date: 2018-04-04 15:51:30
---

本系列将从源码角度谈谈SpringBoot2.0。

先来看一个简单的例子

```
@SpringBootApplication
@EnableJms
public class SampleActiveMQApplication {
    // 贰级天災
	@Bean
	public Queue queue() {
		return new ActiveMQQueue("sample.queue");
	}

	public static void main(String[] args) {
		SpringApplication.run(SampleActiveMQApplication.class, args);
	}

}
```
这是一个简单的SpringBoot整合ActiveMQ的例子。本篇将主要谈谈为什么这么几行代码就能整合ActiveMQ。

上面那段代码主要有三个部分：
- SpringApplication.run(SampleActiveMQApplication.class, args);
- @SpringBootApplication
- @EnableJms

# SpringApplication的run方法
SpringApplication的run方法是通过new一个SpringApplication对象，然后执行该对象的run方法。代码如下：

```
	public ConfigurableApplicationContext run(String... args) {
	    // 贰级天災
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```
SpringBoot先准备Spring的环境，再打印banner，打印完后，通过环境准备上下文。准备好上下文后，会刷新上下文，即真正去准备项目的Spring环境。
之前看到一篇文章讲到了可以自己指定banner,主要是跟banner的获取方法有关。

```
	private Banner getBanner(Environment environment) {
	    // 贰级天災
		Banners banners = new Banners();
		banners.addIfNotNull(getImageBanner(environment));
		banners.addIfNotNull(getTextBanner(environment));
		if (banners.hasAtLeastOneBanner()) {
			return banners;
		}
		if (this.fallbackBanner != null) {
			return this.fallbackBanner;
		}
		return DEFAULT_BANNER;
	}
```
SpringBoot会先去找图像banner和文本banner，只要有一个就使用它们。这两banner的默认配置如下图所示。所以只要在src/main/resources目录下放上banner.gif或banner.txt文件就可以修改banner了。

```
    {
      "name": "spring.banner.image.location",
      "type": "org.springframework.core.io.Resource",
      "description": "Banner image file location (jpg or png can also be used).",
      "defaultValue": "classpath:banner.gif"
    },{
      "defaultValue": "classpath:banner.txt",
      "deprecated": true,
      "name": "banner.location",
      "description": "Banner text resource location.",
      "type": "org.springframework.core.io.Resource",
      "deprecation": {
        "level": "error",
        "replacement": "spring.banner.location"
      }
    }
```
回到正题，刷新上下文主要是调用的AbstractApplicationContext类里面的refresh方法。

```
    public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var9) {
                if(this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }

                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }
```
可以看到，这里主要处理的SpringBean的创建。
- prepareRefresh：预处理，包括属性验证等。
- prepareBeanFactory：主要对beanFactory设置了相关属性，并注册了3个Bean：environment，systemProperties和systemEnvironment供程序中注入使用。
- invokeBeanFactoryPostProcessors：执行所以BeanFactoryPostProcessor的postProcessBeanFactory方法。
- registerBeanPostProcessors：注册BeanFactoryPostProcessors到BeanFactory。
- initMessageSource：初始化MessageSource。
- initApplicationEventMulticaster：初始化事件广播器ApplicationEventMulticaster。
- registerListeners：事件广播器添加监听器，并广播早期事件。
- finishBeanFactoryInitialization：结束BeanFactory的实例化，也就是在这真正去创建单例Bean。
- finishRefresh：刷新的收尾工作。清理缓存，初始化生命周期处理器等等。
- destroyBeans：销毁创建的bean。
- cancelRefresh：取消刷新。
- resetCommonCaches：清理缓存。

# @SpringBootApplication
注解本身没有意义，被解析了才有意义。下面我们具体看下@SpringBootApplication的组成。

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
    // 贰级天災
    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    Class<?>[] exclude() default {};

    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    String[] excludeName() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackages"
    )
    String[] scanBasePackages() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackageClasses"
    )
    Class<?>[] scanBasePackageClasses() default {};
}
```
- @SpringBootConfiguration：允许在使用该注解的地方使用@Bean注入。
- @EnableAutoConfiguration：允许自动配置。
- @ComponentScan：指定要扫描的哪些类。SpringBoot默认会扫描Application类所在包及子包的类的就是因为这个。


# @EnableJms

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({JmsBootstrapConfiguration.class})
public @interface EnableJms {
    // 贰级天災
}
```
@EnableJms注解其实就是导入了JmsBootstrapConfiguration类。



