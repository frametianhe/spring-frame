# PropertyResourceConfigurer

org.springframework.beans.factory.config.PropertyResourceConfigurer
允许为了配置从个人的bean属性值从属性文件，例如properties文件，覆盖一个上下文的bean属性用客户端配置文件，这两个抽象类有两个实现，PropertyOverrideConfigurer， 键值对的方式直接对属性的值进行覆盖，PropertyPlaceholderConfigurer占位符的方式从属性文件中读取属性对上下文的bean属性值就行覆盖。
org.springframework.beans.factory.config.PropertyResourceConfigurer#postProcessBeanFactory

```java
@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		try {
			Properties mergedProps = mergeProperties();

			// Convert the merged properties, if necessary.如果需要，转换合并的属性。
			convertProperties(mergedProps);

			// Let the subclass process the properties.对子类的属性也就行覆盖
			processProperties(beanFactory, mergedProps);
		}
		catch (IOException ex) {
			throw new BeanInitializationException("Could not load properties", ex);
		}
	}
```
重写了org.springframework.beans.factory.config.BeanFactoryPostProcessor#postProcessBeanFactory方法，次方法在应用程序上下文初始化完毕，所有BeanDefinition都已加载但是还没实例化bean，可以覆盖或添加属性。
这里的处理是从配置文件读取属性值把改java bean的属性和子类的属性都覆盖。

org.springframework.beans.factory.config.PlaceholderConfigurerSupport
用于解析bean定义属性值中的占位符的属性资源配置器的抽象基类，实现将值从属性文件或其他属性源提取到bean定义中。

```java
protected boolean ignoreUnresolvablePlaceholders = false;
```
如果占位符没解析成功是否忽略，默认不忽略直接抛异常。

```java
@Nullable
	private BeanFactory beanFactory;
```
BeanFactory。

```java
protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
			StringValueResolver valueResolver) {

		BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);

		String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
		for (String curName : beanNames) {
			// Check that we're not parsing our own bean definition,
			// to avoid failing on unresolvable placeholders in properties file locations.//检查我们没有解析我们自己的bean定义，
//以避免在属性文件位置中无法解析的占位符失败。
			if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
				BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
				try {
					visitor.visitBeanDefinition(bd);
				}
				catch (Exception ex) {
					throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, ex.getMessage(), ex);
				}
			}
		}

		// New in Spring 2.5: resolve placeholders in alias target names and aliases as well.Spring 2.5的新特性:还可以解析别名目标名称和别名中的占位符。
		beanFactoryToProcess.resolveAliases(valueResolver);

		// New in Spring 3.0: resolve placeholders in embedded values such as annotation attributes.Spring 3.0的新功能:解析嵌入值(如注释属性)中的占位符。
		beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
	}
```
设置bean属性值。

org.springframework.beans.factory.config.PropertyOverrideConfigurer
在应用程序上下文定义中重写bean属性值的属性资源配置程序。它将属性文件中的值推入bean定义，如果有多个PropertyOverrideConfigurers为同一个bean属性定义不同的值，最后一个将胜出(由于覆盖机制)。

```java
public static final String DEFAULT_BEAN_NAME_SEPARATOR = ".";
```
bean属性分隔符。

```java
private boolean ignoreInvalidKeys = false;
```
忽略无效的属性名。

```java
@Override
	protected void processProperties(ConfigurableListableBeanFactory beanFactory, Properties props)
			throws BeansException {

		for (Enumeration<?> names = props.propertyNames(); names.hasMoreElements();) {
			String key = (String) names.nextElement();
			try {
				processKey(beanFactory, key, props.getProperty(key));
			}
			catch (BeansException ex) {
				String msg = "Could not process key '" + key + "' in PropertyOverrideConfigurer";
				if (!this.ignoreInvalidKeys) {
					throw new BeanInitializationException(msg, ex);
				}
				if (logger.isDebugEnabled()) {
					logger.debug(msg, ex);
				}
			}
		}
	}
```
org.springframework.beans.factory.config.PropertyOverrideConfigurer#processKey

```java
protected void processKey(ConfigurableListableBeanFactory factory, String key, String value)
			throws BeansException {

		int separatorIndex = key.indexOf(this.beanNameSeparator);
		if (separatorIndex == -1) {
			throw new BeanInitializationException("Invalid key '" + key +
					"': expected 'beanName" + this.beanNameSeparator + "property'");
		}
		String beanName = key.substring(0, separatorIndex);
		String beanProperty = key.substring(separatorIndex+1);
		this.beanNames.add(beanName);
		applyPropertyValue(factory, beanName, beanProperty, value);
		if (logger.isDebugEnabled()) {
			logger.debug("Property '" + key + "' set to value [" + value + "]");
		}
	}
```
org.springframework.beans.factory.config.PropertyOverrideConfigurer#applyPropertyValue

```java
protected void applyPropertyValue(
			ConfigurableListableBeanFactory factory, String beanName, String property, String value) {

		BeanDefinition bd = factory.getBeanDefinition(beanName);
		BeanDefinition bdToUse = bd;
		while (bd != null) {
			bdToUse = bd;
			bd = bd.getOriginatingBeanDefinition();
		}
		PropertyValue pv = new PropertyValue(property, value);
		pv.setOptional(this.ignoreInvalidKeys);
		bdToUse.getPropertyValues().addPropertyValue(pv);
	}
```
处理bean的属性值。

org.springframework.context.support.PropertySourcesPlaceholderConfigurer
解析${}对当前Spring环境及其属性集的bean定义属性值和@Value注释的占位符。

```java
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		if (this.propertySources == null) {
			this.propertySources = new MutablePropertySources();
			if (this.environment != null) {
				this.propertySources.addLast(
					new PropertySource<Environment>(ENVIRONMENT_PROPERTIES_PROPERTY_SOURCE_NAME, this.environment) {
						@Override
						@Nullable
						public String getProperty(String key) {
							return this.source.getProperty(key);
						}
					}
				);
			}
			try {
				PropertySource<?> localPropertySource =
						new PropertiesPropertySource(LOCAL_PROPERTIES_PROPERTY_SOURCE_NAME, mergeProperties());
				if (this.localOverride) {
					this.propertySources.addFirst(localPropertySource);
				}
				else {
					this.propertySources.addLast(localPropertySource);
				}
			}
			catch (IOException ex) {
				throw new BeanInitializationException("Could not load properties", ex);
			}
		}

		processProperties(beanFactory, new PropertySourcesPropertyResolver(this.propertySources));
		this.appliedPropertySources = this.propertySources;
	}
```
覆盖了父类实现的org.springframework.beans.factory.config.BeanFactoryPostProcessor#postProcessBeanFactory这个方法。
这里的处理是处理占位符给bean属性赋值。
org.springframework.context.support.PropertySourcesPlaceholderConfigurer#processProperties(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, org.springframework.core.env.ConfigurablePropertyResolver)

```java
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
			final ConfigurablePropertyResolver propertyResolver) throws BeansException {

		propertyResolver.setPlaceholderPrefix(this.placeholderPrefix);
		propertyResolver.setPlaceholderSuffix(this.placeholderSuffix);
		propertyResolver.setValueSeparator(this.valueSeparator);

		StringValueResolver valueResolver = strVal -> {
			String resolved = (ignoreUnresolvablePlaceholders ?
					propertyResolver.resolvePlaceholders(strVal) :
					propertyResolver.resolveRequiredPlaceholders(strVal));
			if (trimValues) {
				resolved = resolved.trim();
			}
			return (resolved.equals(nullValue) ? null : resolved);
		};

		doProcessProperties(beanFactoryToProcess, valueResolver);
	}
```
访问BeanFactory的BeanDefinition并尝试替换${}占位符。

