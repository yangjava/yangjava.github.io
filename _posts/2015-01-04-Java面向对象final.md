---
layout: post
categories: Java
description: none
keywords: Java
---
# Java面向对象final

## final关键字
final关键字的中文含义是“最终的”。在Java中，final关键字可以用来修饰**类、方法和变量（包括成员变量和局部变量）**。

一旦你将引用声明作final，你将不能改变这个引用了，编译器会检查代码，如果你试图将变量再次初始化的话，编译器会报编译错误。

### final类
当用final修饰一个类时，表明这个类不能被继承，也不能产生子类。最常见是就是String类，任何类都无法继承它。

```java
public final class String
        implements java.io.Serializable, Comparable<String>, CharSequence {
    
}
```
如果一个类你永远不会让他被继承（子类继承往往可以重写父类的方法和改变父类属性，会带来一定的安全隐患），就可以用final进行修饰。

注意，final类中的成员变量可以根据需要设置为final，但是它的所有成员方法会被隐式地指定为final方法。在使用final修饰类的时候，要注意谨慎选择，除非这个类真的在以后不会用来继承或者出于安全的考虑，尽量不要将类设计为final类。

### final方法
当父类的方法被final修饰的时候，子类不能重写父类的该方法，比如在Object中，getClass()方法就是final的，我们就不能重写该方法。

```java
public class Object {
    public final native Class<?> getClass();
}
```

如果想禁止该方法在子类中被重写的，可以设置该方法为为final。注意：因为重写的前提是子类可以从父类中继承此方法，如果父类中final修饰的方法同时访问控制权限为private，将会导致子类中不能直接继承到此方法，此时子类中就可以定义相同的方法名和参数。

第二个原因是效率，final方法比非final方法要快，因为在编译的时候已经静态绑定了，不需要在运行时再动态绑定。
（注：类的private方法会隐式地被指定为final方法）

### final变量
final成员变量表示常量，只能被赋值一次，赋值后值不再改变（final要求地址值不能改变）

当final修饰一个基本数据类型时，表示该基本数据类型的值一旦在初始化后便不能发生变化；如果final修饰一个引用类型时，则在对其初始化之后便不能再让其指向其他对象了，但该引用所指向的对象的内容是可以发生变化的。本质上是一回事，因为引用的值是一个地址，final要求值，即地址的值不发生变化。

final修饰一个成员变量（属性），必须要显示初始化。这里有两种初始化方式，一种是在变量声明的时候初始化；第二种方法是在声明变量的时候不赋初值，但是要在这个变量所在的类的所有的构造函数中对这个变量赋初值。

final关键字修饰的变量可以分为属性、局部变量和形参。无论修饰哪种变量，其含义都是相同的，即变量一旦赋值就不能改变。

#### 成员变量
Java中，成员变量分为类变量（static修饰）和实例变量。针对这两种类型的变量赋初值的时机是不同的，类变量可以在声明变量的时候直接赋初值或者在静态代码块中给类变量赋初值。而实例变量可以在声明变量的时候给实例变量赋初值，在非静态初始化块中以及构造器中赋初值。因此类变量有两个时机赋初值，而实例变量则可以有三个时机赋初值。被final修饰的变量必须在上述时机赋初值，否则编译器会报错。总结一下

- final修饰的类变量：必须要在静态初始化块中指定初始值或者声明该类变量时指定初始值，而且只能在这两个地方之一进行指定，一旦赋值后不能再修改。
- final修饰的实例变量：必要要在非静态初始化块，声明该实例变量或者在构造器中指定初始值，而且只能在这三个地方之一进行指定，一旦赋值后不能再修改。

例如：
```java
public class FinalTest {

    // final修饰属性，就是常量
    public final static double PI = 3.14;

    final int x = 100;

    public static void test1(String[] args) {
        // final修饰局部变量
        final int y = 0;
    }

    // final修饰形参
    public static void test2(final int z) {

    }

    public void test3(){
        // 没有在声明的同时赋值
        final int e;
        // 只能赋值一次
        e = 100;
    }

    // 声明的同时赋值
    final int f = 200;
    
}
```

#### 局部变量
final局部变量由程序员进行显式初始化，如果final局部变量已经进行了初始化则后面就不能再次进行更改，如果final变量未进行初始化，可以进行赋值，当且仅有一次赋值，一旦赋值之后再次赋值就会出错。

### 宏变量与宏替换

#### 宏变量
如果一个变量满足一下三个条件时，该变量就会成为一个宏变量，即是一个常量
- 被final修饰符修饰
- 在定义该final变量时就指定了初始值
- 该初始值在编译时就能够唯一指定

```
final String a = "hello";
final String b = a;
final String c = getHello();
```
变量a被final修饰，且初始化的时候就声明了初始值，且该初始值在编译的时候就可以唯一指定，即”hello“，所以a就是一个宏变量。而变量b和c虽然满足一、二条件，但是初始值在编译期间无法唯一确定，所以b和c不是。

#### 宏替换
如果一个变量是宏变量，那么编译器会把程序所有用到该变量的地方直接替换成该变量的值，这就是宏替换

例如
```
public static void main(String[] args) {
    String hw = "hello world";

    String hello = "hello";
    final String finalWorld2 = "hello";//宏变量，值为hello
    final String finalWorld3 = hello;
    final String finalWorld4 = "he" + "llo";//宏变量，值为hello

    String hw1 = hello + " world";
    String hw2 = finalWorld2 + " world";//相当于String hw2 = "hello" + " world";也就相当于String hw2="hello world";
    String hw3 = finalWorld3 + " world";
    String hw4 = finalWorld4 + " world";//相当于String hw4 = "hello" + " world";也就相当于String hw2="hello world";

    System.out.println(hw == hw1); //false
    System.out.println(hw == hw2); //true
    System.out.println(hw == hw3); //false
    System.out.println(hw == hw4); //true
}
```
根据上面对宏变量的分析，我们知道finalWorld2和finalWorld4是属于宏变量，因此后续程序会直接使用其值“hello”来代替finalWorld2和finalWorld4。因此hw2和hw4等同于"hello world"，所以hw=hw2=hw4。

## final使用总结
- final关键字提高了性能。JVM和Java应用都会缓存final变量。
- final变量可以安全的在多线程环境下进行共享，而不需要额外的同步开销。
- 使用final关键字，JVM会对方法、变量及类进行优化。

## 关于final的重要知识点
- final关键字可以用于成员变量、本地变量、方法以及类。
- final成员变量必须在声明的时候初始化或者在构造器中初始化，否则就会报编译错误。
- 你不能够对final变量再次赋值。
- 本地变量必须在声明时赋值。
- 在匿名类中所有变量都必须是final变量。
- final方法不能被重写。
- final类不能被继承。
- final关键字不同于finally关键字，后者用于异常处理。
- final关键字容易与finalize()方法搞混，后者是在Object类中定义的方法，是在垃圾回收之前被JVM调用的方法。
- 接口中声明的所有变量本身是final的。
- final和abstract这两个关键字是反相关的，final类就不可能是abstract的。
- final方法在编译阶段绑定，称为静态绑定(static binding)。
- 没有在声明时初始化final变量的称为空白final变量(blank final variable)，它们必须在构造器中初始化，或者调用this()初始化。不这么做的话，编译器会报错“final变量(变量名)需要进行初始化”。
- 将类、方法、变量声明为final能够提高性能，这样JVM就有机会进行估计，然后优化。
- 按照Java代码惯例，final变量就是常量，而且通常常量名要大写。
- 对于集合对象声明为final指的是引用不能被更改，但是你可以向其中增加，删除或者改变内容。

## final域重排序规则
然而在多线程的层面，final也有其自己的内存语义。主要体现在final域的重排序上，下面我们来介绍final的重排序规则

#### final域为基本类型
```java
public class FinalExample {
    int i; // 普通变量
    final int j; // final变量
    static FinalExample obj;

    public FinalExample() { // 构造函数
        i = 1; // 写普通域
        j = 2; // 写final域
    }

    public static void writer() { // 写线程A执行
        obj = new FinalExample();
    }

    public static void reader() { // 读线程B执行
        FinalExample object = obj; // 读对象引用
        int a = object.i; // 读普通域
        int b = object.j; // 读final域
    }
}
```









































