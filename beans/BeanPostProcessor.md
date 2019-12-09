# BeanPostProcessor

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1575127408457-d398b033-571d-4768-b72e-754b00bd0985.png#align=left&display=inline&height=245&name=image.png&originHeight=489&originWidth=901&size=48691&status=done&style=none&width=450.5)

beanPostProcessor的目的是为了对beanDefinition、bean初始化过程进行干预处理。

BeanPostProcessor工厂钩子接口

```java
@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
```

在bean初始化之前调用返回处理后的bean，类似于InitializingBean接口的afterPropertiesSet方法或者bean的init方法。

```java
@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
```

在bean初始化之后调用返回处理后的bean，如InitializingBean的afterPropertiesSet或自定义init-method执行之后。

DestructionAwareBeanPostProcessor 下级子接口，bean销毁前的处理器。

```java
void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;
```

在bean销毁之前执行，可以执行bean的销毁方法，如DisposableBean的destroy方法。这个方法仅限于单例bean。

```java
default boolean requiresDestruction(Object bean) {
		return true;
	}
```

确定给定的bean实例是否需要此后置处理器，默认true。

org.springframework.beans.factory.support.MergedBeanDefinitionPostProcessor用于在运行时合并BeanDefinition的后处理器回调接口，在bean初始化之前可以添加一些缓存的元数据，也可以修改BeanDefinition，只允许修改定义属性。

```java
void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
```

对指定bean的给定合并bean定义进行后处理。

org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor bean实例化之前、实例化后的处理器接口，在属性设置和自动注入之前。一般用于禁止特定bean的默认实例化，这个接口一般在spring框架内部使用，开发中建议使用InstantiationAwareBeanPostProcessorAdapter。

```java
@Nullable
	default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		return null;
	}
```

在bean初始化之前执行这个方法，返回的bean对象可以是一个代替目标bean使用的代理，有效地抑制了目标bean的默认实例化。这个方法不会应用于有factoryMethod的bean。

```java
default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		return true;
	}
```

通过构造方法或工厂方法初始化bean之后，在属性设置或自动依赖注入之前执行，可以对bean实例执行自定义字段注入的回调处理，默认返回true。

```java
	@Nullable
	default PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

		return pvs;
	}
```

在BeanFactory将给定的属性值依赖注入到指定的bean之前后置处理。

org.springframework.beans.factory.config.SmartInstantiationAwareBeanPostProcessor检查最后处理bean类型的回调，这个接口一般在spring框架内部使用，开发中一般使用InstantiationAwareBeanPostProcessorAdapter。

```java
@Nullable
	default Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException {
		return null;
	}
```

返回bean的类型，默认返回null。

```java
@Nullable
	default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName)
			throws BeansException {

		return null;
	}
```

指定这个bean的候选构造器。

```java
default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
		return bean;
	}
```

获取指定bean的早期引用，一般用于解决循环依赖，默认按原样返回指定的bean。

org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessorAdapter抽象类是org.springframework.beans.factory.config.BeanPostProcessor、org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor、org.springframework.beans.factory.config.SmartInstantiationAwareBeanPostProcessor接口的默认适配实现，开发中一般不直接使用这三个接口，继承这个抽象类。

org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor调用带init和destory方法的beanPostProcessor实现，org.springframework.beans.factory.InitializingBean 、org.springframework.beans.factory.DisposableBean，nit和destroy注释可以应用于任何可见性的方法:public、package-protected、protected或private。可以注释多个这样的方法，但是建议只分别注释一个init方法和一个destroy方法。@javax.annotation.PostConstruct、@javax.annotation.PreDestroy 带有这两个注解的初始化、销毁方法。

```java
@Nullable
	private Class<? extends Annotation> initAnnotationType;

	@Nullable
	private Class<? extends Annotation> destroyAnnotationType;
```
保存init注解的类型和destory注解的类型。

```java
@Nullable
	private final transient Map<Class<?>, LifecycleMetadata> lifecycleMetadataCache = new ConcurrentHashMap<>(256);
```
保存init和destory方法的元数据。

org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#postProcessMergedBeanDefinition

```java
@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		LifecycleMetadata metadata = findLifecycleMetadata(beanType);
		metadata.checkConfigMembers(beanDefinition);
	}
```
org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#findLifecycleMetadata

```java
	private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {
		if (this.lifecycleMetadataCache == null) {
			// Happens after deserialization, during destruction...在反物质化之后，在销毁过程中……
			return buildLifecycleMetadata(clazz);
		}
		// Quick check on the concurrent map first, with minimal locking.首先快速检查并发映射，并使用最少的锁。
		LifecycleMetadata metadata = this.lifecycleMetadataCache.get(clazz);
		if (metadata == null) {
			synchronized (this.lifecycleMetadataCache) {
				metadata = this.lifecycleMetadataCache.get(clazz);
				if (metadata == null) {
					metadata = buildLifecycleMetadata(clazz);
					this.lifecycleMetadataCache.put(clazz, metadata);
				}
				return metadata;
			}
		}
		return metadata;
	}
```
org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#buildLifecycleMetadata

```java
	private LifecycleMetadata buildLifecycleMetadata(final Class<?> clazz) {
		final boolean debug = logger.isDebugEnabled();
		LinkedList<LifecycleElement> initMethods = new LinkedList<>();
		LinkedList<LifecycleElement> destroyMethods = new LinkedList<>();
		Class<?> targetClass = clazz;

		do {
			final LinkedList<LifecycleElement> currInitMethods = new LinkedList<>();
			final LinkedList<LifecycleElement> currDestroyMethods = new LinkedList<>();

			ReflectionUtils.doWithLocalMethods(targetClass, new ReflectionUtils.MethodCallback() {
				@Override
				public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {
//				 判断方法上是否有@PostConstruct这个注解
					if (initAnnotationType != null && method.isAnnotationPresent(initAnnotationType)) {
						LifecycleElement element = new LifecycleElement(method);
						currInitMethods.add(element);
						if (debug) {
							logger.debug("Found init method on class [" + clazz.getName() + "]: " + method);
						}
					}
//				@PreDestroy 判断方法上是否有这个注解
					if (destroyAnnotationType != null && method.isAnnotationPresent(destroyAnnotationType)) {
						currDestroyMethods.add(new LifecycleElement(method));
						if (debug) {
							logger.debug("Found destroy method on class [" + clazz.getName() + "]: " + method);
						}
					}
				}
			});

			initMethods.addAll(0, currInitMethods);
			destroyMethods.addAll(currDestroyMethods);
			targetClass = targetClass.getSuperclass();
		}
		while (targetClass != null && targetClass != Object.class);

		return new LifecycleMetadata(clazz, initMethods, destroyMethods);
	}
```
解析待@javax.annotation.PostConstruct、@javax.annotation.PreDestroy的初始化、销毁的方法。

org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization

```java
@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
		try {
			metadata.invokeInitMethods(bean, beanName);
		}
		catch (InvocationTargetException ex) {
			throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Failed to invoke init method", ex);
		}
		return bean;
	}
```

```java
@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
```
在bean初始化之后执行方法。

org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#postProcessBeforeDestruction

```java
@Override
	public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
//		找到bean创建和销毁的metadata信息
		LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
		try {
//			执行bean的销毁方法
			metadata.invokeDestroyMethods(bean, beanName);
		}
		catch (InvocationTargetException ex) {
			String msg = "Invocation of destroy method failed on bean with name '" + beanName + "'";
			if (logger.isDebugEnabled()) {
				logger.warn(msg, ex.getTargetException());
			}
			else {
				logger.warn(msg + ": " + ex.getTargetException());
			}
		}
		catch (Throwable ex) {
			logger.error("Failed to invoke destroy method on bean with name '" + beanName + "'", ex);
		}
	}
```
在bean销毁之前执行的方法。
org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor.LifecycleMetadata#invokeDestroyMethods执行销毁方法

```java
public void invokeDestroyMethods(Object target, String beanName) throws Throwable {
			Collection<LifecycleElement> checkedDestroyMethods = this.checkedDestroyMethods;
			Collection<LifecycleElement> destroyMethodsToUse =
					(checkedDestroyMethods != null ? checkedDestroyMethods : this.destroyMethods);
			if (!destroyMethodsToUse.isEmpty()) {
				boolean debug = logger.isDebugEnabled();
				for (LifecycleElement element : destroyMethodsToUse) {
					if (debug) {
						logger.debug("Invoking destroy method on bean '" + beanName + "': " + element.getMethod());
					}
//					执行bean的销毁方法
					element.invoke(target);
				}
			}
		}
```

```java
public void invokeInitMethods(Object target, String beanName) throws Throwable {
			Collection<LifecycleElement> checkedInitMethods = this.checkedInitMethods;
			Collection<LifecycleElement> initMethodsToIterate =
					(checkedInitMethods != null ? checkedInitMethods : this.initMethods);
			if (!initMethodsToIterate.isEmpty()) {
				boolean debug = logger.isDebugEnabled();
				for (LifecycleElement element : initMethodsToIterate) {
					if (debug) {
						logger.debug("Invoking init method on bean '" + beanName + "': " + element.getMethod());
					}
					element.invoke(target);
				}
			}
		}
```
执行bean的init方法。

org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor 自动注入带注解的属性，setter方法和其他注入方式，检查@Autowired、@Value、@Inject注解。找到一个构造参数进行注入，如果有多个构造参数可以加注入注解，这些构造参数可以是非public的，这个注解和context:annotation-config、context:component-scan等价。这个处理器还处理@Lookup注解，这个注解用来标注prototype类型的bean在单例bean中每次获取不同的对象，底层是cglib动态代理实现。

```java
private final Set<Class<? extends Annotation>> autowiredAnnotationTypes = new LinkedHashSet<>();
```
自动依赖注入的注解类型。

```java
private final Map<Class<?>, Constructor<?>[]> candidateConstructorsCache = new ConcurrentHashMap<>(256);
```
候选的构造方法缓存。

```java
private final Map<String, InjectionMetadata> injectionMetadataCache = new ConcurrentHashMap<>(256);
```
依赖注入的方法。

依赖注入的类型包括@Autowired、@Value、@Inject注解。

```java
@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {、
//		找到自动注入的元数据
		InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
		metadata.checkConfigMembers(beanDefinition);
	}
```
执行BeanDefinition的后置处理方法。
org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#findAutowiringMetadata查询依赖注入的元数据

```java
	private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
		// Fall back to class name as cache key, for backwards compatibility with custom callers.返回到类名作为缓存键，以便向后兼容自定义调用程序。
		String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
		// Quick check on the concurrent map first, with minimal locking.首先快速检查并发映射，并使用最少的锁。
		InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
		if (InjectionMetadata.needsRefresh(metadata, clazz)) {
			synchronized (this.injectionMetadataCache) {
				metadata = this.injectionMetadataCache.get(cacheKey);
				if (InjectionMetadata.needsRefresh(metadata, clazz)) {
					if (metadata != null) {
						metadata.clear(pvs);
					}
//					构建自动准入的元数据
					metadata = buildAutowiringMetadata(clazz);
					this.injectionMetadataCache.put(cacheKey, metadata);
				}
			}
		}
		return metadata;
	}
```
查询自动注入元数据。
org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#buildAutowiringMetadata

```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
		LinkedList<InjectionMetadata.InjectedElement> elements = new LinkedList<>();
		Class<?> targetClass = clazz;

		do {
			final LinkedList<InjectionMetadata.InjectedElement> currElements = new LinkedList<>();

			ReflectionUtils.doWithLocalFields(targetClass, new ReflectionUtils.FieldCallback() {
				@Override
				public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
//				查找依赖注入注解的属性值
					AnnotationAttributes ann = AutowiredAnnotationBeanPostProcessor.this.findAutowiredAnnotation(field);
					if (ann != null) {
//						属性是静态的
						if (Modifier.isStatic(field.getModifiers())) {
							if (logger.isWarnEnabled()) {
								logger.warn("Autowired annotation is not supported on static fields: " + field);
							}
							return;
						}
//					判断required的值
						boolean required = AutowiredAnnotationBeanPostProcessor.this.determineRequiredStatus(ann);
						currElements.add(new AutowiredFieldElement(field, required));
					}
				}
			});

			ReflectionUtils.doWithLocalMethods(targetClass, new ReflectionUtils.MethodCallback() {
				@Override
				public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {
//					找到依赖注入方法的桥接方法，这里是桥接模式实现
					Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
					if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
						return;
					}
//					找到桥接方法上的注解
					AnnotationAttributes ann = AutowiredAnnotationBeanPostProcessor.this.findAutowiredAnnotation(bridgedMethod);
					if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
						if (Modifier.isStatic(method.getModifiers())) {
							if (logger.isWarnEnabled()) {
								logger.warn("Autowired annotation is not supported on static methods: " + method);
							}
							return;
						}
						if (method.getParameterCount() == 0) {
							if (logger.isWarnEnabled()) {
								logger.warn("Autowired annotation should only be used on methods with parameters: " +
										method);
							}
						}
//						确定是否是必须的
						boolean required = AutowiredAnnotationBeanPostProcessor.this.determineRequiredStatus(ann);
//					查找方法的属性
						PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
						currElements.add(new AutowiredMethodElement(method, required, pd));
					}
				}
			});

			elements.addAll(0, currElements);
			targetClass = targetClass.getSuperclass();
		}
		while (targetClass != null && targetClass != Object.class);

		return new InjectionMetadata(clazz, elements);
	}
```
查找属性上的@Autowired、@Value、@Inject注解的值，如果属性是静态的警告，不建议依赖注入的属性是静态的，解析required是否必须注入一个可用的属性值，查找类上方法的桥接方法的依赖注入注解，解析required是否必须注入一个可用的属性值，找到的结果最后构建依赖注入元数据。


org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#determineCandidateConstructors

```java
	@Override
	@Nullable
	public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName)
			throws BeanCreationException {

		// Let's check for lookup methods here..让我们检查查找方法这里..
		if (!this.lookupMethodsChecked.contains(beanName)) {
			try {
				ReflectionUtils.doWithMethods(beanClass, method -> {
					Lookup lookup = method.getAnnotation(Lookup.class);
					if (lookup != null) {
						Assert.state(beanFactory != null, "No BeanFactory available");
						LookupOverride override = new LookupOverride(method, lookup.value());
						try {
							RootBeanDefinition mbd = (RootBeanDefinition) beanFactory.getMergedBeanDefinition(beanName);
							mbd.getMethodOverrides().addOverride(override);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(beanName,
								"Cannot apply @Lookup to beans without corresponding bean definition");
						}
					}
				});
			}
			catch (IllegalStateException ex) {
				throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
			}
			this.lookupMethodsChecked.add(beanName);
		}

		// Quick check on the concurrent map first, with minimal locking.
		Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
		if (candidateConstructors == null) {
			// Fully synchronized resolution now...
			synchronized (this.candidateConstructorsCache) {
				candidateConstructors = this.candidateConstructorsCache.get(beanClass);
				if (candidateConstructors == null) {
					Constructor<?>[] rawCandidates;
					try {
						rawCandidates = beanClass.getDeclaredConstructors();
					}
					catch (Throwable ex) {
						throw new BeanCreationException(beanName,
								"Resolution of declared constructors on bean Class [" + beanClass.getName() +
								"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
					}
					List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
					Constructor<?> requiredConstructor = null;
					Constructor<?> defaultConstructor = null;
					Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);
					int nonSyntheticConstructors = 0;
					for (Constructor<?> candidate : rawCandidates) {
						if (!candidate.isSynthetic()) {
							nonSyntheticConstructors++;
						}
						else if (primaryConstructor != null) {
							continue;
						}
						AnnotationAttributes ann = findAutowiredAnnotation(candidate);
						if (ann == null) {
							Class<?> userClass = ClassUtils.getUserClass(beanClass);
							if (userClass != beanClass) {
								try {
									Constructor<?> superCtor =
											userClass.getDeclaredConstructor(candidate.getParameterTypes());
									ann = findAutowiredAnnotation(superCtor);
								}
								catch (NoSuchMethodException ex) {
									// Simply proceed, no equivalent superclass constructor found...
								}
							}
						}
						if (ann != null) {
							if (requiredConstructor != null) {
								throw new BeanCreationException(beanName,
										"Invalid autowire-marked constructor: " + candidate +
										". Found constructor with 'required' Autowired annotation already: " +
										requiredConstructor);
							}
							boolean required = determineRequiredStatus(ann);
							if (required) {
								if (!candidates.isEmpty()) {
									throw new BeanCreationException(beanName,
											"Invalid autowire-marked constructors: " + candidates +
											". Found constructor with 'required' Autowired annotation: " +
											candidate);
								}
								requiredConstructor = candidate;
							}
							candidates.add(candidate);
						}
						else if (candidate.getParameterCount() == 0) {
							defaultConstructor = candidate;
						}
					}
					if (!candidates.isEmpty()) {
						// Add default constructor to list of optional constructors, as fallback.
						if (requiredConstructor == null) {
							if (defaultConstructor != null) {
								candidates.add(defaultConstructor);
							}
							else if (candidates.size() == 1 && logger.isWarnEnabled()) {
								logger.warn("Inconsistent constructor declaration on bean with name '" + beanName +
										"': single autowire-marked constructor flagged as optional - " +
										"this constructor is effectively required since there is no " +
										"default constructor to fall back to: " + candidates.get(0));
							}
						}
						candidateConstructors = candidates.toArray(new Constructor<?>[0]);
					}
					else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
						candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
					}
					else if (nonSyntheticConstructors == 2 && primaryConstructor != null
							&& defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};
					}
					else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor};
					}
					else {
						candidateConstructors = new Constructor<?>[0];
					}
					this.candidateConstructorsCache.put(beanClass, candidateConstructors);
				}
			}
		}
		return (candidateConstructors.length > 0 ? candidateConstructors : null);
	}
```

如果beanName上不包括lookup的方法，循环获取这个类上的所有方法的@Lookup注解，找到了这个注解，从BeanFactory中获取beanName的mergedBeanDefinition，标识beanDefinition中的需要覆盖的方法，这类是cglib动态代理会覆盖方法。
org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#determineCandidateConstructors

```java
	@Override
	@Nullable
	public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName)
			throws BeanCreationException {

		// Let's check for lookup methods here..让我们检查查找方法这里..
		if (!this.lookupMethodsChecked.contains(beanName)) {
			try {
				ReflectionUtils.doWithMethods(beanClass, method -> {
					Lookup lookup = method.getAnnotation(Lookup.class);
					if (lookup != null) {
						Assert.state(beanFactory != null, "No BeanFactory available");
						LookupOverride override = new LookupOverride(method, lookup.value());
						try {
							RootBeanDefinition mbd = (RootBeanDefinition) beanFactory.getMergedBeanDefinition(beanName);
							mbd.getMethodOverrides().addOverride(override);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(beanName,
								"Cannot apply @Lookup to beans without corresponding bean definition");
						}
					}
				});
			}
			catch (IllegalStateException ex) {
				throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
			}
			this.lookupMethodsChecked.add(beanName);
		}

		// Quick check on the concurrent map first, with minimal locking.首先快速检查并发映射，并使用最少的锁。
		Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
		if (candidateConstructors == null) {
			// Fully synchronized resolution now...完全同步的分辨率现在…
			synchronized (this.candidateConstructorsCache) {
				candidateConstructors = this.candidateConstructorsCache.get(beanClass);
				if (candidateConstructors == null) {
					Constructor<?>[] rawCandidates;
					try {
						rawCandidates = beanClass.getDeclaredConstructors();
					}
					catch (Throwable ex) {
						throw new BeanCreationException(beanName,
								"Resolution of declared constructors on bean Class [" + beanClass.getName() +
								"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
					}
					List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
					Constructor<?> requiredConstructor = null;
					Constructor<?> defaultConstructor = null;
//					找到beanClass的唯一的构造方法
					Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);
					int nonSyntheticConstructors = 0;
					for (Constructor<?> candidate : rawCandidates) {
						if (!candidate.isSynthetic()) {
							nonSyntheticConstructors++;
						}
						else if (primaryConstructor != null) {
							continue;
						}
//						找到构造方法上的依赖注入的注解
						AnnotationAttributes ann = findAutowiredAnnotation(candidate);
						if (ann == null) {
//							如果beanClass是cglib动态代理的类找到此类的父类
							Class<?> userClass = ClassUtils.getUserClass(beanClass);
							if (userClass != beanClass) {
								try {
									Constructor<?> superCtor =
											userClass.getDeclaredConstructor(candidate.getParameterTypes());
//									找到父类构造方法上的依赖注入的注解
									ann = findAutowiredAnnotation(superCtor);
								}
								catch (NoSuchMethodException ex) {
									// Simply proceed, no equivalent superclass constructor found...简单地继续，没有找到等价的超类构造函数…
								}
							}
						}
						if (ann != null) {
							if (requiredConstructor != null) {
								throw new BeanCreationException(beanName,
										"Invalid autowire-marked constructor: " + candidate +
										". Found constructor with 'required' Autowired annotation already: " +
										requiredConstructor);
							}
//							查询required属性
							boolean required = determineRequiredStatus(ann);
							if (required) {
								if (!candidates.isEmpty()) {
									throw new BeanCreationException(beanName,
											"Invalid autowire-marked constructors: " + candidates +
											". Found constructor with 'required' Autowired annotation: " +
											candidate);
								}
								requiredConstructor = candidate;
							}
							candidates.add(candidate);
						}
						else if (candidate.getParameterCount() == 0) {
							defaultConstructor = candidate;
						}
					}
					if (!candidates.isEmpty()) {
						// Add default constructor to list of optional constructors, as fallback.将默认构造函数作为回退添加到可选构造函数列表中。
						if (requiredConstructor == null) {
							if (defaultConstructor != null) {
								candidates.add(defaultConstructor);
							}
							else if (candidates.size() == 1 && logger.isWarnEnabled()) {
								logger.warn("Inconsistent constructor declaration on bean with name '" + beanName +
										"': single autowire-marked constructor flagged as optional - " +
										"this constructor is effectively required since there is no " +
										"default constructor to fall back to: " + candidates.get(0));
							}
						}
						candidateConstructors = candidates.toArray(new Constructor<?>[0]);
					}
					else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
						candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
					}
					else if (nonSyntheticConstructors == 2 && primaryConstructor != null
							&& defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};
					}
					else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor};
					}
					else {
						candidateConstructors = new Constructor<?>[0];
					}
					this.candidateConstructorsCache.put(beanClass, candidateConstructors);
				}
			}
		}
		return (candidateConstructors.length > 0 ? candidateConstructors : null);
	}
```
确定候选的构造方法，如果beanName上没有lookup的方法，循环这个类上的方法获取方法上的@Lookup注解，如果找到了标识beanDefinition中需要方法覆盖，否则查询指定beanClass的候选构造方法，找到唯一的构造方法，查询构造方法上的依赖注入的注解，如果没找到，如果这个beanClass是cglib动态类的类型就查询这个类的父类的构造方法上方的依赖注入的注解，获取required属性，返回候选的构造方法集合。

org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#postProcessPropertyValues

```java
	@Override
	public PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {

		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
		try {
			metadata.inject(bean, beanName, pvs);
		}
		catch (BeanCreationException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
		}
		return pvs;
	}
```
找到bean上的制动注入的元数据，进行依赖注入。

org.springframework.beans.factory.annotation.InjectionMetadata#inject

```java
	public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
		Collection<InjectedElement> checkedElements = this.checkedElements;
		Collection<InjectedElement> elementsToIterate =
				(checkedElements != null ? checkedElements : this.injectedElements);
		if (!elementsToIterate.isEmpty()) {
			boolean debug = logger.isDebugEnabled();
			for (InjectedElement element : elementsToIterate) {
				if (debug) {
					logger.debug("Processing injected element of bean '" + beanName + "': " + element);
				}
				element.inject(target, beanName, pvs);
			}
		}
	}
```
org.springframework.beans.factory.annotation.InjectionMetadata.InjectedElement#inject

```java
	protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs)
				throws Throwable {

//			如果是属性调用属性的set方法进行设置值
			if (this.isField) {
				Field field = (Field) this.member;
				ReflectionUtils.makeAccessible(field);
				field.set(target, getResourceToInject(target, requestingBeanName));
			}
			else {
//				如果已经注入过跳过
				if (checkPropertySkipping(pvs)) {
					return;
				}
				try {
//					如果是构造方法或者工厂方法调用方法注入属性值
					Method method = (Method) this.member;
					ReflectionUtils.makeAccessible(method);
					method.invoke(target, getResourceToInject(target, requestingBeanName));
				}
				catch (InvocationTargetException ex) {
					throw ex.getTargetException();
				}
			}
		}
```
org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#processInjection

```java
public void processInjection(Object bean) throws BeanCreationException {
		Class<?> clazz = bean.getClass();
		InjectionMetadata metadata = findAutowiringMetadata(clazz.getName(), clazz, null);
		try {
			metadata.inject(bean, null, null);
		}
		catch (BeanCreationException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					"Injection of autowired dependencies failed for class [" + clazz + "]", ex);
		}
	}
```
处理@Autowired的属性和方法，查询类上属性和方法的自动注入元数据进行注入，属性注入就是调用属性的set方法或者调用构造方法和工厂方设置依赖注入的属性值。

```java
private void registerDependentBeans(@Nullable String beanName, Set<String> autowiredBeanNames) {
		if (beanName != null) {
			for (String autowiredBeanName : autowiredBeanNames) {
				if (this.beanFactory != null && this.beanFactory.containsBean(autowiredBeanName)) {
					this.beanFactory.registerDependentBean(autowiredBeanName, beanName);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Autowiring by type from bean name '" + beanName +
							"' to bean named '" + autowiredBeanName + "'");
				}
			}
		}
	}
```
如果BeanFactory中包含指定的beanDefinition注册依赖注入对的bean。

org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor 这个处理器和context:annotation-config、context:component-scan标签结合使用。

```java
private Class<? extends Annotation> requiredAnnotationType = Required.class;
```
默认检查的注解是@Required注解。

```java
@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		if (beanFactory instanceof ConfigurableListableBeanFactory) {
			this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
		}
	}
```
这个类实现了BeanFactoryAware接口覆盖这个方法设置BeanFactory。

```java
@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
	}
```
这个类实现了beanDefinition后置处理器的方法，子类可以覆盖这个方法默认什么也不处理。

org.springframework.beans.factory.config.BeanFactoryPostProcessor BeanFactory后置处理器，允许修改beanDefinition，修改bean的属性值，在创建bean之前执行。

org.springframework.beans.factory.config.BeanFactoryPostProcessor#postProcessBeanFactory 应用程序上下文初始化之后修改BeanFactory，BeanDefinition都已加载但是还没实例化，允许覆盖和添加属性。

org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor BeanDefinitionRegistry后置处理器，可以注册更多的beanDefinition。

```java
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
```

应用上下文初始化完毕后可以修改BeanDefinition注册表，在初始化bean实例化之前，可以在下个后置处理阶段之前可以注册更多的BeanDefinition。
