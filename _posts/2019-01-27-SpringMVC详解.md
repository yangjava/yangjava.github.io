---
layout: post
categories: Spring
description: none
keywords: Spring
---
# SpringMVC详解
Spring的MVC是基于Servlet功能实现的，通过实现Servlet接口的DispatcherServlet来封装其核心功能实现，通过将请求分派给处理程序，同时带有可配置的处理程序映射、视图解析、本地语言、主题解析以及上载文件支持。

## SpringMVC流程
SpringMVC是一个表现层的框架，用于处理请求的映射。当一个请求到达SpringMVC时，首先是DispatchServlet进行处理，它通过HandlerMapping找到请求对应的Controller是谁，然后路由到该Controller执行业务逻辑，并得到一个ModelAndView。最后Spring会通过ViewResolver解析该ModelAndView，得到一个View返回给用户。

DispatchServlet是SpringMVC的前端控制器，所有的请求都会先到达这里。它本身是委派模式中的委派类，用于将处理请求的具体工作委派给对应组件。HandlerMapping是第一个接收到委派的组件。它自身维护了请求路径与Controller方法的关系，通过它能够找到request对应的Controller是谁。

接下来的工作到达了Controller。这是开发人员唯一需要关注的组件，只需要完成对应业务，然后将需要返回的数据封装成一个ModelAndView即可。ModelAndView底层是一个k-v结构的数据集合，通过LinkedHashMap实现。在得到ModelAndView后，DispatchServlet会委派ViewResolver执行视图解析，得到一个View。View最终将数据写入到Response完成请求。

DispatcherServlet本身就是个Servlet，实现了Servlet规范的同时，也实现了Spring的ApplicationAware接口，可以得到Spring上下文。在Spring容器初始化完毕后，会触发SpringMVC的九大组件的初始化，其方法定义在DispatchServlet\#initStrategies里，代码如下：
```
protected void initStrategies(ApplicationContext context) {  
    initMultipartResolver(context);  
    initLocaleResolver(context);  
    initThemeResolver(context);  
    initHandlerMappings(context);  
    initHandlerAdapters(context);  
    initHandlerExceptionResolvers(context);  
    initRequestToViewNameTranslator(context);  
    initViewResolvers(context);  
    initFlashMapManager(context);  
}  
```

接下来简单介绍下这九大组件：
- MultipartResolver。
这个组件的作用，是把普通的HttpServletRequest进行包装，使其具有文件上传的功能。它内部保存了文件名-文件对象的键值对集合，同时提供了一系列与文件处理相关的方法。

- LocalResolver。
看名字知道它应该和国际化有关。通过LocalResolver可以得到一个Local对象，它支持根据语言类型获取对应语言的信息。

- ThemeResolver。
Theme的意思是主题，Web应用的主体自然是页面样式了。这个Resolver的作用是根据配置得到对应的主题信息。主题内容（比如css，js等）放到properties文件里，每一个主题对应一个文件。

- HandlerMappings。
这是一个比较重要的组件，它维护了请求路径与对应Controller的关系，通过它可以根据请求路径得到对应的Handler。每个Handler对应一个Controller里负责处理请求的方法。

- HandlerAdapters。
这是一个适配器模式。由于Handler的格式各异，而交给DispatcherServlet进行后续处理时，需要统一的格式。因为DispatchServlet实现了Servlet规范，所以其doDispatch方法有固定的方法签名。此时适配器模式就登场了。

- ExceptionResolver。
看名字知道它是异常处理器。当发生异常时，通过它可以在程序发生异常时得到一个ModelAndView，进而呈现给用户。

- RequestToViewNameTranslator。
这个名字很明显，其作用是实现Request到ViewName的转换。有时候业务Handler并未返回视图信息，就需要通过这个组件从Request里获取视图名称，然后根据视图名称查找View进行后续渲染。

- ViewResolver。
其作用是根据ModelAndView和Local，Theme渲染视图，得到一个View。由于我们可能使用各种模板引擎，所以这里还会选择模板引擎进行对应处理。

- FlashMapManager。
看名字，它是一个FlashMap的管理器。而FlashMap的作用是进行重定向。要理解这个组件的功能，需要先了解下什么是FlashMap。

这里简单介绍下FlashMap。我们知道，转发是Request的行为，直接由后台将请求转给其它服务器；而重定向是response的行为，http规定，响应头收到“location”属性的url时，需要再次发起请求，重定向就是基于此机制实现的。FlashMap就是用于处理重定向过程中的问题的。

由于重定向是两个请求，无法在一个请求里实现参数共享。但重定向后需要带参数的场景很常见：我们新增一条数据后，重定向到一个查询页面，需要根据刚刚新增的数据ID查询对应数据。此时，如果不把参数拼接到URL里就不太好处理了，但我们并不推荐直接拼接参数到URL。那么我们可以在执行重定向之前，把参数写入到Request里的OUTPUT\_FLASH\_MAP\_ATTRIBUTE属性里。这样，重定向之后的上下文就可以获取重定向的参数信息了。FlashMap就对应上面的属性的参数。而FlashMapManager就是用来管理FlashMap的。

## SpringMVC初始化流程
接下来看看SpringMVC的工作流程。其工作流程可以分为初始化流程和请求处理流程。这里先看看SpringMVC是如何初始化的。首先来看入口。

### SpringMVC初始化入口
由于SpringMVC实现了Servlet规范，所以我们可以从Servlet规范入手分析其入口。依据Servlet规范，我们需要在WEB-INF里面的web.xml里配置Servlet处理的相关信息。在web.xml种，我们找到了DispatcherServlet。由于它遵守Servlet规范，所以Servlet容器在初始化该Servlet时，会调用它的init方法。

最终我们发现，DispatcherServlet\#init方法在其父类实现的，也就是HttpServletBean\#init。这个方法只干了两件事：加载配置，然后初始化Servlet。加载配置其实就是加载固定目录下的properties文件，没啥好看的。重点是init方法里的initServletBean，它会调用子类的方法完成Spring容器的初始化。

在这里，其子类方法是FrameworkServlet\#initWebApplicationContext此方法干了两件最重要的事情：执行容器初始化，对容器的refresh行为注册监听。精简后的代码如下：
```
protected WebApplicationContext initWebApplicationContext() {  
    WebApplicationContext wac = null;  
    //其它无关逻辑，比如设置辅助参数，缓存查找，降级方案等，代码略  
    ...  
    //启动容器  
    configureAndRefreshWebApplicationContext(cwac);  
    //注册监听  
    if(!this.refreshEventReceived) {  
        onRefresh(wac);  
    }  
    return wac;  
}  

```
上述代码中，启动容器的方法就是前面分析过的refresh方法，只不过在调用refresh之前设置了一些web相关的参数而已。之后注册了onRefresh监听，当容器初始化完毕后，会触发此监听，来到DispatcherServlet\#onRefresh方法里。在这里，我们发现了initStrategies方法，也就是初始化SpringMVC九大组件的委派方法。

在这个大委派方法中，我们重点关注HandlerMapping组件的初始化，它是SpringMVC能做映射的根本条件。由上文可知，它应该会维护一个URL与业务类的映射关系，接下来我们看看它是如何维护一个URL-Handler的映射的。

## HandlerMapping的映射关系
其实，DispatcherServlet#initHandlerMappings的作用是找到容器里所有的HandlerMapping，并非维护一个映射关系。由于九大组件的初始化是在容器启动后，那么真正维护映射关系的操作，应该是在容器启动时，或者实际使用时通过懒加载的方式初始化。这里还是先看看如何获取容器里所有的HandlerMapping的。上述方法的职责被委派给BeanFactoryUtils#beansOfTypeIncludingAncestors方法，这里直接看它即可，精简后的代码如下：
```
//注意参数中的type是HandlerMapping.class  
public static <T> Map<String, T> beansOfTypeIncludingAncestors(ListableBeanFactory lbf,  
                    Class<T> type, boolean includeNonSingletons, boolean allowEagerInit){  
    Map<String, T> result = new LinkedHashMap<String, T>(4);  
      
    //这个方法在从容器里获取HandlerMapping。注意这里是一个递归，每次都调用容器的父容器执行当前方法。  
    Map<String, T> parentResult = beansOfTypeIncludingAncestors(  
        (ListableBeanFactory) hbf.getParentBeanFactory(), type, includeNonSingletons  
        , allowEagerInit);  
    for (Map.Entry<String, T> entry : parentResult.entrySet()) {  
        String beanName = entry.getKey();  
        result.put(beanName, entry.getValue());  
    }  
    return result;  
}  
```
可见这里是通过type递归查找父类的ApplicationContext，从而获取了一个类型为HandlerMapping的Bean集合。在这个集合中，key是Bean的名称，Value是Bean的实例。这个方法里确实没有发现url-handler的映射关系，那么它是在哪里初始化的呢？

其实，这要从Spring的DI开始说起。Spring完成DI后，会调用前置和后置处理器。这些处理器会执行对应的Processor。其中有一个Processor里会调用Aware的set方法。关于Aware前文已经介绍过了。这里要关注的Aware是ApplicationContextAware。这个Aware有一个重要的子类，叫ApplicationObjectSupport。这个类的setApplicationContext方法就是初始化url-handler关系的入口了，精简后的代码如下：
```
public final void setApplicationContext(@Nullable ApplicationContext context){  
    this.applicationContext = context;  
    this.messageSourceAccessor = new MessageSourceAccessor(context);  
    initApplicationContext(context);  
}  
  
//钩子函数，不包含实现  
protected void initApplicationContext() throws BeansException {  
}  
```
我们知道钩子函数都是给子类实现的。ApplicationContextAware也有个重要子类，叫AbstractDetectingUrlHandlerMapping，它实现了initApplicationContext方法，代码如下：
```
public void initApplicationContext() throws ApplicationContextException {  
    super.initApplicationContext();  
    detectHandlers();  
}  

```
这里的AbstractDetectingUrlHandlerMapping\#detectHandlers就是初始化handlerMapping集合的方法了。它的实现逻辑是：遍历ApplicationContext中的所有Bean，然后找到Bean上的所有URL。按照SpringMVC的用法，我们会在Controller上通过注解或别的方式告知容器，该Controller的方法能处理哪些URL。得到URL后，SpringMVC会将其注册进一个Map集合里。这个集合中，Key就是URL，Value就是URL对应的Handler。精简后的代码如下：
```
protected void detectHandlers() throws BeansException {  
    //得到所有的Bean  
    String[] beanNames = BeanFactoryUtils  
            .beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class)    
    //遍历之  
    for (String beanName : beanNames) {  
        //解析Bean得到URL  
        String[] urls = determineUrlsForHandler(beanName);  
        //把Bean与Bean对应的URL关系保存起来  
        registerHandler(urls, beanName);  
    }  
}  
```
由于不同场景解析URL的方式不同，所以这里的determineUrlsForHandler是一个钩子函数，提供了基于配置和基于注解的解析方式，这是模板方法模式+策略模式的组合。由于实际使用中，我们更多的是用注解配置URL，所以这里重点看看注解版的实现，它对应的方法DefaultAnnotationHandlerMapping\#determineUrlsForHandler。这个方法会解析Bean上的URL。由于URL可以放在类上也可以放在方法上，所以会解析两种情况，精简后的代码如下：
```
protected String[] determineUrlsForHandler(String beanName) {  
    //得到Bean的Class  
    ApplicationContext context = getApplicationContext();  
    Class<?> handlerType = context.getType(beanName);  
    //获取Class中的类级别的@requestMapping注解  
    RequestMapping mapping = context.findAnnotationOnBean(beanName, RequestMapping.class);  
    //不为空就开始解析  
    if (mapping != null) {  
        //省略部分初始化逻辑  
        ...  
        //得到注解上的值  
        String[] typeLevelPatterns = mapping.value();  
        //解析过程很长，这里列出一部分片段如下：  
        //正则表达式  
        String[] methodLevelPatterns = determineUrlsForHandlerMethods(handlerType, true);  
        for (String typeLevelPattern : typeLevelPatterns) {  
            if (!typeLevelPattern.startsWith("/")) {  
                typeLevelPattern = "/" + typeLevelPattern;  
            }  
            //省略部分解析逻辑  
            ...  
        }  
        //最终返回解析到的URL数组  
        return StringUtils.toStringArray(urls);  
    }else {  
        //类级别的@RequestMapping为空，那可能定义到方法级别了  
        return determineUrlsForHandlerMethods(handlerType, false);  
    }      
}  

```
上述代码会解析出Bean对应的一组URL。接下来的逻辑，会根据BeanName获取Bean，同时将URL与对应Bean的关系保存起来。负责保存映射关系的代码是AbstractUrlHandlerMapping\#registerHandler\(\)，精简后逻辑如下：

```
//用于保存映射关系的成员变量  
private final Map<String, Object> handlerMap = new LinkedHashMap<String, Object>();  
  
//执行保存  
protected void registerHandler(String urlPath, Object handler){  
    if (urlPath.equals("/")) {  
        setRootHandler(resolvedHandler);  
    }else if (urlPath.equals("/*")) {  
        setDefaultHandler(resolvedHandler);  
    }else {  
        this.handlerMap.put(urlPath, resolvedHandler);  
    }  
}  
```
至此，我们已经得到了URL与对应Bean的映射关系。接下来当请求到达时，就可以根据url找到对应的Handler进行处理了。


## SpringMVC处理请求的调用链
我们仍然从Servlet规范入手。一个请求在Servlet里被处理，一定是调用Servlet的service方法。这里DispatcherServlet重写了父类的service方法，并调用了自身的doDispatch完成请求，所以直接从DispatcherServlet\#doDispatch看起即可。

调用链的委派方法doDispatch

DispatcherServlet\#doDispatch是一个模板方法模式和委派模式的组合，其职责是组织请求处理的整个链条。上文分析的整个请求处理流程都能在这段代码里找到对应代码：根据request拿到Handler，执行Handler得到ModelAndView，渲染ModelAndView得到View，执行view完成请求，释放资源。精简后的代码如下：
```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response){  
    try {  
        //转换文件请求  
        processedRequest = checkMultipart(request);  
        multipartRequestParsed = (processedRequest != request);  
        //根据请求得到handler，这里得到的是一个拦截器链。  
        HandlerExecutionChain mappedHandler= getHandler(processedRequest);  
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());  
        //执行拦截器链上的Interceptor  
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {  
            return;  
        }  
        //执行这个handler  
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());  
        //对结果视图的包装和额外处理  
        applyDefaultViewName(processedRequest, mv);  
        mappedHandler.applyPostHandle(processedRequest, response, mv);  
        //执行渲染  
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);  
    }catch (Exception ex) {  
        //异常处理  
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);  
    }finally {  
        //资源回收和后置处理逻辑  
        ...  
    }  
}  
```

请求调用链--初始化拦截器链

简单分析下上述代码。首先，getHandler方法得到的是一个拦截器链，这条责任链包含了业务Handler和程序里配置的拦截器。

以上述getHandler方法为入口查找构建责任链的地方，能找到AbstractHandlerMapping\#getHandler方法。AbstractHandlerMapping我们有印象，这里保存了URL与对应Controller的对应关系。这里看看它的getHandler方法，精简后的代码如下：
```
public final HandlerExecutionChain getHandler(HttpServletRequest request){  
    //根据request拿到Handler  
    Object handler = getHandlerInternal(request);  
    //降级和校验逻辑，代码略  
    ...  
    //构建责任链  
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);  
    //这里好像还分情况对责任链进行了包装，暂时看不懂  
    if (CorsUtils.isCorsRequest(request)) {  
      CorsConfiguration globalConfig =this.globalCorsConfigSource.getCorsConfiguration(request);  
      CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);  
      CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig)   
                                  : handlerConfig);  
      executionChain = getCorsHandlerExecutionChain(request, executionChain, config);  
   }  
   return executionChain;  
}  
```
上述代码中，有两个地方需要注意：一个是getHandlerInternal方法根据request获取Hander，另一个就是获取拦截器链的方法，也就是AbstractHandlerMapping\#getHandlerExecutionChain方法。这里先看看如何根据request获取Handler，然后再看看如何获取拦截器链的。

根据request获取Handler的方法就是上面方法中的getHandlerInternal方法。此方法只是根据请求拿到路径并对操作加锁而已，真正实现功能的是AbstractHandlerMethodMapping\#lookupHandlerMethod方法。它会根据之前保存的URL-Handler的映射关系，得到一系列的URL-Method的映射关系，并对其进行排序，之后选出一个最优的结果并返回。精简后的代码如下：
```
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request){  
    //保存待匹配的项  
    List<Match> matches = new ArrayList<>();  
    List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);  
    if (directPathMatches != null) {  
        addMatchingMappings(directPathMatches, matches, request);  
    }  
    if (matches.isEmpty()) {  
        addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);  
    }  
    if (!matches.isEmpty()) {  
        //构建比较器，并根据比较器进行排序  
        Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));  
        Collections.sort(matches, comparator);  
        //排在最前面的即为最优解  
        Match bestMatch = matches.get(0);  
        //排第二位的为第二优先解  
        Match secondBestMatch = matches.get(1);  
        //如果第一优先解和第二优先解比较的结果是一样的，说明项目里配置了重复的Mapping，此时会抛出异常  
        if (comparator.compare(bestMatch, secondBestMatch) == 0) {  
            throw new IllegalStateException...  
        }  
    else{  
        //未匹配到  
        return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);  
    }  
}  
```
上述代码解答了URL到Method的映射。但是到底是按何种优先级进行选择的呢？这就需要看Comparator的compare方法了。这里的比较器对应的方法是RequestMappingInfo\#compareTo，它定义了一系列规则，越靠前的规则权重越高。

接下来再看看拦截器链的初始化，这个逻辑相对简单，直接从容器里找匹配的Interceptor即可。精简后的代码如下：
```
protected HandlerExecutionChain getHandlerExecutionChain(Object handler,   
                                                HttpServletRequest request) {  
    //新建了一个责任链对象，它内部可以保存责任链上的节点，也就是拦截器。  
    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?  
        (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));  
    //根据request得到URL  
    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);  
    //然后遍历容器里的HanderInterceptor  
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {  
        //对于MappedInterceptor，需要看看URL是否匹配。匹配才加入拦截器链  
        if (interceptor instanceof MappedInterceptor) {  
            MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;  
            if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {  
                chain.addInterceptor(mappedInterceptor.getInterceptor());  
            }  
        }else {  
            //其它的拦截器直接加进来了  
            chain.addInterceptor(interceptor);  
        }  
    }  
    return chain;  
}  
```
拿到这条责任链后，会使用HandlerAdapter对其进行包装。HandlerAdapter是一个接口，它定义了两个比较重要的方法。一个可以判断当前HandlerAdapter是否匹配对应的Handler，另一个提供了handle方法用于调用用户业务代码。定义如下：

```
public interface HandlerAdapter {  
    //判断当前Adapter是否匹配Handler  
   boolean supports(Object handler);  
   //委派用户业务代码  
   ModelAndView handle(HttpServletRequest request,  
                       HttpServletResponse response, Object handler) throws Exception;  
   //获取上次更新时间  
   long getLastModified(HttpServletRequest request, Object handler);  
}  

```
这里对这条责任链（它是一个Handler）进行包装的逻辑，其实就是遍历容器里所有的HandlerAdapter，调用其supports方法，并返回第一个匹配上的HandlerAdapter。这个方法是DispatcherServlet\#getHandlerAdapter，精简的代码如下：
```
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {  
    for (HandlerAdapter ha : this.handlerAdapters) {  
        if (ha.supports(handler)) {  
            return ha;  
        }  
    }  
}  
```

### 请求调用链--执行前置Interceptor
上述代码拿到了一个HandlerAdapter，但接下来并不会直接调用其hanle方法,而是先调用了拦截器链的applyPrehandle方法。这个方法会执行拦截器方法，并返回一个布尔值：false则中断当前执行，直接return；true则继续往下走。这个方法中会循环所有的拦截器，并调用其preHandle方法，代码在HandlerExecutionChain\#applyPreHandle方法， 精简后如下：
```
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response){  
    //拿到所有拦截器  
    HandlerInterceptor[] interceptors = getInterceptors();  
    //遍历之  
    for (int i = 0; i < interceptors.length; i++) {  
        HandlerInterceptor interceptor = interceptors[i];  
        //调用其preHandle方法。返回false则进入此if判断。  
        if (!interceptor.preHandle(request, response, this.handler)) {  
            //调用拦截器的后置通知方法。  
            triggerAfterCompletion(request, response, null);  
            return false;  
        }  
        this.interceptorIndex = i;  
    }  
    return true;  
}  
```

为了方便理解上述代码，这里简单看下SpringMVC中拦截器的定义：
```
public interface HandlerInterceptor {  
    //前置处理，执行业务Controller前会先执行  
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response,  
                               Object handler)throws Exception {  
      return true;  
   }  
   //调用handler后，且页面渲染前调用  
   default void postHandle(HttpServletRequest request, HttpServletResponse response,  
                       Object handler,@Nullable ModelAndView modelAndView) throws Exception {}  
    //后置处理。调用业务Controller并完成渲染后执行  
   default void afterCompletion(HttpServletRequest request, HttpServletResponse response,  
                               Object handler,@Nullable Exception ex) throws Exception {  
   }  
}  

```
看到这些接口，发现就是我们定义一个拦截器时需要实现的方法。这里SpringMVC会在调用业务代码以前执行它们，如果前置处理器返回false，则会执行后置处理器，请求至此结束，实现了“拦截器返回false则中断请求”的语义。如果没返回false，则会接着执行业务代码对应的Handler。

### 请求调用链--执行Handler
执行完拦截器链的前置处理器，会紧接着执行“ha.handle”，也就是AbstractHandleMethodAdapter的handle方法。但这个方法透传了，真正实现其职责的是RequestMappingHandlerAdapter\#invokeHandlerMethod方法，它会对业务Handler进行包装，并调用包装以后的代码，之后构建一个ModelAndView并返回。这个方法在真正执行业务handler方法前，做了很多前置处理，主要是设置各种工具类，创建需要用到的工厂类（比如用于创建Model的ModelFactory等等），这里忽略这些逻辑对应的代码，直接看主干：

```
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,  
            HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {  
    //包装请求  
    ServletWebRequest webRequest = new ServletWebRequest(request, response);  
    //对传入的Handler进行包装，并为包装类设置一系列将要用到的工具  
    ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);  
    //比如方法参数的接收器等等，同时还构建了一系列工具，代码略  
    invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);  
    invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);  
    ...  
    //当然还会涉及到一些参数解析，视图初始化的工具：  
    ModelAndViewContainer mavContainer = new ModelAndViewContainer();  
    //比如之前说到的FlashMap参数就是这里设置的  
    mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));  
    ...  
    //最后执行业务代码。  
    invocableMethod.invokeAndHandle(webRequest, mavContainer);  
    //然后包装一个ModelAndView并返回  
    return getModelAndView(mavContainer, modelFactory, webRequest);  
}  
```
这里要关注的方法，主要是invokeAndHandle方法，也就是ServletInvocableHandlerMethod\#invokeAndHandle方法。它会执行对应的业务handler，并将得到的结果进行处理。这里需要对结果进行处理，是因为执行结果可能包含了各种参数，比如对应到http请求/响应头的，普通的执行结果，异步调用结果等。精简后的代码如下：

```
public void invokeAndHandle(ServletWebRequest webRequest,ModelAndViewContainer mavContainer,  
                            Object... providedArgs) throws Exception {  
    //执行业务代码  
    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);  
    //设置响应状态  
    setResponseStatus(webRequest);  
    //如果没有返回值就做一些结束标记的设置等，并直接返回。代码略  
    ...  
    //如果有结果，就执行结果的转换  
    this.returnValueHandlers.handleReturnValue(  
        returnValue, getReturnValueType(returnValue), mavContainer, webRequest);  
}  
```
接下来要看的自然是invokeForRequest方法了。这里为了防止迷路，先来回顾下之前得到的HandleMethod到哪里去了。上述方法中，invokeHandlerMethod里传入的还是根据URL匹配到的HandleMethod，到后来被包装成了ServletInvocableHandlerMethod。

接下来开始执行invokeForRequest方法，其实会调用其父类的方法：InvocableHandlerMethod\#invokeForRequest。这里自然能拿到之前传入的HandleMethod。这里的逻辑是：先根据request解析出参数列表，然后反射调用HandlerMethod。精简后的代码如下：
```
public Object invokeForRequest(NativeWebRequest request,ModelAndViewContainer mavContainer,  
                            Object... providedArgs) throws Exception {  
    //根据request获取参数列表  
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);  
    //反射调用  
    Object returnValue = doInvoke(args);  
    //返回调用结果  
    return returnValue;  
}  
```
先来看这个反射调用。它仅仅是通过JDK提供的反射入口调用HandleMethod而已，代码如下：
```
protected Object doInvoke(Object... args) throws Exception {  
    //改访问权限  
    ReflectionUtils.makeAccessible(getBridgedMethod());  
    //执行调用。这里的getBean就是获取当前HandleMethod的Object而已。  
    return getBridgedMethod().invoke(getBean(), args);  
}   
```
这里更加值得关注的是根据request获取参数列表的方法，也就是getMethodArgumentValues方法。它会完成Request里的参数和HandleMethod所需参数的匹配工作，并返回匹配好的参数列表。怎么知道URL的参数和Handler的参数是如何对应的呢？通常，我们可以使用@RequestParam注解告知Spring。但实际使用中，我们不写任何注解仍然能实现参数注入。

其实。这里的解析工作分为两种：

1，有注解就根据注解匹配参数并进行绑定。

2，没有注解，则使用参数名称进行绑定。由于Java的反射只能获取参数类型，名称都被变成了类似“arg0，arg1”这样的毫无意义的字符。此时如何根据request里的参数名称匹配到这种变形了的参数名称呢？SpringMVC 解决这个问题的方法是用ASM字节码框架读取字节码文件，并根据字节码文件获取方法的参数名称，从而完成匹配的。

通过上述逻辑可知：如果使用注解设置了参数，可以更快地执行方法，因为省去了字节码技术匹配参数的逻辑。

### 请求调用链--执行后置Interceptor
handler执行结束，且页面渲染前，会执行拦截器的后置处理方法。后置拦截器的处理就是doDispatch方法里的applyPostHandle。分析了前置处理器后，看它的代码就很容易了。HandlerExecutionChain\#applyPostHandle逻辑如下：
```
void applyPostHandle(HttpServletRequest request,HttpServletResponse response,ModelAndView mv){  
    //拿到拦截器  
    HandlerInterceptor[] interceptors = getInterceptors();  
    //循环执行  
    for (int i = interceptors.length - 1; i >= 0; i--) {  
        HandlerInterceptor interceptor = interceptors[i];  
        interceptor.postHandle(request, response, this.handler, mv);  
    }  
}  
```
可见就是循环执行拦截器的后置方法而已。

### 请求调用链--渲染结果集
渲染操作涉及到了很多的功能，比如国际化，模板引擎等等。但这里从doDispatch方法的入口看起还没那么复杂。这里执行渲染的方法是processDispatchResult，也就是DispatcherServlet\#processDispatchResult。它的逻辑是：转换和执行异常视图的逻辑，然后调用render方法执行渲染。精简后的逻辑如下：
```
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,  
                    HandlerExecutionChain mappedHandler,ModelAndView mv,Exception exception){  
    boolean errorView = false;  
    //发生了异常,则需要执行异常拦截器链并得到一个异常视图.  
    if (exception != null) {  
        if (exception instanceof ModelAndViewDefiningException) {  
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();  
        }else {  
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);  
            mv = processHandlerException(request, response, handler, exception);  
            errorView = (mv != null);  
    }  
    //渲染  
    if (mv != null && !mv.wasCleared()) {  
        render(mv, request, response);  
    }  
    //后置处理  
    if (mappedHandler != null) {  
        mappedHandler.triggerAfterCompletion(request, response, null);  
    }  
}  
```
这里再看看render方法，也就是DispatcherServlet\#render。它真正完成了渲染，但它本质上仍然是在委派其它类完成职责。精简后的代码如下：
```
protected void render(ModelAndView mv,HttpServletRequest request,HttpServletResponse response){  
    //国际化组件  
    Locale locale =this.localeResolver.resolveLocale(request)  
    response.setLocale(locale);  
    View view;  
    String viewName = mv.getViewName();  
    //根据视图名称匹配视图  
    if (viewName != null) {  
        view = resolveViewName(viewName, mv.getModelInternal(), locale, request);  
    }  
    //完成页面渲染  
    view.render(mv.getModelInternal(), request, response);  
}  

```
这里看两个方法：根据视图名称匹配视图，以及通过视图渲染页面。先看前者，也就是DispatcherServlet\#resolveViewName。其逻辑是遍历容器里的ViewResolver，找到一个合适的View。所以真正获取View的方法，其实是ViewResolver\#resolveViewName。

在SpringMVC中，定义了几个ViewResolver，各自有不同的职责，但最终都是用于得到一个View。这里View才是主角，基本上常见的媒体类型都有对应的View。但它们都有一些公共的逻辑，所以这里AbstractView\#render方法定义为一个模板方法，其逻辑是获取所有的返回值（包括Spring自动加上的和前文Handle方法返回的），然后将它们组织起来，一起执行子类的渲染逻辑。由于不同的子类处理的媒体类型不同，这个子类各异的操作当然是定义为钩子函数了。精简后的代码如下：
```
public void render(@Nullable Map<String, ?> model, HttpServletRequest request,  
                                    HttpServletResponse response) throws Exception {  
    //合并返回值  
    Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);  
    //设置Response的Header的一些公共属性  
    prepareResponse(request, response);  
    //钩子函数，子类各异的实现，这里根据不同的媒体类型完成了渲染  
    renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);  
}  
```
不同子类其重写的renderMergedOutputModel方法不同，但本质上都是做对应的返回值转换工作，然后通过response拿到一个输出流并把值写进去。

至此，MVC分析完毕。
















