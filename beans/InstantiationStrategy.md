# InstantiationStrategy

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1575269221333-ff327abb-ca9b-4cb4-a87f-35b27d9272f7.png#align=left&display=inline&height=113&name=image.png&originHeight=225&originWidth=275&size=12813&status=done&style=none&width=137.5)
这里是策略模式对的实现。
org.springframework.beans.factory.support.InstantiationStrategy bean初始化策略接口，负责创建对于RootBeanDefinition的实例。

```java
Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)
			throws BeansException;
```
返回一个给定名称的bean。

```java
Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			Constructor<?> ctor, @Nullable Object... args) throws BeansException;
```
返回一个给定名称的bean，通过给定的构造函数创建。

```java
Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			@Nullable Object factoryBean, Method factoryMethod, @Nullable Object... args)
			throws BeansException;
```
返回一个给定名称的bean，通过给定的工厂方法创建。

org.springframework.beans.factory.support.SimpleInstantiationStrategy 简单初始化策略实现，不支持方法注入。

```java
private static final ThreadLocal<Method> currentlyInvokedFactoryMethod = new ThreadLocal<>();
```
这里使用threadLocal保证正在执行的方法线程安全的。
org.springframework.beans.factory.support.SimpleInstantiationStrategy#instantiate(org.springframework.beans.factory.support.RootBeanDefinition, java.lang.String, org.springframework.beans.factory.BeanFactory)

```java
@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.如果没有重写，不要使用CGLIB覆盖该类。
		if (!bd.hasMethodOverrides()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
						}
						else {
							constructorToUse =	clazz.getDeclaredConstructor();
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
//			使用构造方法创建类
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.必须生成CGLIB子类
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}
```
如果beanDefinition没有方法覆盖就不用cglib覆盖该类，获取beanDefinition的beanClass，如果class是interface就报错BeanInstantiationException，获取构造方法进行bean初始化。否则生成cglib子类进行初始化。

```java
protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		throw new UnsupportedOperationException("Method Injection not supported in SimpleInstantiationStrategy");
	}
```
cglib初始化类，这里不支持。
org.springframework.beans.factory.support.SimpleInstantiationStrategy#instantiate(org.springframework.beans.factory.support.RootBeanDefinition, java.lang.String, org.springframework.beans.factory.BeanFactory, java.lang.reflect.Constructor<?>, java.lang.Object...)

```java
@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			final Constructor<?> ctor, @Nullable Object... args) {

		if (!bd.hasMethodOverrides()) {
			if (System.getSecurityManager() != null) {
				// use own privileged to change accessibility (when security is on)使用自己的特权来更改可访问性(当打开安全性时)
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					ReflectionUtils.makeAccessible(ctor);
					return null;
				});
			}
//			调用公共构造参数创建对象
			return (args != null ? BeanUtils.instantiateClass(ctor, args) : BeanUtils.instantiateClass(ctor));
		}
		else {
			return instantiateWithMethodInjection(bd, beanName, owner, ctor, args);
		}
	}
```
如果beanDefinition没有方法覆盖，调用构造方法初始化bean，否则cglib初始化类。
org.springframework.beans.factory.support.SimpleInstantiationStrategy#instantiate(org.springframework.beans.factory.support.RootBeanDefinition, java.lang.String, org.springframework.beans.factory.BeanFactory, java.lang.Object, java.lang.reflect.Method, java.lang.Object...)

```java
@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			@Nullable Object factoryBean, final Method factoryMethod, @Nullable Object... args) {

		try {
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged(new PrivilegedAction<Object>() {
					@Override
					public Object run() {
						ReflectionUtils.makeAccessible(factoryMethod);
						return null;
					}
				});
			}
			else {
				ReflectionUtils.makeAccessible(factoryMethod);
			}

//			这里当前执行的工厂方法是用threadLocal保证线程安全的
			Method priorInvokedFactoryMethod = currentlyInvokedFactoryMethod.get();
			try {
				currentlyInvokedFactoryMethod.set(factoryMethod);
//				执行工厂方法
				Object result = factoryMethod.invoke(factoryBean, args);
				if (result == null) {
					result = new NullBean();
				}
				return result;
			}
			finally {
				if (priorInvokedFactoryMethod != null) {
					currentlyInvokedFactoryMethod.set(priorInvokedFactoryMethod);
				}
				else {
//					最后从threadLocal中删除使用的工厂方法以免内存泄漏
					currentlyInvokedFactoryMethod.remove();
				}
			}
		}
		catch (IllegalArgumentException ex) {
			throw new BeanInstantiationException(factoryMethod,
					"Illegal arguments to factory method '" + factoryMethod.getName() + "'; " +
					"args: " + StringUtils.arrayToCommaDelimitedString(args), ex);
		}
		catch (IllegalAccessException ex) {
			throw new BeanInstantiationException(factoryMethod,
					"Cannot access factory method '" + factoryMethod.getName() + "'; is it public?", ex);
		}
		catch (InvocationTargetException ex) {
			String msg = "Factory method '" + factoryMethod.getName() + "' threw exception";
			if (bd.getFactoryBeanName() != null && owner instanceof ConfigurableBeanFactory &&
					((ConfigurableBeanFactory) owner).isCurrentlyInCreation(bd.getFactoryBeanName())) {
				msg = "Circular reference involving containing bean '" + bd.getFactoryBeanName() + "' - consider " +
						"declaring the factory method as static for independence from its containing instance. " + msg;
			}
			throw new BeanInstantiationException(factoryMethod, msg, ex.getTargetException());
		}
	}
```
按指定的beanName采用工厂方法初始化 ，从threadLocal中获取当前线程使用的方法，反射调用工厂方法进行初始化，如果初始化的对象是null返回NullBean的对象。最后删除threadLocal中的方法，不做此操作可能会造成内存泄漏。

org.springframework.beans.factory.support.CglibSubclassingInstantiationStrategy cglib子类初始化，BeanFactory中使用的默认初始化策略，如果方法需要被容器覆盖以实现方法注入使用cglib动态创建子类。

```java
@Override
	protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		return instantiateWithMethodInjection(bd, beanName, owner, null);
	}
```
org.springframework.beans.factory.support.CglibSubclassingInstantiationStrategy#instantiateWithMethodInjection(org.springframework.beans.factory.support.RootBeanDefinition, java.lang.String, org.springframework.beans.factory.BeanFactory, java.lang.reflect.Constructor<?>, java.lang.Object...)
```java
@Override
	protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			@Nullable Constructor<?> ctor, @Nullable Object... args) {

		// Must generate CGLIB subclass...
		return new CglibSubclassCreator(bd, owner).instantiate(ctor, args);
	}
```
org.springframework.beans.factory.support.CglibSubclassingInstantiationStrategy.CglibSubclassCreator#instantiate

```java
public Object instantiate(@Nullable Constructor<?> ctor, @Nullable Object... args) {
			Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
			Object instance;
			if (ctor == null) {
				instance = BeanUtils.instantiateClass(subclass);
			}
			else {
				try {
					Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
					instance = enhancedSubclassConstructor.newInstance(args);
				}
				catch (Exception ex) {
					throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
							"Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
				}
			}
			// SPR-10785: set callbacks directly on the instance instead of in the
			// enhanced class (via the Enhancer) in order to avoid memory leaks.
			Factory factory = (Factory) instance;
			factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
					new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
					new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
			return instance;
		}
```
cglib创建子类，如果没有指定构造方法采用默认构造方法创建子类，否则按餐数类型获取子类的构造方法，调用子类的构造方法创建对象，在创建的实例设置回调。
