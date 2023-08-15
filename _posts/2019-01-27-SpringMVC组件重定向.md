---
layout: post
categories: [SpringMV]
description: none
keywords: Spring
---
# SpringMVC组件重定向
flashMap，专门用来解决重定向时参数的传递问题。

## flashMap
在重定向时，如果需要传递参数，但是又不想放在地址栏中，我们就可以通过 flashMap 来传递参数，松哥先来一个简单的例子大家看看效果：

首先我们定义一个简单的页面，里边就一个 post 请求提交按钮，如下：
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<form action="/order">
    <input type="submit" value="提交">
</form>
</body>
</html>
```
然后在服务端接收该请求，并完成重定向：
```
@Controller
public class OrderController {
    @PostMapping("/order")
    public String order(HttpServletRequest req) {
        FlashMap flashMap = (FlashMap) req.getAttribute(DispatcherServlet.OUTPUT_FLASH_MAP_ATTRIBUTE);
        flashMap.put("name", "江南一点雨");
        return "redirect:/orderlist";
    }

    @GetMapping("/orderlist")
    @ResponseBody
    public String orderList(Model model) {
        return (String) model.getAttribute("name");
    }
}
```
首先在 order 接口中，获取到 flashMap 属性，然后存入需要传递的参数，这些参数最终会被 SpringMVC 自动放入重定向接口的 Model 中，这样我们在 orderlist 接口中，就可以获取到该属性了。

当然，这是一个比较粗糙的写法，我们还可以通过 RedirectAttributes 来简化这一步骤：
```
@Controller
public class OrderController {
    @PostMapping("/order")
    public String order(RedirectAttributes attr) {
        attr.addFlashAttribute("site", "www.javaboy.org");
        attr.addAttribute("name", "微信公众号：江南一点雨");
        return "redirect:/orderlist";
    }

    @GetMapping("/orderlist")
    @ResponseBody
    public String orderList(Model model) {
        return (String) model.getAttribute("site");
    }
}
```
RedirectAttributes 中有两种添加参数的方式：

addFlashAttribute：将参数放到 flashMap 中。
addAttribute：将参数放到 URL 地址中。
经过前面的讲解，现在小伙伴们应该大致明白了 flashMap 的作用了，就是在你进行重定向的时候，不通过地址栏传递参数。

很多小伙伴可能会有疑问，重定向其实就是浏览器发起了一个新的请求，这新的请求怎么就获取到上一个请求保存的参数呢？这我们就要来看看 SpringMVC 的源码了。


## 源码分析
首先这里涉及到一个关键类叫做 FlashMapManager，如下：
```
public interface FlashMapManager {
 @Nullable
 FlashMap retrieveAndUpdate(HttpServletRequest request, HttpServletResponse response);
 void saveOutputFlashMap(FlashMap flashMap, HttpServletRequest request, HttpServletResponse response);
}
```
两个方法含义一眼就能看出来：
- retrieveAndUpdate：这个方法用来恢复参数，并将恢复过的的参数和超时的参数从保存介质中删除。
- saveOutputFlashMap：将参数保存保存起来。
  FlashMapManager 的实现类如下：

从这个继承类中，我们基本上就能确定默认的保存介质时 session。具体的保存逻辑则是在 AbstractFlashMapManager 类中。

整个参数传递的过程可以分为三大步：

第一步，首先我们将参数设置到 outputFlashMap 中，有两种设置方式：我们前面的代码 req.getAttribute(DispatcherServlet.OUTPUT_FLASH_MAP_ATTRIBUTE) 就是直接获取 outputFlashMap 对象然后把参数放进去；第二种方式就是通过在接口中添加 RedirectAttributes 参数，然后把需要传递的参数放入 RedirectAttributes 中，这样当处理器处理完毕后，会自动将其设置到 outputFlashMap 中，具体逻辑在 RequestMappingHandlerAdapter#getModelAndView 方法中：
```
private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
  ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {
 //省略...
 if (model instanceof RedirectAttributes) {
  Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
  HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
  if (request != null) {
   RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
  }
 }
 return mav;
}
```
可以看到，如果 model 是 RedirectAttributes 的实例的话，则通过 getOutputFlashMap 方法获取到 outputFlashMap 属性，然后相关的属性设置进去。

这是第一步，就是将需要传递的参数，先保存到 flashMap 中。

第二步，重定向对应的视图是 RedirectView，在它的 renderMergedOutputModel 方法中，会调用 FlashMapManager 的 saveOutputFlashMap 方法，将 outputFlashMap 保存到 session 中，如下：
```
protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request,
  HttpServletResponse response) throws IOException {
 String targetUrl = createTargetUrl(model, request);
 targetUrl = updateTargetUrl(targetUrl, model, request, response);
 // Save flash attributes
 RequestContextUtils.saveOutputFlashMap(targetUrl, request, response);
 // Redirect
 sendRedirect(request, response, targetUrl, this.http10Compatible);
}
```
RequestContextUtils.saveOutputFlashMap 方法最终就会调用到 FlashMapManager 的 saveOutputFlashMap 方法，将 outputFlashMap 保存下来。我们来大概看一下保存逻辑：
```
public final void saveOutputFlashMap(FlashMap flashMap, HttpServletRequest request, HttpServletResponse response) {
 if (CollectionUtils.isEmpty(flashMap)) {
  return;
 }
 String path = decodeAndNormalizePath(flashMap.getTargetRequestPath(), request);
 flashMap.setTargetRequestPath(path);
 flashMap.startExpirationPeriod(getFlashMapTimeout());
 Object mutex = getFlashMapsMutex(request);
 if (mutex != null) {
  synchronized (mutex) {
   List<FlashMap> allFlashMaps = retrieveFlashMaps(request);
   allFlashMaps = (allFlashMaps != null ? allFlashMaps : new CopyOnWriteArrayList<>());
   allFlashMaps.add(flashMap);
   updateFlashMaps(allFlashMaps, request, response);
  }
 }
 else {
  List<FlashMap> allFlashMaps = retrieveFlashMaps(request);
  allFlashMaps = (allFlashMaps != null ? allFlashMaps : new ArrayList<>(1));
  allFlashMaps.add(flashMap);
  updateFlashMaps(allFlashMaps, request, response);
 }
}
```
其实这里的逻辑也很简单，保存之前会给 flashMap 设置两个属性，一个是重定向的 url 地址，另一个则是过期时间，过期时间默认 180 秒，这两个属性在第三步加载 flashMap 的时候会用到。然后将 flashMap 放入集合中，并调用 updateFlashMaps 方法存入 session 中。

第三步，当重定向请求到达 DispatcherServlet#doService 方法后，此时会调用 FlashMapManager#retrieveAndUpdate 方法从 Session 中获取 outputFlashMap 并设置到 Request 属性中备用（最终会被转化到 Model 中的属性），相关代码如下：
```
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
 //省略...
 if (this.flashMapManager != null) {
  FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
  if (inputFlashMap != null) {
   request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
  }
  request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
  request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
 }
 //省略...
}
```

注意这里获取出来的 outputFlashMap 换了一个名字，变成了 inputFlashMap，其实是同一个东西。

我们可以大概看一下获取的逻辑 AbstractFlashMapManager#retrieveAndUpdate：
```
public final FlashMap retrieveAndUpdate(HttpServletRequest request, HttpServletResponse response) {
 List<FlashMap> allFlashMaps = retrieveFlashMaps(request);
 if (CollectionUtils.isEmpty(allFlashMaps)) {
  return null;
 }
 List<FlashMap> mapsToRemove = getExpiredFlashMaps(allFlashMaps);
 FlashMap match = getMatchingFlashMap(allFlashMaps, request);
 if (match != null) {
  mapsToRemove.add(match);
 }
 if (!mapsToRemove.isEmpty()) {
  Object mutex = getFlashMapsMutex(request);
  if (mutex != null) {
   synchronized (mutex) {
    allFlashMaps = retrieveFlashMaps(request);
    if (allFlashMaps != null) {
     allFlashMaps.removeAll(mapsToRemove);
     updateFlashMaps(allFlashMaps, request, response);
    }
   }
  }
  else {
   allFlashMaps.removeAll(mapsToRemove);
   updateFlashMaps(allFlashMaps, request, response);
  }
 }
 return match;
}
```
- 首先调用 retrieveFlashMaps 方法从 session 中获取到所有的 FlashMap。
- 调用 getExpiredFlashMaps 方法获取所有过期的 FlashMap，FlashMap 默认的过期时间是 180s。
- 获取和当前请求匹配的 getMatchingFlashMap，具体的匹配逻辑就两点：重定向地址要和当前请求地址相同；预设参数要相同。一般来说我们不需要配置预设参数，所以这一条可以忽略。如果想要设置，则首先给 flashMap 设置，像这样：flashMap.addTargetRequestParam("aa", "bb");，然后在重定向的地址栏也加上这个参数：return "redirect:/orderlist?aa=bb"; 即可。
- 将获取到的匹配的 FlashMap 对象放入 mapsToRemove 集合中（这个匹配到的 FlashMap 即将失效，放入集合中一会被清空）。
- 将 allFlashMaps 集合中的所有 mapsToRemove 数据清空，同时调用 updateFlashMaps 方法更新 session 中的 FlashMap。
- 最终将匹配到的 flashMap 返回。

这就是整个获取 flashMap 的方法，整体来看还是非常 easy 的，并没有什么难点。
