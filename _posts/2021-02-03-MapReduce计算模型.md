---
layout: post
categories: [Hadoop]
description: none
keywords: Hadoop
---
# MapReduce计算模型
2004年，Google发表了一篇论文，向全世界的人们介绍了MapReduce。现在已经到处都有人在谈论MapReduce（微软、雅虎等大公司也不例外）。在Google发表论文时，MapReduce的最大成就是重写了Google的索引文件系统。而现在，谁也不知道它还会取得多大的成就。MapReduce被广泛地应用于日志分析、海量数据排序、在海量数据中查找特定模式等场景中。Hadoop根据Google的论文实现了MapReduce这个编程框架，并将源代码完全贡献了出来。本章就是要向大家介绍MapReduce这个流行的编程框架。

## 为什么要用MapReduce
MapReduce的流行是有理由的。它非常简单、易于实现且扩展性强。大家可以通过它轻易地编写出同时在多台主机上运行的程序，也可以使用Ruby、Python、PHP和C++等非Java类语言编写Map或Reduce程序，还可以在任何安装Hadoop的集群中运行同样的程序，不论这个集群有多少台主机。MapReduce适合处理海量数据，因为它会被多台主机同时处理，这样通常会有较快的速度。

## MapReduce计算模型
要了解MapReduce，首先需要了解MapReduce的载体是什么。在Hadoop中，用于执行MapReduce任务的机器有两个角色：一个是JobTracker，另一个是TaskTracker。JobTracker是用于管理和调度工作的，TaskTracker是用于执行工作的。一个Hadoop集群中只有一台JobTracker。

## MapReduce Job
在Hadoop中，每个MapReduce任务都被初始化为一个Job。每个Job又可以分为两个阶段：Map阶段和Reduce阶段。这两个阶段分别用两个函数来表示，即Map函数和Reduce函数。Map函数接收一个＜key, value＞形式的输入，然后产生同样为＜key, value＞形式的中间输出，Hadoop会负责将所有具有相同中间key值的value集合到一起传递给Reduce函数，Reduce函数接收一个如＜key，（list of values）＞形式的输入，然后对这个value集合进行处理并输出结果，Reduce的输出也是＜key, value＞形式的。

## Hadoop中的Hello World程序
大家初次接触编程时学习的不论是哪种语言，看到的第一个示例程序可能都是“Hello World”。在Hadoop中也有一个类似于Hello World的程序。这就是WordCount。本节会结合这个程序具体讲解与MapReduce程序有关的所有类。
```java
import java.io.IOException；

import java.util.*；

import org.apache.hadoop.fs.Path；

import org.apache.hadoop.conf.*；

import org.apache.hadoop.io.*；

import org.apache.hadoop.mapred.*；

import org.apache.hadoop.util.*；

public class WordCount{

    public static class Map extends MapReduceBase implements Mapper＜LongWritable，

    Text, Text, IntWritable＞{

        private final static IntWritable one=new IntWritable（1）；

        private Text word=new Text（）；

        public void map（LongWritable key, Text value, OutputCollector＜Text，

        IntWritable＞output, Reporter reporter）throws IOException{

            String line=value.toString（）；

            StringTokenizer tokenizer=new StringTokenizer（line）；

            while（tokenizer.hasMoreTokens（））{

                word.set（tokenizer.nextToken（））；

                output.collect（word, one）；

            }

        }

    }

    public static class Reduce extends MapReduceBase implements Reducer＜Text，

    IntWritable, Text, IntWritable＞{

        public void reduce（Text key, Iterator＜IntWritable＞values, OutputCollector＜Text，

        IntWritable＞output, Reporter reporter）throws IOException{

            int sum=0；

            while（values.hasNext（））{

                sum+=values.next（）.get（）；

            }

            output.collect（key, new IntWritable（sum））；

        }

    }

    public static void main（String[]args）throws Exception{

        JobConf conf=new JobConf（WordCount.class）；

        conf.setJobName（"wordcount"）；

        conf.setOutputKeyClass（Text.class）；

        conf.setOutputValueClass（IntWritable.class）；

        conf.setMapperClass（Map.class）；

        conf.setReducerClass（Reduce.class）；

        conf.setInputFormat（TextInputFormat.class）；

        conf.setOutputFormat（TextOutputFormat.class）；

        FileInputFormat.setInputPaths（conf, new Path（args[0]））；

        FileOutputFormat.setOutputPath（conf, new Path（args[1]））；

        JobClient.runJob（conf）；

    }

}
```
同时，为了叙述方便，设定两个输入文件，如下：
```java
echo"Hello World Bye World"＞file01

echo"Hello Hadoop Goodbye Hadoop"＞file02
```
看到这个程序，相信很多读者会对众多的预定义类感到很迷惑。其实这些类非常简单明了。首先，WordCount程序的代码虽多，但是执行过程却很简单，在本例中，它首先将输入文件读进来，然后交由Map程序处理，Map程序将输入读入后切出其中的单词，并标记它的数目为1，形成＜word，1＞的形式，然后交由Reduce处理，Reduce将相同key值（也就是word）的value值收集起来，形成＜word, list of 1＞的形式，之后将这些1值加起来，即为单词的个数，最后将这个＜key, value＞对以TextOutputFormat的形式输出到HDFS中。


## MapReduce任务的优化
相信每个程序员在编程时都会问自己两个问题“我如何完成这个任务”，以及“怎么能让程序运行得更快”。同样，MapReduce计算模型的多次优化也是为了更好地解答这两个问题。
MapReduce计算模型的优化涉及了方方面面的内容，但是主要集中在两个方面：一是计算性能方面的优化；二是I/O操作方面的优化。这其中，又包含六个方面的内容。
- 任务调度
任务调度是Hadoop中非常重要的一环，这个优化又涉及两个方面的内容。计算方面：Hadoop总会优先将任务分配给空闲的机器，使所有的任务能公平地分享系统资源。I/O方面：Hadoop会尽量将Map任务分配给InputSplit所在的机器，以减少网络I/O的消耗。
- 数据预处理与InputSplit的大小
MapReduce任务擅长处理少量的大数据，而在处理大量的小数据时，MapReduce的性能就会逊色很多。因此在提交MapReduce任务前可以先对数据进行一次预处理，将数据合并以提高MapReduce任务的执行效率，这个办法往往很有效。如果这还不行，可以参考Map任务的运行时间，当一个Map任务只需要运行几秒就可以结束时，就需要考虑是否应该给它分配更多的数据。通常而言，一个Map任务的运行时间在一分钟左右比较合适，可以通过设置Map的输入数据大小来调节Map的运行时间。在FileInputFormat中（除了CombineFileInputFormat），Hadoop会在处理每个Block后将其作为一个InputSplit，因此合理地设置block块大小是很重要的调节方式。除此之外，也可以通过合理地设置Map任务的数量来调节Map任务的数据输入。
- Map和Reduce任务的数量
合理地设置Map任务与Reduce任务的数量对提高MapReduce任务的效率是非常重要的。默认的设置往往不能很好地体现出MapReduce任务的需求，不过，设置它们的数量也要有一定的实践经验。
首先要定义两个概念—Map/Reduce任务槽。Map/Reduce任务槽就是这个集群能够同时运行的Map/Reduce任务的最大数量。比如，在一个具有1200台机器的集群中，设置每台机器最多可以同时运行10个Map任务，5个Reduce任务。那么这个集群的Map任务槽就是12000，Reduce任务槽是6000。任务槽可以帮助对任务调度进行设置。
设置MapReduce任务的Map数量主要参考的是Map的运行时间，设置Reduce任务的数量就只需要参考任务槽的设置即可。一般来说，Reduce任务的数量应该是Reduce任务槽的0.95倍或是1.75倍，这是基于不同的考虑来决定的。当Reduce任务的数量是任务槽的0.95倍时，如果一个Reduce任务失败，Hadoop可以很快地找到一台空闲的机器重新执行这个任务。当Reduce任务的数量是任务槽的1.75倍时，执行速度快的机器可以获得更多的Reduce任务，因此可以使负载更加均衡，以提高任务的处理速度。
- Combine函数
Combine函数是用于本地合并数据的函数。在有些情况下，Map函数产生的中间数据会有很多是重复的，比如在一个简单的WordCount程序中，因为词频是接近与一个zipf分布的，每个Map任务可能会产生成千上万个＜the，1＞记录，若将这些记录一一传送给Reduce任务是很耗时的。所以，MapReduce框架运行用户写的combine函数用于本地合并，这会大大减少网络I/O操作的消耗。此时就可以利用combine函数先计算出在这个Block中单词the的个数。合理地设计combine函数会有效地减少网络传输的数据量，提高MapReduce的效率。
在MapReduce程序中使用combine很简单，只需在程序中添加如下内容：
```java
job.setCombinerClass（combine.class）；

```
在WordCount程序中，可以指定Reduce类为combine函数，具体如下：
```java
job.setCombinerClass（Reduce.class）；
```
- 压缩
编写MapReduce程序时，可以选择对Map的输出和最终的输出结果进行压缩（同时可以选择压缩方式）。在一些情况下，Map的中间输出可能会很大，对其进行压缩可以有效地减少网络上的数据传输量。对最终结果的压缩虽然会减少数据写HDFS的时间，但是也会对读取产生一定的影响，因此要根据实际情况来选择（第7章中提供了一个小实验来验证压缩的效果）。
- 自定义comparator
在Hadoop中，可以自定义数据类型以实现更复杂的目的，比如，当读者想实现k-means算法（一个基础的聚类算法）时可以定义k个整数的集合。自定义Hadoop数据类型时，推荐自定义comparator来实现数据的二进制比较，这样可以省去数据序列化和反序列化的时间，提高程序的运行效率