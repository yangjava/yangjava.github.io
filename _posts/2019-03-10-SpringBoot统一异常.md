---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot统一异常
SpringBoot接口如何对异常进行统一封装，并统一返回呢？

## 为什么要优雅的处理异常
如果我们不统一的处理异常，经常会在controller层有大量的异常处理的代码， 比如：
```java
@Slf4j
@Api(value = "User Interfaces", tags = "User Interfaces")
@RestController
@RequestMapping("/user")
public class UserController {

    /**
     * http://localhost:8080/user/add .
     *
     * @param userParam user param
     * @return user
     */
    @ApiOperation("Add User")
    @ApiImplicitParam(name = "userParam", type = "body", dataTypeClass = UserParam.class, required = true)
    @PostMapping("add")
    public ResponseEntity<String> add(@Valid @RequestBody UserParam userParam) {
        // 每个接口充斥着大量的异常处理
        try {
            // do something
        } catch(Exception e) {
            return ResponseEntity.fail("error");
        }
        return ResponseEntity.ok("success");
    }
}
```
## 实现案例
简单展示通过@ControllerAdvice进行统一异常处理。
### @ControllerAdvice异常统一处理
对于400参数错误异常
```java
/**
 * Global exception handler.
 *
 */
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * exception handler for bad request.
     *
     * @param e
     *            exception
     * @return ResponseResult
     */
    @ResponseBody
    @ResponseStatus(code = HttpStatus.BAD_REQUEST)
    @ExceptionHandler(value = { BindException.class, ValidationException.class, MethodArgumentNotValidException.class })
    public ResponseResult<ExceptionData> handleParameterVerificationException(@NonNull Exception e) {
        ExceptionData.ExceptionDataBuilder exceptionDataBuilder = ExceptionData.builder();
        log.warn("Exception: {}", e.getMessage());
        if (e instanceof BindException) {
            BindingResult bindingResult = ((MethodArgumentNotValidException) e).getBindingResult();
            bindingResult.getAllErrors().stream().map(DefaultMessageSourceResolvable::getDefaultMessage)
                    .forEach(exceptionDataBuilder::error);
        } else if (e instanceof ConstraintViolationException) {
            if (e.getMessage() != null) {
                exceptionDataBuilder.error(e.getMessage());
            }
        } else {
            exceptionDataBuilder.error("invalid parameter");
        }
        return ResponseResultEntity.fail(exceptionDataBuilder.build(), "invalid parameter");
    }

}
```
对于自定义异常
```java
/**
 * handle business exception.
 *
 * @param businessException
 *            business exception
 * @return ResponseResult
 */
@ResponseBody
@ExceptionHandler(BusinessException.class)
public ResponseResult<BusinessException> processBusinessException(BusinessException businessException) {
    log.error(businessException.getLocalizedMessage(), businessException);
    // 这里可以屏蔽掉后台的异常栈信息，直接返回"business error"
    return ResponseResultEntity.fail(businessException, businessException.getLocalizedMessage());
}

```
对于其它异常
```java
/**
 * handle other exception.
 *
 * @param exception
 *            exception
 * @return ResponseResult
 */
@ResponseBody
@ExceptionHandler(Exception.class)
public ResponseResult<Exception> processException(Exception exception) {
    log.error(exception.getLocalizedMessage(), exception);
    // 这里可以屏蔽掉后台的异常栈信息，直接返回"server error"
    return ResponseResultEntity.fail(exception, exception.getLocalizedMessage());
}
```
Controller接口 （接口中无需处理异常）
```java
@Slf4j
@Api(value = "User Interfaces", tags = "User Interfaces")
@RestController
@RequestMapping("/user")
public class UserController {

    /**
     * http://localhost:8080/user/add .
     *
     * @param userParam user param
     * @return user
     */
    @ApiOperation("Add User")
    @ApiImplicitParam(name = "userParam", type = "body", dataTypeClass = UserParam.class, required = true)
    @PostMapping("add")
    public ResponseEntity<UserParam> add(@Valid @RequestBody UserParam userParam) {
        return ResponseEntity.ok(userParam);
    }
}
```

## @ControllerAdvice还可以怎么用？
除了通过@ExceptionHandler注解用于全局异常的处理之外，@ControllerAdvice还有两个用法：
### @InitBinder注解
用于请求中注册自定义参数的解析，从而达到自定义请求参数格式的目的；比如，在@ControllerAdvice注解的类中添加如下方法，来统一处理日期格式的格式化
```java
@InitBinder
public void handleInitBinder(WebDataBinder dataBinder){
    dataBinder.registerCustomEditor(Date.class,
            new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), false));
}

```
Controller中传入参数（string类型）自动转化为Date类型
```java
@GetMapping("testDate")
public Date processApi(Date date) {
    return date;
}
```

### @ModelAttribute注解
用来预设全局参数，比如最典型的使用Spring Security时将添加当前登录的用户信息（UserDetails)作为参数。
```java
@ModelAttribute("currentUser")
public UserDetails modelAttribute() {
    return (UserDetails) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
}
```
所有controller类中requestMapping方法都可以直接获取并使用currentUser

```java
@PostMapping("saveSomething")
public ResponseEntity<String> saveSomeObj(@ModelAttribute("currentUser") UserDetails operator) {
    // 保存操作，并设置当前操作人员的ID（从UserDetails中获得）
    return ResponseEntity.success("ok");
}
```

## @ControllerAdvice是如何起作用的（原理）？
DispatcherServlet中onRefresh方法是初始化ApplicationContext后的回调方法，它会调用initStrategies方法，主要更新一些servlet需要使用的对象，包括国际化处理，requestMapping，视图解析等等。
```java
/**
    * This implementation calls {@link #initStrategies}.
    */
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

/**
    * Initialize the strategy objects that this servlet uses.
    * <p>May be overridden in subclasses in order to initialize further strategy objects.
    */
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context); // 文件上传
    initLocaleResolver(context); // i18n国际化
    initThemeResolver(context); // 主题
    initHandlerMappings(context); // requestMapping
    initHandlerAdapters(context); // adapters
    initHandlerExceptionResolvers(context); // 异常处理
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```
在RequestMappingHandlerAdapter中的afterPropertiesSet去处理advice
```java
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
        // 缓存所有modelAttribute注解方法
        Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
        if (!attrMethods.isEmpty()) {
            this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
        }
        // 缓存所有initBinder注解方法
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
@ExceptionHandler处理@ExceptionHandler显然是在上述initHandlerExceptionResolvers(context)方法中。同样的，从BeanFactory中获取HandlerExceptionResolver
```java
/**
    * Initialize the HandlerExceptionResolver used by this class.
    * <p>If no bean is defined with the given name in the BeanFactory for this namespace,
    * we default to no exception resolver.
    */
private void initHandlerExceptionResolvers(ApplicationContext context) {
    this.handlerExceptionResolvers = null;

    if (this.detectAllHandlerExceptionResolvers) {
        // Find all HandlerExceptionResolvers in the ApplicationContext, including ancestor contexts.
        Map<String, HandlerExceptionResolver> matchingBeans = BeanFactoryUtils
                .beansOfTypeIncludingAncestors(context, HandlerExceptionResolver.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerExceptionResolvers = new ArrayList<>(matchingBeans.values());
            // We keep HandlerExceptionResolvers in sorted order.
            AnnotationAwareOrderComparator.sort(this.handlerExceptionResolvers);
        }
    }
    else {
        try {
            HandlerExceptionResolver her =
                    context.getBean(HANDLER_EXCEPTION_RESOLVER_BEAN_NAME, HandlerExceptionResolver.class);
            this.handlerExceptionResolvers = Collections.singletonList(her);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, no HandlerExceptionResolver is fine too.
        }
    }

    // Ensure we have at least some HandlerExceptionResolvers, by registering
    // default HandlerExceptionResolvers if no other resolvers are found.
    if (this.handlerExceptionResolvers == null) {
        this.handlerExceptionResolvers = getDefaultStrategies(context, HandlerExceptionResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerExceptionResolvers declared in servlet '" + getServletName() +
                    "': using default strategies from DispatcherServlet.properties");
        }
    }
}
```












