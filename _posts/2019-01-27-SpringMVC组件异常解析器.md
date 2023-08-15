---
layout: post
categories: [SpringMV]
description: none
keywords: Spring
---
# SpringMVC组件异常解析器
SpringMVC 中针对异常问题有一套完整的处理体系

## 异常解析器概览
在 SpringMVC 的异常体系中，处于最顶层的大 Boss 是 HandlerExceptionResolver，这是一个接口，里边只有一个方法：
```
public interface HandlerExceptionResolver {
 @Nullable
 ModelAndView resolveException(
   HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);
}
```
resolveException 方法就用来解析请求处理过程中所产生的异常，并最终返回一个 ModelAndView。

我们来看下 HandlerExceptionResolver 的实现类：
直接实现 HandlerExceptionResolver 接口的类有三个：

- HandlerExceptionResolverComposite：这个一看又是一个组合。
- DefaultErrorAttributes：这个用来保存异常属性。
- AbstractHandlerExceptionResolver：这个的子类比较多：
  - SimpleMappingExceptionResolver：通过提前配置好的异常类和 View 之间的对应关系来解析异常。
  - AbstractHandlerMethodExceptionResolver：处理使用 @ExceptionHandler 注解自定义的异常类型。
  - DefaultHandlerExceptionResolver：按照不同类型来处理异常。
  - ResponseStatusExceptionResolver：处理含有 @ResponseStatus 注解的异常。

在 SpringMVC 中，大致的异常解析器就是这些，接下来我们来逐个学习这些异常解析器。

## AbstractHandlerExceptionResolver
AbstractHandlerExceptionResolver 是真正干活的异常解析器的父类，我们就先从他的 resolveException 方法开始看起。
```
@Override
@Nullable
public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
 if (shouldApplyTo(request, handler)) {
  prepareResponse(ex, response);
  ModelAndView result = doResolveException(request, response, handler, ex);
  if (result != null) {
   logException(ex, request);
  }
  return result;
 }
 else {
  return null;
 }
}
```
- 首先调用 shouldApplyTo 方法判断当前解析器是否可以处理传入的处理器所抛出的异常，如果不支持，则直接返回 null，这个异常将交给下一个 HandlerExceptionResolver 去处理。
- 调用 prepareResponse 方法处理 response。
- 调用 doResolveException 方法实际处理异常，这是一个模版方法，具体的实现在子类中。
- 调用 logException 方法记录异常日志信息。

记录异常日志没啥好说的，doResolveException 则是一个空的模版方法，所以这里对我们来说主要就是两个方法：shouldApplyTo 和 prepareResponse，我们分别来看。

### shouldApplyTo
```
protected boolean shouldApplyTo(HttpServletRequest request, @Nullable Object handler) {
 if (handler != null) {
  if (this.mappedHandlers != null && this.mappedHandlers.contains(handler)) {
   return true;
  }
  if (this.mappedHandlerClasses != null) {
   for (Class<?> handlerClass : this.mappedHandlerClasses) {
    if (handlerClass.isInstance(handler)) {
     return true;
    }
   }
  }
 }
 return !hasHandlerMappings();
}
```

这里涉及到了两个对象：mappedHandlers 和 mappedHandlerClasses：
- mappedHandlers：存储的是处理器对象（Controller 或者 Controller 中的方法）
- mappedHandlerClasses：存储的是处理器的 Class。

我们在配置异常解析器的时候可以配置这两个对象，进而实现该异常处理器只为某一个处理器服务，但是一般来说没这种需求，所以大家仅做了解即可。

如果开发者一开始配置了 mappedHandlers 或者 mappedHandlerClasses，则用这两个和处理器去比较，否则就直接返回 true，表示支持该异常处理。

### prepareResponse
prepareResponse 方法比较简单，主要是处理一下响应头的缓存字段。
```
protected void prepareResponse(Exception ex, HttpServletResponse response) {
 if (this.preventResponseCaching) {
  preventCaching(response);
 }
}
protected void preventCaching(HttpServletResponse response) {
 response.addHeader(HEADER_CACHE_CONTROL, "no-store");
}
```
这是 AbstractHandlerExceptionResolver 的大致内容，可以看到还是非常 easy 的，接下来我们来看它的实现类。

### AbstractHandlerMethodExceptionResolver
AbstractHandlerMethodExceptionResolver 主要是重写了 shouldApplyTo 方法和 doResolveException 方法，一个一个来看。

shouldApplyTo
```
@Override
protected boolean shouldApplyTo(HttpServletRequest request, @Nullable Object handler) {
 if (handler == null) {
  return super.shouldApplyTo(request, null);
 }
 else if (handler instanceof HandlerMethod) {
  HandlerMethod handlerMethod = (HandlerMethod) handler;
  handler = handlerMethod.getBean();
  return super.shouldApplyTo(request, handler);
 }
 else if (hasGlobalExceptionHandlers() && hasHandlerMappings()) {
  return super.shouldApplyTo(request, handler);
 }
 else {
  return false;
 }
}
```
判断逻辑基本上都还是调用父类的 shouldApplyTo 方法去处理。

doResolveException
```
@Override
@Nullable
protected final ModelAndView doResolveException(
  HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
 HandlerMethod handlerMethod = (handler instanceof HandlerMethod ? (HandlerMethod) handler : null);
 return doResolveHandlerMethodException(request, response, handlerMethod, ex);
}
@Nullable
protected abstract ModelAndView doResolveHandlerMethodException(
  HttpServletRequest request, HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception ex);
```
doResolveException 是具体的异常处理方法，但是它里边却没有实质性操作，具体的事情交给 doResolveHandlerMethodException 方法去做了，而该方法是一个抽象方法，具体的实现在子类中。

### ExceptionHandlerExceptionResolver
AbstractHandlerMethodExceptionResolver 只有一个子类就是 ExceptionHandlerExceptionResolver，来看下它的 doResolveHandlerMethodException 方法：
```
@Override
@Nullable
protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
  HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {
 ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
 if (exceptionHandlerMethod == null) {
  return null;
 }
 if (this.argumentResolvers != null) {
  exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
 }
 if (this.returnValueHandlers != null) {
  exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
 }
 ServletWebRequest webRequest = new ServletWebRequest(request, response);
 ModelAndViewContainer mavContainer = new ModelAndViewContainer();
 ArrayList<Throwable> exceptions = new ArrayList<>();
 try {
  if (logger.isDebugEnabled()) {
   logger.debug("Using @ExceptionHandler " + exceptionHandlerMethod);
  }
  // Expose causes as provided arguments as well
  Throwable exToExpose = exception;
  while (exToExpose != null) {
   exceptions.add(exToExpose);
   Throwable cause = exToExpose.getCause();
   exToExpose = (cause != exToExpose ? cause : null);
  }
  Object[] arguments = new Object[exceptions.size() + 1];
  exceptions.toArray(arguments);  // efficient arraycopy call in ArrayList
  arguments[arguments.length - 1] = handlerMethod;
  exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, arguments);
 }
 catch (Throwable invocationEx) {
  // Any other than the original exception (or a cause) is unintended here,
  // probably an accident (e.g. failed assertion or the like).
  if (!exceptions.contains(invocationEx) && logger.isWarnEnabled()) {
   logger.warn("Failure in @ExceptionHandler " + exceptionHandlerMethod, invocationEx);
  }
  // Continue with default processing of the original exception...
  return null;
 }
 if (mavContainer.isRequestHandled()) {
  return new ModelAndView();
 }
 else {
  ModelMap model = mavContainer.getModel();
  HttpStatus status = mavContainer.getStatus();
  ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, status);
  mav.setViewName(mavContainer.getViewName());
  if (!mavContainer.isViewReference()) {
   mav.setView((View) mavContainer.getView());
  }
  if (model instanceof RedirectAttributes) {
   Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
   RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
  }
  return mav;
 }
}
```
这个方法虽然比较长，但是很好理解：

首先查找到带有 @ExceptionHandler 注解的方法，封装成一个 ServletInvocableHandlerMethod 对象。
如果找到了对应的方法，则为 exceptionHandlerMethod 配置参数解析器、视图解析器等，。
接下来定义一个 exceptions 数组，如果发生的异常存在异常链，则将整个异常链存入 exceptions 数组中。
exceptions 数组再加上 handlerMethod，共同组成方法参数，调用 exceptionHandlerMethod.invokeAndHandle 完成自定义异常方法的执行，执行结果被保存再 mavContainer 中。
如果请求到此结束，则直接构造一个 ModelAndView 返回。
否则从 mavContainer 中取出各项信息，构建新的 ModelAndView 返回。同时，如果存在重定向参数，也将之保存下来。
这就是 ExceptionHandlerExceptionResolver 的大致工作流程，可以看到，还是非常 easy 的。

### DefaultHandlerExceptionResolver
这个看名字就知道是一个默认的异常处理器，用来处理一些常见的异常类型，我们来看一下它的 doResolveException 方法：
```
@Override
@Nullable
protected ModelAndView doResolveException(
  HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
 try {
  if (ex instanceof HttpRequestMethodNotSupportedException) {
   return handleHttpRequestMethodNotSupported(
     (HttpRequestMethodNotSupportedException) ex, request, response, handler);
  }
  else if (ex instanceof HttpMediaTypeNotSupportedException) {
   return handleHttpMediaTypeNotSupported(
     (HttpMediaTypeNotSupportedException) ex, request, response, handler);
  }
  else if (ex instanceof HttpMediaTypeNotAcceptableException) {
   return handleHttpMediaTypeNotAcceptable(
     (HttpMediaTypeNotAcceptableException) ex, request, response, handler);
  }
  else if (ex instanceof MissingPathVariableException) {
   return handleMissingPathVariable(
     (MissingPathVariableException) ex, request, response, handler);
  }
  else if (ex instanceof MissingServletRequestParameterException) {
   return handleMissingServletRequestParameter(
     (MissingServletRequestParameterException) ex, request, response, handler);
  }
  else if (ex instanceof ServletRequestBindingException) {
   return handleServletRequestBindingException(
     (ServletRequestBindingException) ex, request, response, handler);
  }
  else if (ex instanceof ConversionNotSupportedException) {
   return handleConversionNotSupported(
     (ConversionNotSupportedException) ex, request, response, handler);
  }
  else if (ex instanceof TypeMismatchException) {
   return handleTypeMismatch(
     (TypeMismatchException) ex, request, response, handler);
  }
  else if (ex instanceof HttpMessageNotReadableException) {
   return handleHttpMessageNotReadable(
     (HttpMessageNotReadableException) ex, request, response, handler);
  }
  else if (ex instanceof HttpMessageNotWritableException) {
   return handleHttpMessageNotWritable(
     (HttpMessageNotWritableException) ex, request, response, handler);
  }
  else if (ex instanceof MethodArgumentNotValidException) {
   return handleMethodArgumentNotValidException(
     (MethodArgumentNotValidException) ex, request, response, handler);
  }
  else if (ex instanceof MissingServletRequestPartException) {
   return handleMissingServletRequestPartException(
     (MissingServletRequestPartException) ex, request, response, handler);
  }
  else if (ex instanceof BindException) {
   return handleBindException((BindException) ex, request, response, handler);
  }
  else if (ex instanceof NoHandlerFoundException) {
   return handleNoHandlerFoundException(
     (NoHandlerFoundException) ex, request, response, handler);
  }
  else if (ex instanceof AsyncRequestTimeoutException) {
   return handleAsyncRequestTimeoutException(
     (AsyncRequestTimeoutException) ex, request, response, handler);
  }
 }
 catch (Exception handlerEx) {
 }
 return null;
}
```
可以看到，这里实际上就是根据不同的异常类型，然后调用不同的类去处理该异常。这里相关的处理都比较容易，以 HttpRequestMethodNotSupportedException 为例，异常处理就是对 response 对象做一些配置，如下：
```
protected ModelAndView handleHttpRequestMethodNotSupported(HttpRequestMethodNotSupportedException ex,
  HttpServletRequest request, HttpServletResponse response, @Nullable Object handler) throws IOException {
 String[] supportedMethods = ex.getSupportedMethods();
 if (supportedMethods != null) {
  response.setHeader("Allow", StringUtils.arrayToDelimitedString(supportedMethods, ", "));
 }
 response.sendError(HttpServletResponse.SC_METHOD_NOT_ALLOWED, ex.getMessage());
 return new ModelAndView();
}
```
配置响应头，然后 sendError，最后返回一个空的 ModelAndView 对象。

## ResponseStatusExceptionResolver
这个用来处理 ResponseStatusException 类型的异常，或者使用了 @ResponseStatus 注解标记的普通异常类。我们来看下它的 doResolveException 方法：
```
@Override
@Nullable
protected ModelAndView doResolveException(
  HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
 try {
  if (ex instanceof ResponseStatusException) {
   return resolveResponseStatusException((ResponseStatusException) ex, request, response, handler);
  }
  ResponseStatus status = AnnotatedElementUtils.findMergedAnnotation(ex.getClass(), ResponseStatus.class);
  if (status != null) {
   return resolveResponseStatus(status, request, response, handler, ex);
  }
  if (ex.getCause() instanceof Exception) {
   return doResolveException(request, response, handler, (Exception) ex.getCause());
  }
 }
 catch (Exception resolveEx) {
 }
 return null;
}
```
可以看到，首先判断异常类型是不是 ResponseStatusException，如果是，则直接调用 resolveResponseStatusException 方法进行异常信息处理，如果不是，则去查找到异常类上的 @ResponseStatus 注解，并从中查找出相关的异常信息，然后调用 resolveResponseStatus 方法进行处理。

可以看到，ResponseStatusExceptionResolver 处理的异常类型有两种：

直接继承自 ResponseStatusException 的异常类，这种异常类可以直接从里边提取出来想要的信息。
通过 @ResponseStatus 注解的普通异常类，这种情况下异常信息从 @ResponseStatus 注解中提取出来。
这个比较简单，没啥好说的。

### SimpleMappingExceptionResolver
SimpleMappingExceptionResolver 则是根据不同的异常显示不同的 error 页面。可能有的小伙伴还没用过 SimpleMappingExceptionResolver，所以松哥这里先简单说一下用法。

SimpleMappingExceptionResolver 的配置非常简单，直接提供一个 SimpleMappingExceptionResolver 的实例即可，如下：
```
@Bean
SimpleMappingExceptionResolver simpleMappingExceptionResolver() {
    SimpleMappingExceptionResolver resolver = new SimpleMappingExceptionResolver();
    Properties mappings = new Properties();
    mappings.put("java.lang.ArithmeticException", "11");
    mappings.put("java.lang.NullPointerException", "22");
    resolver.setExceptionMappings(mappings);
    Properties statusCodes = new Properties();
    statusCodes.put("11", "500");
    statusCodes.put("22", "500");
    resolver.setStatusCodes(statusCodes);
    return resolver;
}
```
在 mappings 中配置异常和 view 之间的对应关系，要写异常类的全路径，后面的 11、22 则表示视图名称；statusCodes 中配置了视图和响应状态码之间的映射关系。配置完成后，如果我们的项目在运行时抛出了 ArithmeticException 异常，则会展示出 11 视图，如果我们的项目在运行时抛出了 NullPointerException 异常，则会展示出 22 视图。

这是用法，了解了用法之后我们再来看源码，就容易理解了，我们直接来看 doResolveException 方法：
```
@Override
@Nullable
protected ModelAndView doResolveException(
  HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
 String viewName = determineViewName(ex, request);
 if (viewName != null) {
  Integer statusCode = determineStatusCode(request, viewName);
  if (statusCode != null) {
   applyStatusCodeIfPossible(request, response, statusCode);
  }
  return getModelAndView(viewName, ex, request);
 }
 else {
  return null;
 }
}
```
首先调用 determineViewName 方法确定视图的名称。
接下来调用 determineStatusCode 查看视图是否有对应的 statusCode。
调用 applyStatusCodeIfPossible 方法将 statusCode 设置到 response 上，这个方法很简单，不多说。
调用 getModelAndView 方法构造一个 ModelAndView 对象返回，在构造时，同时设置异常参数，异常的信息的 key 默认就是 exception。
在上面这个过程中，有两个比较长的方法，松哥这里需要和大家额外多说两句。

### determineViewName
这个就是根据异常类型找到视图名，我们来看下具体的查找方式：

```
@Nullable
protected String determineViewName(Exception ex, HttpServletRequest request) {
 String viewName = null;
 if (this.excludedExceptions != null) {
  for (Class<?> excludedEx : this.excludedExceptions) {
   if (excludedEx.equals(ex.getClass())) {
    return null;
   }
  }
 }
 if (this.exceptionMappings != null) {
  viewName = findMatchingViewName(this.exceptionMappings, ex);
 }
 if (viewName == null && this.defaultErrorView != null) {
  viewName = this.defaultErrorView;
 }
 return viewName;
}
```
如果当前异常包含在 excludedExceptions 中，则直接返回 null（意思是当前异常被忽略处理了，直接按照默认方式来）。
如果 exceptionMappings 不为 null，则直接调用 findMatchingViewName 方法查找异常对应的视图名（exceptionMappings 变量就是前面我们配置的映射关系），具体的查找方式就是遍历我们前面配置的映射表。
如果没找到对应的 viewName，并且用户配置了 defaultErrorView，则将 defaultErrorView 赋值给 viewName，并将 viewName 返回。

determineStatusCode
```
@Nullable
protected Integer determineStatusCode(HttpServletRequest request, String viewName) {
 if (this.statusCodes.containsKey(viewName)) {
  return this.statusCodes.get(viewName);
 }
 return this.defaultStatusCode;
}
```
这个就比较容易，直接去 statusCodes 中查看是否有视图对应的状态码，如果有则直接返回，如果没有，就返回一个默认的。

## HandlerExceptionResolverComposite
最后，还有一个 HandlerExceptionResolverComposite 需要和大家介绍下，这是一个组合的异常处理器，用来代理哪些真正干活的异常处理器。
```
@Override
@Nullable
public ModelAndView resolveException(
  HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
 if (this.resolvers != null) {
  for (HandlerExceptionResolver handlerExceptionResolver : this.resolvers) {
   ModelAndView mav = handlerExceptionResolver.resolveException(request, response, handler, ex);
   if (mav != null) {
    return mav;
   }
  }
 }
 return null;
}
```
