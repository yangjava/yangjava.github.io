---
layout: post
categories: [Teambition]
description: none
keywords: Teambition
---
# Teambition开发指南
Teambition是一个简单、高效的项目协作工具,是一款企业协作软件,很多企业用来作为任务跟踪管理和BUG管理工具.

## Teambition简介

2019年阿里收购了Teambition,不知道未来使用的企业会不会越来越多.

Teambition有导入导出功能,可以导入导出excel,csv文件,在工作中很方便.一般来说,除了一开始使用时会批量导入任务,其他时候很少使用批量导入,任务直接创建就可以了.

企业在定期(如每周)跟踪任务进度时,会经常需要批量导出.但批量导出的格式比较固定,有时候可能不符合我们的要求,如需要按老板指定的格式,或者需要将数据添加到其他平台上等.

Teambition提供了API接口,我们可以注册成为开发者,然后通过接口获取Teambition的数据,按照需求的格式保存和分析.

## Teambition API文档

Teambition API文档 [https://open.teambition.com/help/docs/5db8f7e77baeb50014957fc1](https://open.teambition.com/help/docs/5db8f7e77baeb50014957fc1)

Teambition V3 API文档 [https://open.teambition.com/redoc](https://open.teambition.com/redoc)
open API
两种使用方式：

公有云：默认地址 https://open.teambition.com/api
私有云：将 https://open.teambition.com/api 替换成 http://{私有化部署域名}/gateway， 其它部分不变。比如：

```text
//查询用户信息

公有云调用地址：
https://open.teambition.com/api/user/query

私有化部署调用地址：
http://{私有化环境域名}/gateway/user/query
```

## 相关接口

- teambition-项目相关API
  - 创建项目 api/project/create
  - 更新项目 api/project/update
  - 获取项目项列表 api/project/info
  - 添加项目成员 api/project/member/add
  
- teambition-任务相关API
  - 创建任务 api/task/create
  - 更新任务 api/task/update
  - 获取任务详情 api/task/query

## 注册Teambition开发者 

1. 登录Teambition,要有管理员的权限,点击左上角的菜单按钮,然后点击进入企业的"管理后台".
2. 然后点击"应用管理"
3. 在应用管理页面点击"立即创建"创建应用,弹出创建应用的窗口
4. 在应用创建窗口中填写"应用名称",这个应用名称是自定义的,所属企业为当前管理员用户所在的企业,然后点击确定,就会进入teambition开发者中心的"基本信息"界面
5. 在"基本信息"界面,已经默认生成了当前应用的Client ID和Client Secret,我们在调用Teambition API时,会使用到这两个值
6. 在teambition开发者中心的"OAuth 2配置"页面,填入回调地址,这里的回调地址填企业Teambition首页的地址就可以了,然后点保存,下方的"OAuth 2介绍"里介绍了通过Client_id和redirect_url获取一次性code,然后根据code获取access_token的步骤,Teambition所有的API都要通过access_token来调用
7. 在teambition开发者中心的"Webhook 配置"页面填写teambition服务器地址,点击保存
   完成以上步骤后,就可以根据Client ID,Client Secret,redirect_url获取到access_token,然后调用Teambition的API

## 获取Teambition access_token

这一步是调用Teambition API最重要的步骤,需要花点时间完成.直接上代码,在注释里说明每步的作用.

发送请求通过强大的requests库,因为获取code是通过回调URL携带回来的,登录过程需要点击"授权登录"按钮,所以会使用selenium库模拟浏览器输入内容和点击按钮,具体的使用方法考:

Python使用Selenium模拟浏览器输入内容和鼠标点击

sleep是因为Teambition登录时会有加载过程,用sleep来等待页面加载.

记得下载对应浏览器版本的chromedriver.exe到代码所在目录下,然后把company_id, client_id, client_secret, user_email, user_password换成自己的值,然后运行就可以了获得token了.

打印出token后,将token值赋值给__init__()下的self.token,后面的方法就可以直接使用token值了.完整伪代码如下：

```python
import requests
from selenium import webdriver
import time
 
 
class GetTeamBitionEvents(object):
 
    def __init__(self):
        # 进入teambition企业首页的url,可以通用,后面拼接不同的企业id即进入不同企业的首页
        self.company_url = 'https://www.teambition.com/organization/'
        # 登录Teambition默认进入您的企业首页，url中有企业的id,复制到此,企业创建后id不会改变
        self.company_id = 'xxxxxxxxxxxxxxxxxxxxxx'
        # 在开发者中心OAuth2配置处填的回调地址，与这里拼接的回调地址保持一致
        self.callback_url = self.company_url + self.company_id
        # 在teambition开发者中心，创建应用时的Client ID,复制到此
        self.client_id = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
        # 在teambition开发者中心，创建应用时的Client Secret,复制到此
        self.client_secret = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
        # 获取一次性code的url
        self.auth_url = 'https://account.teambition.com/oauth2/authorize?client_id=' + self.client_id + '&redirect_uri=' + self.callback_url
        # 具有管理员权限的用户邮箱,填您的登录邮箱
        self.user_email = 'wuyazi@alibaba.cn'
        # 您的登录密码
        self.user_password = 'qwert!@#$%'
        # 获取token的url
        self.token_url = 'https://account.teambition.com/oauth2/access_token'
        self.token = ''
 
    def get_code(self):
        """
        模拟浏览器获取code
        """
        # 声明一个浏览器对象，指定使用chrome浏览器
        browser = webdriver.Chrome()
        try:
            # get打开指定的url，传入要打开的url
            browser.get(self.auth_url)
            browser.find_element_by_name('email').send_keys(self.user_email)
            browser.find_element_by_name('password').send_keys(self.user_password)
            browser.find_element_by_class_name('anim-blue-all').click()
            time.sleep(2)
            browser.find_element_by_class_name('authorize-btn').click()
            code = browser.current_url.split('=')[1]
            time.sleep(10)
            browser.close()
            return code
        except Exception as e:
            print("模拟登录获取code失败：{}".format(e))
            browser.close()
 
    def get_token(self):
        """
        根据code获取token
        """
        code = self.get_code()
        access_data = {'client_id': self.client_id, 'client_secret': self.client_secret, 'code': code}
        result = requests.post(self.token_url, data=access_data)
        return result.text
 
 
if __name__ == '__main__':
    tb = GetTeamBitionEvents()
    token = tb.get_token()
    print(token)
```

appAccessToken，使用 App ID 和 App Secret 计算得到，示例代码如下：

```java
import com.auth0.jwt.JWT;
import com.auth0.jwt.algorithms.Algorithm;

public static final Long   EXPIRES_IN = 1 * 3600 * 1000L;
public static final String TOKEN_APPID = "_appId";

public static String genAppToken(String appId, String appSecret) {
    if (StringUtils.isEmpty(appId) || StringUtils.isEmpty(appSecret)) {
        return null;
    }

    Algorithm algorithm = Algorithm.HMAC256(appSecret);
    long timestamp = System.currentTimeMillis();
    Date issuedAt = new Date(timestamp);
    Date expiresAt = new Date(timestamp + EXPIRES_IN);

    return JWT.create()
            .withClaim(TOKEN_APPID, appId)
            .withIssuedAt(issuedAt)
            .withExpiresAt(expiresAt)
            .sign(algorithm);
}
```
其中 jwt 库的版本是：
```xml
<dependency> 
      <groupId>com.auth0</groupId>  
      <artifactId>java-jwt</artifactId>  
      <version>3.4.0</version> 
</dependency>  
```




## 根据access_token调用Teambition API
Teambition API decumentation:  https://docs.teambition.com/

Teambition接口文档中提供了所有可以调用的接口,如果有更新,调用的时候以最新的为准就可以了.

在上面的代码后面（同一个类里）,加入如下两个类方法,分别是根据url获取数据的方法,根据项目id获取项目中所有事件的方法.

按照接口文档调用时如果有问题，也可以与Teambition后台开发人员联系.

```text
    def get_by_url(self, url):
        """
        根据url获取结果
        """
        params = {'access_token': self.token['access_token']}
        try:
            result = requests.get(url, params=params)
            return result.text
        except Exception as e:
            print(e)
            return
 
    def get_event_by_project(self, project_id):
        """
        根据项目id获取项目中的events
        """
        url = 'https://api.teambition.com/api/projects/' + project_id + '/tasks'
        params = {'access_token': self.token['access_token'], 'all': 'true'}
        try:
            result = requests.get(url, params=params)
            return result.text
        except Exception as e:
            print(e)
            return
```
可以看到,调用非常简单,直接使用requests发送请求,就会返回json数据,从数据中解析我们需要的数值即可.

然后根据自己需要的值到接口文档中找到适合的API,如法炮制~~~


(事实上,python有一个第三方库就叫teambitiom,对接口做了封装,但亲自试用了,很多接口反而不通,应该是很久没有人维护了，所以直接调 Teambition API即可)


## 使用SpringBoot同步Teambition信息

使用RestTemplate发送get或者post请求
```java
package com.demo.tb;

import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.RestTemplate;

import javax.annotation.Resource;
import java.util.Iterator;
import java.util.Map;

@Component
public class TbRestTemplate {
    @Resource
    private RestTemplate restTemplate;

    /**
     * TB Post请求
     *
     * @param url
     * @param body
     * @param responseType
     * @param <T>
     * @return
     */
    public <T> T post(String url, String body, Class<T> responseType) {
        // 请求头
        HttpHeaders headers = new HttpHeaders();
        headers.add("X-Tenant-Type", "organization");
        headers.add("X-Tenant-Id", "****");
        headers.add("Authorization", "Bearer **");
        headers.add("Content-Type", "application/json");

        HttpEntity<String> request = new HttpEntity<>(body, headers);
        ResponseEntity<T> responseBody = restTemplate.postForEntity(url, request, responseType);
        return responseBody.getBody();
    }


    public <T> T get(String url, Map<String, Object> map, Class<T> responseType) {
        MultiValueMap<String, Object> param = new LinkedMultiValueMap<String, Object>();
        map.forEach(param::add);
        //请求头
        HttpHeaders headers = new HttpHeaders();
        headers.add("X-Tenant-Type", "organization");
        headers.add("X-Tenant-Id", "****");
        headers.add("Authorization", "Bearer **");
        //封装请求头
        HttpEntity<MultiValueMap<String, Object>> formEntity = new HttpEntity<>(headers);


        Iterator<Map.Entry<String, Object>> iterator = map.entrySet().iterator();

        StringBuilder stringBuilder = new StringBuilder();
        stringBuilder.append(url).append("?");
        while (iterator.hasNext()) {
            Map.Entry<String, Object> next = iterator.next();
            stringBuilder.append(next.getKey()).append("=").append(next.getValue());
            if (iterator.hasNext()) {
                stringBuilder.append("&");
            }
        }
        String fullUrl = stringBuilder.toString();

        //4. 有请求头，有参数，result4.getBody()获取响应参数
        ResponseEntity<T> responseEntity = restTemplate.exchange(fullUrl, HttpMethod.GET, formEntity, responseType);
        return responseEntity.getBody();
    }


}

```

## 调用TB开发平台接口测试

```java
package com.tb;

import com.demo.Application;
import com.demo.tb.ProjectResult;
import com.demo.tb.TbRestTemplate;
import com.demo.utils.JacksonUtils;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;
import java.util.HashMap;

@SpringBootTest(classes = Application.class)
@Slf4j
public class TbTest {


    @Resource
    private TbRestTemplate tbRestTemplate;

    @Test
    public void test1() {
        String url = "https://open.teambition.com/api/project/query";
        // 请求体
        HashMap<String, Object> hashMap = new HashMap<>();
        hashMap.put("isArchived", 0);
        hashMap.put("creatorId", "617606715442e87415dec44c");
        hashMap.put("pageSize", 200);
        String body = JacksonUtils.toJson(hashMap);
        ProjectResult projectResult = tbRestTemplate.post(url, body, ProjectResult.class);
        log.info("projectResult:{}", projectResult);
    }

    @Test
    public void test2() {
        String url = "https://open.teambition.com/api/org/member/list";
        // 请求体
        HashMap<String, Object> hashMap = new HashMap<>();
        hashMap.put("orgId", "5fbb4ce9a3dbf859c88dcebd");
        hashMap.put("pageSize", 200);

        String s = tbRestTemplate.get(url, hashMap, String.class);
        log.info("s:{}", s);
    }

}

```

## 返回结果
```java
package com.demo.tb;

import lombok.Data;
import lombok.ToString;

import java.io.Serializable;
import java.util.Date;

@Data
@ToString
public class Project implements Serializable {

    private String projectId;
    private String name;
    private String pinyin;
    private String py;
    private String description;
    private String cover;
    private String type;
    private String visibility;
    private Integer isArchived;
    private String creatorId;
    private String modifierId;
    private Date created;
    private Date updated;
}

package com.demo.tb;

        import lombok.Data;
        import lombok.ToString;

        import java.io.Serializable;
        import java.util.List;

@Data
@ToString
public class ProjectResult extends TbResult implements Serializable {

  private List<Project> result;
}

package com.demo.tb;

        import lombok.Data;
        import lombok.ToString;

        import java.io.Serializable;
@Data
@ToString
public class TbResult implements Serializable {

  private Integer code;

  private String errorMessage;

}



```

Teambition接口调用还是比较简单的。




