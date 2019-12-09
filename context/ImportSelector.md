# ImportSelector

org.springframework.context.annotation.ImportSelector
一般与@Import一起使用确定引入哪个@Configuration类。

```java
String[] selectImports(AnnotationMetadata importingClassMetadata);
```
根据引入的@Configuration注解元数据选择导入的类的名称。


org.springframework.context.annotation.Import
可以引入一个或多个@Configuration类，类似xml的<import/>，在导入的@Configuration类中声明的@Bean定义应该使用@Autowired注入来访问。

```java
Class<?>[] value();
```
引入的@Configuration类的class。

Import、ImportSelector、ImportBeanDefinitionRegistrar 组合使用。

org.springframework.context.annotation.ImportBeanDefinitionRegistrar
通过在处理@Configuration类时注册额外bean定义的类型来实现的接口。

```java
public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry);
```
根据导入的@Configuration类的给定注释元数据来注册bean定义。

org.springframework.context.annotation.ImportResource
和@Import一样，这个注释提供了类似于Spring XML中<import/>元素的功能，将使用XmlBeanDefinitionReader来解析Spring <bean /> XML文件。

```java
@AliasFor("locations")
	String[] value() default {};
```
指定BeanDefinition的位置。

```java
@AliasFor("value")
	String[] locations() default {};
```
指定BeanDefinition的位置。


```java
Class<? extends BeanDefinitionReader> reader() default BeanDefinitionReader.class;
```
xml资源都将使用XmlBeanDefinitionReader来处理。

org.springframework.context.annotation.AdviceModeImportSelector
基于注释的AdviceMode值来选择导入。

org.springframework.context.annotation.AdviceModeImportSelector#selectImports(org.springframework.core.type.AnnotationMetadata)

```java
@Override
	public final String[] selectImports(AnnotationMetadata importingClassMetadata) {
		Class<?> annType = GenericTypeResolver.resolveTypeArgument(getClass(), AdviceModeImportSelector.class);
		Assert.state(annType != null, "Unresolvable type argument for AdviceModeImportSelector");

//		从引入类注解元数据中获取注解属性值
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(importingClassMetadata, annType);
		if (attributes == null) {
			throw new IllegalArgumentException(String.format(
				"@%s is not present on importing class '%s' as expected",
				annType.getSimpleName(), importingClassMetadata.getClassName()));
		}

		AdviceMode adviceMode = attributes.getEnum(this.getAdviceModeAttributeName());
//		根据adviceMode选择引入的类
		String[] imports = selectImports(adviceMode);
		if (imports == null) {
			throw new IllegalArgumentException(String.format("Unknown AdviceMode: '%s'", adviceMode));
		}
		return imports;
	}
```
根据引入的类的注解元数据解析引入的类。

```java
@Nullable
	protected abstract String[] selectImports(AdviceMode adviceMode);
```
根据adviceMode获取引入的类，子类需要覆盖这个模板方法，这里是模板方法设计模式实现。

org.springframework.context.annotation.DeferredImportSelector
在处理完所有@Configuration bean后运行的ImportSelector，当选择的导入是有条件的时，这种类型的选择器会特别有用，这个类支持@Order标识优先级。

org.springframework.context.annotation.EnableAspectJAutoProxy
@EnableAspectJAutoProxy仅适用于其本地应用程序上下文，允许在不同级别上选择性地代理bean。

```java
boolean proxyTargetClass() default false;
```
指示是否要创建基于子类的(CGLIB)代理，而不是标准的基于Java接口的代理。默认false。

org.springframework.scheduling.annotation.EnableAsync
和<task:executor>是等价。

```java
Class<? extends Annotation> annotation() default Annotation.class;
```
在类和方法上监测@Async注解。被这个注解标识的方法会被异步执行。

```java
boolean proxyTargetClass() default false;
```
指示是否要创建基于子类的(CGLIB)代理，而不是标准的基于Java接口的代理。

org.springframework.scheduling.annotation.AsyncConfigurationSelector

```java
@Override
	@Nullable
	public String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] { ProxyAsyncConfiguration.class.getName() };
			case ASPECTJ:
				return new String[] { ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME };
			default:
				return null;
		}
	}
```
这个方法是子类重写父类的方法，如果是PROXY的方式选择ProxyAsyncConfiguration这个类，如果是ASPECTJ选择AspectJAsyncConfiguration这个类。

org.springframework.context.annotation.AspectJAutoProxyRegistrar
根据给定的@EnableAspectJAutoProxy注释，在当前的BeanDefinitionRegistry中注册一个注释。

```java
@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

//		获取EnableAspectJAutoProxy的属性值
		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
//		确认是采用proxy还是aopContext的代理方式
		if (enableAspectJAutoProxy != null) {
			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}
```
确认是采用proxy还是采用代理增强。

org.springframework.context.annotation.AutoProxyRegistrar
自动创建代理并注册中。

```java
@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		boolean candidateFound = false;
//		获取注解元数据中的属性值
		Set<String> annoTypes = importingClassMetadata.getAnnotationTypes();
		for (String annoType : annoTypes) {
			AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
			if (candidate == null) {
				continue;
			}
			Object mode = candidate.get("mode");
//			获取proxyTargetClass属性值
			Object proxyTargetClass = candidate.get("proxyTargetClass");
			if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
					Boolean.class == proxyTargetClass.getClass()) {
				candidateFound = true;
				if (mode == AdviceMode.PROXY) {
//					注册自动代理的BeanDefinition
					AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
					if ((Boolean) proxyTargetClass) {
//						注册cglib自动杆代理的BeanDefinition
						AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
						return;
					}
				}
			}
		}
		if (!candidateFound) {
			String name = getClass().getSimpleName();
			logger.warn(String.format("%s was imported but no annotations were found " +
					"having both 'mode' and 'proxyTargetClass' attributes of type " +
					"AdviceMode and boolean respectively. This means that auto proxy " +
					"creator registration and configuration may not have occurred as " +
					"intended, and components may not be proxied as expected. Check to " +
					"ensure that %s has been @Import'ed on the same class where these " +
					"annotations are declared; otherwise remove the import of %s " +
					"altogether.", name, name, name));
		}
	}
```
根据代理方式注册不同代理的BeanDefinition，如果设置了proxyTargetClass=true就注册cglib动态代理的BeanDefinition。

org.springframework.scheduling.annotation.AbstractAsyncConfiguration

```java
//	@EnableAsync注解的属性值
	@Nullable
	protected AnnotationAttributes enableAsync;

//	异步执行的线程池
	@Nullable
	protected Executor executor;

//	线程池异常处理handler
	@Nullable
	protected AsyncUncaughtExceptionHandler exceptionHandler;
```

```java
@Override
	public void setImportMetadata(AnnotationMetadata importMetadata) {
		this.enableAsync = AnnotationAttributes.fromMap(
				importMetadata.getAnnotationAttributes(EnableAsync.class.getName(), false));
		if (this.enableAsync == null) {
			throw new IllegalArgumentException(
					"@EnableAsync is not present on importing class " + importMetadata.getClassName());
		}
	}
```
获取@EnableAsync的属性值。

```java
@Autowired(required = false)
	void setConfigurers(Collection<AsyncConfigurer> configurers) {
		if (CollectionUtils.isEmpty(configurers)) {
			return;
		}
		if (configurers.size() > 1) {
			throw new IllegalStateException("Only one AsyncConfigurer may exist");
		}
		AsyncConfigurer configurer = configurers.iterator().next();
//		获取线程池
		this.executor = configurer.getAsyncExecutor();
		this.exceptionHandler = configurer.getAsyncUncaughtExceptionHandler();
	}
```
通过依赖注入从AsyncConfigurer中获取executor、exceptionHandler。

org.springframework.scheduling.annotation.ProxyAsyncConfiguration
加载AsyncConfiguration。

```java
	@Bean(name = TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
		Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
//		创建AsyncAnnotationBeanPostProcessor，BeanPostProcessor 前面spring ioc源码解析部分介绍过是对bean初始化过程的管理
		AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
		Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
		if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
			bpp.setAsyncAnnotationType(customAsyncAnnotation);
		}
//		对线程池执行器进行赋值
		if (this.executor != null) {
			bpp.setExecutor(this.executor);
		}
		if (this.exceptionHandler != null) {
			bpp.setExceptionHandler(this.exceptionHandler);
		}
//		这里默认是jdk的动态代理，可以@EnableAsync这个注解的属性值，也可以指定
		bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
		bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
		return bpp;
	}
```
创建AsyncAnnotationBeanPostProcessor。

org.springframework.scheduling.annotation.AsyncConfigurer
可以实现这个接口定制化自己的异步配置。

```java
@Nullable
	default Executor getAsyncExecutor() {
		return null;
	}

@Nullable
	default AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
		return null;
	}
```
可以自定义线程执行器和异常handler。

org.springframework.scheduling.annotation.AsyncConfigurerSupport org.springframework.scheduling.annotation.AsyncConfigurer的默认实现，用户可以继承这个类。

```java
@Override
	public Executor getAsyncExecutor() {
		return null;
	}

	@Override
	@Nullable
	public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
		return null;
	}
```
可以重写这两个方法自定义自己的线程执行器和异常handler。

org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler
异步方法执行异常处理的handler。

```java
void handleUncaughtException(Throwable ex, Method method, Object... params);
```
对具体执行异常的异步方法进行处理。


