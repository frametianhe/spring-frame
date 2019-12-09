# BeanRegistry

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1575003099251-0005ff55-9494-4a1f-ae76-3327f7993ce3.png#align=left&display=inline&height=135&name=image.png&originHeight=269&originWidth=418&size=18131&status=done&style=none&width=209)

SingletonBeanRegistry顶层接口，单例bean注册器接口，registerSingleton方法，注册单例bean，不会调用InitializingBean的afterPropertiesSet方法。getSingleton获取单例对象，containsSingleton是否包含单例对象，getSingletonNames获取单例对象的名称集合。

AliasRegistry接口，提供registerAlias注册别名、removeAlias删除别名、getAliases获取别名等方法。

DefaultSingletonBeanRegistry默认实现

```java
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```
单例对象存储在singletonObjects。

```java
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```
立即初始化的单例object保存在earlySingletonObjects。

```java
private final Set<String> registeredSingletons = new LinkedHashSet<>(256);
```
注册单例bean保存在registeredSingletons。

```java
private final Map<String, Object> disposableBeans = new LinkedHashMap<>();
```
实现销毁接口的bean。

```java
private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);
```
有依赖关系的bean。
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#registerSingleton注册单例bean

```java
@Override
	public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
		Assert.notNull(beanName, "Bean name must not be null");
		Assert.notNull(singletonObject, "Singleton object must not be null");
		synchronized (this.singletonObjects) {
			Object oldObject = this.singletonObjects.get(beanName);
			if (oldObject != null) {
				throw new IllegalStateException("Could not register object [" + singletonObject +
						"] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
			}
			addSingleton(beanName, singletonObject);
		}
	}
```
这里注册是同步注册。


org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingleton添加单例bean到注册表中

```java
	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```


org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingletonFactory添加单例factory用来创建单例bean

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
```


org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean) 获取单例bean

```java
@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```
先从singletonObjects中获取创建好的单例bean，如果没有获取到从earlySingletonObjects中获取单例bean，从singletonFactories中获取ObjectFactory创建对象后保存在earlySingletonObjects中并删除singletonFactories中的ObjectFactory。这里是用三个map解决spring初始化循环依赖的。

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)

```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
//		同步单例对象注册表
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.//同时隐式地出现了单例对象——>
//如果是，则继续执行，因为异常指示了该状态。
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```
获取指定beanName的单例bean，如果获取不到就创建并注册到单例bean注册表中。

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#removeSingleton从单例bean的注册表中删除单例bean

```java
protected void removeSingleton(String beanName) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.remove(beanName);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.remove(beanName);
		}
	}
```

org.springframework.beans.factory.DisposableBean接口，实现destroy方法，在beanFactory销毁bean的时候回调用这个方法，注册DisposableBean，org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#registerDisposableBean

```java
public void registerDisposableBean(String beanName, DisposableBean bean) {
		synchronized (this.disposableBeans) {
			this.disposableBeans.put(beanName, bean);
		}
	}
```
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#registerContainedBean，注册两个bean的包含关系，例如一个是内部bean，一个是外部bean，根据销毁顺序将包含的bean注册为依赖于包含的bean

```java
public void registerContainedBean(String containedBeanName, String containingBeanName) {
		// A quick check for an existing entry upfront, avoiding synchronization...预先快速检查现有条目，避免同步……
		Set<String> containedBeans = this.containedBeanMap.get(containingBeanName);
		if (containedBeans != null && containedBeans.contains(containedBeanName)) {
			return;
		}

		// No entry yet -> fully synchronized manipulation of the containedBeans Set还没有条目——>对containedbean集的完全同步操作
		synchronized (this.containedBeanMap) {
			containedBeans = this.containedBeanMap.get(containingBeanName);
			if (containedBeans == null) {
				containedBeans = new LinkedHashSet<>(8);
				this.containedBeanMap.put(containingBeanName, containedBeans);
			}
			containedBeans.add(containedBeanName);
		}
		registerDependentBean(containedBeanName, containingBeanName);
	}
```
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#registerDependentBean 注册有关系依赖的bean

```java
	public void registerDependentBean(String beanName, String dependentBeanName) {
		// A quick check for an existing entry upfront, avoiding synchronization...预先快速检查现有条目，避免同步……
		String canonicalName = canonicalName(beanName);
//		按别名查找依赖的beanName集合
		Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
		if (dependentBeans != null && dependentBeans.contains(dependentBeanName)) {
			return;
		}

		// No entry yet -> fully synchronized manipulation of the dependentBeans Set还没有条目——对dependentbean集的>完全同步的操作
		synchronized (this.dependentBeanMap) {
			dependentBeans = this.dependentBeanMap.get(canonicalName);
			if (dependentBeans == null) {
				dependentBeans = new LinkedHashSet<>(8);
				this.dependentBeanMap.put(canonicalName, dependentBeans);
			}
			dependentBeans.add(dependentBeanName);
		}
		synchronized (this.dependenciesForBeanMap) {
			Set<String> dependenciesForBean = this.dependenciesForBeanMap.get(dependentBeanName);
			if (dependenciesForBean == null) {
				dependenciesForBean = new LinkedHashSet<>(8);
				this.dependenciesForBeanMap.put(dependentBeanName, dependenciesForBean);
			}
			dependenciesForBean.add(canonicalName);
		}
	}
```
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroySingletons 销毁单例bean

```java
	public void destroySingletons() {
		if (logger.isDebugEnabled()) {
			logger.debug("Destroying singletons in " + this);
		}
//		这里使用ConcurrentHashMap本地缓存单例的bean实例，访问次数比较多，提搞并发量
		synchronized (this.singletonObjects) {
			this.singletonsCurrentlyInDestruction = true;
		}

		String[] disposableBeanNames;
//		这里是用LinkedHashMap本地缓存销毁的bean实例
		synchronized (this.disposableBeans) {
			disposableBeanNames = StringUtils.toStringArray(this.disposableBeans.keySet());
		}
		for (int i = disposableBeanNames.length - 1; i >= 0; i--) {
//			销毁单例的bean
			destroySingleton(disposableBeanNames[i]);
		}

		this.containedBeanMap.clear();
		this.dependentBeanMap.clear();
		this.dependenciesForBeanMap.clear();

//		同步清空缓存
		synchronized (this.singletonObjects) {
			this.singletonObjects.clear();
			this.singletonFactories.clear();
			this.earlySingletonObjects.clear();
			this.registeredSingletons.clear();
			this.singletonsCurrentlyInDestruction = false;
		}
	}
```
销毁单例的bean，单例bean注册表、单例工厂注册表、早期创建的单例bean注册表，注册的单例bean注册表清空，有包含关系的bean注册表、有依赖关系的bean注册表清空。

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroySingleton销毁单例的bean

```java
public void destroySingleton(String beanName) {
		// Remove a registered singleton of the given name, if any.删除单例的bean，从本地缓存中删除
		removeSingleton(beanName);

		// Destroy the corresponding DisposableBean instance.
		DisposableBean disposableBean;
		synchronized (this.disposableBeans) {
//			从本地缓存中删除
			disposableBean = (DisposableBean) this.disposableBeans.remove(beanName);
		}
//		bean销毁的逻辑
		destroyBean(beanName, disposableBean);
	}
```
删除单例bean注册表中的bean，删除disposableBeans注册表中的对象。

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroyBean

```java
	protected void destroyBean(String beanName, @Nullable DisposableBean bean) {
		// Trigger destruction of dependent beans first... 先触发依赖的bean销毁，从本地缓存中删除
		Set<String> dependencies = this.dependentBeanMap.remove(beanName);
		if (dependencies != null) {
			if (logger.isDebugEnabled()) {
				logger.debug("Retrieved dependent beans for bean '" + beanName + "': " + dependencies);
			}
			for (String dependentBeanName : dependencies) {
//				这里用了一个递归删除单例bean，当这个bean没有依赖的bean要删除的时候，递归结束
				destroySingleton(dependentBeanName);
			}
		}

		// Actually destroy the bean now... 这里开始删除单例bean
		if (bean != null) {
			try {
//				bean可以实现DisposableBean这个接口，重写父类的bean destory的方法
				bean.destroy();
			}
			catch (Throwable ex) {
				logger.error("Destroy method on bean with name '" + beanName + "' threw an exception", ex);
			}
		}

		// Trigger destruction of contained beans...从本地缓存中销毁内部bean
		Set<String> containedBeans = this.containedBeanMap.remove(beanName);
		if (containedBeans != null) {
			for (String containedBeanName : containedBeans) {
//				这个地方还是递归调用，删除单例bean，当这个bean没有内部bean时递归结束
				destroySingleton(containedBeanName);
			}
		}

		// Remove destroyed bean from other beans' dependencies. 从其他bean依赖中删除销毁的bean
		synchronized (this.dependentBeanMap) {
			for (Iterator<Map.Entry<String, Set<String>>> it = this.dependentBeanMap.entrySet().iterator(); it.hasNext();) {
				Map.Entry<String, Set<String>> entry = it.next();
				Set<String> dependenciesToClean = entry.getValue();
				dependenciesToClean.remove(beanName);
				if (dependenciesToClean.isEmpty()) {
					it.remove();
				}
			}
		}

		// Remove destroyed bean's prepared dependency information.删除销毁的bean准备的依赖信息
		this.dependenciesForBeanMap.remove(beanName);
	}
```
删除依赖bean注册表dependentBeanMap中的bean，递归处理上面的逻辑，调用实现DisposableBean接口的destory方法，删除包含的bean注册表containedBeanMap中的bean，递归处理上面的逻辑，删除依赖bean注册表中的bean。

FactoryBeanRegistrySupport支持单例bean注册的基类。

```java
private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<>(16);
```
factoryBeanObjectCache保存factoryBean对象的缓存。

org.springframework.beans.factory.support.FactoryBeanRegistrySupport#getTypeForFactoryBean 获取factoryBean的类型

```java
@Nullable
	protected Class<?> getTypeForFactoryBean(final FactoryBean<?> factoryBean) {
		try {
			if (System.getSecurityManager() != null) {
				return AccessController.doPrivileged((PrivilegedAction<Class<?>>)
						factoryBean::getObjectType, getAccessControlContext());
			}
			else {
				return factoryBean.getObjectType();
			}
		}
		catch (Throwable ex) {
			// Thrown from the FactoryBean's getObjectType implementation.
			logger.warn("FactoryBean threw exception from getObjectType, despite the contract saying " +
					"that it should return null if the type of its object cannot be determined yet", ex);
			return null;
		}
	}
```
org.springframework.beans.factory.FactoryBean#getObjectType 这个方法是模板方法实现。

org.springframework.beans.factory.support.FactoryBeanRegistrySupport#getObjectFromFactoryBean从factoryBean中获取对象

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
//			从factoryBean创建对象
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
如果bean是单例的，给定的beanName在单例bean注册表中，同步单例bean注册表操作，从factoryBean对象缓存中获取factoryBean对象，如果没有获取到就调用factoryBean的getObject方法获取对象，再次判断factoryBeanObjectCache缓存中是否有需要的单例bean，如果有直接返回，如果没有，需要处理后置处理，执行postProcessObjectFromFactoryBean钩子方法，子类可以覆盖，把后置处理后的单例bean存储在factoryBeanObjectCache注册表中。

如果bean不是单例的并且不在单例bean注册表中，调用factoryBean的getObject方法获取bean，需要需要猴孩子处理就执行postProcessObjectFromFactoryBean后置处理方法，最后返回单例bean。

org.springframework.beans.factory.support.FactoryBeanRegistrySupport#doGetObjectFromFactoryBean从factoryBean获取对象就是调用factoryBean的getObject方法

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
org.springframework.beans.factory.support.FactoryBeanRegistrySupport#postProcessObjectFromFactoryBean 从factoryBean中获取bean实例的后置处理方法，子类可以覆盖，这个默认什么也不做

```java
protected Object postProcessObjectFromFactoryBean(Object object, String beanName) throws BeansException {
		return object;
	}
```
org.springframework.beans.factory.support.FactoryBeanRegistrySupport#removeSingleton 删除指定beanName的单例bean

```java
@Override
	protected void removeSingleton(String beanName) {
		super.removeSingleton(beanName);
		this.factoryBeanObjectCache.remove(beanName);
	}
```
删除单例bean注册表和factoryBean对象注册表中指定的beanName的bean。

org.springframework.beans.factory.support.FactoryBeanRegistrySupport#destroySingletons 销毁单例bean

```java
@Override
	public void destroySingletons() {
		super.destroySingletons();
//		清空beanFactory缓存
		this.factoryBeanObjectCache.clear();
	}
```
销毁单例bean，清空factoryBean对象注册表。

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroySingletons销毁单例bean

```java
	public void destroySingletons() {
		if (logger.isDebugEnabled()) {
			logger.debug("Destroying singletons in " + this);
		}
//		这里使用ConcurrentHashMap本地缓存单例的bean实例，访问次数比较多，提搞并发量
		synchronized (this.singletonObjects) {
			this.singletonsCurrentlyInDestruction = true;
		}

		String[] disposableBeanNames;
//		这里是用LinkedHashMap本地缓存销毁的bean实例
		synchronized (this.disposableBeans) {
			disposableBeanNames = StringUtils.toStringArray(this.disposableBeans.keySet());
		}
		for (int i = disposableBeanNames.length - 1; i >= 0; i--) {
//			销毁单例的bean
			destroySingleton(disposableBeanNames[i]);
		}

		this.containedBeanMap.clear();
		this.dependentBeanMap.clear();
		this.dependenciesForBeanMap.clear();

//		同步清空缓存
		synchronized (this.singletonObjects) {
			this.singletonObjects.clear();
			this.singletonFactories.clear();
			this.earlySingletonObjects.clear();
			this.registeredSingletons.clear();
			this.singletonsCurrentlyInDestruction = false;
		}
	}
```
销毁disposableBeans注册表的单例bean。清空containedBeanMap 包含bean的注册表，dependentBeanMap依赖bean的注册表，dependenciesForBeanMap依赖项的注册表，单例bean注册表。

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroySingleton 销毁单例bean

```java
public void destroySingleton(String beanName) {
		// Remove a registered singleton of the given name, if any.删除单例的bean，从本地缓存中删除
		removeSingleton(beanName);

		// Destroy the corresponding DisposableBean instance.
		DisposableBean disposableBean;
		synchronized (this.disposableBeans) {
//			从本地缓存中删除
			disposableBean = (DisposableBean) this.disposableBeans.remove(beanName);
		}
//		bean销毁的逻辑
		destroyBean(beanName, disposableBean);
	}
```
删除单例bean注册表中的bean，删除disposableBeans中的bean，销毁单例bean，org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroyBean

```java
	protected void destroyBean(String beanName, @Nullable DisposableBean bean) {
		// Trigger destruction of dependent beans first... 先触发依赖的bean销毁，从本地缓存中删除
		Set<String> dependencies = this.dependentBeanMap.remove(beanName);
		if (dependencies != null) {
			if (logger.isDebugEnabled()) {
				logger.debug("Retrieved dependent beans for bean '" + beanName + "': " + dependencies);
			}
			for (String dependentBeanName : dependencies) {
//				这里用了一个递归删除单例bean，当这个bean没有依赖的bean要删除的时候，递归结束
				destroySingleton(dependentBeanName);
			}
		}

		// Actually destroy the bean now... 这里开始删除单例bean
		if (bean != null) {
			try {
//				bean可以实现DisposableBean这个接口，重写父类的bean destory的方法
				bean.destroy();
			}
			catch (Throwable ex) {
				logger.error("Destroy method on bean with name '" + beanName + "' threw an exception", ex);
			}
		}

		// Trigger destruction of contained beans...从本地缓存中销毁内部bean
		Set<String> containedBeans = this.containedBeanMap.remove(beanName);
		if (containedBeans != null) {
			for (String containedBeanName : containedBeans) {
//				这个地方还是递归调用，删除单例bean，当这个bean没有内部bean时递归结束
				destroySingleton(containedBeanName);
			}
		}

		// Remove destroyed bean from other beans' dependencies. 从其他bean依赖中删除销毁的bean
		synchronized (this.dependentBeanMap) {
			for (Iterator<Map.Entry<String, Set<String>>> it = this.dependentBeanMap.entrySet().iterator(); it.hasNext();) {
				Map.Entry<String, Set<String>> entry = it.next();
				Set<String> dependenciesToClean = entry.getValue();
				dependenciesToClean.remove(beanName);
				if (dependenciesToClean.isEmpty()) {
					it.remove();
				}
			}
		}

		// Remove destroyed bean's prepared dependency information.删除销毁的bean准备的依赖信息
		this.dependenciesForBeanMap.remove(beanName);
	}
```
删除dependentBeanMap依赖bean注册表中的bean，递归删除单例bean依赖的bean，调用bean的destroy方法执行bean的销毁逻辑，删除containedBeanMap包含bean注册表的单例bean，递归删除包含bean注册表中bean依赖的bean，删除dependentBeanMap依赖bean注册表中的bean，删除dependenciesForBeanMap依赖项注册表中的bean。
