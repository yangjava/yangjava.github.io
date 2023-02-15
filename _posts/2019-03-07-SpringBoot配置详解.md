---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot配置
SpringBoot使用一个全局的配置文件，配置文件名称固定。

## SpringBoot配置文件
SpringBoot可以识别两种格式的配置文件，分别是yml文件与properties文件，我们可以将application.properties文件换成application.yml，这两个文件都可以被SpringBoot自动识别并加载，但是如果是自定义的配置文件，就最好还是使用properties格式的文件，因为SpringBoot中暂时还并未提供手动加载yml格式文件的功能

application.properties配置文件欲被SpringBoot自动加载，需要放置到指定的位置：src/main/resource目录下，一般自定义的配置文件也位于此目录之下。

## application.yml的配置
配置类智能提示
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

## @Value
普通变量获取配置信息
可以通过@Value注解在字段上给字段赋值，@Value与bean标签中的value配置作用一致，因此用法也一致。
```java
@Component
public class Person {
    /**
     * 1. @Value类似于bean标签中的value配置;
     * 2. bean中的value可以配置：字面量;${key}获取环境变量、配置文件中的key值,#{SpEl}spring的EL表达式;
     * 因此@Value也可以配置上面的三种值。
     * 3. @Value获取配置信息,可以不写get/set方法。
     * <bean id = "person" class="xxxxx">
     *     <property name="address" value="xxx"></property>
     * </bean>
     */
    @Value("小明")
    private String name;
    @Value("#{11*2}")
    private int age;
    @Value("${person.address}")
    private String address;
    private List<Object> like;
    private Map<String, Object> maps;
    private Dog dog;
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
    public String getAddress() {
        return address;
    }
    public void setAddress(String address) {
        this.address = address;
    }
    public List<Object> getLike() {
        return like;
    }
    public void setLike(List<Object> like) {
        this.like = like;
    }
    public Map<String, Object> getMaps() {
        return maps;
    }
    public void setMaps(Map<String, Object> maps) {
        this.maps = maps;
    }
    public Dog getDog() {
        return dog;
    }
    public void setDog(Dog dog) {
        this.dog = dog;
    }
    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", address='" + address + '\'' +
                ", like=" + like +
                ", maps=" + maps +
                ", dog=" + dog +
                '}';
    }
}
```

## @Value注解配置默认值
@Value在注解模式下读取配置文件注入属性值
```java
	@Value("${name}")	
	private String name;
```
但是，如果配置文件中没有设置name的值，spring在启动的时候会报错。这时需要给name配置默认值，代码如下：
```java
    @Value("${name:bob}")
	private String name;
```
除了String类型外，其他类型也可配置默认值：
```java
    @Value("${age:18}")
	private int age;
```

## @Value注入列表或者数组
可以使用split()方法在一行中注入“列表”。
配置如下：
config.properties
```properties
server.name=hydra,zeus
server.id=100,102,103
```
AppConfigTest.java
```java
 
import java.util.List;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.context.support.PropertySourcesPlaceholderConfigurer;
 
@Configuration
@PropertySource(value="classpath:config.properties")
public class AppConfigTest {
	
	@Value("#{'${server.name}'.split(',')}")
	private List<String> servers;
	
	@Value("#{'${server.id}'.split(',')}")
	private List<Integer> serverId;
	
	//To resolve ${} in @Value
	@Bean
	public static PropertySourcesPlaceholderConfigurer propertyConfigInDev() {
		return new PropertySourcesPlaceholderConfigurer();
	}

}
```

## 注意如果配置项已逗号分隔，无需使用split方法，spring默认支持逗号的分隔。
可以指定默认值，下面的3种写法Spring都是支持的。
```java
	@Value("#{'${server.id:127.0.0.1,192.168.1.18}'.split(',')}")
	private List<String> serverId;
 
	@Value("${server.id:127.0.0.1,192.168.1.18}")
	private List<String> serverId;
	
	@Value("${server.id:127.0.0.1,192.168.1.18}")
	private String[] serverId;

```

## @Value给静态变量注入值
@Value不能直接给静态变量赋值，即使编译不报错，但是结果赋值不成功。
```java
    @Value("${fastdfs.cache.redis.expireTime:86400}")
    public static int EXPIRE_TIME;
```
静态变量赋值，@Value注解需要放在方法上
类上加上@Component注解，方法名（例如setExpireTime，方法名可以随便写，通常以set开头）和参数名（例如expireTime，参数名也可以随便写），如下所示：
```java
    /**
     * 缓存过期时间，单位秒
     */
    public static int EXPIRE_TIME;

    /**
     * 
     * @param expireTime 过期时间
     */
    @Value("${fastdfs.cache.redis.expireTime:86400}")
    public void setExpireTime(int expireTime) {
        EXPIRE_TIME = expireTime;
    }

```

## @PropertySource
用来加载指定的配置文件

```java
/**
 * 想获取配置信息，必须是容器中的bean，其实和bean初始化逻辑一种，配置文件的信息其实就是属性填充
 * @ConfigurationProperties
 *    1. 告诉Spring对象的值从配置文件中获取，prefix前缀表示从配置文件中哪个配置项下面获取属性值;
 *    2. 用来读取全局配置文件(resources/application.yml,resource/application.properties),
 *       如果全局配置文件信息太多,就需要按场景拆分,为了能够读取到配置需要使用@PropertySource注解指定配置文件
 * @PropertySource 指定具体需要映射的配置文件,可以通过数组一次性指定多个。
 *
 */
@Component
@PropertySource(value = "classpath:person.properties")
@ConfigurationProperties(prefix = "person")
@Validated
public class Person {
    private String name;

    private int age;

    private String address;

    private List<Object> like;

    private Map<String, Object> maps;

    private Dog dog;
    
	...
}

```

## @ImportResource
用来导入Spring的配置文件，让配置文件里面的内容生效。
SpringBoot里面没有Spring的配置文件，我们自己编写的配置文件是不能识别的，
想让Spring的配置文件生效，加载进来，我们需要通过@ImportResource显示的标注在一个配置类上。
```java
@ImportResource(locations = {"classpath:bean.xml"})
@SpringBootApplication
public class PracticeApplication {
    public static void main(String[] args) {
        SpringApplication.run(PracticeApplication.class, args);
    }
}
// 测试类
@SpringBootTest
class PracticeApplicationTests {
    @Autowired
    private ApplicationContext applicationContext;

    @Test
    public void testBean() {
        System.out.println(applicationContext.containsBean("helloService"));
    }
}

```
## SpringBoot推荐使用全注解配置
SpringBoot推荐给容器添加组件的方式是使用全注解的方式，而不是上面的@ImportResource导入xml配置的方式。

使用@Configuration配置来代替Spring配置文件；
使用@Bean给容器中添加组件；
```java
/**
 * @Configuaration:类似Spring中的XML配置,可以让我们直接使用java编码的方式去配置Spring的组件。
 * @Bean:
 *   1. 类似于Spring中xml配置文件的bean标签；作用是把方法返回的对象添加到Spring容器中；
 *   2. 添加到容器中的对象的默认id值是方法名称。
 *
 */
@Configuration
public class MyAppConfig {
    @Bean
    public HelloService helloSer() {
        System.out.println("helloSer 对象创建");
        return new HelloService();
    }
}

// 测试类
@SpringBootTest
class PracticeApplicationTests {
    @Autowired
    private ApplicationContext applicationContext;
    @Test
    public void testBean() {
        System.out.println(applicationContext.containsBean("helloSer"));
    }
}

```

## 配置文件占位符
随机数占位符
可以在配置文件application.properties或者其他properties的配置文件中使用随机数占位符
```properties
person.name=zhangsan${random.uuid}
// 比如下面这些类型
${random.value}、${random.int}、${random.long}、${random.int(10)}、${random.int[1024,65536]}
```
占位符代表的值
如果占位符之前配置的值，就是代表配置的值，如果之前没有配置就是占位符key，还可以使用：指定默认值（在占位符没有配置值的情况下，取默认值）
```properties
person.name=zhangsan${random.uuid}
person.age=18
person.address=china_${person.name}
person.like=${person.hello:like}_movie
```

## Profile
Profile是Spring对不同环境提供不同配置功能的支持，可以通过激活、指定参数等方式快速切换环境。

### 多profile文件
我们在配置文件（默认配置文件：application.properties）编写的时候，文件名可以是：application-{profile}.properties/yml
例如：application-dev.properties、application-prod.properties

- 激活指定profile
配置文件方式激活
在默认配置文件中指定spring.profiles.active=xxx，就可以激活相应的profile配置文件。
比如：spring.profiles.active=prod
- 命令行激活（优先级最高）
启动时添加 --spring.profiles.active=xxx
命令行激活方式优先级高于配置文件/虚拟机方式，就是如果命令行、配置文件、虚拟机同时配置了，命令行的集合方式生效
java -jar practiceSpringboot.jar --spring.profiles.active=dev


## 配置文件加载位置
spring boot启动会扫描以下位置的application.properties或者application.yml文件作为Spring boot的默认配置文件。
1. file:./config/ 项目目录下的config文件夹下
   比如：与src同一个目录的config文件就是项目目录下的config文件夹。
2. file:./ 项目目录下
3. classpath:/config/ 类路径下的config文件夹下，比如src/main/resource/config/
4. classpath:/ 类路径下，比如src/main/resource/

上面的优先级从高到低排序（1的优先级大于2的优先级…），所有位置的文件都会被加载，高优先级配置内容会覆盖低优先级配置内容；比如classpath:/config/ 下面的配置会覆盖 classpath:/ 下面的相同配置。
我们也可以通过配置spring.config.location来改变默认配置。
springboot会从这四个位置全部加载主配置文件，互补配置（把4个地方的配置不同的key合在一起，相同的key高优先级覆盖低优先级）。

### 指定加载位置
我们还可以通过在命令行传入：spring.config.location来改变默认的配置文件位置，优先级最高，相同的key会被覆盖。
项目打包好以后，我们可以使用命令行参数的形式，启动项目的时候来指定配置文件的新位置；指定配置文件和默认加载的这些配置文件共同起作用形成互补配置。

## 外部配置加载顺序
SpringBoot也可以从以下位置加载配置；
顺序优先级由高到低，相同的key高优先级覆盖低优先级，所有配置形成互补配置。

1. 命令行参数。
2. 来自java:comp/env的JNDI属性。
3. java系统属性（System.getProperties()）。
4. 操作系统环境变量。
5. RandomValuePropertySource配置的random.*属性值。
6. jar包外部的application-{profile}.properties或application.yml{带spring.profile}配置文件。
7. jar包内部的application-{profile}.properties或application.yml{带spring.profile}配置文件。
8. jar包外部的application.properties或者application.yml（不带spring.profile）配置文件。
9. jar包内部的application.properties或者application.yml（不带spring.profile）配置文件。
10. @Configuration注解类上的 @PropertySource
11. 通过SpringApplication.setDefaultProperties指定的默认属性。






