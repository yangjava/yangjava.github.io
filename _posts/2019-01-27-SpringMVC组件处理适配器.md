---
layout: post
categories: [SpringMV]
description: none
keywords: Spring
---
# SpringMVC组件处理适配器
SpringMVC 中通过 HandlerAdapter 来让 Handler 得到执行，为什么拿到 Handler 之后不直接执行呢？那是因为 SpringMVC 中我们定义 Handler 的方式多种多样（虽然日常开发中我们都是使用注解来定义，但是实际上还有其他方式），不同的 Handler 当然对应不同的执行方式，所以这中间就需要一个适配器 HandlerAdapter。

## HandlerAdapter 体系
我们先来看看 HandlerAdapter 大致的继承关系：

这里除了 RequestMappingHandlerAdapter 比较复杂之外，其他几个都是比较容易的，容易是因为这几个调用的方法都是固定的，复杂则是因为调用的方法不固定。我们先来看看这几个容易的。

## HandlerAdapter
HttpRequestHandlerAdapter、SimpleServletHandlerAdapter、HandlerFunctionAdapter、SimpleControllerHandlerAdapter，这四个里边，HandlerFunctionAdapter 用来处理 HandlerFunction 类型的接口，函数式 Web 框架现在呼声很高，但是实际应用的并不多，因此这里我们暂且忽略它，来看另外三个。

### HttpRequestHandlerAdapter
HttpRequestHandlerAdapter 主要用来处理实现了 HttpRequestHandler 接口的 handler，例如像下面这种：
```
public class HelloController01 implements HttpRequestHandler {
    @Override
    public void handleRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        response.getWriter().write("hello");
    }
}
```
HttpRequestHandlerAdapter#handle 方法也很简单，直接调用 handleRequest：
```
public class HttpRequestHandlerAdapter implements HandlerAdapter {
 @Override
 public boolean supports(Object handler) {
  return (handler instanceof HttpRequestHandler);
 }
 @Override
 @Nullable
 public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
   throws Exception {
  ((HttpRequestHandler) handler).handleRequest(request, response);
  return null;
 }
 @Override
 public long getLastModified(HttpServletRequest request, Object handler) {
  if (handler instanceof LastModified) {
   return ((LastModified) handler).getLastModified(request);
  }
  return -1L;
 }
}
```
可以看到，supports 方法判断 handler 的类型是否是 HttpRequestHandler，handle 方法则直接调用 handleRequest。

### SimpleControllerHandlerAdapter
SimpleControllerHandlerAdapter 主要用来处理实现了 Controller 接口的 handler，像下面这样：
```
public class HelloController02 implements Controller {
    @Override
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        return new ModelAndView("hello");
    }
}
```
SimpleControllerHandlerAdapter 的定义也很简单：
```
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

 @Override
 public boolean supports(Object handler) {
  return (handler instanceof Controller);
 }

 @Override
 @Nullable
 public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
   throws Exception {

  return ((Controller) handler).handleRequest(request, response);
 }

 @Override
 public long getLastModified(HttpServletRequest request, Object handler) {
  if (handler instanceof LastModified) {
   return ((LastModified) handler).getLastModified(request);
  }
  return -1L;
 }

}
```

### SimpleServletHandlerAdapter
这个用来处理实现了 Servlet 接口的 handler，在实际开发中我们很少用到这种，我就不多说，我们直接来看它的源码：
```
public class SimpleServletHandlerAdapter implements HandlerAdapter {

 @Override
 public boolean supports(Object handler) {
  return (handler instanceof Servlet);
 }

 @Override
 @Nullable
 public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
   throws Exception {

  ((Servlet) handler).service(request, response);
  return null;
 }

 @Override
 public long getLastModified(HttpServletRequest request, Object handler) {
  return -1;
 }

}
```
可以看到，也很简单！handle 方法中直接调用 servlet 的 service 方法。

可以看到，这三种 HandlerAdapter 简单的原因主要是因为要调用的方法比较简单，直接调用就可以了。而 RequestMappingHandlerAdapter 复杂是因为调用的方法名不固定，所以复杂。

## RequestMappingHandlerAdapter
RequestMappingHandlerAdapter 继承自 AbstractHandlerMethodAdapter，我们先来看看 AbstractHandlerMethodAdapter。

### AbstractHandlerMethodAdapter
```
public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {
 private int order = Ordered.LOWEST_PRECEDENCE;
 public AbstractHandlerMethodAdapter() {
  super(false);
 }
 public void setOrder(int order) {
  this.order = order;
 }
 @Override
 public int getOrder() {
  return this.order;
 }
 @Override
 public final boolean supports(Object handler) {
  return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
 }
 protected abstract boolean supportsInternal(HandlerMethod handlerMethod);
 @Override
 @Nullable
 public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
   throws Exception {

  return handleInternal(request, response, (HandlerMethod) handler);
 }
 @Nullable
 protected abstract ModelAndView handleInternal(HttpServletRequest request,
   HttpServletResponse response, HandlerMethod handlerMethod) throws Exception;
 @Override
 public final long getLastModified(HttpServletRequest request, Object handler) {
  return getLastModifiedInternal(request, (HandlerMethod) handler);
 }
 protected abstract long getLastModifiedInternal(HttpServletRequest request, HandlerMethod handlerMethod);
}
```
AbstractHandlerMethodAdapter 还是比较简单的，前面我们说的三个方法还是没变，只不过三个方法里边分别调用了 supportsInternal、handleInternal 以及 getLastModifiedInternal，而这些 xxxInternal 都将在子类 RequestMappingHandlerAdapter 中被实现。另外 AbstractHandlerMethodAdapter 实现了 Ordered 接口，意味着在配置的时候还可以设置优先级。

这就是 AbstractHandlerMethodAdapter，比较简单。

## RequestMappingHandlerAdapter
RequestMappingHandlerAdapter 承担了大部分的执行工作，通过前面的介绍，我们已经大致知道 RequestMappingHandlerAdapter 中主要实现了三个方法：supportsInternal、handleInternal 以及 getLastModifiedInternal。其中 supportsInternal 总是返回 true，意味着在 RequestMappingHandlerAdapter 中不做任何比较，getLastModifiedInternal 则直接返回 -1，所以对它来说，最重要的其实是 handleInternal 方法。

整体上来说，RequestMappingHandlerAdapter 要解决三方面的问题：

处理请求参数。
调用处理器执行请求。
处理请求响应。
三个问题第一个最复杂！大家想想我们平时定义接口时，接口参数是不固定的，有几个参数、参数类型是什么都不确定，有的时候我们还利用 @ControllerAdvice 注解标记了一些全局参数，等等这些都要考虑进来，所以这一步是最复杂的，剩下的两步就比较容易了。对于参数的处理，SpringMVC 中提供了很多参数解析器，在接下来的源码分析中，我们将一步一步见识到这些参数解析器。

### 初始化过程
那么接下来我们就先来看 RequestMappingHandlerAdapter 的初始化过程，由于它实现了 InitializingBean 接口，因此 afterPropertiesSet 方法会被自动调用，在 afterPropertiesSet 方法中，完成了各种参数解析器的初始化：
```
@Override
public void afterPropertiesSet() {
 // Do this first, it may add ResponseBody advice beans
 initControllerAdviceCache();
 if (this.argumentResolvers == null) {
  List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
  this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
 }
 if (this.initBinderArgumentResolvers == null) {
  List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
  this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
 }
 if (this.returnValueHandlers == null) {
  List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
  this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
 }
}
private void initControllerAdviceCache() {
 if (getApplicationContext() == null) {
  return;
 }
 List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
 List<Object> requestResponseBodyAdviceBeans = new ArrayList<>();
 for (ControllerAdviceBean adviceBean : adviceBeans) {
  Class<?> beanType = adviceBean.getBeanType();
  if (beanType == null) {
   throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
  }
  Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
  if (!attrMethods.isEmpty()) {
   this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
  }
  Set<Method> binderMethods = MethodIntrospector.selectMethods(beanType, INIT_BINDER_METHODS);
  if (!binderMethods.isEmpty()) {
   this.initBinderAdviceCache.put(adviceBean, binderMethods);
  }
  if (RequestBodyAdvice.class.isAssignableFrom(beanType) || ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
   requestResponseBodyAdviceBeans.add(adviceBean);
  }
 }
 if (!requestResponseBodyAdviceBeans.isEmpty()) {
  this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
 }
}
```
这里首先调用 initControllerAdviceCache 方法对包含了 @ControllerAdvice 注解的类进行处理
具体的处理思路如下：

首先查找到所有标记了 @ControllerAdvice 注解的 Bean，将查找结果保存到 adviceBeans 变量中。
接下来遍历 adviceBeans，找到对象中包含 @ModelAttribute 注解的方法，将查找的结果保存到 modelAttributeAdviceCache 变量中。
找到对象中包含 @InitBinder 注解的方法，将查找的结果保存到 initBinderAdviceCache 变量中。
查找实现了 RequestBodyAdvice 或者 ResponseBodyAdvice 接口的类，并最终将查找结果添加到 requestResponseBodyAdvice 中。实现了 RequestBodyAdvice 或者 ResponseBodyAdvice 接口的类一般需要用 @ControllerAdvice 注解标记，关于这两个接口的详细用法，读者可以参考松哥之前的文章，传送门如何优雅的实现 Spring Boot 接口参数加密解密？。另外这里还需要注意，找到的 adviceBean 并没有直接放到全局变量中，而是先放在局部变量中，然后才添加到全局的 requestResponseBodyAdvice 中，这种方式可以确保 adviceBean 始终处于集合的最前面。
这就是 initControllerAdviceCache 方法的处理逻辑，主要是解决了一些全局参数的处理问题。

我们再回到 afterPropertiesSet 方法中，接下来就是对 argumentResolvers、initBinderArgumentResolvers 以及 returnValueHandlers 的初始化了。

前两个参数相关的解析器，都是通过 getDefaultXXX 方法获取的，并且把获取的结果添加到 HandlerMethodArgumentResolverComposite 中，这种 xxxComposite，大家一看就知道是一个责任链模式，这个里边管理了诸多的参数解析器，但是它自己不干活，需要工作的时候，它就负责遍历它所管理的参数解析器，让那些参数解析器去处理参数问题。我们先来看看它的 getDefaultXXX 方法：
```
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
 List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);
 // Annotation-based argument resolution
 resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
 resolvers.add(new RequestParamMapMethodArgumentResolver());
 resolvers.add(new PathVariableMethodArgumentResolver());
 resolvers.add(new PathVariableMapMethodArgumentResolver());
 resolvers.add(new MatrixVariableMethodArgumentResolver());
 resolvers.add(new MatrixVariableMapMethodArgumentResolver());
 resolvers.add(new ServletModelAttributeMethodProcessor(false));
 resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
 resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
 resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
 resolvers.add(new RequestHeaderMapMethodArgumentResolver());
 resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
 resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
 resolvers.add(new SessionAttributeMethodArgumentResolver());
 resolvers.add(new RequestAttributeMethodArgumentResolver());
 // Type-based argument resolution
 resolvers.add(new ServletRequestMethodArgumentResolver());
 resolvers.add(new ServletResponseMethodArgumentResolver());
 resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
 resolvers.add(new RedirectAttributesMethodArgumentResolver());
 resolvers.add(new ModelMethodProcessor());
 resolvers.add(new MapMethodProcessor());
 resolvers.add(new ErrorsMethodArgumentResolver());
 resolvers.add(new SessionStatusMethodArgumentResolver());
 resolvers.add(new UriComponentsBuilderMethodArgumentResolver());
 if (KotlinDetector.isKotlinPresent()) {
  resolvers.add(new ContinuationHandlerMethodArgumentResolver());
 }
 // Custom arguments
 if (getCustomArgumentResolvers() != null) {
  resolvers.addAll(getCustomArgumentResolvers());
 }
 // Catch-all
 resolvers.add(new PrincipalMethodArgumentResolver());
 resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
 resolvers.add(new ServletModelAttributeMethodProcessor(true));
 return resolvers;
}
private List<HandlerMethodArgumentResolver> getDefaultInitBinderArgumentResolvers() {
 List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(20);
 // Annotation-based argument resolution
 resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
 resolvers.add(new RequestParamMapMethodArgumentResolver());
 resolvers.add(new PathVariableMethodArgumentResolver());
 resolvers.add(new PathVariableMapMethodArgumentResolver());
 resolvers.add(new MatrixVariableMethodArgumentResolver());
 resolvers.add(new MatrixVariableMapMethodArgumentResolver());
 resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
 resolvers.add(new SessionAttributeMethodArgumentResolver());
 resolvers.add(new RequestAttributeMethodArgumentResolver());
 // Type-based argument resolution
 resolvers.add(new ServletRequestMethodArgumentResolver());
 resolvers.add(new ServletResponseMethodArgumentResolver());
 // Custom arguments
 if (getCustomArgumentResolvers() != null) {
  resolvers.addAll(getCustomArgumentResolvers());
 }
 // Catch-all
 resolvers.add(new PrincipalMethodArgumentResolver());
 resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
 return resolvers;
}
```
这两个方法虽然挺长的，但是内容都比较简单。在 getDefaultArgumentResolvers 方法中，解析器被分为四类：

通过注解标记的参数对应的解析器。
通过类型解析的解析器。
自定义的解析器。
兜底的解析器（大部分前面搞不定的参数后面的都能搞定，简单数据类型就是在这里处理的）。
getDefaultInitBinderArgumentResolvers 方法也是类似的，我就不再赘述。

所谓的解析器实际上就是把参数从对应的介质中提取出来，然后交给方法对应的变量。如果项目有需要，也可以自定义参数解析器，自定义的参数解析器设置到 RequestMappingHandlerAdapter#customArgumentResolvers 属性上，在调用的时候，前两种参数解析器都匹配不上的时候，自定义的参数解析器才会有用，而且这个顺序是固定的无法改变的。

## 请求执行过程
根据前面的介绍，请求执行的入口方法实际上就是 handleInternal，所以这里我们就从 handleInternal 方法开始分析：
```
@Override
protected ModelAndView handleInternal(HttpServletRequest request,
  HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
 ModelAndView mav;
 checkRequest(request);
 // Execute invokeHandlerMethod in synchronized block if required.
 if (this.synchronizeOnSession) {
  HttpSession session = request.getSession(false);
  if (session != null) {
   Object mutex = WebUtils.getSessionMutex(session);
   synchronized (mutex) {
    mav = invokeHandlerMethod(request, response, handlerMethod);
   }
  }
  else {
   // No HttpSession available -> no mutex necessary
   mav = invokeHandlerMethod(request, response, handlerMethod);
  }
 }
 else {
  // No synchronization on session demanded at all...
  mav = invokeHandlerMethod(request, response, handlerMethod);
 }
 if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
  if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
   applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
  }
  else {
   prepareResponse(response);
  }
 }
 return mav;
}
```
不过这个 handleInternal 方法看起来平平无奇，没啥特别之处。仔细看起来，它里边就做了三件事：

checkRequest 方法检查请求。
invokeHandlerMethod 方法执行处理器方法，这个是核心。
处理缓存问题。
那么接下来我们就对这三个问题分别进行分析。

### checkRequest
```
protected final void checkRequest(HttpServletRequest request) throws ServletException {
 // Check whether we should support the request method.
 String method = request.getMethod();
 if (this.supportedMethods != null && !this.supportedMethods.contains(method)) {
  throw new HttpRequestMethodNotSupportedException(method, this.supportedMethods);
 }
 // Check whether a session is required.
 if (this.requireSession && request.getSession(false) == null) {
  throw new HttpSessionRequiredException("Pre-existing session required but none found");
 }
}
```
检查就检查了两个东西：

是否支持请求方法
当 supportedMethods 不为空的时候，去检查是否支持请求方法。默认情况下，supportedMethods 为 null，所以默认情况下是不检查请求方法的。如果需要检查，可以在注册 RequestMappingHandlerAdapter 的时候进行配置，如果在构造方法中设置 restrictDefaultSupportedMethods 变量为 true，那么默认情况下只支持 GET、POST、以及 HEAD 三种请求，不过这个参数改起来比较麻烦，默认在父类 AbstractHandlerMethodAdapter 类中写死了。

是否需要 session
如果 requireSession 为 true，就检查一下 session 是否存在，默认情况下，requireSession 为 false，所以这里也是不检查的。

invokeHandlerMethod
接下来就是 invokeHandlerMethod 方法的调用了：
```
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
  HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
 ServletWebRequest webRequest = new ServletWebRequest(request, response);
 try {
  WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
  ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
  ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
  if (this.argumentResolvers != null) {
   invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
  }
  if (this.returnValueHandlers != null) {
   invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
  }
  invocableMethod.setDataBinderFactory(binderFactory);
  invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
  ModelAndViewContainer mavContainer = new ModelAndViewContainer();
  mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
  modelFactory.initModel(webRequest, mavContainer, invocableMethod);
  mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
  AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
  asyncWebRequest.setTimeout(this.asyncRequestTimeout);
  WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
  asyncManager.setTaskExecutor(this.taskExecutor);
  asyncManager.setAsyncWebRequest(asyncWebRequest);
  asyncManager.registerCallableInterceptors(this.callableInterceptors);
  asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);
  if (asyncManager.hasConcurrentResult()) {
   Object result = asyncManager.getConcurrentResult();
   mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
   asyncManager.clearConcurrentResult();
   LogFormatUtils.traceDebug(logger, traceOn -> {
    String formatted = LogFormatUtils.formatValue(result, !traceOn);
    return "Resume with async result [" + formatted + "]";
   });
   invocableMethod = invocableMethod.wrapConcurrentResult(result);
  }
  invocableMethod.invokeAndHandle(webRequest, mavContainer);
  if (asyncManager.isConcurrentHandlingStarted()) {
   return null;
  }
  return getModelAndView(mavContainer, modelFactory, webRequest);
 }
 finally {
  webRequest.requestCompleted();
 }
}
```
这个方法里涉及到的组件比较多，不过大部分组件在前面的文章中松哥都已经和大家介绍过了，因此这里理解起来应该并不难。

首先获取一个 WebDataBinderFactory 对象，该对象将用来构建 WebDataBinder。
接下来获取一个 ModelFactory 对象，该对象用来初始化/更新 Model 对象
接下来创建 ServletInvocableHandlerMethod 对象，一会方法的调用，将由它完成。
接下来给 invocableMethod 把需要的参数都安排上。
构造一个 ModelAndViewContainer 对象，将来用来存储 Model 和 View。
把 FlashMap 中的数据先添加进 ModelAndViewContainer 容器中。
接下来初始化 Model，处理 @SessionAttributes 注解和 WebDataBinder 定义的全局数据，同时配置是否在重定向时忽略 defaultModel。
接下来处理异步请求情况，判断是否有异步请求结果。
调用 invokeAndHandle 方法去真正执行接口方法。
如果是异步请求，则直接返回即可。
接下来调用 getModelAndView 方法去构造 ModelAndView 并返回，在该方法中，首先会去更新 Model，更新的时候会去处理 SessionAttribute 同时配置 BindingResult；然后会根据 ModelAndViewContainer 去创建一个 ModelAndView 对象；最后，如果 ModelAndViewContainer 中的 Model 是 RedirectAttributes 类型，则将其设置到 FlashMap 中。
最后设置请求完成。

处理缓存
缓存的处理主要是针对响应头的 Cache-Control 字段。







































