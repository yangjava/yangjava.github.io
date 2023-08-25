---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring扩展点initPropertySources
Spring的强大之处不仅仅在于它为Java开发者提供了极大便利，更在于它的开放式架构，使得用户可以拥有最大扩展Spring的能力。

## initPropertySources
initPropertySources() ：这个方法是为了给用户自己实现初始化逻辑，可以初始化一些属性资源。因此Spring并没有实现这个方法。
```
protected void initPropertySources() {
		// For subclasses: do nothing by default.
	}
```
在AbstractApplicationContext类中有一个initPropertySources方法是留给子类扩展，它是在refresh()的第一个方法prepareRefresh();方法中调用。
```
protected void prepareRefresh() {
		// Switch to active.
		// 设置容器启动的时间
		this.startupDate = System.currentTimeMillis();
		// 容器的关闭标志位
		this.closed.set(false);
		// 容器的激活标志位
		this.active.set(true);

		// 记录日志
		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// Initialize any placeholder property sources in the context environment.
		// 留给子类覆盖，初始化属性资源
		initPropertySources();

		// Validate that all properties marked as required are resolvable:
		// see ConfigurablePropertyResolver#setRequiredProperties
		// 创建并获取环境对象，验证需要的属性文件是否都已经放入环境中
		getEnvironment().validateRequiredProperties();

		// Store pre-refresh ApplicationListeners...
		// 判断刷新前的应用程序监听器集合是否为空，如果为空，则将监听器添加到此集合中
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			// 如果不等于空，则清空集合元素对象
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		// 创建刷新前的监听事件集合
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```
所以我们可以继承此类或其子类来重写initPropertySources方法，实现一些扩展。
```
public class MyClassPathXmlApplicationContext extends ClassPathXmlApplicationContext {


    public MyClassPathXmlApplicationContext(String... configLocations){
        super(configLocations);
    }

    @Override
    protected void initPropertySources() {
        System.out.println("扩展initPropertySource");
        //这里添加了一个name属性到Environment里面，以方便我们在后面用到
        getEnvironment().getSystemProperties().put("name","bobo");
        //这里要求Environment中必须包含username属性，如果不包含，则抛出异常
        getEnvironment().setRequiredProperties("username");
    }
}
```
此处我们做了两个扩展：
第一，向Environment中添加了一个属性值。
第二：我们设置了一个必要的系统属性username，当Environment中不包含username属性时系统会抛出异常。
```
public class Test {

    public static void main(String[] args) {
        MyClassPathXmlApplicationContext ac = new MyClassPathXmlApplicationContext("applicationContext.xml");

//        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring-${username}.xml");
    }
}
```