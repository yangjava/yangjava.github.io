---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot配置加载原理
Springboot加载yml配置文件

## Springboot加载yml配置文件
Springboot 启动过程是一个复杂的流程，现在将yml加载拆分出来单独研究一下
ApplicationArguments 提供对用于运行的参数的访问
ConfigurableEnvironment 的propertySources 存放yml的配置参数
getSearchLocations 获取默认文件放置路径 classpath:/,classpath:/config/,file:./,file:./config/

```java
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			*********其它跟本业务无关的先注掉***********************
		}
		*********其它跟本业务无关的先注掉***********************
		return context;
	}

```
getOrCreateEnvironment 返回一个标准的servlet环境类
```java
private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		listeners.environmentPrepared(environment);
		bindToSpringApplication(environment);
		if (this.webApplicationType == WebApplicationType.NONE) {
			environment = new EnvironmentConverter(getClassLoader())
					.convertToStandardEnvironmentIfNecessary(environment);
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}

```
EventPublishingRunListener实现了SpringApplicationRunListener接口
environmentPrepared配置environment
```java
public void environmentPrepared(ConfigurableEnvironment environment) {
		this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(
				this.application, this.args, environment));
	}

```

```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
			//执行这一行
				invokeListener(listener, event);
			}
		}
	}
	************************一直往下跟

```

loadPostProcessors加载4个进程类
1、SystemEnvironmentPropertySourceEnvironmentPostProcessor
2、SpringApplicationJsonEnvironmentPostProcessor
3、CloudFoundryVcapEnvironmentPostProcessor
4、ConfigFileApplicationListener 加载配置文件

```java
private void onApplicationEnvironmentPreparedEvent(
			ApplicationEnvironmentPreparedEvent event) {
		List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
		postProcessors.add(this);
		AnnotationAwareOrderComparator.sort(postProcessors);
		for (EnvironmentPostProcessor postProcessor : postProcessors) {
			postProcessor.postProcessEnvironment(event.getEnvironment(),
					event.getSpringApplication());
		}
	}

```

进入ConfigFileApplicationListener 的addPropertySources 方法加载文件，实例化environment
```java
protected void addPropertySources(ConfigurableEnvironment environment,
			ResourceLoader resourceLoader) {
		RandomValuePropertySource.addToEnvironment(environment);
		new Loader(environment, resourceLoader).load();
	}

```

内部类 Loader 和load方法
```java
public void load() {
			this.profiles = Collections.asLifoQueue(new LinkedList<Profile>());
			this.processedProfiles = new LinkedList<>();
			this.activatedProfiles = false;
			this.loaded = new LinkedHashMap<>();
			initializeProfiles();
			while (!this.profiles.isEmpty()) {
				Profile profile = this.profiles.poll();
				//如果profiles文件不为空，那么加载文件
				load(profile, this::getPositiveProfileFilter,
						addToLoaded(MutablePropertySources::addLast, false));
				this.processedProfiles.add(profile);
			}
			load(null, this::getNegativeProfileFilter,
					addToLoaded(MutablePropertySources::addFirst, true));
			addLoadedPropertySources();
		}

```

getSearchLocations 获取默认文件放置路径 classpath:/,classpath:/config/,file:./,file:./config/
然后遍历加载路径下的文件名 application 后缀为properties，XML，yml，yaml
```java
private void load(Profile profile, DocumentFilterFactory filterFactory,
				DocumentConsumer consumer) {
			getSearchLocations().forEach((location) -> {
				boolean isFolder = location.endsWith("/");
				Set<String> names = (isFolder ? getSearchNames() : NO_SEARCH_NAMES);
				names.forEach(
						(name) -> load(location, name, profile, filterFactory, consumer));
			});
		}

```
最终遍历到你的yml存放的目录，然后执行加载
```java
private void load(PropertySourceLoader loader, String location, Profile profile,
				DocumentFilter filter, DocumentConsumer consumer) {
			try {
				Resource resource = this.resourceLoader.getResource(location);
				String description = getDescription(location, resource);
				if (profile != null) {
					description = description + " for profile " + profile;
				}
				if (resource == null || !resource.exists()) {
					this.logger.trace("Skipped missing config " + description);
					return;
				}
				if (!StringUtils.hasText(
						StringUtils.getFilenameExtension(resource.getFilename()))) {
					this.logger.trace("Skipped empty config extension " + description);
					return;
				}
				String name = "applicationConfig: [" + location + "]";
				List<Document> documents = loadDocuments(loader, name, resource);
				if (CollectionUtils.isEmpty(documents)) {
					this.logger.trace("Skipped unloaded config " + description);
					return;
				}
				List<Document> loaded = new ArrayList<>();
				for (Document document : documents) {
					if (filter.match(document)) {
						maybeActivateProfiles(document.getActiveProfiles());
						addProfiles(document.getIncludeProfiles());
						loaded.add(document);
					}
				}
				Collections.reverse(loaded);
				if (!loaded.isEmpty()) {
					loaded.forEach((document) -> consumer.accept(profile, document));
					this.logger.debug("Loaded config file " + description);
				}
			}
			catch (Exception ex) {
				throw new IllegalStateException("Failed to load property "
						+ "source from location '" + location + "'", ex);
			}
		}

```

```java
private List<Document> loadDocuments(PropertySourceLoader loader, String name,
				Resource resource) throws IOException {
				//获取resource 对应的缓存key
			DocumentsCacheKey cacheKey = new DocumentsCacheKey(loader, resource);
			List<Document> documents = this.loadDocumentsCache.get(cacheKey);
			if (documents == null) {
			//缓存中没有才执行load 方法加载
				List<PropertySource<?>> loaded = loader.load(name, resource);
				documents = asDocuments(loaded);
				this.loadDocumentsCache.put(cacheKey, documents);
			}
			return documents;
		}

```
OriginTrackedYamlLoader yaml文件加载器
```java
public List<Map<String, Object>> load() {
		final List<Map<String, Object>> result = new ArrayList<>();
		//回调函数 result.add(getFlattenedMap(map)
		process((properties, map) -> result.add(getFlattenedMap(map)));
		//返回最后加载的数据
		return result;
	}

protected void process(MatchCallback callback) {
		Yaml yaml = createYaml();
		for (Resource resource : this.resources) {
		//最后加载文件的方法 process(callback, yaml, resource)
			boolean found = process(callback, yaml, resource);
			if (this.resolutionMethod == ResolutionMethod.FIRST_FOUND && found) {
				return;
			}
		}
	}
```

这里是引用
```java
private boolean process(MatchCallback callback, Yaml yaml, Resource resource) {
		int count = 0;
		try {
			if (logger.isDebugEnabled()) {
				logger.debug("Loading from YAML: " + resource);
			}
			Reader reader = new UnicodeReader(resource.getInputStream());
			try {
				for (Object object : yaml.loadAll(reader)) {
				//将加载的yml文件内容转换成map，然后执行callback 方法将map里的内容放入result 中
					if (object != null && process(asMap(object), callback)) {
						count++;
						if (this.resolutionMethod == ResolutionMethod.FIRST_FOUND) {
							break;
						}
					}
				}
				//省略****
			}
			finally {
				reader.close();
			}
		}
		catch (IOException ex) {
			handleProcessError(resource, ex);
		}
		return (count > 0);
	}
```
URLClassLoader以流的形式加载文件返回 BufferedInputStream，然后包装成Reader
```
public InputStream getResourceAsStream(String name) {
        URL url = getResource(name);
        try {
            if (url == null) {
                return null;
            }
            URLConnection urlc = url.openConnection();
            InputStream is = urlc.getInputStream();
            if (urlc instanceof JarURLConnection) {
                JarURLConnection juc = (JarURLConnection)urlc;
                JarFile jar = juc.getJarFile();
                synchronized (closeables) {
                    if (!closeables.containsKey(jar)) {
                        closeables.put(jar, null);
                    }
                }
            } else if (urlc instanceof sun.net.www.protocol.file.FileURLConnection) {
                synchronized (closeables) {
                    closeables.put(is, null);
                }
            }
            return is;
        } catch (IOException e) {
            return null;
        }
    }
```

## 核心配置文件 bootstrap & application
Spring Boot 中有以下两种配置文件
- bootstrap (.yml 或者 .properties)
- application (.yml 或者 .properties)

SpringBoot bootstrap配置文件不生效问题
单独使用SpringBoot，发现其中的bootstrap.properties文件无法生效，改成yaml格式也无济于事。 最后调查发现原来是因为SpringBoot本身并不支持，需要和Spring Cloud 的组件结合——只有加上Spring Cloud Context依赖才能生效。

```xml
<!--需要引入该jar才能使bootstrap配置文件生效-->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-context</artifactId>
</dependency>
```

bootstrap/ application 的区别

在 Spring Boot 中有两种上下文，一种是 bootstrap, 另外一种是 application, bootstrap 是应用程序的父上下文，也就是说 bootstrap 加载优先于 applicaton。bootstrap 主要用于从额外的资源来加载配置信息，还可以在本地外部配置文件中解密属性。这两个上下文共用一个环境，它是任何Spring应用程序的外部属性的来源。bootstrap 里面的属性会优先加载，它们默认也不能被本地相同配置覆盖。
因此，对比 application 配置文件，bootstrap 配置文件具有以下几个特性。

boostrap 由父 ApplicationContext 加载，比 applicaton 优先加载
boostrap 里面的属性不能被覆盖

application 配置文件这个容易理解，主要用于 Spring Boot 项目的自动化配置。

bootstrap 配置文件有以下几个应用场景。

使用 Spring Cloud Config 配置中心时，这时需要在 bootstrap 配置文件中添加连接到配置中心的配置属性来加载外部配置中心的配置信息；
一些固定的不能被覆盖的属性
一些加密/解密的场景；
技术上，bootstrap.yml 是被一个父级的 Spring ApplicationContext 加载的。这个父级的 Spring ApplicationContext是先加载的，在加载application.yml 的 ApplicationContext之前。

### 启动上下文
Spring Cloud会创建一个Bootstrap Context，作为Spring应用的Application Context的父上下文。初始化的时候，Bootstrap Context负责从外部源加载配置属性并解析配置。这两个上下文共享一个从外部获取的Environment。Bootstrap属性有高优先级，默认情况下，它们不会被本地配置覆盖。 Bootstrap context和Application Context有着不同的约定，所以新增了一个bootstrap.yml文件，而不是使用application.yml (或者application.properties)。保证Bootstrap Context和Application Context配置的分离。

下面是一个例子： bootstrap.yml
```yml
spring:
  application:
    name: foo
  cloud:
    config:
      uri: ${SPRING_CONFIG_URI:http://localhost:8888}
```
推荐在bootstrap.yml or application.yml里面配置spring.application.name. 你可以通过设置spring.cloud.bootstrap.enabled=false来禁用bootstrap。






























