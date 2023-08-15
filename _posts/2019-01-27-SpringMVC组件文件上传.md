---
layout: post
categories: [SpringMV]
description: none
keywords: Spring
---
# SpringMVC组件文件上传


## 文件解析方案
对于上传文件的请求，SpringMVC 中目前共有两种不同的解析方案：
- StandardServletMultipartResolver
- CommonsMultipartResolver
StandardServletMultipartResolver 支持 Servlet3.0 中标准的文件上传方案，使用非常简单；CommonsMultipartResolver 则需要结合 Apache Commons fileupload 组件一起使用，这种方式兼容低版本的 Servlet。

### StandardServletMultipartResolver
先来回顾下 StandardServletMultipartResolver 的用法。

使用 StandardServletMultipartResolver，可以直接通过 HttpServletRequest 自带的 getPart 方法获取上传文件并保存，这是一种标准的操作方式，这种方式也不用添加任何额外的依赖，只需要确保 Servlet 的版本在 3.0 之上即可。

首先我们需要为 Servlet 配置 multipart-config，哪个 Servlet 负责处理上传文件，就为哪个 Servlet 配置 multipart-config。在 SpringMVC 中，我们的请求都是通过 DispatcherServlet 进行分发的，所以我们就为 DispatcherServlet 配置 multipart-config。

配置方式如下：
```
<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</serv
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring-servlet.xml</param-value>
    </init-param>
    <multipart-config>
        <location>/tmp</location>
        <max-file-size>1024</max-file-size>
        <max-request-size>10240</max-request-size>
    </multipart-config>
</servlet>
<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```
然后在 SpringMVC 的配置文件中提供一个 StandardServletMultipartResolver 实例，注意该实例的 id 必须为 multipartResolver

```
<bean class="org.springframework.web.multipart.support.StandardServletMultipartResolver" id="multipartResolver">
</bean>
```

配置完成后，我们就可以开发一个文件上传接口了，如下：
```
@RestController
public class FileUploadController {
    SimpleDateFormat sdf = new SimpleDateFormat("/yyyy/MM/dd/");

    @PostMapping("/upload")
    public String fileUpload(MultipartFile file, HttpServletRequest req) {
        String format = sdf.format(new Date());
        String realPath = req.getServletContext().getRealPath("/img") + format;
        File folder = new File(realPath);
        if (!folder.exists()) {
            folder.mkdirs();
        }
        String oldName = file.getOriginalFilename();
        String newName = UUID.randomUUID().toString() + oldName.substring(oldName.lastIndexOf("."));
        try {
            file.transferTo(new File(folder, newName));
            return req.getScheme() + "://" + req.getRemoteHost() + ":" + req.getServerPort() + "/img" + format + newName;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "error";
    }

    @PostMapping("/upload2")
    public String fileUpload2(HttpServletRequest req) throws IOException, ServletException {
        StandardServletMultipartResolver resolver = new StandardServletMultipartResolver();
        MultipartFile file = resolver.resolveMultipart(req).getFile("file");
        String format = sdf.format(new Date());
        String realPath = req.getServletContext().getRealPath("/img") + format;
        File folder = new File(realPath);
        if (!folder.exists()) {
            folder.mkdirs();
        }
        String oldName = file.getOriginalFilename();
        String newName = UUID.randomUUID().toString() + oldName.substring(oldName.lastIndexOf("."));
        try {
            file.transferTo(new File(folder, newName));
            return req.getScheme() + "://" + req.getRemoteHost() + ":" + req.getServerPort() + "/img" + format + newName;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "error";
    }

    @PostMapping("/upload3")
    public String fileUpload3(HttpServletRequest req) throws IOException, ServletException {
        String other_param = req.getParameter("other_param");
        System.out.println("other_param = " + other_param);
        String format = sdf.format(new Date());
        String realPath = req.getServletContext().getRealPath("/img") + format;
        File folder = new File(realPath);
        if (!folder.exists()) {
            folder.mkdirs();
        }
        Part filePart = req.getPart("file");
        String oldName = filePart.getSubmittedFileName();
        String newName = UUID.randomUUID().toString() + oldName.substring(oldName.lastIndexOf("."));
        try {
            filePart.write(realPath + newName);
            return req.getScheme() + "://" + req.getRemoteHost() + ":" + req.getServerPort() + "/img" + format + newName;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "error";
    }
}
```
我这里一共提供了三个文件上传接口，其实最终都是通过 StandardServletMultipartResolver 进行处理的。
第一个接口是我们在 SpringMVC 框架中常见的一种文件上传处理方式，直接在参数中写上 MultipartFile，这个 MultipartFile 其实就是从当前请求中解析出来的，具体负责参数解析工作的就是 RequestParamMethodArgumentResolver。

第二个接口其实是一种古老的文件上传实现方案，参数就是普通的 HttpServletRequest，然后在参数里边，我们再手动利用 StandardServletMultipartResolver 实例进行解析（这种情况可以不用自己 new 一个 StandardServletMultipartResolver 实例，直接将 Spring 容器中的注入进来即可）。

第三个接口我们就利用了 Servlet3.0 的 API，调用 getPart 获取文件，然后再调用对象的 write 方法将文件写出去即可。

大致上一看，感觉办法还挺多，其实仔细看，万变不离其宗，一会我们看完源码，相信小伙伴们还能变化出更多写法。

## CommonsMultipartResolver
CommonsMultipartResolver 估计很多人都比较熟悉，这个兼容性很好，就是有点过时了。使用 CommonsMultipartResolver 需要我们首先引入 commons-fileupload 依赖：
```
<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.4</version>
</dependency>
```
然后在 SpringMVC 的配置文件中提供 CommonsMultipartResolver 实例，如下：
```
<bean class="org.springframework.web.multipart.commons.CommonsMultipartResolver" id="multipartResolver">-->
</bean>
```
接下来开发文件上传接口就行了：
```
@PostMapping("/upload")
public String fileUpload(MultipartFile file, HttpServletRequest req) {
    String format = sdf.format(new Date());
    String realPath = req.getServletContext().getRealPath("/img") + format;
    File folder = new File(realPath);
    if (!folder.exists()) {
        folder.mkdirs();
    }
    String oldName = file.getOriginalFilename();
    String newName = UUID.randomUUID().toString() + oldName.substring(oldName.lastIndexOf("."));
    try {
        file.transferTo(new File(folder, newName));
        return req.getScheme() + "://" + req.getRemoteHost() + ":" + req.getServerPort() + "/img" + format + newName;
    } catch (IOException e) {
        e.printStackTrace();
    }
    return "error";
}
```

## StandardServletMultipartResolver
不废话，直接来看看源码：
```
public class StandardServletMultipartResolver implements MultipartResolver {
 private boolean resolveLazily = false;
 public void setResolveLazily(boolean resolveLazily) {
  this.resolveLazily = resolveLazily;
 }
 @Override
 public boolean isMultipart(HttpServletRequest request) {
  return StringUtils.startsWithIgnoreCase(request.getContentType(), "multipart/");
 }
 @Override
 public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException {
  return new StandardMultipartHttpServletRequest(request, this.resolveLazily);
 }
 @Override
 public void cleanupMultipart(MultipartHttpServletRequest request) {
  if (!(request instanceof AbstractMultipartHttpServletRequest) ||
    ((AbstractMultipartHttpServletRequest) request).isResolved()) {
   try {
    for (Part part : request.getParts()) {
     if (request.getFile(part.getName()) != null) {
      part.delete();
     }
    }
   }
   catch (Throwable ex) {
   }
  }
 }

}
```
这里满打满算就四个方法，其中一个还是 set 方法，我们来看另外三个功能性方法：
- isMultipart
这个方法主要是用来判断当前请求是不是文件上传请求，这里的判断思路很简单，就看请求的 content-type 是不是以 multipart/ 开头，如果是，则这就是一个文件上传请求，否则就不是文件上传请求。
- resolveMultipart
这个方法负责将当前请求封装一个 StandardMultipartHttpServletRequest 对象然后返回。
- cleanupMultipart
这个方法负责善后，主要完成了缓存的清理工作。

在这个过程中涉及到 StandardMultipartHttpServletRequest 对象，我们也来稍微说一下：
```
public StandardMultipartHttpServletRequest(HttpServletRequest request, boolean lazyParsing)
  throws MultipartException {
 super(request);
 if (!lazyParsing) {
  parseRequest(request);
 }
}
private void parseRequest(HttpServletRequest request) {
 try {
  Collection<Part> parts = request.getParts();
  this.multipartParameterNames = new LinkedHashSet<>(parts.size());
  MultiValueMap<String, MultipartFile> files = new LinkedMultiValueMap<>(parts.size());
  for (Part part : parts) {
   String headerValue = part.getHeader(HttpHeaders.CONTENT_DISPOSITION);
   ContentDisposition disposition = ContentDisposition.parse(headerValue);
   String filename = disposition.getFilename();
   if (filename != null) {
    if (filename.startsWith("=?") && filename.endsWith("?=")) {
     filename = MimeDelegate.decode(filename);
    }
    files.add(part.getName(), new StandardMultipartFile(part, filename));
   }
   else {
    this.multipartParameterNames.add(part.getName());
   }
  }
  setMultipartFiles(files);
 }
 catch (Throwable ex) {
  handleParseFailure(ex);
 }
}
```
StandardMultipartHttpServletRequest 对象在构建的过程中，会自动进行请求解析，调用 getParts 方法获取所有的项，然后进行判断，将文件和普通参数分别保存下来备用。

## CommonsMultipartResolver
再来看 CommonsMultipartResolver。

先来看它的 isMultipart 方法：
```
@Override
public boolean isMultipart(HttpServletRequest request) {
 return ServletFileUpload.isMultipartContent(request);
}
public static final boolean isMultipartContent(
        HttpServletRequest request) {
    if (!POST_METHOD.equalsIgnoreCase(request.getMethod())) {
        return false;
    }
    return FileUploadBase.isMultipartContent(new ServletRequestContext(request));
}
```
ServletFileUpload.isMultipartContent 方法其实就在我们引入的 commons-fileupload 包中。它的判断逻辑分两步：首先检查是不是 POST 请求，然后检查 content-type 是不是以 multipart/ 开始。

再来看它的 resolveMultipart 方法：
```
@Override
public MultipartHttpServletRequest resolveMultipart(final HttpServletRequest request) throws MultipartException {
 if (this.resolveLazily) {
  return new DefaultMultipartHttpServletRequest(request) {
   @Override
   protected void initializeMultipart() {
    MultipartParsingResult parsingResult = parseRequest(request);
    setMultipartFiles(parsingResult.getMultipartFiles());
    setMultipartParameters(parsingResult.getMultipartParameters());
    setMultipartParameterContentTypes(parsingResult.getMultipartParameterContentTypes());
   }
  };
 }
 else {
  MultipartParsingResult parsingResult = parseRequest(request);
  return new DefaultMultipartHttpServletRequest(request, parsingResult.getMultipartFiles(),
    parsingResult.getMultipartParameters(), parsingResult.getMultipartParameterContentTypes());
 }
}
```
根据 resolveLazily 属性值，选择两种不同的策略将当前对象重新构建成一个 DefaultMultipartHttpServletRequest 对象。如果 resolveLazily 为 true，则在 initializeMultipart 方法中进行请求解析，否则先解析，再构建 DefaultMultipartHttpServletRequest 对象。

具体的解析方法如下：
```
protected MultipartParsingResult parseRequest(HttpServletRequest request) throws MultipartException {
 String encoding = determineEncoding(request);
 FileUpload fileUpload = prepareFileUpload(encoding);
 try {
  List<FileItem> fileItems = ((ServletFileUpload) fileUpload).parseRequest(request);
  return parseFileItems(fileItems, encoding);
 }
 catch (FileUploadBase.SizeLimitExceededException ex) {
  //...
 }
}
protected MultipartParsingResult parseFileItems(List<FileItem> fileItems, String encoding) {
 MultiValueMap<String, MultipartFile> multipartFiles = new LinkedMultiValueMap<>();
 Map<String, String[]> multipartParameters = new HashMap<>();
 Map<String, String> multipartParameterContentTypes = new HashMap<>();
 for (FileItem fileItem : fileItems) {
  if (fileItem.isFormField()) {
   String value;
   String partEncoding = determineEncoding(fileItem.getContentType(), encoding);
   try {
    value = fileItem.getString(partEncoding);
   }
   catch (UnsupportedEncodingException ex) {
    value = fileItem.getString();
   }
   String[] curParam = multipartParameters.get(fileItem.getFieldName());
   if (curParam == null) {
    multipartParameters.put(fileItem.getFieldName(), new String[] {value});
   }
   else {
    String[] newParam = StringUtils.addStringToArray(curParam, value);
    multipartParameters.put(fileItem.getFieldName(), newParam);
   }
   multipartParameterContentTypes.put(fileItem.getFieldName(), fileItem.getContentType());
  }
  else {
   CommonsMultipartFile file = createMultipartFile(fileItem);
   multipartFiles.add(file.getName(), file);
  }
 }
 return new MultipartParsingResult(multipartFiles, multipartParameters, multipartParameterContentTypes);
}
```
这里的解析就是首先获取到 FileItem 集合，然后调用 parseFileItems 方法进行进一步的解析。在进一步的解析中，会首先判断这是文件还是普通参数，如果是普通参数，则保存到 multipartParameters 中，具体保存过程中还会判断是否为数组，然后再将参数的 ContentType 保存到 multipartParameterContentTypes 中，文件则保存到 multipartFiles 中，最后由三个 Map 构成一个 MultipartParsingResult 对象并返回。

至此，StandardServletMultipartResolver 和 CommonsMultipartResolver 源码就和大家说完了，可以看到，还是比较容易的。

## 解析流程
最后，我们再来梳理一下解析流程。

以如下接口为例（因为在实际开发中一般都是通过如下方式上传文件）：
```
@PostMapping("/upload")
public String fileUpload(MultipartFile file, HttpServletRequest req) {
    String format = sdf.format(new Date());
    String realPath = req.getServletContext().getRealPath("/img") + format;
    File folder = new File(realPath);
    if (!folder.exists()) {
        folder.mkdirs();
    }
    String oldName = file.getOriginalFilename();
    String newName = UUID.randomUUID().toString() + oldName.substring(oldName.lastIndexOf("."));
    try {
        file.transferTo(new File(folder, newName));
        return req.getScheme() + "://" + req.getRemoteHost() + ":" + req.getServerPort() + "/img" + format + newName;
    } catch (IOException e) {
        e.printStackTrace();
    }
    return "error";
}
```
这里 MultipartFile 对象主要就是在参数解析器中获取的。这里涉及到的参数解析器是 RequestParamMethodArgumentResolver。

在 RequestParamMethodArgumentResolver#resolveName 方法中有如下一行代码：
```
if (servletRequest != null) {
 Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
 if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
  return mpArg;
 }
}
```
这个方法会进行请求解析，返回 MultipartFile 对象或者 MultipartFile 数组。

```
@Nullable
public static Object resolveMultipartArgument(String name, MethodParameter parameter, HttpServletRequest request)
  throws Exception {
 MultipartHttpServletRequest multipartRequest =
   WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class);
 boolean isMultipart = (multipartRequest != null || isMultipartContent(request));
 if (MultipartFile.class == parameter.getNestedParameterType()) {
  if (!isMultipart) {
   return null;
  }
  if (multipartRequest == null) {
   multipartRequest = new StandardMultipartHttpServletRequest(request);
  }
  return multipartRequest.getFile(name);
 }
 else if (isMultipartFileCollection(parameter)) {
  if (!isMultipart) {
   return null;
  }
  if (multipartRequest == null) {
   multipartRequest = new StandardMultipartHttpServletRequest(request);
  }
  List<MultipartFile> files = multipartRequest.getFiles(name);
  return (!files.isEmpty() ? files : null);
 }
 else if (isMultipartFileArray(parameter)) {
  if (!isMultipart) {
   return null;
  }
  if (multipartRequest == null) {
   multipartRequest = new StandardMultipartHttpServletRequest(request);
  }
  List<MultipartFile> files = multipartRequest.getFiles(name);
  return (!files.isEmpty() ? files.toArray(new MultipartFile[0]) : null);
 }
 else if (Part.class == parameter.getNestedParameterType()) {
  if (!isMultipart) {
   return null;
  }
  return request.getPart(name);
 }
 else if (isPartCollection(parameter)) {
  if (!isMultipart) {
   return null;
  }
  List<Part> parts = resolvePartList(request, name);
  return (!parts.isEmpty() ? parts : null);
 }
 else if (isPartArray(parameter)) {
  if (!isMultipart) {
   return null;
  }
  List<Part> parts = resolvePartList(request, name);
  return (!parts.isEmpty() ? parts.toArray(new Part[0]) : null);
 }
 else {
  return UNRESOLVABLE;
 }
}
```
首先获取 multipartRequest 对象，然后再从中获取文件或者文件数组。如果我们使用 StandardServletMultipartResolver 做文件上传，这里获取到的 multipartRequest 就是 StandardMultipartHttpServletRequest；如果我们使用 CommonsMultipartResolver 做文件上传，这里获取到的 multipartRequest 就是 DefaultMultipartHttpServletRequest。












