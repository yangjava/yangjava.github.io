---
layout: post
categories: [Java]
description: none
keywords: Java
---
# Java时间Instant
Java Instant 是一个日期和时间相关的类，它表示时间轴上的一个点，精确到纳秒。 在 Java 8 中引入了 Instant 类，可以方便地进行时间戳的操作和转换。

## 时间戳
在JAVA8之前的版本，去获取时间戳（毫秒级别）常用的办法有两种
```
// 方法一：构建日期Date类然后调用getTime方法
Date date = new Date();
System.out.println(date.getTime());
 
// 方法二：使用System类静态方法获取
System.out.println(System.currentTimeMillis());
```
由于Date类大部分方法已经废弃，而且上面两种方法的时间戳只能精确到毫秒级别，所以我们有必要了解下jdk1.8推出的Instant类，该类可以将时间戳精确到纳秒级别。

## Instant类
时间点 该类对象表示的是时间线上的一点，这个时间点存在标准的UTC时间，注意这个时间并不是指北京时间或东京时间而是指世界时间。
```
// 获取当前时间 2022-09-26T03:12:58.517Z（比当地时间相差8个小时）
System.out.println(Instant.now());
 
// 获取系统默认时间戳 2022-09-26T11:12:58.517+08:00[Asia/Shanghai]
System.out.println(Instant.now().atZone(ZoneId.systemDefault()));
```

在Instant时间线上存在三个重要的点位，最大点、最小点、原点也就是说小于1970-01-01的时间戳就为负数，超过1970-01-01的时间戳就为正数
```
// 时间线上最大点  +1000000000-12-31T23:59:59.999999999Z
System.out.println(Instant.MAX);

// 时间线上最小点  -1000000000-01-01T00:00:00Z
System.out.println(Instant.MIN);

// 时间线上原点   1970-01-01T00:00:00Z
System.out.println(Instant.EPOCH);

// 输出结果为-8369623
System.out.println(Instant.parse("1969-09-26T03:06:17.323Z").getEpochSecond());

```

## 时间表示
在Instant中采用两个字段表示时间戳
```
/**
 * The number of seconds from the epoch of 1970-01-01T00:00:00Z.
 * 该字段表示Instant时间距离原点1970-01-01T00:00:00Z的时间（单位秒）
 */
private final long seconds;
/**
 * The number of nanoseconds, later along the time-line, from the seconds field.
 * This is always positive, and never exceeds 999,999,999.
 * 该字段表示Instant当前时间的纳秒数这个值不会超过999,999,999，因为1秒=1000_000_000纳秒
 */
private final int nanos;

```

## Instant实例化
普通实例化分为如下几种

- 使用 now() 方法获取当前时间的 Instant 对象
```
// 获取当前时间
Instant instant = Instant.now();
```

- 通过 ofEpochSecond() 或 ofEpochMilli() 方法从时间戳创建 Instant 对象
```
// 构建毫秒级Instant对象，同样从时间1970-01-01T00:00:00Z开始计算(距离原点5000毫秒)
Instant instantFromSeconds = Instant.ofEpochSecond(1684216800);
Instant instantFromMillis = Instant.ofEpochMilli(1684216800000L);
```

- 通过解析字符串创建 Instant 对象
```
// 字符串转Instant
Instant instantFromString = Instant.parse("2022-05-16T12:34:56.789Z");
```

- 还有一种特殊的如下，可以构建纳秒级的Instant对象
```
// 构建纳秒级Instant对象，同样从时间1970-01-01T00:00:00Z开始计算
// 参数：epochSecond（秒）,nanoAdjustment(纳秒)
// 结果为：1970-01-01T00:00:05.000001111Z
Instant instant5 = Instant.ofEpochSecond(5, 1111);
```

不过我们需要注意Instant.ofEpochSecond方法的源码，如下
```
static final long NANOS_PER_SECOND = 1000_000_000L;
/**
 * @param epochSecond  秒从1970-01-01T00:00:00Z开始计算
 * @param nanoAdjustment  纳秒
 */
public static Instant ofEpochSecond(long epochSecond, long nanoAdjustment) {
    // Math.floorDiv是除法运算，返回小于或等于商的整数 Math.floorDiv(25, 3)=8
    // Math.addExact加法运算，Math.addExact(1, 2)=3
    long secs = Math.addExact(epochSecond, Math.floorDiv(nanoAdjustment, NANOS_PER_SECOND));
    // Math.floorMod是模运算，Math.floorMod(9, 20)=9
    int nos = (int)Math.floorMod(nanoAdjustment, NANOS_PER_SECOND);
    return create(secs, nos);
}
```

可以使用 toString() 方法将 Instant 对象格式化为 ISO 8601 格式的字符串。
```
System.out.println(now.toString()); // 输出：2022-05-16T12:34:56.789Z
```
## Instant获取参数
```
Instant instant = Instant.now();
// 时区相差8小时 2022-09-26T07:04:19.110Z
System.out.println(instant);

System.out.println("秒："+instant.getEpochSecond());

System.out.println("毫秒："+instant.toEpochMilli());
// 1毫秒 = 1000 000 纳秒
System.out.println("纳秒："+instant.getNano());

```

## Instant时间点比较
由于时间点位于时间线上，所以可以直接进行对比。
```
Instant instant1 = Instant.parse("2022-09-26T07:04:19.110Z");
Instant instant2 = Instant.parse("2022-09-26T07:04:19.110Z");
Instant instant3 = Instant.parse("2022-08-26T07:04:19.110Z");

// 相等为0
System.out.println(instant1.compareTo(instant2));
// instant1大于instant3 为1
System.out.println(instant1.compareTo(instant3));
// instant1小于instant3 为-1
System.out.println(instant3.compareTo(instant1));

// true
System.out.println(instant1.isAfter(instant3));
// false
System.out.println(instant1.isBefore(instant3));

```

## Instant时间点运算
```
Instant instant1 = Instant.parse("2022-09-26T07:04:19.110Z");

// 在instant1的基础上增加2秒，值为：2022-09-26T07:04:21.110Z
System.out.println(instant1.plusSeconds(2));

// 在instant1的基础上增加1毫秒，值为：2022-09-26T07:04:19.111Z
System.out.println(instant1.plusMillis(1));

// 在instant1的基础上增加1001纳秒，值为：2022-09-26T07:04:19.110001001Z
System.out.println(instant1.plusNanos(1001));

// 在instant1的基础上增加1秒，值为：2022-09-26T07:04:20.110Z
// 该值取决于后面指定的单位，可以从ChronoUnit枚举类获取
System.out.println(instant1.plus(1, ChronoUnit.SECONDS));

// 在instant1的基础上减去1秒，值为：2022-09-26T07:04:18.110Z
// plus是增加，minus是减少，逻辑类似可以参考上面plus相关A
System.out.println(instant1.minusSeconds(1));

```
Instant时间点计算时需要注意，无论是调用plus或者minus相关API都会重新创建新对象。

## Instant 对象转换
还可以将 Instant 对象转换为其他日期时间相关类的对象，例如 LocalDateTime、ZonedDateTime 和 OffsetDateTime。
```
Instant now = Instant.now();
LocalDateTime localDateTime = LocalDateTime.ofInstant(now, ZoneId.systemDefault());
ZonedDateTime zonedDateTime = ZonedDateTime.ofInstant(now, ZoneId.of("America/New_York"));
OffsetDateTime offsetDateTime = OffsetDateTime.ofInstant(now, ZoneOffset.ofHours(8));
```



















































