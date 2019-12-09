# applicationContext之RefreshableApplicationContext

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1575814877914-1a75aa23-c1ec-4a86-a843-30d961f4cd1d.png#align=left&display=inline&height=266&name=image.png&originHeight=532&originWidth=776&size=55678&status=done&style=none&width=388)

org.springframework.context.ApplicationContext 
顶层接口，为应用程序提供配置。

```java
@Nullable
	String getId();
```
返回应用程序上下文的唯一id。

```java
String getApplicationName();
```
返回应用应用程序名称。

```java
long getStartupDate();
```
第一次加载上下文的时间。

```java
ApplicationContext getParent();
```
返回上一级上下文。

```java
AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;
```
返回AutowireCapableBeanFactory。

org.springframework.context.Lifecycle
启动、停止声明周期控制方法的接口。

```java
void start();
```
程序启动。

```java
void stop();
```
程序停止。

```java
boolean isRunning();
```
程序是否在运行。

org.springframework.context.ConfigurableApplicationContext
配置上下文，实现了程序声明周期上下文接口。

```java
void setParent(@Nullable ApplicationContext parent);
```
设置上级applicationContext

```java
void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);
```
添加一个新的BeanFactoryPostProcessor，在上下文配置期间调用。

```java
void addApplicationListener(ApplicationListener<?> listener);
```
添加一个新的ApplicationListener，它将在上下文事件(如上下文刷新和上下文关闭)中得到通知。

```java
void refresh() throws BeansException, IllegalStateException;
```
加载或刷新配置，在调用这个方法之后所有的单例bean都应该被实例化。

```java
void registerShutdownHook();
```
注册一个jvm关闭的钩子方法。

```java
	@Override
	void close();
```
关闭程序上下文。

```java
boolean isActive();
```
确定上下文是够是激活的，也就是是否刷新过一次还没关闭。

```java
ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```
返回上下文的BeanFactory。

org.springframework.context.LifecycleProcessor
声明周期处理器。

```java
void onRefresh();
```
上下文刷新通知。

```java
void onClose();
```
上下文关闭通知。

org.springframework.context.SmartLifecycle
对那些需要在应用程序上下文刷新或关闭时启动的对象的生命周期接口org.springframework.context.Lifecycle的扩展。

```java
boolean isAutoStartup();
```
如果这个生命周期组件在包含ApplicationContext被刷新的时候被容器自动启动，则返回true。
	false表示组件是打算通过一个显式的start()调用开始的，类似于一个简单的生命周期实现。

```java
void stop(Runnable callback);
```
指示生命周期组件在当前运行时必须停止。

org.springframework.context.support.DefaultLifecycleProcessor
默认实现生命周期处理器策略。

```java
@Override
	public void start() {
		startBeans(false);
		this.running = true;
	}
```
启动实现生命周期的所有已注册bean并没有运行。修改running程序启动开关为启动，这里这个属性用volatile保证线程之间的可见性。
org.springframework.context.support.DefaultLifecycleProcessor#startBeans

```java
private void startBeans(boolean autoStartupOnly) {
//		获取声明周期内监管的bean
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<>();
		lifecycleBeans.forEach((beanName, bean) -> {
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
//				获取生命周期组
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(beanName, bean);
			}
		});
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				phases.get(key).start();
			}
		}
	}
```
启动bean，调用bean实现的声明周期的启动方法。
org.springframework.context.support.DefaultLifecycleProcessor#getLifecycleBeans

```java
	protected Map<String, Lifecycle> getLifecycleBeans() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		Map<String, Lifecycle> beans = new LinkedHashMap<>();
//		获取实现了Lifecycle这个接口的beanNames
		String[] beanNames = beanFactory.getBeanNamesForType(Lifecycle.class, false, false);
		for (String beanName : beanNames) {
			String beanNameToRegister = BeanFactoryUtils.transformedBeanName(beanName);
//			是否实现了factoryBean接口
			boolean isFactoryBean = beanFactory.isFactoryBean(beanNameToRegister);
			String beanNameToCheck = (isFactoryBean ? BeanFactory.FACTORY_BEAN_PREFIX + beanName : beanName);
			if ((beanFactory.containsSingleton(beanNameToRegister) &&
					(!isFactoryBean || matchesBeanType(Lifecycle.class, beanNameToCheck, beanFactory))) ||
					matchesBeanType(SmartLifecycle.class, beanNameToCheck, beanFactory)) {
//				从beanFactory中获取bean
				Object bean = beanFactory.getBean(beanNameToCheck);
				if (bean != this && bean instanceof Lifecycle) {
					beans.put(beanNameToRegister, (Lifecycle) bean);
				}
			}
		}
		return beans;
	}
```
从BeanFactory中获取实现Lifecycle接口的bean。
org.springframework.context.support.DefaultLifecycleProcessor.LifecycleGroup#start

```java
public void start() {
			if (this.members.isEmpty()) {
				return;
			}
			if (logger.isInfoEnabled()) {
				logger.info("Starting beans in phase " + this.phase);
			}
			Collections.sort(this.members);
			for (LifecycleGroupMember member : this.members) {
				if (this.lifecycleBeans.containsKey(member.name)) {
					doStart(this.lifecycleBeans, member.name, this.autoStartupOnly);
				}
			}
		}
```
org.springframework.context.support.DefaultLifecycleProcessor#doStart

```java
private void doStart(Map<String, ? extends Lifecycle> lifecycleBeans, String beanName, boolean autoStartupOnly) {
		Lifecycle bean = lifecycleBeans.remove(beanName);
		if (bean != null && !this.equals(bean)) {
//			从BeanFactory中获取指定beanName的依赖的beanName集合
			String[] dependenciesForBean = getBeanFactory().getDependenciesForBean(beanName);
			for (String dependency : dependenciesForBean) {
				doStart(lifecycleBeans, dependency, autoStartupOnly);
			}
			if (!bean.isRunning() &&
					(!autoStartupOnly || !(bean instanceof SmartLifecycle) || ((SmartLifecycle) bean).isAutoStartup())) {
				if (logger.isDebugEnabled()) {
					logger.debug("Starting bean '" + beanName + "' of type [" + bean.getClass() + "]");
				}
				try {
//					调用声命周期的启动方法
					bean.start();
				}
				catch (Throwable ex) {
					throw new ApplicationContextException("Failed to start bean '" + beanName + "'", ex);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Successfully started bean '" + beanName + "'");
				}
			}
		}
	}
```
调用bean和依赖bean的实现的生命周期的启动方法。
```java
@Override
	public void stop() {
		stopBeans();
		this.running = false;
	}
```
停止所有实现生命周期的注册bean并正在运行。修改running程序启动开关为停止。
org.springframework.context.support.DefaultLifecycleProcessor#stopBeans

```java
	private void stopBeans() {
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<>();
		lifecycleBeans.forEach((beanName, bean) -> {
			int shutdownOrder = getPhase(bean);
			LifecycleGroup group = phases.get(shutdownOrder);
			if (group == null) {
				group = new LifecycleGroup(shutdownOrder, this.timeoutPerShutdownPhase, lifecycleBeans, false);
				phases.put(shutdownOrder, group);
			}
			group.add(beanName, bean);
		});
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<>(phases.keySet());
			keys.sort(Collections.reverseOrder());
			for (Integer key : keys) {
				phases.get(key).stop();
			}
		}
	}
```
停止bean处理逻辑。
org.springframework.context.support.DefaultLifecycleProcessor.LifecycleGroup#stop

```java
	public void stop() {
			if (this.members.isEmpty()) {
				return;
			}
			if (logger.isInfoEnabled()) {
				logger.info("Stopping beans in phase " + this.phase);
			}
			this.members.sort(Collections.reverseOrder());
			CountDownLatch latch = new CountDownLatch(this.smartMemberCount);
			Set<String> countDownBeanNames = Collections.synchronizedSet(new LinkedHashSet<>());
			for (LifecycleGroupMember member : this.members) {
				if (this.lifecycleBeans.containsKey(member.name)) {
					doStop(this.lifecycleBeans, member.name, latch, countDownBeanNames);
				}
				else if (member.bean instanceof SmartLifecycle) {
					// already removed, must have been a dependent已删除，一定是一个从属
					latch.countDown();
				}
			}
			try {
//				阻塞等待所有实现声命周期接口的方法处理完
				latch.await(this.timeout, TimeUnit.MILLISECONDS);
				if (latch.getCount() > 0 && !countDownBeanNames.isEmpty() && logger.isWarnEnabled()) {
					logger.warn("Failed to shut down " + countDownBeanNames.size() + " bean" +
							(countDownBeanNames.size() > 1 ? "s" : "") + " with phase value " +
							this.phase + " within timeout of " + this.timeout + ": " + countDownBeanNames);
				}
			}
			catch (InterruptedException ex) {
				Thread.currentThread().interrupt();
			}
		}
```
org.springframework.context.support.DefaultLifecycleProcessor#doStop

```java
private void doStop(Map<String, ? extends Lifecycle> lifecycleBeans, final String beanName,
			final CountDownLatch latch, final Set<String> countDownBeanNames) {

		Lifecycle bean = lifecycleBeans.remove(beanName);
		if (bean != null) {
//			从BeanFactory中获取指定bean的依赖的bean名称集合
			String[] dependentBeans = getBeanFactory().getDependentBeans(beanName);
			for (String dependentBean : dependentBeans) {
				doStop(lifecycleBeans, dependentBean, latch, countDownBeanNames);
			}
			try {
				if (bean.isRunning()) {
					if (bean instanceof SmartLifecycle) {
						if (logger.isDebugEnabled()) {
							logger.debug("Asking bean '" + beanName + "' of type [" + bean.getClass() + "] to stop");
						}
						countDownBeanNames.add(beanName);
//						调用生命周期的停止方法
						((SmartLifecycle) bean).stop(() -> {
//							CountDownLatch实现同步控制
							latch.countDown();
							countDownBeanNames.remove(beanName);
							if (logger.isDebugEnabled()) {
								logger.debug("Bean '" + beanName + "' completed its stop procedure");
							}
						});
					}
					else {
						if (logger.isDebugEnabled()) {
							logger.debug("Stopping bean '" + beanName + "' of type [" + bean.getClass() + "]");
						}
//						声明周期停止方法
						bean.stop();
						if (logger.isDebugEnabled()) {
							logger.debug("Successfully stopped bean '" + beanName + "'");
						}
					}
				}
				else if (bean instanceof SmartLifecycle) {
					// don't wait for beans that aren't running不要等待没有运行的bean
					latch.countDown();
				}
			}
			catch (Throwable ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Failed to stop bean '" + beanName + "'", ex);
				}
			}
		}
	}
```
同步等待所有实现生命周期的bean和依赖bean的执行完stop方法。

```java
@Override
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}
```
上下文刷新通知方法，启动bean。修改running程序运行开关为运行中。

```java
@Override
	public void onClose() {
		stopBeans();
		this.running = false;
	}
```
程序上下文关闭通知方法，停止bean，修改running程序运行开关为停止。

org.springframework.context.support.AbstractApplicationContext
应用程序上下文接口的抽象实现。简单地实现公共上下文功能。使用模板方法设计模式，需要具体的子类来实现抽象方法。这个类会自动注册BeanFactoryPostProcessors、BeanPostProcessors和applicationlistener。

```java
@Nullable
	private ApplicationContext parent;
```
上一级上下文。

```java
private final List<BeanFactoryPostProcessor> beanFactoryPostProcessors = new ArrayList<>();
```
应用于程序上下文刷新的BeanFactoryPostProcessor。

```java
private long startupDate;
```
上下文启动的时间。

```java
private final AtomicBoolean active = new AtomicBoolean();
```
上下文是否处于激活状态的原子开关。

```java
private final AtomicBoolean closed = new AtomicBoolean();
```
程序上下文是否关闭的原子开关。

```java
@Nullable
	public LifecycleProcessor lifecycleProcessor;
```
声明周期的处理器。

```java
@Override
	public AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException {
		return getBeanFactory();
	}
```
获取BeanFactory，模板方法实现，子类可以实现这个模板方法org.springframework.context.support.AbstractApplicationContext#getBeanFactory

```java
@Override
	public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```

```java
@Override
	public void publishEvent(ApplicationEvent event) {
		publishEvent(event, null);
	}
```
发布事件。
org.springframework.context.support.AbstractApplicationContext#publishEvent(java.lang.Object, org.springframework.core.ResolvableType)

```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
		}

		// Decorate event as an ApplicationEvent if necessary必要时将事件装饰为ApplicationEvent
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized如果可能的话，现在就多播——或者在初始化多播之后延迟
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
//			广播事件
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...通过父上下文发布事件…
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}
```
org.springframework.context.support.AbstractApplicationContext#refresh

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
//		门面模式，刷新和销毁上下文必须是同步的，这里加了一个锁
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.准备这个上下文来刷新。
//			上下文刷新的前置工作
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.初始化内部beanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.允许在上下文子类中处理bean工厂的后处理。spring的扩展点之一
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.调用在上下文中注册为bean的工厂处理器。
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.注册bean的处理器，拦截bean的创建
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.为这个上下文初始化消息源。
				initMessageSource();

				// Initialize event multicaster for this context.为这个上下文初始化事件多播器。
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.在特定的上下文子类中初始化其他特殊bean。spring的扩展点之一
				onRefresh();

				// Check for listener beans and register them.检查侦听器bean并注册它们。
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.实例化所有剩余的(非延迟-init)单例。
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.最后一步发布刷新事件
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.销毁已经创建的单例，以避免悬空资源。
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
//				重新设置Spring核心中的常用内省缓存，因为我们可能再也不需要单例bean的元数据了
				resetCommonCaches();
			}
		}
	}
```
上下文刷新方法，这个方法是门面模式、模板方法模式实现。
刷新过程是同步的。处理准备刷新逻辑处理，获取BeanFactory，准备beanFactory，执行BeanFactory的后置处理器，执行BeanFactory的后置处理器，注册beanPostProcessors，初始化事件传播器，执行上下文刷新方法org.springframework.context.support.AbstractApplicationContext#onRefresh，这里是模板方法实现，子类需要覆盖这个模板方法。实例化非延迟加载的单例bean，完成啥下文刷新处理。出现异常销毁bean，取消刷新逻辑，最后删除单例bean的元数据。
org.springframework.context.support.AbstractApplicationContext#prepareRefresh

```java
	protected void prepareRefresh() {
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		// Initialize any placeholder property sources in the context environment 解析占位符
		initPropertySources();

		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
//		验证Required的属性
		getEnvironment().validateRequiredProperties();

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```
上下文刷新前置处理。
设置设置上下文关闭开关false，设置上下文激活开关true，处理属性占位符表示的资源，org.springframework.context.support.AbstractApplicationContext#initPropertySources

```java
protected void initPropertySources() {
		// For subclasses: do nothing by default.
	}
```
需要子类覆盖这个方法，父类中不做任何处理。
org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
//		刷新beanFactory
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```
指定上下文刷新的BeanFactory，org.springframework.context.support.AbstractApplicationContext#refreshBeanFactory，子类需要覆盖这个方法，获取BeanFactory，org.springframework.context.support.AbstractApplicationContext#getBeanFactory子类需要覆盖这个方法。
org.springframework.context.support.AbstractApplicationContext#prepareBeanFactory

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.告诉内部bean工厂使用上下文的类装入器等。
		beanFactory.setBeanClassLoader(getClassLoader());
//		配置bean表达式解析器
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.配置beanFactory的上下文回调
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
//		忽略自动装配的接口
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.在普通工厂中没有注册为可解析类型的BeanFactory接口。
//		MessageSource注册(并为自动装配找到)为bean。
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.将早期用于检测内部bean的后处理器注册为applicationlistener。
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.为类型匹配设置一个临时类加载器。
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```
BeanFactory的前置处理，设置BeanExpressionResolver、PropertyEditorRegistrar，添加ApplicationContextAwareProcessor，添加忽略自动注入的接口，注册解析依赖的类型。添加org.springframework.context.support.ApplicationListenerDetector BeanPostProcessor。

org.springframework.context.support.AbstractApplicationContext#postProcessBeanFactory

```java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	}
```
BeanDefinition注册之后bean初始化之前执行。可以对BeanFactory进行操作。
org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```
调用所有已注册的BeanFactoryPostProcessor，必须在singleton实例化之前调用。

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
//		执行BeanDefinition、beanFactory的后置处理器
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)//检测LoadTimeWeaver并准备编织，如果同时发现
//(例如，通过ConfigurationClassPostProcessor注册的@Bean方法)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```
执行BeanDefinition、beanFactory的后置处理器，org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, java.util.List<org.springframework.beans.factory.config.BeanFactoryPostProcessor>)

```java
	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<>();

		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<>();
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new LinkedList<>();

			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
//					bean定义注册
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					registryProcessors.add(registryProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.//不要在这里初始化factorybean:我们需要保留所有常规bean
//未初始化，让bean工厂的后置处理器应用于它们!
//将实现的beandefinitionregistrypostprocessor分开
//优先顺序，顺序等等。
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.首先，调用实现priorityor的beandefinitionregistrypostprocessor。
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
//					从BeanFactory中获取bean进行注册
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
//			对PostProcessors进行排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.接下来，调用实现Ordered的beandefinitionregistrypostprocessor。
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.最后，调用所有其他的beandefinitionregistrypostprocessor，直到没有其他的出现。
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}

			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.现在，调用到目前为止处理的所有处理器的postProcessBeanFactory回调。
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.调用用上下文实例注册的工厂处理器。
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!//不要在这里初始化factorybean:我们需要保留所有常规bean
//未初始化，让bean工厂的后置处理器应用于它们!
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.//分隔实现priorityor的beanfactorypostprocessor，
//点了餐，剩下的。
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above跳过-已经在第一阶段处理以上
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.首先，调用实现priorityor的beanfactorypostprocessor。
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.接下来，调用实现Ordered的beanfactorypostprocessor。
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.最后，调用所有其他beanfactorypostprocessor。
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...//清除缓存的合并bean定义，因为后处理器可能有
//修改了原始的元数据，例如替换了值中的占位符…
		beanFactory.clearMetadataCache();
	}
```
执行BeanDefinition、BeanFactory的后置处理器。
org.springframework.context.support.AbstractApplicationContext#registerBeanPostProcessors

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
	}
```
实例化并调用所有已注册的BeanPostProcessor bean，如果给定的话，要遵守明确的顺序。
 必须在应用程序bean的任何实例化之前调用。
org.springframework.context.support.PostProcessorRegistrationDelegate#registerBeanPostProcessors(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, org.springframework.context.support.AbstractApplicationContext)

```java
	public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.//注册BeanPostProcessorChecker，记录信息信息时
//一个bean是在BeanPostProcessor实例化期间创建的，也就是在什么时候
//一个bean不能被所有的beanpostprocessor处理。
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.//在实现priorityor的beanpostprocessor之间进行分离，
//点了餐，剩下的。
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
//				从BeanFactory中获取BeanPostProcessor
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.首先，注册实现priorityor的beanpostprocessor。
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.接下来，注册实现Ordered的beanpostprocessor。
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.现在，注册所有常规的beanpostprocessor。
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.最后，重新注册所有的内部beanpostprocessor。
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).//将检测内部bean的后处理器重新注册为applicationlistener，
//将它移到处理器链的末端(用于获取代理等)。
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```
对BeanDefinition、BeanFactory的后置处理器beanPostProcessors进行注册。
org.springframework.context.support.AbstractApplicationContext#initApplicationEventMulticaster

```java
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
						APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
						"': using default [" + this.applicationEventMulticaster + "]");
			}
		}
	}
```
初始化ApplicationEventMulticaster。如果上下文没有定义，则使用SimpleApplicationEventMulticaster。
org.springframework.context.support.AbstractApplicationContext#onRefresh

```java
protected void onRefresh() throws BeansException {
		// For subclasses: do nothing by default.
	}
```
子类需要覆盖这个方法，这个方法用来处理这个上下文刷新操作，进行bean初始化，实例化单例的bean。
org.springframework.context.support.AbstractApplicationContext#registerListeners

```java
protected void registerListeners() {
		// Register statically specified listeners first.首先注册静态指定的侦听器。
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!//不要在这里初始化factorybean:我们需要保留所有常规bean
//未初始化，让后处理器应用于他们!
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...发布早期的应用事件，现在我们终于有一个多播…
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```
添加监听器到上下文事件广播器中，发布提前要处理的应用上下文事件。
org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization

```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.为上下文设置转换服务
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
//			给beanFactory设置转换器
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.//如果没有bean后处理器，注册一个默认的嵌入式值解析器
//(例如PropertyPlaceholderConfigurer bean)以前注册过:
//此时，主要用于解析注释属性值。
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early. 初始化LoadTimeWeaverAware bean
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.停止使用临时类加载器进行类型匹配。
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.允许缓存所有bean定义元数据，而不期望进一步更改。
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.实例化所有剩余的(非延迟-init)单例。
		beanFactory.preInstantiateSingletons();
	}

```
设置conversionService，冻结配置，初始化单例非延迟加载的bean。
org.springframework.beans.factory.support.DefaultListableBeanFactory#freezeConfiguration

```java
@Override
	public void freezeConfiguration() {
		this.configurationFrozen = true;
		this.frozenBeanDefinitionNames = StringUtils.toStringArray(this.beanDefinitionNames);
	}
```
设置配置冻结属性，锁定BeanDefinitionNames。
org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons

```java
@Override
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
//		迭代一个副本，以允许init方法，它反过来注册新的bean定义。虽然这可能不是常规工厂引导的一部分，但它可以正常工作。
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...触发立即加载的bean初始化
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
//			非抽象、非延迟加载的bean立即加载
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
//						如果是立即加载
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...为所有适用的bean触发后初始化回调…
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
//						单例bean创建完成后回调
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```
单例非延迟加载的bean进行初始化，执行bean的org.springframework.beans.factory.SmartInitializingSingleton#afterSingletonsInstantiated在单例bean实例化后调用。
org.springframework.context.support.AbstractApplicationContext#finishRefresh

```java
protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).清除上下文级别的资源缓存(例如扫描的ASM元数据)。
		clearResourceCaches();

		// Initialize lifecycle processor for this context.为这个上下文初始化生命周期处理器。
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.首先传播刷新到生命周期处理器。
		getLifecycleProcessor().onRefresh();

		// Publish the final event.发布最后的事件。
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```
完成上下文的刷新处理，这个方法是门面模式实现。
org.springframework.context.support.AbstractApplicationContext#initLifecycleProcessor

```java
protected void initLifecycleProcessor() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
			this.lifecycleProcessor =
					beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
			}
		}
		else {
			DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
			defaultProcessor.setBeanFactory(beanFactory);
			this.lifecycleProcessor = defaultProcessor;
			beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate LifecycleProcessor with name '" +
						LIFECYCLE_PROCESSOR_BEAN_NAME +
						"': using default [" + this.lifecycleProcessor + "]");
			}
		}
	}
```
初始化LifecycleProcessor，如果BeanFactory中存在lifecycleProcessor bean就获取返回，如果没有就创建DefaultLifecycleProcessor就注册单例bean lifecycleProcessor到BeanFactory中。调用org.springframework.context.LifecycleProcessor#onRefresh上下文刷新通知方法，可以自定义自己的实现。
发布ContextRefreshedEvent事件。

org.springframework.context.support.AbstractApplicationContext#destroyBeans

```java
protected void destroyBeans() {
//		获取beanFactory，并销毁单例的bean
		getBeanFactory().destroySingletons();
	}
```
销毁单例bean的模板方法，org.springframework.beans.factory.config.ConfigurableBeanFactory#destroySingletons

```java
void destroySingletons();
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#destroySingletons

```java
@Override
	public void destroySingletons() {
		super.destroySingletons();
//		清除记录的单例beanName的缓存
		this.manualSingletonNames.clear();
		clearByTypeCache();
	}
```
销毁单例bean。
org.springframework.context.support.AbstractApplicationContext#cancelRefresh

```java
protected void cancelRefresh(BeansException ex) {
		this.active.set(false);
	}
```
取消上下文的刷新，设置上下文是否是激活的active标识为false。
org.springframework.context.support.AbstractApplicationContext#registerShutdownHook

```java
	@Override
	public void registerShutdownHook() {
		if (this.shutdownHook == null) {
			// No shutdown hook registered yet.
			this.shutdownHook = new Thread() {
				@Override
				public void run() {
					synchronized (startupShutdownMonitor) {
						doClose();
					}
				}
			};
			Runtime.getRuntime().addShutdownHook(this.shutdownHook);
		}
	}
```
注册jvm关闭钩子方法，org.springframework.context.support.AbstractApplicationContext#doClose关闭程序上下文，发布ContextClosedEvent事件，关闭应用程序上下文中BeanFactory的单例bean，关闭jvm关闭时的钩子方法。

```java
	protected void doClose() {
		if (this.active.get() && this.closed.compareAndSet(false, true)) {
			if (logger.isInfoEnabled()) {
				logger.info("Closing " + this);
			}

			LiveBeansView.unregisterApplicationContext(this);

			try {
				// Publish shutdown event.
				publishEvent(new ContextClosedEvent(this));
			}
			catch (Throwable ex) {
				logger.warn("Exception thrown from ApplicationListener handling ContextClosedEvent", ex);
			}

			// Stop all Lifecycle beans, to avoid delays during individual destruction.停止所有生命周期bean，以避免在个别销毁期间出现延迟。
			if (this.lifecycleProcessor != null) {
				try {
					this.lifecycleProcessor.onClose();
				}
				catch (Throwable ex) {
					logger.warn("Exception thrown from LifecycleProcessor on context close", ex);
				}
			}

//			门面模式
			// Destroy all cached singletons in the context's BeanFactory.销毁上下文的BeanFactory中所有缓存的单例。
			destroyBeans();

			// Close the state of this context itself.关闭上下文本身的状态。
			closeBeanFactory();

			// Let subclasses do some final clean-up if they wish...让子类做一些最后的清理，如果他们希望…
			onClose();

			this.active.set(false);
		}
	}
```
如果上下文是否激活标识active为true，上下文关闭标识closed标识为false可以设置为true，发布ContextClosedEvent事件，执行org.springframework.context.LifecycleProcessor#onClose方法，上下文关闭通知。销毁单例bean，关闭BeanFactory，执行子类的上下文关闭方法，上下文是否激活标识active为false。org.springframework.context.support.AbstractApplicationContext#destroyBeans

```java
protected void destroyBeans() {
//		获取beanFactory，并销毁单例的bean
		getBeanFactory().destroySingletons();
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#destroySingletons

```java
@Override
	public void destroySingletons() {
		super.destroySingletons();
//		清除记录的单例beanName的缓存
		this.manualSingletonNames.clear();
		clearByTypeCache();
	}
```
销毁单例的bean。
org.springframework.context.support.AbstractApplicationContext#onClose

```java
protected void onClose() {
		// For subclasses: do nothing by default.
	}
```
子类可以覆盖上下文关闭方法，doClose方法执行完之后执行这个方法。

```java
@Deprecated
	public void destroy() {
		close();
	}
```
org.springframework.context.support.AbstractApplicationContext#close

```java
@Override
	public void close() {
		synchronized (this.startupShutdownMonitor) {
			doClose();
			// If we registered a JVM shutdown hook, we don't need it anymore now:
			// We've already explicitly closed the context.//如果我们注册了一个JVM关机挂钩，我们现在不需要它了:
//我们已经明确地关闭了上下文。
			if (this.shutdownHook != null) {
				try {
					Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
				}
				catch (IllegalStateException ex) {
					// ignore - VM is already shutting down
				}
			}
		}
	}
```
关闭应用程序上下文，销毁BeanFactory中的所有bean，doClose方法进行实际的关闭过程，删除jvm的关闭钩子。

```java
@Override
	public void start() {
		getLifecycleProcessor().start();
		publishEvent(new ContextStartedEvent(this));
	}
```
重写生命周期的start方法，发布ContextStartedEvent事件。

```java
@Override
	public void stop() {
		getLifecycleProcessor().stop();
		publishEvent(new ContextStoppedEvent(this));
	}
```
重写生命周期的stop方法，发布ContextStoppedEvent事件。

```java
@Override
	public boolean isRunning() {
		return (this.lifecycleProcessor != null && this.lifecycleProcessor.isRunning());
	}
```
返回程序上下文是否在运行running属性值。

```java
protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;
```
子类必须实现这个方法来执行实际的配置加载。

```java
protected abstract void closeBeanFactory();
```
子类必须实现此方法以释放其内部beanFactory。

```java
@Override
	public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```
子类必须返回其内部的beanFactory。

org.springframework.context.support.AbstractRefreshableApplicationContext
应用程序上下文实现的基类，支持多次调用refresh()，每次创建一个新的内部BeanFactory实例。由子类实现的惟一方法是loadbeandefinition，它在每次刷新时被调用。一个具体的实现应该将bean定义加载到给定的DefaultListableBeanFactory中，通常委托给一个或多个特定的beanDefinitionReader。

```java
@Nullable
	private Boolean allowBeanDefinitionOverriding;
```
是否允许BeanDefinition被覆盖。

```java
@Nullable
	private Boolean allowCircularReferences;
```
是否允许bean之前的循环引用。

```java
@Nullable
	private DefaultListableBeanFactory beanFactory;
```
内部维护一个BeanFactory。
org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory

```java
	@Override
	protected final void refreshBeanFactory() throws BeansException {
//		如果存在beanFactory
		if (hasBeanFactory()) {
//			销毁bean
			destroyBeans();
//			关闭beanFactory
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```
创建一个BeanFactory，实现上下文底层的BeanFactory刷新。
如果存在BeanFactory，销毁beans，关闭BeanFactory，否则创建DefaultListableBeanFactory类型BeanFactory实例，定制化beanFactory，加载BeanDefinition。
org.springframework.context.support.AbstractApplicationContext#destroyBeans

```java
protected void destroyBeans() {
//		获取beanFactory，并销毁单例的bean
		getBeanFactory().destroySingletons();
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#destroySingletons

```java
@Override
	public void destroySingletons() {
		super.destroySingletons();
//		清除记录的单例beanName的缓存
		this.manualSingletonNames.clear();
		clearByTypeCache();
	}
```
销毁单例beans。
org.springframework.context.support.AbstractRefreshableApplicationContext#closeBeanFactory

```java
@Override
	protected final void closeBeanFactory() {
		synchronized (this.beanFactoryMonitor) {
			if (this.beanFactory != null) {
				this.beanFactory.setSerializationId(null);
				this.beanFactory = null;
			}
		}
	}
```
关闭BeanFactory。
org.springframework.context.support.AbstractRefreshableApplicationContext#createBeanFactory

```java
protected DefaultListableBeanFactory createBeanFactory() {
		return new DefaultListableBeanFactory(getInternalParentBeanFactory());
	}
```
返回上一级BeanFactory。
org.springframework.context.support.AbstractRefreshableApplicationContext#customizeBeanFactory

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
//		是否允许bean定义覆盖
		if (this.allowBeanDefinitionOverriding != null) {
			beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
//		是否允许bean之间循环引用
		if (this.allowCircularReferences != null) {
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
```
设置BeanFactory的allowBeanDefinitionOverriding是否允许BeanDefinition覆盖，allowCircularReferences是否允许bean之间循环引用。
org.springframework.context.support.AbstractRefreshableApplicationContext#loadBeanDefinitions

```java
protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory)
			throws BeansException, IOException;
```
加载BeanDefinition，子类需要实现，这里是模板方法实现。
org.springframework.context.support.AbstractRefreshableApplicationContext#cancelRefresh

```java
@Override
	protected void cancelRefresh(BeansException ex) {
		synchronized (this.beanFactoryMonitor) {
			if (this.beanFactory != null)
				this.beanFactory.setSerializationId(null);
		}
		super.cancelRefresh(ex);
	}
```
取消刷新，设置上下文是否是激活的active属性为false。

org.springframework.context.support.AbstractRefreshableConfigApplicationContext
用于添加指定配置位置的通用处理的AbstractRefreshableApplicationContext子类。作为基于xml的应用程序上下文实现的基类，如ClassPathXmlApplicationContext和FileSystemXmlApplicationContext。
org.springframework.context.support.AbstractRefreshableConfigApplicationContext#setConfigLocation

```java
public void setConfigLocation(String location) {
		setConfigLocations(StringUtils.tokenizeToStringArray(location, CONFIG_LOCATION_DELIMITERS));
	}
```
设置山下文配置文件，多个配置文件用，分开。
org.springframework.context.support.AbstractRefreshableConfigApplicationContext#setConfigLocations

```java
public void setConfigLocations(@Nullable String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
//				解析配置文件后把配置文件路径放到spring上下文中
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}
```
解析上下文spring配置文件路径，支持占位符。

```java
@Nullable
	protected String[] getConfigLocations() {
		return (this.configLocations != null ? this.configLocations : getDefaultConfigLocations());
	}
```
返回默认的spring上下文的配置文件路径，这里默认配置文件路径没有设置，需要子类进行方法覆盖进行处理。
org.springframework.context.support.AbstractRefreshableConfigApplicationContext#getDefaultConfigLocations

```java
@Nullable
	protected String[] getDefaultConfigLocations() {
		return null;
	}
```
org.springframework.context.support.AbstractRefreshableConfigApplicationContext#afterPropertiesSet

```java
@Override
	public void afterPropertiesSet() {
		if (!isActive()) {
			refresh();
		}
	}
```
覆盖了这个方法，org.springframework.beans.factory.InitializingBean#afterPropertiesSet，BeanFactory设置了所有的bean属性之后调用，和bean的初始化方法作用类似。这里的处理是如果上下文active属性是false，上下文没有做过刷新操作调用模板方法org.springframework.context.support.AbstractApplicationContext#refresh进行上下文刷新处理。

org.springframework.context.support.AbstractXmlApplicationContext
为ApplicationContext实现提供了方便的基类，从包含XmlBeanDefinitionReader理解的bean定义的XML文档中提取配置。
org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.support.DefaultListableBeanFactory)

```java
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.为给定的BeanFactory创建一个新的XmlBeanDefinitionReader。
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.使用该上下文的资源加载环境配置bean定义阅读器。
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
//		加载资源
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.允许子类提供读取器的自定义初始化，然后继续加载bean定义。
//		初始化bean定义
		initBeanDefinitionReader(beanDefinitionReader);
//		加载bean定义
		loadBeanDefinitions(beanDefinitionReader);
	}
```
通过XmlBeanDefinitionReader加载bean定义。
初始化XmlBeanDefinitionReader并初始化，加载BeanDefinition。
org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.xml.XmlBeanDefinitionReader)

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
//		获取配置文件路径
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
//			加载bean定义
			reader.loadBeanDefinitions(configLocations);
		}
	}
```
使用给定的XmlBeanDefinitionReader加载bean定义。
获取spring上下文指定的配置文件并加载BeanDefinition。

```java
@Nullable
	protected Resource[] getConfigResources() {
		return null;
	}
```
返回指定的spring上下文配置文件。

org.springframework.context.support.ClassPathXmlApplicationContext
独立的XML应用程序上下文，从类路径中获取上下文定义文件，在多个配置位置的情况下，稍后的bean定义将覆盖前面加载文件中定义的那些。这可以通过使用额外的XML文件来有意地重写某些bean定义。
org.springframework.context.support.ClassPathXmlApplicationContext#ClassPathXmlApplicationContext(java.lang.String[], boolean, org.springframework.context.ApplicationContext)

```java
	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```
设置spring上下文配置文件路径，如果需要进行上下文刷新调用org.springframework.context.support.AbstractApplicationContext#refresh模板方法进行上下文刷新。
org.springframework.context.support.ClassPathXmlApplicationContext#ClassPathXmlApplicationContext(java.lang.String[], java.lang.Class<?>, org.springframework.context.ApplicationContext)

```java
public ClassPathXmlApplicationContext(String[] paths, Class<?> clazz, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		Assert.notNull(paths, "Path array must not be null");
		Assert.notNull(clazz, "Class argument must not be null");
		this.configResources = new Resource[paths.length];
		for (int i = 0; i < paths.length; i++) {
			this.configResources[i] = new ClassPathResource(paths[i], clazz);
		}
		refresh();
	}
```
设置spring上下文配置文件，刷新上下文。

org.springframework.context.support.FileSystemXmlApplicationContext
独立的XML应用程序上下文，从文件系统或url中获取上下文定义文件，在多个配置位置的情况下，稍后的bean定义将覆盖前面加载文件中定义的那些。这可以通过使用额外的XML文件来有意地重写某些bean定义。
org.springframework.context.support.FileSystemXmlApplicationContext#FileSystemXmlApplicationContext(java.lang.String[], boolean, org.springframework.context.ApplicationContext)

```java
public FileSystemXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```
设置spring上下文配置文件，如果需要上下文刷新调用org.springframework.context.support.AbstractApplicationContext#refresh方法进行上下文刷新。
org.springframework.context.support.FileSystemXmlApplicationContext#getResourceByPath

```java
@Override
	protected Resource getResourceByPath(String path) {
		if (path.startsWith("/")) {
			path = path.substring(1);
		}
		return new FileSystemResource(path);
	}
```
根据路径解析spring上下文配置文件路径。
