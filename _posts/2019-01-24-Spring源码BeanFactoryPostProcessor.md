---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码工厂后置处理器
本文分析的方法是 AbstractApplicationContext#invokeBeanFactoryPostProcessors

## BeanFactoryPostProcessor & BeanDefinitionRegistryPostProcessor
由于 invokeBeanFactoryPostProcessors 方法中主要就是对BeanFactoryPostProcessor 的处理，所以这里简单的介绍一下 BeanFactoryPostProcessor 及其子接口 BeanDefinitionRegistryPostProcessor。

BeanFactoryPostProcessor 相比较于 BeanPostProcessor 方法是很简单的，只有一个方法，其子接口也就一个方法。 但是他们俩的功能又是类似的，区别就是作用域并不相同。

BeanFactoryPostProcessor的作用域范围是容器级别的。它只和你使用的容器有关。如果你在容器中定义一个BeanFactoryPostProcessor ，它仅仅对此容器中的bean进行后置处理。

BeanFactoryPostProcessor 不会对定义在另一个容器中的bean进行后置处理，即使这两个容器都在同一容器中。

BeanFactoryPostProcessor 可以对 bean的定义(配置元数据)进行处理。Spring IOC 容器允许 BeanFactoryPostProcessor 在容器实际实例化任何其他bean之前读取配置元数据，并有可能修改它，也即是说 BeanFactoryPostProcessor 是直接修改了bean的定义，BeanPostProcessor 则是对bean创建过程中进行干涉。

BeanDefinitionRegistryPostProcessor 和 BeanFactoryPostProcessor 的区别在于：
- BeanDefinitionRegistryPostProcessor .postProcessBeanDefinitionRegistry 方法针对是BeanDefinitionRegistry类型的ConfigurableListableBeanFactory，可以实现对BeanDefinition的增删改查等操作，但是对于非 ConfigurableListableBeanFactory 类型的BeanFactory，并不起作用。
- BeanFactoryPostProcessor .postProcessBeanFactory 针对的是所有的BeanFactory。
- postProcessBeanDefinitionRegistry 的调用时机在postProcessBeanFactory 之前。

## 源码分析

### BeanFactory
需要注意的是，我们这里的 BeanFactory 实际类型是 DefaultListableBeanFactory。

下面我们可以看到DefaultListableBeanFactory 实现了 BeanDefinitionRegistry 接口。

invokeBeanFactoryPostProcessors 方法的作用是激活BeanFactoryPostProcessor 和 BeanDefinitionRegistryPostProcessor 。

为了更好的了解下面的代码，我们先了解几个代码中的规则：

BeanFactoryPostProcessor 在本次分析中分为两种类型： BeanFactoryPostProcessor 和其子接口 BeanDefinitionRegistryPostProcessor 。BeanDefinitionRegistryPostProcessor 相较于 BeanFactoryPostProcessor ，增加了一个方法如下。
```
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry var1) throws BeansException;
}
```
需要注意的是，BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry 这个方法仅仅针对于 BeanDefinitionRegistry 类型的 BeanFactory 生效，这一点根据其入参就可以看到。

BeanFactoryPostProcessor 针对所有的 BeanFactory ，即对于所有类型的BeanFactory 都会调用其方法；BeanDefinitionRegistryPostProcessor 仅对 BeanDefinitionRegistry 子类的BeanFactory 起作用，非BeanDefinitionRegistry类型则直接处理即可。


BeanFactoryPostProcessor 的注入分为两种方式：
- 配置注入方式
即通过注解或者xml的方式动态的注入到容器中的BeanFactoryPostProcessor
- 硬编码注入方式
这种方式是直接调用 AbstractApplicationContext#addBeanFactoryPostProcessor 方法将 BeanFactoryPostProcessor 添加到 AbstractApplicationContext#beanFactoryPostProcessors 属性中。

硬编码注入的BeanFactoryPostProcessor 并不需要也不支持接口排序，而配置注入的方式因为Spring无法保证加载的顺序，所以通过支持PriorityOrdered、Ordered排序接口的排序。

在下面代码分析中会由四个集合
- regularPostProcessors ： 记录通过硬编码方式注册的BeanFactoryPostProcessor 类型的处理器
- registryProcessors：记录通过硬编码方式注册的BeanDefinitionRegistryPostProcessor 类型的处理器
- currentRegistryProcessors ： 记录通过配置方式注册的 BeanDefinitionRegistryPostProcessor 类型的处理器
- processedBeans ： 记录当前已经处理过的BeanFactoryPostProcessor 和 BeanDefinitionRegistryPostProcessor

其实调用顺序可以归纳为： 硬编码先于配置，postProcessBeanDefinitionRegistry 先于postProcessBeanFactory

下面我们来看具体代码：
AbstractApplicationContext#invokeBeanFactoryPostProcessors 方法内容如下
```
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```

可以看到主要功能还是在PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors()); 这一句上。我们先来看看 getBeanFactoryPostProcessors() 得到的是什么

### getBeanFactoryPostProcessors()
```
	private final List<BeanFactoryPostProcessor> beanFactoryPostProcessors = new ArrayList<>();
		
	@Override
	public void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor) {
		Assert.notNull(postProcessor, "BeanFactoryPostProcessor must not be null");
		this.beanFactoryPostProcessors.add(postProcessor);
	}

	/**
	 * Return the list of BeanFactoryPostProcessors that will get applied
	 * to the internal BeanFactory.
	 */
	public List<BeanFactoryPostProcessor> getBeanFactoryPostProcessors() {
		return this.beanFactoryPostProcessors;
	}
```
可以看到 getBeanFactoryPostProcessors() 方法仅仅是将 beanFactoryPostProcessors 集合返回了出去而已。那么 beanFactoryPostProcessors 集合是通过 set方法添加的。这就是我们上面提到过的，beanFactoryPostProcessors 实际上是 硬编码形式注册的BeanDefinitionRegistryPostProcessor 类型的处理器集合。

### invokeBeanFactoryPostProcessors
通过上一步，我们可以知道 入参中的 beanFactoryPostProcessors 集合是硬编码注册的 集合。对于下面的分析我们就好理解了。

下面代码主要是对于 BeanDefinitionRegistry 类型 BeanFactory的处理以及 BeanFactoryPostProcessor 调用顺序问题的处理。实际上并不复杂。
```
	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<>();
		// 对BeanDefinitionRegistry类型的处理，这里是交由BeanDefinitionRegistryPostProcessor来处理
    	// 这里判断BeanFactory 如果是 BeanDefinitionRegistry 子类 则需要进行BeanDefinitionRegistryPostProcessor 的处理，否则直接按照 BeanFactoryPostProcessor处理即可。
    	// 关于为什么BeanDefinitionRegistry 比较特殊上面也说过，因为BeanDefinitionRegistryPostProcessor 只能处理 BeanDefinitionRegistry 的子类，所以这里需要区分是否是 BeanDefinitionRegistry 类型
		if (beanFactory instanceof BeanDefinitionRegistry) {
            // 下面逻辑看似复杂，其实就两步：
            // 1. 获取所有硬编码的 BeanDefinitionRegistryPostProcessor 类型，激活postProcessBeanDefinitionRegistry 方法
            // 2. 获取所有配置的BeanDefinitionRegistryPostProcessor，激活postProcessBeanDefinitionRegistry 方法
            
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
            // 记录通过硬编码方式注册的BeanFactoryPostProcessor 类型的处理器
			List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
          	//  记录通过硬编码方式注册的BeanDefinitionRegistryPostProcessor  类型的处理器
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();
			// 遍历硬编码注册的后处理器(都保存AbstractApplicationContext#beanFactoryPostProcessors 中，这里通过参数beanFactoryPostProcessors传递过来)
			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
                    // 激活 硬编码的处理器的BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry 方法。
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
                    // 保存到 registryProcessors中
					registryProcessors.add(registryProcessor);
				}
				else {
                    // 非BeanDefinitionRegistryPostProcessor 类型的硬编码注入对象 保存到regularPostProcessors中
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
            // 记录通过配置方式注册的 BeanDefinitionRegistryPostProcessor  类型的处理器
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();
            
            // 获取所有的配置的 BeanDefinitionRegistryPostProcessor 的beanName
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            // 筛选出 PriorityOrdered 接口的实现类，优先执行
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                    // 记录到currentRegistryProcessors中
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
            // 进行排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
            // 激活 postProcessBeanDefinitionRegistry 方法
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
             // 筛选出 Ordered 接口的实现类，第二执行
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
            // 排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
            // 激活
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
            // 最后获取没有实现排序接口的 BeanDefinitionRegistryPostProcessor ，进行激活。
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
                // 排序
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
                // 激活
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}
			// 到这里，所有的 BeanDefinitionRegistryPostProcessor 的postProcessBeanDefinitionRegistry 都已经激活结束，开始激活 postProcessBeanFactory 方法
            // registryProcessors 记录的是硬编码注入的BeanDefinitionRegistryPostProcessor，这里激活的是 postProcessBeanFactory 方法
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
            // regularPostProcessors 中记录的是 硬编码注入的 BeanFactoryPostProcessor 
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
            // 如果 beanFactory instanceof BeanDefinitionRegistry = false，那么BeanDefinitionRegistryPostProcessor.的postProcessBeanDefinitionRegistry 并不生效，就直接激活postProcessBeanFactory方法即可。
            // 激活 硬编码注册的 BeanFactoryPostProcessor.postProcessBeanFactory 方法
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}
    	// 到这一步，所有的硬编码方式注入的后处理器都处理完毕，下面开始处理配置注入的后处理器。
    
    	// 获取所有后处理器的beanName,用于后面处理
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
    	// 创建几个保存不同排序的集合，按照实现的排序接口调用
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    	// 排序激活 PriorityOrdered 接口的 后处理器
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    	// 排序激活 Ordered 接口的 后处理器
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
    	// 排序激活 没有实现排序接口的 后处理器
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
    	// 清除缓存。
		beanFactory.clearMetadataCache();
	}
```














