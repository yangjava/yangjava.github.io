---
layout: post
categories: [Python]
description: none
keywords: Python
---
# ScrapyShell终端
Scrapy shell即scrapy 终端。那么学习scrapy 终端的目的是什么？以及如何使用？我们带着问题往下看。

## 介绍
Scrapy shell的本质目的是为了快速调式代码，而不用去运行蜘蛛了。当然你也可以测试其他的python代码，因为它是一个常规的Python终端。

Shell是用来测试XPath或CSS表达式，查看其的工作方式及从爬取的网页中提取的数据。在编写spider时，还提供了交互性测试表达式的功能，避免每次修改后运行spider的麻烦。

一般建议安装IPython，Scrapy终端将使用 IPython (替代标准Python终端)。IPython 终端与其他相比更为强大，提供智能的自动补全，高亮输出，及其他特性。特别是在Unix系统(IPython 在Unix下工作的很好)。

一旦你熟悉了 Scrapy Shell，你就会发现它是开发和调试蜘蛛的宝贵工具。

## 启动终端
可以使用shell来启动Scrapy 终端：
```
scrapy shell <url>
```
Windows 环境直接打开cmd执行就行了，但是前提是安装好Python环境。

使用终端

Scrappy shell是一个普通的python控制台（或者 IPython 控制台），它提供一些额外的快捷功能，以方便。示例如下：
```
scrapy shell https://www.baidu.com
```

## 可用快捷方式
- shelp() -打印有关可用对象和快捷方式列表的帮助
- fetch(request_or_url) -从给定的请求中获取新的响应，并相应地更新所有相关对象。
- view(response) -在本地Web浏览器中打开给定的response。这将在body标签中添加一个 tag ，以便外部链接（如图像和样式表）正确显示。值得注意得是，这将在您的计算机中创建一个临时文件，并且该文件不会自动删除。

## 可用的Scrapy对象
Scrapy Shell自动从下载的页面创建一些方便的对象，例如 Response 对象与 Selector 对象（用于HTML和XML内容）。如下：

- crawler -当前 Crawler 对象。
- spider -用于处理URL的蜘蛛， 对当前URL没有处理的蜘蛛时则为一个 Spider 对象。。
- request -当前获取Url的 Request 对象。也可以使用使用 fetch 捷径修改request。
- response -当前获取Url的 Response 对象
- settings - 当前的 Scrapy settings

## 示例
下面以百度首页为例：

- 启动终端
```
scrapy shell https://www.baidu.com
```

- 获取页面的title
```
response.xpath('//title/text()').get()
```

- 修改request
```
fetch("https://www.zhihu.com/people/10xiansheng/posts")
```

## 从spider调用shell
有时想在spider的某个位置中查看被处理的response， 以确认到达期望的response。

这可以通过 scrapy.shell.inspect_response 函数来实现。
```
import scrapy  
  
class mySpider(scrapy.Spider):  
    name="mySpider"  
    start_urls=[  
  
            "http://www.zhihu.com/people/10xiansheng/posts?page=1",  
        ]  
  
    def parse(self, response):  
        from scrapy.shell import inspect_response  
        inspect_response(response, self)
```
先使用命令执行蜘蛛，就会自动启动终端，查看当前的response.url为当前蜘蛛执行的url。

需要注意的是：此时执行fetch命令是无效的，因为此时的终端屏蔽了Scrapy引擎。退出该终端时会继续从蜘蛛停止的地方恢复执行。















