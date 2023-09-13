---
layout: post
categories: [Java]
description: none
keywords: Java
---
# Java时间Local
LocalDateTime、LocalDate、LocalTime 是Java8全新的日期框架，较之前的util.Date以及Calander使用起来更加的方便直观，下面介绍几种常见的日期对象用法。

## 时间Local
LocalDateTime：日期加时间的日期对象，包含年月日时分秒
LocalDate：日期类，包含年月日
LocalTime：时间类，包含时分秒
LocalDate、LocalTime、LocalDateTime、Instant均为不可变对象，修改这些对象对象会返回一个副本。而并非在原对象上修改。

## LocalDateTime用法
获取当前的时间
```
LocalDateTime now = LocalDateTime.now();
```

根据年月日格式化时间
```
LocalDateTime localDateTime = LocalDateTime.of(2021, 6, 16, 16, 37, 20, 814 * 1000 * 1000);
LocalDateTime ldt = LocalDateTime.now().withYear(2021).withMonth(6).withDayOfMonth(16).withHour(10).withMinute(10).withSecond(59).withNano(999 * 1000 * 1000);
```
注意：最后一个参数是纳秒

获取时区东八区的时间
```
LocalDateTime datetime = LocalDateTime.now(ZoneId.of("Asia/Shanghai"));
LocalDateTime datetime2 = LocalDateTime.now(ZoneId.of("+8"));
```

从LocalDateTime中获取年月日等时间信息
```
LocalDateTime now = LocalDateTime.now();
int year = now.getYear();
int month = now.getMonthValue();
int dayOfYear = now.getDayOfYear();
int dayOfMonth = now.getDayOfMonth();

int hour = now.getHour();
int minute = now.getMinute();
int second = now.getSecond();
int nano = now.getNano();

System.out.println(year + " " + month + " " + dayOfYear + " " + dayOfMonth);
System.out.println(hour + " " + minute + " " + second + " " + nano);
```

从LocalDateTime中获取LocalDate与LocalTime
```
LocalDate localDate = now.toLocalDate();
LocalTime localTime = now.toLocalTime();
```

日期计算
```
LocalDateTime now = LocalDateTime.now();
LocalDateTime tomorrow = now.plusDays(1L);
tomorrow = tomorrow.plusHours(2L);
tomorrow = tomorrow.plusMinutes(10L);

LocalDateTime yesterday = now.minus(Duration.ofDays(1));
yesterday = yesterday.plusHours(2L);
yesterday = yesterday.plusMinutes(10L);

System.out.println(tomorrow);
System.out.println(yesterday);
```
注意：LocalDateTime、LocalDate、LocalTime 日期对象每次修改都会创建一个新的对象，而非在原有对象基础上修改


获取某天的整点时间，通过with修改值
```
LocalDateTime now = LocalDateTime.now();
LocalDateTime ldt1 = now.withHour(10).withMinute(0).withSecond(0).withNano(0);
```

获取毫秒值
```
Instant instant = Instant.now();
long min = instant.toEpochMilli();
System.out.println(min);
```

计算两个时间的时间差
下面这种方式可以获取两个时间点相差N年N月N日N时N分N秒
```
LocalDateTime fromDateTime = LocalDateTime.of(1992, 6, 11, 10, 23, 55);
LocalDateTime toDateTime = LocalDateTime.now();
LocalDateTime tempDateTime = LocalDateTime.from(fromDateTime);

long years = tempDateTime.until(toDateTime, ChronoUnit.YEARS);
tempDateTime = tempDateTime.plusYears(years);

long months = tempDateTime.until(toDateTime, ChronoUnit.MONTHS);
tempDateTime = tempDateTime.plusMonths(months);

long days = tempDateTime.until(toDateTime, ChronoUnit.DAYS);
tempDateTime = tempDateTime.plusDays(days);

long hours = tempDateTime.until(toDateTime, ChronoUnit.HOURS);
tempDateTime = tempDateTime.plusHours(hours);

long minutes = tempDateTime.until(toDateTime, ChronoUnit.MINUTES);
tempDateTime = tempDateTime.plusMinutes(minutes);

long seconds = tempDateTime.until(toDateTime, ChronoUnit.SECONDS);

System.out.println("" + java.time.Duration.between(tempDateTime, toDateTime).toHours());
System.out.println(years + " years "
        + months + " months "
        + days + " days "
        + hours + " hours "
        + minutes + " minutes "
        + seconds + " seconds.");

```

如果只想获取两个时间相差的天数或者小时数，可以使用以下方法
```
LocalDateTime ldt1 = LocalDateTime.of(1992, 6, 11, 10, 23, 55);
LocalDateTime ldt2 = LocalDateTime.now();
System.out.println(Duration.between(ldt1, ldt2).toDays());
System.out.println(Duration.between(ldt1, ldt2).toHours());
```

## LocalDateTime与Date、String互转
LocalDateTime 转 Date
```
public static Date localDateTime2Date(LocalDateTime localDateTime) {
    return Date.from(localDateTime.atZone(ZoneId.systemDefault()).toInstant());
}

```

LocalDateTime 转 String
```
private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
public static String localDateTime2String(LocalDateTime localDateTime) {
    return localDateTime.format(DATE_TIME_FORMATTER);
}
```
注意：DateTimeFormatter是线程安全的类，可以将此类放到常量中。

String 转 LocalDateTime
```
private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
public static LocalDateTime string2LocalDateTime(String str) {
    return LocalDateTime.parse(str, DATE_TIME_FORMATTER);
}

```
注意：SimpleDateFormat是非线程安全的类，在多线程操作时会报错，所以此对象每次用的时候最好新建。或者作为方法入参传递进来。

Date 转 LocalDateTime
```
public static LocalDateTime date2LocalDateTime(Date date) {
    return date.toInstant()
            .atZone(ZoneId.systemDefault())
            .toLocalDateTime();
}
```

Date 转 String
```
public static String date2String(Date date) {
    SimpleDateFormat formatter = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    return formatter.format(date);
}

```

## LocalDate
LocalDate是日期处理类，具体API如下：
```
// 获取当前日期
LocalDate now = LocalDate.now();
// 设置日期
LocalDate localDate = LocalDate.of(2019, 9, 10);
// 获取年
int year = localDate.getYear();     //结果：2019
int year1 = localDate.get(ChronoField.YEAR); //结果：2019
// 获取月
Month month = localDate.getMonth();   // 结果：SEPTEMBER
int month1 = localDate.get(ChronoField.MONTH_OF_YEAR); //结果：9
// 获取日
int day = localDate.getDayOfMonth();   //结果：10
int day1 = localDate.get(ChronoField.DAY_OF_MONTH); // 结果：10
// 获取星期
DayOfWeek dayOfWeek = localDate.getDayOfWeek();   //结果：TUESDAY
int dayOfWeek1 = localDate.get(ChronoField.DAY_OF_WEEK); //结果：2
```

## LocalTime
LocalTime是时间处理类，具体API如下：
```
// 获取当前时间
LocalTime now = LocalTime.now();
// 设置时间
LocalTime localTime = LocalTime.of(13, 51, 10);
//获取小时
int hour = localTime.getHour();    // 结果：13
int hour1 = localTime.get(ChronoField.HOUR_OF_DAY); // 结果：13
//获取分
int minute = localTime.getMinute();  // 结果：51
int minute1 = localTime.get(ChronoField.MINUTE_OF_HOUR); // 结果：51
//获取秒
int second = localTime.getSecond();   // 结果：10
int second1 = localTime.get(ChronoField.SECOND_OF_MINUTE); // 结果：102.3 LocalDateTime
```

LocalDateTime可以设置年月日时分秒，相当于LocalDate+LocalTime，具体API如下：
```
// 获取当前日期时间
LocalDateTime localDateTime = LocalDateTime.now();
// 设置日期
LocalDateTime localDateTime1 = LocalDateTime.of(2019, Month.SEPTEMBER, 10, 14, 46, 56);
LocalDateTime localDateTime2 = LocalDateTime.of(localDate, localTime);
LocalDateTime localDateTime3 = localDate.atTime(localTime);
LocalDateTime localDateTime4 = localTime.atDate(localDate);
// 获取LocalDate
LocalDate localDate2 = localDateTime.toLocalDate();
// 获取LocalTime
LocalTime localTime2 = localDateTime.toLocalTime();

LocalDateTime now = LocalDateTime.now();
//获取年，月，日，时，分，秒
int year = now.get(ChronoField.YEAR);
int month = now.get(ChronoField.MONTH_OF_YEAR);
int day = now.get(ChronoField.DAY_OF_MONTH);
int hour = now.get(ChronoField.HOUR_OF_DAY);
int minute = now.get(ChronoField.MINUTE_OF_HOUR);
int second = now.get(ChronoField.SECOND_OF_MINUTE);

//打印 2020-7-4 18:7:51
System.out.println(year + "-" + month + "-" + day +
" " + hour + ":" + minute + ":" + second);
```





