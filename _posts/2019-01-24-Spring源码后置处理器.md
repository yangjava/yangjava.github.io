---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码后置处理器
BeanPostProcessor 翻译过来就是Bean后处理器。

## BeanPostProcessor
BeanPostProcessor 是 Spring提供给我们的一个非常重要的扩展接口，并且Spring内部的很多功能也是通过 BeanPostProcessor 来完成的(目前看到最典型的就是 AnnotationAwareAspectJAutoProxyCreator 的 注入)。

## BeanPostProcessor 的种类
BeanPostProcessor 在Spring 中的子类非常多(idea 显是有46个)，比如

- InstantiationAwareBeanPostProcessorAdapter ： 在Spring 的bean加载过程中起了非常重要的作用
- AnnotationAwareAspectJAutoProxyCreator ： bean 创建过程中的 属性注入时起作用
- AspectJAwareAdvisorAutoProxyCreator ： Aspect 的 AOP 功能实现也全仰仗BeanPostProcessor 的特性。

## BeanPostProcessor 的创建
个人认为 Bean的 创建时可以认为分为两个过程： 一是Bean对应的BeanDefinition 的创建。二是Bean 实例的创建。

因为在 Spring容器中，Bean的创建并非仅仅通过反射创建就结束了，在创建过程中，需要考虑到Bean针对Spring容器中的一些属性，所以BeanDefinition 中不仅仅包含了 Bean Class 文件信息，还包含了 当前Bean在Spring容器中的一些属性，比如在容器中的作用域、是否懒加载、别名等信息。当Bean 进行实例化创建时需要依赖于对应的BeanDefinition 提供对应的信息。

而由于 BeanPostProcessor 是参与了 Bean 创建过程。所以其创建一定在普通 Bean 之前。实际上 BeanPostProcessor 的创建时在 Spring 启动时容器刷新的时候。

BeanPostProcessor 的 BeanDefinition 创建时机和普通 Bean没有区别，都是在Spring 启动时的BeanFactoryPostProcessor 中完成(确切的说是 ConfigurationClassPostProcessor 中完成)。

而BeanPostProcessor 的实例创建要优先于普通bean创建，Spring启动过程中会调用.AbstractApplicationContext#registerBeanPostProcessors 方法。 在这个方法中，Spring 会从容器中获取到所有BeanPostProcessor 类型的beanName， 通过 beanFactory.getBean 方法获取到对应实例，进行排序后注册到 BeanFactory.beanPostProcessors 属性中，其中 BeanFactory.beanPostProcessors 的定义如下
```
	private final List<BeanPostProcessor> beanPostProcessors = new BeanPostProcessorCacheAwareList();
```
当容器需要执行 BeanPostProcessor 方法时可以直接从 beanPostProcessors 中获取即可。

## 基本介绍
日常使用中，我们一般编写一个类来实现 BeanPostProcessor 或者 InstantiationAwareBeanPostProcessor 接口，根据每个方法的调用时机，来完成响应的工作。

下面介绍一下接口方法，这里通过 InstantiationAwareBeanPostProcessor 接口来介绍 。InstantiationAwareBeanPostProcessor 是 BeanPostProcessor 的子接口，在 BeanPostProcessor 基础上又扩展了三个方法。

```
@Component
public class DemoPostProcesser implements InstantiationAwareBeanPostProcessor {

    //InstantiationAwareBeanPostProcessor 提供的方法， 在 Class<T> -> T 的转换过程中
    // 此时bean还没创建，可以通过这方法代替 Spring 容器创建的方法
    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        System.out.println("DemoPostProcesser.postProcessBeforeInstantiation   ###  1");
        return null;
    }

    //InstantiationAwareBeanPostProcessor 提供的方法， 返回的值代表是否需要继续注入属性，
    // 如果返回true，则会调用postProcessProperties和postProcessPropertyValues 来注入属性
    // 此时bean已经创建，属性尚未注入
    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        System.out.println("DemoPostProcesser.postProcessAfterInstantiation   ###  2");
        return true;
    }
    //InstantiationAwareBeanPostProcessor 提供的方法，可以在这个方法中进行bean属性的注入，Aop 就是在此方法中进行了代理的创建
    // 只有postProcessAfterInstantiation 返回true 时才会调用
    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
        System.out.println("DemoPostProcesser.postProcessProperties   ###  3");
        return pvs;
    }
    //InstantiationAwareBeanPostProcessor 提供的方法，可以在这个方法中进行bean属性的注入, 这个方法已经过时，使用postProcessProperties 代理
    // 只有postProcessAfterInstantiation 返回true 时 且 postProcessProperties 返回 null 时调用
    @Override
    public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        System.out.println("DemoPostProcesser.postProcessPropertyValues   ###  4");
        return pvs;
    }

    // BeanPostProcessor 提供的方法，在bean初始化前调用，这时候的bean大体已经创建完成了，在完成一些其他属性的注入
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("DemoPostProcesser.postProcessBeforeInitialization   ###  5");
        return bean;
    }
    // BeanPostProcessor 提供的方法，在bean初始化后调用，这时候的bean 已经创建完成了
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("DemoPostProcesser.postProcessAfterInitialization   ###  6");
        return bean;
    }
}
```
这里可以看到 postProcessPropertyValues 方法并没有调用，因为对于一个 过时的方法 没必要必须要调用它，前面也提到 postProcessAfterInstantiation 返回true 并且 postProcessProperties 返回不为null 才会调用该方法，这里postProcessProperties 返回不为null ，所以不会调用postProcessPropertyValues 方法

## 源码
Spring 在启动过程中，会将所有实现了 BeanPostProcessor 接口的子类保存到 AbstractBeanFactory 中的 beanPostProcessors 集合中，如下：
```
private final List<BeanPostProcessor> beanPostProcessors = new CopyOnWriteArrayList<>();
```
在适当的时候(这个适当的时候就根据 每个接口方法定义的意义来判断)， Spring会获取 所有的BeanPostProcessor 子类集合，即 beanPostProcessors ，经过类型过滤后，调用对应的方法。比如，就是对 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation 方法的调用流程(其余方法的调用也类似)：

```
	protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
		// 因为只有 InstantiationAwareBeanPostProcessor  类型才有postProcessBeforeInstantiation 方法，所以这里要判断一下是不是 InstantiationAwareBeanPostProcessor   类型
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
				if (result != null) {
					return result;
				}
			}
		}
		return null;
	}
```

### InstantiationAwareBeanPostProcessor
下面来介绍在 Spring 创建流程中 每个方法的实际调用位置：

InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation 方法在 Bean 创建 之前调用，我所理解的目的是替换Spring 容器创建bean， 当 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation 返回不为null时，则不会再通过Spring 创建bean，而是使用 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation 返回的bean。

```
			...
			// 该方法中 调用了postProcessBeforeInstantiation 方法，并且可能调用 postProcessAfterInitialization 方法
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
			...
			// 真正去创建bean
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			...
```
resolveBeforeInstantiation 方法内容如下，可以看到 当 applyBeanPostProcessorsBeforeInstantiation 方法(applyBeanPostProcessorsBeforeInstantiation 调用了 postProcessBeforeInstantiation 方法) 返回值不为 null，则会调用 applyBeanPostProcessorsAfterInitialization 方法，从而调用postProcessAfterInitialization 方法。因为这里bean返回不为null，则代表bean创建成功了，就会调用创建成功后的方法，即 postProcessAfterInitialization。
```
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
```
这里需要注意，在 resolveBeforeInstantiation 方法中，当 bean ！= null 时 调用了 applyBeanPostProcessorsAfterInitialization 方法，即 BeanPostProcessor.postProcessAfterInitialization 方法。这是因为如果 bean != null， 则说明 bean 的创建已经完成，那么这里则是最后调用bean 的后置处理的机会，即调用BeanPostProcessor.postProcessAfterInitialization 方法的最后机会。

InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation 、InstantiationAwareBeanPostProcessor.postProcessProperties 、InstantiationAwareBeanPostProcessor.postProcessPropertyValues 的调用场景只有一处，在AbstractAutowireCapableBeanFactory#populateBean 方法中，此时bean已经创建完成，正在进行属性注入。而 InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation 的返回值决定是否继续注入

大致代码如下
```
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		...
		// 调用 postProcessAfterInstantiation 方法吗，如果返回false，直接return;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						return;
					}
				}
			}
		}

		... 
		if (hasInstAwareBpps) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					// 调用 postProcessProperties 注入属性
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						// 调用 postProcessPropertyValues 注入属性
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
					pvs = pvsToUse;
				}
			}
		}
	...
	}
```
通过代码我们可以比较清楚的看到整体逻辑：

若InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation 返回true，才会执行下面步骤
调用 InstantiationAwareBeanPostProcessor.postProcessProperties 注入属性，
若InstantiationAwareBeanPostProcessor.postProcessProperties 返回为null，才会执行下面步骤
调用 InstantiationAwareBeanPostProcessor.postProcessPropertyValues 注入属性

## BeanPostProcessor
BeanPostProcessor.postProcessBeforeInitialization 调用时机是bean已经创建完成，但是尚未初始化，即一些属性配置尚未完成(我看到的就一个 init-method 的配置)。使用场景也只有一处，即
AbstractAutowireCapableBeanFactory#initializeBean(String, Object, RootBeanDefinition)。

程序走到这里，代表 resolveBeforeInstantiation 方法 中的 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation 并未代理 bean的创建，在这里则是由Spring 创建的bean的时候来调用。
```
	protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		....
		// 调用 BP.postProcessBeforeInitialization
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}
		// 初始化属性，貌似就初始化了 一个 init-method
		...
		if (mbd == null || !mbd.isSynthetic()) {
			// 调用 BP.postProcessAfterInitialization
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

InstantiationAwareBeanPostProcessor.postProcessAfterInitialization 的调用都被封装到 AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization 方法中。
而applyBeanPostProcessorsAfterInitialization 方法的调用

applyBeanPostProcessorsAfterInitialization 方法在下面三个方法中调用：

AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation
AbstractAutowireCapableBeanFactory#initializeBean(String, Object, RootBeanDefinition)
AbstractAutowireCapableBeanFactory#postProcessObjectFromFactoryBean


resolveBeforeInstantiation 方法的调用有如下三处
- AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[]) 这里的调用实际上就是上面讲的 IBP.postProcessBeforeInstantiation 的调用后续，所以这里不再重复
- AbstractAutowireCapableBeanFactory#getSingletonFactoryBeanForTypeCheck
- AbstractAutowireCapableBeanFactory#getNonSingletonFactoryBeanForTypeCheck















