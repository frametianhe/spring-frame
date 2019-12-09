# BeanFactory之AbstractBeanFactory②

org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean这一部分逻辑

```java
	else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.//失败，如果我们已经创建了这个bean实例:
//我们假设在一个循环引用中。
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
					// Delegation to parent with explicit args.使用显式参数委托给父进程。
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
//				检查bean定义如果是abstract不让创建
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
//						创建当前bean依赖的bean
						getBean(dep);
					}
				}

				// Create bean instance.创建bean对象
//				如果bean定义是单例对象
				if (mbd.isSingleton()) {
//					如果bean定义是单例，获取单例的bean
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
//							bean初始化
								return AbstractBeanFactory.this.createBean(beanName, mbd, args);
							} catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.//从单例缓存中显式删除实例:它可能已经被放在那里了
//急切被创建进程，以允许循环引用解析。
//还要删除接收到该bean的临时引用的所有bean。
								AbstractBeanFactory.this.destroySingleton(beanName);
								throw ex;
							}
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
```
从你这里开始，如果从当前factoryBean中获取不到就从上级factory中获取，创建完毕标识bean已创建，从BeanDefinition中获取依赖的beanName，注册依赖的bean并获取依赖的bean。如果BeanDefinition是单例的就获取单例的bean，如果获取到的bean从BeanFactory对象注册表中获取bean对象，如果注册表中获取不到就从factoryBean中创建bean。如果BeanDefinition是原型的就获取原型的bean，如果获取到的bean从BeanFactory对象注册表中获取bean对象，如果注册表中获取不到就从factoryBean中创建bean。如果BeanDefinition是scope的就获取scope的bean，如果获取到的bean从BeanFactory对象注册表中获取bean对象，如果注册表中获取不到就从factoryBean中创建bean。

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#registerDependentBean注册依赖的bean在bean销毁之前销毁

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
按别名同步获取bean依赖的beanName或bean的依赖项。

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)获取单例的bean，从单例bean注册表中获取对象，如果获取不到从objectFactory中获取对象，从BeanFactory中获取对象，如果出现异常了就销毁单例bean。如果创建完成就注册到单例bean的注册表中。

```java
@Override
						public Object getObject() throws BeansException {
							try {
//							bean初始化
								return AbstractBeanFactory.this.createBean(beanName, mbd, args);
							} catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.//从单例缓存中显式删除实例:它可能已经被放在那里了
//急切被创建进程，以允许循环引用解析。
//还要删除接收到该bean的临时引用的所有bean。
								AbstractBeanFactory.this.destroySingleton(beanName);
								throw ex;
							}
						}
```

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])创建单例的bean，在bean初始化之前执行BeanPostProcessors，创建单例bean，处理BeanPostProcessors并注册依赖的bean。

```java
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.//确保此时bean类已经解析，并且
//		在动态解析类的情况下，克隆bean定义
//不能存储在共享的合并bean定义中。
//		解析BeanDefinition中指定的bean的class对象
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.给beanpostprocessor一个机会来返回代理，而不是目标bean实例。
//			bean初始化之前执行beanProccessors
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
//			bean创建
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isDebugEnabled()) {
				logger.debug("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```
处理bean初始化之前的beanProccessors并创建bean。

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation bean初始化之前的操作

```java
@Nullable
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
//		处理在bean初始化之前操作的开关已打开
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.确保此时bean类已经被解析。
//			判断是否有bean初始化之前或者在bean初始化之后给对象赋值之前的操作，一般用于对bean的再次初始化
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
//				根据BeanDefinition确定bean类型
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
//					调用bean初始化前处理器
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
//						bean初始化以后调用处理器
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
```
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInstantiation调用bean前置处理器，这个方法在bean初始化之前做一些事情，返回包装后的bean，类似于InitializingBean接口的afterPropertiesSet方法或者bean的init方法

```java
@Nullable
	protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
//		保存在BeanFactory list中
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
//				执行beanPostProccessor的postProcessBeforeInstantiation方法
				Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
				if (result != null) {
					return result;
				}
			}
		}
		return null;
	}
```
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization执行bean的初始化后置处理，这个方法一般被factoryBean创建对象之后调用或者factoryBean创建的对象执行，这个方法也将被applyBeanPostProcessorsBeforeInstantiation方法调用。

返回方法org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])创建bean

```java
Object beanInstance = doCreateBean(beanName, mbdToUse, args);
```
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean默认bean初始化、工厂方法、构造参数。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
//		bean定义如果是单例
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
//			创建bean对象
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.允许后处理器修改合并后的bean定义。
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
//			填充依赖注入的bean对象
			populateBean(beanName, mbd, instanceWrapper);
//			初始化bean
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

//		尽早的缓存单例对象以便循环引用
		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
//				初始化的bean没被包装又有依赖
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
//					获取依赖的bean
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.注册的bean作为可销毁的
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```
如果BeanDefinition是单例的，从factoryBean本地缓存中查询bean对象，如果没查询到就创建bean对象，同步处理MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition方法，创建早期对象避免循环引用并添加到单例bean本地缓存中，填充依赖注入的bean对象并执行bean初始化回调方法和beanPostProcessor的回调方法。查询依赖注入的bean并进行依赖注入，最后注册可以销毁的bean。

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance创建bean对象

```java
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

//		类必须是public，构造方法必须是public
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

//		如果工厂方法不为空，就用工厂方法创建bean
		if (mbd.getFactoryMethodName() != null)  {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
//				使用有参构造方法注入
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
//				使用默认构造参数初始化bean
				return instantiateBean(beanName, mbd);
			}
		}

		// Need to determine the constructor...
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
//			有参构造参数注入
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// No special handling: simply use no-arg constructor.无需特殊处理:只需使用无arg构造函数。
		return instantiateBean(beanName, mbd);
	}
```
检查类必须是public的，构造方法必须是public的，如果工厂方法不为空就先用工厂方法创建对象，如果需要自动注入先用构造方法注入否则就初始化bean，从beanPostProcessors中查询构造方法，如果BeanDefinition中是按构造参数自动注入就采取构造参数注入，最后初始化bean。

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#instantiateUsingFactoryMethod使用工厂方法实例化bean，该方法可能是静态的，BeanDefinition中指定的类不是factoryBean而是采用依赖注入配置的工厂对象。

```java
protected BeanWrapper instantiateUsingFactoryMethod(
			String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {

		return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
	}
```
org.springframework.beans.factory.support.ConstructorResolver#instantiateUsingFactoryMethod遍历执行工厂对象的静态实例方法进行匹配，并且参数一致，如果匹配到就是用工厂方法返回bean对象，否则就报错。

```java
	public BeanWrapper instantiateUsingFactoryMethod(
			final String beanName, final RootBeanDefinition mbd, @Nullable final Object[] explicitArgs) {

		BeanWrapperImpl bw = new BeanWrapperImpl();
		this.beanFactory.initBeanWrapper(bw);

		Object factoryBean;
		Class<?> factoryClass;
		boolean isStatic;

		String factoryBeanName = mbd.getFactoryBeanName();
		if (factoryBeanName != null) {
//			工厂类名不能和beanName相同
			if (factoryBeanName.equals(beanName)) {
				throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
						"factory-bean reference points back to the same bean definition");
			}
			factoryBean = this.beanFactory.getBean(factoryBeanName);
			if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) {
				throw new ImplicitlyAppearedSingletonException();
			}
			factoryClass = factoryBean.getClass();
			isStatic = false;
		}
		else {
			// It's a static factory method on the bean class. bean定义必须要指定一个bean引用
			if (!mbd.hasBeanClass()) {
				throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
						"bean definition declares neither a bean class nor a factory-bean reference");
			}
			factoryBean = null;
			factoryClass = mbd.getBeanClass();
			isStatic = true;
		}

		Method factoryMethodToUse = null;
		ArgumentsHolder argsHolderToUse = null;
		Object[] argsToUse = null;

		if (explicitArgs != null) {
			argsToUse = explicitArgs;
		}
		else {
			Object[] argsToResolve = null;
			synchronized (mbd.constructorArgumentLock) {
				factoryMethodToUse = (Method) mbd.resolvedConstructorOrFactoryMethod;
				if (factoryMethodToUse != null && mbd.constructorArgumentsResolved) {
					// Found a cached factory method...
					argsToUse = mbd.resolvedConstructorArguments;
					if (argsToUse == null) {
						argsToResolve = mbd.preparedConstructorArguments;
					}
				}
			}
			if (argsToResolve != null) {
				argsToUse = resolvePreparedArguments(beanName, mbd, bw, factoryMethodToUse, argsToResolve);
			}
		}

		if (factoryMethodToUse == null || argsToUse == null) {
			// Need to determine the factory method...
			// Try all methods with this name to see if they match the given arguments.
			factoryClass = ClassUtils.getUserClass(factoryClass);

			Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
			List<Method> candidateSet = new ArrayList<>();
			for (Method candidate : rawCandidates) {
				if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
					candidateSet.add(candidate);
				}
			}
			Method[] candidates = candidateSet.toArray(new Method[0]);
			AutowireUtils.sortFactoryMethods(candidates);

			ConstructorArgumentValues resolvedValues = null;
//			默认类型自动装配，判断是否是构造参数装配
			boolean autowiring = (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
			int minTypeDiffWeight = Integer.MAX_VALUE;
			Set<Method> ambiguousFactoryMethods = null;

			int minNrOfArgs;
			if (explicitArgs != null) {
				minNrOfArgs = explicitArgs.length;
			}
			else {
				// We don't have arguments passed in programmatically, so we need to resolve the
				// arguments specified in the constructor arguments held in the bean definition.
				if (mbd.hasConstructorArgumentValues()) {
					ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
					resolvedValues = new ConstructorArgumentValues();
					minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
				}
				else {
					minNrOfArgs = 0;
				}
			}

			LinkedList<UnsatisfiedDependencyException> causes = null;

			for (Method candidate : candidates) {
				Class<?>[] paramTypes = candidate.getParameterTypes();

				if (paramTypes.length >= minNrOfArgs) {
					ArgumentsHolder argsHolder;

					if (explicitArgs != null){
						// Explicit arguments given -> arguments length must match exactly.
						if (paramTypes.length != explicitArgs.length) {
							continue;
						}
						argsHolder = new ArgumentsHolder(explicitArgs);
					}
					else {
						// Resolved constructor arguments: type conversion and/or autowiring necessary.
						try {
							String[] paramNames = null;
							ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
							if (pnd != null) {
								paramNames = pnd.getParameterNames(candidate);
							}
							argsHolder = createArgumentArray(
									beanName, mbd, resolvedValues, bw, paramTypes, paramNames, candidate, autowiring);
						}
						catch (UnsatisfiedDependencyException ex) {
							if (this.beanFactory.logger.isTraceEnabled()) {
								this.beanFactory.logger.trace("Ignoring factory method [" + candidate +
										"] of bean '" + beanName + "': " + ex);
							}
							// Swallow and try next overloaded factory method.
							if (causes == null) {
								causes = new LinkedList<>();
							}
							causes.add(ex);
							continue;
						}
					}

					int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
							argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
					// Choose this factory method if it represents the closest match.
					if (typeDiffWeight < minTypeDiffWeight) {
						factoryMethodToUse = candidate;
						argsHolderToUse = argsHolder;
						argsToUse = argsHolder.arguments;
						minTypeDiffWeight = typeDiffWeight;
						ambiguousFactoryMethods = null;
					}
					// Find out about ambiguity: In case of the same type difference weight
					// for methods with the same number of parameters, collect such candidates
					// and eventually raise an ambiguity exception.
					// However, only perform that check in non-lenient constructor resolution mode,
					// and explicitly ignore overridden methods (with the same parameter signature).
					else if (factoryMethodToUse != null && typeDiffWeight == minTypeDiffWeight &&
							!mbd.isLenientConstructorResolution() &&
							paramTypes.length == factoryMethodToUse.getParameterCount() &&
							!Arrays.equals(paramTypes, factoryMethodToUse.getParameterTypes())) {
						if (ambiguousFactoryMethods == null) {
							ambiguousFactoryMethods = new LinkedHashSet<>();
							ambiguousFactoryMethods.add(factoryMethodToUse);
						}
						ambiguousFactoryMethods.add(candidate);
					}
				}
			}

			if (factoryMethodToUse == null) {
				if (causes != null) {
					UnsatisfiedDependencyException ex = causes.removeLast();
					for (Exception cause : causes) {
						this.beanFactory.onSuppressedException(cause);
					}
					throw ex;
				}
				List<String> argTypes = new ArrayList<>(minNrOfArgs);
				if (explicitArgs != null) {
					for (Object arg : explicitArgs) {
						argTypes.add(arg != null ? arg.getClass().getSimpleName() : "null");
					}
				}
				else if (resolvedValues != null){
					Set<ValueHolder> valueHolders = new LinkedHashSet<>(resolvedValues.getArgumentCount());
					valueHolders.addAll(resolvedValues.getIndexedArgumentValues().values());
					valueHolders.addAll(resolvedValues.getGenericArgumentValues());
					for (ValueHolder value : valueHolders) {
						String argType = (value.getType() != null ? ClassUtils.getShortName(value.getType()) :
								(value.getValue() != null ? value.getValue().getClass().getSimpleName() : "null"));
						argTypes.add(argType);
					}
				}
				String argDesc = StringUtils.collectionToCommaDelimitedString(argTypes);
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"No matching factory method found: " +
						(mbd.getFactoryBeanName() != null ?
							"factory bean '" + mbd.getFactoryBeanName() + "'; " : "") +
						"factory method '" + mbd.getFactoryMethodName() + "(" + argDesc + ")'. " +
						"Check that a method with the specified name " +
						(minNrOfArgs > 0 ? "and arguments " : "") +
						"exists and that it is " +
						(isStatic ? "static" : "non-static") + ".");
			}
			else if (void.class == factoryMethodToUse.getReturnType()) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Invalid factory method '" + mbd.getFactoryMethodName() +
						"': needs to have a non-void return type!");
			}
			else if (ambiguousFactoryMethods != null) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Ambiguous factory method matches found in bean '" + beanName + "' " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
						ambiguousFactoryMethods);
			}

			if (explicitArgs == null && argsHolderToUse != null) {
				argsHolderToUse.storeCache(mbd, factoryMethodToUse);
			}
		}

		try {
			Object beanInstance;

			if (System.getSecurityManager() != null) {
				final Object fb = factoryBean;
				final Method factoryMethod = factoryMethodToUse;
				final Object[] args = argsToUse;
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						beanFactory.getInstantiationStrategy().instantiate(mbd, beanName, beanFactory, fb, factoryMethod, args),
						beanFactory.getAccessControlContext());
			}
			else {
				beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
						mbd, beanName, this.beanFactory, factoryBean, factoryMethodToUse, argsToUse);
			}

			bw.setBeanInstance(beanInstance);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean instantiation via factory method failed", ex);
		}
	}
```
查询BeanDefinition中的factoryBean名称并从BeanFactory中查询factoryBean对象，如果不是factoryBean也没找到工厂方法和可用的参数，找到工厂类的所有候选方法，如果是静态的并且是工厂方法添加到集合中，然后对候选方法进行排序，顺序是按方法修饰符public参数比较多排在前面，非public参数比较少的排在后面，如果有参数就解析参数的值，相同类型的参数就按权重查找，最后按BeanFactory的初始化策略初始化bean，这里的默认初始化策略是cglib，org.springframework.beans.factory.support.SimpleInstantiationStrategy#instantiate(org.springframework.beans.factory.support.RootBeanDefinition, java.lang.String, org.springframework.beans.factory.BeanFactory, java.lang.Object, java.lang.reflect.Method, java.lang.Object...)初始化bean

```java
@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			@Nullable Object factoryBean, final Method factoryMethod, @Nullable Object... args) {

		try {
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					ReflectionUtils.makeAccessible(factoryMethod);
					return null;
				});
			}
			else {
				ReflectionUtils.makeAccessible(factoryMethod);
			}

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
从threadLocal中查询到本线程的工厂方法创建bean对象。

返回方法org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance

```java
if (resolved) {
			if (autowireNecessary) {
//				使用有参构造方法注入
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
//				使用默认构造参数初始化bean
				return instantiateBean(beanName, mbd);
			}
		}
```
如果构造方法解析完并且要自动依赖注入，采用构造方法依赖注入，否则就默认构造参数初始化bean。

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireConstructor构造参数自动注入

```java
	public BeanWrapper autowireConstructor(final String beanName, final RootBeanDefinition mbd,
			@Nullable Constructor<?>[] chosenCtors, @Nullable final Object[] explicitArgs) {

		BeanWrapperImpl bw = new BeanWrapperImpl();
		this.beanFactory.initBeanWrapper(bw);

		Constructor<?> constructorToUse = null;
		ArgumentsHolder argsHolderToUse = null;
		Object[] argsToUse = null;

		if (explicitArgs != null) {
			argsToUse = explicitArgs;
		}
		else {
			Object[] argsToResolve = null;
			synchronized (mbd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse != null && mbd.constructorArgumentsResolved) {
					// Found a cached constructor...
					argsToUse = mbd.resolvedConstructorArguments;
					if (argsToUse == null) {
						argsToResolve = mbd.preparedConstructorArguments;
					}
				}
			}
			if (argsToResolve != null) {
//				解析bean定义中的方法
				argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve);
			}
		}

		if (constructorToUse == null) {
			// Need to resolve the constructor.判断是否是构造参数注入
			boolean autowiring = (chosenCtors != null ||
					mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
			ConstructorArgumentValues resolvedValues = null;

			int minNrOfArgs;
			if (explicitArgs != null) {
				minNrOfArgs = explicitArgs.length;
			}
			else {
				ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
				resolvedValues = new ConstructorArgumentValues();
//				解析构造参数
				minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
			}

			// Take specified constructors, if any.使用指定的构造函数(如果有的话)。
			Constructor<?>[] candidates = chosenCtors;
			if (candidates == null) {
				Class<?> beanClass = mbd.getBeanClass();
				try {
					candidates = (mbd.isNonPublicAccessAllowed() ?
							beanClass.getDeclaredConstructors() : beanClass.getConstructors());
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Resolution of declared constructors on bean Class [" + beanClass.getName() +
							"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
				}
			}
			AutowireUtils.sortConstructors(candidates);
			int minTypeDiffWeight = Integer.MAX_VALUE;
			Set<Constructor<?>> ambiguousConstructors = null;
			LinkedList<UnsatisfiedDependencyException> causes = null;

			for (Constructor<?> candidate : candidates) {
				Class<?>[] paramTypes = candidate.getParameterTypes();

				if (constructorToUse != null && argsToUse.length > paramTypes.length) {
					// Already found greedy constructor that can be satisfied ->
					// do not look any further, there are only less greedy constructors left.
					break;
				}
				if (paramTypes.length < minNrOfArgs) {
					continue;
				}

				ArgumentsHolder argsHolder;
				if (resolvedValues != null) {
					try {
//						@ConstructorProperties注释解析
						String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
						if (paramNames == null) {
							ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
							if (pnd != null) {
								paramNames = pnd.getParameterNames(candidate);
							}
						}
//						解析构造参数并转换成数组
						argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
								getUserDeclaredConstructor(candidate), autowiring);
					}
					catch (UnsatisfiedDependencyException ex) {
						if (this.beanFactory.logger.isTraceEnabled()) {
							this.beanFactory.logger.trace(
									"Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
						}
						// Swallow and try next constructor.
						if (causes == null) {
							causes = new LinkedList<>();
						}
						causes.add(ex);
						continue;
					}
				}
				else {
					// Explicit arguments given -> arguments length must match exactly.
					if (paramTypes.length != explicitArgs.length) {
						continue;
					}
					argsHolder = new ArgumentsHolder(explicitArgs);
				}

				int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
						argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
				// Choose this constructor if it represents the closest match.
				if (typeDiffWeight < minTypeDiffWeight) {
					constructorToUse = candidate;
					argsHolderToUse = argsHolder;
					argsToUse = argsHolder.arguments;
					minTypeDiffWeight = typeDiffWeight;
					ambiguousConstructors = null;
				}
				else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
					if (ambiguousConstructors == null) {
						ambiguousConstructors = new LinkedHashSet<>();
						ambiguousConstructors.add(constructorToUse);
					}
					ambiguousConstructors.add(candidate);
				}
			}

			if (constructorToUse == null) {
				if (causes != null) {
					UnsatisfiedDependencyException ex = causes.removeLast();
					for (Exception cause : causes) {
						this.beanFactory.onSuppressedException(cause);
					}
					throw ex;
				}
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Could not resolve matching constructor " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
			}
			else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Ambiguous constructor matches found in bean '" + beanName + "' " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
						ambiguousConstructors);
			}

			if (explicitArgs == null) {
				argsHolderToUse.storeCache(mbd, constructorToUse);
			}
		}

		try {
//			bean初始化策略，如果容器中bean方法需要被覆盖，采用cglib创建子类
			final InstantiationStrategy strategy = beanFactory.getInstantiationStrategy();
			Object beanInstance;

			if (System.getSecurityManager() != null) {
				final Constructor<?> ctorToUse = constructorToUse;
				final Object[] argumentsToUse = argsToUse;
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
//						初始化bean
						strategy.instantiate(mbd, beanName, beanFactory, ctorToUse, argumentsToUse),
						beanFactory.getAccessControlContext());
			}
			else {
				beanInstance = strategy.instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
			}

			bw.setBeanInstance(beanInstance);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean instantiation via constructor failed", ex);
		}
	}
```
解析构造方法，如果允许获取非public的构造方法就获取构造方法，如果不允许获取非public的方法就获取public的方法，对获取的构造方法进行排序，public的参数多的排在前面，非public的参数少的排在后面，解析@ConstructorProperties直接，创建参数数组引用了工厂方法或构造方法，查询到默认的bean初始化策略进行初始化，默认是cglib的初始化策略，初始化就是调用public的构造方法进行初始化

返回方法org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance，如果不是自动注入就初始化bean，确定给bean的构造方法，解析SmartInstantiationAwareBeanPostProcessor，这个处理器指定构造器给这个bean，可以获得早期创建的bean，就是初始化了对象但是没设置属性值，这种对象是为了解决循环依赖设计的，如果BeanDefinition中是按构造方法自动注入就执行按构造方法自动注入逻辑，否则就初始化bean。

返回方法org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

```java
synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
//					执行在bean初始化之前对BeanDefinition的处理
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}
```
执行MergedBeanDefinitionPostProcessor，org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors的postProcessMergedBeanDefinition方法

```java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
//			MergedBeanDefinitionPostProcessor一般用来对BeanDefinition做元数据处理或者在初始化之前修改BeanDefinition
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
//				执行postProcessMergedBeanDefinition方法，merge BeanDefinition后的处理逻辑
				bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
			}
		}
	}
```
org.springframework.beans.factory.support.MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition方法是来对BeanDefinition进行修改或者在bean初始化之前对BeanDefinition做些处理，比如增加一些缓存的元数据。

```java
addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
//					查询bean的早期引用避免循环引用
					return AbstractAutowireCapableBeanFactory.this.getEarlyBeanReference(beanName, mbd, bean);
				}
			});
```
创建早期的bean引用，org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#getEarlyBeanReference

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
				}
			}
		}
		return exposedObject;
	}
```
从org.springframework.beans.factory.config.SmartInstantiationAwareBeanPostProcessor中获取早期的bean引用。org.springframework.beans.factory.config.SmartInstantiationAwareBeanPostProcessor接口的getEarlyBeanReference方法可有获取的一个早期的bean引用，这个早期的bean引用是spring为了解决循环依赖而存在的，只是初始化了bean并没设置属性。

```java
Object exposedObject = bean;
		try {
//			填充依赖注入的bean对象
			populateBean(beanName, mbd, instanceWrapper);
//			初始化bean并执行bean初始化的init方法和beanPostProcessor方法
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}
```
填充依赖注入的bean对象，初始化bean并执行bean的init方法和beanPostProcessor。

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean填充依赖注入的bean对象。

```java
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.跳过空实例的属性填充阶段。
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.//给任何实例化awarebeanpostprocessor修改的机会
//在属性设置之前bean的状态。
//支持字段注入的样式。
		boolean continueWithPropertyPopulation = true;

//		是否注册了初始化bean的BeanPostProcessors
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

//		不能属性依赖注入直接返回
		if (!continueWithPropertyPopulation) {
			return;
		}

		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

//		如果是按名称注入或者按类型注入
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.如果可以的话，在autowire的基础上添加属性值。
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
//				按名称注入
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.如果可以的话，按类型添加基于autowire的属性值。
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
//				按类型注入
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}

		if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
```

```java
if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
//					如果已经依赖注入成功了不允许再次注入
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}
```

解析InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation方法，bean初始化之后自动注入之前可以执行这个方法返回true，不可以执行这个方法也就是已经依赖注入过就不会执行这个方法。

```java
if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.如果可以的话，在autowire的基础上添加属性值。
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
//				按名称注入
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.如果可以的话，按类型添加基于autowire的属性值。
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
//				按类型注入
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}
```
按名称或按类型依赖注入。

按名称依赖注入，org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireByName

```java
protected void autowireByName(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
//			beanFactory中是否包含属性名的bean，如果包含从BeanFactory中获取bean
			if (containsBean(propertyName)) {
				Object bean = getBean(propertyName);
				pvs.add(propertyName, bean);
//				注册依赖的bean
				registerDependentBean(propertyName, beanName);
				if (logger.isDebugEnabled()) {
					logger.debug("Added autowiring by name from bean name '" + beanName +
							"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
							"' by name: no matching bean found");
				}
			}
		}
	}
```
找到BeanDefinition中的属性名，循环属属性名判断BeanFactory中是否包含beanName的BeanDefinition，如果包含就从BeanFactory中获取对象进行依赖注入。

org.springframework.beans.factory.support.AbstractBeanFactory#containsBean判断BeanFactory中是否包含指定名称的bean

```java
@Override
	public boolean containsBean(String name) {
		String beanName = transformedBeanName(name);
//		单例bean注册表中或BeanDefinition注册表中是否包含beanName
		if (containsSingleton(beanName) || containsBeanDefinition(beanName)) {
//			是否是factoryBean
			return (!BeanFactoryUtils.isFactoryDereference(name) || isFactoryBean(name));
		}
		// Not found -> check parent.
		BeanFactory parentBeanFactory = getParentBeanFactory();
		return (parentBeanFactory != null && parentBeanFactory.containsBean(originalBeanName(name)));
	}
```
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#registerDependentBean注册依赖的bean

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
返回方法org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean 

```java
if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
//				按类型注入
				autowireByType(beanName, mbd, bw, newPvs);
			}
```
按类型依赖注入bean。
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireByType找到bean的属性名解析依赖的bean并注册。

org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveDependency解析依赖的bean

```java
	@Override
	@Nullable
	public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

		descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
//		查询构造参数类型
		if (Optional.class == descriptor.getDependencyType()) {
//			为指定的依赖项创建一个包装器
			return createOptionalDependency(descriptor, requestingBeanName);
		}
		else if (ObjectFactory.class == descriptor.getDependencyType() ||
				ObjectProvider.class == descriptor.getDependencyType()) {
			return new DependencyObjectProvider(descriptor, requestingBeanName);
		}
		else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
			return new Jsr330ProviderFactory().createDependencyProvider(descriptor, requestingBeanName);
		}
		else {
//			返回延迟加载的代理对象
			Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
					descriptor, requestingBeanName);
			if (result == null) {
				result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
			}
			return result;
		}
	}
```
如果是Optional类型的依赖就创建Optional类型的依赖，否则就直接解析依赖项。

org.springframework.beans.factory.support.DefaultListableBeanFactory#createOptionalDependency创建Optional类型的依赖项

```java
private Optional<?> createOptionalDependency(
			DependencyDescriptor descriptor, @Nullable String beanName, final Object... args) {

		DependencyDescriptor descriptorToUse = new NestedDependencyDescriptor(descriptor) {
			@Override
			public boolean isRequired() {
				return false;
			}
//			BeanFactory创建对象
			@Override
			public Object resolveCandidate(String beanName, Class<?> requiredType, BeanFactory beanFactory) {
				return (!ObjectUtils.isEmpty(args) ? beanFactory.getBean(beanName, args) :
						super.resolveCandidate(beanName, requiredType, beanFactory));
			}
		};
//		解析依赖
		return Optional.ofNullable(doResolveDependency(descriptorToUse, beanName, null, null));
	}
```
从factoryBean中获取bean之后包装成Optional。

org.springframework.beans.factory.support.DefaultListableBeanFactory#doResolveDependency解析依赖

```java
	@Nullable
	public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

//		获取注入点
		InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
		try {
			Object shortcut = descriptor.resolveShortcut(this);
			if (shortcut != null) {
				return shortcut;
			}

			Class<?> type = descriptor.getDependencyType();
//			默认值没有依赖
			Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
			if (value != null) {
				if (value instanceof String) {
					String strVal = resolveEmbeddedValue((String) value);
					BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
//					表达式解析，这里是支持表达式配置的bean用表达式解析器进行解析
					value = evaluateBeanDefinitionString(strVal, bd);
				}
				TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
				return (descriptor.getField() != null ?
//						调用转换器进行转换
						converter.convertIfNecessary(value, type, descriptor.getField()) :
						converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
			}

//			解析多元化的bean注入 collection，map，array
			Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
			if (multipleBeans != null) {
				return multipleBeans;
			}

			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
			if (matchingBeans.isEmpty()) {
				if (isRequired(descriptor)) {
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				return null;
			}

			String autowiredBeanName;
			Object instanceCandidate;

//			如果按照类型找到的bean是多个
			if (matchingBeans.size() > 1) {
//				获取自动注入的beanName
				autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
				if (autowiredBeanName == null) {
//					找到不autowiredBeanName，报错没有唯一的bean
					if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
						return descriptor.resolveNotUnique(type, matchingBeans);
					}
					else {
						// In case of an optional Collection/Map, silently ignore a non-unique case:
						// possibly it was meant to be an empty collection of multiple regular beans
						// (before 4.3 in particular when we didn't even look for collection beans).
						return null;
					}
				}
				instanceCandidate = matchingBeans.get(autowiredBeanName);
			}
			else {
				// We have exactly one match.
				Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
				autowiredBeanName = entry.getKey();
				instanceCandidate = entry.getValue();
			}

			if (autowiredBeanNames != null) {
				autowiredBeanNames.add(autowiredBeanName);
			}
			if (instanceCandidate instanceof Class) {
				instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
			}
			Object result = instanceCandidate;
			if (result instanceof NullBean) {
				if (isRequired(descriptor)) {
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				result = null;
			}
			if (!ClassUtils.isAssignableValue(type, result)) {
				throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
			}
			return result;
		}
		finally {
			ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
		}
	}
```
如果依赖的bean是string类型的值，支持expression表达式，从factoryBean中解析对象，解析collection、map、array这种类型的bean，查找依赖注入的bean，如果按类型找到多个依赖注入的bean找到一个唯一的bean。

解析多个值的bean，org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveMultipleBeans

```java
	@Nullable
	private Object resolveMultipleBeans(DependencyDescriptor descriptor, @Nullable String beanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) {

		Class<?> type = descriptor.getDependencyType();
		if (type.isArray()) {
			Class<?> componentType = type.getComponentType();
			ResolvableType resolvableType = descriptor.getResolvableType();
			Class<?> resolvedArrayType = resolvableType.resolve();
			if (resolvedArrayType != null && resolvedArrayType != type) {
				type = resolvedArrayType;
				componentType = resolvableType.getComponentType().resolve();
			}
			if (componentType == null) {
				return null;
			}
//			查找自动注入的bean实例
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, componentType,
					new MultiElementDescriptor(descriptor));
			if (matchingBeans.isEmpty()) {
				return null;
			}
			if (autowiredBeanNames != null) {
				autowiredBeanNames.addAll(matchingBeans.keySet());
			}
			TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
			Object result = converter.convertIfNecessary(matchingBeans.values(), type);
			if (getDependencyComparator() != null && result instanceof Object[]) {
				Arrays.sort((Object[]) result, adaptDependencyComparator(matchingBeans));
			}
			return result;
		}
		else if (Collection.class.isAssignableFrom(type) && type.isInterface()) {
			Class<?> elementType = descriptor.getResolvableType().asCollection().resolveGeneric();
			if (elementType == null) {
				return null;
			}
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, elementType,
					new MultiElementDescriptor(descriptor));
			if (matchingBeans.isEmpty()) {
				return null;
			}
			if (autowiredBeanNames != null) {
				autowiredBeanNames.addAll(matchingBeans.keySet());
			}
			TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
			Object result = converter.convertIfNecessary(matchingBeans.values(), type);
			if (getDependencyComparator() != null && result instanceof List) {
				((List<?>) result).sort(adaptDependencyComparator(matchingBeans));
			}
			return result;
		}
		else if (Map.class == type) {
			ResolvableType mapType = descriptor.getResolvableType().asMap();
			Class<?> keyType = mapType.resolveGeneric(0);
			if (String.class != keyType) {
				return null;
			}
			Class<?> valueType = mapType.resolveGeneric(1);
			if (valueType == null) {
				return null;
			}
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, valueType,
					new MultiElementDescriptor(descriptor));
			if (matchingBeans.isEmpty()) {
				return null;
			}
			if (autowiredBeanNames != null) {
				autowiredBeanNames.addAll(matchingBeans.keySet());
			}
			return matchingBeans;
		}
		else {
			return null;
		}
	}
```
如果是array、collection、map类型按beanName和类型查找依赖的bean，org.springframework.beans.factory.support.DefaultListableBeanFactory#findAutowireCandidates查找指定类型的bean实例

```java
	protected Map<String, Object> findAutowireCandidates(
			@Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {

		String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
				this, requiredType, true, descriptor.isEager());
		Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);
		for (Class<?> autowiringType : this.resolvableDependencies.keySet()) {
			if (autowiringType.isAssignableFrom(requiredType)) {
				Object autowiringValue = this.resolvableDependencies.get(autowiringType);
//				jdk动态代理获取自动注入的对象
				autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
				if (requiredType.isInstance(autowiringValue)) {
					result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
					break;
				}
			}
		}
//		遍历候选的beanName
		for (String candidate : candidateNames) {
//			候选的引用不是自己又是自动装配的候选引用
			if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
				addCandidateEntry(result, candidate, descriptor, requiredType);
			}
		}
		if (result.isEmpty() && !indicatesMultipleBeans(requiredType)) {
			// Consider fallback matches if the first pass failed to find anything...如果第一次传球没有找到任何东西，可以考虑退场。
			DependencyDescriptor fallbackDescriptor = descriptor.forFallbackMatch();
			for (String candidate : candidateNames) {
				if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, fallbackDescriptor)) {
					addCandidateEntry(result, candidate, descriptor, requiredType);
				}
			}
			if (result.isEmpty()) {
				// Consider self references as a final pass...把自我推荐当作最终的通行证……
				// but in the case of a dependency collection, not the very same bean itself.但是在依赖项集合的情况下，不是非常相同的bean本身。
				for (String candidate : candidateNames) {
					if (isSelfReference(beanName, candidate) &&
							(!(descriptor instanceof MultiElementDescriptor) || !beanName.equals(candidate)) &&
							isAutowireCandidate(candidate, fallbackDescriptor)) {
						addCandidateEntry(result, candidate, descriptor, requiredType);
					}
				}
			}
		}
		return result;
	}
```
这个方法是按类型获取指定的bean实例，在为指定的bean依赖注入时掉用。按名称获取beanName集合，遍历按类型依赖注入bean的注册表resolvableDependencies找到bean，在按bean类型找到的beanName进行匹配。

```java
protected Map<String, Object> findAutowireCandidates(
			@Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {

//		按类型获取beanName集合
		String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
				this, requiredType, true, descriptor.isEager());
		Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);
		for (Class<?> autowiringType : this.resolvableDependencies.keySet()) {
			if (autowiringType.isAssignableFrom(requiredType)) {
//				按类型依赖注入的值保存在这里
				Object autowiringValue = this.resolvableDependencies.get(autowiringType);
//				jdk动态代理获取自动注入的对象
				autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
				if (requiredType.isInstance(autowiringValue)) {
					result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
					break;
				}
			}
		}
//		遍历候选的beanName
		for (String candidate : candidateNames) {
//			候选的引用不是自己又是自动装配的候选引用
			if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
				addCandidateEntry(result, candidate, descriptor, requiredType);
			}
		}
		if (result.isEmpty() && !indicatesMultipleBeans(requiredType)) {
			// Consider fallback matches if the first pass failed to find anything...如果第一次传球没有找到任何东西，可以考虑退场。
			DependencyDescriptor fallbackDescriptor = descriptor.forFallbackMatch();
			for (String candidate : candidateNames) {
				if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, fallbackDescriptor)) {
					addCandidateEntry(result, candidate, descriptor, requiredType);
				}
			}
			if (result.isEmpty()) {
				// Consider self references as a final pass...把自我推荐当作最终的通行证……
				// but in the case of a dependency collection, not the very same bean itself.但是在依赖项集合的情况下，不是非常相同的bean本身。
				for (String candidate : candidateNames) {
					if (isSelfReference(beanName, candidate) &&
							(!(descriptor instanceof MultiElementDescriptor) || !beanName.equals(candidate)) &&
							isAutowireCandidate(candidate, fallbackDescriptor)) {
						addCandidateEntry(result, candidate, descriptor, requiredType);
					}
				}
			}
		}
		return result;
	}
```
org.springframework.beans.factory.BeanFactoryUtils#beanNamesForTypeIncludingAncestors(org.springframework.beans.factory.ListableBeanFactory, java.lang.Class<?>, boolean, boolean)按指定的类型获取到所有的beanName集合，org.springframework.beans.factory.support.DefaultListableBeanFactory#getBeanNamesForType(java.lang.Class<?>, boolean, boolean)

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
org.springframework.beans.factory.support.DefaultListableBeanFactory#doGetBeanNamesForType

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
遍历BeanDefinition名称注册表beanDefinitionNames，如果beanName在别名注册表aliasMap中不存在，获取当前BeanFactory中merge后的BeanDefinition，如果BeanDefinition不是抽象的且 允许提前初始化、配置的有beanClass、不是延迟加载，对应的factoryBean在单例bean注册表中不存在，如果指定的beanName不是factoryBean且在单例bean注册表中或者指定的beanName是单例bean找到，如果没找到单例bean但是是factoryBean也找到，最后查找手动注册的单例bean注册表manualSingletonNames，如果是factoryBean，是单例bean和指定的类型匹配返回，否则beanName和指定的类型匹配返回。

返回方法org.springframework.beans.factory.support.DefaultListableBeanFactory#findAutowireCandidates
按类型从resolvableDependencies 解析依赖bean注册表中查询依赖的bean value，解析依赖注入的value值，org.springframework.beans.factory.support.AutowireUtils#resolveAutowiringValue

```java
public static Object resolveAutowiringValue(Object autowiringValue, Class<?> requiredType) {
//		自动注入的值是objectFactory且自动注入的值不是必须的
		if (autowiringValue instanceof ObjectFactory && !requiredType.isInstance(autowiringValue)) {
//			自动注入的值转换成objectFactory
			ObjectFactory<?> factory = (ObjectFactory<?>) autowiringValue;
//			自动注入的值是接口
			if (autowiringValue instanceof Serializable && requiredType.isInterface()) {
//				采用jdk动态代理创建对象
				autowiringValue = Proxy.newProxyInstance(requiredType.getClassLoader(),
						new Class<?>[] {requiredType}, new ObjectFactoryDelegatingInvocationHandler(factory));
			}
			else {
				return factory.getObject();
			}
		}
		return autowiringValue;
	}
```
如果注入的bean不是指定类型的bean，但是是ObjectFactory类型，采用jdk动态代理调用objectFactory接口实现类的getObject方法创建对象。
org.springframework.beans.factory.support.AutowireUtils.ObjectFactoryDelegatingInvocationHandler

```java
private static class ObjectFactoryDelegatingInvocationHandler implements InvocationHandler, Serializable {

		private final ObjectFactory<?> objectFactory;

		public ObjectFactoryDelegatingInvocationHandler(ObjectFactory<?> objectFactory) {
			this.objectFactory = objectFactory;
		}

		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			String methodName = method.getName();
			if (methodName.equals("equals")) {
				// Only consider equal when proxies are identical.
				return (proxy == args[0]);
			}
			else if (methodName.equals("hashCode")) {
				// Use hashCode of proxy.
				return System.identityHashCode(proxy);
			}
			else if (methodName.equals("toString")) {
				return this.objectFactory.toString();
			}
			try {
				return method.invoke(this.objectFactory.getObject(), args);
			}
			catch (InvocationTargetException ex) {
				throw ex.getTargetException();
			}
		}
	}

}
```
这里是动态代理设计模式实现，动态代理的好处就是简化代码、程序解耦、提高代码扩展性。

返回方法org.springframework.beans.factory.support.DefaultListableBeanFactory#findAutowireCandidates下面逻辑

```java
for (String candidate : candidateNames) {
//			候选的引用不是自己又是自动装配的候选引用
			if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
				addCandidateEntry(result, candidate, descriptor, requiredType);
			}
		}
```
循环按执行类型找到的beanName集合，如果不是自己引用自己又是自动依赖注入的bean就添加依赖注入bean的注册表中。org.springframework.beans.factory.support.DefaultListableBeanFactory#isAutowireCandidate(java.lang.String, org.springframework.beans.factory.config.DependencyDescriptor) beanName是不是自动依赖注入的候选

```java
@Override
	public boolean isAutowireCandidate(String beanName, DependencyDescriptor descriptor)
			throws NoSuchBeanDefinitionException {

		return isAutowireCandidate(beanName, descriptor, getAutowireCandidateResolver());
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#isAutowireCandidate(java.lang.String, org.springframework.beans.factory.config.DependencyDescriptor, org.springframework.beans.factory.support.AutowireCandidateResolver)

```java
protected boolean isAutowireCandidate(String beanName, DependencyDescriptor descriptor, AutowireCandidateResolver resolver)
			throws NoSuchBeanDefinitionException {

		String beanDefinitionName = BeanFactoryUtils.transformedBeanName(beanName);
		if (containsBeanDefinition(beanDefinitionName)) {
//			判断该bean是否是自动注入的候选项
			return isAutowireCandidate(beanName, getMergedLocalBeanDefinition(beanDefinitionName), descriptor, resolver);
		}
		else if (containsSingleton(beanName)) {
			return isAutowireCandidate(beanName, new RootBeanDefinition(getType(beanName)), descriptor, resolver);
		}

		BeanFactory parent = getParentBeanFactory();
		if (parent instanceof DefaultListableBeanFactory) {
			// No bean definition found in this factory -> delegate to parent.
			return ((DefaultListableBeanFactory) parent).isAutowireCandidate(beanName, descriptor, resolver);
		}
		else if (parent instanceof ConfigurableListableBeanFactory) {
			// If no DefaultListableBeanFactory, can't pass the resolver along.
			return ((ConfigurableListableBeanFactory) parent).isAutowireCandidate(beanName, descriptor);
		}
		else {
			return true;
		}
	}
```
按指定的beanName找到在BeanFactory中的BeanDefinition名称，如果BeanDefinition名称在当前BeanFactory的beanDefinitionMap BeanDefinition注册表中，按BeanDefinitionName找到merge后的BeanDefinition，判断mergeBeanDefinition是否是自动依赖注入的候选。如果当前BeanFactory中不包含指定BeanDefinitionName的BeanDefinition，但是beanName在单例bean注册表中，获取beanName指定的类型构建RootBeanDefinition再次判断是否是自动依赖注入的类型，最后当前factoryBean中不存在BeanDefinition去parentBeanFactory中去判断。

org.springframework.beans.factory.support.DefaultListableBeanFactory#isAutowireCandidate(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, org.springframework.beans.factory.config.DependencyDescriptor, org.springframework.beans.factory.support.AutowireCandidateResolver)判断指定的benaDefinition并进行依赖注入。

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
		return resolver.isAutowireCandidate(
				new BeanDefinitionHolder(mbd, beanName, getAliases(beanDefinitionName)), descriptor);
	}
```
解析BeanDefinition指定的beanClass，如果beanDefiunition中配置了解析构造方法或工厂方法的开关就进行构造方法或者工厂方法解析。
org.springframework.beans.factory.support.ConstructorResolver#resolveFactoryMethodIfPossible 解析工厂方法

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
找到factoryBean的类型，org.springframework.beans.factory.support.AbstractBeanFactory#getType

```java
@Override
	@Nullable
	public Class<?> getType(String name) throws NoSuchBeanDefinitionException {
		String beanName = transformedBeanName(name);

		// Check manually registered singletons.检查手动注册的单例。
//		按beanName获取bean的单例对象
		Object beanInstance = getSingleton(beanName, false);
//		单例对象不为空，不是nullBean
		if (beanInstance != null && beanInstance.getClass() != NullBean.class) {
//			是factoryBean
			if (beanInstance instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
//				调用factoryBean.getObjectType()获取factoryBean的类型
				return getTypeForFactoryBean((FactoryBean<?>) beanInstance);
			}
			else {
//				如果不是factoryBean直接调用getClass方法
				return beanInstance.getClass();
			}
		}

		// No singleton instance found -> check bean definition.没有找到单例实例->检查bean定义。
//		如果在当级beanFactory中查找不到就去上级BeanFactory中查找
		BeanFactory parentBeanFactory = getParentBeanFactory();
//		找到了parentBeanFactory且当前BeanFactory中不包含BeanDefinition
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// No bean definition found in this factory -> delegate to parent.在这个工厂中没有找到bean定义——>委托给父类。
//			从parentBeanFactory中获取bean的类型
			return parentBeanFactory.getType(originalBeanName(name));
		}

//		获取指定beanName在当前BeanFactory中merge后的beanDefinition
		RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

		// Check decorated bean definition, if any: We assume it'll be easier
		// to determine the decorated bean's type than the proxy's type.//检查修饰过的bean定义(如果有的话):我们假设它会更简单
//来确定装饰bean的类型，而不是代理的类型。
		BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
//		找到了BeanDefinitionHolder
		if (dbd != null && !BeanFactoryUtils.isFactoryDereference(name)) {
//			获取MergedBeanDefinition
			RootBeanDefinition tbd = getMergedBeanDefinition(dbd.getBeanName(), dbd.getBeanDefinition(), mbd);
			Class<?> targetClass = predictBeanType(dbd.getBeanName(), tbd);
//			找到了bean类型并且不是factoryBean
			if (targetClass != null && !FactoryBean.class.isAssignableFrom(targetClass)) {
				return targetClass;
			}
		}

//		从BeanDefinition中解析指定beanName的beanClass
		Class<?> beanClass = predictBeanType(beanName, mbd);

		// Check bean class whether we're dealing with a FactoryBean.检查bean类是否正在处理FactoryBean。
//		再次判断beanClass!=null且不是factoryBean
		if (beanClass != null && FactoryBean.class.isAssignableFrom(beanClass)) {
			if (!BeanFactoryUtils.isFactoryDereference(name)) {
				// If it's a FactoryBean, we want to look at what it creates, not at the factory class.如果它是一个FactoryBean，我们希望查看它创建了什么，而不是在factory类中。
				return getTypeForFactoryBean(beanName, mbd);
			}
			else {
				return beanClass;
			}
		}
		else {
			return (!BeanFactoryUtils.isFactoryDereference(name) ? beanClass : null);
		}
	}
```
org.springframework.beans.factory.support.DefaultListableBeanFactory#isAutowireCandidate(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, org.springframework.beans.factory.config.DependencyDescriptor, org.springframework.beans.factory.support.AutowireCandidateResolver)返回方法

```java
return resolver.isAutowireCandidate(
				new BeanDefinitionHolder(mbd, beanName, getAliases(beanDefinitionName)), descriptor);
```
查询beanDefinitionName的别名构建BeanDefinitionHolder，判断BeanDefinition中是否设置了自动注入候选项的选项，org.springframework.beans.factory.support.AutowireCandidateResolver#isAutowireCandidate

```java
default boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
		return bdHolder.getBeanDefinition().isAutowireCandidate();
	}
```
返回方法org.springframework.beans.factory.support.DefaultListableBeanFactory#doResolveDependency

```java
//查询依赖注入的bean
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
			if (matchingBeans.isEmpty()) {
				if (isRequired(descriptor)) {
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				return null;
			}
```
按类型、beanName查询到依赖注入候选的bean后，判断是不是必须注入bean，如果是必须的，找到bead当前BeanFactory的单例bean注册表中查询单例bean，如果查询不到就从parentBeanFactory中的单例bean注册表中查找，如果还没查询到就报错。如果按类型找到了依赖注入的多个bean，按@Primary、@Priority确定唯一的一个bean，org.springframework.beans.factory.support.DefaultListableBeanFactory#determineAutowireCandidate

```java
@Nullable
	protected String determineAutowireCandidate(Map<String, Object> candidates, DependencyDescriptor descriptor) {
		Class<?> requiredType = descriptor.getDependencyType();
//		确定的@Primary的候选项
		String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
		if (primaryCandidate != null) {
			return primaryCandidate;
		}
//		确定@Priority的候选项，优先级底层是比较器实现，最低的值是做高的优先级
		String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
		if (priorityCandidate != null) {
			return priorityCandidate;
		}
		// Fallback
		for (Map.Entry<String, Object> entry : candidates.entrySet()) {
			String candidateName = entry.getKey();
			Object beanInstance = entry.getValue();
			if ((beanInstance != null && this.resolvableDependencies.containsValue(beanInstance)) ||
					matchesBeanName(candidateName, descriptor.getDependencyName())) {
				return candidateName;
			}
		}
		return null;
	}
```
如果按指定类型找到了多个bean，就去查询配置的@Primary、@Priority找到唯一的一个bean。如果没找到唯一的bean就报错NoUniqueBeanDefinitionException。如果找到就直接返回bean。如果按指定的类型找到了一个bean，按autowiredBeanName去beanF获取依赖注入的bean，在判断找到的bean，如果是NullBean，依赖的bean配置的是required的，按指定的类型继续去BeanFactory、parentBeanFactory中查询bean，如果找不到就报错NoSuchBeanDefinitionException。最后判断找到的bean和指定的type不一致就报错BeanNotOfRequiredTypeException。

```java
	else {
				// We have exactly one match.
				Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
				autowiredBeanName = entry.getKey();
				instanceCandidate = entry.getValue();
			}

			if (autowiredBeanNames != null) {
				autowiredBeanNames.add(autowiredBeanName);
			}
			if (instanceCandidate instanceof Class) {
				instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
			}
			Object result = instanceCandidate;
			if (result instanceof NullBean) {
				if (isRequired(descriptor)) {
//					去BeanFactory、parentBeanFactory中查询
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				result = null;
			}
//			找到的bean和指定的type不一致
			if (!ClassUtils.isAssignableValue(type, result)) {
				throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
			}
			return result;
```
返回方法org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveDependency

```java
else {
//			返回延迟加载的代理对象
			Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
					descriptor, requestingBeanName);
			if (result == null) {
				result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
			}
			return result;
		}
```
如果依赖注入的bean不是Optional类型，解析延迟加载的bean。

返回方法org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireByType

```java
for (String autowiredBeanName : autowiredBeanNames) {
//						注册依赖bean
						registerDependentBean(autowiredBeanName, beanName);
						if (logger.isDebugEnabled()) {
							logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
									propertyName + "' to bean named '" + autowiredBeanName + "'");
						}
					}
```
按找到的autowiredBeanNames进行依赖bean注册，org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#registerDependentBean

```java
	public void registerDependentBean(String beanName, String dependentBeanName) {
		// A quick check for an existing entry upfront, avoiding synchronization...预先快速检查现有条目，避免同步……
		String canonicalName = canonicalName(beanName);
//		按别名查找依赖的beanName集合
		Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
//		如果已经注册直接返回
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
返回方法org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean

```java
boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
```
如果注册了InstantiationAwareBeanPostProcessor，就执行InstantiationAwareBeanPostProcessor的postProcessPropertyValues方法，这个方法在bean依赖注入之前检查bean的合法性。

```java
if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
```
如果依赖注入的属性值不为空进行依赖注入,org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyPropertyValues 设置给定的属性值，如果是集合类型collection、array、map、set、list等类型直接注入，如果是referenceBean类型的bean去BeanFactory中查找bean用类型转换器转换后注入。

返回org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean方法

```java
exposedObject = initializeBean(beanName, exposedObject, mbd);
```
初始化bean并执行bean初始化的init方法和beanPostProcessor方法，org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
//				执行aware的方法
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
//			执行初始化之前的beanProccessors
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
//			执行初始化方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
//			执行bean初始化以后的beanProccessors
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```
初始化bean，执行aware方法，执行beanProcessor的postProcessBeforeInitialization，执行初始化方法，执行beanProcessor的postProcessAfterInitialization方法。
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeAwareMethods执行aware方法

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
```
执行org.springframework.beans.factory.BeanNameAware#setBeanName，设置beanName，在bean的属性设置之后，在init方法、InitializingBean.afterPropertiesSet()执行之前调用。

执行org.springframework.beans.factory.BeanFactoryAware#setBeanFactory，这个方法返回BeanFactory，在bean属性设置之后在InitializingBean.afterPropertiesSet()、init方法之前执行。

```java
Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
//			执行初始化之前的beanProccessors
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}
```
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization

```java
@Override
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
			Object current = beanProcessor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```
执行org.springframework.beans.factory.config.BeanPostProcessor#postProcessBeforeInitialization，在bean初始化之前执行，可以对bean进行设置，类似于InitializingBean接口的afterPropertiesSet方法或者bean的init方法，返回的是对bean进一步封装的bean。

```java
try {
//			执行初始化方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
```
执行init方法，org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeInitMethods

```java
	protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

//		是否实现了InitializingBean接口
		boolean isInitializingBean = (bean instanceof InitializingBean);
//		bean定义是否包含afterPropertiesSet这个方法
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
//						执行afterPropertiesSet方法
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null && bean.getClass() != NullBean.class) {
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
//				执行客户自定义的init方法
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
```
如果bean实现了org.springframework.beans.factory.InitializingBean接口，执行org.springframework.beans.factory.InitializingBean#afterPropertiesSet方法，设置BeanFactoryAware、ApplicationContextAware所有的bean属性之后被beanFactory执行。

如果bean不是NullBean，查询BeanDefinition的init方法，init方法名称不和afterPropertiesSet一样执行客户自定义的init方法，org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeCustomInitMethod

```java
protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd)
			throws Throwable {

//		从bean定义中获取初始化方法
		String initMethodName = mbd.getInitMethodName();
		Assert.state(initMethodName != null, "No init method set");
		final Method initMethod = (mbd.isNonPublicAccessAllowed() ?
				BeanUtils.findMethod(bean.getClass(), initMethodName) :
				ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));

		if (initMethod == null) {
			if (mbd.isEnforceInitMethod()) {
				throw new BeanDefinitionValidationException("Couldn't find an init method named '" +
						initMethodName + "' on bean with name '" + beanName + "'");
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("No default init method named '" + initMethodName +
							"' found on bean with name '" + beanName + "'");
				}
				// Ignore non-existent default lifecycle methods.
				return;
			}
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Invoking init method  '" + initMethodName + "' on bean with name '" + beanName + "'");
		}

		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				ReflectionUtils.makeAccessible(initMethod);
				return null;
			});
			try {
				AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
//						执行初始化方法
					initMethod.invoke(bean), getAccessControlContext());
			}
			catch (PrivilegedActionException pae) {
				InvocationTargetException ex = (InvocationTargetException) pae.getException();
				throw ex.getTargetException();
			}
		}
		else {
			try {
				ReflectionUtils.makeAccessible(initMethod);
				initMethod.invoke(bean);
			}
			catch (InvocationTargetException ex) {
				throw ex.getTargetException();
			}
		}
	}
```
反射执行bean自定义的init方法。
返回方法org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)

```java
if (mbd == null || !mbd.isSynthetic()) {
//			执行bean初始化以后的beanProccessors
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
```
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization

```java
@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
//		beanPostProccessor保存在BeanFactory的list中
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
//			执行postProcessAfterInitialization方法
			Object current = beanProcessor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```
执行org.springframework.beans.factory.config.BeanPostProcessor#postProcessAfterInitialization，bean初始化之后执行，这个方法将被FactoryBean对象执行和被FactoryBean创建的对象，这个回调函数也将被InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation这个方法调用。

返回org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

```java
if (earlySingletonExposure) {
//			查询早期的单例对象
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
//				初始化的bean没被包装又有依赖
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
//					获取依赖的bean
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}
```
如果早期的bean已经创建，按指定beanName从单例bean注册表中获取早期bean引用。判断经过beanPostProcessor处理之前的bean和beanPostProcessor处理之后的bean一致添加依赖的bean，最后注册可销毁的bean。

```java
try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
```
org.springframework.beans.factory.support.AbstractBeanFactory#registerDisposableBeanIfNecessary

```java
	protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
		AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
//		不是原型模式，可以销毁
		if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
//			如果是单例
			if (mbd.isSingleton()) {
				// Register a DisposableBean implementation that performs all destruction
				// work for the given bean: DestructionAwareBeanPostProcessors,
				// DisposableBean interface, custom destroy method.
//				注册销毁的bean，这一部分在bean定义解析的部分介绍过，这里不作介绍了
				registerDisposableBean(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
			else {
				// A bean with a custom scope...
				Scope scope = this.scopes.get(mbd.getScope());
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
				}
//				销毁后的回调方法
				scope.registerDestructionCallback(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
		}
	}
```
如果bean不是prototype且需要销毁。
org.springframework.beans.factory.support.AbstractBeanFactory#requiresDestruction

```java
protected boolean requiresDestruction(Object bean, RootBeanDefinition mbd) {
		return (bean.getClass() != NullBean.class &&
				(DisposableBeanAdapter.hasDestroyMethod(bean, mbd) || (hasDestructionAwareBeanPostProcessors() &&
						DisposableBeanAdapter.hasApplicableProcessors(bean, getBeanPostProcessors()))));
	}
```
bean不是NullBean类型的且bean有销毁的方法或者配置的有DestructionAwareBeanPostProcessor
org.springframework.beans.factory.support.DisposableBeanAdapter#hasDestroyMethod判断bean是否有销毁方法

```java
public static boolean hasDestroyMethod(Object bean, RootBeanDefinition beanDefinition) {
		if (bean instanceof DisposableBean || bean instanceof AutoCloseable) {
			return true;
		}
		String destroyMethodName = beanDefinition.getDestroyMethodName();
		if (AbstractBeanDefinition.INFER_METHOD.equals(destroyMethodName)) {
			return (ClassUtils.hasMethod(bean.getClass(), CLOSE_METHOD_NAME) ||
					ClassUtils.hasMethod(bean.getClass(), SHUTDOWN_METHOD_NAME));
		}
		return StringUtils.hasLength(destroyMethodName);
	}
```
如果bean实现了DisposableBean接口，spring销毁bean的时候回调用destroy方法，实现AutoCloseable接口会自动调用close方法，所以这里不用处理直接返回。获取BeanDefinition中的销毁方法。
org.springframework.beans.factory.support.DisposableBeanAdapter#hasApplicableProcessors

```java
public static boolean hasApplicableProcessors(Object bean, List<BeanPostProcessor> postProcessors) {
		if (!CollectionUtils.isEmpty(postProcessors)) {
			for (BeanPostProcessor processor : postProcessors) {
				if (processor instanceof DestructionAwareBeanPostProcessor) {
					DestructionAwareBeanPostProcessor dabpp = (DestructionAwareBeanPostProcessor) processor;
					if (dabpp.requiresDestruction(bean)) {
						return true;
					}
				}
			}
		}
		return false;
	}
```
执行org.springframework.beans.factory.config.DestructionAwareBeanPostProcessor#requiresDestruction方法，默认返回true。

如果bean是单例的，注册可以销毁的bean在disposableBeans注册表中。如果不是单例，按BeanDefinition的scope属性获取scopes列表中的值，执行org.springframework.beans.factory.config.Scope#registerDestructionCallback方法。

返回org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean

```java
if (mbd.isSingleton()) {
//					如果bean定义是单例，获取单例的bean
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
//							bean初始化
								return AbstractBeanFactory.this.createBean(beanName, mbd, args);
							} catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.//从单例缓存中显式删除实例:它可能已经被放在那里了
//急切被创建进程，以允许循环引用解析。
//还要删除接收到该bean的临时引用的所有bean。
								AbstractBeanFactory.this.destroySingleton(beanName);
								throw ex;
							}
						}
					});
```
如果创建单例的bean失败就销毁，org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroySingleton

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
从单例bean注册表中删除指定beanName的bean，删除disposableBeans可销毁的bean注册表中的bean，销毁bean，org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroyBean

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
删除dependentBeanMap依赖bean注册表中的指定beanName的dependentBeanName的bean，调用bean的org.springframework.beans.factory.DisposableBean#destroy方法，删除containedBeanMap包含bean的注册表中containedBeanName的bean，删除dependenciesForBeanMap依赖项列表指定beanName的依赖beanName集合。

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)查询单例对象

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
从singletonObjects单例bean注册表中获取指定beanName的bean，如果获取不到，调用org.springframework.beans.factory.ObjectFactory#getObject方法创建对象并添加singletonObjects单例bean注册表中。
org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance 从获取的bean中获取bean

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
如果bean是NullBean返回，如果bean不是FactoryBean返回，如果是factoryBean，从factoryBeanObjectCache缓存中找到指定beanName的bean，如果没找到从factoryBean中获取对象，org.springframework.beans.factory.support.FactoryBeanRegistrySupport#getObjectFromFactoryBean

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
如果factoryBean是单例且单例bean singletonObjects注册表中包含指定beanName的bean，从factoryBeanObjectCache缓存中找到指定beanName的bean，如果获取不到，调用factoryBean的getObject方法返回bean，如果还获取不到创建NullBean返回，再次从factoryBeanObjectCache缓存中获取指定beanName的bean，如果获取不到，需要后置处理，执行org.springframework.beans.factory.support.FactoryBeanRegistrySupport#postProcessObjectFromFactoryBean方法，从factoryBean中获取bean以后执行的方法，默认返回bean不做任何处理，子类可以覆盖这个方法，把创建的bean注册到factoryBeanObjectCache中。

如果factoryBean不是单例且单例bean singletonObjects注册表中不包含指定beanName的bean直接调用factoryBean的getObject方法获取bean，执行org.springframework.beans.factory.support.FactoryBeanRegistrySupport#postProcessObjectFromFactoryBean方法。

org.springframework.beans.factory.support.FactoryBeanRegistrySupport#doGetObjectFromFactoryBean从factoryBean创建对象，就是调用factoryBean的getObject方法，如果获取不到就创建NullBean返回。

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
返回方法org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean

```java
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
```
如果BeanDefinition是prototype就创建原型的bean，从创建的bean中获取bean。

```java
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
```
按BeanDefinition中的scopeName从scopes列表中获取，从scope中获取bean，在从获取的bean中获取指定的bean。最后如果所需的bean类型和获得的bean类型不一致就调用类型转换器进行转换。
