---
layout: post
categories: [Java]
description: none
keywords: Java
---
# Java源码Class类

## 
我们来看下Class类的源码，源码太多了，挑了几个重点：
```
public final class Class<T> implements java.io.Serializable,  
                              GenericDeclaration,  
                              Type,  
                              AnnotatedElement {  
    private static final int ANNOTATION= 0x00002000;  
    private static final int ENUM      = 0x00004000;  
    private static final int SYNTHETIC = 0x00001000;  
  
    private static native void registerNatives();  
    static {  
        registerNatives();  
    }  
  
    /*  
     * Private constructor. Only the Java Virtual Machine creates Class objects.  
     * This constructor is not used and prevents the default constructor being  
     * generated.  
     */  
    private Class(ClassLoader loader) { //私有化的 构造器  
        // Initialize final field for classLoader.  The initialization value of non-null  
        // prevents future JIT optimizations from assuming this final field is null.  
        classLoader = loader;  
    }  
    ...  
      
    // reflection data that might get invalidated when JVM TI RedefineClasses() is called  
    private static class ReflectionData<T> {  
        volatile Field[] declaredFields;//字段  
        volatile Field[] publicFields;  
        volatile Method[] declaredMethods;//方法  
        volatile Method[] publicMethods;  
        volatile Constructor<T>[] declaredConstructors;//构造器  
        volatile Constructor<T>[] publicConstructors;  
        // Intermediate results for getFields and getMethods  
        volatile Field[] declaredPublicFields;  
        volatile Method[] declaredPublicMethods;  
        volatile Class<?>[] interfaces;//接口  
  
        // Value of classRedefinedCount when we created this ReflectionData instance  
        final int redefinedCount;  
  
        ReflectionData(int redefinedCount) {  
            this.redefinedCount = redefinedCount;  
        }  
    }  
      ...  
     //注释数据  
     private volatile transient AnnotationData annotationData;  
  
    private AnnotationData annotationData() {  
        while (true) { // retry loop  
            AnnotationData annotationData = this.annotationData;  
            int classRedefinedCount = this.classRedefinedCount;  
            if (annotationData != null &&  
                annotationData.redefinedCount == classRedefinedCount) {  
                return annotationData;  
            }  
            // null or stale annotationData -> optimistically create new instance  
            AnnotationData newAnnotationData = createAnnotationData(classRedefinedCount);  
            // try to install it  
            if (Atomic.casAnnotationData(this, annotationData, newAnnotationData)) {  
                // successfully installed new AnnotationData  
                return newAnnotationData;  
            }  
        }  
    }   
    ...  
```
Class类的构造方法是private， 只有JVM能创建Class实例 ，我们开发人员 是无法创建Class实例的，JVM在构造Class对象时，需要传入一个 类加载器 。

类也是可以用来存储数据的，Class类就像 普通类的模板 一样，用来保存“类所有相关信息”的类 。
























