# applicationContext之GenericApplicationContext

org.springframework.context.support.GenericApplicationContext
通用的ApplicationContext实现，它包含一个单一的内部DefaultListableBeanFactory实例，实现BeanDefinitionRegistry接口，典型的用法是通过BeanDefinitionRegistry接口注册各种bean定义，然后调用refresh()来用应用程序上下文来初始化这些bean，refresh()只能调用一次。

```java
private final DefaultListableBeanFactory beanFactory;
```
内部BeanFactory。

```java
private final AtomicBoolean refreshed = new AtomicBoolean();
```
上下文刷新开关。

```java
public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}
```
创建applicationContext，默认创建BeanFactory。

```java
public void setAllowBeanDefinitionOverriding(boolean allowBeanDefinitionOverriding) {
		this.beanFactory.setAllowBeanDefinitionOverriding(allowBeanDefinitionOverriding);
	}
```
设置BeanFactory属性allowBeanDefinitionOverriding，是否允许BeanDefinition覆盖。

```java
public void setAllowCircularReferences(boolean allowCircularReferences) {
		this.beanFactory.setAllowCircularReferences(allowCircularReferences);
	}
```
设置BeanFactory属性allowCircularReferences，是否允许bean之间循环依赖。

```java
@Override
	protected final void refreshBeanFactory() throws IllegalStateException {
//		可以是用atomic实现的一个线程安全，也可以使volital和锁来实现
		if (!this.refreshed.compareAndSet(false, true)) {
			throw new IllegalStateException(
					"GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
		}
		this.beanFactory.setSerializationId(getId());
	}
```

```java
@Override
	protected final void refreshBeanFactory() throws IllegalStateException {
//		可以是用atomic实现的一个线程安全，也可以使volatile和锁来实现
		if (!this.refreshed.compareAndSet(false, true)) {
			throw new IllegalStateException(
					"GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
		}
		this.beanFactory.setSerializationId(getId());
	}
```
刷新BeanFactory，如果没刷新过，设置BeanFactory的序列化id。

```java
@Override
	protected void cancelRefresh(BeansException ex) {
		this.beanFactory.setSerializationId(null);
		super.cancelRefresh(ex);
	}
```
设置BeanFactory的序列化id为null，取消刷新操作，设置上下文是否激活active属性为false。

```java
@Override
	protected final void closeBeanFactory() {
		this.beanFactory.setSerializationId(null);
	}
```
关闭BeanFactory，设置BeanFactory的序列化id为null。

```java
@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		this.beanFactory.registerBeanDefinition(beanName, beanDefinition);
	}
```
注册BeanDefinition。

```java
@Override
	public void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
		this.beanFactory.removeBeanDefinition(beanName);
	}
```
删除BeanDefinition。

```java
	@Override
	public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
		return this.beanFactory.getBeanDefinition(beanName);
	}
```
查询BeanDefinition。

```java
@Override
	public void registerAlias(String beanName, String alias) {
//		注册别名
		this.beanFactory.registerAlias(beanName, alias);
	}
```
注册别名。

```java
@Override
	public void removeAlias(String alias) {
		this.beanFactory.removeAlias(alias);
	}
```
删除别名。

```java
public final <T> void registerBean(Class<T> beanClass, BeanDefinitionCustomizer... customizers) {
		registerBean(null, beanClass, null, customizers);
	}
```
注册bean。

```java
public <T> void registerBean(@Nullable String beanName, Class<T> beanClass, @Nullable Supplier<T> supplier,
			BeanDefinitionCustomizer... customizers) {

		BeanDefinitionBuilder builder = (supplier != null ?
				BeanDefinitionBuilder.genericBeanDefinition(beanClass, supplier) :
				BeanDefinitionBuilder.genericBeanDefinition(beanClass));
		BeanDefinition beanDefinition = builder.applyCustomizers(customizers).getRawBeanDefinition();

		String nameToUse = (beanName != null ? beanName : beanClass.getName());
		registerBeanDefinition(nameToUse, beanDefinition);
	}
```
执行org.springframework.beans.factory.config.BeanDefinitionCustomizer#customize后进行BeanDefinition注册。

org.springframework.context.support.StaticApplicationContext
ApplicationContext实现，它支持bean编程注册，而不是从外部配置源读取bean定义。主要用于测试。

```java
public void registerSingleton(String name, Class<?> clazz) throws BeansException {
		GenericBeanDefinition bd = new GenericBeanDefinition();
		bd.setBeanClass(clazz);
		getDefaultListableBeanFactory().registerBeanDefinition(name, bd);
	}
```
创建GenericBeanDefinition，在BeanDefinition设置指定的clazz，注册BeanDefinition。

```java
public void registerSingleton(String name, Class<?> clazz, MutablePropertyValues pvs) throws BeansException {
		GenericBeanDefinition bd = new GenericBeanDefinition();
		bd.setBeanClass(clazz);
		bd.setPropertyValues(pvs);
		getDefaultListableBeanFactory().registerBeanDefinition(name, bd);
	}
```
创建GenericBeanDefinition，在BeanDefinition设置指定的clazz，设置属性值，注册BeanDefinition。

```java
public void registerPrototype(String name, Class<?> clazz) throws BeansException {
		GenericBeanDefinition bd = new GenericBeanDefinition();
		bd.setScope(GenericBeanDefinition.SCOPE_PROTOTYPE);
		bd.setBeanClass(clazz);
		getDefaultListableBeanFactory().registerBeanDefinition(name, bd);
	}
```
创建GenericBeanDefinition，在BeanDefinition设置指定的clazz，设置scope属性为prototype，注册BeanDefinition。

```java
public void registerPrototype(String name, Class<?> clazz, MutablePropertyValues pvs) throws BeansException {
		GenericBeanDefinition bd = new GenericBeanDefinition();
		bd.setScope(GenericBeanDefinition.SCOPE_PROTOTYPE);
		bd.setBeanClass(clazz);
		bd.setPropertyValues(pvs);
		getDefaultListableBeanFactory().registerBeanDefinition(name, bd);
	}
```
创建GenericBeanDefinition，在BeanDefinition设置指定的clazz，设置scope属性为prototype，设置属性值，注册BeanDefinition。

org.springframework.context.support.GenericXmlApplicationContext
使用内置的XML支持方便的应用程序上下文。这是一种灵活的方法，可以替代ClassPathXmlApplicationContext和FileSystemXmlApplicationContext，通过设置器来配置，最终refresh()调用激活上下文，在多个配置文件的情况下，稍后文件中的bean定义将覆盖前面文件中定义的那些。通过附加到列表的额外配置文件，可以利用这种方法故意覆盖某些bean定义。

```java
private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
```
XmlBeanDefinitionReader实例。

```java
public GenericXmlApplicationContext(Resource... resources) {
		load(resources);
		refresh();
	}
```
加载指定的上下文配置文件资源加载BeanDefinition，refresh()上下文刷新。

```java
public GenericXmlApplicationContext(String... resourceLocations) {
		load(resourceLocations);
		refresh();
	}
```
从指定配置文件路径加载BeanDefinition，refresh()上下文刷新。

```java
public GenericXmlApplicationContext(Class<?> relativeClass, String... resourceNames) {
		load(relativeClass, resourceNames);
		refresh();
	}
```
从指定的类路径、指定的配置文件名加载BeanDefinition，refresh()上下文刷新。

```java
public void load(Resource... resources) {
		this.reader.loadBeanDefinitions(resources);
	}
```
从指定的资源加载BeanDefinition。

```java
public void load(String... resourceLocations) {
		this.reader.loadBeanDefinitions(resourceLocations);
	}
```
从指定的配置文件加载BeanDefinition。

```java
public void load(Class<?> relativeClass, String... resourceNames) {
		Resource[] resources = new Resource[resourceNames.length];
		for (int i = 0; i < resourceNames.length; i++) {
			resources[i] = new ClassPathResource(resourceNames[i], relativeClass);
		}
		this.load(resources);
	}
```
从指定的class路径、配置文件名加载BeanDefinition。

org.springframework.context.annotation.AnnotationConfigRegistry
用于注释配置应用程序上下文、定义寄存器和扫描方法的通用接口。

```java
void register(Class<?>... annotatedClasses);
```
注册一个或多个@Configuration配置类。

```java
void scan(String... basePackages);
```
在指定的包进行BeanDefinition扫描。

org.springframework.context.annotation.AnnotationConfigApplicationContext
独立的应用程序上下文，扫描@Configuration、@Component注解，配置类方法上的@Bean注解。

```java
private final AnnotatedBeanDefinitionReader reader;
```
BeanDefinitionReader。

```java
private final ClassPathBeanDefinitionScanner scanner;
```
classpath BeanDefinitionScanner。

```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
		refresh();
	}
```
注册配置类并进行上下文刷新。

```java
public AnnotationConfigApplicationContext(String... basePackages) {
		this();
		scan(basePackages);
		refresh();
	}
```
扫描指定包的BeanDefinition并刷新上下文。

```java
public void register(Class<?>... annotatedClasses) {
		Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
		this.reader.register(annotatedClasses);
	}
```
注册配置类。

```java
public void scan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		this.scanner.scan(basePackages);
	}
```
扫描指定包的BeanDefinition。

```java
public <T> void registerBean(Class<T> annotatedClass, Object... constructorArguments) {
		registerBean(null, annotatedClass, constructorArguments);
	}
```
注册bean。
org.springframework.context.annotation.AnnotationConfigApplicationContext#registerBean(java.lang.String, java.lang.Class<T>, java.lang.Object...)

```java
public <T> void registerBean(@Nullable String beanName, Class<T> annotatedClass, Object... constructorArguments) {
		this.reader.doRegisterBean(annotatedClass, null, beanName, null,
				new BeanDefinitionCustomizer() {
					@Override
					public void customize(BeanDefinition bd) {
						for (Object arg : constructorArguments) {
							bd.getConstructorArgumentValues().addGenericArgumentValue(arg);
						}
					}
				});
	}
```

```java
@Override
	public <T> void registerBean(@Nullable String beanName, Class<T> beanClass, @Nullable Supplier<T> supplier,
			BeanDefinitionCustomizer... customizers) {

		this.reader.doRegisterBean(beanClass, supplier, beanName, null, customizers);
	}
```
注册bean。

org.springframework.context.ApplicationContextAware
当一个对象需要访问一组协作bean时，实现这个接口是有意义的。

```java
void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
```
applicationContext生命周期方法，可以获得applicationContext处理自己的逻辑。

org.springframework.context.support.ApplicationObjectSupport
用于应用程序对象的方便的超类。

```java
@Nullable
	private ApplicationContext applicationContext;
```
上下文对象。

```java
	@Override
	public final void setApplicationContext(@Nullable ApplicationContext context) throws BeansException {
		if (context == null && !isContextRequired()) {
			// Reset internal context state.
			this.applicationContext = null;
			this.messageSourceAccessor = null;
		}
		else if (this.applicationContext == null) {
			// Initialize with passed-in context.
			if (!requiredContextClass().isInstance(context)) {
				throw new ApplicationContextException(
						"Invalid application context: needs to be of type [" + requiredContextClass().getName() + "]");
			}
			this.applicationContext = context;
			this.messageSourceAccessor = new MessageSourceAccessor(context);
			initApplicationContext(context);
		}
		else {
			// Ignore reinitialization if same context passed in.
			if (this.applicationContext != context) {
				throw new ApplicationContextException(
						"Cannot reinitialize with different application context: current one is [" +
						this.applicationContext + "], passed-in one is [" + context + "]");
			}
		}
	}
```
重写了org.springframework.context.ApplicationContextAware#setApplicationContext方法。

```java
protected void initApplicationContext(ApplicationContext context) throws BeansException {
		initApplicationContext();
	}
```

```java
protected void initApplicationContext() throws BeansException {
	}

```
子类可以覆盖这个方法自定义上下文初始化行为。

org.springframework.context.support.ApplicationContextAwareProcessor
BeanPostProcessor实现将应用程序上下文传递给实现环境感知的bean。

```java
private final ConfigurableApplicationContext applicationContext;
```
applicationContext。

```java
	@Override
	@Nullable
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}
```
重写了org.springframework.beans.factory.config.BeanPostProcessor#postProcessBeforeInitialization方法。

```java
private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
```
执行aware，如org.springframework.context.ApplicationEventPublisherAware#setApplicationEventPublisher、org.springframework.context.ApplicationContextAware#setApplicationContext。

```java
@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		return bean;
	}
```
重写了org.springframework.beans.factory.config.BeanPostProcessor#postProcessAfterInitialization方法，bean初始化之后的处理逻辑，默认不做处理。
