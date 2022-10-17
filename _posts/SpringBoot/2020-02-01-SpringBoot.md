# SpringBoot

**Spring Boot makes it easy to create stand-alone, production-grade Spring based Applications that you can "just run".**

**We take an opinionated view of the Spring platform and third-party libraries so you can get started with minimum fuss. Most Spring Boot applications need minimal Spring configuration.**

官网介绍：

Spring Boot使创建独立的、基于生产级Spring的应用程序变得很容易，您可以“直接运行”这些应用程序。

我们对Spring平台和第三方库有自己的见解，这样您就可以轻松入门了。大多数Spring引导应用程序只需要很少的Spring配置。

## 简介

Spring Boot 是基于 Spring 框架基础上推出的一个全新的框架, 旨在让开发者可以轻松地创建一个可独立运行的，生产级别的应用程序。利用控制反转的核心特性，并通过依赖注入实现控制反转来实现管理对象生命周期容器化，利用面向切面编程进行声明式的事务管理，整合多种持久化技术管理数据访问，提供大量优秀的Web框架方便开发等等。

Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化Spring应用初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。Spring Boot其实就是一个整合很多可插拔的组件（框架），内嵌了使用工具（比如内嵌了Tomcat、Jetty等），方便开发人员快速搭建和开发的一个框架。

### 特征

- 搭建项目快，几秒钟就可以搭建完成；
- 让测试变的简单，内置了JUnit、Spring Boot Test等多种测试框架，方便测试；
- Spring Boot让配置变的简单，Spring Boot的核心理念：约定大约配置，约定了某种命名规范，可以不用配置，就可以完成功能开发，比如模型和表名一致就可以不用配置，直接进行CRUD（增删改查）的操作，只有表名和模型不一致的时候，配置名称即可；
- 内嵌容器，省去了配置Tomcat的繁琐；
- 方便监控，使用Spring Boot Actuator组件提供了应用的系统监控，可以查看应用配置的详细信息；

## 由来

在开始了解Spring Boot之前，我们需要先了解一下Spring，因为Spring Boot的诞生和Spring是息息相关的，Spring Boot是Spring发展到一定程度的一个产物，但并不是Spring的替代品，Spring Boot是为了让程序员更好的使用Spring。说到这里可能有些人会迷糊，那到底Spring和Spring Boot有着什么样的联系呢？

### Spring发展史

在开始之前我们先了解一下Spring，Spring的前身是interface21，这个框架最初是为了解决EJB开发笨重臃肿的问题，为J2EE提供了另一种简单又实用的解决方案，并在2004年3月发布了Spring 1.0正式版之后，就引起了Java界广泛的关注和热评，从此Spring在Java界势如破竹迅速走红，一路成为Java界一颗璀璨夺目的明星，至今无可替代，也一度成为J2EE开发中真正意义上的标准了，而他的创始人Rod Johnson也在之后声名大噪，名利双收，现在是一名优秀的天使投资人，走上了人生的巅峰。

### Spring Boot诞生

那既然Spring已经这么优秀了，为什么还有了之后Spring Boot？

因为随着Spring发展的越来越火，Spring也慢慢从一个小而精的框架变成了，一个覆盖面广大而全的框架，另一方面随着新技术的发展，比如nodejs、golang、Ruby的兴起，让Spring逐渐看着笨重起来，大量繁琐的XML配置和第三方整合配置，让Spring使用者痛苦不已，这个时候急需一个解决方案，来解决这些问题。

就在这个节骨眼上Spring Boot应运而生，2013年Spring Boot开始研发，2014年4月Spring Boot 1.0正式发布，Spring Boot诞生之初就受到业界的广泛关注，很多个人和企业陆续开始尝试，随着Spring Boot 2.0的发布，又一次把Spring Boot推向了公众的视野，也有越来越多了的中大型企业把Spring Boot使用到正式的生产环境了。值得一提的是Spring官方也把Spring Boot作为首要的推广项目，放到了官网的首位。

### Spring Boot版本号说明

那版本号后面的英文代表什么含义呢？

具体含义，如下文所示：

- SNAPSHOT：快照版，表示开发版本，随时可能修改；
- M1（Mn）：M是milestone的缩写，也就是里程碑版本；
- RC1（RCn）：RC是release candidates的缩写，也就是发布预览版；
- Release：正式版，也可能没有任何后缀也表示正式版；

## springBoot核心功能

- 独立运行的spring项目：Spring Boot可以以jar包形式直接运行，如java-jar xxxjar优点是：节省服务器资源
- 内嵌servlet 容器：Spring Boot 可以选择内嵌Tomcat，Jetty，这样我们无须以war包形式部署项目。
- 
- 提供starter 简化Maven 配置：在Spring Boot 项目中为我们提供了很多的spring-boot-starter-xxx的项目（我们把这个依赖可以称之为**起步依赖**），我们导入指定的这些项目的坐标，就会自动导入和该模块相关的依赖包：
      例如我们后期再使用Spring Boot 进行web开发我们就需要导入spring-boot-starter-web这个项目的依赖，导入这个依赖以后！那么Spring Boot就会自动导入web开发所需要的其他的依赖包
- 自动配置 spring:Spring Boot 会根据在类路径中的jar包，类，为jar包里的类自动配置Bean，这样会极大减少我们要使用的配置。
  　　当然Spring Boot只考虑了大部分开发场景，并不是所有的场景，如果在实际的开发中我们需要自动配置Bean，而Spring Boot不能满足，则可以自定义自动配置。
- 准生产的应用监控：Spring Boot 提供基于http，sh，telnet对运行时的项目进行监控
- 无代码生成和xml配置：Spring Boot大量使用spring4.x提供的注解新特性来实现无代码生成和xml 配置。spring4.x提倡使用Java配置和注解配置组合，而Spring Boot不需要任何xml配置即可实现spring的所有配置。

## Hello World示例程序

### Maven依赖

```
<dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>2.3.0.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

```

### 配置文件application.yml

yml文件和properties配置文件具有同样的功能。二者的区别在于：

- yml文件的层级更加清晰直观，但是书写时需要注意格式缩进对齐。yml格式配置文件更有利于表达复杂数据结构的配置。比如：列表，对象（后面章节会详细说明）。
- properties阅读上不如yml直观，好处在于书写时不用特别注意格式缩进对齐。

```yam
server:
  port: 8888   # web应用服务端口
```

引入spring-boot-starter-web依赖

```xml
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

一个hello world测试Controller

```java
@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String hello(String name) {
        return "hello world, " +name;
    }
}
```

项目结构目录整体上符合maven规范要求：

| 目录位置                                  | 功能                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| src/main/java                             | 项目java文件存放位置，初始化包含主程序入口 XxxApplication，可以通过直接运行该类来 启动 Spring Boot应用 |
| src/main/resources                        | 存放静态资源，图片、CSS、JavaScript、web页面模板文件等       |
| src/test                                  | 单元测试代码目录                                             |
| .gitignore                                | git版本管理排除文件                                          |
| target文件夹                              | 项目代码构建打包结果文件存放位置，不需要人为维护             |
| pom.xml                                   | maven项目配置文件                                            |
| application.properties（application.yml） | 用于存放程序的各种依赖模块的配置信息，比如服务端口，数据库连接配置等 |

##### 依赖

```xml
    <!--Hello World项目的父工程是org.springframework.boot-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.1.RELEASE</version>
        <relativePath/>
    </parent>

    <!--
        org.springframework.boot他的父项目是spring-boot-dependencies
        他来真正管理Spring Boot应用里面的所有依赖版本；
        Spring Boot的版本仲裁中心；
        以后我们导入依赖默认是不需要写版本；（没有在dependencies里面管理的依赖自然需要声明版本号）
    -->
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.2.1.RELEASE</version>
    <relativePath>../../spring-boot-dependencies</relativePath>
  </parent>
```

##### 启动器

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

**spring-boot-starter**-`web`：

 `spring-boot-starter`：spring-boot场景启动器；帮我们导入了web模块正常运行所依赖的组件；

Spring Boot将所有的功能场景都抽取出来，做成一个个的starters（启动器），只需要在项目里面引入这些starter相关场景的所有依赖都会导入进来。要用什么功能就导入什么场景的启动器。

##### 主程序类，主入口类

```java
@SpringBootApplication
public class HelloWorldApplication {

    public static void main(String[] args) {
        //启动
        SpringApplication.run(HelloWorldApplication.class, args);
    }

}
```

`@SpringBootApplication`: Spring Boot应用标注在某个类上说明这个类是SpringBoot的主配置类，SpringBoot就应该运行这个类的main方法来启动SpringBoot应用；

看一下`@SpringBootApplication`这个注解类的源码

```java
@Target({ElementType.TYPE})			//可以给一个类型进行注解，比如类、接口、枚举
@Retention(RetentionPolicy.RUNTIME)  //可以保留到程序运行的时候，它会被加载进入到 JVM 中
@Documented						   //将注解中的元素包含到 Javadoc 中去。
@Inherited						   //继承，比如A类上有该注解，B类继承A类，B类就也拥有该注解
@SpringBootConfiguration
@EnableAutoConfiguration			//开启自动配置
/*
*创建一个配置类，在配置类上添加 @ComponentScan 注解。
*该注解默认会扫描该类所在的包下所有的配置类，相当于之前的 <context:component-scan>。
*/
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
```

* `@SpringBootConfiguration`：Spring Boot的配置类；标注在某个类上，表示这是一个Spring Boot的配置类；

  ```java
   @Target({ElementType.TYPE})
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Configuration
   public @interface SpringBootConfiguration
  ```

  * `@Configuration`：配置类上来标注这个注解；

    配置类 ----- 配置文件；配置类也是容器中的一个组件；@Component

    ```java
    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Component
    public @interface Configuration 
    ```

* `@EnableAutoConfiguration`：开启自动配置功能；

  以前我们需要配置的东西，Spring Boot帮我们自动配置；`@**EnableAutoConfiguration**`告诉SpringBoot开启自动配置功能；这样自动配置才能生效；

  ```java
   @Target({ElementType.TYPE})
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Inherited
   @AutoConfigurationPackage
   @Import({AutoConfigurationImportSelector.class})
   public @interface EnableAutoConfiguration
  ```

  * `@AutoConfigurationPackage`：自动配置包

    ```java
    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @Import({Registrar.class})
    public @interface AutoConfigurationPackage
    ```

    * `@Import`：Spring的底层注解@Import，给容器中导入一个组件

      导入的组件由`org.springframework.boot.autoconfigure.AutoConfigurationPackages.Registrar`将主配置类（`@SpringBootApplication`标注的类）的所在包及下面所有子包里面的所有组件扫描到Spring容器；
      这里controller包是在主程序所在的包下，所以会被扫描到，我们在springboot包下创建一个test包，把主程序放在test包下，这样启动就只会去扫描test包下的内容而controller包就不会被扫描到，再访问开始的hello就是404

  * `@Import({AutoConfigurationImportSelector.class})`

    `AutoConfigurationImportSelector.class`将所有需要导入的组件以全类名的方式返回；这些组件就会被添加到容器中；会给容器中导入非常多的自动配置类（xxxAutoConfiguration）；就是给容器中导入这个场景需要的所有组件，并配置好这些组件；

    有了自动配置类，免去了我们手动编写配置注入功能组件等的工作；

Spring Boot在启动的时候从类路径下的META-INF/spring.factories中获取EnableAutoConfiguration指定的值，将这些值作为自动配置类导入到容器中，自动配置类就生效，帮我们进行自动配置工作；以前我们需要自己配置的东西，自动配置类都帮我们完成了；

## SpringBoot配置

#### 默认配置文件

用于配置容器端口名、数据库链接信息、日志级别。pom是项目编程的配置，properties是软件部署的配置。

```
src/main/resources/application.properties
```

##### yaml配置文件实例

```
environments:
    dev:
        url: http://dev.bar.com
        name: Developer Setup
    prod:
        url: http://foo.bar.com
        name: My Cool App
```

##### 等价的properties配置文件

```
environments.dev.url=http://dev.bar.com
environments.dev.name=Developer Setup
environments.prod.url=http://foo.bar.com
environments.prod.name=My Cool App
```

##### yaml的自定义参数

* 定义自定义的参数

```
book.name=SpringCloudInAction
book.author=ZhaiYongchao
```

* 通过占位符的方式加载自定义的参数

```
@Component
public class Book {

    @Value("${book.name}")
    private String name;
    @Value("${book.author}")
    private String author;

    // 省略getter和setter
}
```

* 通过SpEL表达式加载自定义参数

'''
#{...}
'''

##### 使用随机数配置

${random}的配置方式主要有一下几种，读者可作为参考使用。

```
# 随机字符串
com.didispace.blog.value=${random.value}
# 随机int
com.didispace.blog.number=${random.int}
# 随机long
com.didispace.blog.bignumber=${random.long}
# 10以内的随机数
com.didispace.blog.test1=${random.int(10)}
# 10-20的随机数
com.didispace.blog.test2=${random.int[10,20]}
```


##### 通过命令行配置

在启动java应用是，添加配置参数

```
java -jar xxx.jar --server.port=8888
```

##### 默认值

占位符获取之前配置的值，如果没有可以是用“冒号”指定默认值
格式例如，xxxxx.yyyy是属性层级及名称，如果该属性不存在，冒号后面填写默认值

```yaml
${xxxxx.yyyy:默认值}
```

##### 配置文件值注入

使用@Value获取配置值

```
@Data
@Component
public class Family {

    @Value("${family.family-name}")
    private String familyName;

}
```

使用@ConfigurationProperties获取配置值

```
@Data
@Component
@ConfigurationProperties(prefix = "family")
public class Family {

    //@Value("${family.family-name}")
    private String familyName;
    private Father father;
    private Mother mother;
    private Child child;

}
```

比较一下二者

|                          | @ConfigurationProperties | @Value             |
| :----------------------- | :----------------------- | :----------------- |
| 功能                     | 批量注入属性到java类     | 一个个属性指定注入 |
| 松散语法绑定             | 支持                     | 不支持             |
| SpEL                     | 不支持                   | 支持               |
| 复杂数据类型(对象、数组) | 支持                     | 不支持             |
| JSR303数据校验           | 支持                     | 不支持             |

#### 多环境配置

##### 配置方法

对于多环境的配置，各种项目构建工具或是框架的基本思路是一致的，通过配置多份不同环境的配置文件，再通过打包命令指定需要打包的内容之后进行区分打包。

在Spring Boot中多环境配置文件名需要满足application-{profile}.properties的格式，其中{profile}对应你的环境标识，比如：

* application-dev.properties：开发环境
* application-test.properties：测试环境
* application-prod.properties：生产环境

**通过配置`application.yml`**

`application.yml`是默认使用的配置文件，在其中通过`spring.profiles.active`设置使用哪一个配置文件，下面代码表示使用`application-prod.yml`配置，如果`application-prod.yml`和`application.yml`配置了相同的配置，比如都配置了运行端口，那`application-prod.yml`的优先级更高

```
#需要使用的配置文件
spring:
  profiles:
    active: prod
```

**VM options、Program arguments、Active Profile**

VM options设置启动参数 `-Dspring.profiles.active=prod`

Program arguments设置 `--spring.profiles.active=prod`

Active Profile 设置 prod

**命令行方式**

将项目打成jar包，在jar包的目录下打开命令行，使用如下命令启动：

```
java -jar spring-boot-profile.jar --spring.profiles.active=prod
```

#### 配置加载顺序

1. 命令行中传入的参数。
1. SPRING_APPLICATION_JSON中的属性。SPRING_APPLICATION_JSON是以JSON格式配置在系统环境变量中的内容。
1. java:comp/env中的JNDI属性。
1. Java的系统属性，可以通过System.getProperties()获得的内容。
1. 操作系统的环境变量
1. 通过random.*配置的随机属性
1. 位于当前应用jar包之外，针对不同{profile}环境的配置文件内容，例如：application-{profile}.properties或是YAML定义的配置文件
1. 位于当前应用jar包之内，针对不同{profile}环境的配置文件内容，例如：application-{profile}.properties或是YAML定义的配置文件
1. 位于当前应用jar包之外的application.properties和YAML配置内容
1. 位于当前应用jar包之内的application.properties和YAML配置内容
1. 在@Configuration注解修改的类中，通过@PropertySource注解定义的属性
1. 应用默认属性，使用SpringApplication.setDefaultProperties定义的内容1. 

#### 配置文件属性绑定

#### Spring Boot 2.0 新特性

* 移除特殊字符
* 全小写

##### 列表类型

> 必须使用连续下标索引进行配置。

* properties中使用[]在定位列表类型

```
pring.my-example.url[0]=http://example.com
spring.my-example.url[1]=http://spring.io
```

* properties中使用,分割列表类型。

```
pring.my-example.url[0]=http://example.com
spring.my-example.url[1]=http://spring.io
```

* yaml中使用列表

```
spring:
  my-example:
    url:
      - http://example.com
      - http://spring.io

```

* yaml文件中使用,分割列表

```
spring:
  my-example:
    url: http://example.com, http://spring.io
```

##### Map类型

Map类型在properties和yaml中的标准配置方式如下：

* properties格式：

```
spring.my-example.foo=bar
spring.my-example.hello=world
```

* yaml格式：

```
spring:
  my-example:
    foo: bar
    hello: world
```

##### 环境属性绑定

##### 简单类型

在环境变量中通过小写转换与.替换_来映射配置文件中的内容，比如：环境变量SPRING_JPA_DATABASEPLATFORM=mysql的配置会产生与在配置文件中设置spring.jpa.databaseplatform=mysql一样的效果。

##### List类型

由于环境变量中无法使用[和]符号，所以使用_来替代。任何由下划线包围的数字都会被认为是[]的数组形式。

```
MY_FOO_1_ = my.foo[1]
MY_FOO_1_BAR = my.foo[1].bar
MY_FOO_1_2_ = my.foo[1][2]
```

##### 系统属性绑定

##### 简单类型

系统属性与文件配置中的类似，都以移除特殊字符并转化小写后实现绑定

##### list类型

系统属性的绑定也与文件属性的绑定类似，通过[]来标示，比如：

```
-D"spring.my-example.url[0]=http://example.com"
-D"spring.my-example.url[1]=http://spring.io"
```

同样的，他也支持逗号分割的方式，比如：

```
-Dspring.my-example.url=http://example.com,http://spring.io
```

##### 属性读取

在Spring应用程序的environment中读取属性的时候，每个属性的唯一名称符合如下规则：

* 通过.分离各个元素
* 最后一个.将前缀与属性名称分开
* 必须是字母（a-z）和数字(0-9)
* 必须是小写字母
* 用连字符-来分隔单词
* 唯一允许的其他字符是[和]，用于List的索引
* 不能以数字开头

```
this.environment.containsProperty("spring.jpa.database-platform")
```

##### 新的绑定API

简单类型

假设在propertes配置中有这样一个配置：

```
com.didispace.foo=bar
```

我们为它创建对应的配置类：

```
@Data
@ConfigurationProperties(prefix = "com.didispace")
public class FooProperties {

    private String foo;

}
```

接下来，通过最新的Binder就可以这样来拿配置信息了：

```
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(Application.class, args);

        Binder binder = Binder.get(context.getEnvironment());

        // 绑定简单配置
        FooProperties foo = binder.bind("com.didispace", Bindable.of(FooProperties.class)).get();
        System.out.println(foo.getFoo());
    }
}
```



    @RestController
    @RequestMapping(value = "/users")     // 通过这里配置使下面的映射都在/users下
    public class UserController {
    // 创建线程安全的Map，模拟users信息的存储
    static Map<Long, User> users = Collections.synchronizedMap(new HashMap<Long, User>());
    
    /**
     * 处理"/users/"的GET请求，用来获取用户列表
     *
     * @return
     */
    @GetMapping("/")
    public List<User> getUserList() {
        // 还可以通过@RequestParam从页面中传递参数来进行查询条件或者翻页信息的传递
        List<User> r = new ArrayList<User>(users.values());
        return r;
    }
    
    /**
     * 处理"/users/"的POST请求，用来创建User
     *
     * @param user
     * @return
     */
    @PostMapping("/")
    public String postUser(@RequestBody User user) {
        // @RequestBody注解用来绑定通过http请求中application/json类型上传的数据
        users.put(user.getId(), user);
        return "success";
    }
    
    /**
     * 处理"/users/{id}"的GET请求，用来获取url中id值的User信息
     *
     * @param id
     * @return
     */
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // url中的id可通过@PathVariable绑定到函数的参数中
        return users.get(id);
    }
    
    /**
     * 处理"/users/{id}"的PUT请求，用来更新User信息
     *
     * @param id
     * @param user
     * @return
     */
    @PutMapping("/{id}")
    public String putUser(@PathVariable Long id, @RequestBody User user) {
        User u = users.get(id);
        u.setName(user.getName());
        u.setAge(user.getAge());
        users.put(id, u);
        return "success";
    }
    
    /**
     * 处理"/users/{id}"的DELETE请求，用来删除User
     *
     * @param id
     * @return
     */
    @DeleteMapping("/{id}")
    public String deleteUser(@PathVariable Long id) {
        users.remove(id);
        return "success";
    }
    }

#### 加载外部配置文件

如何加载外部自定义配置文件

##### 使用`@PropertySource`加载自定义`yml`或`properties`文件

因为`@PropertySource`默认不支持读取YAML格式外部配置文件，所以我们继承`DefaultPropertySourceFactory` ，然后对它的`createPropertySource`进行一下修改

```
public class MixPropertySourceFactory extends DefaultPropertySourceFactory {

  @Override
  public PropertySource<?> createPropertySource(String name, EncodedResource resource) throws IOException {
    String sourceName = name != null ? name : resource.getResource().getFilename();
    if (!resource.getResource().exists()) {
      return new PropertiesPropertySource(sourceName, new Properties());
    } else if (sourceName.endsWith(".yml") || sourceName.endsWith(".yaml")) {
      Properties propertiesFromYaml = loadYml(resource);
      return new PropertiesPropertySource(sourceName, propertiesFromYaml);
    } else {
      return super.createPropertySource(name, resource);
    }
  }

  private Properties loadYml(EncodedResource resource) throws IOException {
    YamlPropertiesFactoryBean factory = new YamlPropertiesFactoryBean();
    factory.setResources(resource.getResource());
    factory.afterPropertiesSet();
    return factory.getObject();
  }
}
```

然后基于上一节的代码，在Family类的上面加上如下注解即可。

```java
@PropertySource(value = {"classpath:family.yml"}, factory = MixPropertySourceFactory.class)
public class Family {
```

如果是读取`properties`配置文件，加`@PropertySource(value = {"classpath:family.properties"})`即可。不需要定义`MixPropertySourceFactory`。

##### 使用`@ImportResource`加载Spring的xml配置文件

在没有注解的时代，spring的相关配置都是通过xml来完成的，如：beans.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="testBeanService" class="club.krislin.bootlaunch.service.TestBeanService"></bean>
</beans>
```

在启动类上加

```
@ImportResource(locations = {"classpath:beans.xml"})
```

#### SpEL结合@Value注解读取配置文件属性

创建一个配置文件employee.properties,内容如下：

```properties
employee.names=张三,李四,王五,赵六
employee.type=联络员,全勤,兼职
employee.age={one:'27', two : '35', three : '34', four: '26'}
```

- 上文中names和type属性分别代表雇员employee的名字和分类,是字符串类型属性
- age属性代表雇员的年龄，是一组键值对、类对象数据结构

创建一个配置类 `EmployeeConfig` ，代码如下:

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

import org.springframework.context.annotation.PropertySource;
@Configuration
@PropertySource (name = "employeeProperties", value = "classpath:employee.properties")
@ConfigurationProperties
public class EmployeeConfig {

    //使用SpEL读取employee.properties配置文件
    @Value ("#{'${employee.names}'.split(',')}")
    private List<String> employeeNames;
 
    ......
}
```

@Value注解读取配置文件*employee.properties*的相关属性，@Value注解可以将属性值注入到由Spring 管理的Bean中。

```java
@Value ("#{'${employee.names}'.split(',')}")
private List<String> employeeNames;
```

上面的例子中，我们使用SpEL表达式读取了employee.names属性，并将其从字符串属性，以逗号为分隔符转换为List类型。属性值注入完成之后，employeeNames=[张三,李四,王五,赵六]

假如我们需要获取第一位（数组下标从0开始）雇员的姓名，可以使用如下的SpEL表达式：

```java
@Value ("#{'${employee.names}'.split(',')[0]}")
 private String firstEmployeeName;
```

属性值注入完成之后，firstEmployeeName=‘’张三‘’

我们还可以使用@Value注解将键值对、类对象的数据结构转换为java的Map数据类型

```java
@Value ("#{${employee.age}}")
 private Map<String, Integer> employeeAge;
```

加入我们需要根据Map的Key获取Value属性，可以使用如下的SpEL表达式：

```java
@Value ("#{${employee.age}.two}")
private String employeeAgeTwo;
```

如果我们不确定，Map中的某个key是否存在，可以使用如下的SpEL表达式。如果key存在就获取对应的value，如果不存在就获得默认值31

```java
@Value ("#{${employee.age}['five'] ?: 31}") 
private Integer ageWithDefaultValue;
```

## SpringBoot容器

两种方法配置调整SpringBoot应用容器的参数

- 修改配置文件
- 自定义配置类

使用配置文件定制修改相关配置

在application.properties / application.yml配置所需要的属性
server.xx开头的是所有servlet容器通用的配置，server.tomcat.xx开头的是tomcat特有的参数，其它类似。

关于修改配置文件application.properties。
[SpringBoot项目详细的配置文件修改文档](https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html#common-application-properties)

其中比较重要的有：

| 参数                                | 说明                                                       |
| :---------------------------------- | :--------------------------------------------------------- |
| server.connection-timeout=          | 连接的超时时间                                             |
| server.tomcat.max-connections=10000 | 接受的最大请求连接数                                       |
| server.tomcat.accept-count=100      | 当所求的线程处于工作中，被放入请求队列等待的最大的请求数量 |
| server.tomcat.max-threads=200       | 最大的工作线程池数量                                       |
| server.tomcat.min-spare-threads=10  | 最小的工作线程池数量                                       |

SpringBoot2.x定制和修改Servlet容器的相关配置，使用配置类

步骤：
1.建立一个配置类，加上@Configuration注解
2.添加定制器ConfigurableServletWebServerFactory
3.将定制器返回

```java
@Configuration
public class TomcatCustomizer {

    @Bean
    public ConfigurableServletWebServerFactory configurableServletWebServerFactory(){
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.setPort(8585);
        return factory;
    }
}
```

pom配置

SpringBoot默认是使用tomcat作为默认的应用容器。如果需要把tomcat替换为jetty或者undertow，需要先把tomcat相关的jar包排除出去。如下代码所示

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

如果使用Jetty容器，那么添加

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

如果使用Undertow容器，那么添加

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency
```

切换Servlet容器

以切换到undertow为例，修改一下容器配置

```yaml
server:
  port: 8888
  # 下面是配置undertow作为服务器的参数
  undertow:
    # 设置IO线程数, 它主要执行非阻塞的任务,它们会负责多个连接, 默认设置每个CPU核心一个线程
    io-threads: 4
    # 工作任务线程池，默认为io-threads的8倍
    worker-threads: 32
```

打war包部署到外置tomcat容器

修改打包方式

```xml
<packaging>war</packaging>
```

将上面的代码加入到pom.xml文件刚开始的位置，如下：

排除内置tomcat的依赖

我们使用外置的tomcat，自然要将内置的嵌入式tomcat的相关jar排除。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

新增加一个类继承SpringBootServletInitializer实现configure：

为什么继承该类，SpringBootServletInitializer源码注释：
Note that a WebApplicationInitializer is only needed if you are building a war file and deploying it.
If you prefer to run an embedded web server then you won't need this at all.
注意，如果您正在构建WAR文件并部署它，则需要WebApplicationInitializer。如果你喜欢运行一个嵌入式Web服务器，那么你根本不需要这个。

```java
public class ServletInitializer extends SpringBootServletInitializer { 
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        //此处的Application.class为带有@SpringBootApplication注解的启动类
        return builder.sources(BootLaunchApplication.class);
    } 
}
```

> 注意事项：
> 使用外部Tomcat部署访问的时候，application.properties(或者application.yml)中的如下配置将失效，请使用外置的tomcat的端口，tomcat的webapps下项目名进行访问。

```yaml
server.port=
server.servlet.context-path=
```

build要有finalName标签

pom.xml中的构建build代码段，要有应用最终构建打包的名称。

```xml
    <finalName>boot-launch</finalName>
```

打包与运行

war方式打包，打包结果将存储在项目的target目录下面。然后将war包部署到外置Tomcat上面：

```
mvn clean package -Dmaven.test.skip=true
```

在外置tomcat中运行：${Tomcat_home}/bin/目录下执行startup.bat(windows)[或者startup.sh](http://xn--startup-gf7nh96s.sh/)(linux)，然后通过浏览器访问应用，测试效果。

## Jasypt整合SpringBoot

> [Jasypt](http://jasypt.org/)是一个Java库，允许开发人员以很简单的方式添加基本加密功能，而无需深入研究加密原理。利用它可以实现高安全性的，基于标准的加密技术，无论是单向和双向加密。加密密码，文本，数字，二进制文件。

1. 高安全性的，基于标准的加密技术，无论是单向和双向加密。加密密码，文本，数字，二进制文件…
2. 集成Hibernate的。
3. 可集成到Spring应用程序中，与Spring Security集成。
4. 集成的能力，用于加密的应用程序（即数据源）的配置。
5. 特定功能的高性能加密的multi-processor/multi-core系统。
6. 与任何JCE（Java Cryptography Extension）提供者使用开放的API

为了方便，简单编写了一个bat脚本方便使用。

```bat
@echo off
set/p input=待加密的明文字符串：
set/p password=加密密钥(盐值)：
echo 加密中......
java -cp jasypt-1.9.2.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI  ^
input=%input% password=%password% ^
algorithm=PBEWithMD5AndDES
pause
```

- 使用 `jasypt-1.9.2.jar`中的`org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI`类进行加密
- input参数是待加密的字符串，password参数是加密的密钥(盐值)
- 使用PBEWithMD5AndDES算法进行加密

**注意：`jasypt-1.9.2.jar` 文件需要和bat脚本放在相同目录下。

**注意：相同的盐值(密钥)，每次加密的结果是不同的。**

首先引入`Jasypt`的maven坐标

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>1.18</version>
</dependency>
```

在`properties`或`yml`文件中需要对明文进行加密的地方的地方，使用ENC()包裹，如原值："happy family"，加密后使用`ENC(密文)`替换。

为了方便测试，在`properties`或`yml`文件中，做如下配置

```yaml
# 设置盐值（加密解密密钥），我们配置在这里只是为了测试方便
# 生产环境中，切记不要这样直接进行设置，可通过环境变量、命令行等形式进行设置。下文会讲
jasypt:
  encryptor:
    password: 123456
```

简单来说，就是在需要加密的值使用`ENC(`和`)`进行包裹，即：`ENC(密文)`。之后像往常一样使用`@Value("${}")`获取该配置即可，获取的是解密之后的值。

## 整合jdbc

maven依赖包

```xml
<!-- spring JDBC -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<!-- MySQL驱动 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

配置文件application.yml，增加数据库连接、用户名、密码相关的配置。driver-class-name请根据自己使用的数据库和数据库版本准确填写。

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb?useUnicode=true&characterEncoding=utf-8
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
```

测试spring jdbc集成

spring jdbc集成完毕之后，我们来写代码做一个基本的测试。首先我们新建一张测试表article

```sql
CREATE TABLE `article` (
	`id` INT(11) NOT NULL AUTO_INCREMENT,
	`author` VARCHAR(32) NOT NULL,
	`title` VARCHAR(32) NOT NULL,
	`content` VARCHAR(512) NOT NULL,
	`create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
	PRIMARY KEY (`id`)
)
COMMENT='文章'
ENGINE=InnoDB;
```

DAO层代码:

- jdbcTemplate.update适合于insert 、update和delete操作；
- jdbcTemplate.queryForObject用于查询单条记录返回结果
- jdbcTemplate.query用于查询结果列表
- BeanPropertyRowMapper可以将数据库字段的值向对象映射，满足驼峰标识也可以自动映射。如:数据库create_time字段映射到createTime属性。

```java
@Repository
public class ArticleJDBCDAO {

    @Resource
    private JdbcTemplate jdbcTemplate;

    //保存文章
    public void save(Article article) {
        //jdbcTemplate.update适合于insert 、update和delete操作；
        jdbcTemplate.update("INSERT INTO article(author, title,content,create_time) values(?, ?, ?, ?)",
                article.getAuthor(),
                article.getTitle(),
                article.getContent(),
                article.getCreateTime());

    }

    //删除文章
    public void deleteById(Long id) {
        //jdbcTemplate.update适合于insert 、update和delete操作；
        jdbcTemplate.update("DELETE FROM article WHERE id = ?",new Object[]{id});

    }

    //更新文章
    public void updateById(Article article) {
        //jdbcTemplate.update适合于insert 、update和delete操作；
        jdbcTemplate.update("UPDATE article SET author = ?, title = ? ,content = ?,create_time = ? WHERE id = ?",
                article.getAuthor(),
                article.getTitle(),
                article.getContent(),
                article.getCreateTime(),
                article.getId());

    }

    //根据id查找文章
    public Article findById(Long id) {
        //queryForObject用于查询单条记录返回结果
        return (Article) jdbcTemplate.queryForObject("SELECT * FROM article WHERE id=?", new Object[]{id}, new BeanPropertyRowMapper(Article.class));
    }

    //查询所有
    public List<Article> findAll(){
        //query用于查询结果列表
        return (List<Article>) jdbcTemplate.query("SELECT * FROM article ",  new BeanPropertyRowMapper(Article.class));
    }


}
```

service层接口:

```java
public interface ArticleRestService {

    public Article saveArticle(Article article);

    public void deleteArticle(Long id);

    public void updateArticle(Article article);

    public Article getArticle(Long id);

    public List<Article> getAll();
}
```

service层接口实现

```java
@Slf4j
@Service
public class ArticleRestJDBCService implements ArticleRestService{

    @Resource
    private
    ArticleJDBCDAO articleJDBCDAO;

    @Transactional
    public Article saveArticle( Article article) {
        articleJDBCDAO.save(article);
        //int a = 2/0；  //人为制造一个异常，用于测试事务
        return article;
    }

    public void deleteArticle(Long id){
        articleJDBCDAO.deleteById(id);
    }

    public void updateArticle(Article article){
        articleJDBCDAO.updateById(article);
    }

    public Article getArticle(Long id){
        return articleJDBCDAO.findById(id);
    }

    public List<Article> getAll(){
        return articleJDBCDAO.findAll();
    }
}
```

重点测试一下事务的回滚。在saveArticle方法上使用了@Trasactional注解，该注解基本功能为事务管理，保证saveArticle方法一旦有异常，所有的数据库操作就回滚。

## Spring JDBC 多数据源整合Spring boot

配置多个数据源

application.yml配置2个数据源，第一个叫做primary，第二个叫做secondary。注意两个数据源连接的是不同的库，testdb和testdb2.

```java
spring:
  datasource:
    primary:
      jdbc-url: jdbc:mysql://localhost:3306/testdb?useUnicode=true&characterEncoding=utf-8
      username: root
      password: 123456
      driver-class-name: com.mysql.jdbc.Driver
    secondary:
      jdbc-url: jdbc:mysql://localhost:3306/testdb2?useUnicode=true&characterEncoding=utf-8
      username: root
      password: 123456
      driver-class-name: com.mysql.jdbc.Driver
```

通过Java Config将数据源注入到Spring上下文。

- primaryJdbcTemplate使用primaryDataSource数据源操作数据库testdb。
- secondaryJdbcTemplate使用secondaryDataSource数据源操作数据库testdb2。

```java
@Configuration
public class DataSourceConfig {
    @Primary
    @Bean(name = "primaryDataSource")
    @Qualifier("primaryDataSource")
    @ConfigurationProperties(prefix="spring.datasource.primary")
    public DataSource primaryDataSource() {
            return DataSourceBuilder.create().build();
    }

    @Bean(name = "secondaryDataSource")
    @Qualifier("secondaryDataSource")
    @ConfigurationProperties(prefix="spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name="primaryJdbcTemplate")
    public JdbcTemplate primaryJdbcTemplate (
        @Qualifier("primaryDataSource") DataSource dataSource ) {
        return new JdbcTemplate(dataSource);
    }

    @Bean(name="secondaryJdbcTemplate")
    public JdbcTemplate secondaryJdbcTemplate(
            @Qualifier("secondaryDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

ArticleJDBCDAO改造

- 在上一节的代码的基础上改造
- 注入primaryJdbcTemplate作为默认的数据库操作对象。
- 将jdbcTemplate作为参数传入ArticleJDBCDAO的方法，不同的template操作不同的库。

```java
@Repository
public class ArticleJDBCDAO {

    @Resource
    private JdbcTemplate primaryJdbcTemplate;


    //以保存文章为例，其他的照做
    public void save(Article article,JdbcTemplate jdbcTemplate ) {
        if(jdbcTemplate == null){
            jdbcTemplate= primaryJdbcTemplate;
        }

        //jdbcTemplate.update适合于insert 、update和delete操作；
        jdbcTemplate.update("INSERT INTO article(author, title,content,create_time) values(?, ?, ?, ?)",
                article.getAuthor(),
                article.getTitle(),
                article.getContent(),
                article.getCreateTime());
    
    }


}
```

测试同时向两个数据库保存数据

在src/test/java目录下，加入如下单元测试类，并进行测试。正常情况下，在testdb和testdb2数据库的article表，将分别插入一条数据。

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringJdbcTest {

    @Resource
    private ArticleJDBCDAO articleJDBCDAO;
    @Resource
    private JdbcTemplate primaryJdbcTemplate;
    @Resource
    private JdbcTemplate secondaryJdbcTemplate;


    @Test
    public void testJdbc() {
        articleJDBCDAO.save(
                Article.builder()
                .author("zimug").title("primaryJdbcTemplate").content("ceshi").createTime(new Date())
                .build(),
                primaryJdbcTemplate);
        articleJDBCDAO.save(
                Article.builder()
               .author("zimug").title("secondaryJdbcTemplate").content("ceshi").createTime(new Date())
               .build(),
                secondaryJdbcTemplate);
    }


}
```

## Mybatis整合SpringBoot

整合Mybatis

第一步：引入maven依赖包，包括mybatis相关依赖包和mysql驱动包。

```xml
      <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <!-- alibaba的druid数据库连接池 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.20</version>
        </dependency>
```

第二步：保证application.yml里面有数据库连接的配置。并配置mybatis的xml文件存放位置，下文配置的xml文件目录位置是resources/mapper。

```yaml
server:
  port: 8888

spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
  datasource:
    url: jdbc:mysql://localhost:3306/testdb?useUnicode=true&characterEncoding=utf-8&serverTimezone=UTC
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
    # 使用druid数据源
    type: com.alibaba.druid.pool.DruidDataSource
    filters: stat
    maxActive: 20
    initialSize: 1
    maxWait: 60000
    minIdle: 1
    timeBetweenEvictionRunsMillis: 60000
    minEvictableIdleTimeMillis: 300000
    validationQuery: select 'x'
    testWhileIdle: true
    testOnBorrow: false
    testOnReturn: false
    poolPreparedStatements: true
    maxOpenPreparedStatements: 20

## 该配置节点为独立的节点，有很多同学容易将这个配置放在spring的节点下，导致配置无法被识别
mybatis:
  mapper-locations: classpath:mapper/*.xml  #注意：一定要对应mapper映射xml文件的所在路径
  type-aliases-package: club.krislin.entity  # 注意：对应实体类的路径
```

第三步：配置Mybatis的Mapper(dao)的扫描路径

```java
@SpringBootApplication
@MapperScan(basePackages = {"club.krislin.dao"})
public class BootLaunchApplication {

    public static void main(String[] args) {
        SpringApplication.run(BootLaunchApplication.class, args);
    }

}
```

安装mybatis generator插件

这个插件将帮助我们根据数据库表结构生成Mybatis操作接口及实体类定义等内容。能极大的方便我们开发，减少手写代码量。

增删改查实现代码

Service层接口

```java
public interface ArticleRestService {

     ArticleVO saveArticle(ArticleVO article);

     void deleteArticle(Long id);

     void updateArticle(ArticleVO article);

     ArticleVO getArticle(Long id);

     List<ArticleVO> getAll();public interface IArticleService {

    /**
     * 通过ID查询单条数据
     *
     * @param id 主键
     * @return 实例对象
     */
    Article queryById(Integer id);

    /**
     * 查询多条数据
     *
     * @param offset 查询起始位置
     * @param limit 查询条数
     * @return 对象列表
     */
    List<Article> queryAllByLimit(int offset, int limit);

    /**
     * 新增数据
     *
     * @param article 实例对象
     * @return 实例对象
     */
    Article insert(Article article);

    /**
     * 修改数据
     *
     * @param article 实例对象
     * @return 实例对象
     */
    Article update(Article article);

    /**
     * 通过主键删除数据
     *
     * @param id 主键
     * @return 是否成功
     */
    boolean deleteById(Integer id);
}
```

Service接口实现

```java
@Service("articleService")
public class ArticleServiceImpl implements IArticleService {
    @Resource
    private ArticleDao articleDao;

    /**
     * 通过ID查询单条数据
     *
     * @param id 主键
     * @return 实例对象
     */
    @Override
    public Article queryById(Integer id) {
        return this.articleDao.queryById(id);
    }

    /**
     * 查询多条数据
     *
     * @param offset 查询起始位置
     * @param limit 查询条数
     * @return 对象列表
     */
    @Override
    public List<Article> queryAllByLimit(int offset, int limit) {
        return this.articleDao.queryAllByLimit(offset, limit);
    }

    /**
     * 新增数据
     *
     * @param article 实例对象
     * @return 实例对象
     */
    @Override
    public Article insert(Article article) {
        this.articleDao.insert(article);
        return article;
    }

    /**
     * 修改数据
     *
     * @param article 实例对象
     * @return 实例对象
     */
    @Override
    public Article update(Article article) {
        this.articleDao.update(article);
        return this.queryById(article.getId());
    }

    /**
     * 通过主键删除数据
     *
     * @param id 主键
     * @return 是否成功
     */
    @Override
    public boolean deleteById(Integer id) {
        return this.articleDao.deleteById(id) > 0;
    }
}
```

测试

ArticleTest测试类

```java
@SpringBootTest
public class ArticleTest {
    @Autowired
    IArticleService articleService;

    @Test
    public void test(){
        Article article = articleService.queryById(1);
        System.out.println(article);
    }
}
```

## Mybatis多数据源整合SpringBoot

修改application.yml为双数据源

在application.yml配置双数据源，第一个数据源访问testdb库，第二个数据源访问testdb2库

```java
spring:
  datasource:
    primary:
      url: jdbc:mysql://localhost:3306/testdb?useUnicode=true&characterEncoding=utf-8&serverTimezone=UTC
      username: root
      password: 123456
      driver-class-name: com.mysql.cj.jdbc.Driver
    secondary:
      url: jdbc:mysql://localhost:3306/testdb2?useUnicode=true&characterEncoding=utf-8&serverTimezone=UTC
      username: root
      password: 123456
      driver-class-name: com.mysql.cj.jdbc.Driver
```

主数据源配置

去掉SpringBoot程序主入口上的@MapperScan注解，将注解移到下面的MyBatis专用配置类上方。 DataSource数据源、SqlSessionFactory、TransactionManager事务管理器、SqlSessionTemplate依据不同的数据源分别配置。第一组是primary，第二组是secondary。

```java
@Configuration
@MapperScan(basePackages = "club.krislin.testdb",    //数据源primary-testdb库接口存放目录
        sqlSessionTemplateRef = "primarySqlSessionTemplate")
public class PrimaryDataSourceConfig {

    @Bean(name = "primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary")   //数据源primary配置
    @Primary
    public DataSource testDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "primarySqlSessionFactory")
    @Primary
    public SqlSessionFactory testSqlSessionFactory(
                        @Qualifier("primaryDataSource") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
         //设置XML文件存放位置，如果参考上一篇Mybatis最佳实践，将xml和java放在同一目录下，这里不用配置
        //bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mybatis/mapper/test1/*.xml"));
        return bean.getObject();
    }

    @Bean(name = "primaryTransactionManager")
    @Primary
    public DataSourceTransactionManager testTransactionManager(
                        @Qualifier("primaryDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "primarySqlSessionTemplate")
    @Primary
    public SqlSessionTemplate testSqlSessionTemplate(
                        @Qualifier("primarySqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

}
```

第二个数据源配置

参照primary配置，书写第二份secondary数据源配置的代码。

```java
@Configuration
@MapperScan(basePackages = "club.krislin.testdb2",     //注意这里testdb2目录
        sqlSessionTemplateRef = "secondarySqlSessionTemplate")
public class SecondaryDataSourceConfig {

    @Bean(name = "secondaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.secondary")    //注意这里secondary配置
    public DataSource testDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "secondarySqlSessionFactory")
    public SqlSessionFactory testSqlSessionFactory(
                        @Qualifier("secondaryDataSource") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        //bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mybatis/mapper/test1/*.xml"));
        return bean.getObject();
    }

    @Bean(name = "secondaryTransactionManager")
    public DataSourceTransactionManager testTransactionManager(
                        @Qualifier("secondaryDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "secondarySqlSessionTemplate")
    public SqlSessionTemplate testSqlSessionTemplate(
                        @Qualifier("secondarySqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

}
```

测试用例

将自动生成的代码（自己写Mapper和实体类也可以），分别存放于testdb和testdb2两个文件夹

测试代码：

```java
@SpringBootTest
public class DoubleDataSourcesTest {
    @Autowired
    ArticleDao articleDao;

    @Autowired
    MessageDao messageDao;

    @Test
    public void ArticleTest(){
        Article article = Article.builder()
                .author("james")
                .content("如何打篮球")
                .title("篮球")
                .createTime(new Date())
                .build();

        articleDao.insert(article);

        List<Article> articles = articleDao.queryAllByLimit(0, 5);
        for (Article article1:articles){
            System.out.println(article1);
        }
    }

    @Test
    public void MessageTest(){
        Message message = Message.builder()
                .author("kobe")
                .content("凌晨四点半的洛杉矶")
                .title("奋斗")
                .createTime(new Date())
                .build();
        messageDao.insert(message);

        List<Message> messages = messageDao.queryAllByLimit(0, 5);
        for (Message message1:messages){
            System.out.println(message1);
        }
    }
}

```

## Mongodb整合SpringBoot

mongodb简介

`MongoDB`（来自于英文单词“Humongous”，中文含义为“庞大”）是可以应用于各种规模的企业、各个行业以及各类应用程序的开源数据库。作为一个适用于敏捷开发的数据库，`MongoDB`的数据模式可以随着应用程序的发展而灵活地更新。与此同时，它也为开发人员 提供了传统数据库的功能：二级索引，完整的查询系统以及严格一致性等等。 `MongoDB`能够使企业更加具有敏捷性和可扩展性，各种规模的企业都可以通过使用MongoDB来创建新的应用，提高与客户之间的工作效率，加快产品上市时间，以及降低企业成本。

**`MongoDB`是专为可扩展性，高性能和高可用性而设计的数据库。** 它可以从单服务器部署扩展到大型、复杂的多数据中心架构。利用内存计算的优势，`MongoDB`能够提供高性能的数据读写操作。 `MongoDB`的本地复制和自动故障转移功能使您的应用程序具有企业级的可靠性和操作灵活性。

简单来说，`MongoDB`是一个基于**分布式文件存储的数据库**，它是一个介于关系数据库和非关系数据库之间的产品，其主要目标是在键/值存储方式（提供了高性能和高度伸缩性）和传统的RDBMS系统（具有丰富的功能）之间架起一座桥梁，它集两者的优势于一身。

`MongoDB`支持的数据结构非常松散，是类似`json`的**bson格式**，因此可以存储比较复杂的数据类型，也因为他的存储格式也使得它所存储的数据在Nodejs程序应用中使用非常流畅。

**传统的关系数据库一般由数据库（database）、表（table）、记录（record）三个层次概念组成，MongoDB是由数据库（database）、集合（collection）、文档对象（document）三个层次组成。MongoDB对于关系型数据库里的表，但是集合中没有列、行和关系概念，这体现了模式自由的特点。**

`MongoDB`中的**一条记录就是一个文档**，是一个数据结构，由字段和值对组成。MongoDB文档与JSON对象类似。字段的值有可能包括其它文档、数组以及文档数组。MongoDB支持OS X、Linux及Windows等操作系统，并提供了Python，PHP，Ruby，Java及C++语言的驱动程序，社区中也提供了对Erlang及.NET等平台的驱动程序。

集成spring data mongodb

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

application.yml配置连接

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://root:123456@localhost:27017/testdb
```

注意，这里填写格式

```
    单机模式：mongodb://name:pwd@ip:port/database
    集群模式：mongodb://name:pwd@ip1:port1,ip2:port2/database
```

在项目入口启动类上面加一个注解。开启Mongodb审计功能.

```java
@EnableMongoAuditing
```

实现CURD

创建实体

```java
@Document(collection="article")//集合名
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class Article  implements Serializable {

    private static final long serialVersionUID = -8985545025018238754L;

    @Id
    private String id;

    @Indexed
    private String author;
    private String title;
    @Field("msgContent")
    private String content;

    @CreatedDate
    private Date createTime;
    private List<Reader> reader;


}
```

这里注意：

1. 一定要实现Serializable 接口，否则在序列化的时候会报错。
2. @Document(collection=“article”) 表示：操作的集合为:`article`。
3. 另外，针对`@CreatedDate`注解，也和之前的`jpa`用法一样，创建时会自动赋值，需要在启动类中添加`@EnableMongoAuditing`注解使其生效！
4. 可使用`@Field`注解，可指定存储的键值名称，默认就是类字段名。如设置`@Field("msgContent")`后
5. `@Id`主键，不可重复，自带索引，可以在定义的列名上标注，需要自己生成并维护不重复的约束。如果自己不设置@Id主键，mongo会自动生成一个唯一主键，并且插入时效率远高于自己设置主键。
6. `@Indexed`声明该字段需要加索引，加索引后以该字段为条件检索将大大提高速度。
   唯一索引的话是@Indexed(unique = true)。

数据库操作的Dao

```java
public interface ArticleDao extends MongoRepository<Article,String> {
        //支持关键字查询，和JPA的用法一样
        Article findByAuthor(String author);
}
```

还可以使用使用mongoTemplate来完成数据操作，如:

```java
        @Autowired
        MongoTemplate mongoTemplate;

        //使用 save和insert都可以进行插入
        //区别：当存在"_id"时
        //insert 插入已经存在的id时 会异常
        //save 则会进行更新
        //简单来说 save 就是不存在插入 存在更新
        mongoTemplate.insert(msg);
        mongoTemplate.save(msg);


        //查找 article根据Criteria 改造查询条件
        Query query = new Query(Criteria.where("author").is("zimug"));
        Article article = mongoTemplate.find(query, Article.class);
   
```

更多用法参考:https://docs.spring.io/spring-data/mongodb/docs/2.1.9.RELEASE/reference/html/

## Spring data rest整合SpringBoot

介绍spring data rest

Spring Data REST是基于Spring Data的repository之上，可以把 repository **自动**输出为REST资源，目前支持Spring Data JPA、Spring Data MongoDB、Spring Data Neo4j、Spring Data GemFire、Spring Data Cassandra的 repository **自动**转换成REST服务。注意是**自动**。

实现rest接口的最快方式

```xml
<dependency> 
    <groupId>org.springframework.boot</groupId> 
    <artifactId>spring-boot-starter-data-rest</artifactId> 
</dependency>
```

```java
@RepositoryRestResource(collectionResourceRel = "article",path="articles")
public interface ArticleDao extends MongoRepository<Article, String> {

}
```

参数：

1. collectionResourceRel 对应的是mongodb数据库资源文档的名称
2. path是Rest接口资源的基础路径
   如：GET /articles/{id}
   DELETE /articles/{id}

就简单的这样一个实现，Spring Data Rest就可以基于article资源，生成一套GET、PUT、POST、DELETE的增删改查的REST接口。

## Actuator整合SpringBoot

Spring Boot Actuator简介

Spring Boot作为构建微服务节点的方案，一定要提供全面而且细致的监控指标，使微服务更易于管理！微服务不同于单体应用，微服务的每个服务节点都单独部署，独立运行，大型的微服务项目甚至有成百上千个服务节点。这就为我们进行系统监控与运维提出了挑战。为了应对这个挑战，其中最重要的工作之一就是：微服务节点能够合理的暴露服务的相关监控指标，用以对服务进行健康检查、监控管理，从而进行合理的流量规划与安排系统运维工作！                          

Spring Boot Actuator 可以监控我们的应用程序，收集流量和数据库状态、健康状态等监控指标。在生产环境中，我们可以方便的通过HTTP请求获取应用的状态信息，以JSON格式数据响应。				            

**Spring Boot Actuator监控端点的分类**

- 静态配置类：主要是一些静态配置信息，比如： Spring Bean 加载信息、yml 或properties配置信息、环境变量信息、请求接口关系映射信息等；
- 动态指标类：主要用于展现程序运行期状态，例如内存堆栈信息、请求链信息、健康指标信息等；
- 操作控制类：主要是shutdown功能，用户可以远程发送HTTP请求，从而关闭监控功能。

Actuator开启与配置

在Spring Boot2.x项目中开启Actuator非常简单，只需要引入如下的mavn坐标即可。

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```

Spring Boot Actuator启用之后默认开放了两个端点的访问：

- `/actuator/health`用以监控应用状态。返回值是应用状态信息，包含四种状态DOWN(应用不正常), OUT_OF_SERVICE(服务不可用),UP(状态正常), UNKNOWN(状态未知)。如果服务状态正常，我们访问`http:/lhost:port/actuator/health`得到如下响应信息：

```
{
    "status" : "UP"
}
```

- `/actuator/info` 用来响应应用相关信息，默认为空。可以根据我们自己的需要，向服务调用者暴露相关信息。如下所示,配置属性可以随意起名，但都要挂在info下面：

```
info.app-name=spring-boot-actuator-demo
info.description=spring-boot-actuator-demo indexs monitor 
```

如果我们希望开放更多的监控端点给服务调用者，需要配置：**开放部分监控端点**，端点名称用逗号分隔

```properties
management.endpoints.web.exposure.exclude=beans,env
```

**开放所有监控端点:**

```properties
management.endpoints.web.exposure.include=*
```

星号在YAML配置文件中中有特殊的含义，所以使用YAML配置文件一定要加引号，如下所示：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

常用监控端点说明

注意下图中的服务启用，不等于对外开放访问。对外开放访问的服务端点一定要先开启服务。如果服务不是默认开启的，使用如下方式开启:

```properties
# shutdown是服务端点名称，可以替换
management.endpoint.shutdown.enabled=true
```

| ID             | 描述                                                         | 服务是否默认启用 |
| :------------- | :----------------------------------------------------------- | :--------------- |
| auditevents    | 应用程序的审计事件相关信息                                   | Yes              |
| beans          | 应用中所有Spring Beans的完整列表                             | Yes              |
| conditions     | （configuration and auto-configuration classes）的状态及它们被应用或未被应用的原因 | Yes              |
| configprops    | `@ConfigurationProperties`的集合列表                         | Yes              |
| env            | Spring的 `ConfigurableEnvironment`的属性                     | Yes              |
| flyway         | flyway 数据库迁移路径，如果有的话                            | Yes              |
| liquibase      | Liquibase数据库迁移路径，如果有的话                          | Yes              |
| metrics        | 应用的`metrics`指标信息                                      | Yes              |
| mappings       | 所有`@RequestMapping`路径的集合列表                          | Yes              |
| scheduledtasks | 应用程序中的`计划任务`                                       | Yes              |
| sessions       | 允许从Spring会话支持的会话存储中检索和删除(retrieval and deletion)用户会话。使用Spring Session对反应性Web应用程序的支持时不可用。 | Yes              |
| shutdown       | 允许应用以优雅的方式关闭（默认情况下不启用）                 | No               |
| threaddump     | 执行一个线程dump                                             | Yes              |

如果使用web应用(Spring MVC, Spring WebFlux, 或者 Jersey)，你还可以使用以下端点：

| ID         | 描述                                                         | 默认启用 |
| :--------- | :----------------------------------------------------------- | :------- |
| heapdump   | 返回一个GZip压缩的`hprof`堆dump文件                          | Yes      |
| jolokia    | 通过HTTP暴露`JMX beans`（当Jolokia在类路径上时，WebFlux不可用） | Yes      |
| logfile    | 返回`日志文件内容`（如果设置了logging.file或logging.path属性的话），支持使用HTTP **Range**头接收日志文件内容的部分信息 | Yes      |
| prometheus | 以可以被Prometheus服务器抓取的格式显示`metrics`信息          | Yes      |

## Springboot注解

### @Autowired

@Autowired顾名思义，就是自动装配，其作用是为了消除代码Java代码里面的getter/setter与bean属性中的property。当然，getter看个人需求，如果私有属性需要对外提供的话，应当予以保留。@Autowired默认按类型匹配的方式，在容器查找匹配的Bean，当有且仅有一个匹配的Bean时，Spring将其注入@Autowired标注的变量中。这里@Autowired注解的意思就是，当Spring发现@Autowired注解时，将自动在代码上下文中找到和其匹配（默认是类型匹配）的Bean，并自动注入到相应的地方去

### @Controller

修饰class，用来创建处理http请求的对象.用于标注控制层组件(如struts中的action) 

### @RestController

Spring4之后加入的注解，原来在@Controller中返回json需要@ResponseBody来配合

### @ResponseBody

默认返回json格式

### @RequestMapping：

配置url映射。现在更多的也会直接用以Http Method直接关联的映射注解来定义，比如：GetMapping、PostMapping、DeleteMapping、PutMapping等


### @Service

服务层组件，用于标注业务层组件,表示定义一个bean，自动根据bean的类名实例化一个首写字母为小写的bean，例如Chinese实例化为chinese，如果需要自己改名字则:@Service("你自己改的bean名")。   

### @Repository

持久层组件，用于标注数据访问组件，即DAO组件 

### @Component

泛指组件，当组件不好归类的时候，我们可以使用这个注解进行标注。 

## MyBatis注解

## 1 jdbc配置

> 数据库与数据源不是同一个东西。。。


### 数据源配置

pom.xml

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

### 嵌入式数据库支持

嵌入式数据库支持：H2、HSQL、Derby。不需要任何配置，被集成到springboot的jar包当中。

```
<dependency>
    <groupId>org.hsqldb</groupId>
    <artifactId>hsqldb</artifactId>
    <scope>runtime</scope>
</dependency>
```

### 连接mysql数据库

* 引入mysql依赖包

```
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

* 配置数据源信息

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=dbuser
spring.datasource.password=dbpass
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver//定义了数据引擎
```

### 连接JNDI数据源

JNDI，避免了程序与数据库之间的紧耦合，是指更容易配置和部署。

JNDI不需要用户使用java代码与数据库建立连接，而是将连接交给应用服务器进行管理。java负责与应用服务器上的JNDI通信。

```
spring.datasource.jndi-name=java:jboss/datasources/customers
```

## 2 使用jdbcTemplate操作数据库

### 准备数据库

```
CREATE TABLE `User` (
  `name` varchar(100) COLLATE utf8mb4_general_ci NOT NULL,
  `age` int NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
```

### 编写领域对象（并不是MVC的一部分。数据层，实现数据访问）

```
@Data
@NoArgsConstructor
public class User {

    private String name;
    private Integer age;

}
```

### 编写数据访问对象（并非MVC的一部分。服务层，实现业务逻辑）

* 定义包含插入、删除、查询的抽象接口UserService

```
定义包含有插入、删除、查询的抽象接口UserService
public interface UserService {

    /**
     * 新增一个用户
     *
     * @param name
     * @param age
     */
    int create(String name, Integer age);

    /**
     * 根据name查询用户
     *
     * @param name
     * @return
     */
    List<User> getByName(String name);

    /**
     * 根据name删除用户
     *
     * @param name
     */
    int deleteByName(String name);

    /**
     * 获取用户总量
     */
    int getAllUsers();

    /**
     * 删除所有用户
     */
    int deleteAllUsers();

}
```

* 通过jdbcTemplate实现Userservice中定义的操作。

```
@Service
public class UserServiceImpl implements UserService {

    private JdbcTemplate jdbcTemplate;

    UserServiceImpl(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public int create(String name, Integer age) {
        return jdbcTemplate.update("insert into USER(NAME, AGE) values(?, ?)", name, age);
    }

    @Override
    public List<User> getByName(String name) {
        List<User> users = jdbcTemplate.query("select NAME, AGE from USER where NAME = ?", (resultSet, i) -> {
            User user = new User();
            user.setName(resultSet.getString("NAME"));
            user.setAge(resultSet.getInt("AGE"));
            return user;
        }, name);
        return users;
    }

    @Override
    public int deleteByName(String name) {
        return jdbcTemplate.update("delete from USER where NAME = ?", name);
    }

    @Override
    public int getAllUsers() {
        return jdbcTemplate.queryForObject("select count(1) from USER", Integer.class);
    }

    @Override
    public int deleteAllUsers() {
        return jdbcTemplate.update("delete from USER");
    }

}
```

### 编写单元测试用例

创建对UserService的单元测试用例，通过创建、删除和查询来验证数据库操作的正确性。

```
@RunWith(SpringRunner.class)
@SpringBootTest
public class Chapter31ApplicationTests {

    @Autowired
    private UserService userSerivce;

    @Before
    public void setUp() {
        // 准备，清空user表
        userSerivce.deleteAllUsers();
    }

    @Test
    public void test() throws Exception {
        // 插入5个用户
        userSerivce.create("Tom", 10);
        userSerivce.create("Mike", 11);
        userSerivce.create("Didispace", 30);
        userSerivce.create("Oscar", 21);
        userSerivce.create("Linda", 17);

        // 查询名为Oscar的用户，判断年龄是否匹配
        List<User> userList = userSerivce.getByName("Oscar");
        Assert.assertEquals(21, userList.get(0).getAge().intValue());

        // 查数据库，应该有5个用户
        Assert.assertEquals(5, userSerivce.getAllUsers());

        // 删除两个用户
        userSerivce.deleteByName("Tom");
        userSerivce.deleteByName("Mike");

        // 查数据库，应该有5个用户
        Assert.assertEquals(3, userSerivce.getAllUsers());

    }

}
```

## 1 基本概念

### JDBC

java数据库链接，java database connectivity。java语言用来规范客户访问数据库的应用程序接口。提供了查询、更新数据库的方法。java.sql与javax.sql主要包括以下类：

* DriverManager:负责加载不同的驱动程序Driver，返回相应的数据库连接Connection。
* Driver：对应数据库的驱动程序。
* Connection：数据库连接，负责与数据库进行通信。可以产生SQL的statement.
* Statement:用来执行SQL查询和更新。
* CallableStatement：用以调用数据库中的存储过程。
* SQLException：代表数据库联机额的建立和关闭和SQL语句中发生的例情况。

### 数据源

1. 封装关于数据库访问的各种参数，实现统一管理。
2. 通过数据库的连接池管理，节省开销并提高效率。

> 简单理解，就是在用户程序与数据库之间，建立新的缓冲地带，用来对用户的请求进行优化，对数据库的访问进行整合。

常见的数据源：DBCP、C3P0、Druid、HikariCP。


## 2 HikariCP默认数据源配置

### 通用配置

以spring.datasource.*的形式存在，包括数据库连接地址、用户名、密码。

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

### 数据源连接配置

以spring.datasource.<数据源名称>.*的形式存在，

```
spring.datasource.hikari.minimum-idle=10//最小空闲连接
spring.datasource.hikari.maximum-pool-size=20//最大连接数
spring.datasource.hikari.idle-timeout=500000//控线连接超时时间
spring.datasource.hikari.max-lifetime=540000//最大存活时间
spring.datasource.hikari.connection-timeout=60000//连接超时时间
spring.datasource.hikari.connection-test-query=SELECT 1//用于测试连接是否可用的查询语句
```

## 1 基本配置

### pom.xml配置druid依赖

```
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.21</version>
</dependency>
```

### application.properties配置数据库连接信息

以spring.datasource.druid作为前缀

```
spring.datasource.druid.url=jdbc:mysql://localhost:3306/test
spring.datasource.druid.username=root
spring.datasource.druid.password=
spring.datasource.druid.driver-class-name=com.mysql.cj.jdbc.Driver
```

### 配置druid连接池

> 具体的信息可以自己查询相关的内容。

```
spring.datasource.druid.initialSize=10
spring.datasource.druid.maxActive=20
spring.datasource.druid.maxWait=60000
spring.datasource.druid.minIdle=1
spring.datasource.druid.timeBetweenEvictionRunsMillis=60000
spring.datasource.druid.minEvictableIdleTimeMillis=300000
spring.datasource.druid.testWhileIdle=true
spring.datasource.druid.testOnBorrow=true
spring.datasource.druid.testOnReturn=false
spring.datasource.druid.poolPreparedStatements=true
spring.datasource.druid.maxOpenPreparedStatements=20
spring.datasource.druid.validationQuery=SELECT 1
spring.datasource.druid.validation-query-timeout=500
spring.datasource.druid.filters=stat
```

### 配置druid监控

* 在pom.xml中增加依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

* 在application.properties中添加druid监控配置

```
spring.datasource.druid.stat-view-servlet.enabled=true
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
spring.datasource.druid.stat-view-servlet.reset-enable=true
spring.datasource.druid.stat-view-servlet.login-username=admin
spring.datasource.druid.stat-view-servlet.login-password=admin
```

> 对象关系映射模型Hibernate。用来实现非常轻量级的对象的封装。将对象与数据库建立映射关系。实现增删查改。
> MyBatis与Hibernate非常相似。对象关系映射模型ORG。java对象与关系数据库映射的模型。

## 1 配置MyBatis

### 在pom.xml中添加MyBatis依赖

```
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.1</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

### 在application.properties中配置mysql的链接配置

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```

### 创建关系数据表

```
CREATE TABLE `User` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `name` varchar(100) COLLATE utf8mb4_general_ci DEFAULT NULL,
  `age` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
```

### 创建数据表的java对象

```
@Data
@NoArgsConstructor
public class User {

    private Long id;

    private String name;
    private Integer age;

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
}
```

### 创建数据表的操作接口

```
@Mapper
public interface UserMapper {

    @Select("SELECT * FROM USER WHERE NAME = #{name}")
    User findByName(@Param("name") String name);

    @Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
    int insert(@Param("name") String name, @Param("age") Integer age);

}
```

## 2 MyBatis参数传递

### 使用@Param参数传递

```
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
int insert(@Param("name") String name, @Param("age") Integer age);
```

### 使用map 传递参数

```
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name,jdbcType=VARCHAR}, #{age,jdbcType=INTEGER})")
int insertByMap(Map<String, Object> map);
//调用
Map<String, Object> map = new HashMap<>();
map.put("name", "CCC");
map.put("age", 40);
userMapper.insertByMap(map);
```

### 使用普通java对象

```
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
int insertByUser(User user);
```

## 3 增删查改操作

```
public interface UserMapper {

    @Select("SELECT * FROM user WHERE name = #{name}")
    User findByName(@Param("name") String name);

    @Insert("INSERT INTO user(name, age) VALUES(#{name}, #{age})")
    int insert(@Param("name") String name, @Param("age") Integer age);

    @Update("UPDATE user SET age=#{age} WHERE name=#{name}")
    void update(User user);

    @Delete("DELETE FROM user WHERE id =#{id}")
    void delete(Long id);
}
```

对增删查改的调用

```
@Transactional
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {

	@Autowired
	private UserMapper userMapper;

	@Test
	@Rollback
	public void testUserMapper() throws Exception {
		// insert一条数据，并select出来验证
		userMapper.insert("AAA", 20);
		User u = userMapper.findByName("AAA");
		Assert.assertEquals(20, u.getAge().intValue());
		// update一条数据，并select出来验证
		u.setAge(30);
		userMapper.update(u);
		u = userMapper.findByName("AAA");
		Assert.assertEquals(30, u.getAge().intValue());
		// 删除这条数据，并select验证
		userMapper.delete(u.getId());
		u = userMapper.findByName("AAA");
		Assert.assertEquals(null, u);
	}
}
```

## 4 使用MyBatis的XML方式

### 在应用主类中增加mapper的扫描包配置：

```
@MapperScan("com.didispace.chapter36.mapper")
@SpringBootApplication
public class Chapter36Application {

	public static void main(String[] args) {
		SpringApplication.run(Chapter36Application.class, args);
	}

}
```

### Mapper包下创建User表

```
public interface UserMapper {

    User findByName(@Param("name") String name);

    int insert(@Param("name") String name, @Param("age") Integer age);

}
```

### 在配置文件中通过mybatis.mapper-locations参数指定xml配置的位置

```
mybatis.mapper-locations=classpath:mapper/*.xml
```

### xml配置目录下创建User表的mapper配置

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.didispace.chapter36.mapper.UserMapper">
    <select id="findByName" resultType="com.didispace.chapter36.entity.User">
        SELECT * FROM USER WHERE NAME = #{name}
    </select>

    <insert id="insert">
        INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})
    </insert>
</mapper>
```

### 对xml方式进行调用

```
@Slf4j
@RunWith(SpringRunner.class)
@SpringBootTest
@Transactional
public class Chapter36ApplicationTests {

    @Autowired
    private UserMapper userMapper;

    @Test
    @Rollback
    public void test() throws Exception {
        userMapper.insert("AAA", 20);
        User u = userMapper.findByName("AAA");
        Assert.assertEquals(20, u.getAge().intValue());
    }

}
```

# SpringBoot源码

# 核心原理入门

springboot的有两大核心

- 依赖管理
- 自动装配

## **依赖管理**

### POM文件

#### 父项目

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.1.RELEASE</version>
    <relativePath/>
</parent>
```

其父项目是

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.0.1.RELEASE</version>
    <relativePath>../../spring-boot-dependencies</relativePath>
</parent>
```

该父项目是真正管理Spring Boot应用里面的所有依赖的版本：Spring Boot的版本仲裁中心，所以以后导入的依赖默认是不需要版本号。如下

```
  <properties>
    <activemq.version>5.16.3</activemq.version>
    <antlr2.version>2.7.7</antlr2.version>
    <appengine-sdk.version>1.9.91</appengine-sdk.version>
    <artemis.version>2.17.0</artemis.version>
    <aspectj.version>1.9.7</aspectj.version>
    <assertj.version>3.19.0</assertj.version>
    <atomikos.version>4.0.6</atomikos.version>
    <awaitility.version>4.0.3</awaitility.version>
    <build-helper-maven-plugin.version>3.2.0</build-helper-maven-plugin.version>
    <byte-buddy.version>1.10.22</byte-buddy.version>
    <caffeine.version>2.9.2</caffeine.version>
    <cassandra-driver.version>4.11.3</cassandra-driver.version>
    <classmate.version>1.5.1</classmate.version>
    <commons-codec.version>1.15</commons-codec.version>
    <commons-dbcp2.version>2.8.0</commons-dbcp2.version>
    <commons-lang3.version>3.12.0</commons-lang3.version>
    <commons-pool.version>1.6</commons-pool.version>
    <commons-pool2.version>2.9.0</commons-pool2.version>
    <couchbase-client.version>3.1.7</couchbase-client.version>
    <db2-jdbc.version>11.5.6.0</db2-jdbc.version>
    <dependency-management-plugin.version>1.0.11.RELEASE</dependency-management-plugin.version>
    <derby.version>10.14.2.0</derby.version>
    <dropwizard-metrics.version>4.1.25</dropwizard-metrics.version>
    <ehcache.version>2.10.9.2</ehcache.version>
    <ehcache3.version>3.9.5</ehcache3.version>
    <elasticsearch.version>7.12.1</elasticsearch.version>
    <embedded-mongo.version>3.0.0</embedded-mongo.version>
    <flyway.version>7.7.3</flyway.version>
    <freemarker.version>2.3.31</freemarker.version>
    <git-commit-id-plugin.version>4.0.5</git-commit-id-plugin.version>
    <glassfish-el.version>3.0.3</glassfish-el.version>
    <glassfish-jaxb.version>2.3.5</glassfish-jaxb.version>
    <groovy.version>3.0.8</groovy.version>
    <gson.version>2.8.7</gson.version>
    <h2.version>1.4.200</h2.version>
    <hamcrest.version>2.2</hamcrest.version>
    <hazelcast.version>4.1.5</hazelcast.version>
    <hazelcast-hibernate5.version>2.2.1</hazelcast-hibernate5.version>
    <hibernate.version>5.4.32.Final</hibernate.version>
    <hibernate-validator.version>6.2.0.Final</hibernate-validator.version>
    <hikaricp.version>4.0.3</hikaricp.version>
    <hsqldb.version>2.5.2</hsqldb.version>
    <htmlunit.version>2.49.1</htmlunit.version>
    <httpasyncclient.version>4.1.4</httpasyncclient.version>
    <httpclient.version>4.5.13</httpclient.version>
    <httpclient5.version>5.0.4</httpclient5.version>
    <httpcore.version>4.4.14</httpcore.version>
    <httpcore5.version>5.1.1</httpcore5.version>
    <infinispan.version>12.1.7.Final</infinispan.version>
    <influxdb-java.version>2.21</influxdb-java.version>
    <jackson-bom.version>2.12.4</jackson-bom.version>
    <jakarta-activation.version>1.2.2</jakarta-activation.version>
    <jakarta-annotation.version>1.3.5</jakarta-annotation.version>
    <jakarta-jms.version>2.0.3</jakarta-jms.version>
    <jakarta-json.version>1.1.6</jakarta-json.version>
    <jakarta-json-bind.version>1.0.2</jakarta-json-bind.version>
    <jakarta-mail.version>1.6.7</jakarta-mail.version>
    <jakarta-persistence.version>2.2.3</jakarta-persistence.version>
    <jakarta-servlet.version>4.0.4</jakarta-servlet.version>
    <jakarta-servlet-jsp-jstl.version>1.2.7</jakarta-servlet-jsp-jstl.version>
    <jakarta-transaction.version>1.3.3</jakarta-transaction.version>
    <jakarta-validation.version>2.0.2</jakarta-validation.version>
    <jakarta-websocket.version>1.1.2</jakarta-websocket.version>
    <jakarta-ws-rs.version>2.1.6</jakarta-ws-rs.version>
    <jakarta-xml-bind.version>2.3.3</jakarta-xml-bind.version>
    <jakarta-xml-soap.version>1.4.2</jakarta-xml-soap.version>
    <jakarta-xml-ws.version>2.3.3</jakarta-xml-ws.version>
    <janino.version>3.1.6</janino.version>
    <javax-activation.version>1.2.0</javax-activation.version>
    <javax-annotation.version>1.3.2</javax-annotation.version>
    <javax-cache.version>1.1.1</javax-cache.version>
    <javax-jaxb.version>2.3.1</javax-jaxb.version>
    <javax-jaxws.version>2.3.1</javax-jaxws.version>
    <javax-jms.version>2.0.1</javax-jms.version>
    <javax-json.version>1.1.4</javax-json.version>
    <javax-jsonb.version>1.0</javax-jsonb.version>
    <javax-mail.version>1.6.2</javax-mail.version>
    <javax-money.version>1.1</javax-money.version>
    <javax-persistence.version>2.2</javax-persistence.version>
    <javax-transaction.version>1.3</javax-transaction.version>
    <javax-validation.version>2.0.1.Final</javax-validation.version>
    <javax-websocket.version>1.1</javax-websocket.version>
    <jaxen.version>1.2.0</jaxen.version>
    <jaybird.version>4.0.3.java8</jaybird.version>
    <jboss-logging.version>3.4.2.Final</jboss-logging.version>
    <jboss-transaction-spi.version>7.6.1.Final</jboss-transaction-spi.version>
    <jdom2.version>2.0.6</jdom2.version>
    <jedis.version>3.6.3</jedis.version>
    <jersey.version>2.33</jersey.version>
    <jetty-el.version>9.0.48</jetty-el.version>
    <jetty-jsp.version>2.2.0.v201112011158</jetty-jsp.version>
    <jetty-reactive-httpclient.version>1.1.10</jetty-reactive-httpclient.version>
    <jetty.version>9.4.43.v20210629</jetty.version>
    <jmustache.version>1.15</jmustache.version>
    <johnzon.version>1.2.14</johnzon.version>
    <jolokia.version>1.6.2</jolokia.version>
    <jooq.version>3.14.13</jooq.version>
    <json-path.version>2.5.0</json-path.version>
    <json-smart.version>2.4.7</json-smart.version>
    <jsonassert.version>1.5.0</jsonassert.version>
    <jstl.version>1.2</jstl.version>
    <jtds.version>1.3.1</jtds.version>
    <junit.version>4.13.2</junit.version>
    <junit-jupiter.version>5.7.2</junit-jupiter.version>
    <kafka.version>2.7.1</kafka.version>
    <kotlin.version>1.5.21</kotlin.version>
    <kotlin-coroutines.version>1.5.1</kotlin-coroutines.version>
    <lettuce.version>6.1.4.RELEASE</lettuce.version>
    <liquibase.version>4.3.5</liquibase.version>
    <log4j2.version>2.14.1</log4j2.version>
    <logback.version>1.2.5</logback.version>
    <lombok.version>1.18.20</lombok.version>
    <mariadb.version>2.7.4</mariadb.version>
    .........
    
```

### 启动器(spring-boot-starter)

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

 **spring-boot-starter** : spring boot场景启动器；帮助导入web模块正常运行所依赖的组件；

 Spring Boot将所有的功能场景抽取出来，做成一个个的starter(启动器)，只需要在项目中引入这些starter，那么相关的场景的所有依赖都会导入进项目中。要用什么功能就导入什么场景的启动器。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
</dependency>
```

SpringBoot能够如此方便便捷，其实都是得益于这些“开箱即用”的依赖模块，那SpringBoot设计者约定这些“开箱即用”的依赖模块的命名都以`spring-boot-starter-`开始，并且这些模块都位于`org.springframework.boot`包或者命名空间下面。我们也可以模仿者来实现自己的自动配置依赖模块，也已`spring-boot-starter-`开头，是不是就很"正宗"呢？(虽然SpringBoot官方不建议我们这样做，以免跟官方提供的混淆，但是其实我们使用自己的groupId，这样命名应该不是啥问题)。

　　这些starter其实都有约定好的默认配置，但是它也允许我们调整这些默认配置，以便完成定制化的需求，我们可以改变默认配置的常见方式有以下几种：

- 命令行参数(Command Line Args)
- 系统环境变量(Environment Variables)
- 位于文件系统中的配置文件
- 位于classpath中的配置文件
- 固化到代码中的配置项

　　这几种方式从上到下优先级从高到低排列，高优先级的配置会覆盖优先级低的配置。还有就是不管位于文件系统还是classpath中的配置文件，SpringBoot应用默认的文件名称都是`application.properties`,可以放在当前项目的根目录下或者名称为config的子目录下。

　　SpringBoot其实提供了很多这样的模块，我们就挑几个我们常用的这样的模块来解析，其他的大家就举一反三。以达到在工作和开发中灵活运用这些spring-boot-starter模块的效果。

#### spring-boot-starter-logging(应用日志)

如果我们在maven依赖中添加了`spring-boot-starter-logging`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```

　　那也就意味着我们的SpringBoot应用自动使用logback作为日志框架，在启动的时候，由`org.springframework.boot.logging.LoggingApplicationListener`根据情况初始化并使用。默认情况下，SpringBoot已经给我们提供好了很多默认的日志配置，我们只需要将`spring-boot-starter-logging`作为依赖加入到你的SpringBoot应用就可以了，但是如果我们要对这些默认配置进行定制，可以有两种方式进行：

- 遵守logback的约定，在classpath中使用定制化的logback.xml配置文件。

- 在文件系统中任意一个地方提供自己的logback.xml配置文件，然后通过如下配置来`application.properties`中指定我们日志系统配置文件位置：

  ```properties
  logging.config=/{your config file location}}/logback.xml
  ```

如果我们已经习惯了log4j或log4j2,那我们只需要把`spring-boot-starter-logging`换成如下的starter就好。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j</artifactId>
</dependency>
或
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

#### spring-boot-starter-web(快速构建web应用)

现如今，我们在工作中大部分实际用的还是SpringMVC开发的web应用，SpringBoot当然贴心的为我们开发了一个web项目模块，让我们更加方便的开发web应用。maven依赖如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

　　这样我们就可以得到一个可以直接执行的Web应用，然后我们运行`mvn spring-boot:run`，就能直接启动一个基于嵌入式tomcat容器的Web应用了，然后就可以像这篇文章中定义controller来供用户访问了。但是呢，这简单的表象之下，其实却隐藏着很多约定，我们要把这些潜规则了解清楚才能更好地应用`spring-boot-starter-web`。

##### 项目结构的“潜规则”

 　　传统的Java Web项目中，我们的静态文件以及页面模板都是放在`src/main/webapp`目录下，但是在SpringBoot应用中，这些文件被统一放在`src/main/resources`相应的子目录下：

- `src/main/resources/static`目录用于存放各种静态资源，如：js、css、image等。
- `src/main/resources/template`目录用于存放模板文件。

> 细心地我们会发现SpringBoot的web应用已经变成了jar包而再是war包，如果我们还是希望以war包的形式发布也是可以的。

##### SpringMVC框架层面的约定及定制

`spring-boot-starter-web`默认将为我们自动配置如下一些SpringMVC必要的组件：

- ViewResolver，如：`ContentNegotiatingViewResolver`和`BeanNameViewResolver`。
- Converter，如：`GenericConverter`和`Formatter`等bean被注册到IoC容器。
- 默认添加一系列`HttpMessageConverter`用于支持对Web请求和相应的类型转换。
- 自动配置和注册`MessageCodesResolver`。
- 其他必要组件…

##### 嵌入式Web容器的约定和定制

　　我们知道`spring-boot-starter-web`默认把嵌入式tomcat作为web容器来对外提供HTTP服务，默认使用8080端口对外监听和提供服务。这里我们可能会有两个疑问：

- 我们不想使用默认的嵌入式tomcat容器怎么办？

  很简单，我们只需要引入`spring-boot-starter-jetty`或`spring-boot-starter-undertow`依赖就能替代默认嵌入式tomcat容器了。

- 我们想要把启动后提供服务的端口改掉怎么办？

  我们可以通过在配置文件中修改启动端口就可以了，如：

  ```properties
  server.port=9000
  ```

其实，`spring-boot-starter-web`提供了很多以`server.`作为前缀的配置以用来修改嵌入式容器的配置，如：

```properties
server.port
server.address
server.ssl.*
server.tomcat.*
```

那若这些还满足不了你，SpringBoot甚至都允许我们直接对嵌入式Web容器实例进行定制化，我们通过向IoC容器中注册一个`EmbeddedServletContainerCustomizer`类型的组件来实现：

```java
package com.springbootdemo;

import org.springframework.boot.context.embedded.ConfigurableEmbeddedServletContainer;
import org.springframework.boot.context.embedded.EmbeddedServletContainerCustomizer;

/**
 * @description: 自定义内嵌容器配置
 */
public class DemoEmbeddedTomcatCustomizer implements EmbeddedServletContainerCustomizer {
    @Override
    public void customize(ConfigurableEmbeddedServletContainer container) {
        container.setPort(9111);
        container.setContextPath("/demo");
        // ...
    }
}
```

如果还要再深入的定制，那就需要实现对应内嵌容器的Factory并注册到IoC容器：

- TomcatEmbeddedServletContainerFactory
- JettyEmbeddedServletContainerFactory
- UndertowEmbeddedServletContainerFactory

　　但是，我们几乎没有可能需要这样的定制化，也不建议这样的定制化，使用SpringBoot默认的`spring-boot-starter-web`提供的配置项列表已经很简单、很完整了。

#### spring-boot-starter-jdbc(数据访问)

我们知道，现实中大多数的Java应用都需要访问数据库，那SpringBoot肯定不会放过这个组件，它会很贴心的为我们自动配置好相应的数据访问工具。我们只需要在`pom.xml`中添加以下依赖就好了：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

　　这样，在我们没有配置任何DataSource的情况下，SpringBoot会默认为我们自动配置一个基于嵌入式数据的DataSource，这种自动配置适合于测试场景，生产环境不适合。大多数情况下，我们都会自己配置DataSource实例，或通过自动配置模块提供的配置参数对DataSource实例配置自定义的参数。

若我们的SpringBoot应用只依赖一个数据库，那我们直接使用自动配置模块提供的配置参数最方便快捷：

```properties
spring.datasource.url=jdbc:mysql://{db host}:{db port}/{db name}
spring.datasource.username={db user name}
spring.datasource.password={db password}
```

　　有的小伙伴说了：那我自己配置一个DataSource行不行？答案是当然可以，SpringBoot会很智能的优先选择使用我们自己配置的这个DataSource，但是感觉多此一举！你要知道，SpringBoot除了自动帮我们配置DataSource以外，还自动帮我们配置了相应的`JdbcTemplate`以及`DataSourceTransactionManager`等相关的组件，我们只需要在需要使用的地方直接使用`@Autowired`注解引用就好了。

　　那SpringBoot是不是一直贴心呢？很明显不是的，如果我们的单个项目需要依赖和访问多个数据库，这个时候就不行了，就算是我们在ApplicationContext中配置了多个DataSource实例来访问多个数据库：

```java
@Bean
public DataSource dataSource1() throws Throwable {
    DruidDataSource ds = new DruidDataSource();
    ds.setUrl(...);
    ds.setUsername(...);
    ds.setPassword(...);
    // set other db setting
    return ds;
}
@Bean
public DataSource dataSource2() throws Throwable {
    DruidDataSource ds = new DruidDataSource();
    ds.setUrl(...);
    ds.setUsername(...);
    ds.setPassword(...);
    // set other db setting
    return ds;
}
```

启动项目时，你就会发现如下的异常:

```java
No qualifying bean of type [javax.sql.DataSource] is defined: expected single matching bean but found 2...
```

那怎么解决这个问题呢？有两种方式：

- 在SpringBoot的启动类上“动手脚”

  ```java
  @SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    DataSourceTransactionManagerAutoConfiguration.class
  })
  public class DemoSpringBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoSpringBootApplication.class, args);
    }
  }
  ```

  这也就是说我们需要排除掉SpringBoot默认的DataSource的相关的自动配置。

- 使用`@primary`注解

  那我们既要配置两个数据源，又要使用SpringBoot默认的DataSource，这时我们就可以为我们配置的两个DataSource中的任意一个使用`@primary`注解就可以了。

  ```java
  @Bean
  @Primary
  public DataSource dataSource1() throws Throwable {
    DruidDataSource ds = new DruidDataSource();
    ds.setUrl(...);
    ds.setUsername(...);
    ds.setPassword(...);
    // set other db setting
    return ds;
  }
  @Bean
  public DataSource dataSource2() throws Throwable {
    DruidDataSource ds = new DruidDataSource();
    ds.setUrl(...);
    ds.setUsername(...);
    ds.setPassword(...);
    // set other db setting
    return ds;
  }
  ```

  除此之外，SpringBoot还提供了很多其他数据源访问相关的自动配置模块，如：`spring-boot-starter-jpa`、`spring-boot-starter-mongodb`等。

#### 自定义starter

首先定义一个配置类模块：

```
@Configuration
@ConditionalOnProperty(name = "enabled.autoConfituration", matchIfMissing = true)
public class MyAutoConfiguration {

    static {
        System.out.println("myAutoConfiguration init...");
    }

    @Bean
    public SimpleBean simpleBean(){
        return new SimpleBean();
    }

}
```

然后定义一个starter模块，里面无需任何代码，pom也无需任何依赖，只需在META-INF下面建一个 `spring.factories`文件，添加如下配置：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.springdemo.MyAutoConfiguration
```

最后只需在启动类项目的pom中引入我们的 starter 模块即可。

**springBoot为我们提供的配置类有多个，但是我们不可能会全部引入。按条件注解 @Conditional或者@ConditionalOnProperty等相关注解进行判断，决定是否需要装配。**

我们自定义的配置类也是以相同的逻辑进行装配，我们指定了以下注解：

```
@ConditionalOnProperty(name = "enabled.autoConfituration", matchIfMissing = true)
```

默认为 true，所以自定义的starter成功执行。

## 自动配置

springboot是基于spring的新型的轻量级框架，最厉害的地方当属**自动配置**。那我们就可以根据启动流程和相关原理来看看，如何实现传奇的自动配置。

### springboot的启动类入口

springboot有自己独立的启动类（独立程序）

```
@SpringBootApplication
public class HelloWorldApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloWorldApplication.class, args);
    }
    
}
```

**@SpringBootApplication**

- Spring Boot应用标注在某个类上，说明这个类是SpringBoot的主配置类，SpringBoot就应该运行这个类的main方法来启动SpringBoot应用。

### SpringBootApplication注解

```
@Target(ElementType.TYPE) // 注解的适用范围，其中TYPE用于描述类、接口（包括包注解类型）或enum声明
@Retention(RetentionPolicy.RUNTIME) // 注解的生命周期，保留到class文件中（三个生命周期）
@Documented // 表明这个注解应该被javadoc记录
@Inherited // 子类可以继承该注解
@SpringBootConfiguration // 继承了Configuration，表示当前是注解类
@EnableAutoConfiguration // 开启springboot的注解功能，springboot的四大神器之一，其借助@import的帮助
@ComponentScan(excludeFilters = { // 扫描路径设置
@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
...
}　
```

在其中比较重要的有三个注解，分别是：

- @SpringBootConfiguration // 继承了Configuration，表示当前是注解类
- @EnableAutoConfiguration // 开启springboot的注解功能，springboot的四大神器之一，其借助@import的帮助
- @ComponentScan(excludeFilters = { // 扫描路径设置（具体使用待确认）

#### @SpringBootConfiguration注解　　

**@SpringBootConfiguration**

- Spring Boot的配置类
- 标注在某个类上，表示这是一个Spring Boot的配置类

注解定义如下：

```
@Configuration
public @interface SpringBootConfiguration {}
```

其实就是一个**Configuration配置类**，意思是HelloWorldMainApplication最终会被注册到Spring容器中

**配置bean方式的不同**　

- xml配置文件的形式配置bean

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd"
default-lazy-init="true">
<!--bean定义-->
</beans>
```

- java configuration的配置形式配置bean

```
@Configuration
public class MockConfiguration{
    //bean定义
}
```

**注入bean方式的不同**

- xml配置文件的形式注入bean

```
<bean id="mockService" class="..MockServiceImpl">
...
</bean>
```

- java configuration的配置形式注入bean

```
@Configuration
public class MockConfiguration{
    @Bean
    public MockService mockService(){
        return new MockServiceImpl();
    }
}
```

任何一个标注了@Bean的方法，其返回值将作为一个bean定义注册到Spring的IoC容器，方法名将默认成该bean定义的id。

**表达bean之间依赖关系的不同**

- xml配置文件的形式表达依赖关系

```
<bean id="mockService" class="..MockServiceImpl">
　　<propery name ="dependencyService" ref="dependencyService" />
</bean>
<bean id="dependencyService" class="DependencyServiceImpl"></bean>
```

- java configuration配置的形式表达**依赖关系（重点）**

**如果一个bean A的定义依赖其他bean B,则直接调用对应的JavaConfig类中依赖bean B的创建方法就可以了。**

```
@Configuration
public class MockConfiguration{
　　@Bean
　　public MockService mockService(){
    　　return new MockServiceImpl(dependencyService());
　　}
　　@Bean
　　public DependencyService dependencyService(){
    　　return new DependencyServiceImpl();
　　}
}
```

#### @ComponentScan注解

ComponentScan注解作用

- 对应xml配置中的元素
- **（重点）**ComponentScan的功能其实就是自动扫描并加载符合条件的组件（比如@Component和@Repository等）或者bean定义
- 将这些bean定义加载到**IoC**容器中

我们可以通过basePackages等属性来**细粒度**的定制@ComponentScan自动扫描的范围，如果不指定，则**默认**Spring框架实现会从声明@ComponentScan所在类的package进行扫描。

注：所以SpringBoot的启动类最好是放在root ``package``下，因为默认不指定basePackages

#### **@EnableAutoConfiguration**注解

- 开启自动配置功能
- 以前使用Spring需要配置的信息，Spring Boot帮助自动配置；
- **@EnableAutoConfiguration**通知SpringBoot开启自动配置功能，这样自动配置才能生效。

　　此注解顾名思义是可以自动配置，所以应该是springboot中最为重要的注解。在spring框架中就提供了各种以@Enable开头的注解，例如： @EnableScheduling、@EnableCaching、@EnableMBeanExport等； @EnableAutoConfiguration的理念和做事方式其实一脉相承简单概括一下就是，借助@Import的支持，收集和注册特定场景相关的bean定义。　　

- @EnableScheduling是通过@Import将Spring调度框架相关的bean定义都加载到IoC容器【定时任务、时间调度任务】
- @EnableMBeanExport是通过@Import将JMX相关的bean定义加载到IoC容器【监控JVM运行时状态】
- @EnableAutoConfiguration也是借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器。

@EnableAutoConfiguration作为一个复合Annotation,其自身定义关键信息如下：

```
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage【重点注解】
@Import(AutoConfigurationImportSelector.class)【重点注解】
public @interface EnableAutoConfiguration {
...
}
```

### EnableAutoConfiguration注解

最重要的两个注解

- @AutoConfigurationPackage
- @Import(AutoConfigurationImportSelector.class)

#### AutoConfigurationPackage注解

- 自动配置包注解

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
 
}
```

**@Import(AutoConfigurationPackages.Registrar.class)**：默认将主配置类(**@SpringBootApplication**)所在的包及其子包里面的所有组件扫描到Spring容器中。如下

```
@Order(Ordered.HIGHEST_PRECEDENCE)
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {
          //默认将会扫描@SpringBootApplication标注的主配置类所在的包及其子包下所有组件
        register(registry, new PackageImport(metadata).getPackageName());
    }

    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        return Collections.<Object>singleton(new PackageImport(metadata));
    }
}
```

- 注册当前启动类的根package；
- 注册org.springframework.boot.autoconfigure.AutoConfigurationPackages的BeanDefinition。

#### Import(AutoConfigurationImportSelector.class)注解

**AutoConfigurationImportSelector**： 导入哪些组件的选择器，将所有需要导入的组件以全类名的方式返回，这些组件就会被添加到容器中。

**AutoConfigurationImportSelector 实现**了 DeferredImportSelector 从 ImportSelector继承的方法：**selectImports**。

```
@Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                .loadMetadata(this.beanClassLoader);
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        configurations = removeDuplicates(configurations);
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = filter(configurations, autoConfigurationMetadata);
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return StringUtils.toStringArray(configurations);
    }
```

我们主要看List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes)会给容器中注入众多的自动配置类（xxxAutoConfiguration）。

```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
            AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
        getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    //...
    return configurations;
}

protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}

public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    try {
        //从类路径的META-INF/spring.factories中加载所有默认的自动配置类
        Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                                 ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        List<String> result = new ArrayList<String>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
            //获取EnableAutoConfiguration指定的所有值,也就是EnableAutoConfiguration.class的值
            String factoryClassNames = properties.getProperty(factoryClassName);
            result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
        }
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

就是给容器中导入这个场景需要的所有组件，并配置好这些组件。其实是去加载各个组件jar下的  public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"外部文件。

SpringBoot启动的时候从类路径下的 **META-INF/spring.factories**中获取EnableAutoConfiguration指定的值，并将这些值作为自动配置类导入到容器中，自动配置类就会生效，最后完成自动配置工作。EnableAutoConfiguration默认在spring-boot-autoconfigure这个包中，如下图

```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.autoconfigure.integration.IntegrationPropertiesEnvironmentPostProcessor

... ...
```

该方法在springboot启动流程——bean实例化前被执行，返回要实例化的类信息列表；

如果获取到类信息，spring可以通过类加载器将类加载到jvm中，现在我们已经通过spring-boot的starter依赖方式依赖了我们需要的组件，那么这些组件的类信息在select方法中就可以被获取到。

```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
 List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
 Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
 return configurations;
 }
```

其返回一个自动配置类的类名列表，方法调用了loadFactoryNames方法，查看该方法

```
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
 String factoryClassName = factoryClass.getName();
 return (List)loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
 }
```

自动配置器会跟根据传入的factoryClass.getName()到项目系统路径下所有的spring.factories文件中找到相应的key，从而加载里面的类。

**（重点）**其中，最关键的要属@Import(AutoConfigurationImportSelector.class)，借助AutoConfigurationImportSelector，@EnableAutoConfiguration可以帮助SpringBoot应用将所有符合条件(spring.factories)的**bean定义**（如Java Config@Configuration配置）都加载到当前SpringBoot创建并使用的IoC容器。 

### SpringFactoriesLoader详解

借助于Spring框架**原有**的一个工具类：**SpringFactoriesLoader**的支持，@EnableAutoConfiguration可以智能的自动配置功效才得以大功告成！

SpringFactoriesLoader属于Spring框架私有的一种扩展方案，其主要功能就是从**指定的配置文件META-INF/spring.factories**加载配置,**加载工厂类**。

SpringFactoriesLoader为Spring工厂加载器，该对象提供了loadFactoryNames方法，入参为factoryClass和classLoader即需要传入**工厂类**名称和对应的类加载器，方法会根据指定的classLoader，加载该类加器搜索路径下的指定文件，即spring.factories文件；

传入的工厂类为接口，而文件中对应的类则是接口的实现类，或最终作为实现类。

```
public abstract class SpringFactoriesLoader {
//...
　　public static <T> List<T> loadFactories(Class<T> factoryClass, ClassLoader classLoader) {
　　　　...
　　}
   
   
　　public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
　　　　....
　　}
}
```

配合@EnableAutoConfiguration使用的话，它更多是提供一种配置查找的功能支持，即根据@EnableAutoConfiguration的完整类名org.springframework.boot.autoconfigure.EnableAutoConfiguration作为查找的Key,获取对应的一组@Configuration类　　

上图就是从SpringBoot的autoconfigure依赖包中的META-INF/spring.factories配置文件中摘录的一段内容，可以很好地说明问题。

（重点）所以，@EnableAutoConfiguration自动配置的魔法其实就变成了：

从classpath中搜寻所有的META-INF/spring.factories配置文件，并将其中org.springframework.boot.autoconfigure.EnableAutoConfiguration对应的**配置项**通过**反射（Java Refletion）**实例化为对应的标注了**@Configuration**的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。

### 自动配置奥秘

`@EnableAutoConfiguration`借助SpringFactoriesLoader可以将标注了`@Configuration`这个注解的JavaConfig类一并汇总并加载到最终的ApplicationContext，这么说只是很简单的解释，其实基于`@EnableAutoConfiguration`的自动配置功能拥有非常强大的调控能力。比如我们可以通过配合基于条件的配置能力或定制化加载顺序，对自动化配置进行更加细粒度的调整和控制。

#### 基于条件的自动配置

这个基于条件的自动配置来源于Spring框架中的"基于条件的配置"特性。在Spring框架中，我们可以使用`@Conditional`这个注解配合`@Configuration`或`@Bean`等注解来干预一个配置或bean定义是否能够生效，它最终实现的效果或者语义类如下伪代码：

```
if (复合@Conditional规定的条件) {
    加载当前配置（Enable Current Configuration）或者注册当前bean定义;
}
```

要实现基于条件的配置，我们需要通过`@Conditional`注解指定自己Condition实现类就可以了(可以应用于类型Type的注解或者方法Method的注解)

```
@Conditional({DemoCondition1.class, DemoCondition2.class})
```

最重要的是，`@Conditional`注解可以作为一个Meta Annotaion用来标注其他注解实现类，从而构建各种复合注解，比如SpringBoot的autoconfigre模块就基于这一优良的革命传统，实现了一批这样的注解(在org.springframework.boot.autoconfigure.condition包下):

- @ConditionalOnClass
- @ConditionalOnBean
- @CondtionalOnMissingClass
- @CondtionalOnMissingBean
- @CondtionalOnProperty
- ……

　　有了这些复合Annotation的配合，我们就可以结合@EnableAutoConfiguration实现基于条件的自动配置了。其实说白了，SpringBoot能够如此的盛行，很重要的一部分就是它默认提供了一系列自动配置的依赖模块，而这些依赖模块都是基于以上的@Conditional复合注解实现的，这也就说明这些所有的依赖模块都是按需加载的，只有复合某些特定的条件，这些依赖模块才会生效，这也解释了为什么自动配置是“智能”的。

#### 定制化自动配置的顺序

在实现自动配置的过程中，我们除了可以提供基于条件的配置之外，我们还能对当前要提供的配置或组件的加载顺序进行个性化调整，以便让这些配置或者组件之间的依赖分析和组装能够顺利完成。

　　最经典的是我们可以通过使用`@org.springframework.boot.autoconfigure.AutoConfigureBefore`或者`@org.springframework.boot.autoconfigure.AutoConfigureAfter`让当前配置或者组件在某个其他组件之前或者之后进行配置。例如，假如我们希望某些JMX操作相关的bean定义在MBeanServer配置完成以后在进行配置，那我们就可以提供如下配置：

```
@Configuration
@AutoConfigureAfter(JmxAutoConfiguration.class)
public class AfterMBeanServerReadyConfiguration {
    @Autowired
    MBeanServer mBeanServer;

    // 通过@Bean添加其他必要的bean定义
}
```









## SpringBoot启动流程

springboot有自己独立的启动类（独立程序）

```
@SpringBootApplication
public class HelloWorldApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloWorldApplication.class, args);
    }
    
}
```

![springboot启动流程](springboot\springboot启动流程.png)

**SpringApplication.run(HelloWorldMainApplication.class, args);**我们跟进去看看

```
// 调用静态类，参数对应的就是HelloWorldMainApplication.class以及main方法中的args
public static ConfigurableApplicationContext run(Class<?> primarySource,String... args) {
    return run(new Class<?>[] { primarySource }, args);
}
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
    return (new SpringApplication(sources)).run(args);
}
```

它实际上会构造一个**SpringApplication**的实例，并把我们的启动类**HelloWorldApplication.class**作为参数传进去，然后运行它的run方法

SpringApplication构造器

```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    //把HelloWorldMainApplication.class设置为属性存储起来
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //设置应用类型是Standard还是Web
    this.webApplicationType = deduceWebApplicationType();
    //设置初始化器(Initializer),最后会调用这些初始化器
    setInitializers((Collection) getSpringFactoriesInstances( ApplicationContextInitializer.class));
    //设置监听器(Listener)
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

- **设置应用类型:**首先运行deduceWebEnvironment方法(代码中标记1处)，该方法的作用是根据classpath里面是否存在某些特征类({“javax.servlet.Servlet”, “org.springframework.web.context.ConfigurableWebApplicationContext” })来决定是创建一个Web类型的ApplicationContext还是创建一个标准Standalone类型的ApplicationContext.
- **设置初始化器(Initializer):**使用SpringFactoriesLoader在应用的classpath中查找并加载所有可用的`ApplicationContextInitializer`。
- **设置监听器(Listener):**使用`SpringFactoriesLoader`在应用的classpath中查找并加载所有可用的`ApplicationListener`。
- 推断并设置main方法的定义类。

先将HelloWorldApplication.class存储在this.primarySources属性中

### 设置应用类型

```
private WebApplicationType deduceWebApplicationType() {
    if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)
            && !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    return WebApplicationType.SERVLET;
}

// 相关常量
private static final String REACTIVE_WEB_ENVIRONMENT_CLASS = "org.springframework."
        + "web.reactive.DispatcherHandler";
private static final String MVC_WEB_ENVIRONMENT_CLASS = "org.springframework."
        + "web.servlet.DispatcherServlet";
private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
        "org.springframework.web.context.ConfigurableWebApplicationContext" };
```

这里主要是通过类加载器判断`REACTIVE`相关的Class是否存在，如果不存在，则web环境即为`SERVLET`类型。这里设置好web环境类型，在后面会根据类型初始化对应环境。大家还记得我们引入的依赖吗？

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

spring-boot-starter-web 的pom又会引入Tomcat和spring-webmvc，如下

```
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-webmvc</artifactId>
  <version>5.0.5.RELEASE</version>
  <scope>compile</scope>
</dependency>
```

 我们来看看spring-webmvc这个jar包下的这个类

```
org.springframework.web.servlet.DispatcherServlet
```

很明显spring-webmvc中存在DispatcherServlet这个类，也就是我们以前SpringMvc的核心Servlet，通过类加载能加载DispatcherServlet这个类，那么我们的应用类型自然就是WebApplicationType.SERVLET

```
public enum WebApplicationType {
    NONE,
    SERVLET,
    REACTIVE;

    private WebApplicationType() {
    }
}
```

### 设置初始化器(Initializer)

```
//设置初始化器(Initializer),最后会调用这些初始化器
setInitializers((Collection) getSpringFactoriesInstances( ApplicationContextInitializer.class));
```

我们先来看看**getSpringFactoriesInstances( ApplicationContextInitializer.class)**

```
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

// 这里的入参type就是ApplicationContextInitializer.class
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // 使用Set保存names来避免重复元素
    Set<String> names = new LinkedHashSet<>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 根据names来进行实例化
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    // 对实例进行排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

这里面首先会根据入参type读取所有的names(是一个String集合)，然后根据这个集合来完成对应的实例化操作：

```
// 入参就是ApplicationContextInitializer.class
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
  String factoryClassName = factoryClass.getName();

  try {
      //从类路径的META-INF/spring.factories中加载所有默认的自动配置类
      Enumeration<URL> urls = classLoader != null?classLoader.getResources("META-INF/spring.factories"):ClassLoader.getSystemResources("META-INF/spring.factories");
      ArrayList result = new ArrayList();

      while(urls.hasMoreElements()) {
          URL url = (URL)urls.nextElement();
          Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
          //获取ApplicationContextInitializer.class的所有值
          String factoryClassNames = properties.getProperty(factoryClassName);
          result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
      }

      return result;
  } catch (IOException var8) {
      throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + "META-INF/spring.factories" + "]", var8);
  }
}
```

这个方法会尝试从类路径的META-INF/spring.factories处读取相应配置文件，然后进行遍历，读取配置文件中Key为：org.springframework.context.ApplicationContextInitializer的value。以spring-boot-autoconfigure这个包为例，它的META-INF/spring.factories部分定义如下所示：

```
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer
```

这两个类名会被读取出来，然后放入到Set<String>集合中，准备开始下面的实例化操作：

```
// parameterTypes: 上一步得到的names集合
private <T> List<T> createSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
        Set<String> names) {
    List<T> instances = new ArrayList<T>(names.size());
    for (String name : names) {
        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            //确认被加载类是ApplicationContextInitializer的子类
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            //反射实例化对象
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            //加入List集合中
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException(
                    "Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```

确认被加载的类确实是org.springframework.context.ApplicationContextInitializer的子类，然后就是得到构造器进行初始化，最后放入到实例列表中。

因此，所谓的初始化器就是org.springframework.context.ApplicationContextInitializer的实现类，这个接口是这样定义的：

```
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

    void initialize(C applicationContext);

}
```

在Spring上下文被刷新之前进行初始化的操作。典型地比如在Web应用中，注册Property Sources或者是激活Profiles。Property Sources比较好理解，就是配置文件。Profiles是Spring为了在不同环境下(如DEV，TEST，PRODUCTION等)，加载不同的配置项而抽象出来的一个实体。

### 设置监听器(Listener)

下面开始设置监听器：

```
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
```

我们还是跟进代码看看**getSpringFactoriesInstances**

```
// 这里的入参type是：org.springframework.context.ApplicationListener.class
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<String>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

可以发现，这个加载相应的类名，然后完成实例化的过程和上面在设置初始化器时如出一辙，同样，还是以spring-boot-autoconfigure这个包中的spring.factories为例，看看相应的Key-Value：

```
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
```

这10个监听器会贯穿springBoot整个生命周期。至此，对于SpringApplication实例的初始化过程就结束了。

### SpringApplication.run方法

完成了SpringApplication实例化，下面开始调用run方法：

```
public ConfigurableApplicationContext run(String... args) {
    // 计时工具
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();

    configureHeadlessProperty();

    // 第一步：获取并启动监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

        // 第二步：根据SpringApplicationRunListeners以及参数来准备环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners,applicationArguments);
        configureIgnoreBeanInfo(environment);

        // 准备Banner打印器 - 就是启动Spring Boot的时候打印在console上的ASCII艺术字体
        Banner printedBanner = printBanner(environment);

        // 第三步：创建Spring容器
        context = createApplicationContext();

        exceptionReporters = getSpringFactoriesInstances(
                SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);

        // 第四步：Spring容器前置处理
        prepareContext(context, environment, listeners, applicationArguments,printedBanner);

        // 第五步：刷新容器
        refreshContext(context);
　　　　 // 第六步：Spring容器后置处理
        afterRefresh(context, applicationArguments);

  　　　 // 第七步：发出结束执行的事件
        listeners.started(context);
        // 第八步：执行Runners
        this.callRunners(context, applicationArguments);
        stopWatch.stop();
        // 返回容器
        return context;
    }
    catch (Throwable ex) {
        handleRunFailure(context, listeners, exceptionReporters, ex);
        throw new IllegalStateException(ex);
    }
}
```

- 第一步：获取并启动监听器
- 第二步：根据SpringApplicationRunListeners以及参数来准备环境
- 第三步：创建Spring容器
- 第四步：Spring容器前置处理
- 第五步：刷新容器
- 第六步：Spring容器后置处理
- 第七步：发出结束执行的事件
- 第八步：执行Runners

 下面具体分析。

#### 第一步：获取并启动监听器

从命名我们就可以知道它是一个监听者，那纵观整个启动流程我们会发现，它其实是用来在整个启动流程中接收不同执行点事件通知的监听者。源码如下：

**获取监听器**

跟进`getRunListeners`方法：

```
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

这里仍然利用了getSpringFactoriesInstances方法来获取实例，大家可以看看前面的这个方法分析，从META-INF/spring.factories中读取Key为org.springframework.boot.**SpringApplicationRunListener**的Values：

```
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

getSpringFactoriesInstances中反射获取实例时会触发`EventPublishingRunListener`的构造函数，我们来看看`EventPublishingRunListener`的构造函数：

```
 public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    private final SpringApplication application;
    private final String[] args;
    //广播器
    private final SimpleApplicationEventMulticaster initialMulticaster;

    public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
        this.initialMulticaster = new SimpleApplicationEventMulticaster();
        Iterator var3 = application.getListeners().iterator();

        while(var3.hasNext()) {
            ApplicationListener<?> listener = (ApplicationListener)var3.next();
            //将上面设置到SpringApplication的十一个监听器全部添加到SimpleApplicationEventMulticaster这个广播器中
            this.initialMulticaster.addApplicationListener(listener);
        }

    }
    //略...
}
```

我们看到**EventPublishingRunListener里面有一个广播器，EventPublishingRunListener 的构造方法将SpringApplication的十一个监听器全部添加到SimpleApplicationEventMulticaster这个广播器中，**我们来看看是如何添加到广播器：

```
 public abstract class AbstractApplicationEventMulticaster implements ApplicationEventMulticaster, BeanClassLoaderAware, BeanFactoryAware {
    //广播器的父类中存放保存监听器的内部内
    private final AbstractApplicationEventMulticaster.ListenerRetriever defaultRetriever = new AbstractApplicationEventMulticaster.ListenerRetriever(false);

    @Override
    public void addApplicationListener(ApplicationListener<?> listener) {
        synchronized (this.retrievalMutex) {
            Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
            if (singletonTarget instanceof ApplicationListener) {
                this.defaultRetriever.applicationListeners.remove(singletonTarget);
            }
            //内部类对象
            this.defaultRetriever.applicationListeners.add(listener);
            this.retrieverCache.clear();
        }
    }

    private class ListenerRetriever {
        //保存所有的监听器
        public final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet();
        public final Set<String> applicationListenerBeans = new LinkedHashSet();
        private final boolean preFiltered;

        public ListenerRetriever(boolean preFiltered) {
            this.preFiltered = preFiltered;
        }

        public Collection<ApplicationListener<?>> getApplicationListeners() {
            LinkedList<ApplicationListener<?>> allListeners = new LinkedList();
            Iterator var2 = this.applicationListeners.iterator();

            while(var2.hasNext()) {
                ApplicationListener<?> listener = (ApplicationListener)var2.next();
                allListeners.add(listener);
            }

            if (!this.applicationListenerBeans.isEmpty()) {
                BeanFactory beanFactory = AbstractApplicationEventMulticaster.this.getBeanFactory();
                Iterator var8 = this.applicationListenerBeans.iterator();

                while(var8.hasNext()) {
                    String listenerBeanName = (String)var8.next();

                    try {
                        ApplicationListener<?> listenerx = (ApplicationListener)beanFactory.getBean(listenerBeanName, ApplicationListener.class);
                        if (this.preFiltered || !allListeners.contains(listenerx)) {
                            allListeners.add(listenerx);
                        }
                    } catch (NoSuchBeanDefinitionException var6) {
                        ;
                    }
                }
            }

            AnnotationAwareOrderComparator.sort(allListeners);
            return allListeners;
        }
    }
    //略...
}
```

上述方法定义在SimpleApplicationEventMulticaster父类AbstractApplicationEventMulticaster中。关键代码为this.defaultRetriever.applicationListeners.add(listener);，这是一个内部类，用来保存所有的监听器。也就是在这一步，将spring.factories中的监听器传递到SimpleApplicationEventMulticaster中。我们现在知道EventPublishingRunListener中有一个广播器SimpleApplicationEventMulticaster，SimpleApplicationEventMulticaster广播器中又存放所有的监听器。

**启动监听器**

我们上面一步通过`getRunListeners`方法获取的监听器为`EventPublishingRunListener，从名字可以看出是启动事件发布监听器，主要用来发布启动事件。`

```
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    private final SpringApplication application;
    private final String[] args;
    private final SimpleApplicationEventMulticaster initialMulticaster;
```

我们先来看看**SpringApplicationRunListener这个接口**

```
package org.springframework.boot;
public interface SpringApplicationRunListener {

    // 在run()方法开始执行时，该方法就立即被调用，可用于在初始化最早期时做一些工作
    void starting();
    // 当environment构建完成，ApplicationContext创建之前，该方法被调用
    void environmentPrepared(ConfigurableEnvironment environment);
    // 当ApplicationContext构建完成时，该方法被调用
    void contextPrepared(ConfigurableApplicationContext context);
    // 在ApplicationContext完成加载，但没有被刷新前，该方法被调用
    void contextLoaded(ConfigurableApplicationContext context);
    // 在ApplicationContext刷新并启动后，CommandLineRunners和ApplicationRunner未被调用前，该方法被调用
    void started(ConfigurableApplicationContext context);
    // 在run()方法执行完成前该方法被调用
    void running(ConfigurableApplicationContext context);
    // 当应用运行出错时该方法被调用
    void failed(ConfigurableApplicationContext context, Throwable exception);
}
```

SpringApplicationRunListener接口在Spring Boot 启动初始化的过程中各种状态时执行，我们也可以添加自己的监听器，在SpringBoot初始化时监听事件执行自定义逻辑，我们先来看看SpringBoot启动时第一个启动事件listeners.starting()：

```
@Override
public void starting() {
    //关键代码，先创建application启动事件`ApplicationStartingEvent`
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```

这里先创建了一个启动事件**ApplicationStartingEvent**，我们继续跟进SimpleApplicationEventMulticaster，有个核心方法：

```
@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    //通过事件类型ApplicationStartingEvent获取对应的监听器
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        //获取线程池，如果为空则同步处理。这里线程池为空，还未没初始化。
        Executor executor = getTaskExecutor();
        if (executor != null) {
            //异步发送事件
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            //同步发送事件
            invokeListener(listener, event);
        }
    }
}
```

这里会根据事件类型`ApplicationStartingEvent`获取对应的监听器，在容器启动之后执行响应的动作，有如下4种监听器：

```
0 = {LoggingApplicationListener@1339} 
1 = {BackgroundPreinitializer@1340} 
2 = {DelegatingApplicationListener@1341} 
3 = {LiquibaseServiceLocatorApplicationListener@1342} 
```

我们选择springBoot 的日志监听器来进行讲解，核心代码如下：

```
@Override
public void onApplicationEvent(ApplicationEvent event) {
    //在springboot启动的时候
    if (event instanceof ApplicationStartedEvent) {
        onApplicationStartedEvent((ApplicationStartedEvent) event);
    }
    //springboot的Environment环境准备完成的时候
    else if (event instanceof ApplicationEnvironmentPreparedEvent) {
        onApplicationEnvironmentPreparedEvent(
                (ApplicationEnvironmentPreparedEvent) event);
    }
    //在springboot容器的环境设置完成以后
    else if (event instanceof ApplicationPreparedEvent) {
        onApplicationPreparedEvent((ApplicationPreparedEvent) event);
    }
    //容器关闭的时候
    else if (event instanceof ContextClosedEvent && ((ContextClosedEvent) event)
            .getApplicationContext().getParent() == null) {
        onContextClosedEvent();
    }
    //容器启动失败的时候
    else if (event instanceof ApplicationFailedEvent) {
        onApplicationFailedEvent();
    }
}
```

因为我们的事件类型为`ApplicationEvent`，所以会执行`onApplicationStartedEvent((ApplicationStartedEvent) event);`。springBoot会在运行过程中的不同阶段，发送各种事件，来执行对应监听器的对应方法。

对于开发者来说，基本没有什么常见的场景要求我们必须实现一个自定义的SpringApplicationRunListener，即使是SpringBoot中也只默认实现了一个`org.springframework.boot.context.eventEventPublishingRunListener`, 用来在SpringBoot的整个启动流程中的不同时间点发布不同类型的应用事件(SpringApplicationEvent)。那些对这些应用事件感兴趣的ApplicationListener可以接受并处理(这也解释了为什么在SpringApplication实例化的时候加载了一批ApplicationListener，但在run方法执行的过程中并没有被使用)。

　　如果我们真的在实际场景中自定义实现SpringApplicationRunListener,有一个点需要注意：**任何一个SpringApplicationRunListener实现类的构造方法都需要有两个构造参数，一个参数的类型就是我们的org.springframework.boot.SpringApplication,另外一个参数就是args参数列表的String[]**:

```
public class DemoSpringApplicationRunListener implements SpringApplicationRunListener {
    private final SpringApplication application;
    private final String[] args;
    public DemoSpringApplicationRunListener(SpringApplication sa, String[] args) {
        this.application = sa;
        this.args = args;
    }

    @Override
    public void starting() {
        System.out.println("自定义starting");
    }

    @Override
    public void environmentPrepared(ConfigurableEnvironment environment) {
        System.out.println("自定义environmentPrepared");
    }

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        System.out.println("自定义contextPrepared");
    }

    @Override
    public void contextLoaded(ConfigurableApplicationContext context) {
        System.out.println("自定义contextLoaded");
    }

    @Override
    public void finished(ConfigurableApplicationContext context, Throwable exception) {
        System.out.println("自定义finished");
    }
}
```

接着，我们还要满足SpringFactoriesLoader的约定，在当前SpringBoot项目的classpath下新建META-INF目录，并在该目录下新建spring.fatories文件，文件内容如下:

```properties
org.springframework.boot.SpringApplicationRunListener=\
    com.springbootdemo.DemoSpringApplicationRunListener
```

#### 第二步：环境构建

```
ConfigurableEnvironment environment = prepareEnvironment(listeners,applicationArguments);
```

跟进去该方法：

```
private ConfigurableEnvironment prepareEnvironment(
        SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    //获取对应的ConfigurableEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    //配置
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    //发布环境已准备事件，这是第二次发布事件
    listeners.environmentPrepared(environment);
    bindToSpringApplication(environment);
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

来看一下`getOrCreateEnvironment()`方法，前面已经提到，`environment`已经被设置了`servlet`类型，所以这里创建的是环境对象是`StandardServletEnvironment`。

```
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    if (this.webApplicationType == WebApplicationType.SERVLET) {
        return new StandardServletEnvironment();
    }
    return new StandardEnvironment();
}
```

接下来看一下`listeners.environmentPrepared(environment);`，上面已经提到了，这里是第二次发布事件。什么事件呢？来看一下根据事件类型获取到的监听器：

```
0 = {ConfigFileApplicationListener@1747} 
1 = {AnsiOutputApplicationListener@1748} 
2 = {LoggingApplicationListener@1339} 
3 = {ClasspathLoggingApplicationListener@1749} 
4 = {BackgroundPreinitializer@1340} 
5 = {DelegatingApplicationListener@1341} 
6 = {FileEncodingApplicationListener@1750} 
```

主要来看一下**`ConfigFileApplicationListener`**，该监听器非常核心，主要用来处理项目配置。项目中的 properties 和yml文件都是其内部类所加载。具体来看一下：

```
   private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
        List<EnvironmentPostProcessor> postProcessors = this.loadPostProcessors();
        postProcessors.add(this);
        AnnotationAwareOrderComparator.sort(postProcessors);
        Iterator var3 = postProcessors.iterator();

        while(var3.hasNext()) {
            EnvironmentPostProcessor postProcessor = (EnvironmentPostProcessor)var3.next();
            postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
        }

    }
```

EnvironmentPostProcessor级别元素为

```
0 = {SystemEnvironmentPropertySourceEnvironmentPostProcessor@1614} 
1 = {SpringApplicationJsonEnvironmentPostProcessor@1615} 
2 = {CloudFoundryVcapEnvironmentPostProcessor@1616} 
3 = {ConfigFileApplicationListener@1589} 
4 = {SpringBootTestRandomPortEnvironmentPostProcessor@1617} 
5 = {DebugAgentEnvironmentPostProcessor@1618} 
```

首先还是会去读`spring.factories` 文件，`List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();`获取的处理类有以下四种：

```
# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor，
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor，
org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor
```

在执行完上述三个监听器流程后，`ConfigFileApplicationListener`会执行该类本身的逻辑。由其内部类`Loader`加载项目制定路径下的配置文件：

```
private static final String DEFAULT_SEARCH_LOCATIONS = "classpath:/,classpath:/config/,file:./,file:./config/";
```

至此，项目的变量配置已全部加载完毕，来一起看一下：

SpringApplication类中

```
    private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
        ConfigurableEnvironment environment = this.getOrCreateEnvironment();
        this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
        ConfigurationPropertySources.attach((Environment)environment);
        listeners.environmentPrepared((ConfigurableEnvironment)environment);
        this.bindToSpringApplication((ConfigurableEnvironment)environment);
        if (!this.isCustomEnvironment) {
            environment = (new EnvironmentConverter(this.getClassLoader())).convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
        }

        ConfigurationPropertySources.attach((Environment)environment);
        return (ConfigurableEnvironment)environment;
    }
```

this.bindToSpringApplication((ConfigurableEnvironment)environment);中的propertySourceList

```
propertySourceList = {CopyOnWriteArrayList@2668}  size = 7
 0 = {ConfigurationPropertySourcesPropertySource@2670} "ConfigurationPropertySourcesPropertySource {name='configurationProperties'}"
 1 = {PropertySource$StubPropertySource@2671} "StubPropertySource {name='servletConfigInitParams'}"
 2 = {PropertySource$StubPropertySource@2672} "StubPropertySource {name='servletContextInitParams'}"
 3 = {PropertiesPropertySource@2673} "PropertiesPropertySource {name='systemProperties'}"
 4 = {SystemEnvironmentPropertySourceEnvironmentPostProcessor$OriginAwareSystemEnvironmentPropertySource@2674} "OriginAwareSystemEnvironmentPropertySource {name='systemEnvironment'}"
 5 = {RandomValuePropertySource@2675} "RandomValuePropertySource {name='random'}"
 6 = {OriginTrackedMapPropertySource@2676} "OriginTrackedMapPropertySource {name='applicationConfig: [classpath:/application.properties]'}"
```

这里一共7个配置文件，取值顺序由上到下。也就是说前面的配置变量会覆盖后面同名的配置变量。项目配置变量的时候需要注意这点。



#### 第三步：创建容器

```
context = createApplicationContext();
```

继续跟进该方法：

```
public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext";
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
            case SERVLET:
                contextClass = Class.forName(DEFAULT_WEB_CONTEXT_CLASS);
                break;
            case REACTIVE:
                contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                break;
            default:
                contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, "
                            + "please specify an ApplicationContextClass",
                    ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

这里创建容器的类型 还是根据`webApplicationType`进行判断的，该类型为`SERVLET`类型，所以会通过反射装载对应的字节码，也就是**AnnotationConfigServletWebServerApplicationContext**



#### **第四步：Spring容器前置处理**

这一步主要是在容器刷新之前的准备动作。包含一个非常关键的操作：**将启动类注入容器，为后续开启自动化配置奠定基础。**

```
prepareContext(context, environment, listeners, applicationArguments,printedBanner);
```

继续跟进该方法：

```
private void prepareContext(ConfigurableApplicationContext context,
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments, Banner printedBanner) {
    //设置容器环境，包括各种变量
    context.setEnvironment(environment);
    //执行容器后置处理
    postProcessApplicationContext(context);
    //执行容器中的ApplicationContextInitializer（包括 spring.factories和自定义的实例）
    applyInitializers(context);
　　//发送容器已经准备好的事件，通知各监听器
    listeners.contextPrepared(context);

    //注册启动参数bean，这里将容器指定的参数封装成bean，注入容器
    context.getBeanFactory().registerSingleton("springApplicationArguments",
            applicationArguments);
    //设置banner
    if (printedBanner != null) {
        context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
    }
    //获取我们的启动类指定的参数，可以是多个
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    //加载我们的启动类，将启动类注入容器
    load(context, sources.toArray(new Object[0]));
    //发布容器已加载事件。
    listeners.contextLoaded(context);
}
```

**调用初始化器**

```
protected void applyInitializers(ConfigurableApplicationContext context) {
    // 1. 从SpringApplication类中的initializers集合获取所有的ApplicationContextInitializer
    for (ApplicationContextInitializer initializer : getInitializers()) {
        // 2. 循环调用ApplicationContextInitializer中的initialize方法
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
                initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```

这里终于用到了在创建SpringApplication实例时设置的初始化器了，依次对它们进行遍历，并调用initialize方法。我们也可以自定义初始化器，并实现**initialize**方法，然后放入META-INF/spring.factories配置文件中Key为：org.springframework.context.ApplicationContextInitializer的value中，这里我们自定义的初始化器就会被调用，是我们项目初始化的一种方式

**加载启动指定类（重点）**

大家先回到文章最开始看看，在创建**SpringApplication**实例时，先将**HelloWorldMainApplication.class**存储在this.primarySources属性中，现在就是用到这个属性的时候了，我们来看看**getAllSources（）**

```
public Set<Object> getAllSources() {
    Set<Object> allSources = new LinkedHashSet();
    if (!CollectionUtils.isEmpty(this.primarySources)) {
        //获取primarySources属性，也就是之前存储的HelloWorldMainApplication.class
        allSources.addAll(this.primarySources);
    }

    if (!CollectionUtils.isEmpty(this.sources)) {
        allSources.addAll(this.sources);
    }

    return Collections.unmodifiableSet(allSources);
}
```

很明显，获取了this.primarySources属性，也就是我们的启动类**HelloWorldMainApplication.class，**我们接着看load(context, sources.toArray(new Object[0]));

```
protected void load(ApplicationContext context, Object[] sources) {
    BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    loader.load();
}

private int load(Class<?> source) {
    if (isGroovyPresent()
            && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
        // Any GroovyLoaders added in beans{} DSL can contribute beans here
        GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source,
                GroovyBeanDefinitionSource.class);
        load(loader);
    }
    if (isComponent(source)) {
        //以注解的方式，将启动类bean信息存入beanDefinitionMap，也就是将HelloWorldMainApplication.class存入了beanDefinitionMap
        this.annotatedReader.register(source);
        return 1;
    }
    return 0;
}
```

启动类HelloWorldApplication.class被加载到 beanDefinitionMap中，后续该启动类将作为开启自动化配置的入口。

**通知监听器，容器已准备就绪**

```
listeners.contextLoaded(context);
```

主还是针对一些日志等监听器的响应处理。

#### 第五步：刷新容器

执行到这里，springBoot相关的处理工作已经结束，接下的工作就交给了spring。我们来看看**refreshContext(context);**

```
protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    //调用创建的容器applicationContext中的refresh()方法
    ((AbstractApplicationContext)applicationContext).refresh();
}
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        /**
         * 刷新上下文环境
         */
        prepareRefresh();

        /**
         * 初始化BeanFactory，解析XML，相当于之前的XmlBeanFactory的操作，
         */
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        /**
         * 为上下文准备BeanFactory，即对BeanFactory的各种功能进行填充，如常用的注解@Autowired @Qualifier等
         * 添加ApplicationContextAwareProcessor处理器
         * 在依赖注入忽略实现*Aware的接口，如EnvironmentAware、ApplicationEventPublisherAware等
         * 注册依赖，如一个bean的属性中含有ApplicationEventPublisher(beanFactory)，则会将beanFactory的实例注入进去
         */
        prepareBeanFactory(beanFactory);

        try {
            /**
             * 提供子类覆盖的额外处理，即子类处理自定义的BeanFactoryPostProcess
             */
            postProcessBeanFactory(beanFactory);

            /**
             * 激活各种BeanFactory处理器,包括BeanDefinitionRegistryBeanFactoryPostProcessor和普通的BeanFactoryPostProcessor
             * 执行对应的postProcessBeanDefinitionRegistry方法 和  postProcessBeanFactory方法
             */
            invokeBeanFactoryPostProcessors(beanFactory);

            /**
             * 注册拦截Bean创建的Bean处理器，即注册BeanPostProcessor，不是BeanFactoryPostProcessor，注意两者的区别
             * 注意，这里仅仅是注册，并不会执行对应的方法，将在bean的实例化时执行对应的方法
             */
            registerBeanPostProcessors(beanFactory);

            /**
             * 初始化上下文中的资源文件，如国际化文件的处理等
             */
            initMessageSource();

            /**
             * 初始化上下文事件广播器，并放入applicatioEventMulticaster,如ApplicationEventPublisher
             */
            initApplicationEventMulticaster();

            /**
             * 给子类扩展初始化其他Bean
             */
            onRefresh();

            /**
             * 在所有bean中查找listener bean，然后注册到广播器中
             */
            registerListeners();

            /**
             * 设置转换器
             * 注册一个默认的属性值解析器
             * 冻结所有的bean定义，说明注册的bean定义将不能被修改或进一步的处理
             * 初始化剩余的非惰性的bean，即初始化非延迟加载的bean
             */
            finishBeanFactoryInitialization(beanFactory);

            /**
             * 通过spring的事件发布机制发布ContextRefreshedEvent事件，以保证对应的监听器做进一步的处理
             * 即对那种在spring启动后需要处理的一些类，这些类实现了ApplicationListener<ContextRefreshedEvent>，
             * 这里就是要触发这些类的执行(执行onApplicationEvent方法)
             * 另外，spring的内置Event有ContextClosedEvent、ContextRefreshedEvent、ContextStartedEvent、ContextStoppedEvent、RequestHandleEvent
             * 完成初始化，通知生命周期处理器lifeCycleProcessor刷新过程，同时发出ContextRefreshEvent通知其他人
             */
            finishRefresh();
        }

        finally {
    
            resetCommonCaches();
        }
    }
}
```

`refresh`方法在spring整个源码体系中举足轻重，是实现 ioc 和 aop的关键。我之前也有文章分析过这个过程，大家可以去看看



#### 第六步：Spring容器后置处理

```
protected void afterRefresh(ConfigurableApplicationContext context,
        ApplicationArguments args) {
}
```

扩展接口，设计模式中的模板方法，默认为空实现。如果有自定义需求，可以重写该方法。比如打印一些启动结束log，或者一些其它后置处理。



#### 第七步：发出结束执行的事件

```
public void started(ConfigurableApplicationContext context) {
    //这里就是获取的EventPublishingRunListener
    Iterator var2 = this.listeners.iterator();

    while(var2.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
        //执行EventPublishingRunListener的started方法
        listener.started(context);
    }
}

public void started(ConfigurableApplicationContext context) {
    //创建ApplicationStartedEvent事件，并且发布事件
    //我们看到是执行的ConfigurableApplicationContext这个容器的publishEvent方法，和前面的starting是不同的
    context.publishEvent(new ApplicationStartedEvent(this.application, this.args, context));
}
```

获取EventPublishingRunListener监听器，并执行其started方法，并且将创建的Spring容器传进去了，创建一个ApplicationStartedEvent事件，并执行ConfigurableApplicationContext 的publishEvent方法，也就是说这里是在Spring容器中发布事件，并不是在SpringApplication中发布事件，和前面的starting是不同的，前面的starting是直接向SpringApplication中的11个监听器发布启动事件。



#### 第八步：执行Runners

我们再来看看最后一步**callRunners**(context, applicationArguments);

```
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<Object>();
    //获取容器中所有的ApplicationRunner的Bean实例
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    //获取容器中所有的CommandLineRunner的Bean实例
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<Object>(runners)) {
        if (runner instanceof ApplicationRunner) {
            //执行ApplicationRunner的run方法
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            //执行CommandLineRunner的run方法
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

我们也可以自定义一些ApplicationRunner或者CommandLineRunner，实现其run方法，并注入到Spring容器中,在SpringBoot启动完成后，会执行所有的runner的run方法

## 启动时初始化数据

在我们用 springboot 搭建项目的时候，有时候会碰到在项目启动时初始化一些操作的需求 ，针对这种需求 spring boot为我们提供了以下几种方案供我们选择:

- `ApplicationRunner `与 `CommandLineRunner `接口
- `Spring容器初始化时InitializingBean接口和@PostConstruct`
- Spring的事件机制

### ApplicationRunner与CommandLineRunner

我们可以实现 `ApplicationRunner `或 `CommandLineRunner `接口， 这两个接口工作方式相同，都只提供单一的run方法，该方法在SpringApplication.run(…)完成之前调用，不知道大家还对我上一篇文章结尾有没有印象，我们先来看看这两个接口

```
public interface ApplicationRunner {
    void run(ApplicationArguments var1) throws Exception;
}

public interface CommandLineRunner {
    void run(String... var1) throws Exception;
}
```

都只提供单一的run方法，接下来我们来看看具体的使用

#### ApplicationRunner

构造一个类实现ApplicationRunner接口

```
//需要加入到Spring容器中
@Component
public class ApplicationRunnerTest implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("ApplicationRunner");
    }
}
```

很简单，首先要使用**@Component**将实现类加入到Spring容器中，为什么要这样做我们待会再看，然后实现其run方法实现自己的初始化数据逻辑就可以了

#### CommandLineRunner

对于这两个接口而言，我们可以通过Order注解或者使用Ordered接口来指定调用顺序， `@Order() `中的值越小，优先级越高

```
//需要加入到Spring容器中
@Component
@Order(1)
public class CommandLineRunnerTest implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        System.out.println("CommandLineRunner...");
    }
}
```

同样需要加入到Spring容器中，CommandLineRunner的参数是最原始的参数，没有进行任何处理，ApplicationRunner的参数是ApplicationArguments,是对原始参数的进一步封装

CommandLineRunner并不是Spring框架原有的概念，它属于SpringBoot应用特定的回调扩展接口：

```
public interface CommandLineRunner {
    /**
     * Callback used to run the bean.
     * @param args incoming main method arguments
     * @throws Exception on error
     */
    void run(String... args) throws Exception;

}
```

关于这货，我们需要关注的点有两个：

1. 所有CommandLineRunner的执行时间点是在SpringBoot应用的Application完全初始化工作之后(这里我们可以认为是SpringBoot应用启动类main方法执行完成之前的最后一步)。
2. 当前SpringBoot应用的ApplicationContext中的所有CommandLinerRunner都会被加载执行(无论是手动注册还是被自动扫描注册到IoC容器中)。

跟其他几个扩展点接口类型相似，我们建议CommandLineRunner的实现类使用@org.springframework.core.annotation.Order进行标注或者实现`org.springframework.core.Ordered`接口，便于对他们的执行顺序进行排序调整，这是非常有必要的，因为我们不希望不合适的CommandLineRunner实现类阻塞了后面其他CommandLineRunner的执行。**这个接口非常有用和重要，我们需要重点关注。**

#### 源码分析

SpringApplication.run方法的最后一步第八步：执行Runners源码如下：

```
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<Object>();
    //获取容器中所有的ApplicationRunner的Bean实例
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    //获取容器中所有的CommandLineRunner的Bean实例
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<Object>(runners)) {
        if (runner instanceof ApplicationRunner) {
            //执行ApplicationRunner的run方法
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            //执行CommandLineRunner的run方法
            callRunner((CommandLineRunner) runner, args);
        }
    }
}
```

很明显，是直接从Spring容器中获取ApplicationRunner和CommandLineRunner的实例，并调用其run方法，这也就是为什么我要使用@Component将ApplicationRunner和CommandLineRunner接口的实现类加入到Spring容器中了。

## InitializingBean

在spring初始化bean的时候，如果bean实现了 `InitializingBean `接口，在对象的所有属性被初始化后之后才会调用afterPropertiesSet()方法

```
@Component
public class InitialingzingBeanTest implements InitializingBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean..");
    }
}
```

我们可以看出spring初始化bean肯定会在 ApplicationRunner和CommandLineRunner接口调用之前。

## @PostConstruct

```
@Component
public class PostConstructTest {

    @PostConstruct
    public void postConstruct() {
        System.out.println("init...");
    }
}
```

我们可以看到，只用在方法上添加**@PostConstruct注解，**并将类注入到Spring容器中就可以了。我们来看看**@PostConstruct注解的方法是何时执行的**

在Spring初始化bean时，对bean的实例赋值时，populateBean方法下面有一个initializeBean(beanName, exposedObject, mbd)方法，这个就是用来执行用户设定的初始化操作。我们看下方法体：

```
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            // 激活 Aware 方法
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        // 对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 后处理器
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 激活用户自定义的 init 方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 后处理器
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

我们看到会先执行后处理器然后执行**invokeInitMethods方法**，我们来看下applyBeanPostProcessorsBeforeInitialization

```
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)  
        throws BeansException {  

    Object result = existingBean;  
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {  
        result = beanProcessor.postProcessBeforeInitialization(result, beanName);  
        if (result == null) {  
            return result;  
        }  
    }  
    return result;  
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

获取容器中所有的后置处理器，循环调用后置处理器的**postProcessBeforeInitialization方法，这里我们来看一个BeanPostProcessor**

```
public class CommonAnnotationBeanPostProcessor extends InitDestroyAnnotationBeanPostProcessor implements InstantiationAwareBeanPostProcessor, BeanFactoryAware, Serializable {
    public CommonAnnotationBeanPostProcessor() {
        this.setOrder(2147483644);
        //设置初始化参数为PostConstruct.class
        this.setInitAnnotationType(PostConstruct.class);
        this.setDestroyAnnotationType(PreDestroy.class);
        this.ignoreResourceType("javax.xml.ws.WebServiceContext");
    }
    //略...
}
```

在构造器中设置了一个属性为**PostConstruct.class，**再次观察CommonAnnotationBeanPostProcessor这个类，它继承自InitDestroyAnnotationBeanPostProcessor。InitDestroyAnnotationBeanPostProcessor顾名思义，就是在Bean初始化和销毁的时候所作的一个前置/后置处理器。查看InitDestroyAnnotationBeanPostProcessor类下的postProcessBeforeInitialization方法：

```
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
   LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());  
   try {  
       metadata.invokeInitMethods(bean, beanName);  
   }  
   catch (InvocationTargetException ex) {  
       throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());  
   }  
   catch (Throwable ex) {  
       throw new BeanCreationException(beanName, "Couldn't invoke init method", ex);  
   }  
    return bean;  
}  

private LifecycleMetadata buildLifecycleMetadata(final Class clazz) {  
       final LifecycleMetadata newMetadata = new LifecycleMetadata();  
       final boolean debug = logger.isDebugEnabled();  
       ReflectionUtils.doWithMethods(clazz, new ReflectionUtils.MethodCallback() {  
           public void doWith(Method method) {  
              if (initAnnotationType != null) {  
                   //判断clazz中的methon是否有initAnnotationType注解，也就是PostConstruct.class注解
                  if (method.getAnnotation(initAnnotationType) != null) {  
                     //如果有就将方法添加进LifecycleMetadata中
                     newMetadata.addInitMethod(method);  
                     if (debug) {  
                         logger.debug("Found init method on class [" + clazz.getName() + "]: " + method);  
                     }  
                  }  
              }  
              if (destroyAnnotationType != null) {  
                    //判断clazz中的methon是否有destroyAnnotationType注解
                  if (method.getAnnotation(destroyAnnotationType) != null) {  
                     newMetadata.addDestroyMethod(method);  
                     if (debug) {  
                         logger.debug("Found destroy method on class [" + clazz.getName() + "]: " + method);  
                     }  
                  }  
              }  
           }  
       });  
       return newMetadata;  
} 
```

在这里会去判断某方法是否有**PostConstruct.class注解**，如果有，则添加到init/destroy队列中，后续一一执行。**@PostConstruct注解的方法会在此时执行，我们接着来看invokeInitMethods**

```
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
        throws Throwable {

    // 是否实现 InitializingBean
    // 如果实现了 InitializingBean 接口，则只掉调用bean的 afterPropertiesSet()
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isDebugEnabled()) {
            logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((InitializingBean) bean).afterPropertiesSet();
                    return null;
                }, getAccessControlContext());
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            // 直接调用 afterPropertiesSet()
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    if (mbd != null && bean.getClass() != NullBean.class) {
        // 判断是否指定了 init-method()，
        // 如果指定了 init-method()，则再调用制定的init-method
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            // 利用反射机制执行
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

首先检测当前 bean 是否实现了 InitializingBean 接口，如果实现了则调用其 `afterPropertiesSet()`，然后再检查是否也指定了 `init-method()`，如果指定了则通过反射机制调用指定的 `init-method()`。

我们也可以发现**@PostConstruct会在实现 InitializingBean 接口的afterPropertiesSet()方法之前执行**

## 内置Servlet容器源码分析（Tomcat）

Spring Boot默认使用Tomcat作为嵌入式的Servlet容器，只要引入了spring-boot-start-web依赖，则默认是用Tomcat作为Servlet容器：

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

## Servlet容器的使用

### 默认servlet容器

我们看看spring-boot-starter-web这个starter中有什么

```
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>2.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-json</artifactId>
      <version>2.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
      <version>2.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
      <version>2.2.0.RELEASE</version>
      <scope>compile</scope>
      <exclusions>
        <exclusion>
          <artifactId>tomcat-embed-el</artifactId>
          <groupId>org.apache.tomcat.embed</groupId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-web</artifactId>
      <version>5.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>5.2.0.RELEASE</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
```

核心就是引入了tomcat和SpringMvc，我们先来看tomcat

Spring Boot默认支持Tomcat，Jetty，和Undertow作为底层容器。

org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory

而Spring Boot默认使用Tomcat，一旦引入spring-boot-starter-web模块，就默认使用Tomcat容器。

### 切换servlet容器

那如果我么想切换其他Servlet容器呢，只需如下两步：

- 将tomcat依赖移除掉
- 引入其他Servlet容器依赖

引入jetty：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <!--移除spring-boot-starter-web中的tomcat-->
            <artifactId>spring-boot-starter-tomcat</artifactId>
            <groupId>org.springframework.boot</groupId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <!--引入jetty-->
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### Servlet容器自动配置原理

#### EmbeddedServletContainerAutoConfiguration

其中**EmbeddedServletContainerAutoConfiguration**是嵌入式Servlet容器的自动配置类，该类在**spring-boot-autoconfigure.jar中的web模块**可以找到。

```
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration,
```

我们可以看到**EmbeddedServletContainerAutoConfiguration被配置在spring.factories中，**SpringBoot自动配置将EmbeddedServletContainerAutoConfiguration配置类加入到IOC容器中，接着我们来具体看看这个配置类：

```
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration
@ConditionalOnWebApplication// 在Web环境下才会起作用
@Import(BeanPostProcessorsRegistrar.class)// 会Import一个内部类BeanPostProcessorsRegistrar
public class EmbeddedServletContainerAutoConfiguration {

    @Configuration
    // Tomcat类和Servlet类必须在classloader中存在
    // 文章开头我们已经导入了web的starter，其中包含tomcat和SpringMvc
    // 那么classPath下会存在Tomcat.class和Servlet.class
    @ConditionalOnClass({ Servlet.class, Tomcat.class })
    // 当前Spring容器中不存在EmbeddedServletContainerFactory类型的实例
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
    public static class EmbeddedTomcat {

        @Bean
        public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
            // 上述条件注解成立的话就会构造TomcatEmbeddedServletContainerFactory这个EmbeddedServletContainerFactory
            return new TomcatEmbeddedServletContainerFactory();
        }
    }
    
    @Configuration
    @ConditionalOnClass({ Servlet.class, Server.class, Loader.class,
            WebAppContext.class })
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
    public static class EmbeddedJetty {

        @Bean
        public JettyEmbeddedServletContainerFactory jettyEmbeddedServletContainerFactory() {
            return new JettyEmbeddedServletContainerFactory();
        }

    }
    
    @Configuration
    @ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
    public static class EmbeddedUndertow {

        @Bean
        public UndertowEmbeddedServletContainerFactory undertowEmbeddedServletContainerFactory() {
            return new UndertowEmbeddedServletContainerFactory();
        }

    }
    
    //other code...
}
```

在这个自动配置类中配置了三个容器工厂的Bean，分别是：

- **TomcatEmbeddedServletContainerFactory**
- **JettyEmbeddedServletContainerFactory**
- **UndertowEmbeddedServletContainerFactory**

这里以大家熟悉的Tomcat为例，首先Spring Boot会判断当前环境中是否引入了Servlet和Tomcat依赖，并且当前容器中没有自定义的**EmbeddedServletContainerFactory**的情况下，则创建Tomcat容器工厂。其他Servlet容器工厂也是同样的道理。

#### EmbeddedServletContainerFactory

- 嵌入式Servlet容器工厂

```
public interface EmbeddedServletContainerFactory {

    EmbeddedServletContainer getEmbeddedServletContainer( ServletContextInitializer... initializers);
}
```

内部只有一个方法，用于获取嵌入式的Servlet容器。

该工厂接口主要有三个实现类，分别对应三种嵌入式Servlet容器的工厂类，如图所示：

#### TomcatEmbeddedServletContainerFactory

以Tomcat容器工厂TomcatEmbeddedServletContainerFactory类为例：

```
public class TomcatEmbeddedServletContainerFactory extends AbstractEmbeddedServletContainerFactory implements ResourceLoaderAware {
    
    //other code...
    
    @Override
    public EmbeddedServletContainer getEmbeddedServletContainer( ServletContextInitializer... initializers) {
        //创建一个Tomcat
        Tomcat tomcat = new Tomcat();
        
       //配置Tomcat的基本环节
        File baseDir = (this.baseDirectory != null ? this.baseDirectory: createTempDir("tomcat"));
        tomcat.setBaseDir(baseDir.getAbsolutePath());
        Connector connector = new Connector(this.protocol);
        tomcat.getService().addConnector(connector);
        customizeConnector(connector);
        tomcat.setConnector(connector);
        tomcat.getHost().setAutoDeploy(false);
        configureEngine(tomcat.getEngine());
        for (Connector additionalConnector : this.additionalTomcatConnectors) {
            tomcat.getService().addConnector(additionalConnector);
        }
        prepareContext(tomcat.getHost(), initializers);
        
        //包装tomcat对象，返回一个嵌入式Tomcat容器，内部会启动该tomcat容器
        return getTomcatEmbeddedServletContainer(tomcat);
    }
}
```

首先会创建一个Tomcat的对象，并设置一些属性配置，最后调用**getTomcatEmbeddedServletContainer(tomcat)方法，内部会启动tomcat，**我们来看看：

```
protected TomcatEmbeddedServletContainer getTomcatEmbeddedServletContainer(
    Tomcat tomcat) {
    return new TomcatEmbeddedServletContainer(tomcat, getPort() >= 0);
}
```

该函数很简单，就是来创建Tomcat容器并返回。看看TomcatEmbeddedServletContainer类：

```
public class TomcatEmbeddedServletContainer implements EmbeddedServletContainer {

    public TomcatEmbeddedServletContainer(Tomcat tomcat, boolean autoStart) {
        Assert.notNull(tomcat, "Tomcat Server must not be null");
        this.tomcat = tomcat;
        this.autoStart = autoStart;
        
        //初始化嵌入式Tomcat容器，并启动Tomcat
        initialize();
    }
    
    private void initialize() throws EmbeddedServletContainerException {
        TomcatEmbeddedServletContainer.logger
                .info("Tomcat initialized with port(s): " + getPortsDescription(false));
        synchronized (this.monitor) {
            try {
                addInstanceIdToEngineName();
                try {
                    final Context context = findContext();
                    context.addLifecycleListener(new LifecycleListener() {

                        @Override
                        public void lifecycleEvent(LifecycleEvent event) {
                            if (context.equals(event.getSource())
                                    && Lifecycle.START_EVENT.equals(event.getType())) {
                                // Remove service connectors so that protocol
                                // binding doesn't happen when the service is
                                // started.
                                removeServiceConnectors();
                            }
                        }

                    });

                    // Start the server to trigger initialization listeners
                    //启动tomcat
                    this.tomcat.start();

                    // We can re-throw failure exception directly in the main thread
                    rethrowDeferredStartupExceptions();

                    try {
                        ContextBindings.bindClassLoader(context, getNamingToken(context),
                                getClass().getClassLoader());
                    }
                    catch (NamingException ex) {
                        // Naming is not enabled. Continue
                    }

                    // Unlike Jetty, all Tomcat threads are daemon threads. We create a
                    // blocking non-daemon to stop immediate shutdown
                    startDaemonAwaitThread();
                }
                catch (Exception ex) {
                    containerCounter.decrementAndGet();
                    throw ex;
                }
            }
            catch (Exception ex) {
                stopSilently();
                throw new EmbeddedServletContainerException(
                        "Unable to start embedded Tomcat", ex);
            }
        }
    }
}
```

到这里就启动了嵌入式的Servlet容器，其他容器类似。

### Servlet容器启动原理

#### SpringBoot启动过程

我们回顾一下前面讲解的SpringBoot启动过程，也就是run方法：

```
public ConfigurableApplicationContext run(String... args) {
    // 计时工具
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();

    configureHeadlessProperty();

    // 第一步：获取并启动监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

        // 第二步：根据SpringApplicationRunListeners以及参数来准备环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners,applicationArguments);
        configureIgnoreBeanInfo(environment);

        // 准备Banner打印器 - 就是启动Spring Boot的时候打印在console上的ASCII艺术字体
        Banner printedBanner = printBanner(environment);

        // 第三步：创建Spring容器
        context = createApplicationContext();

        exceptionReporters = getSpringFactoriesInstances(
                SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);

        // 第四步：Spring容器前置处理
        prepareContext(context, environment, listeners, applicationArguments,printedBanner);

        // 第五步：刷新容器
        refreshContext(context);

　　　　 // 第六步：Spring容器后置处理
        afterRefresh(context, applicationArguments);

  　　　 // 第七步：发出结束执行的事件
        listeners.started(context);
        // 第八步：执行Runners
        this.callRunners(context, applicationArguments);
        stopWatch.stop();
        // 返回容器
        return context;
    }
    catch (Throwable ex) {
        handleRunFailure(context, listeners, exceptionReporters, ex);
        throw new IllegalStateException(ex);
    }
}
```

我们回顾一下**第三步：创建Spring容器**

```
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
            + "annotation.AnnotationConfigApplicationContext";

public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework."
            + "boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext";

protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            //根据应用环境，创建不同的IOC容器
            contextClass = Class.forName(this.webEnvironment
                                         ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
}
```

创建IOC容器，如果是web应用，则创建AnnotationConfigEmbeddedWebApplicationContext的IOC容器；如果不是，则创建AnnotationConfigApplicationContext的IOC容器；很明显我们创建的容器是AnnotationConfigEmbeddedWebApplicationContext，接着我们来看看*第五步，刷新容器refreshContext(context);

```
private void refreshContext(ConfigurableApplicationContext context) {
    refresh(context);
}

protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    //调用容器的refresh()方法刷新容器
    ((AbstractApplicationContext) applicationContext).refresh();
}
```

#### 容器刷新过程

调用抽象父类AbstractApplicationContext的**refresh**()方法；

**`AbstractApplicationContext`**

```
 1 public void refresh() throws BeansException, IllegalStateException {
 2     synchronized (this.startupShutdownMonitor) {
 3         /**
 4          * 刷新上下文环境
 5          */
 6         prepareRefresh();
 7 
 8         /**
 9          * 初始化BeanFactory，解析XML，相当于之前的XmlBeanFactory的操作，
10          */
11         ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
12 
13         /**
14          * 为上下文准备BeanFactory，即对BeanFactory的各种功能进行填充，如常用的注解@Autowired @Qualifier等
15          * 添加ApplicationContextAwareProcessor处理器
16          * 在依赖注入忽略实现*Aware的接口，如EnvironmentAware、ApplicationEventPublisherAware等
17          * 注册依赖，如一个bean的属性中含有ApplicationEventPublisher(beanFactory)，则会将beanFactory的实例注入进去
18          */
19         prepareBeanFactory(beanFactory);
20 
21         try {
22             /**
23              * 提供子类覆盖的额外处理，即子类处理自定义的BeanFactoryPostProcess
24              */
25             postProcessBeanFactory(beanFactory);
26 
27             /**
28              * 激活各种BeanFactory处理器,包括BeanDefinitionRegistryBeanFactoryPostProcessor和普通的BeanFactoryPostProcessor
29              * 执行对应的postProcessBeanDefinitionRegistry方法 和  postProcessBeanFactory方法
30              */
31             invokeBeanFactoryPostProcessors(beanFactory);
32 
33             /**
34              * 注册拦截Bean创建的Bean处理器，即注册BeanPostProcessor，不是BeanFactoryPostProcessor，注意两者的区别
35              * 注意，这里仅仅是注册，并不会执行对应的方法，将在bean的实例化时执行对应的方法
36              */
37             registerBeanPostProcessors(beanFactory);
38 
39             /**
40              * 初始化上下文中的资源文件，如国际化文件的处理等
41              */
42             initMessageSource();
43 
44             /**
45              * 初始化上下文事件广播器，并放入applicatioEventMulticaster,如ApplicationEventPublisher
46              */
47             initApplicationEventMulticaster();
48 
49             /**
50              * 给子类扩展初始化其他Bean
51              */
52             onRefresh();
53 
54             /**
55              * 在所有bean中查找listener bean，然后注册到广播器中
56              */
57             registerListeners();
58 
59             /**
60              * 设置转换器
61              * 注册一个默认的属性值解析器
62              * 冻结所有的bean定义，说明注册的bean定义将不能被修改或进一步的处理
63              * 初始化剩余的非惰性的bean，即初始化非延迟加载的bean
64              */
65             finishBeanFactoryInitialization(beanFactory);
66 
67             /**
68              * 通过spring的事件发布机制发布ContextRefreshedEvent事件，以保证对应的监听器做进一步的处理
69              * 即对那种在spring启动后需要处理的一些类，这些类实现了ApplicationListener<ContextRefreshedEvent>，
70              * 这里就是要触发这些类的执行(执行onApplicationEvent方法)
71              * spring的内置Event有ContextClosedEvent、ContextRefreshedEvent、ContextStartedEvent、ContextStoppedEvent、RequestHandleEvent
72              * 完成初始化，通知生命周期处理器lifeCycleProcessor刷新过程，同时发出ContextRefreshEvent通知其他人
73              */
74             finishRefresh();
75         }
76 
77         finally {
78     
79             resetCommonCaches();
80         }
81     }
82 }
```

我们看第52行的方法：

```
protected void onRefresh() throws BeansException {

}
```

很明显抽象父类AbstractApplicationContext中的onRefresh是一个空方法，并且使用protected修饰，也就是其子类可以重写onRefresh方法，那我们看看其子类AnnotationConfigEmbeddedWebApplicationContext中的onRefresh方法是如何重写的，AnnotationConfigEmbeddedWebApplicationContext又继承EmbeddedWebApplicationContext，如下：

```
public class AnnotationConfigEmbeddedWebApplicationContext extends EmbeddedWebApplicationContext {
```

那我们看看其父类EmbeddedWebApplicationContext 是如何重写onRefresh方法的：

**EmbeddedWebApplicationContext**

```
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        //核心方法：会获取嵌入式的Servlet容器工厂，并通过工厂来获取Servlet容器
        createEmbeddedServletContainer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start embedded container", ex);
    }
}
```

在createEmbeddedServletContainer方法中会获取嵌入式的Servlet容器工厂，并通过工厂来获取Servlet容器：

```
 1 private void createEmbeddedServletContainer() {
 2     EmbeddedServletContainer localContainer = this.embeddedServletContainer;
 3     ServletContext localServletContext = getServletContext();
 4     if (localContainer == null && localServletContext == null) {
 5         //先获取嵌入式Servlet容器工厂
 6         EmbeddedServletContainerFactory containerFactory = getEmbeddedServletContainerFactory();
 7         //根据容器工厂来获取对应的嵌入式Servlet容器
 8         this.embeddedServletContainer = containerFactory.getEmbeddedServletContainer(getSelfInitializer());
 9     }
10     else if (localServletContext != null) {
11         try {
12             getSelfInitializer().onStartup(localServletContext);
13         }
14         catch (ServletException ex) {
15             throw new ApplicationContextException("Cannot initialize servlet context",ex);
16         }
17     }
18     initPropertySources();
19 }
```

关键代码在第6和第8行，**先获取Servlet容器工厂，然后****根据容器工厂来获取对应的嵌入式Servlet容器**

#### 获取Servlet容器工厂

```
protected EmbeddedServletContainerFactory getEmbeddedServletContainerFactory() {
    //从Spring的IOC容器中获取EmbeddedServletContainerFactory.class类型的Bean
    String[] beanNames = getBeanFactory().getBeanNamesForType(EmbeddedServletContainerFactory.class);
    //调用getBean实例化EmbeddedServletContainerFactory.class
    return getBeanFactory().getBean(beanNames[0], EmbeddedServletContainerFactory.class);
}
```

我们看到先从Spring的IOC容器中获取EmbeddedServletContainerFactory.class类型的Bean，然后调用getBean实例化EmbeddedServletContainerFactory.class，大家还记得我们第一节Servlet容器自动配置类EmbeddedServletContainerAutoConfiguration中注入Spring容器的对象是什么吗？当我们引入spring-boot-starter-web这个启动器后，会注入TomcatEmbeddedServletContainerFactory这个对象到Spring容器中，所以这里获取到的**Servlet容器工厂是TomcatEmbeddedServletContainerFactory，然后调用

TomcatEmbeddedServletContainerFactory的getEmbeddedServletContainer方法获取Servlet容器，并且启动Tomcat，大家可以看看文章开头的getEmbeddedServletContainer方法。

大家看一下第8行代码获取Servlet容器方法的参数getSelfInitializer()，这是个啥？我们点进去看看

```
private ServletContextInitializer getSelfInitializer() {
    //创建一个ServletContextInitializer对象，并重写onStartup方法，很明显是一个回调方法
    return new ServletContextInitializer() {
        public void onStartup(ServletContext servletContext) throws ServletException {
            EmbeddedWebApplicationContext.this.selfInitialize(servletContext);
        }
    };
}
```

创建一个ServletContextInitializer对象，并重写onStartup方法，很明显是一个回调方法，这里给大家留一点疑问：

- **ServletContextInitializer对象创建过程是怎样的？**
- **onStartup是何时调用的？**
- **onStartup方法的作用是什么？**

**`ServletContextInitializer`是 Servlet 容器初始化的时候，提供的初始化接口。**

## SpringBoot如何实现SpringMvc的？

### 自定义Servlet、Filter、Listener

Spring容器中声明ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean

```
@Bean
public ServletRegistrationBean customServlet() {
    return new ServletRegistrationBean(new CustomServlet(), "/custom");
}

private static class CustomServlet extends HttpServlet {
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("receive by custom servlet");
    }
}
```

先自定义一个**Servlet，重写**service实现自己的业务逻辑，然后通过@Bean注解往Spring容器中注入一个**ServletRegistrationBean类型的bean实例，并且实例化一个自定义的Servlet作为参数，这样就将自定义的Servlet加入Tomcat中了。**

### @ServletComponentScan注解和@WebServlet、@WebFilter以及@WebListener注解配合使用

@ServletComponentScan注解启用ImportServletComponentScanRegistrar类，是个ImportBeanDefinitionRegistrar接口的实现类，会被Spring容器所解析。ServletComponentScanRegistrar内部会解析@ServletComponentScan注解，然后会在Spring容器中注册ServletComponentRegisteringPostProcessor，是个BeanFactoryPostProcessor，会去解析扫描出来的类是不是有@WebServlet、@WebListener、@WebFilter这3种注解，有的话把这3种类型的类转换成ServletRegistrationBean、FilterRegistrationBean或者ServletListenerRegistrationBean，然后让Spring容器去解析：

```
@SpringBootApplication
@ServletComponentScan
public class EmbeddedServletApplication {
 ... 
}

@WebServlet(urlPatterns = "/simple")
public class SimpleServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("receive by SimpleServlet");
    }
}
```

### 在Spring容器中声明Servlet、Filter或者Listener

```
@Bean(name = "dispatcherServlet")
public DispatcherServlet myDispatcherServlet() {
    return new DispatcherServlet();
}
```

我们发现往Tomcat中添加Servlet、Filter或者Listener还是挺容易的，大家还记得以前SpringMVC是怎么配置**DispatcherServlet**的吗？在web.xml中：

```
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

和我们SpringBoot中配置Servlet相比是不是复杂很多，虽然SpringBoot中自定义Servlet很简单，但是其底层却不简单，下面我们来分析一下其原理

## ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean

我们来看看这几个特殊的类:

### **ServletRegistrationBean**

```
public class ServletRegistrationBean extends RegistrationBean {
    //存放目标Servlet实例
    private Servlet servlet;
    //存放Servlet的urlMapping
    private Set<String> urlMappings;
    private boolean alwaysMapUrl;
    private int loadOnStartup;
    private MultipartConfigElement multipartConfig;


    public ServletRegistrationBean(Servlet servlet, String... urlMappings) {
        this(servlet, true, urlMappings);
    }

    public ServletRegistrationBean(Servlet servlet, boolean alwaysMapUrl, String... urlMappings) {
        this.urlMappings = new LinkedHashSet();
        this.alwaysMapUrl = true;
        this.loadOnStartup = -1;
        Assert.notNull(servlet, "Servlet must not be null");
        Assert.notNull(urlMappings, "UrlMappings must not be null");
        this.servlet = servlet;
        this.alwaysMapUrl = alwaysMapUrl;
        this.urlMappings.addAll(Arrays.asList(urlMappings));
    }
    
    public void onStartup(ServletContext servletContext) throws ServletException {
        Assert.notNull(this.servlet, "Servlet must not be null");
        String name = this.getServletName();
        if (!this.isEnabled()) {
            logger.info("Servlet " + name + " was not registered (disabled)");
        } else {
            logger.info("Mapping servlet: '" + name + "' to " + this.urlMappings);
            Dynamic added = servletContext.addServlet(name, this.servlet);
            if (added == null) {
                logger.info("Servlet " + name + " was not registered (possibly already registered?)");
            } else {
                this.configure(added);
            }
        }
    }
    
    //略
}
```

在我们例子中我们通过return new ServletRegistrationBean(new CustomServlet(), "/custom");就知道，ServletRegistrationBean里会存放目标Servlet实例和urlMapping,并且继承RegistrationBean这个类

### **FilterRegistrationBean**

```
public class FilterRegistrationBean extends AbstractFilterRegistrationBean {
    //存放目标Filter对象
    private Filter filter;

    public FilterRegistrationBean() {
        super(new ServletRegistrationBean[0]);
    }

    public FilterRegistrationBean(Filter filter, ServletRegistrationBean... servletRegistrationBeans) {
        super(servletRegistrationBeans);
        Assert.notNull(filter, "Filter must not be null");
        this.filter = filter;
    }

    public Filter getFilter() {
        return this.filter;
    }

    public void setFilter(Filter filter) {
        Assert.notNull(filter, "Filter must not be null");
        this.filter = filter;
    }
}

abstract class AbstractFilterRegistrationBean extends RegistrationBean {
    private static final EnumSet<DispatcherType> ASYNC_DISPATCHER_TYPES;
    private static final EnumSet<DispatcherType> NON_ASYNC_DISPATCHER_TYPES;
    private static final String[] DEFAULT_URL_MAPPINGS;
    private Set<ServletRegistrationBean> servletRegistrationBeans = new LinkedHashSet();
    private Set<String> servletNames = new LinkedHashSet();
    private Set<String> urlPatterns = new LinkedHashSet();
    //重写onStartup方法
    public void onStartup(ServletContext servletContext) throws ServletException {
        Filter filter = this.getFilter();
        Assert.notNull(filter, "Filter must not be null");
        String name = this.getOrDeduceName(filter);
        if (!this.isEnabled()) {
            this.logger.info("Filter " + name + " was not registered (disabled)");
        } else {
            Dynamic added = servletContext.addFilter(name, filter);
            if (added == null) {
                this.logger.info("Filter " + name + " was not registered (possibly already registered?)");
            } else {
                this.configure(added);
            }
        }
    }
    //略...
}
```

我们看到FilterRegistrationBean 中也保存了**目标Filter对象，并且继承了RegistrationBean

### **ServletListenerRegistrationBean**

```
public class ServletListenerRegistrationBean<T extends EventListener> extends RegistrationBean {
    //存放了目标listener
    private T listener;

    public ServletListenerRegistrationBean() {
    }

    public ServletListenerRegistrationBean(T listener) {
        Assert.notNull(listener, "Listener must not be null");
        Assert.isTrue(isSupportedType(listener), "Listener is not of a supported type");
        this.listener = listener;
    }

    public void setListener(T listener) {
        Assert.notNull(listener, "Listener must not be null");
        Assert.isTrue(isSupportedType(listener), "Listener is not of a supported type");
        this.listener = listener;
    }

    public void onStartup(ServletContext servletContext) throws ServletException {
        if (!this.isEnabled()) {
            logger.info("Listener " + this.listener + " was not registered (disabled)");
        } else {
            try {
                servletContext.addListener(this.listener);
            } catch (RuntimeException var3) {
                throw new IllegalStateException("Failed to add listener '" + this.listener + "' to servlet context", var3);
            }
        }
    }
    //略...
}
```

ServletListenerRegistrationBean也是一样，那我们来看看RegistrationBean这个类

```
public abstract class RegistrationBean implements ServletContextInitializer, Ordered {
    ...
}
public interface ServletContextInitializer {
    void onStartup(ServletContext var1) throws ServletException;
}
```

我们发现**RegistrationBean 实现了****ServletContextInitializer这个接口，并且有一个onStartup方法，**ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean都实现了**onStartup方法。

ServletContextInitializer是 Servlet 容器初始化的时候，提供的初始化接口。所以，Servlet 容器初始化会获取并触发所有的`FilterRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean实例中onStartup方法

那到底是何时触发这些类的onStartup方法呢？

当Tomcat容器启动时，会执行`callInitializers`，然后获取所有的**`ServletContextInitializer，循环执行`**`onStartup`方法触发回调方法。那`FilterRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean实例是何时加入到`Initializers集合的呢？这要回顾一下我们上一篇文章Tomcat的启动过程

## Servlet容器的启动

大家可以看看我上一篇文章，我这里简单的复制一下代码

**EmbeddedWebApplicationContext**

```
 1 @Override
 2 protected void onRefresh() {
 3     super.onRefresh();
 4     try {
 5         //核心方法：会获取嵌入式的Servlet容器工厂，并通过工厂来获取Servlet容器
 6         createEmbeddedServletContainer();
 7     }
 8     catch (Throwable ex) {
 9         throw new ApplicationContextException("Unable to start embedded container", ex);
10     }
11 }
12 
13 private void createEmbeddedServletContainer() {
14     EmbeddedServletContainer localContainer = this.embeddedServletContainer;
15     ServletContext localServletContext = getServletContext();
16     if (localContainer == null && localServletContext == null) {
17         //先获取嵌入式Servlet容器工厂
18         EmbeddedServletContainerFactory containerFactory = getEmbeddedServletContainerFactory();
19         //根据容器工厂来获取对应的嵌入式Servlet容器
20         this.embeddedServletContainer = containerFactory.getEmbeddedServletContainer(getSelfInitializer());
21     }
22     else if (localServletContext != null) {
23         try {
24             getSelfInitializer().onStartup(localServletContext);
25         }
26         catch (ServletException ex) {
27             throw new ApplicationContextException("Cannot initialize servlet context",ex);
28         }
29     }
30     initPropertySources();
31 }
```

关键代码在第20行，先通过getSelfInitializer()获取到所有的Initializer，传入Servlet容器中，那核心就在getSelfInitializer()方法：

```
1 private ServletContextInitializer getSelfInitializer() {
2     //只是创建了一个ServletContextInitializer实例返回
3     //所以Servlet容器启动的时候，会调用这个对象的onStartup方法
4     return new ServletContextInitializer() {
5         public void onStartup(ServletContext servletContext) throws ServletException {
6             EmbeddedWebApplicationContext.this.selfInitialize(servletContext);
7         }
8     };
9 }
```

我们看到只是创建了一个ServletContextInitializer实例返回，所以Servlet容器启动的时候，会调用这个对象的onStartup方法，那我们来分析其onStartup中的逻辑，也就是selfInitialize方法，并将Servlet上下文对象传进去了

**selfInitialize**

```
 1 private void selfInitialize(ServletContext servletContext) throws ServletException {
 2     prepareWebApplicationContext(servletContext);
 3     ConfigurableListableBeanFactory beanFactory = getBeanFactory();
 4     ExistingWebApplicationScopes existingScopes = new ExistingWebApplicationScopes(beanFactory);
 5     WebApplicationContextUtils.registerWebApplicationScopes(beanFactory,getServletContext());
 6     existingScopes.restore();
 7     WebApplicationContextUtils.registerEnvironmentBeans(beanFactory,getServletContext());
 8     //这里便是获取所有的 ServletContextInitializer 实现类，会获取所有的注册组件
 9     for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
10         //执行所有ServletContextInitializer的onStartup方法
11         beans.onStartup(servletContext);
12     }
13 }
```

关键代码在第9和第11行，先获取所有的ServletContextInitializer 实现类，然后遍历执行所有ServletContextInitializer的onStartup方法



### **获取所有的ServletContextInitializer**

我们来看看getServletContextInitializerBeans方法**
**

```
protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
    return new ServletContextInitializerBeans(getBeanFactory());
}
```

ServletContextInitializerBeans对象是对`ServletContextInitializer`的一种包装：

```
 1 public class ServletContextInitializerBeans extends AbstractCollection<ServletContextInitializer> {
 2     private final MultiValueMap<Class<?>, ServletContextInitializer> initializers = new LinkedMultiValueMap();
 3     //存放所有的ServletContextInitializer
 4     private List<ServletContextInitializer> sortedList;
 5 
 6     public ServletContextInitializerBeans(ListableBeanFactory beanFactory) {
 7         //执行addServletContextInitializerBeans
 8         this.addServletContextInitializerBeans(beanFactory);
 9         //执行addAdaptableBeans
10         this.addAdaptableBeans(beanFactory);
11         List<ServletContextInitializer> sortedInitializers = new ArrayList();
12         Iterator var3 = this.initializers.entrySet().iterator();
13 
14         while(var3.hasNext()) {
15             Entry<?, List<ServletContextInitializer>> entry = (Entry)var3.next();
16             AnnotationAwareOrderComparator.sort((List)entry.getValue());
17             sortedInitializers.addAll((Collection)entry.getValue());
18         }
19         this.sortedList = Collections.unmodifiableList(sortedInitializers);
20     }
21 
22     private void addServletContextInitializerBeans(ListableBeanFactory beanFactory) {
23         Iterator var2 = this.getOrderedBeansOfType(beanFactory, ServletContextInitializer.class).iterator();
24 
25         while(var2.hasNext()) {
26             Entry<String, ServletContextInitializer> initializerBean = (Entry)var2.next();
27             this.addServletContextInitializerBean((String)initializerBean.getKey(), (ServletContextInitializer)initializerBean.getValue(), beanFactory);
28         }
29 
30     }
31 
32     private void addServletContextInitializerBean(String beanName, ServletContextInitializer initializer, ListableBeanFactory beanFactory) {
33         if (initializer instanceof ServletRegistrationBean) {
34             Servlet source = ((ServletRegistrationBean)initializer).getServlet();
35             this.addServletContextInitializerBean(Servlet.class, beanName, initializer, beanFactory, source);
36         } else if (initializer instanceof FilterRegistrationBean) {
37             Filter source = ((FilterRegistrationBean)initializer).getFilter();
38             this.addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
39         } else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
40             String source = ((DelegatingFilterProxyRegistrationBean)initializer).getTargetBeanName();
41             this.addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
42         } else if (initializer instanceof ServletListenerRegistrationBean) {
43             EventListener source = ((ServletListenerRegistrationBean)initializer).getListener();
44             this.addServletContextInitializerBean(EventListener.class, beanName, initializer, beanFactory, source);
45         } else {
46             this.addServletContextInitializerBean(ServletContextInitializer.class, beanName, initializer, beanFactory, initializer);
47         }
48 
49     }
50 
51     private void addServletContextInitializerBean(Class<?> type, String beanName, ServletContextInitializer initializer, ListableBeanFactory beanFactory, Object source) {
52         this.initializers.add(type, initializer);
53         if (source != null) {
54             this.seen.add(source);
55         }
56 
57         if (logger.isDebugEnabled()) {
58             String resourceDescription = this.getResourceDescription(beanName, beanFactory);
59             int order = this.getOrder(initializer);
60             logger.debug("Added existing " + type.getSimpleName() + " initializer bean '" + beanName + "'; order=" + order + ", resource=" + resourceDescription);
61         }
62 
63     }
64 
65     private void addAdaptableBeans(ListableBeanFactory beanFactory) {
66         MultipartConfigElement multipartConfig = this.getMultipartConfig(beanFactory);
67         this.addAsRegistrationBean(beanFactory, Servlet.class, new ServletContextInitializerBeans.ServletRegistrationBeanAdapter(multipartConfig));
68         this.addAsRegistrationBean(beanFactory, Filter.class, new ServletContextInitializerBeans.FilterRegistrationBeanAdapter(null));
69         Iterator var3 = ServletListenerRegistrationBean.getSupportedTypes().iterator();
70 
71         while(var3.hasNext()) {
72             Class<?> listenerType = (Class)var3.next();
73             this.addAsRegistrationBean(beanFactory, EventListener.class, listenerType, new ServletContextInitializerBeans.ServletListenerRegistrationBeanAdapter(null));
74         }
75 
76     }
77     
78     public Iterator<ServletContextInitializer> iterator() {
79         //返回所有的ServletContextInitializer
80         return this.sortedList.iterator();
81     }
82 
83     //略...
84 }
```

我们看到ServletContextInitializerBeans 中有一个存放所有ServletContextInitializer的集合sortedList，就是在其构造方法中获取所有的ServletContextInitializer，并放入sortedList集合中，那我们来看看其构造方法的逻辑，看到第8行先调用

addServletContextInitializerBeans方法：　　

```
1 private void addServletContextInitializerBeans(ListableBeanFactory beanFactory) {
2     //从Spring容器中获取所有ServletContextInitializer.class 类型的Bean
3     for (Entry<String, ServletContextInitializer> initializerBean : getOrderedBeansOfType(beanFactory, ServletContextInitializer.class)) {
4         //添加到具体的集合中
5         addServletContextInitializerBean(initializerBean.getKey(),initializerBean.getValue(), beanFactory);
6     }
7 }
```

我们看到先从Spring容器中获取所有**ServletContextInitializer.class** 类型的Bean，这里我们自定义的ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean就被获取到了，然后调用addServletContextInitializerBean方法：

```
 1 private void addServletContextInitializerBean(String beanName, ServletContextInitializer initializer, ListableBeanFactory beanFactory) {
 2     //判断ServletRegistrationBean类型
 3     if (initializer instanceof ServletRegistrationBean) {
 4         Servlet source = ((ServletRegistrationBean)initializer).getServlet();
 5         //将ServletRegistrationBean加入到集合中
 6         this.addServletContextInitializerBean(Servlet.class, beanName, initializer, beanFactory, source);
 7     //判断FilterRegistrationBean类型
 8     } else if (initializer instanceof FilterRegistrationBean) {
 9         Filter source = ((FilterRegistrationBean)initializer).getFilter();
10         //将ServletRegistrationBean加入到集合中
11         this.addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
12     } else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
13         String source = ((DelegatingFilterProxyRegistrationBean)initializer).getTargetBeanName();
14         this.addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
15     } else if (initializer instanceof ServletListenerRegistrationBean) {
16         EventListener source = ((ServletListenerRegistrationBean)initializer).getListener();
17         this.addServletContextInitializerBean(EventListener.class, beanName, initializer, beanFactory, source);
18     } else {
19         this.addServletContextInitializerBean(ServletContextInitializer.class, beanName, initializer, beanFactory, initializer);
20     }
21 
22 }
23 
24 private void addServletContextInitializerBean(Class<?> type, String beanName, 
25                             ServletContextInitializer initializer, ListableBeanFactory beanFactory, Object source) {
26     //加入到initializers中
27     this.initializers.add(type, initializer);
28 }
```

很明显，判断从Spring容器中获取的ServletContextInitializer类型，如ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean，并加入到initializers集合中去，我们再来看构造器中的另外一个方法**addAdaptableBeans(beanFactory)：**

```
 1 private void addAdaptableBeans(ListableBeanFactory beanFactory) {
 2     //从beanFactory获取所有Servlet.class和Filter.class类型的Bean，并封装成RegistrationBean对象，加入到集合中
 3     this.addAsRegistrationBean(beanFactory, Servlet.class, new ServletContextInitializerBeans.ServletRegistrationBeanAdapter(multipartConfig));
 4     this.addAsRegistrationBean(beanFactory, Filter.class, new ServletContextInitializerBeans.FilterRegistrationBeanAdapter(null));
 5 }
 6 
 7 private <T, B extends T> void addAsRegistrationBean(ListableBeanFactory beanFactory, Class<T> type, Class<B> beanType, ServletContextInitializerBeans.RegistrationBeanAdapter<T> adapter) {
 8     //从Spring容器中获取所有的Servlet.class和Filter.class类型的Bean
 9     List<Entry<String, B>> beans = this.getOrderedBeansOfType(beanFactory, beanType, this.seen);
10     Iterator var6 = beans.iterator();
11 
12     while(var6.hasNext()) {
13         Entry<String, B> bean = (Entry)var6.next();
14         if (this.seen.add(bean.getValue())) {
15             int order = this.getOrder(bean.getValue());
16             String beanName = (String)bean.getKey();
17             //创建Servlet.class和Filter.class包装成RegistrationBean对象
18             RegistrationBean registration = adapter.createRegistrationBean(beanName, bean.getValue(), beans.size());
19             registration.setName(beanName);
20             registration.setOrder(order);
21             this.initializers.add(type, registration);
22             if (logger.isDebugEnabled()) {
23                 logger.debug("Created " + type.getSimpleName() + " initializer for bean '" + beanName + "'; order=" + order + ", resource=" + this.getResourceDescription(beanName, beanFactory));
24             }
25         }
26     }
27 
28 }
```

我们看到先从beanFactory获取所有Servlet.class和Filter.class类型的Bean，然后通过**ServletRegistrationBeanAdapter和****FilterRegistrationBeanAdapter两个适配器将Servlet.class和Filter.class封装成****RegistrationBean**

```
private static class ServletRegistrationBeanAdapter implements ServletContextInitializerBeans.RegistrationBeanAdapter<Servlet> {
    private final MultipartConfigElement multipartConfig;

    ServletRegistrationBeanAdapter(MultipartConfigElement multipartConfig) {
        this.multipartConfig = multipartConfig;
    }

    public RegistrationBean createRegistrationBean(String name, Servlet source, int totalNumberOfSourceBeans) {
        String url = totalNumberOfSourceBeans == 1 ? "/" : "/" + name + "/";
        if (name.equals("dispatcherServlet")) {
            url = "/";
        }
        //还是将Servlet.class实例封装成ServletRegistrationBean对象
        //这和我们自己创建ServletRegistrationBean对象是一模一样的
        ServletRegistrationBean bean = new ServletRegistrationBean(source, new String[]{url});
        bean.setMultipartConfig(this.multipartConfig);
        return bean;
    }
}

private static class FilterRegistrationBeanAdapter implements ServletContextInitializerBeans.RegistrationBeanAdapter<Filter> {
    private FilterRegistrationBeanAdapter() {
    }

    public RegistrationBean createRegistrationBean(String name, Filter source, int totalNumberOfSourceBeans) {
        //Filter.class实例封装成FilterRegistrationBean对象
        return new FilterRegistrationBean(source, new ServletRegistrationBean[0]);
    }
}
```

代码中注释很清楚了还是将Servlet.class实例封装成ServletRegistrationBean对象，将Filter.class实例封装成FilterRegistrationBean对象，这和我们自己定义ServletRegistrationBean对象是一模一样的，现在所有的ServletRegistrationBean、FilterRegistrationBean

Servlet.class、Filter.class都添加到List<ServletContextInitializer> sortedList这个集合中去了，接着就是遍历这个集合，执行其**onStartup**方法了

### ServletContextInitializer的**onStartup**方法

**ServletRegistrationBean**

```
public class ServletRegistrationBean extends RegistrationBean {
    private static final Log logger = LogFactory.getLog(ServletRegistrationBean.class);
    private static final String[] DEFAULT_MAPPINGS = new String[]{"/*"};
    private Servlet servlet;
    
    public void onStartup(ServletContext servletContext) throws ServletException {
        Assert.notNull(this.servlet, "Servlet must not be null");
        String name = this.getServletName();
        //调用ServletContext的addServlet
        Dynamic added = servletContext.addServlet(name, this.servlet);
    }
    
    //略...
}

private javax.servlet.ServletRegistration.Dynamic addServlet(String servletName, String servletClass, Servlet servlet, Map<String, String> initParams) throws IllegalStateException {
    if (servletName != null && !servletName.equals("")) {
        if (!this.context.getState().equals(LifecycleState.STARTING_PREP)) {
            throw new IllegalStateException(sm.getString("applicationContext.addServlet.ise", new Object[]{this.getContextPath()}));
        } else {
            Wrapper wrapper = (Wrapper)this.context.findChild(servletName);
            if (wrapper == null) {
                wrapper = this.context.createWrapper();
                wrapper.setName(servletName);
                this.context.addChild(wrapper);
            } else if (wrapper.getName() != null && wrapper.getServletClass() != null) {
                if (!wrapper.isOverridable()) {
                    return null;
                }

                wrapper.setOverridable(false);
            }

            if (servlet == null) {
                wrapper.setServletClass(servletClass);
            } else {
                wrapper.setServletClass(servlet.getClass().getName());
                wrapper.setServlet(servlet);
            }

            if (initParams != null) {
                Iterator i$ = initParams.entrySet().iterator();

                while(i$.hasNext()) {
                    Entry<String, String> initParam = (Entry)i$.next();
                    wrapper.addInitParameter((String)initParam.getKey(), (String)initParam.getValue());
                }
            }

            return this.context.dynamicServletAdded(wrapper);
        }
    } else {
        throw new IllegalArgumentException(sm.getString("applicationContext.invalidServletName", new Object[]{servletName}));
    }
}
```

看到没，ServletRegistrationBean 中的 onStartup先获取Servlet的name，然后调用ServletContext的addServlet将Servlet加入到Tomcat中，这样我们就能发请求给这个Servlet了。

**AbstractFilterRegistrationBean**

```
public void onStartup(ServletContext servletContext) throws ServletException {
    Filter filter = this.getFilter();
    Assert.notNull(filter, "Filter must not be null");
    String name = this.getOrDeduceName(filter);
    //调用ServletContext的addFilter
    Dynamic added = servletContext.addFilter(name, filter);
}
```

AbstractFilterRegistrationBean也是同样的原理，先获取目标Filter，然后调用ServletContext的**addFilter**将Filter加入到Tomcat中，这样Filter就能拦截我们请求了。

## DispatcherServletAutoConfiguration

最熟悉的莫过于，在Spring Boot在自动配置SpringMVC的时候，会自动注册SpringMVC前端控制器：**DispatcherServlet**，该控制器主要在**DispatcherServletAutoConfiguration**自动配置类中进行注册的。DispatcherServlet是SpringMVC中的核心分发器。DispatcherServletAutoConfiguration也在spring.factories中配置了

**DispatcherServletConfiguration**

```
 1 @Configuration
 2 @ConditionalOnWebApplication
 3 // 先看下ClassPath下是否有DispatcherServlet.class字节码
 4 // 我们引入了spring-boot-starter-web，同时引入了tomcat和SpringMvc,肯定会存在DispatcherServlet.class字节码
 5 @ConditionalOnClass({DispatcherServlet.class})
 6 // 这个配置类的执行要在EmbeddedServletContainerAutoConfiguration配置类生效之后执行
 7 // 毕竟要等Tomcat启动后才能往其中注入DispatcherServlet
 8 @AutoConfigureAfter({EmbeddedServletContainerAutoConfiguration.class})
 9 protected static class DispatcherServletConfiguration {
10   public static final String DEFAULT_DISPATCHER_SERVLET_BEAN_NAME = "dispatcherServlet";
11   public static final String DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME = "dispatcherServletRegistration";
12   @Autowired
13   private ServerProperties server;
14 
15   @Autowired
16   private WebMvcProperties webMvcProperties;
17 
18   @Autowired(required = false)
19   private MultipartConfigElement multipartConfig;
20 
21   // Spring容器注册DispatcherServlet
22   @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
23   public DispatcherServlet dispatcherServlet() {
24     // 直接构造DispatcherServlet，并设置WebMvcProperties中的一些配置
25     DispatcherServlet dispatcherServlet = new DispatcherServlet();
26     dispatcherServlet.setDispatchOptionsRequest(this.webMvcProperties.isDispatchOptionsRequest());
27     dispatcherServlet.setDispatchTraceRequest(this.webMvcProperties.isDispatchTraceRequest());
28     dispatcherServlet.setThrowExceptionIfNoHandlerFound(this.webMvcProperties.isThrowExceptionIfNoHandlerFound());
29     return dispatcherServlet;
30   }
31 
32   @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
33   public ServletRegistrationBean dispatcherServletRegistration() {
34     // 直接使用DispatcherServlet和server配置中的servletPath路径构造ServletRegistrationBean
35     // ServletRegistrationBean实现了ServletContextInitializer接口，在onStartup方法中对应的Servlet注册到Servlet容器中
36     // 所以这里DispatcherServlet会被注册到Servlet容器中，对应的urlMapping为server.servletPath配置
37     ServletRegistrationBean registration = new ServletRegistrationBean(dispatcherServlet(), this.server.getServletMapping());
38     registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
39     if (this.multipartConfig != null) {
40       registration.setMultipartConfig(this.multipartConfig);
41     }
42     return registration;
43   }
44 
45   @Bean // 构造文件上传相关的bean
46   @ConditionalOnBean(MultipartResolver.class)
47   @ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
48   public MultipartResolver multipartResolver(MultipartResolver resolver) {
49     return resolver;
50   }
51 
52 }
```

先看下ClassPath下是否有DispatcherServlet.class字节码， 我们引入了spring-boot-starter-web，同时引入了tomcat和SpringMvc,肯定会存在DispatcherServlet.class字节码，如果没有导入spring-boot-starter-web，则这个配置类将不会生效

然后往Spring容器中注册DispatcherServlet实例，接着又加入ServletRegistrationBean实例，并把DispatcherServlet实例作为参数，上面我们已经学过了ServletRegistrationBean的逻辑，在Tomcat启动的时候，会获取所有的ServletRegistrationBean，并执行其中的onstartup方法，将DispatcherServlet注册到Servlet容器中，这样就类似原来的web.xml中配置的dispatcherServlet。

```
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

所以只要导入了spring-boot-starter-web这个starter，SpringBoot就有了Tomcat容器，并且往Tomcat容器中注册了DispatcherServlet对象，这样就能接收到我们的请求了

## 事务源码解析

### SSM使用事务

#### 导入JDBC依赖包

众所周知，凡是需要跟数据库打交道的，基本上都要添加jdbc的依赖，在Spring项目中，加入的是spring-jdbc依赖：

```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
</dependency>
```

### 配置版事务

在使用配置文件的方式中，通常会在Spring的配置文件中配置事务管理器，并注入数据源：

```
<!-- 注册数据源 -->
<bean id="dataSource" class="...">
    <property name="" value=""/>
</bean>

<!-- 注册事务管理器 -->
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>

<!-- 开启事务注解 -->
<tx:annotation-driven transaction-manager="txManager" />
```

接下来可以直接在业务层Service的方法上或者类上添加**@Transactional**。



### 注解版事务

首先需要注册两个Bean，分别对应上面Spring配置文件中的两个Bean：

```
@EnableTransactionManagement//重要
@Configuration
public class TxConfig {

    @Bean
    public DataSource dataSource() {
        ComboPooledDataSource dataSource = new ComboPooledDataSource();
        dataSource.setUser("...");
        dataSource.setPassword("...");
        dataSource.setDriverClass("...");
        dataSource.setJdbcUrl("...");
        return dataSource;
    }
  
    //重要
    @Bean
    public PlatformTransactionManager platformTransactionManager() {
        return new DataSourceTransactionManager(dataSource());//放入数据源
    }
}
```

我们看到往Spring容器中注入了DataSource 和PlatformTransactionManager 对象，并且通过@EnableTransactionManagement注解开启了事务，和上面的XML配置是一一对应的。PlatformTransactionManager这个Bean非常重要，要使用事务管理，就必须要在IOC容器中注册一个事务管理器。

```
public interface PlatformTransactionManager {
    //获取一个Transaction
    TransactionStatus getTransaction(TransactionDefinition var1) throws TransactionException;
    //提交事务
    void commit(TransactionStatus var1) throws TransactionException;
    //回滚事务
    void rollback(TransactionStatus var1) throws TransactionException;
}
```

我们看到事务管理器的作用就是获取事务，提交回滚事务。DataSourceTransactionManager是PlatformTransactionManager的一个实现类，大家可以看看我以前的文章[spring5 源码深度解析----- Spring事务 是怎么通过AOP实现的？（100%理解Spring事务）](https://www.cnblogs.com/java-chen-hao/p/11635380.html) 看一下Spring的声明式事务的源码。下面我们来看看SpringBoot是如何自动配置事务的

## SpringBoot自动配置事务



### 引入JDBC

众所周知，在SpringBoot中凡是需要跟数据库打交道的，基本上都要显式或者隐式添加jdbc的依赖：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

也就是jdbc的场景启动器，我们点进去看看

其实也是引入了`spring-jdbc的依赖，接下来我们要看两个重要的事务自动配置类`

## DataSourceTransactionManagerAutoConfiguration

我们看到在spring.factories中配置了事务管理器自动配置类DataSourceTransactionManagerAutoConfiguration，我们进去看看

```
 1 @Configuration
 2 //在类路径下有这个类存在PlatformTransactionManager时，这个配置类才会生效
 3 //而前面我们已经引入了spring-boot-starter-jdbc，那自然是存在了
 4 @ConditionalOnClass({ JdbcTemplate.class, PlatformTransactionManager.class })
 5 @AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)
 6 @EnableConfigurationProperties(DataSourceProperties.class)
 7 public class DataSourceTransactionManagerAutoConfiguration {
 8 
 9     @Configuration
10     @ConditionalOnSingleCandidate(DataSource.class)
11     static class DataSourceTransactionManagerConfiguration {
12 
13         private final DataSource dataSource;
14 
15         private final TransactionManagerCustomizers transactionManagerCustomizers;
16 
17         DataSourceTransactionManagerConfiguration(DataSource dataSource,
18                 ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
19             this.dataSource = dataSource;
20             this.transactionManagerCustomizers = transactionManagerCustomizers
21                     .getIfAvailable();
22         }
23 
24         @Bean
25         //没有当Spring容器中不存在PlatformTransactionManager这个对象时，创建DataSourceTransactionManager
26         //也就是如果我们自定义了DataSourceTransactionManager并注入Spring容器，这里将不会执行
27         @ConditionalOnMissingBean(PlatformTransactionManager.class)
28         public DataSourceTransactionManager transactionManager(DataSourceProperties properties) {
29             //创建DataSourceTransactionManager注入Spring容器，并且把dataSource传进去
30             DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(this.dataSource);
31             if (this.transactionManagerCustomizers != null) {
32                 this.transactionManagerCustomizers.customize(transactionManager);
33             }
34             return transactionManager;
35         }
36 
37     }
38 
39 }
```

很明显只要我们导入了**spring-boot-starter-jdbc场景启动器**，并且我们没有自定义DataSourceTransactionManager，那么事务管理器自动配置类DataSourceTransactionManagerAutoConfiguration会自动为我们创建DataSourceTransactionManager并注入Spring容器中。但是这还不够，我们前面还是需要通过**@EnableTransactionManagement开启事务呢**，如果不开启事务，@Transactional是不起任何作用的。下面我们就来看看是如何开启事务的

## TransactionAutoConfiguration

我们看到在spring.factories中配置了事务自动开启配置类TransactionAutoConfiguration，我们进去看看

```
 1 @Configuration
 2 //和DataSourceTransactionManagerAutoConfiguration中是一样的
 3 //引入了spring-boot-starter-jdbc，那自然是存在了PlatformTransactionManager
 4 @ConditionalOnClass({PlatformTransactionManager.class})
 5 //这个自动配置类必须要在DataSourceTransactionManagerAutoConfiguration这个自动配置类之后才能生效
 6 //也就是前面我们已经往Spring容器中注入了DataSourceTransactionManager这个对象才执行这个配置类
 7 @AutoConfigureAfter({JtaAutoConfiguration.class, HibernateJpaAutoConfiguration.class, DataSourceTransactionManagerAutoConfiguration.class, Neo4jDataAutoConfiguration.class})
 8 @EnableConfigurationProperties({TransactionProperties.class})
 9 public class TransactionAutoConfiguration {
10     public TransactionAutoConfiguration() {
11     }
12 
13     @Configuration
14     @ConditionalOnBean({PlatformTransactionManager.class})
15     @ConditionalOnMissingBean({AbstractTransactionManagementConfiguration.class})
16     public static class EnableTransactionManagementConfiguration {
17         public EnableTransactionManagementConfiguration() {
18         }
19 
20         @Configuration
21         //重点：通过 @EnableTransactionManagement注解开启事务
22         //可以看到和我们自己使用@EnableTransactionManagement是一样的
23         @EnableTransactionManagement(
24             proxyTargetClass = true
25         )
26         @ConditionalOnProperty(
27             prefix = "spring.aop",
28             name = {"proxy-target-class"},
29             havingValue = "true",
30             matchIfMissing = true
31         )
32         public static class CglibAutoProxyConfiguration {
33             public CglibAutoProxyConfiguration() {
34             }
35         }
36 
37         @Configuration
38         @EnableTransactionManagement(
39             proxyTargetClass = false
40         )
41         @ConditionalOnProperty(
42             prefix = "spring.aop",
43             name = {"proxy-target-class"},
44             havingValue = "false",
45             matchIfMissing = false
46         )
47         public static class JdkDynamicAutoProxyConfiguration {
48             public JdkDynamicAutoProxyConfiguration() {
49             }
50         }
51     }
52 
53     @Configuration
54     @ConditionalOnSingleCandidate(PlatformTransactionManager.class)
55     public static class TransactionTemplateConfiguration {
56         private final PlatformTransactionManager transactionManager;
57 
58         public TransactionTemplateConfiguration(PlatformTransactionManager transactionManager) {
59             this.transactionManager = transactionManager;
60         }
61 
62         @Bean
63         @ConditionalOnMissingBean
64         public TransactionTemplate transactionTemplate() {
65             return new TransactionTemplate(this.transactionManager);
66         }
67     }
68 }
```

我们看到TransactionAutoConfiguration这个自动配置类必须要在DataSourceTransactionManagerAutoConfiguration这个配置类之后才能生效，也就是前面我们已经往Spring容器中注入了DataSourceTransactionManager这个对象才执行这个配置类，然后通过

@EnableTransactionManagement这个注解开启事务，其实和我们自己使用@EnableTransactionManagement是一样的

因此，只要我们在SpringBoot中引入了**spring-boot-starter-jdbc**这个场景启动器，就会帮我们自动开启事务了，我们只需要使用@Transactional就可以了

## mybatis-spring-boot-starter

大多数时候我们在SpringBoot中会引入Mybatis这个orm框架，Mybaits的场景启动器如下

```
<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-starter</artifactId>
  <version>1.3.0</version>
</dependency>
```

我们点进去看看

我们看到mybatis-spring-boot-starter这个场景启动器是引入了spring-boot-starter-jdbc这个场景启动器的，因此只要我们在SpringBoot中使用Mybaits，是自动帮我们开启了Spring事务的

## 总结

springboot 开启事物很简单，只需要加一行注解**@Transactional**就可以了，前提你用的是jdbctemplate, jpa, Mybatis，这种常见的orm。

# 整合Mybatis

本篇我们在SpringBoot中整合Mybatis这个orm框架，毕竟分析一下其自动配置的源码，我们先来回顾一下以前Spring中是如何整合Mybatis的，大家可以看看我这篇文章[Mybaits 源码解析 （十）----- Spring-Mybatis框架使用与源码解析](https://www.cnblogs.com/java-chen-hao/p/11833780.html) 

## Spring-Mybatis使用

### 添加maven依赖

```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>4.3.8.RELEASE</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.mybatis/mybatis-spring -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>1.3.2</version>
</dependency>
```

### 在src/main/resources下添加mybatis-config.xml文件

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <typeAliases>
        <typeAlias alias="User" type="com.chenhao.bean.User" />
    </typeAliases>
    <plugins>
        <plugin interceptor="com.github.pagehelper.PageInterceptor">
            <property name="helperDialect" value="mysql"/>
        </plugin>
    </plugins>

</configuration>
```

### 在src/main/resources/mapper路径下添加User.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    
<mapper namespace="com.chenhao.mapper.UserMapper">
    <select id="getUser" parameterType="int"
        resultType="com.chenhao.bean.User">
        SELECT *
        FROM USER
        WHERE id = #{id}
    </select>
</mapper>
```

### 在src/main/resources/路径下添加beans.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">
 
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/test"></property>
        <property name="username" value="root"></property>
        <property name="password" value="root"></property>
    </bean>
 
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:mybatis-config.xml"></property>
        <property name="dataSource" ref="dataSource" />
        <property name="mapperLocations" value="classpath:mapper/*.xml" />
    </bean>
    
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.chenhao.mapper" />
    </bean>
 
</beans>
```

### 注解的方式

- 以上分析都是在spring的XML配置文件applicationContext.xml进行配置的，mybatis-spring也提供了基于注解的方式来配置sqlSessionFactory和Mapper接口。
- sqlSessionFactory主要是在@Configuration注解的配置类中使用@Bean注解的名为sqlSessionFactory的方法来配置；
- Mapper接口主要是通过在@Configuration注解的配置类中结合@MapperScan注解来指定需要扫描获取mapper接口的包。

```
@Configuration
@MapperScan("com.chenhao.mapper")
public class AppConfig {

  @Bean
  public DataSource dataSource() {
     return new EmbeddedDatabaseBuilder()
            .addScript("schema.sql")
            .build();
  }
 
  @Bean
  public DataSourceTransactionManager transactionManager() {
    return new DataSourceTransactionManager(dataSource());
  }
 
  @Bean
  public SqlSessionFactory sqlSessionFactory() throws Exception {
    //创建SqlSessionFactoryBean对象
    SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
    //设置数据源
    sessionFactory.setDataSource(dataSource());
    //设置Mapper.xml路径
    sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/*.xml"));
    // 设置MyBatis分页插件
    PageInterceptor pageInterceptor = new PageInterceptor();
    Properties properties = new Properties();
    properties.setProperty("helperDialect", "mysql");
    pageInterceptor.setProperties(properties);
    sessionFactory.setPlugins(new Interceptor[]{pageInterceptor});
    return sessionFactory.getObject();
  }
}
```

最核心的有两点：

- 创建一个SqlSessionFactoryBean，并设置数据源和Mapper.xml路径，其中会解析Mapper.xml文件，最后通过getObject()返回一个SqlSessionFactory 注入Spring容器中
- 通过@MapperScan扫描所有Mapper接口，扫描过程会将Mapper接口生成MapperFactoryBean这个特殊的Bean，并且在其getObject()通过SqlSession().getMapper(this.mapperInterface)生成每个mapper接口真实的代理类

**MapperFactoryBean**

```
//最终注入Spring容器的就是这里的返回对象
public T getObject() throws Exception {
    //获取父类setSqlSessionFactory方法中创建的SqlSessionTemplate
    //通过SqlSessionTemplate获取mapperInterface的代理类
    //我们例子中就是通过SqlSessionTemplate获取com.chenhao.mapper.UserMapper的代理类
    //获取到Mapper接口的代理类后，就把这个Mapper的代理类对象注入Spring容器
    return this.getSqlSession().getMapper(this.mapperInterface);
}
```

接下来我们看看SpringBoot是如何引入Mybatis的

## SpringBoot引入Mybatis

### 添加mybatis依赖

```
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.9</version>
</dependency>
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.2</version>
</dependency>
```

### 全局配置文件中配置数据源和mybatis属性

```
spring:
  datasource:
    url: jdbc:mysql:///springboot
    username: root
    password: admin
    type: com.alibaba.druid.pool.DruidDataSource
    initialSize: 5
    minIdle: 5
    maxActive: 20
mybatis:
  config-location: classpath:mybatis/mybatis-config.xml
  mapper-locations: classpath:mybatis/mapper/*.xml
  type-aliases-package: org.com.cay.spring.boot.entity
```

### 加入Mapper扫描注解@MapperScan

```
@SpringBootApplication
@EnableScheduling
@ServletComponentScan
@MapperScan("com.supplychain.app.mapper")
public class Application {

    public static void main(String[] args) {
        TimeZone.setDefault(TimeZone.getTimeZone("GMT+8"));
        System.setProperty("user.timezone", "GMT+8");
        SpringApplication.run(Application.class, args);
    }
}
```

## 源码解析

### **mybatis-spring-boot-starter**

我们看到mybatis-spring-boot-starter实际上引入了jdbc的场景启动器，这一块我们上一篇文章已经分析过了，还引入了mybatis-spring的依赖，最终还引入了mybatis-spring-boot-autoconfigure这个依赖，其实mybatis-spring-boot-starter只是引入各种需要的依赖，最核心的代码是在引入的mybatis-spring-boot-autoconfigure这个项目当中，我们来看看这个项目

我们看到mybatis-spring-boot-autoconfigure也像spring-boot-autoconfigure一样配置了spring.factories这个配置文件，并且在配置文件中配置了MybatisAutoConfiguration这个自动配置类，我们知道SpringBoot启动时会获取所有spring.factories配置文件中的自动配置类并且进行解析其中的Bean，那么我们就来看看MybatisAutoConfiguration这个自动配置类做了啥？

### MybatisAutoConfiguration

```
 1 @org.springframework.context.annotation.Configuration
 2 @ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
 3 @ConditionalOnBean(DataSource.class)
 4 //引入MybatisProperties配置类
 5 @EnableConfigurationProperties(MybatisProperties.class)
 6 @AutoConfigureAfter(DataSourceAutoConfiguration.class)
 7 public class MybatisAutoConfiguration {
 8 
 9     private final MybatisProperties properties;
10 
11     private final Interceptor[] interceptors;
12 
13     private final ResourceLoader resourceLoader;
14 
15     private final DatabaseIdProvider databaseIdProvider;
16 
17     private final List<ConfigurationCustomizer> configurationCustomizers;
18 
19     public MybatisAutoConfiguration(MybatisProperties properties,
20                                     ObjectProvider<Interceptor[]> interceptorsProvider,
21                                     ResourceLoader resourceLoader,
22                                     ObjectProvider<DatabaseIdProvider> databaseIdProvider,
23                                     ObjectProvider<List<ConfigurationCustomizer>> configurationCustomizersProvider) {
24         this.properties = properties;
25         this.interceptors = interceptorsProvider.getIfAvailable();
26         this.resourceLoader = resourceLoader;
27         this.databaseIdProvider = databaseIdProvider.getIfAvailable();
28         this.configurationCustomizers = configurationCustomizersProvider.getIfAvailable();
29     }
30 
31     @Bean
32     @ConditionalOnMissingBean
33     //往Spring容器中注入SqlSessionFactory对象
34     //并且设置数据源、MapperLocations(Mapper.xml路径)等
35     public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
36         SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
37         factory.setDataSource(dataSource);
38         factory.setVfs(SpringBootVFS.class);
39         if (StringUtils.hasText(this.properties.getConfigLocation())) {
40             factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
41         }
42         Configuration configuration = this.properties.getConfiguration();
43         if (configuration == null && !StringUtils.hasText(this.properties.getConfigLocation())) {
44             configuration = new Configuration();
45         }
46         if (configuration != null && !CollectionUtils.isEmpty(this.configurationCustomizers)) {
47             for (ConfigurationCustomizer customizer : this.configurationCustomizers) {
48                 customizer.customize(configuration);
49             }
50         }
51         factory.setConfiguration(configuration);
52         if (this.properties.getConfigurationProperties() != null) {
53             factory.setConfigurationProperties(this.properties.getConfigurationProperties());
54         }
55         if (!ObjectUtils.isEmpty(this.interceptors)) {
56             factory.setPlugins(this.interceptors);
57         }
58         if (this.databaseIdProvider != null) {
59             factory.setDatabaseIdProvider(this.databaseIdProvider);
60         }
61         if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
62             factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
63         }
64         if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
65             factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
66         }
67         if (!ObjectUtils.isEmpty(this.properties.resolveMapperLocations())) {
68             factory.setMapperLocations(this.properties.resolveMapperLocations());
69         }
70         //获取SqlSessionFactoryBean的getObject()中的对象注入Spring容器，也就是SqlSessionFactory对象
71         return factory.getObject();
72     }
73 
74     @Bean
75     @ConditionalOnMissingBean
76     //往Spring容器中注入SqlSessionTemplate对象
77     public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
78         ExecutorType executorType = this.properties.getExecutorType();
79         if (executorType != null) {
80             return new SqlSessionTemplate(sqlSessionFactory, executorType);
81         } else {
82             return new SqlSessionTemplate(sqlSessionFactory);
83         }
84     }
85     
86     //other code...
87 }
```

在自动配置的时候会导入一个Properties配置类MybatisProperties，咱们来看一下

```
@ConfigurationProperties(prefix = MybatisProperties.MYBATIS_PREFIX)
public class MybatisProperties {

    public static final String MYBATIS_PREFIX = "mybatis";

    /**
     * Location of MyBatis xml config file.
     */
    private String configLocation;

    /**
     * Locations of MyBatis mapper files.
     */
    private String[] mapperLocations;

    /**
     * Packages to search type aliases. (Package delimiters are ",; \t\n")
     */
    private String typeAliasesPackage;

    /**
     * Packages to search for type handlers. (Package delimiters are ",; \t\n")
     */
    private String typeHandlersPackage;

    /**
     * Indicates whether perform presence check of the MyBatis xml config file.
     */
    private boolean checkConfigLocation = false;

    /**
     * Execution mode for {@link org.mybatis.spring.SqlSessionTemplate}.
     */
    private ExecutorType executorType;

    /**
     * Externalized properties for MyBatis configuration.
     */
    private Properties configurationProperties;

    /**
     * A Configuration object for customize default settings. If {@link #configLocation}
     * is specified, this property is not used.
     */
    @NestedConfigurationProperty
    private Configuration configuration;

    //other code...
}
```

该Properties配置类作用主要用于与yml/properties中以mybatis开头的属性进行一一对应，如下

```
mybatis:
  config-location: classpath:mybatis/mybatis-config.xml
  mapper-locations: classpath:mybatis/mapper/*.xml
  type-aliases-package: org.com.cay.spring.boot.entity
```

在**MybatisAutoConfiguration**自动配置类中，SpringBoot默认自动配置了两个Bean，分别是**SqlSessionFactory**和**SqlSessionTemplate**。我们看到上面代码中第71行，其实是返回的factory.getObject();，也就是注入Spring容器中的是SqlSessionFactory对象，SqlSessionFactory主要是将Properties配置类中的属性赋值到SqlSessionFactoryBean中，类似以前xml中配置的SqlSessionFactory

```
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"></property>
        <!-- 自动扫描mapping.xml文件 -->
        <property name="mapperLocations" value="classpath:com/cn/mapper/*.xml"></property>
        ...
</bean>
```

另外一个Bean为SqlSessionTemplate，通过SqlSessionFactory来生成SqlSession代理类：

```
public class SqlSessionTemplate implements SqlSession, DisposableBean {

    private final SqlSessionFactory sqlSessionFactory;

    private final ExecutorType executorType;

    private final SqlSession sqlSessionProxy;

    private final PersistenceExceptionTranslator exceptionTranslator;

    //other code...
    
    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
          PersistenceExceptionTranslator exceptionTranslator) {

        notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
        notNull(executorType, "Property 'executorType' is required");

        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;
        this.exceptionTranslator = exceptionTranslator;
        
        //生成SqlSessioin代理类
        this.sqlSessionProxy = (SqlSession) newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[] { SqlSession.class },
            new SqlSessionInterceptor());
    }
}
```

而@MapperScan注解是和Spring整合Mybatis的使用是一样的，都是在配置类上指定Mapper接口的路径，大家可以看一下我以前的一篇文章[Mybaits 源码解析 （十一）----- @MapperScan将Mapper接口生成代理注入到Spring-静态代理和动态代理结合使用](https://www.cnblogs.com/java-chen-hao/p/11839958.html)

# 集成AOP

**正文**

本篇主要集成Sping一个重要功能AOP

我们还是先回顾一下以前Spring中是如何使用AOP的，大家可以看看我这篇文章[spring5 源码深度解析----- AOP的使用及AOP自定义标签](https://www.cnblogs.com/java-chen-hao/p/11589531.html)

## Spring中使用AOP



### 引入Aspect

```
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>${aspectj.version}</version>
</dependency>
```



### 创建用于拦截的bean

```
public class TestBean {
    private String message = "test bean";

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public void test(){
        System.out.println(this.message);
    }
}
```



### 创建Advisor

```
@Aspect
public class AspectJTest {
    @Pointcut("execution(* *.test(..))")
    public void test(){
    }
    
    @Before("test()")
    public void beforeTest(){
        System.out.println("beforeTest");
    }
    
    @Around("test()")
    public Object aroundTest(ProceedingJoinPoint p){
        System.out.println("around.....before");
        Object o = null;
        try{
            o = p.proceed();
        }catch(Throwable e){
            e.printStackTrace();
        }
        System.out.println("around.....after");
        return o;
    }
    
    @After("test()")
    public void afterTest()
    {
        System.out.println("afterTest");
    }
 }
```



### 创建配置文件

要在Spring中开启AOP功能，还需要在配置文件中作如下声明，开启AOP：

```
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <aop:aspectj-autoproxy/>
    <bean id="test" class="com.yhl.myspring.demo.aop.TestBean">
        <property name="message" value="这是一个苦逼的程序员"/>
    </bean>
    <bean id="aspect" class="com.yhl.myspring.demo.aop.AspectJTest"/>
</beans>
```



### 注解开启AOP

开启AOP<aop:aspectj-autoproxy/>也可以使用注解的方式，如下，使用**@EnableAspectJAutoProxy**配置在任何一个@Configratrion或者@Component上

## SpringBoot集成AOP

### 添加pom依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

引入了AOP的场景启动器，我们点击去看看

还是引入了spring-aop和aspectj的依赖，和我们Spring集成AOP是引入了相同的包，接着我们直接就可以创建Advisor了，如上AspectJTest这个类，但是我们并没有通过**@EnableAspectJAutoProxy开启AOP呢？那是因为AOP的自动配置类帮我们开启了**

### AopAutoConfiguration

一旦导入了spring-boot-starter-aop依赖后，SpringBoot就会启动AOP的自动配置类AopAutoConfiguration：

我们来看看AopAutoConfiguration这个自动配置类

```
@Configuration
@ConditionalOnClass({ EnableAspectJAutoProxy.class, Aspect.class, Advice.class })
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

    @Configuration
    //使用注解开启AOP功能
    @EnableAspectJAutoProxy(proxyTargetClass = false)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = true)
    public static class JdkDynamicAutoProxyConfiguration {

    }

    @Configuration
    //使用注解开启AOP功能
    @EnableAspectJAutoProxy(proxyTargetClass = true)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = false)
    public static class CglibAutoProxyConfiguration {

    }

}
```

不管使用jdk代理还是cglib代理，都有@EnableAspectJAutoProxy注解，所以只要导入spring-boot-starter-aop依赖后，就自动帮我们开启了AOP，我们可以直接添加切面使用AOP了

@EnableAspectJAutoProxy这个注解是整个AOP的灵魂，其作用和aop:aspectj-autoproxy/是一样的

# [基于SpringBoot的Environment源码理解实现分散配置](https://www.cnblogs.com/throwable/p/9411100.html)

# 前提

org.springframework.core.env.Environment是当前应用运行环境的公开接口，主要包括应用程序运行环境的两个关键方面：配置文件(profiles)和属性。Environment继承自接口PropertyResolver，而PropertyResolver提供了属性访问的相关方法。这篇文章从源码的角度分析Environment的存储容器和加载流程，然后基于源码的理解给出一个生产级别的扩展。

本文较长，请用一个舒服的姿势阅读。

**本文已经转移到个人博客中维护，因为维护多个地方的内容太麻烦**：

- coding-page：[基于SpringBoot的Environment源码理解实现分散配置](http://throwable.coding.me/2018/12/16/spring-boot-environment-configuration-spread)
- github-page：[基于SpringBoot的Environment源码理解实现分散配置](https://zjcscut.github.io/2018/12/16/spring-boot-environment-configuration-spread)

# Environment类体系

[![spe-1](http://pazkqls86.bkt.clouddn.com/spe-1.png)](http://pazkqls86.bkt.clouddn.com/spe-1.png)

- PropertyResolver：提供属性访问功能。
- ConfigurablePropertyResolver：继承自PropertyResolver，主要提供属性类型转换(基于org.springframework.core.convert.ConversionService)功能。
- Environment：继承自PropertyResolver，提供访问和判断profiles的功能。
- ConfigurableEnvironment：继承自ConfigurablePropertyResolver和Environment，并且提供设置激活的profile和默认的profile的功能。
- ConfigurableWebEnvironment：继承自ConfigurableEnvironment，并且提供配置Servlet上下文和Servlet参数的功能。
- AbstractEnvironment：实现了ConfigurableEnvironment接口，默认属性和存储容器的定义，并且实现了ConfigurableEnvironment种的方法，并且为子类预留可覆盖了扩展方法。
- StandardEnvironment：继承自AbstractEnvironment，非Servlet(Web)环境下的标准Environment实现。
- StandardServletEnvironment：继承自StandardEnvironment，Servlet(Web)环境下的标准Environment实现。

reactive相关的暂时不研究。

# Environment提供的方法

一般情况下，我们在SpringMVC项目中启用到的是StandardServletEnvironment，它的父接口问ConfigurableWebEnvironment，我们可以查看此接口提供的方法：

[![spe-2](http://pazkqls86.bkt.clouddn.com/spe-2.png)](http://pazkqls86.bkt.clouddn.com/spe-2.png)

# Environment的存储容器

Environment的静态属性和存储容器都是在AbstractEnvironment中定义的，ConfigurableWebEnvironment接口提供的`getPropertySources()`方法可以获取到返回的MutablePropertySources实例，然后添加额外的PropertySource。实际上，Environment的存储容器就是org.springframework.core.env.PropertySource的子类集合，AbstractEnvironment中使用的实例是org.springframework.core.env.MutablePropertySources，下面看下PropertySource的源码：

```java
public abstract class PropertySource<T> {

	protected final Log logger = LogFactory.getLog(getClass());

	protected final String name;

	protected final T source;

    public PropertySource(String name, T source) {
		Assert.hasText(name, "Property source name must contain at least one character");
		Assert.notNull(source, "Property source must not be null");
		this.name = name;
		this.source = source;
	}

    @SuppressWarnings("unchecked")
	public PropertySource(String name) {
		this(name, (T) new Object());
	}

    public String getName() {
		return this.name;
	}

	public T getSource() {
		return this.source;
	} 

	public boolean containsProperty(String name) {
		return (getProperty(name) != null);
	} 

	@Nullable
	public abstract Object getProperty(String name);     

 	@Override
	public boolean equals(Object obj) {
		return (this == obj || (obj instanceof PropertySource &&
				ObjectUtils.nullSafeEquals(this.name, ((PropertySource<?>) obj).name)));
	}  

	@Override
	public int hashCode() {
		return ObjectUtils.nullSafeHashCode(this.name);
	}  
//省略其他方法和内部类的源码            
}
```

源码相对简单，预留了一个`getProperty`抽象方法给子类实现，**重点需要关注的是覆写了的`equals`和`hashCode`方法，实际上只和`name`属性相关，这一点很重要，说明一个PropertySource实例绑定到一个唯一的name，这个name有点像HashMap里面的key**，部分移除、判断方法都是基于name属性。PropertySource的最常用子类是MapPropertySource、PropertiesPropertySource、ResourcePropertySource、StubPropertySource、ComparisonPropertySource：

- MapPropertySource：source指定为Map实例的PropertySource实现。
- PropertiesPropertySource：source指定为Map实例的PropertySource实现，内部的Map实例由Properties实例转换而来。
- ResourcePropertySource：继承自PropertiesPropertySource，source指定为通过Resource实例转化为Properties再转换为Map实例。
- StubPropertySource：PropertySource的一个内部类，source设置为null，实际上就是空实现。
- ComparisonPropertySource：继承自ComparisonPropertySource，所有属性访问方法强制抛出异常，作用就是一个不可访问属性的空实现。

AbstractEnvironment中的属性定义：

```java
public static final String IGNORE_GETENV_PROPERTY_NAME = "spring.getenv.ignore";
public static final String ACTIVE_PROFILES_PROPERTY_NAME = "spring.profiles.active";
public static final String DEFAULT_PROFILES_PROPERTY_NAME = "spring.profiles.default";
protected static final String RESERVED_DEFAULT_PROFILE_NAME = "default";

private final Set<String> activeProfiles = new LinkedHashSet<>();

private final Set<String> defaultProfiles = new LinkedHashSet<>(getReservedDefaultProfiles());

private final MutablePropertySources propertySources = new MutablePropertySources(this.logger);

private final ConfigurablePropertyResolver propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);
```

上面的propertySources(MutablePropertySources类型)属性就是用来存放PropertySource列表的，PropertySourcesPropertyResolver是ConfigurablePropertyResolver的实现，默认的profile就是字符串default。MutablePropertySources的内部属性如下：

```java
private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();
```

没错，这个就是最底层的存储容器，也就是环境属性都是存放在一个CopyOnWriteArrayList<PropertySource<?>>实例中。MutablePropertySources是PropertySources的子类，它提供了`get(String name)`、`addFirst`、`addLast`、`addBefore`、`addAfter`、`remove`、`replace`等便捷方法，方便操作propertySourceList集合的元素，这里挑选`addBefore`的源码分析：

```java
public void addBefore(String relativePropertySourceName, PropertySource<?> propertySource) {
	if (logger.isDebugEnabled()) {
		logger.debug("Adding PropertySource '" + propertySource.getName() +
					"' with search precedence immediately higher than '" + relativePropertySourceName + "'");
	}
    //前一个PropertySource的name指定为relativePropertySourceName时候必须和添加的PropertySource的name属性不相同
	assertLegalRelativeAddition(relativePropertySourceName, propertySource);
    //尝试移除同名的PropertySource
	removeIfPresent(propertySource);
    //获取前一个PropertySource在CopyOnWriteArrayList中的索引
	int index = assertPresentAndGetIndex(relativePropertySourceName);
    //添加当前传入的PropertySource到指定前一个PropertySource的索引，相当于relativePropertySourceName对应的PropertySource后移到原来索引值+1的位置
	addAtIndex(index, propertySource);
}

protected void assertLegalRelativeAddition(String relativePropertySourceName, PropertySource<?> propertySource) {
	String newPropertySourceName = propertySource.getName();
	if (relativePropertySourceName.equals(newPropertySourceName)) {
		throw new IllegalArgumentException(
					"PropertySource named '" + newPropertySourceName + "' cannot be added relative to itself");
	}
}

protected void removeIfPresent(PropertySource<?> propertySource) {
	this.propertySourceList.remove(propertySource);
}

private int assertPresentAndGetIndex(String name) {
	int index = this.propertySourceList.indexOf(PropertySource.named(name));
	if (index == -1) {
		throw new IllegalArgumentException("PropertySource named '" + name + "' does not exist");
	}
	return index;
}

private void addAtIndex(int index, PropertySource<?> propertySource) {
    //注意，这里会再次尝试移除同名的PropertySource
	removeIfPresent(propertySource);
	this.propertySourceList.add(index, propertySource);
}
```

大多数PropertySource子类的修饰符都是public，可以直接使用，这里写个小demo：

```java
MutablePropertySources mutablePropertySources = new MutablePropertySources();
Map<String, Object> map = new HashMap<>(8);
map.put("name", "throwable");
map.put("age", 25);
MapPropertySource mapPropertySource = new MapPropertySource("map", map);
mutablePropertySources.addLast(mapPropertySource);
Properties properties = new Properties();
PropertiesPropertySource propertiesPropertySource = new PropertiesPropertySource("prop", properties);
properties.put("name", "doge");
properties.put("gourp", "group-a");
mutablePropertySources.addBefore("map", propertiesPropertySource);
System.out.println(mutablePropertySources);
```

# Environment加载过程源码分析

Environment加载的源码位于`SpringApplication#prepareEnvironment`：

```java
	private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
        //创建ConfigurableEnvironment实例
		ConfigurableEnvironment environment = getOrCreateEnvironment();
        //启动参数绑定到ConfigurableEnvironment中
		configureEnvironment(environment, applicationArguments.getSourceArgs());
        //发布ConfigurableEnvironment准备完毕事件
		listeners.environmentPrepared(environment);
        //绑定ConfigurableEnvironment到当前的SpringApplication实例中
		bindToSpringApplication(environment);
        //这一步是非SpringMVC项目的处理，暂时忽略
		if (this.webApplicationType == WebApplicationType.NONE) {
			environment = new EnvironmentConverter(getClassLoader())
					.convertToStandardEnvironmentIfNecessary(environment);
		}
        //绑定ConfigurationPropertySourcesPropertySource到ConfigurableEnvironment中，name为configurationProperties，实例是SpringConfigurationPropertySources，属性实际是ConfigurableEnvironment中的MutablePropertySources
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```

这里重点看下`getOrCreateEnvironment`方法：

```java
private ConfigurableEnvironment getOrCreateEnvironment() {
	if (this.environment != null) {
		return this.environment;
	}
    //在SpringMVC项目，ConfigurableEnvironment接口的实例就是新建的StandardServletEnvironment实例
	if (this.webApplicationType == WebApplicationType.SERVLET) {
		return new StandardServletEnvironment();
	}
	return new StandardEnvironment();
}
//REACTIVE_WEB_ENVIRONMENT_CLASS=org.springframework.web.reactive.DispatcherHandler
//MVC_WEB_ENVIRONMENT_CLASS=org.springframework.web.servlet.DispatcherServlet
//MVC_WEB_ENVIRONMENT_CLASS={"javax.servlet.Servlet","org.springframework.web.context.ConfigurableWebApplicationContext"}
//这里，默认就是WebApplicationType.SERVLET
private WebApplicationType deduceWebApplicationType() {
	if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)
		&& !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)) {
		return WebApplicationType.REACTIVE;
	}
	for (String className : WEB_ENVIRONMENT_CLASSES) {
		if (!ClassUtils.isPresent(className, null)) {
			return WebApplicationType.NONE;
		}
	}
	return WebApplicationType.SERVLET;
}
```

还有一个地方要重点关注：发布ConfigurableEnvironment准备完毕事件`listeners.environmentPrepared(environment)`，实际上这里用到了同步的EventBus，事件的监听者是ConfigFileApplicationListener，具体处理逻辑是`onApplicationEnvironmentPreparedEvent`方法：

```java
private void onApplicationEnvironmentPreparedEvent(
			ApplicationEnvironmentPreparedEvent event) {
	List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
	postProcessors.add(this);
	AnnotationAwareOrderComparator.sort(postProcessors);
    //遍历所有的EnvironmentPostProcessor对Environment实例进行处理
	for (EnvironmentPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessEnvironment(event.getEnvironment(),
					event.getSpringApplication());
	}
}

//从spring.factories文件中加载，一共有四个实例
//ConfigFileApplicationListener
//CloudFoundryVcapEnvironmentPostProcessor
//SpringApplicationJsonEnvironmentPostProcessor
//SystemEnvironmentPropertySourceEnvironmentPostProcessor
List<EnvironmentPostProcessor> loadPostProcessors() {
	return SpringFactoriesLoader.loadFactories(EnvironmentPostProcessor.class,
				getClass().getClassLoader());
}
```

实际上，处理工作大部分都在ConfigFileApplicationListener中，见它的`postProcessEnvironment`方法：

```java
public void postProcessEnvironment(ConfigurableEnvironment environment,
			SpringApplication application) {
	addPropertySources(environment, application.getResourceLoader());
}

protected void addPropertySources(ConfigurableEnvironment environment,
			ResourceLoader resourceLoader) {
	RandomValuePropertySource.addToEnvironment(environment);
	new Loader(environment, resourceLoader).load();
}
```

主要的配置环境加载逻辑在内部类Loader，Loader会匹配多个路径下的文件把属性加载到ConfigurableEnvironment中，加载器主要是PropertySourceLoader的实例，例如我们用到application-${profile}.yaml文件做应用主配置文件，使用的是YamlPropertySourceLoader，这个时候activeProfiles也会被设置到ConfigurableEnvironment中。加载完毕之后，ConfigurableEnvironment中基本包含了所有需要加载的属性(activeProfiles是这个时候被写入ConfigurableEnvironment)。值得注意的是，几乎所有属性都是key-value形式存储，如xxx.yyyy.zzzzz=value、xxx.yyyy[0].zzzzz=value-1、xxx.yyyy[1].zzzzz=value-2。`Loader`中的逻辑相对复杂，有比较多的遍历和过滤条件，这里不做展开。

# Environment属性访问源码分析

上文提到过，都是委托到PropertySourcesPropertyResolver，先看它的构造函数：

```java
@Nullable
private final PropertySources propertySources;

public PropertySourcesPropertyResolver(@Nullable PropertySources propertySources) {
		this.propertySources = propertySources;
	}
```

只依赖于一个PropertySources实例，在SpringBoot的SpringMVC项目中就是MutablePropertySources的实例。重点分析一下最复杂的一个方法：

```java
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
	if (this.propertySources != null) {
        //遍历所有的PropertySource
		for (PropertySource<?> propertySource : this.propertySources) {
			if (logger.isTraceEnabled()) {
				logger.trace("Searching for key '" + key + "' in PropertySource '" +
							propertySource.getName() + "'");
			}
			Object value = propertySource.getProperty(key);
            //选用第一个不为null的匹配key的属性值
			if (value != null) {
				if (resolveNestedPlaceholders && value instanceof String) {
                    //处理属性占位符，如${server.port}，底层委托到PropertyPlaceholderHelper完成
					value = resolveNestedPlaceholders((String) value);
				}
				logKeyFound(key, propertySource, value);
                //如果需要的话，进行一次类型转换，底层委托到DefaultConversionService完成
				return convertValueIfNecessary(value, targetValueType);
			}
		}
	}
	if (logger.isDebugEnabled()) {
		logger.debug("Could not find key '" + key + "' in any property source");
	}
	return null;
}
```

这里的源码告诉我们，如果出现多个PropertySource中存在同名的key，返回的是第一个PropertySource对应key的属性值的处理结果，因此我们如果需要自定义一些环境属性，需要十分清楚各个PropertySource的顺序。

# 扩展-实现分散配置

在不使用SpringCloud配置中心的情况下，一般的SpringBoot项目的配置文件如下：

```markdown
- src
 - main
  - resources
   - application-prod.yaml
   - application-dev.yaml
   - application-test.yaml
```

随着项目发展，配置项越来越多，导致了application-${profile}.yaml迅速膨胀，大的配置文件甚至超过一千行，为了简化和划分不同功能的配置，可以考虑把配置文件拆分如下：

```markdown
- src
 - main
  - resources
   - profiles
     - dev
       - business.yaml
       - mq.json
       - datasource.properties
     - prod
       - business.yaml
       - mq.json
       - datasource.properties
     - test  
       - business.yaml
       - mq.json  
       - datasource.properties
   - application-prod.yaml
   - application-dev.yaml
   - application-test.yaml
```

外层的application-${profile}.yaml只留下项目的核心配置如server.port等，其他配置打散放在/profiles/${profile}/各自的配置文件中。实现方式是：依据当前配置的spring.profiles.active属性，读取类路径中指定文件夹下的配置文件中，加载到Environment中，需要注意这一个加载步骤必须在Spring刷新上下文方法最后一步`finishRefresh`之前完成(这一点原因可以参考之前在[个人博客](http://throwable.club/)写过的SpringBoot刷新上下文源码的分析)，否则有可能会影响到占位符属性的自动装配(例如使用了@Value("${filed}"))。

先定义一个属性探索者接口：

```java
public interface PropertySourceDetector {

    /**
     * 获取支持的文件后缀数组
     *
     * @return String[]
     */
    String[] getFileExtensions();

    /**
     * 加载目标文件属性到环境中
     *
     * @param environment environment
     * @param name        name
     * @param resource    resource
     * @throws IOException IOException
     */
    void load(ConfigurableEnvironment environment, String name, Resource resource) throws IOException;
}
```

然后需要一个抽象属性探索者把Resource转换为字符串，额外提供Map的缩进、添加PropertySource到Environment等方法：

```java
public abstract class AbstractPropertySourceDetector implements PropertySourceDetector {

    private static final String SERVLET_ENVIRONMENT_CLASS = "org.springframework.web."
            + "context.support.StandardServletEnvironment";

    public boolean support(String fileExtension) {
        String[] fileExtensions = getFileExtensions();
        return null != fileExtensions &&
                Arrays.stream(fileExtensions).anyMatch(extension -> extension.equals(fileExtension));
    }

    private String findPropertySource(MutablePropertySources sources) {
        if (ClassUtils.isPresent(SERVLET_ENVIRONMENT_CLASS, null) && sources
                .contains(StandardServletEnvironment.JNDI_PROPERTY_SOURCE_NAME)) {
            return StandardServletEnvironment.JNDI_PROPERTY_SOURCE_NAME;
        }
        return StandardEnvironment.SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME;
    }

    protected void addPropertySource(ConfigurableEnvironment environment, PropertySource<?> source) {
        MutablePropertySources sources = environment.getPropertySources();
        String name = findPropertySource(sources);
        if (sources.contains(name)) {
            sources.addBefore(name, source);
        } else {
            sources.addFirst(source);
        }
    }

    protected Map<String, Object> flatten(Map<String, Object> map) {
        Map<String, Object> result = new LinkedHashMap<>();
        flatten(null, result, map);
        return result;
    }

    private void flatten(String prefix, Map<String, Object> result, Map<String, Object> map) {
        String namePrefix = (prefix != null ? prefix + "." : "");
        map.forEach((key, value) -> extract(namePrefix + key, result, value));
    }

    @SuppressWarnings("unchecked")
    private void extract(String name, Map<String, Object> result, Object value) {
        if (value instanceof Map) {
            flatten(name, result, (Map<String, Object>) value);
        } else if (value instanceof Collection) {
            int index = 0;
            for (Object object : (Collection<Object>) value) {
                extract(name + "[" + index + "]", result, object);
                index++;
            }
        } else {
            result.put(name, value);
        }
    }

    protected String getContentStringFromResource(Resource resource) throws IOException {
        return StreamUtils.copyToString(resource.getInputStream(), Charset.forName("UTF-8"));
    }
}
```

上面的方法参考SpringApplicationJsonEnvironmentPostProcessor，然后编写各种类型配置属性探索者的实现：

```java
//Json
@Slf4j
public class JsonPropertySourceDetector extends AbstractPropertySourceDetector {

    private static final JsonParser JSON_PARSER = JsonParserFactory.getJsonParser();

    @Override
    public String[] getFileExtensions() {
        return new String[]{"json"};
    }

    @Override
    public void load(ConfigurableEnvironment environment, String name, Resource resource) throws IOException {
        try {
            Map<String, Object> map = JSON_PARSER.parseMap(getContentStringFromResource(resource));
            Map<String, Object> target = flatten(map);
            addPropertySource(environment, new MapPropertySource(name, target));
        } catch (Exception e) {
            log.warn("加载Json文件属性到环境变量失败,name = {},resource = {}", name, resource);
        }
    }
}
//Properties
public class PropertiesPropertySourceDetector extends AbstractPropertySourceDetector {

    @Override
    public String[] getFileExtensions() {
        return new String[]{"properties", "conf"};
    }

    @SuppressWarnings("unchecked")
    @Override
    public void load(ConfigurableEnvironment environment, String name, Resource resource) throws IOException {
        Map map = PropertiesLoaderUtils.loadProperties(resource);
        addPropertySource(environment, new MapPropertySource(name, map));
    }
}
//Yaml
@Slf4j
public class YamlPropertySourceDetector extends AbstractPropertySourceDetector {

    private static final JsonParser YAML_PARSER = new YamlJsonParser();

    @Override
    public String[] getFileExtensions() {
        return new String[]{"yaml", "yml"};
    }

    @Override
    public void load(ConfigurableEnvironment environment, String name, Resource resource) throws IOException {
        try {
            Map<String, Object> map = YAML_PARSER.parseMap(getContentStringFromResource(resource));
            Map<String, Object> target = flatten(map);
            addPropertySource(environment, new MapPropertySource(name, target));
        } catch (Exception e) {
            log.warn("加载Yaml文件属性到环境变量失败,name = {},resource = {}", name, resource);
        }
    }
}
```

子类的全部PropertySource都是MapPropertySource，name为文件的名称，所有PropertySource都用`addBefore`方法插入到`systemProperties`的前面，主要是为了提高匹配属性的优先级。接着需要定义一个属性探索者的合成类用来装载所有的子类：

```java
public class PropertySourceDetectorComposite implements PropertySourceDetector {

    private static final String DEFAULT_SUFFIX = "properties";
    private final List<AbstractPropertySourceDetector> propertySourceDetectors = new ArrayList<>();

    public void addPropertySourceDetector(AbstractPropertySourceDetector sourceDetector) {
        propertySourceDetectors.add(sourceDetector);
    }

    public void addPropertySourceDetectors(List<AbstractPropertySourceDetector> sourceDetectors) {
        propertySourceDetectors.addAll(sourceDetectors);
    }

    public List<AbstractPropertySourceDetector> getPropertySourceDetectors() {
        return Collections.unmodifiableList(propertySourceDetectors);
    }

    @Override
    public String[] getFileExtensions() {
        List<String> fileExtensions = new ArrayList<>(8);
        for (AbstractPropertySourceDetector propertySourceDetector : propertySourceDetectors) {
            fileExtensions.addAll(Arrays.asList(propertySourceDetector.getFileExtensions()));
        }
        return fileExtensions.toArray(new String[0]);
    }

    @Override
    public void load(ConfigurableEnvironment environment, String name, Resource resource) throws IOException {
        if (resource.isFile()) {
            String fileName = resource.getFile().getName();
            int index = fileName.lastIndexOf(".");
            String suffix;
            if (-1 == index) {
                //如果文件没有后缀,当作properties处理
                suffix = DEFAULT_SUFFIX;
            } else {
                suffix = fileName.substring(index + 1);
            }
            for (AbstractPropertySourceDetector propertySourceDetector : propertySourceDetectors) {
                if (propertySourceDetector.support(suffix)) {
                    propertySourceDetector.load(environment, name, resource);
                    return;
                }
            }
        }
    }
}
```

最后添加一个配置类作为入口：

```java
public class PropertySourceDetectorConfiguration implements ImportBeanDefinitionRegistrar {

    private static final String PATH_PREFIX = "profiles";

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) registry;
        ConfigurableEnvironment environment = beanFactory.getBean(ConfigurableEnvironment.class);
        List<AbstractPropertySourceDetector> propertySourceDetectors = new ArrayList<>();
        configurePropertySourceDetectors(propertySourceDetectors, beanFactory);
        PropertySourceDetectorComposite propertySourceDetectorComposite = new PropertySourceDetectorComposite();
        propertySourceDetectorComposite.addPropertySourceDetectors(propertySourceDetectors);
        String[] activeProfiles = environment.getActiveProfiles();
        ResourcePatternResolver resourcePatternResolver = new PathMatchingResourcePatternResolver();
        try {
            for (String profile : activeProfiles) {
                String location = PATH_PREFIX + File.separator + profile + File.separator + "*";
                Resource[] resources = resourcePatternResolver.getResources(location);
                for (Resource resource : resources) {
                    propertySourceDetectorComposite.load(environment, resource.getFilename(), resource);
                }
            }
        } catch (IOException e) {
            throw new IllegalStateException(e);
        }
    }

    private void configurePropertySourceDetectors(List<AbstractPropertySourceDetector> propertySourceDetectors,
                                                  DefaultListableBeanFactory beanFactory) {
        Map<String, AbstractPropertySourceDetector> beansOfType = beanFactory.getBeansOfType(AbstractPropertySourceDetector.class);
        for (Map.Entry<String, AbstractPropertySourceDetector> entry : beansOfType.entrySet()) {
            propertySourceDetectors.add(entry.getValue());
        }
        propertySourceDetectors.add(new JsonPropertySourceDetector());
        propertySourceDetectors.add(new YamlPropertySourceDetector());
        propertySourceDetectors.add(new PropertiesPropertySourceDetector());
    }
}
```

准备就绪，在/resources/profiles/dev下面添加两个文件app.json和conf：

```java
//app.json
{
  "app": {
    "name": "throwable",
    "age": 25
  }
}
//conf
name=doge
```

项目的application.yaml添加属性spring.profiles.active: dev，最后添加一个CommandLineRunner的实现用来观察数据：

```java
@Slf4j
@Component
public class CustomCommandLineRunner implements CommandLineRunner {

    @Value("${app.name}")
    String name;
    @Value("${app.age}")
    Integer age;
    @Autowired
    ConfigurableEnvironment configurableEnvironment;

    @Override
    public void run(String... args) throws Exception {
        log.info("name = {},age = {}", name, age);
    }
}
```

[![spe-3](http://pazkqls86.bkt.clouddn.com/spe-3.png)](http://pazkqls86.bkt.clouddn.com/spe-3.png)

自动装配的属性值和Environment实例中的属性和预期一样，改造是成功的。

# 小结

Spring中的环境属性管理的源码个人认为是最清晰和简单的：从文件中读取数据转化为key-value结构，key-value结构存放在一个PropertySource实例中，然后得到的多个PropertySource实例存放在一个CopyOnWriteArrayList中，属性访问的时候总是遍历CopyOnWriteArrayList中的PropertySource进行匹配。可能相对复杂的就是占位符的解析和参数类型的转换，后者牵连到Converter体系，这些不在本文的讨论范围内。最后附上一张Environment存储容器的示例图：

# Spring面试题

## SpringBoot配置扫描包范围

java的spring框架，得知使用注解需要配置包扫描的范围，然而在SpringBoot项目中的配置文件里找不到如spring类似的配置

```
<context:component-scan base-package=”XX.XX”/> 
```

经查阅资料SpringBoot其实有默认的包扫描机制，启动类所在的当前包以及包的子类都会默认被扫描，所以新手在学习这个框架的时候，有时候可能因为bean和启动类不在一个文件夹下导致扫描不到引起的注解失败问题。

> 启动类：项目的入口函数，一般命名规范是xxxApplication.java，并且带有@SpringBootApplication的注解，也有我们常见的java中的main函数。

**如何修改包扫描的位置？**

1. 方法一：

在启动类的SpringBootApplication注解中配置scanBasePackages即可，如下

```
@SpringBootApplication(scanBasePackages = "org.demo.service")
```

也可以配置多个包路径

```
@SpringBootApplication(scanBasePackages = {"org.demo.bean","org.demo.service"})
```

1. 方法二：

在启动类里添加@ComponentScan注解配置basePackages

```
@ComponentScan(basePackages = {"org.demo.bean","org.demo.service"})
```

两个配置方法选择其一即可。

## 如何管理不在Springboot包扫描路径下的Bean？

方法一、在Spring Boot Application 主类上 使用@Import 注解

方法二、创建spring.factories文件



　