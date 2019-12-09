# BeanFactory之AbstractBeanFactory③

org.springframework.beans.factory.support.AbstractBeanFactory#addBeanPostProcessor添加BeanPostProcessor

```java
@Override
	public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
		Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
		this.beanPostProcessors.remove(beanPostProcessor);
		this.beanPostProcessors.add(beanPostProcessor);
//		处理bean再次初始化的，代理、延迟加载的处理器
		if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
			this.hasInstantiationAwareBeanPostProcessors = true;
		}
//		bean销毁的处理器
		if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
			this.hasDestructionAwareBeanPostProcessors = true;
		}
	}
```
beanPostProcessors就保存在beanPostProcessors list中，如果beanPostProcessor是InstantiationAwareBeanPostProcessor类型设置hasInstantiationAwareBeanPostProcessors=true，是DestructionAwareBeanPostProcessor类型设置hasDestructionAwareBeanPostProcessors=true。

org.springframework.beans.factory.support.AbstractBeanFactory#destroyBean(java.lang.String, java.lang.Object)销毁bean

```java
@Override
	public void destroyBean(String beanName, Object beanInstance) {
		destroyBean(beanName, beanInstance, getMergedLocalBeanDefinition(beanName));
	}
```
org.springframework.beans.factory.support.AbstractBeanFactory#destroyBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)

```java
protected void destroyBean(String beanName, Object bean, RootBeanDefinition mbd) {
		new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), getAccessControlContext()).destroy();
	}
```
org.springframework.beans.factory.support.DisposableBeanAdapter#DisposableBeanAdapter(java.lang.Object, java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.util.List<org.springframework.beans.factory.config.BeanPostProcessor>, java.security.AccessControlContext) 创建DisposableBeanAdapter，找到bean的销毁方法和DestructionAwareBeanPostProcessor。

org.springframework.beans.factory.support.DisposableBeanAdapter#destroy

```java
@Override
	public void destroy() {
//		执行beanPostProcessors，beanPostProcessors用对对bean的过程进行处理的抽象
		if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
			for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
//				在bean销毁之前进行一些处理
				processor.postProcessBeforeDestruction(this.bean, this.beanName);
			}
		}

//		如果bean实现了DisposableBean接口
		if (this.invokeDisposableBean) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking destroy() on bean with name '" + this.beanName + "'");
			}
			try {
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((DisposableBean) bean).destroy();
						return null;
					}, acc);
				}
				else {
//					bean实现DisposableBean接口的方式，注解调用子类destroy方法
					((DisposableBean) bean).destroy();
				}
			}
			catch (Throwable ex) {
				String msg = "Invocation of destroy method failed on bean with name '" + this.beanName + "'";
				if (logger.isDebugEnabled()) {
					logger.warn(msg, ex);
				}
				else {
					logger.warn(msg + ": " + ex);
				}
			}
		}

		if (this.destroyMethod != null) {
//			执行bean定义中指定的bean销毁方法
			invokeCustomDestroyMethod(this.destroyMethod);
		}
		else if (this.destroyMethodName != null) {
			Method methodToCall = determineDestroyMethod(this.destroyMethodName);
			if (methodToCall != null) {
				invokeCustomDestroyMethod(methodToCall);
			}
		}
	}
```
执行org.springframework.beans.factory.config.DestructionAwareBeanPostProcessor#postProcessBeforeDestruction方法，在bean销毁之前调用。执行org.springframework.beans.factory.DisposableBean#destroy方法。最后执行bean自定义的销毁方法。
