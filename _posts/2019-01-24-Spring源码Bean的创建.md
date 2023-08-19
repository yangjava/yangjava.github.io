---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码Bean的创建
方法是 AbstractAutowireCapableBeanFactory#createBeanInstance，功能是 Spring 具体创建bean的过程。

## createBeanInstance 概述
createBeanInstance 根据方法名就知道，是创建bean的实例，也就注定了这个方法的不平凡。下面就来一步一步的剖析他。

整个方法的源码如下：
```
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		// 解析bean，获取class
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
		// beanClass != null && 当前类不是public && 不允许访问非公共构造函数和方法。抛出异常
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
		// 1. 是否有bean的 Supplier 接口，如果有，通过回调来创建bean
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		// 2. 如果工厂方法不为空，则使用工厂方法初始化策略
		// 通过 @Bean 注解方法注入的bean 或者xml 配置注入 的BeanDefinition 会存在这个值。而注入这个bean的方法就是工厂方法。后面会详细解读
		if (mbd.getFactoryMethodName() != null) {
			// 执行工厂方法，创建bean
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		// 3. 尝试使用构造函数构建bean，后面详解
		// 经过上面两步，Spring确定没有其他方式来创建bean，所以打算使用构造函数来进行创建bean。 但是 bean 的构造函数可能有多个，需要确定使用哪一个。
		// 这里实际上是一个缓存，resolved 表示构造函数是否已经解析完成；autowireNecessary 表示是否需要自动装配
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				// 一个类可能有多个不同的构造函数，每个构造函数参数列表不同，所以调用前需要根据参数锁定对应的构造函数或工程方法
				// 如果这个bean的构造函数或者工厂方法已经解析过了，会保存到 mbd.resolvedConstructorOrFactoryMethod 中。这里来判断是否已经解析过了。
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		// 如果已经解析过则使用功能解析好的构造函数方法，不需要再次解析。这里的是通过 mbd.resolvedConstructorOrFactoryMethod 属性来缓存解析过的构造函数。
		if (resolved) {
			if (autowireNecessary) {
				// 4. 构造函数自动注入
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				// 5. 使用默认构造函数构造
				return instantiateBean(beanName, mbd);
			}
		}

		// Candidate constructors for autowiring?
		// 6. 根据参数解析构造函数，并将解析出来的构造函数缓存到mdb 的  resolvedConstructorOrFactoryMethod  属性中
		// 到这一步，说明 bean 是第一次加载，所以没有对构造函数进行相关缓存(resolved 为 false)
		// 调用 determineConstructorsFromBeanPostProcessors  方法来获取指定的构造函数列表。后面详解
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		// 获取最优的构造函数
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			// 构造函数自动注入
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// No special handling: simply use no-arg constructor.
		// 使用默认构造函数构造
		return instantiateBean(beanName, mbd);
	}
```
我们可以看到：

- 如果RootBeanDefinition 中存在 Supplier 供应商接口，则使用 Supplier 的回调来创建bean。 Supplier是用来替代声明式指定的工厂。
- 如果RootBeanDefinition 中存在 factoryMethodName 属性，或者在配置文件中配置了factory-method，Spring会尝试使用 instantiateUsingFactoryMethod 方法，根据RootBeanDefinition 中的配置生成bean实例。需要注意的是，如果一个类中中的方法被 @Bean注解修饰，那么Spring则会将其封装成一个 ConfigurationClassBeanDefinition。此时 factoryMethodName 也被赋值。所以也会调用instantiateUsingFactoryMethod 方法通过反射完成方法的调用，并将结果注入Spring容器中。
- 当以上两种都没有配置时，Spring则打算通过bean的构造函数来创建bean。首先会判断是否有缓存，即构造函数是否已经被解析过了， 因为一个bean可能会存在多个构造函数，这时候Spring会根据参数列表的来判断使用哪个构造函数进行实例化。但是判断过程比较消耗性能，所以Spring将判断好的构造函数缓存到RootBeanDefinition 中的 resolvedConstructorOrFactoryMethod 属性中。
- 如果存在缓存，则不需要再次解析筛选构造函数，直接调用 autowireConstructor 或者 instantiateBean 方法创建bean。有参构造调用 autowireConstructor 方法，无参构造调用 instantiateBean 方法。
- 如果不存在缓存则需要进行解析，这里通过 determineConstructorsFromBeanPostProcessors 方法调用了 SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors 的后处理器方法来进行解析，Spring 默认的实现在AutowiredAnnotationBeanPostProcessor.determineCandidateConstructors 方法中。通过determineCandidateConstructors 方法获取到了候选的构造函数(因为满足条件的构造函数可能不止一个，需要进行进一步的选择)。
- 获取解析后的候选的构造函数列表 ctors 后(最终的构造函数就从这个列表中选取)，开始调用 autowireConstructor 或者 instantiateBean 方法创建bean。在autowireConstructor 中，进行了候选构造函数的选举，选择最合适的构造函数来构建bean，如果缓存已解析的构造函数，则不用选举，直接使用解析好的构造来进行bean的创建。

## createBeanInstance 详解

### 调用 Supplier 接口 - obtainFromSupplier
这部分的功能 ： 若 RootBeanDefinition 中设置了 Supplier 则使用 Supplier 提供的bean替代Spring要生成的bean
```
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

	...

	protected BeanWrapper obtainFromSupplier(Supplier<?> instanceSupplier, String beanName) {
		Object instance;
		// 这里做了一个类似
		String outerBean = this.currentlyCreatedBean.get();
		this.currentlyCreatedBean.set(beanName);
		try {
			// 从 Supplier 接口中获取 bean实例
			instance = instanceSupplier.get();
		}
		finally {
			if (outerBean != null) {
				this.currentlyCreatedBean.set(outerBean);
			}
			else {
				this.currentlyCreatedBean.remove();
			}
		}

		if (instance == null) {
			instance = new NullBean();
		}
		// 包装成 BeanWrapper 
		BeanWrapper bw = new BeanWrapperImpl(instance);
		initBeanWrapper(bw);
		return bw;
	}
```
关于其应用场景 ： 指定一个用于创建bean实例的回调，以替代声明式指定的工厂方法。主要是考虑反射调用目标方法不如直接调用目标方法效率高。

