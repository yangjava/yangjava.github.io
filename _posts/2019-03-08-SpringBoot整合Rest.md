---
layout: post
categories: SpringBoot
description: none
keywords: SpringBoot
---
# SpringBoot整合Rest
RestTemplate是一个同步的web http客户端请求模板工具。是基于spring框架的底层的一个知识点。

RestTemplate默认是使用HttpURLConnection，可以通过构造方法替换底层的执行引擎，常见的引擎又HttpClient、Netty、OkHttp
如果要想替换直接如构造方法所示即可
```java
//底层执行引擎
RestTemplate template=new RestTemplate();

//底层执行引擎ClientHttp
RestTemplate template=new RestTemplate(ClientHttpRequestFactory requestFactory);
```
## 什么是RestTemplate
RestTemplate 是从 Spring3.0 开始支持的一个 HTTP 请求工具，它提供了常见的REST请求方案的模版，例如 GET 请求、POST 请求、PUT 请求、DELETE 请求以及一些通用的请求执行方法 exchange 以及 execute。
RestTemplate 继承自 InterceptingHttpAccessor 并且实现了 RestOperations 接口，其中 RestOperations 接口定义了基本的 RESTful 操作，这些操作在 RestTemplate 中都得到了实现。
传统情况下在java代码里访问Restful服务，一般使用Apache的HttpClient。不过此种方法使用起来太繁琐。Spring提供了一种简单便捷的模板类RestTemplate来进行操作：

## SpringBoot引入RestTemplate

```java

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.ClientHttpRequestFactory;
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.http.converter.StringHttpMessageConverter;
import org.springframework.web.client.RestTemplate;

import java.nio.charset.Charset;

@Configuration
public class RestConfig {
    /**
     * 创建HTTP客户端工厂
     *
     * @throws Exception
     */
    @Bean(name = "clientHttpRequestFactory")
    public ClientHttpRequestFactory clientHttpRequestFactory() throws Exception {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        // 数据读取超时时间，即SocketTimeout
        factory.setReadTimeout(180000);
        // 连接超时
        factory.setConnectTimeout(5000);
        return factory;
    }

    /**
     * 初始化RestTemplate,并加入spring的Bean工厂，由spring统一管理
     */
    @Bean(name = "restTemplate")
    public RestTemplate restTemplate(ClientHttpRequestFactory clientHttpRequestFactory) {
        RestTemplate restTemplate =  new RestTemplate(clientHttpRequestFactory);
        restTemplate.getMessageConverters().set(1, new StringHttpMessageConverter(Charset.forName("UTF-8")));
        return restTemplate;
    }

}
```
**RestTemplate 注入失败问题**??
解决方法：
1. 在项目的启动类下创建resttemplate
```java
   @Autowired
    private RestTemplateBuilder builder;
    @Bean
    public RestTemplate restTemplate() {
        return builder.build();
    }```
```
2. 创建配置文件
```java
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        RestTemplate restTemplate = builder.build();
        restTemplate.getMessageConverters().add(new MappingJackson2HttpMessageConverter());
        return restTemplate;
    }
```

使用RestTemplate处理Get请求
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;
 
/**
 * 测试restTemplate
 */
@RestController
@RequestMapping("/test")
public class HttpController{
    
    @Autowired
    private RestTemplate restTemplate;
    
    /**
     * 测试
     */
    @RequestMapping(value = "/http", method = RequestMethod.GET)
    public String http() {
        String id = "?";
        String name = "?";
        //参数
        MultiValueMap<String, Object> param = new LinkedMultiValueMap<String, Object>();
        param.add("id", id);
        param.add("name", name);
        
        //请求头
        HttpHeaders headers = new HttpHeaders();
        headers.add("accessToken", "?");
        
        //封装请求头
        HttpEntity<MultiValueMap<String, Object>> formEntity = new HttpEntity<MultiValueMap<String, Object>>(headers);
        
        try {
            //访问地址
            String url = "http://localhost:8080/?";
            
            //1. 有参数，没有请求头，拼接方式
            String result1 = restTemplate.getForObject(url + "?id="+id+"&name="+name, String.class);
            //2. 有参数，没有请求头，占位符方式
            String result2 = restTemplate.getForObject(url + "?id={id}&name={name}", String.class, param);
            //3. 有请求头，没参数，result3.getBody()获取响应参数
            ResponseEntity<String> result3 = restTemplate.exchange(url, HttpMethod.GET, formEntity, String.class);
            //4. 有请求头，有参数，result4.getBody()获取响应参数
            ResponseEntity<String> result4 = restTemplate.exchange(url+"?id="+id+"&name="+name, HttpMethod.GET, formEntity, String.class);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "success";
    }
}

```
## RestTemplate 添加请求头headers和请求体body
```java
//headers & cookie
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON);
 
headers.add("basecret", config.getBasecret());
headers.add("baid", config.getBaid());
 
List<String> cookies = new ArrayList<>();
cookies.add("COOKIE_USER" + Strings.nullToEmpty(config.getCookie()));
headers.put(HttpHeaders.COOKIE, cookies);

```

## POST请求
1、 调用postForObject方法   
2、使用postForEntity方法  
3、调用exchange方法
postForObject和postForEntity方法的区别主要在于可以在postForEntity方法中设置header的属性，当需要指定header的属性值的时候，使用postForEntity方法。exchange方法和postForEntity类似，但是更灵活，exchange还可以调用get请求。使用这三种方法传递参数，Map不能定义为以下两种类型

Map<String, Object> paramMap = new HashMap<String, Object>();
Map<String, Object> paramMap = new LinkedHashMap<String, Object>();

把Map类型换成LinkedMultiValueMap后，参数成功传递到后台

MultiValueMap<String, Object> paramMap = new LinkedMultiValueMap<String, Object>();
```java

MultiValueMap<String, Object> paramMap = new LinkedMultiValueMap<String, Object>();
paramMap.add("dt", "20190225");
 
// 1、使用postForObject请求接口
String result = template.postForObject(url, paramMap, String.class);
 
// 2、使用postForEntity请求接口
HttpHeaders headers = new HttpHeaders();
HttpEntity<MultiValueMap<String, Object>> httpEntity = new HttpEntity<MultiValueMap<String, Object>>(paramMap,headers);
ResponseEntity<String> response2 = template.postForEntity(url, httpEntity, String.class);
 
// 3、使用exchange请求接口
ResponseEntity<String> response3 = template.exchange(url, HttpMethod.POST, httpEntity, String.class);

```
如果post请求体是个Json的表单
```java
       //JSONObject userInfo = new JSONObject();
        Map<String, Object> userInfo = Maps.newHashMap();
        userInfo.put("phone", accountForm.getPhone());
        userInfo.put("job", accountForm.getJob());
        userInfo.put("email", accountForm.getEmail());
 
        Map<String, Object> postBody = Maps.newHashMap();
        postBody.put("userInfo", userInfo);
 
        HttpEntity<Map> requestEntity = new HttpEntity<>(postBody, headers);
 
         try {
             ResponseEntity<String> result = restTemplate.postForEntity(config.getCreateWithAuthUrl(), requestEntity, String.class);
             JsonNode jsonNode = JsonUtils.toJsonNode(result.getBody());
             if (jsonNode.get("errno").asInt() == 200 || jsonNode.get("errno").asInt() == 505) {
                 return true;
             }
 
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }

```
## GET请求
如果是get请求，又想要把参数封装到map里面进行传递的话，Map需要使用HashMap，且url需要使用占位符，如下
```java
RestTemplate restTemplate2 = new RestTemplate();
String url = "http://127.0.0.1:8081/interact/getData?dt={dt}&ht={ht}";
       
// 封装参数，这里是HashMap
Map<String, Object> paramMap = new HashMap<String, Object>();
paramMap.put("dt", "20190225");
paramMap.put("ht", "10");
 
//1、使用getForObject请求接口
String result1 = template.getForObject(url, String.class, paramMap);
System.out.println("result1====================" + result1);
 
//2、使用exchange请求接口
HttpHeaders headers = new HttpHeaders();
headers.set("id", "lidy");
HttpEntity<MultiValueMap<String, Object>> httpEntity = new HttpEntity<MultiValueMap<String, Object>>(null,headers);
ResponseEntity<String> response2 = template.exchange(url, HttpMethod.GET, httpEntity, String.class,paramMap);

```