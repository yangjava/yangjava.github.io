---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码Bean的加载

## doGetBean概述
Spring 在容器刷新的后期 通过调用AbstractApplicationContext#finishBeanFactoryInitialization 方法来实例化了所有的非惰性bean。在这里面就通过 beanFactory.preInstantiateSingletons(); 调用了一个非常关键的方法 AbstractBeanFactory#getBean(java.lang.String)，而其实际上调用的是 AbstractBeanFactory#doGetBean 方法

```
    public Object getBean(String name) throws BeansException {
        return this.doGetBean(name, (Class)null, (Object[])null, false);
    }
```
如果说，Spring中，Bean 的发现是在 ConfigurationClassPostProcessor 中进行的。那么Bean的创建就是在 doGetBean方法中进行的。 doGetBean 完成了单例Bean的完整创建过程，包括bean的创建，BeanPostProcessor 的方法调用、init-method等方法的调用、Aware 等接口的实现。
下面，我们开始来分析这个 doGetBean。

关于Bean 的加载过程，AbstractApplicationContext#finishBeanFactoryInitialization 经过几次跳转，最终会跳转到 AbstractBeanFactory#doGetBean 方法。每个bean的创建都会经历此方法，所以本文的主要内容是分析 AbstractBeanFactory#doGetBean 。

## DefaultListableBeanFactory
需要强调的是调用 doGetBean 方法的是 AbstractBeanFactory 的子类DefaultListableBeanFactory。这是 Springboot默认的BeanFactory类型。

注意这里说的 BeanFactory 并不是指 ApplicationContext，而是 ApplicationContext 内部的一个BeanFactory 。对 Springboot 来说，Springboot 默认的上下文是 AnnotationConfigServletWebServerApplicationContext，AnnotationConfigServletWebServerApplicationContext 内部还有一个单独的BeanFactory对象(DefaultListableBeanFactory)。

虽然 AnnotationConfigServletWebServerApplicationContext 也实现了 BeanFactory接口，但是实际上对于一般的BeanFactory 请求是AnnotationConfigServletWebServerApplicationContext 直接委托给内部的 BeanFactory来解决。 而这里所说的BeanFactory实际上是 AnnotationConfigServletWebServerApplicationContext 内部的 DefaultListableBeanFactory。

## DefaultSingletonBeanRegistry
在这里我们只需要知道DefaultListableBeanFactory 继承了 DefaultSingletonBeanRegistry类，拥有了 DefaultSingletonBeanRegistry 一系列的 集合类型来保存Bean相关信息。

大体如下，其中我们主要只需要关注前四个即可：
```

	/** Cache of singleton objects: bean name to bean instance. */
	//	用于保存BeanName和创建bean实例之间的关系，即缓存bean。 beanname -> instance 
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name to ObjectFactory. */
	// 用于保存BeanName和常见bean的工厂之间的关系。beanname-> ObjectFactory
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name to bean instance. */
	// 也是保存BeanName和创建bean实例之间的关系，与singletonObjects 不同的是，如果一个单例bean被保存在此，则当bean还在创建过程中(比如 A类中有B类属性，当创建A类时发现需要先创建B类，这时候Spring又跑去创建B类，A类就会添加到该集合中，表示正在创建)，就可以通过getBean方法获取到了，其目的是用来检测循环引用。
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

	/** Set of registered singletons, containing the bean names in registration order. */
	// 用来保存当前所有已经注册的bean
	private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

	/** Names of beans that are currently in creation. */
	// 用来保存当前正在创建的Bean。也是为了解决循环依赖的问题
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));

	/** Names of beans currently excluded from in creation checks. */
	// 用来保存当前从创建检查中排除的bean名称
	private final Set<String> inCreationCheckExclusions =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));

	/** List of suppressed Exceptions, available for associating related causes. */
	// 初始化过程中的异常列表
	@Nullable
	private Set<Exception> suppressedExceptions;

	/** Flag that indicates whether we're currently within destroySingletons. */
	// 标志是否在销毁BeanFactory过程中
	private boolean singletonsCurrentlyInDestruction = false;

	/** Disposable bean instances: bean name to disposable instance. */
	// 一次性bean实例：beanName -> 一次性实例。暂未明白
	private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

	/** Map between containing bean names: bean name to Set of bean names that the bean contains. */
	// 包含的Bean名称之间的映射：BeanName  -> Bean包含的BeanName集合
	private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);

	/** Map between dependent bean names: bean name to Set of dependent bean names. */
	// bean dependent(依赖的集合) : beanName -> 依赖该beanName 的 bean，即 key代表的bean 被value 所依赖
	private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

	/** Map between depending bean names: bean name to Set of bean names for the bean's dependencies. */
	// bean 被哪些bean依赖 ：  beanName -> beanName 所依赖的 bean。即 key 依赖于value这些bean
	private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);
```

## 关键缓存
下面挑出来几个关键缓存集合来描述：

- singletonObjects ：ConcurrentHashMap。
最简单最重要的缓存Map。保存关系是 beanName ：bean实例关系。单例的bean在创建完成后都会保存在 singletonObjects 中，后续使用直接从singletonObjects 中获取。
- singletonFactories ：HashMap。
这是为了解决循环依赖问题，用于提前暴露对象，保存形式是 beanName : ObjectFactory<?>。
- earlySingletonObjects ：HashMap。
这也是为了解决循环依赖问题。和 singletonFactories 互斥。因为 singletonFactories 保存的是 ObjectFactory。而earlySingletonObjects 个人认为是 singletonFactories 更进一步的缓存，保存的是 ObjectFactory#getObject的结果。
- registeredSingletons ：LinkedHashSet，
用于保存注册过的beanName，
- singletonsCurrentlyInCreation ： 
保存当前正在创建的bean。当一个bean开始创建时将保存其beanName，创建完成后将其移除
- dependentBeanMap ：
保存bean的依赖关系，比如A对象依赖于 B对象，会出现 B ：A。即保存的是key 被value依赖
- dependenciesForBeanMap ：
保存bean的依赖关系，不过和dependentBeanMap 反了过来。A对象依赖于 B对象，会出现 A ：B。保存的是key 依赖于 value

我们拿一个简单的场景解释一下 singletonFactories 和 earlySingletonObjects 的关系 。
下面的代码是 DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean) 。我们可以看到其判断逻辑，

- 锁定 singletonObjects，毕竟要操作 singletonObjects
- 从 singletonFactories 中获取 ObjectFactory 缓存
- 如果存在 ObjectFactory 缓存，则更进一步提取ObjectFactory#getObject 的singletonObject 对象。将singletonObject 保存到 earlySingletonObjects 缓存中，同时从 singletonFactories 中移除。
```
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```
在创建过程中会被添加到 singletonFactories 中，但当bean被循环依赖时会被添加到 earlySingletonObjects 中。也即是说 earlySingletonObjects 中的bean都是被循环依赖的。

## BeanDefinition
这里简单介绍一下，顾名思义，BeanDefinition是bean的信息，一个BeanDefinition 描述和定义了创建一个bean需要的所有信息，属性，构造函数参数以及访问它们的方法。还有其他一些信息，比如这些定义来源自哪个类等等。

对于XML 配置方式的Spring方式来说， BeanDefinition 是配置文件 < bean > 元素标签在容器中的内容表示形式。<bean> 元素标签拥有class、scope、lazy-init等配置属性，BeanDefinition 则提供了相应的beanClass、scope、lazyinit属性，BeanDefinition 和 <bean> 中的属性是一一对应的。其中RootBeanDefinition 是最常用的实现类，一般对应< bean > 元素标签。

## doGetBean 概述
下面我们开始进入正题，进行 AbstractBeanFactory#doGetBean的内容分析。这个方法是一切的核心(Bean的创建过程也是在这个方法中完成)。首先我们先来整体过一遍方法代码。后面将会对一些关键点进行详细解释。
```
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
		// 1. 提取出对应的beanName。会去除一些特殊的修饰符比如 "&"
		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		// 2. 尝试从缓存获取或者singletonFacotries中的ObjectFactory中获取。后续细讲
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			... 打印日志
			// 3. 返回对应的实例，有时候存在诸如BeanFactory的情况并不直接返回实例本身，而是返回指定方法返回的实例。这一步主要还是针对FactoryBean的处理。
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		else {
			// 4. 只有单例情况才会尝试解决循环依赖，原型模式直接抛出异常。因为原型模式无法解决循环依赖问题。参考衍生篇关于循环依赖的内容
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			// 获取父级的 BeanFactory
			BeanFactory parentBeanFactory = getParentBeanFactory();
			// 5. 如果 beanDefinitionMap 中也就是在所有已经加载的类中不包含beanName，则尝试从parentBeanFactory中检测
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				// 递归到BeanFactory中检测
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}
			// 如果不仅仅做类型检查则是创建bean，这里需要记录
			if (!typeCheckOnly) {
				// 这里是将 当前创建的beanName 保存到 alreadyCreated 集合中。alreadyCreated 中的bean表示当前bean已经创建了，在进行循环依赖判断的时候会使用
				markBeanAsCreated(beanName);
			}

			try {
				// 6. 将当前 beanName 的 BeanDefinition 和父类BeanDefinition 属性进行一个整合
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				// 7. 寻找bean的依赖
                // 获取初始化的依赖项
				String[] dependsOn = mbd.getDependsOn();
				// 如果需要依赖，则递归实例化依赖bean
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// 缓存依赖调用
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
                // 8 针对不同的Scope 进行bean的创建
				// 实例化依赖的bean便可以实例化mdb本身了
				// singleton 模式的创建
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// Prototype 模式的创建
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					// 指定scope上实例化bean
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
        // 9 类型转换
		// 检查需要的类型是否符合bean 的实际类型
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```
综上所属，doGetBean的 大体流程如下：

- 将传入的beanName 转化为合适的beanName。因为这里可能传入bean的别名，比如 FactoryBean 这里就是传入 “&beanName” ， 这一步就会将其转化为 “beanName”
- 尝试从单例缓存中获取 bean，主要是尝试从 singletonObjects 和 singletonFactories 中获取实例。（这一步中还使用了 earlySingletonObjects 来判断循环依赖的问题）
- 如果第二步获取到了bean，则会针对处理 FactoryBean 这种特殊情况，以获取到正确的bean。（因为Factorybean 的话可能需要将其 getObject 方法的返回值作为bean注入到容器中）。
- 如果第二步没有获取到bean，则会检测其原型模式下的循环依赖情况，如果原型模式下有循环依赖，则直接抛出异常，因为原型模式下无法解决循环依赖。
- 如果第四步没有抛出异常，则会判断 当前BeanFactory 中是否包含该beanName 的定义信息，如果不包含，则会递归去 parentBeanFactory 中去寻找beanName的定义信息.
- 随后查询beanName 的 BeanDefinition 是否具有 父类的BeanDefinition， 如果有，则将 父类的一些属性和子类合并，形成一个新的BeanDefinition ： mdb
- 获取mdb中的 depends-on 属性，优先将依赖的bean创建，随后再创建当前bean。
- 到这一步，则说明当前bean尚未创建，则会根据 singleton 或者 prototype 或其他逻辑，走不同的流程来创建bean
- 创建bean结束后，根据调用者需要的类型进行一个类型转换。比如调用者希望返回一个Integer，这里得到的结果却是String，则会进行一个类型的转换。

## doGetBean 详解

### 转换 beanName
```
final String beanName = transformedBeanName(name);
```
这一步的目的是为了去除 beanName 的别名，获取bean的真正beanName。

这里的name传入的可能是 bean的别名，或者是FactoryBean类型的bean。所以需要一系列的解析，解析包括

- 去除 FactoryBean 的修饰符。也就是说如果 name = “&name” 或者 name = “&&name” 这种多&& 情况也会去除& 使得 name = “name”。
- 取指定alias所表示的最终beanName。比如别名A指向B的bean，则会返回B。

代码如下：
```
	protected String transformedBeanName(String name) {
		return canonicalName(BeanFactoryUtils.transformedBeanName(name));
	}
	
	....
	// 从别名中获取真正的beanName
	public String canonicalName(String name) {
		String canonicalName = name;
		// Handle aliasing...
		String resolvedName;
		do {
			// 从aliasMap 中获取到真实的beanName
			resolvedName = this.aliasMap.get(canonicalName);
			if (resolvedName != null) {
				canonicalName = resolvedName;
			}
		}
		while (resolvedName != null);
		return canonicalName;
	}

	....
	// BeanFactoryUtils 中
	public static String transformedBeanName(String name) {
		Assert.notNull(name, "'name' must not be null");
		// 如果不是以 & 开头直接返回
		if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
			return name;
		}
		// 否则剪切到 开头的 & ，直至开头没有 &
		return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
			do {
				beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
			}
			while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
			return beanName;
		});
	}
```

### 尝试从缓存中加载单例
```
Object sharedInstance = getSingleton(beanName);
```
对于单例Bean来说， Spring只会在同一容器中创建一次，并将创建好的Bean 实例保存到 singletonObjects 中，下次再获取bean实例，直接从 singletonObjects 中获取。

这一步的目的是从尝试从缓存中获取实例。需要注意的是，Spring为了解决循环依赖问题， 在创建bean的原则是不等bean 创建完成就会将创建bean的ObjectFactory提早曝光加入到缓存()中，一旦下一个bean创建时需要依赖上一个bean，则直接使用ObjectFactory。

代码逻辑如下：

- 尝试从singletonObjects 中获取Bean 实例。获取不到说明该bean尚未创建成功
- isSingletonCurrentlyInCreation 返回 true，则说明当前bean 的创建过程存在循环依赖。下面的逻辑就是为了尝试解决循环依赖
- 尝试从 earlySingletonObjects 中获取，获取不到说明该bean并未在创建过程中。(为了解决循环依赖的问题)
- 当 allowEarlyReference = true时，这个是针对循环引用的操作，是允许循环引用。
- 随后 从singletonFactories 中加载 ObjectFactory，并将结果保存到 earlySingletonObjects 中，同时将 singletonFactories 中关于bean的定义移除。（earlySingletonObjects 和 singletonFactories 互斥）

这里我们看到
singletonFactories 的映射关系是 beanName : ObjectFactory
earlySingletonObjects 的映射关系是 beanName : ObjectFactory#getObject
个人理解 earlySingletonObjects 是 singletonFactories 更进一步的缓存，所以二者互斥，相同的对象，一个缓存中存在即可。

```
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// 尝试从单例缓存 singletonObjects 中加载。
		Object singletonObject = this.singletonObjects.get(beanName);
		// 单例缓存中没有对象 && 当前单例bean正在创建中，这是为了解决循环依赖的问题
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			// 如果单例缓存中不存在该bean，则加锁进行接下来的处理
			synchronized (this.singletonObjects) {
				// 如果此时bean正在加载(bean 在 earlySingletonObjects 中)，则直接将singletonObject 返回。
				singletonObject = this.earlySingletonObjects.get(beanName);
				// allowEarlyReference = true 才会允许循环依赖
				if (singletonObject == null && allowEarlyReference) {
					// 当某些方法需要提前初始化的时候则会调用addSingletonFactory 将对应的ObjectFactory初始化策略存储在singletonFactories中
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						// 调用预先设定的getObject方法
						singletonObject = singletonFactory.getObject();
						// 记录在缓存中，earlySingletonObjects 和 singletonFactories互斥
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

### 尝试从FactoryBean中获取对象
```
	bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
```

当我们结束上一步之后，经过 sharedInstance != null && args == null 的判断后就会调用该方法。作为上一步获取到的结果 sharedInstance 。我们需要判断其是否是 FactoryBean 的 实现类，如果是，则需要将其getObject() 的结果注入。所以该方法的功能简单来说就是用来检测当前bean是否是FactoryBean类型的bean，如果是，则调用其getObject() 方法，并将其返回值作为bean。

代码逻辑大体如下：

- 首先是在 AbstractAutowireCapableBeanFactory#getObjectForBeanInstance 中，添加依赖bean信息。随后跳转到 AbstractBeanFactory#getObjectForBeanInstance 中
- 判断程序是否想获取 FactoryBean实例(beanName 是否以 & 开头)。如果是判断当前beanInstance是否是 FactoryBean。如果是则返回，否则抛出异常
- 如果不是想获取FactoryBean，那么就是想获取bean实例了。那么判断此时的beanInstance是普通的bean还是FactoryBean类型，如果是普通的bean则直接返回。
- 此时beanInstance 必定是 FactoryBean类型并且程序想获取bean实例。那么首先尝试从缓存 factoryBeanObjectCache 中获取。获取失败，则调用FactoryBean#getObject 方法来获取bean实例。并且在允许调用后置方法的情况下(shouldPostProcess 为true)，调用BeanPostProcessor#postProcessAfterInitialization 的方法。


下面我们来看详细代码

### AbstractAutowireCapableBeanFactory#getObjectForBeanInstance
首先调用的是 AbstractAutowireCapableBeanFactory#getObjectForBeanInstance
```
	private final NamedThreadLocal<String> currentlyCreatedBean = new NamedThreadLocal<>("Currently created bean");

	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
		// 获取当前线程正在创建的bean。currentlyCreatedBean 是一个 ThreadLocal
		String currentlyCreatedBean = this.currentlyCreatedBean.get();
		// 如果当前线程正在创建其他bean，则说明currentlyCreatedBean  的创建依赖于 beanName。则去保存这个依赖关系
		if (currentlyCreatedBean != null) {
			registerDependentBean(beanName, currentlyCreatedBean);
		}

		return super.getObjectForBeanInstance(beanInstance, name, beanName, mbd);
	}

	....
	// 注册依赖关系的bean
	public void registerDependentBean(String beanName, String dependentBeanName) {
		// 获取真实的beanName
		String canonicalName = canonicalName(beanName);
		// 保存依赖关系。dependentBeanMap： key 被 value 依赖
		synchronized (this.dependentBeanMap) {
			Set<String> dependentBeans =
					this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
			if (!dependentBeans.add(dependentBeanName)) {
				return;
			}
		}
		// dependenciesForBeanMap : key 依赖于bean
		synchronized (this.dependenciesForBeanMap) {
			Set<String> dependenciesForBean =
					this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
			dependenciesForBean.add(canonicalName);
		}
	}
```
这里我们可以知道其中有一个逻辑是判断当前线程是否存在创建中的 currentlyCreatedBean ，存在则说明 currentlyCreatedBean 依赖于正在创建的bean。因为对于bean的创建来说，如果发现当前bean依赖于其他bean，则会转向优先创建依赖的bean。

### AbstractBeanFactory#getObjectForBeanInstance
随后通过super.getObjectForBeanInstance(beanInstance, name, beanName, mbd); 调用了 AbstractBeanFactory#getObjectForBeanInstance
```
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		// 1. 检测name 是否是想获取 工厂类 (name 以 & 开头) 
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			// 以&开头又不是FactoryBean实现类，则抛出异常
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
			if (mbd != null) {
				mbd.isFactoryBean = true;
			}
			return beanInstance;
		}

		// 2. 此时bean可能是 FactoryBean 或者 普通的bean。判断如果 beanInstance 不是 FactoryBean而是普通的bean, 就直接返回
		if (!(beanInstance instanceof FactoryBean)) {
			return beanInstance;
		}
		// 3. 到这一步就可以确定，当前beanInstance 是FactoryBean，并且需要获取getObject() 的结果
		Object object = null;
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		else {
			// 尝试从缓存中加载bean。这一步是从 factoryBeanObjectCache 集合中获取
			// 在后面获取 bean 成功后，可能会将 其缓存到 factoryBeanObjectCache  中
			object = getCachedObjectForFactoryBean(beanName);
		}
		
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// containsBeanDefinition 检测  beanDefinitionMap中也就是所有已经加载的类中检测是否定义beanName
			if (mbd == null && containsBeanDefinition(beanName)) {
				// 合并父类bean 定义的属性
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			// 4. 这一步中对FactoryBean进行了解析。
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

### doGetObjectFromFactoryBean
getObjectFromFactoryBean 方法中 调用 doGetObjectFromFactoryBean 方法来获取 FactoryBean 中的 bean实例。下面我们来看一下getObjectFromFactoryBean 代码：
```
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		// 判断是否是单例模式 && singletonObjects 尚未缓存该bean （containsSingleton调用的是 singletonObjects ）
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
				// 尝试从 factoryBeanObjectCache 缓存中获取
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					// 在这个方法中进行解析。调用 FactoryBean 的 getObject 方法
					object = doGetObjectFromFactoryBean(factory, beanName);

					// 因为是单例模式，所以要保证变量的全局唯一。所以这里如果缓存中已经创建好了bean则替换为已经创建好的bean
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						// 如果允许调用bean的后置处理器。因为这里是直接将bean创建返回了，如果要调用后置方法则只能在这里调用。
						if (shouldPostProcess) {
							if (isSingletonCurrentlyInCreation(beanName)) {
								// Temporarily return non-post-processed object, not storing it yet..
								return object;
							}
							// 将beanName 添加到 singletonsCurrentlyInCreation 中缓存，表示当前bean正在创建中
							beforeSingletonCreation(beanName);
							try {
								// 调用了ObjectFactory的后置处理器。
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
							finally {
							// 将beanName 从 singletonsCurrentlyInCreation 中移除，表示当前bean已经创建结束
								afterSingletonCreation(beanName);
							}
						}
						// return this.singletonObjects.containsKey(beanName); 如果 singletonObjects缓存中存在当前beanName，则将其缓存到 factoryBeanObjectCache 中。
						if (containsSingleton(beanName)) {
							// 这里保存的是 beanName : FactoryBean
							this.factoryBeanObjectCache.put(beanName, object);
						}
					}
				}
				return object;
			}
		}
		else {
			// FactoryBean 非单例直接调用 getObject 方法
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			// 如果允许调用后置方法，则调用postProcessObjectFromFactoryBean 方法
			if (shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}

```

### 原型模式的依赖检查 - isPrototypeCurrentlyInCreation
```
	if (isPrototypeCurrentlyInCreation(beanName)) {
		throw new BeanCurrentlyInCreationException(beanName);
	}
```
简而言之就是 单例模式下才会尝试去解决循环依赖的问题，而原型模式则无法解决。也就是 isPrototypeCurrentlyInCreation 返回true，则抛出异常。 

需要注意的是还有一个 方法是 判断单例模式的依赖检查 ： isSingletonCurrentlyInCreation

递归 parentBeanFactory
这一步的逻辑比较简单，如下：

首先通过 containsBeanDefinition(beanName) 方法判断当前beanFactory中是否有bean的定义，如果有，皆大欢喜。直接进入下一步。
如果没有，且 parentBeanFactory 不为空，则会通过递归的方式，尝试从 parentBeanFactory 中加载bean定义。
```
// 5. 如果 beanDefinitionMap 中也就是在所有已经加载的类中不包含beanName，则尝试从parentBeanFactory中检测
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				// 递归到BeanFactory中检测
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}
```

其中 DefaultListableBeanFactory#containsBeanDefinition 代码如下
```
	@Override
	public boolean containsBeanDefinition(String beanName) {
		Assert.notNull(beanName, "Bean name must not be null");
		return this.beanDefinitionMap.containsKey(beanName);
	}
```

合并 BeanDefinition
BeanDefinition 顾名思义，就是关于bean 的定义信息。通过xml的bean定义可以很清楚的看到一些属性定义。

所以这一步就是检查当前 BeanDefinition 是否有父BeanDefinition ，如果有将一些属性和当前bean合并，生成一个 RootBeanDefinition。
```
	final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
		checkMergedBeanDefinition(mbd, beanName, args);
```
这一块的调用链路：getMergedLocalBeanDefinition -> getMergedBeanDefinition -> getMergedBeanDefinition。所以我们 这里直接来看 getMergedBeanDefinition 方法。

首先，mdb 即 MergedBeanDefinition 的缩写，即一个合并的beanDefinition。
```
	// 如果给定bean的定义是子bean定义，则通过与父级合并返回RootBeanDefinition。
	protected RootBeanDefinition getMergedBeanDefinition(
			String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
			throws BeanDefinitionStoreException {
		
		synchronized (this.mergedBeanDefinitions) {
			RootBeanDefinition mbd = null;
			RootBeanDefinition previous = null;

			// Check with full lock now in order to enforce the same merged instance.
			if (containingBd == null) {
				mbd = this.mergedBeanDefinitions.get(beanName);
			}

			if (mbd == null || mbd.stale) {
				previous = mbd;
				// 判断如果parentName为空则没必要进行合并了，直接克隆返回即可
				if (bd.getParentName() == null) {
					// Use copy of given root bean definition.
					if (bd instanceof RootBeanDefinition) {
						mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
					}
					else {
						mbd = new RootBeanDefinition(bd);
					}
				}
				else {
					// Child bean definition: needs to be merged with parent.
					BeanDefinition pbd;
					try {
						// 转换beanName
						String parentBeanName = transformedBeanName(bd.getParentName());
						// 递归调用，解析更上层的parent BeanDefinition 
						if (!beanName.equals(parentBeanName)) {
							pbd = getMergedBeanDefinition(parentBeanName);
						}
						else {
							BeanFactory parent = getParentBeanFactory();
							if (parent instanceof ConfigurableBeanFactory) {
								pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
							}
							else {
								throw new NoSuchBeanDefinitionException(parentBeanName,
										"Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
										"': cannot be resolved without a ConfigurableBeanFactory parent");
							}
						}
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
								"Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
					}
					// Deep copy with overridden values.
					// 深拷贝
					mbd = new RootBeanDefinition(pbd);
					mbd.overrideFrom(bd);
				}

				// Set default singleton scope, if not configured before.
				if (!StringUtils.hasLength(mbd.getScope())) {
					mbd.setScope(SCOPE_SINGLETON);
				}

				if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
					mbd.setScope(containingBd.getScope());
				}

				// Cache the merged bean definition for the time being
				// (it might still get re-merged later on in order to pick up metadata changes)
				if (containingBd == null && isCacheBeanMetadata()) {
					this.mergedBeanDefinitions.put(beanName, mbd);
				}
			}
			if (previous != null) {
				copyRelevantMergedBeanDefinitionCaches(previous, mbd);
			}
			return mbd;
		}
	}
```

### 寻找依赖
这一步也是针对 BeanDefinition 的 dependsOn 属性来说的（对应注解则是 @DependsOn。主要是 优先加载Bean 的 depends-on依赖。

注：
在 BeanDefinition 加载过程中，通过扫描路径加载的时候，通过 ClassPathBeanDefinitionScanner#doScan方法时会调用AnnotationConfigUtils#processCommonDefinitionAnnotations 来进行BeanDefinition 封装，其中就包含了诸多注解的解析，如下：

因为bean 的初始化过程中很可能会用到某些属性，而某些属性很可能是动态配置的，并且配置成依赖于其他的bean，那么这个时候就有必要先加载依赖的bean，所以，在Spring的加载顺寻中，在初始化某一个bean的时候首先会初始化这个bean所对应的依赖。
```
		// 获取依赖项
		String[] dependsOn = mbd.getDependsOn();
		if (dependsOn != null) {
			for (String dep : dependsOn) {
				// 判断是否有循环依赖的情况 ： A依赖B，B依赖A
				if (isDependent(beanName, dep)) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
				}
				// 注册依赖信息，将依赖信息保存到 dependentBeanMap、dependenciesForBeanMap中
				registerDependentBean(dep, beanName);
				try {
					// 获取依赖的bean，这一步又回到了最初的getBean
					getBean(dep);
				}
				catch (NoSuchBeanDefinitionException ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
				}
			}
		}
```
下面我们来看一看 isDependent(beanName, dep) 的处理逻辑
```
	private boolean isDependent(String beanName, String dependentBeanName, @Nullable Set<String> alreadySeen) {
		// 是否已经检测过了，上层一直是null
		if (alreadySeen != null && alreadySeen.contains(beanName)) {
			return false;
		}
		// 格式化beanName
		String canonicalName = canonicalName(beanName);
		// 获取 依赖于 beanName 的 bean集合
		Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
		if (dependentBeans == null) {
			return false;
		}
		// 如果 依赖于 beanName中存在 dependentBeanName 则说明存在循环依赖。
		// 代码走到这里说明是beanName 创建过程中要依赖 dependentBeanName。但是dependentBeans.contains(dependentBeanName) = true 则说明dependentBeanName依赖于beanName
		// 造成了 A依赖B，B依赖A的情况
		if (dependentBeans.contains(dependentBeanName)) {
			return true;
		}
		// 递归，确定没有A->B->C-A 这种长链路的循环依赖情况
		for (String transitiveDependency : dependentBeans) {
			if (alreadySeen == null) {
				alreadySeen = new HashSet<>();
			}
			alreadySeen.add(beanName);
			if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
				return true;
			}
		}
		return false;
	}
```
注：
@DependsOn：注解是在另外一个实例创建之后才创建当前实例，也就是，最终两个实例都会创建，只是顺序不一样
@ConditionalOnBean ：注解是只有当另外一个实例存在时，才创建，否则不创建，也就是，最终有可能两个实例都创建了，有可能只创建了一个实例，也有可能一个实例都没创建

## bean创建
上面这么多逻辑，都是准备工作。确定缓存中没有当前bean，并且当前bean的创建合法。准备开始创建。不过这里考虑到不同的 Scope，所以针对不同的Scope进行不同的初始化操作。创建bean 的过程也很复杂。

下面代码就是根据 Singleton、Prototype 或者其他Scope 走不同的流程创建bean。
```
				// Create bean instance.
				if (mbd.isSingleton()) {
					// 单例的创建
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							destroySingleton(beanName);
							throw ex;
						}
					});
					// 解决FactoryBean的问题
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					// 原型模式的回调
					Object prototypeInstance = null;
					try {
						// 保存当前线程正在创建的beanName 到 prototypesCurrentlyInCreation 中
						beforePrototypeCreation(beanName);
						// 直接创建bean
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						// 移除当前线程正在创建的beanName 从 prototypesCurrentlyInCreation 中
						afterPrototypeCreation(beanName);
					}
					// 解决FactoryBean的问题
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
						// 保存当前线程正在创建的beanName 到 prototypesCurrentlyInCreation 中
							beforePrototypeCreation(beanName);
							try {
								// 创建bean
								return createBean(beanName, mbd, args);
							}
							finally {
								// 移除当前线程正在创建的beanName 从 prototypesCurrentlyInCreation 中
								afterPrototypeCreation(beanName);
							}
						});
						//  解决FactoryBean的问题
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
```

## 类型转换
到这里整个流程基本就结束了。通常对该方法的调用参数 requiredType 是null，。但某些情况可能会出现返回的bean是个String类型，但是requiredType 传入的却是Integer类型，这时候就会触发这一步的操作，将String类型转换为Integer类型。
```
	if (requiredType != null && !requiredType.isInstance(bean)) {
		try {
			T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
			if (convertedBean == null) {
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
			return convertedBean;
		}
		catch (TypeMismatchException ex) {
			if (logger.isTraceEnabled()) {
				logger.trace("Failed to convert bean '" + name + "' to required type '" +
						ClassUtils.getQualifiedName(requiredType) + "'", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
```














