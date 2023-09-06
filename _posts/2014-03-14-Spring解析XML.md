---
layout: post
categories: [XML]
description: none
keywords: XML
---
# Spring解析XML


## 
从以下这一小段代码说起：
```
new XmlBeanFactory(new ClassPathResource("springContext.xml"));
```
这小段看似简单仅仅实例化了两个对象，但是这只是表象。

## XmlBeanFactory的构造函数
查看代码发现XmlBeanFactory有两个构造函数：
```
private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);

public XmlBeanFactory(Resource resource) throws BeansException {
    this(resource, null);
}

public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
    super(parentBeanFactory);
    this.reader.loadBeanDefinitions(resource);
}
```
最终会执行的代码是：
```
//执行父类带参构造函数
super(parentBeanFactory);
//执行XmlBeanDefinitionReader的loadBeanDefinitions
this.reader.loadBeanDefinitions(resource);
```
可以看到this.reader.loadBeanDefinitions才是资源加载的真正实现。XmlBeanFactory相当于 DefaultListableBeanFactory + XmlBeanDefinitionReader，既可以对bean进行一些操作，又可以读取XML。

Resource用来封装配置文件。

## XmlBeanDefinitionReader读取XML
一路走下去，方法到了doLoadBeanDefinitions方法，开始解析XML文件的地方（做了些简化）：
```
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
       ...

        Document doc = doLoadDocument(inputSource, resource);
        return registerBeanDefinitions(doc, resource);
       ...
}
```
进入doLoadDocument(inputSource, resource)，实际上是使用配置的documentLoader来解析XML的：
```
protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
    return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
            getValidationModeForResource(resource), isNamespaceAware());
}
```
常见的XML解析方式有：DOM、SAX、dom4j等，那么spring中使用的是哪种方式呢？
spring中采用的是DOM方式，所要做的一切就是得到org.w3c.dom.Document。

loadDocment 传递的参数有：
- EntityResolver ： 指定XML验证方式，实现先从本地获取XSD或者DTD文件
- errorHandler ： 解析错误处理类
- ValidationMode ： 验证方式XSD、DTD
- isNamespaceAware ：是否支持命名空间

## 验证模式介绍
ValidationMode:比较常用的验证方式有两种：DTD和XSD

### DTD模式：
要使用DTD进行验证，需要在XML文件的头部声明，同时需要制定DTD文件的位置。
spring中使用DTD如下：
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC  "-//SPRING//DTD BEAN 2.0//EN"  "http://www.springframework.org/dtd/spring-beans-2.0.dtd">
```
XML的DOCTYPE有两个属性publicId和systemId，这两个东西都可以用来制定DTD文件的位置。所不同的是systemId指定的是具体的资源文件，它可以是相对路径也可以是绝对路径，而publicId则是使用定义的名称查找资源文件

publicId 是一个公共资源的知名标识，解析器可以根据这个名字得到一个资源（一般是DTD），如果根据这个名字没能找到一个资源，那么会根据systemId（一般是一个URL）来获取这个资源，如果还获取不到，那么会报错。也就是说，publicId和systemId是一个网络资源的标识，前者标识名字，后者标识URL，一般是成对出现。

解析器获取publicId标识的资源，一般是通过网络下载，这会导致解析速度变慢，或者解析失败。因此，可以在解析前编写验证文件，或者制定验证文件的地址。

### XSD模式：
使用XSD验证，除了声明命名空间外xmlns，还要通过schemaLocation来指定该命名空间验证文件的位置。
```
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">
</beans> -->
```

## spring中对xml的验证
```
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
            ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

        DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
        if (logger.isDebugEnabled()) {
            logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
        }
        DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
        return builder.parse(inputSource);
    }
```
和咱们想象的差不多，先创建DocumentBuilderFactory，然后DocumentBuilder，然后parse。
```
protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
            throws ParserConfigurationException {
        //创建DocumentBuilderFactory
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        //是否支持命名空间
        factory.setNamespaceAware(namespaceAware);

        //如果需要进行xml验证，而且是XSD验证，强制支持命名空间
        if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
            factory.setValidating(true);
            if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
                // Enforce namespace aware for XSD...
                factory.setNamespaceAware(true);
                try {
                    factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
                }
                catch (IllegalArgumentException ex) {
                    ParserConfigurationException pcex = new ParserConfigurationException(
                            "Unable to validate using XSD: Your JAXP provider [" + factory +
                            "] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
                            "Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
                    pcex.initCause(ex);
                    throw pcex;
                }
            }
        }

        return factory;
    }

//没啥需要解释的
protected DocumentBuilder createDocumentBuilder(
            DocumentBuilderFactory factory, EntityResolver entityResolver, ErrorHandler errorHandler)
            throws ParserConfigurationException {

        DocumentBuilder docBuilder = factory.newDocumentBuilder();
        if (entityResolver != null) {
            docBuilder.setEntityResolver(entityResolver);
        }
        if (errorHandler != null) {
            docBuilder.setErrorHandler(errorHandler);
        }
        return docBuilder;
    }
```

## 另外看下spring的entityResolver
如果没有特备指定entityResolver，则使用默认的entityResolver。
getResourceLoader是XmlBeanDefinitionReader父类AbstractBeanDefinitionReader中的方法，查看后发现resourceLoader == null。
所以真正使用的是DelegatingEntityResolver。
```
protected EntityResolver getEntityResolver() {
        if (this.entityResolver == null) {
            // Determine default EntityResolver to use.
            ResourceLoader resourceLoader = getResourceLoader();
            if (resourceLoader != null) {
                this.entityResolver = new ResourceEntityResolver(resourceLoader);
            }
            else {
                this.entityResolver = new DelegatingEntityResolver(getBeanClassLoader());
            }
        }
        return this.entityResolver;
    }

```
下面在DelegatingEntityResolver：
```
public DelegatingEntityResolver(ClassLoader classLoader) {
        //DTD验证EntityResolver
        this.dtdResolver = new BeansDtdResolver();
        //XSD验证EntityResolver
        this.schemaResolver = new PluggableSchemaResolver(classLoader);
    }
```
在EntityResolver指定本地验证文件的位置，如果找不到，再通过systemId的位置去找。













