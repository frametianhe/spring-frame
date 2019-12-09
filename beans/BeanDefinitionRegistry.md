# BeanDefinitionRegistry

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1575003066401-b863a15b-13a8-4039-b9e3-b857df37e2af.png#align=left&display=inline&height=137&name=image.png&originHeight=273&originWidth=290&size=14342&status=done&style=none&width=145)
org.springframework.core.AliasRegistry顶层接口，别名注册上篇介绍过了。

下级子接口org.springframework.beans.factory.support.BeanDefinitionRegistry，此接口是BeanDefinition注册接口。

```java
void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;
```

org.springframework.beans.factory.support.BeanDefinitionRegistry#registerBeanDefinition 注册一个BeanDefinition到注册表中。

```java
void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
```
在BeanDefinition注册表中删除指定beanName的BeanDefinition。

```java
BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
```
获取BeanDefinition注册表中指定beanName的BeanDefinition。

```java
boolean containsBeanDefinition(String beanName);
```
判断BeanDefinition注册表中是否包含指定beanName的BeanDefinition。

```java
String[] getBeanDefinitionNames();
```
获取BeanDefinition注册表中所有的BeanDefinition名称集合。

```java
boolean isBeanNameInUse(String beanName);
```
在BeanDefinition注册表中是否指定的beanName的BeanDefinition已注册。

org.springframework.beans.factory.support.SimpleBeanDefinitionRegistry提供了
BeanDefinitionRegistry的简单实现。

```java
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(64);
```
内部维护了一个BeanDefinition注册表beanDefinitionMap。

```java
@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "'beanName' must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");
		this.beanDefinitionMap.put(beanName, beanDefinition);
	}
```

```java
@Override
	public void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
		if (this.beanDefinitionMap.remove(beanName) == null) {
			throw new NoSuchBeanDefinitionException(beanName);
		}
	}
```

```java
@Override
	public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
		BeanDefinition bd = this.beanDefinitionMap.get(beanName);
		if (bd == null) {
			throw new NoSuchBeanDefinitionException(beanName);
		}
		return bd;
	}
```

```java
@Override
	public boolean containsBeanDefinition(String beanName) {
		return this.beanDefinitionMap.containsKey(beanName);
	}
```
注册、删除、获取BeanDefinition，判断BeanDefinition注册表中是否包含指定beanName的BeanDefinition都是对beanDefinitionMap BeanDefinition注册表map的操作。
