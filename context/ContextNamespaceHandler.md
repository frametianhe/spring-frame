# ContextNamespaceHandler

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1575798407797-f41e63a4-0d53-4f7c-b695-473f377d2fbd.png#align=left&display=inline&height=550&name=image.png&originHeight=550&originWidth=951&size=70773&status=done&style=none&width=951)

org.springframework.context.config.ContextNamespaceHandler

```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
		registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
		registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
		registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
		registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
		registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
	}

}
```
spring自己实现的namespaceHandler继承了org.springframework.beans.factory.xml.NamespaceHandlerSupport，重写了init方法注册BeanDefinitionParser，自己实现namespaceHandler也可以这样做。

注册property-placeholder 节点，属性加载解析器org.springframework.context.config.PropertyPlaceholderBeanDefinitionParser。

注册property-override 节点，属性是否可以覆盖解析器 org.springframework.context.config.PropertyOverrideBeanDefinitionParser。

注册annotation-config节点，注解配置beanDefinition解析器 org.springframework.context.annotation.AnnotationConfigBeanDefinitionParser。

注册component-scan节点，扫描注解BeanDefinition解析器 org.springframework.context.annotation.ComponentScanBeanDefinitionParser。

这里主要介绍这几个常用的。

```java
protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
		this.parsers.put(elementName, parser);
	}
```
org.springframework.beans.factory.xml.NamespaceHandlerSupport#parsers

```java
private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();
```
注册就是先保存到NamespaceHandlerSupport的parsers map中。

org.springframework.context.config.PropertyPlaceholderBeanDefinitionParser 属性占位符BeanDefinition解析器，<context:property-placeholder/>节点。

org.springframework.context.config.PropertyPlaceholderBeanDefinitionParser#doParse

```java
@Override
	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
		super.doParse(element, parserContext, builder);

//		ignore-unresolvable属性值，不能解析直接忽略
		builder.addPropertyValue("ignoreUnresolvablePlaceholders",
				Boolean.valueOf(element.getAttribute("ignore-unresolvable")));

		String systemPropertiesModeName = element.getAttribute(SYSTEM_PROPERTIES_MODE_ATTRIBUTE);
		if (StringUtils.hasLength(systemPropertiesModeName) &&
				!systemPropertiesModeName.equals(SYSTEM_PROPERTIES_MODE_DEFAULT)) {
			builder.addPropertyValue("systemPropertiesModeName", "SYSTEM_PROPERTIES_MODE_" + systemPropertiesModeName);
		}

		if (element.hasAttribute("value-separator")) {
			builder.addPropertyValue("valueSeparator", element.getAttribute("value-separator"));
		}
		if (element.hasAttribute("trim-values")) {
			builder.addPropertyValue("trimValues", element.getAttribute("trim-values"));
		}
		if (element.hasAttribute("null-value")) {
			builder.addPropertyValue("nullValue", element.getAttribute("null-value"));
		}
	}
```

org.springframework.context.config.AbstractPropertyLoadingBeanDefinitionParser#doParse

```java
	@Override
	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
//		获取location属性
		String location = element.getAttribute("location");
		if (StringUtils.hasLength(location)) {
//			得到解析完占位符的路径
			location = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(location);
//			路径可以是多个，按照,分开
			String[] locations = StringUtils.commaDelimitedListToStringArray(location);
			builder.addPropertyValue("locations", locations);
		}

//		properties-ref属性值
		String propertiesRef = element.getAttribute("properties-ref");
		if (StringUtils.hasLength(propertiesRef)) {
			builder.addPropertyReference("properties", propertiesRef);
		}

//		file-encoding 属性文件编码
		String fileEncoding = element.getAttribute("file-encoding");
		if (StringUtils.hasLength(fileEncoding)) {
			builder.addPropertyValue("fileEncoding", fileEncoding);
		}

		String order = element.getAttribute("order");
		if (StringUtils.hasLength(order)) {
			builder.addPropertyValue("order", Integer.valueOf(order));
		}

//		ignore-resource-not-found 没找到属性文件忽略
		builder.addPropertyValue("ignoreResourceNotFound",
				Boolean.valueOf(element.getAttribute("ignore-resource-not-found")));

//		找到相同的属性文件名是否可以覆盖
		builder.addPropertyValue("localOverride",
				Boolean.valueOf(element.getAttribute("local-override")));

		builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	}
```

org.springframework.context.config.PropertyOverrideBeanDefinitionParser 属性是否可以覆盖，解析<context:property-override/>节点。

org.springframework.context.config.PropertyOverrideBeanDefinitionParser#doParse

```java
@Override
	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
		super.doParse(element, parserContext, builder);

//		属性值不能解析直接忽略
		builder.addPropertyValue("ignoreInvalidKeys",
				Boolean.valueOf(element.getAttribute("ignore-unresolvable")));

	}
```

org.springframework.context.annotation.AnnotationConfigBeanDefinitionParser 注解配置BeanDefinition解析器，<context: annotationconfig />节点。

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
注册注解配置处理器，ContextAnnotationAutowireCandidateResolver、ConfigurationClassPostProcessor、AutowiredAnnotationBeanPostProcessor、RequiredAnnotationBeanPostProcessor、CommonAnnotationBeanPostProcessor等，注册beanDefinition组件。

org.springframework.context.annotation.ComponentScanBeanDefinitionParser bean扫描beanDefinition解析器，<context:component-scan/>节点

```java
private static final String BASE_PACKAGE_ATTRIBUTE = "base-package";
```
指定包名扫描。

```java
//	排除的过滤器
	private static final String EXCLUDE_FILTER_ELEMENT = "exclude-filter";

//	包含的过滤器
	private static final String INCLUDE_FILTER_ELEMENT = "include-filter";
```
扫描的时候可以指定过滤器。

```java
private static final String FILTER_EXPRESSION_ATTRIBUTE = "expression";
```
过滤器支持expression表达式。

org.springframework.context.annotation.ComponentScanBeanDefinitionParser#parse

```java
@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
//		获取base-package属性
		String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);
//		扫描的包名支持占位符
		basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
//		扫描的包可以是多个用分隔符分开
		String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,
				ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

		// Actually scan for bean definitions and register them. 扫描bean定义注册
		ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
//		bean定义扫描
		Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
//		注册BeanDefinition组件
		registerComponents(parserContext.getReaderContext(), beanDefinitions, element);

		return null;
	}
```
BeanDefinition解析。base-package可以指定多个包名，配置BeanDefinition扫描器ClassPathBeanDefinitionScanner，BeanDefinition扫描器进行BeanDefinition扫描，注册BeanDefinition组件。

org.springframework.context.annotation.ComponentScanBeanDefinitionParser#configureScanner

```java
	protected ClassPathBeanDefinitionScanner configureScanner(ParserContext parserContext, Element element) {
		boolean useDefaultFilters = true;
//		如果有这个use-default-filters属性 使用默认的filter进行包扫描
		if (element.hasAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE)) {
			useDefaultFilters = Boolean.valueOf(element.getAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE));
		}

		// Delegate bean definition registration to scanner class.创建bean定义扫描器
		ClassPathBeanDefinitionScanner scanner = createScanner(parserContext.getReaderContext(), useDefaultFilters);
//		设置BeanDefinition解析的默认选项
		scanner.setBeanDefinitionDefaults(parserContext.getDelegate().getBeanDefinitionDefaults());
//		设置自动装配模式
		scanner.setAutowireCandidatePatterns(parserContext.getDelegate().getAutowireCandidatePatterns());

		if (element.hasAttribute(RESOURCE_PATTERN_ATTRIBUTE)) {
			scanner.setResourcePattern(element.getAttribute(RESOURCE_PATTERN_ATTRIBUTE));
		}

		try {
//			解析beanName生成器
			parseBeanNameGenerator(element, scanner);
		}
		catch (Exception ex) {
			parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
		}

		try {
//			解析bean的scope
			parseScope(element, scanner);
		}
		catch (Exception ex) {
			parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
		}

//		解析扫描过滤器
		parseTypeFilters(element, scanner, parserContext);

		return scanner;
	}
```
配置BeanDefinition扫描器。解析BeanDefinition扫描过滤器，创建ClassPathBeanDefinitionScanner，设置ClassPathBeanDefinitionScanner的BeanDefinition默认配置，设置bean是否默认自动注入配置，解析beanName生成器。解析scope，解析banDefinition扫描过滤器。


org.springframework.context.annotation.ComponentScanBeanDefinitionParser#createScanner

```java
protected ClassPathBeanDefinitionScanner createScanner(XmlReaderContext readerContext, boolean useDefaultFilters) {
//		创建BeanDefinition扫描器
		return new ClassPathBeanDefinitionScanner(readerContext.getRegistry(), useDefaultFilters,
				readerContext.getEnvironment(), readerContext.getResourceLoader());
	}
```
根据BeanDefinitionRegistry、是否需要添加默认BeanDefinition扫描过滤器创建BeanDefinition扫描器ClassPathBeanDefinitionScanner。

org.springframework.context.annotation.ClassPathBeanDefinitionScanner#ClassPathBeanDefinitionScanner(org.springframework.beans.factory.support.BeanDefinitionRegistry, boolean, org.springframework.core.env.Environment, org.springframework.core.io.ResourceLoader)

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
如果需要注册默认的BeanDefinition扫描过滤器就注册。

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
这里扫描@Component注解、@Repository、@Service、@Controller 这三个注解是都是基于@Component作用是一样的，spring为了区分mvc分层，我们也可以这样扩展自己的注解加入到spring的体系中来。还支持@ManagedBean，@Named 这个注解一般和@Inject注解一起使用。这个注解组合类似于spring的@Qualifier、@Autowired注解。

返回org.springframework.context.annotation.ComponentScanBeanDefinitionParser#configureScanner

```java
scanner.setBeanDefinitionDefaults(parserContext.getDelegate().getBeanDefinitionDefaults());
```
注册默认的BeanDefinition配置，org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#getBeanDefinitionDefaults

```java
public BeanDefinitionDefaults getBeanDefinitionDefaults() {
		BeanDefinitionDefaults bdd = new BeanDefinitionDefaults();
//		解析<beans>中是否开启延迟加载
		bdd.setLazyInit("TRUE".equalsIgnoreCase(this.defaults.getLazyInit()));
//		设置自动注入模式
		bdd.setAutowireMode(getAutowireMode(DEFAULT_VALUE));
//		设置初始化方法和销毁方法
		bdd.setInitMethodName(this.defaults.getInitMethod());
		bdd.setDestroyMethodName(this.defaults.getDestroyMethod());
		return bdd;
	}
```
注册默认的是否延迟加载设置，自动注入设置，初始化和销毁方法。

```java
try {
//			解析bean的scope
			parseScope(element, scanner);
		}
```
解析scope，org.springframework.context.annotation.ComponentScanBeanDefinitionParser#parseScope

```java
	protected void parseScope(Element element, ClassPathBeanDefinitionScanner scanner) {
		// Register ScopeMetadataResolver if class name provided.
		if (element.hasAttribute(SCOPE_RESOLVER_ATTRIBUTE)) {
			if (element.hasAttribute(SCOPED_PROXY_ATTRIBUTE)) {
				throw new IllegalArgumentException(
						"Cannot define both 'scope-resolver' and 'scoped-proxy' on <component-scan> tag");
			}
			ScopeMetadataResolver scopeMetadataResolver = (ScopeMetadataResolver) instantiateUserDefinedStrategy(
					element.getAttribute(SCOPE_RESOLVER_ATTRIBUTE), ScopeMetadataResolver.class,
					scanner.getResourceLoader().getClassLoader());
			scanner.setScopeMetadataResolver(scopeMetadataResolver);
		}

//		获取scoped-proxy属性
		if (element.hasAttribute(SCOPED_PROXY_ATTRIBUTE)) {
			String mode = element.getAttribute(SCOPED_PROXY_ATTRIBUTE);
//			cglib动态代理
			if ("targetClass".equals(mode)) {
				scanner.setScopedProxyMode(ScopedProxyMode.TARGET_CLASS);
			}
//			jdk动态代理
			else if ("interfaces".equals(mode)) {
				scanner.setScopedProxyMode(ScopedProxyMode.INTERFACES);
			}
			else if ("no".equals(mode)) {
				scanner.setScopedProxyMode(ScopedProxyMode.NO);
			}
			else {
				throw new IllegalArgumentException("scoped-proxy only supports 'no', 'interfaces' and 'targetClass'");
			}
		}
	}
```
scope的代理模式是jdk和cglib。

```java
//		解析扫描过滤器
		parseTypeFilters(element, scanner, parserContext);
```
解析BeanDefinition类型扫描器，org.springframework.context.annotation.ComponentScanBeanDefinitionParser#parseTypeFilters

```java
	protected void parseTypeFilters(Element element, ClassPathBeanDefinitionScanner scanner, ParserContext parserContext) {
		// Parse exclude and include filter elements.解析排除和包含筛选器元素。
		ClassLoader classLoader = scanner.getResourceLoader().getClassLoader();
		NodeList nodeList = element.getChildNodes();
		for (int i = 0; i < nodeList.getLength(); i++) {
			Node node = nodeList.item(i);
			if (node.getNodeType() == Node.ELEMENT_NODE) {
				String localName = parserContext.getDelegate().getLocalName(node);
				try {
//					include-filter节点
					if (INCLUDE_FILTER_ELEMENT.equals(localName)) {
//						创建类型过滤器
						TypeFilter typeFilter = createTypeFilter((Element) node, classLoader, parserContext);
						scanner.addIncludeFilter(typeFilter);
					}
//					exclude-filter节点
					else if (EXCLUDE_FILTER_ELEMENT.equals(localName)) {
						TypeFilter typeFilter = createTypeFilter((Element) node, classLoader, parserContext);
						scanner.addExcludeFilter(typeFilter);
					}
				}
				catch (ClassNotFoundException ex) {
					parserContext.getReaderContext().warning(
							"Ignoring non-present type filter class: " + ex, parserContext.extractSource(element));
				}
				catch (Exception ex) {
					parserContext.getReaderContext().error(
							ex.getMessage(), parserContext.extractSource(element), ex.getCause());
				}
			}
		}
	}
```
返回org.springframework.context.annotation.ComponentScanBeanDefinitionParser#parse

```java
Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
```
扫描BeanDefinition，org.springframework.context.annotation.ClassPathBeanDefinitionScanner#doScan

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		for (String basePackage : basePackages) {
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
//				解析scope的metadata信息
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
//				解析beanName
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
//					解析一般的bean定义属性
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
//					解析一般的bean定义注解
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
//					注册bean定义
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```
解析scope的候选BeanDefinition名称，如果是AnnotatedBeanDefinition处理annotation的BeanDefinition，处理scope代理的BeanDefinition，最后注册BeanDefinition到BeanFactory。

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
解析scope的元数据，比如proxyMode。

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
处理annotation的BeanDefinition。

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
解析scope类型的BeanDefinition设置scopedProxyMode并创建动态代理的BeanDefinition。

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
注册BeanDefinition，有别名注册别名。

返回org.springframework.context.annotation.ComponentScanBeanDefinitionParser#parse

```java
registerComponents(parserContext.getReaderContext(), beanDefinitions, element);
```
注册BeanDefinition组件。

org.springframework.context.annotation.ComponentScanBeanDefinitionParser#registerComponents

```java
	protected void registerComponents(
			XmlReaderContext readerContext, Set<BeanDefinitionHolder> beanDefinitions, Element element) {

		Object source = readerContext.extractSource(element);
		CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), source);

		for (BeanDefinitionHolder beanDefHolder : beanDefinitions) {
			compositeDef.addNestedComponent(new BeanComponentDefinition(beanDefHolder));
		}

		// Register annotation config processors, if necessary.
		boolean annotationConfig = true;
//		节点有annotation-config属性
		if (element.hasAttribute(ANNOTATION_CONFIG_ATTRIBUTE)) {
			annotationConfig = Boolean.valueOf(element.getAttribute(ANNOTATION_CONFIG_ATTRIBUTE));
		}
		if (annotationConfig) {
			Set<BeanDefinitionHolder> processorDefinitions =
//					注册注解解析器
					AnnotationConfigUtils.registerAnnotationConfigProcessors(readerContext.getRegistry(), source);
			for (BeanDefinitionHolder processorDefinition : processorDefinitions) {
				compositeDef.addNestedComponent(new BeanComponentDefinition(processorDefinition));
			}
		}

//		触发组件注册事件
		readerContext.fireComponentRegistered(compositeDef);
	}
```
如果配置了annotation-config节点，注册annotationConfigProcessors，org.springframework.context.annotation.AnnotationConfigUtils#registerAnnotationConfigProcessors(org.springframework.beans.factory.support.BeanDefinitionRegistry, java.lang.Object) 这个方法前面介绍过了。

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
org.springframework.context.annotation.ComponentScanBeanDefinitionParser#instantiateUserDefinedStrategy

```java
private Object instantiateUserDefinedStrategy(
			String className, Class<?> strategyType, @Nullable ClassLoader classLoader) {

		Object result;
		try {
//			反射调用构造方法创建对象
			result = ReflectionUtils.accessibleConstructor(ClassUtils.forName(className, classLoader)).newInstance();
		}
		catch (ClassNotFoundException ex) {
			throw new IllegalArgumentException("Class [" + className + "] for strategy [" +
					strategyType.getName() + "] not found", ex);
		}
		catch (Throwable ex) {
			throw new IllegalArgumentException("Unable to instantiate class [" + className + "] for strategy [" +
					strategyType.getName() + "]: a zero-argument constructor is required", ex);
		}

		if (!strategyType.isAssignableFrom(result.getClass())) {
			throw new IllegalArgumentException("Provided class name must be an implementation of " + strategyType);
		}
		return result;
	}
```
bean默认初始化策略是反射调类的构造方法初始化。
