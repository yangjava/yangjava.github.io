---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot自定义Banner
SpringBoot项目在启动的时候，我们可以看到控制台会打印一个Spring的图案，这个是SpringBoot默认的Banner图标。  

## SpringBoot默认Banner
我们在启动 Spring Boot 项目时会在控制台打印如下内容（logo 和版本信息）：
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.0.RELEASE)
```

## 如何更改Banner
那么如何自定义成自己项目的Banner呢？

### Banner配置
在 resources 资源文件夹下直接新建 banner.txt 文件，内容如下，运行项目可看到效果。
```
////////////////////////////////////////////////////////////////////
//                          _ooOoo_                               //
//                         o8888888o                              //
//                         88" . "88                              //
//                         (| ^_^ |)                              //
//                         O\  =  /O                              //
//                      ____/`---'\____                           //
//                    .'  \\|     |//  `.                         //
//                   /  \\|||  :  |||//  \                        //
//                  /  _||||| -:- |||||-  \                       //
//                  |   | \\\  -  /// |   |                       //
//                  | \_|  ''\---/''  |   |                       //
//                  \  .-\__  `-`  ___/-. /                       //
//                ___`. .'  /--.--\  `. . ___                     //
//              ."" '<  `.___\_<|>_/___.'  >'"".                  //
//            | | :  `- \`.;`\ _ /`;.`/ - ` : | |                 //
//            \  \ `-.   \_ __\ /__ _/   .-` /  /                 //
//      ========`-.____`-.___\_____/___.-`____.-'========         //
//                           `=---='                              //
//      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^        //
//            佛祖保佑       永不宕机     永无BUG                    //
////////////////////////////////////////////////////////////////////
```
总结：此方式简单直接，适合单一固定样式的 banner，需要频繁切换则不太适用。

### 多个Banner通过配置切换    
SpringBoot默认找寻的文件名是banner.txt，更换文件名会导致找不到banner图案，可以在配置文件中指定banner文件名。   
  
在application.yml中添加配置

```yaml
spring:
  banner:
    charset: UTF-8
    location: classpath:banner.txt
```
在 resources 资源文件夹可新建多个 banner 样式文件,可通过配置随意切换 banner 样式。

## SpringApplication启动时设置参数

```java
import org.springframework.boot.Banner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(Application.class);
        /**
         * Banner.Mode.OFF:关闭;
         * Banner.Mode.CONSOLE:控制台输出，默认方式;
         * Banner.Mode.LOG:日志输出方式;
         */
        application.setBannerMode(Banner.Mode.OFF); // here
        application.run(args);
    }

}
```
SpringApplication还可以设置自定义的Banner的接口类

## Banner的设计

通过输入文字来自定义：http://patorjk.com/software/taag/#p=display&f=Graffiti&t=Type%20Something%20
现有的样式，搜索获取 http://www.bootschool.net/ascii
http://www.network-science.de/ascii/ 
http://www.degraeve.com/img2txt.php

## Banner中其它配置信息

除了文件信息，还有哪些信息可以配置呢？比如Spring默认还带有SpringBoot当前的版本号信息。
在banner.txt中，还可以进行一些设置：
```
# springboot的版本号 
${spring-boot.version}             
 
# springboot的版本号前面加v后上括号 
${spring-boot.formatted-version}

# MANIFEST.MF文件中的版本号 
${application.version}              
 
# MANIFEST.MF文件的版本号前面加v后上括号 
${application.formatted-version}

# MANIFEST.MF文件中的程序名称
${application.title}

# ANSI样色/样式等
${Ansi.NAME} (or ${AnsiColor.NAME}, ${AnsiBackground.NAME}, ${AnsiStyle.NAME})
```

简单的测试如下（注意必须打包出Jar, 才会生成resources/META-INF/MANIFEST.MF）：
```
# springboot的版本号
2.3.0.RELEASE

# springboot的版本号前面加v后上括号
 (v2.3.0.RELEASE)

```

## 动画Banner的设计
SpringBoot2是支持图片形式的Banner
```yaml
spring:
  main:
    banner-mode: console
    show-banner: true
  banner:
    charset: UTF-8
    image:
      margin: 0
      height: 10
      invert: false
      location: classpath:xxx.png
```
需要选择合适的照片，不然效果不好, 所以这种方式很少使用

## Banner的参数设置

banner的参数设定可以通过两种形式，一种是代码的形式，一种是配置文件的形式。

使用代码的形式首先要将默认的main方法进行改造，手动创建SpringApplication对象，然后设置相应的参数。示例代码：

```
public static void main(String[] args) {

        SpringApplication app = new SpringApplication(SpringbootBannerApplication.class);
        app.setBannerMode(Banner.Mode.CONSOLE);

        Banner banner = new ImageBanner(new ClassPathResource("banner1.png"));
        app.setBanner(banner);
        app.run(args);
        }
```
通过配置文件设置就比较简单，直接在application.properties中进行配置，springboot已经帮我们预制好了相应的参数。
```properties
spring.banner.location=classpath:banner1.png
spring.banner.image.margin=2
spring.banner.image.height=76
spring.banner.charset=UTF-8
spring.banner.image.invert=false
spring.banner.image.location=banner1.png
spring.main.banner-mode=console
spring.main.show-banner=true
```
其中spring.main.show-banner来控制是否打印banner，在新版本中不建议使用，可以使用spring.main.banner-mode代替，将其值设置为OFF即可关闭banner的打印。  

引入文本banner通过spring.banner.location来指定，引入图片相关的banner需要通过spring.banner.image.location来指定路径，否则会出现乱码情况。  

如果不想显示banner，可以在代码中通过setBannerMode(Banner.Mode.OFF)方法或通过参数配置spring.main.banner-mode=off来关闭banner的打印。