---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring扩展点BeanDefinitionRegistryPostProcessor
它是Spring框架的一个扩展点，用于对Bean定义的注册过程进行干预和定制。 继承BeanFactoryPostProcessor接口，并在其基础上扩展了一个新的方法，即：postProcessBeanDefinitionRegistry()方法。

## BeanDefinitionRegistryPostProcessor 用途：
在Spring容器初始化时，首先会读取应用程序中的配置文件，并解析出所有的Bean定义，然后将这些Bean定义注册到容器中。

在这个过程中，BeanDefinitionRegistryProcessor提供了一种机制，允许开发人员在Bean定义注册之前和之后对Bean定义进行自定义处理，例如添加，修改或删除Bean定义等。

BeanDefinitionRegistryPostProcessor 具体原理：
具体来说，BeanDefinitionRegistryPostProcessor提供了以下两个方法：
- postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)：该方法在所有Bean定义加载完成之后，Bean实例化之前被调用，允许开发人员对Bean定义进行自定义修改，例如添加，修改或删除Bean定义等。
- postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory): 该方法是继承自BeanFactoryPostProcessor接口的方法，用于在BeanFactory完成实例化之后对BeanFactory进行后置处理。

那么我们通过BeanDefinitionRegistryPostProcessor接口，开发人员可以在Spring容器启动时干预Bean的注册过程，从而实现对Bean的自定义处理。
例如：
可以通过该接口来动态注册Bean定义，从而实现基于注解或者其他方式的自动发现和注册Bean。
同时，该接口也提供了一种扩展点，允许开发人员在Bean定义注册之后再进行后置处理，例如对Bean的属性进行统一设置，验证等操作。

Spring容器初始化时，从资源中读取到bean的相关定义后，保存在BeanDefinitionMap，在实例化bean的操作就是依据这些bean的定义来做的，而在实例化之前，Spring允许我们通过自定义扩展来改变bean的定义，定义一旦变了，后面的实例也就变了，而beanFactory后置处理器，即BeanFactoryPostProcessor就是用来改变bean定义的

BeanDefinitionRegistryPostProcessor继承了BeanFactoryPostProcessor接口，BeanFactoryPostProcessor的实现类在其postProcessBeanFactory方法被调用时，可以对bean的定义进行控制，因此BeanDefinitionRegistryPostProcessor的实现类一共要实现以下两个方法：

void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException：
该方法的实现中，主要用来对bean定义做一些改变。

void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException：
该方法用来注册更多的bean到spring容器中，详细观察入参BeanDefinitionRegistry接口，看看这个参数能带给我们什么能力。

从BeanDefinitionRegistry可以看到，BeanDefinitionRegistry提供了丰富的方法来操作BeanDefinition，判断、注册、移除等方法都准备好了，我们在编写postProcessBeanDefinitionRegistry方法的内容时，就能直接使用入参registry的这些方法来完成判断和注册、移除等操作。

org.springframework.context.support.AbstractApplicationContext#refresh中的invokeBeanFactoryPostProcessors(beanFactory);用来找出所有beanFactory后置处理器，并且调用这些处理器来改变bean的定义。

invokeBeanFactoryPostProcessors(beanFactory)实际上是委托org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());方法处理的。

所有实现了BeanDefinitionRegistryPostProcessor接口的bean，其postProcessBeanDefinitionRegistry方法都会调用，然后再调用其postProcessBeanFactory方法，这样一来，我们如果自定义了BeanDefinitionRegistryPostProcessor接口的实现类，那么我们开发的postProcessBeanDefinitionRegistry和postProcessBeanFactory方法都会被执行一次

示例代码：
```
@Component
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor;
import org.springframework.beans.factory.support.RootBeanDefinition;
import org.springframework.stereotype.Component;

/**
 * 自定义的BeanDefinitionRegistryPostProcessor
 */
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

    /**
     * 重写BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法
     */
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        // 创建一个RootBeanDefinition实例，该实例对应的BeanClass是MyBean
        RootBeanDefinition beanDefinition = new RootBeanDefinition(MyBean.class);
        // 向BeanDefinitionRegistry注册MyBean的BeanDefinition
        registry.registerBeanDefinition("myBean", beanDefinition);
    }

    /**
     * 重写BeanDefinitionRegistryPostProcessor的postProcessBeanFactory方法
     */
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // do nothing
    }

    /**
     * 自定义的Bean
     */
    public static class MyBean {
        private String message = "Hello, World!";

        public String getMessage() {
            return message;
        }

        public void setMessage(String message) {
            this.message = message;
        }
    }
}
```

