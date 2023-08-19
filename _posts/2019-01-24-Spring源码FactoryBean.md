---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码FactoryBean

## 简介
一般情况下，Spring通过反射机制利用bean的class属性指定实现类来实例化bean。但是在某些情况下，实例化bean 的过程比较复杂，在XML配置下如果按照传统的编码方式，则需要进行大量配置，灵活性受限，这时候Spring 提供了一个 org.springframework.beans.factory.FactoryBean 的工厂类接口。可以通过该接口定制实例化bean 的逻辑。

除此之外，FactoryBean在Spring框架中都占有很重要的地位，Spring自身就提供了非常多的FactoryBean的实现。

首先我们来看看 FactoryBean 接口的定义。方法比较简单，见名知意。
```
public interface FactoryBean<T> {
	// 属性名
	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

	// 获取bean实例，实例化bean 的逻辑就在这里实现
	@Nullable
	T getObject() throws Exception;
	// 获取bean类型
	@Nullable
	Class<?> getObjectType();
	// 是否是单例
	default boolean isSingleton() {
		return true;
	}

}
```
- T getObject() 
返回由FactoryBean创建的bean实例，如果isSingleton() 返回true，则会将该实例方法Spring容器的单例缓冲池中
- Class<?> getObjectType()
返回由FactoryBean 创建的bean 的类型
- boolean isSingleton() 
返回由FactoryBean 创建的bean实例的作用域是singleton还是Prototype。

FactoryBean就是一种方法，要么是解决生成BeanDefinition的困难；要么是解决反射实例化bean时候遇到的困难，将spring不好自动化处理的逻辑写到getObject方法内。

比如 ：在Mybatis 中，默认情况下 Mybatis会将所有的Mapper生成的代理对象保存到 SqlSession 中，如果需要获取 Mapper对象可以通过SqlSession.geMapper 获取。而当使用Spring 集成 Mybatis 框架时，Spring会将所有的Bean 都保存到单独的缓存中，当进行属性注入时从缓存中获取对象并赋值。

而Mapper 对象保存到 SqlSession中，那么Spring 如何获取Mapper对象则成了一个问题。这时 Mybatis 为每个Mapper对象创建了一个 MapperFactoryBean。当Spring需要获取 Mapper 对象时，会通过MapperFactoryBean 来获取，而 MapperFactoryBean#getObject 实现如下，MapperFactoryBean会从 SqlSession 中获取 Mapper。

```
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
```

## FactoryBean 简单使用

下面举个简单的例子进一步说明FactoryBean 的用法。
```
@Component
public class DemoFactoryBean implements FactoryBean<DemoBean> {
    @Override
    public DemoBean getObject() throws Exception {
        System.out.println("DemoFactoryBean.getObject");
        return new DemoBean();
    }

    @Override
    public Class<?> getObjectType() {
        return DemoBean.class;
    }
}
```

```
@SpringBootApplication
public class BeanInitDemoApplication {
    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(BeanInitDemoApplication.class, args);
        Object demoFactoryBean = run.getBean("demoFactoryBean");
        Object bean = run.getBean("&demoFactoryBean");
        System.out.println("BeanInitDemoApplication.main");
    }

}
```
如上一个简单的例子，调用结果如下，可以看到：我们这里直接调用 run.getBean("demoFactoryBean"); 返回的并不是 DemoFactoryBean ，而是DemoBean 。

而我们使用 run.getBean("&demoFactoryBean"); 返回的结果却是DemoFactoryBean 。

实际上，当调用 run.getBean("demoFactoryBean"); 时，Spring通过反射会发现 DemoFactoryBean 实现了FactoryBean接口，则会直接调用 其getObject() 方法，并将方法的返回值注入到Spring容器中。而如果想要获得DemoFactoryBean 实例，则需要在 beanName前加上 & ，即 run.getBean("&demoFactoryBean");

## 源码解读
对于 FactoryBean的处理在 AbstractBeanFactory#getObjectForBeanInstance 方法中完成，我们需要判断其是否是 FactoryBean 的 实现类，如果是，则需要将其getObject() 的结果注入。所以该方法的功能简单来说就是用来检测当前bean是否是FactoryBean类型的bean，如果是，则调用其getObject() 方法，并将其返回值作为bean
```
	bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
```
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

### getObjectFromFactoryBean
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

### doGetObjectFromFactoryBean
doGetObjectFromFactoryBean 方法的实现在 FactoryBeanRegistrySupport#getObjectFromFactoryBean 中，作用是 从 FactoryBean 中获取 Object 对象。代码如下：
```
	private Object doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName) throws BeanCreationException {
		Object object;
		try {
			// 如果设置了安全管理器，尝试通过特权方式从 FactoryBean 中获取 Object
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				// 直接从 FactoryBean 中获取 Object
				object = factory.getObject();
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// Do not accept a null value for a FactoryBean that's not fully
		// initialized yet: Many FactoryBeans just return null then.
		if (object == null) {
			// 如果当期bean是单例 && 正在创建中，则抛出异常
			if (isSingletonCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(
						beanName, "FactoryBean which is currently in creation returned null from getObject");
			}
			object = new NullBean();
		}
		return object;
	}

```









