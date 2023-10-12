---
layout: post
categories: [Hadoop]
description: none
keywords: Hadoop
---
# Hadoop实战HDFS接口

## HDFS API介绍
涉及的主要类：
- Configuration： 该类的对象封转了客户端或者服务器的配置。
- FileSystem： 该类的对象是一个文件系统对象，可以用该对象的一些方法来对文件进行操作，通过 FileSystem 的静态方法 get 获得该对象。
```
FileSystem fs = FileSystem.get(conf);
```
get 方法从 conf 中的一个参数 fs.defaultFS 的配置值判断具体是什么类型的文件系统。如果我们的代码中没有指定 fs.defaultFS，并且工程 classpath 下也没有给定相应的配置，conf 中的默认值就来自于 hadoop 的 jar 包中的 core-default.xml，默认值为：file:///，则获取的将不是一个 DistributedFileSystem 的实例，而是一个本地文件系统的客户端对象。

Java API官方文档：https://hadoop.apache.org/docs/r3.3.1/api/index.html

## Java API
本地访问HDFS最主要的方式是HDFS提供的Java应用程序接口，其他的访问方式都建立在这些应用程序接口之上。为了访问HDFS，HDFS客户端必须拥有一份HDFS的配置文件，也就是hdfs-site.xml文件，以获取NameNode的相关信息，每个应用程序也必须能访问Hadoop程序库JAR文件，也就是在$HADOOP_HOME、$HADOOP_HOME/lib下面的jar文件。

Hadoop是由Java编写的，所以通过Java API可以调用所有的HDFS的交互操作接口，最常用的是FileSystem类，它也是命令行hadoop fs的实现

### java.net.URL
```java
import java.io.IOException;
import java.io.InputStream;
import java.net.MalformedURLException;
import java.net.URL;
import org.apache.hadoop.fs.FsUrlStreamHandlerFactory;
import org.apache.hadoop.io.IOUtils;

public class MyCat {

 　　static{
 　　　　URL.setURLStreamHandlerFactory(new FsUrlStreamHandlerFactory());
 　　}

 　　public static void main(String[] args) throws MalformedURLException, IOException{
 　　　　InputStream in = null;
 　　　　try {
 　　　　　　in = new URL(args[0]).openStream();
 　　　　　　IOUtils.copyBytes(in, System.out, 4096,false);
 　　　　} finally{
 　　　　　　IOUtils.closeStream(in);
 　　　　}
 　　}
}
```
导出为xx.jar文件，执行命令：
```text
hadoop jar xx.jar hdfs://master:9000/user/hadoop/test
```
执行完成后，屏幕上输出HDFS的文件/user/hadoop/test的文件内容。

该程序是从HDFS读取文件的最简单的方式，即用java.net.URL对象打开数据流。代码中，静态代码块的作用是让Java程序识别Hadoop的HDFS url。

### org.apache.hadoop.fs.FileSystem
虽然上面的方式是最简单的方式，但是在实际开发中，访问HDFS最常用的类还是FileSystem类。
```java
import java.io.IOException;
import java.io.InputStream;
import java.net.URI;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;

public class FileSystemCat {

 　　public static void main(String[] args) throws IOException {
 　　　　String uri = "hdfs://master:9000/user/hadoop/test";
 　　　　Configuration conf = new Configuration();
 　　　　FileSystem fs = FileSystem.get(URI.create(uri), conf);

 　　　　InputStream in = null;
 　　　　try {
 　　　　　　in = fs.open(new Path(uri));
 　　　　　　IOUtils.copyBytes(in, System.out, 4096,false);
 　　　　} finally{
 　　　　　　IOUtils.closeStream(in);
 　　　　}
 　　}
}
```
导出为xx.jar文件，上传至集群任意一节点，执行命令：
```text
hadoop jar xx.jar com.hdfsclient.FileSystemCat
```
执行完成后控制台会输出文件内容。
FileSystem类的实例获取是通过工厂方法：
```java
public static FileSystem get(URI uri,Configuration conf) throws IOException
```
其中Configuration对象封装了HDFS客户端或者HDFS集群的配置，该方法通过给定的URI方案和权限来确定要使用的文件系统。得到FileSystem实例之后，调用open()函数获得文件的输入流：
```text
Public FSDataInputStream open(Path f) throws IOException
```

写入文件
```java
import java.io.BufferedInputStream;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.URI;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;

public class FileCopyFromLocal {
 　　 public static void main(String[] args) throws IOException {
 　　　　//本地文件路径
 　　　　String source = "/home/hadoop/test";
 　　　　String destination = "hdfs://master:9000/user/hadoop/test2";
 　　　　InputStream in = new BufferedInputStream(new FileInputStream(source));
 　　　　Configuration conf = new Configuration();
 　　　　FileSystem fs = FileSystem.get(URI.create(destination),conf);
 　　　　OutputStream out = fs.create(new Path(destination));
 　　　　IOUtils.copyBytes(in, out, 4096,true);
 　　 }
}
```
导出为xx.jar文件，上传至集群任意一节点，执行命令：
```java
hadoop jar xx.jar com.hdfsclient.FileCopyFromLocal
```
FileSystem实例的create()方法返回FSDataOutputStream对象，与FSDataInputStream类不同的是，FSDataOutputStream不允许在文件中定位，这是因为HDFS只允许一个已打开的文件顺序写入，或在现有文件的末尾追加数据。

创建HDFS的目录
```java
import java.io.IOException;
import java.net.URI;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

public class CreateDir {

 　　public static void main(String[] args){
 　　　　String uri = "hdfs://master:9000/user/test";
 　　　　Configuration conf = new Configuration();
 　　　　try {
 　　　　　　FileSystem fs = FileSystem.get(URI.create(uri), conf);
 　　　　　　Path dfs=new Path("hdfs://master:9000/user/test");
 　　　　 　　　　fs.mkdirs(dfs);
 　　　　} catch (IOException e) {
 　　　　　　e.printStackTrace();
 　　　　}
　　}
}
```
导出为xx.jar文件，上传至集群任意一节点，执行命令：
```java
hadoop jar xx.jar com.hdfsclient.CreateDir
```
运行完成后可以用命令hadoop dfs -ls验证目录是否创建成功。

删除HDFS上的文件或目录。
```java
import java.io.IOException;
import java.net.URI;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

public class DeleteFile {
　　 public static void main(String[] args){
　　　　String uri = "hdfs://master:9000/user/hadoop/test";
 　　　　Configuration conf = new Configuration();
 　　　　try {
 　　　　　　FileSystem fs = FileSystem.get(URI.create(uri), conf);
 　　　　　　Path delef=new Path("hdfs://master:9000/user/hadoop");
 　　　　　　 boolean isDeleted=fs.delete(delef,false);
 　　　　　　 //是否递归删除文件夹及文件夹下的文件
 　　　　　　 //boolean isDeleted=fs.delete(delef,true);
 　　　　　　 System.out.println(isDeleted);
 　　　　} catch (IOException e) {
 　　　　　　e.printStackTrace();
 　　　　}
　　}
}
```
导出为xx.jar文件，上传至集群任意一节点
```java
hadoop jar xx.jar com.hdfsclient.DeleteFile
```
如果需要递归删除文件夹，则需将fs.delete(arg0, arg1)方法的第二个参数设为true。

查看文件是否存在。
```java
import java.io.IOException;
import java.net.URI;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

public class CheckFileIsExist {
　　public static void main(String[] args){
　　　　String uri = "hdfs://master:9000/user/hadoop/test";
 　　　　Configuration conf = new Configuration();

 　　　　try {
 　　　　　　FileSystem fs = FileSystem.get(URI.create(uri), conf);
 　　　　　　Path path=new Path(url);
 　　　　　　　　boolean isExists=fs.exists(path);
 　　　　　　 　　System.out.println(isExists);
 　　　　} catch (IOException e) {
 　　　　　　e.printStackTrace();
 　　　　}
　　}
}
```
导出为xx.jar文件，上传至集群任意一台节点，执行命令：
```java
hadoop jar xx.jar com.hdfsclient.CheckFileIsExist
```
列出目录下的文件或目录名称
```java
import java.io.IOException;
import java.net.URI;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

public class ListFiles {
　　　　public static void main(String[] args){
　　　　　String uri = "hdfs://master:9000/user";
 　　　　Configuration conf = new Configuration();
 　　　　try {
 　　　　　　 FileSystem fs = FileSystem.get(URI.create(uri), conf);
 　　　　　　 Path path =new Path(uri);
 　　　　　　 FileStatus stats[]=fs.listStatus(path);
 　　　　　　 for(int i = 0; i < stats.length; ++i){
 　　　　　　　　　　System.out.println(stats[i].getPath().toString());
 　　　　　　}
 　　　　　　 fs.close();
 　　　　}　 catch (IOException e) {
 　　　　　　e.printStackTrace();
 　　　　}
　　}
}
```
导出为xx.jar文件，上传至集群任意一节点，执行命令：
```java
hadoop jar xx.jar com.hdfsclient.ListFiles
```
运行后，控制台会打印出/user目录下的目录名称或文件名。

文件存储的位置信息。
```java
import java.io.IOException;
import java.net.URI;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.BlockLocation;
import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

public class LocationFile {
　　public static void main(String[] args){
　　　　String uri = "hdfs://master:9000/user/test/test";
 　　　　Configuration conf = new Configuration();
　　　　try {
 　　　　　　FileSystem fs = FileSystem.get(URI.create(uri), conf);
 　　　　　　Path fpath=new Path(uri);
 　　　　　　 FileStatus filestatus = fs.getFileStatus(fpath);
 　　　　　　 BlockLocation[] blkLocations = fs.getFileBlockLocations(filestatus, 0, 
　　　　　　　　　　　　filestatus.getLen());
 　　　　　　 int blockLen = blkLocations.length;
 　　　　　　 for(int i=0;i<blockLen;i++){
 　　　　　　　　 String[] hosts = blkLocations[i].getHosts();
 　　　　　　　　 System.out.println("block_"+i+"_location:"+hosts[0]);
 　　　　　　 }
 　　　　} catch (IOException e) {
 　　　　　　e.printStackTrace();
 　　　　}
　　}
}
```
导出为xx.jar文件，上传至集群任意一节点，执行命令：
```java
hadoop jar xx.jar com.hdfsclient.LocationFile
```
HDFS的存储由DataNode的块完成，执行成功后，控制台会输出：
```java
block_0_location:slave1
block_1_location:slave2
block_2_location:slave3
```
表示该文件被分为3个数据块存储，存储的位置分别为slave1、slave2、slave3。

### SequenceFile
SequeceFile是HDFS API提供的一种二进制文件支持，这种二进制文件直接将<key, value>对序列化到文件中，所以SequenceFile是不能直接查看的，可以通过hadoop dfs -text命令查看，后面跟要查看的SequenceFile的HDFS路径。
写入SequenceFile。
```java
import java.io.IOException;
import java.net.URI;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.SequenceFile;
import org.apache.hadoop.io.Text;

public class SequenceFileWriter {

　　private static final String[] text = {
 　　　　"两行黄鹂鸣翠柳",
 　　　　"一行白鹭上青天",
 　　　　"窗含西岭千秋雪",
 　　　　"门泊东吴万里船",
 　　};

 　　public static void main(String[] args) {
 　　　　String uri = "hdfs://master:9000/user/hadoop/testseq";
 　　　　Configuration conf = new Configuration();
 　　　　SequenceFile.Writer writer = null;
 　　　　try {
 　　　　　　FileSystem fs = FileSystem.get(URI.create(uri), conf);
 　　　　　　Path path =new Path(uri);
 　　　　　　IntWritable key = new IntWritable();
 　　　　　　Text value = new Text();
 　　　　　　writer = SequenceFile.createWriter(fs, conf, path, key.getClass(), 
　　　　　　　　　　　　　 value.getClass());

 　　　　　　 for (int i = 0;i<100;i++){
 　　　　　　　　key.set(100-i);
 　　　　　　　　value.set(text[i%text.length]);
 　　　　　　　　writer.append(key, value);
 　　　　　　 }

 　　　　} catch (IOException e) {
 　　　　　　 e.printStackTrace();
 　　　　} finally {
 　　　　　　 IOUtils.closeStream(writer);
 　　　　}
 　　}
}
```
可以看到SequenceFile.Writer的构造方法需要指定键值对的类型。如果是日志文件，那么时间戳作为key，日志内容是value是非常合适的。
导出为xx.jar文件，上传至集群任意一节点，执行命令：
```java
hadoop jar xx.jar com.hdfsclient.SequenceFileWriter
```

读取SequenceFile。
```java
import java.io.IOException;
import java.net.URI;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;
import org.apache.hadoop.io.SequenceFile;
import org.apache.hadoop.io.Writable;
import org.apache.hadoop.util.ReflectionUtils;

public class SequenceFileReader {
 　　public static void main(String[] args) {
 　　　　String uri = "hdfs://master:9000/user/hadoop/testseq";
 　　　　Configuration conf = new Configuration();
 　　　　SequenceFile.Reader reader = null;

 　　　　try {
 　　　　　　FileSystem fs = FileSystem.get(URI.create(uri), conf);
 　　　　　　Path path = new Path(uri);
 　　　　　　reader = new SequenceFile.Reader(fs, path, conf);
 　　　　　　Writable key = (Writable) ReflectionUtils.newInstance(reader.getKeyClass(),   
　　　　　　　　　　　　　　　　conf);
 　　　　　　Writable value = (Writable) ReflectionUtils.newInstance(reader.getValueClass(),   
　　　　　　　　　　　　　　　　conf);
 　　　　　　long position = reader.getPosition();　　　　　　
 　　　　　　while(reader.next(key,value)){
 　　　　　　　　System.out.printf("[%s]\t%s\n",key,value);
 　　　　　　　　position = reader.getPosition();
 　　　　　　}
 　　　　} catch (IOException e) {
 　　　　　　e.printStackTrace();
 　　　　} finally {
 　　　　　　IOUtils.closeStream(reader);
 　　　　}
 　　}
}
```

### HDFS创建目录
```
@Test
public void testMkdirs() throws IOException, URISyntaxException, InterruptedException {

    Configuration configuration = new Configuration();
    // FileSystem fs = FileSystem.get(new URI("hdfs://hadoop102:8020"), configuration);
    FileSystem fs = FileSystem.get(new URI("hdfs://192.168.68.101:8020"), configuration,"hadoop");
    // 创建目录
    fs.mkdirs(new Path("/hdfsapi"));
    // 关闭资源
    fs.close();
}
```