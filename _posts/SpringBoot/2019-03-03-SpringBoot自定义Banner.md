---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot自定义Banner
SpringBoot项目在启动的时候，我们可以看到控制台会打印一个Spring的图案，这个是SpringBoot默认的Banner图标。  

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.0.RELEASE)
```

那么如何自定义成自己项目的Banner呢？

## 如何更改Banner

- Banner配置
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

- 多个Banner通过配置切换    

SpringBoot默认找寻的文件名是banner.txt，更换文件名会导致找不到banner图案，可以在配置文件中指定banner文件名。   
  
在application.yml中添加配置

```yaml
spring:
  banner:
    charset: UTF-8
    location: classpath:banner.txt
```
在 resources 资源文件夹可新建多个 banner 样式文件,可通过配置随意切换 banner 样式。

- SpringApplication启动时设置参数
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

### 图片Banner是如何起作用的？

发现 Springboot 可以把图片转换成 ASCII 图案，那么它是怎么做的呢？以此为例，我们看下Spring 的Banner是如何生成的呢？

- 获取Banner
-    优先级是环境变量中的Image优先,格式在IMAGE_EXTENSION中
-    然后才是banner.txt 
-    没有的话就用SpringBootBanner 
- 如果是图片
-    获取图片Banner（属性配置等） 
-    转换成ascii

- 获取banner

SpringApplicationBannerPrinter中定义了默认位置、名称以及图片格式等。    

getBanner方法可以同时拿到文本和图片两种格式的banner，也就是说，当两种格式都配置的话，是可以同时打印出来的。

```java
class SpringApplicationBannerPrinter {
    static final String BANNER_LOCATION_PROPERTY = "spring.banner.location";
    static final String BANNER_IMAGE_LOCATION_PROPERTY = "spring.banner.image.location";
    static final String DEFAULT_BANNER_LOCATION = "banner.txt";
    static final String[] IMAGE_EXTENSION = new String[]{"gif", "jpg", "png"};
    private static final Banner DEFAULT_BANNER = new SpringBootBanner(); // 默认的Spring Banner
    private final ResourceLoader resourceLoader;
    private final Banner fallbackBanner;

    // 获取Banner，优先级是环境变量中的Image优先,格式在IMAGE_EXTENSION中，然后才是banner.txt
    private Banner getBanner(Environment environment) {
        SpringApplicationBannerPrinter.Banners banners = new SpringApplicationBannerPrinter.Banners();
        banners.addIfNotNull(this.getImageBanner(environment));
        banners.addIfNotNull(this.getTextBanner(environment));
        if (banners.hasAtLeastOneBanner()) {
            return banners;
        } else {
            return this.fallbackBanner != null ? this.fallbackBanner : DEFAULT_BANNER;
        }
    }
}
```
获取图片Banner
```java
private Banner getImageBanner(Environment environment) {
        String location = environment.getProperty("spring.banner.image.location");
        if (StringUtils.hasLength(location)) {
        Resource resource = this.resourceLoader.getResource(location);
        return resource.exists() ? new ImageBanner(resource) : null;
        } else {
        String[] var3 = IMAGE_EXTENSION;
        int var4 = var3.length;

        for(int var5 = 0; var5 < var4; ++var5) {
        String ext = var3[var5];
        Resource resource = this.resourceLoader.getResource("banner." + ext);
        if (resource.exists()) {
        return new ImageBanner(resource);
        }
        }

        return null;
        }
        }
```
获取图片的配置等
```java
private void printBanner(Environment environment, PrintStream out) throws IOException {
        int width = (Integer)this.getProperty(environment, "width", Integer.class, 76);
        int height = (Integer)this.getProperty(environment, "height", Integer.class, 0);
        int margin = (Integer)this.getProperty(environment, "margin", Integer.class, 2);
        boolean invert = (Boolean)this.getProperty(environment, "invert", Boolean.class, false); // 图片的属性
        BitDepth bitDepth = this.getBitDepthProperty(environment);
        ImageBanner.PixelMode pixelMode = this.getPixelModeProperty(environment);
        ImageBanner.Frame[] frames = this.readFrames(width, height); // 读取图片的帧

        for(int i = 0; i < frames.length; ++i) {
        if (i > 0) {
        this.resetCursor(frames[i - 1].getImage(), out);
        }

        this.printBanner(frames[i].getImage(), margin, invert, bitDepth, pixelMode, out);
        this.sleep(frames[i].getDelayTime());
        }

        }
```
转换成ascii

```java
private void printBanner(BufferedImage image, int margin, boolean invert, BitDepth bitDepth, ImageBanner.PixelMode pixelMode, PrintStream out) {
        AnsiElement background = invert ? AnsiBackground.BLACK : AnsiBackground.DEFAULT;
        out.print(AnsiOutput.encode(AnsiColor.DEFAULT));
        out.print(AnsiOutput.encode(background));
        out.println();
        out.println();
        AnsiElement lastColor = AnsiColor.DEFAULT;
        AnsiColors colors = new AnsiColors(bitDepth);

        for(int y = 0; y < image.getHeight(); ++y) {
        int x;
        for(x = 0; x < margin; ++x) {
        out.print(" ");
        }

        for(x = 0; x < image.getWidth(); ++x) {
        Color color = new Color(image.getRGB(x, y), false);
        AnsiElement ansiColor = colors.findClosest(color);
        if (ansiColor != lastColor) {
        out.print(AnsiOutput.encode(ansiColor));
        lastColor = ansiColor;
        }

        out.print(this.getAsciiPixel(color, invert, pixelMode)); // // 像素点转换成字符输出
        }

        out.println();
        }

        out.print(AnsiOutput.encode(AnsiColor.DEFAULT));
        out.print(AnsiOutput.encode(AnsiBackground.DEFAULT));
        out.println();
        }
```

在SpringBootBanner中定义了默认的banner图案以及版本号获取的方法。
```java
class SpringBootBanner implements Banner {
    private static final String[] BANNER = new String[]{"", "  .   ____          _            __ _ _", " /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\", "( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\", " \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )", "  '  |____| .__|_| |_|_| |_\\__, | / / / /", " =========|_|==============|___/=/_/_/_/"};
    private static final String SPRING_BOOT = " :: Spring Boot :: ";
    private static final int STRAP_LINE_SIZE = 42;

    SpringBootBanner() {
    }

    public void printBanner(Environment environment, Class<?> sourceClass, PrintStream printStream) {
        String[] var4 = BANNER;
        int var5 = var4.length;

        for(int var6 = 0; var6 < var5; ++var6) {
            String line = var4[var6];
            printStream.println(line);
        }

        String version = SpringBootVersion.getVersion();
        version = version != null ? " (v" + version + ")" : "";
        StringBuilder padding = new StringBuilder();

        while(padding.length() < 42 - (version.length() + " :: Spring Boot :: ".length())) {
            padding.append(" ");
        }

        printStream.println(AnsiOutput.toString(new Object[]{AnsiColor.GREEN, " :: Spring Boot :: ", AnsiColor.DEFAULT, padding.toString(), AnsiStyle.FAINT, version}));
        printStream.println();
    }
}
```

- Banner接口
  在未指定banner.txt或banner图片时，默认图形存储在哪里了呢？下面我们了解一下Banner接口。
  
```java
@FunctionalInterface
public interface Banner {
    void printBanner(Environment environment, Class<?> sourceClass, PrintStream out);

    public static enum Mode {
        OFF,
        CONSOLE,
        LOG;

        private Mode() {
        }
    }
}
```
在banner接口中提供了打印banner的方法和一个枚举类。枚举类有三个值：OFF、CONSOLE、LOG，用来控制banner的打印，分别对应：关闭打印、控制台打印和日志打印。    
banner接口的实现主要有ResourceBanner、ImageBanner、SpringBootBanner和其他内部类的实现。其中上面看到的图形的打印就来自于SpringBootBanner。

### Banner的参数设置

banner的参数设定可以通过两种形式，一种是代码的形式，一种是配置文件的形式。

使用代码的形式首先要将默认的main方法进行改造，手动创建SpringApplication对象，然后设置相应的参数。示例代码：

```java
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