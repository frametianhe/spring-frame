# BeanFactory之DefaultListableBeanFactory

BeanFactory默认实现。基于BeanDefinition对象的BeanFactory。在访问bean之前先创建BeanDefinition。

```java
private final Map<Class<?>, Object> resolvableDependencies = new ConcurrentHashMap<>(16);
```
按依赖类型注入的bean注册表。

```java
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
```
BeanDefinition注册表。

```java
private final Map<Class<?>, String[]> allBeanNamesByType = new ConcurrentHashMap<>(64);
```
按类型注入的单例bean和非单例bean名称集合。

```java
private final Map<Class<?>, String[]> singletonBeanNamesByType = new ConcurrentHashMap<>(64);
```
按注入类型的单例bean名称集合。

```java
private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
```
BeanDefinition名称集合。

```java
private volatile Set<String> manualSingletonNames = new LinkedHashSet<>(16);
```
手动注册单例的名称集合。
org.springframework.beans.factory.support.DefaultListableBeanFactory#setAutowireCandidateResolver设置自动依赖bean的候选解析器

```java
public void setAutowireCandidateResolver(final AutowireCandidateResolver autowireCandidateResolver) {
		Assert.notNull(autowireCandidateResolver, "AutowireCandidateResolver must not be null");
		if (autowireCandidateResolver instanceof BeanFactoryAware) {
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					((BeanFactoryAware) autowireCandidateResolver).setBeanFactory(DefaultListableBeanFactory.this);
					return null;
				}, getAccessControlContext());
			}
			else {
				((BeanFactoryAware) autowireCandidateResolver).setBeanFactory(this);
			}
		}
		this.autowireCandidateResolver = autowireCandidateResolver;
	}
```
执行org.springframework.beans.factory.BeanFactoryAware#setBeanFactory，这个方法可以获取到banFactory，这个方法在正常的bean属性注入后，在InitializingBean.afterPropertiesSet()或bean的init方法之前执行。

org.springframework.beans.factory.support.DefaultListableBeanFactory#getBean(java.lang.Class<T>)按类型查询bean

```java
@Override
	public <T> T getBean(Class<T> requiredType) throws BeansException {
		return getBean(requiredType, (Object[]) null);
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#getBean(java.lang.Class<T>, java.lang.Object...)按指定的类型和参数查询bean

```java
	@Override
	public <T> T getBean(Class<T> requiredType, @Nullable Object... args) throws BeansException {
//		按类型解析bean
		NamedBeanHolder<T> namedBean = resolveNamedBean(requiredType, args);
//		从bean持有者中获得instance
		if (namedBean != null) {
			return namedBean.getBeanInstance();
		}
//		如果当前factory中没有查询到就从parentBeanFactory中查询
		BeanFactory parent = getParentBeanFactory();
		if (parent != null) {
			return (args != null ? parent.getBean(requiredType, args) : parent.getBean(requiredType));
		}
		throw new NoSuchBeanDefinitionException(requiredType);
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveNamedBean(java.lang.Class<T>, java.lang.Object...)按指定类型和参数返回bean

```java
@SuppressWarnings("unchecked")
	@Nullable
	private <T> NamedBeanHolder<T> resolveNamedBean(Class<T> requiredType, @Nullable Object... args) throws BeansException {
		Assert.notNull(requiredType, "Required type must not be null");
//		按类型查询bean名称集合
		String[] candidateNames = getBeanNamesForType(requiredType);

		if (candidateNames.length > 1) {
			List<String> autowireCandidates = new ArrayList<>(candidateNames.length);
			for (String beanName : candidateNames) {
//				BeanDefinition不存在或者BeanDefinition设置的是自动注入
				if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
					autowireCandidates.add(beanName);
				}
			}
			if (!autowireCandidates.isEmpty()) {
				candidateNames = StringUtils.toStringArray(autowireCandidates);
			}
		}

		if (candidateNames.length == 1) {
			String beanName = candidateNames[0];
//			按beanName查询bean
			return new NamedBeanHolder<>(beanName, getBean(beanName, requiredType, args));
		}
		else if (candidateNames.length > 1) {
			Map<String, Object> candidates = new LinkedHashMap<>(candidateNames.length);
			for (String beanName : candidateNames) {
//			bean单例注册表中包含beanName的bean且参数为空
				if (containsSingleton(beanName) && args == null) {
//					按beanName查询bean
					Object beanInstance = getBean(beanName);
					candidates.put(beanName, (beanInstance instanceof NullBean ? null : beanInstance));
				}
				else {
					candidates.put(beanName, getType(beanName));
				}
			}
//			查询注入的唯一候选的bean
			String candidateName = determinePrimaryCandidate(candidates, requiredType);
//			如果没找到唯一的bean按优先级获得唯一的bean
			if (candidateName == null) {
				candidateName = determineHighestPriorityCandidate(candidates, requiredType);
			}
			if (candidateName != null) {
				Object beanInstance = candidates.get(candidateName);
				if (beanInstance == null || beanInstance instanceof Class) {
					beanInstance = getBean(candidateName, requiredType, args);
				}
				return new NamedBeanHolder<>(candidateName, (T) beanInstance);
			}
			throw new NoUniqueBeanDefinitionException(requiredType, candidates.keySet());
		}

		return null;
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#getBeanNamesForType(java.lang.Class<?>, boolean, boolean)按类型获得所有的beanName集合

```java
@Override
	public String[] getBeanNamesForType(@Nullable Class<?> type, boolean includeNonSingletons, boolean allowEagerInit) {
		if (!isConfigurationFrozen() || type == null || !allowEagerInit) {
//			根据指定的类型获取bean名称集合
			return doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, allowEagerInit);
		}
		Map<Class<?>, String[]> cache =
				(includeNonSingletons ? this.allBeanNamesByType : this.singletonBeanNamesByType);
		String[] resolvedBeanNames = cache.get(type);
		if (resolvedBeanNames != null) {
			return resolvedBeanNames;
		}
		resolvedBeanNames = doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, true);
		if (ClassUtils.isCacheSafe(type, getBeanClassLoader())) {
			cache.put(type, resolvedBeanNames);
		}
		return resolvedBeanNames;
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#doGetBeanNamesForType按类型获得beanName集合

```java
	private String[] doGetBeanNamesForType(ResolvableType type, boolean includeNonSingletons, boolean allowEagerInit) {
		List<String> result = new ArrayList<>();

		// Check all bean definitions.检查所有的bean定义
		for (String beanName : this.beanDefinitionNames) {
			// Only consider bean as eligible if the bean name
			// is not defined as alias for some other bean.//只考虑bean的名称是否合适
//没有被定义为其他bean的别名。
//			如果beanName不是别名
			if (!isAlias(beanName)) {
				try {
//					在当前工厂中获取merge后的BeanDefinition
					RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
					// Only check bean definition if it is complete.只有在bean定义完成时才检查它。
//					BeanDefinition不是抽象的且允许提前初始化，BeanDefinition中执行了beanClass或者不是延迟加载，不是factoryBean或者不在单例bean注册表中
					if (!mbd.isAbstract() && (allowEagerInit ||
							(mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading()) &&
									!requiresEagerInitForType(mbd.getFactoryBeanName()))) {
						// In case of FactoryBean, match object created by FactoryBean.在FactoryBean的情况下，匹配由FactoryBean创建的对象。
//						判断bean是否是factoryBean
						boolean isFactoryBean = isFactoryBean(beanName, mbd);
//						找到BeanDefinition的持有者
						BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
//						找到了单例bean
						boolean matchFound =
								(allowEagerInit || !isFactoryBean ||
										(dbd != null && !mbd.isLazyInit()) || containsSingleton(beanName)) &&
								(includeNonSingletons ||
										(dbd != null ? mbd.isSingleton() : isSingleton(beanName))) &&
								isTypeMatch(beanName, type);
//						没找到bean找到了factoryBean
						if (!matchFound && isFactoryBean) {
							// In case of FactoryBean, try to match FactoryBean instance itself next. 在FactoryBean的情况下，试着将FactoryBean实例本身与下一个进行匹配。
//							beanName=&beanName
							beanName = FACTORY_BEAN_PREFIX + beanName;
							matchFound = (includeNonSingletons || mbd.isSingleton()) && isTypeMatch(beanName, type);
						}
						if (matchFound) {
							result.add(beanName);
						}
					}
				}
				catch (CannotLoadBeanClassException ex) {
					if (allowEagerInit) {
						throw ex;
					}
					// Probably contains a placeholder: let's ignore it for type matching purposes.可能包含占位符:出于类型匹配的目的，让我们忽略它。
					if (this.logger.isDebugEnabled()) {
						this.logger.debug("Ignoring bean class loading failure for bean '" + beanName + "'", ex);
					}
					onSuppressedException(ex);
				}
				catch (BeanDefinitionStoreException ex) {
					if (allowEagerInit) {
						throw ex;
					}
					// Probably contains a placeholder: let's ignore it for type matching purposes.可能包含占位符:出于类型匹配的目的，让我们忽略它。
					if (this.logger.isDebugEnabled()) {
						this.logger.debug("Ignoring unresolvable metadata in bean definition '" + beanName + "'", ex);
					}
					onSuppressedException(ex);
				}
			}
		}

		// Check manually registered singletons too.也检查手动注册的单例。
		for (String beanName : this.manualSingletonNames) {
			try {
				// In case of FactoryBean, match object created by FactoryBean.对于FactoryBean，匹配FactoryBean创建的对象。
				if (isFactoryBean(beanName)) {
					if ((includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type)) {
						result.add(beanName);
						// Match found for this bean: do not match FactoryBean itself anymore.为这个bean找到匹配:不再匹配FactoryBean本身。
						continue;
					}
					// In case of FactoryBean, try to match FactoryBean itself next.对于FactoryBean，接下来尝试匹配FactoryBean本身。
					beanName = FACTORY_BEAN_PREFIX + beanName;
				}
				// Match raw bean instance (might be raw FactoryBean).匹配原始bean实例(可能是原始的FactoryBean)。
				if (isTypeMatch(beanName, type)) {
					result.add(beanName);
				}
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Shouldn't happen - probably a result of circular reference resolution...不应该发生——可能是循环引用解析的结果……
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to check manually registered singleton with name '" + beanName + "'", ex);
				}
			}
		}

		return StringUtils.toStringArray(result);
	}
```
循环beanDefinitionNames BeanDefinition名称注册表，如果beanName不是别名，按beanName查询BeanDefinition，如果BeanDefinition不是抽象的，允许早期初始化或有beanClass或不是延迟加载且有factoryBeanName，不是factoryBean，在单例bean注册表中，如果允许早期初始化或不是factoryBean或不是延迟初始化在到你bean注册表中，是单例bean，单例bean和指定的beanName和type一致返回bean。

如果没找到循环manualSingletonNames手动注册的beanName注册表，如果beanName是factoryBean或者beanName是单例，bean和指定的beanName和type一致返回。

返回org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveNamedBean(java.lang.Class<T>, java.lang.Object...)

```java
if (candidateNames.length > 1) {
			List<String> autowireCandidates = new ArrayList<>(candidateNames.length);
			for (String beanName : candidateNames) {
//				BeanDefinition不存在或者BeanDefinition设置的是自动注入
				if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
					autowireCandidates.add(beanName);
				}
			}
			if (!autowireCandidates.isEmpty()) {
				candidateNames = StringUtils.toStringArray(autowireCandidates);
			}
		}
```
如果找到候选的beanNames是多个，循环beanNames列表，beanName不在BeanDefinition注册表中，BeanDefinition配置的是自动依赖注入，添加到候选的beanName集合中。

如果找到候选的beanName是一个按beanName从BeanFactory查询bean。如果找到候选的单例beanName集合中是多个，循环候选beanNames集合，如果beanName在单例bean注册表中，参数是null，按beanName查询bean，否则添加到候选beanNames集合中。

按指定的Primary和优先级找到唯一候选的beanName，按beanName查询bean返回。

返回org.springframework.beans.factory.support.DefaultListableBeanFactory#getBean(java.lang.Class<T>, java.lang.Object...)，按类型找到的NamedBeanHolder获取beanInstance，如果在当前BeanFactory查询不到就从parentBeanFactory中查询。

```java
@Override
	public boolean containsBeanDefinition(String beanName) {
		Assert.notNull(beanName, "Bean name must not be null");
		return this.beanDefinitionMap.containsKey(beanName);
	}
```
判断beanName是否在BeanDefinition中。
org.springframework.beans.factory.support.DefaultListableBeanFactory#getBeansOfType(java.lang.Class<T>, boolean, boolean)按类型获得bean集合

```java
	@Override
	@SuppressWarnings("unchecked")
	public <T> Map<String, T> getBeansOfType(@Nullable Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
			throws BeansException {

//		按类型查询beanName集合
		String[] beanNames = getBeanNamesForType(type, includeNonSingletons, allowEagerInit);
		Map<String, T> result = new LinkedHashMap<>(beanNames.length);
		for (String beanName : beanNames) {
			try {
//				从BeanFactory中获得beanInstance，这个的beanInstance可能是factoryBean的对象
				Object beanInstance = getBean(beanName);
				result.put(beanName, (beanInstance instanceof NullBean ? null : (T) beanInstance));
			}
			catch (BeanCreationException ex) {
				Throwable rootCause = ex.getMostSpecificCause();
				if (rootCause instanceof BeanCurrentlyInCreationException) {
					BeanCreationException bce = (BeanCreationException) rootCause;
					String exBeanName = bce.getBeanName();
					if (exBeanName != null && isCurrentlyInCreation(exBeanName)) {
						if (this.logger.isDebugEnabled()) {
							this.logger.debug("Ignoring match to currently created bean '" + exBeanName + "': " +
									ex.getMessage());
						}
						onSuppressedException(ex);
						// Ignore: indicates a circular reference when autowiring constructors.
						// We want to find matches other than the currently created bean itself.// Ignore:表示自动装配构造函数时的循环引用。
//我们希望找到与当前创建的bean本身不同的匹配项。
						continue;
					}
				}
				throw ex;
			}
		}
		return result;
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#getBeanNamesForAnnotation按annotation查询beanName集合。

```java
	@Override
	public String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType) {
		List<String> results = new ArrayList<>();
//		循环BeanDefinition名称集合
		for (String beanName : this.beanDefinitionNames) {
//			按beanName查询BeanDefinition
			BeanDefinition beanDefinition = getBeanDefinition(beanName);
//			BeanDefinition不是抽象的能找到beanName指定annotation类型的annotation
			if (!beanDefinition.isAbstract() && findAnnotationOnBean(beanName, annotationType) != null) {
				results.add(beanName);
			}
		}
//		循环手动注册的单例bean注册表，没找到beanName，按beanName和指定的annotation类型能找到annotation
		for (String beanName : this.manualSingletonNames) {
			if (!results.contains(beanName) && findAnnotationOnBean(beanName, annotationType) != null) {
				results.add(beanName);
			}
		}
		return StringUtils.toStringArray(results);
	}
```
按beanName和annotation类型找到beanNames集合。
org.springframework.beans.factory.support.DefaultListableBeanFactory#getBeansWithAnnotation按annotation获得bean

```java
	@Override
	public Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) {
//		按annotation类型获得beanName集合
		String[] beanNames = getBeanNamesForAnnotation(annotationType);
		Map<String, Object> results = new LinkedHashMap<>(beanNames.length);
		for (String beanName : beanNames) {
//			按beanName从BeanFactory中查询bean
			results.put(beanName, getBean(beanName));
		}
		return results;
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#findAnnotationOnBean找到bean上面的annotation

```java
	@Override
	@Nullable
	public <A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
			throws NoSuchBeanDefinitionException{

		A ann = null;
//		查询beanName的类型
		Class<?> beanType = getType(beanName);
		if (beanType != null) {
//			按指定的annotation类型获取指定类型上的annotation
			ann = AnnotationUtils.findAnnotation(beanType, annotationType);
		}
//		没找到annotation但是beanName在BeanDefinition注册表中
		if (ann == null && containsBeanDefinition(beanName)) {
//			查询beanName的BeanDefinition
			BeanDefinition bd = getMergedBeanDefinition(beanName);
			if (bd instanceof AbstractBeanDefinition) {
				AbstractBeanDefinition abd = (AbstractBeanDefinition) bd;
				if (abd.hasBeanClass()) {
//					获得BeanDefinition中的beanClass上指定annotation类型的annotation
					ann = AnnotationUtils.findAnnotation(abd.getBeanClass(), annotationType);
				}
			}
		}
		return ann;
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#isAutowireCandidate(java.lang.String, org.springframework.beans.factory.config.DependencyDescriptor, org.springframework.beans.factory.support.AutowireCandidateResolver)

```java
	protected boolean isAutowireCandidate(String beanName, DependencyDescriptor descriptor, AutowireCandidateResolver resolver)
			throws NoSuchBeanDefinitionException {

		String beanDefinitionName = BeanFactoryUtils.transformedBeanName(beanName);
//		如果BeanDefinition存在
		if (containsBeanDefinition(beanDefinitionName)) {
//			判断该bean是否是自动注入的候选项
			return isAutowireCandidate(beanName, getMergedLocalBeanDefinition(beanDefinitionName), descriptor, resolver);
		}
//		如果beanName在单例bean注册表中
		else if (containsSingleton(beanName)) {
			return isAutowireCandidate(beanName, new RootBeanDefinition(getType(beanName)), descriptor, resolver);
		}

		BeanFactory parent = getParentBeanFactory();
		if (parent instanceof DefaultListableBeanFactory) {
			// No bean definition found in this factory -> delegate to parent.在这个工厂中没有找到bean定义——>委托给父类。
			return ((DefaultListableBeanFactory) parent).isAutowireCandidate(beanName, descriptor, resolver);
		}
		else if (parent instanceof ConfigurableListableBeanFactory) {
			// If no DefaultListableBeanFactory, can't pass the resolver along.如果没有DefaultListableBeanFactory，就不能传递解析器。
			return ((ConfigurableListableBeanFactory) parent).isAutowireCandidate(beanName, descriptor);
		}
		else {
			return true;
		}
	}
```
判断BeaName是否是自动依赖注入。如果BeanDefinitionName存在，判断是否是自动依赖注入，判断beanName是否在单例bean注册表中，判断是否是自动依赖注入。从parentBeanFactory判断，如果parentBeanFactory是DefaultListableBeanFactory或ConfigurableListableBeanFactory判断是否是自动依赖注入。

org.springframework.beans.factory.support.DefaultListableBeanFactory#isAutowireCandidate(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, org.springframework.beans.factory.config.DependencyDescriptor, org.springframework.beans.factory.support.AutowireCandidateResolver)

```java
	protected boolean isAutowireCandidate(String beanName, RootBeanDefinition mbd,
			DependencyDescriptor descriptor, AutowireCandidateResolver resolver) {

		String beanDefinitionName = BeanFactoryUtils.transformedBeanName(beanName);
//		将bean定义解析为bean类
		resolveBeanClass(mbd, beanDefinitionName);
		if (mbd.isFactoryMethodUnique) {
			boolean resolve;
			synchronized (mbd.constructorArgumentLock) {
				resolve = (mbd.resolvedConstructorOrFactoryMethod == null);
			}
			if (resolve) {
//				解析工厂方法
				new ConstructorResolver(this).resolveFactoryMethodIfPossible(mbd);
			}
		}
//		返回BeanDefinition中配置的自动注入选项
		return resolver.isAutowireCandidate(
				new BeanDefinitionHolder(mbd, beanName, getAliases(beanDefinitionName)), descriptor);
	}
```
解析BeanDefinition中的beanClass，如果resolvedConstructorOrFactoryMethod开关配置的是解析构造参数和工厂方法，进行构造参数解析。最后判断bean是否可以自动依赖注入。

org.springframework.beans.factory.support.ConstructorResolver#resolveFactoryMethodIfPossible解析工厂方法

```java
	public void resolveFactoryMethodIfPossible(RootBeanDefinition mbd) {
		Class<?> factoryClass;
		boolean isStatic;
//		如果factoryBeanName不为空从BeanFactory中获取
		if (mbd.getFactoryBeanName() != null) {
			factoryClass = this.beanFactory.getType(mbd.getFactoryBeanName());
			isStatic = false;
		}
		else {
//			否则从BeanDefinition中获取beanClass
			factoryClass = mbd.getBeanClass();
			isStatic = true;
		}
		Assert.state(factoryClass != null, "Unresolvable factory class");
//		如果此类是cglib动态代理类获取这个类的父类
		factoryClass = ClassUtils.getUserClass(factoryClass);

//		查找给定bean定义的候选方法
		Method[] candidates = getCandidateMethods(factoryClass, mbd);
		Method uniqueCandidate = null;
		for (Method candidate : candidates) {
//			如果方法是静态工厂方法
			if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
				if (uniqueCandidate == null) {
					uniqueCandidate = candidate;
				}
//				如果找到的参数类型不匹配退出
				else if (!Arrays.equals(uniqueCandidate.getParameterTypes(), candidate.getParameterTypes())) {
					uniqueCandidate = null;
					break;
				}
			}
		}
		synchronized (mbd.constructorArgumentLock) {
//			找到静态工厂方法
			mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
		}
	}
```
找到factoryBeanName所在的beanClass，如果beanClass是cglib类找到cglib的父类，获取factoryClass的候选方法，如果候选方法是静态工厂方法就判断参数类型是够一致返回。

org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons初始化单例bean的潜质处理。

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
循环beanDefinitionNames列表，按beanName获取mergedBeanDefinition，如果BeanDefinition不是抽象的，是单例的，不是延迟加载，如果beanName是factoryBean，从BeanFactory中获取bean，如果bean是factoryBean，如果factoryBean是SmartFactoryBean，判断org.springframework.beans.factory.SmartFactoryBean#isEagerInit 是否立即加载，默认不是立即加载，如果是立即加载从BeanFactory中获取beanName的bean，如果beanName不是factoryBean直接按beanName从BeanFactory中获取bean。

循环BeanDefinitionNames列表，按beanName从单例bean注册表中查询bean，如果bean是SmartInitializingSingleton，执行org.springframework.beans.factory.SmartInitializingSingleton#afterSingletonsInstantiated，这个方法在创建了常规的bean之后调用。

org.springframework.beans.factory.support.DefaultListableBeanFactory#registerSingleton

```java
	@Override
	public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
		super.registerSingleton(beanName, singletonObject);

//		检查bean创建阶段已经开始
		if (hasBeanCreationStarted()) {
			// Cannot modify startup-time collection elements anymore (for stable iteration)不能再修改启动时间集合元素(为了稳定的迭代)
			synchronized (this.beanDefinitionMap) {
				if (!this.beanDefinitionMap.containsKey(beanName)) {
					Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames.size() + 1);
					updatedSingletons.addAll(this.manualSingletonNames);
					updatedSingletons.add(beanName);
					this.manualSingletonNames = updatedSingletons;
				}
			}
		}
		else {
			// Still in startup registration phase仍在启动注册阶段
			if (!this.beanDefinitionMap.containsKey(beanName)) {
				this.manualSingletonNames.add(beanName);
			}
		}

//		清除按类型注入的缓存
		clearByTypeCache();
	}
```
注册单例bean，org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#registerSingleton

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
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingleton

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
同步单例bean注册表，如果beanName的bean已经注册过直接报错，否则就把单例bean注册到注册表中。

如果bean在创建阶段，如果beanDefinitionMap中不包括beanName，就把beanName加到manualSingletonNames列表中。如果bean不是在创建阶段，beanDefinitionMap中不包括beanName，就把beanName加到manualSingletonNames列表中。最后清空allBeanNamesByType、singletonBeanNamesByType列表。

org.springframework.beans.factory.support.DefaultListableBeanFactory#destroySingleton销毁单例的bean

```java
@Override
	public void destroySingleton(String beanName) {
		super.destroySingleton(beanName);
		this.manualSingletonNames.remove(beanName);
//		释放缓存
		clearByTypeCache();
	}
```
销毁单例的bean，清空manualSingletonNames中beanName的bean，清空allBeanNamesByType、singletonBeanNamesByType列表。

org.springframework.beans.factory.support.DefaultListableBeanFactory#destroySingletons清空所有的单例bean

```java
@Override
	public void destroySingletons() {
		super.destroySingletons();
//		清除记录的单例beanName的缓存
		this.manualSingletonNames.clear();
		clearByTypeCache();
	}
```
销毁所有的单例bean注册表中的bean，清空factoryBeanObjectCache列表。
