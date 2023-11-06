---
layout: post
categories: [JVM]
description: none
keywords: JVM
---
# HotSpot的JNI详解
JNI调用如何使用，JNI的库文件是如何加载的，下面来详细探讨下JNI API，这API是做什么的，有啥注意事项，这是后续JNI开发的基础。

## 数据类型
Java的数据类型分为基本类型（primitive type，又称原生类型或者原始类型）和引用类型（reference type）,其中基本类型又分为数值类型，boolean类型和returnAddress类型三类。
returnAddress类型在Java语言中没有对应的数据类型，由JVM使用表示指向某个字节码的指针。JVM定义了boolean类型，但是对boolean类型的支持非常有限，boolean类型没有任何专供boolean值使用的字节码指令，

java语言表达式操作boolean值，都是使用int类型对应的字节码指令完成的，boolean数组的访问修改共用byte数组的baload和bstore指令；

JVM规范中明确了1表示true，0表示false，但是未明确boolean类型的长度，Hotspot使用C++中无符号的char类型表示boolean类型，即boolean类型占8位。数值类型分为整数类型和浮点数类型，如下：

整数类型包含：
byte类型：值为8位有符号二进制补码整数，默认值为0
short类型：值为16位有符号二进制补码整数，默认值为0
int类型：值为32位有符号二进制补码整数，默认值为0
long类型：值为64位有符号二进制补码整数，默认值为0
char类型：值为16位无符号整数，用于表示指向多文种平面的Unicode码点，默认值是Unicode的null码点（‘\u0000’）

浮点类型包括：
float类型：值为单精度浮点数集合中的元素，如果虚拟机支持的话是单精度扩展指数集合中的元素，默认值是正数0
double类型：值为双精度浮点数集合中的元素，如果虚拟机支持的话是双精度扩展指数集合中的元素，默认值是正数0
float类型和double类型的长度在JVM规范中未明确说明，在Hotspot中float和double就是C++中对应的float和double类型，所以长度分别是32位和64位，所谓双精度是相对单精度而言的，即用于存储小数部分的位数更多

引用类型在JVM中有三种，类类型（class type），数组类型（array type）和接口类型（interface type），数组类型最外面的一维元素的类型称为该数组的组件类型，组件类型也可以是数组类型，如果组件类型不是元素类型则称为该数组的元素类型，引用类型其实就是C++中的指针。

JVM规范中并没有强引用，软引用，弱引用和虚引用的概念，JVM定义的引用就是强引用，软引用，弱引用和虚引用是JDK结合垃圾回收机制提供的功能支持而已。

## JNI数据类型
JNI数据类型其实就是Java数据类型在Hotspot中的具体表示或者对应的C/C++类型，类型的定义参考OpenJDK hotspot/src/share/prims/jni.h中，如下图：
```
/*
 * JNI Types
 */

#ifndef JNI_TYPES_ALREADY_DEFINED_IN_JNI_MD_H

typedef unsigned char   jboolean;
typedef unsigned short  jchar;
typedef short           jshort;
typedef float           jfloat;
typedef double          jdouble;

typedef jint            jsize;

#ifdef __cplusplus

class _jobject {};
class _jclass : public _jobject {};
class _jthrowable : public _jobject {};
class _jstring : public _jobject {};
class _jarray : public _jobject {};
class _jbooleanArray : public _jarray {};
class _jbyteArray : public _jarray {};
class _jcharArray : public _jarray {};
class _jshortArray : public _jarray {};
class _jintArray : public _jarray {};
class _jlongArray : public _jarray {};
class _jfloatArray : public _jarray {};
class _jdoubleArray : public _jarray {};
class _jobjectArray : public _jarray {};

typedef _jobject *jobject;
typedef _jclass *jclass;
typedef _jthrowable *jthrowable;
typedef _jstring *jstring;
typedef _jarray *jarray;
typedef _jbooleanArray *jbooleanArray;
typedef _jbyteArray *jbyteArray;
typedef _jcharArray *jcharArray;
typedef _jshortArray *jshortArray;
typedef _jintArray *jintArray;
typedef _jlongArray *jlongArray;
typedef _jfloatArray *jfloatArray;
typedef _jdoubleArray *jdoubleArray;
typedef _jobjectArray *jobjectArray;

#else

struct _jobject;

typedef struct _jobject *jobject;
typedef jobject jclass;
typedef jobject jthrowable;
typedef jobject jstring;
typedef jobject jarray;
typedef jarray jbooleanArray;
typedef jarray jbyteArray;
typedef jarray jcharArray;
typedef jarray jshortArray;
typedef jarray jintArray;
typedef jarray jlongArray;
typedef jarray jfloatArray;
typedef jarray jdoubleArray;
typedef jarray jobjectArray;

#endif

typedef jobject jweak;

typedef union jvalue {
    jboolean z;
    jbyte    b;
    jchar    c;
    jshort   s;
    jint     i;
    jlong    j;
    jfloat   f;
    jdouble  d;
    jobject  l;
} jvalue;

struct _jfieldID;
typedef struct _jfieldID *jfieldID;

struct _jmethodID;
typedef struct _jmethodID *jmethodID;

/* Return values from jobjectRefType */
typedef enum _jobjectType {
     JNIInvalidRefType    = 0,
     JNILocalRefType      = 1,
     JNIGlobalRefType     = 2,
     JNIWeakGlobalRefType = 3
} jobjectRefType;


#endif /* JNI_TYPES_ALREADY_DEFINED_IN_JNI_MD_H */

/*
 * jboolean constants
 */

#define JNI_FALSE 0
#define JNI_TRUE 1
```
部分类型跟CPU架构相关的，通过宏定义jni_md.h引入
```
#ifdef TARGET_ARCH_x86
# include "jni_x86.h"
#endif
#ifdef TARGET_ARCH_sparc
# include "jni_sparc.h"
#endif
#ifdef TARGET_ARCH_zero
# include "jni_zero.h"
#endif
#ifdef TARGET_ARCH_arm
# include "jni_arm.h"
#endif
#ifdef TARGET_ARCH_ppc
# include "jni_ppc.h"
#endif
```
通常的服务器都是x86_64架构的，其定义的类型如下：
```
  #define JNICALL
  typedef int jint;
#if defined(_LP64)
  typedef long jlong;
#else
  typedef long long jlong;
#endif

#else
  #define JNIEXPORT __declspec(dllexport)
  #define JNIIMPORT __declspec(dllimport)
  #define JNICALL __stdcall

  typedef int jint;
  typedef __int64 jlong;
#endif

typedef signed char jbyte;
```
从上面的定义可以得出，JVM中除基本数据类型外，所有的引用类型都是指针，JVM这里只是定义了空白的类来区分不同的引用类型，具体处理指针时会将指针强转成合适的数据类型，如jobject指针会强转成Oop指针

Java同JNI数据类型的对应关系
怎么去验证Java数据类型和JNI数据类型的对应关系了？可以通过javah生成的本地方法头文件，比对Java方法和对应的本地方法的参数可以看出两者的对应关系，如下示例：
```
package jni;
 
import java.util.List;
 
public class JniTest{
 
    static
    {
        System.loadLibrary("HelloWorld");
    }
 
    public native static void say(boolean a, byte b, char c, short d, int e, long f, float g, double h, String s, List l,Throwable t,Class cl,
                                  boolean[] a2, byte[] b2, char[] c2, short[] d2, int[] e2, long[] f2, float[] g2, double[] h2,String[] s2);
 
}
```
生成的jni_JniTest.h头文件如下：
```
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class jni_JniTest */
 
#ifndef _Included_jni_JniTest
#define _Included_jni_JniTest
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     jni_JniTest
 * Method:    say
 * Signature: (ZBCSIJFDLjava/lang/String;Ljava/util/List;Ljava/lang/Throwable;Ljava/lang/Class;[Z[B[C[S[I[J[F[D[Ljava/lang/String;)V
 */
JNIEXPORT void JNICALL Java_jni_JniTest_say
  (JNIEnv *, jclass, jboolean, jbyte, jchar, jshort, jint, jlong, jfloat, jdouble, jstring, jobject, jthrowable, jclass, jbooleanArray, jbyteArray, jcharArray, jshortArray, jintArray, jlongArray, jfloatArray, jdoubleArray, jobjectArray);
 
#ifdef __cplusplus
}
#endif
#endif
```
本地方法的第一个入参都是JNIEnv指针，第二个入参根据本地方法是否是静态方法区别处理，如果是静态方法，第二个入参是该类的Class即jclass，如果是普通方式则是当前类实例的引用即jobject，其他的入参跟Java方法的入参一样，将Java的数据类型映射到JNI数据类型即可。

在Java代码调用本地方法同调用Java一样都是值传递，基本类型参数传递的是参数值，对象类型参数传递的是对象引用，JVM必须跟踪所有传递到本地方法的对象引用，确保所引用的对象没有被垃圾回收掉，本地代码也需要及时通知JVM回收某个对象，垃圾回收器需要能够回收本地代码不再引用的对象。

| Java数据类型  | JNI数据类型       | x86 C++类型         | 长度    |
|-----------|---------------|-------------------|-------|
| boolean   | jboolean      | unsigned char     | 8     |
| byte      | jbyte         | signed char       | 8     |
| char      | jchar         | unsigned short    | 16    |
| short     | jshort        | short             | 16    |
| int       | jint          | int               | 32    |
| long      | jlong         | long long         | 64    |
| float     | jfloat        | float             | 32    |
| double    | jdouble       | double            | 64    |
| String    | jstring       | \_jstring *       | 32/64 |
| Class     | jclass        | \_jclass *        | 32/64 |
| Throwable | jthrowable    | \_jthrowable *    | 32/64 |
| boolean[] | jbooleanArray | \_jbooleanArray * | 32/64 |
| byte[]    | jbyteArray    | \_jbyteArray *    | 32/64 |
| char[]    | jcharArray    | \_jcharArray *    | 32/64 |
| short[]   | jshortArray   | \_jshortArray *   | 32/64 |
| int[]     | jintArray     | \_jintArray *     | 32/64 |
| long[]    | jlongArray    | \_jlongArray *    | 32/64 |
| float[]   | jfloatArray   | \_jfloatArray *   | 32/64 |
| double[]  | jdoubleArray  | \_jdoubleArray *  | 32/64 |
| Object[]  | jobjectArray  | \_jobjectArray *  | 32/64 | 


## API定义
JNI标准API的发展
早期不同厂商的JVM实现提供的JNI API接口有比较大的差异，这些差异导致开发者必须编写不同的代码适配不同的平台，简单的介绍下这些API接口：

- JDK 1.0 Native Method Interface
该版本的API因为依赖保守的垃圾回收器且通过C结构体的方式访问Java对象的字段而被废弃

- Netscape's Java Runtime Interface
Netscape就是大名鼎鼎的创造了JavaScript语言的网景公司，他制定了一套通用的API（简称JRI），并在设计之初就考虑了可移植性，但是存在诸多的争议。

- Microsoft's Raw Native Interface and Java/COM interface
微软的JVM实现提供两种本地接口，Raw Native Interface (RNI)和Java/COM interface，前者很大程度了保持了对JDK JNI API接口的向后兼容，与之最大的区别是本地代码必须通过RNI接口同垃圾回收器交互；后者是一个语言独立的本地接口，Java代码可以同COM对象交互，一个Java类可以对外暴露成一个COM类。

这些API经过各厂商充分讨论，因为各种问题最终没有成为标准API

## JNIEnv和JavaVM定义
本地代码调用JNI API的入口只有两个JNIEnv和JavaVM类，这两个都在jni.h中定义，如下：
```
struct JNIEnv_ {
    const struct JNINativeInterface_ *functions;
#ifdef __cplusplus

    jint GetVersion() {
        return functions->GetVersion(this);
    }
    jclass DefineClass(const char *name, jobject loader, const jbyte *buf,
                       jsize len) {
        return functions->DefineClass(this, name, loader, buf, len);
    }
    jclass FindClass(const char *name) {
        return functions->FindClass(this, name);
    }
    jmethodID FromReflectedMethod(jobject method) {
        return functions->FromReflectedMethod(this,method);
    }
```

```
struct JavaVM_ {
    const struct JNIInvokeInterface_ *functions;
#ifdef __cplusplus

    jint DestroyJavaVM() {
        return functions->DestroyJavaVM(this);
    }
    jint AttachCurrentThread(void **penv, void *args) {
        return functions->AttachCurrentThread(this, penv, args);
    }
    jint DetachCurrentThread() {
        return functions->DetachCurrentThread(this);
    }

    jint GetEnv(void **penv, jint version) {
        return functions->GetEnv(this, penv, version);
    }
    jint AttachCurrentThreadAsDaemon(void **penv, void *args) {
        return functions->AttachCurrentThreadAsDaemon(this, penv, args);
    }
#endif
};

```

两者的实现其实是对结构体JNINativeInterface_和JNIInvokeInterface_的简单包装而已，两者定义如下：
```
struct JNINativeInterface_ {
    void *reserved0;
    void *reserved1;
    void *reserved2;

    void *reserved3;
    jint (JNICALL *GetVersion)(JNIEnv *env);

    jclass (JNICALL *DefineClass)
      (JNIEnv *env, const char *name, jobject loader, const jbyte *buf,
       jsize len);
    jclass (JNICALL *FindClass)
      (JNIEnv *env, const char *name);

    jmethodID (JNICALL *FromReflectedMethod)
      (JNIEnv *env, jobject method);
    jfieldID (JNICALL *FromReflectedField)
      (JNIEnv *env, jobject field);
```

```
struct JNIInvokeInterface_ {
    void *reserved0;
    void *reserved1;
    void *reserved2;

    jint (JNICALL *DestroyJavaVM)(JavaVM *vm);

    jint (JNICALL *AttachCurrentThread)(JavaVM *vm, void **penv, void *args);

    jint (JNICALL *DetachCurrentThread)(JavaVM *vm);

    jint (JNICALL *GetEnv)(JavaVM *vm, void **penv, jint version);

    jint (JNICALL *AttachCurrentThreadAsDaemon)(JavaVM *vm, void **penv, void *args);
};

```
从定义上可以看出两者的结构类似于C++中的虚函数表，结构体中没有定义方法而是方法指针，这样一方面实现了C++下对C的兼容，C的结构体中不能定义方法但是可以定义方法指针，C++的结构体基本被扩展成class，可以定义方法和继承；另一方面这种做法实现了接口与实现分离的效果，调用API的本地代码与JVM中具体的实现类解耦，虚拟机可以基于此结构轻松的提供两种实现版本的JNI API，比如其中一个对入参严格校验，另一个只做关键参数最少的校验，让方法指针指向不同的实现即可，API调用方完全无感知。

其中的JNI interface pointer就是传入本地方法的参数JNIEnv指针， 该指针指向的JNIEnv对象本身包含了一个指向JNINativeInterface_结构体的指针，即图中的Pointer，JNINativeInterface_结构体在内存中相当于一个指针数组，即图中的Array of pointers to JNI functions，指针数组中的每个指针都是具体的方法实现的指针。注意JVM规范要求同一个线程内多次JNI调用接收的JNIEnv或者JavaVM指针都是同一个指针，且该指针只在该线程内有效，因此本地代码不能讲该指针从当前线程传递到另一个线程中。

## 异常处理
所有的JNI方法同大多数的C库函数一样不检查传入的参数的正确性，这点由调用方负责检查，如果参数错误可能导致JVM直接宕机。大多数情况下，JNI方法通过返回一个特定的错误码或者抛出一个Java异常的方式报错，调用方可以通过ExceptionOccurred()方法判断是否发生了异常，如本地方法调用Java方法，判断Java方法执行期间是否发生了异常，并通过该方法获取异常的详细信息。

JNI允许本地方法抛出或者捕获Java异常，未被本地方法捕获的异常会向上传递给方法的调用方。本地方法有两种方式处理异常，一种是直接返回，导致异常在调用本地方法的Java方法中被抛出；一种是调用ExceptionClear()方法清除这个异常，然后继续执行本地方法的逻辑。当异常产生，本地方法必须先清除该异常才能调用其他的JNI方法，当异常尚未处理时，只有下列方法可以被安全调用：
- ExceptionOccurred()
- ExceptionDescribe()
- ExceptionClear()
- ExceptionCheck()
- ......

异常处理相关API如下：
- jint Throw(JNIEnv *env, jthrowable obj); 重新抛出Java异常
- jint ThrowNew(JNIEnv *env, jclass clazz,const char *message); 抛出一个指定类型和错误提示的异常
- jthrowable ExceptionOccurred(JNIEnv *env); 判断当前线程是否存在未捕获的异常，如果存在则返回该异常对象，不存在返回NULL，仅限于A方法调用B方法接受后，A方法调用此方法判断B方法是否抛出异常的场景，B方法可以本地或者Java方法。
- void ExceptionDescribe(JNIEnv *env); 将当前线程的未捕获异常打印到系统标准错误输出流中如stderr，如果该异常是Throwable的子类则实际调用该类的printStackTrace方法打印。注意执行过程中获取到异常实例jthrowable后就会调用ExceptionClear清除
- void ExceptionClear(JNIEnv *env); 清除当前线程的未捕获异常
- void FatalError(JNIEnv *env, const char *msg); 抛出一个致命异常，会直接导致JVM宕机
- jboolean ExceptionCheck(JNIEnv *env); 检查当前线程是否存在异常，不返回具体的异常对象，若存在返回true

```
package jni;
 
public class ThrowTest {
 
    static {
        System.load("/home/openjdk/cppTest/ThrowTest.so");
    }
 
 
    public static native void rethrowException(Exception e);
 
    public static native void handlerException();
 
    public static void main(String[] args) {
        handlerException();
        rethrowException(new UnsupportedOperationException("Unsurpported ThrowTest"));
    }
 
 
}
```

```
#include "ThrowTest.h"
#include <stdio.h>
 
JNIEXPORT void JNICALL Java_jni_ThrowTest_rethrowException(JNIEnv * env,
		jclass cls, jthrowable e) {
	printf("Java_jni_ThrowTest_rethrowException\n");
	env->Throw(e);
}
 
void throwNewException(JNIEnv * env) {
	printf("throwNewException\n");
	jclass unsupportedExceptionCls = env->FindClass(
			"java/lang/UnsupportedOperationException");
	env->ThrowNew(unsupportedExceptionCls, "throwNewException Test\n");
}
 
JNIEXPORT void JNICALL Java_jni_ThrowTest_handlerException(JNIEnv * env,
		jclass cls) {
	throwNewException(env);
	jboolean result = env->ExceptionCheck();
	printf("ExceptionCheck result->%d\n", result);
	env->ExceptionDescribe();
	result = env->ExceptionCheck();
	printf("ExceptionCheck for ExceptionDescribe result->%d\n", result);
	throwNewException(env);
	jthrowable e = env->ExceptionOccurred();
	if (e) {
		printf("ExceptionOccurred not null\n");
	} else {
		printf("ExceptionOccurred null\n");
	}
	env->ExceptionClear();
	printf("ExceptionClear\n");
	e = env->ExceptionOccurred();
	if (e) {
		printf("ExceptionOccurred not null\n");
	} else {
		printf("ExceptionOccurred null\n");
	}
}
```

## 引用操作
jni.h中关于引用只定义了一个枚举，如下：
```
/* Return values from jobjectRefType */
typedef enum _jobjectType {
     JNIInvalidRefType    = 0,
     JNILocalRefType      = 1,
     JNIGlobalRefType     = 2,
     JNIWeakGlobalRefType = 3
} jobjectRefType;

```
jobjectRefType表示引用类型，仅限于本地方法使用，具体如下：

- JNIInvalidRefType表示无效引用；
- JNILocalRefType表示本地引用，当本地方法返回后这些引用会自动回收掉，JVM在开始执行一个新的本地方法前会先执行EnsureLocalCapacity评估当前线程能否创建至少16个本地引用，然后调用PushLocalFrame创建一个新的用于保存该方法执行过程中的本地引用的Frame，当本地方法结束调用PopLocalFrame释放调用该Frame，同时释放Frame中保存的即该本地方法执行过程中创建的所有本地引用。本地方法接受的Java对象类型参数或者返回值都是JNILocalRefType类型，JNI允许开发者根据JNILocalRefType类型引用创建一个JNIGlobalRefType类型引用。大部分情形开发者依赖程序自动回收掉本地引用即可，但是部分情形下需要开发者手动显示释放本地引用，比如本地方法中访问了一个大的Java对象，这时会创建一个指向该对象的本地引用，如果本地方法已经不在使用该引用了，需要手动释放该引用，否则该引用会阻止垃圾回收器回收掉该对象；再比如通过本地方法遍历某个包含大量Java对象的数组，遍历的时候会为每个需要访问的Java对象创建一个本地引用，如果这些引用不手动释放而是等到方法结束才释放就会导致内存溢出问题。JNI允许开发者在本地方法执行的任意时点删除本地引用从而允许垃圾回收器回收该引用指向的对象，为了确保这点，所有的JNI方法要求不能创建额外的本地引用，除非它是作为返回值。创建的本地引用只在当前线程内有效，因此不能将一个本地引用传递到另一个线程中。
- JNIGlobalRefType表示全局引用，必须通过DeleteGlobalRef方法删除才能释放该全局引用，从而允许垃圾回收器回收该引用指向的对象
- JNIWeakGlobalRefType表示全局弱应用，只在JVM内部使用，跟Java中的弱引用一样，当垃圾回收器运行的时候，如果一个对象仅被弱全局引用所引用，则这个对象将会被回收，如果指向的对象被回收掉了，全局弱引用会指向NULL，开发者可以通过IsSameObject方法判断弱引用是否等于NULL从而判断指向的对象是否被回收掉了。全局弱引用比Java中的SoftReference 或者WeakReference都要弱，如果都指向了同一个对象，则只有SoftReference 或者WeakReference都被回收了，全局弱引用才会等于NULL。全局弱应用跟PhantomReferences的交互不明确，应该避免两者间的交互。
综上，这里的引用其实就是Java中new一个对象返回的引用，本地引用相当于Java方法中的局部变量对Java对象实例的引用，全局引用相当于Java类的静态变量对Java对象实例的引用，其本质跟C++智能指针模板一样，是对象指针的二次包装，通过包装避免了该指针指向的对象被垃圾回收器回收掉，因此JNI中通过隐晦的引用访问Java对象的消耗比通过指针直接访问要高点，但是这是JVM对象和内存管理所必须的。

## 引用API

相关API如下：
- jobject NewGlobalRef(JNIEnv *env, jobject obj); 创建一个指向obj的全局引用，obj可以是本地或者全局引用，全局引用只能通过显示调用DeleteGlobalRef()释放。
- void DeleteGlobalRef(JNIEnv *env, jobject globalRef); 删除一个全局引用
- jobject NewLocalRef(JNIEnv *env, jobject ref); 创建一个指向对象ref的本地引用
- void DeleteLocalRef(JNIEnv *env, jobject localRef); 删除一个本地引用
- jint EnsureLocalCapacity(JNIEnv *env, jint capacity); 评估当前线程是否能够创建指定数量的本地引用，如果可以返回0，否则返回负数并抛出OutOfMemoryError异常。在执行本地方法前JVM会自动评估当前线程能否创建至少16个本地引用。JVM允许创建超过评估数量的本地引用，如果创建过多导致JVM内存不足JVM会抛出一个FatalError。
- jint PushLocalFrame(JNIEnv *env, jint capacity); 创建一个新的支持创建给定数量的本地引用的Frame，如果可以返回0，否则返回负数并抛出OutOfMemoryError异常。注意在之前的Frame中创建的本地引用在新的Frame中依然有效。
- jobject PopLocalFrame(JNIEnv *env, jobject result); POP掉当前的本地引用Frame，然后释放其中的所有本地引用，如果result不为NULL，则返回该对象在前一个即当前Frame之前被push的Frame中的本地引用
- jweak NewWeakGlobalRef(JNIEnv *env, jobject obj); 创建一个指向对象obj的弱全局引用，jweak是jobject的别名，如果obj是null则返回NULL，如果内存不足则抛出OutOfMemoryError异常
- void DeleteWeakGlobalRef(JNIEnv *env, jweak obj); 删除弱全局引用。
- jobjectRefType GetObjectRefType(JNIEnv* env, jobject obj); 获取某个对象引用的引用类型，JDK1.6引入的

## 类和对象操作
类操作API如下：
- jint GetVersion(JNIEnv *env);  获取当前JVM的JNI接口版本，该版本在jni.cpp中通过全局变量CurrentVersion指定。
- jclass DefineClass(JNIEnv *env, const char *name, jobject loader,const jbyte *buf, jsize bufLen);  加载某个类，name表示类名，loader表示类加载器实例，buf表示class文件的字节数据，bufLen表示具体的字节数，此方法跟ClassLoader的实现基本一致。
- jclass FindClass(JNIEnv *env, const char *name); 根据类名查找某个类，如果该类未加载则会调用合适的类加载器加载并链接该类。会优先使用定义了调用FindClass的本地方法的类的类加载器加载指定类名的类，如果是JDK标准类没有对应的类加载器则使用ClassLoader.getSystemClassLoader返回的系统类加载器加载。
- jclass GetSuperclass(JNIEnv *env, jclass clazz); 获取父类，如果该类是某个接口或者是Object则返回NULL
- jboolean IsAssignableFrom(JNIEnv *env, jclass clazz1,jclass clazz2); 判断clazz1能否安全的类型转换成clazz2

对象操作相关API：
- jobject AllocObject(JNIEnv *env, jclass clazz); 分配一个Java对象并返回该对象的本地引用，注意该方法并未调用任何构造方法
- jobject NewObject(JNIEnv *env, jclass clazz,jmethodID methodID, ...); 调用指定的构造方法创建一个Java对象并返回该对象的本地引用，最后的三个点表示该构造方法的多个参数
- jobject NewObjectA(JNIEnv *env, jclass clazz,jmethodID methodID, const jvalue *args); 同上，不过构造方法的入参是一个参数数组的指针
- jobject NewObjectV(JNIEnv *env, jclass clazz,jmethodID methodID, va_list args);同上，不过构造参数被放在va_list列表中
- jclass GetObjectClass(JNIEnv *env, jobject obj);获取Java对象所属的Java类
- jboolean IsInstanceOf(JNIEnv *env, jobject obj,jclass clazz); 判断某个对象是否是某个类的实例
- jboolean IsSameObject(JNIEnv *env, jobject ref1,jobject ref2); 判断两个引用是否引用相同的对象

## 字段和方法操作
jfieldID和jmethodID定义

JNI中使用jfieldID来标识一个某个类的字段，jmethodID来标识一个某个类的方法，jfieldID和jmethodID都是根据他们的字段名（方法名）和描述符确定的，通过jfieldID读写字段或者通过jmethodID调用方法都会避免二次查找，但是这两个ID不能阻止该字段或者方法所属的类被卸载，如果类被卸载了则jfieldID和jmethodID失效了需要重新计算，因此如果希望长期使用jfieldID和jmethodID则需要保持对该类的持续引用，即建立对该类的全局引用，两者的定义在jni.h中，如下：
```
typedef union jvalue {
    jboolean z;
    jbyte    b;
    jchar    c;
    jshort   s;
    jint     i;
    jlong    j;
    jfloat   f;
    jdouble  d;
    jobject  l;
} jvalue;

struct _jfieldID;
typedef struct _jfieldID *jfieldID;

struct _jmethodID;
typedef struct _jmethodID *jmethodID;
```
