---
layout: post
categories: Spring
description: none
keywords: Spring
---
# Spring加载Bean
Spring 的核心功能就是实现对 Bean 的管理，比如 Bean 的注册、注入、依赖等。

## bean的加载
对于加载bean的功能，在Spring中的调用方式为：
```java
MyTestBean bean=(MyTestBean) bf.getBean("myTestBean")
```
这句代码实现了什么样的功能呢？我们可以先快速体验一下Spring中代码是如何实现的。
```java
public Object getBean(String name) throws BeansException {
         return doGetBean(name, null, null, false);
}

protected <T> T doGetBean(
             final String name, final Class<T> requiredType, final Object[] args,   
boolean typeCheckOnly) throws BeansException {
         //提取对应的beanName
         final String beanName = transformedBeanName(name);
         Object bean;

         /*
          * 检查缓存中或者实例工厂中是否有对应的实例
          * 为什么首先会使用这段代码呢,
          * 因为在创建单例bean的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，
          * Spring创建bean的原则是不等bean创建完成就会将创建bean的ObjectFactory提早曝光
          * 也就是将ObjectFactory加入到缓存中，一旦下个bean创建时候需要依赖上个bean则直接使用ObjectFactory
          */
         //直接尝试从缓存获取或者singletonFactories中的ObjectFactory中获取
         Object sharedInstance = getSingleton(beanName);
         if (sharedInstance != null && args == null) {
             if (logger.isDebugEnabled()) {
                 if (isSingletonCurrentlyInCreation(beanName)) {
                     logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                             "' that is not fully initialized yet - a consequence of a circular reference");
                 }
                 else {
                     logger.debug("Returning cached instance of singleton bean '" +  
 beanName + "'");
                 }
             }
//返回对应的实例，有时候存在诸如BeanFactory的情况并不是直接返回实例本身而是返回指定方法返回的实例
             bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
         }else {
             //只有在单例情况才会尝试解决循环依赖，原型模式情况下，如果存在
             //A中有B的属性，B中有A的属性，那么当依赖注入的时候，就会产生当A还未创建完的时候因为
             //对于B的创建再次返回创建A，造成循环依赖，也就是下面的情况
             //isPrototypeCurrentlyInCreation(beanName)为true
             if (isPrototypeCurrentlyInCreation(beanName)) {
                 throw new BeanCurrentlyInCreationException(beanName);
             }

             BeanFactory parentBeanFactory = getParentBeanFactory();
             //如果beanDefinitionMap中也就是在所有已经加载的类中不包括beanName则尝试从  
             //parentBeanFactory中检测
             if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                 String nameToLookup = originalBeanName(name);
                 //递归到BeanFactory中寻找
                 if (args != null) {
                     return (T) parentBeanFactory.getBean(nameToLookup, args);
                 }
                 else {
                     return parentBeanFactory.getBean(nameToLookup, requiredType);
                 }
             }
             //如果不是仅仅做类型检查则是创建bean，这里要进行记录
             if (!typeCheckOnly) {
                 markBeanAsCreated(beanName);
             }

             //将存储XML配置文件的GernericBeanDefinition转换为RootBeanDefinition，如果指定  
             //BeanName是子Bean的话同时会合并父类的相关属性
             final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
             checkMergedBeanDefinition(mbd, beanName, args);

             String[] dependsOn = mbd.getDependsOn();
             //若存在依赖则需要递归实例化依赖的bean
             if (dependsOn != null) {
                 for (String dependsOnBean : dependsOn) {
                     getBean(dependsOnBean);
                     //缓存依赖调用
                     registerDependentBean(dependsOnBean, beanName);
                 }
             }

             //实例化依赖的bean后便可以实例化mbd本身了
             //singleton模式的创建
             if (mbd.isSingleton()) {
                 sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                     public Object getObject() throws BeansException {
                         try {
                             return createBean(beanName, mbd, args);
                         }
                         catch (BeansException ex) {
                             destroySingleton(beanName);
                             throw ex;
                         }
                     }
                 });
                 bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
             }else if (mbd.isPrototype()) {
                 //prototype模式的创建(new)
                 Object prototypeInstance = null;
                 try {
                     beforePrototypeCreation(beanName);
                     prototypeInstance = createBean(beanName, mbd, args);
                 }
                 finally {
                     afterPrototypeCreation(beanName);
                 }
                 bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
             }else {
                 //指定的scope上实例化bean
                 String scopeName = mbd.getScope();
                 final Scope scope = this.scopes.get(scopeName);
                 if (scope == null) {
                     throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
                 }
                 try {
                     Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                         public Object getObject() throws BeansException {
                             beforePrototypeCreation(beanName);
                             try {
                                 return createBean(beanName, mbd, args);
                             }
                             finally {
                                 afterPrototypeCreation(beanName);
                             }
                         }
                     });
                     bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                 }
                 catch (IllegalStateException ex) {
                     throw new BeanCreationException(beanName,
                             "Scope '" + scopeName + "' is not active for the current thread; " +
                             "consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                             ex);
                 }
             }
         }

         //检查需要的类型是否符合bean的实际类型
         if (requiredType != null && bean != null && !requiredType.isAssignableFrom   
(bean.getClass())) {
             try {
                 return getTypeConverter().convertIfNecessary(bean, requiredType);
             }
             catch (TypeMismatchException ex) {
                 if (logger.isDebugEnabled()) {
                     logger.debug("Failed to convert bean '" + name + "' to required   
type [" +
                             ClassUtils.getQualifiedName(requiredType) + "]", ex);
                 }
                 throw new BeanNotOfRequiredTypeException(name, requiredType, bean.   
getClass());
             }
         }
         return (T) bean;
}
```

Spring加载bean的过程大致如下。
### 转换对应beanName
或许很多人不理解转换对应beanName是什么意思，传入的参数name不就是beanName吗？其实不是，这里传入的参数可能是别名，也可能是FactoryBean，所以需要进行一系列的解析，这些解析内容包括如下内容。 
  - 去除FactoryBean的修饰符，也就是如果name="&aa"，那么会首先去除&而使name="aa"。
  - 取指定alias所表示的最终beanName，例如别名A指向名称为B的bean则返回B；若别名A指向别名B，别名B又指向名称为C的bean则返回C。

### 尝试从缓存中加载单例
单例在Spring的同一个容器内只会被创建一次，后续再获取bean，就直接从单例缓存中获取了。当然这里也只是尝试加载，首先尝试从缓存中加载，如果加载不成功则再次尝试从singletonFactories中加载。因为在创建单例bean的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，在Spring中创建bean的原则是不等bean创建完成就会将创建bean的ObjectFactory提早曝光加入到缓存中，一旦下一个bean创建时候需要依赖上一个bean则直接使用ObjectFactory

### bean的实例化
如果从缓存中得到了bean的原始状态，则需要对bean进行实例化。这里有必要强调一下，缓存中记录的只是最原始的bean状态，并不一定是我们最终想要的bean。举个例子，假如我们需要对工厂bean进行处理，那么这里得到的其实是工厂bean的初始状态，但是我们真正需要的是工厂bean中定义的factory-method方法中返回的bean，而getObjectForBeanInstance就是完成这个工作的，后续会详细讲解。

### 原型模式的依赖检查
只有在单例情况下才会尝试解决循环依赖，如果存在A中有B的属性，B中有A的属性，那么当依赖注入的时候，就会产生当A还未创建完的时候因为对于B的创建再次返回创建A，造成循环依赖，也就是情况：isPrototypeCurrentlyInCreation(beanName)判断true。

### 检测parentBeanFactory
从代码上看，如果缓存没有数据的话直接转到父类工厂上去加载了，这是为什么呢？

可能读者忽略了一个很重要的判断条件：parentBeanFactory != null && !containsBean Definition (beanName)，parentBeanFactory != null。parentBeanFactory如果为空，则其他一切都是浮云，这个没什么说的，但是!containsBeanDefinition(beanName)就比较重要了，它是在检测如果当前加载的XML配置文件中不包含beanName所对应的配置，就只能到parentBeanFactory去尝试下了，然后再去递归的调用getBean方法。

### 将存储XML配置文件的GernericBeanDefinition转换为RootBeanDefinition
因为从XML配置文件中读取到的bean信息是存储在GernericBeanDefinition中的，但是所有的bean后续处理都是针对于RootBeanDefinition的，所以这里需要进行一个转换，转换的同时如果父类bean不为空的话，则会一并合并父类的属性。

### 寻找依赖
因为bean的初始化过程中很可能会用到某些属性，而某些属性很可能是动态配置的，并且配置成依赖于其他的bean，那么这个时候就有必要先加载依赖的bean，所以，在Spring的加载顺寻中，在初始化某一个bean的时候首先会初始化这个bean所对应的依赖。

### 针对不同的scope进行bean的创建
我们都知道，在Spring中存在着不同的scope，其中默认的是singleton，但是还有些其他的配置诸如prototype、request之类的。在这个步骤中，Spring会根据不同的配置进行不同的初始化策略。

### 类型转换
程序到这里返回bean后已经基本结束了，通常对该方法的调用参数requiredType是为空的，但是可能会存在这样的情况，返回的bean其实是个String，但是requiredType却传入Integer类型，那么这时候本步骤就会起作用了，它的功能是将返回的bean转换为requiredType所指定的类型。当然，String转换为Integer是最简单的一种转换，在Spring中提供了各种各样的转换器，用户也可以自己扩展转换器来满足需求。

## FactoryBean的使用
Spring通过反射机制利用bean的class属性指定实现类来实例化bean 。在某些情况下，实例化bean过程比较复杂，如果按照传统的方式，则需要在<bean>中提供大量的配置信息，配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring为此提供了一个org.Springframework.bean.factory.FactoryBean的工厂类接口，用户可以通过实现该接口定制实例化bean的逻辑。

FactoryBean接口对于Spring框架来说占有重要的地位，Spring 自身就提供了70多个FactoryBean的实现。它们隐藏了实例化一些复杂bean的细节，给上层应用带来了便利。从Spring 3.0 开始， FactoryBean开始支持泛型，即接口声明改为FactoryBean<T> 的形式：
```java
package org.Springframework.beans.factory;  
public interface FactoryBean<T> {  
   T getObject() throws Exception;  
   Class<?> getObjectType();  
   boolean isSingleton();  
}
```
在该接口中还定义了以下3个方法。
- T getObject()：返回由FactoryBean创建的bean实例，如果isSingleton()返回true，则该实例会放到Spring容器中单实例缓存池中。
- boolean isSingleton()：返回由FactoryBean创建的bean实例的作用域是singleton还是prototype。
- Class<T> getObjectType()：返回FactoryBean创建的bean类型。

当配置文件中<bean>的class属性配置的实现类是FactoryBean时，通过 getBean()方法返回的不是FactoryBean本身，而是FactoryBean#getObject()方法所返回的对象，相当于FactoryBean#getObject()代理了getBean()方法。例如：如果使用传统方式配置下面Car的<bean>时，Car的每个属性分别对应一个<property>元素标签。
```java
public   class  Car  {  
        private   int maxSpeed ;  
        private  String brand ;  
        private   double price ;  
       //get/set方法
}
```
如果用FactoryBean的方式实现就会灵活一些，下例通过逗号分割符的方式一次性地为Car的所有属性指定配置值：
```java
public   class  CarFactoryBean  implements  FactoryBean<Car>  {  
    private  String carInfo ;  
    public  Car getObject ()   throws  Exception  {  
        Car car =  new  Car () ;  
        String []  infos =  carInfo .split ( "," ) ;  
        car.setBrand ( infos [ 0 ]) ;  
        car.setMaxSpeed ( Integer. valueOf ( infos [ 1 ])) ;  
        car.setPrice ( Double. valueOf ( infos [ 2 ])) ;  
        return  car;  
    }  
    public  Class<Car> getObjectType ()   {  
        return  Car. class ;  
    }  
    public   boolean  isSingleton ()   {  
        return   false ;  
    }  
    public  String getCarInfo ()   {  
        return   this . carInfo ;  
    }  

    // 接受逗号分割符设置属性信息  
    public   void  setCarInfo ( String carInfo )   {  
        this . carInfo  = carInfo;  
    }  
}
```
有了这个CarFactoryBean后，就可以在配置文件中使用下面这种自定义的配置方式配置Car Bean了：
```java
<bean id="car" class="com.test.factorybean.CarFactoryBean" carInfo="超级跑车,400,2000000"/>
```
当调用getBean("car") 时，Spring通过反射机制发现CarFactoryBean实现了FactoryBean的接口，这时Spring容器就调用接口方法CarFactoryBean#getObject()方法返回。如果希望获取CarFactoryBean的实例，则需要在使用getBean(beanName) 方法时在beanName前显示的加上 "&" 前缀，例如getBean("&car")。

## 缓存中获取单例bean
介绍过FactoryBean的用法后，我们就可以了解bean加载的过程了。前面已经提到过，单例在Spring的同一个容器内只会被创建一次，后续再获取bean直接从单例缓存中获取，当然这里也只是尝试加载，首先尝试从缓存中加载，然后再次尝试尝试从singletonFactories中加载。因为在创建单例bean的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，Spring创建bean的原则是不等bean创建完成就会将创建bean的ObjectFactory提早曝光加入到缓存中，一旦下一个bean创建时需要依赖上个bean，则直接使用ObjectFactory。
```java
public Object getSingleton(String beanName) {
     //参数true设置标识允许早期依赖    
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
         //检查缓存中是否存在实例
         Object singletonObject = this.singletonObjects.get(beanName);
         if (singletonObject == null) {
             //如果为空，则锁定全局变量并进行处理
             synchronized (this.singletonObjects) {
                 //如果此bean正在加载则不处理
                 singletonObject = this.earlySingletonObjects.get(beanName);
                 if (singletonObject == null && allowEarlyReference) {
                     //当某些方法需要提前初始化的时候则会调用addSingletonFactory方法将对应的  
                     //ObjectFactory初始化策略存储在singletonFactories
                     ObjectFactory singletonFactory = this.singletonFactories.get   
(beanName);
                     if (singletonFactory != null) {
                         //调用预先设定的getObject方法
                         singletonObject = singletonFactory.getObject();
                         //记录在缓存中，earlySingletonObjects和singletonFactories互斥
                         this.earlySingletonObjects.put(beanName, singletonObject);
                         this.singletonFactories.remove(beanName);
                     }
                 }
             }
         }
         return (singletonObject != NULL_OBJECT ? singletonObject : null);
     }
```
这个方法因为涉及循环依赖的检测，以及涉及很多变量的记录存取，所以让很多读者摸不着头脑。这个方法首先尝试从singletonObjects里面获取实例，如果获取不到再从earlySingleton- Objects里面获取，如果还获取不到，再尝试从singletonFactories里面获取beanName对应的ObjectFactory，然后调用这个ObjectFactory的getObject来创建bean，并放到earlySingleton- Objects里面去，并且从singletonFacotories里面remove掉这个ObjectFactory，而对于后续的所有内存操作都只为了循环依赖检测时候使用，也就是在allowEarlyReference为true的情况下才会使用。

这里涉及用于存储bean的不同的map，可能让读者感到崩溃，简单解释如下。
- singletonObjects：用于保存BeanName和创建bean实例之间的关系，bean name --> bean instance。
- singletonFactories：用于保存BeanName和创建bean的工厂之间的关系，bean name --> ObjectFactory。
- earlySingletonObjects：也是保存BeanName和创建bean实例之间的关系，与singletonObjects的不同之处在于，当一个单例bean被放到这里面后，那么当bean还在创建过程中，就可以通过getBean方法获取到了，其目的是用来检测循环引用。
- registeredSingletons：用来保存当前所有已注册的bean。

## 从bean的实例中获取对象
在getBean方法中，getObjectForBeanInstance是个高频率使用的方法，无论是从缓存中获得bean还是根据不同的scope策略加载bean。总之，我们得到bean的实例后要做的第一步就是调用这个方法来检测一下正确性，其实就是用于检测当前bean是否是FactoryBean类型的bean，如果是，那么需要调用该bean对应的FactoryBean实例中的getObject()作为返回值。

无论是从缓存中获取到的bean还是通过不同的scope策略加载的bean都只是最原始的bean状态，并不一定是我们最终想要的bean。举个例子，假如我们需要对工厂bean进行处理，那么这里得到的其实是工厂bean的初始状态，但是我们真正需要的是工厂bean中定义的factory-method方法中返回的bean，而getObjectForBeanInstance方法就是完成这个工作的。
```java
protected Object getObjectForBeanInstance(
             Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

         //如果指定的name是工厂相关(以&为前缀)且beanInstance又不是FactoryBean类型则验证不通过
         if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
             throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.  
 getClass());
         }

         //现在我们有了个bean的实例，这个实例可能会是正常的bean或者是FactoryBean
         //如果是FactoryBean我们使用它创建实例，但是如果用户想要直接获取工厂实例而不是工厂的  
         //getObject方法对应的实例那么传入的name应该加入前缀&
         if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils. IsFactory   
Dereference(name)) {
             return beanInstance;
         }

         //加载FactoryBean
         Object object = null;
         if (mbd == null) {
             //尝试从缓存中加载bean
             object = getCachedObjectForFactoryBean(beanName);
         }
         if (object == null) {
             //到这里已经明确知道beanInstance一定是FactoryBean类型
             FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
             //containsBeanDefinition检测beanDefinitionMap中也就是在所有已经加载的类中检测  
             //是否定义beanName
             if (mbd == null && containsBeanDefinition(beanName)) {
                  //将存储XML配置文件的GernericBeanDefinition转换为RootBeanDefinition，如  
              //果指定BeanName是子Bean的话同时会合并父类的相关属性
                 mbd = getMergedLocalBeanDefinition(beanName);
             }
             //是否是用户定义的而不是应用程序本身定义的
             boolean synthetic = (mbd != null && mbd.isSynthetic());
             object = getObjectFromFactoryBean(factory, beanName, !synthetic);
         }
         return object;
}
```
真正的核心代码却委托给了getObjectFromFactoryBean，我们来看看getObjectForBeanInstance中的所做的工作。
- 对FactoryBean正确性的验证。
- 对非FactoryBean不做任何处理。
- 对bean进行转换。
- 将从Factory中解析bean的工作委托给getObjectFromFactoryBean。
```java
protected Object getObjectFromFactoryBean(FactoryBean factory, String beanName, Boolean  
 shouldPostProcess) {
         //如果是单例模式
         if (factory.isSingleton() && containsSingleton(beanName)) {
             synchronized (getSingletonMutex()) {
                 Object object = this.factoryBeanObjectCache.get(beanName);
                 if (object == null) {
                     object = doGetObjectFromFactoryBean(factory, beanName, shouldPostProcess);
                     this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
                 }
                 return (object != NULL_OBJECT ? object : null);
             }
         }
         else {
             return doGetObjectFromFactoryBean(factory, beanName, shouldPostProcess);
         }
     }
```
很遗憾，在这个代码中我们还是没有看到想要看到的代码，在这个方法里只做了一件事情，就是返回的bean如果是单例的，那就必须要保证全局唯一，同时，也因为是单例的，所以不必重复创建，可以使用缓存来提高性能，也就是说已经加载过就要记录下来以便于下次复用，否则的话就直接获取了。

在doGetObjectFromFactoryBean方法中我们终于看到了我们想要看到的方法，也就是object = factory.getObject()，是的，就是这句代码，我们的历程犹如剥洋葱一样，一层一层的直到最内部的代码实现，虽然很简单。
```java
private Object doGetObjectFromFactoryBean(
             final FactoryBean factory, final String beanName, final boolean shouldPostProcess)
             throws BeanCreationException {

         Object object;
         try {
             //需要权限验证
             if (System.getSecurityManager() != null) {
                 AccessControlContext acc = getAccessControlContext();
                 try {
                     object = AccessController.doPrivileged(new PrivilegedException  
Action< Object>() {
                         public Object run() throws Exception {
                                 return factory.getObject();
                             }
                         }, acc);
                 }
                 catch (PrivilegedActionException pae) {
                     throw pae.getException();
                 }
             }
             else {
                 //直接调用getObject方法
                 object = factory.getObject();
             }
         }
         catch (FactoryBeanNotInitializedException ex) {
             throw new BeanCurrentlyInCreationException(beanName, ex.toString());
         }
         catch (Throwable ex) {
             throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
         }
         if (object == null && isSingletonCurrentlyInCreation(beanName)) {
             throw new BeanCurrentlyInCreationException(
                     beanName, "FactoryBean which is currently in creation returned   
null from getObject");
         }

         if (object != null && shouldPostProcess) {
             try {
                 //调用ObjectFactory的后处理器
                 object = postProcessObjectFromFactoryBean(object, beanName);
             }
             catch (Throwable ex) {
                 throw new BeanCreationException(beanName, "Post-processing of the   
FactoryBean's object failed", ex);
             }
         }

         return object;
}
```
上面我们已经讲述了FactoryBean的调用方法，如果bean声明为FactoryBean类型，则当提取bean时提取的并不是FactoryBean，而是FactoryBean中对应的getObject方法返回的bean，而doGetObjectFromFactoryBean正是实现这个功能的。但是，我们看到在上面的方法中除了调用object = factory.getObject()得到我们想要的结果后并没有直接返回，而是接下来又做了些后处理的操作，这个又是做什么用的呢？于是我们跟踪进入AbstractAutowireCapableBeanFactory类的postProcessObjectFromFactoryBean方法：
AbstractAutowireCapableBeanFactory.java
```java
protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
         return applyBeanPostProcessorsAfterInitialization(object, beanName);
}
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
             throws BeansException {

         Object result = existingBean;
         for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
             result = beanProcessor.postProcessAfterInitialization(result, beanName);
             if (result == null) {
                 return result;
             }
         }
         return result;
}
```

## 获取单例
从缓存中获取单例的过程，那么，如果缓存中不存在已经加载的单例bean就需要从头开始bean的加载过程了，而Spring中使用getSingleton的重载方法实现bean的加载过程。
```java
public Object getSingleton(String beanName, ObjectFactory singletonFactory) {
         Assert.notNull(beanName, "'beanName' must not be null");
         //全局变量需要同步
         synchronized (this.singletonObjects) {
             //首先检查对应的bean是否已经加载过，因为singleton模式其实就是复用以创建的bean，  
             //所以这一步是必须的
             Object singletonObject = this.singletonObjects.get(beanName);
             //如果为空才可以进行singleto的bean的初始化
             if (singletonObject == null) {
                 if (this.singletonsCurrentlyInDestruction) {
                     throw new BeanCreationNotAllowedException(beanName,
                             "Singleton bean creation not allowed while the singletons of this factory are in destruction " +
                             "(Do not request a bean from a BeanFactory in a destroy   
method implementation!)");
                 }
                 if (logger.isDebugEnabled()) {
                     logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
                 }
                 beforeSingletonCreation(beanName);
                 boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
                 if (recordSuppressedExceptions) {
                     this.suppressedExceptions = new LinkedHashSet<Exception>();
                 }
                 try {
                     //初始化bean
                     singletonObject = singletonFactory.getObject();
                 }
                 catch (BeanCreationException ex) {
                     if (recordSuppressedExceptions) {
                         for (Exception suppressedException : this.suppressedExceptions) {
                             ex.addRelatedCause(suppressedException);
                         }
                     }
                     throw ex;
                 }
                 finally {
                     if (recordSuppressedExceptions) {
                         this.suppressedExceptions = null;
                     }
                     afterSingletonCreation(beanName);
                 }
                 //加入缓存
                 addSingleton(beanName, singletonObject);
             }
             return (singletonObject != NULL_OBJECT ? singletonObject : null);
         }
}
```
上述代码中其实是使用了回调方法，使得程序可以在单例创建的前后做一些准备及处理操作，而真正的获取单例bean的方法其实并不是在此方法中实现的，其实现逻辑是在ObjectFactory类型的实例singletonFactory中实现的。而这些准备及处理操作包括如下内容。
- 检查缓存是否已经加载过。
- 若没有加载，则记录beanName的正在加载状态。
- 加载单例前记录加载状态。
可能你会觉得beforeSingletonCreation方法是个空实现，里面没有任何逻辑，但其实不是，这个函数中做了一个很重要的操作：记录加载状态，也就是通过this.singletonsCurrentlyIn- Creation.add(beanName)将当前正要创建的bean记录在缓存中，这样便可以对循环依赖进行检测。
```java
protected void beforeSingletonCreation(String beanName) {
         if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletons   
CurrentlyInCreation.add(beanName)) {
             throw new BeanCurrentlyInCreationException(beanName);
         }
}
```
- 通过调用参数传入的ObjectFactory的个体Object方法实例化bean。
- 加载单例后的处理方法调用。 当bean加载结束后需要移除缓存中对该bean的正在加载状态的记录。
````java
protected void afterSingletonCreation(String beanName) {
         if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletons   
CurrentlyInCreation.remove(beanName)) {
             throw new IllegalStateException("Singleton '" + beanName + "' isn't   
currently in creation");
         }
}
````
- 将结果记录至缓存并删除加载bean过程中所记录的各种辅助状态。
````java
protected void addSingleton(String beanName, Object singletonObject) {
         synchronized (this.singletonObjects) {
             this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));
             this.singletonFactories.remove(beanName);
             this.earlySingletonObjects.remove(beanName);
             this.registeredSingletons.add(beanName);
         }
}
````
- 返回处理结果。 虽然我们已经从外部了解了加载bean的逻辑架构，但现在我们还并没有开始对bean加载功能的探索，之前提到过，bean的加载逻辑其实是在传入的ObjectFactory类型的参数singletonFactory中定义的，我们反推参数的获取，得到如下代码：
```java
sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                     public Object getObject() throws BeansException {
                         try {
                             return createBean(beanName, mbd, args);
                         }
                         catch (BeansException ex) {
                             destroySingleton(beanName);
                             throw ex;
                         }
                     }
                 });
```
ObjectFactory的核心部分其实只是调用了createBean的方法，所以我们还需要到createBean方法中追寻真理。
















































































