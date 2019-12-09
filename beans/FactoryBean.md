# FactoryBean

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1575204877817-16ffb548-2432-4ef5-adc5-038013bbae85.png#align=left&display=inline&height=170&name=image.png&originHeight=340&originWidth=1201&size=46180&status=done&style=none&width=600.5)

这里开始介绍factoryBean和bean初始化过程中的前置、后置处理方法。

Aware标记超接口，标识bean可以介绍spring ioc容器的回调，具体实现是这个接口的子接口实现。

org.springframework.beans.factory.DisposableBean bean可以实现这个接口，在bean销毁之前会调用这个类的destory方法。

org.springframework.beans.factory.BeanFactoryAware

```java
void setBeanFactory(BeanFactory beanFactory) throws BeansException;
```

这个方法可以对BeanFactory进行处理，在bean属性设置之后，在InitializingBean.afterPropertiesSet或自定义init方法之前调用。

org.springframework.beans.factory.InitializingBean bean可以实现这个接口，执行自定义bean的初始化或者自定义bean的init方法。

```java
void afterPropertiesSet() throws Exception;
```

在BeanFactory设置了提供的所有bean属性，执行BeanFactoryAware之后执行这个方法。实现这个接口一般在spring ioc上下文初始化完毕初始化自己的实现。

org.springframework.beans.factory.BeanNameAware

```java
void setBeanName(String name);
```

在填充普通bean属性之后调用，但在InitializingBean.afterPropertiesSet或自定义init方法之前调用。

org.springframework.beans.factory.FactoryBean 如果bean实现了这个接口，这个bean将作为具体bean实例的factoryBean，通过这个bean的getObject方法创建对象。

```java
@Nullable
	T getObject() throws Exception;
```

返回这个factory管理的bean实例。

```java
@Nullable
	Class<?> getObjectType();
```

返回factoryBean创建的对象类型。

```java
default boolean isSingleton() {
		return true;
	}
```

返回这个factory管理的对象是不是单例的，默认返回true，factoryBean一般管理一个单例实例。

org.springframework.beans.factory.SmartFactoryBean，factoryBean的扩展接口，factoryBean的isSingleton方法返回false bean也不一定prototype类型的，这个接口可以判断bean是不是单例的，这个接口主要在spring 内部使用。

```java
default boolean isPrototype() {
		return false;
	}
```

判断factory管理的bean是不是prototype的，默认返回false。

```java
default boolean isEagerInit() {
		return false;
	}
```

这个factoryBean是否期望立即初始化，一般factoryBean不会立即初始化，getObject方法只在实际访问时调用。如果返回true，立即调用getObject方法，默认返回false。

org.springframework.beans.factory.SmartInitializingSingleton，在单例bean实例化结束时调用这个类的后置处理方法，在单例的bean实例化后执行一些初始化逻辑，这个场景和InitializingBean的afterPropertiesSet方法类似。

```java
void afterSingletonsInstantiated();
```

在单例bean初始化之后调用并保证已创建了所有的单例bean。

org.springframework.beans.factory.support.DisposableBeanAdapter DisposableBean的适配器，这里用装饰器模式实现

```java
@SuppressWarnings("serial")
class DisposableBeanAdapter implements DisposableBean, Runnable, Serializable {

	private static final String CLOSE_METHOD_NAME = "close";

	private static final String SHUTDOWN_METHOD_NAME = "shutdown";

	private static final Log logger = LogFactory.getLog(DisposableBeanAdapter.class);

//	销毁bean
	private final Object bean;
```

```java
private final boolean invokeDisposableBean;
```
判断bean是否实现了DisposableBean接口。

```java
private final boolean nonPublicAccessAllowed;
```
是否允许非public的方法。

```java
@Nullable
	private List<DestructionAwareBeanPostProcessor> beanPostProcessors;
```
DestructionAwareBeanPostProcessor bean销毁后置处理器保存在这里。

```java
@Override
	public void run() {
//		这里是异步执行销毁方法
		destroy();
	}
```
这个类实现了runnable接口，在run方法中执行destory方法。

org.springframework.beans.factory.support.DisposableBeanAdapter#destroy

```java
	@Override
	public void destroy() {
//		执行beanPostProcessors，beanPostProcessors用对对bean的过程进行处理的抽象
		if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
			for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
//				在bean销毁之前进行一些处理
				processor.postProcessBeforeDestruction(this.bean, this.beanName);
			}
		}

//		如果bean实现了DisposableBean接口
		if (this.invokeDisposableBean) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking destroy() on bean with name '" + this.beanName + "'");
			}
			try {
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						@Override
						public Object run() throws Exception {
//							调用DisposableBean的sestory方法在bean销毁之前可以做一些处理
							((DisposableBean) bean).destroy();
							return null;
						}
					}, acc);
				}
				else {
//					bean实现DisposableBean接口的方式，注解调用子类destroy方法
					((DisposableBean) bean).destroy();
				}
			}
			catch (Throwable ex) {
				String msg = "Invocation of destroy method failed on bean with name '" + this.beanName + "'";
				if (logger.isDebugEnabled()) {
					logger.warn(msg, ex);
				}
				else {
					logger.warn(msg + ": " + ex);
				}
			}
		}

		if (this.destroyMethod != null) {
//			执行bean定义中指定的bean销毁方法
			invokeCustomDestroyMethod(this.destroyMethod);
		}
		else if (this.destroyMethodName != null) {
			Method methodToCall = determineDestroyMethod(this.destroyMethodName);
			if (methodToCall != null) {
				invokeCustomDestroyMethod(methodToCall);
			}
		}
	}
```
循环beanPostProcessors列表处理org.springframework.beans.factory.config.DestructionAwareBeanPostProcessor#postProcessBeforeDestruction方法，这个方法在前面介绍beanPostProcessor的时候介绍过了。可有在这个bean销毁之前做点什么。
如果bean实现了DisposableBean接口执行org.springframework.beans.factory.DisposableBean#destroy方法。

org.springframework.beans.factory.config.AbstractFactoryBean，这个是factoryBean的模板方法模式实现的超类，getObject方法返回创建的对象。子类需要实现org.springframework.beans.factory.config.AbstractFactoryBean#createInstance这个模板方法。

```java
@Override
	public void afterPropertiesSet() throws Exception {
		if (isSingleton()) {
			this.initialized = true;
			this.singletonInstance = createInstance();
			this.earlySingletonInstance = null;
		}
	}
```
这个方法是重写org.springframework.beans.factory.InitializingBean接口的方法，这个方法在BeanFactory设置了所有的bean属性之后执行，一般重写这个方法会初始化一些自己的实现。如果是单例模式就调用子类的createInstance方法创建对象。

```java
@Override
	public final T getObject() throws Exception {
		if (isSingleton()) {
			return (this.initialized ? this.singletonInstance : getEarlySingletonInstance());
		}
		else {
//			创建非单例对象
			return createInstance();
		}
	}
```
放回bean实例，如果是单例模式，如果已经创建了对象直接返回创建的单例对象，否则直接返回早期的单例对象。否则调用子类的createInstance方法创建对象。

org.springframework.beans.factory.config.AbstractFactoryBean#getEarlySingletonInstance

```java
private T getEarlySingletonInstance() throws Exception {
		Class<?>[] ifcs = getEarlySingletonInterfaces();
		if (ifcs == null) {
			throw new FactoryBeanNotInitializedException(
					getClass().getName() + " does not support circular references");
		}
		if (this.earlySingletonInstance == null) {
//			jdk动态代理创建对象
			this.earlySingletonInstance = (T) Proxy.newProxyInstance(
					this.beanClassLoader, ifcs, new EarlySingletonInvocationHandler());
		}
		return this.earlySingletonInstance;
	}
```
获取一个早期的单例对象，这个对象为了解决循环引用，这里采用的jdk动态代理设计模式。

```java
@Override
	public void destroy() throws Exception {
		if (isSingleton()) {
			destroyInstance(this.singletonInstance);
		}
	}
```
bean销毁方法，重写了DisposableBean的sestory方法，如果是单例的销毁单例对象，org.springframework.beans.factory.config.AbstractFactoryBean#destroyInstance

```java
protected void destroyInstance(@Nullable T instance) throws Exception {
	}
```
子类可以覆盖这个方法，默认空实现。

```java
@Override
	@Nullable
	public abstract Class<?> getObjectType();
```
子类也需要实现这个模板方法，返回bean的类型。

```java
protected abstract T createInstance() throws Exception;
```
模板方法实现，子类需要实现这个方法。

```java
protected void destroyInstance(@Nullable T instance) throws Exception {
	}
```
销毁单例对象的回调方法，子类可以覆盖这个方法对这个单例对象进行处理。
