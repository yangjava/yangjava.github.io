---
layout: post
categories: [XML]
description: none
keywords: XML
---
# JavaDOM解析XML
DOM是由w3c制定的为HTML和XML文档编写的应用程序接口，全称叫做W3C DOM，它使得开发者能够修改html和xml的内容和展现方式，将网页与脚本或编程语言连接起来。

## DOM解析
DOM全称 Document Object Model，文档对象模型。DOM 是 W3C（万维网联盟）的标准。DOM 是 W3C（万维网联盟）的标准。DOM 定义了访问 HTML 和 XML 文档的标准。

W3C 文档对象模型 （DOM） 是中立于平台和语言的接口，它允许程序和脚本动态地访问和更新文档的内容、结构和样式。

## W3C 标准
HTML DOM 定义了所有 HTML 元素的对象和属性，以及访问它们的方法。 是关于如何HTML 元素的增删改查的标准。

XML 即可扩展标记语言（EXtensible Markup Language），类似 HTML。 XML DOM 定义了所有 XML 元素的对象和属性，以及访问它们的方法。

## DOM基本思想
其基本思想为：
- 一次性把xml文档加载进内存
- 在内存中将整个XML数据转换成一个Document树形对象[Document对象]
- 将XML中的标签，属性，文本都作为一个结点对象
- 解析XML的时候，先将整个xml一次性读入到内存中，封装成树对象
- 再对树上的结点进行操作[增删改查]

## 核心操作类或接口
其核心操作类或接口为：

### Document
此接口代表了整个 XML 文档，是整棵 DOM 树的根，
提供了对文档中的数据进行访问和操作的入口，通过 Document 结点可以访问 XML 文件中所有的元素内容。

### 节点类Node
此接口在整个 DOM 树种具有举足轻重的低位，DOM 操作的核心接口中有很大一部分接口是从 Node 接口继承过来的，例如：Document、Element 、Attr 、Text等接口。
在 DOM树中，每一个 Node 接口代表了 DOM 树中的一个结点。
### 节点列表类NodeList
此接口表示的是结点的集合，一般用于列出该结点下的所有子结点。
### 元素类Element
是Node类最主要的子接口，在元素中可以包含属性（Attribute），故Element中有getAttribute(“Element”) 方法用来获得指定名字的属性值。

因为XML文件的数据类型为：标签 属性 文本
故而实际应用时， 整个XML文档被当做一个 Documnet对象。标签是一个 Element对象。 属性是一个 Attr 对象。文本是一个 Text对象。

## java 编码实现解析xml
DOM解析XML是将整个XML作为一个对象，占用内存较多。另外一个java官方的XML解析方式SAX是边扫描边解析，自顶向下依次解析，占用内存较少。

xml文件的内容
```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>
    <person id="1">
        <name>Tom</name>
        <age>20</age>
    </person>
    <person id="2">
        <name>Mary</name>
        <age>25</age>
    </person>
</root>

```

XML与java对象之间的转换
```java
import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException;
import javax.xml.transform.Transformer;
import javax.xml.transform.TransformerException;
import javax.xml.transform.TransformerFactory;
import javax.xml.transform.dom.DOMSource;
import javax.xml.transform.stream.StreamResult;

import org.w3c.dom.Document;
import org.w3c.dom.Element;
import org.w3c.dom.Node;
import org.w3c.dom.NodeList;
import java.io.File;
import java.io.IOException;

public class Test {
    public static void main(String[] args) {
        try {
            // 获取 DOM 解析器工厂实例
            DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();

            // 获取 DOM 解析器
            DocumentBuilder builder = factory.newDocumentBuilder();

            // 解析XML文件
            Document doc = builder.parse(new File("D:\\person.xml"));

            // 获取根节点
            Element root = doc.getDocumentElement();

            // 获取 person 节点列表
            NodeList personList = root.getElementsByTagName("person");

            // 遍历 person 节点列表
            for (int i = 0; i < personList.getLength(); i++) {
                Node node = personList.item(i);

                // 判断当前 person 节点是否为 id=2 的节点
                if (node instanceof Element && "2".equals(((Element) node).getAttribute("id"))) {
                    Element person = (Element) node;
                    // 修改 age 节点的值
                    NodeList ageList = person.getElementsByTagName("age");
                    if (ageList.getLength() > 0) {
                        //获取节点
                        System.out.print(ageList.item(0).getNodeName() + ":");
                        //获取节点值
                        System.out.println(ageList.item(0).getFirstChild().getNodeValue());
                        Element age = (Element) ageList.item(0);
                        age.setTextContent("30");
                    }
                }
            }

            // 将修改后的文档写回到原文件
            TransformerFactory transformerFactory = TransformerFactory.newInstance();
            Transformer transformer = transformerFactory.newTransformer();
            DOMSource source = new DOMSource(doc);
            StreamResult result = new StreamResult(new File("D:\\person_new.xml"));
            transformer.transform(source, result);

            System.out.println("XML文件修改成功！");
        } catch (ParserConfigurationException | IOException | TransformerException | org.xml.sax.SAXException e) {
            e.printStackTrace();
        }
    }
}
```

## java实现对XML格式的验证
可以使用两种验证模式（DTD、XSD）保证XML文件格式正确，DTD和XSD均是XML约束描述语言，是XML文件的验证机制。本文以DTD为例。
DTD文件格式请参考：http://www.cnblogs.com/zhengcheng/p/4278899.html
看下面student.xml:
```
<?xml version="1.0"?>
<!DOCTYPE 学生名册 SYSTEM "student.dtd">
<学生名册>
    <学生 学号="t1">
        <姓名>张三</姓名>
        <性别>男</性别>
        <年龄>20</年龄>
    </学生>
    <学生 学号="t2">
        <姓名>李四</姓名>
        <性别>女</性别>
        <年龄>19</年龄>
    </学生>
</学生名册>
```
我们看到上面这个XML指定的DTD验证文件为student.dtd:
```
<?xml version="1.0" encoding="UTF-8"?>

<!ELEMENT 学生名册  (学生*)>
<!ELEMENT 学生 (姓名,性别,年龄)>
<!ELEMENT 姓名 (#PCDATA)>
<!ELEMENT 性别 (#PCDATA)>
<!ELEMENT 年龄 (#PCDATA)>
<!ATTLIST 学生 学号 ID #REQUIRED>

```

那么java DOM解析XML如何实现验证？

下面使用DOM解析student.xml:
```
public class test {

    public static void main(String[] args) {
        DocumentBuilderFactory buildFactory = DocumentBuilderFactory.newInstance();
        //开启XML格式验证
        buildFactory.setValidating(true);
        try {
            DocumentBuilder build = buildFactory.newDocumentBuilder();
            //指定验证出错处理类MyErrorHandle
            build.setErrorHandler(new MyErrorHandler());
            //自定义解析方式，如果不设置，则使用默认实现
            build.setEntityResolver(new MyResolveEntity());
            Document doc = build.parse("student.xml");

            getStudents(doc);
        } catch (ParserConfigurationException e) {
            e.printStackTrace();
        } catch (SAXException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void getStudents(Document doc) {
        Element root = doc.getDocumentElement();
        NodeList nodeList = root.getElementsByTagName("学生");

        for(int i=0;i<nodeList.getLength();i++){
            Node node = nodeList.item(i);
            NamedNodeMap map = node.getAttributes();
            System.out.println(map.item(0).getTextContent());

            //子节点
            NodeList childList = node.getChildNodes();
            for(int j=0;j<childList.getLength();j++){
                Node childNode = childList.item(j);
                System.out.println(childNode.getTextContent());
            }
        }
    }
}

public class MyErrorHandler implements ErrorHandler{

    @Override
    public void warning(SAXParseException exception) throws SAXException {
        // TODO Auto-generated method stub

    }

    @Override
    public void error(SAXParseException exception) throws SAXException {
        System.out.println("发生了错误！"+exception.getMessage());

    }

    @Override
    public void fatalError(SAXParseException exception) throws SAXException {
        // TODO Auto-generated method stub

    }

}

public class MyResolveEntity implements EntityResolver{

    @Override
    public InputSource resolveEntity(String publicId, String systemId) throws SAXException, IOException {
        return new InputSource("student.dtd");
        //return null;
    }

}
```
如果不设置setEntityResolver，则会使用XML中指定位置的DTD文件进行验证，
```
<!DOCTYPE 学生名册 SYSTEM "student.dtd">
```
student.dtd即指定了验证文件的位置。





























