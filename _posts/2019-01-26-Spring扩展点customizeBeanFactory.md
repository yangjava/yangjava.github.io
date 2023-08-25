---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring扩展点customizeBeanFactory
Spring的强大之处不仅仅在于它为Java开发者提供了极大便利，更在于它的开放式架构，使得用户可以拥有最大扩展Spring的能力。

## customizeBeanFactory
```
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
		// 如果属性allowBeanDefinitionOverriding不为空，设置给beanFactory对象相应属性，是否允许覆盖同名称的不同定义的对象
		if (this.allowBeanDefinitionOverriding != null) {
			beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		// 如果属性allowCircularReferences不为空，设置给beanFactory对象相应属性，是否允许bean之间存在循环依赖
		if (this.allowCircularReferences != null) {
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
```
在AbstractRefreshableApplicationContext类中有一个customizeBeanFactory方法是留给子类扩展，它是在refresh()的第二个方法obtainFreshBeanFactory()–>refreshBeanFactory()方法中调用。

```
protected final void refreshBeanFactory() throws BeansException {
		// 如果存在beanFactory，则销毁beanFactory
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			// 创建DefaultListableBeanFactory对象
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			// 为了序列化指定id，可以从id反序列化到beanFactory对象
			beanFactory.setSerializationId(getId());
			// 定制beanFactory，设置相关属性，包括是否允许覆盖同名称的不同定义的对象以及循环依赖
			customizeBeanFactory(beanFactory);
			// 初始化documentReader,并进行XML文件读取及解析,默认命名空间的解析，自定义标签的解析
			loadBeanDefinitions(beanFactory);
			this.beanFactory = beanFactory;
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```
此方法是用来实现BeanFactory的属性设置，主要是设置两个属性：
- allowBeanDefinitionOverriding：是否允许覆盖同名称的不同定义的对象。
- allowCircularReferences：是否允许bean之间的循环依赖。

如下例子：
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

    @Override
    protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
        super.setAllowBeanDefinitionOverriding(false);
        super.setAllowCircularReferences(false);
        super.customizeBeanFactory(beanFactory);
    }
}
```

```
public class Test {

    public static void main(String[] args) {
        MyClassPathXmlApplicationContext ac = new MyClassPathXmlApplicationContext("applicationContext.xml");

//        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring-${username}.xml");
    }
}
```



















