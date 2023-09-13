---
layout: post
categories: [JDK]
description: none
keywords: JDK
---
# JDK源码AutoClose
AutoCloseable意为可自动关闭的。顾名思义，为在try-with-resources语句中使用的“资源”对象（如文件、套接字句柄等）可在块结束后自动关闭，需要实现此接口。

## 源码理解
```
public interface AutoCloseable { /*...*/ }
```
AutoCloseable接口，在try-with-resources代码块退出时，会自动调用接口的close方法，避免资源耗尽的异常等情况。

参考java文档，如下：
```
An object that may hold resources (such as file or socket handles) until it is closed. 
The close() method of an AutoCloseable object is called automatically 
when exiting a try-with-resources block for which the object 
has been declared in the resource specification header. 
This construction ensures prompt release, avoiding resource exhaustion exceptions and errors that may otherwise occur.

```

## 使用
```
public class TestAutoC {

    public void run1(){
        try(ACL acl = new ACL(); BCL bcl = new BCL()){
            acl.run();
            bcl.run();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void run2(){
        ACL acl = new ACL();
        BCL bcl = new BCL();
        try {
            acl.run();
            bcl.run();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void run3(){
        ACL acl;
        BCL bcl;
        try {
            acl = new ACL();
            bcl = new BCL();
            acl.run();
            bcl.run();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void run4(){
        try(ACL acl = new ACL(); BCL bcl = new BCL()){
            acl.run();
            int i = 0;
            if(i == 0){
                throw new RuntimeException("unknown error");
            }
            bcl.run();
        }
        catch (Exception e) {
            e.printStackTrace();
            throw (RuntimeException)e;
        }

    }

    public static void main(String[] args) {
        TestAutoC autoC = new TestAutoC();
        autoC.run1();
        System.out.println("*************");
        autoC.run2();
        System.out.println("*************");
        autoC.run3();
        System.out.println("*************");
        autoC.run4();
    }

}


class ACL implements AutoCloseable{
    {
        System.out.println("ACL instance init");
    }

    public void run(){
        System.out.println("ACL run");
    }

    @Override
    public void close() throws Exception {
        System.out.println("ACL close");
    }
}

class BCL implements AutoCloseable{
    {
        System.out.println("BCL instance init");
    }

    public void run(){
        System.out.println("BCL run");
    }

    @Override
    public void close() throws Exception {
        System.out.println("BCL close");
    }
}

```

```
ACL instance init
BCL instance init
ACL run
BCL run
BCL close
ACL close
*************
ACL instance init
BCL instance init
ACL run
BCL run
*************
ACL instance init
BCL instance init
ACL run
BCL run
*************
ACL instance init
BCL instance init
ACL run
BCL close
ACL close
java.lang.RuntimeException: unknown error
	at com.xiaoxu.test.TestAutoC.run4(TestAutoC.java:48)
	at com.xiaoxu.test.TestAutoC.main(TestAutoC.java:67)
Exception in thread "main" java.lang.RuntimeException: unknown error
	at com.xiaoxu.test.TestAutoC.run4(TestAutoC.java:48)
	at com.xiaoxu.test.TestAutoC.main(TestAutoC.java:67)

```
执行结果可知，在try-with-resources代码块结束时才会自动调用close方法（try代码块中的方法已执行），而仅仅在try代码块中执行（非try-with-resources）不会自动调用close方法，且使用了try-with-resources代码块时，即便catch抛出异常，也会执行close方法：

并且上述close方法调用的顺序是，try-with-resources代码块中顺序定义靠后的资源，优先调用close方法释放。在使用try-with-resources代码块时，须加上catch语句，用于捕获close方法的Exception异常，否则提示：Unhandled exception from auto-closeable resource: java.lang.Exception（或者方法直接显示声明throws Exception，在外层逻辑接收此异常）。

当然，如果资源使用时，需要添加其他操作，那么可以采用在try代码块中嵌入另外一个资源的try代码块的方式，依然可以达到最外层嵌套的try-with-resources代码块最后调用close方法的效果：
```
public void run5() throws Exception{
    try(ACL acl = new ACL()){
        acl.run();
        System.out.println("acl run over");
        try(BCL bcl = new BCL()){
            bcl.run();
        }
        System.out.println("bcl run over");
    }
}

```

执行结果如下：
```
ACL instance init
ACL run
acl run over
BCL instance init
BCL run
BCL close
bcl run over
ACL close

```
需注意，若使用嵌套的try-with-resources代码块释放资源时，若外层调用抛出异常，则内部的try-with-resources代码块还未开始执行，也就不会有资源的定义和释放，若内层try-with-resources代码块抛出异常，也不会影响内层和外层已定义资源的释放，所以不会出现问题，如下可示：

```
public void run6() throws Exception{
    try(ACL acl = new ACL()){
        acl.run();
        int i = 0;
        if(i == 0){
            throw new RuntimeException("unknown error");
        }
        try(BCL bcl = new BCL()){
            bcl.run();
        }
    }
}

```

```
ACL instance init
ACL run
ACL close
Exception in thread "main" java.lang.RuntimeException: unknown error
	at com.xiaoxu.test.TestAutoC.run6(TestAutoC.java:81)
	at com.xiaoxu.test.TestAutoC.main(TestAutoC.java:114)

```

```
public void run7() throws Exception{
    try(ACL acl = new ACL()){
        acl.run();
        int i = 0;
        try(BCL bcl = new BCL()){
            bcl.run();
            if(i == 0){
                throw new RuntimeException("unknown error");
            }
        }
    }
}

```

```
ACL instance init
ACL run
BCL instance init
BCL run
BCL close
ACL close
Exception in thread "main" java.lang.RuntimeException: unknown error
	at com.xiaoxu.test.TestAutoC.run7(TestAutoC.java:70)
	at com.xiaoxu.test.TestAutoC.main(TestAutoC.java:117)

```



























