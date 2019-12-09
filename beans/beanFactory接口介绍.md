# beanFactory接口介绍


![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1574919968142-034b9dbc-8c52-40c7-8672-95fc94bc226d.png#align=left&display=inline&height=230&name=image.png&originHeight=459&originWidth=792&size=42335&status=done&width=396)

BeanFactory顶层接口，spring ioc容器的根接口，子接口有ListableBeanFactory、ConfigurableBeanFactory等，下面将一一介绍，根据BeanDefinition创建单例、原型的对象。

```java
Object getBean(String name) throws BeansException;
```

根据名称返回一个单例或者原型的对象，该方法使用BeanFactory代替单例、原型设计模式，如果在当前factory实例中找不到bean，将去父factory查找实例。

```java
<T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;
```

根据名称和类型返回一个单例或者原型的对象，该方法使用BeanFactory代替单例、原型设计模式，如果在当前factory实例中找不到bean，将去父factory查找实例，如果类型不匹配将抛出BeanNotOfRequiredTypeException。

```java
Object getBean(String name, Object... args) throws BeansException;
```

根据名称和构造参数、工厂方法参数返回一个单例或原型的bean，如果在当前factory实例中找不到bean，将去父factory查找实例。

这里总结下依赖注入常用的几种方式，除了setter、构造参数注入，还有工厂方法注入，面试的时候问很少有人能回答出来第三种，spring还声明了其他的方式也是根据这三种方式延伸而来，开发中最长用的还是这三种，spring建议使用的是构造参数注入，但是构造参数不能太多，要不使得代码比较臃肿。

```java
<T> T getBean(Class<T> requiredType) throws BeansException;
```

根据类型返回单例或者原型的bean，在ListableBeanFactory中查找，也可以转换成按名称进行查找。

```java
<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
```

根据类型和构造方法参数、工厂方法参数返回一个单例或原型的bean，改方法在ListableBeanFactory中查找，也可以转换成按名称查找。

```java
boolean containsBean(String name);
```

判断BeanFactory中是否包单例或原型的bean，当前beanFactory中查找不到会到上一级beanFactory中查找，匹配的bean和getBean返回的bean实例不一定是一致的。

```java
boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
```

根据名称判断是不是单例的bean，当前factory中查找不到会进入上一级factory中查找。
isPrototype(String name)，根据名称判断是不是原型的bean，当前factory中查找不到会进入上一级factory中查找。
这个类中还重载了一些其他的方法，这里不一一介绍了。

```java
String[] getAliases(String name);
```

getAliases(String name)，根据名称查找bean的别名集合。

HierarchicalBeanFactory，BeanFactory的下一级子接口。

```java
BeanFactory getParentBeanFactory();
```

返回上一级beanFactory。

```java
boolean containsLocalBean(String name);
```

判断当前beanFactory中判断是否包含bean，不会去上一级BeanFactory中查找。

HierarchicalBeanFactory的下一级子接口ConfigurableBeanFactory，配置接口实现的BeanFactory，在spring框架内部使用。

```java
void setParentBeanFactory(BeanFactory parentBeanFactory) throws IllegalStateException;
```

设置上一级BeanFactory。

```java
void setConversionService(@Nullable ConversionService conversionService);
```

设置类型转换器。

```java
void addBeanPostProcessor(BeanPostProcessor beanPostProcessor);
```

添加BeanPostProcessor到BeanFactory中，在配置bean的时候，如果有多个执行顺序按添加的顺序执行。

```java
void registerAlias(String beanName, String alias) throws BeanDefinitionStoreException;
```

注册别名。

```java
boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException;
```

判断bean是否实现了factoryBean接口。

```java
void registerDependentBean(String beanName, String dependentBeanName);
```

注册一个依赖的bean给指定的bean，在这个bean销毁之前销毁。

```java
String[] getDependentBeans(String beanName);
```

获取指定bean依赖的bean。

```java
void destroyBean(String beanName, Object beanInstance);
```

销毁bean，destroySingletons销毁BeanFactory中的所有的单例bean。

AutowireCapableBeanFactory beanFactory子接口，能够自动装配的BeanFactory接口。

```java
<T> T createBean(Class<T> beanClass) throws BeansException;
```

初始化bean。

```java
void autowireBean(Object existingBean) throws BeansException;
```

自动注入bean。

```java
Object configureBean(Object existingBean, String beanName) throws BeansException;
```

配置给定的bean。

```java
Object createBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException;
```

使用指定的自动注入策略创建bean。

```java
Object autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException;
```

使用指定自动注入策略初始化bean，如果是按构造参数注入会指定InstantiationAwareBeanPostProcessor，不会执行beanpostprocessor或bean的任何进一步初始化。

```java
void autowireBeanProperties(Object existingBean, int autowireMode, boolean dependencyCheck)
			throws BeansException;
```

按名称或类型自动装配给定bean实例的bean属性，不会执行beanpostprocessor或bean的任何进一步初始化。

```java
	void applyBeanPropertyValues(Object existingBean, String beanName) throws BeansException;
```

将具有给定名称的bean definition的属性值应用于给定的bean实例，不会执行beanpostprocessor或bean的任何进一步初始化。

```java
	Object initializeBean(Object existingBean, String beanName) throws BeansException;
```

初始化给定的原始bean，会执行beanpostprocessor。

```java
Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException;
```

将beanpostprocessor应用于给定的现有bean实例，调用它们的postprocessbefore初始化方法。返回的bean实例可能是原始bean的包装器。

```java
Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException;
```

将beanpostprocessor应用于给定的现有bean实例，调用它们的postprocessafter初始化方法。返回的bean实例可能是原始bean的包装器。

```java
	void destroyBean(Object existingBean);
```

destroyBean(Object existingBean)销毁给定的bean实例。

ListableBeanFactory BeanFactory子接口，可以是一个分级的beanFactory。

```java
	boolean containsBeanDefinition(String beanName);
```

检查BeanFactory是否包含指定的BeanDefinition。

```java
String[] getBeanDefinitionNames();
```
返回beanFactory中所有的beanName。

```java
	String[] getBeanNamesForType(ResolvableType type);
```
返回指定类型的beanName。

```java
	String[] getBeanNamesForType(@Nullable Class<?> type);
```
返回指定类型的beanName。

```java
	String[] getBeanNamesForType(@Nullable Class<?> type, boolean includeNonSingletons, boolean allowEagerInit);
```
返回指定类型的beanName。

```java
	<T> Map<String, T> getBeansOfType(@Nullable Class<T> type) throws BeansException;
```
返回指定类型的bean名称和bean对象。

```java
	String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType);
```
返回指定注解的beanNames。

```java
	Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) throws BeansException;
```
返回指定注解类型的bean名称和bean对象。

```java
<A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
			throws NoSuchBeanDefinitionException;
```
在指定的类型上获取注解，如果找不到就遍历其接口和超类。

ConfigurableListableBeanFactory BeanFactory子接口，配置接口提供的BeanFactory实现。

```java
void ignoreDependencyType(Class<?> type);
```
忽略自动注入的依赖类型。

```java
void ignoreDependencyInterface(Class<?> ifc);
```
忽略指定自动注入的接口。

```java
	void registerResolvableDependency(Class<?> dependencyType, @Nullable Object autowiredValue);
```
根据指定的依赖注入方式注入依赖类型。

```java
	boolean isAutowireCandidate(String beanName, DependencyDescriptor descriptor)
			throws NoSuchBeanDefinitionException;
```
判断bean是否是依赖注入的类型。

```java
	BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
```
返回当前factory中指定beanName的BeanDefinition，不会查找上级BeanFactory。

```java
	void preInstantiateSingletons() throws BeansException;
```
确保实例化了所有非lazy-init单例，如果需要，通常在工厂设置结束时调用。
