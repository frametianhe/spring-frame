# AutowireCandidateResolver

![image.png](https://cdn.nlark.com/yuque/0/2019/png/578902/1575270766426-63b0b1e0-2939-4b8a-b8f5-d9c3645c28bf.png#align=left&display=inline&height=159&name=image.png&originHeight=317&originWidth=365&size=21861&status=done&style=none&width=182.5)

自动注入候选对象解析器，策略模式实现。

org.springframework.beans.factory.support.AutowireCandidateResolver 判断一个bean是否可以自动注入依赖的bean

```java
default boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
		return bdHolder.getBeanDefinition().isAutowireCandidate();
	}
```
查询BeanDefinition选项是否自动注入依赖的bean。

```java
default boolean isRequired(DependencyDescriptor descriptor) {
		return descriptor.isRequired();
	}
```
确定bean是否必须注入。

```java
@Nullable
	default Object getSuggestedValue(DependencyDescriptor descriptor) {
		return null;
	}
```
确定是否给依赖设置默认值，默认是null。

```java
@Nullable
	default Object getLazyResolutionProxyIfNecessary(DependencyDescriptor descriptor, @Nullable String beanName) {
		return null;
	}
```
如果需要注入，构建一个为依赖对象延迟策略的代理对象，默认是null。

org.springframework.beans.factory.support.SimpleAutowireCandidateResolver 没有依赖注解可以获取的简单实现。

```java
@Override
	public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
		return bdHolder.getBeanDefinition().isAutowireCandidate();
	}
```
从beanDefinition中判断是够是自动注入。

```java
@Override
	public boolean isRequired(DependencyDescriptor descriptor) {
		return descriptor.isRequired();
	}
```
判断依赖注入的bean是否是必须注入。

```java
@Override
	@Nullable
	public Object getSuggestedValue(DependencyDescriptor descriptor) {
		return null;
	}

	@Override
	@Nullable
	public Object getLazyResolutionProxyIfNecessary(DependencyDescriptor descriptor, @Nullable String beanName) {
		return null;
	}
```
这两个方法和父类一样默认返回null。

org.springframework.beans.factory.support.GenericTypeAwareAutowireCandidateResolver 泛型类型依赖注入解析器

```java
@Override
	public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
		if (!super.isAutowireCandidate(bdHolder, descriptor)) {
			// If explicitly false, do not proceed with any other checks...
			return false;
		}
		return checkGenericTypeMatch(bdHolder, descriptor);
	}
```
是否是自动注入，判断BeanDefinition中选项是否是自动注入，如果不是直接返回，检查给定的泛型类型是够和BeanDefinition中的类型一致，如果一致直接返回。在检查工厂方法返回类型是否一致，如果一致直接返回。

org.springframework.beans.factory.support.GenericTypeAwareAutowireCandidateResolver#getReturnTypeForFactoryMethod

```java
	@Nullable
	protected ResolvableType getReturnTypeForFactoryMethod(RootBeanDefinition rbd, DependencyDescriptor descriptor) {
		// Should typically be set for any kind of factory method, since the BeanFactory
		// pre-resolves them before reaching out to the AutowireCandidateResolver...//通常应设置为任何类型的工厂方法，因为BeanFactory
//在到达AutowireCandidateResolver之前预先解析它们…
		ResolvableType returnType = rbd.factoryMethodReturnType;
		if (returnType == null) {
			Method factoryMethod = rbd.getResolvedFactoryMethod();
			if (factoryMethod != null) {
				returnType = ResolvableType.forMethodReturnType(factoryMethod);
			}
		}
		if (returnType != null) {
			Class<?> resolvedClass = returnType.resolve();
			if (resolvedClass != null && descriptor.getDependencyType().isAssignableFrom(resolvedClass)) {
				// Only use factory method metadata if the return type is actually expressive enough
				// for our dependency. Otherwise, the returned instance type may have matched instead
				// in case of a singleton instance having been registered with the container already.只有在返回类型具有足够的表达能力时才使用工厂方法元数据
//为了我们的依赖。否则，返回的实例类型可能已经匹配
//				如果一个单例实例已经在容器中注册了。
				return returnType;
			}
		}
		return null;
	}
```
返回工厂方法返回的类型。

org.springframework.beans.factory.annotation.QualifierAnnotationAutowireCandidateResolver 自动注入注解解析器，主要解析@Qualifier、@Value注解。

```java
private final Set<Class<? extends Annotation>> qualifierTypes = new LinkedHashSet<>(2);
```
用LinkedHashSet保存自动注入注解的类型。
org.springframework.beans.factory.annotation.QualifierAnnotationAutowireCandidateResolver#isAutowireCandidate 判断BeanDefinition设置是否是自动注入。

```java
@Override
	public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
		boolean match = super.isAutowireCandidate(bdHolder, descriptor);
		if (match) {
//			检查@Qualifier注解
			match = checkQualifiers(bdHolder, descriptor.getAnnotations());
			if (match) {
				MethodParameter methodParam = descriptor.getMethodParameter();
				if (methodParam != null) {
					Method method = methodParam.getMethod();
					if (method == null || void.class == method.getReturnType()) {
						match = checkQualifiers(bdHolder, methodParam.getMethodAnnotations());
					}
				}
			}
		}
		return match;
	}
```
org.springframework.beans.factory.support.GenericTypeAwareAutowireCandidateResolver#isAutowireCandidate

```java
@Override
	public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
		if (!super.isAutowireCandidate(bdHolder, descriptor)) {
			// If explicitly false, do not proceed with any other checks...如果显式为假，则不进行任何其他检查…
			return false;
		}
		return checkGenericTypeMatch(bdHolder, descriptor);
	}
```
检查自动注入的类型是否一致，判断指定的类型和bean的类型和工厂方法返回的类型是够一致。
org.springframework.beans.factory.annotation.QualifierAnnotationAutowireCandidateResolver#checkQualifiers

```java
	protected boolean checkQualifiers(BeanDefinitionHolder bdHolder, Annotation[] annotationsToSearch) {
		if (ObjectUtils.isEmpty(annotationsToSearch)) {
			return true;
		}
		SimpleTypeConverter typeConverter = new SimpleTypeConverter();
		for (Annotation annotation : annotationsToSearch) {
			Class<? extends Annotation> type = annotation.annotationType();
			boolean checkMeta = true;
			boolean fallbackToMeta = false;
			if (isQualifier(type)) {
				if (!checkQualifier(bdHolder, annotation, typeConverter)) {
					fallbackToMeta = true;
				}
				else {
					checkMeta = false;
				}
			}
			if (checkMeta) {
				boolean foundMeta = false;
				for (Annotation metaAnn : type.getAnnotations()) {
					Class<? extends Annotation> metaType = metaAnn.annotationType();
					if (isQualifier(metaType)) {
						foundMeta = true;
						// Only accept fallback match if @Qualifier annotation has a value...
						// Otherwise it is just a marker for a custom qualifier annotation.仅当@Qualifier注释有一个值时才接受回退匹配…
//						否则它只是自定义限定符注释的一个标记。
						if ((fallbackToMeta && StringUtils.isEmpty(AnnotationUtils.getValue(metaAnn))) ||
								!checkQualifier(bdHolder, metaAnn, typeConverter)) {
							return false;
						}
					}
				}
				if (fallbackToMeta && !foundMeta) {
					return false;
				}
			}
		}
		return true;
	}
```
根据BeanDefinition匹配给定的注解类型。

检查beanDefinition和指定的注解类型，org.springframework.beans.factory.annotation.QualifierAnnotationAutowireCandidateResolver#checkQualifier

```java
	protected boolean checkQualifier(
			BeanDefinitionHolder bdHolder, Annotation annotation, TypeConverter typeConverter) {

//		获取注解类型
		Class<? extends Annotation> type = annotation.annotationType();
		RootBeanDefinition bd = (RootBeanDefinition) bdHolder.getBeanDefinition();

//		从BeanDefinition中获取指定注解名称的注解自动注入注解类型
		AutowireCandidateQualifier qualifier = bd.getQualifier(type.getName());
		if (qualifier == null) {
			qualifier = bd.getQualifier(ClassUtils.getShortName(type));
		}
//		如果没有获取到自定注入的注解
		if (qualifier == null) {
			// First, check annotation on qualified element, if any首先，检查限定元素上的注释(如果有的话)
			Annotation targetAnnotation = getQualifiedElementAnnotation(bd, type);
			// Then, check annotation on factory method, if applicable然后，如果适用，检查工厂方法的注释
			if (targetAnnotation == null) {
				targetAnnotation = getFactoryMethodAnnotation(bd, type);
			}
			if (targetAnnotation == null) {
//				获取BeanDefinition被装饰后的beanDefinition
				RootBeanDefinition dbd = getResolvedDecoratedDefinition(bd);
				if (dbd != null) {
//					获取装饰的BeanDefinition工厂方法上的自动注入的注解
					targetAnnotation = getFactoryMethodAnnotation(dbd, type);
				}
			}
			if (targetAnnotation == null) {
				// Look for matching annotation on the target class查找目标类上的匹配注释
				if (getBeanFactory() != null) {
					try {
						Class<?> beanType = getBeanFactory().getType(bdHolder.getBeanName());
						if (beanType != null) {
//							获取cglib子类的父类上的自动注入的注解
							targetAnnotation = AnnotationUtils.getAnnotation(ClassUtils.getUserClass(beanType), type);
						}
					}
					catch (NoSuchBeanDefinitionException ex) {
						// Not the usual case - simply forget about the type check...不是通常的情况-只是忘记类型检查…
					}
				}
//				没有找到自动注入的注解获取BeanDefinition中指定的beanClass cglib子类的父类的自动注入的注解
				if (targetAnnotation == null && bd.hasBeanClass()) {
					targetAnnotation = AnnotationUtils.getAnnotation(ClassUtils.getUserClass(bd.getBeanClass()), type);
				}
			}
			if (targetAnnotation != null && targetAnnotation.equals(annotation)) {
				return true;
			}
		}

//		获取自动注入注解的属性值
		Map<String, Object> attributes = AnnotationUtils.getAnnotationAttributes(annotation);
		if (attributes.isEmpty() && qualifier == null) {
			// If no attributes, the qualifier must be present如果没有属性，限定符必须出现
			return false;
		}
		for (Map.Entry<String, Object> entry : attributes.entrySet()) {
			String attributeName = entry.getKey();
			Object expectedValue = entry.getValue();
			Object actualValue = null;
			// Check qualifier first
			if (qualifier != null) {
				actualValue = qualifier.getAttribute(attributeName);
			}
			if (actualValue == null) {
				// Fall back on bean definition attribute使用bean定义属性
				actualValue = bd.getAttribute(attributeName);
			}
			if (actualValue == null && attributeName.equals(AutowireCandidateQualifier.VALUE_KEY) &&
					expectedValue instanceof String && bdHolder.matchesName((String) expectedValue)) {
				// Fall back on bean name (or alias) match 使用bean名称(或别名)匹配
				continue;
			}
			if (actualValue == null && qualifier != null) {
				// Fall back on default, but only if the qualifier is present返回到默认值，但是只有在限定符出现的情况下
				actualValue = AnnotationUtils.getDefaultValue(annotation, attributeName);
			}
			if (actualValue != null) {
				actualValue = typeConverter.convertIfNecessary(actualValue, expectedValue.getClass());
			}
			if (!expectedValue.equals(actualValue)) {
				return false;
			}
		}
		return true;
	}
```
从beanDefinition中获取指定注解名称的自动注入注解，如果没找到，检查元素上的注解类型，如果还没找到，获取BeanDefinition工厂方法上的注解类型，如果还没找到，获取装饰的BeanDefinition的工厂方法的注解，如果还没获取到，从BeanDefinitionHolder中获取beanName的类型，获取这个类型的父类上的自动注入注解，如果还没获取到从beanDefinition指定的beanClass的父类获取自动注入的注解，注解获取到了获取注解的属性值，获取@Qualifier注解的value值类型转换判断值一致返回true。

```java
@Override
	public boolean isRequired(DependencyDescriptor descriptor) {
		if (!super.isRequired(descriptor)) {
			return false;
		}
		Autowired autowired = descriptor.getAnnotation(Autowired.class);
		return (autowired == null || autowired.required());
	}
```
判断@Autowired注解的required属性值。

```java
@Override
	@Nullable
	public Object getSuggestedValue(DependencyDescriptor descriptor) {
		Object value = findValue(descriptor.getAnnotations());
		if (value == null) {
			MethodParameter methodParam = descriptor.getMethodParameter();
			if (methodParam != null) {
				value = findValue(methodParam.getMethodAnnotations());
			}
		}
		return value;
	}
```
判断给定的依赖项是否声明了@Value注解，如果声明了解析这个注解指定的参数值。

