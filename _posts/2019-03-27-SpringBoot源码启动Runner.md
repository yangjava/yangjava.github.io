---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码启动Runner
SpringBoot中的Runner接口是用来在Spring应用启动后执行一些初始化逻辑的接口。

## Runner接口概述
Runner是一个函数式接口，只有一个方法run()，用来定义初始化逻辑。

Runner接口的主要作用是能够让开发者在Spring应用启动之后，进行一些自己想要的初始化操作。

这里所说的Runner接口是由SpringBoot提供的，包含 ApplicationRunner 与 CommandLineRunner

从目录和代码层面看，这两个接口互相没有任何从属关系，且从接口内容上看，也只有入参的类型不同：
```java
@FunctionalInterface  
public interface ApplicationRunner {  
    void run(ApplicationArguments args) throws Exception;  
}
```

```java
@FunctionalInterface  
public interface CommandLineRunner {  
    void run(String... args) throws Exception;  
}
```
之所以将这两者放在一起讨论，是因为他们在Spring项目启动时，扮演了相似的功能角色。

使用方法也十分简单：实现 CommandLineRunner 或者 ApplicationRunner 接口，并重写un()方法即可。

## Runner实战
在run()方法中，我们可以通过Autowired注入一些服务或数据源，并进行一些初始化操作，代码如下：
```
@Component
public class DataInitializer implements CommandLineRunner {

    @Autowired
    private SomeService someService;

    @Override
    public void run(String... args) throws Exception {
        // 初始化数据
        myService.initializeData();
    }
}
```

## 二者的区别
从包路径和接口内容两个方面来看，CommandLineRunner 与 ApplicationRunner接口整体上略有相似，但又不是完全相同。

对于二者的区别，我们可以从调用入口、接口入参两个角度来进行理解：

### 调用入口
在Spring启动的run方法中，我们可以看到，在run方法的最后，callRunner方法里面，从Spring上下文中获取到了所有的ApplicationRunner类型和CommandLineRunner类型的Bean。

再将这些类型的Bean放到一个runner集合中，循环遍历并调用各自的run方法。
```
	private void callRunners(ApplicationContext context, ApplicationArguments args) {
		List<Object> runners = new ArrayList<>();
		runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
		runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
		AnnotationAwareOrderComparator.sort(runners);
		for (Object runner : new LinkedHashSet<>(runners)) {
			if (runner instanceof ApplicationRunner) {
				callRunner((ApplicationRunner) runner, args);
			}
			if (runner instanceof CommandLineRunner) {
				callRunner((CommandLineRunner) runner, args);
			}
		}
	}
```
如果是ApplicationRunner的话,则执行如下代码:

```
private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
    try {
        runner.run(args);
    } catch (Exception var4) {
        throw new IllegalStateException("Failed to execute ApplicationRunner", var4);
    }
}
```

如果是CommandLineRunner的话,则执行如下代码:

```
private void callRunner(CommandLineRunner runner, ApplicationArguments args) {
    try {
        runner.run(args.getSourceArgs());
    } catch (Exception var4) {
        throw new IllegalStateException("Failed to execute CommandLineRunner", var4);
    }
}
```
### 接口入参
ApplicationRunner接口 和 CommandLineRunner接口都是函数式接口，只有一个run()方法。

二者的区别在于参数不同。

ApplicationRunner中run()参数是ApplicationArguments对象。
```java
@FunctionalInterface  
public interface ApplicationRunner {  
    void run(ApplicationArguments args) throws Exception;  
}
```
CommandLineRunner中run()参数是数量可变的字符串。

```java
@FunctionalInterface  
public interface CommandLineRunner {  
    void run(String... args) throws Exception;  
}
```
Spring类型的入参我们很好理解，方法可以接收任意数量的字符串参数。

ApplicationArguments类型的入参在这里需要说明一下。

该类型是Spring Boot提供的一个类，用于获取应用程序启动时传入的命令行参数。它包含了以下信息：

- 命令行参数：包括长选项（例如 --foo）和短选项（例如 -f）。
- 非选项参数：即以空格分隔的非选项参数，例如文件名、路径等。
- 选项参数的值：如果一个选项参数需要一个值，例如 --foo=bar，那么这个值将存储在ApplicationArguments中。

使用ApplicationArguments可以轻松地获取和处理命令行参数，从而实现更加灵活的应用程序配置和启动。

## 实战
我们先分别实现一下ApplicationRunner和CommandLineRunner看一下效果

直接创建一个新的类，然后实现ApplicationRunner，实现run方法，run方法里面很简单，就是输出一句话，我们先看看springboot启动以后会不会输出。这里要注意一点，你必须把你的这个类添加到spring容器当中，也就是要加上@Component，不然是不会生效的。
```
@Component
public class MyRunner01 implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("我是ApplicationRunner");
    }
}
```
然后我们再试试CommandLineRunner，什么都不变，就把刚刚的抄一遍就好
```
@Component
public class MyRunner02 implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        System.out.println("我是CommandLineRunner");
    }
}
```
两个一起开启，同时执行
创建了两个类，他们可以同时输出，并且ApplicationRunner在前面，因为上面我刚刚说了，他们addall的顺序就是ApplicationRunner在前面，所以它先执行。

### 如何设置优先级
那么很常见的场景就是我就是想让CommandLineRunner优先输出因为我有很多指令参数需要处理，或者是我相同的ApplicationRunner有很多，我业务场景顺序敏感需要对他们进行链式执行。那我们刚刚也看到启动源码中其实有对它们进行优先级处理的排序。

两个办法，一个是再实现ORdered接口，还有一个是使用Order注解。当然，人生苦短，我选注解。
```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD})
@Documented
public @interface Order {

   /**
    * The order value.
    * <p>Default is {@link Ordered#LOWEST_PRECEDENCE}.
    * @see Ordered#getOrder()
    */
   int value() default Ordered.LOWEST_PRECEDENCE;

}
```
注解有一个属性，就是value，这个value是一个int数字，这个数字决定了优先级顺序，越小就越靠前(默认值是Integer的最大值)。比如我以刚刚两个为例，我把实现CommandLineRunner的类order变成1，实现ApplicationRunner的order为2。

结果就是order更小的优先执行了，所以如果你有多个Runner可以使用这个方式来按照你期望的串行化执行。

## 进阶彩蛋
在实际项目中可能还是会在一个Component下面。 也就是在一个Component下用方法返回一个bean，注入到ioc容器当中，而不是把一个类继承Runner接口然后作为Component注入ioc容器当中，这样的好处就是用一个类统一管理。

当然哈，也不能一棍子打死，我这样处理适用于逻辑比较少的业务。具体代码如下
```
@Component
public class MyRunner03 {
    @Bean
    public ApplicationRunner app01() {
        return e -> System.out.println("我是app01");
    }

    @Bean
    public ApplicationRunner app02() {
        return e -> System.out.println("我是app02");
    }

    @Bean
    public CommandLineRunner cmd01() {
        return e -> System.out.println("我是cmd01");
    }

    @Bean
    public CommandLineRunner cmd02() {
        return e -> System.out.println("我是cmd02");
    }

}
```








