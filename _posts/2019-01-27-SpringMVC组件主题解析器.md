---
layout: post
categories: [SpringMV]
description: none
keywords: Spring
---
# SpringMVC组件主题解析
Theme，就是主题，点一下就给网站更换一个主题，相信大家都用过类似功能，这个其实和前面所说的国际化功能很像，代码其实也很像，今天我们就来捋一捋。

## 一键换肤
来做一个简单的需求，假设我的页面上有三个按钮，点击之后就能一键换肤

我们来看下这个需求怎么实现。

首先三个按钮分别对应了三个不同的样式，我们先把这三个不同的样式定义出来，分别如下：
blue.css：
```
body{
background-color: #05e1ff;
}
```

green.css：
```
body{
background-color: #aaff9c;
}
```

red.css：
```
body{
background-color: #ff0721;
}
```

主题的定义，往往是一组样式，因此我们一般都是在一个 properties 文件中将同一主题的样式配置在一起，这样方便后期加载。

所以接下来我们在 resources 目录下新建 theme 目录，然后在 theme 目录中创建三个文件，内容如下：

blue.properties：
```
index.body=/css/blue.css
```

green.properties：
```
index.body=/css/green.css
```

red.properties：
```
index.body=/css/red.css
```

在不同的 properties 配置文件中引入不同的样式，但是样式定义的 key 都是 index.body，这样方便后期在页面中引入。

接下来在 SpringMVC 容器中配置三个 Bean，如下：
```
<mvc:interceptors>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <bean class="org.springframework.web.servlet.theme.ThemeChangeInterceptor">
            <property name="paramName" value="theme"/>
        </bean>
    </mvc:interceptor>
</mvc:interceptors>
<bean id="themeSource" class="org.springframework.ui.context.support.ResourceBundleThemeSource">
    <property name="basenamePrefix" value="theme."/>
</bean>
<bean id="themeResolver" class="org.springframework.web.servlet.theme.SessionThemeResolver">
    <property name="defaultThemeName" value="blue"/>
</bean>
```

首先配置拦截器 ThemeChangeInterceptor，这个拦截器用来解析主题参数，参数的 key 为 theme，例如请求地址是 /index?theme=blue，该拦截器就会自动设置系统主题为 blue。当然也可以不配置拦截器，如果不配置的话，就可以单独提供一个修改主题的接口，然后手动修改主题，类似下面这样：

```
@Autowired
private ThemeResolver themeResolver;
@RequestMapping(path = "/01/{theme}",method = RequestMethod.GET)
public String theme1(@PathVariable("theme") String themeStr, HttpServletRequest request, HttpServletResponse response){
    themeResolver.setThemeName(request,response, themeStr);
    return "redirect:/01";
}
```
themeStr 就是新的主题名称，将其配置给 themeResolver 即可。

接下来配置 ResourceBundleThemeSource，这个 Bean 主要是为了加载主题文件，需要配置一个 basenamePrefix 属性，如果我们的主题文件放在文件夹中，这个 basenamePrefix 的值就是 文件夹名称.。

接下来配置主题解析器，主题解析器有三种，分别是 CookieThemeResolver、FixedThemeResolver、SessionThemeResolver，这里我们使用的是 SessionThemeResolver，主题信息将被保存在 Session 中，只要 Session 不变，主题就一直有效。

配置完成后，我们再来提供一个测试页面，如下：
```
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
    <link rel="stylesheet" href="<spring:theme code="index.body" />" >
</head>
<body>
<div>
    一键切换主题：<br/>
    <a href="/index?theme=blue">托帕蓝</a>
    <a href="/index?theme=red">多巴胺红</a>
    <a href="/index?theme=green">石竹青</a>
</div>
<br/>
</body>
</html>
```

最关键的是：
```
<link rel="stylesheet" href="<spring:theme code="index.body" />" >
```
css 样式不直接写，而是引用我们在 properties 文件中定义的 index.body，这样将根据当前主题加载不同的 css 文件。

最后再提供一个处理器，如下：
```
@GetMapping(path = "/index")
public  String getPage(){
    return "index";
}
```

## 原理分析
主题这块涉及到的东西主要就是主题解析器，主题解析器和我们前面所说的国际化的解析器非常类似，但是比它更简单，我们一起来分析下。

先来看下 ThemeResolver 接口：
```
public interface ThemeResolver {
 String resolveThemeName(HttpServletRequest request);
 void setThemeName(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable String themeName);
}
```
这个接口中就两个方法：

resolveThemeName：从当前请求中解析出主题的名字。
setThemeName：设置当前主题。

ThemeResolver 主要有三个实现类

### CookieThemeResolver
直接上源码吧：
```
@Override
public String resolveThemeName(HttpServletRequest request) {
 String themeName = (String) request.getAttribute(THEME_REQUEST_ATTRIBUTE_NAME);
 if (themeName != null) {
  return themeName;
 }
 String cookieName = getCookieName();
 if (cookieName != null) {
  Cookie cookie = WebUtils.getCookie(request, cookieName);
  if (cookie != null) {
   String value = cookie.getValue();
   if (StringUtils.hasText(value)) {
    themeName = value;
   }
  }
 }
 if (themeName == null) {
  themeName = getDefaultThemeName();
 }
 request.setAttribute(THEME_REQUEST_ATTRIBUTE_NAME, themeName);
 return themeName;
}
@Override
public void setThemeName(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable String themeName) {
 if (StringUtils.hasText(themeName)) {
  request.setAttribute(THEME_REQUEST_ATTRIBUTE_NAME, themeName);
  addCookie(response, themeName);
 } else {
  request.setAttribute(THEME_REQUEST_ATTRIBUTE_NAME, getDefaultThemeName());
  removeCookie(response);
 }
}
```
先来看 resolveThemeName 方法：

首先会尝试直接从请求中获取主题名称，如果获取到了，就直接返回。
如果第一步没有获取到主题名称，接下来就尝试从 Cookie 中获取主题名称，Cookie 也是从当前请求中提取，利用 WebUtils 工具进行解析，如果解析到了主题名称，就赋值给 themeName 变量。
如果前面没有获取到主题名称，就使用默认的主题名称，开发者可以自行配置默认的主题名称，如果不配置，就是 theme。
将解析出来的 theme 保存到 request 中，以备后续使用。
再来看 setThemeName 方法：

如果存在 themeName 就进行设置，同时将 themeName 添加到 Cookie 中。
如果不存在 themeName，就设置一个默认的主题名，同时从 response 中移除 Cookie。
可以看到，整个实现思路还是非常简单的。

### AbstractThemeResolver
```
public abstract class AbstractThemeResolver implements ThemeResolver {
 public static final String ORIGINAL_DEFAULT_THEME_NAME = "theme";
 private String defaultThemeName = ORIGINAL_DEFAULT_THEME_NAME;
 public void setDefaultThemeName(String defaultThemeName) {
  this.defaultThemeName = defaultThemeName;
 }
 public String getDefaultThemeName() {
  return this.defaultThemeName;
 }
}
```
AbstractThemeResolver 主要提供了配置默认主题的能力。

### FixedThemeResolver
```
public class FixedThemeResolver extends AbstractThemeResolver {

 @Override
 public String resolveThemeName(HttpServletRequest request) {
  return getDefaultThemeName();
 }

 @Override
 public void setThemeName(
   HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable String themeName) {

  throw new UnsupportedOperationException("Cannot change theme - use a different theme resolution strategy");
 }

}
```
FixedThemeResolver 就是使用默认的主题名称，并且不允许修改主题。

### SessionThemeResolver
```
public class SessionThemeResolver extends AbstractThemeResolver {
 public static final String THEME_SESSION_ATTRIBUTE_NAME = SessionThemeResolver.class.getName() + ".THEME";
 @Override
 public String resolveThemeName(HttpServletRequest request) {
  String themeName = (String) WebUtils.getSessionAttribute(request, THEME_SESSION_ATTRIBUTE_NAME);
  return (themeName != null ? themeName : getDefaultThemeName());
 }
 @Override
 public void setThemeName(
   HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable String themeName) {
  WebUtils.setSessionAttribute(request, THEME_SESSION_ATTRIBUTE_NAME,
    (StringUtils.hasText(themeName) ? themeName : null));
 }
}
```
resolveThemeName：从 session 中取出主题名称并返回，如果 session 中的主题名称为 null，就返回默认的主题名称。
setThemeName：将主题配置到请求中。
不想多说，因为很简单。

## ThemeChangeInterceptor
最后我们再来看一看 ThemeChangeInterceptor 拦截器，这个拦截器会自动从请求中提取出主题参数，并设置到请求中，核心部分在 preHandle 方法中：
```
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
  throws ServletException {
 String newTheme = request.getParameter(this.paramName);
 if (newTheme != null) {
  ThemeResolver themeResolver = RequestContextUtils.getThemeResolver(request);
  if (themeResolver == null) {
   throw new IllegalStateException("No ThemeResolver found: not in a DispatcherServlet request?");
  }
  themeResolver.setThemeName(request, response, newTheme);
 }
 return true;
}
```
从请求中提取出 theme 参数，并设置到 themeResolver 中。