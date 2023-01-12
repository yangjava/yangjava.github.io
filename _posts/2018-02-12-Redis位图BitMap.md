---
layout: post
categories: Redis
description: none
keywords: Redis
---


### Bitmap(位图)

Bitmap就是位图，其实也就是字节数组（byte array），用一串连续的2进制数字（0或1）表示，每一位所在的位置为偏移(offset)，位图就是用每一个二进制位来存放或者标记某个元素对应的值。通常是用来判断某个数据存不存在的，因为是用bit为单位来存储所以Bitmap本身会极大的节省储存空间。常用命令如下：

| 命令                        | 作用                                                | 时间复杂度 |
| --------------------------- | --------------------------------------------------- | ---------- |
| setbit key offset val       | 给指定key的值的第offset赋值val                      | O(1)       |
| getbit key offset           | 获取指定key的第offset位                             | O(1)       |
| bitcount key start end      | 返回指定key中[start,end]中为1的数量                 | O(n)       |
| bitop operation destkey key | 对不同的二进制存储数据进行位运算(AND、OR、NOT、XOR) | O(n)       |



**应用案例**

有1亿用户，5千万登陆用户，那么统计每日用户的登录数。每一位标识一个用户ID，当某个用户访问我们的网站就在Bitmap中把标识此用户的位设置为1。使用set集合和Bitmap存储的对比：

| 数据类型 | 每个 userid 占用空间                                         | 需要存储的用户量 | 全部占用内存量               |
| -------- | ------------------------------------------------------------ | ---------------- | ---------------------------- |
| set      | 32位也就是4个字节（假设userid用的是整型，实际很多网站用的是长整型） | 50,000,000       | 32位 * 50,000,000 = 200 MB   |
| Bitmap   | 1 位（bit）                                                  | 100,000,000      | 1 位 * 100,000,000 = 12.5 MB |



**应用场景**

- 用户在线状态
- 用户签到状态
- 统计独立用户





### BloomFilter(布隆过滤)

![Redis-BloomFilter](C:/Users/DELL/Downloads/lemon-guide-main/images/Middleware/Redis-BloomFilter.jpg)

当一个元素被加入集合时，通过K个散列函数将这个元素映射成一个位数组中的K个点（使用多个哈希函数对**元素key (bloom中不存value)** 进行哈希，算出一个整数索引值，然后对位数组长度进行取模运算得到一个位置，每个无偏哈希函数都会得到一个不同的位置），把它们置为1。检索时，我们只要看看这些点是不是都是1就（大约）知道集合中有没有它了：

- 如果这些点有任何一个为0，则被检元素一定不在
- 如果都是1，并不能完全说明这个元素就一定存在其中，有可能这些位置为1是因为其他元素的存在，这就是布隆过滤器会出现误判的原因



**应用场景**

- **解决缓存穿透**：事先把存在的key都放到redis的**Bloom Filter** 中，他的用途就是存在性检测，如果 BloomFilter 中不存在，那么数据一定不存在；如果 BloomFilter 中存在，实际数据也有可能会不存
- **黑名单校验**：假设黑名单的数量是数以亿计的，存放起来就是非常耗费存储空间的，布隆过滤器则是一个较好的解决方案。把所有黑名单都放在布隆过滤器中，再收到邮件时，判断邮件地址是否在布隆过滤器中即可
- **Web拦截器**：用户第一次请求，将请求参数放入布隆过滤器中，当第二次请求时，先判断请求参数是否被布隆过滤器命中，从而提高缓存命中率



#### 基于Bitmap数据结构

```java
import com.google.common.base.Preconditions;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.util.Collection;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@Service
public class RedisService {

    @Resource
    private RedisTemplate<String, Object> redisTemplate;

    /**
     * 根据给定的布隆过滤器添加值
     */
    public <T> void addByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
        Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            redisTemplate.opsForValue().setBit(key, i, true);
        }
    }

    /**
     * 根据给定的布隆过滤器判断值是否存在
     */
    public <T> boolean includeByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
        Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            if (!redisTemplate.opsForValue().getBit(key, i)) {
                return false;
            }
        }

        return true;
    }
}
```



#### 基于RedisBloom模块

RedisBloom模块提供了四种数据类型：

- **Bloom Filter (布隆过滤器)**
- **Cuckoo Filter（布谷鸟过滤器）**
- **Count-Mins-Sketch**
- **Top-K**

`Bloom Filter` 和 `Cuckoo` 用于确定（以给定的确定性）集合中是否存在某项。使用 `Count-Min Sketch` 来估算子线性空间中的项目数，使用 `Top-K` 维护K个最频繁项目的列表。

```shell
# 1.git 下载
[root@test ~]# git clone https://github.com/RedisBloom/RedisBloom.git
[root@test ~]# cd redisbloom
[root@test ~]# make

# 2.wget 下载
[root@test ~]# wget https://github.com/RedisBloom/RedisBloom/archive/v2.0.3.tar.gz
[root@test ~]# tar -zxvf RedisBloom-2.0.3.tar.gz
[root@test ~]# cd RedisBloom-2.0.3/
[root@test ~]# make

# 3.修改Redis Conf
[root@test ~]#vim /etc/redis.conf
# 在文件中添加下行
loadmodule /root/RedisBloom-2.0.3/redisbloom.so

# 4.启动Redis server
[root@test ~]# /redis-server /etc/redis.conf
# 或者启动服务时加载os文件
[root@test ~]# /redis-server /etc/redis.conf --loadmodule /root/RedisBloom/redisbloom.so

# 5.测试RedisBloom
[root@test ~]# redis-cli
127.0.0.1:6379> bf.add bloomFilter foo
127.0.0.1:6379> bf.exists bloomFilter foo
127.0.0.1:6379> cf.add cuckooFilter foo
127.0.0.1:6379> cf.exists cuckooFilter foo
```