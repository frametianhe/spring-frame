# BeanFactory之AbstractBeanFactory①

AbstractBeanFactory BeanFactory的抽象实现。

```java
private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<>();
```
BeanPostProcessor都保存在beanPostProcessors list中。

```java
private boolean hasInstantiationAwareBeanPostProcessors;
```
是否有bean初始化类的BeanPostProcessor。

```java
private boolean hasDestructionAwareBeanPostProcessors;
```
是否有bean销毁类的BeanPostProcessor。    

org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean获取bean实例

```java
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.为手动注册的singleton检查单例缓存。
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
//				bean是否正在创建
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
//			检查bean定义是否在beanFactory
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.beanName=&beanName
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
//					递归调用
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

//			标识bean已创建
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.当前依赖的bean初始化
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
//						确定依赖的bean是否已经初始化
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
//						注册依赖的bean
						registerDependentBean(dep, beanName);
//						递归调用
						getBean(dep);
					}
				}

				// Create bean instance.创建bean对象
//				如果bean定义是单例对象
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
//							bean初始化
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
//							发生异常销毁单例的bean
							destroySingleton(beanName);
							throw ex;
						}
					});
//					从factoryBean中创建的bean对象
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
//					bean定义为原型模式
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
//						原型的bean创建之前维护内存中的缓存map映射信息
						beforePrototypeCreation(beanName);
//						原型的bean创建
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.检查所需类型是否与实际bean实例的类型匹配。
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
//				如果类型不一致进行类型转换
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean这一部分逻辑
```java
	if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
//				bean是否正在创建
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
```

先单例bean注册表中获取单例bean实例，如果这个bean是普通的单例bean直接返回，如果是factoryBean对象从factoryBean中获取bean，org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance获取指定bean实例的对象

```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.//如果bean不是工厂，不要让调用代码破坏工厂。
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
			}
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		//现在我们有了bean实例，它可能是一个普通的bean或一个FactoryBean。
//如果它是一个FactoryBean，我们使用它来创建一个bean实例，除非。
//打电话的人实际上想要一个关于工厂的资料。
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd == null) {
//			先从缓存中获取对象
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.从FactoryBean中获取的缓存对象，如果它是一个单例对象。
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
//			从factoryBean中获取对象
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```
从本地缓存中获取factoryBean对象然后从factoryBean对象中获取对象。

org.springframework.beans.factory.support.FactoryBeanRegistrySupport#getObjectFromFactoryBean 从BeanFactory中获取对象

```java
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
//		如果单例对象
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
//				先从缓存ConcurrentHashMap中获取
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
//					从factoryBean中获取
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)//在调用getObject()的时候，只有在处理和存储的时候才会用到
//(例如，由于自定义getBean调用触发的循环引用处理)
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
//						是否需要后续处理
						if (shouldPostProcess) {
							try {
//								从factoryBean获取对象后的钩子方法
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
						}
						this.factoryBeanObjectCache.put(beanName, object);
					}
				}
				return object;
			}
		}
		else {
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}
```
如果bean是单例的，单例对象注册表中包含beanName，同步单例bean注册表，先按beanName从factoryBean缓存中按factoryBean的beanName获取对象，如果获取不到就从factoryBean中创建对象，在次从factoryBean对象注册表中获取，如果获取到直接返回，如果回去不到，如果需要处理PostProcessor，获取处理postProcessor后的对象。最后把对象注册到factoryBean对象注册表中。如果不是单例的bean，直接从factoryBean中创建对象，进行postProcessor处理。

从factoryBean中创建对象org.springframework.beans.factory.support.FactoryBeanRegistrySupport#doGetObjectFromFactoryBean

```java
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
			throws BeanCreationException {

		Object object;
		try {
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
//				调用factoryBean的getObject方法获取对象
				object = factory.getObject();
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// Do not accept a null value for a FactoryBean that's not fully
		// initialized yet: Many FactoryBeans just return null then.//不要接受不完整的FactoryBean的空值
//初始化:许多factorybean只返回null。
		if (object == null) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(
						beanName, "FactoryBean which is currently in creation returned null from getObject");
			}
			object = new NullBean();
		}
		return object;
	}
```
factory.getObject();这个方法需要实现factoryBean接口的子类实现。
