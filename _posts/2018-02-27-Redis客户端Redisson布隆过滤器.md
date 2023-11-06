---
layout: post
categories: [Redis]
description: none
keywords: Redis
---
# Redis客户端Redisson布隆过滤器


## 什么是布隆过滤器
布隆过滤器（Bloom Filter）是1970年由布隆提出的。它实际上是一个很长的二进制向量和一系列随机映射函数。布隆过滤器可以用于检索一个元素是否在一个集合中。它的优点是空间效率和查询时间都比一般的算法要好的多，缺点是有一定的误识别率和删除困难。


上面这句介绍比较全面的描述了什么是布隆过滤器，如果还是不太好理解的话，就可以把布隆过滤器理解为一个set集合，我们可以通过add往里面添加元素，通过contains来判断是否包含某个元素。

由于本文讲述布隆过滤器时会结合Redis来讲解，因此类比为Redis中的Set数据结构会比较好理解，而且Redis中的布隆过滤器使用的指令与Set集合非常类似（后续会讲到）。

学习布隆过滤器之前有必要先聊下它的优缺点，因为好的东西我们才想要嘛！

布隆过滤器的优点：
- 时间复杂度低，增加和查询元素的时间复杂为O(N)，（N为哈希函数的个数，通常情况比较小）
- 保密性强，布隆过滤器不存储元素本身
- 存储空间小，如果允许存在一定的误判，布隆过滤器是非常节省空间的（相比其他数据结构如Set集合）

布隆过滤器的缺点：
- 有点一定的误判率，但是可以通过调整参数来降低
- 无法获取元素本身
- 很难删除元素

## 布隆过滤器的使用场景
布隆过滤器可以告诉我们 “某样东西一定不存在或者可能存在”，也就是说布隆过滤器说这个数不存在则一定不存，布隆过滤器说这个数存在可能不存在（误判，后续会讲），利用这个判断是否存在的特点可以做很多有趣的事情。
- 解决Redis缓存穿透问题（面试重点）
- 邮件过滤，使用布隆过滤器来做邮件黑名单过滤
- 对爬虫网址进行过滤，爬过的不再爬
- 解决新闻推荐过的不再推荐(类似抖音刷过的往下滑动不再刷到)
- HBase\RocksDB\LevelDB等数据库内置布隆过滤器，用于判断数据是否存在，可以减少数据库的IO请求

## 布隆过滤器的原理
数据结构
布隆过滤器它实际上是一个很长的二进制向量和一系列随机映射函数。以Redis中的布隆过滤器实现为例，Redis中的布隆过滤器底层是一个大型位数组（二进制数组）+多个无偏hash函数。
一个大型位数组（二进制数组）：

多个无偏hash函数：
无偏hash函数就是能把元素的hash值计算的比较均匀的hash函数，能使得计算后的元素下标比较均匀的映射到位数组中。

如下就是一个简单的布隆过滤器示意图，其中k1、k2代表增加的元素，a、b、c即为无偏hash函数，最下层则为二进制数组。

## 空间计算
在布隆过滤器增加元素之前，首先需要初始化布隆过滤器的空间，也就是上面说的二进制数组，除此之外还需要计算无偏hash函数的个数。布隆过滤器提供了两个参数，分别是预计加入元素的大小n，运行的错误率f。布隆过滤器中有算法根据这两个参数会计算出二进制数组的大小l，以及无偏hash函数的个数k。
它们之间的关系比较简单：
- 错误率越低，位数组越长，控件占用较大
- 错误率越低，无偏hash函数越多，计算耗时较长

如下地址是一个免费的在线布隆过滤器在线计算的网址：https://krisives.github.io/bloom-calculator/

## 增加元素
往布隆过滤器增加元素，添加的key需要根据k个无偏hash函数计算得到多个hash值，然后对数组长度进行取模得到数组下标的位置，然后将对应数组下标的位置的值置为1
- 通过k个无偏hash函数计算得到k个hash值
- 依次取模数组长度，得到数组索引
- 将计算得到的数组索引下标位置数据修改为1

例如，key = Liziba，无偏hash函数的个数k=3，分别为hash1、hash2、hash3。三个hash函数计算后得到三个数组下标值，并将其值修改为1.

## 查询元素
布隆过滤器最大的用处就在于判断某样东西一定不存在或者可能存在，而这个就是查询元素的结果。其查询元素的过程如下：

- 通过k个无偏hash函数计算得到k个hash值
- 依次取模数组长度，得到数组索引
- 判断索引处的值是否全部为1，如果全部为1则存在（这种存在可能是误判），如果存在一个0则必定不存在
关于误判，其实非常好理解，hash函数在怎么好，也无法完全避免hash冲突，也就是说可能会存在多个元素计算的hash值是相同的，那么它们取模数组长度后的到的数组索引也是相同的，这就是误判的原因。

例如李子捌和李子柒的hash值取模后得到的数组索引都是1，但其实这里只有李子捌，如果此时判断李子柒在不在这里，误判就出现啦！因此布隆过滤器最大的缺点误判只要知道其判断元素是否存在的原理就很容易明白了！

## 删除元素
布隆过滤器对元素的删除不太支持，目前有一些变形的特定布隆过滤器支持元素的删除！关于为什么对删除不太支持，其实也非常好理解，hash冲突必然存在，删除肯定是很苦难的！

## Java集成Redis使用布隆过滤器
Redis经常会被问道缓存击穿问题，比较优秀的解决办法是使用布隆过滤器，也有使用空对象解决的，但是最好的办法肯定是布隆过滤器，我们可以通过布隆过滤器来判断元素是否存在，避免缓存和数据库都不存在的数据进行查询访问！

在如下的代码中只要通过bloomFilter.contains(xxx)即可，我这里演示的还是误判率！

引入pom依赖
```
<dependency>
  <groupId>org.redisson</groupId>
  <artifactId>redisson-spring-boot-starter</artifactId>
  <version>3.16.0</version>
</dependency>
```

编写测试代码
```
import org.redisson.Redisson;
import org.redisson.api.RBloomFilter;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
 
/**
 * <p>
 *      Java集成Redis使用布隆过滤器防止缓存穿透方案
 * </p>
 *
 */
public class RedisBloomFilterTest {
 
    /** 预计插入的数据 */
    private static Integer expectedInsertions = 10000;
    /** 误判率 */
    private static Double fpp = 0.01;
 
    public static void main(String[] args) {
        // Redis连接配置，无密码
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.211.108:6379");
        // config.useSingleServer().setPassword("123456");
 
        // 初始化布隆过滤器
        RedissonClient client = Redisson.create(config);
        RBloomFilter<Object> bloomFilter = client.getBloomFilter("user");
        bloomFilter.tryInit(expectedInsertions, fpp);
 
        // 布隆过滤器增加元素
        for (Integer i = 0; i < expectedInsertions; i++) {
            bloomFilter.add(i);
        }
 
        // 统计元素
        int count = 0;
        for (int i = expectedInsertions; i < expectedInsertions*2; i++) {
            if (bloomFilter.contains(i)) {
                count++;
            }
        }
        System.out.println("误判次数" + count);
 
    }
 
}
```





