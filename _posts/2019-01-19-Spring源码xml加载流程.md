---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码xml加载流程
Spring配置文件加载流程图，来了解Spring配置文件加载流程，接着根据这个工作流程一步一步的阅读源码。主要加载xml配置文件的属性值到当前工厂中，最重要的就是BeanDefinition。

## xml加载流程
从容器启动开始，代码如下所示。实际上这个是父类 `AbstractApplicationContext` 中的方法。
```
public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            // Prepare this context for refreshing.
            // 准备刷新spring上下文，可以不看
            prepareRefresh();
 
            // Tell the subclass to refresh the internal bean factory.
            // 1：创建beanFactory工厂 2：解析xml文件，扫描注解 生成beanDefinition对象 3：注册到beanDefinitionRegistry缓存中
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
 
            // Prepare the bean factory for use in this context.
            // beanFactory 的一些准备工作，可以不看
            prepareBeanFactory(beanFactory);
 
            try {
                // Allows post-processing of the bean factory in context subclasses.
                // 钩子方法，由子类实现
                postProcessBeanFactory(beanFactory);
 
                // Invoke factory processors registered as beans in the context.
                // 注册beanFactoryPostProcessor对象
                invokeBeanFactoryPostProcessors(beanFactory);
 
                // Register bean processors that intercept bean creation.
                // 注册beanPostProcessor实例，在bean创建的时候实现拦截
                registerBeanPostProcessors(beanFactory);
 
                // Initialize message source for this context.
                initMessageSource();
 
                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();
 
                // Initialize other special beans in specific context subclasses.
                onRefresh();
 
                // Check for listener beans and register them.
                registerListeners();
 
                // Instantiate all remaining (non-lazy-init) singletons.
                /**
                 * 在之前已经实例化了BeanFactoryPostProcessor以及beanPostProcessor
                 * 下面开始实例化剩下的所有非懒加载的单例对象
                 */
                finishBeanFactoryInitialization(beanFactory);
 
                // Last step: publish corresponding event.
                finishRefresh();
            }
 
            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }
 
                // Destroy already created singletons to avoid dangling resources.
                destroyBeans();
 
                // Reset 'active' flag.
                cancelRefresh(ex);
 
                // Propagate exception to caller.
                throw ex;
            }
 
            finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
            }
        }
    }
```

## 配置文件加载入口 obtainFreshBeanFactory()
我们这里挑选重要的步骤进行分析，不重要的方法直接跳过了。主要看一下这个方法，从英文看，是让子类刷新内部的bean工厂。

首先Spring容器的启动我们进入的是Spring的容器刷新方法：`refresh()`，接着我们进入子方法 `obtainFreshBeanFactory()`。

该方法先创建容器对象：DefaultListableBeanFactory，然后加载xml配置文件的属性值到当前工厂中，最重要的就是BeanDefinition。
```
// Tell the subclass to refresh the internal bean factory.
// 1：创建beanFactory工厂 2：解析xml文件，扫描注解 生成beanDefinition对象 3：注册到beanDefinitionRegistry缓存中
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
// 创建BeanFactory：判断是否存在bean工厂，如果存在就进行销毁；再重新实例化一个bean工厂。
    refreshBeanFactory();
// 再调用AbstractRefreshableApplicationContext的getBeanFactory()方法获取在refresh()创建的bean工厂    
    return getBeanFactory();
}
```

我们看一下refreshBeanFactory，这个方法是钩子方法，在AbstractApplicationContext中并没有实现，而是交给子类实现：
```
	protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;

```

调到了AbstractRefreshableApplicationContext，这个判断如果beanFactory存在，就销毁实例，关闭工厂，然后重新创建一个beanFactory
```
protected final void refreshBeanFactory() throws BeansException {
    //如果beanFactory存在，就销毁实例，关闭工厂
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 创建一个beanFactory工厂容器。创建DefaultListableBeanFactory对象
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 为了序列化指定id,可以从id反序列化到beanFactory对象
        beanFactory.setSerializationId(getId());
        // 定制beanFactory,设置相关属性，包括是否允许覆盖同名称的不同定义的对象以及循环依赖
        customizeBeanFactory(beanFactory);
        // 加载xml配置文件 初始化documentReader,并进行XML文件读取及解析
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

## XML文件读取及解析 - loadBeanDefinitions()
加载xml配置，把beanFactory作为入参传入
```
// 加载xml配置文件
loadBeanDefinitions(beanFactory);
```

调用到AbstractXmlApplicationContext，这个类继承了AbstractRefreshableConfigApplicationContext类。

loadBeanDefinitions():方法就是初始化documentReader,并进行XML文件读取及解析，从这一步Spring开始它的配置文件加载流程。

具体代码如下：
```
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    // 创建一个xml的beanDefinitionReader,并通过回调设置到beanFactory中
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
 
    // Configure the bean definition reader with this context's
    // resource loading environment.
    // 给reader对象设置环境变量
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    // 设置资源加载器
    beanDefinitionReader.setResourceLoader(this);
    // 设置实体处理器
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
 
    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    // 初始化beanDefinitionReader对象，此处设置配置文件是否要进行验证[使用的适配器模式]
    initBeanDefinitionReader(beanDefinitionReader);
    // 开始完成beanDefinition的加载
    loadBeanDefinitions(beanDefinitionReader);
}
```
下面简单概括一下上面的初始化documentReader,并进行XML文件读取及解析的步骤

- XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory); 
创建一个xml的beanDefinitionReader,并通过回调设置到beanFactory中
- beanDefinitionReader.setEnvironment(this.getEnvironment());
给reader对象设置环境变量，便于配置文件里的占位符替换需要的值
- beanDefinitionReader.setResourceLoader(this);
设置资源加载器
- beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
设置实体处理器，当我们真正解析xml的时候才会使用它。
- initBeanDefinitionReader(beanDefinitionReader);
初始化beanDefinitionReader对象，此处设置配置文件是否要进行验证
- loadBeanDefinitions(beanDefinitionReader);
开始完成beanDefinition的加载。**重点**


## 开始完成beanDefinition的加载 - loadBeanDefinitions(beanDefinitionReader)
然后将xml解析的任务委托给了XmlBeanDefinitionReader对象来操作，调用loadBeanDefinitions

我们可以看出该方法主要是获取配置文件的位置，总共有两种方式获取配置文件的位置：
- 以Resource的方式获取配置文件的资源位置。
- 以String的形式获得配置文件的位置。

```
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    // 在classPathXmlApplicationContext构造器中，已经初始化了configLocation
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        // xml的阅读器加载xml配置文件
        reader.loadBeanDefinitions(configLocations);
    }
}
```

委托给reader来解析 reader.loadBeanDefinitions()。我们可以看到该方法循环String数组，开始处理每一个单一的location
```
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
    Assert.notNull(locations, "Location array must not be null");
    int count = 0;
    // 如果有多个xml文件，会循环解析，我们这里只有一个xml配置文件
    for (String location : locations) {
        count += loadBeanDefinitions(location);
    }
    return count;
}
```

Xml解析器将xml所在的路径location，封装成一个资源文件Resource，然后调用loadBeanDefinitions（resources）
```
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    ResourceLoader resourceLoader = getResourceLoader();
    if (resourceLoader == null) {
        throw new BeanDefinitionStoreException(
                "Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
    }
 
    if (resourceLoader instanceof ResourcePatternResolver) {
        // Resource pattern matching available.
        try {
            // 首先把location路径封装成一个资源文件
            Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
            int count = loadBeanDefinitions(resources);
            if (actualResources != null) {
                Collections.addAll(actualResources, resources);
            }
            if (logger.isTraceEnabled()) {
                logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
            }
            return count;
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "Could not resolve bean definition resource pattern [" + location + "]", ex);
        }
    }
    else {
        // Can only load single resources by absolute URL.
        // 这个Resource仅仅能够加载单个的绝对路径的xml配置文件
        Resource resource = resourceLoader.getResource(location);
        int count = loadBeanDefinitions(resource);
        if (actualResources != null) {
            actualResources.add(resource);
        }
        if (logger.isTraceEnabled()) {
            logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
        }
        return count;
    }
}
```
- ResourceLoader resourceLoader = getResourceLoader();：获取resourceLoader对象
- getResources(location)：调用DefaultResourceLoader的getResource完成具体的Resource定位
- loadBeanDefinitions(resources)：根据resource获取文件，并加载到beanDefinition中

接着向下看，上一步将resouce包装成可编码的EncodedResource对象，下面的方法，从资源Resource中拿到输入流InputStream，维护到InputSource中，然后调用doLoaderBeanDefinitions解析
```
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (logger.isTraceEnabled()) {
        logger.trace("Loading XML bean definitions from " + encodedResource);
    }
 
    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (currentResources == null) {
        currentResources = new HashSet<>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
                "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    try {
        // 从resource资源中获取输入流
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            // 加载beanDefinition
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            inputStream.close();
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(
                "IOException parsing XML document from " + encodedResource.getResource(), ex);
    }
    finally {
        currentResources.remove(encodedResource);
        if (currentResources.isEmpty()) {
            this.resourcesCurrentlyBeingLoaded.remove();
        }
    }
}
```

## 逻辑处理的核心步骤 - doLoadBeanDefinitions()
该方法是我们加载配置文件的核心，先获取XML文件的document对象，这个解析过程是由documentLoader完成的，最终开始将resource读取成一个document文档，根据文档的节点信息封装成一个个的BeanDefinition对象

```
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
        throws BeanDefinitionStoreException {
 
    try {
        // 将输入流解析为Document文件
        Document doc = doLoadDocument(inputSource, resource);
        int count = registerBeanDefinitions(doc, resource);
        if (logger.isDebugEnabled()) {
            logger.debug("Loaded " + count + " bean definitions from " + resource);
        }
        return count;
    }
}   
```
解析为document对象，之后就要注册beanDefinition了，在spring的加载过程中，BeanDefinition是一个重要的数据结构，它是在创建对象之前，对象数据的一种存在形式

docoment对象的解析过程委托给了BeanDefinitionDocumentReader对象来完成：
```
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    // 解析document文件，然后注册registerBeanDefinitions
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

```
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    // 注册beanDefinition，将document中root元素传入
    doRegisterBeanDefinitions(doc.getDocumentElement());
}
```

委托给document的解析器，入参为document的根元素，就是spring-context.xml的beans元素：
```
protected void doRegisterBeanDefinitions(Element root) {
    // Any nested <beans> elements will cause recursion in this method. In
    // order to propagate and preserve <beans> default-* attributes correctly,
    // keep track of the current (parent) delegate, which may be null. Create
    // the new (child) delegate with a reference to the parent for fallback purposes,
    // then ultimately reset this.delegate back to its original (parent) reference.
    // this behavior emulates a stack of delegates without actually necessitating one.
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);
 
    if (this.delegate.isDefaultNamespace(root)) {
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                    profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            // We cannot use Profiles.of(...) since profile expressions are not supported
            // in XML config. See SPR-12458 for details.
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                            "] not matching: " + getReaderContext().getResource());
                }
                return;
            }
        }
    }
 
    // 这里有两个钩子方法，典型的模板设计，由子类去实现
    preProcessXml(root);
    // 具体的解析document对象，注册beanDefinition的逻辑在这里实现
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);
 
    this.delegate = parent;
}
```

具体的解析工作在parseBeanDefinition中，在这里就是具体解析默认标签和自定义标签的流程。
```
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    //判断根元素的命名空间是否为空或者是 xmlns="http://www.springframework.org/schema/beans"
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                if (delegate.isDefaultNamespace(ele)) {
                    // 解析默认标签，例如：bean
                    parseDefaultElement(ele, delegate);
                }
                else {
                     // 解析自定义标签 例如：context
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```
启动spring流程的第一步，解析配置文件，当然我们这里是以xml配置的方式分析，也可能是注解配置的方法。

- 创建applicationContext对象，将xml文件的路径维护到AbstractRefreshableApplicationContext的属性上
- refresh启动spring流程，这里是spring启动的核心流程
- 第一步 obtainBeanFactory ，这这个方法里，会创建bean工厂，加载xml文件，委托给XmlBeanDefinitionReader解析
- XmlBeanDefinitionReader  将xml字符串路径封装为Resource对象，再转为InputStream流，最后把输入流生成Document对象，然后委托给BeanDefinitionDocumentReader解析。



































