# BeanFactory之StaticListableBeanFactory

静态BeanFactory实现，允许以编程的方式注册现有的单例bean，不支持运行bean或别名。ListableBeanFactory接口的实现，管理bean实例，而不是基于beanDefinition创建bean实例。

```java
private final Map<String, Object> beans;
```
bean对象存在beans map中。

```java
public void addBean(String name, Object bean) {
		this.beans.put(name, bean);
	}
```
添加bean到BeanFactory。

org.springframework.beans.factory.support.StaticListableBeanFactory#getBean(java.lang.String) 按beanName从BeanFatory中查询bean

```java
@Override
	public Object getBean(String name) throws BeansException {
		String beanName = BeanFactoryUtils.transformedBeanName(name);
		Object bean = this.beans.get(beanName);

		if (bean == null) {
			throw new NoSuchBeanDefinitionException(beanName,
					"Defined beans are [" + StringUtils.collectionToCommaDelimitedString(this.beans.keySet()) + "]");
		}

		// Don't let calling code try to dereference the
		// bean factory if the bean isn't a factory//不要让调用代码试图取消引用
//如果这个bean不是一个工厂，那么它就是一个bean工厂
		if (BeanFactoryUtils.isFactoryDereference(name) && !(bean instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(beanName, bean.getClass());
		}

		if (bean instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
			try {
				Object exposedObject = ((FactoryBean<?>) bean).getObject();
				if (exposedObject == null) {
					throw new BeanCreationException(beanName, "FactoryBean exposed null object");
				}
				return exposedObject;
			}
			catch (Exception ex) {
				throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
			}
		}
		else {
			return bean;
		}
	}
```
按beanName从beanFactory的beans map中获取bean，如果获取不到，如果bean是factory引用不是factoryBean类型报错BeanIsNotAFactoryException。如果bean类型是factoryBean，bean不是factoryBean引用，调用org.springframework.beans.factory.FactoryBean#getObject获取bean实例，如果不是factoryBean就直接返回bean。

org.springframework.beans.factory.support.StaticListableBeanFactory#getBean(java.lang.String, java.lang.Class<T>)

```java
@Override
	@SuppressWarnings("unchecked")
	public <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException {
		Object bean = getBean(name);
		if (requiredType != null && !requiredType.isInstance(bean)) {
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
		return (T) bean;
	}
```
按指定的beanName和类型获取bean，按name从BeanFactory中获取bean，如果指定的类型找到的bean不是这个类型的直接报错BeanNotOfRequiredTypeException，否则直接返回bean。

org.springframework.beans.factory.support.StaticListableBeanFactory#getBean(java.lang.String, java.lang.Object...) 按指定的name和参数查询bean

```java
@Override
	public Object getBean(String name, Object... args) throws BeansException {
		if (!ObjectUtils.isEmpty(args)) {
			throw new UnsupportedOperationException(
					"StaticListableBeanFactory does not support explicit bean creation arguments");
		}
		return getBean(name);
	}
```
如果参数不为空，按name从BeanFactory中获取bean。

org.springframework.beans.factory.support.StaticListableBeanFactory#getBean(java.lang.Class<T>) 按指定的类型获取bean

```java
@Override
	public <T> T getBean(Class<T> requiredType) throws BeansException {
		String[] beanNames = getBeanNamesForType(requiredType);
		if (beanNames.length == 1) {
			return getBean(beanNames[0], requiredType);
		}
		else if (beanNames.length > 1) {
			throw new NoUniqueBeanDefinitionException(requiredType, beanNames);
		}
		else {
			throw new NoSuchBeanDefinitionException(requiredType);
		}
	}
```
按指定的类型找到beanName集合。如果找到了一个按beanName和类型从BeanFactory中获取bean，如果按类型找到了多个beanName报错NoUniqueBeanDefinitionException。

org.springframework.beans.factory.support.StaticListableBeanFactory#getBean(java.lang.Class<T>, java.lang.Object...)按指定的类型和参数从BeanFactory中获取bean

```java
	@Override
	public <T> T getBean(Class<T> requiredType, Object... args) throws BeansException {
		if (!ObjectUtils.isEmpty(args)) {
			throw new UnsupportedOperationException(
					"StaticListableBeanFactory does not support explicit bean creation arguments");
		}
		return getBean(requiredType);
	}
```
如果参数不为空报错，按指定的类型从BeanFactory中获取bean。

org.springframework.beans.factory.support.StaticListableBeanFactory#isSingleton判断指定name的bean是不是单例。

```java
@Override
	public boolean isSingleton(String name) throws NoSuchBeanDefinitionException {
		Object bean = getBean(name);
		// In case of FactoryBean, return singleton status of created object.对于FactoryBean，返回创建对象的单例状态。
		return (bean instanceof FactoryBean && ((FactoryBean<?>) bean).isSingleton());
	}
```
从指定的name从benaFactory中获取bean，如果bean是单例且org.springframework.beans.factory.FactoryBean#isSingleton返回true。

org.springframework.beans.factory.support.StaticListableBeanFactory#isPrototype 指定name的bean是否是原型的。

```java
@Override
	public boolean isPrototype(String name) throws NoSuchBeanDefinitionException {
		Object bean = getBean(name);
		// In case of FactoryBean, return prototype status of created object.对于FactoryBean，返回创建对象的原型状态。
		return ((bean instanceof SmartFactoryBean && ((SmartFactoryBean<?>) bean).isPrototype()) ||
				(bean instanceof FactoryBean && !((FactoryBean<?>) bean).isSingleton()));
	}
```
按指定的name从BeanFactory中获取bean，如果bean是SmartFactoryBean且org.springframework.beans.factory.SmartFactoryBean#isPrototype返回true，或者bean是FactoryBean类型且org.springframework.beans.factory.FactoryBean#isSingleton返回false。

org.springframework.beans.factory.support.StaticListableBeanFactory#getType按指定name获取类型

```java
@Override
	public Class<?> getType(String name) throws NoSuchBeanDefinitionException {
		String beanName = BeanFactoryUtils.transformedBeanName(name);

		Object bean = this.beans.get(beanName);
		if (bean == null) {
			throw new NoSuchBeanDefinitionException(beanName,
					"Defined beans are [" + StringUtils.collectionToCommaDelimitedString(this.beans.keySet()) + "]");
		}

		if (bean instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
			// If it's a FactoryBean, we want to look at what it creates, not the factory class.如果它是一个FactoryBean，我们希望查看它创建了什么，而不是factory类。
			return ((FactoryBean<?>) bean).getObjectType();
		}
		return bean.getClass();
	}
```
按指定的name从BeanFactory中获取bean，如果bean==null报错NoSuchBeanDefinitionException。如果bean是factoryBean类型且指定name的bean不是factory引用，调用org.springframework.beans.factory.FactoryBean#getObjectType获取类型，否则返回bean的class类型。

org.springframework.beans.factory.support.StaticListableBeanFactory#getBeanNamesForType(org.springframework.core.ResolvableType)按指定类型获取beanName集合

```java
@Override
	public String[] getBeanNamesForType(@Nullable ResolvableType type) {
		boolean isFactoryType = false;
		if (type != null) {
			Class<?> resolved = type.resolve();
//			如果类型是factoryBean
			if (resolved != null && FactoryBean.class.isAssignableFrom(resolved)) {
				isFactoryType = true;
			}
		}
		List<String> matches = new ArrayList<>();
		for (Map.Entry<String, Object> entry : this.beans.entrySet()) {
//			循环beans列表中的beanName和bean
			String name = entry.getKey();
			Object beanInstance = entry.getValue();
//			如果bean是factoryBean类型不是factory类型
			if (beanInstance instanceof FactoryBean && !isFactoryType) {
//				从factoryBean中获取objectType
				Class<?> objectType = ((FactoryBean<?>) beanInstance).getObjectType();
				if (objectType != null && (type == null || type.isAssignableFrom(objectType))) {
					matches.add(name);
				}
			}
			else {
				if (type == null || type.isInstance(beanInstance)) {
					matches.add(name);
				}
			}
		}
		return StringUtils.toStringArray(matches);
	}
```
如果指定的类型是factoryBean，是factory类型，循环beans列表中的beanName和beanInstance，如果beanInstance是factoryBean且不是factory类型，调用org.springframework.beans.factory.FactoryBean#getObjectType方法获取objectType，如果类型一致匹配到返回。

org.springframework.beans.factory.support.StaticListableBeanFactory#getBeansOfType(java.lang.Class<T>, boolean, boolean)按指定的类型获取beans

```java
@Override
	@SuppressWarnings("unchecked")
	public <T> Map<String, T> getBeansOfType(@Nullable Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
			throws BeansException {
//		如果bean是factoryBean类型
		boolean isFactoryType = (type != null && FactoryBean.class.isAssignableFrom(type));
		Map<String, T> matches = new LinkedHashMap<>();

//		循环beans列表
		for (Map.Entry<String, Object> entry : this.beans.entrySet()) {
			String beanName = entry.getKey();
			Object beanInstance = entry.getValue();
			// Is bean a FactoryBean? 是factoryBean不是factoryBean
			if (beanInstance instanceof FactoryBean && !isFactoryType) {
				// Match object created by FactoryBean.
				FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
//				获取factoryBean的objectType
				Class<?> objectType = factory.getObjectType();
				if ((includeNonSingletons || factory.isSingleton()) &&
						objectType != null && (type == null || type.isAssignableFrom(objectType))) {
//					按指定的name和类型从BeanFactory中获取bean返回
					matches.put(beanName, getBean(beanName, type));
				}
			}
			else {
//				找到类型一致的bean
				if (type == null || type.isInstance(beanInstance)) {
					// If type to match is FactoryBean, return FactoryBean itself.
					// Else, return bean instance.//如果要匹配的类型是FactoryBean，那么返回FactoryBean本身。
// Else，返回bean实例。
					if (isFactoryType) {
						beanName = FACTORY_BEAN_PREFIX + beanName;
					}
					matches.put(beanName, (T) beanInstance);
				}
			}
		}
		return matches;
	}
```
如果bean是factoryBean类型，循环BeanFactory中beans列表，如果bean是factoryBean类型且不是factory类型，调用org.springframework.beans.factory.FactoryBean#getObjectType方法获取objectType，如果factoryBean是单例且指定的类型和objectType匹配，按指定的name和type从BeanFactory中获取bean返回，否则找到类型的bean返回。

org.springframework.beans.factory.support.StaticListableBeanFactory#getBeanNamesForAnnotation 找到beanName和annotation的类型的annotation。

```java
@Override
	public String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType) {
		List<String> results = new ArrayList<>();
		for (String beanName : this.beans.keySet()) {
			if (findAnnotationOnBean(beanName, annotationType) != null) {
				results.add(beanName);
			}
		}
		return StringUtils.toStringArray(results);
	}
```
org.springframework.beans.factory.support.StaticListableBeanFactory#findAnnotationOnBean

```java
	@Override
	@Nullable
	public <A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
			throws NoSuchBeanDefinitionException{

//		找到beanName的类型
		Class<?> beanType = getType(beanName);
//		找到bean的类型和annotation的类型找到annotation
		return (beanType != null ? AnnotationUtils.findAnnotation(beanType, annotationType) : null);
	}
```
org.springframework.beans.factory.support.StaticListableBeanFactory#getBeansWithAnnotation

```java
	@Override
	public Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType)
			throws BeansException {

		Map<String, Object> results = new LinkedHashMap<>();
//		循环beans列表
		for (String beanName : this.beans.keySet()) {
//			按beanName和annotation的类型找到annotation
			if (findAnnotationOnBean(beanName, annotationType) != null) {
//				按指定的beanName从BeanFactory中查询bean返回
				results.put(beanName, getBean(beanName));
			}
		}
		return results;
	}
```
找到annotation的beans。

beanFactory介绍到这里就介绍完了，这里简单介绍下，单一职责原则设计的接口，AbstractBeanFactory是一般的bean进行处理的BeanFactory抽象实现，AbstractAutowireCapableBeanFactory是依赖注入bean的抽象实现，BeanFactory支持多级实现，其中运用了装饰器设计模式、代理模式、模板方法模式保证了系统架构的扩展性。具体实现类DefaultListableBeanFactory支持BeanDefinition创建bean实例，实现类StaticListableBeanFactory管理bean，可以手动添加bean到BeanFactory，BeanDefinition注册表、单例bean注册表用map实现，BeanDefinition实现过程提供了beanPostProcessor处理器可以对bean进行修改在bean初始化之前，提供初始化前后的前置、后置处理接口，可以对bean在构建BeanDefinition到初始化整个过程进行管理，提高了架构的灵活性，spring之所以可以很方便的整合其他框架优秀之处也在于此。

