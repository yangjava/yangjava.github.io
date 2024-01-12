---
layout: post
categories: [Java]
description: none
keywords: Java
---
# Java比较compare


## 
java异常：Comparison method violates its general contract解决

Comparison method violates its general contract，是因为sort排序中重写compare方法引发的异常，在sort排序中重写的方法一定要满足:可逆比较
```
Comparator<Integer> c = (o1, o2) -> { if (o1 > o2) { return 1; } else { return -1; } };
```
如上面的比较器就没有满足可逆性，当o1和o2相等时，o1和o2比较，返回-1，表示o1小于o2；但是当这两个元素交换位置时，o2比o1，结果返回还是-1，表示o2小于o1。这样就有两个元素互换比较，o1<o2并且o2<o1这两个结果相互矛盾，在某些情况下会出现异常。

因为JDK7以后，Arrays.sort方法换了排序方式，使用TimSort来进行排序，新的实现在自定义比较器违背比较规则的情况下有可能会抛出异常，原来的实现则是忽略了这个异常。所以为了保证不抛出异常，对比较器的比较规则要求比较严格：

- 自反性：x与y的比较结果和y与x的比价结果相反

- 传递性：如果x>y并且y>z，那么x>z

- 对称性：如果x=y，那么x与z的比较结果和y与z的比较结果相同

分析了一下代码，代码里确实写了一段排序，并且没有加等于号
```

items = items.stream().sorted(
                    Comparator.comparing(ContentVo::getProperties, (x, y) -> {
                        if (Integer.valueOf(x.get("status").toString()) > Integer.valueOf(y.get("status").toString())) {
                            return -1;
                        } else {
                            return 1;
                        }
                    }).thenComparing(ContentVo::getProperties, (x, y) -> {
                        Date date1 = new Date();
                        Date date2 = new Date();
                        try {
                            date1 = DateUtils.parseDate(x.get("createTime").toString());
                            date2 = DateUtils.parseDate(y.get("createTime").toString());
                        } catch (ParseException e) {
                            e.printStackTrace();
                        }
                        if (date1.before(date2)) {
                            return 1;
                        } else {
                            return -1;
                        }
                    })
            ).collect(Collectors.toList());
```
于是修改了一下

解决方法

想要真正解决这个问题，并不是像很多网上帖子上所说加个等于条件就可以，这个是因为逆比较引发的问题，就应该从根源上解决这个问题：让compare方法在逆比较时不会出现矛盾





