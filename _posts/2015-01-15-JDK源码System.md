---
layout: post
categories: [JDK]
description: none
keywords: JDK
---
# JDK源码System
System类是在Java程序中作为一个标准的系统类，实现了控制台与程序之间的输入输出流，系统的初始化与获取系统环境变量、数组的复制、返回一个精准的时间以及一些简单的对虚拟机的操作等。

## System类
System类是一个与Class类一样的直接注册进虚拟机的类，也就是直接与虚拟机打交道的类。

首先编译期间开始初始化，初始化方法在 `initializeSystemClass` 中
```
    /* register the natives via the static initializer.
     *
     * VM will invoke the initializeSystemClass method to complete
     * the initialization for this class separated from clinit.
     * Note that to use properties set by the VM, see the constraints
     * described in the initializeSystemClass method.
     */
    private static native void registerNatives();
    static {
        registerNatives();
    }
```
System类是一个不可被继承的类，同事不能由外部直接创建，只能有jvm来创建该类的实例。那么它是如何实现控制台的输入与输出的呢？

```
public final static InputStream in = null;
public final static PrintStream out = null;
public final static PrintStream err = null;
```

在其源码中定义了三个静态成员变量，也就是我们常写的System.out、System.in以及System.err。接着：
```
public static void setIn(InputStream in) {
        checkIO();
        setIn0(in);
    }
public static void setOut(PrintStream out) {
        checkIO();
        setOut0(out);
    }
public static void setErr(PrintStream err) {
        checkIO();
        setErr0(err);
    }
    
private static void checkIO() {
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("setIO"));
        }
    }

    private static native void setIn0(InputStream in);
    private static native void setOut0(PrintStream out);
    private static native void setErr0(PrintStream err);

```
检查完权限后，将控制台的流用Native方法写进、写出jvm中，System.out、System.in以及System.err三者并不是内部类，只是System类的成员变量，而已。

## arraycopy
下面介绍System类中常见的方法：
```
public static native long currentTimeMillis();
public static native long nanoTime();
```
currentTimeMillis方法返回一个当前的时间戳，是以毫秒为单位的，用来计算时间的，而nanoTime方法是也是返回一个当前的时间戳，是以纳秒为单位的，有的操作系统时间的最小精确度是10毫秒，所以这两个个方法可能会导致一些偏差。

```
public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);

```
arraycopy方法是在阅读Java源码中常见到的数组复制方法，是由jvm直接来复制的，不仅仅是复制数组，它还可以调节数组所占空间的大小。

### identityHashCode
identityHashCode方法是根据内存地址来获取其哈希值的：
```
public static native int identityHashCode(Object x);
```
以String类来做例子：
```
String name1=new String("蕾姆");
String name2=new String("蕾姆");
System.out.println("identityHashCode的值");
System.out.println(System.identityHashCode(name1));
System.out.println(System.identityHashCode(name2));
System.out.println("hashCode的值");
System.out.println(name1.hashCode());
System.out.println(name2.hashCode());
//打印：
//identityHashCode的值
//1836019240
//325040804
//hashCode的值
//1082376
//1082376
```

得出的结果是identityHashCode的值是不同的，再看另外一个例子：
```
String name1="蕾姆";
String name2="蕾姆";
System.out.println("identityHashCode的值");
System.out.println(System.identityHashCode(name1));
System.out.println(System.identityHashCode(name2));
System.out.println("hashCode的值");
System.out.println(name1.hashCode());
System.out.println(name2.hashCode());
//打印：
//identityHashCode的值
//1836019240
//1836019240
//hashCode的值
//1082376
//1082376
```
因为两个"蕾姆"的地址是一样的，所以两者的identityHashCode值相同。

### getProperty
```
private static Properties props;
private static native Properties initProperties(Properties props);
public static String getProperty(String key) {
        checkKey(key);
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPropertyAccess(key);
        }

        return props.getProperty(key);
    }
```
## 系统信息的访问，如外部属性和环境变量等

### getProperty
getProperty方法获取制定属性的系统变量值的，下面列出常用的系统环境变量值：
- file.separator	文件分隔符（在 UNIX 系统中是“/”）
- path.separator	路径分隔符（在 UNIX 系统中是“:”）
- line.separator	行分隔符（在 UNIX 系统中是“/n”）
- user.name	用户的账户名称
- user.home	用户的主目录
- user.dir	用户的当前工作目录

比如获取当前用户的主目录：
```
public static void main(String args[]) throws TestException {
        System.out.println(System.getProperty("user.home"));
    }
    //输出：C:\Users\Admin
```

### getenv
获取系统环境变量值：
```
public static String getenv(String name) {
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("getenv."+name));
        }

        return ProcessEnvironment.getenv(name);
    }
```

接下来就是一些简单的jvm操作了：
```
public static void exit(int status) {
        Runtime.getRuntime().exit(status);
    }
```
该方法是终止虚拟机的操作，参数解释为状态码，非 0 的状态码表示异常终止。 而且，这个方法永远不会正常返回。 

这是唯一一个能够退出程序并不执行finally的情况。因为这时候进程已经被杀死了，如下代码所示：
```
public static void main(String args[]) throws TestException {
        System.out.println("hello");
        System.exit(0);
        System.out.println("world");
    }

 public static void main(String args[]) throws TestException {
        try{
            System.out.println("hello");
            System.exit(0);
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            System.out.println("world");
        }
    }
//两者都只会输出hello
```

### 垃圾回收相关操作
System类还提供类调用jvm的垃圾回收器的方法，该方法让 jvm做了一些努力来回收未用对象或失去了所有引用的对象，以便能够快速地重用这些对象当前占用的内存。
```
public static void gc() {
        Runtime.getRuntime().gc();
    }
```
但我们基本不会自己调用该方法来回收内存，将垃圾回收交给jvm就行了。联想到finalize方法，该方法是用来回收没有被引用的对象的，调用System.gc方法便会调用这个方法，但是我们也不会用到finalize方法，因为这个方法设计出来是用来让c++程序员适应的。在Java编程思想中提到了使用到finalize方法可能的场景：
```
class SSS{
    protected void finalize(){
        System.out.println("回收了");
    }
}
public class Test {
    public static void main(String args[]) throws TestException {
        new SSS();
        System.gc();
    }
}
//输出：回收了
```
也就是说，重写finalize方法，我们就能知道该对象实在何时被回收的了。

## 源码
System.getProperty 可以拿到当前系统属性，比如当前操作系统的属性、动态链接库位置、编码集、当前虚拟机的版本等等一系列系统属性。当然，你可以把它理解为整个系统上下文的一个存储数据的集合，你可以往里面set属性，任何地点get取出，并且线程安全。下面案例是展示了默认情况下所有的属性（所有的key都展示出来了，项目中如需使用，可以先遍历一次再寻找）
```
public static void main(String[] args) {
 
  Properties properties = System.getProperties();
 
  Iterator<Map.Entry<Object, Object>> iterator = properties.entrySet().iterator();
 
  while (iterator.hasNext()){
    Map.Entry<Object, Object> next = iterator.next();
    System.out.println("key:"+next.getKey()+";value:"+next.getValue());
  }
}
```
接下来直接看getProperty源码，实现非常非常非常非常的简单。在System类中维护了一个Properties类，直接从Properties中取出数据。而Properties实现了Hashtable，所以线程也是安全的。
```
public static String getProperty(String key) {
 
  …………
  
  return props.getProperty(key);
}
```
此时，我们更加关心Properties的数据来源，所以我们找到初始化Properties的地方。
```

private static void initializeSystemClass() {
 
  props = new Properties();
  initProperties(props);  
 
  …………
 
}
 
private static native Properties initProperties(Properties props);
```
而在Java层面找破头都找不到谁调用的initializeSystemClass方法，实际上这里是JVM初始化的过程中调用的此方法，所以肯定找不到，所以我们看到Hotspot中初始化的源码 src/share/vm/runtime/thread.cpp 文件中 create_vm 方法对于System类的初始化
```
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {
 
  …………
 
  // 加载并初始化java_lang_System类。
  initialize_class(vmSymbols::java_lang_System(), CHECK_0);
  call_initializeSystemClass(CHECK_0);
 
  …………
}
 
 
static void call_initializeSystemClass(TRAPS) {
  // 拿到加载好的类对象（在Hotspot中Klass代表类）
  Klass* k =  SystemDictionary::resolve_or_fail(vmSymbols::java_lang_System(), true, CHECK);
  instanceKlassHandle klass (THREAD, k);
 
  JavaValue result(T_VOID);
 
  // 调用java_lang_System类中静态方法initializeSystemClass
  // 这恰好，也是我们在Java层面寻找不到的调用
  JavaCalls::call_static(&result, klass, vmSymbols::initializeSystemClass_name(),
                                         vmSymbols::void_method_signature(), CHECK);

```
既然initializeSystemClass方法的调用方我们找到了，此时就需要看到initProperties这个native方法对于Properties的初始化工作。

/src/share/native/java/lang/System.c 文件中对于initProperties这个native方法的实现。
```
JNIEXPORT jobject JNICALL
Java_java_lang_System_initProperties(JNIEnv *env, jclass cla, jobject props)
{
    char buf[128];
 
    // 获取到系统属性
    java_props_t *sprops = GetJavaProperties(env);
 
    // 获取到Properties的put方法签名
    jmethodID putID = (*env)->GetMethodID(env,
                                          (*env)->GetObjectClass(env, props),
                                          "put",
            "(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;");
 
    // 获取到Properties的remove方法签名
    jmethodID removeID = (*env)->GetMethodID(env,
                                          (*env)->GetObjectClass(env, props),
                                          "remove",
            "(Ljava/lang/Object;)Ljava/lang/Object;");
 
    // 获取到Properties的getProperty方法签名
    jmethodID getPropID = (*env)->GetMethodID(env,
                                          (*env)->GetObjectClass(env, props),
                                          "getProperty",
            "(Ljava/lang/String;)Ljava/lang/String;");
 
    // 调用Properties的put方法往Properties里面添加属性
    PUTPROP(props, "java.specification.version",
            JDK_MAJOR_VERSION "." JDK_MINOR_VERSION);
    PUTPROP(props, "java.specification.name",
            "Java Platform API Specification");
    PUTPROP(props, "os.name", sprops->os_name);
    PUTPROP(props, "os.version", sprops->os_version);
    PUTPROP(props, "os.arch", sprops->os_arch);
 
    …………
 
    // 把JVM启动时设置的参数添加
    // 可能是JVM自带的，也有可能是-D等等命令行设置的
    ret = JVM_InitProperties(env, props);
 
    …………
 
    return ret;
}
```
- 通过GetJavaProperties方法获取到系统属性
- initProperties这个native方法把Properties传入，然后通过PUTPROP宏往Properties里面添加各种系统属性
- 调用JVM_InitProperties方法，把JVM自带的，也有可能是-D等等命令行设置的系统属性添加到Properties中

```
JVM_ENTRY(jobject, JVM_InitProperties(JNIEnv *env, jobject properties))
  JVMWrapper("JVM_InitProperties");
  ResourceMark rm;
 
  Handle props(THREAD, JNIHandles::resolve_non_null(properties));
 
  // 把JVM的参数添加到Properties中
  for (SystemProperty* p = Arguments::system_properties(); p != NULL; p = p->next()) {
    PUTPROP(props, p->key(), p->value());
  }
 
  …………
 
  return properties;
JVM_END
 
static SystemProperty*  system_properties()   { return _system_properties; }
```
这里调用Arguments::system_properties()方法得到SystemProperty对象，SystemProperty对象存放JVM所有的系统参数，所以这里是在遍历JVM中所有的系统参数。最后，我们可以看一下这些JVM参数何时添加的。src/share/vm/runtime/arguments.cpp 文件中init_system_properties方法

```

void Arguments::init_system_properties() {
 
  // 添加到SystemProperty集合中
  PropertyList_add(&_system_properties, new SystemProperty("java.vm.specification.name",
                                                                 "Java Virtual Machine Specification",  false));
  PropertyList_add(&_system_properties, new SystemProperty("java.vm.version", VM_Version::vm_release(),  false));
  PropertyList_add(&_system_properties, new SystemProperty("java.vm.name", VM_Version::vm_name(),  false));
  PropertyList_add(&_system_properties, new SystemProperty("java.vm.info", VM_Version::vm_info_string(),  true));
 
 
  _java_ext_dirs = new SystemProperty("java.ext.dirs", NULL,  true);
  _java_endorsed_dirs = new SystemProperty("java.endorsed.dirs", NULL,  true);
  _sun_boot_library_path = new SystemProperty("sun.boot.library.path", NULL,  true);
  _java_library_path = new SystemProperty("java.library.path", NULL,  true);
  _java_home =  new SystemProperty("java.home", NULL,  true);
  _sun_boot_class_path = new SystemProperty("sun.boot.class.path", NULL,  true);
 
  _java_class_path = new SystemProperty("java.class.path", "",  true);
 
  // 添加到SystemProperty集合中
  PropertyList_add(&_system_properties, _java_ext_dirs);
  PropertyList_add(&_system_properties, _java_endorsed_dirs);
  PropertyList_add(&_system_properties, _sun_boot_library_path);
  PropertyList_add(&_system_properties, _java_library_path);
  PropertyList_add(&_system_properties, _java_home);
  PropertyList_add(&_system_properties, _java_class_path);
  PropertyList_add(&_system_properties, _sun_boot_class_path);
 
  os::init_system_properties_values();
}
```
还有在解析-D 等等java命令参数时也会添加，因为-D设置的属性是添加到System的Properties中

src/share/vm/runtime/arguments.cpp 文件中 parse_each_vm_init_arg方法
```

// 解析-D
else if (match_option(option, "-D", &tail)) {
  // 添加到SystemProperty集合中
  if (!add_property(tail)) {
    return JNI_ENOMEM;
  }
  …………
}
 
bool Arguments::add_property(const char* prop) {
  const char* eq = strchr(prop, '=');
  char* key;
  const static char ns[1] = {0};
  char* value = (char *)ns;
 
  // 解析key
  size_t key_len = (eq == NULL) ? strlen(prop) : (eq - prop);
  key = AllocateHeap(key_len + 1, mtInternal);
  strncpy(key, prop, key_len);
  key[key_len] = '\0';
 
  // 解析value
  // key和value用=号隔开
  if (eq != NULL) {
    size_t value_len = strlen(prop) - key_len - 1;
    value = AllocateHeap(value_len + 1, mtInternal);
    strncpy(value, &prop[key_len + 1], value_len + 1);
  }
 
  // 添加到SystemProperty集合中
  PropertyList_unique_add(&_system_properties, key, value);
  return true;
}
```