---
layout: post
categories: Kubernetes
description: none
keywords: Kubernetes
---
# Kubernetes开发指南

本章将引入REST的概念，详细说明Kubernetes API的概念和使用方法，并举例说明如何基于Jersey和Fabric8框架访问Kubernetes API，深入分析基于这两个框架访问Kubernetes API的优缺点，最后对Kubernetes API的扩展进行详细说明。下面从REST开始说起。

## Kubernetes API概述
Kubernetes API是集群系统中的重要组成部分，Kubernetes中各种资源（对象）的数据都通过该API接口被提交到后端的持久化存储（etcd）中，Kubernetes集群中的各部件之间通过该API接口实现解耦合，同时Kubernetes集群中一个重要且便捷的管理工具kubectl也是通过访问该API接口实现其强大的管理功能的。Kubernetes API中的资源对象都拥有通用的元数据，资源对象也可能存在嵌套现象，比如在一个Pod里面嵌套多个Container。创建一个API对象是指通过API调用创建一条有意义的记录，该记录一旦被创建，Kubernetes就将确保对应的资源对象会被自动创建并托管维护。

在Kubernetes系统中，在大多数情况下，API定义和实现都符合标准的HTTP REST格式，比如通过标准的HTTP动词（POST、PUT、GET、DELETE）来完成对相关资源对象的查询、创建、修改、删除等操作。但同时，Kubernetes也为某些非标准的REST行为实现了附加的API接口，例如Watch某个资源的变化、进入容器执行某个操作等。另外，某些API接口可能违背严格的REST模式，因为接口返回的不是单一的JSON对象，而是其他类型的数据，比如JSON对象流或非结构化的文本日志数据等。

Kubernetes开发人员认为，任何成功的系统都会经历一个不断成长和不断适应各种变更的过程。因此，他们期望Kubernetes API是不断变更和增长的。同时，他们在设计和开发时，有意识地兼容了已存在的客户需求。通常，我们不希望将新的API资源和新的资源域频繁地加入系统，资源或域的删除需要一个严格的审核流程。

Kubernetes API文档官网为https://kubernetes.io/docs/reference，可以通过相关链接查看不同版本的API文档，例如Kubernetes 1.14版本的链接为https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14。

在Kubernetes 1.13版本及之前的版本中，Master的API Server服务提供了Swagger格式的API网页。Swagger UI是一款REST API文档在线自动生成和功能测试软件，关于Swagger的内容请访问官网http://swagger.io。我们通过设置kube-apiserver服务的启动参数--enable-swagger-ui=true来启用Swagger UI页面，其访问地址为http://<master-ip>:<master-port>/swagger-ui/。假设API Server启动了192.168.18.3服务器上的8080端口，则可以通过访问http://192.168.18.3:8080/swagger-ui/来查看API列表。

以创建一个Pod为例，找到Rest API的访问路径为“/api/v1/namespaces/{namespace}/pods”。

## Kubernetes API
在Kubernetes API中，一个API的顶层（Top Level）元素由kind、apiVersion、metadata、spec和status组成，接下来分别进行说明。
- kind
kind表明对象有以下三大类别。
  - 对象（objects）：代表系统中的一个永久资源（实体），例如Pod、RC、Service、Namespace及Node等。通过操作这些资源的属性，客户端可以对该对象进行创建、修改、删除和获取操作。
  - 列表（list）：一个或多个资源类别的集合。所有列表都通过items域获得对象数组，例如PodLists、ServiceLists、NodeLists。大部分被定义在系统中的对象都有一个返回所有资源集合的端点，以及零到多个返回所有资源集合的子集的端点。某些对象有可能是单例对象（singletons），例如当前用户、系统默认用户等，这些对象没有列表。
  - 简单类别（simple）：该类别包含作用在对象上的特殊行为和非持久实体。该类别限制了使用范围，它有一个通用元数据的有限集合，例如Binding、Status。

- apiVersion
apiVersion表明API的版本号，当前版本默认只支持v1。

- Metadata
Metadata是资源对象的元数据定义，是集合类的元素类型，包含一组由不同名称定义的属性。在Kubernetes中每个资源对象都必须包含以下3种Metadata。
  - namespace：对象所属的命名空间，如果不指定，系统则会将对象置于名为default的系统命名空间中。
  - name：对象的名称，在一个命名空间中名称应具备唯一性。
  - uid：系统为每个对象都生成的唯一ID，符合RFC 4122规范的定义。
此外，每种对象都还应该包含以下几个重要元数据。
  - labels：用户可定义的“标签”，键和值都为字符串的map，是对象进行组织和分类的一种手段，通常用于标签选择器，用来匹配目标对象。
  - annotations：用户可定义的“注解”，键和值都为字符串的map，被Kubernetes内部进程或者某些外部工具使用，用于存储和获取关于该对象的特定元数据。
  - resourceVersion：用于识别该资源内部版本号的字符串，在用于Watch操作时，可以避免在GET操作和下一次Watch操作之间造成的信息不一致，客户端可以用它来判断资源是否改变。该值应该被客户端看作不透明，且不做任何修改就返回给服务端。客户端不应该假定版本信息具有跨命名空间、跨不同资源类别、跨不同服务器的含义。
  - creationTimestamp：系统记录创建对象时的时间戳，符合RFC 3339规范。
  - deletionTimestamp：系统记录删除对象时的时间戳，符合RFC 3339规范。
  - selfLink：通过API访问资源自身的URL，例如一个Pod的link可能是“/api/v1/namespaces/ default/pods/frontend-o8bg4”。

- spec
spec是集合类的元素类型，用户对需要管理的对象进行详细描述的主体部分都在spec里给出，它会被Kubernetes持久化到etcd中保存，系统通过spec的描述来创建或更新对象，以达到用户期望的对象运行状态。spec的内容既包括用户提供的配置设置、默认值、属性的初始化值，也包括在对象创建过程中由其他相关组件（例如schedulers、auto-scalers）创建或修改的对象属性，比如Pod的Service IP地址。如果spec被删除，那么该对象将会从系统中删除。

- Status
Status用于记录对象在系统中的当前状态信息，它也是集合类元素类型，status在一个自动处理的进程中被持久化，可以在流转的过程中生成。如果观察到一个资源丢失了它的状态（Status），则该丢失的状态可能被重新构造。以Pod为例，Pod的status信息主要包括conditions、containerStatuses、hostIP、phase、podIP、startTime等，其中比较重要的两个状态属性如下。
  - phase：描述对象所处的生命周期阶段，phase的典型值是Pending（创建中）、Running、Active（正在运行中）或Terminated（已终结），这几种状态对于不同的对象可能有轻微的差别，此外，关于当前phase附加的详细说明可能包含在其他域中。
  - condition：表示条件，由条件类型和状态值组成，目前仅有一种条件类型：Ready，对应的状态值可以为True、False或Unknown。一个对象可以具备多种condition，而condition的状态值也可能不断发生变化，condition可能附带一些信息，例如最后的探测时间或最后的转变时间。
Kubernetes从1.14版本开始，使用OpenAPI（https://www.openapis.org）的格式对API进行查询，其访问地址为http://<master-ip>: <master-port>/openapi/v2。例如，使用命令行工具curl进行查询：

## Kubernetes API版本的演进策略
为了在兼容旧版本的同时不断升级新的API，Kubernetes提供了多版本API的支持能力，每个版本的API都通过一个版本号路径前缀进行区分，例如/api/v1beta3。在通常情况下，新旧几个不同的API版本都能涵盖所有的Kubernetes资源对象，在不同的版本之间，这些API接口存在一些细微差别。Kubernetes开发团队基于API级别选择版本而不是基于资源和域级别，是为了确保API能够清晰、连续地描述一个系统资源和行为的视图，能够控制访问的整个过程和控制实验性API的访问。
API的版本号通常用于描述API的成熟阶段，例如：
- v1表示GA稳定版本；
- v1beta3表示Beta版本（预发布版本）；
- v1alpha1表示Alpha版本（实验性的版本）。
当某个API的实现达到一个新的GA稳定版本时（如v2），旧的GA版本（如v1）和Beta版本（例如v2beta1）将逐渐被废弃，Kubernetes建议废弃的时间如下。
- 对于旧的GA版本（如v1），Kubernetes建议废弃的时间应不少于12个月或3个大版本Release的时间，选择最长的时间。
- 对旧的Beta版本（如v2beta1），Kubernetes建议废弃的时间应不少于9个月或3个大版本Release的时间，选择最长的时间。
- 对旧的Alpha版本，则无须等待，可以直接废弃。
完整的API更新和废弃策略请参考官方网站https://kubernetes.io/docs/reference/usingapi/deprecation-policy/的说明。

## API Groups（API组）
为了更容易对API进行扩展，Kubernetes使用API Groups（API组）进行标识。API Groups以REST URL中的路径进行定义。当前支持两类API groups。
- Core Groups（核心组），也可以称之为Legacy Groups，作为Kubernetes最核心的API，其特点是没有“组”的概念，例如“v1”，在资源对象的定义中表示为“apiVersion:v1”。
- 具有分组信息的API，以/apis/$GROUP_NAME/$VERSION URL路径进行标识，在资源对象的定义中表示为“apiVersion: $GROUP_NAME/$VERSION”，例如：“apiVersion: batch/v1”“apiVersion: extensions:v1beta1”“apiVersion: apps/v1beta1”等，详细的API列表请参见官网https://kubernetes.io/docs/reference，目前根据Kubernetes的不同版本有不同的API说明页面。

例如，由于Pod属于核心资源对象，所以不存在某个扩展API Group，页面显示为Core，在Pod的定义中为“apiVersion: v1”。 StatefulSet则属于名为apps的API组，版本号为v1，在StatefulSet的定义中为“apiVersion: apps/v1”。

如果要启用或禁用特定的API组，则需要在API Server的启动参数中设置--runtime-config进行声明，例如，--runtime-config=batch/v2alpha1表示启用API组“batch/v2alpha1”；也可以设置--runtime-config=batch/v1=false表示禁用API组“batch/v1”。多个API组的设置以逗号分隔。在当前的API Server服务中，DaemonSets、Deployments、HorizontalPodAutoscalers、Ingress、Jobs和ReplicaSets所属的API组是默认启用的。

## API REST的方法说明
API资源使用REST模式，对资源对象的操作方法如下。
- GET /<资源名的复数格式>：获得某一类型的资源列表，例如GET /pods返回一个Pod资源列表。
- POST /<资源名的复数格式>：创建一个资源，该资源来自用户提供的JSON对象。
- GET /<资源名复数格式>/<名称>：通过给出的名称获得单个资源，例如GET/pods/first返回一个名为first的Pod。
- DELETE /<资源名复数格式>/<名称>：通过给出的名称删除单个资源，在删除选项（DeleteOptions）中可以指定优雅删除（Grace Deletion）的时间（GracePeriodSeconds），该选项表明了从服务端接收到删除请求到资源被删除的时间间隔（单位为s）。不同的类别（Kind）可能为优雅删除时间（Grace Period）声明默认值。用户提交的优雅删除时间将覆盖该默认值，包括值为0的优雅删除时间。
- PUT /<资源名复数格式>/<名称>：通过给出的资源名和客户端提供的JSON对象来更新或创建资源。
- PATCH /<资源名复数格式>/<名称>：选择修改资源详细指定的域。
对于PATCH操作，目前Kubernetes API通过相应的HTTP首部“Content-Type”对其进行识别。
目前支持以下三种类型的PATCH操作。
- JSON Patch, Content-Type: application/json-patch+json。在RFC6902的定义中，JSON Patch是执行在资源对象上的一系列操作，例如 {"op": "add", "path": "/a/b/c", "value": ["foo","bar"]}。详情请查看RFC6902说明，网址为https://tools.ietf.org/html/rfc6902。
- Merge Patch, Content-Type: application/merge-json-patch+json。在RFC7386的定义中，Merge Patch必须包含对一个资源对象的部分描述，这个资源对象的部分描述就是一个JSON对象。该JSON对象被提交到服务端，并和服务端的当前对象合并，从而创建一个新的对象。详情请查看RFC73862说明，网址为https://tools.ietf.org/html/rfc7386。
- Strategic Merge Patch, Content-Type: application/strategic-merge-patch+json。Strategic Merge Patch是一个定制化的Merge Patch实现。接下来将详细讲解Strategic Merge Patch。

在标准的JSON Merge Patch中，JSON对象总被合并（Merge），但是资源对象中的列表域总被替换，用户通常不希望如此。例如，我们通过下列定义创建一个Pod资源对象：
```yaml
spec:
containers:
- name: nginx
image: nginx-1.0
```
接着，我们希望添加一个容器到这个Pod中，代码和上传的JSON对象如下：
```yaml
PATH /api/vi/namespaces/default/pods/pod-name
spec:
containers:
- name: log-tailer
image: log-tailer-1.0
```
如果我们使用标准的Merge Patch，则其中的整个容器列表将被单个“log-tailer”容器所替换，然而我们的目的是使两个容器列表合并。

为了解决这个问题，Strategic Merge Patch添加元数据到API对象中，并通过这些新元数据来决定哪个列表被合并，哪个列表不被合并。当前这些元数据作为结构标签，对于API对象自身来说是合法的。对于客户端来说，这些元数据作为Swagger annotations也是合法的。在上述例子中向containers中添加了patchStrategy域，且它的值为merge，通过添加patchMergeKey，它的值为name。也就是说，containers中的列表将会被合并而不是替换，合并的依据为name域的值。

此外，Kubernetes API添加了资源变动的观察者模式的API接口。
- GET /watch/<资源名复数格式>：随时间变化，不断接收一连串的JSON对象，这些JSON对象记录了给定资源类别内所有资源对象的变化情况。
- GET /watch/<资源名复数格式>/<name>：随时间变化，不断接收一连串的JSON对象，这些JSON对象记录了某个给定资源对象的变化情况。
上述接口改变了返回数据的基本类别，watch动词返回的是一连串JSON对象，而不是单个JSON对象。并不是所有对象类别都支持观察者模式的API接口。

另外，Kubernetes还增加了HTTP Redirect与HTTP Proxy这两种特殊的API接口，前者实现资源重定向访问，后者则实现HTTP请求的代理。

## API Server响应说明
API Server在响应用户请求时附带一个状态码，该状态码符合HTTP规范。
- 200 OK 表明请求完全成功
- 201 Create 表明创建类的请求完全成功
- 204 NoContent 表明请求完全成功，同时HTTP响应不包含响应体。在响应OPTIONS方法的HTTP请求时返回。
- 307 TemporaryRedirect 表明请求资源的地址被改变，建议客户端使用Location首部给出的临时URL来定位资源
- 400 BadRequest 表明请求是非法的，建议用户不要重试，修改该请求
- 401 Unauthorized 表明请求能够到达服务端，且服务端能够理解用户请求，但是拒绝做更多的事情，因为客户端必须提供认证信息。
如果客户端提供了认证信息，则返回该状态码，表明服务端指出所提供的认证信息不合适或非法。
- 403 Forbidden 表明请求能够到达服务端，且服务端能够理解用户请求，但是拒绝做更多的事情，因为该请求被设置成拒绝访问。建议用户不要重试，修改该请求。
- 404 NotFound 表明所请求得资源不存在。建议用户不要重试，修改该请求。
- 405 MethodNotAllowed 表明所请求中带有该资源不支持的方法。建议用户不要重试，修改该请求。
- 409 Conflict 表明客户端尝试创建的资源已经存在，或者由于冲突，请求的更新操作不能被完成
- 422 UnprocessableEntity 表明由于所提供的作为请求部分的数据非法，创建或修改操作不能被完成。
- 429 TooManyRequests 表明超出了客户端访问频率的限制或者服务端接收到多于它能处理的请求。建议客户端读取相应的Retry-After首部，然后等待该首部指出的时间后再重试。
- 500 InternalServerError 表明服务端能被请求访问到，但是不能理解用户的请求；或者在服务端内产生非预期的一个错误，而且该错误无法被认知；
或者服务端不能在一个合理的时间内完成处理（这可能是服务器临时负载过重造成的，或和其他服务器通信时的一个临时通信故障造成的）
- 503 ServiceUnavailable 表明被请求的服务无效。建议用户不要重试，修改该请求。
- 504 ServiceTimeout 表明请求在给定的时间内无法完成。客户端仅在为请求指定超时(Timeout)参数时得到该响应。

在调用API接口发生错误时，Kubernetes将会返回一个状态类别（Status Kind）。下面是两种常见的错误场景。
- 当一个操作不成功时（例如，当服务端返回一个非2xx HTTP状态码时）。
- 当一个HTTP DELETE方法调用失败时。
状态对象被编码成JSON格式，同时该JSON对象被作为请求的响应体。该状态对象包含人和机器使用的域，在这些域中包含来自API的关于失败原因的详细信息。状态对象中的信息补充了对HTTP状态码的说明。例如：
```
# curl -v -k -H "Authorization: Bearer admin" https://10.240.122.184:443/api/v1/namespaces/default/pods/grafana
```
输出：
```json
{
"apiVersion": "v1",
"kind": "Status",
"metadata": {},
"status": "Failure",
"message": "pods \"grafana \"not found",
"reason": "NotFound",
"details": {
"name": "grafana",
"kind": "pods"
},
"code": 404
}
```
其中：
- status域包含两个可能的值：Success或Failure。
- message域包含对错误的描述信息。
- reason域包含说明该操作失败原因的描述。
- details可能包含和reason域相关的扩展数据。每个reason域都可以定义它的扩展的details域。该域是可选的，返回数据的格式是不确定的，不同的reason类型返回的details域的内容不一样。

## 使用Java程序访问Kubernetes API
本节介绍如何使用Java程序访问Kubernetes API。在Kubernetes官网上列出了多个访问Kubernetes API的开源项目，其中有两个是用Java语言开发的开源项目，一个是OSGI，另一个是Fabric8。

## Jersey
Jersey是一个RESTful请求服务Java框架。与Struts类似，它可以和Hibernate、Spring框架整合。我们不仅能很方便地通过它开发RESTful Web Service，还可以将它作为客户端方便地访问RESTful Web Service服务端。
如果没有一个好的工具包，则很难开发一个能够用不同的媒介（Media）类型无缝地暴露你的数据，以及很好地抽象客户端、服务端通信的底层通信的RESTful Web Services。为了能够简化使用Java开发RESTful Web Service及其客户端的流程，业界设计了JAX-RS API。
Jersey RESTful Web Services框架是一个开源的高质量的框架，它为用Java语言开发RESTful Web Service及其客户端而生，支持JAX-RS APIs。Jersey不仅支持JAX-RS APIs，而且在此基础上扩展了API接口，这些扩展更加方便并简化了RESTful Web Services及其客户端的开发。
由于Kubernetes API Server是RESTful Web Service，因此此处选用Jersey框架开发RESTful Web Service客户端，用来访问Kubernetes API。



对Kubernetes API的访问包含如下3个方面。
- 指明访问资源的类型。
- 访问时的一些选项（参数），比如命名空间、对象的名称、过滤方式（标签和域）、子目录、访问的目标是否是代理和是否用watch方式访问等。
- 访问的方法，比如增、删、改、查。

在使用Jersey框架访问Kubernetes API之前，我们需要为这3个方面定义3个对象。第1个定义的对象是ResourceType，它定义了访问资源的类型；第2个定义的对象是Params，它定义了访问API时的一些选项，以及通过这些选项如何生成完整的URL；第3个定义的对象是RestfulClient，它是一个接口，该接口定义了访问API的方法（Method）。
- ResourceType是一个ENUM类型的对象，定义了Kubernetes的各种资源对象类型，代码如下：
- Params定义访问API时的选项及通过这些选项如何生成完整的URL，代码如下：
- 接口对象RestfulClient定义了访问API接口的所有方法，其代码如下：

img
其中，get和list方法对应Kubernetes API的GET方法；create方法对应API中的POST方法；delete方法对应API中的DELETE方法；update方法对应API中的PATCH方法；replace方法对应API中的PUT方法；options方法对应API中的OPTIONS方法；head方法对应API中的HEAD方法。

该接口基于Jersey框架的实现类如下：

img
img
img
img
img
在该对象中包含如下代码：

img
该段代码的作用是使Jersey客户端支持除标准REST方法外的方法，比如PATCH方法。该段代码能访问除watcher外的所有Kubernetes API接口，在后续的章节中会举例说明如何访问Kubernetes API。

## Fabric8
Fabric8包含多款工具包，Kubernetes Client只是其中之一，也是在Kubernetes官网中提到的Java Client API之一。 因为该工具包已经对访问Kubernetes API客户端做了较好的封装，因此其访问代码比较简单。

Fabric 8的Kubernetes API客户端工具包只能访问Node、Service、Pod、Endpoints、Events、Namespace、PersistenetVolumeclaims、PersistenetVolume、ReplicationController、ResourceQuota、Secret和ServiceAccount这几种资源类型，不能使用OPTIONS和HEAD方法访问资源，且不能以代理方式访问资源，但其对以watcher方式访问资源做了很好的支持。

引入Maven依赖
```xml
        <dependency>
            <groupId>io.fabric8</groupId>
            <artifactId>kubernetes-client</artifactId>
            <version>5.12.1</version>
        </dependency>
```
参考地址：https://github.com/fabric8io/kubernetes-client
简单介绍一下工程中各个模块的作用：
- kubernetes-client：这是最主要的模块最终打包出来的sdk jar包就是这个模块生成的。
在该工程下最主要的就是DefaultKubernetesClient.class 因为后续操作k8s的client 就是由这个DefaultKubernetesClient 提供，同时可以从这个类中看出这个类提供k8s 系统级别的所有组件资源，这个也是整个sdk的入口。
- kubernetes-model-generator：是client 依赖的实际资源类的生成模块，该模块是通过maven jsonschema2pojo 插件自动生成client依赖类
由于k8s类型众多重复的字段也非常多，这样避免了简单而又繁琐的代码重复书写，后续的client会用到一个模式，我姑且叫“包装模式”很是该工程的巧妙之处。生成class的文件来至kube-schema.json（该文件又可以通过go写的工具来生成json，目前支持k8s原生资源，自定义的需要手动在里面添加

首先要说一个很重要的对象----client 。在每次操作集群前都需要先连接到k8s集群，通过连接方法获得的就是KubernetesClient接口下的client对象。之后的所有针对k8s集群的操作都无一例外地要传入这个参数，所以KubernetesClient接口自然成为了我们一窥Fabric8源码的突破口。
```java
import io.fabric8.kubernetes.client.ConfigBuilder;
import io.fabric8.kubernetes.client.DefaultKubernetesClient;
import io.fabric8.kubernetes.client.KubernetesClient;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

@Component
public class Fabric8K8sClient {
    
    private final static ConcurrentMap<String, KubernetesClient> clientMap = new ConcurrentHashMap<>();
    
    public KubernetesClient getKubernetesClient(String k8sApiserver,String k8sToken) {
        return Fabric8K8sClient.getKubernetesClient(k8sApiserver, k8sToken);
    }

    public static KubernetesClient getKubernetesClient(String k8sApiServer, String k8sToken) {
        String k8sConfig = k8sApiServer + "_" + k8sToken;
        return clientMap.computeIfAbsent(k8sConfig, key -> new DefaultKubernetesClient(new ConfigBuilder()
                .withTrustCerts(true)
                .withMasterUrl(k8sApiServer)
                .withOauthToken(k8sToken)
                .build()));

    }
}
```
可以使用token登陆或者使用证书登陆
要初始化Fabric8的KubernetesClient，需要4项配置，分别是：
k8s.url：K8s集群地址
k8s.client-crt：客户端CA证书
k8s.client-key：客户端RSA私钥
k8s.ca-crt：K8s集群CA证书
```properties
# k8s 配置
k8s.url=https://10.103.18.42:6443
k8s.client-crt=classpath:k8s/apiserver-kubelet-client.crt
k8s.client-key=classpath:k8s/apiserver-kubelet-client.key
k8s.ca-crt=classpath:k8s/ca.crt
```
- 创建资源
这个Java库使用了大量的Builder模式来创建对象，创建命令空间如下：
```java
Namespace namespace = new NamespaceBuilder()
        .withNewMetadata()
        .withName("pkslow")
        .addToLabels("reason", "pkslow-sample")
        .endMetadata()
        .build();
        client.namespaces().createOrReplace(namespace);
```
非常灵活，上面例子添加了名字和标签，最后通过createOrReplace方法可新建，如果存在可替换。
- 对于Pod也是类似的：
```java
Pod pod = new PodBuilder()
        .withNewMetadata()
        .withName("nginx")
        .addToLabels("app", "nginx")
        .endMetadata()
        .withNewSpec()
        .addNewContainer()
        .withName("nginx")
        .withImage("nginx:1.19.5")
        .endContainer()
        .endSpec()
        .build();
        client.pods().inNamespace("pkslow").createOrReplace(pod);
```
指定名字、标签和镜像后就可以创建了。

- 查看资源
查看资源可以查询所有，或者通过条件options来过滤，具体代码如下：
```java
// 查看命名空间
NamespaceList namespaceList = client.namespaces().list();
        namespaceList.getItems()
        .forEach(namespace ->
        System.out.println(namespace.getMetadata().getName() + ":" + namespace.getStatus().getPhase()));

// 查看Pod
        ListOptions options = new ListOptions();
        options.setLabelSelector("app=nginx");
        Pod nginx = client.pods().inNamespace("pkslow")
        .list(options)
        .getItems()
        .get(0);
        System.out.println(nginx);
```
- 修改资源
修改资源是通过edit方法来实现的，可通过命名空间和名字来定位到资源，然后进行修改，示例代码如下：
```java
// 修改命名空间
client.namespaces().withName("pkslow")
        .edit(n -> new NamespaceBuilder(n)
        .editMetadata()
        .addToLabels("project", "pkslow")
        .endMetadata()
        .build()
        );

// 修改Pod
        client.pods().inNamespace("pkslow").withName("nginx")
        .edit(p -> new PodBuilder(p)
        .editMetadata()
        .addToLabels("app-version", "1.0.1")
        .endMetadata()
        .build()
        );

```
- 删除资源
删除资源也是类似的，先定位再操作：
```java
client.pods().inNamespace("pkslow")
        .withName("nginx")
        .delete();

```
- 通过yaml文件操作
我们还可以直接通过yaml文件来描述资源，而不用Java来定义，这样可以更直观和方便。完成yaml文件的编写后，Load成对应的对象，再进行各种增删改查操作，示例如下：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    myapp: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      myapp: nginx
  template:
    metadata:
      labels:
        myapp: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
            - containerPort: 80
```
Java代码如下：
```java
Deployment deployment = client.apps().deployments()
        .load(Fabric8KubernetesClientSamples.class.getResourceAsStream("/deployment.yaml"))
        .get();
        client.apps().deployments().inNamespace("pkslow")
        .createOrReplace(deployment);

```
- 监听事件
我们还可以通过监听资源的事件，来进行对应的反应，比如有人删除了Pod就记录日志到数据库等，这个功能还是非常有用的。示例代码如下：
```java
client.pods().inAnyNamespace().watch(new Watcher<Pod>() {
@Override
public void eventReceived(Action action, Pod pod) {
        System.out.println("event " + action.name() + " " + pod.toString());
        }

@Override
public void onClose(WatcherException e) {
        System.out.println("Watcher close due to " + e);

        }
        });
```
通过一个Watcher监听了Pod的所有动作事件，然后打印动作名和对应的Pod。输出后的日志如下：
```java
event ADDED Pod(apiVersion=v1, kind=Pod, metadata=ObjectMeta(
event MODIFIED Pod(apiVersion=v1, kind=Pod, metadata=ObjectMeta(
event DELETED Pod(apiVersion=v1, kind=Pod, metadata=ObjectMeta(
event MODIFIED Pod(apiVersion=v1, kind=Pod, metadata=ObjectMeta(
```

## 其他客户端库
为了让开发人员更方便地访问Kubernetes的RESTful API，Kubernetes社区推出了针对Go、Python、Java、dotNet、JavaScript等编程语言的客户端库，这些库由特别兴趣小组（SIG）API Machinary维护，其官方网站为https://github.com/kubernetes/community/tree/master/sig-api-machinery。

## Kubernetes API的扩展
随着Kubernetes的发展，用户对Kubernetes的扩展性也提出了越来越高的要求。从1.7版本开始，Kubernetes引入扩展API资源的能力，使得开发人员在不修改Kubernetes核心代码的前提下可以对Kubernetes API进行扩展，仍然使用Kubernetes的语法对新增的API进行操作，这非常适用于在Kubernetes上通过其API实现其他功能（例如第三方性能指标采集服务）或者测试实验性新特性（例如外部设备驱动）。

在Kubernetes中，所有对象都被抽象定义为某种资源对象，同时系统会为其设置一个API入口（API Endpoint），对资源对象的操作（如新增、删除、修改、查看等）都需要通过Master的核心组件API Server调用资源对象的API来完成。与API Server的交互可以通过kubectl命令行工具或访问其RESTful API进行。每个API都可以设置多个版本，在不同的API URL路径下区分，例如“/api/v1”或“/apis/extensions/v1beta1”等。使用这种机制后，用户可以很方便地定义这些API资源对象（YAML配置），并将其提交给Kubernetes（调用RESTful API），来完成对容器应用的各种管理工作。

Kubernetes系统内置的Pod、RC、Service、ConfigMap、Volume等资源对象已经能够满足常见的容器应用管理要求，但如果用户希望将其自行开发的第三方系统纳入Kubernetes，并使用Kubernetes的API对其自定义的功能或配置进行管理，就需要对API进行扩展了。目前Kubernetes提供了以下两种机制供用户扩展API。
- 使用CRD机制：复用Kubernetes的API Server，无须编写额外的API Server。用户只需要定义CRD，并且提供一个CRD控制器，就能通过Kubernetes的API管理自定义资源对象了，同时要求用户的CRD对象符合API Server的管理规范。
- 使用API聚合机制：用户需要编写额外的API Server，可以对资源进行更细粒度的控制（例如，如何在各API版本之间切换），要求用户自行处理对多个API版本的支持。

本节主要对CRD和API聚合这两种API扩展机制的概念和用法进行详细说明。

## 使用CRD扩展API资源
CRD是Kubernetes从1.7版本开始引入的特性，在Kubernetes早期版本中被称为TPR（ThirdPartyResources，第三方资源）。TPR从Kubernetes 1.8版本开始被停用，被CRD全面替换。

CRD本身只是一段声明，用于定义用户自定义的资源对象。但仅有CRD的定义并没有实际作用，用户还需要提供管理CRD对象的CRD控制器（CRD Controller），才能实现对CRD对象的管理。CRD控制器通常可以通过Go语言进行开发，并需要遵循Kubernetes的控制器开发规范，基于客户端库client-go进行开发，需要实现Informer、ResourceEventHandler、Workqueue等组件具体的功能处理逻辑，详细的开发过程请参考官方示例（https://github.com/kubernetes/sample-controller）和client-go库（https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md）的详细说明。

### 创建CRD的定义

与其他资源对象一样，对CRD的定义也使用YAML配置进行声明。以Istio系统中的自定义资源VirtualService为例，配置文件crd-virtualservice.yaml的内容如下：

img
img
CRD定义中的关键字段如下。

（1）group：设置API所属的组，将其映射为API URL中“/apis/”的下一级目录，设置networking.istio.io生成的API URL路径为“/apis/networking.istio.io”。

（2）scope：该API的生效范围，可选项为Namespaced（由Namespace限定）和Cluster（在集群范围全局生效，不局限于任何Namespace），默认值为Namespaced。

（3）versions：设置此CRD支持的版本，可以设置多个版本，用列表形式表示。目前还可以设置名为version的字段，只能设置一个版本，在将来的Kubernetes版本中会被弃用，建议使用versions进行设置。如果该CRD支持多个版本，则每个版本都会在API URL“/apis/networking.istio.io”的下一级进行体现，例如“/apis/networking.istio.io/v1”或“/apis/networking.istio.io/v1alpha3”等。每个版本都可以设置下列参数。

◎　name：版本的名称，例如v1、v1alpha3等。

◎　served：是否启用，在被设置为true时表示启用。

◎　storage：是否进行存储，只能有一个版本被设置为true。

（4）names：CRD的名称，包括单数、复数、kind、所属组等名称的定义，可以设置如下参数。

◎　kind：CRD的资源类型名称，要求以驼峰式命名规范进行命名（单词的首字母都大写），例如VirtualService。

◎　listKind：CRD列表，默认被设置为<kind>List格式，例如VirtualServiceList。

◎　singular：单数形式的名称，要求全部小写，例如virtualservice。

◎　plural：复数形式的名称，要求全部小写，例如virtualservices。

◎　shortNames：缩写形式的名称，要求全部小写，例如vs。

◎　categories：CRD所属的资源组列表。例如，VirtualService属于istio-io组和networking-istio-io组，用户通过查询istio-io组和networking-istio-io组，也可以查询到该CRD实例。

使用kubectl create命令完成CRD的创建：

img
在CRD创建成功后，由于本例的scope设置了Namespace限定，所以可以通过API Endpoint“/apis/networking.istio.io/v1alpha3/namespaces/<namespace>/virtualservices/”管理该CRD资源。

用户接下来就可以基于该CRD的定义创建相应的自定义资源对象了。

2.基于CRD的定义创建自定义资源对象

基于CRD的定义，用户可以像创建Kubernetes系统内置的资源对象（如Pod）一样创建CRD资源对象。在下面的例子中，virtualservice-helloworld.yaml定义了一个类型为VirtualService的资源对象：

img
img
除了需要设置该CRD资源对象的名称，还需要在spec段设置相应的参数。在spec中可以设置的字段是由CRD开发者自定义的，需要根据CRD开发者提供的手册进行配置。这些参数通常包含特定的业务含义，由CRD控制器进行处理。

使用kubectl create命令完成CRD资源对象的创建：

img
然后，用户就可以像操作Kubernetes内置的资源对象（如Pod、RC、Service）一样去操作CRD资源对象了，包括查看、更新、删除和watch等操作。

查看CRD资源对象：

img
也可以通过CRD所属的categories进行查询：

img
3.CRD的高级特性

随着Kubernetes的演进，CRD也在逐步添加一些高级特性和功能，包括subresources子资源、校验（Validation）机制、自定义查看CRD时需要显示的列，以及finalizer预删除钩子。

（1）CRD的subresources子资源

Kubernetes从1.11版本开始，在CRD的定义中引入了名为subresources的配置，可以设置的选项包括status和scale两类。

◎　stcatus：启用/status路径，其值来自CRD的.status字段，要求CRD控制器能够设置和更新这个字段的值。

◎　scale：启用/scale路径，支持通过其他Kubernetes控制器（如HorizontalPodAutoscaler控制器）与CRD资源对象实例进行交互。用户通过kubectl scale命令也能对该CRD资源对象进行扩容或缩容操作，要求CRD本身支持以多个副本的形式运行。

下面是一个设置了subresources的CRD示例：

img
基于该CRD的定义，创建一个自定义资源对象my-crontab.yaml：

img
之后就能通过API Endpoint查看该资源对象的状态了：

img
并查看该资源对象的扩缩容（scale）信息：

img
用户还可以使用kubectl scale命令对Pod的副本数量进行调整，例如：

img
（2）CRD的校验（Validation）机制

Kubernetes从1.8版本开始引入了基于OpenAPI v3 schema或validatingadmissionwebhook的校验机制，用于校验用户提交的CRD资源对象配置是否符合预定义的校验规则。该机制到Kubernetes 1.13版本时升级为Beta版。要使用该功能，需要为kube-apiserver服务开启--feature-gates=CustomResourceValidation=true特性开关。

下面的例子为CRD定义中的两个字段（cronSpec和replicas）设置了校验规则：

img
img
校验规则如下。

◎　spec.cronSpec：必须为字符串类型，并且满足正则表达式的格式。

◎　spec.replicas：必须将其设置为1～10的整数。

对于不符合要求的CRD资源对象定义，系统将拒绝创建。

例如，下面的my-crontab.yaml示例违反了CRD中validation设置的校验规则，即cronSpec没有满足正则表达式的格式，replicas的值大于10：

img
创建时，系统将报出validation失败的错误信息：

img
（3）自定义查看CRD时需要显示的列

从Kubernetes 1.11版本开始，通过kubectl get命令能够显示哪些字段由服务端（API Server）决定，还支持在CRD中设置需要在查看（get）时显示的自定义列，在spec.additionalPrinterColumns字段设置即可。

在下面的例子中设置了3个需要显示的自定义列Spec、Replicas和Age，并在JSONPath字段设置了自定义列的数据来源：

img
img
通过kubectl get命令查看CronTab资源对象，会显示出这3个自定义的列：

img
（4）Finalizer（CRD资源对象的预删除钩子方法）

Finalizer设置的方法在删除CRD资源对象时进行调用，以实现CRD资源对象的清理工作。

在下面的例子中为CRD“CronTab”设置了一个finalizer（也可以设置多个），其值为URL“finalizer.stable.example.com”：

img
在用户发起删除该资源对象的请求时，Kubernetes不会直接删除这个资源对象，而是在元数据部分设置时间戳“metadata.deletionTimestamp”的值，标记为开始删除该CRD对象。然后控制器开始执行finalizer定义的钩子方法“finalizer.stable.example.com”进行清理工作。对于耗时较长的清理操作，还可以设置metadata.deletionGracePeriodSeconds超时时间，在超过这个时间后由系统强制终止钩子方法的执行。在控制器执行完钩子方法后，控制器应负责删除相应的finalizer。当全部finalizer都触发控制器执行钩子方法并都被删除之后，Kubernetes才会最终删除该CRD资源对象。

4.小结

CRD极大扩展了Kubernetes的能力，使用户像操作Pod一样操作自定义的各种资源对象。CRD已经在一些基于Kubernetes的第三方开源项目中得到广泛应用，包括CSI存储插件、Device Plugin（GPU驱动程序）、Istio（Service Mesh管理）等，已经逐渐成为扩展Kubernetes能力的标准。

9.4.2　使用API聚合机制扩展API资源
API聚合机制是Kubernetes 1.7版本引入的特性，能够将用户扩展的API注册到kube-apiserver上，仍然通过API Server的HTTP URL对新的API进行访问和操作。为了实现这个机制，Kubernetes在kube-apiserver服务中引入了一个API聚合层（API Aggregation Layer），用于将扩展API的访问请求转发到用户服务的功能。

设计API聚合机制的主要目标如下。

◎　增加API的扩展性：使得开发人员可以编写自己的API Server来发布他们的API，而无须对Kubernetes核心代码进行任何修改。

◎　无须等待Kubernetes核心团队的繁杂审查：允许开发人员将其API作为单独的API Server发布，使集群管理员不用对Kubernetes核心代码进行修改就能使用新的API，也就无须等待社区繁杂的审查了。

◎　支持实验性新特性API开发：可以在独立的API聚合服务中开发新的API，不影响系统现有的功能。

◎　确保新的API遵循Kubernetes的规范：如果没有API聚合机制，开发人员就可能会被迫推出自己的设计，可能不遵循Kubernetes规范。

总的来说，API聚合机制的目标是提供集中的API发现机制和安全的代理功能，将开发人员的新API动态地、无缝地注册到Kubernetes API Server中进行测试和使用。

下面对API聚合机制的使用方式进行详细说明。

1.在Master的API Server中启用API聚合功能

为了能够将用户自定义的API注册到Master的API Server中，首先需要配置kube-apiserver服务的以下启动参数来启用API聚合功能。

◎　--requestheader-client-ca-file=/etc/kubernetes/ssl_keys/ca.crt：客户端CA证书。

◎　--requestheader-allowed-names=：允许访问的客户端common names列表，通过header中--requestheader-username-headers参数指定的字段获取。客户端common names的名称需要在client-ca-file中进行设置，将其设置为空值时，表示任意客户端都可访问。

◎　--requestheader-extra-headers-prefix=X-Remote-Extra-：请求头中需要检查的前缀名。

◎　--requestheader-group-headers=X-Remote-Group：请求头中需要检查的组名。

◎　--requestheader-username-headers=X-Remote-User：请求头中需要检查的用户名。

◎　--proxy-client-cert-file=/etc/kubernetes/ssl_keys/kubelet_client.crt：在请求期间验证Aggregator的客户端CA证书。

◎　--proxy-client-key-file=/etc/kubernetes/ssl_keys/kubelet_client.key：在请求期间验证Aggregator的客户端私钥。

如果kube-apiserver所在的主机上没有运行kube-proxy，即无法通过服务的ClusterIP进行访问，那么还需要设置以下启动参数：

img
在设置完成重启kube-apiserver服务，就启用API聚合功能了。

2.注册自定义APIService资源

在启用了API Server的API聚合功能之后，用户就能将自定义API资源注册到Kubernetes Master的API Server中了。用户只需配置一个APIService资源对象，就能进行注册了。APIService示例的YAML配置文件如下：

img
在这个APIService中设置的API组名为custom.metrics.k8s.io，版本号为v1beta1，这两个字段将作为API路径的子目录注册到API路径“/apis/”下。注册成功后，就能通过Master API路径“/apis/custom.metrics.k8s.io/v1beta1”访问自定义的API Server了。

在service段中通过name和namespace设置了后端的自定义API Server，本例中的服务名为custom-metrics-server，命名空间为custom-metrics。

通过kubectl create命令将这个APIService定义发送给Master，就完成了注册操作。

之后，通过Master API Server对“/apis/custom.metrics.k8s.io/v1beta1”路径的访问都会被API聚合层代理转发到后端服务custom-metrics-server.custom-metrics.svc上了。

3.实现和部署自定义API Server

仅仅注册APIService资源还是不够的，用户对“/apis/custom.metrics.k8s.io/v1beta1”路径的访问实际上都被转发给了custom-metrics-server.custom-metrics.svc服务。这个服务通常能以普通Pod的形式在Kubernetes集群中运行。当然，这个服务需要由自定义API的开发者提供，并且需要遵循Kubernetes的开发规范，详细的开发示例可以参考官方给出的示例（https://github.com/kubernetes/sample-apiserver）。

下面是部署自定义API Server的常规操作步骤。

（1）确保APIService API已启用，这需要通过kube-apiserver的启动参数--runtime-config进行设置，默认是启用的。

（2）建议创建一个RBAC规则，允许添加APIService资源对象，因为API扩展对整个Kubernetes集群都生效，所以不推荐在生产环境中对API扩展进行开发或测试。

（3）创建一个新的Namespace用于运行扩展的API Server。

（4）创建一个CA证书用于对自定义API Server的HTTPS安全访问进行签名。

（5）创建服务端证书和秘钥用于自定义API Server的HTTPS安全访问。服务端证书应该由上面提及的CA证书进行签名，也应该包含含有DNS域名格式的CN名称。

（6）在新的Namespace中使用服务端证书和秘钥创建Kubernetes Secret对象。

（7）部署自定义API Server实例，通常可以以Deployment形式进行部署，并且将之前创建的Secret挂载到容器内部。该Deployment也应被部署在新的Namespace中。

（8）确保自定义的API Server通过Volume加载了Secret中的证书，这将用于后续的HTTPS握手校验。

（9）在新的Namespace中创建一个Service Account对象。

（10）创建一个ClusterRole用于对自定义API资源进行操作。

（11）使用之前创建的ServiceAccount为刚刚创建的ClusterRole创建一个ClusterRolebinding。

（12）使用之前创建的ServiceAccount为系统ClusterRole“system:auth-delegator”创建一个ClusterRolebinding，以使其可以将认证决策代理转发给Kubernetes核心API Server。

（13）使用之前创建的ServiceAccount为系统Role“extension-apiserver-authenticationreader”创建一个Rolebinding，以允许自定义API Server访问名为“extension-apiserverauthentication”的系统ConfigMap。

（14）创建APIService资源对象。

（15）访问APIService提供的API URL路径，验证对资源的访问能否成功。

下面以部署Metrics Server为例，说明一个聚合API的实现方式。

随着API聚合机制的出现，Heapster也进入弃用阶段，逐渐被Metrics Server替代。Metrics Server通过聚合API提供Pod和Node的资源使用数据，供HPA控制器、VPA控制器及kubectl top命令使用。Metrics Server的源码可以在GitHub代码库（https://github.com/kubernetes-incubator/metrics-server）找到，在部署完成后，Metrics Server将通过Kubernetes核心API Server的“/apis/metrics.k8s.io/v1beta1”路径提供Pod和Node的监控数据。

首先，部署Metrics Server实例，在下面的YAML配置中包含一个ServiceAccount、一个Deployment和一个Service的定义：

img
img
然后，创建Metrics Server所需的RBAC权限配置：

img
img
最后，定义APIService资源，主要设置自定义API的组（group）、版本号（version）及对应的服务（metrics-server.kube-system）：

img
img
在所有资源都成功创建之后，在命名空间kube-system中会看到新建的metrics-server Pod。

通过Kubernetes Master API Server的URL“/apis/metrics.k8s.io/v1beta1”就能查询到Metrics Server提供的Pod和Node的性能数据了：

img
img
来源：
https://www.ai2news.com

