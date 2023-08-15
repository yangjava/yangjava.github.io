---
layout: post
categories: [SpringMV]
description: none
keywords: Spring
---
# SpringMVC参数解析器

## 
在一个 Web 请求中，参数我们无非就是放在地址栏或者请求体中，个别请求可能放在请求头中。

放在地址栏中，我们可以通过如下方式获取参数：
```
String javaboy = request.getParameter("name ");
```
放在请求体中，如果是 key/value 形式，我们可以通过如下方式获取参数：
```
String javaboy = request.getParameter("name ");
```
如果是 JSON 形式，我们则通过如果如下方式获取到输入流，然后解析成 JSON 字符串，再通过 JSON 工具转为对象：
```
BufferedReader reader = new BufferedReader(new InputStreamReader(request.getInputStream()));
String json = reader.readLine();
reader.close();
User user = new ObjectMapper().readValue(json, User.class);
```
如果参数放在请求头中，我们可以通过如下方式获取：
```
String javaboy = request.getHeader("name");
```

如果你用的是 Jsp/Servlet 那一套技术栈，那么参数获取无外乎这几种方式。

如果用了 SpringMVC 框架，有的小伙伴们可能会觉得参数获取方式太丰富了，各种注解如 @RequestParam、@RequestBody、@RequestHeader、@PathVariable，参数可以是 key/value 形式，也可以是 JSON 形式，非常丰富！但是，无论多么丰富，最底层获取参数的方式无外乎上面几种。

那有小伙伴要问了，SpringMVC 到底是怎么样从 request 中把参数提取出来直接给我们用的呢？例如下面这个接口：
```
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello(String name) {
        return "hello "+name;
    }
}
```
我们都知道 name 参数是从 HttpServletRequest 中提取出来的，到底是怎么提取出来的？

## 自定义参数解析器
为了搞清楚这个问题，我们先来自定义一个参数解析器看看。

自定义参数解析器需要实现 HandlerMethodArgumentResolver 接口，我们先来看看该接口：
```
public interface HandlerMethodArgumentResolver {
 boolean supportsParameter(MethodParameter parameter);
 @Nullable
 Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
   NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;

}
```
这个接口中就两个方法：
- supportsParameter：该方法表示是否启用这个参数解析器，返回 true 表示启用，返回 false 表示不启用。
- resolveArgument：这是具体的解析过程，就是从 request 中取出参数的过程，方法的返回值就对应了接口中参数的值。

自定义参数解析器只需要实现该接口即可。

假设我现在有这样一个需求（实际上在 Spring Security 中获取当前登录用户名非常方便，这里只是为了该案例而做，勿抬杠）：

假设我现在系统安全框架使用了 Spring Security，如果我在接口的参数上添加了 @CurrentUserName 注解，那么该参数的值就是当前登录的用户名，像下面这样：
```
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello(@CurrentUserName String name) {
        return "hello "+name;
    }
}
```
要实现这个功能，非常 easy，首先我们自定义一个 @CurrentUserName 注解，如下：
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface CurrentUserName {
}
```
接下来我们自定义参数解析器 CurrentUserNameHandlerMethodArgumentResolver，如下：
```
public class CurrentUserNameHandlerMethodArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType().isAssignableFrom(String.class)&&parameter.hasParameterAnnotation(CurrentUserName.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        User user = (User) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        return user.getUsername();
    }
}
```
supportsParameter：如果参数类型是 String，并且参数上有 @CurrentUserName 注解，则使用该参数解析器。
resolveArgument：该方法的返回值就是参数的具体值，当前登录用户名从 SecurityContextHolder 中获取即可

最后，我们再将自定义的参数解析器配置到 HandlerAdapter 中，配置方式如下：
```
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserNameHandlerMethodArgumentResolver());
    }
}
```
至此，就算配置完成了。

接下来启动项目，用户登录成功后，访问 /hello 接口，就可以看到返回当前登录用户数据了。

这就是我们自定义的一个参数类型解析器。可以看到，非常 Easy。

在 SpringMVC 中，默认也有很多 HandlerMethodArgumentResolver 的实现类，他们处理的问题也都类似。

## PrincipalMethodArgumentResolver
如果我们在项目中使用了 Spring Security，我们可以通过如下方式获取当前登录用户信息：
```
@GetMapping("/hello2")
public String hello2(Principal principal) {
    return "hello " + principal.getName();
}
```

即直接在当前接口的参数中添加 Principal 类型的参数即可，该参数描述了当前登录用户信息，这个用过 Spring Security 的小伙伴应该都知道（不熟悉 Spring Security 的小伙伴可以在公众号【江南一点雨】后台回复 ss）。

那么这个功能是怎么实现的呢？当然就是 PrincipalMethodArgumentResolver 在起作用了！

我们一起来看下这个参数解析器：
```
public class PrincipalMethodArgumentResolver implements HandlerMethodArgumentResolver {

 @Override
 public boolean supportsParameter(MethodParameter parameter) {
  return Principal.class.isAssignableFrom(parameter.getParameterType());
 }

 @Override
 public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
   NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

  HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
  if (request == null) {
   throw new IllegalStateException("Current request is not of type HttpServletRequest: " + webRequest);
  }

  Principal principal = request.getUserPrincipal();
  if (principal != null && !parameter.getParameterType().isInstance(principal)) {
   throw new IllegalStateException("Current user principal is not of type [" +
     parameter.getParameterType().getName() + "]: " + principal);
  }

  return principal;
 }

}
```
supportsParameter：这个方法主要是判断参数类型是不是 Principal，如果参数类型是 Principal，就支持。
resolveArgument：这个方法的逻辑很简单，首先获取原生的请求，再从请求中获取 Principal 对象返回即可。

## RequestParamMapMethodArgumentResolver
```
@RestController
public class HelloController {
    @PostMapping("/hello")
    public void hello(@RequestParam MultiValueMap map) throws IOException {
        //省略...
    }
}
```
这个接口很多小伙伴可能都写过，使用 Map 去接收前端传来的参数，那么这里用到的参数解析器就是 RequestParamMapMethodArgumentResolver。
```
public class RequestParamMapMethodArgumentResolver implements HandlerMethodArgumentResolver {

 @Override
 public boolean supportsParameter(MethodParameter parameter) {
  RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
  return (requestParam != null && Map.class.isAssignableFrom(parameter.getParameterType()) &&
    !StringUtils.hasText(requestParam.name()));
 }

 @Override
 public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
   NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

  ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);

  if (MultiValueMap.class.isAssignableFrom(parameter.getParameterType())) {
   // MultiValueMap
   Class<?> valueType = resolvableType.as(MultiValueMap.class).getGeneric(1).resolve();
   if (valueType == MultipartFile.class) {
    MultipartRequest multipartRequest = MultipartResolutionDelegate.resolveMultipartRequest(webRequest);
    return (multipartRequest != null ? multipartRequest.getMultiFileMap() : new LinkedMultiValueMap<>(0));
   }
   else if (valueType == Part.class) {
    HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
    if (servletRequest != null && MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
     Collection<Part> parts = servletRequest.getParts();
     LinkedMultiValueMap<String, Part> result = new LinkedMultiValueMap<>(parts.size());
     for (Part part : parts) {
      result.add(part.getName(), part);
     }
     return result;
    }
    return new LinkedMultiValueMap<>(0);
   }
   else {
    Map<String, String[]> parameterMap = webRequest.getParameterMap();
    MultiValueMap<String, String> result = new LinkedMultiValueMap<>(parameterMap.size());
    parameterMap.forEach((key, values) -> {
     for (String value : values) {
      result.add(key, value);
     }
    });
    return result;
   }
  }

  else {
   // Regular Map
   Class<?> valueType = resolvableType.asMap().getGeneric(1).resolve();
   if (valueType == MultipartFile.class) {
    MultipartRequest multipartRequest = MultipartResolutionDelegate.resolveMultipartRequest(webRequest);
    return (multipartRequest != null ? multipartRequest.getFileMap() : new LinkedHashMap<>(0));
   }
   else if (valueType == Part.class) {
    HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
    if (servletRequest != null && MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
     Collection<Part> parts = servletRequest.getParts();
     LinkedHashMap<String, Part> result = CollectionUtils.newLinkedHashMap(parts.size());
     for (Part part : parts) {
      if (!result.containsKey(part.getName())) {
       result.put(part.getName(), part);
      }
     }
     return result;
    }
    return new LinkedHashMap<>(0);
   }
   else {
    Map<String, String[]> parameterMap = webRequest.getParameterMap();
    Map<String, String> result = CollectionUtils.newLinkedHashMap(parameterMap.size());
    parameterMap.forEach((key, values) -> {
     if (values.length > 0) {
      result.put(key, values[0]);
     }
    });
    return result;
   }
  }
 }

}
```
supportsParameter：参数类型是 Map，并且使用了 @RequestParam 注解，并且 @RequestParam 注解中没有配置 name 属性，就可以使用该参数解析器。
resolveArgument：具体解析分为两种情况：MultiValueMap 和其他 Map，前者中又分三种情况：MultipartFile、Part 或者其他普通请求，前两者可以处理文件上传，第三个就是普通参数。如果是普通 Map，则直接获取到原始请求参数放到一个 Map 集合中返回即可。

像下面这种接口，参数是怎么解析的：
```
@GetMapping("/hello2")
public void hello2(String name) {
    System.out.println("name = " + name);
}
```
抑或者像下面这种接口，参数是怎么解析的：
```
@GetMapping("/hello/{id}")
public void hello3(@PathVariable Long id) {
    System.out.println("id = " + id);
}
```
这是我们日常中最常见的参数定义方式，相信很多小伙伴对此很感兴趣。由于这块涉及到一个非常庞大的类 AbstractNamedValueMethodArgumentResolver

## 参数解析器
HandlerMethodArgumentResolver 就是我们口口声声说的参数解析器，它的实现类还是蛮多的，因为每一种类型的参数都对应了一个参数解析器：
为了理解方便，我们可以将这些参数解析器分为四大类：

- xxxMethodArgumentResolver：这就是一个普通的参数解析器。
- xxxMethodProcessor：不仅可以当作参数解析器，还可以处理对应类型的返回值。
- xxxAdapter：这种不做参数解析，仅仅用来作为 WebArgumentResolver 类型的参数解析器的适配器。
- HandlerMethodArgumentResolverComposite：这个看名字就知道是一个组合解析器，它是一个代理，具体代理其他干活的那些参数解析器。
大致上可以分为这四类，其中最重要的当然就是前两种了。

## 参数解析器概览
接下来我们来先来大概看看这些参数解析器分别都是用来干什么的。
- MapMethodProcessor 这个用来处理 Map/ModelMap 类型的参数，解析完成后返回 model。
- PathVariableMethodArgumentResolver 这个用来处理使用了 @PathVariable 注解并且参数类型不为 Map 的参数，参数类型为 Map 则使用 PathVariableMapMethodArgumentResolver 来处理。
- PathVariableMapMethodArgumentResolver 见上。
- ErrorsMethodArgumentResolver 这个用来处理 Error 参数，例如我们做参数校验时的 BindingResult。
- AbstractNamedValueMethodArgumentResolver 这个用来处理 key/value 类型的参数，如请求头参数、使用了 @PathVariable 注解的参数以及 Cookie 等。
- RequestHeaderMethodArgumentResolver 这个用来处理使用了 @RequestHeader 注解，并且参数类型不是 Map 的参数（参数类型是 Map 的使用 RequestHeaderMapMethodArgumentResolver）。
- RequestHeaderMapMethodArgumentResolver 见上。
- RequestAttributeMethodArgumentResolver 这个用来处理使用了 @RequestAttribute 注解的参数。
- RequestParamMethodArgumentResolver 这个功能就比较广了。使用了 @RequestParam 注解的参数、文件上传的类型 MultipartFile、或者一些没有使用任何注解的基本类型（Long、Integer）以及 String 等，都使用该参数解析器处理。需要注意的是，如果 @RequestParam 注解的参数类型是 Map，则该注解必须有 name 值，否则解析将由 RequestParamMapMethodArgumentResolver 完成。
- RequestParamMapMethodArgumentResolver 见上。
- AbstractCookieValueMethodArgumentResolver 这个是一个父类，处理使用了 @CookieValue 注解的参数。
- ServletCookieValueMethodArgumentResolver 这个处理使用了 @CookieValue 注解的参数。
- MatrixVariableMethodArgumentResolver 这个处理使用了 @MatrixVariable 注解并且参数类型不是 Map 的参数，如果参数类型是 Map，则使用 MatrixVariableMapMethodArgumentResolver 来处理。
- MatrixVariableMapMethodArgumentResolver 见上。
- SessionAttributeMethodArgumentResolver 这个用来处理使用了 @SessionAttribute 注解的参数。
- ExpressionValueMethodArgumentResolver 这个用来处理使用了 @Value 注解的参数。
- ServletResponseMethodArgumentResolver 这个用来处理 ServletResponse、OutputStream 以及 Writer 类型的参数。
- ModelMethodProcessor 这个用来处理 Model 类型参数，并返回 model。
- ModelAttributeMethodProcessor 这个用来处理使用了 @ModelAttribute 注解的参数。
- SessionStatusMethodArgumentResolver 这个用来处理 SessionStatus 类型的参数。
- PrincipalMethodArgumentResolver 这个用来处理 Principal 类型参数
- AbstractMessageConverterMethodArgumentResolver 这是一个父类，当使用 HttpMessageConverter 解析 requestbody 类型参数时，相关的处理类都会继承自它。
- RequestPartMethodArgumentResolver 这个用来处理使用了 @RequestPart 注解、MultipartFile 以及 Part 类型的参数。
- AbstractMessageConverterMethodProcessor 这是一个工具类，不承担参数解析任务。
- RequestResponseBodyMethodProcessor 这个用来处理添加了 @RequestBody 注解的参数。
- HttpEntityMethodProcessor 这个用来处理 HttpEntity 和 RequestEntity 类型的参数。
- ContinuationHandlerMethodArgumentResolver
- AbstractWebArgumentResolverAdapter 这种不做参数解析，仅仅用来作为 WebArgumentResolver 类型的参数解析器的适配器。
- ServletWebArgumentResolverAdapter 这个给父类提供 request。
- UriComponentsBuilderMethodArgumentResolver 这个用来处理 UriComponentsBuilder 类型的参数。
- ServletRequestMethodArgumentResolver 这个用来处理 WebRequest、ServletRequest、MultipartRequest、HttpSession、Principal、InputStream、Reader、HttpMethod、Locale、TimeZone、ZoneId 类型的参数。
- HandlerMethodArgumentResolverComposite 这个看名字就知道是一个组合解析器，它是一个代理，具体代理其他干活的那些参数解析器。
- RedirectAttributesMethodArgumentResolver 这个用来处理 RedirectAttributes 类型的参数，RedirectAttributes 松哥在之前的文章中和大家介绍过：SpringMVC 中的参数还能这么传递？涨姿势了！。

好了，各个参数解析器的大致功能就给大家介绍完了，接下来我们选择其中一种，来具体说说它的源码。

## AbstractNamedValueMethodArgumentResolver
AbstractNamedValueMethodArgumentResolver 是一个抽象类，一些键值对类型的参数解析器都是通过继承它实现的，它里边定义了很多这些键值对类型参数解析器的公共操作。

AbstractNamedValueMethodArgumentResolver 中也是应用了很多模版模式，例如它没有实现 supportsParameter 方法，该方法的具体实现在不同的子类中，resolveArgument 方法它倒是实现了，我们一起来看下：

```
@Override
@Nullable
public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
  NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
 NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
 MethodParameter nestedParameter = parameter.nestedIfOptional();
 Object resolvedName = resolveEmbeddedValuesAndExpressions(namedValueInfo.name);
 if (resolvedName == null) {
  throw new IllegalArgumentException(
    "Specified name must not resolve to null: [" + namedValueInfo.name + "]");
 }
 Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
 if (arg == null) {
  if (namedValueInfo.defaultValue != null) {
   arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
  }
  else if (namedValueInfo.required && !nestedParameter.isOptional()) {
   handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
  }
  arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
 }
 else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
  arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
 }
 if (binderFactory != null) {
  WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
  try {
   arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
  }
  catch (ConversionNotSupportedException ex) {
   throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
     namedValueInfo.name, parameter, ex.getCause());
  }
  catch (TypeMismatchException ex) {
   throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
     namedValueInfo.name, parameter, ex.getCause());
  }
  // Check for null value after conversion of incoming argument value
  if (arg == null && namedValueInfo.defaultValue == null &&
    namedValueInfo.required && !nestedParameter.isOptional()) {
   handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
  }
 }
 handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);
 return arg;
}
```

首先根据当前请求获取一个 NamedValueInfo 对象，这个对象中保存了参数的三个属性：参数名、参数是否必须以及参数默认值。具体的获取过程就是先去缓存中拿，缓存中如果有，就直接返回，缓存中如果没有，则调用 createNamedValueInfo 方法去创建，将创建结果缓存起来并返回。createNamedValueInfo 方法是一个模版方法，具体的实现在子类中。
接下来处理 Optional 类型参数。
resolveEmbeddedValuesAndExpressions 方法是为了处理注解中使用了 SpEL 表达式的情况，例如如下接口：
```
@GetMapping("/hello2")
public void hello2(@RequestParam(value = "${aa.bb}") String name) {
    System.out.println("name = " + name);
}
```
参数名使用了表达式，那么 resolveEmbeddedValuesAndExpressions 方法的目的就是解析出表达式的值，如果没用到表达式，那么该方法会将原参数原封不动返回。

接下来调用 resolveName 方法解析出参数的具体值，这个方法也是一个模版方法，具体的实现在子类中。

如果获取到的参数值为 null，先去看注解中有没有默认值，然后再去看参数值是否是必须的，如果是，则抛异常出来，否则就设置为 null 即可。 

如果解析出来的参数值为空字符串 ""，则也去 resolveEmbeddedValuesAndExpressions 方法中走一遭。

最后则是 WebDataBinder 的处理，解决一些全局参数的问题，WebDataBinder 

大致的流程就是这样。

在这个流程中，我们看到主要有如下两个方法是在子类中实现的：

createNamedValueInfo
resolveName
在加上 supportsParameter 方法，子类中一共有三个方法需要我们重点分析。

那么接下来我们就以 RequestParamMethodArgumentResolver 为例，来看下这三个方法。

## RequestParamMethodArgumentResolver

supportsParameter
```
@Override
public boolean supportsParameter(MethodParameter parameter) {
 if (parameter.hasParameterAnnotation(RequestParam.class)) {
  if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
   RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
   return (requestParam != null && StringUtils.hasText(requestParam.name()));
  }
  else {
   return true;
  }
 }
 else {
  if (parameter.hasParameterAnnotation(RequestPart.class)) {
   return false;
  }
  parameter = parameter.nestedIfOptional();
  if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
   return true;
  }
  else if (this.useDefaultResolution) {
   return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
  }
  else {
   return false;
  }
 }
}
public static boolean isSimpleProperty(Class<?> type) {
 return isSimpleValueType(type) || (type.isArray() && isSimpleValueType(type.getComponentType()));
}
public static boolean isSimpleValueType(Class<?> type) {
 return (Void.class != type && void.class != type &&
   (ClassUtils.isPrimitiveOrWrapper(type) ||
   Enum.class.isAssignableFrom(type) ||
   CharSequence.class.isAssignableFrom(type) ||
   Number.class.isAssignableFrom(type) ||
   Date.class.isAssignableFrom(type) ||
   Temporal.class.isAssignableFrom(type) ||
   URI.class == type ||
   URL.class == type ||
   Locale.class == type ||
   Class.class == type));
}
```
从 supportsParameter 方法中可以非常方便的看出支持的参数类型：

首先参数如果有 @RequestParam 注解的话，则分两种情况：参数类型如果是 Map，则 @RequestParam 注解必须配置 name 属性，否则不支持；如果参数类型不是 Map，则直接返回 true，表示总是支持（想想自己平时使用的时候是不是这样）。
参数如果含有 @RequestPart 注解，则不支持。
检查下是不是文件上传请求，如果是，返回 true 表示支持。
如果前面都没能返回，则使用默认的解决方案，判断是不是简单类型，主要就是 Void、枚举、字符串、数字、日期等等。

createNamedValueInfo
```
@Override
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
 RequestParam ann = parameter.getParameterAnnotation(RequestParam.class);
 return (ann != null ? new RequestParamNamedValueInfo(ann) : new RequestParamNamedValueInfo());
}
private static class RequestParamNamedValueInfo extends NamedValueInfo {
 public RequestParamNamedValueInfo() {
  super("", false, ValueConstants.DEFAULT_NONE);
 }
 public RequestParamNamedValueInfo(RequestParam annotation) {
  super(annotation.name(), annotation.required(), annotation.defaultValue());
 }
}
```
获取注解，读取注解中的属性，构造 RequestParamNamedValueInfo 对象返回。

resolveName
```
@Override
@Nullable
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
 HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
 if (servletRequest != null) {
  Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
  if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
   return mpArg;
  }
 }
 Object arg = null;
 MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
 if (multipartRequest != null) {
  List<MultipartFile> files = multipartRequest.getFiles(name);
  if (!files.isEmpty()) {
   arg = (files.size() == 1 ? files.get(0) : files);
  }
 }
 if (arg == null) {
  String[] paramValues = request.getParameterValues(name);
  if (paramValues != null) {
   arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
  }
 }
 return arg;
}
```
这个方法思路也比较清晰：

前面两个 if 主要是为了处理文件上传请求。
如果不是文件上传请求，则调用 request.getParameterValues 方法取出参数返回即可。

整个过程还是比较 easy 的。小伙伴们可以在此基础之上自行分析 PathVariableMethodArgumentResolver 的原理，也很容易。

## Model
在 SpringMVC 中，大家都知道有一个特殊的参数 Model，它的使用方式像下面这样：
```
@Controller
public class HelloController {
    @GetMapping("/01")
    public String hello(Model model) {
        model.addAttribute("name", "javaboy");
        return "01";
    }
}
```
这样一个看起来人畜无害的普通参数，里边也会包含你的知识盲区吗？说不定真的包含了，不信你就往下看。

### 基本用法
仅仅从使用上来说，Model 有两方面的功能：

携带参数
返回参数
先说携带参数：当我们在一个接口中放上 Model 这个参数之后，这个 Model 不一定是空白的，它里边可能已经有了携带的参数，携带的参数可能来自上一次 @SessionAttributes 注解标记过的参数，也可能来自 @ModelAttribute 注解标记过的全局参数。

在来说返回参数，Model 中的属性，你最终都可以在前端视图中获取到，这个没啥好说的

## @SessionAttributes
@SessionAttributes 作用于处理器类上，这个注解可以把参数存储到 session 中，进而可以实现在多个请求之间传递参数。

@SessionAttributes 的作用类似于 Session 的 Attribute 属性，但不完全一样，一般来说 @SessionAttributes 设置的参数只用于临时的参数传递，而不是长期的保存，参数用完之后可以通过 SessionStatus 将之清除。

通过 @SessionAttributes 注解设置的参数我们可以在三个地方获取：

在当前的视图中直接通过 request.getAttribute 或 session.getAttribute 获取。
例如如下接口：
```
@Controller
@SessionAttributes("name")
public class HelloController {
    @GetMapping("/01")
    public String hello(Model model) {
        model.addAttribute("name", "javaboy");
        return "01";
    }
}
```
name 属性会被临时保存在 session 中，在前端页面中，我们既可以从 request 域中获取也可以从 session 域中获取，以 Thymeleaf 页面模版为例：
```
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<div>
    <div th:text="${#request.getAttribute('name')}"></div>
    <div th:text="${#session.getAttribute('name')}"></div>
</div>
</body>
</html>
```
如果没有使用 @SessionAttributes 注解，那就只能从 request 域中获取，而不能从 session 域中获取。

在后面的请求中，也可以通过 session.getAttribute 获取。
参数既然存在 session 中，那就有一个好处，就是无论是服务器端跳转还是客户端跳转，参数都不会丢失。例如如下接口：
```
@Controller
@SessionAttributes("name")
public class HelloController {
    @GetMapping("/01")
    public String hello(Model model) {
        model.addAttribute("name", "javaboy");
        return "forward:/index";
    }

    @GetMapping("/02")
    public String hello2(Model model) {
        model.addAttribute("name", "javaboy");
        return "redirect:/index";
    }

    @GetMapping("/index")
    public String index() {
        return "01";
    }
}
```
无论开发者访问 `http://localhost:8080/01` 还是 `http://localhost:8080/02` ，都能看到页面，并且 name 属性的值也能在页面上渲染出来。

在后续的请求中，也可以直接从 Model 中获取。
```
@Controller
@SessionAttributes("name")
public class HelloController {
    @GetMapping("/01")
    public String hello(Model model) {
        model.addAttribute("name", "javaboy");
        return "forward:/index";
    }
    @GetMapping("/03")
    @ResponseBody
    public void hello3(Model model) {
        Object name = model.getAttribute("name");
        System.out.println("name = " + name);
    }
}
```
访问完 /01 接口之后，再去访问 /03 接口，也可以拿到 Model 中的数据。

第三种方式还有一个变体，如下：
```
@Controller
@SessionAttributes("name")
public class HelloController {
    @GetMapping("/01")
    public String hello(Model model) {
        model.addAttribute("name", "javaboy");
        return "forward:/index";
    }
    @GetMapping("/04")
    @ResponseBody
    public void hello4(@SessionAttribute("name") String name) {
        System.out.println("name = " + name);
    }
}
```
就是参数中不使用 Model，而是使用 @SessionAttribute 注解，直接将 session 中的属性绑定到参数上。

使用了 @SessionAttributes 注解之后，可以调用 SessionStatus.setComplete 方法来清除数据，注意这个方法只是清除 SessionAttribute 里的参数，而不会清除正常 Session 中的参数。

例如下面这样：
```
@Controller
@SessionAttributes("name")
public class HelloController {
    @GetMapping("/01")
    public String hello(Model model) {
        model.addAttribute("name", "javaboy");
        return "forward:/index";
    }

    @GetMapping("/04")
    @ResponseBody
    public void hello4(@SessionAttribute("name") String name) {
        System.out.println("name = " + name);
    }

    @GetMapping("/05")
    @ResponseBody
    public void hello5(SessionStatus sessionStatus) {
        sessionStatus.setComplete();
    }
}
```
首先访问 /01 接口，访问完了就有数据了，这个时候访问 /04 接口，就会打印出数据，继续访问 /05 接口，访问完成后，再去访问 /04 接口，此时就会发现数据没了，因为被清除了。

现在，大家对 @SessionAttributes 注解的用法应该有了一定的认知了吧。

### ModelFactory
接下来我们就来研究一下 ModelFactory，ModelFactory 是用来维护 Model 的，上面这一切，我们可以从 ModelFactory 中找到端倪。

整体上来说，ModelFactory 包含两方面的功能：1.初始化 Model；2.将 Model 中相应的参数更新到 SessionAtrributes 中。两方面的功能我们分别来看，先来看初始化问题。
```
public void initModel(NativeWebRequest request, ModelAndViewContainer container, HandlerMethod handlerMethod)
  throws Exception {
 Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
 container.mergeAttributes(sessionAttributes);
 invokeModelAttributeMethods(request, container);
 for (String name : findSessionAttributeArguments(handlerMethod)) {
  if (!container.containsAttribute(name)) {
   Object value = this.sessionAttributesHandler.retrieveAttribute(request, name);
   if (value == null) {
    throw new HttpSessionRequiredException("Expected session attribute '" + name + "'", name);
   }
   container.addAttribute(name, value);
  }
 }
}
```
这个 initModel 方法比较逻辑比较简单：

首先它会从 @SessionAttributes 中取出参数，然后合并进 ModelAndViewContainer 容器中。
接下来调用含有 @ModelAttribute 注解的方法，并将结果合并进 ModelAndViewContainer 容器中。
寻找那些既有 @ModelAttribute 注解又有 @SessionAttributes 注解的属性，找到后，如果这些属性不存在于 ModelAndViewContainer 容器中，就从 SessionAttributes 中获取并设置到 ModelAndViewContainer 容器中。
我们先来看看第一个 retrieveAttributes 方法：
```
public Map<String, Object> retrieveAttributes(WebRequest request) {
 Map<String, Object> attributes = new HashMap<>();
 for (String name : this.knownAttributeNames) {
  Object value = this.sessionAttributeStore.retrieveAttribute(request, name);
  if (value != null) {
   attributes.put(name, value);
  }
 }
 return attributes;
}
```
这个其实没啥好说的，因为逻辑很清晰，knownAttributeNames 就是我们在使用 @SessionAttributes 注解时配置的属性名字，属性名字可以是一个数组。遍历 knownAttributeNames 属性，从 session 中获取相关数据存入 Map 集合中。

再来看第二个 invokeModelAttributeMethods 方法：
```
private void invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer container)
  throws Exception {
 while (!this.modelMethods.isEmpty()) {
  InvocableHandlerMethod modelMethod = getNextModelMethod(container).getHandlerMethod();
  ModelAttribute ann = modelMethod.getMethodAnnotation(ModelAttribute.class);
  if (container.containsAttribute(ann.name())) {
   if (!ann.binding()) {
    container.setBindingDisabled(ann.name());
   }
   continue;
  }
  Object returnValue = modelMethod.invokeForRequest(request, container);
  if (modelMethod.isVoid()) {
   if (StringUtils.hasText(ann.value())) {
   }
   continue;
  }
  String returnValueName = getNameForReturnValue(returnValue, modelMethod.getReturnType());
  if (!ann.binding()) {
   container.setBindingDisabled(returnValueName);
  }
  if (!container.containsAttribute(returnValueName)) {
   container.addAttribute(returnValueName, returnValue);
  }
 }
}
```
首先获取含有 @ModelAttribute 注解的方法，然后获取到该注解。
获取 @ModelAttribute 注解，并提取出它的 name 属性值，然后查看 ModelAndViewContainer 容器中是否已经包含了该属性，如果已经包含了，并且在 @ModelAttribute 注解中设置了不绑定，则将该属性添加到 ModelAndViewContainer 容器中的禁止绑定上面去。
接下来通过 invokeForRequest 方法去调用含有 @ModelAttribute 注解的方法，并获取返回值。
如果含有 @ModelAttribute 注解的方法返回值为 void，则该方法到此为止。
接下来解析出返回值的参数名，有的小伙伴们说，参数名不就是 @ModelAttribute 注解中配置的 name 属性吗？这当然没错！但是有时候用户没有配置 name 属性，那么这个时候就会对应一套默认的 name 生成方案。默认的名字生成方案是这样的：
如果返回对象前两个字母都是大写，那就原封不动返回，否则首字母小写后返回。
如果返回类型是数组或者集合，则在真实类型后加上 List，例如 List对象 longList。
有了 returnValueName 之后，再去判断是否要禁止属性绑定。最后如果 ModelAndViewContainer 容器中不包含该属性，则添加进来。
这就是 Model 初始化的过程，可以看到，数据最终都被保存进 ModelAndViewContainer 容器中了，至于在该容器中数据被保存到哪个属性，则要看实际情况，可能是 defaultModel 也可能是 redirectModel

最后我们再来看看 ModelFactory 中修改 Model 的过程：
```
public void updateModel(NativeWebRequest request, ModelAndViewContainer container) throws Exception {
 ModelMap defaultModel = container.getDefaultModel();
 if (container.getSessionStatus().isComplete()){
  this.sessionAttributesHandler.cleanupAttributes(request);
 }
 else {
  this.sessionAttributesHandler.storeAttributes(request, defaultModel);
 }
 if (!container.isRequestHandled() && container.getModel() == defaultModel) {
  updateBindingResult(request, defaultModel);
 }
}
```
修改的时候会首先判断一下是否已经调用了 sessionStatus.setComplete(); 方法，如果调用过了，就执行清除操作，否则就进行正常的更新操作即可，更新的数据就是 ModelAndViewContainer 中的 defaultModel。最后判断是否需要进行页面渲染，如果需要，再给参数分别设置 BindingResult 以备视图使用。

现在，大家应该已经清楚了 ModelFactory 的功能了。

一句话，ModelFactory 在初始化的时候，就直接从 SessionAttributes 以及 ModelAttribute 处加载到数据，放到 ModelAndViewContainer 中，更新的时候，则有可能清除 SessionAttributes 中的数据。「这里大家需要把握一点，就是数据最终被存入 ModelAndViewContainer 中了。」

### 相关的参数解析器
这是 Model 初始化的过程，初始化完成后，参数最终会在参数解析器中被解析，关于参数解析器

这里涉及到的参数解析器就是 ModelMethodProcessor，我们来看下它里边两个关键的方法：
```
@Override
public boolean supportsParameter(MethodParameter parameter) {
 return Model.class.isAssignableFrom(parameter.getParameterType());
}
@Override
@Nullable
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
  NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
 return mavContainer.getModel();
}
```
可以看到，支持的参数类型就是 Model，参数的值则是直接返回 ModelAndViewContainer 中的 model 对象。

这里还有一个类似的参数处理器 MapMethodProcessor：
```
@Override
public boolean supportsParameter(MethodParameter parameter) {
 return (Map.class.isAssignableFrom(parameter.getParameterType()) &&
   parameter.getParameterAnnotations().length == 0);
}
@Override
@Nullable
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
  NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
 return mavContainer.getModel();
}
```
这个是处理 Map 类型的参数，最终返回的也是 ModelAndViewContainer 中的 model，你是否发现什么了？对了，你把 Model 参数换成 Map 或者 ModelMap（ModelMap 本质上也是 Map，使用的参数解析器也是 MapMethodProcessor），最终效果是一样的！

前面我们还使用了 @SessionAttribute 注解，这个注解的 name 属性就绑定了 SessionAttributes 中对应的属性并赋值给变量，它使用的参数解析器是 SessionAttributeMethodArgumentResolver，我们来看下它里边的核心方法：
```
public class SessionAttributeMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver {
 @Override
 public boolean supportsParameter(MethodParameter parameter) {
  return parameter.hasParameterAnnotation(SessionAttribute.class);
 }
 @Override
 @Nullable
 protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) {
  return request.getAttribute(name, RequestAttributes.SCOPE_SESSION);
 }
}
```
可以看到，这个参数最终对应的值就是从 session 中取出对应的 name 属性值。

最后，我们再来梳理一下整个过程：当请求到达后，首先要初始化 Model，初始化 Model 的时候，会根据 @SessionAttributes 注解从 session 中读取相关数据放入 ModelAndViewContainer 中，同时也会加载 @ModelAttribute 注解配置的全局数据到 ModelAndViewContainer 中。最终在参数解析器中，返回 ModelAndViewContainer 中的 model 即可。







