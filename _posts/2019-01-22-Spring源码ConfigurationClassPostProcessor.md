---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码ConfigurationClassPostProcessor
ConfigurationClassPostProcessor 是非常重要的一个 后处理器。 ConfigurationClassPostProcessor 完成了 配置类的解析和保存以及@Component 注解、@Import 等注解的解析工作 。将所有需要注入的bean解析成 BeanDefinition保存到 BeanFactory 中。

## ConfigurationClassPostProcessor
首先来讲解一下 ConfigurationClassPostProcessor 的结构图如下。

可见ConfigurationClassPostProcessor 接口实现了BeanDefinitionRegistryPostProcessor(BeanFactory 的后处理器) PriorityOrdered(设置自己的优先级为最高) 和各种 Aware 接口。

在 Springboot启动后，会通过 SpringApplication#createApplicationContext 来创建应用上下文，默认请情况下我们一般创建 AnnotationConfigServletWebServerApplicationContext 作为应用上下文。而在AnnotationConfigServletWebServerApplicationContext 构造函数中会创建 AnnotatedBeanDefinitionReader。而在 AnnotatedBeanDefinitionReader 构造函数中会调用 AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);，该方法将一些必要Bean(如ConfigurationClassPostProcessor、AutowiredAnnotationBeanPostProcessor、CommonAnnotationBeanPostProcessor 等)注入到了容器中。

我们这里重点看的是 BeanDefinitionRegistryPostProcessor 接口的两个方法：

## BeanDefinitionRegistryPostProcessor
```
// 完成对 @Bean 方法的代理
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
// 允许在Spring容器启动后，在下一个阶段开始前，添加BeanDefinition的定义
void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
```
调用过程主要是在 Spring容器刷新的过程中，其中 postProcessBeanDefinitionRegistry 方法先于 postProcessBeanFactory 方法被调用。

ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry 方法。得知了ConfigurationClassPostProcessor 解析配置类(这里的配置类不仅仅局限于@Configuration 注解，还包括 @Import、 @ImportResource 等注解)，将解析到的需要注入到Spring容器中的bean的BeanDefinition保存起来。在后面的bean 初始化都需要BeanDefinition。

## ConfigurationClassPostProcessor
关注 ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry 方法的解析。所以我们下面来看看 postProcessBeanDefinitionRegistry 方法
```
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
		.... 省略部分代码
		// 关键方法，解析 配置类的定义
		processConfigBeanDefinitions(registry);
	}
```
可以看到 postProcessBeanDefinitionRegistry 方法中并没有处理什么逻辑，真正逻辑在其调用的 processConfigBeanDefinitions 方法中

## processConfigBeanDefinitions
processConfigBeanDefinitions 方法完成了关于配置类的所有解析。

processConfigBeanDefinitions 详细代码如下：
```
	public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		// 获取已经解析的BeanName。这里需要注意的是，Springboot的话，启动类已经被注册。具体的注册时机是在  Springboot启动时候的 SpringApplication#prepareContext方法中。
		String[] candidateNames = registry.getBeanDefinitionNames();
		// 遍历BeanName
		for (String beanName : candidateNames) {
			// 获取BeanDefinition 
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			// 如果bean被解析过(Bean 被解析后会在beanDef 中设置属性 CONFIGURATION_CLASS_ATTRIBUTE )，if 属性成立，这里是为了防止重复解析
			if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			// 1. ConfigurationClassUtils.checkConfigurationClassCandidate 解析了当前bean是否是配置类，关于其详细内容，后面解析 需要注意的是，本文所说的配置类即使满足 full 或 lite 条件的类，而不仅仅是被 @Configuration 修饰的类。
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				// 添加到配置类集合中
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// Return immediately if no @Configuration classes were found
		// 如果没有找到配置类，则直接返回，不需要下面的解析
		if (configCandidates.isEmpty()) {
			return;
		}

		// Sort by previously determined @Order value, if applicable
		// 按照@Order 注解进行排序(如果使用了 @Order 注解的话)
		configCandidates.sort((bd1, bd2) -> {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return Integer.compare(i1, i2);
		});

		.// Detect any custom bean name generation strategy supplied through the enclosing application context
		// 判断如果是 registry  是 SingletonBeanRegistry 类型，则从中获取 beanName 生成器(BeanNameGenerator )。实际上这里是 register 类型是 DefaultListableBeanFactory。是 SingletonBeanRegistry  的子类
		SingletonBeanRegistry sbr = null;
		if (registry instanceof SingletonBeanRegistry) {
			sbr = (SingletonBeanRegistry) registry;
			if (!this.localBeanNameGeneratorSet) {
				BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
						AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
				if (generator != null) {
					this.componentScanBeanNameGenerator = generator;
					this.importBeanNameGenerator = generator;
				}
			}
		}
		// 如果环境变量为空则指定一个标准环境，这里是 StandardServletEnvironment 类型，在前面的启动篇我们可以知道。
		if (this.environment == null) {
			this.environment = new StandardEnvironment();
		}
	
		// Parse each @Configuration class
		// 下面开始解析每一个配置类
		// 准备配置类的解析类ConfigurationClassParser 
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);
		// 用来保存尚未解析的配置类
		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		// 用来保存已经解析的配置类
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		// do..while 循环解析。因为一个配置类可能引入另一个配置类，需要循环解析，直至没有其他需要解析的类。
		do {
			// 2. 开始解析。后面详细分析
			parser.parse(candidates);
			// 3. 这里的校验规则是如果是被 @Configuration修饰且proxyBeanMethods属性为true,则类不能为final。如果@Bean修饰的方法，则必须是可覆盖的.
			// 因为@Configuration(proxyBeanMethods = true) 是需要cglib代理的，所以不能为终态， @Bean所修饰的方法也有一套约束规则，下面详细讲
			// 是否需要代理是根据 类或方法上的 @Scope 注解指定的，默认都是不代理
			parser.validate();
			// configClasses  保存这次解析出的配置类。此时这些ConfigurationClass 中保存了解析出来的各种属性值，等待最后构建 BeanDefinition
			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			// 去除已经解析过的配置类
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			// 4. 注册bean
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);

			candidates.clear();
			// if 如果成立，说明有新的bean注册了，则需要解析新的bean
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				// 获取新的beanName
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<>();
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}
				for (String candidateName : newCandidateNames) {
					if (!oldCandidateNames.contains(candidateName)) {
						// 过滤出未解析的bean检测是否是未解析过的配置类
						BeanDefinition bd = registry.getBeanDefinition(candidateName);
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(bd.getBeanClassName())) {
							// 如果是未解析的配置类，则保存到candidates中
							candidates.add(new BeanDefinitionHolder(bd, candidateName));
						}
					}
				}
				candidateNames = newCandidateNames;
			}
		}
		// 如果 candidates 不为空，则说明有未被解析的配置类，循环解析。
		while (!candidates.isEmpty());

		// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
			// 到这里已经把配置类解析完毕了。
			// 将ImportRegistry  注册为 bean，以支持ImportAware @Configuration 类
		if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
			sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
		}
		// 清除缓存
		if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
			// Clear cache in externally provided MetadataReaderFactory; this is a no-op
			// for a shared cache since it'll be cleared by the ApplicationContext.
			((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
		}
	}
```
这里简单总结一下流程;

- 获取已经注册的Bean, 并筛选出配置类，按照@Order 进行排序，得到配置类集合 configCandidates
- 调用 parser.parse(candidates); 对配置类进行解析
- 调用 this.reader.loadBeanDefinitions(configClasses); 进行配置类的注册
- 检验 registry.getBeanDefinitionCount() > candidateNames.length 是否成立。这里由于第三步会将新解析出来的bean进行注册，如果这里成立，则说明有新的配置类完成了注册，获取到新注册的配置类candidateNames。循环从第二步重新解析，直到没有新注入的配置类。

### checkConfigurationClassCandidate
在 processConfigBeanDefinitions 方法中。判断一个类是否是配置类就是通过 checkConfigurationClassCandidate 方法来判断的，那么我们需要看看这个方法中是怎么实现的。

在这个方法里，关键的部分是 给 BeanDefinition 设置了CONFIGURATION_CLASS_ATTRIBUTE 为 full 或者 lite 设置这两个属性标识，如果一个类满足full或 lite的条件，则会被认为是配置类。需要注意的是，本文所说的配置类即使满足 full 或 lite 条件的类，而不仅仅是被 @Configuration 修饰的类。

首先需要注意的是，在 checkConfigurationClassCandidate 中，配置类的类型分为两种，Full 和 Lite，即完整的配置类和精简的配置类。

full 和 lite 设置的规则如下：
- Full : 即类被 @Configuration 注解修饰 && proxyBeanMethods属性为true (默认为 true)
- Lite : 被 @Component、@ComponentScan、@Import、@ImportResource 修饰的类 或者 类中有被@Bean修饰的方法。

Full 配置类就是我们常规使用的配置类

Lite 配置类就是一些需要其他操作引入一些bean 的类

下面我们来看具体代码：
```
	public static boolean checkConfigurationClassCandidate(
			BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {
		// 获取className
		String className = beanDef.getBeanClassName();
		if (className == null || beanDef.getFactoryMethodName() != null) {
			return false;
		}

		// 解析关于当前被解析类的 注解元数据
		AnnotationMetadata metadata;
		// 如果当前BeanDefinition  是 AnnotatedBeanDefinition(相较于一般的 BeanDefinition，他多了一些注解信息的解析) 类型。直接获取注解元数据即可
		if (beanDef instanceof AnnotatedBeanDefinition &&
				className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
			// Can reuse the pre-parsed metadata from the given BeanDefinition...
			metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
		}
		
		else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
			// Check already loaded Class if present...
			// since we possibly can't even load the class file for this Class.
			
			Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
			// 如果当前类是 BeanFactoryPostProcessor、BeanPostProcessor、AopInfrastructureBean、EventListenerFactory 类型不当做配置类处理，返回false
			if (BeanFactoryPostProcessor.class.isAssignableFrom(beanClass) ||
					BeanPostProcessor.class.isAssignableFrom(beanClass) ||
					AopInfrastructureBean.class.isAssignableFrom(beanClass) ||
					EventListenerFactory.class.isAssignableFrom(beanClass)) {
				return false;
			}
			// 获取数据
			metadata = AnnotationMetadata.introspect(beanClass);
		}
		else {
		// 按照默认规则解析
			try {
				MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
				metadata = metadataReader.getAnnotationMetadata();
			}
			catch (IOException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Could not find class file for introspecting configuration annotations: " +
							className, ex);
				}
				return false;
			}
		}

		
		// 获取bean上的Configuration 注解的属性。如果没有被 @Configuration 修饰 config 则为null
		Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
		// 如果被 @Configuration 修饰 &&  proxyBeanMethods 属性为 true。 @Configuration 的 proxyBeanMethods  属性默认值即为 true。
		if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
			// 设置 CONFIGURATION_CLASS_ATTRIBUTE 为 full
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
		}
		// 如果被 @Configuration 修饰 &&  isConfigurationCandidate(metadata) = true
		// 关于  isConfigurationCandidate(metadata) 的解析在下面
		else if (config != null || isConfigurationCandidate(metadata)) {
			// 设置 CONFIGURATION_CLASS_ATTRIBUTE 为 lite
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
		}
		else {
			return false;
		}

		// It's a full or lite configuration candidate... Let's determine the order value, if any.
		// 按照@Order 注解排序
		Integer order = getOrder(metadata);
		if (order != null) {
			beanDef.setAttribute(ORDER_ATTRIBUTE, order);
		}

		return true;
	}
```

### isConfigurationCandidate
在上面的代码中，我们看到 判断是否是 Lite 的关键方法是 isConfigurationCandidate。其代码如下：

```
	// candidateIndicators  的定义
	private static final Set<String> candidateIndicators = new HashSet<>(8);

	static {
		candidateIndicators.add(Component.class.getName());
		candidateIndicators.add(ComponentScan.class.getName());
		candidateIndicators.add(Import.class.getName());
		candidateIndicators.add(ImportResource.class.getName());
	}	

	public static boolean isConfigurationCandidate(AnnotationMetadata metadata) {
		// Do not consider an interface or an annotation...
		// 不能是接口
		if (metadata.isInterface()) {
			return false;
		}

		// Any of the typical annotations found?
		// 被 candidateIndicators 中的注解修饰。其中 candidateIndicators  注解在静态代码块中加载了
		for (String indicator : candidateIndicators) {
			if (metadata.isAnnotated(indicator)) {
				return true;
			}
		}

		// Finally, let's look for @Bean methods...
		try {
			// 类中包含被 @Bean 注解修饰的方法
			return metadata.hasAnnotatedMethods(Bean.class.getName());
		}
		catch (Throwable ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to introspect @Bean methods on class [" + metadata.getClassName() + "]: " + ex);
			}
			return false;
		}
	}
```

## parser.parse(candidates);
上面解析了如何判断一个类是否是配置类。也完成了配置类的筛选。那么开始进行配置类的解析，在 processConfigBeanDefinitions 方法中，对配置类的解析也只是一句话完成:
```
	parser.parse(candidates);
```
parser.parse(candidates); 的作用是：

将所有的配置类保存到 ConfigurationClassParser#configurationClasses 集合中
```
	private final Map<ConfigurationClass, ConfigurationClass> configurationClasses = new LinkedHashMap<>();
```
解析注解并赋值给每个 ConfigurationClass 对应的属性。如解析 @Import 注解，并通过如下语句将结果保存到 ConfigurationClass.importBeanDefinitionRegistrars 集合中。
```
configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
```
同样的还有 将@ ImportResource 注解保存到ConfigurationClass.importedResources中，将@Bean 修饰的方法 和接口静态方法保存到ConfigurationClass.beanMethods 中。

而在之后的 this.reader.loadBeanDefinitions(configClasses); 中才进行了这些属性的进一步处理

下面我们来具体看代码，其代码如下：
```
	public void parse(Set<BeanDefinitionHolder> configCandidates) {
		for (BeanDefinitionHolder holder : configCandidates) {
			BeanDefinition bd = holder.getBeanDefinition();
			try {
				// 针对不同类型的 BeanDefinition 做一些处理
				if (bd instanceof AnnotatedBeanDefinition) {
					parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
				}
				else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
					parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
				}
				else {
					parse(bd.getBeanClassName(), holder.getBeanName());
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
			}
		}
		// 在这里调用了 AutoConfigurationImportSelector 完成了Springboot的自动化装配
		this.deferredImportSelectorHandler.process();
	}
```
里面的 parse 方法殊途同归，最终都会调用 processConfigurationClass 方法，所以我们直接进入 processConfigurationClass 方法：
```
	protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
		// 判断是否应该跳过当前类的解析。这里面解析了 @Conditional 注解
		if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
			return;
		}
		// 判断是否已经解析过。configurationClasses 中保存着已经解析过的配置类。在下面解析过的类都会被保存到 configurationClasses 中
		// 这里应该是 注入的配置类优先级高于引入的配置类
		// 如果配置类被多次引入则合并属性
		ConfigurationClass existingClass = this.configurationClasses.get(configClass);
		if (existingClass != null) {
			// 一个类被重复解析，那么可能被重复引入了，可能是通过 @Import 注解或者嵌套在其他配置类中被引入。如果这两者都是通过这种方式被引入，那么则进行引入合并
			// 如果当前配置类和之前解析过的配置类都是引入的，则直接合并
			if (configClass.isImported()) {
				if (existingClass.isImported()) {
					existingClass.mergeImportedBy(configClass);
				}
				// Otherwise ignore new imported config class; existing non-imported class overrides it.
				// 否则，忽略新导入的配置类；现有的非导入类将覆盖它
				return;
			}
			else {
				// Explicit bean definition found, probably replacing an import.
				// Let's remove the old one and go with the new one.
				// 如果当前的配置类不是引入的，则移除之前的配置类，重新解析
				this.configurationClasses.remove(configClass);
				this.knownSuperclasses.values().removeIf(configClass::equals);
			}
		}

		// Recursively process the configuration class and its superclass hierarchy.
		SourceClass sourceClass = asSourceClass(configClass, filter);
		do {
			sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
		}
		while (sourceClass != null);
		// 保存解析过的 配置类
		this.configurationClasses.put(configClass, configClass);
	}
```
这里需要注意的是配置类的重复引入优先级的问题 ：
一般来说，Spring有一个自己的规则 ：自身注入方式 优先于 引入方式。这里的引入方式指的被 @Import 或者其他配置类引入。当一个类被多次引入时，会使用自身注入的方式的bean 替代 被引入方式的bean。

如果二者都是引入方式，则进行合并(在 ConfigurationClass 类中有一个importedBy 集合，将新引入的来源保存到 importedBy 中)

看了这么久的源码，也知道了Spring的套路，方法名以do开头的才是真正做事的方法, 所以我们来看 doProcessConfigurationClass 方法。
```
	@Nullable
	protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException {
		// 1. 处理 @Component 注解
		if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// Recursively process any member (nested) classes first
			processMemberClasses(configClass, sourceClass, filter);
		}

		// Process any @PropertySource annotations
		// 2. 处理 @PropertySource 注解
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
			}
			else {
				logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
						"]. Reason: Environment must implement ConfigurableEnvironment");
			}
		}

		// Process any @ComponentScan annotations
		// 3. 处理 @ComponentScan注解
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

		// Process any @Import annotations
		// 4. 处理 @Import 注解
		processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

		// Process any @ImportResource annotations
		// 5. 处理 @ImportResource 注解
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}

		// Process individual @Bean methods
		// 6. 处理 @Bean修饰的方法
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}

		// Process default methods on interfaces
		// 7. 处理其他默认接口方法
		processInterfaces(configClass, sourceClass);

		// Process superclass, if any
		// 处理父类，如果存在
		if (sourceClass.getMetadata().hasSuperClass()) {
			String superclass = sourceClass.getMetadata().getSuperClassName();
			if (superclass != null && !superclass.startsWith("java") &&
					!this.knownSuperclasses.containsKey(superclass)) {
				this.knownSuperclasses.put(superclass, configClass);
				// Superclass found, return its annotation metadata and recurse
				return sourceClass.getSuperClass();
			}
		}

		// No superclass -> processing is complete
		return null;
	}
```
doProcessConfigurationClass 方法中的逻辑很清楚，因为他把大部分的逻辑直接封装成了方法。下面我们就来一个一个分析。

## 处理 @Component 注解
这里对 @Component 的处理其实是处理配置类的内部类，即如果当前类是被 @Component 修饰，则需要判断其内部类是否需要解析。
```
		// 首先判断如果配置类被@Component 修饰，则调用processMemberClasses 方法处理
		if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// Recursively process any member (nested) classes first
			processMemberClasses(configClass, sourceClass, filter);
		}
```
processMemberClasses 方法的代码如下：
代码逻辑也很简单。即如果配置类中有内部类，则判断其内部类是否是配置类，如果是则递归去解析新发现的内部配置类。
```
	private void processMemberClasses(ConfigurationClass configClass, SourceClass sourceClass,
			Predicate<String> filter) throws IOException {
		// 获取内部类
		Collection<SourceClass> memberClasses = sourceClass.getMemberClasses();
		if (!memberClasses.isEmpty()) {
			// 如果有内部类，则遍历内部类，判断内部类是否是配置类，如果是，则添加到 candidates 集合中。
			List<SourceClass> candidates = new ArrayList<>(memberClasses.size());
			for (SourceClass memberClass : memberClasses) {
				// 这里判断的是是否是lite 类型的配置类
				if (ConfigurationClassUtils.isConfigurationCandidate(memberClass.getMetadata()) &&
						!memberClass.getMetadata().getClassName().equals(configClass.getMetadata().getClassName())) {
					candidates.add(memberClass);
				}
			}
			// 进行排序
			OrderComparator.sort(candidates);
			for (SourceClass candidate : candidates) {
				// importStack 用来缓存已经解析过的内部类，这里处理循环引入问题。
				if (this.importStack.contains(configClass)) {
					// 打印循环引用异常
					this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
				}
				else {
					// 解析前入栈，防止循环引入
					this.importStack.push(configClass);
					try {
						// 递归去解析新发现的配置类
						processConfigurationClass(candidate.asConfigClass(configClass), filter);
					}
					finally {
						// 解析完毕出栈
						this.importStack.pop();
					}
				}
			}
		}
	}
```
判断内部类是否是配置类，使用的方法是 ConfigurationClassUtils.isConfigurationCandidate，这里是检测内部类是否满足lite 的配置类规则，并未校验 full的规则。

代码中使用了this.importStack 来防止递归引入。避免了A引入B，B又引入A这种无限循环的情况。

## 处理 @PropertySource 注解
@PropertySource 注解可以引入配置文件使用。在这里进行 @PropertySource 注解的解析，将引入的配置文件加载到环境变量中
```
	// 去重后遍历 PropertySource 注解所指向的属性。注意这里有两个注解@PropertySources 和 @PropertySource。
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), PropertySources.class,
			org.springframework.context.annotation.PropertySource.class)) {
		if (this.environment instanceof ConfigurableEnvironment) {
			// 解析PropertySource  注解
			processPropertySource(propertySource);
		}
		else {
			logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
					"]. Reason: Environment must implement ConfigurableEnvironment");
		}
	}
```
processPropertySource 代码如下，在这里解析每一个@PropertySource 注解属性 :
```
	private void processPropertySource(AnnotationAttributes propertySource) throws IOException {
		// 获取 @PropertySource 注解的各个属性
		String name = propertySource.getString("name");
		if (!StringUtils.hasLength(name)) {
			name = null;
		}
		String encoding = propertySource.getString("encoding");
		if (!StringUtils.hasLength(encoding)) {
			encoding = null;
		}
		// 获取指向的文件路径
		String[] locations = propertySource.getStringArray("value");
		Assert.isTrue(locations.length > 0, "At least one @PropertySource(value) location is required");
		boolean ignoreResourceNotFound = propertySource.getBoolean("ignoreResourceNotFound");

		Class<? extends PropertySourceFactory> factoryClass = propertySource.getClass("factory");
		PropertySourceFactory factory = (factoryClass == PropertySourceFactory.class ?
				DEFAULT_PROPERTY_SOURCE_FACTORY : BeanUtils.instantiateClass(factoryClass));
		// 遍历文件路径
		for (String location : locations) {
			try {
				//  根据路径获取到资源文件并保存到environment 中
				// 解决占位符，获取真正路径
				String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);
				Resource resource = this.resourceLoader.getResource(resolvedLocation);
				//保存 PropertySource 到 environment 中
				addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
			}
			catch (IllegalArgumentException | FileNotFoundException | UnknownHostException ex) {
				// Placeholders not resolvable or resource not found when trying to open it
				if (ignoreResourceNotFound) {
					if (logger.isInfoEnabled()) {
						logger.info("Properties location [" + location + "] not resolvable: " + ex.getMessage());
					}
				}
				else {
					throw ex;
				}
			}
		}
	}
```

## 处理 @ComponentScan、@ComponentScans 注解
@componentScans 指定自动扫描的路径。

```
		// 这里会将 @ComponentScans 中的多个 @ComponentScan 也解析出来封装成一个个AnnotationAttributes对象
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		// 如果当前配置类被 @componentScans 或 @componentScan 注解修饰 && 不应跳过
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
				// 遍历 @ComponentScans、 @ComponentScan
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				// 直接执行扫描，根据指定路径扫描出来bean。
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				// 遍历扫描出来的bean
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				// 获取原始的bean的定义
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					// 检测如果是配置类，则递归调用 parse 解析。
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}
```
这里需要注意 :

- this.componentScanParser.parse 方法完成了指定路径下的bean的扫描，这里不再具体分析。详参：Spring 源码分析补充篇二 ：ClassPathBeanDefinitionScanner#doScan
- 这里校验是否是配置类调用的是 checkConfigurationClassCandidate 方法，即校验了 full或lite的规则，和 处理 @Component 中的内部类的规则并不相同。
- 没错，又是递归，如果扫描到的bean中发现了新的配置类，则递归去解析。
- 之前的我们提过，Springboot 在启动过程中将 启动类注册到了容器中，那么在这里进行递归遍历的时候就会通过启动类指定的默认路径来进行遍历， 完成了Springboot的启动注册。

## 处理 @Import、ImportSelector、ImportBeanDefinitionRegistrar
processImports(configClass, sourceClass, getImports(sourceClass), filter, true); 该方法处理的包括 @Import、ImportSelector、 ImportBeanDefinitionRegistrar。这三个注解或接口都可以完成Bean的引入功能。

- @Import
可以通过 @Import(XXX.class) 的方式，将指定的类注册到容器中
- ImportSelector
Spring会将 ImportSelector#selectImports 方法返回的内容通过反射加载到容器中
- ImportBeanDefinitionRegistrar
可以通过 registerBeanDefinitions 方法声明BeanDefinition 并自己注册到Spring容器中 比如 ： MyBatis 中的 AutoConfiguredMapperScannerRegistrar 对@Mapper 修饰类的注册过程

需要注意的是，这里解析的ImportSelector、ImportBeanDefinitionRegistrar 都是通过 @Import 注解引入的。如果不是通过 @Import 引入(比如直接通过@Component 将ImportSelector、ImportBeanDefinitionRegistrar 注入)的类则不会被解析。

注意 getImports(sourceClass) 方法的作用是解析 @Import 注解

我们直接来看 processImports 方法，注释都比较清楚 :
```
	private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
			boolean checkForCircularImports) {
		// importCandidates 是通过getImports() 方法解析 @Import 注解而来， 如果为空则说明没有需要引入的直接返回
		if (importCandidates.isEmpty()) {
			return;
		}
		// 检测是否是循环引用。
		if (checkForCircularImports && isChainedImportOnStack(configClass)) {
			this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
		}
		else {
			// 解析前先入栈，防止循环引用
			this.importStack.push(configClass);
			try {
				for (SourceClass candidate : importCandidates) {
					// 判断是否是ImportSelector类型。ImportSelector 则需要调用selectImports 方法来获取需要注入的类。
					if (candidate.isAssignable(ImportSelector.class)) {
						// Candidate class is an ImportSelector -> delegate to it to determine imports
						Class<?> candidateClass = candidate.loadClass();
						ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
								this.environment, this.resourceLoader, this.registry);
						Predicate<String> selectorFilter = selector.getExclusionFilter();
						if (selectorFilter != null) {
							exclusionFilter = exclusionFilter.or(selectorFilter);
						}
						if (selector instanceof DeferredImportSelector) {
							this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
						}
						else {
						// 调用 selectImports 方法获取需要引入的类，并递归再次处理。
							String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
							Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
							// 递归解析
							processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
						}
					}
					// 如果是 ImportBeanDefinitionRegistrar 类型，则委托它注册其他bean定义
					else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
						// Candidate class is an ImportBeanDefinitionRegistrar ->
						// delegate to it to register additional bean definitions
						Class<?> candidateClass = candidate.loadClass();
						ImportBeanDefinitionRegistrar registrar =
								ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
										this.environment, this.resourceLoader, this.registry);
						configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
					}
					else {
						// Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
						// process it as an @Configuration class
						this.importStack.registerImport(
								currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
						// 否则递归处理需要引入的类。
						processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
					}
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to process import candidates for configuration class [" +
						configClass.getMetadata().getClassName() + "]", ex);
			}
			finally {
				this.importStack.pop();
			}
		}
	}
```

## 处理 @ImportResource 注解
@ImportResource 就显得很简单了，直接保存到 configClass 中
```
	// Process any @ImportResource annotations
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}
```

### 处理 @Bean修饰的方法
@Bean 也很简单了，直接保存到 configClass 的中
```
// Process individual @Bean methods
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}
```

### 处理接口默认方法
这里是 检测 配置类实现的接口中的默认方法是否被@Bean修饰，如果被修饰则也需要保存到 configClass 中
```
	/**
	 * Register default methods on interfaces implemented by the configuration class.
	 */
	private void processInterfaces(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {
		for (SourceClass ifc : sourceClass.getInterfaces()) {
			Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(ifc);
			for (MethodMetadata methodMetadata : beanMethods) {
				if (!methodMetadata.isAbstract()) {
					// A default method or other concrete method on a Java 8+ interface...
					configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
				}
			}
			processInterfaces(configClass, ifc);
		}
	}
```

### 处理父类
如果存在父类，则将父类返回，对父类进行解析。
```
		// Process superclass, if any
		if (sourceClass.getMetadata().hasSuperClass()) {
			String superclass = sourceClass.getMetadata().getSuperClassName();
			if (superclass != null && !superclass.startsWith("java") &&
					!this.knownSuperclasses.containsKey(superclass)) {
				this.knownSuperclasses.put(superclass, configClass);
				// Superclass found, return its annotation metadata and recurse
				return sourceClass.getSuperClass();
			}
		}
```
这里这么处理是解析到最上层的父类。这里理一下调用顺序：parse -> processConfigurationClass -> doProcessConfigurationClass 。而 doProcessConfigurationClass 有如下一个循环，只有sourceClass = null 才会跳出循环。当 configClass 没有满足上面判断条件的父类时，才会返回null
```
	SourceClass sourceClass = asSourceClass(configClass, filter);
		do {
			sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
		}
		while (sourceClass != null);
	this.configurationClasses.put(configClass, configClass);
```

## parser.validate();
到了 这一步，是对解析出来的配置类进行进一步的校验，确保没有问题

这里我们看看其代码如下
```
	public void validate() {
		for (ConfigurationClass configClass : this.configurationClasses.keySet()) {
			configClass.validate(this.problemReporter);
		}
	}
```
这里可以看到是调用每个 ConfigurationClass 类的 validate 方法进行校验，我们进去看看 ConfigurationClass#validate 的代码 ：

```
	public void validate(ProblemReporter problemReporter) {
		// A configuration class may not be final (CGLIB limitation) unless it declares proxyBeanMethods=false
		// 获取 @Configuration 注解的属性信心
		Map<String, Object> attributes = this.metadata.getAnnotationAttributes(Configuration.class.getName());
		// 如果 @Configuration 存在(attributes != null)  && attributes.get("proxyBeanMethods") == true 才进行进一步的校验
		if (attributes != null && (Boolean) attributes.get("proxyBeanMethods")) {
			// 如果配置类 是  final 修饰，即终态类，则是错误，因为无法动态代理
			if (this.metadata.isFinal()) {
				problemReporter.error(new FinalConfigurationProblem());
			}
			// 对配置类中的 @Bean 注解修饰的方法进行校验
			for (BeanMethod beanMethod : this.beanMethods) {
				beanMethod.validate(problemReporter);
			}
		}
	}
```

这里我们再来看看 @Bean方法的校验 BeanMethod#validate如下：

```
	@Override
	public void validate(ProblemReporter problemReporter) {
		// 如果是静态方法没有约束规则，直接返回。
		if (getMetadata().isStatic()) {
			// static @Bean methods have no constraints to validate -> return immediately
			return;
		}
		// 校验该方法所属的类是否被 @Configuration 修饰。
		if (this.configurationClass.getMetadata().isAnnotated(Configuration.class.getName())) {
			// 判断是否可重写。cglib代理需要方法可重写。不可重写则错误
			if (!getMetadata().isOverridable()) {
				// instance @Bean methods within @Configuration classes must be overridable to accommodate CGLIB
				problemReporter.error(new NonOverridableMethodError());
			}
		}
	}
```

## this.reader.loadBeanDefinitions(configClasses);
上面也说了，在parser.parse(candidates); 方法中，将各种注解的属性值都解析了出来，并保存到了 configClass的各种属性中。而在 this.reader.loadBeanDefinitions(configClasses); 中才真正处理了这些属性。所以我们接下来看看loadBeanDefinitions 的处理流程。

loadBeanDefinitions 遍历了每一个ConfigurationClass ，通过loadBeanDefinitionsForConfigurationClass 方法处理。
```
	public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
		TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
		for (ConfigurationClass configClass : configurationModel) {
			loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
		}
	}
```
所以我们来看看 loadBeanDefinitionsForConfigurationClass 的实现。
可很清楚的看到，每个部分的解析都封装到了不同的方法中。
```
	private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
		// 判断是否应该跳过
		if (trackedConditionEvaluator.shouldSkip(configClass)) {
			String beanName = configClass.getBeanName();
			if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
				this.registry.removeBeanDefinition(beanName);
			}
			this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
			return;
		}
		// 1. 如果配置是被引入的(被 @Import 或者其他配置类内部引入)
		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
		// 2. 遍历配置类中的所有 BeanMethod方法
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}
		// 3. 加载 通过 @ImportResource 的 获取的bean
		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
		// 4. 加载 通过 @Import 的 获取的bean
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
```
下面 我们来详细看看每个方法。

### registerBeanDefinitionForImportedConfigurationClass
这一步的工作很简单，就是将引入的配置类注册为 BeanDefinition。
```
	private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
		AnnotationMetadata metadata = configClass.getMetadata();
		AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);
		
		ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
		configBeanDef.setScope(scopeMetadata.getScopeName());
		String configBeanName = this.importBeanNameGenerator.generateBeanName(configBeanDef, this.registry);
		AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata);

		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
		// 创建代理，根据 scopeMetadata 的代理模式。默认不创建代理。
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		// 注册了BeanBeanDefinition 。这里将BeanDefinition保存到了 DefaultListableBeanFactory#beanDefinitionMap 中
		this.registry.registerBeanDefinition(definitionHolder.getBeanName(), definitionHolder.getBeanDefinition());
		configClass.setBeanName(configBeanName);

		if (logger.isTraceEnabled()) {
			logger.trace("Registered bean definition for imported class '" + configBeanName + "'");
		}
	}
```

这里需要注意的是 AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry); 根据scopeMetadata 的代理模式创建了代理。代理模式有四种，分别为

- DEFAULT ： 默认模式。默认等同于NO
- NO ： 不使用代理
- INTERFACES ： Jdk 动态代理
- TARGET_CLASS ： Cglib代理

在 applyScopedProxyMode 方法中 通过获取ScopeMetadata.getScopedProxyMode() 来判断使用什么代理方式。而ScopeMetadata 的代理方式 是在创建 scopeMetadata 的过程中，获取类上面的@Scope 的 proxyMode 属性来指定的。
```
ScopeMetadata scopeMetadata =  scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
```
resolveScopeMetadata 方法如下
```
	protected Class<? extends Annotation> scopeAnnotationType = Scope.class;
	@Override
	public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
		ScopeMetadata metadata = new ScopeMetadata();
		if (definition instanceof AnnotatedBeanDefinition) {
			AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
			// 获取 @Scope 注解
			AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
					annDef.getMetadata(), this.scopeAnnotationType);
			if (attributes != null) {
			
				metadata.setScopeName(attributes.getString("value"));
				// 获取 @Scope 的proxyMode属性
				ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
				if (proxyMode == ScopedProxyMode.DEFAULT) {
					proxyMode = this.defaultProxyMode;
				}
				// 设置 scopedProxyMode 属性，后面根据此属性判断使用什么代理方式
				metadata.setScopedProxyMode(proxyMode);
			}
		}
		return metadata;
	}
```

## loadBeanDefinitionsForBeanMethod
具体代码如下，基本上就是解析各种注解，创建对应的 BeanDefinition 并注册。
```
	private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
		ConfigurationClass configClass = beanMethod.getConfigurationClass();
		MethodMetadata metadata = beanMethod.getMetadata();
		String methodName = metadata.getMethodName();

		// Do we need to mark the bean as skipped by its condition?
		// 是否应该跳过
		if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
			configClass.skippedBeanMethods.add(methodName);
			return;
		}
		if (configClass.skippedBeanMethods.contains(methodName)) {
			return;
		}
		// 获取被 @Bean修饰的方法
		AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);
		Assert.state(bean != null, "No @Bean annotation attributes");

		// Consider name and any aliases
		// 获取别名
		List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));
		String beanName = (!names.isEmpty() ? names.remove(0) : methodName);

		// Register aliases even when overridden
		// 注册别名
		for (String alias : names) {
			this.registry.registerAlias(beanName, alias);
		}

		// Has this effectively been overridden before (e.g. via XML)?
		// 判断是否已经被定义过
		if (isOverriddenByExistingDefinition(beanMethod, beanName)) {
			if (beanName.equals(beanMethod.getConfigurationClass().getBeanName())) {
				throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),
						beanName, "Bean name derived from @Bean method '" + beanMethod.getMetadata().getMethodName() +
						"' clashes with bean name for containing configuration class; please make those names unique!");
			}
			return;
		}
		// 定义配置类的  BeanDefinition
		ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
		beanDef.setResource(configClass.getResource());
		beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));
		// 处理静态 @Bean 方法和 非静态 @Bean 方法
		if (metadata.isStatic()) {
			// static @Bean method
			if (configClass.getMetadata() instanceof StandardAnnotationMetadata) {
				beanDef.setBeanClass(((StandardAnnotationMetadata) configClass.getMetadata()).getIntrospectedClass());
			}
			else {
				beanDef.setBeanClassName(configClass.getMetadata().getClassName());
			}
			// 设置唯一工厂方法名称
			beanDef.setUniqueFactoryMethodName(methodName);
		}
		else {
			// instance @Bean method
			// 指定要使用的工厂bean（如果有）。这是用于调用指定工厂方法的bean的名称
			beanDef.setFactoryBeanName(configClass.getBeanName());
			// 设置唯一工厂方法名称，内部调用了 setFactoryMethodName(name); 保存 FactoryMethodName
			beanDef.setUniqueFactoryMethodName(methodName);
		}

		if (metadata instanceof StandardMethodMetadata) {
			beanDef.setResolvedFactoryMethod(((StandardMethodMetadata) metadata).getIntrospectedMethod());
		}
		// 设置构造模式 构造注入
		beanDef.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
		// 设置跳过属性检查
		beanDef.setAttribute(org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.
				SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);
		// 处理通用的注解： @Lazy、@Primary、@DependsOn、@Role、@Description。设置到 BeanDefinition 中
		AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);
		// 获取注解的其他属性并设置到 BeanDefinition
		Autowire autowire = bean.getEnum("autowire");
		if (autowire.isAutowire()) {
			beanDef.setAutowireMode(autowire.value());
		}

		boolean autowireCandidate = bean.getBoolean("autowireCandidate");
		if (!autowireCandidate) {
			beanDef.setAutowireCandidate(false);
		}

		String initMethodName = bean.getString("initMethod");
		if (StringUtils.hasText(initMethodName)) {
			beanDef.setInitMethodName(initMethodName);
		}

		String destroyMethodName = bean.getString("destroyMethod");
		beanDef.setDestroyMethodName(destroyMethodName);

		// Consider scoping
		ScopedProxyMode proxyMode = ScopedProxyMode.NO;
		// 处理方法上的 @Scope 注解
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
		if (attributes != null) {
			beanDef.setScope(attributes.getString("value"));
			proxyMode = attributes.getEnum("proxyMode");
			if (proxyMode == ScopedProxyMode.DEFAULT) {
				proxyMode = ScopedProxyMode.NO;
			}
		}

		// Replace the original bean definition with the target one, if necessary
		// 如果有必要(代理模式不同)，替换掉旧的BeanDefinition
		BeanDefinition beanDefToRegister = beanDef;
		if (proxyMode != ScopedProxyMode.NO) {
			BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
					new BeanDefinitionHolder(beanDef, beanName), this.registry,
					proxyMode == ScopedProxyMode.TARGET_CLASS);
			beanDefToRegister = new ConfigurationClassBeanDefinition(
					(RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);
		}

		if (logger.isTraceEnabled()) {
			logger.trace(String.format("Registering bean definition for @Bean method %s.%s()",
					configClass.getMetadata().getClassName(), beanName));
		}
		// 注册BeanDefinition
		this.registry.registerBeanDefinition(beanName, beanDefToRegister);
	}
```
这里特意提一下下面几句的功能
```
// 设置引入该bean 的配置类的类名
beanDef.setFactoryBeanName(configClass.getBeanName());
// 设置 引入bean 的类名
beanDef.setBeanClassName(configClass.getMetadata().getClassName());
// 设置在配置类中引入该bean 的方法名
beanDef.setUniqueFactoryMethodName(methodName);
```
这里会为 @Bean修饰的方法创建出一个 ConfigurationClassBeanDefinition注册到 Spring容器中，ConfigurationClassBeanDefinition 是特指用于表示从配置类（而不是其他任何配置源）创建了Bean定义。在需要确定是否在外部创建bean定义的bean覆盖情况下使用。在后面的Bean实例化过程中，会有多次使用。比如在 AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation 中
```
// 在 determineTargetType 方法中根据  factoryMethodName 是否为空，判断bean注入方式，来获取注入的 Class类型
		Class<?> targetType = determineTargetType(beanName, mbd);
```
以及会在 AbstractAutowireCapableBeanFactory#createBeanInstance 方法中有如下两句。

```
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}
```
而@Bean的修饰的方法会调用instantiateUsingFactoryMethod 方法，通过反射调用方法，并将反射结果注入到Spring容器中，完成 @Bean注解的功能。

## loadBeanDefinitionsFromImportedResources
loadBeanDefinitionsFromImportedResources 从导入的资源加载Bean定义。即通过解析 @ImportResource 注解引入的资源文件，获取到BeanDefinition 并注册。
```
private void loadBeanDefinitionsFromImportedResources(
			Map<String, Class<? extends BeanDefinitionReader>> importedResources) {

		Map<Class<?>, BeanDefinitionReader> readerInstanceCache = new HashMap<>();
		// 遍历引入的资源文件
		importedResources.forEach((resource, readerClass) -> {
			// Default reader selection necessary?
			if (BeanDefinitionReader.class == readerClass) {
				// 处理 .groovy 类型文件
				if (StringUtils.endsWithIgnoreCase(resource, ".groovy")) {
					// When clearly asking for Groovy, that's what they'll get...
					readerClass = GroovyBeanDefinitionReader.class;
				}
				else {
					// Primarily ".xml" files but for any other extension as well
					// 这里使用 XmlBeanDefinitionReader 类型来解析
					readerClass = XmlBeanDefinitionReader.class;
				}
			}
			// 从缓冲中获取
			BeanDefinitionReader reader = readerInstanceCache.get(readerClass);
			// 如果缓存中没有，则创建一个 reader 用于 resource 的解析。
			if (reader == null) {
				try {
					// Instantiate the specified BeanDefinitionReader
					reader = readerClass.getConstructor(BeanDefinitionRegistry.class).newInstance(this.registry);
					// Delegate the current ResourceLoader to it if possible
					if (reader instanceof AbstractBeanDefinitionReader) {
						AbstractBeanDefinitionReader abdr = ((AbstractBeanDefinitionReader) reader);
						abdr.setResourceLoader(this.resourceLoader);
						abdr.setEnvironment(this.environment);
					}
					readerInstanceCache.put(readerClass, reader);
				}
				catch (Throwable ex) {
					throw new IllegalStateException(
							"Could not instantiate BeanDefinitionReader class [" + readerClass.getName() + "]");
				}
			}

			// TODO SPR-6310: qualify relative path locations as done in AbstractContextLoader.modifyLocations
			// 解析resource资源中的内容
			reader.loadBeanDefinitions(resource);
		});
	}
```

### loadBeanDefinitionsFromRegistrars
loadBeanDefinitionsFromRegistrars 方法注册了了@Import 注解引入的内容。这里很简单，将@Import 引入的内容注入到容器中。
```
	private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
		registrars.forEach((registrar, metadata) ->
				registrar.registerBeanDefinitions(metadata, this.registry, this.importBeanNameGenerator));
	}
```


