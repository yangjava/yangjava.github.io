---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码服务异常


## 服务-StatusCheckService
StatusCheckService是用来判断哪些异常不算异常， 核心就是statuscheck.ignored_exceptions和statuscheck.max_recursive_depth两个配置项，

代码如下：
```
/**
 * The <code>StatusCheckService</code> determines whether the span should be tagged in error status if an exception
 * captured in the scope.
 * 用来判断哪些异常不算异常
 */
@DefaultImplementor
public class StatusCheckService implements BootService {

    @Getter
    private String[] ignoredExceptionNames;

    private StatusChecker statusChecker;

    @Override
    public void prepare() throws Throwable {
        // 一条链路如果某个环节出现了异常,默认情况会把异常信息发送给OAP,在SkyWalking UI中看到链路中那个地方出现了异常,方便排查问题
        // 但是在一些场景中,异常是用来控制业务流程的,而不应该把它当做错误
        // 这个配置就是需要忽略的异常
        ignoredExceptionNames = Arrays.stream(Config.StatusCheck.IGNORED_EXCEPTIONS.split(","))
                                      .filter(StringUtil::isNotEmpty)
                                      .toArray(String[]::new);
        // 检查异常时的最大递归程度
        // AException
        //  BException
        //   CException
        // 如果IGNORED_EXCEPTIONS配置的是AException,此时抛出的是CException需要递归找一下是否属于AException的子类
        statusChecker = Config.StatusCheck.MAX_RECURSIVE_DEPTH > 0 ? HIERARCHY_MATCH : OFF;
    }
```












