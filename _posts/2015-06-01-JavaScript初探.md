---
layout: post
categories: JavaScript
description: none
keywords: JavaScript
---
# JavaScript初探
JavaScript是互联网上最流行的脚本语言，这门语言可用于HTML和Web，更可广泛用于服务器、PC、笔记本电脑、平板电脑和智能手机等设备。

## JavaScript概述
JavaScript是一种由Netscape公司的LiveScript发展而来的面向过程的客户端脚本语言，为客户提供更流畅的浏览效果。另外，由于Windows操作系统对其拥有较为完善的支持，并提供二次开发的接口来访问操作系统中各个组件，从而可实现相应的管理功能。

JavaScript是一种解释性的、基于对象的脚本语言（Object-based Scripting Language），其主要是基于客户端运行的，用户单击带有JavaScript脚本的网页，网页里的JavaScript就会被传到浏览器，由浏览器对此做处理。
如下拉菜单、验证表单有效性等大量互动性功能，都是在客户端完成的，不需要和Web 服务器进行任何数据交换。因此，不会增加Web 服务器的负担。几乎所有浏览器都支持JavaScript，如Internet Explorer（IE）、Firefox、Netscape、Mozilla、Opera等。

JavaScript是一种新的描述语言，它可以被嵌入到HTML文件中。JavaScript可以做到回应使用者的需求事件（如form的输入），而不用任何网络来回传输资料，所以当一位使用者输入一项资料时，它不用经过传给服务器端处理再传回来的过程，而直接可以被客户端的应用程序所处理。

## JavaScript的基本特点
JavaScript的主要作用是与HTML、Java 脚本语言（Java小程序）一起实现在一个Web页面中连接多个对象、与Web客户端交互作用，从而可以开发客户端的应用程序等。它是通过嵌入或调入到标准的HTML中实现的。它弥补了HTML的缺陷，是Java与HTML折中的选择，具有如下基本特点。
- 脚本编写语言。
JavaScript是一种采用小程序段方式来实现编程的脚本语言。同其他脚本语言一样，JavaScript是一种解释性语言，在程序运行过程中被逐行地解释。此外，它还可与HTML标识结合在一起，从而方便用户的使用。
- 基于对象的语言。
JavaScript是一种基于对象的语言，同时可以看作一种面向对象的语言。这意味着它能运用自己已经创建的对象。因此，许多功能可以来自于脚本环境中对象的方法与脚本的相互作用。
- 简单性。
JavaScript的简单性主要体现在：首先，它是一种基于Java基本语句和控制流之上的简单而紧凑的设计，从而对于学习Java是一种非常好的过渡；其次，它的变量类型是采用弱类型，并未使用严格的数据类型。
- 安全性。
JavaScript是一种安全性语言。它不允许访问本地硬盘，并不能将数据存入到服务器上，不允许对网络文档进行修改和删除，只能通过浏览器实现信息浏览或动态交互，从而有效地防止数据丢失。
- 动态性。
JavaScript是动态的，它可以直接对用户或客户输入做出响应，无须经过Web服务程序。它采用以事件驱动的方式对用户的反映做出响应。
- 跨平台性。
JavaScript依赖于浏览器本身，与操作环境无关。只要能运行浏览器的计算机，并支持JavaScript的浏览器就可正确执行。

## JavaScript应用初体验
JavaScript是一种脚本语言，需要浏览器进行解释和执行。下面通过一个简单的例子来体验一下JavaScript脚本程序语言。创建一个HTML文件，示例如下。
```javascript
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 
Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml
" xml:lang="en" lang="en">
<head>
<title>Hello, World!</title>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />

<script type="text/javascript">
alert("Hello, World!");
</script>

</head>
<body>
</body>
</html>
```

## 网页中的JavaScript
在网页中添加JavaScript代码，需要使用标签来标识脚本代码的开始和结束。该标签就是<script>，它告诉浏览器，在<script>标签和</script>标签之间的文本块并不是要显示的网页内容，而是需要处理的脚本代码。

在网页中执行JavaScript代码可以分为以下几种情况，分别是在网页头中执行、在网页中执行、在网页的元素事件中执行JavaScript代码，在网页中调用已经存在的JavaScript文件，以及通过JavaScript伪URL引入JavaScript脚本代码。

### 在网页头中执行JavaScript代码
如果不是通过JavaScript脚本生成HTML网页的内容，JavaScript脚本一般放在HTML网页的头部的<head>与</head>标签对之间。这样，不会因为JavaScript影响整个网页的显示结果。执行JavaScript的格式如下：
```
<head>
<title>Hello, World!</title>
<script type="text/javascript">
   // javascript脚本内容
</script>
</head>
```

### 在网页中执行JavaScript代码
当需要使用JavaScript脚本生成HTML网页内容时，如某些JavaScript实现的动态树，就需要把JavaScript放在HTML网页主题部分的<body>与</body>标签对中。执行JavaScript的格式如下：
```
<body>
<script type="text/javascript">
   // javascript脚本内容
</script>
</body>
```
另外，JavaScript代码可以在同一个HTML网页的头部与主题部分同时嵌入，并且在同一个网页中可以多次嵌入JavaScript代码。

### 在网页的元素事件中执行JavaScript代码
在开发Web应用程序的过程中，开发者可以给HTML文档设置不同的事件处理器，一般是设置某HTML元素的属性来引用一个脚本，如可以是一个简单的动作，该属性一般以on开头，如按下鼠标事件OnClick()等。这样，当需要对HTML网页中的该元素进行事件处理（验证用户输入的值是否有效）时，如果事件处理的JavaScript代码量较少，就可以直接在对应的HTML网页的元素事件中嵌入JavaScript代码。

### 在网页中调用已经存在的JavaScript文件
如果JavaScript的内容较长，或者多个HTML网页中都调用相同的JavaScript程序，可以将较长的JavaScript或者通用的JavaScript写成独立的JavaScript文件，直接在HTML网页中调用。执行JavaScript代码的格式如下：
```
<script src="hello.js"> </script>
```

## JavaScript编码规范
JavaScript编码规范包括以下几个方面，分别是文件组织规范、格式化规范、命名规范、注释规范和其他编码规范。

### 文件组织规范
所有的JavaScript文件都要放在项目公共的script文件夹下。
使用的第三方库文件放置在script/lib文件夹下。
可以复用的自定义模块放置在script/commons文件夹下，复用模块如果涉及多个子文件，需要单独建立模块文件夹。
单独页面模块使用的JavaScript文件放置在script/{module_name}文件夹下。
项目模拟的JSON数据放置在script/json文件夹下，按照页面单独建立子文件夹。
JavaScript应用MVC框架时，使用的模板文件放置在script/templates文件夹下，按照页面单独建立子文件夹。

### 格式化规范
始终使用var定义变量
始终使用分号结束一行声明语句。
对于数组和对象不要使用多余的“,”（兼容IE），例如：
定义顶级命名空间，如inBike，在顶级命名空间下自定义私有命名空间，根据模块分级。
所有的模块代码放在匿名自调用函数中，通过给window对象下的自定义命名空间赋值暴露出来，例如：
绑定事件代码需要放置在准备好的文档对象模型函数中执行，例如：
将自定义模块方法放置在对象中，方法名紧挨“:”，“:”与function之间空一格，function()与后面的“{”之间空一格，例如：
字符串使用单引号，例如：
所用的变量使用之前需要定义，定义之后立即初始化，例如：
使用浏览器Console工具之前先要判断是否支持，例如：

### 命名规范
使用驼峰法命名变量和方法名，首字母使用小写；对于类名，首字母大写，例如：
使用$name命名jQuery对象，原生dom元素使用dom开头，对象中私有变量以下画线开头，例如：

### 注释规范
多使用单行注释表明逻辑块的意义，例如：
指明类的构造方法，例如：
标注枚举常量的类型和意义，例如：
使用注释标识方法或者变量的可见性，方便静态检查，例如：

### 其他编码规范
避免使用eval。
对于对象避免使用with，对于数组避免使用for…in。
谨慎使用闭包，避免循环引用。
警惕this所处的上下文，例如：
尽量使用短码，如三目运算符、逻辑开关、自增运算等，例如：
不要在块级作用域中使用function，例如：
在父节点上绑定事件监听，根据事件源分别响应。
对于复杂的页面模块使用依赖管理库，如SeaJS、RequireJS、MVC框架Backbone、Knockout、Stapes等。




