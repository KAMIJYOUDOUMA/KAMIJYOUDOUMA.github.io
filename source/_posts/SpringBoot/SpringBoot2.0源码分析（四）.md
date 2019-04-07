---
title: SpringBoot2.0源码分析（四）：spring-data-jpa分析
categories: SpringBoot
tags:
  - SpringBoot
  - 源码分析
comments: true
abbrlink: 20007
date: 2018-04-18 15:51:30
---
SpringBoot具体整合rabbitMQ可参考：[SpringBoot2.0应用（四）：SpringBoot2.0之spring-data-jpa](https://juejin.im/post/5bd86ec4e51d45763a7b20b8)

# JpaRepositories自动注入
当项目中存在`org.springframework.data.jpa.repository.JpaRepository`类，并且已经注入过数据源`javax.sql.DataSource`，同时没有注入过`org.springframework.data.jpa.repository.config.JpaRepositoryConfigExtension`和`org.springframework.data.jpa.repository.support.JpaRepositoryFactoryBean`时，会通过`@Import`注解导入`org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfigureRegistrar`，由它完成对JPA的支持。`JpaRepositoriesAutoConfigureRegistrar`又继承自`AbstractRepositoryConfigurationSourceSupport`。来看下`AbstractRepositoryConfigurationSourceSupport`的具体内容。
```
public abstract class AbstractRepositoryConfigurationSourceSupport
		implements BeanFactoryAware, ImportBeanDefinitionRegistrar, ResourceLoaderAware,
		EnvironmentAware {

	private ResourceLoader resourceLoader;

	private BeanFactory beanFactory;

	private Environment environment;

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
			BeanDefinitionRegistry registry) {
		new RepositoryConfigurationDelegate(getConfigurationSource(registry),
				this.resourceLoader, this.environment).registerRepositoriesIn(registry,
						getRepositoryConfigurationExtension());
	}

    ......
}
```
可以看出，到`AbstractRepositoryConfigurationSourceSupport`对`Repository`的Bean进行了定义。下面来具体看看Repositoryd的创建。
# Repository的创建
我们先来看下`RepositoryConfigurationDelegate`的`registerRepositoriesIn`方法。
```
	public List<BeanComponentDefinition> registerRepositoriesIn(BeanDefinitionRegistry registry,
			RepositoryConfigurationExtension extension) {

		extension.registerBeansForRoot(registry, configurationSource);

		RepositoryBeanDefinitionBuilder builder = new RepositoryBeanDefinitionBuilder(registry, extension, resourceLoader,
				environment);
		List<BeanComponentDefinition> definitions = new ArrayList<>();

		for (RepositoryConfiguration<? extends RepositoryConfigurationSource> configuration : extension
				.getRepositoryConfigurations(configurationSource, resourceLoader, inMultiStoreMode)) {

			BeanDefinitionBuilder definitionBuilder = builder.build(configuration);

			extension.postProcess(definitionBuilder, configurationSource);

			if (isXml) {
				extension.postProcess(definitionBuilder, (XmlRepositoryConfigurationSource) configurationSource);
			} else {
				extension.postProcess(definitionBuilder, (AnnotationRepositoryConfigurationSource) configurationSource);
			}

			AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
			String beanName = configurationSource.generateBeanName(beanDefinition);
			......
			beanDefinition.setAttribute(FACTORY_BEAN_OBJECT_TYPE, configuration.getRepositoryInterface());

			registry.registerBeanDefinition(beanName, beanDefinition);
			definitions.add(new BeanComponentDefinition(beanDefinition, beanName));
		}

		return definitions;
	}
```
到这里其实只是创建了repository的实体Bean的BeanDefinition。前期准备做好了，实际创建repository是在`RepositoryFactorySupport`的getRepository方法。
```
	public <T> T getRepository(Class<T> repositoryInterface, RepositoryFragments fragments) {
		Assert.notNull(repositoryInterface, "Repository interface must not be null!");
		Assert.notNull(fragments, "RepositoryFragments must not be null!");
     
		RepositoryMetadata metadata = getRepositoryMetadata(repositoryInterface);
		RepositoryComposition composition = getRepositoryComposition(metadata, fragments);
		RepositoryInformation information = getRepositoryInformation(metadata, composition);

		validate(information, composition);

		Object target = getTargetRepository(information);

		// Create proxy
		ProxyFactory result = new ProxyFactory();
		result.setTarget(target);
		result.setInterfaces(repositoryInterface, Repository.class, TransactionalProxy.class);

		if (MethodInvocationValidator.supports(repositoryInterface)) {
			result.addAdvice(new MethodInvocationValidator());
		}

		result.addAdvice(SurroundingTransactionDetectorMethodInterceptor.INSTANCE);
		result.addAdvisor(ExposeInvocationInterceptor.ADVISOR);

		postProcessors.forEach(processor -> processor.postProcess(result, information));

		result.addAdvice(new DefaultMethodInvokingMethodInterceptor());

		ProjectionFactory projectionFactory = getProjectionFactory(classLoader, beanFactory);
		result.addAdvice(new QueryExecutorMethodInterceptor(information, projectionFactory));

		composition = composition.append(RepositoryFragment.implemented(target));
		result.addAdvice(new ImplementationMethodExecutionInterceptor(composition));

		return (T) result.getProxy(classLoader);
	}
```
首先去获取我们写的repository接口的元数据，包括实体的ID类型，管理的实体类型等。接着获取repository的组合，主要包含repository的方法信息。然后再根据它俩的组合得到一个target。这个target其实就是一个SimpleJpaRepository实体，里面包含了一些通用的方法。只有这些还不够，于是有了后面的代理工厂，对这个target进行进一步处理。包括事务支持，异常处理和SQL创造等。我们主要看一下SQL创建。创建的方法在`DeclaredQueryLookupStrategy`的`resolveQuery`中。
```
protected RepositoryQuery resolveQuery(JpaQueryMethod method, EntityManager em, NamedQueries namedQueries) {

			RepositoryQuery query = JpaQueryFactory.INSTANCE.fromQueryAnnotation(method, em, evaluationContextProvider);

			if (null != query) {
				return query;
			}

			query = JpaQueryFactory.INSTANCE.fromProcedureAnnotation(method, em);

			if (null != query) {
				return query;
			}

			String name = method.getNamedQueryName();
			if (namedQueries.hasQuery(name)) {
				return JpaQueryFactory.INSTANCE.fromMethodWithQueryString(method, em, namedQueries.getQuery(name),
						evaluationContextProvider);
			}

			query = NamedQuery.lookupFrom(method, em);

			if (null != query) {
				return query;
			}

			throw new IllegalStateException(
					String.format("Did neither find a NamedQuery nor an annotated query for method %s!", method));
		}
```
该方法的逻辑是先找有注解的，这个包括`@Query`和`@Procedure`，接着是根据关键字创建，然后是通用方法。



