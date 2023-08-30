---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码Banner

## 源码Banner
springboot是通过SpringApplication类中的run()方法启动的，在该方法中调用printBanner()方法打印banner，如下所示
```
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		DefaultBootstrapContext bootstrapContext = createBootstrapContext();
		ConfigurableApplicationContext context = null;
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting(bootstrapContext, this.mainApplicationClass);
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
```
下面我们进入该方法
```
	private Banner printBanner(ConfigurableEnvironment environment) {
		if (this.bannerMode == Banner.Mode.OFF) {
			return null;
		}
		ResourceLoader resourceLoader = (this.resourceLoader != null) ? this.resourceLoader
				: new DefaultResourceLoader(null);
		SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter(resourceLoader, this.banner);
		if (this.bannerMode == Mode.LOG) {
			return bannerPrinter.print(environment, this.mainApplicationClass, logger);
		}
		return bannerPrinter.print(environment, this.mainApplicationClass, System.out);
	}
```

### 关闭banner
在printBanner()方法中我们首先看到对bannerMode的判断，如果是OFF，则直接返回null。
```
		if (this.bannerMode == Banner.Mode.OFF) {
			return null;
		}
```
而前面我们在演示的时候提到过，springboot提供了对应的方法。
```
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(Application.class);
        springApplication.setBannerMode(Banner.Mode.OFF);
    }
}

```

我们看一下Banner.Mode为何物？

## Banner接口
下面我们了解一下Banner接口。
```
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
Banner.Mode表示为Banner的模式，springboot提供了三种模式：OFF关闭、CONSOLE控制台、LOG日志文件。

在banner接口中提供了打印banner的方法和一个枚举类。枚举类有三个值：OFF、CONSOLE、LOG，用来控制banner的打印，分别对应：关闭打印、控制台打印和日志打印。    

banner接口的实现主要有ResourceBanner、ImageBanner、SpringBootBanner和其他内部类的实现。

## SpringApplicationBannerPrinter
SpringApplicationBannerPrinter中定义了默认位置、名称以及图片格式等。  

getBanner方法可以同时拿到文本和图片两种格式的banner，也就是说，当两种格式都配置的话，是可以同时打印出来的。

发现 Springboot 可以把图片转换成 ASCII 图案，那么它是怎么做的呢？以此为例，我们看下Spring 的Banner是如何生成的呢？

- 获取Banner
-    优先级是环境变量中的Image优先,格式在IMAGE_EXTENSION中
-    然后才是banner.txt
-    没有的话就用SpringBootBanner
- 如果是图片
-    获取图片Banner（属性配置等）
-    转换成ascii

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
如果都没有，则默认走SpringBootBanner

## 获取图片Banner
获取图片Banner
```
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
```
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

## SpringBootBanner默认Banner
在SpringBootBanner中定义了默认的banner图案以及版本号获取的方法。
```
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
如果是打印到控制台，`SpringBoot`使用的是`System.out`。









