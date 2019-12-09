# BeanFactory之AbstractAutowireCapableBeanFactory

AbstractAutowireCapableBeanFactory 这个BeanFactory抽象实现，提供bean创建，构造参数解析，属性设置，bean初始化，自动依赖注入，处理ReferenceBean，调用初始化方法。

org.springframework.beans.factory.config.AutowireCapableBeanFactory#resolveDependency(org.springframework.beans.factory.config.DependencyDescriptor, java.lang.String, java.util.Set<java.lang.String>, org.springframework.beans.TypeConverter)模板方法解析依赖bean，默认实现类DefaultListableBeanFactory。

```java
private final Set<Class<?>> ignoredDependencyInterfaces = new HashSet<>();
```
忽略依赖注入的接口。

```java
public AbstractAutowireCapableBeanFactory() {
		super();
		ignoreDependencyInterface(BeanNameAware.class);
		ignoreDependencyInterface(BeanFactoryAware.class);
		ignoreDependencyInterface(BeanClassLoaderAware.class);
	}
```
这三种接口不进行依赖注入。

```java
private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();
```
bean默认实例化策略是cglib。
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.Class<T>)创建指定beanClass的bean

```java
	@Override
	@SuppressWarnings("unchecked")
	public <T> T createBean(Class<T> beanClass) throws BeansException {
		// Use prototype bean definition, to avoid registering bean as dependent bean.使用原型bean定义，以避免将bean注册为从属bean。
		RootBeanDefinition bd = new RootBeanDefinition(beanClass);
		bd.setScope(SCOPE_PROTOTYPE);
		bd.allowCaching = ClassUtils.isCacheSafe(beanClass, getBeanClassLoader());
		// For the nullability warning, see the elaboration in AbstractBeanFactory.doGetBean;
		// in short: This is never going to be null unless user-declared code enforces null.//有关可空性警告，请参阅AbstractBeanFactory.doGetBean;
//简而言之:除非用户声明的代码强制为空，否则它永远不会为空。
		return (T) createBean(beanClass.getName(), bd, null);
	}
```
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])这个方法前面介绍过了。

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireBean依赖注入bean

```java
@Override
	public void autowireBean(Object existingBean) {
		// Use non-singleton bean definition, to avoid registering bean as dependent bean.使用非单例bean定义，以避免将bean注册为从属bean。
		RootBeanDefinition bd = new RootBeanDefinition(ClassUtils.getUserClass(existingBean));
		bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);
		bd.allowCaching = ClassUtils.isCacheSafe(bd.getBeanClass(), getBeanClassLoader());
		BeanWrapper bw = new BeanWrapperImpl(existingBean);
		initBeanWrapper(bw);
		populateBean(bd.getBeanClass().getName(), bd, bw);
	}
```
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean前面介绍过了。

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#configureBean配置bean

```java
	@Override
	public Object configureBean(Object existingBean, String beanName) throws BeansException {
		markBeanAsCreated(beanName);
		BeanDefinition mbd = getMergedBeanDefinition(beanName);
		RootBeanDefinition bd = null;
		if (mbd instanceof RootBeanDefinition) {
			RootBeanDefinition rbd = (RootBeanDefinition) mbd;
			bd = (rbd.isPrototype() ? rbd : rbd.cloneBeanDefinition());
		}
		if (bd == null) {
			bd = new RootBeanDefinition(mbd);
		}
		if (!bd.isPrototype()) {
			bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);
			bd.allowCaching = ClassUtils.isCacheSafe(ClassUtils.getUserClass(existingBean), getBeanClassLoader());
		}
		BeanWrapper bw = new BeanWrapperImpl(existingBean);
		initBeanWrapper(bw);
		populateBean(beanName, bd, bw);
		// For the nullability warning, see the elaboration in AbstractBeanFactory.doGetBean;
		// in short: This is never going to be null unless user-declared code enforces null.//有关可空性警告，请参阅AbstractBeanFactory.doGetBean;
//简而言之:除非用户声明的代码强制为空，否则它永远不会为空。
		return initializeBean(beanName, existingBean, bd);
	}
```
依赖注入bean后初始化bean。
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)前面介绍过了，执行aware方法、bean初始化方法。

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.Class<?>, int, boolean)创建bean

```java
@Override
	public Object createBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException {
		// Use non-singleton bean definition, to avoid registering bean as dependent bean.使用非单例bean定义，以避免将bean注册为从属bean。
		RootBeanDefinition bd = new RootBeanDefinition(beanClass, autowireMode, dependencyCheck);
		bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);
		// For the nullability warning, see the elaboration in AbstractBeanFactory.doGetBean;
		// in short: This is never going to be null unless user-declared code enforces null.//有关可空性警告，请参阅AbstractBeanFactory.doGetBean;
//简而言之:除非用户声明的代码强制为空，否则它永远不会为空。
		return createBean(beanClass.getName(), bd, null);
	}
```
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])前面介绍过了。

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowire

```java
	@Override
	public Object autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException {
		// Use non-singleton bean definition, to avoid registering bean as dependent bean.使用非单例bean定义，以避免将bean注册为从属bean。
		final RootBeanDefinition bd = new RootBeanDefinition(beanClass, autowireMode, dependencyCheck);
		bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);
		if (bd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR) {
			return autowireConstructor(beanClass.getName(), bd, null, null).getWrappedInstance();
		}
		else {
			Object bean;
			final BeanFactory parent = this;
			if (System.getSecurityManager() != null) {
				bean = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						getInstantiationStrategy().instantiate(bd, null, parent),
						getAccessControlContext());
			}
			else {
				bean = getInstantiationStrategy().instantiate(bd, null, parent);
			}
			populateBean(beanClass.getName(), bd, new BeanWrapperImpl(bean));
			return bean;
		}
	}
```
依赖注入，如果BeanDefinition配置的是按go0uzao方法依赖注入执行构造方法依赖注入，否则采用bean初始化策略初始化bean进行依赖注入。
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireConstructor构造方法注入，前面介绍过了。

org.springframework.beans.factory.support.SimpleInstantiationStrategy#instantiate(org.springframework.beans.factory.support.RootBeanDefinition, java.lang.String, org.springframework.beans.factory.BeanFactory)采用bean初始化策略初始化bean

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
如果方法没有被cglib覆盖则调用默认构造参数进行初始化bean，否则采用cglib初始化类。

org.springframework.beans.factory.support.SimpleInstantiationStrategy#instantiate(org.springframework.beans.factory.support.RootBeanDefinition, java.lang.String, org.springframework.beans.factory.BeanFactory, java.lang.Object, java.lang.reflect.Method, java.lang.Object...)采用工厂方法初始化bean

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
创建返回的bean==null创建NullBean返回。

```java
private static final ThreadLocal<Method> currentlyInvokedFactoryMethod = new ThreadLocal<>();
```
工厂方法是用threadLocal保证线程安全的。
