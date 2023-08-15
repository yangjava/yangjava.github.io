---
layout: post
categories: [SpringMV]
description: none
keywords: Spring
---
# SpringMVC组件处理映射器
先来看看这九大组件中的第一个 HandlerMapping。HandlerMapping 叫做处理器映射器。

## 处理映射器
SpringMVC 一个大致的初始化流程以及请求的大致处理流程，在请求处理过程中，涉及到九大组件，分别是：
- HandlerMapping
- HandlerAdapter
- HandlerExceptionResolver
- ViewResolver
- RequestToViewNameTranslator
- LocaleResolver
- ThemeResolver
- MultipartResolver
- FlashMapManager

## 概览
HandlerMapping 叫做处理器映射器，它的作用就是根据当前 request 找到对应的 Handler 和 Interceptor，然后封装成一个 HandlerExecutionChain 对象返回，我们来看下 HandlerMapping 接口：
```
public interface HandlerMapping {
 String BEST_MATCHING_HANDLER_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingHandler";
 @Deprecated
 String LOOKUP_PATH = HandlerMapping.class.getName() + ".lookupPath";
 String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE = HandlerMapping.class.getName() + ".pathWithinHandlerMapping";
 String BEST_MATCHING_PATTERN_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingPattern";
 String INTROSPECT_TYPE_LEVEL_MAPPING = HandlerMapping.class.getName() + ".introspectTypeLevelMapping";
 String URI_TEMPLATE_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".uriTemplateVariables";
 String MATRIX_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".matrixVariables";
 String PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE = HandlerMapping.class.getName() + ".producibleMediaTypes";
 default boolean usesPathPatterns() {
  return false;
 }
 @Nullable
 HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```
可以看到，除了一堆声明的常量外，其实就一个需要实现的方法 getHandler，该方法的返回值就是我们所了解到的 HandlerExecutionChain。

仔细观察就两大类：
- AbstractHandlerMethodMapping
AbstractHandlerMethodMapping 体系下的都是根据方法名进行匹配的
- AbstractUrlHandlerMapping
AbstractUrlHandlerMapping 体系下的都是根据 URL 路径进行匹配的

这两者有一个共同的父类 AbstractHandlerMapping，其他的都是一些辅助接口。

## AbstractHandlerMapping
AbstractHandlerMapping 实现了 HandlerMapping 接口，无论是通过 URL 进行匹配还是通过方法名进行匹配，都是通过继承 AbstractHandlerMapping 来实现的，所以 AbstractHandlerMapping 所做的事情其实就是一些公共的事情，将以一些需要具体处理的事情则交给子类去处理，这其实就是典型的模版方法模式。

AbstractHandlerMapping 间接继承自 ApplicationObjectSupport，并重写了 initApplicationContext 方法（其实该方法也是一个模版方法），这也是 AbstractHandlerMapping 的初始化入口方法，我们一起来看下：
```
@Override
protected void initApplicationContext() throws BeansException {
 extendInterceptors(this.interceptors);
 detectMappedInterceptors(this.adaptedInterceptors);
 initInterceptors();
}
```
三个方法都和拦截器有关。

extendInterceptors
```
protected void extendInterceptors(List<Object> interceptors) {
}
```
extendInterceptors 是一个模版方法，可以在子类中实现，子类实现了该方法之后，可以对拦截器进行添加、删除或者修改，不过在 SpringMVC 的具体实现中，其实这个方法并没有在子类中进行实现。

detectMappedInterceptors
```
protected void detectMappedInterceptors(List<HandlerInterceptor> mappedInterceptors) {
 mappedInterceptors.addAll(BeanFactoryUtils.beansOfTypeIncludingAncestors(
   obtainApplicationContext(), MappedInterceptor.class, true, false).values());
}
```
detectMappedInterceptors 方法会从 SpringMVC 容器以及 Spring 容器中查找所有 MappedInterceptor 类型的 Bean，查找到之后添加到 mappedInterceptors 属性中（其实就是全局的 adaptedInterceptors 属性）。一般来说，我们定义好一个拦截器之后，还要在 XML 文件中配置该拦截器，拦截器以及各种配置信息，最终就会被封装成一个 MappedInterceptor 对象。

initInterceptors
```
protected void initInterceptors() {
 if (!this.interceptors.isEmpty()) {
  for (int i = 0; i < this.interceptors.size(); i++) {
   Object interceptor = this.interceptors.get(i);
   if (interceptor == null) {
    throw new IllegalArgumentException("Entry number " + i + " in interceptors array is null");
   }
   this.adaptedInterceptors.add(adaptInterceptor(interceptor));
  }
 }
}
```
initInterceptors 方法主要是进行拦截器的初始化操作，具体内容是将 interceptors 集合中的拦截器添加到 adaptedInterceptors 集合中。

至此，我们看到，所有拦截器最终都会被存入 adaptedInterceptors 变量中。

AbstractHandlerMapping 的初始化其实也就是拦截器的初始化过程。

为什么 AbstractHandlerMapping 中对拦截器如此重视呢？

其实不是重视，大家想想，AbstractUrlHandlerMapping 和 AbstractHandlerMethodMapping 最大的区别在于查找处理器的区别，一旦处理器找到了，再去找拦截器，但是拦截器都是统一的，并没有什么明显区别，所以拦截器就统一在 AbstractHandlerMapping 中进行处理，而不会去 AbstractUrlHandlerMapping 或者 AbstractHandlerMethodMapping 中处理。


接下来我们再来看看 AbstractHandlerMapping#getHandler 方法，看看处理器是如何获取到的：
```
@Override
@Nullable
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
 Object handler = getHandlerInternal(request);
 if (handler == null) {
  handler = getDefaultHandler();
 }
 if (handler == null) {
  return null;
 }
 // Bean name or resolved handler?
 if (handler instanceof String) {
  String handlerName = (String) handler;
  handler = obtainApplicationContext().getBean(handlerName);
 }
 // Ensure presence of cached lookupPath for interceptors and others
 if (!ServletRequestPathUtils.hasCachedPath(request)) {
  initLookupPath(request);
 }
 HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
 if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
  CorsConfiguration config = getCorsConfiguration(handler, request);
  if (getCorsConfigurationSource() != null) {
   CorsConfiguration globalConfig = getCorsConfigurationSource().getCorsConfiguration(request);
   config = (globalConfig != null ? globalConfig.combine(config) : config);
  }
  if (config != null) {
   config.validateAllowCredentials();
  }
  executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
 }
 return executionChain;
}
```
这个方法的执行流程是这样的：
- 首先调用 getHandlerInternal 方法去尝试获取处理器，getHandlerInternal 方法也是一个模版方法，该方法将在子类中实现。
- 如果没找到相应的处理器，则调用 getDefaultHandler 方法获取默认的处理器，我们在配置 HandlerMapping 的时候可以配置默认的处理器。
- 如果找到的处理器是一个字符串，则根据该字符串找去 SpringMVC 容器中找到对应的 Bean。
- 确保 lookupPath 存在，一会找对应的拦截器的时候会用到。
- 找到 handler 之后，接下来再调用 getHandlerExecutionChain 方法获取 HandlerExecutionChain 对象。
- 接下来 if 里边的是进行跨域处理的，获取到跨域的相关配置，然后进行验证&配置，检查是否允许跨域。跨域这块的配置以及校验还是蛮有意思的，松哥以后专门写文章来和小伙伴们细聊。

接下来我们再来看看第五步的 getHandlerExecutionChain 方法的执行逻辑，正是在这个方法里边把 handler 变成了 HandlerExecutionChain：
```
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
 HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
   (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
 for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
  if (interceptor instanceof MappedInterceptor) {
   MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
   if (mappedInterceptor.matches(request)) {
    chain.addInterceptor(mappedInterceptor.getInterceptor());
   }
  }
  else {
   chain.addInterceptor(interceptor);
  }
 }
 return chain;
}
```
这里直接根据已有的 handler 创建一个新的 HandlerExecutionChain 对象，然后遍历 adaptedInterceptors 集合，该集合里存放的都是拦截器，如果拦截器的类型是 MappedInterceptor，则调用 matches 方法去匹配一下，看一下是否是拦截当前请求的拦截器，如果是，则调用 chain.addInterceptor 方法加入到 HandlerExecutionChain 对象中；如果就是一个普通拦截器，则直接加入到 HandlerExecutionChain 对象中。

这就是 AbstractHandlerMapping#getHandler 方法的大致逻辑，可以看到，这里留了一个模版方法 getHandlerInternal 在子类中实现，接下来我们就来看看它的子类。

## AbstractUrlHandlerMapping
AbstractUrlHandlerMapping，看名字就知道，都是按照 URL 地址来进行匹配的，它的原理就是将 URL 地址与对应的 Handler 保存在同一个 Map 中，当调用 getHandlerInternal 方法时，就根据请求的 URL 去 Map 中找到对应的 Handler 返回就行了。

这里我们就先从他的 getHandlerInternal 方法开始看起：
```
@Override
@Nullable
protected Object getHandlerInternal(HttpServletRequest request) throws Exception {
 String lookupPath = initLookupPath(request);
 Object handler;
 if (usesPathPatterns()) {
  RequestPath path = ServletRequestPathUtils.getParsedRequestPath(request);
  handler = lookupHandler(path, lookupPath, request);
 }
 else {
  handler = lookupHandler(lookupPath, request);
 }
 if (handler == null) {
  // We need to care for the default handler directly, since we need to
  // expose the PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE for it as well.
  Object rawHandler = null;
  if (StringUtils.matchesCharacter(lookupPath, '/')) {
   rawHandler = getRootHandler();
  }
  if (rawHandler == null) {
   rawHandler = getDefaultHandler();
  }
  if (rawHandler != null) {
   // Bean name or resolved handler?
   if (rawHandler instanceof String) {
    String handlerName = (String) rawHandler;
    rawHandler = obtainApplicationContext().getBean(handlerName);
   }
   validateHandler(rawHandler, request);
   handler = buildPathExposingHandler(rawHandler, lookupPath, lookupPath, null);
  }
 }
 return handler;
}

```
- 首先找到 lookupPath，就是请求的路径。这个方法本身松哥就不多说了，之前在Spring5 里边的新玩法！这种 URL 请求让我涨见识了！一文中有过介绍。
- 接下来就是调用 lookupHandler 方法获取 Handler 对象，lookupHandler 有一个重载方法，具体用哪个，主要看所使用的 URL 匹配模式，如果使用了最新的 PathPattern（Spring5 之后的），则使用三个参数的 lookupHandler；如果还是使用之前旧的 AntPathMatcher，则这里使用两个参数的 lookupHandler。
- 如果前面没有获取到 handler 实例，则接下来再做各种尝试，去分别查找 RootHandler、DefaultHandler 等，如果找到的 Handler 是一个 String，则去 Spring 容器中查找该 String 对应的 Bean，再调用 validateHandler 方法来校验找到的 handler 和 request 是否匹配，不过这是一个空方法，子类也没有实现，所以可以忽略之。最后再通过 buildPathExposingHandler 方法给找到的 handler 添加一些参数。

这就是整个 getHandlerInternal 方法的逻辑，实际上并不难，里边主要涉及到 lookupHandler 和 buildPathExposingHandler 两个方法，需要和大家详细介绍下，我们分别来看。

### lookupHandler
lookupHandler 有两个，我们分别来看。
```
@Nullable
protected Object lookupHandler(String lookupPath, HttpServletRequest request) throws Exception {
 Object handler = getDirectMatch(lookupPath, request);
 if (handler != null) {
  return handler;
 }
 // Pattern match?
 List<String> matchingPatterns = new ArrayList<>();
 for (String registeredPattern : this.handlerMap.keySet()) {
  if (getPathMatcher().match(registeredPattern, lookupPath)) {
   matchingPatterns.add(registeredPattern);
  }
  else if (useTrailingSlashMatch()) {
   if (!registeredPattern.endsWith("/") && getPathMatcher().match(registeredPattern + "/", lookupPath)) {
    matchingPatterns.add(registeredPattern + "/");
   }
  }
 }
 String bestMatch = null;
 Comparator<String> patternComparator = getPathMatcher().getPatternComparator(lookupPath);
 if (!matchingPatterns.isEmpty()) {
  matchingPatterns.sort(patternComparator);
  bestMatch = matchingPatterns.get(0);
 }
 if (bestMatch != null) {
  handler = this.handlerMap.get(bestMatch);
  if (handler == null) {
   if (bestMatch.endsWith("/")) {
    handler = this.handlerMap.get(bestMatch.substring(0, bestMatch.length() - 1));
   }
   if (handler == null) {
    throw new IllegalStateException(
      "Could not find handler for best pattern match [" + bestMatch + "]");
   }
  }
  // Bean name or resolved handler?
  if (handler instanceof String) {
   String handlerName = (String) handler;
   handler = obtainApplicationContext().getBean(handlerName);
  }
  validateHandler(handler, request);
  String pathWithinMapping = getPathMatcher().extractPathWithinPattern(bestMatch, lookupPath);
  // There might be multiple 'best patterns', let's make sure we have the correct URI template variables
  // for all of them
  Map<String, String> uriTemplateVariables = new LinkedHashMap<>();
  for (String matchingPattern : matchingPatterns) {
   if (patternComparator.compare(bestMatch, matchingPattern) == 0) {
    Map<String, String> vars = getPathMatcher().extractUriTemplateVariables(matchingPattern, lookupPath);
    Map<String, String> decodedVars = getUrlPathHelper().decodePathVariables(request, vars);
    uriTemplateVariables.putAll(decodedVars);
   }
  }
  return buildPathExposingHandler(handler, bestMatch, pathWithinMapping, uriTemplateVariables);
 }
 // No handler found...
 return null;
}
@Nullable
private Object getDirectMatch(String urlPath, HttpServletRequest request) throws Exception {
 Object handler = this.handlerMap.get(urlPath);
 if (handler != null) {
  // Bean name or resolved handler?
  if (handler instanceof String) {
   String handlerName = (String) handler;
   handler = obtainApplicationContext().getBean(handlerName);
  }
  validateHandler(handler, request);
  return buildPathExposingHandler(handler, urlPath, urlPath, null);
 }
 return null;
}
```
这里首先调用 getDirectMatch 方法直接去 handlerMap 中找对应的处理器，handlerMap 中就保存了请求 URL 和处理器的映射关系，具体的查找过程就是先去 handlerMap 中找，找到了，如果是 String，则去 Spring 容器中找对应的 Bean，然后调用 validateHandler 方法去验证（实际上没有验证，前面已经说了），最后调用 buildPathExposingHandler 方法添加拦截器。
如果 getDirectMatch 方法返回值不为 null，则直接将查找到的 handler 返回，方法到此为止。那么什么情况下 getDirectMatch 方法的返回值不为 null 呢？简单来收就是没有使用通配符的情况下，请求地址中没有通配符，一个请求地址对应一个处理器，只有这种情况，getDirectMatch 方法返回值才不为 null，因为 handlerMap 中保存的是代码的定义，比如我们定义代码的时候，某个处理器的访问路径可能带有通配符，但是当我们真正发起请求的时候，请求路径里是没有通配符的，这个时候再去 handlerMap 中就找不对对应的处理器了。如果用到了定义接口时用到了通配符，则需要在下面的代码中继续处理。
接下来处理通配符的情况。首先定义 matchingPatterns 集合，将当前请求路径和 handlerMap 集合中保存的请求路径规则进行对比，凡是能匹配上的规则都直接存入 matchingPatterns 集合中。具体处理中，还有一个 useTrailingSlashMatch 的可能，有的小伙伴 SpringMVC 用的不熟练，看到这里可能就懵了，这里是这样的，SpringMVC 中，默认是可以匹配结尾 / 的，举个简单例子，如果你定义的接口是 /user，那么请求路径可以是 /user 也可以 /user/，这两种默认都是支持的，所以这里的 useTrailingSlashMatch 分支主要是处理后面这种情况，处理方式很简单，就在 registeredPattern 后面加上 / 然后继续和请求路径进行匹配。
由于一个请求 URL 可能会和定义的多个接口匹配上，所以 matchingPatterns 变量是一个数组，接下来就要对 matchingPatterns 进行排序，排序完成后，选择排序后的第一项作为最佳选项赋值给 bestMatch 变量。默认的排序规则是 AntPatternComparator，当然开发者也可以自定义。

AntPatternComparator 中定义的优先级如下：

| 路由配置	                        | 优先级   |
|------------------------------|:------|
| 不含任何特殊符号的路径，如：配置路由/a/b/c     | 第一优先级 |
| 带有{}的路径，如：/a/{b}/c           | 第二优先级 |
| 带有正则的路径，如：/a/{regex:\d{3}}/c | 第三优先级 |
| 带有*的路径，如：/a/b/*              | 第四优先级 |
| 带有**的路径，如：/a/b/**            | 第五优先级 |
| 最模糊的匹配：/**                   | 最低优先级 |

找到 bestMatch 之后，接下来再根据 bestMatch 去 handlerMap 中找到对应的处理器，直接找如果没找到，就去检查 bestMatch 是否以 / 结尾，如果是以 / 结尾，则去掉结尾的 / 再去 handlerMap 中查找，如果还没找到，那就该抛异常出来了。如果找到的 handler 是 String 类型的，则再去 Spring 容器中查找对应的 Bean，接下来再调用 validateHandler 方法进行验证。
接下来调用 extractPathWithinPattern 方法提取出映射路径，例如定义的接口规则是 myroot/*.html，请求路径是 myroot/myfile.html，那么最终获取到的就是 myfile.html。
接下来的 for 循环是为了处理存在多个最佳匹配规则的情况，在第四步中，我们对 matchingPatterns 进行排序，排序完成后，选择第一项作为最佳选项赋值给 bestMatch，但是最佳选项可能会有多个，这里就是处理最佳选项有多个的情况。
最后调用 buildPathExposingHandler 方法注册两个内部拦截器，该方法下文我会给大家详细介绍。	

lookupHandler 还有一个重载方法，不过只要大家把这个方法的执行流程搞清楚了，重载方法其实很好理解，这里松哥就不再赘述了，唯一要说的就是重载方法用了 PathPattern 去匹配 URL 路径，而这个方法用了 AntPathMatcher 去匹配 URL 路径。

### buildPathExposingHandler 
```
protected Object buildPathExposingHandler(Object rawHandler, String bestMatchingPattern,
  String pathWithinMapping, @Nullable Map<String, String> uriTemplateVariables) {
 HandlerExecutionChain chain = new HandlerExecutionChain(rawHandler);
 chain.addInterceptor(new PathExposingHandlerInterceptor(bestMatchingPattern, pathWithinMapping));
 if (!CollectionUtils.isEmpty(uriTemplateVariables)) {
  chain.addInterceptor(new UriTemplateVariablesHandlerInterceptor(uriTemplateVariables));
 }
 return chain;
}
```
buildPathExposingHandler 方法向 HandlerExecutionChain 中添加了两个拦截器 PathExposingHandlerInterceptor 和 UriTemplateVariablesHandlerInterceptor，这两个拦截器在各自的 preHandle 中分别向 request 对象添加了一些属性，具体添加的属性小伙伴们可以自行查看，这个比较简单，我就不多说了。

在前面的方法中，涉及到一个重要的变量 handlerMap，我们定义的接口和处理器之间的关系都保存在这个变量中，那么这个变量是怎么初始化的呢？这就涉及到 AbstractUrlHandlerMapping 中的另一个方法 registerHandler：
```
protected void registerHandler(String[] urlPaths, String beanName) throws BeansException, IllegalStateException {
 for (String urlPath : urlPaths) {
  registerHandler(urlPath, beanName);
 }
}
protected void registerHandler(String urlPath, Object handler) throws BeansException, IllegalStateException {
 Object resolvedHandler = handler;
 if (!this.lazyInitHandlers && handler instanceof String) {
  String handlerName = (String) handler;
  ApplicationContext applicationContext = obtainApplicationContext();
  if (applicationContext.isSingleton(handlerName)) {
   resolvedHandler = applicationContext.getBean(handlerName);
  }
 }
 Object mappedHandler = this.handlerMap.get(urlPath);
 if (mappedHandler != null) {
  if (mappedHandler != resolvedHandler) {
   throw new IllegalStateException(
     "Cannot map " + getHandlerDescription(handler) + " to URL path [" + urlPath +
     "]: There is already " + getHandlerDescription(mappedHandler) + " mapped.");
  }
 }
 else {
  if (urlPath.equals("/")) {
   setRootHandler(resolvedHandler);
  }
  else if (urlPath.equals("/*")) {
   setDefaultHandler(resolvedHandler);
  }
  else {
   this.handlerMap.put(urlPath, resolvedHandler);
   if (getPatternParser() != null) {
    this.pathPatternHandlerMap.put(getPatternParser().parse(urlPath), resolvedHandler);
   }
  }
 }
}
```
registerHandler(String[],String) 方法有两个参数，第一个就是定义的请求路径，第二个参数则是处理器 Bean 的名字，第一个参数是一个数组，那是因为同一个处理器可以对应多个不同的请求路径。

在重载方法 registerHandler(String,String) 里边，完成了 handlerMap 的初始化，具体流程如下：

如果没有设置 lazyInitHandlers，并且 handler 是 String 类型，那么就去 Spring 容器中找到对应的 Bean 赋值给 resolvedHandler。
根据 urlPath 去 handlerMap 中查看是否已经有对应的处理器了，如果有的话，则抛出异常，一个 URL 地址只能对应一个处理器，这个很好理解。
接下来根据 URL 路径，将处理器进行配置，最终添加到 handlerMap 变量中。
这就是 AbstractUrlHandlerMapping 的主要工作，其中 registerHandler 将在它的子类中调用。

接下来我们来看 AbstractUrlHandlerMapping 的子类。

## SimpleUrlHandlerMapping
为了方便处理，SimpleUrlHandlerMapping 中自己定义了一个 urlMap 变量，这样可以在注册之前做一些预处理，例如确保所有的 URL 都是以 / 开始。SimpleUrlHandlerMapping 在定义时重写了父类的 initApplicationContext 方法，并在该方法中调用了 registerHandlers，在 registerHandlers 中又调用了父类的 registerHandler 方法完成了 handlerMap 的初始化操作：
```
@Override
public void initApplicationContext() throws BeansException {
 super.initApplicationContext();
 registerHandlers(this.urlMap);
}
protected void registerHandlers(Map<String, Object> urlMap) throws BeansException {
 if (urlMap.isEmpty()) {
  logger.trace("No patterns in " + formatMappingName());
 }
 else {
  urlMap.forEach((url, handler) -> {
   // Prepend with slash if not already present.
   if (!url.startsWith("/")) {
    url = "/" + url;
   }
   // Remove whitespace from handler bean name.
   if (handler instanceof String) {
    handler = ((String) handler).trim();
   }
   registerHandler(url, handler);
  });
 }
}
```
这块代码很简单，实在没啥好说的，如果 URL 不是以 / 开头，则手动给它加上 / 即可。有小伙伴们可能要问了，urlMap 的值从哪里来？当然是从我们的配置文件里边来呀，像下面这样：
```
<bean class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="urlMap">
        <map>
            <entry key="/aaa" value-ref="/hello"/>
        </map>
    </property>
</bean>
```

## AbstractDetectingUrlHandlerMapping

AbstractDetectingUrlHandlerMapping 也是 AbstractUrlHandlerMapping 的子类，但是它和 SimpleUrlHandlerMapping 有一些不一样的地方。

不一样的是哪里呢？

AbstractDetectingUrlHandlerMapping 会自动查找到 SpringMVC 容器以及 Spring 容器中的所有 beanName，然后根据 beanName 解析出对应的 URL 地址，再将解析出的 url 地址和对应的 beanName 注册到父类的 handlerMap 变量中。换句话说，如果你用了 AbstractDetectingUrlHandlerMapping，就不用像 SimpleUrlHandlerMapping 那样去挨个配置 URL 地址和处理器的映射关系了。我们来看下 AbstractDetectingUrlHandlerMapping#initApplicationContext 方法：
```
@Override
public void initApplicationContext() throws ApplicationContextException {
 super.initApplicationContext();
 detectHandlers();
}
protected void detectHandlers() throws BeansException {
 ApplicationContext applicationContext = obtainApplicationContext();
 String[] beanNames = (this.detectHandlersInAncestorContexts ?
   BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class) :
   applicationContext.getBeanNamesForType(Object.class));
 for (String beanName : beanNames) {
  String[] urls = determineUrlsForHandler(beanName);
  if (!ObjectUtils.isEmpty(urls)) {
   registerHandler(urls, beanName);
  }
 }
}
```
AbstractDetectingUrlHandlerMapping 重写了父类的 initApplicationContext 方法，并在该方法中调用了 detectHandlers 方法，在 detectHandlers 中，首先查找到所有的 beanName，然后调用 determineUrlsForHandler 方法分析出 beanName 对应的 URL，不过这里的 determineUrlsForHandler 方法是一个空方法，具体的实现在它的子类中，AbstractDetectingUrlHandlerMapping 只有一个子类 BeanNameUrlHandlerMapping，我们一起来看下：
```
public class BeanNameUrlHandlerMapping extends AbstractDetectingUrlHandlerMapping {
 @Override
 protected String[] determineUrlsForHandler(String beanName) {
  List<String> urls = new ArrayList<>();
  if (beanName.startsWith("/")) {
   urls.add(beanName);
  }
  String[] aliases = obtainApplicationContext().getAliases(beanName);
  for (String alias : aliases) {
   if (alias.startsWith("/")) {
    urls.add(alias);
   }
  }
  return StringUtils.toStringArray(urls);
 }

}
```
这个类很简单，里边就一个 determineUrlsForHandler 方法，这个方法的执行逻辑也很简单，就判断 beanName 是不是以 / 开始，如果是，则将之作为 URL。

如果我们想要在项目中使用 BeanNameUrlHandlerMapping，配置方式如下：
```
<bean class="org.javaboy.init.HelloController" name="/hello"/>
<bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping" id="handlerMapping">
</bean>
```
注意，Controller 的 name 必须是以 / 开始，否则该 bean 不会被自动作为处理器。

至此，AbstractUrlHandlerMapping 体系下的东西就和大家分享完了。

## AbstractHandlerMethodMapping
AbstractHandlerMethodMapping 体系下只有三个类，分别是 AbstractHandlerMethodMapping、RequestMappingInfoHandlerMapping 以及 RequestMappingHandlerMapping，如下图：

AbstractUrlHandlerMapping 体系下，一个 Handler 一般就是一个类，但是在 AbstractHandlerMethodMapping 体系下，一个 Handler 就是一个 Mehtod，这也是我们目前使用 SpringMVC 时最常见的用法，即直接用 @RequestMapping 去标记一个方法，该方法就是一个 Handler。

接下来我们就一起来看看 AbstractHandlerMethodMapping。

### 初始化流程
AbstractHandlerMethodMapping 类实现了 InitializingBean 接口，所以 Spring 容器会自动调用其 afterPropertiesSet 方法，在这里将完成初始化操作：

```
@Override
public void afterPropertiesSet() {
 initHandlerMethods();
}
protected void initHandlerMethods() {
 for (String beanName : getCandidateBeanNames()) {
  if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
   processCandidateBean(beanName);
  }
 }
 handlerMethodsInitialized(getHandlerMethods());
}
protected String[] getCandidateBeanNames() {
 return (this.detectHandlerMethodsInAncestorContexts ?
   BeanFactoryUtils.beanNamesForTypeIncludingAncestors(obtainApplicationContext(), Object.class) :
   obtainApplicationContext().getBeanNamesForType(Object.class));
}
protected void processCandidateBean(String beanName) {
 Class<?> beanType = null;
 try {
  beanType = obtainApplicationContext().getType(beanName);
 }
 catch (Throwable ex) {
 }
 if (beanType != null && isHandler(beanType)) {
  detectHandlerMethods(beanName);
 }
}
```
可以看到，具体的初始化又是在 initHandlerMethods 方法中完成的，在该方法中，首先调用 getCandidateBeanNames 方法获取容器中所有的 beanName，然后调用 processCandidateBean 方法对这些候选的 beanName 进行处理，具体的处理思路就是根据 beanName 找到 beanType，然后调用 isHandler 方法判断该 beanType 是不是一个 Handler，isHandler 是一个空方法，在它的子类 RequestMappingHandlerMapping 中被实现了，该方法主要是检查该 beanType 上有没有 @Controller 或者 @RequestMapping 注解，如果有，说明这就是我们想要的 handler，接下来再调用 detectHandlerMethods 方法保存 URL 和 handler 的映射关系：
```
protected void detectHandlerMethods(Object handler) {
 Class<?> handlerType = (handler instanceof String ?
   obtainApplicationContext().getType((String) handler) : handler.getClass());
 if (handlerType != null) {
  Class<?> userType = ClassUtils.getUserClass(handlerType);
  Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
    (MethodIntrospector.MetadataLookup<T>) method -> {
     try {
      return getMappingForMethod(method, userType);
     }
     catch (Throwable ex) {
      throw new IllegalStateException("Invalid mapping on handler class [" +
        userType.getName() + "]: " + method, ex);
     }
    });
  methods.forEach((method, mapping) -> {
   Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
   registerHandlerMethod(handler, invocableMethod, mapping);
  });
 }
}
```
首先找到 handler 的类型 handlerType。
调用 ClassUtils.getUserClass 方法检查是否是 cglib 代理的子对象类型，如果是，则返回父类型，否则将参数直接返回。
接下来调用 MethodIntrospector.selectMethods 方法获取当前 bean 中所有符合要求的 method。
遍历 methods，调用 registerHandlerMethod 方法完成注册。

上面这段代码里又涉及到两个方法：

getMappingForMethod
registerHandlerMethod

我们分别来看：

### getMappingForMethod

getMappingForMethod 是一个模版方法，具体的实现也是在子类 RequestMappingHandlerMapping 里边：
```
@Override
@Nullable
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
 RequestMappingInfo info = createRequestMappingInfo(method);
 if (info != null) {
  RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
  if (typeInfo != null) {
   info = typeInfo.combine(info);
  }
  String prefix = getPathPrefix(handlerType);
  if (prefix != null) {
   info = RequestMappingInfo.paths(prefix).options(this.config).build().combine(info);
  }
 }
 return info;
}
```
首先根据 method 对象，调用 createRequestMappingInfo 方法获取一个 RequestMappingInfo，一个 RequestMappingInfo 包含了一个接口定义的详细信息，例如参数、header、produces、consumes、请求方法等等信息都在这里边。接下来再根据 handlerType 也获取一个 RequestMappingInfo，并调用 combine 方法将两个 RequestMappingInfo 进行合并。接下来调用 getPathPrefix 方法查看 handlerType 上有没有 URL 前缀，如果有，就添加到 info 里边去，最后将 info 返回。

这里要说一下 handlerType 里边的这个前缀是那里来的，我们可以在 Controller 上使用 @RequestMapping 注解，配置一个路径前缀，这样 Controller 中的所有方法都加上了该路径前缀，但是这种方式需要一个一个的配置，如果想一次性配置所有的 Controller 呢？我们可以使用 Spring5.1 中新引入的方法 addPathPrefix 来配置，如下：

```
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setPatternParser(new PathPatternParser()).addPathPrefix("/demo", HandlerTypePredicate.forAnnotation(RestController.class));
    }
}
```
上面这个配置表示，所有的 @RestController 标记的类都自动加上 itboyhub 前缀。有了这个配置之后，上面的 getPathPrefix 方法获取到的就是 /demo 了。

### registerHandlerMethod
当找齐了 URL 和 handlerMethod 之后，接下来就是将这些信息保存下来，方式如下：
```
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
 this.mappingRegistry.register(mapping, handler, method);
}
public void register(T mapping, Object handler, Method method) {
 this.readWriteLock.writeLock().lock();
 try {
  HandlerMethod handlerMethod = createHandlerMethod(handler, method);
  validateMethodMapping(handlerMethod, mapping);
  Set<String> directPaths = AbstractHandlerMethodMapping.this.getDirectPaths(mapping);
  for (String path : directPaths) {
   this.pathLookup.add(path, mapping);
  }
  String name = null;
  if (getNamingStrategy() != null) {
   name = getNamingStrategy().getName(handlerMethod, mapping);
   addMappingName(name, handlerMethod);
  }
  CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
  if (corsConfig != null) {
   corsConfig.validateAllowCredentials();
   this.corsLookup.put(handlerMethod, corsConfig);
  }
  this.registry.put(mapping,
    new MappingRegistration<>(mapping, handlerMethod, directPaths, name, corsConfig != null));
 }
 finally {
  this.readWriteLock.writeLock().unlock();
 }
}
```
首先调用 createHandlerMethod 方法创建 HandlerMethod 对象。
调用 validateMethodMapping 方法对 handlerMethod 进行验证，主要是验证 handlerMethod 是否已经存在。
从 mappings 中提取出 directPaths，就是不包含通配符的请求路径，然后将请求路径和 mapping 的映射关系保存到 pathLookup 中。
找到所有 handler 的简称，调用 addMappingName 方法添加到 nameLookup 中。例如我们在 HelloController 中定义了一个名为 hello 的请求接口，那么这里拿到的就是 HC#hello，HC 是 HelloController 中的大写字母。
初始化跨域配置，并添加到 corsLookup 中。
将构建好的关系添加到 registry 中。

多说一句，第四步这个东西有啥用呢？这个其实是 Spring4 中开始增加的功能，算是一个小彩蛋吧，虽然日常开发很少用，但是我这里还是和大家说一下。

假如你有如下一个接口：
```java
@RestController
@RequestMapping("/javaboy")
public class HelloController {
    @GetMapping("/aaa")
    public String hello99() {
        return "aaa";
    }
}
```
当你请求该接口的时候，不想通过路径，想直接通过方法名，行不行呢？当然可以！

在 jsp 文件中，添加如下超链接：
```
<%@ taglib prefix="s" uri="http://www.springframework.org/tags" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
<a href="${s:mvcUrl('HC#hello99').build()}">Go!</a>
</body>
</html>
```
当这个 jsp 页面渲染完成后，href 属性就自动成了 hello99 方法的请求路径了。这个功能的实现，就依赖于前面第四步的内容。

至此，我们就把 AbstractHandlerMethodMapping 的初始化流程看完了。

## 请求处理
接下来我们来看下当请求到来后，AbstractHandlerMethodMapping 会如何处理。

和前面第三小节一样，这里处理请求的入口方法也是 getHandlerInternal，如下：
```
@Override
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
 String lookupPath = initLookupPath(request);
 this.mappingRegistry.acquireReadLock();
 try {
  HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
  return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
 }
 finally {
  this.mappingRegistry.releaseReadLock();
 }
}
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
 List<Match> matches = new ArrayList<>();
 List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
 if (directPathMatches != null) {
  addMatchingMappings(directPathMatches, matches, request);
 }
 if (matches.isEmpty()) {
  addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request);
 }
 if (!matches.isEmpty()) {
  Match bestMatch = matches.get(0);
  if (matches.size() > 1) {
   Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
   matches.sort(comparator);
   bestMatch = matches.get(0);
   if (CorsUtils.isPreFlightRequest(request)) {
    for (Match match : matches) {
     if (match.hasCorsConfig()) {
      return PREFLIGHT_AMBIGUOUS_MATCH;
     }
    }
   }
   else {
    Match secondBestMatch = matches.get(1);
    if (comparator.compare(bestMatch, secondBestMatch) == 0) {
     Method m1 = bestMatch.getHandlerMethod().getMethod();
     Method m2 = secondBestMatch.getHandlerMethod().getMethod();
     String uri = request.getRequestURI();
     throw new IllegalStateException(
       "Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
    }
   }
  }
  request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.getHandlerMethod());
  handleMatch(bestMatch.mapping, lookupPath, request);
  return bestMatch.getHandlerMethod();
 }
 else {
  return handleNoMatch(this.mappingRegistry.getRegistrations().keySet(), lookupPath, request);
 }
}
```
这里就比较容易，通过 lookupHandlerMethod 找到对应的 HandlerMethod 返回即可，如果 lookupHandlerMethod 方法返回值不为 null，则通过 createWithResolvedBean 创建 HandlerMethod（主要是确认里边的 Bean 等）。lookupHandlerMethod 方法也比较容易：

首先根据 lookupPath 找到匹配条件 directPathMatches，然后将获取到的匹配条件添加到 matches 中（不包含通配符的请求走这里）。
如果 matches 为空，说明根据 lookupPath 没有找到匹配条件，那么直接将所有匹配条件加入 matches 中（包含通配符的请求走这里）。
对 matches 进行排序，并选择排序后的第一个为最佳匹配项，如果前两个排序相同，则抛出异常。
大致的流程就是这样，具体到请求并没有涉及到它的子类。




















