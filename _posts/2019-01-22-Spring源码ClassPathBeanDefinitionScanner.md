---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码ClassPathBeanDefinitionScanner
本文 ClassPathBeanDefinitionScanner#doScan 的作用就是扫描指定目录下的字节码文件，生成对应的BeanDefinition注册到Spring中。

## ClassPathBeanDefinitionScanner
本文 ClassPathBeanDefinitionScanner#doScan 的作用就是扫描指定目录下的字节码文件，生成对应的BeanDefinition注册到Spring中。具体调用时机是在 ConfigurationClassPostProcessor 中，链路如下：
```
// ConfigurationClassPostProcessor#processConfigBeanDefinitions 会对配置类进行解析
ConfigurationClassPostProcessor#processConfigBeanDefinitions
=》 ConfigurationClassParser#parse 
=》 ConfigurationClassParser#processConfigurationClass
=》 ConfigurationClassParser#doProcessConfigurationClass
=》 ComponentScanAnnotationParser#parse
=》 ClassPathBeanDefinitionScanner#doScan
```

## ClassPathBeanDefinitionScanner#doScan
ClassPathBeanDefinitionScanner#doScan 完成了根据指定路径扫描Class 文件，并将筛选后的 Class 文件 生成对应的 BeanDefinition，注册到 Spring 中。

其实现具体如下：
```
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		// 扫描指定的包路径
		for (String basePackage : basePackages) {
			// 1. 获取候选BeanDefinition
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				// Bean作用域解析
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				// beanName 生成
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				// 2. 对 AbstractBeanDefinition 类型的 BeanDefinition 进一步处理
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				// 3. 对 AnnotatedBeanDefinition 的进一步处理
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				// 4. 检查给定候选的 bean 名称，确定相应的 bean 定义是否需要注册或与现有定义冲突
				if (checkCandidate(beanName, candidate)) {
					// 创建 definitionHolder 
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					// 5. 对 BeanDefinitionHolder 填充代理信息
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					// 6. 注册 BeanDefinition
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```
我们这里简述一下整个流程：
doScan 方法首先会扫描指定的包路径下的 字节码文件，同时筛选出可能被注入到容器中的Bean生成BeanDefinition，之后会对 BeanDefinition 进行属性填充。填充完毕后确定Bean无误后则会创建 BeanDefinitionHolder 代理并注册到Spring容器中。

按照注释标注说明一下 ClassPathBeanDefinitionScanner#doScan 的逻辑：

- 获取候选BeanDefinition ： 从指定的包路径获取到字节码文件，筛选出可能注入到Spring容器的Bean生成对应的ScannedGenericBeanDefinition。这里需要注意的是 ScannedGenericBeanDefinition是 AbstractBeanDefinition 和 AnnotatedBeanDefinition 的子类。也即是说第二步和第三步是一定会执行的。
- 对AbstractBeanDefinition 的处理 ： 对于AbstractBeanDefinition 类型，赋值BeanDefinitionDefaults，并设置 autowireCandidate 属性
- 对AnnotatedBeanDefinition 的处理 ： 对通用注解的解析处理，如 如果当前类被@Lazy修饰，则会获取@Lazy 的value 值并保存到 BeanDefinition#lazyInit 属性中。这里解析的注解包括 @Lazy、@Primary、@DependsOn、@Role、@Description
- 候选Bean的检查 ： 对Bean的进一步检查，确定BeanDefinition 可以兼容。
- 对 BeanDefinitionHolder 代理信息的处理 ：这里会根据Bean 指定的代理方式，来对 BeanDefinitionHolder 进行进一步处理。
- 注册 BeanDefinition到容器中 ：将 BeanDefinitionHolder 代理注册到 容器中。

下面我们来详解每一步的过程
## 获取候选BeanDefinition
findCandidateComponents(basePackage); 的实现如下：

```
	// ClassPathScanningCandidateComponentProvider#findCandidateComponents
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		// 1. @Indexed 注解的处理
		if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
			return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
		}
		else {
			// 2. 常规逻辑
			return scanCandidateComponents(basePackage);
		}
	}
```

这里我们可以看到分成了两种情况
- 对 @Indexed 的处理。
- 常规逻辑的处理。

### @Indexed 注解的处理
@Indexed
Spring包org.springframework.stereotype下，除了@Component、@Controller、@Service、@Repository外，在5.0版本中新增了@Indexed注解。@Indexed 为Spring的模式注解添加索引，以提升应用启动性能。随着项目中 @ComponentScan 扫描目录下的类越来越多时，Spring对类的解析耗时就越多。@Indexed注解的引入正是为了解决这个问题。

其解决方案如下：
当项目进行编译打包时，会自动生成 META-INF/spring.components文件，该文件中以key=value 形式保存信息。key为 类的全路径名， value 为 注解的全路径名

项目编译打包时，会在自动生成META-INF/spring.components文件，文件包含被@Indexed注释的类的模式解析结果。当Spring应用上下文进行组件扫描时，META-INF/spring.components会被org.springframework.context.index.CandidateComponentsIndexLoader读取并加载，转换为CandidateComponentsIndex对象，此时组件扫描会读取CandidateComponentsIndex，而不进行实际扫描，从而提高组件扫描效率，减少应用启动时间。

如果使用该功能，需要引入如下依赖：
```
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context-indexer</artifactId>
        </dependency>
```
引入上面依赖后，项目编译时 ，依赖中的 CandidateComponentsIndexer#process 会创建了META-INF/spring.components 文件。

META-INF/spring.components 文件中保存格式为 key=value。

以下两种情况下的类会被保存到 META-INF/spring.components 中：

- 被 @Component 或其子注解修饰的类。
- 被 @Index 子注解所修饰的类，我们可以通过这种方式来扩展索引机制扫描的注解。上图中的 User，则是被 @MyComponent 注解修饰，而这里的 @MyComponent 则被 @Indexed 注解修饰，如下：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Indexed
public @interface MyComponent {
}
```

## 扫描逻辑实现
CandidateComponentsIndex 的初始化 ： ClassPathBeanDefinitionScanner在创建时
会调用 ClassPathScanningCandidateComponentProvider#setResourceLoader 方法，在该方法中会加载 META-INT/spring.components，并生成 CandidateComponentsIndex。如果不存在META-INT/spring.components 则会直接返回null。

对 CandidateComponentsIndex 的处理 ： ClassPathScanningCandidateComponentProvider#addCandidateComponentsFromIndex 实现如下：
```
	private Set<BeanDefinition> addCandidateComponentsFromIndex(CandidateComponentsIndex index, String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<>();
		try {
			Set<String> types = new HashSet<>();
			// 1. 从候选索引 Bean 列表中挑选出满足的 Bean
			for (TypeFilter filter : this.includeFilters) {
				// 1.1 根据 Filter类型获取 印象：AnnotationTypeFilter 返回注解全路径名，AssignableTypeFilter 返回 类型全路径名。我们这里会返回
				String stereotype = extractStereotype(filter);
				if (stereotype == null) {
					throw new IllegalArgumentException("Failed to extract stereotype from " + filter);
				}
				// 1.2 从 index 中根据 basePackage 和 stereotype 筛选出合适的 类的全路径名
				// 满足 stereotype 类型 && bean 是 basePackage 在目录下
				types.addAll(index.getCandidateTypes(basePackage, stereotype));
			}
			// 2. 对 types 进行候选Bean筛选，满足则生成BeanDefinition。
			for (String type : types) {
				MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(type);
				// 2.1进行 excludeFilters 和 includeFilters 的筛选。筛选通过后则创建对应 BeanDefinition  
				if (isCandidateComponent(metadataReader)) {
					ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
					sbd.setSource(metadataReader.getResource());
					// 2.2
					if (isCandidateComponent(sbd)) {
						candidates.add(sbd);
					}
					else {
					}
				}
				else {
				}
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
		}
		return candidates;
	}

	@Nullable
	private String extractStereotype(TypeFilter filter) {
		// 如果是 AnnotationTypeFilter，则将匹配的注解返回
		if (filter instanceof AnnotationTypeFilter) {
			return ((AnnotationTypeFilter) filter).getAnnotationType().getName();
		}
		// 如果是 AssignableTypeFilter，则将匹配的类型返回。
		if (filter instanceof AssignableTypeFilter) {
			return ((AssignableTypeFilter) filter).getTargetType().getName();
		}
		return null;
	}
```
从候选索引 Bean 列表中挑选出满足的 Bean。 默认情况下，includeFilters 中有两个 AnnotationTypeFilter，分别是对 @Component 和 @ManagedBean两个注解的解析，即如果 Bean 被这两个注解修饰，则认为可以作为候选Bean。所以默认情况下 1.1 中会返回值是 org.springframework.stereotype.Component 和 javax.annotation.ManagedBean。1.2 的作用是从 META-INT/spring.components 中筛选出这两个注解修饰的的索引Bean。

这一步则是对Bean的作为候选Bean的条件过滤，包括 excludeFilters、includeFilters等校验。这些校验在下面常规逻辑中有过描述。

## 常规逻辑
除了上述的 @Indexed 情况，一般情况下，我们会执行scanCandidateComponents(basePackage); 的逻辑，其实现如下：
```
	
	static final String DEFAULT_RESOURCE_PATTERN = "**/*.class";


	protected final Log logger = LogFactory.getLog(getClass());

	private String resourcePattern = DEFAULT_RESOURCE_PATTERN;
	// ClassPathScanningCandidateComponentProvider#scanCandidateComponents
	private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<>();
		try {
			// 1. 拼接包路径
			// 如 com.kingfish 会拼接成  classpath*:com/kingfish/**/*.class
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
			// 扫描出来的满足上面 packageSearchPath  条件的字节码文件
			Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
			...
			// 2.遍历所有字节码文件，挑选合适的字节码文件生成BeanDefinition
			for (Resource resource : resources) {
				// 如果资源文件可读
				if (resource.isReadable()) {
					try {
						// MetadataReader 中包含了文件信息和对应类注解信息
						MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
						// 2.1 校验是否是候选组件。这里是针对 excludeFilters 和  includeFilters 的匹配校验
						if (isCandidateComponent(metadataReader)) {
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							sbd.setSource(resource);
							// 2.2 校验是否是候选组件
							if (isCandidateComponent(sbd)) {
								candidates.add(sbd);
							}
							else {
								// ... 日志打印
							}
						}
						else {
							// ... 日志打印
						}
					}
					catch (Throwable ex) {
						throw new BeanDefinitionStoreException(
								"Failed to read candidate component class: " + resource, ex);
					}
				}
				else {
					// ... 日志打印
				}
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
		}
		return candidates;
	}
```
这里的过程其实很简单

- 扫描指定路径下的字节码文件
- 判断当前字节码文件是否可以被注入到容器中
- 如果可以注入到容器中，则生成 ScannedGenericBeanDefinition 作为候选Bean返回接受后面的流程过滤。

这里的关键逻辑在于 2.1 和 2.2 的两次校验，其实现如下：

```
	// 2.1 对 excludeFilters 和 includeFilters  的处理：当前类不能匹配 excludeFilters  && 当前类匹配includeFilters && 满足 @Conditional 注解加载条件
	protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
		// 对 excludeFilters 的校验
		for (TypeFilter tf : this.excludeFilters) {
			if (tf.match(metadataReader, getMetadataReaderFactory())) {
				return false;
			}
		}
		// 对 includeFilters 的校验
		for (TypeFilter tf : this.includeFilters) {
			if (tf.match(metadataReader, getMetadataReaderFactory())) {
				return isConditionMatch(metadataReader);
			}
		}
		return false;
	}
	// 2.2 根据任何@Conditional注释确定给定的类是否是候选组件
	private boolean isConditionMatch(MetadataReader metadataReader) {
		if (this.conditionEvaluator == null) {
			this.conditionEvaluator =
					new ConditionEvaluator(getRegistry(), this.environment, this.resourcePatternResolver);
		}
		return !this.conditionEvaluator.shouldSkip(metadataReader.getAnnotationMetadata());
	}
	
	// 检查类是否不是接口并且不依赖于封闭类
	protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
		AnnotationMetadata metadata = beanDefinition.getMetadata();
		// 类是独立的 && (类是具体实现类 || (类是抽象类但是被@Lookup 注解修饰))
		return (metadata.isIndependent() && (metadata.isConcrete() ||
				(metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));
	}
```

对于 2.2 的过滤就是对 @Conditional 的处理，具体实现不再分析

我们这里关心 2.1 中对 excludeFilters 和 includeFilters 的处理。

我们可以通过如下方式添加 includeFilters 和 excludeFilters 的值：
```
@ComponentScan(includeFilters = {@ComponentScan.Filter(type = FilterType.CUSTOM, classes = CustomTypeIncludeFilter.class)},
        excludeFilters = {@ComponentScan.Filter(type = FilterType.CUSTOM, classes = CumstomTypeExcludeFilter.class)})

```
而默认情况下 excludeFilters 中有三个实现类：

ComponentScanAnnotationParser#parse 方法中实现的匿名类，目的是排除启动类。其实现如下：
```
		scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
			@Override
			protected boolean matchClassName(String className) {
				// 这里的 declaringClass 即是启动类。即如果扫描到了启动类则不再作为候选Bean
				return declaringClass.equals(className);
			}
		});

```
AutoConfigurationExcludeFilter ： 排除被 @Configuration 注解修饰的类 和 通过spring.factories 自动装配方式加载的类。

TypeExcludeFilter ：排除 TypeExcludeFilter 匹配的类。默认没有实现，可以自定义

默认情况下，includeFilters 中有两个 AnnotationTypeFilter，分别是对 @Component 和 @ManagedBean两个注解的解析，即如果 Bean 被这两个注解修饰，则认为可以作为候选Bean。

## 对 AbstractBeanDefinition 进一步处理
ClassPathBeanDefinitionScanner#postProcessBeanDefinition 的实现很简单，是对 BeanDefinition 属性的填充。

```
	// 	org.springframework.context.annotation.ClassPathBeanDefinitionScanner#postProcessBeanDefinition
	protected void postProcessBeanDefinition(AbstractBeanDefinition beanDefinition, String beanName) {
		// 给 beanDefinition 填充默认值
		beanDefinition.applyDefaults(this.beanDefinitionDefaults);
		// 如果设置了autowireCandidatePatterns 则按照其规则来设置 Bean的 autowireCandidate 属性(此 bean 是否是自动装配到其他 bean 的候选者。)
		if (this.autowireCandidatePatterns != null) {
			beanDefinition.setAutowireCandidate(PatternMatchUtils.simpleMatch(this.autowireCandidatePatterns, beanName));
		}
	}
```

## 对 AnnotatedBeanDefinition 进一步处理
AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate)是对一些通用注解的解析，并将注解的属性值填充到 BeanDefinition 中。解析的注解包括 @Lazy、@Primary、@DependsOn、@Role、@Description。实现如下：
```
	static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
		AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
		if (lazy != null) {
			abd.setLazyInit(lazy.getBoolean("value"));
		}
		else if (abd.getMetadata() != metadata) {
			lazy = attributesFor(abd.getMetadata(), Lazy.class);
			if (lazy != null) {
				abd.setLazyInit(lazy.getBoolean("value"));
			}
		}

		if (metadata.isAnnotated(Primary.class.getName())) {
			abd.setPrimary(true);
		}
		AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
		if (dependsOn != null) {
			abd.setDependsOn(dependsOn.getStringArray("value"));
		}

		AnnotationAttributes role = attributesFor(metadata, Role.class);
		if (role != null) {
			abd.setRole(role.getNumber("value").intValue());
		}
		AnnotationAttributes description = attributesFor(metadata, Description.class);
		if (description != null) {
			abd.setDescription(description.getString("value"));
		}
	}
```

## 检查给定候选的 bean 名称
ClassPathBeanDefinitionScanner#checkCandidate 是检查当前 BeanDefinition 是否已经存在，如果存在，二者是否兼容。其实现如下：

```
	protected boolean checkCandidate(String beanName, BeanDefinition beanDefinition) throws IllegalStateException {
		// 如果当前Bean尚未注册则直接返回true，不用再校验
		if (!this.registry.containsBeanDefinition(beanName)) {
			return true;
		}
		BeanDefinition existingDef = this.registry.getBeanDefinition(beanName);
		BeanDefinition originatingDef = existingDef.getOriginatingBeanDefinition();
		if (originatingDef != null) {
			existingDef = originatingDef;
		}
		// 确定给定的新 bean 定义是否与给定的现有 bean 定义兼容。
		// 当现有 bean 定义来自同一源或非扫描源时，默认实现将它们视为兼容
		if (isCompatible(beanDefinition, existingDef)) {
			return false;
		}
		throw new ConflictingBeanDefinitionException("Annotation-specified bean name '" + beanName +
				"' for bean class [" + beanDefinition.getBeanClassName() + "] conflicts with existing, " +
				"non-compatible bean definition of same name and class [" + existingDef.getBeanClassName() + "]");
	}
```

## 对 BeanDefinitionHolder 代理信息的处理
AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry); 中主要是对Bean 代理模式的处理。其实现如下 ：
```
	static BeanDefinitionHolder applyScopedProxyMode(
			ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {
		// 获取代理模式，一般为 No。为 No则不需要代理，直接返回。
		ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
		if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
			return definition;
		}
		boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
		// 该方法会整理bean的代理信息，重新生成一个 BeanDefinitionHolder 和 BeanDefinition 并返回作为Proxy BeanDefinition。
		// 需要注意这里并不会执行代理，因为这里并非创建Bean的场景，所以记录下代理的相关配置信息，等到Bean创建时会根据这些属性信息来创建代理对象。
		return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
	}

```
根据scopeMetadata 的代理模式创建了代理。代理模式有四种，分别为

DEFAULT ： 默认模式。默认等同于NO
NO ： 不使用代理
INTERFACES ： Jdk 动态代理
TARGET_CLASS ： Cglib代理

我们这里暂时将未代理的 BeanDefinition 称为Original BeanDefinition ，代理后的 BeanDefinition 称为 Proxy BeanDefinition，这里的Original BeanDefinition 实际类型是 ScannedGenericBeanDefinition，Proxy BeanDefinition 的实际类型是 RootBeanDefinition。

如果Bean DemoService 需要代理，在 ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass); 方法中会将 Original BeanDefinition 注册到 BeanDefinitionRegistry。其中 beanName 为 scopedTarget.demoService，BeanDefinition 为 Original BeanDefinition。同时 Proxy BeanDefinition 会在第六步中注册到 BeanDefinitionRegistry中，其beanName 为 demoService，BeanDefinition为 Proxy BeanDefinition。也即是说对于 DemoService，此时容器中存在两个 BeanDefinition，一个是 scopedTarget.demoService =》 Original BeanDefinition。另一个是 demoService =》 Proxy BeanDefinition

如 对于 使用 ScopedProxyMode.TARGET_CLASS 的User 类来说：
```
@Component
@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)
public class User {
}
```

## 注册 BeanDefinition
BeanDefinitionReaderUtils#registerBeanDefinition 将 beanName 和 BeanDefinition 关联并注册到 registry 中。同时注册了Bean的别名信息。其实现如下：

```
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
		// 注册 BeanDefinition
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		// 注册别名
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

## 自定义注解扫描
这里可以说一下写本文的初衷 ：Mybatis 的 @Mapper 注解并非Spring容器注解，而Mybatis能够将 @Mapper 注解修饰的类注入到容器中就是靠 ClassPathMapperScanner (继承了 ClassPathBeanDefinitionScanner) 类来给 @Mapper注解修饰的类 创建对应的BeanDefinition。所以想到自己也可以实现一个类似的功能。所以把源码实现看了一下，完成了本篇内容。

因此，这里也模拟Mybatis，自定义注解注入到容器中。
```
// 自定义的注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyComponent {
}


// 自定义的扫描类
public class DemoAnnotationScanner extends ClassPathBeanDefinitionScanner{
    public DemoAnnotationScanner(BeanDefinitionRegistry registry) {
        super(registry);
        // 添加包含条件， 被 @MyComponent 注解修饰的类也会生成BeanDefinition
        addIncludeFilter(new AnnotationTypeFilter(MyComponent.class));
    }
}

// 自定义BeanDefinitionRegistryPostProcessor，完成 路径扫描
@Component
public class DemoBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor, BeanFactoryAware {
    private BeanFactory beanFactory;

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    	// 获取启动类路径
        List<String> packages = AutoConfigurationPackages.get(beanFactory);
        DemoAnnotationScanner demoAnnotationScanner = new DemoAnnotationScanner(registry);
        // 根据指定路径进行扫描
        demoAnnotationScanner.scan(packages.get(0));

    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
}

```
题外话：
BeanDefinitionRegistryPostProcessor 为 BeanFactoryPostProcessor 的子类，作为容器级别的后处理器，我们可以再 BeanFactoryPostProcessor#postProcessBeanDefinitionRegistry 方法中完成 BeanDefinition 注册。

简单梳理一下思路：

我们创建DemoBeanDefinitionRegistryPostProcessor 类继承 BeanDefinitionRegistryPostProcessor ，并在postProcessBeanDefinitionRegistry 方法中获取到Spring启动类目录作为我们的扫描目录，交由DemoAnnotationScanner 来扫描。

DemoAnnotationScanner 获取到启动类目录后，会扫描路径中的所有字节码文件，默认情况下，满足 IncludeFilter 条件的才有可能作为候选Bean。而我们手动添加了 new AnnotationTypeFilter(MyComponent.class) 。即被 @MyComponent 注解修饰的类也会作为候选Bean。最终注册到容器中。





















