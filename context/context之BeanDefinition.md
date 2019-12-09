# context之BeanDefinition

org.springframework.context.annotation.ComponentScans

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface ComponentScans {

	ComponentScan[] value();

}
```
可以配置多个org.springframework.context.annotation.ComponentScan
配置用于@Configuration类的组件扫描指令，和<context:component-scan>等价，可以指定basePackageClasses或basepackage来定义特定的包来扫描，如果没有定义特定的包，则会从声明该注释的类的包中进行扫描。

```java
@AliasFor("basePackages")
	String[] value() default {};
```
指定扫描的多个包名。

```java
@AliasFor("value")
	String[] basePackages() default {};
```
指定扫描的多个包名。

```java
Class<?>[] basePackageClasses() default {};
```
指定包含扫描包的多个类。

```java
Class<? extends ScopeMetadataResolver> scopeResolver() default AnnotationScopeMetadataResolver.class;
```
指定ScopeMetadataResolver，默认值AnnotationScopeMetadataResolver。

```java
ScopedProxyMode scopedProxy() default ScopedProxyMode.DEFAULT;
```
指定scope的默认代理模式，默认值NO

```java
String resourcePattern() default ClassPathScanningCandidateComponentProvider.DEFAULT_RESOURCE_PATTERN;
```
指定扫描的监测文件，默认文件名.class，建议使用includeFilters、excludeFilters。

```java
boolean useDefaultFilters() default true;
```
默认扫描带@Component、@Repository、@Service、@Controller注解的类。

```java
Filter[] includeFilters() default {};
```
指定哪些类型可用于组件扫描。

```java
Filter[] excludeFilters() default {};
```
指定哪些类型不适合组件扫描。

```java
boolean lazyInit() default false;
```
指定是否需要延迟初始化bean。

filter的类型支持以下几种

```java
ANNOTATION
```
注解类型。

```java
ASPECTJ
```
AspectJ类型模式表达式。

```java
REGEX
```
正则表达式模式。

```java
CUSTOM
```
自定义的类型。自定义类型可以实现org.springframework.core.type.filter.TypeFilter接口。

org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider
从扫描的包中查找BeanDefinition。

```java
static final String DEFAULT_RESOURCE_PATTERN = "**/*.class";
```
默认文件名。

```java
private final List<TypeFilter> includeFilters = new LinkedList<>();

	private final List<TypeFilter> excludeFilters = new LinkedList<>();
```
扫描包含和不包含的类型过滤器。

```java
public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters) {
		this(useDefaultFilters, new StandardEnvironment());
	}
```
如果指定useDefaultFilters=true，会自动扫描@Component、@Repository、@Service、@Controller注解。

```java
public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters, Environment environment) {
		if (useDefaultFilters) {
			registerDefaultFilters();
		}
		setEnvironment(environment);
		setResourceLoader(null);
	}
```
org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#registerDefaultFilters

```java
	protected void registerDefaultFilters() {
//		注册@Component类型的过滤器，只要带有这个注解的bean定义都会被扫描，也包括包装这个注解的注解
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
//			@ManagedBean也会被扫描
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
//			@Named也会被扫描
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```
注册默认扫描的注解@Component、@Repository、@Service、@Controller、@ManagedBean、@Named。

org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#findCandidateComponents

```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
			return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
		}
		else {
//			扫描候选组件
			return scanCandidateComponents(basePackage);
		}
	}
```
扫描候选组件。如果添支持索引添加了@Indexed注解从索引中添加候选组件，底层是map实现。如果没有就扫描候选组件。

org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#scanCandidateComponents

```java
	private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<>();
		try {
//			解析扫描的包路径，从classpath下面，包名支持占位符
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
			Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
						MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
//						根据配置的typeFilter判断是不是候选组件
						if (isCandidateComponent(metadataReader)) {
//							创建BeanDefinition
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							sbd.setResource(resource);
							sbd.setSource(resource);
//							如果BeanDefinition是候选组件，判断是否有带@Lookup注解的方法
							if (isCandidateComponent(sbd)) {
								if (debugEnabled) {
									logger.debug("Identified candidate component class: " + resource);
								}
								candidates.add(sbd);
							}
							else {
								if (debugEnabled) {
									logger.debug("Ignored because not a concrete top-level class: " + resource);
								}
							}
						}
						else {
							if (traceEnabled) {
								logger.trace("Ignored because not matching any filter: " + resource);
							}
						}
					}
					catch (Throwable ex) {
						throw new BeanDefinitionStoreException(
								"Failed to read candidate component class: " + resource, ex);
					}
				}
				else {
					if (traceEnabled) {
						logger.trace("Ignored because not readable: " + resource);
					}
				}
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
		}
		return candidates;
	}
```
根据指定的包名和typeFilter解析class文件并创建BeanDefinition。这里也用了元数据缓存做了优化。

org.springframework.context.annotation.ClassPathBeanDefinitionScanner
[@Component、](#)Repository、@Service、@Controller、@ManagedBean、@Named 注解扫描器。

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		this.registry = registry;

//		如果用默认的包扫描过滤器
		if (useDefaultFilters) {
//			注册默认的过滤器
			registerDefaultFilters();
		}
		setEnvironment(environment);
		setResourceLoader(resourceLoader);
	}
```
按指定对的BeanDefinitionRegistry和useDefaultFilters 使用使用默认的typeFilter创建ClassPathBeanDefinitionScanner。org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#registerDefaultFilters

```java
protected void registerDefaultFilters() {
//		注册@Component类型的过滤器，只要带有这个注解的bean定义都会被扫描，也包括包装这个注解的注解
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
//			@ManagedBean也会被扫描
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
//			@Named也会被扫描
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```

org.springframework.context.annotation.ClassPathBeanDefinitionScanner#scan

```java
public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

//		扫描BeanDefinition
		doScan(basePackages);

		// Register annotation config processors, if necessary.如果需要，注册注释配置处理器。
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
```
按指定的包扫描BeanDefinition，注册AnnotationConfigProcessors。

org.springframework.context.annotation.ClassPathBeanDefinitionScanner#doScan
这个方法前面介绍过了。

org.springframework.context.annotation.ClassPathBeanDefinitionScanner#postProcessBeanDefinition

```java
protected void postProcessBeanDefinition(AbstractBeanDefinition beanDefinition, String beanName) {
//		从beans标签配置中获取默认配置
		beanDefinition.applyDefaults(this.beanDefinitionDefaults);
		if (this.autowireCandidatePatterns != null) {
			beanDefinition.setAutowireCandidate(PatternMatchUtils.simpleMatch(this.autowireCandidatePatterns, beanName));
		}
	}
```
设置BeanDefinition的默认属性，是否延迟加载、自动注入模式、依赖检查、初始化方法、销毁方法等设置。

org.springframework.context.annotation.ClassPathBeanDefinitionScanner#registerBeanDefinition

```java
protected void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) {
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry);
	}
```
org.springframework.beans.factory.support.BeanDefinitionReaderUtils#registerBeanDefinition

```java
public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
//		注册bean定义
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.注册别名(如果有的话)。
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
//				注册别名
				registry.registerAlias(beanName, alias);
			}
		}
	}
```
注册BeanDefinition和别名。

org.springframework.context.annotation.ScopeMetadataResolver
解决bean定义范围的策略接口。

```java
ScopeMetadata resolveScopeMetadata(BeanDefinition definition);
```
解析BeanDefinition上方的scope元数据。

org.springframework.context.annotation.AnnotationScopeMetadataResolver
基于@Scope注解的scope元数据解析器。

```java
@Override
	public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
		ScopeMetadata metadata = new ScopeMetadata();
		if (definition instanceof AnnotatedBeanDefinition) {
			AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
//			解析@Scope
			AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
					annDef.getMetadata(), this.scopeAnnotationType);
			if (attributes != null) {
				metadata.setScopeName(attributes.getString("value"));
				ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
//				默认不开启代理
				if (proxyMode == ScopedProxyMode.DEFAULT) {
					proxyMode = this.defaultProxyMode;
				}
				metadata.setScopedProxyMode(proxyMode);
			}
		}
		return metadata;
	}
```
解析@Scope的元数据中value、proxyMode等属性值。

org.springframework.context.annotation.ScopedProxyMode
代理模式有以下几种

```java
DEFAULT
```
默认值NO

```java
NO
```
不需要创建代理，当使用非单实例作用域的实例时，这种代理模式通常不会有用，它应该支持使用接口或TARGET_CLASS代理模式。

```java
INTERFACES
```
创建一个JDK动态代理，实现由目标对象类公开的所有接口。

```java
TARGET_CLASS
```
创建基于类的代理(使用CGLIB)。

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader
根据一个给定的配置类，向给定的BeanDefinitionRegistry注册bean定义。

```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
		TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
		for (ConfigurationClass configClass : configurationModel) {
			loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
		}
	}
```
加载BeanDefinition到BeanDefinition注册表中。

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass

```java
	private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
			TrackedConditionEvaluator trackedConditionEvaluator) {

		if (trackedConditionEvaluator.shouldSkip(configClass)) {
			String beanName = configClass.getBeanName();
			if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
				this.registry.removeBeanDefinition(beanName);
			}
			this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
			return;
		}

		if (configClass.isImported()) {
//			将@Configuration的类本身注册为bean定义
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
//			根据bean的method加载bean定义
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}
		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}

```
从配置类加载BeanDefinition，判断配置类是不是引入的，如果是加载这个配置类的BeanDefinition，解析这个配置类上有@Bean注解的方法注册BeanDefinition，从引入的.groovy、.xml文件中加载BeanDefinition，从import BeanDefinition注册器中加载BeanDefinition。

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#registerBeanDefinitionForImportedConfigurationClass

```java
private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
//		获取配置类的元数据
		AnnotationMetadata metadata = configClass.getMetadata();
		AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);

//		解析scope的源数据
		ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
		configBeanDef.setScope(scopeMetadata.getScopeName());
//		生成beanName
		String configBeanName = this.importBeanNameGenerator.generateBeanName(configBeanDef, this.registry);
//		解析一般的bean定义属性
		AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata);

		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
//		bean定义注册
		this.registry.registerBeanDefinition(definitionHolder.getBeanName(), definitionHolder.getBeanDefinition());
		configClass.setBeanName(configBeanName);

		if (logger.isDebugEnabled()) {
			logger.debug("Registered bean definition for imported class '" + configBeanName + "'");
		}
	}
```
注册配置类的BeanDefinition。

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForBeanMethod

```java
	private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
		ConfigurationClass configClass = beanMethod.getConfigurationClass();
		MethodMetadata metadata = beanMethod.getMetadata();
		String methodName = metadata.getMethodName();

		// Do we need to mark the bean as skipped by its condition?是否需要根据bean的条件将其标记为跳过?
		if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
			configClass.skippedBeanMethods.add(methodName);
			return;
		}
		if (configClass.skippedBeanMethods.contains(methodName)) {
			return;
		}

//		从配置类方法源数据中获取@Bean的属性值
		AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);
		Assert.state(bean != null, "No @Bean annotation attributes");

		// Consider name and any aliases
		List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));
		String beanName = (!names.isEmpty() ? names.remove(0) : methodName);

		// Register aliases even when overridden 注册别名
		for (String alias : names) {
			this.registry.registerAlias(beanName, alias);
		}

		// Has this effectively been overridden before (e.g. via XML)? 这是否已经被有效地覆盖了(例如通过XML)?
		if (isOverriddenByExistingDefinition(beanMethod, beanName)) {
			if (beanName.equals(beanMethod.getConfigurationClass().getBeanName())) {
				throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),
						beanName, "Bean name derived from @Bean method '" + beanMethod.getMetadata().getMethodName() +
						"' clashes with bean name for containing configuration class; please make those names unique!");
			}
			return;
		}

		ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
		beanDef.setResource(configClass.getResource());
		beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));

		if (metadata.isStatic()) {
			// static @Bean method
			beanDef.setBeanClassName(configClass.getMetadata().getClassName());
			beanDef.setFactoryMethodName(methodName);
		}
		else {
			// instance @Bean method
			beanDef.setFactoryBeanName(configClass.getBeanName());
			beanDef.setUniqueFactoryMethodName(methodName);
		}
//		设置自动装配模式为构造方法装配
		beanDef.setAutowireMode(RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
//		默认Required属性值是true
		beanDef.setAttribute(RequiredAnnotationBeanPostProcessor.SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

//		解析一般的bean定义属性值
		AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);

//		解析autowire属性
		Autowire autowire = bean.getEnum("autowire");
		if (autowire.isAutowire()) {
			beanDef.setAutowireMode(autowire.value());
		}

//		解析initMethod初始化方法
		String initMethodName = bean.getString("initMethod");
		if (StringUtils.hasText(initMethodName)) {
			beanDef.setInitMethodName(initMethodName);
		}

//		解析destroyMethod方法
		String destroyMethodName = bean.getString("destroyMethod");
		beanDef.setDestroyMethodName(destroyMethodName);

		// Consider scoping 默认scope属性值是no
		ScopedProxyMode proxyMode = ScopedProxyMode.NO;
//		获取@Scope的值
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
		if (attributes != null) {
			beanDef.setScope(attributes.getString("value"));
			proxyMode = attributes.getEnum("proxyMode");
			if (proxyMode == ScopedProxyMode.DEFAULT) {
				proxyMode = ScopedProxyMode.NO;
			}
		}

		// Replace the original bean definition with the target one, if necessary 如果需要，将原始bean定义替换为目标函数。
		BeanDefinition beanDefToRegister = beanDef;
		if (proxyMode != ScopedProxyMode.NO) {
//			cglib动态代理
			BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
					new BeanDefinitionHolder(beanDef, beanName), this.registry,
					proxyMode == ScopedProxyMode.TARGET_CLASS);
			beanDefToRegister = new ConfigurationClassBeanDefinition(
					(RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);
		}

		if (logger.isDebugEnabled()) {
			logger.debug(String.format("Registering bean definition for @Bean method %s.%s()",
					configClass.getMetadata().getClassName(), beanName));
		}

//		注册bean定义
		this.registry.registerBeanDefinition(beanName, beanDefToRegister);
	}
```
从配置类带@Bean的方法加载BeanDefinition。

如果指定了多个beanName，注册别名。检是否需要覆盖xml的BeanDefinition。设置为这样的我方法为BeanDefinition的工厂方法。设置自动注入模式为构造方法注入。默认跳过required检查，解析init方法、derstory方法、proxyMode，最后注册BeanDefinition。

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsFromImportedResources

```java
	private void loadBeanDefinitionsFromImportedResources(
			Map<String, Class<? extends BeanDefinitionReader>> importedResources) {

		Map<Class<?>, BeanDefinitionReader> readerInstanceCache = new HashMap<>();

		for (Map.Entry<String, Class<? extends BeanDefinitionReader>> entry : importedResources.entrySet()) {
			String resource = entry.getKey();
			Class<? extends BeanDefinitionReader> readerClass = entry.getValue();

			// Default reader selection necessary?
			if (BeanDefinitionReader.class == readerClass) {
//				加载groovy脚本
				if (StringUtils.endsWithIgnoreCase(resource, ".groovy")) {
					// When clearly asking for Groovy, that's what they'll get...
					readerClass = GroovyBeanDefinitionReader.class;
				}
				else {
					// Primarily ".xml" files but for any other extension as well主要”。xml"文件，但任何其他扩展以及
					readerClass = XmlBeanDefinitionReader.class;
				}
			}

			BeanDefinitionReader reader = readerInstanceCache.get(readerClass);
			if (reader == null) {
				try {
					// Instantiate the specified BeanDefinitionReader
					reader = readerClass.getConstructor(BeanDefinitionRegistry.class).newInstance(this.registry);
					// Delegate the current ResourceLoader to it if possible
					if (reader instanceof AbstractBeanDefinitionReader) {
						AbstractBeanDefinitionReader abdr = ((AbstractBeanDefinitionReader) reader);
						abdr.setResourceLoader(this.resourceLoader);
						abdr.setEnvironment(this.environment);
					}
					readerInstanceCache.put(readerClass, reader);
				}
				catch (Throwable ex) {
					throw new IllegalStateException(
							"Could not instantiate BeanDefinitionReader class [" + readerClass.getName() + "]");
				}
			}

			// TODO SPR-6310: qualify relative path locations as done in AbstractContextLoader.modifyLocations 加载bean定义
			reader.loadBeanDefinitions(resource);
		}
	}
```
从引入的.groovy文件和.xml文件采用不同的BeanDefinitionReader加载BeanDefinition。

org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(java.lang.String, java.util.Set<org.springframework.core.io.Resource>)

```java
	public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

//		从classpath下面加载
		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
//				递归调用
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
//			这里调用bean定义加载
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
	}
```
org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.Resource...)

```java
	@Override
	public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int counter = 0;
		for (Resource resource : resources) {
			counter += loadBeanDefinitions(resource);
		}
		return counter;
	}
```
org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.Resource) 这里以XmlBeanDefinitionReader为例，org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.support.EncodedResource)

```java
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

//		resourcesCurrentlyBeingLoaded threadLocal保证线程安全
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
//				加载bean定义
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
//				防止内存泄漏
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```
org.springframework.beans.factory.xml.XmlBeanDefinitionReader#doLoadBeanDefinitions

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
//			加载xml文档
			Document doc = doLoadDocument(inputSource, resource);
//			注册bean定义
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
```
org.springframework.beans.factory.xml.XmlBeanDefinitionReader#registerBeanDefinitions

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
//		创建bean定义阅读器
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
//		注册bean定义之前先创建
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#registerBeanDefinitions

```java
@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
//		获取xml文档根节点
		Element root = doc.getDocumentElement();
//		从xml文件中解析bean定义
		doRegisterBeanDefinitions(root);
	}
```
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions

```java
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
//		任何嵌套的<bean >元素都会导致这个方法的递归。为了正确地传播和保存<bean >缺省属性-*属性，请跟踪当前(父类)委托，
// 它可能是空的。创建新的(子)委托，并将其引用给父进程，以进行回退，然后将其重置为它的原始(父)引用。这种行为模拟了一组委托，而实际上并没有必要。
//		加载bean定义
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

//		如果根节点是命名空间 http://www.springframework.org/schema/beans
		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
//				解析环境配置文件
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

//		这里是spring的一个扩展点，在解析bean定义之前做点什么
		preProcessXml(root);
//		bean定义解析
		parseBeanDefinitions(root, this.delegate);
//		这里也是spring的一个扩展点，在bean定义解析之后可以做点什么
		postProcessXml(root);

		this.delegate = parent;
	}
```
解析_<beans/>、<bean />节点的BeanDefinition，这里实现了_preProcessXml、postProcessXml xml文件解析之前和之后的钩子方法。

这里创建了一个BeanDefinitionParserDelegate对象，创建的过程中初始化了默认的BeanDefinition配置。

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#createDelegate

```java
protected BeanDefinitionParserDelegate createDelegate(
			XmlReaderContext readerContext, Element root, @Nullable BeanDefinitionParserDelegate parentDelegate) {
//		bean定义解析委托
		BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
//		bean定义默认配置
		delegate.initDefaults(root, parentDelegate);
		return delegate;
	}
```
创建BeanDefinitionParserDelegate对象。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#initDefaults(org.w3c.dom.Element, org.springframework.beans.factory.xml.BeanDefinitionParserDelegate) 初始化默认的BeanDefinition配置。

```java
public void initDefaults(Element root, @Nullable BeanDefinitionParserDelegate parent) {
//		解析默认的bean定义解析配置
		populateDefaults(this.defaults, (parent != null ? parent.defaults : null), root);
//		触发默认的bean定义事件
		this.readerContext.fireDefaultsRegistered(this.defaults);
	}
```
org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#populateDefaults

```java
	protected void populateDefaults(DocumentDefaultsDefinition defaults, @Nullable DocumentDefaultsDefinition parentDefaults, Element root) {
//		解析default标签值 是否是default-lazy-init，默认值是false，父配置文件的属性会覆盖子配置文件的属性
		String lazyInit = root.getAttribute(DEFAULT_LAZY_INIT_ATTRIBUTE);
		if (DEFAULT_VALUE.equals(lazyInit)) {
			// Potentially inherited from outer <beans> sections, otherwise falling back to false.
			lazyInit = (parentDefaults != null ? parentDefaults.getLazyInit() : FALSE_VALUE);
		}
		defaults.setLazyInit(lazyInit);

//		default-merge 是否是这个值，默认值是false
		String merge = root.getAttribute(DEFAULT_MERGE_ATTRIBUTE);
		if (DEFAULT_VALUE.equals(merge)) {
			// Potentially inherited from outer <beans> sections, otherwise falling back to false.
			merge = (parentDefaults != null ? parentDefaults.getMerge() : FALSE_VALUE);
		}
		defaults.setMerge(merge);

//		default-autowire 是否是这个值，默认值是no
		String autowire = root.getAttribute(DEFAULT_AUTOWIRE_ATTRIBUTE);
		if (DEFAULT_VALUE.equals(autowire)) {
			// Potentially inherited from outer <beans> sections, otherwise falling back to 'no'.
			autowire = (parentDefaults != null ? parentDefaults.getAutowire() : AUTOWIRE_NO_VALUE);
		}
		defaults.setAutowire(autowire);

//		default-autowire-candidate 是否最为候选bean被自动注入
		if (root.hasAttribute(DEFAULT_AUTOWIRE_CANDIDATES_ATTRIBUTE)) {
			defaults.setAutowireCandidates(root.getAttribute(DEFAULT_AUTOWIRE_CANDIDATES_ATTRIBUTE));
		}
		else if (parentDefaults != null) {
			defaults.setAutowireCandidates(parentDefaults.getAutowireCandidates());
		}

//		default-init-method
		if (root.hasAttribute(DEFAULT_INIT_METHOD_ATTRIBUTE)) {
			defaults.setInitMethod(root.getAttribute(DEFAULT_INIT_METHOD_ATTRIBUTE));
		}
		else if (parentDefaults != null) {
			defaults.setInitMethod(parentDefaults.getInitMethod());
		}

//		default-destroy-method
		if (root.hasAttribute(DEFAULT_DESTROY_METHOD_ATTRIBUTE)) {
			defaults.setDestroyMethod(root.getAttribute(DEFAULT_DESTROY_METHOD_ATTRIBUTE));
		}
		else if (parentDefaults != null) {
			defaults.setDestroyMethod(parentDefaults.getDestroyMethod());
		}

		defaults.setSource(this.readerContext.extractSource(root));
	}

```
注册BeanDefinition默认配置。

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseBeanDefinitions

```java
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
//		根节点是否是命名空间 http://www.springframework.org/schema/beans
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```
BeanDefinition解析。

如果是默认的命名空间[http://www.springframework.org/schema/beans](http://www.springframework.org/schema/beans)，命名空间下的子节点是Element节点，解析默认的Element如果不是解析自定义的Element，如果不是默认的命名空间也解析自定义的Element。

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseDefaultElement

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
//		import 标签解析
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
//		alias
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
//		bean
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// 内部beans标签
			doRegisterBeanDefinitions(ele);
		}
	}
```
解析默认的节点。

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#importBeanDefinitionResource

```java
protected void importBeanDefinitionResource(Element ele) {
//		resource 解析属性
		String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
		if (!StringUtils.hasText(location)) {
			getReaderContext().error("Resource location must not be empty", ele);
			return;
		}

		// Resolve system properties: e.g. "${user.dir}" 占位符系统属性解析
		location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);

		Set<Resource> actualResources = new LinkedHashSet<>(4);

		// Discover whether the location is an absolute or relative URI 判断路径是相对还是绝对
		boolean absoluteLocation = false;
		try {
			absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
		}
		catch (URISyntaxException ex) {
			// cannot convert to an URI, considering the location relative
			// unless it is the well-known Spring prefix "classpath*:"
		}

		// Absolute or relative?
		if (absoluteLocation) {
//			如果是绝地路径
			try {
				int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
				}
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error(
						"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
			}
		}
		else {
			// No URL -> considering resource location as relative to the current file.
			try {
				int importCount;
				Resource relativeResource = getReaderContext().getResource().createRelative(location);
				if (relativeResource.exists()) {
//					调用bean定义加载
					importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
					actualResources.add(relativeResource);
				}
				else {
					String baseLocation = getReaderContext().getResource().getURL().toString();
//					调用bean定义加载
					importCount = getReaderContext().getReader().loadBeanDefinitions(
							StringUtils.applyRelativePath(baseLocation, location), actualResources);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
				}
			}
			catch (IOException ex) {
				getReaderContext().error("Failed to resolve current resource location", ele, ex);
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
						ele, ex);
			}
		}
		Resource[] actResArray = actualResources.toArray(new Resource[0]);
//		触发import bean定义解析完毕
		getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
	}
```

找到import的xml资源递归调用org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(java.lang.String, java.util.Set<org.springframework.core.io.Resource>)方法加载BeanDefinition。

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#processAliasRegistration

```java
protected void processAliasRegistration(Element ele) {
//		获取name属性值
		String name = ele.getAttribute(NAME_ATTRIBUTE);
//		获取alias属性值
		String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
		boolean valid = true;
		if (!StringUtils.hasText(name)) {
			getReaderContext().error("Name must not be empty", ele);
			valid = false;
		}
		if (!StringUtils.hasText(alias)) {
			getReaderContext().error("Alias must not be empty", ele);
			valid = false;
		}
		if (valid) {
			try {
//				从xml读取上下文中获取bean定义注册器注册别名
				getReaderContext().getRegistry().registerAlias(name, alias);
			}
			catch (Exception ex) {
				getReaderContext().error("Failed to register alias '" + alias +
						"' for bean with name '" + name + "'", ele, ex);
			}
//			触发别名已注册事件
			getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
		}
	}
```
处理别名注册。

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#processBeanDefinition

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
//		解析<bean>节点
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```
处理<bean>节点的BeanDefinition。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionElement(org.w3c.dom.Element, org.springframework.beans.factory.config.BeanDefinition)

```java
	@Nullable
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
//		获取id属性值
		String id = ele.getAttribute(ID_ATTRIBUTE);
//		获取name属性值
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

		List<String> aliases = new ArrayList<>();
		if (StringUtils.hasLength(nameAttr)) {
//			name属性值可以是多个，用，分开，会当做别名进行注册
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

//		把id的值当做beanName
		String beanName = id;
//		如果id值没有，会把name属性值下标是0的当做beanName，并从别名map中删除这个名字
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isDebugEnabled()) {
				logger.debug("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
//			验证当前执行的beanName和alias没有在其他bean中使用
			checkNameUniqueness(beanName, aliases, ele);
		}

//		bean节点bean定义解析
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				try {
					if (containingBean != null) {
//						自动生成beanName
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
//						如果不是匿名bean
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}
```
解析<bean>元素。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionElement(org.w3c.dom.Element, java.lang.String, org.springframework.beans.factory.config.BeanDefinition)

```java
	@Nullable
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
//		解析class属性值，可以输入空格
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
//		解析parent属性值
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);

//			解析bean标签的属性
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
//			description 属性解析
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

//			解析键值对的标签 key value
			parseMetaElements(ele, bd);
//			bean覆盖解析
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
//			方法重写解析
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

//			解析构造参数节点
			parseConstructorArgElements(ele, bd);
//			解析property节点
			parsePropertyElements(ele, bd);
//			qualifier属性解析
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```
解析class属性、parent属性，创建beanDefinition，解析<bean>元素的属性值，解析lookup-method属性值，解析replaced-method属性值，解析构造函数constructor-arg属性值，解析property属性值，解析qualifier属性值。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#createBeanDefinition

```java
protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
			throws ClassNotFoundException {

//		如果指定类加载器会立即加载
		return BeanDefinitionReaderUtils.createBeanDefinition(
				parentName, className, this.readerContext.getBeanClassLoader());
	}
```

```java
public static AbstractBeanDefinition createBeanDefinition(
			@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {

		GenericBeanDefinition bd = new GenericBeanDefinition();
		bd.setParentName(parentName);
		if (className != null) {
			if (classLoader != null) {
				bd.setBeanClass(ClassUtils.forName(className, classLoader));
			}
			else {
				bd.setBeanClassName(className);
			}
		}
		return bd;
```
创建BeanDefinition，BeanDefinition之间有层级关系时用的是GenericBeanDefinition并不是RootBeanDefinition和ChildBeanDefinition。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionAttributes

```java
public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
			@Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
//加载属性值到bean定义中
//		singleton 1.0支持，高版本用scope代替
		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
			error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
		}
//		scope
		else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
			bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
		}
		else if (containingBean != null) {
			// Take default from containing bean in case of an inner bean definition.如果有内部bean，scope以内部bean的scope为准
			bd.setScope(containingBean.getScope());
		}

//		abstract 是否是抽象类
		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
		}

//		lazy-init
		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
//		如果值是default，beans标签属性值中获取
		if (DEFAULT_VALUE.equals(lazyInit)) {
			lazyInit = this.defaults.getLazyInit();
		}
		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

//		autowire
		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
		bd.setAutowireMode(getAutowireMode(autowire));

//		depends-on 依赖的bean会先加载，可以是多个用,分开
		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
		}
//		autowire-candidate
		String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
		if ("".equals(autowireCandidate) || DEFAULT_VALUE.equals(autowireCandidate)) {
			String candidatePattern = this.defaults.getAutowireCandidates();
			if (candidatePattern != null) {
				String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
				bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
			}
		}
		else {
			bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
		}

//		primary 如果有两个名称一致的bean，以这个为主
		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
		}

//		init-method
		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
			bd.setInitMethodName(initMethodName);
		}
		else if (this.defaults.getInitMethod() != null) {
//			如果beans标签指定了init方法，以这个为主
			bd.setInitMethodName(this.defaults.getInitMethod());
			bd.setEnforceInitMethod(false);
		}

//		destroy-method 如果beans标签制定了，以这个为主
		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
			bd.setDestroyMethodName(destroyMethodName);
		}
		else if (this.defaults.getDestroyMethod() != null) {
			bd.setDestroyMethodName(this.defaults.getDestroyMethod());
			bd.setEnforceDestroyMethod(false);
		}

//		factory-method
		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
		}
//		factory-bean
		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
		}

		return bd;
	}
```
解析<bean>元素的属性值。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseLookupOverrideSubElements

```java
public void parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
//			lookup-method
			if (isCandidateElement(node) && nodeNameEquals(node, LOOKUP_METHOD_ELEMENT)) {
				Element ele = (Element) node;
//				name属性
				String methodName = ele.getAttribute(NAME_ATTRIBUTE);
//				bean属性
				String beanRef = ele.getAttribute(BEAN_ELEMENT);
				LookupOverride override = new LookupOverride(methodName, beanRef);
				override.setSource(extractSource(ele));
				overrides.addOverride(override);
			}
		}
	}
```
解析lookup-method属性值。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseReplacedMethodSubElements

```java
public void parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
//			replaced-method
			if (isCandidateElement(node) && nodeNameEquals(node, REPLACED_METHOD_ELEMENT)) {
				Element replacedMethodEle = (Element) node;
//				name属性
				String name = replacedMethodEle.getAttribute(NAME_ATTRIBUTE);
//				replacer属性
				String callback = replacedMethodEle.getAttribute(REPLACER_ATTRIBUTE);
				ReplaceOverride replaceOverride = new ReplaceOverride(name, callback);
				// Look for arg-type match elements.
				List<Element> argTypeEles = DomUtils.getChildElementsByTagName(replacedMethodEle, ARG_TYPE_ELEMENT);
				for (Element argTypeEle : argTypeEles) {
					String match = argTypeEle.getAttribute(ARG_TYPE_MATCH_ATTRIBUTE);
					match = (StringUtils.hasText(match) ? match : DomUtils.getTextValue(argTypeEle));
					if (StringUtils.hasText(match)) {
						replaceOverride.addTypeIdentifier(match);
					}
				}
				replaceOverride.setSource(extractSource(replacedMethodEle));
				overrides.addOverride(replaceOverride);
			}
		}
	}
```
解析replaced-method属性值。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseConstructorArgElements

```java
public void parseConstructorArgElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
//			constructor-arg
			if (isCandidateElement(node) && nodeNameEquals(node, CONSTRUCTOR_ARG_ELEMENT)) {
				parseConstructorArgElement((Element) node, bd);
			}
		}
	}
```
解析constructor-arg属性值。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseConstructorArgElement

```java
	public void parseConstructorArgElement(Element ele, BeanDefinition bd) {
//		index属性
		String indexAttr = ele.getAttribute(INDEX_ATTRIBUTE);
//		type属性
		String typeAttr = ele.getAttribute(TYPE_ATTRIBUTE);
//		name属性
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		if (StringUtils.hasLength(indexAttr)) {
			try {
				int index = Integer.parseInt(indexAttr);
				if (index < 0) {
					error("'index' cannot be lower than 0", ele);
				}
				else {
					try {
						this.parseState.push(new ConstructorArgumentEntry(index));
						Object value = parsePropertyValue(ele, bd, null);
						ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
						if (StringUtils.hasLength(typeAttr)) {
							valueHolder.setType(typeAttr);
						}
						if (StringUtils.hasLength(nameAttr)) {
							valueHolder.setName(nameAttr);
						}
						valueHolder.setSource(extractSource(ele));
						if (bd.getConstructorArgumentValues().hasIndexedArgumentValue(index)) {
							error("Ambiguous constructor-arg entries for index " + index, ele);
						}
						else {
							bd.getConstructorArgumentValues().addIndexedArgumentValue(index, valueHolder);
						}
					}
					finally {
						this.parseState.pop();
					}
				}
			}
			catch (NumberFormatException ex) {
				error("Attribute 'index' of tag 'constructor-arg' must be an integer", ele);
			}
		}
		else {
			try {
				this.parseState.push(new ConstructorArgumentEntry());
				Object value = parsePropertyValue(ele, bd, null);
				ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
				if (StringUtils.hasLength(typeAttr)) {
					valueHolder.setType(typeAttr);
				}
				if (StringUtils.hasLength(nameAttr)) {
					valueHolder.setName(nameAttr);
				}
				valueHolder.setSource(extractSource(ele));
				bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
			}
			finally {
				this.parseState.pop();
			}
		}
	}
```

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parsePropertyElements

```java
public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
//			property节点
			if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
				parsePropertyElement((Element) node, bd);
			}
		}
	}
```
解析property属性值。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parsePropertyElement

```java
	public void parsePropertyElement(Element ele, BeanDefinition bd) {
//		name属性
		String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
		if (!StringUtils.hasLength(propertyName)) {
			error("Tag 'property' must have a 'name' attribute", ele);
			return;
		}
		this.parseState.push(new PropertyEntry(propertyName));
		try {
			if (bd.getPropertyValues().contains(propertyName)) {
				error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
				return;
			}
//			解析属性值
			Object val = parsePropertyValue(ele, bd, propertyName);
			PropertyValue pv = new PropertyValue(propertyName, val);
			parseMetaElements(ele, pv);
			pv.setSource(extractSource(ele));
			bd.getPropertyValues().addPropertyValue(pv);
		}
		finally {
			this.parseState.pop();
		}
	}
```

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseQualifierElements

```java
public void parseQualifierElements(Element beanEle, AbstractBeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
//			找到qualifier节点
			if (isCandidateElement(node) && nodeNameEquals(node, QUALIFIER_ELEMENT)) {
//				解析qualifier节点
				parseQualifierElement((Element) node, bd);
			}
		}
	}
```
解析qualifier属性值。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseQualifierElement

```java
	public void parseQualifierElement(Element ele, AbstractBeanDefinition bd) {
//		解析type属性
		String typeName = ele.getAttribute(TYPE_ATTRIBUTE);
		if (!StringUtils.hasLength(typeName)) {
			error("Tag 'qualifier' must have a 'type' attribute", ele);
			return;
		}
		this.parseState.push(new QualifierEntry(typeName));
		try {
			AutowireCandidateQualifier qualifier = new AutowireCandidateQualifier(typeName);
			qualifier.setSource(extractSource(ele));
//			解析value属性值
			String value = ele.getAttribute(VALUE_ATTRIBUTE);
			if (StringUtils.hasLength(value)) {
				qualifier.setAttribute(AutowireCandidateQualifier.VALUE_KEY, value);
			}
			NodeList nl = ele.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
//				属性值是attribute
				if (isCandidateElement(node) && nodeNameEquals(node, QUALIFIER_ATTRIBUTE_ELEMENT)) {
					Element attributeEle = (Element) node;
//					获得key值
					String attributeName = attributeEle.getAttribute(KEY_ATTRIBUTE);
//					获得value值
					String attributeValue = attributeEle.getAttribute(VALUE_ATTRIBUTE);
					if (StringUtils.hasLength(attributeName) && StringUtils.hasLength(attributeValue)) {
						BeanMetadataAttribute attribute = new BeanMetadataAttribute(attributeName, attributeValue);
						attribute.setSource(extractSource(attributeEle));
						qualifier.addMetadataAttribute(attribute);
					}
					else {
						error("Qualifier 'attribute' tag must have a 'name' and 'value'", attributeEle);
						return;
					}
				}
			}
			bd.addQualifier(qualifier);
		}
		finally {
			this.parseState.pop();
		}
	}
```

返回org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#processBeanDefinition

```java
if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
```
处理完BeanDefinition返回BeanDefinitionHolder，如果需要装饰解析后的BeanDefinition进行这个逻辑，如果不需要注册BeanDefinition。
 
org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#decorateBeanDefinitionIfRequired(org.w3c.dom.Element, org.springframework.beans.factory.config.BeanDefinitionHolder, org.springframework.beans.factory.config.BeanDefinition)

```java
	public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
			Element ele, BeanDefinitionHolder definitionHolder, @Nullable BeanDefinition containingBd) {

		BeanDefinitionHolder finalDefinition = definitionHolder;

		// Decorate based on custom attributes first.
		NamedNodeMap attributes = ele.getAttributes();
		for (int i = 0; i < attributes.getLength(); i++) {
			Node node = attributes.item(i);
//			解析必须要加载的bean定义
			finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
		}

		// Decorate based on custom nested elements.
		NodeList children = ele.getChildNodes();
		for (int i = 0; i < children.getLength(); i++) {
			Node node = children.item(i);
			if (node.getNodeType() == Node.ELEMENT_NODE) {
				finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
			}
		}
		return finalDefinition;
	}
```
判断对BeanDefinition是否需要进行装饰。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#decorateIfRequired

```java
	public BeanDefinitionHolder decorateIfRequired(
			Node node, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

		String namespaceUri = getNamespaceURI(node);
//		不是默认的命名空间 http://www.springframework.org/schema/beans
		if (namespaceUri != null && !isDefaultNamespace(namespaceUri)) {
//			解析NamespaceHandler
			NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
			if (handler != null) {
//				状态xml配置形式依赖注入的bean定义解析 ref元素
				BeanDefinitionHolder decorated =
						handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
				if (decorated != null) {
					return decorated;
				}
			}
			else if (namespaceUri.startsWith("http://www.springframework.org/")) {
				error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", node);
			}
			else {
				// A custom namespace, not to be handled by Spring - maybe "xml:...".
				if (logger.isDebugEnabled()) {
					logger.debug("No Spring NamespaceHandler found for XML schema namespace [" + namespaceUri + "]");
				}
			}
		}
		return originalDef;
	}
```
判断namespaceUri是不是默认的[http://www.springframework.org/schema/beans](http://www.springframework.org/schema/beans)调用NamespaceHandlerResolver对namespaceUri进行解析返回NamespaceHandler，如果解析到了NamespaceHandler就调用NamespaceHandler的decorate方法对BeanDefinition进行装饰。

org.springframework.beans.factory.xml.DefaultNamespaceHandlerResolver#resolve

```java
@Override
	@Nullable
	public NamespaceHandler resolve(String namespaceUri) {
//		获取handlerMappings，从META-INF/spring.handlers这个路径加载
		Map<String, Object> handlerMappings = getHandlerMappings();
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			String className = (String) handlerOrClassName;
			try {
				Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
//				handler必须是这个类型 NamespaceHandler
				if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
					throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
							"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
				}
//				创建namespaceHandler类
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
//				初始化namespaceHandler类
				namespaceHandler.init();
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}
			catch (ClassNotFoundException ex) {
				throw new FatalBeanException("Could not find NamespaceHandler class [" + className +
						"] for namespace [" + namespaceUri + "]", ex);
			}
			catch (LinkageError err) {
				throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" +
						className + "] for namespace [" + namespaceUri + "]", err);
			}
		}
	}
```
先获取handlerMappings，根据namespaceUri获取namespaceHandler，初始化namespaceHandler并调用init方法初始化。

返回org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#decorateIfRequired

```java
if (handler != null) {
//				状态xml配置形式依赖注入的bean定义解析 ref元素
				BeanDefinitionHolder decorated =
						handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
				if (decorated != null) {
					return decorated;
				}
			}
```
如果找到了namespaceHandler就调用decorate方法对BeanDefinition进行装饰。

org.springframework.beans.factory.xml.NamespaceHandlerSupport#decorate

```java
@Override
	@Nullable
	public BeanDefinitionHolder decorate(
			Node node, BeanDefinitionHolder definition, ParserContext parserContext) {

		BeanDefinitionDecorator decorator = findDecoratorForNode(node, parserContext);
		return (decorator != null ? decorator.decorate(node, definition, parserContext) : null);
	}
```
抽象类中实现，从本地缓存中找到BeanDefinitionDecorator并调用decorate方法对原来的BeanDefinition进行装饰。

org.springframework.beans.factory.xml.SimpleConstructorNamespaceHandler#decorate

```java
@Override
	public BeanDefinitionHolder decorate(Node node, BeanDefinitionHolder definition, ParserContext parserContext) {
		if (node instanceof Attr) {
			Attr attr = (Attr) node;
			String argName = StringUtils.trimWhitespace(parserContext.getDelegate().getLocalName(attr));
			String argValue = StringUtils.trimWhitespace(attr.getValue());

			ConstructorArgumentValues cvs = definition.getBeanDefinition().getConstructorArgumentValues();
			boolean ref = false;

			// handle -ref arguments
			if (argName.endsWith(REF_SUFFIX)) {
				ref = true;
				argName = argName.substring(0, argName.length() - REF_SUFFIX.length());
			}

			ValueHolder valueHolder = new ValueHolder(ref ? new RuntimeBeanReference(argValue) : argValue);
			valueHolder.setSource(parserContext.getReaderContext().extractSource(attr));

			// handle "escaped"/"_" arguments
			if (argName.startsWith(DELIMITER_PREFIX)) {
				String arg = argName.substring(1).trim();

				// fast default check
				if (!StringUtils.hasText(arg)) {
					cvs.addGenericArgumentValue(valueHolder);
				}
				// assume an index otherwise
				else {
					int index = -1;
					try {
						index = Integer.parseInt(arg);
					}
					catch (NumberFormatException ex) {
						parserContext.getReaderContext().error(
								"Constructor argument '" + argName + "' specifies an invalid integer", attr);
					}
					if (index < 0) {
						parserContext.getReaderContext().error(
								"Constructor argument '" + argName + "' specifies a negative index", attr);
					}

					if (cvs.hasIndexedArgumentValue(index)){
						parserContext.getReaderContext().error(
								"Constructor argument '" + argName + "' with index "+ index+" already defined using <constructor-arg>." +
								" Only one approach may be used per argument.", attr);
					}

					cvs.addIndexedArgumentValue(index, valueHolder);
				}
			}
			// no escaping -> ctr name
			else {
				String name = Conventions.attributeNameToPropertyName(argName);
				if (containsArgWithName(name, cvs)){
					parserContext.getReaderContext().error(
							"Constructor argument '" + argName + "' already defined using <constructor-arg>." +
							" Only one approach may be used per argument.", attr);
				}
				valueHolder.setName(Conventions.attributeNameToPropertyName(argName));
				cvs.addGenericArgumentValue(valueHolder);
			}
		}
		return definition;
	}
```
解析ref节点元素对BeanDefinition进行装饰。

org.springframework.beans.factory.xml.SimplePropertyNamespaceHandler#decorate

```java
	@Override
	public BeanDefinitionHolder decorate(Node node, BeanDefinitionHolder definition, ParserContext parserContext) {
		if (node instanceof Attr) {
			Attr attr = (Attr) node;
			String propertyName = parserContext.getDelegate().getLocalName(attr);
			String propertyValue = attr.getValue();
			MutablePropertyValues pvs = definition.getBeanDefinition().getPropertyValues();
			if (pvs.contains(propertyName)) {
				parserContext.getReaderContext().error("Property '" + propertyName + "' is already defined using " +
						"both <property> and inline syntax. Only one approach may be used per property.", attr);
			}
			if (propertyName.endsWith(REF_SUFFIX)) {
				propertyName = propertyName.substring(0, propertyName.length() - REF_SUFFIX.length());
				pvs.add(Conventions.attributeNameToPropertyName(propertyName), new RuntimeBeanReference(propertyValue));
			}
			else {
				pvs.add(Conventions.attributeNameToPropertyName(propertyName), propertyValue);
			}
		}
		return definition;
	}
```
解析ref节点对BeanDefinition进行装饰。


返回org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#processBeanDefinition

```java
if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.注册最后的修饰实例。
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
```
对装饰后的BeanDefinition进行注册。

返回org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseBeanDefinitions

```java
				}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
```
如果不是默认的spring的namespace就解析自定的节点。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseCustomElement(org.w3c.dom.Element, org.springframework.beans.factory.config.BeanDefinition)

```java
	@Nullable
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
//		获取解析命名空间的handler
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
//		handler解析
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```
根据namespaceUri找到NamespaceHandler调用parse方法进行beBeanDefinition解析。

org.springframework.beans.factory.xml.NamespaceHandlerSupport#parse

```java
@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
//		获得bean定义解析器
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		return (parser != null ? parser.parse(element, parserContext) : null);
	}
```
根据节点名称从本地缓存parsers map中找到BeanDefinitionParser调用parse方法进行BeanDefinition解析。这里模板方法实现，先调用父类org.springframework.beans.factory.xml.AbstractBeanDefinitionParser#parse

```java
	@Override
	@Nullable
	public final BeanDefinition parse(Element element, ParserContext parserContext) {
//		解析bean定义的模板方法
		AbstractBeanDefinition definition = parseInternal(element, parserContext);
		if (definition != null && !parserContext.isNested()) {
			try {
//				解析id属性
				String id = resolveId(element, definition, parserContext);
				if (!StringUtils.hasText(id)) {
					parserContext.getReaderContext().error(
							"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
									+ "' when used as a top-level tag", element);
				}
				String[] aliases = null;
				if (shouldParseNameAsAliases()) {
//					解析name属性
					String name = element.getAttribute(NAME_ATTRIBUTE);
					if (StringUtils.hasLength(name)) {
						aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
					}
				}
//				创建BeanDefinitionHolder
				BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
//				从上下文获取beanDefinition并注册BeanDefinition
				registerBeanDefinition(holder, parserContext.getRegistry());
				if (shouldFireEvents()) {
//					创建BeanComponentDefinition
					BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
//					在解析完BeanDefinition后的钩子方法
					postProcessComponentDefinition(componentDefinition);
//					注册BeanDefinition
					parserContext.registerComponent(componentDefinition);
				}
			}
			catch (BeanDefinitionStoreException ex) {
				String msg = ex.getMessage();
				parserContext.getReaderContext().error((msg != null ? msg : ex.toString()), element);
				return null;
			}
		}
		return definition;
	}
```
调用子类的org.springframework.beans.factory.xml.AbstractBeanDefinitionParser#parseInternal模板方法，解析公共的id、name属性，解析完BeanDefinition后进行注册，处理org.springframework.beans.factory.xml.AbstractBeanDefinitionParser#postProcessComponentDefinition BeanDefinition注册后的钩子方法。

返回org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass

```java
loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
```
获取配置类的BeanDefinition注册器加载BeanDefinition。

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsFromRegistrars

```java
private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
		registrars.forEach(new BiConsumer<ImportBeanDefinitionRegistrar, AnnotationMetadata>() {
			@Override
			public void accept(ImportBeanDefinitionRegistrar registrar, AnnotationMetadata metadata) {
				registrar.registerBeanDefinitions(metadata, ConfigurationClassBeanDefinitionReader.this.registry);
			}
		});
	}
```
org.springframework.context.annotation.AspectJAutoProxyRegistrar#registerBeanDefinitions

```java
@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

//		获取EnableAspectJAutoProxy的属性值
		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
//		确认是采用proxy还是采用代理增强
		if (enableAspectJAutoProxy != null) {
			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}
```
[@EnableAspectJAutoProxy配置的benaDefinition注册。](#) 如果是proxyTargetClass cglib动态代理配置BeanDefinition，如果是exposeProxy 动态代理增强配置BeanDefinition。

org.springframework.context.annotation.AutoProxyRegistrar#registerBeanDefinitions

```java
@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		boolean candidateFound = false;
//		获取注解元数据中的属性值
		Set<String> annoTypes = importingClassMetadata.getAnnotationTypes();
		for (String annoType : annoTypes) {
			AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
			if (candidate == null) {
				continue;
			}
			Object mode = candidate.get("mode");
//			获取proxyTargetClass属性值
			Object proxyTargetClass = candidate.get("proxyTargetClass");
			if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
					Boolean.class == proxyTargetClass.getClass()) {
				candidateFound = true;
				if (mode == AdviceMode.PROXY) {
//					注册自动代理的BeanDefinition
					AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
					if ((Boolean) proxyTargetClass) {
//						注册cglib自动杆代理的BeanDefinition
						AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
						return;
					}
				}
			}
		}
		if (!candidateFound) {
			String name = getClass().getSimpleName();
			logger.warn(String.format("%s was imported but no annotations were found " +
					"having both 'mode' and 'proxyTargetClass' attributes of type " +
					"AdviceMode and boolean respectively. This means that auto proxy " +
					"creator registration and configuration may not have occurred as " +
					"intended, and components may not be proxied as expected. Check to " +
					"ensure that %s has been @Import'ed on the same class where these " +
					"annotations are declared; otherwise remove the import of %s " +
					"altogether.", name, name, name));
		}
	}
```
解析动态代理mode属性，如果是proxyTargetClass cgflib动态代理，配置BeanDefinition。

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#registerBeanDefinitionForImportedConfigurationClass

```java
private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
//		获取配置类的元数据
		AnnotationMetadata metadata = configClass.getMetadata();
		AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);

//		解析scope的源数据
		ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
		configBeanDef.setScope(scopeMetadata.getScopeName());
//		生成beanName
		String configBeanName = this.importBeanNameGenerator.generateBeanName(configBeanDef, this.registry);
//		解析一般的bean定义属性
		AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata);

		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
//		bean定义注册
		this.registry.registerBeanDefinition(definitionHolder.getBeanName(), definitionHolder.getBeanDefinition());
		configClass.setBeanName(configBeanName);

		if (logger.isDebugEnabled()) {
			logger.debug("Registered bean definition for imported class '" + configBeanName + "'");
		}
	}
```
将@Import的@Configuration 配置类本身注册为BeanDefinition。解析scope元数据、生成beanName、解析一般bean属性注解、解析scope类型BeanDefinition，注册BeanDefinition。

org.springframework.context.annotation.AnnotationScopeMetadataResolver#resolveScopeMetadata

```java
@Override
	public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
		ScopeMetadata metadata = new ScopeMetadata();
		if (definition instanceof AnnotatedBeanDefinition) {
			AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
//			解析@Scope
			AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
					annDef.getMetadata(), this.scopeAnnotationType);
			if (attributes != null) {
				metadata.setScopeName(attributes.getString("value"));
				ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
//				默认不开启代理
				if (proxyMode == ScopedProxyMode.DEFAULT) {
					proxyMode = this.defaultProxyMode;
				}
				metadata.setScopedProxyMode(proxyMode);
			}
		}
		return metadata;
	}
```
解析scope元数据。

org.springframework.context.annotation.AnnotationConfigUtils#processCommonDefinitionAnnotations(org.springframework.beans.factory.annotation.AnnotatedBeanDefinition, org.springframework.core.type.AnnotatedTypeMetadata)

```java
static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
//		解析lazy注解
		AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
		if (lazy != null) {
//			获取value的属性值是否开启延迟加载
			abd.setLazyInit(lazy.getBoolean("value"));
		}
		else if (abd.getMetadata() != metadata) {
			lazy = attributesFor(abd.getMetadata(), Lazy.class);
			if (lazy != null) {
				abd.setLazyInit(lazy.getBoolean("value"));
			}
		}

//		获取Primary注解值
		if (metadata.isAnnotated(Primary.class.getName())) {
			abd.setPrimary(true);
		}
//		解析DependsOn注解
		AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
		if (dependsOn != null) {
//			获取value属性值
			abd.setDependsOn(dependsOn.getStringArray("value"));
		}

		if (abd instanceof AbstractBeanDefinition) {
			AbstractBeanDefinition absBd = (AbstractBeanDefinition) abd;
//			解析Role注解
			AnnotationAttributes role = attributesFor(metadata, Role.class);
			if (role != null) {
//				获取value属性值
				absBd.setRole(role.getNumber("value").intValue());
			}
//			解析Description注解
			AnnotationAttributes description = attributesFor(metadata, Description.class);
			if (description != null) {
//				获取value属性值
				absBd.setDescription(description.getString("value"));
			}
		}
	}
```
处理一般注解的BeanDefinition。

org.springframework.context.annotation.AnnotationConfigUtils#applyScopedProxyMode

```java
static BeanDefinitionHolder applyScopedProxyMode(
			ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {

		ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
//		如果没有开启动态代理，直接返回bean定义
		if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
			return definition;
		}
//		如果是cglib动态代理
		boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
//		创建动态代理
		return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
	}
```
解析代理的BeanDefinition，如果是cglib创建cglib动态代理的BeanDefinition。

org.springframework.context.annotation.ScopedProxyCreator#createScopedProxy

```java
public static BeanDefinitionHolder createScopedProxy(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry, boolean proxyTargetClass) {

		return ScopedProxyUtils.createScopedProxy(definitionHolder, registry, proxyTargetClass);
	}
```
org.springframework.aop.scope.ScopedProxyUtils#createScopedProxy

```java
	public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition,
			BeanDefinitionRegistry registry, boolean proxyTargetClass) {

		String originalBeanName = definition.getBeanName();
		BeanDefinition targetDefinition = definition.getBeanDefinition();
//		代理bean的名称 scopedTarget.+原来beanName
		String targetBeanName = getTargetBeanName(originalBeanName);

		// Create a scoped proxy definition for the original bean name,
		// "hiding" the target bean in an internal target definition.//为原始bean名创建一个作用域代理定义，
//在内部目标定义中“隐藏”目标bean。
		RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
		proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
		proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
		proxyDefinition.setSource(definition.getSource());
		proxyDefinition.setRole(targetDefinition.getRole());

//		设置目标beanName
		proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
		if (proxyTargetClass) {
			targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			// ScopedProxyFactoryBean's "proxyTargetClass" default is TRUE, so we don't need to set it explicitly here.
		}
		else {
			proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
		}

		// Copy autowire settings from original bean definition.从原始bean定义复制自动装配设置。
		proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
		proxyDefinition.setPrimary(targetDefinition.isPrimary());
		if (targetDefinition instanceof AbstractBeanDefinition) {
			proxyDefinition.copyQualifiersFrom((AbstractBeanDefinition) targetDefinition);
		}

		// The target bean should be ignored in favor of the scoped proxy.应该忽略目标bean，而使用范围确定的代理。
		targetDefinition.setAutowireCandidate(false);
		targetDefinition.setPrimary(false);

		// Register the target bean as separate bean in the factory.将目标bean注入成beanFactory单独的bean
		registry.registerBeanDefinition(targetBeanName, targetDefinition);

		// Return the scoped proxy definition as primary bean definition
		// (potentially an inner bean).返回作用域代理定义作为主bean定义
//(可能是内部bean)。
		return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
	}

```
设置代理bean的名称，设置代理bean的自动注入BeanDefinition，注册BeanDefinition。

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForBeanMethod

```java
	private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
		ConfigurationClass configClass = beanMethod.getConfigurationClass();
		MethodMetadata metadata = beanMethod.getMetadata();
		String methodName = metadata.getMethodName();

		// Do we need to mark the bean as skipped by its condition?是否需要根据bean的条件将其标记为跳过?
		if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
			configClass.skippedBeanMethods.add(methodName);
			return;
		}
		if (configClass.skippedBeanMethods.contains(methodName)) {
			return;
		}

//		从配置类方法源数据中获取@Bean的属性值
		AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);
		Assert.state(bean != null, "No @Bean annotation attributes");

		// Consider name and any aliases
		List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));
		String beanName = (!names.isEmpty() ? names.remove(0) : methodName);

		// Register aliases even when overridden 注册别名
		for (String alias : names) {
			this.registry.registerAlias(beanName, alias);
		}

		// Has this effectively been overridden before (e.g. via XML)? 这是否已经被有效地覆盖了(例如通过XML)?
		if (isOverriddenByExistingDefinition(beanMethod, beanName)) {
			if (beanName.equals(beanMethod.getConfigurationClass().getBeanName())) {
				throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),
						beanName, "Bean name derived from @Bean method '" + beanMethod.getMetadata().getMethodName() +
						"' clashes with bean name for containing configuration class; please make those names unique!");
			}
			return;
		}

		ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
		beanDef.setResource(configClass.getResource());
		beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));

		if (metadata.isStatic()) {
			// static @Bean method
			beanDef.setBeanClassName(configClass.getMetadata().getClassName());
			beanDef.setFactoryMethodName(methodName);
		}
		else {
			// instance @Bean method
			beanDef.setFactoryBeanName(configClass.getBeanName());
			beanDef.setUniqueFactoryMethodName(methodName);
		}
//		设置自动装配模式为构造方法装配
		beanDef.setAutowireMode(RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
//		默认Required属性值是true
		beanDef.setAttribute(RequiredAnnotationBeanPostProcessor.SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

//		解析一般的bean定义属性值
		AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);

//		解析autowire属性
		Autowire autowire = bean.getEnum("autowire");
		if (autowire.isAutowire()) {
			beanDef.setAutowireMode(autowire.value());
		}

//		解析initMethod初始化方法
		String initMethodName = bean.getString("initMethod");
		if (StringUtils.hasText(initMethodName)) {
			beanDef.setInitMethodName(initMethodName);
		}

//		解析destroyMethod方法
		String destroyMethodName = bean.getString("destroyMethod");
		beanDef.setDestroyMethodName(destroyMethodName);

		// Consider scoping 默认scope属性值是no
		ScopedProxyMode proxyMode = ScopedProxyMode.NO;
//		获取@Scope的值
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
		if (attributes != null) {
			beanDef.setScope(attributes.getString("value"));
			proxyMode = attributes.getEnum("proxyMode");
			if (proxyMode == ScopedProxyMode.DEFAULT) {
				proxyMode = ScopedProxyMode.NO;
			}
		}

		// Replace the original bean definition with the target one, if necessary 如果需要，将原始bean定义替换为目标函数。
		BeanDefinition beanDefToRegister = beanDef;
		if (proxyMode != ScopedProxyMode.NO) {
//			cglib动态代理
			BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
					new BeanDefinitionHolder(beanDef, beanName), this.registry,
					proxyMode == ScopedProxyMode.TARGET_CLASS);
			beanDefToRegister = new ConfigurationClassBeanDefinition(
					(RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);
		}

		if (logger.isDebugEnabled()) {
			logger.debug(String.format("Registering bean definition for @Bean method %s.%s()",
					configClass.getMetadata().getClassName(), beanName));
		}

//		注册bean定义
		this.registry.registerBeanDefinition(beanName, beanDefToRegister);
	}

```
从配置的类bean方法加载@Bean的BeanDefinition并注册，获取@Bean注解的name属性，如果指定了多个注册别名，如果方法是静态的注册BeanDefinition的静态方法，设置自动注入配置默认构造参数自动注入，默认BeanDefinition属性设置跳过skipRequiredCheck，处理一般注解的BeanDefinition，解析自动注入设置、初始化方法、销毁方法、scope类型BeanDefinition，注册BeanDefinition。

org.springframework.context.annotation.AnnotationConfigUtils#processCommonDefinitionAnnotations(org.springframework.beans.factory.annotation.AnnotatedBeanDefinition, org.springframework.core.type.AnnotatedTypeMetadata)

```java
	static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
//		解析lazy注解
		AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
		if (lazy != null) {
//			获取value的属性值是否开启延迟加载
			abd.setLazyInit(lazy.getBoolean("value"));
		}
		else if (abd.getMetadata() != metadata) {
			lazy = attributesFor(abd.getMetadata(), Lazy.class);
			if (lazy != null) {
				abd.setLazyInit(lazy.getBoolean("value"));
			}
		}

//		获取Primary注解值
		if (metadata.isAnnotated(Primary.class.getName())) {
			abd.setPrimary(true);
		}
//		解析DependsOn注解
		AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
		if (dependsOn != null) {
//			获取value属性值
			abd.setDependsOn(dependsOn.getStringArray("value"));
		}

		if (abd instanceof AbstractBeanDefinition) {
			AbstractBeanDefinition absBd = (AbstractBeanDefinition) abd;
//			解析Role注解
			AnnotationAttributes role = attributesFor(metadata, Role.class);
			if (role != null) {
//				获取value属性值
				absBd.setRole(role.getNumber("value").intValue());
			}
//			解析Description注解
			AnnotationAttributes description = attributesFor(metadata, Description.class);
			if (description != null) {
//				获取value属性值
				absBd.setDescription(description.getString("value"));
			}
		}
	}
```
注册一般注解的BeanDefinition。

org.springframework.aop.scope.ScopedProxyUtils#createScopedProxy

```java
	public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition,
			BeanDefinitionRegistry registry, boolean proxyTargetClass) {

		String originalBeanName = definition.getBeanName();
		BeanDefinition targetDefinition = definition.getBeanDefinition();
//		代理bean的名称 scopedTarget.+原来beanName
		String targetBeanName = getTargetBeanName(originalBeanName);

		// Create a scoped proxy definition for the original bean name,
		// "hiding" the target bean in an internal target definition.//为原始bean名创建一个作用域代理定义，
//在内部目标定义中“隐藏”目标bean。
		RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
		proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
		proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
		proxyDefinition.setSource(definition.getSource());
		proxyDefinition.setRole(targetDefinition.getRole());

//		设置目标beanName
		proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
		if (proxyTargetClass) {
			targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			// ScopedProxyFactoryBean's "proxyTargetClass" default is TRUE, so we don't need to set it explicitly here.
		}
		else {
			proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
		}

		// Copy autowire settings from original bean definition.从原始bean定义复制自动装配设置。
		proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
		proxyDefinition.setPrimary(targetDefinition.isPrimary());
		if (targetDefinition instanceof AbstractBeanDefinition) {
			proxyDefinition.copyQualifiersFrom((AbstractBeanDefinition) targetDefinition);
		}

		// The target bean should be ignored in favor of the scoped proxy.应该忽略目标bean，而使用范围确定的代理。
		targetDefinition.setAutowireCandidate(false);
		targetDefinition.setPrimary(false);

		// Register the target bean as separate bean in the factory.将目标bean注入成beanFactory单独的bean
		registry.registerBeanDefinition(targetBeanName, targetDefinition);

		// Return the scoped proxy definition as primary bean definition
		// (potentially an inner bean).返回作用域代理定义作为主bean定义
//(可能是内部bean)。
		return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
	}
```
创建scope类型的BeanDefinition并注册。

org.springframework.context.annotation.CommonAnnotationBeanPostProcessor
处理一般注解的beanPostProcessor，支持@PostConstruct、@PreDestroy注解，继承了InitDestroyAnnotationBeanPostProcessor，和_context:annotation-config、context:component-scan xml标签一起使用。_
_
```java
private final transient Map<String, InjectionMetadata> injectionMetadataCache = new ConcurrentHashMap<>(256);
```
方法注入源数据缓存。

```java
public CommonAnnotationBeanPostProcessor() {
		setOrder(Ordered.LOWEST_PRECEDENCE - 3);
		setInitAnnotationType(PostConstruct.class);
		setDestroyAnnotationType(PreDestroy.class);
		ignoreResourceType("javax.xml.ws.WebServiceContext");
	}
```
设置初始化方法、销毁方法注解类型。

```java
@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
//		处理mergeBeanDefinition的处理器
		super.postProcessMergedBeanDefinition(beanDefinition, beanType, beanName);
		InjectionMetadata metadata = findResourceMetadata(beanName, beanType, null);
		metadata.checkConfigMembers(beanDefinition);
	}
```
beanDefinition修改后的处理方法，重写了org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#postProcessMergedBeanDefinition方法，这个方法是实现了org.springframework.beans.factory.support.MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition方法，先调用了父类查询init和destory的方法。

org.springframework.context.annotation.CommonAnnotationBeanPostProcessor#findResourceMetadata

```java
	private InjectionMetadata findResourceMetadata(String beanName, final Class<?> clazz, @Nullable PropertyValues pvs) {
		// Fall back to class name as cache key, for backwards compatibility with custom callers.返回类名作为缓存键，以便向后兼容自定义调用程序。
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
//					解析@Resource注解依赖注入的方法
					metadata = buildResourceMetadata(clazz);
					this.injectionMetadataCache.put(cacheKey, metadata);
				}
			}
		}
		return metadata;
	}
```
找到属性上有@Resource注解的属性，找到设置这个属性的方法，构造依赖注入的元数据。

```java
@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		return null;
	}
```
重写了org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation方法，没做任何处理。在实例化这个bean之前可以对这个bean操作。

```java
@Override
	public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		return true;
	}
```
重写了org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation方法，默认返回true。在bean进行构造方法或工厂方法实例化之后依赖注入属性值之前调用。

```java
@Override
	public PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

		InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass(), pvs);
		try {
			metadata.inject(bean, beanName, pvs);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
		}
		return pvs;
	}
```
重写了org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor#postProcessPropertyValues方法，在属性值进行依赖注入之前对属性值进行处理，比如这个属性值是够设置了必须进行依赖注入。找到@Resource注解的依赖注入元数据进行注入。
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
循环依赖注入节点进行注入。
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
执行属性的set方法进行进行属性设置。
org.springframework.context.annotation.CommonAnnotationBeanPostProcessor#buildLazyResourceProxy

```java
protected Object buildLazyResourceProxy(final LookupElement element, final @Nullable String requestingBeanName) {
		TargetSource ts = new TargetSource() {
			@Override
			public Class<?> getTargetClass() {
				return element.lookupType;
			}
			@Override
			public boolean isStatic() {
				return false;
			}
			@Override
			public Object getTarget() {
				return getResource(element, requestingBeanName);
			}
			@Override
			public void releaseTarget(Object target) {
			}
		};
		ProxyFactory pf = new ProxyFactory();
		pf.setTargetSource(ts);
		if (element.lookupType.isInterface()) {
			pf.addInterface(element.lookupType);
		}
		ClassLoader classLoader = (this.beanFactory instanceof ConfigurableBeanFactory ?
				((ConfigurableBeanFactory) this.beanFactory).getBeanClassLoader() : null);
		return pf.getProxy(classLoader);
	}
```
获取@Lazy的延迟加载对象。

org.springframework.context.annotation.CommonAnnotationBeanPostProcessor#getResource

```java
protected Object getResource(LookupElement element, @Nullable String requestingBeanName) throws BeansException {
		if (StringUtils.hasLength(element.mappedName)) {
			return this.jndiFactory.getBean(element.mappedName, element.lookupType);
		}
		if (this.alwaysUseJndiLookup) {
			return this.jndiFactory.getBean(element.name, element.lookupType);
		}
		if (this.resourceFactory == null) {
			throw new NoSuchBeanDefinitionException(element.lookupType,
					"No resource factory configured - specify the 'resourceFactory' property");
		}
//		从BeanFactory中找到@Resource注解的对象进行依赖注入
		return autowireResource(this.resourceFactory, element, requestingBeanName);
	}
```
从jndiFactory中@Resource的对象进行依赖注入。

org.springframework.context.annotation.AnnotationConfigBeanDefinitionParser
<context: annotationconfig /> 元素解析器。
org.springframework.context.annotation.AnnotationConfigBeanDefinitionParser#parse

```java
	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		Object source = parserContext.extractSource(element);

		// Obtain bean definitions for all relevant BeanPostProcessors.注册注解处理器
		Set<BeanDefinitionHolder> processorDefinitions =
				AnnotationConfigUtils.registerAnnotationConfigProcessors(parserContext.getRegistry(), source);

		// Register component for the surrounding <context:annotation-config> element.为周围的<context: annotationconfig >元素注册组件。
		CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
		parserContext.pushContainingComponent(compDefinition);

		// Nest the concrete beans in the surrounding component.将具体的bean嵌套在周围的组件中。
		for (BeanDefinitionHolder processorDefinition : processorDefinitions) {
			parserContext.registerComponent(new BeanComponentDefinition(processorDefinition));
		}

		// Finally register the composite component.最后注册复合组件。
		parserContext.popAndRegisterContainingComponent();

		return null;
	}
```
注册annotationConfig的解析器，解析BeanDefinition。
org.springframework.context.annotation.AnnotationConfigUtils#registerAnnotationConfigProcessors(org.springframework.beans.factory.support.BeanDefinitionRegistry, java.lang.Object)

```java
	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {

//		从BeanDefinitionRegistry中解析BeanFactory
		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
		if (beanFactory != null) {
			if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
//				设置依赖选项的比较器，默认比较@Order 顺序、@Priority 优先级
				beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
			}
//			设置默认的自动注入解析器ContextAnnotationAutowireCandidateResolver
			if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
				beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
			}
		}

		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(4);

//		BeanDefinition中不包含org.springframework.context.annotation.internalConfigurationAnnotationProcessor BeanDefinition
		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
//			注册org.springframework.context.annotation.internalConfigurationAnnotationProcessor beanDefinition
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

//		beanDefinition中不包含org.springframework.context.annotation.internalAutowiredAnnotationProcessor BeanDefinition
		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
//			注册org.springframework.context.annotation.internalAutowiredAnnotationProcessor BeanDefinition
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

//		BeanDefinition中不包含org.springframework.context.annotation.internalRequiredAnnotationProcessor BeanDefinition就注册
		if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
//		BeanDefinition中不包含org.springframework.context.annotation.internalCommonAnnotationProcessor就注册
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
//		BeanDefinition中不包含org.springframework.context.annotation.internalPersistenceAnnotationProcessor就注册
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

//		BeanDefinition中不包含org.springframework.context.event.internalEventListenerProcessor就注册
		if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
		}
//		BeanDefinition中不包含org.springframework.context.event.internalEventListenerFactory就注册
		if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
		}

		return beanDefs;
	}
```
注册annotationConfig的解析器。
