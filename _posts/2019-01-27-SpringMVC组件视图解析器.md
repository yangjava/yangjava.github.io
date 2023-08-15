---
layout: post
categories: [SpringMV]
description: none
keywords: Spring
---
# SpringMVC组件视图解析器
ViewResolver 其实就是我们心心念念的视图解析器，用过 SpringMVC 的小伙伴都知道 SpringMVC 中有一个视图解析器

## 概览
首先我们来大概看一下 ViewResolver 接口是什么样子的：
```
public interface ViewResolver {
 @Nullable
 View resolveViewName(String viewName, Locale locale) throws Exception;
}
```
这个接口中只有一个方法，可以看到，非常简单，就是通过视图名和 Locale，找到对应的 View 返回即可。

继承自 ViewResolver 接口的类有四个，作用如下：
- ContentNegotiatingViewResolver：支持 MediaType 和后缀的视图解析器。
- BeanNameViewResolver：这个是直接根据视图名去 Spring 容器中查找相应的 Bean 并返回。
- AbstractCachingViewResolver：具有缓存功能的视图解析器。
- ViewResolverComposite：这是一个组合的视图解析器，届时可以用来代理其他具体干活的视图解析器。

接下来我们就对这四个视图解析器逐一进行介绍，先从最简单的 BeanNameViewResolver 开始吧。

## BeanNameViewResolver
BeanNameViewResolver 的处理方式非常简单粗暴，直接根据 viewName 去 Spring 容器中查找相应的 Bean 并返回，如下：
```
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws BeansException {
 ApplicationContext context = obtainApplicationContext();
 if (!context.containsBean(viewName)) {
  return null;
 }
 if (!context.isTypeMatch(viewName, View.class)) {
  return null;
 }
 return context.getBean(viewName, View.class);
}
```
先去判断下有没有相应的 Bean，然后再检查下 Bean 的类型对不对，都没问题，直接查找返回即可。

## ContentNegotiatingViewResolver
ContentNegotiatingViewResolver 其实是目前广泛使用的一个视图解析器，主要是添加了对 MediaType 的支持。ContentNegotiatingViewResolver 这个是 Spring3.0 中引入的的视图解析器，它不负责具体的视图解析，而是根据当前请求的 MIME 类型，从上下文中选择一个合适的视图解析器，并将请求工作委托给它。

这里我们就先来看看 ContentNegotiatingViewResolver#resolveViewName 方法：
```
public View resolveViewName(String viewName, Locale locale) throws Exception {
 RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
 List<MediaType> requestedMediaTypes = getMediaTypes(((ServletRequestAttributes) attrs).getRequest());
 if (requestedMediaTypes != null) {
  List<View> candidateViews = getCandidateViews(viewName, locale, requestedMediaTypes);
  View bestView = getBestView(candidateViews, requestedMediaTypes, attrs);
  if (bestView != null) {
   return bestView;
  }
 }
 if (this.useNotAcceptableStatusCode) {
  return NOT_ACCEPTABLE_VIEW;
 }
 else {
  return null;
 }
}
```
这里的代码逻辑也比较简单：

- 首先是获取到当前的请求对象，可以直接从 RequestContextHolder 中获取。然后从当前请求对象中提取出 MediaType。
- 如果 MediaType 不为 null，则根据 MediaType，找到合适的视图解析器，并将解析出来的 View 返回。
- 如果 MediaType 为 null，则为两种情况，如果 useNotAcceptableStatusCode 为 true，则返回 NOT_ACCEPTABLE_VIEW 视图，这个视图其实是一个 406 响应，表示客户端错误，服务器端无法提供与 Accept-Charset 以及 Accept-Language 消息头指定的值相匹配的响应；如果 useNotAcceptableStatusCode 为 false，则返回 null。
- 现在问题的核心其实就变成 getCandidateViews 方法和 getBestView 方法了，看名字就知道，前者是获取所有的候选 View，后者则是从这些候选 View 中选择一个最佳的 View，我们一个一个来看。

先来看 getCandidateViews：
```
private List<View> getCandidateViews(String viewName, Locale locale, List<MediaType> requestedMediaTypes)
  throws Exception {
 List<View> candidateViews = new ArrayList<>();
 if (this.viewResolvers != null) {
  for (ViewResolver viewResolver : this.viewResolvers) {
   View view = viewResolver.resolveViewName(viewName, locale);
   if (view != null) {
    candidateViews.add(view);
   }
   for (MediaType requestedMediaType : requestedMediaTypes) {
    List<String> extensions = this.contentNegotiationManager.resolveFileExtensions(requestedMediaType);
    for (String extension : extensions) {
     String viewNameWithExtension = viewName + '.' + extension;
     view = viewResolver.resolveViewName(viewNameWithExtension, locale);
     if (view != null) {
      candidateViews.add(view);
     }
    }
   }
  }
 }
 if (!CollectionUtils.isEmpty(this.defaultViews)) {
  candidateViews.addAll(this.defaultViews);
 }
 return candidateViews;
}
```
获取所有的候选 View 分为两个步骤：
- 调用各个 ViewResolver 中的 resolveViewName 方法去加载出对应的 View 对象。
- 根据 MediaType 提取出扩展名，再根据扩展名去加载 View 对象，在实际应用中，这一步我们都很少去配置，所以一步基本上是加载不出来 View 对象的，主要靠第一步。

第一步去加载 View 对象，其实就是根据你的 viewName，再结合 ViewResolver 中配置的 prefix、suffix、templateLocation 等属性，找到对应的 View，方法执行流程依次是 resolveViewName->createView->loadView。

唯一需要说的一个重点就是最后的 loadView 方法，我们来看下这个方法：
```
protected View loadView(String viewName, Locale locale) throws Exception {
 AbstractUrlBasedView view = buildView(viewName);
 View result = applyLifecycleMethods(viewName, view);
 return (view.checkResource(locale) ? result : null);
}
```
在这个方法中，View 加载出来后，会调用其 checkResource 方法判断 View 是否存在，如果存在就返回 View，不存在就返回 null。

这是一个非常关键的步骤，但是我们常用的视图对此的处理却不尽相同：
- FreeMarkerView：会老老实实检查。
- ThymeleafView：没有检查这个环节（Thymeleaf 的整个 View 体系不同于 FreeMarkerView 和 JstlView）。
- JstlView：检查结果总是返回 true。

至此，我们就找到了所有的候选 View，但是大家需要注意，这个候选 View 不一定存在，在有 Thymeleaf 的情况下，返回的候选 View 不一定可用，在 JstlView 中，候选 View 也不一定真的存在。

接下来调用 getBestView 方法，从所有的候选 View 中找到最佳的 View。getBestView 方法的逻辑比较简单，就是查找看所有 View 的 MediaType，然后和请求的 MediaType 数组进行匹配，第一个匹配上的就是最佳 View，这个过程它不会检查视图是否真的存在，所以就有可能选出来一个压根没有的视图，最终导致 404。

这就是 ContentNegotiatingViewResolver#resolveViewName 方法的工作过程。

那么这里还涉及到一个问题，ContentNegotiatingViewResolver 中的 ViewResolver 是从哪里来的？这个有两种来源：默认的和手动配置的。我们来看如下一段初始化代码：
```
@Override
protected void initServletContext(ServletContext servletContext) {
 Collection<ViewResolver> matchingBeans =
   BeanFactoryUtils.beansOfTypeIncludingAncestors(obtainApplicationContext(), ViewResolver.class).values();
 if (this.viewResolvers == null) {
  this.viewResolvers = new ArrayList<>(matchingBeans.size());
  for (ViewResolver viewResolver : matchingBeans) {
   if (this != viewResolver) {
    this.viewResolvers.add(viewResolver);
   }
  }
 }
 else {
  for (int i = 0; i < this.viewResolvers.size(); i++) {
   ViewResolver vr = this.viewResolvers.get(i);
   if (matchingBeans.contains(vr)) {
    continue;
   }
   String name = vr.getClass().getName() + i;
   obtainApplicationContext().getAutowireCapableBeanFactory().initializeBean(vr, name);
  }
 }
 AnnotationAwareOrderComparator.sort(this.viewResolvers);
 this.cnmFactoryBean.setServletContext(servletContext);
}
```
- 首先获取到 matchingBeans，这个是获取到了 Spring 容器中的所有视图解析器。
- 如果 viewResolvers 变量为 null，也就是开发者没有给 ContentNegotiatingViewResolver 配置视图解析器，此时会把查到的 matchingBeans 赋值给 viewResolvers。
- 如果开发者为 ContentNegotiatingViewResolver 配置了相关的视图解析器，则去检查这些视图解析器是否存在于 matchingBeans 中，如果不存在，则进行初始化操作。
- 这就是 ContentNegotiatingViewResolver 所做的事情。

## AbstractCachingViewResolver
视图这种文件有一个特点，就是一旦开发好了不怎么变，所以将之缓存起来提高加载速度就显得尤为重要了。事实上我们使用的大部分视图解析器都是支持缓存功能，也即 AbstractCachingViewResolver 实际上有很多用武之地。

我们先来大致了解一下 AbstractCachingViewResolver，然后再来学习它的子类。
```
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws Exception {
 if (!isCache()) {
  return createView(viewName, locale);
 }
 else {
  Object cacheKey = getCacheKey(viewName, locale);
  View view = this.viewAccessCache.get(cacheKey);
  if (view == null) {
   synchronized (this.viewCreationCache) {
    view = this.viewCreationCache.get(cacheKey);
    if (view == null) {
     view = createView(viewName, locale);
     if (view == null && this.cacheUnresolved) {
      view = UNRESOLVED_VIEW;
     }
     if (view != null && this.cacheFilter.filter(view, viewName, locale)) {
      this.viewAccessCache.put(cacheKey, view);
      this.viewCreationCache.put(cacheKey, view);
     }
    }
   }
  }
  else {
  }
  return (view != UNRESOLVED_VIEW ? view : null);
 }
}
```
- 首先如果没有开启缓存，则直接调用 createView 方法创建视图返回。
- 调用 getCacheKey 方法获取缓存的 key。
- 去 viewAccessCache 中查找缓存 View，找到了就直接返回。
- 去 viewCreationCache 中查找缓存 View，找到了就直接返回，没找到就调用 createView 方法创建新的 View，并将 View 放到两个缓存池中。
- 这里有两个缓存池，两个缓存池的区别在于，viewAccessCache 的类型是 ConcurrentHashMap，而 viewCreationCache 的类型是 LinkedHashMap。前者支持并发访问，效率非常高；后者则限制了缓存最大数，效率低于前者。当后者缓存数量达到上限时，会自动删除它里边的元素，在删除自身元素的过程中，也会删除前者 viewAccessCache 中对应的元素。

那么这里还涉及到一个方法，那就是 createView，我们也来稍微看一下：
```
@Nullable
protected View createView(String viewName, Locale locale) throws Exception {
 return loadView(viewName, locale);
}
@Nullable
protected abstract View loadView(String viewName, Locale locale) throws Exception;
```
可以看到，createView 中调用了 loadView，而 loadView 则是一个抽象方法，具体的实现要去子类中查看了。

这就是缓存 View 的查找过程。

直接继承 AbstractCachingViewResolver 的视图解析器有四种：ResourceBundleViewResolver、XmlViewResolver、UrlBasedViewResolver 以及 ThymeleafViewResolver，其中前两种从 Spring5.3 开始就已经被废弃掉了，因此这里松哥就不做过多介绍，我们主要来看下后两者。

## UrlBasedViewResolver
UrlBasedViewResolver 重写了父类的 getCacheKey、createView、loadView 三个方法：

- getCacheKe
```
@Override
protected Object getCacheKey(String viewName, Locale locale) {
 return viewName;
}
```
父类的 getCacheKey 是 viewName + '_' + locale，现在变成了 viewName。

- createView
```
@Override
protected View createView(String viewName, Locale locale) throws Exception {
 if (!canHandle(viewName, locale)) {
  return null;
 }
 if (viewName.startsWith(REDIRECT_URL_PREFIX)) {
  String redirectUrl = viewName.substring(REDIRECT_URL_PREFIX.length());
  RedirectView view = new RedirectView(redirectUrl,
    isRedirectContextRelative(), isRedirectHttp10Compatible());
  String[] hosts = getRedirectHosts();
  if (hosts != null) {
   view.setHosts(hosts);
  }
  return applyLifecycleMethods(REDIRECT_URL_PREFIX, view);
 }
 if (viewName.startsWith(FORWARD_URL_PREFIX)) {
  String forwardUrl = viewName.substring(FORWARD_URL_PREFIX.length());
  InternalResourceView view = new InternalResourceView(forwardUrl);
  return applyLifecycleMethods(FORWARD_URL_PREFIX, view);
 }
 return super.createView(viewName, locale);
}
```
首先调用 canHandle 方法判断是否支持这里的逻辑视图。

接下来判断逻辑视图名前缀是不是 redirect:，如果是，则表示这是一个重定向视图，则构造 RedirectView 进行处理。

接下来判断逻辑视图名前缀是不是 forward:，如果是，则表示这是一个服务端跳转，则构造 InternalResourceView 进行处理。

如果前面都不是，则调用父类的 createView 方法去构建视图，这最终会调用到子类的 loadView 方法。

- loadView
```
@Override
protected View loadView(String viewName, Locale locale) throws Exception {
 AbstractUrlBasedView view = buildView(viewName);
 View result = applyLifecycleMethods(viewName, view);
 return (view.checkResource(locale) ? result : null);
}
```
这里边就干了三件事：

调用 buildView 方法构建 View。
调用 applyLifecycleMethods 方法完成 View 的初始化。
检车 View 是否存在并返回。

第三步比较简单，没啥好说的，主要就是检查视图文件是否存在，像我们常用的 Jsp 视图解析器以及 Freemarker 视图解析器都会去检查，但是 Thymeleaf 不会去检查（具体参见：SpringMVC 中如何同时存在多个视图解析器一文）。这里主要是前两步，要和大家着重说一下，这里又涉及到两个方法 buildView 和 applyLifecycleMethods。

- buildView
这个方法就是用来构建视图的：
```
protected AbstractUrlBasedView buildView(String viewName) throws Exception {
 AbstractUrlBasedView view = instantiateView();
 view.setUrl(getPrefix() + viewName + getSuffix());
 view.setAttributesMap(getAttributesMap());
 String contentType = getContentType();
 if (contentType != null) {
  view.setContentType(contentType);
 }
 String requestContextAttribute = getRequestContextAttribute();
 if (requestContextAttribute != null) {
  view.setRequestContextAttribute(requestContextAttribute);
 }
 Boolean exposePathVariables = getExposePathVariables();
 if (exposePathVariables != null) {
  view.setExposePathVariables(exposePathVariables);
 }
 Boolean exposeContextBeansAsAttributes = getExposeContextBeansAsAttributes();
 if (exposeContextBeansAsAttributes != null) {
  view.setExposeContextBeansAsAttributes(exposeContextBeansAsAttributes);
 }
 String[] exposedContextBeanNames = getExposedContextBeanNames();
 if (exposedContextBeanNames != null) {
  view.setExposedContextBeanNames(exposedContextBeanNames);
 }
 return view;
}
```
首先调用 instantiateView 方法，根据我们在配置视图解析器时提供的 viewClass，构建一个 View 对象返回。

给 view 配置 url，就是前缀+viewName+后缀，其中前缀后缀都是我们在配置视图解析器的时候提供的。

同理，如果用户在配置视图解析器时提供了 content-type，也将其设置给 View 对象。

配置 requestContext 的属性名称。

配置 exposePathVariables，也就是通过 @PathVaribale 注解标记的参数信息。

配置 exposeContextBeansAsAttributes，表示是否可以在 View 中使用容器中的 Bean，该参数我们可以在配置视图解析器时提供。

配置 exposedContextBeanNames，表示可以在 View 中使用容器中的哪些 Bean，该参数我们可以在配置视图解析器时提供。
就这样，视图就构建好了，是不是非常 easy！


- applyLifecycleMethods
```
protected View applyLifecycleMethods(String viewName, AbstractUrlBasedView view) {
 ApplicationContext context = getApplicationContext();
 if (context != null) {
  Object initialized = context.getAutowireCapableBeanFactory().initializeBean(view, viewName);
  if (initialized instanceof View) {
   return (View) initialized;
  }
 }
 return view;
}
```
这个就是 Bean 的初始化，没啥好说的。

UrlBasedViewResolver 的子类还是比较多的，其中有两个比较有代表性的，分别是我们使用 JSP 时所用的 InternalResourceViewResolver 以及当我们使用 Freemarker 时所用的 FreeMarkerViewResolver，由于这两个我们比较常见，因此松哥在这里再和大家介绍一下这两个组件。

## InternalResourceViewResolver
当我们使用 JSP 时，可能会用到这个视图解析器。

InternalResourceViewResolver 主要干了 4 件事：

通过 requiredViewClass 方法规定了视图。
```
@Override
protected Class<?> requiredViewClass() {
 return InternalResourceView.class;
}
```
在构造方法中调用 requiredViewClass 方法去确定视图，如果项目中引入了 JSTL，则会将视图调整为 JstlView。

重写了 instantiateView 方法，会根据实际情况初始化不同的 View：
```
@Override
protected AbstractUrlBasedView instantiateView() {
 return (getViewClass() == InternalResourceView.class ? new InternalResourceView() :
   (getViewClass() == JstlView.class ? new JstlView() : super.instantiateView()));
}
```
会根据实际情况初始化 InternalResourceView 或者 JstlView，或者调用父类的方法完成 View 的初始化。

buildView 方法也重写了，如下：
```
@Override
protected AbstractUrlBasedView buildView(String viewName) throws Exception {
 InternalResourceView view = (InternalResourceView) super.buildView(viewName);
 if (this.alwaysInclude != null) {
  view.setAlwaysInclude(this.alwaysInclude);
 }
 view.setPreventDispatchLoop(true);
 return view;
}
```

这里首先调用父类方法构建出 InternalResourceView，然后配置 alwaysInclude，表示是否允许在使用 forward 的情况下也允许使用 include，最后面的 setPreventDispatchLoop 方法则是防止循环调用。

## FreeMarkerViewResolver
FreeMarkerViewResolver 和 UrlBasedViewResolver 之间还隔了一个 AbstractTemplateViewResolver，AbstractTemplateViewResolver 比较简单，里边只是多出来了五个属性而已
- exposeRequestAttributes：是否将 RequestAttributes 暴露给 View 使用。
- allowRequestOverride：当 RequestAttributes 和 Model 中的数据同名时，是否允许 RequestAttributes 中的参数覆盖 Model 中的同名参数。
- exposeSessionAttributes：是否将 SessionAttributes 暴露给 View 使用。
- allowSessionOverride：当 SessionAttributes 和 Model 中的数据同名时，是否允许 SessionAttributes 中的参数覆盖 Model 中的同名参数。
- exposeSpringMacroHelpers：是否将 RequestContext 暴露出来供 Spring Macro 使用。

这就是 AbstractTemplateViewResolver 特性，比较简单，再来看 FreeMarkerViewResolver。
```
public class FreeMarkerViewResolver extends AbstractTemplateViewResolver {
 public FreeMarkerViewResolver() {
  setViewClass(requiredViewClass());
 }
 public FreeMarkerViewResolver(String prefix, String suffix) {
  this();
  setPrefix(prefix);
  setSuffix(suffix);
 }
 @Override
 protected Class<?> requiredViewClass() {
  return FreeMarkerView.class;
 }
 @Override
 protected AbstractUrlBasedView instantiateView() {
  return (getViewClass() == FreeMarkerView.class ? new FreeMarkerView() : super.instantiateView());
 }
}
```
FreeMarkerViewResolver 的源码就很简单了，配置一下前后缀、重写 requiredViewClass 方法提供 FreeMarkerView，重写 instantiateView 方法完成 View 的初始化。

ThymeleafViewResolver 继承自 AbstractCachingViewResolver，具体的工作流程和前面的差不多，因此这里也就不做过多介绍了。需要注意的是，ThymeleafViewResolver#loadView 方法并不会去检查视图模版是否存在，所以有可能会最终会返回一个不存在的视图

## ViewResolverComposite
最后我们再来看下 ViewResolverComposite，ViewResolverComposite 其实我们在前面的源码分析中已经多次见到过这种模式了，通过 ViewResolverComposite 来代理其他的 ViewResolver，不同的是，这里的 ViewResolverComposite 还为其他 ViewResolver 做了一些初始化操作。为对应的 ViewResolver 分别配置了 applicationContext 以及 servletContext。这里的代码比较简单，我就不贴出来了，最后在 ViewResolverComposite#resolveViewName 方法中，遍历其他视图解析器进行处理：
```
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws Exception {
 for (ViewResolver viewResolver : this.viewResolvers) {
  View view = viewResolver.resolveViewName(viewName, locale);
  if (view != null) {
   return view;
  }
 }
 return null;
}
```