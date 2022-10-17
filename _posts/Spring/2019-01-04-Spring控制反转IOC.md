
> 思考

- Hello 对象是谁创建的 ?  hello 对象是由Spring创建的
- Hello 对象的属性是怎么设置的 ?  hello 对象的属性是由Spring容器设置的

这个过程就叫控制反转 :

- 控制 : 谁来控制对象的创建 , 传统应用程序的对象是由程序本身控制创建的 , 使用Spring后 , 对象是由Spring来创建的
- 反转 : 程序本身不创建对象 , 而变成被动的接收对象 .

## 控制反转IOC

**控制反转IoC(Inversion of Control)，是一种设计思想，DI(依赖注入)是实现IoC的一种方法**，也有人认为DI只是IoC的另一种说法。没有IoC的程序中 , 我们使用面向对象编程 , 对象的创建与对象间的依赖关系完全硬编码在程序中，对象的创建由程序自己控制，控制反转后将对象的创建转移给第三方，个人认为所谓控制反转就是：获得依赖对象的方式反转了。

**控制反转是一种通过描述（XML或注解）并通过第三方去生产或获取特定对象的方式。在Spring中实现控制反转的是IoC容器，其实现方法是依赖注入（Dependency Injection,DI）。**

IOC(DI)：控制反转（依赖注入）

所谓的IOC称之为控制反转，==简单来说就是将对象的创建的权利及对象的生命周期的管理过程交由Spring框架来处理，从此在开发过程中不再需要关注对象的创建和生命周期的管理，而是在需要时由Spring框架提供，这个由spring框架管理对象创建和生命周期的机制称之为控制反转。==而在创建对象的过程中Spring可以依据配置对对象的属性进行设置，这个过称之为依赖注入,也即DI。

### Spring获取Bean的方式

#### 通过id获取

返回结果：

​	找到对应的唯一对象：返回该对象

​	找不到对应的对象：抛出异常：NoSuchBeanDefinitionException

​	不可能找到多个对象：==因为id不能重复，不可能出现此情况==

```java
@Test
public void test(){
    ApplicationContext context = 
        new ClassPathXmlApplicationContext("applicationContext.xml");

    //方式一：通过id获取

    Person p = (Person) context.getBean("person11");
    System.out.println(p);

    ((ClassPathXmlApplicationContext) context).close();
}
```

#### 通过class获取

返回结果：

​	找到对应的唯一对象：返回该对象

​	找不到对应的对象：抛出异常：NoSuchBeanDefinitionException

​	找到多个对象：抛出异常：NoUniqueBeanDefinitionException

特点：

- 如果Spring IOC在通过class获取bean时，找不到该类型的bean时，会去检查是否存在该类型的子孙类型的bean
    - 有：返回该子孙类型的bean
    - 没有/找到多个：抛出异常
- 符合java面向对象思想中的多态的特性



```java
@Test
public void test(){
    ApplicationContext context = 
        new ClassPathXmlApplicationContext("applicationContext.xml");

    //方式二：通过class获取
    Person p1 = context.getBean(Person.class);
    p1.say();
    System.out.println(p1);

    ((ClassPathXmlApplicationContext) context).close();
}
```

#### 通过id+class获取

返回结果：

​	找到对应的唯一对象：返回该对象

​	找不到对应的对象：抛出异常：NoSuchBeanDefinitionException

​	找到多个对象：==id+class能唯一定位一个bean==，不可能出现查到多个bean的情况

```java
@Test
public void test(){
    ApplicationContext context = 
        new ClassPathXmlApplicationContext("applicationContext.xml");

    //方式三：通过id+class获取
    Person p = context.getBean("person1", Person.class);
    System.out.println(p);

    ((ClassPathXmlApplicationContext) context).close();
}
```

