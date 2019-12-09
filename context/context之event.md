# context之event

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1575871128578-45fa51d0-c4c8-4320-85b9-537508dfcdb2.png#align=left&display=inline&height=294&name=image.png&originHeight=588&originWidth=729&size=67889&status=done&style=none&width=364.5)

本次开始介绍spring实现的事务模型，观察者模式的实现，我们常用的有java自带的观察者模式，还有guava提供的观察者模式实现的事件模型，如果项目中集成了spring，用spring的事件模型还是很方便的。

先介绍了spring ioc实现哪些事件。

org.springframework.context.ApplicationEvent
application事件抽象实现，这个抽象类集成了java.util.EventObject。

```java
private final long timestamp;
```
事件发生的时间。
org.springframework.context.PayloadApplicationEvent
可传递属性的application事件，主要用于框架内部使用。

```java
private final T payload;
```
事件传播的参数。

org.springframework.context.event.ApplicationContextEvent
用于应用程序上下文的事件的基类。

```java
public final ApplicationContext getApplicationContext() {
		return (ApplicationContext) getSource();
	}
```
获取事件源。

org.springframework.context.event.ContextStoppedEvent
当应用程序上下文停止时引发的事件。

org.springframework.context.event.ContextClosedEvent
当应用程序上下文关闭时引发的事件。

org.springframework.context.event.ContextRefreshedEvent
在ApplicationContext被初始化或刷新时引发的事件。

程序可有监听应用上下文刷新、关闭事件实现自己的业务逻辑。

org.springframework.context.event.ContextStartedEvent
当应用程序上下文启动时引发的事件。

org.springframework.context.ApplicationListener
由应用程序事件侦听器实现的接口。基于标准java.util。观察者设计模式的EventListener接口。ApplicationListener可以通用地声明它感兴趣的事件类型。当在Spring ApplicationContext中注册时，事件将相应地进行筛选，而侦听器只会被调用来匹配事件对象。

```java
void onApplicationEvent(E event);
```
处理应用程序的事件。

org.springframework.context.event.SmartApplicationListener
标准ApplicationListener接口的扩展。

```java
boolean supportsEventType(Class<? extends ApplicationEvent> eventType);
```
确定该侦听器是否实际支持给定的事件类型。

```java
boolean supportsSourceType(@Nullable Class<?> sourceType);
```
确定该侦听器是否实际支持给定的源类型。

org.springframework.context.event.GenericApplicationListener
标准ApplicationListener接口的扩展。适当处理基于通用的事件。

```java
boolean supportsEventType(ResolvableType eventType);
```
确定该侦听器是否实际支持给定的事件类型。

```java
boolean supportsSourceType(@Nullable Class<?> sourceType);
```
确定该侦听器是否实际支持给定的源类型。

org.springframework.context.event.GenericApplicationListenerAdapter
org.springframework.context.event.GenericApplicationListener的装饰器实现，这里是装饰器模式实现。

```java
private final ApplicationListener<ApplicationEvent> delegate;
```
被装饰的ApplicationListener。

```java
@Override
	public void onApplicationEvent(ApplicationEvent event) {
		this.delegate.onApplicationEvent(event);
	}
```
事件到达事件。

org.springframework.context.event.SourceFilteringListener
org.springframework.context.event.GenericApplicationListener、org.springframework.context.event.SmartApplicationListener装饰器实现，这里是装饰器模式实现。它从指定的事件源筛选事件，调用它的委托listener来匹配应用程序事件对象。

```java
@Nullable
	private GenericApplicationListener delegate;
```
被装饰的ApplicationListener。

```java
protected SourceFilteringListener(Object source) {
		this.source = source;
	}
```
返回事件listener的事件源。

```java
@Override
	public void onApplicationEvent(ApplicationEvent event) {
		if (event.getSource() == this.source) {
			onApplicationEventInternal(event);
		}
	}
```
事件到达事件，org.springframework.context.event.SourceFilteringListener#onApplicationEventInternal

```java
protected void onApplicationEventInternal(ApplicationEvent event) {
		if (this.delegate == null) {
			throw new IllegalStateException(
					"Must specify a delegate object or override the onApplicationEventInternal method");
		}
		this.delegate.onApplicationEvent(event);
	}
```
调用listener的委托对象delegate处理事件到来事件。

org.springframework.context.event.EventListener
标记方法作为应用程序事件的侦听器的注释。@EventListener注释的处理是通过内部EventListenerMethodProcessor bean来执行的。则该方法可以声明一个反映事件类型的单个参数。如果一个带注释的方法支持多个事件类型，那么这个注释可以使用类属性引用一个或多个受支持的事件类型。通过<context:annotation-config/>或<context:component-scan/>元素使用XML配置时自动注册。还可以定义要调用某个事件的侦听器的顺序。为此，在这个事件监听器注释旁边添加@Order注释。

```java
@AliasFor("classes")
	Class<?>[] value() default {};

@AliasFor("value")
	Class<?>[] classes() default {};
```
这个侦听器处理的事件类。

```java
String condition() default "";
```
用于使事件处理有条件的Spring表达式。

org.springframework.context.support.ApplicationListenerDetector
BeanPostProcessor检测实现ApplicationListener接口的bean。

```java
private final transient AbstractApplicationContext applicationContext;
```
applicationContext。

```java
private final transient Map<String, Boolean> singletonNames = new ConcurrentHashMap<>(256);
```
单例applicationListener的name
```java
@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		this.singletonNames.put(beanName, beanDefinition.isSingleton());
	}
```
BeanDefinition后置处理，重写了org.springframework.beans.factory.support.MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition方法。

```java
@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}
```
bean初始化之前操作，默认返回原始的bean。重写了org.springframework.beans.factory.config.BeanPostProcessor#postProcessBeforeInitialization方法。

```java
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		if (bean instanceof ApplicationListener) {
			// potentially not detected as a listener by getBeanNamesForType retrievalgetBeanNamesForType检索可能没有检测到侦听器
			Boolean flag = this.singletonNames.get(beanName);
			if (Boolean.TRUE.equals(flag)) {
				// singleton bean (top-level or inner): register on the fly单例bean(顶层或内部):动态注册
				this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
			}
			else if (Boolean.FALSE.equals(flag)) {
				if (logger.isWarnEnabled() && !this.applicationContext.containsBean(beanName)) {
					// inner bean with other scope - can't reliably process events
					logger.warn("Inner bean '" + beanName + "' implements ApplicationListener interface " +
							"but is not reachable for event multicasting by its containing ApplicationContext " +
							"because it does not have singleton scope. Only top-level listener beans are allowed " +
							"to be of non-singleton scope.");
				}
				this.singletonNames.remove(beanName);
			}
		}
		return bean;
	}
```
bean初始化以后后置处理，添加到applicationListener到applicationContext中。

```java
@Override
	public void postProcessBeforeDestruction(Object bean, String beanName) {
		if (bean instanceof ApplicationListener) {
			try {
				ApplicationEventMulticaster multicaster = this.applicationContext.getApplicationEventMulticaster();
				multicaster.removeApplicationListener((ApplicationListener<?>) bean);
				multicaster.removeApplicationListenerBean(beanName);
			}
			catch (IllegalStateException ex) {
				// ApplicationEventMulticaster not initialized yet - no need to remove a listenerApplicationEventMulticaster还没有初始化-不需要移除一个监听器
			}
		}
	}
```
bean销毁前的后置处理，删除applicationListener，重写了org.springframework.beans.factory.config.DestructionAwareBeanPostProcessor#postProcessBeforeDestruction方法。

```java
@Override
	public boolean requiresDestruction(Object bean) {
		return (bean instanceof ApplicationListener);
	}
```
重写了org.springframework.beans.factory.config.DestructionAwareBeanPostProcessor#requiresDestruction方法，确定给定的bean实例是否需要此后处理器销毁。

org.springframework.context.event.EventListenerMethodProcessor
将EventListener注释方法注册为单独的ApplicationListener实例。

```java
@Nullable
	private ConfigurableApplicationContext applicationContext;
```
applicationContext。

```java
private final Set<Class<?>> nonAnnotatedClasses = Collections.newSetFromMap(new ConcurrentHashMap<>(64));
```
带@EventListener注解的类，这里是用线程安全的set实现的缓存，用ConcurrentHashMap实现的线程安全的set。

```java
@Override
	public void setApplicationContext(ApplicationContext applicationContext) {
		Assert.isTrue(applicationContext instanceof ConfigurableApplicationContext,
				"ApplicationContext does not implement ConfigurableApplicationContext");
		this.applicationContext = (ConfigurableApplicationContext) applicationContext;
	}
```
重写了org.springframework.context.ApplicationContextAware#setApplicationContext方法，可以获得applicationContext实现自己的业务逻辑。

```java
	@Override
	public void afterSingletonsInstantiated() {//所有的单例bean创建完成后执行这个方法
//		下面开始创建ApplicationListener，获得创建ApplicationListener的工厂集合
		List<EventListenerFactory> factories = getEventListenerFactories();
//		获得配置上下文对象
		ConfigurableApplicationContext context = getApplicationContext();
//		获得bean的名字
		String[] beanNames = context.getBeanNamesForType(Object.class);
		for (String beanName : beanNames) {
			if (!ScopedProxyUtils.isScopedTarget(beanName)) {
				Class<?> type = null;
				try {
//					获取bean的初始目标类的类型
					type = AutoProxyUtils.determineTargetClass(context.getBeanFactory(), beanName);
				}
				catch (Throwable ex) {
					// An unresolvable bean type, probably from a lazy bean - let's ignore it.
					if (logger.isDebugEnabled()) {
						logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
					}
				}
				if (type != null) {
//					type表示的类或接口的超类或超接口是否和ScopedObject一样
					if (ScopedObject.class.isAssignableFrom(type)) {
						try {
							Class<?> targetClass = AutoProxyUtils.determineTargetClass(
									context.getBeanFactory(), ScopedProxyUtils.getTargetBeanName(beanName));
							if (targetClass != null) {
								type = targetClass;
							}
						}
						catch (Throwable ex) {
							// An invalid scoped proxy arrangement - let's ignore it.
							if (logger.isDebugEnabled()) {
								logger.debug("Could not resolve target bean for scoped proxy '" + beanName + "'", ex);
							}
						}
					}
					try {
						processBean(factories, beanName, type);
					}
					catch (Throwable ex) {
						throw new BeanInitializationException("Failed to process @EventListener " +
								"annotation on bean with name '" + beanName + "'", ex);
					}
				}
			}
		}
	}
```
重写了org.springframework.beans.factory.SmartInitializingSingleton#afterSingletonsInstantiated方法，单例bean实例化完毕后的后置处理。
这里的处理是解析带有@EventListener注解的Listener。
org.springframework.context.event.EventListenerMethodProcessor#processBean

```java
	protected void processBean(
			final List<EventListenerFactory> factories, final String beanName, final Class<?> targetType) {

//	如果目标对象不是注解类型
		if (!this.nonAnnotatedClasses.contains(targetType)) {
			Map<Method, EventListener> annotatedMethods = null;
			try {
//				找到加了@EventListener这个注解的方法
				annotatedMethods = MethodIntrospector.selectMethods(targetType,
						new MethodIntrospector.MetadataLookup<EventListener>() {
							@Override
							public EventListener inspect(Method method) {
								return AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class);
							}
						});
			}
			catch (Throwable ex) {
				// An unresolvable type in a method signature, probably from a lazy bean - let's ignore it.方法签名中不可解析的类型，可能来自懒bean——让我们忽略它。
				if (logger.isDebugEnabled()) {
					logger.debug("Could not resolve methods for bean with name '" + beanName + "'", ex);
				}
			}
			if (CollectionUtils.isEmpty(annotatedMethods)) {
				this.nonAnnotatedClasses.add(targetType);
				if (logger.isTraceEnabled()) {
					logger.trace("No @EventListener annotations found on bean class: " + targetType.getName());
				}
			}
			else {
				// Non-empty set of methods 获取配置上下文对象
				ConfigurableApplicationContext context = getApplicationContext();
				for (Method method : annotatedMethods.keySet()) {
					for (EventListenerFactory factory : factories) {
						if (factory.supportsMethod(method)) {
//							获取可调用的方法
							Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));
//							创建applicationListener对象，返回的是GenericApplicationListener的adaptor ApplicationListenerMethodAdapter对象
							ApplicationListener<?> applicationListener =
									factory.createApplicationListener(beanName, targetType, methodToUse);
							if (applicationListener instanceof ApplicationListenerMethodAdapter) {
//								如果applicationListenr的具体类型是ApplicationListenerMethodAdapter，就进行初始化
								((ApplicationListenerMethodAdapter) applicationListener).init(context, this.evaluator);
							}
//							如果applicationListenr的具体类型是ApplicationListenerMethodTransactionalAdapter，把这个监听器配置到spring配置上下文中，这个监听器类型是支持事务的监听器，在spring-tx包源码解析中会具体解析
							context.addApplicationListener(applicationListener);
							break;
						}
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug(annotatedMethods.size() + " @EventListener methods processed on bean '" +
							beanName + "': " + annotatedMethods);
				}
			}
		}
	}
```
解析targetType带@EventListener的方法，如果没有这个注解，把这个类型过滤掉添加到nonAnnotatedClasses set中，如果找到targetType上带@EventListener注解的方法，找到支持这个方法的EventListenerFactory创建ApplicationListener并添加applicationContext中。

org.springframework.context.event.EventListenerFactory
用于为@EventListener注释的方法创建ApplicationListener的策略接口

```java
boolean supportsMethod(Method method);
```
指定该beanFactory是否支持指定的方法。

```java
ApplicationListener<?> createApplicationListener(String beanName, Class<?> type, Method method);
```
为指定的方法创建一个ApplicationListener。

org.springframework.context.event.DefaultEventListenerFactory
支持常规@EventListener注释的默认EventListenerFactory实现。

```java
public boolean supportsMethod(Method method) {
		return true;
	}
```
默认支持所有方法。

```java
@Override
	public ApplicationListener<?> createApplicationListener(String beanName, Class<?> type, Method method) {
		return new ApplicationListenerMethodAdapter(beanName, type, method);
	}
```
创建ApplicationListener。

先介绍下org.springframework.context.event.ApplicationListenerMethodAdapter
GenericApplicationListener适配器，它将事件的处理委托给@EventListener注释方法。

```java
private final String beanName;
```
listener。

```java
private final Method method;
```
listener的方法。

```java
private final Class<?> targetClass;
```
listener的class。

```java
private final Method bridgedMethod;
```
listener方法的桥接方法。

```java
@Nullable
	private ApplicationContext applicationContext;
```
applicationContext。

```java
@Override
	public void onApplicationEvent(ApplicationEvent event) {
		processEvent(event);
	}
```
事件到达方法。
org.springframework.context.event.ApplicationListenerMethodAdapter#processEvent
处理指定的ApplicationEvent。

```java
public void processEvent(ApplicationEvent event) {
		Object[] args = resolveArguments(event);
		if (shouldHandle(event, args)) {
//			执行事件监听器
			Object result = doInvoke(args);
			if (result != null) {
//				发布事件
				handleResult(result);
			}
			else {
				logger.trace("No result object given - no result to handle");
			}
		}
	}
```
解析事件的参数，执行listener，发布listener方法的执行结果。
org.springframework.context.event.ApplicationListenerMethodAdapter#handleResult

```java
protected void handleResult(Object result) {
		if (result.getClass().isArray()) {
			Object[] events = ObjectUtils.toObjectArray(result);
			for (Object event : events) {
//				发布事件
				publishEvent(event);
			}
		}
		else if (result instanceof Collection<?>) {
			Collection<?> events = (Collection<?>) result;
			for (Object event : events) {
				publishEvent(event);
			}
		}
		else {
			publishEvent(result);
		}
	}
```
处理listener方法的处理结果发布事件。

org.springframework.context.ApplicationEventPublisher
封装事件发布功能的接口。用作ApplicationContext的超接口。

```java
default void publishEvent(ApplicationEvent event) {
		publishEvent((Object) event);
	}
```
发布事件模板方法，子类需要实现下面的这个方法。

```java
void publishEvent(Object event);
```
发布事件。

org.springframework.context.ApplicationEventPublisherAware
任何对象希望被其运行的ApplicationEventPublisher(通常是ApplicationContext)通知的任何对象实现的接口。

```java
void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher);
```
可以获得ApplicationEventPublisher。此方法在bean属性设置之后，初始化回调比如InitializingBean的afterPropertiesSet或自定义的init方法之前，ApplicationContextAware setApplicationContext之前调用。

org.springframework.context.event.EventPublicationInterceptor
将ApplicationEvent发布到所有应用程序侦听器的侦听器。

```java
@Override
	public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {//注入事件发布器
		this.applicationEventPublisher = applicationEventPublisher;
	}
```
重写了org.springframework.context.ApplicationEventPublisherAware#setApplicationEventPublisher方法，设置applicationEventPublisher对象。

```java
@Override
	public void afterPropertiesSet() throws Exception {
		if (this.applicationEventClassConstructor == null) {
			throw new IllegalArgumentException("Property 'applicationEventClass' is required");
		}
	}
```
重写了org.springframework.beans.factory.InitializingBean#afterPropertiesSet方法，当BeanFactory设置了所提供的所有bean属性(并满足BeanFactoryAware和applicationcontext ware)之后由它调用。
这里的处理是校验applicationEventClassConstructor参数。

```java
@Override
	public Object invoke(MethodInvocation invocation) throws Throwable {
		Object retVal = invocation.proceed();

		Assert.state(this.applicationEventClassConstructor != null, "No ApplicationEvent class set");
		ApplicationEvent event = (ApplicationEvent)
				this.applicationEventClassConstructor.newInstance(invocation.getThis());

		Assert.state(this.applicationEventPublisher != null, "No ApplicationEventPublisher available");
		this.applicationEventPublisher.publishEvent(event);

		return retVal;
	}
```
执行listener方法，创建ApplicationEvent，发布事件。

org.springframework.context.event.ApplicationEventMulticaster
接口由可以管理多个ApplicationListener对象的对象实现，并向它们发布事件。

```java
void addApplicationListener(ApplicationListener<?> listener);
```
添加一个监听器来通知所有事件。

```java
void addApplicationListenerBean(String listenerBeanName);
```
添加一个侦听器bean，以通知所有事件。

```java
void removeApplicationListener(ApplicationListener<?> listener);
```
从通知列表中删除侦听器。

```java
void removeApplicationListenerBean(String listenerBeanName);
```
从通知列表中删除一个侦听器bean。

```java
void removeAllListeners();
```
删除在这个多播器注册的所有侦听器。

```java
void multicastEvent(ApplicationEvent event);
```
将给定的应用程序事件组播到适当的侦听器。

```java
void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);
```
将给定的应用程序事件组播到适当的侦听器。

org.springframework.context.event.AbstractApplicationEventMulticaster
抽象实现ApplicationEventMulticaster接口，提供基本的侦听器注册功能。

```java
@Nullable
	private BeanFactory beanFactory;
```
BeanFactory。

```java
@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		this.beanFactory = beanFactory;
		if (beanFactory instanceof ConfigurableBeanFactory) {
			ConfigurableBeanFactory cbf = (ConfigurableBeanFactory) beanFactory;
			if (this.beanClassLoader == null) {
				this.beanClassLoader = cbf.getBeanClassLoader();
			}
//			设置单例兑现注册表
			this.retrievalMutex = cbf.getSingletonMutex();
		}
	}
```
重写了org.springframework.beans.factory.BeanFactoryAware#setBeanFactory方法，将beanFactory提供给bean实例的回调，在填充普通bean属性之后，但在初始化回调(如InitializingBean.afterPropertiesSet()或自定义init-方法)之前调用。
这里的实现是设置BeanFactory、单例对象注册表。

```java
	@Override
	public void addApplicationListener(ApplicationListener<?> listener) {
		synchronized (this.retrievalMutex) {
			// Explicitly remove target for a proxy, if registered already,
			// in order to avoid double invocations of the same listener.//显式删除代理的目标，如果已经注册，
//为了避免对同一个监听器的重复调用。
			Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
			if (singletonTarget instanceof ApplicationListener) {
				this.defaultRetriever.applicationListeners.remove(singletonTarget);
			}
			this.defaultRetriever.applicationListeners.add(listener);
			this.retrieverCache.clear();
		}
	}
```
添加applicationListener。

```java
@Override
	public void addApplicationListenerBean(String listenerBeanName) {
		synchronized (this.retrievalMutex) {
			this.defaultRetriever.applicationListenerBeans.add(listenerBeanName);
			this.retrieverCache.clear();
		}
	}
```
添加ApplicationListenerBean。

```java
@Override
	public void removeApplicationListener(ApplicationListener<?> listener) {
		synchronized (this.retrievalMutex) {
			this.defaultRetriever.applicationListeners.remove(listener);
			this.retrieverCache.clear();
		}
	}
```
删除ApplicationListener。

```java
@Override
	public void removeApplicationListenerBean(String listenerBeanName) {
		synchronized (this.retrievalMutex) {
			this.defaultRetriever.applicationListenerBeans.remove(listenerBeanName);
			this.retrieverCache.clear();
		}
	}
```
删除ApplicationListenerBean。

```java
@Override
	public void removeAllListeners() {
		synchronized (this.retrievalMutex) {
			this.defaultRetriever.applicationListeners.clear();
			this.defaultRetriever.applicationListenerBeans.clear();
			this.retrieverCache.clear();
		}
	}
```
删除所有Listener。

org.springframework.context.event.SimpleApplicationEventMulticaster
简单实现ApplicationEventMulticaster接口。向所有已注册的侦听器发送所有事件，将其留给侦听器，以忽略它们不感兴趣的事件。侦听器通常会在传入的事件对象上执行相应的instanceof检查。默认情况下，所有侦听器都在调用线程中调用。这允许恶意侦听器阻塞整个应用程序，但增加了最小开销。指定一个可选的任务执行器，让侦听器在不同的线程中执行，例如从线程池中执行。

```java
@Nullable
	private Executor taskExecutor;
```
事件执行线程池。

```java
@Override
	public void multicastEvent(ApplicationEvent event) {
		multicastEvent(event, resolveDefaultEventType(event));
	}
```
解析事件类型发布事件。

```java
	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
//		获取监听器
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
//				执行事件监听器
				executor.execute(new Runnable() {
					@Override
					public void run() {
						SimpleApplicationEventMulticaster.this.invokeListener(listener, event);
					}
				});
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```
查询listener，如果线程池executor不为空，调用线程池异步执行listener，否则就同步执行listener。

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
//				执行事件监听器
				doInvokeListener(listener, event);
			}
			catch (Throwable err) {
				errorHandler.handleError(err);
			}
		}
		else {
			doInvokeListener(listener, event);
		}
	}
```
如果设置了ErrorHandler，执行listener，如果出现异常执行ErrorHandler org.springframework.util.ErrorHandler#handleError方法，否则执行listener。

```java
	private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
		try {
			listener.onApplicationEvent(event);
		}
		catch (ClassCastException ex) {
			String msg = ex.getMessage();
			if (msg == null || matchesClassCastMessage(msg, event.getClass().getName())) {
				// Possibly a lambda-defined listener which we could not resolve the generic event type for
				// -> let's suppress the exception and just log a debug message.可能是一个lambda定义的监听器，我们无法为其解析通用事件类型
				让我们来抑制这个异常并记录一条调试消息。
				Log logger = LogFactory.getLog(getClass());
				if (logger.isDebugEnabled()) {
					logger.debug("Non-matching event type for listener: " + listener, ex);
				}
			}
			else {
				throw ex;
			}
		}
	}
```
执行org.springframework.context.ApplicationListener#onApplicationEvent listener的事件到达事件。
