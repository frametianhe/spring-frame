# BeanDefinition

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1574522847124-805e380f-a6ac-4579-8cf8-5c4489a77013.png#align=left&display=inline&height=522&name=image.png&originHeight=1044&originWidth=2764&size=177008&status=done&width=1382)

spring ioc设计者把bean的初始化过程分为namespaceHandler 元数据装载，BeanDefinition 在bean初始化之前的元数据，然后把把BeanDefinition注册到BeanFactory中，然后BeanPostProcessor是在BeanDefinition创建过程到bean初始化过程进行干预的实现，aware对bean初始化过程进行干预的实现。本次开始介绍BeanDefinition。

BeanDefinition的顶层接口是BeanMetadataElement接口。

```java
Object getSource();
```

返回元素的配置对象。

ComponentDefinition子接口，这个接口是组合的BeanDefinition抽象，可以把一组BeanDefinition和RuntimeBeanReferences组装成一个BeanDefinition。

```java
BeanDefinition[] getBeanDefinitions();
```
返回组合BeanDefinition中的所有BeanDefinition。

```java
BeanReference[] getBeanReferences();
```

返回组合BeanDefinition中的BeanReference。

AbstractComponentDefinition这个是ComponentDefinition的抽象实现。

```java
@Override
	public BeanDefinition[] getBeanDefinitions() {
		return new BeanDefinition[0];
	}
```

返回BeanDefinition空数组。

```java
@Override
	public BeanReference[] getBeanReferences() {
		return new BeanReference[0];
	}
```

返回BeanReference空数组。

BeanComponentDefinition具体实现

```java
private BeanDefinition[] innerBeanDefinitions;

	private BeanReference[] beanReferences;
```
beanDefinition和BeanReference保存在数组中。


BeanDefinitionHolder beanDefinition的持有者

```java
private final BeanDefinition beanDefinition;

	private final String beanName;

	@Nullable
	private final String[] aliases;
```
提供了BeanDefinition对象，beanName，和BeanDefinition的别名。
CompositeComponentDefinition是ComponentDefinition的具体实现，开发中用组合BeanDefinition一般都使用这个。

```java
public void addNestedComponent(ComponentDefinition component) {
		Assert.notNull(component, "ComponentDefinition must not be null");
		this.nestedComponents.add(component);
	}
```
方法可以添加内部的组合BeanDefinition。

DefaultsDefinition子接口就是继承了BeanMetadataElement接口。

DocumentDefaultsDefinition是DefaultsDefinition的实现，提供了属性lazyInit是否延迟初始化，autowire是否自动注册，merge BeanDefinition属性是否可以merge，autowireCandidates属性依赖注入的模式，initMethod初始化方法，destroyMethod销毁方法，source BeanDefinition元数据对象。

ImportDefinition实现，引入的BeanDefinition。

BeanDefinition beanDefinition接口，描述一个bean的元数据信息，包括bean的属性值，构造参数值，具体实现，BeanFactoryPostProcessor可以对BeanDefinition进行干预。

```java
String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON
String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE
```

bean的scope singleton、prototype。

```java
@Nullable
	String getBeanClassName();
```
获取beanDefinition指定的className。

```java
void setScope(@Nullable String scope);
```
覆盖beanDefinition的scope属性。

```java
void setLazyInit(boolean lazyInit);
```
覆盖BeanDefinition是否延迟初始化属性。

```java
void setDependsOn(@Nullable String... dependsOn);
```
设置初始化依赖的beanNames。

```java
void setAutowireCandidate(boolean autowireCandidate);
```
覆盖bean的自动依赖注入的方式。

```java
void setPrimary(boolean primary);
```
如果bean被自动依赖注入的时候，按类型注入，这个类型有多个实现，注入这个。

```java
void setFactoryBeanName(@Nullable String factoryBeanName);
```
设置factoryBean name。

```java
ConstructorArgumentValues getConstructorArgumentValues()
```
返回bean的BeanDefinition中的构造参数值。

```java
MutablePropertyValues getPropertyValues();
```
获取beanDefinition的属性值。

还提供了一些获取BeanDefinition信息的方法，这里不一一介绍了。

AnnotatedBeanDefinition是beanDefinition的下一级接口，解析annotation的beanDefinition。

AbstractBeanDefinition是beanDefinition的抽象实现

```java
public static final String SCOPE_DEFAULT = ""
```
BeanDefinition的scope属性默认是单例。

```java
public static final int AUTOWIRE_NO = AutowireCapableBeanFactory.AUTOWIRE_NO
```
bean自动注入属性值是不自动注入。

```java
public static final int AUTOWIRE_BY_NAME = AutowireCapableBeanFactory.AUTOWIRE_BY_NAME
```
按名称注入。

```java
public static final int AUTOWIRE_BY_TYPE = AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE
```
按类型注入。

```java
public static final int AUTOWIRE_CONSTRUCTOR = AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR
```
按构造方法注入。

```java
private volatile Object beanClass
```
保存了beanClass对象。

```java
private boolean lazyInit = false
```
默认不延迟初始化。

```java
private int autowireMode = AUTOWIRE_NO
```
默认不自动注入。

```java
private String[] dependsOn
```
bean初始化依赖的beanNames。

```java
@Nullable
	private ConstructorArgumentValues constructorArgumentValues;

	@Nullable
	private MutablePropertyValues propertyValues;
```
构造参数值，属性值。

```java
@Nullable
	private String initMethodName;

	@Nullable
	private String destroyMethodName;
```
初始化方法和销毁方法。

```java
@Override
	public boolean isSingleton() {
		return SCOPE_SINGLETON.equals(scope) || SCOPE_DEFAULT.equals(scope);
	}
```
是否单例，SCOPE_SINGLETON = "singleton"，SCOPE_DEFAULT = ""，scope不指定默认就是单例模式。

org.springframework.beans.factory.support.AbstractBeanDefinition#getResolvedAutowireMode
```java
public int getResolvedAutowireMode() {
		if (this.autowireMode == AUTOWIRE_AUTODETECT) {
			// Work out whether to apply setter autowiring or constructor autowiring.
			// If it has a no-arg constructor it's deemed to be setter autowiring,
			// otherwise we'll try constructor autowiring.
			Constructor<?>[] constructors = getBeanClass().getConstructors();
			for (Constructor<?> constructor : constructors) {
				if (constructor.getParameterCount() == 0) {
					return AUTOWIRE_BY_TYPE;
				}
			}
			return AUTOWIRE_CONSTRUCTOR;
		}
		else {
			return this.autowireMode;
		}
```
如果指定了构造参数采取构造参数注入，如果没有指定构造参数就采用按类型自动注入，构造参数注入，构造参数多了会显得代码臃肿，构造参数少了可以。这里可以看出优先使用的是构造参数注入。

```java
private Supplier<?> instanceSupplier;
```
属性指定创建bean的回调逻辑。

GenericBeanDefinition标准BeanDefinition的实现。
AnnotatedGenericBeanDefinition 注解的标准BeanDefinition实现。
ScannedGenericBeanDefinition 基于ASM ClassReader的GenericBeanDefinition实现，**ASM**是一个通用的Java字节码操作和分析框架。它可以用于修改现有类或直接以二进制形式动态生成类。ASM提供了一些常见的字节码转换和分析算法，可以从中构建自定义复杂转换和代码分析工具。ASM提供与其他Java字节码框架类似的功能，但专注于 [性能](https://asm.ow2.io/performance.html)。因为它的设计和实现尽可能小而且快，所以它非常适合在动态系统中使用，简单的说就是这种技术比反射效率高。
RootBeanDefinition实现，在运行时在springFactory中支持特定的bean的合并。
ChildBeanDefinition实现，继承了父BeanDefinition的设置。
spring2.5以后ChildBeanDefinition的出现，可以用parentName灵活的配置父的BeanDefinition，一般使用这个，RootBeanDefinition和ChildBeanDefinition用的比较少了。
AliasDefinition 别名的BeanDefinition。
AutowireCandidateQualifier 依赖注入候选确定实现，解析@Autowire、@Qualifier注解。
BeanReference接口，引用对象抽象。
RuntimeBeanReference实现，一个属性值不可变占位符类，引用工厂的另一个bean，在运行时解析。
RuntimeBeanNameReference实现，当工厂bean引用另一个bean，一个引用bean的不可变占位符在运行时解析。
BeanDefinitionParser BeanDefinition解析器，parse方法解析节点返回BeanDefinition。
AbstractBeanDefinitionParser 抽象的beanDefinition解析器实现，开发中自定义BeanDefinition解析器一般继承这个类，org.springframework.beans.factory.xml.AbstractBeanDefinitionParser#parse BeanDefinition解析方法，这里是模板方法设计，org.springframework.beans.factory.xml.AbstractBeanDefinitionParser#parseInternal方法提供了元素解析BeanDefinition的实现，用户需要实现这个方法。

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
				BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
				registerBeanDefinition(holder, parserContext.getRegistry());
				if (shouldFireEvents()) {
					BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
					postProcessComponentDefinition(componentDefinition);
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
解析beanDefinition后执行解析完BeanDefinition钩子方法，注册BeanDefinition，然后发布注册完BeanDefinition事件。

org.springframework.beans.factory.xml.AbstractBeanDefinitionParser#registerBeanDefinition注册beanDefinition

```java
protected void registerBeanDefinition(BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {
		BeanDefinitionReaderUtils.registerBeanDefinition(definition, registry);
	}
```
org.springframework.beans.factory.support.BeanDefinitionReaderUtils#registerBeanDefinition 注册BeanDefinition

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
这里按beanName和bean的别名注册后面逻辑介绍。

AbstractSingleBeanDefinitionParser 单例bean的BeanDefinition解析器抽象实现。
org.springframework.beans.factory.xml.AbstractSingleBeanDefinitionParser#parseInternal解析beanDefinition方法

```java
@Override
	protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
		String parentName = getParentName(element);
		if (parentName != null) {
		}
			builder.getRawBeanDefinition().setParentName(parentName);
		Class<?> beanClass = getBeanClass(element);
		if (beanClass != null) {
			builder.getRawBeanDefinition().setBeanClass(beanClass);
		}
		else {
			String beanClassName = getBeanClassName(element);
			if (beanClassName != null) {
				builder.getRawBeanDefinition().setBeanClassName(beanClassName);
			}
		}
		builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
//		获取内部bean的bean定义
		BeanDefinition containingBd = parserContext.getContainingBeanDefinition();
//		如果匿名bean有scope，外部bean的scope以这个为主
		if (containingBd != null) {
			// Inner bean definition must receive same scope as containing bean.
			builder.setScope(containingBd.getScope());
		}
//		是否延迟加载
		if (parserContext.isDefaultLazyInit()) {
			// Default-lazy-init applies to custom bean definitions as well.
			builder.setLazyInit(true);
		}
		doParse(element, parserContext, builder);
		return builder.getBeanDefinition();
	}
```
设置BeanDefinition的scope属性和lazyInit延迟加载属性，默认true延迟加载。调用doParse模板方法解析beanDefinition。
todo

```java
protected void doParse(Element element, BeanDefinitionBuilder builder) {
	}
```
doParse解析BeanDefinition方法需要用户自己实现。

BeanDefinitionReader BeanDefinition读取的接口，getRegistry获取BeanDefinition注册器，loadBeanDefinitions加载BeanDefinition，AbstractBeanDefinitionReader BeanDefinition的抽象实现

```java
private final BeanDefinitionRegistry registry;
```
内部维护了一个BeanDefinition注册器实例。

org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(java.lang.String, java.util.Set<org.springframework.core.io.Resource>)加载beanDefinition

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
XmlBeanDefinitionReader xml的BeanDefinition读取器，org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.support.EncodedResource)加载BeanDefinition

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
org.springframework.beans.factory.xml.XmlBeanDefinitionReader#doLoadBeanDefinitions 加载BeanDefinition

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
//			加载xml文档
			Document doc = doLoadDocument(inputSource, resource);
//			注册bean定义
			return registerBeanDefinitions(doc, resource);
		}
```
读取xml文档解析BeanDefinition，然后注册BeanDefinition，注册BeanDefinition后面再介绍。

BeanDefinitionDocumentReader接口提供了registerBeanDefinitions 注册beanDefinition方法。DefaultBeanDefinitionDocumentReader默认实现，org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#registerBeanDefinitions注册BeanDefinition

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
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions 注册<beans>节点下的<bean>节点的BeanDefinition

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
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#createDelegate 创建BeanDefinition委托

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
org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#initDefaults(org.w3c.dom.Element, org.springframework.beans.factory.xml.BeanDefinitionParserDelegate) 初始化默认配置

```java
public void initDefaults(Element root, @Nullable BeanDefinitionParserDelegate parent) {
//		解析默认的bean定义解析配置
		populateDefaults(this.defaults, (parent != null ? parent.defaults : null), root);
//		触发默认的bean定义事件
		this.readerContext.fireDefaultsRegistered(this.defaults);
	}
```
org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#populateDefaults解析beans标签属性

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
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#preProcessXml

```java
protected void preProcessXml(Element root) {
	}
```
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#postProcessXml

```java
protected void postProcessXml(Element root) {
	}
```
这个方法在BeanDefinition处理前后子类可以重写自己定义xml元素加载成beanDefinition。
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseBeanDefinitions 解析BeanDefinition

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
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseDefaultElement 解析默认标签的BeanDefinition

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
org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseCustomElement(org.w3c.dom.Element, org.springframework.beans.factory.config.BeanDefinition) 解析自定义的namespaceHandler

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
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#importBeanDefinitionResource 解析import beanDefinition

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
org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#processAliasRegistration 解析 alias BeanDefinition

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
把BeanDefinition注册到beanFactory后面的逻辑中介绍。

org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionElement(org.w3c.dom.Element, org.springframework.beans.factory.config.BeanDefinition) 解析 bean BeanDefinition

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

