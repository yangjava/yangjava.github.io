---
layout: post
categories: [SpringMV]
description: none
keywords: Spring
---
# SpringMVC组件本地化解析
对于LocaleResolver，其主要作用在于根据不同的用户区域展示不同的视图，而用户的区域也称为Locale，该信息是可以由前端直接获取的。通过这种方式，可以实现一种国际化的目的，比如针对美国用户可以提供一个视图，而针对中国用户则可以提供另一个视图。

解析视图需要两个参数：一是视图名，另一个是Locale。视图名是处理器返回的，Locale是从哪里来的？这就是LocaleResolver要做的事情

## 本地化解析
国际化，也叫 i18n，为啥叫这个名字呢？因为国际化英文是 internationalization ，在 i 和 n 之间有 18 个字母，所以叫 i18n。我们的应用如果做了国际化就可以在不同的语言环境下，方便的进行切换，最常见的就是中文和英文之间的切换，国际化这个功能也是相当的常见。

## 国际化配置
还是先来说说用法，再来说源码，这样大家不容易犯迷糊。如何处理国际化问题。

首先国际化我们可能有两种需求：

在页面渲染时实现国际化（这个借助于 Spring 标签实现）
在接口中获取国际化匹配后的消息
大致上就是上面这两种场景。接下来通过一个简单的用法来和大家演示下具体玩法。

首先我们在项目的 resources 目录下新建语言文件，language_en_US.properties 和 language_zh-CN.properties：

language_en_US.properties：
```
login.username=Username
login.password=Password
```

language_zh-CN.properties：
```
login.username=用户名
login.password=用户密码
```
这两个分别对应英中文环境。配置文件写好之后，还需要在 SpringMVC 容器中提供一个 ResourceBundleMessageSource 实例去加载这两个实例，如下：
```
<bean class="org.springframework.context.support.ResourceBundleMessageSource" id="messageSource">
    <property name="basename" value="language"/>
    <property name="defaultEncoding" value="UTF-8"/>
</bean>
```
这里配置了文件名 language 和默认的编码格式。

接下来我们新建一个 login.jsp 文件，如下：
```
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
<spring:message code="login.username"/> <input type="text"> <br>
<spring:message code="login.password"/> <input type="text"> <br>
</body>
</html>
```
在这个文件中，我们通过 spring:message 标签来引用变量，该标签会根据当前的实际情况，选择合适的语言文件。

接下来我们为 login.jsp 提供一个控制器：
```
@Controller
public class LoginController {
    @Autowired
    MessageSource messageSource;
    @GetMapping("/login")
    public String login() {
        String username = messageSource.getMessage("login.username", null, LocaleContextHolder.getLocale());
        String password = messageSource.getMessage("login.password", null, LocaleContextHolder.getLocale());
        System.out.println("username = " + username);
        System.out.println("password = " + password);
        return "login";
    }
}
```
控制器中直接返回 login 视图即可。

另外我这还注入了 MessageSource 对象，主要是为了向大家展示如何在处理器中获取国际化后的语言文字。

配置完成后，启动项目进行测试。

默认情况下，系统是根据请求头的中 Accept-Language 字段来判断当前的语言环境的，该这个字段由浏览器自动发送，我们这里为了测试方便，可以使用 POSTMAN 进行测试，然后手动设置 Accept_Language 字段。
```
Accept-Language= zh-CN
Accept-Language= en_US
```
上面这个是基于 AcceptHeaderLocaleResolver 来解析出当前的区域和语言的。

有的时候，我们希望语言环境直接通过请求参数来传递，而不是通过请求头来传递，这个需求我们通过 SessionLocaleResolver 或者 CookieLocaleResolver 都可以实现。

先来看 SessionLocaleResolver。

首先在 SpringMVC 配置文件中提供 SessionLocaleResolver 的实例，同时配置一个拦截器，如下：
```
<mvc:interceptors>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
            <property name="paramName" value="locale"/>
        </bean>
    </mvc:interceptor>
</mvc:interceptors>
<bean class="org.springframework.web.servlet.i18n.SessionLocaleResolver" id="localeResolver">
</bean>
```
SessionLocaleResolver 是负责区域解析的，这个没啥好说的。拦截器 LocaleChangeInterceptor 则主要是负责参数解析的，我们在配置拦截器的时候，设置了参数名为 locale（默认即此），也就是说我们将来可以通过 locale 参数来传递当前的环境信息。

此时我们可以直接通过 locale 参数来控制当前的语言环境，这个 locale 参数就是在前面所配置的 LocaleChangeInterceptor 拦截器中被自动解析的。

如果你不想配置 LocaleChangeInterceptor 拦截器也是可以的，直接自己手动解析 locale 参数然后设置 locale 也行，像下面这样：
```
@Controller
public class LoginController {
    @Autowired
    MessageSource messageSource;
    @GetMapping("/login")
    public String login(String locale,HttpSession session) {
        if ("zh-CN".equals(locale)) {
            session.setAttribute(SessionLocaleResolver.LOCALE_SESSION_ATTRIBUTE_NAME, new Locale("zh", "CN"));
        } else if ("en-US".equals(locale)) {
            session.setAttribute(SessionLocaleResolver.LOCALE_SESSION_ATTRIBUTE_NAME, new Locale("en", "US"));
        }
        String username = messageSource.getMessage("login.username", null, LocaleContextHolder.getLocale());
        String password = messageSource.getMessage("login.password", null, LocaleContextHolder.getLocale());
        System.out.println("username = " + username);
        System.out.println("password = " + password);
        return "login";
    }
}
```
SessionLocaleResolver 所实现的功能也可以通过 CookieLocaleResolver 来实现，不同的是前者将解析出来的区域信息保存在 session 中，而后者则保存在 Cookie 中。保存在 session 中，只要 session 没有发生变化，后续就不用再次传递区域语言参数了，保存在 Cookie 中，只要 Cookie 没变，后续也不用再次传递区域语言参数了

使用 CookieLocaleResolver 的方式很简单，直接在 SpringMVC 中提供 CookieLocaleResolver 的实例即可，如下：
```
<bean class="org.springframework.web.servlet.i18n.CookieLocaleResolver" id="localeResolver"/>
```

注意这里也需要使用到 LocaleChangeInterceptor 拦截器，如果不使用该拦截器，则需要自己手动解析并配置语言环境，手动解析并配置的方式如下：
```
@GetMapping("/login3")
public String login3(String locale, HttpServletRequest req, HttpServletResponse resp) {
    CookieLocaleResolver resolver = new CookieLocaleResolver();
    if ("zh-CN".equals(locale)) {
        resolver.setLocale(req, resp, new Locale("zh", "CN"));
    } else if ("en-US".equals(locale)) {
        resolver.setLocale(req, resp, new Locale("en", "US"));
    }
    String username = messageSource.getMessage("login.username", null, LocaleContextHolder.getLocale());
    String password = messageSource.getMessage("login.password", null, LocaleContextHolder.getLocale());
    System.out.println("username = " + username);
    System.out.println("password = " + password);
    return "login";
}
```
配置完成后，启动项目进行测试，这次测试的方式跟 SessionLocaleResolver 的测试方式一致

## LocaleResolver
国际化这块主要涉及到的组件是 LocaleResolver，这是一个开放的接口，官方默认提供了四个实现。当前该使用什么环境，主要是通过 LocaleResolver 来进行解析的。

### LocaleResolver
```
public interface LocaleResolver {
 Locale resolveLocale(HttpServletRequest request);
 void setLocale(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable Locale locale);

}
```
这里两个方法：
resolveLocale：根据当前请求解析器出 Locale 对象。
设置 Locale 对象。

我们来看看 LocaleResolver 的继承关系：
最终负责实现的其实就四个：
- AcceptHeaderLocaleResolver：根据请求头中的 Accept-Language 字段来确定当前的区域语言等。
- SessionLocaleResolver：根据请求参数来确定区域语言等，确定后会保存在 Session 中，只要 Session 不变，Locale 对象就一直有效。
- CookieLocaleResolver：根据请求参数来确定区域语言等，确定后会保存在 Cookie 中，只要 Session 不变，Locale 对象就一直有效。
- FixedLocaleResolver：配置时直接提供一个 Locale 对象，以后不能修改。

接下来我们就对这几个类逐一进行分析。

### AcceptHeaderLocaleResolver
AcceptHeaderLocaleResolver 直接实现了 LocaleResolver 接口，我们来看它的 resolveLocale 方法：
```
@Override
public Locale resolveLocale(HttpServletRequest request) {
 Locale defaultLocale = getDefaultLocale();
 if (defaultLocale != null && request.getHeader("Accept-Language") == null) {
  return defaultLocale;
 }
 Locale requestLocale = request.getLocale();
 List<Locale> supportedLocales = getSupportedLocales();
 if (supportedLocales.isEmpty() || supportedLocales.contains(requestLocale)) {
  return requestLocale;
 }
 Locale supportedLocale = findSupportedLocale(request, supportedLocales);
 if (supportedLocale != null) {
  return supportedLocale;
 }
 return (defaultLocale != null ? defaultLocale : requestLocale);
}
```
- 首先去获取默认的 Locale 对象。
- 如果存在默认的 Locale 对象，并且请求头中没有设置 Accept-Language 字段，则直接返回默认的 Locale。
- 从 request 中取出当前的 Locale 对象，然后查询出支持的 supportedLocales，如果 supportedLocales 或者 supportedLocales 中包含 requestLocale，则直接返回 requestLocale。
- 如果前面还是没有匹配成功的，则从 request 中取出 locales 集合，然后再去和支持的 locale 进行比对，选择匹配成功的 locale 返回。
- 如果前面都没能返回，则判断 defaultLocale 是否为空，如果不为空，就返回 defaultLocale，否则返回 defaultLocale。

再来看看它的 setLocale 方法，直接抛出异常，意味着通过请求头处理 Locale 是不允许修改的。
```
@Override
public void setLocale(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable Locale locale) {
 throw new UnsupportedOperationException(
   "Cannot change HTTP accept header - use a different locale resolution strategy");
}
```

### SessionLocaleResolver
SessionLocaleResolver 的实现多了一个抽象类 AbstractLocaleContextResolver，AbstractLocaleContextResolver 中增加了对 TimeZone 的支持，我们先来看下 AbstractLocaleContextResolver：
```
public abstract class AbstractLocaleContextResolver extends AbstractLocaleResolver implements LocaleContextResolver {
 @Nullable
 private TimeZone defaultTimeZone;
 public void setDefaultTimeZone(@Nullable TimeZone defaultTimeZone) {
  this.defaultTimeZone = defaultTimeZone;
 }
 @Nullable
 public TimeZone getDefaultTimeZone() {
  return this.defaultTimeZone;
 }
 @Override
 public Locale resolveLocale(HttpServletRequest request) {
  Locale locale = resolveLocaleContext(request).getLocale();
  return (locale != null ? locale : request.getLocale());
 }
 @Override
 public void setLocale(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable Locale locale) {
  setLocaleContext(request, response, (locale != null ? new SimpleLocaleContext(locale) : null));
 }

}
```
可以看到，多了一个 TimeZone 属性。从请求中解析出 Locale 还是调用了 resolveLocaleContext 方法，该方法在子类中被实现，另外调用 setLocaleContext 方法设置 Locale，该方法的实现也在子类中。

我们来看下它的子类 SessionLocaleResolver：
```
@Override
public Locale resolveLocale(HttpServletRequest request) {
 Locale locale = (Locale) WebUtils.getSessionAttribute(request, this.localeAttributeName);
 if (locale == null) {
  locale = determineDefaultLocale(request);
 }
 return locale;
}
```
直接从 Session 中获取 Locale，默认的属性名是 SessionLocaleResolver.class.getName() + ".LOCALE"，如果 session 中不存在 Locale 信息，则调用 determineDefaultLocale 方法去加载 Locale，该方法会首先找到 defaultLocale，如果 defaultLocale 不为 null 就直接返回，否则就从 request 中获取 Locale 返回。

再来看 setLocaleContext 方法，就是将解析出来的 Locale 保存起来。
```
@Override
public void setLocaleContext(HttpServletRequest request, @Nullable HttpServletResponse response,
  @Nullable LocaleContext localeContext) {
 Locale locale = null;
 TimeZone timeZone = null;
 if (localeContext != null) {
  locale = localeContext.getLocale();
  if (localeContext instanceof TimeZoneAwareLocaleContext) {
   timeZone = ((TimeZoneAwareLocaleContext) localeContext).getTimeZone();
  }
 }
 WebUtils.setSessionAttribute(request, this.localeAttributeName, locale);
 WebUtils.setSessionAttribute(request, this.timeZoneAttributeName, timeZone);
}
```
保存到 Session 中即可。大家可以看到，这种保存方式其实和我们前面演示的自己保存代码基本一致，殊途同归。

### FixedLocaleResolver
FixedLocaleResolver 有三个构造方法，无论调用哪一个，都会配置默认的 Locale：
```
public FixedLocaleResolver() {
 setDefaultLocale(Locale.getDefault());
}
public FixedLocaleResolver(Locale locale) {
 setDefaultLocale(locale);
}
public FixedLocaleResolver(Locale locale, TimeZone timeZone) {
 setDefaultLocale(locale);
 setDefaultTimeZone(timeZone);
}
```
要么自己传 Locale 进来，要么调用 Locale.getDefault() 方法获取默认的 Locale。

再来看 resolveLocale 方法：
```
@Override
public Locale resolveLocale(HttpServletRequest request) {
 Locale locale = getDefaultLocale();
 if (locale == null) {
  locale = Locale.getDefault();
 }
 return locale;
}
```
需要注意的是它的 setLocaleContext 方法，直接抛异常出来，也就意味着 Locale 在后期不能被修改。
```
@Override
public void setLocaleContext( HttpServletRequest request, @Nullable HttpServletResponse response,
  @Nullable LocaleContext localeContext) {
 throw new UnsupportedOperationException("Cannot change fixed locale - use a different locale resolution strategy");
}
```

### CookieLocaleResolver
CookieLocaleResolver 和 SessionLocaleResolver 比较类似，只不过存储介质变成了 Cookie