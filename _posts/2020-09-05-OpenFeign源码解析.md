---
layout: post
categories: [OpenFeign]
description: none
keywords: OpenFeign
---
# OpenFeign源码解析

## 核心组件与概念
阅读OpenFeign源码时，可以沿着两条路线进行，一是FeignServiceClient这样的被@FeignClient注解修饰的接口类（后续简称为FeignClient接口类）如何创建，也就是其Bean实例是如何被创建的；二是调用FeignServiceClient对象的网络请求相关的函数时，OpenFeign是如何发送网络请求的。而OpenFeign相关的类也可以以此来进行分类，一部分是用来初始化相应的Bean实例的，一部分是用来在调用方法时发送网络请求。

OpenFeign关键类，其中比较重要的类为FeignClientFactoryBean、FeignContext和SynchronousMethodHandler。
- FeignClientFactoryBean是创建@FeignClient修饰的接口类Bean实例的工厂类；
- FeignContext是配置组件的上下文环境，保存着相关组件的不同实例，这些实例由不同的FeignConfiguration配置类构造出来；
- SynchronousMethodHandler是MethodHandler的子类，可以在FeignClient相应方法被调用时发送网络请求，然后再将请求响应转化为函数返回值进行输出。
OpenFeign会首先进行相关BeanDefinition的动态注册，然后当Spring容器注入相关实例时会进行实例的初始化，最后当FeignClient接口类实例的函数被调用时会发送网络请求。

## 动态注册BeanDefinition
OpenFeign可以通过多种方式进行自定义配置，配置的变化会导致接口类初始化时使用不同的Bean实例，从而控制OpenFeign的相关行为，比如说网络请求的编解码、压缩和日志处理。可以说，了解OpenFeign配置和实例初始化的流程与原理对于我们学习和使用OpenFeign有着至关重要的作用，而且Spring Cloud的所有项目的配置和实例初始化过程的原理基本相同，了解了OpenFeign的原理，就可以触类旁通，一通百通了。

### FeignClientsRegistrar
@EnableFeignClients的基本作用，它就像是OpenFeign的开关一样，一切OpenFeign的相关操作都是从它开始的。@EnableFeignClients有三个作用，一是引入FeignClientsRegistrar；二是指定扫描FeignClient的包信息，就是指定FeignClient接口类所在的包名；三是指定FeignClient接口类的自定义配置类。@EnableFeignClients注解的定义如下所示：
```java
//EnableFeignClients.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
//ImportBeanDefinitionRegistrar的子类,用于处理@FeignClient注解
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
    // 下面三个函数都是为了指定需要扫描的包
    String[] value() default {};
    String[] basePackages() default {};
    Class<?>[] basePackageClasses() default {};
    // 指定自定义feign client的自定义配置，可以配置Decoder、Encoder和Contract等组件,
        FeignClientsConfiguration是默认的配置类
    Class<?>[] defaultConfiguration() default {};
    // 指定被@FeignClient修饰的类，如果不为空，那么路径自动检测机制会被关闭
    Class<?>[] clients() default {};
}
```
上面的代码中，FeignClientsRegistrar是ImportBeanDefinitionRegistrar的子类，Spring用ImportBeanDefinitionRegistrar来动态注册BeanDefinition。OpenFeign通过FeignClientsRegistrar来处理@FeignClient修饰的FeignClient接口类，将这些接口类的BeanDefinition注册到Spring容器中，这样就可以使用@Autowired等方式来自动装载这些FeignClient接口类的Bean实例。FeignClientsRegistrar的部分代码如下所示：
```java
//FeignClientsRegistrar.java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar,
        ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
    ...
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
        //从EnableFeignClients的属性值来构建Feign的自定义Configuration进行注册
        registerDefaultConfiguration(metadata, registry);
        //扫描package，注册被@FeignClient修饰的接口类的Bean信息
        registerFeignClients(metadata, registry);
    }
    ...
}
```
如上述代码所示，FeignClientsRegistrar的registerBeanDefinitions方法主要做了两个事情，一是注册@EnableFeignClients提供的自定义配置类中的相关Bean实例，二是根据@EnableFeignClients提供的包信息扫描@FeignClient注解修饰的FeignCleint接口类，然后进行Bean实例注册。

@EnableFeignClients的自定义配置类是被@Configuration注解修饰的配置类，它会提供一系列组装FeignClient的各类组件实例。这些组件包括：Client、Targeter、Decoder、Encoder和Contract等。接下来看看registerDefaultConfiguration的代码实现，如下所示：
```java
//FeignClientsRegistrar.java
private void registerDefaultConfiguration(AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {
    //获取到metadata中关于EnableFeignClients的属性值键值对
    Map<String, Object> defaultAttrs = metadata
            .getAnnotationAttributes(EnableFeignClients.class.getName(), true);
    // 如果EnableFeignClients配置了defaultConfiguration类，那么才进行下一步操作，如果没有，
       会使用默认的FeignConfiguration
    if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
        String name;
        if (metadata.hasEnclosingClass()) {
            name = "default." + metadata.getEnclosingClassName();
        }
        else {
            name = "default." + metadata.getClassName();
        }
        registerClientConfiguration(registry, name,
                defaultAttrs.get("defaultConfiguration"));
    }
}
```
如上述代码所示，registerDefaultConfiguration方法会判断@EnableFeignClients注解是否设置了defaultConfiguration属性。如果有，则将调用registerClientConfiguration方法，进行BeanDefinitionRegistry的注册。registerClientConfiguration方法的代码如下所示。
```java
// FeignClientsRegistrar.java
private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
    Object configuration) {
    // 使用BeanDefinitionBuilder来生成BeanDefinition，并注册到registry上
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
        .genericBeanDefinition(FeignClientSpecification.class);
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(configuration);
    registry.registerBeanDefinition(
        name + "." + FeignClientSpecification.class.getSimpleName(),
        builder.getBeanDefinition());
}
```
BeanDefinitionRegistry是Spring框架中用于动态注册BeanDefinition信息的接口，调用其registerBeanDefinition方法可以将BeanDefinition注册到Spring容器中，其中name属性就是注册BeanDefinition的名称。

FeignClientSpecification类实现了NamedContextFactory.Specification接口，它是OpenFeign组件实例化的重要一环，它持有自定义配置类提供的组件实例，供OpenFeign使用。Spring Cloud框架使用NamedContextFactory创建一系列的运行上下文（ApplicationContext），来让对应的Specification在这些上下文中创建实例对象。这样使得各个子上下文中的实例对象相互独立，互不影响，可以方便地通过子上下文管理一系列不同的实例对象。NamedContextFactory有三个功能，一是创建AnnotationConfigApplicationContext子上下文；二是在子上下文中创建并获取Bean实例；三是当子上下文消亡时清除其中的Bean实例。在OpenFeign中，FeignContext继承了NamedContextFactory，用于存储各类OpenFeign的组件实例。

FeignAutoConfiguration是OpenFeign的自动配置类，它会提供FeignContext实例。并且将之前注册的FeignClientSpecification通过setConfigurations方法设置给FeignContext实例。这里处理了默认配置类FeignClientsConfiguration和自定义配置类的替换问题。如果FeignClientsRegistrar没有注册自定义配置类，那么configurations将不包含FeignClientSpecification对象，否则会在setConfigurations方法中进行默认配置类的替换。FeignAutoConfiguration的相关代码如下所示：
```java
//FeignAutoConfiguration.java
@Autowired(required = false)
private List<FeignClientSpecification> configurations = new ArrayList<>();
@Bean
public FeignContext feignContext() {
    FeignContext context = new FeignContext();
    context.setConfigurations(this.configurations);
    return context;
}
//FeignContext.java
public class FeignContext extends NamedContextFactory<FeignClientSpecification> {
    public FeignContext() {
        //将默认的FeignClientConfiguration作为参数传递给构造函数
        super(FeignClientsConfiguration.class, "feign", "feign.client.name");
    }
}
```
NamedContextFactory是FeignContext的父类，其createContext方法会创建具有名称的Spring的AnnotationConfigApplicationContext实例作为当前上下文的子上下文。这些AnnotationConfigApplicationContext实例可以管理OpenFeign组件的不同实例。NamedContextFactory的实现如下代码所示：
```java
//NamedContextFactory.java
protected AnnotationConfigApplicationContext createContext(String name) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    //获取该name所对应的configuration,如果有的话，就注册都子context中
    if (this.configurations.containsKey(name)) {
        for (Class<?> configuration : this.configurations.get(name)
                .getConfiguration()) {
            context.register(configuration);
        }
    }
    // 注册default的Configuration，也就是FeignClientsRegistrar类的registerDefaultConfiguration
       方法中注册的Configuration
    for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
        if (entry.getKey().startsWith("default.")) {
            for (Class<?> configuration : entry.getValue().getConfiguration()) {
                context.register(configuration);
            }
        }
    }
    // 注册PropertyPlaceholderAutoConfiguration和FeignClientsConfiguration配置类
    context.register(PropertyPlaceholderAutoConfiguration.class,
            this.defaultConfigType);
    // 设置子context的Environment的propertySource属性源
    // propertySourceName = feign; propertyName = feign.client.name
    context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
            this.propertySourceName,
            Collections.<String, Object> singletonMap(this.propertyName, name)));
    // 所有context的parent都相同，这样的话，一些相同的Bean可以通过parent context来获取
    if (this.parent != null) {
        context.setParent(this.parent);
    }
    context.setDisplayName(generateDisplayName(name));
    context.refresh();
    return context;
}
```
而由于NamedContextFactory实现了DisposableBean接口，当NamedContextFactory实例消亡时，Spring框架会调用其destroy方法，清除掉自己创建的所有子上下文和自身包含的所有组件实例。NamedContextFactory的destroy方法如下所示：
```java
//NamedContextFactory.java
@Override
public void destroy() {
    Collection<AnnotationConfigApplicationContext> values = this.contexts.values();
    for(AnnotationConfigApplicationContext context : values) {
        context.close();
    }
    this.contexts.clear();
}
```
NamedContextFactory会创建出AnnotationConfigApplicationContext实例，并以name作为唯一标识，然后每个AnnotationConfigApplicationContext实例都会注册部分配置类，从而可以给出一系列的基于配置类生成的组件实例，这样就可以基于name来管理一系列的组件实例，为不同的FeignClient准备不同配置组件实例，比如说Decoder、Encoder等。我们会在后续的讲解中详细介绍配置类Bean实例的获取。

### 扫描类信息
FeignClientsRegistrar做的第二件事情是扫描指定包下的类文件，注册@FeignClient注解修饰的接口类信息，如下所示：
```java
//FeignClientsRegistrar.java
public void registerFeignClients(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    //生成自定义的ClassPathScanningProvider
    ClassPathScanningCandidateComponentProvider scanner = getScanner();
    scanner.setResourceLoader(this.resourceLoader);
    Set<String> basePackages;
    //获取EnableFeignClients所有属性的键值对
    Map<String, Object> attrs = metadata
            .getAnnotationAttributes(EnableFeignClients.class.getName());
    //依照Annotation来进行TypeFilter，只会扫描出被FeignClient修饰的类
    AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
            FeignClient.class);
    final Class<?>[] clients = attrs == null ? null
            : (Class<?>[]) attrs.get("clients");
    //如果没有设置clients属性，那么需要扫描basePackage，所以设置了AnnotationTypeFilter,
           并且去获取basePackage
    if (clients == null || clients.length == 0) {
        scanner.addIncludeFilter(annotationTypeFilter);
        basePackages = getBasePackages(metadata);
    }
    //代码有删减，遍历上述过程中获取的basePackages列表
    for (String basePackage : basePackages) {
        //获取basepackage下的所有BeanDefinition
        Set<BeanDefinition> candidateComponents = scanner
                .findCandidateComponents(basePackage);
        for (BeanDefinition candidateComponent : candidateComponents) {
            if (candidateComponent instanceof AnnotatedBeanDefinition) {
                AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
                AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
                //从这些BeanDefinition中获取FeignClient的属性值
                Map<String, Object> attributes = annotationMetadata
                        .getAnnotationAttributes(
                                FeignClient.class.getCanonicalName());
                String name = getClientName(attributes);
                //对单独某个FeignClient的configuration进行配置
                registerClientConfiguration(registry, name,
                        attributes.get("configuration"));
                //注册FeignClient的BeanDefinition
                registerFeignClient(registry, annotationMetadata, attributes);
            }
        }
    }
}
```
如上述代码所示，FeignClientsRegistrar的registerFeignClients方法依据@EnableFeignClients的属性获取要扫描的包路径信息，然后获取这些包下所有被@FeignClient注解修饰的接口类的BeanDefinition，最后调用registerFeignClient动态注册BeanDefinition。

registerFeignClients方法中有一些细节值得认真学习，有利于加深了解Spring框架。首先是如何自定义Spring类扫描器，即如何使用ClassPathScanningCandidateComponentProvider和各类TypeFilter。

OpenFeign使用了AnnotationTypeFilter，来过滤出被@FeignClient修饰的类，getScanner方法的具体实现如下所示：
```java
//FeignClientsRegistrar.java
protected ClassPathScanningCandidateComponentProvider getScanner() {
    return new ClassPathScanningCandidateComponentProvider(false, this.environment) {
        @Override
        protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
            boolean isCandidate = false;
            //判断beanDefinition是否为内部类，否则直接返回false
            if (beanDefinition.getMetadata().isIndependent()) {
                //判断是否为接口类，所实现的接口只有一个，并且该接口是Annotation。否则直接
                       返回true
                if (!beanDefinition.getMetadata().isAnnotation()) {
                    isCandidate = true;
                }
            }
            return isCandidate;
        }
    };
}
```
ClassPathScanningCandidateComponentProvider的作用是遍历指定路径的包下的所有类。比如指定包路径为com/test/openfeign，它会找出com.test.openfeign包下所有的类，将所有的类封装成Resource接口集合。Resource接口是Spring对资源的封装，有FileSystemResource、ClassPathResource、UrlResource等多种实现。接着ClassPathScanningCandidateComponentProvider类会遍历Resource集合，通过includeFilters和excludeFilters两种过滤器进行过滤操作。includeFilters和excludeFilters是TypeFilter接口类型实例的集合，TypeFilter接口是一个用于判断类型是否满足要求的类型过滤器。excludeFilters中只要有一个TypeFilter满足条件，这个Resource就会被过滤掉；而includeFilters中只要有一个TypeFilter满足条件，这个Resource就不会被过滤。如果一个Resource没有被过滤，它会被转换成ScannedGenericBeanDefinition添加到BeanDefinition集合中。

## 实例初始化
FeignClientFactoryBean是工厂类，Spring容器通过调用它的getObject方法来获取对应的Bean实例。被@FeignClient修饰的接口类都是通过FeignClientFactoryBean的getObject方法来进行实例化的，具体实现如下代码所示：

```java
//FeignClientFactoryBean.java
public Object getObject() throws Exception {
    FeignContext context = applicationContext.getBean(FeignContext.class);
    Feign.Builder builder = feign(context);
    if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
        this.url = "http://" + this.url;
    }
    String url = this.url + cleanPath();
    //调用FeignContext的getInstance方法获取Client对象
    Client client = getOptional(context, Client.class);
    //因为有具体的Url,所以就不需要负载均衡，所以除去LoadBalancerFeignClient实例
    if (client != null) {
        if (client instanceof LoadBalancerFeignClient) {
            client = ((LoadBalancerFeignClient)client).getDelegate();
        }
        builder.client(client);
    }
    Targeter targeter = get(context, Targeter.class);
    return targeter.target(this, builder, context, new HardCodedTarget<>(
            this.type, this.name, url));
}
```
这里就用到了FeignContext的getInstance方法，我们在前边已经讲解了FeignContext的作用，getOptional方法调用了FeignContext的getInstance方法，从FeignContext的对应名称的子上下文中获取到Client类型的Bean实例，其具体实现如下所示：
```java
//NamedContextFactory.java
public <T> T getInstance(String name, Class<T> type) {
    AnnotationConfigApplicationContext context = getContext(name);
    if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
            type).length > 0) {
        //从对应的context中获取Bean实例,如果对应的子上下文没有则直接从父上下文中获取
        return context.getBean(type);
    }
    return null;
}
```
默认情况下，子上下文并没有这些类型的BeanDefinition，只能从父上下文中获取，而父上下文Client类型的BeanDefinition是在FeignAutoConfiguration中进行注册的。但是当子上下文注册的配置类提供了Client实例时，子上下文会直接将自己配置类的Client实例进行返回，否则都是由父上下文返回默认Client实例。Client在FeignAutoConfiguration中的配置如下所示。
```java
//FeignAutoConfiguration.java
@Bean
@ConditionalOnMissingBean(Client.class)
public Client feignClient(HttpClient httpClient) {
    return new ApacheHttpClient(httpClient);
}
```
Targeter是一个接口，它的target方法会生成对应的实例对象。它有两个实现类，分别为DefaultTargeter和HystrixTargeter。OpenFeign使用HystrixTargeter这一层抽象来封装关于Hystrix的实现。DefaultTargeter的实现如下所示，只是调用了Feign.Builder的target方法：
```java
//DefaultTargeter.java
class DefaultTargeter implements Targeter {
@Override
    public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
                        Target.HardCodedTarget<T> target) {
        return feign.target(target);
    }
}
```
而Feign.Builder是由FeignClientFactoryBean对象的feign方法创建的。Feign.Builder会设置FeignLoggerFactory、EncoderDecoder和Contract等组件，这些组件的Bean实例都是通过FeignContext获取的，也就是说这些实例都是可配置的，你可以通过OpenFeign的配置机制为不同的FeignClient配置不同的组件实例。feign方法的实现如下所示：
```java
//FeignClientFactoryBean.java
protected Feign.Builder feign(FeignContext context) {
    FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
    Logger logger = loggerFactory.create(this.type);
    Feign.Builder builder = get(context, Feign.Builder.class)
        .logger(logger)
        .encoder(get(context, Encoder.class))
        .decoder(get(context, Decoder.class))
        .contract(get(context, Contract.class));
    configureFeign(context, builder);
    return builder;
}
```
Feign.Builder负责生成被@FeignClient修饰的FeignClient接口类实例。它通过Java反射机制，构造InvocationHandler实例并将其注册到FeignClient上，当FeignClient的方法被调用时，InvocationHandler的回调函数会被调用，OpenFeign会在其回调函数中发送网络请求。build方法如下所示：
```java
//Feign.Builder
public Feign build() {
    SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
        new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, 
        logger, logLevel, decode404);
    ParseHandlersByName handlersByName = new ParseHandlersByName(contract, options, encoder, decoder, errorDecoder, synchronousMethodHandlerFactory);
    return new ReflectiveFeign(handlersByName, invocationHandlerFactory);
}
```
ReflectiveFeign的newInstance方法是生成FeignClient实例的关键实现。它主要做了两件事情，一是扫描FeignClient接口类的所有函数，生成对应的Handler；二是使用Proxy生成FeignClient的实例对象，代码如下所示：
```java
//ReflectiveFeign.java
public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
    for (Method method : target.type().getMethods()) {
        if (method.getDeclaringClass() == Object.class) {
            continue;
        } else if(Util.isDefault(method)) {
            //为每个默认方法生成一个DefaultMethodHandler
            defaultMethodHandler handler = new DefaultMethodHandler(method);
            defaultMethodHandlers.add(handler);
            methodToHandler.put(method, handler);
        } else {
            methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
        }
    }
    //生成java reflective的InvocationHandler
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[] {target.type()}, handler);
    //将defaultMethodHandler绑定到proxy中
    for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
        defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
}
```

### 扫描函数信息
在扫描FeignClient接口类所有函数生成对应Handler的过程中，OpenFeign会生成调用该函数时发送网络请求的模板，也就是RequestTemplate实例。RequestTemplate中包含了发送网络请求的URL和函数参数填充的信息。@RequestMapping、@PathVariable等注解信息也会包含到RequestTemplate中，用于函数参数的填充。ParseHandlersByName类的apply方法就是这一过程的具体实现。它首先会使用Contract来解析接口类中的函数信息，并检查函数的合法性，然后根据函数的不同类型来为每个函数生成一个BuildTemplateByResolvingArgs对象，最后使用SynchronousMethodHandler.Factory来创建MethodHandler实例。ParseHandlersByName的apply实现如下代码所示：

```java
//ParseHandlersByName.java
public Map<String, MethodHandler> apply(Target key) {
    // 获取type的所有方法的信息,会根据注解生成每个方法的RequestTemplate
    List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
    Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
    for (MethodMetadata md : metadata) {
    BuildTemplateByResolvingArgs buildTemplate;
    if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
        buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder);
    } else if (md.bodyIndex() != null) {
        buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder);
    } else {
        buildTemplate = new BuildTemplateByResolvingArgs(md);
    }
    result.put(md.configKey(),
        factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
    }
    return result;
}
```
OpenFeign默认的Contract实现是SpringMvcContract。SpringMvcContract的父类为BaseContract，而BaseContract是Contract众多子类中的一员，其他还有JAXRSContract和HystrixDelegatingContract等。Contract的parseAndValidateMetadata方法会解析与HTTP请求相关的所有函数的基本信息和注解信息，代码如下所示：
```java
//SpringMvcContract.java
@Override
public MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
    this.processedMethods.put(Feign.configKey(targetType, method), method);
    //调用父类BaseContract的函数
    MethodMetadata md = super.parseAndValidateMetadata(targetType, method);
    RequestMapping classAnnotation = findMergedAnnotation(targetType,
            RequestMapping.class);
    //处理RequestMapping注解
    if (classAnnotation != null) {
        if (!md.template().headers().containsKey(ACCEPT)) {
            parseProduces(md, method, classAnnotation);
        }
        if (!md.template().headers().containsKey(CONTENT_TYPE)) {
            parseConsumes(md, method, classAnnotation);
        }
        parseHeaders(md, method, classAnnotation);
    }
    return md;
}
```
BaseContract的parseAndValidateMetadata方法会依次解析接口类的注解，函数注解和函数的参数注解，将这些注解包含的信息封装到MethodMetadata对象中，然后返回，代码如下所示：
```java
//BaseContract.java
protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
    MethodMetadata data = new MethodMetadata();
    //函数的返回值
    data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
    //函数Feign相关的唯一配置键
    data.configKey(Feign.configKey(targetType, method));
    //获取并处理修饰class的注解信息
    if(targetType.getInterfaces().length == 1) {
        processAnnotationOnClass(data, targetType.getInterfaces()[0]);
    }
    //调用子类processAnnotationOnClass的实现
    processAnnotationOnClass(data, targetType);
    //处理修饰method的注解信息
    for (Annotation methodAnnotation : method.getAnnotations()) {
        processAnnotationOnMethod(data, methodAnnotation, method);
    }
    //函数参数类型
    Class<?>[] parameterTypes = method.getParameterTypes();
    Type[] genericParameterTypes = method.getGenericParameterTypes();
    //函数参数的注解类型
    Annotation[][] parameterAnnotations = method.getParameterAnnotations();
    int count = parameterAnnotations.length;
    //依次处理各个函数参数注解
    for (int i = 0; i < count; i++) {
        boolean isHttpAnnotation = false;
        if (parameterAnnotations[i] != null) {
            // 处理参数的注解，并且返回该参数来指明是否为将要发送请求的body。除了body之外，还
               可能是path，param等
            isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
        }
        if (parameterTypes[i] == URI.class) {
            data.urlIndex(i);
        } else if (!isHttpAnnotation) {
            //表明发送请求body的参数位置和参数类型
            data.bodyIndex(i);
            data.bodyType(Types.resolve(targetType, targetType, genericParameterTypes[i]));
        }
    }
    return data;
}
```
processAnnotationOnClass方法用于处理接口类注解。该函数在parseAndValidateMetadata方法中可能会被调用两次，如果targetType只继承或者实现一种接口时，先处理该接口的注解，再处理targetType的注解；否则只会处理targetType的注解。@RequestMapping在修饰FeignClient接口类时，其value所代表的值会被记录下来，它是该FeignClient下所有请求URL的前置路径，处理接口类注解的函数代码如下所示：
```java
//SpringMvcContract.java
protected void processAnnotationOnClass(MethodMetadata data, Class<?> clz) {
    if (clz.getInterfaces().length == 0) {
        //获取RequestMapping的注解信息，并设置MethodMetadata.template的数据
        RequestMapping classAnnotation = findMergedAnnotation(clz,
                RequestMapping.class);
        if (classAnnotation != null) {
            if (classAnnotation.value().length > 0) {
                String pathValue = emptyToNull(classAnnotation.value()[0]);
                pathValue = resolve(pathValue);
                if (!pathValue.startsWith("/")) {
                    pathValue = "/" + pathValue;
                }
                //处理@RequestMapping的value,一般都是发送请求的path
                data.template().insert(0, pathValue);
            }
        }
    }
}
```
processAnnotationOnMethod方法的主要作用是处理修饰函数的注解。它会首先校验该函数是否被@RequestMapping修饰，如果没有就会直接返回。然后获取该函数所对应的HTTP请求的方法，默认的方法是GET。接着会处理@RequestMapping中的value属性，解析value属性中的pathValue，比如说value属性值为/instance/{instanceId}，那么pathValue的值就是instanceId。最后处理消费（consumes）和生产（produces）相关的信息，记录媒体类型（media types），代码如下所示：

```java
//SpringMvcContract.java
protected void processAnnotationOnMethod(MethodMetadata data,
        Annotation methodAnnotation, Method method) {
    if (!RequestMapping.class.isInstance(methodAnnotation) && !methodAnnotation
            .annotationType().isAnnotationPresent(RequestMapping.class)) {
        return;
    }
    RequestMapping methodMapping = findMergedAnnotation(method, RequestMapping.class);
    // 处理HTTP Method
    RequestMethod[] methods = methodMapping.method();
    //默认的method是GET
    if (methods.length == 0) {
        methods = new RequestMethod[] { RequestMethod.GET };
    }
    data.template().method(methods[0].name());
    // 处理请求的路径
    checkAtMostOne(method, methodMapping.value(), "value");
    if (methodMapping.value().length > 0) {
        String pathValue = emptyToNull(methodMapping.value()[0]);
        if (pathValue != null) {
            pathValue = resolve(pathValue);
            // Append path from @RequestMapping if value is present on method
            if (!pathValue.startsWith("/")
                    && !data.template().toString().endsWith("/")) {
                pathValue = "/" + pathValue;
            }
            data.template().append(pathValue);
        }
    }
    // 处理生产
    parseProduces(data, method, methodMapping);
    // 处理消费
    parseConsumes(data, method, methodMapping);
    // 处理头部
    parseHeaders(data, method, methodMapping);
    data.indexToExpander(new LinkedHashMap<Integer, Param.Expander>());
}
```
而processAnnotationsOnParameter方法则主要处理修饰函数参数的注解。它会根据注解类型来调用不同的AnnotatedParameterProcessor的实现类，解析注解的属性信息。函数参数的注解类型包括@RequestParam、@RequestHeader和@PathVariable。processAnnotationsOnParameter方法的具体实现如下代码所示：
```java
//SpringMvcContract.java
protected boolean processAnnotationsOnParameter(MethodMetadata data,
        Annotation[] annotations, int paramIndex) {
    boolean isHttpAnnotation = false;
    AnnotatedParameterProcessor.AnnotatedParameterContext context = new SimpleAnnotatedParameterContext(
            data, paramIndex);
    Method method = this.processedMethods.get(data.configKey());
    //遍历所有的参数注解
    for (Annotation parameterAnnotation : annotations) {
        //不同的注解类型有不同的Processor
        AnnotatedParameterProcessor processor = this.annotatedArgumentProcessors
                .get(parameterAnnotation.annotationType());
        if (processor != null) {
            Annotation processParameterAnnotation;
            //如果没有缓存的Processor，则生成一个
            processParameterAnnotation = synthesizeWithMethodParameterNameAsFallbackValue(
                    parameterAnnotation, method, paramIndex);
            isHttpAnnotation |= processor.processArgument(context,
                    processParameterAnnotation, method);
        }
    }
    return isHttpAnnotation;
}
```
AnnotatedParameterProcessor是一个接口，有三个实现类：PathVariableParameterProcessor、RequestHeaderParameterProcessor和RequestParamParameterProcessor，三者分别用于处理@RequestParam、@RequestHeader和@PathVariable注解。
```java
//PathVariableParameterProcessor.java
public boolean processArgument(AnnotatedParameterContext context, Annotation annotation, Method method) {
    //ANNOTATION就是@PathVariable,所以就获取它的值，也就是@RequestMapping value中{}内的值
    String name = ANNOTATION.cast(annotation).value();
    //将name设置为ParameterName
    context.setParameterName(name);
    MethodMetadata data = context.getMethodMetadata();
    //当varName在url、queries、headers中不存在时，将name添加到formParams中。因为无法找到
      对应的值
    String varName = '{' + name + '}';
    if (!data.template().url().contains(varName)
            && !searchMapValues(data.template().queries(), varName)
            && !searchMapValues(data.template().headers(), varName)) {
        data.formParams().add(name);
    }
    return true;
}
```
如上述代码所示，PathVariableParameterProcessor的processArgument方法用于处理被@PathVariable注解修饰的参数。

ParseHandlersByName的apply方法通过Contract的parseAndValidatateMetadata方法获得了接口类中所有方法的元数据，这些信息中包含了每个方法所对应的网络请求信息。比如说请求的路径（path）、参数（params）、头部（headers）和body。接下来apply方法会为每个方法生成一个MethodHandler。SynchronousMethodHandler.Factory的create方法能直接创建SynchronousMethodHandler对象并返回，如下所示：
```java
//SynchronousMethodHandler.Factory
public MethodHandler create(Target<?> target, MethodMetadata md,
    RequestTemplate.Factory buildTemplateFromArgs,
    Options options, Decoder decoder, ErrorDecoder errorDecoder) {
    return new SynchronousMethodHandler(target, client, retryer, requestInterceptors, logger, logLevel, md, buildTemplateFromArgs,
                        options, decoder, errorDecoder, decode404);
}
```
ParseHandlersByName的apply方法作为ReflectiveFeign的newInstance方法的第一部分，其作用就是解析对应接口类的所有方法信息，并生成对应的MethodHandler。

### 生成Proxy接口类
ReflectiveFeign#newInstance方法的第二部分就是生成相应接口类的实例对象，并设置方法处理器，如下所示：

```java
//ReflectiveFeign.java
//生成Java反射的InvocationHandler
InvocationHandler handler = factory.create(target, methodToHandler);
T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[] {target.type()}, handler);
//将defaultMethodHandler绑定到proxy中。
for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
    defaultMethodHandler.bindTo(proxy);
}
return proxy;

```
OpenFeign使用Proxy的newProxyInstance方法来创建FeignClient接口类的实例，然后将InvocationHandler绑定到接口类实例上，用于处理接口类函数调用，如下所示：
```java
//Default.java
static final class Default implements InvocationHandlerFactory {
    @Override
    public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
        return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
    }
}
```
Default实现了InvocationHandlerFactory接口，其create方法返回ReflectiveFeign.FeignInvocationHandler实例。

ReflectiveFeign的内部类FeignInvocationHandler是InvocationHandler的实现类，其主要作用是将接口类相关函数的调用分配给对应的MethodToHandler实例，即SynchronousMethodHandler来处理。当调用接口类实例的函数时，会直接调用到FeignInvocationHandler的invoke方法。invoke方法会根据函数名称来调用不同的MethodHandler实例的invoke方法，如下所示：
```java
//FeignInvocationHandler.java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if ("equals".equals(method.getName())) {
        try {
            Object
                otherHandler =
                args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]): null;
            return equals(otherHandler);
        } catch (IllegalArgumentException e) {
            return false;
        }
    } else if ("hashCode".equals(method.getName())) {
        return hashCode();
    } else if ("toString".equals(method.getName())) {
        return toString();
    }
    //dispatch就是Map<Method, MethodHandler>，所以就是将某个函数的调用交给对应的MethodHandler
      来处理
    return dispatch.get(method).invoke(args);
}
```

## 函数调用和网络请求
在配置和实例生成结束之后，就可以直接使用FeignClient接口类的实例，调用它的函数来发送网络请求。在调用其函数的过程中，由于设置了MethodHandler，所以最终函数调用会执行SynchronousMethodHandler的invoke方法。在该方法中，OpenFeign会将函数的实际参数值与之前生成的RequestTemplate进行结合，然后发送网络请求。

OpenFeign发送网络请求时几个关键类的交互流程大概分为三个阶段：一是将函数实际参数值添加到RequestTemplate中；二是调用Target生成具体的Request对象；三是调用Client来发送网络请求，然后将Response转化为对象进行返回。

invoke方法的代码如下所示：
```java
//SynchronousMethodHandler.java
final class SynchronousMethodHandler implements MethodHandler {
    public Object invoke(Object[] argv) throws Throwable {
        //根据函数参数创建RequestTemplate实例，buildTemplateFromArgs是RequestTemplate.
          Factory接口的实例，在当前状况下是
        //BuildTemplateByResolvingArgs类的实例
        RequestTemplate template = buildTemplateFromArgs.create(argv);
        Retryer retryer = this.retryer.clone();
        while (true) {
            try {
                return executeAndDecode(template);
            } catch (RetryableException e) {
                retryer.continueOrPropagate(e);
                if (logLevel != Logger.Level.NONE) {
                    logger.logRetry(metadata.configKey(), logLevel);
                }
                continue;
            }
        }
    }
}
```
如上代码所示，SynchronousMethodHandler的invoke方法先创建了RequestTemplate对象。在该对象的创建过程中，使用到之前收集的函数信息MethodMetadata。遍历MethodMetadata中参数相关的indexToName，然后根据索引从invoke的参数数组中获得对应的值，将其填入对应的键值对中。然后依次处理查询和头部相关的参数值。invoke方法调用RequestTemplate.Factory的create方法创建RequestTemplate对象，代码如下所示：
```java
//RequestTemplate.Factory
public RequestTemplate create(Object[] argv) {
    RequestTemplate mutable = new RequestTemplate(metadata.template());
    //设置URL
        if (metadata.urlIndex() != null) {
        int urlIndex = metadata.urlIndex();
        checkArgument(argv[urlIndex] != null, "URI parameter %s was null", urlIndex);
        mutable.insert(0, String.valueOf(argv[urlIndex]));
    }
    Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
    //遍历MethodMetadata中所有关于参数的索引及其对应名称的配置信息
    for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
    int i = entry.getKey();
    //entry.getKey就是参数的索引
    Object value = argv[entry.getKey()];
    if (value != null) { // Null values are skipped.
        //indexToExpander保存着将各种类型参数的值转换为string类型的Expander转换器
        if (indexToExpander.containsKey(i)) {
        //将value值转换为string
        value = expandElements(indexToExpander.get(i), value);
        }
        for (String name : entry.getValue()) {
        varBuilder.put(name, value);
        }
    }
    }
    RequestTemplate template = resolve(argv, mutable, varBuilder);
    //设置queryMap参数
    if (metadata.queryMapIndex() != null) {
    template = addQueryMapQueryParameters((Map<String, Object>) argv[metadata.queryMapIndex()], template);
    }
    //设置headersMap参数
    if (metadata.headerMapIndex() != null) {
    template = addHeaderMapHeaders((Map<String, Object>) argv[metadata.headerMapIndex()], template);
    }
    return template;
}
```
resolve首先会替换URL中的pathValues，然后对URL进行编码，接着将所有头部信息进行转化，最后处理请求的Body数据，如下所示：
```java
//RequestTemplate.Factory
RequestTemplate resolve(Map<String, ?> unencoded, Map<String, Boolean> alreadyEncoded) {
    //替换query数值，将{queryVariable}替换成实际值
    replaceQueryValues(unencoded, alreadyEncoded);
    Map<String, String> encoded = new LinkedHashMap<String, String>();
    //把所有的参数都进行编码
    for (Entry<String, ?> entry : unencoded.entrySet()) {
        final String key = entry.getKey();
        final Object objectValue = entry.getValue();
        String encodedValue = encodeValueIfNotEncoded(key, objectValue, alreadyEncoded);
        encoded.put(key, encodedValue);
    }
    //编码url
    String resolvedUrl = expand(url.toString(), encoded).replace("+", "%20");
    if (decodeSlash) {
        resolvedUrl = resolvedUrl.replace("%2F", "/");
    }
    url = new StringBuilder(resolvedUrl);
    Map<String, Collection<String>> resolvedHeaders = new LinkedHashMap<String, Collection<String>>();
    //将头部都进行串行化
    for (String field : headers.keySet()) {
        Collection<String> resolvedValues = new ArrayList<String>();
        for (String value : valuesOrEmpty(headers, field)) {
            String resolved = expand(value, unencoded);
            resolvedValues.add(resolved);
        }
        resolvedHeaders.put(field, resolvedValues);
    }
    headers.clear();
    headers.putAll(resolvedHeaders);
    //处理body
    if (bodyTemplate != null) {
        body(urlDecode(expand(bodyTemplate, encoded)));
    }
    return this;
}
```
executeAndDecode方法会根据RequestTemplate生成Request对象，然后交给Client实例发送网络请求，最后返回对应的函数返回类型的实例。executeAndDecode方法的具体实现如下所示：
```java
//SynchronousMethodHandler.java
Object executeAndDecode(RequestTemplate template) throws Throwable {
    //根据RequestTemplate生成Request
    Request request = targetRequest(template);
    Response response;
    //client发送网络请求，client可能为okhttpclient和apacheClient
    try {
        response = client.execute(request, options);
        response.toBuilder().request(request).build();
    } catch (IOException e) {
        //...
    }
    try {
        //如果response的类型就是函数返回类型，那么可以直接返回
        if (Response.class == metadata.returnType()) {
            if (response.body() == null) {
                return response;
            }
            // 设置body
            byte[] bodyData = Util.toByteArray(response.body().asInputStream());
            return response.toBuilder().body(bodyData).build();
          }
        } catch (IOException e) {
            //...
    }
}
```

OpenFeign也提供了RequestInterceptor机制，在由RequestTemplate生成Request的过程中，会调用所有RequestInterceptor对RequestTemplate进行处理。而Target是生成JAXRS 2.0网络请求Request的接口类。RequestInterceptor处理的具体实现如下所示：
```java
//SynchronousMethodHandler.java
//按照RequestTemplate来创建Request
Request targetRequest(RequestTemplate template) {
    //使用请求拦截器为每个请求添加固定的header信息。例如BasicAuthRequestInterceptor，
    //它是添加Authorization header字段的
    for (RequestInterceptor interceptor : requestInterceptors) {
        interceptor.apply(template);
    }
    return target.apply(new RequestTemplate(template));
}
```

Client是用来发送网络请求的接口类，有OkHttpClient和RibbonClient两个子类。OkhttpClient调用OkHttp的相关组件进行网络请求的发送。OkHttpClient的具体实现如下所示：
```java
//OkHttpClient.java
public feign.Response execute(feign.Request input, feign.Request.Options options)
        throws IOException {
    //将feign.Request转换为Oktthp的Request对象
    Request request = toOkHttpRequest(input);
    //使用Okhttp的同步操作发送网络请求
    Response response = requestOkHttpClient.newCall(request).execute();
    //将Okhttp的Response转换为feign.Response
    return toFeignResponse(response).toBuilder().request(input).build();
}
```

