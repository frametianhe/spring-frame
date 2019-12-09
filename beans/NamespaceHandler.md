# NamespaceHandler


![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1574502426983-66821567-f954-4955-b3e8-252363cbeb33.png#align=left&display=inline&height=426&name=image.png&originHeight=852&originWidth=1626&size=81274&status=done&width=813)
NamespaceHandler接口，是spring BeanDifinition命名空间顶层接口。
parse方法，解析元素返回beanDifinition
decorate方法，这是一个装饰方法，解析指定的节点成beanDifinition后包装成beanDifinitionHolder。

NamespaceHandlerSupport抽象类实现了NamespaceHandler接口，在开发中如果想要扩展自己的namespacehandler一般会继承这个抽象类。

```java
private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();
```
namespace的节点和BeanDefinitionParser保存在本地map中。

```java
	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
//		获得bean定义解析器
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		return (parser != null ? parser.parse(element, parserContext) : null);
	}
```
从本地map中按指定的节点找到BeanDefinitionParser调用parse方法进行beanDifinition解析返回BeanDefinition。

```java
@Nullable
	private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
		String localName = parserContext.getDelegate().getLocalName(element);
		BeanDefinitionParser parser = this.parsers.get(localName);
		if (parser == null) {
			parserContext.getReaderContext().fatal(
					"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
		}
		return parser;
	}
```
查找BeanDefinitionParser。

```java
@Override
	@Nullable
	public BeanDefinitionHolder decorate(
			Node node, BeanDefinitionHolder definition, ParserContext parserContext) {

		BeanDefinitionDecorator decorator = findDecoratorForNode(node, parserContext);
		return (decorator != null ? decorator.decorate(node, definition, parserContext) : null);
	}
```
找到指定节点的BeanDefinitionDecorator调用decorate方法返回BeanDefinitionHolder。

```java
protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
		this.parsers.put(elementName, parser);
	}
```
注册BeanDefinitionParser，就是保存在本地parsers map中。

```java
protected final void registerBeanDefinitionDecorator(String elementName, BeanDefinitionDecorator dec) {
		this.decorators.put(elementName, dec);
	}
```
注册BeanDefinitionDecorator，也是保存在decorators map中。

UtilNamespaceHandler namespaceHandler的实现类

```java
@Override
	public void init() {
		registerBeanDefinitionParser("constant", new ConstantBeanDefinitionParser());
		registerBeanDefinitionParser("property-path", new PropertyPathBeanDefinitionParser());
		registerBeanDefinitionParser("list", new ListBeanDefinitionParser());
		registerBeanDefinitionParser("set", new SetBeanDefinitionParser());
		registerBeanDefinitionParser("map", new MapBeanDefinitionParser());
		registerBeanDefinitionParser("properties", new PropertiesBeanDefinitionParser());
	}
```
可以看到这个namespaceHandler解析constant、list、set、map、properties集合。

先看下ListBeanDefinitionParser的doparse方法
SetBeanDefinitionParser
```java
@Override
		protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
			List<Object> parsedList = parserContext.getDelegate().parseListElement(element, builder.getRawBeanDefinition());
			builder.addPropertyValue("sourceList", parsedList);

//			解析list-class标签
			String listClass = element.getAttribute("list-class");
			if (StringUtils.hasText(listClass)) {
				builder.addPropertyValue("targetListClass", listClass);
			}

//			获取scope属性值
			String scope = element.getAttribute(SCOPE_ATTRIBUTE);

			if (StringUtils.hasLength(scope)) {
				builder.setScope(scope);
			}
		}
```
解析list-class标签和这个标签的targetListClass属性最后添加到BeanDefinitionBuilder中。
SetBeanDefinitionParser的doParse方法

```java
	@Override
		protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
			Set<Object> parsedSet = parserContext.getDelegate().parseSetElement(element, builder.getRawBeanDefinition());
			builder.addPropertyValue("sourceSet", parsedSet);

			String setClass = element.getAttribute("set-class");
			if (StringUtils.hasText(setClass)) {
				builder.addPropertyValue("targetSetClass", setClass);
			}

			String scope = element.getAttribute(SCOPE_ATTRIBUTE);
			if (StringUtils.hasLength(scope)) {
				builder.setScope(scope);
			}
		}
```
解析set-class标签和targetSetClass属性添加到BeanDefinitionBuilder中。
MapBeanDefinitionParser的doParse方法

```java
@Override
		protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
			Map<Object, Object> parsedMap = parserContext.getDelegate().parseMapElement(element, builder.getRawBeanDefinition());
			builder.addPropertyValue("sourceMap", parsedMap);

			String mapClass = element.getAttribute("map-class");
			if (StringUtils.hasText(mapClass)) {
				builder.addPropertyValue("targetMapClass", mapClass);
			}

			String scope = element.getAttribute(SCOPE_ATTRIBUTE);
			if (StringUtils.hasLength(scope)) {
				builder.setScope(scope);
			}
		}
```
解析map-class标签和targetMapClass属性添加到BeanDefinitionBuilder中。
PropertiesBeanDefinitionParser的doParse方法

```java
@Override
		protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
			Properties parsedProps = parserContext.getDelegate().parsePropsElement(element);
			builder.addPropertyValue("properties", parsedProps);

			String location = element.getAttribute("location");
			if (StringUtils.hasLength(location)) {
				location = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(location);
				String[] locations = StringUtils.commaDelimitedListToStringArray(location);
				builder.addPropertyValue("locations", locations);
			}

			builder.addPropertyValue("ignoreResourceNotFound",
					Boolean.valueOf(element.getAttribute("ignore-resource-not-found")));

			builder.addPropertyValue("localOverride",
					Boolean.valueOf(element.getAttribute("local-override")));

			String scope = element.getAttribute(SCOPE_ATTRIBUTE);
			if (StringUtils.hasLength(scope)) {
				builder.setScope(scope);
			}
		}
```
解析location属性，properties文件的路径，文件可以有多个用,分开，把locations属性值是properties文件数组添加到BeanDefinitionBuilder中，添加BeanDefinitionBuilder ignoreResourceNotFound属性，也就是配置的ignore-resource-not-found参数值，没找到属性文件是否忽略。添加BeanDefinitionBuilder的localOverride属性，local-override参数值，属性值重复了是否可以覆盖。

SimplePropertyNamespaceHandler实现，可以将自定义的属性直接映射到Bean属性，具体实现方org.springframework.beans.factory.xml.SimplePropertyNamespaceHandler#decorate

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
解析属性是-ref结尾的，指定的是RuntimeBeanReference类型的bean。
SimpleConstructorNamespaceHandler实现，可以直接把自定义属性值添加到bean中，简写的构造方法参数属性。

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
按index顺序解析名称以为-ref结尾的属性，值也是RuntimeBeanReference这个类型的。
NamespaceHandlerResolver namespaceHandler解析的顶层接口，resolve解析命名空间uri返回NamespaceHandler。默认实现DefaultNamespaceHandlerResolver。

```java
public static final String DEFAULT_HANDLER_MAPPINGS_LOCATION = "META-INF/spring.handlers";
```
namespaceHandler指定的位置是这里，把实现的namespaceHandler放到这里，spring ioc加载jar包的时候就会解析到。

```java
private volatile Map<String, Object> handlerMappings;
```
namespaceHandler映射保存到本地map中，格式是这样的，http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler

```java
public DefaultNamespaceHandlerResolver() {
//		spring.handlers解析
		this(null, DEFAULT_HANDLER_MAPPINGS_LOCATION);
	}
```
构造方法，默认加载_DEFAULT_HANDLER_MAPPINGS_LOCATION _= "META-INF/spring.handlers"这个路径的namespaceHandler。

org.springframework.beans.factory.xml.DefaultNamespaceHandlerResolver#resolve 根据namespaceUrl解析namespaceHandler。

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
先获取handlerMapping，org.springframework.beans.factory.xml.DefaultNamespaceHandlerResolver#getHandlerMappings

```java
	private Map<String, Object> getHandlerMappings() {
		Map<String, Object> handlerMappings = this.handlerMappings;
		if (handlerMappings == null) {
			synchronized (this) {
				handlerMappings = this.handlerMappings;
				if (handlerMappings == null) {
					try {
						Properties mappings =
								PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
						if (logger.isDebugEnabled()) {
							logger.debug("Loaded NamespaceHandler mappings: " + mappings);
						}
						Map<String, Object> mappingsToUse = new ConcurrentHashMap<>(mappings.size());
						CollectionUtils.mergePropertiesIntoMap(mappings, mappingsToUse);
						handlerMappings = mappingsToUse;
						this.handlerMappings = handlerMappings;
					}
					catch (IOException ex) {
						throw new IllegalStateException(
								"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
					}
				}
			}
		}
		return handlerMappings;
	}
```
如果handlerMappings 本地map中没有对象handlerMapping，也就是META-INF/spring.handlers这个路径没有配置对应的namespaceHandler就从handlerMappingsLocation指定的配置文件中加载，如果配置文件是.xml就解析xml配置文件。解析完最后保存到handlerMappings map中。

```java
//创建namespaceHandler类
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
//				初始化namespaceHandler类
				namespaceHandler.init();
```
初始化namespaceHandler调用init方法。初始化方法前面介绍过了就是注册BeanDefinitionParse。
