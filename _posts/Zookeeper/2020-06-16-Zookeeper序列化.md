
---
layout: post
categories: Zookeeper
description: none
keywords: Zookeeper
---
## 序列化

序列化主要在zookeeper.jute包中，其中涉及的主要接口如下

**· InputArchive**

**· OutputArchive**

**· Index**

**· Record**

#### InputArchive

其是所有反序列化器都需要实现的接口，其方法如下

```
public interface InputArchive {
    // 读取byte类型
    public byte readByte(String tag) throws IOException;
    // 读取boolean类型
    public boolean readBool(String tag) throws IOException;
    // 读取int类型
    public int readInt(String tag) throws IOException;
    // 读取long类型
    public long readLong(String tag) throws IOException;
    // 读取float类型
    public float readFloat(String tag) throws IOException;
    // 读取double类型
    public double readDouble(String tag) throws IOException;
    // 读取String类型
    public String readString(String tag) throws IOException;
    // 通过缓冲方式读取
    public byte[] readBuffer(String tag) throws IOException;
    // 开始读取记录
    public void readRecord(Record r, String tag) throws IOException;
    // 开始读取记录
    public void startRecord(String tag) throws IOException;
    // 结束读取记录
    public void endRecord(String tag) throws IOException;
    // 开始读取向量
    public Index startVector(String tag) throws IOException;
    // 结束读取向量
    public void endVector(String tag) throws IOException;
    // 开始读取Map
    public Index startMap(String tag) throws IOException;
    // 结束读取Map
    public void endMap(String tag) throws IOException;
}
```

InputArchive实现类如下

BinaryInputArchive

CsvInputArchive

XmlInputArchive

#### OutputArchive

其是所有序列化器都需要实现此接口，其方法如下。

```
public interface OutputArchive {
    // 写Byte类型
    public void writeByte(byte b, String tag) throws IOException;
    // 写boolean类型
    public void writeBool(boolean b, String tag) throws IOException;
    // 写int类型
    public void writeInt(int i, String tag) throws IOException;
    // 写long类型
    public void writeLong(long l, String tag) throws IOException;
    // 写float类型
    public void writeFloat(float f, String tag) throws IOException;
    // 写double类型
    public void writeDouble(double d, String tag) throws IOException;
    // 写String类型
    public void writeString(String s, String tag) throws IOException;
    // 写Buffer类型
    public void writeBuffer(byte buf[], String tag)
        throws IOException;
    // 写Record类型
    public void writeRecord(Record r, String tag) throws IOException;
    // 开始写Record
    public void startRecord(Record r, String tag) throws IOException;
    // 结束写Record
    public void endRecord(Record r, String tag) throws IOException;
    // 开始写Vector
    public void startVector(List v, String tag) throws IOException;
    // 结束写Vector
    public void endVector(List v, String tag) throws IOException;
    // 开始写Map
    public void startMap(TreeMap v, String tag) throws IOException;
    // 结束写Map
    public void endMap(TreeMap v, String tag) throws IOException;

}
```

OutputArchive的类结构如下

BinaryOutputArchive

CsvOutputArchive

XmlOutputArchive

#### Index

其用于迭代反序列化器的迭代器。

```
public interface Index {
    // 是否已经完成
    public boolean done();
    // 下一项
    public void incr();
}
```

Index的类结构如下

BinaryIndex

```
    static private class BinaryIndex implements Index {
        // 元素个数
        private int nelems;
        // 构造函数
        BinaryIndex(int nelems) {
            this.nelems = nelems;
        }
        // 是否已经完成
        public boolean done() {
            return (nelems <= 0);
        }
        // 移动一项
        public void incr() {
            nelems--;
        }
    }
```

CsxIndex

```
    private class CsvIndex implements Index {
        // 是否已经完成
        public boolean done() {
            char c = '\0';
            try {
                // 读取字符
                c = (char) stream.read();
                // 推回缓冲区 
                stream.unread(c);
            } catch (IOException ex) {
            }
            return (c == '}') ? true : false;
        }
        // 什么都不做
        public void incr() {}
    }
```

XmlIndex

```
    private class XmlIndex implements Index {
        // 是否已经完成
        public boolean done() {
            // 根据索引获取值
            Value v = valList.get(vIdx);
            if ("/array".equals(v.getType())) { // 判断是否值的类型是否为/array
                // 设置索引的值
                valList.set(vIdx, null);
                // 索引加1
                vIdx++;
                return true;
            } else {
                return false;
            }
        }
        // 什么都不做
        public void incr() {}
    }
```

Record

所有用于网络传输或者本地存储的类型都实现该接口，其方法如下

```
public interface Record {
    // 序列化
    public void serialize(OutputArchive archive, String tag)
        throws IOException;
    // 反序列化
    public void deserialize(InputArchive archive, String tag)
        throws IOException;
}
```

所有的实现类都需要实现seriallize和deserialize方法。

#### **示例**

下面通过一个示例来理解OutputArchive和InputArchive的搭配使用。

```


import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.util.Set;
import java.util.TreeMap;

import org.apache.jute.BinaryInputArchive;
import org.apache.jute.BinaryOutputArchive;
import org.apache.jute.Index;
import org.apache.jute.InputArchive;
import org.apache.jute.OutputArchive;
import org.apache.jute.Record;

public class ArchiveTest {
    public static void main( String[] args ) throws IOException {
        String path = "F:\\test.txt";
        // write operation
        OutputStream outputStream = new FileOutputStream(new File(path));
        BinaryOutputArchive binaryOutputArchive = BinaryOutputArchive.getArchive(outputStream);
        
        binaryOutputArchive.writeBool(true, "boolean");
        byte[] bytes = "leesf".getBytes();
        binaryOutputArchive.writeBuffer(bytes, "buffer");
        binaryOutputArchive.writeDouble(13.14, "double");
        binaryOutputArchive.writeFloat(5.20f, "float");
        binaryOutputArchive.writeInt(520, "int");
        Person person = new Person(25, "leesf");
        binaryOutputArchive.writeRecord(person, "leesf");
        TreeMap<String, Integer> map = new TreeMap<String, Integer>();
        map.put("leesf", 25);
        map.put("dyd", 25);
        Set<String> keys = map.keySet();
        binaryOutputArchive.startMap(map, "map");
        int i = 0;
        for (String key: keys) {
            String tag = i + "";
            binaryOutputArchive.writeString(key, tag);
            binaryOutputArchive.writeInt(map.get(key), tag);
            i++;
        }
        
        binaryOutputArchive.endMap(map, "map");
        
        
        // read operation
        InputStream inputStream = new FileInputStream(new File(path));
        BinaryInputArchive binaryInputArchive = BinaryInputArchive.getArchive(inputStream);
        
        System.out.println(binaryInputArchive.readBool("boolean"));
        System.out.println(new String(binaryInputArchive.readBuffer("buffer")));
        System.out.println(binaryInputArchive.readDouble("double"));
        System.out.println(binaryInputArchive.readFloat("float"));
        System.out.println(binaryInputArchive.readInt("int"));
        Person person2 = new Person();
        binaryInputArchive.readRecord(person2, "leesf");
        System.out.println(person2);       
        
        Index index = binaryInputArchive.startMap("map");
        int j = 0;
        while (!index.done()) {
            String tag = j + "";
            System.out.println("key = " + binaryInputArchive.readString(tag) 
                + ", value = " + binaryInputArchive.readInt(tag));
            index.incr();
            j++;
        }
    }
    
    static class Person implements Record {
        private int age;
        private String name;
        
        public Person() {
            
        }
        
        public Person(int age, String name) {
            this.age = age;
            this.name = name;
        }

        public void serialize(OutputArchive archive, String tag) throws IOException {
            archive.startRecord(this, tag);
            archive.writeInt(age, "age");
            archive.writeString(name, "name");
            archive.endRecord(this, tag);
        }

        public void deserialize(InputArchive archive, String tag) throws IOException {
            archive.startRecord(tag);
            age = archive.readInt("age");
            name = archive.readString("name");
            archive.endRecord(tag);            
        }    
        
        public String toString() {
            return "age = " + age + ", name = " + name;
        }
    }
}
```