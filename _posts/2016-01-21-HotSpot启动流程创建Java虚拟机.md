---
layout: post
categories: [JVM]
description: none
keywords: JVM
---
## HotSpot启动流程创建Java虚拟机


## 虚拟机入口
到这一步就进入了jvm运行真正的入口。这一步会执行很多操作调用很多子函数，最终加载 java 主类，并使用 jni 调用 java 的静态main函数。
调用流程：
jdk/src/solaris/bin/java_md_solinux.c ContinueInNewThread0()     =>    调用pthread_create创建新线程,启动JavaMain入口
jdk/src/share/bin/java.c JavaMain()    =>     真正的jvm初始化入口
jdk/src/share/bin/java.c InitializeJVM()     =>    初始化jvm,调用CreateJavaVM进行初始化,调用libjvm中的库函数JNI_CreateJavaVM
jdk/src/share/bin/java.c LoadMainClass()     =>    加载一个类并验证主类是否存在
jdk/src/share/bin/java.c     =>    调用main方法,封装成jni，jni_invoke_static() 通过jni解释调用java静态方法
Debug.java main()    =>     java静态主函数调用
源码如下：
作用：虚拟机的入口函数。位置：jdk8u-dev/jdk/src/share/bin/java.c JavaMain
```
int JNICALL
JavaMain(void * _args)
{
    JavaMainArgs *args = (JavaMainArgs *)_args;         //获取参数
    int argc = args->argc;
    char **argv = args->argv;
    int mode = args->mode;
    char *what = args->what;
    InvocationFunctions ifn = args->ifn;                //当前虚拟机导致的函数指针
                                                        //该机制可以保证同一环境配置多个jdk版本

    JavaVM *vm = 0;
    JNIEnv *env = 0;
    jclass mainClass = NULL;                            //main函数class
    jclass appClass = NULL;                             // 正在启动的实际应用程序类
    jmethodID mainID;
    jobjectArray mainArgs;
    int ret = 0;
    jlong start, end;

    RegisterThread();                                   //window/类unix为空实现，macos特别处理

    /* 初始化虚拟机 */
    start = CounterGet();
    //通过CreateJavaVM导出jvm到&vm, &env
    //&vm主要提供虚拟整体的操作
    //&env主要提供虚拟启动的环境的操作
    if (!InitializeJVM(&vm, &env, &ifn)) {                                                                      
        JLI_ReportErrorMessage(JVM_ERROR1);
        exit(1);
    }

    if (showSettings != NULL) {
        ShowSettings(env, showSettings);
        CHECK_EXCEPTION_LEAVE(1);
    }

    if (printVersion || showVersion) {
        PrintJavaVersion(env, showVersion);
        CHECK_EXCEPTION_LEAVE(0);
        if (printVersion) {
            LEAVE();
        }
    }

    /* 如果没有指定class或者jar，则直接退出 */
    if (printXUsage || printUsage || what == 0 || mode == LM_UNKNOWN) {
        PrintUsage(env, printXUsage);
        CHECK_EXCEPTION_LEAVE(1);
        LEAVE();
    }

    FreeKnownVMs();  /* after last possible PrintUsage() */

    /* 记录初始化jvm时间 */
    if (JLI_IsTraceLauncher()) {
        end = CounterGet();
        JLI_TraceLauncher("%ld micro seconds to InitializeJVM\n",
               (long)(jint)Counter2Micros(end-start));
    }

    /* 接下来，从argv获取应用的参数，打印参数 */
    if (JLI_IsTraceLauncher()){
        int i;
        printf("%s is '%s'\n", launchModeNames[mode], what);
        printf("App's argc is %d\n", argc);
        for (i=0; i < argc; i++) {
            printf("    argv[%2d] = '%s'\n", i, argv[i]);
        }
    }

    ret = 1;

    /*
     * Get the application's main class.
     *
     * See bugid 5030265.  The Main-Class name has already been parsed
     * from the manifest, but not parsed properly for UTF-8 support.
     * Hence the code here ignores the value previously extracted and
     * uses the pre-existing code to reextract the value.  This is
     * possibly an end of release cycle expedient.  However, it has
     * also been discovered that passing some character sets through
     * the environment has "strange" behavior on some variants of
     * Windows.  Hence, maybe the manifest parsing code local to the
     * launcher should never be enhanced.
     *
     * Hence, future work should either:
     *     1)   Correct the local parsing code and verify that the
     *          Main-Class attribute gets properly passed through
     *          all environments,
     *     2)   Remove the vestages of maintaining main_class through
     *          the environment (and remove these comments).
     *
     * This method also correctly handles launching existing JavaFX
     * applications that may or may not have a Main-Class manifest entry.
     */

    /* 加载Main class */
    //从环境中取出java主类
    mainClass = LoadMainClass(env, mode, what);
    /* 检查是否指定main class， 不存在则退出虚拟机*/
    CHECK_EXCEPTION_NULL_LEAVE(mainClass);
   
    /* 获取应用的class文件 */
	//JavaFX gui应用相关可忽略
    appClass = GetApplicationClass(env);
    /* 检查是否指定app class， 不存在返回-1 */
    NULL_CHECK_RETURN_VALUE(appClass, -1);
    
    /*window和类unix为空实现，在macos下设定gui程序的程序名称*/
	//JavaFX gui应用相关可忽略
    PostJVMInit(env, appClass, vm);
    
    /*从mainclass中加载静态main方法*/
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");
    CHECK_EXCEPTION_NULL_LEAVE(mainID);

    /* Build platform specific argument array */
    //组装main函数参数
    mainArgs = CreateApplicationArgs(env, argv, argc);
    CHECK_EXCEPTION_NULL_LEAVE(mainArgs);

    /* Invoke main method. */
    /* 调用main方法，并把参数传递过去 */
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);

    /*
     * The launcher's exit code (in the absence of calls to
     * System.exit) will be non-zero if main threw an exception.
     */
    ret = (*env)->ExceptionOccurred(env) == NULL ? 0 : 1;
    //main方法执行完毕，JVM退出，包含两步：
    //(*vm)->DetachCurrentThread，让当前Main线程同启动线程断联
    //创建一个新的名为DestroyJavaVM的线程，让该线程等待所有的非后台进程退出，并在最后执行(*vm)->DestroyJavaVM方法。
    LEAVE();
}
```
初始化虚拟机代码如下
```
    /* 初始化虚拟机 */
    start = CounterGet();
    //通过CreateJavaVM导出jvm到&vm, &env
    //&vm主要提供虚拟整体的操作
    //&env主要提供虚拟启动的环境的操作
    if (!InitializeJVM(&vm, &env, &ifn)) {                                                                      
        JLI_ReportErrorMessage(JVM_ERROR1);
        exit(1);
    }

```
## 函数 InitializeJVM()
在运行 jvm 之前会调用这个子函数先初始化jvm，调用CreateJavaVM进行初始化，调用libjvm中的库函数JNI_CreateJavaVM，这部分代码实现就是源码下面hotspot模块了 。

jdk/src/share/bin/java.c InitializeJVM()     =>    初始化jvm,调用CreateJavaVM进行初始化,调用libjvm中的库函数JNI_CreateJavaVM
hotspot/src/share/vm/prims/jni.cpp JNI_CreateJavaVM() 进入jvm初始化阶段,Threads::create_vm()执行入口
代码如下：
```
/*
 * Initializes the Java Virtual Machine. Also frees options array when
 * finished.
 */
static jboolean
InitializeJVM(JavaVM **pvm, JNIEnv **penv, InvocationFunctions *ifn)
{
    JavaVMInitArgs args;
    jint r;

    memset(&args, 0, sizeof(args));
    args.version  = JNI_VERSION_1_2;
    args.nOptions = numOptions;
    args.options  = options;
    args.ignoreUnrecognized = JNI_FALSE;

    if (JLI_IsTraceLauncher()) {
        int i = 0;
        printf("JavaVM args:\n    ");
        printf("version 0x%08lx, ", (long)args.version);
        printf("ignoreUnrecognized is %s, ",
               args.ignoreUnrecognized ? "JNI_TRUE" : "JNI_FALSE");
        printf("nOptions is %ld\n", (long)args.nOptions);
        for (i = 0; i < numOptions; i++)
            printf("    option[%2d] = '%s'\n",
                   i, args.options[i].optionString);
    }
    // 调用 libjvm.so 中的 JNI_CreateJavaVM 创建虚拟机
    r = ifn->CreateJavaVM(pvm, (void **)penv, &args);
    JLI_MemFree(options);
    return r == JNI_OK;
}
```

## 函数 JNI_CreateJavaVM ()
由hotspot模块实现的jvm创建。
调用流程：
jdk/src/share/bin/java.c InitializeJVM()     =>    初始化jvm,调用CreateJavaVM进行初始化,调用libjvm中的库函数JNI_CreateJavaVM
hotspot/src/share/vm/prims/jni.cpp JNI_CreateJavaVM()      =>    进入jvm初始化阶段,Threads::create_vm()执行入口
代码如下：
```
_JNI_IMPORT_OR_EXPORT_ jint JNICALL JNI_CreateJavaVM(JavaVM **vm, void **penv, void *args) {
  //通过Atomic::xchg方法修改全局volatile变量vm_created为1，该变量默认为0，如果返回1则说明JVM已经创建完成或者创建中，返回JNI_EEXIST错误码，如果返回0则说明JVM未创建
  if (Atomic::xchg(1, &vm_created) == 1) {
    return JNI_EEXIST;   // already created, or create attempt in progress
  }
  //通过Atomic::xchg方法修改全局volatile变量safe_to_recreate_vm为0，该变量默认为1，如果返回0则说明JVM已经在重新创建了，返回JNI_ERR错误码，如果返回1则说明JVM未创建
  if (Atomic::xchg(0, &safe_to_recreate_vm) == 0) {
    return JNI_ERR;  // someone tried and failed and retry not allowed.
  }
  assert(vm_created == 1, "vm_created is true during the creation");
  bool can_try_again = true;
  //完成JVM的初始化，如果初始化过程中出现不可恢复的异常则can_try_again会被置为false
  result = Threads::create_vm((JavaVMInitArgs*) args, &can_try_again);
  //初始化正常
  if (result == JNI_OK) {
    //获取当前线程，即执行create_vm的线程，也是JNI_CreateJavaVM执行完毕后执行main方法的线程
    JavaThread *thread = JavaThread::current();
    /*JavaVM赋值，main_vm是jni.h中的全局变量，最终实现指向全局变量jni_InvokeInterface */
    *vm = (JavaVM *)(&main_vm);
    //JNIEnv赋值
    *(JNIEnv**)penv = thread->jni_environment();
    // 记录应用的启动时间
    RuntimeService::record_application_start();
    // 通知JVMTI应用启动
    if (JvmtiExport::should_post_thread_life()) {
       JvmtiExport::post_thread_start(thread);
    }
    EventThreadStart event;
    if (event.should_commit()) {
      event.set_javalangthread(java_lang_Thread::thread_id(thread->threadObj()));
      event.commit();
    }
    //根据配置加载类路径中所有的类
    if (CompileTheWorld) ClassLoader::compile_the_world();
    if (ReplayCompiles) ciReplay::replay(thread);
    // win* 系统添加异常处理的wrapper
    CALL_TEST_FUNC_WITH_WRAPPER_IF_NEEDED(test_error_handler);
    CALL_TEST_FUNC_WITH_WRAPPER_IF_NEEDED(execute_internal_vm_tests);
    //设置当前线程的线程状态
    ThreadStateTransition::transition_and_fence(thread, _thread_in_vm, _thread_in_native);
  } else {
  //如果create_vm初始化失败
    if (can_try_again) {
      // 如果可以重试则恢复默认值1
      safe_to_recreate_vm = 1;
    }
    //将vm和penv置空，vm_created重置成0
    *vm = 0;
    *(JNIEnv**)penv = 0;
    OrderAccess::release_store(&vm_created, 0);
  }
  return result;
}
```

## 函数 Threads::create_vm ()
最终进入这里才会真正初始化 jvm 需要的所有东西。
调用流程：
```
hotspot/src/share/vm/runtime.c Threads::create_vm()     =>    jvm初始化入口
hotspot/src/share/vm/runtime.c is_supported_jni_version()     =>     检测JNI版本
hotspot/src/share/vm/utilities.ostream.cpp ostream_init()      =>    输入输出流初始化
hotspot/src/share/vm/runtime/arguments.cpp process_sun_java_launcher_properties
hotspot/src/share/vm/runtime/init.cpp init_globals()      =>    全局模块初始化入口
hotspot/src/share/vm/services/management.cpp management_init()      =>    监控服务，线程服务，运行时服务，类加载服务初始化
hotspot/src/share/vm/interpreter/bytecodes.cpp bytecodes_init()      =>    字节码解释器初始化
hotspot/src/share/vm/classfile/classLoader.cpp classLoader_init()      =>    类加载器初始化第一阶段：zip,jimage入口，设置启动类路径
hotspot/src/share/vm/code/codeCache.cpp codeCache_init()      =>    代码缓存初始化
hotspot/src/share/vm/runtime/vm_version.cpp VM_Version_init()
hotspot/src/share/vm/runtime/os.cpp os_init_globals()
hotspot/src/share/vm/runtime/stubRoutines.cpp stubRoutines_init1()
hotspot/src/share/vm/memory.universe.cpp universe_init()      =>    堆空间,元空间,AOTLoader,SymbolTable,StringTable,G1收集器初始化
hotspot/src/share/vm/interpreter/interpreter.cpp interpreter_init()      =>    模板解释器初始化
hotspot/src/share/vm/intepreter/invocationCounter.cpp invocationCounter_init()      =>    热点统计初始化
hotspot/src/share/vm/gc_implementation/shared/markSweep.cpp marksweep_init()      =>    标记清除GC初始化
hotspot/src/share/vm/utilities/accesssFlags.cpp accessFlags_init()
hotspot/src/share/vm/interpreter/templateTable.cpp templateTable_init()      =>    模板表初始化
hotspot/src/share/vm/runtime/interfaceSupport.cpp InterfaceSupport_init()
hotspot/src/share/vm/runtime/sharedRuntime.cpp SharedRuntime::generate_stubs()      =>    生成部分例程入口
hotspot/src/share/vm/memory/universe.cpp universe2_init() vmSymbols，     =>    系统字典，预加载类，构建基本数据对象模型
hotspot/src/share/vm/memory/referenceProcessor.cpp referenceProcessor_init()      =>    对象引用策略初始化
hotspot/src/share/vm/runtime/jniHandles.cpp jni_handles_init()      =>    JNI引用句柄
hotspot/src/share/vm/code/vtableStubs.cpp vtableStubs_init()      =>    虚表例程初始化
hotspot/src/share/vm/code/icBuffer.cpp InlineCacheBuffer_init()      =>    内联缓冲初始化
hotspot/src/share/vm/complier/complierOracle.cpp compilerOracle_init()      =>    编译器初始化
hotspot/src/share/vm/runtime/compilationPolicy.cpp compilationPolicy_init()      =>    编译策略，根据编译等级确定从c1,c2编译器
hotspot/src/share/vm/complier/complierBroker.cpp compileBroker_init()      =>    编译器初始化
hotspot/src/share/vm/classfile/javaClasses.cpp javaClasses_init()      =>    java基础类相关计算
hotspot/src/share/vm/runtime/stubRoutines.cpp stubRoutines_init1()
```

代码如下：
```
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {
  /调用JDK_Version_init();
  //加载libjava.so、libverify.so库，通过调用库中导出的JDK_GetVersionInfo0查询当前虚拟机版本
  extern void JDK_Version_init();
  // Check version
  // 检查当前版本是否在支持范围内，主要不支持1.1版本 高于1.1都返回true
  if (!is_supported_jni_version(args->version)) return JNI_EVERSION;
  // 输入输出流初始化
  ostream_init();
  // 处理java启动属性.
  // Process java launcher properties.
  Arguments::process_sun_java_launcher_properties(args);
  // os模块初始化
  // 初始化系统环境，例如：获取当前的进程pid、获取系统时钟、设置内存页大小
  // 获取cpu数、获取物理内存大小
  os::init();
  // 系统参数初始化
  // 设置系统属性, key-value
  // 设置了java虚拟机信息、清空由os::init_system_properties_values()方法设置的值
  Arguments::init_system_properties();
  // 获取jdk版本号，作为接下来参数解析的依据
  JDK_Version_init();
    // 设定jdk版本后再去更新系统版本、供应商属性值，此处主要设定3个参数
  // jdk1.7以后的版本厂商修改为oracle,之前为sun
  // 1、java.vm.specification.vendor     可选值为：Oracle Corporation / Sun Microsystems Inc.
  // 2、java.vm.specification.version    可选值为：1.0  /  1.7  1.8  1.9 etc
  // 3、java.vm.vendor                   可选值为: Oracle Corporation / Sun Microsystems Inc.
  Arguments::init_version_specific_system_properties();
  // 参数转换
    // Parse arguments
  // 解析参数,生成java虚拟机运行期间的一些参数指标
  // -XX:Flags=   -XX:+PrintVMOptions  -XX:-PrintVMOptions    -XX:+IgnoreUnrecognizedVMOptions
  // -XX:-IgnoreUnrecognizedVMOptions  -XX:+PrintFlagsInitial -XX:NativeMemoryTracking
  jint parse_result = Arguments::parse(args);
  if (parse_result != JNI_OK) return parse_result;
  //主要完成大页支持以及linux上的coredump_filter配置
  os::init_before_ergo();
    //调整运行环境得参数及一些指标
  //1、设置参数及指标  如果是server模式并且没有指定gc策略，则默认使用UsePaallelGC
  // 64位下启用普通对象压缩
  //2、设置是否使用共享存储空间(开启对象压缩、指针压缩后关闭)
  //3、检查gc一致性
  //4、依据物理内存设置对内存大小
  //5、设置各种gc的参数标志：parallel、cms、g1、panew
  //6、初始化jvm的平衡因子参数
  //7、设定字节码重写标志
  //8、如果指定-XX:+AggressiveOpts参数，则设定加快编译，依赖于EliminateAutoBox及DoEscapeAnalysis标志位及C2下
  //9、根据是否指定gama lanucher及是否是调试状态设定暂停状态位
  jint ergo_result = Arguments::apply_ergo();
  if (ergo_result != JNI_OK) return ergo_result;
  //暂停
  if (PauseAtStartup) {
    os::pause();
  }
  // JVM创建时间统计 
  //记录jvm启动开始时间
  TraceVmCreationTime create_vm_timer;
  create_vm_timer.start();
  TraceTime timer("Create VM", TraceStartupTime);
  // Initialize the os module after parsing the args
  
    //解析参数后，初始化系统模块
  //1、如果使用linux下的posix线程库cpu时钟的话，使用pthread_getcpuclockid方法获取jvm线程
  //   的cpu时钟id，然后基于这个线程的时钟进行时间统计计数
  //2、分配一个linux内存单页并标记位为可循环读的安全入口点
  //3、初始化暂停/恢复运行的支持，主要通过注册信号处理程序支持该功能
  //4、注册信号处理程序
  //5、安装信号处理程序
  //6、计算线程栈大小
  //7、设置glibc、linux线程版本信息
  //8、设置linux系统进程文件描述符值
  //9、创建用于线程创建的线程锁
  //11、初始化线程的优先级
  jint os_init_2_result = os::init_2();
  if (os_init_2_result != JNI_OK) return os_init_2_result;
  //调整参数，主要针对numa架构下ParallelGC、OarallelOldGc调整堆内存参数
  jint adjust_after_os_result = Arguments::adjust_after_os();
  if (adjust_after_os_result != JNI_OK) return adjust_after_os_result;
  // 初始化 TLS
    // intialize TLS
  // 初始化线程本地存储,通过ptread_create_key创建一个线程特有的key_t,
  // 后面通过pthread_setspecific\pthread_getspecific设置、获取线程私有数据
  ThreadLocalStorage::init();
    // 启动本地内存跟踪，因此它可以在工作线程启动之前开始记录内存活动。 
  // 这是引导的第一阶段，JVM当前以单线程模式运行。
  // MemTracker::bootstrap_single_thread();
  // Initialize output stream logging
  // 初始化jvm的日志输出流，如果指定-Xloggc:logfilepath则根据参数指定生成输出流到文件
  ostream_init_log();
  // Convert -Xrun to -agentlib: if there is no JVM_OnLoad
  // Must be before create_vm_init_agents()
  // 如果指定了-Xrun参数，如：hprof性能分析、jdwp远程调试
  if (Arguments::init_libraries_at_startup()) {
    convert_vm_init_libraries_to_agents();
  }
  // Launch -agentlib/-agentpath and converted -Xrun agents
  if (Arguments::init_agents_at_startup()) {
    create_vm_init_agents();
  }
  // Initialize Threads state
  _thread_list = NULL;
  _number_of_threads = 0;
  _number_of_non_daemon_threads = 0;
  // Initialize global data structures and create system classes in heap
    // 1、初始化全局结构体
  // 2、在内存中创建jvm体系的class文件
  // 3、初始化事件记录日志对象
  // 4、初始化全局资源锁mutex
  // 5、内存池初始化large_pool、medium_pool、small_poll、tiny_pool
  // 6、启用该perfdata功能。此选项默认启用以允许JVM监视和性能测试，如果指定 -XX:+UsePerfData
  vm_init_globals();
  // 将java主线程附加到系统当前线程上,线程栈基和栈大小
  // 创建新的主java线程, 设置主线程状态为运行在jvm里面
  JavaThread* main_thread = new JavaThread();
  main_thread->set_thread_state(_thread_in_vm);
  // 记录栈基址、栈大小值
  main_thread->record_stack_base_and_size();
  // 初始化主线程的本地存储
  main_thread->initialize_thread_local_storage();
  // 绑定jniHandleBlock指针, 处理jni逻辑句柄
  main_thread->set_active_handles(JNIHandleBlock::allocate_block());
  
  // 设定main_thread为主线程
  if (!main_thread->set_as_starting_thread()) {
    vm_shutdown_during_initialization(
      "Failed necessary internal allocation. Out of swap space");
    delete main_thread;
    *canTryAgain = false; // don't let caller call JNI_CreateJavaVM again
    return JNI_ENOMEM;
  }
  // Enable guard page *after* os::create_main_thread(), otherwise it would
  // crash Linux VM, see notes in os_linux.cpp.
  main_thread->create_stack_guard_pages();
    //初始化Java级同步子系统
  ObjectMonitor::Initialize() ;
    // 初始化全局模块
  jint status = init_globals();
  if (status != JNI_OK) {
    delete main_thread;
    *canTryAgain = false; // don't let caller call JNI_CreateJavaVM again
    return status;
  }
  // 在 heap 堆创建完成之后调用
  main_thread->cache_global_variables();
  HandleMark hm;
  { MutexLocker mu(Threads_lock);
    Threads::add(main_thread);
  }
  // Any JVMTI raw monitors entered in onload will transition into
  // real raw monitor. VM is setup enough here for raw monitor enter.
  JvmtiExport::transition_pending_onload_raw_monitors();
  // Create the VMThread
  { TraceTime timer("Start VMThread", TraceStartupTime);
    VMThread::create();
    Thread* vmthread = VMThread::vm_thread();
    if (!os::create_thread(vmthread, os::vm_thread))
      vm_exit_during_initialization("Cannot create VM thread. Out of system resources.");
    // Wait for the VM thread to become ready, and VMThread::run to initialize
    // Monitors can have spurious returns, must always check another state flag
    {
      MutexLocker ml(Notify_lock);
      os::start_thread(vmthread);
      while (vmthread->active_handles() == NULL) {
        Notify_lock->wait();
      }
    }
  }
  assert (Universe::is_fully_initialized(), "not initialized");
  if (VerifyDuringStartup) {
    // Make sure we're starting with a clean slate.
    VM_Verify verify_op;
    VMThread::execute(&verify_op);
  }
  EXCEPTION_MARK;
  // 到这里虚拟机已经初始化成功了,但还未执行任何java的字节码代码
  if (DumpSharedSpaces) {
    MetaspaceShared::preload_and_dump(CHECK_0);
    ShouldNotReachHere();
  }
  JvmtiExport::enter_start_phase();
  // Notify JVMTI agents that VM has started (JNI is up) - nop if no agents.
  JvmtiExport::post_vm_start();
  {
    TraceTime timer("Initialize java.lang classes", TraceStartupTime);
    if (EagerXrunInit && Arguments::init_libraries_at_startup()) {
      create_vm_init_libraries();
    }
    initialize_class(vmSymbols::java_lang_String(), CHECK_0);
    // 初始化 java_lang.System 下面的类
    initialize_class(vmSymbols::java_lang_System(), CHECK_0);
    initialize_class(vmSymbols::java_lang_ThreadGroup(), CHECK_0);
    Handle thread_group = create_initial_thread_group(CHECK_0);
    Universe::set_main_thread_group(thread_group());
    // 装载Thread class对象
    initialize_class(vmSymbols::java_lang_Thread(), CHECK_0);
    //创建thread实例
    oop thread_object = create_initial_thread(thread_group, main_thread, CHECK_0);
    main_thread->set_threadObj(thread_object);
    // 标价java主线程开始运行
    java_lang_Thread::set_thread_status(thread_object,java_lang_Thread::RUNNABLE);
    initialize_class(vmSymbols::java_lang_Class(), CHECK_0);
    initialize_class(vmSymbols::java_lang_reflect_Method(), CHECK_0);
    initialize_class(vmSymbols::java_lang_ref_Finalizer(),  CHECK_0);
    call_initializeSystemClass(CHECK_0);
    JDK_Version::set_runtime_name(get_java_runtime_name(THREAD));
    JDK_Version::set_runtime_version(get_java_runtime_version(THREAD));
    initialize_class(vmSymbols::java_lang_OutOfMemoryError(), CHECK_0);
    initialize_class(vmSymbols::java_lang_NullPointerException(), CHECK_0);
    initialize_class(vmSymbols::java_lang_ClassCastException(), CHECK_0);
    initialize_class(vmSymbols::java_lang_ArrayStoreException(), CHECK_0);
    initialize_class(vmSymbols::java_lang_ArithmeticException(), CHECK_0);
    initialize_class(vmSymbols::java_lang_StackOverflowError(), CHECK_0);
    initialize_class(vmSymbols::java_lang_IllegalMonitorStateException(), CHECK_0);
    initialize_class(vmSymbols::java_lang_IllegalArgumentException(), CHECK_0);
  }
  initialize_class(vmSymbols::java_lang_Compiler(), CHECK_0);
  reset_vm_info_property(CHECK_0);
  quicken_jni_functions();
  if (TRACE_INITIALIZE() != JNI_OK) {
    vm_exit_during_initialization("Failed to initialize tracing backend");
  }
  // 设置标记jvm初始化运行完成
  set_init_completed();
  Metaspace::post_initialize();
  SystemDictionary::compute_java_system_loader(THREAD);
  if (HAS_PENDING_EXCEPTION) {
    vm_exit_during_initialization(Handle(THREAD, PENDING_EXCEPTION));
  }
  JvmtiExport::enter_live_phase();
  os::signal_init();
  if (!DisableAttachMechanism) {
    AttachListener::vm_start();
    if (StartAttachListener || AttachListener::init_at_startup()) {
      AttachListener::init();
    }
  }
  if (!EagerXrunInit && Arguments::init_libraries_at_startup()) {
    create_vm_init_libraries();
  }
  JvmtiExport::post_vm_initialized();
  if (TRACE_START() != JNI_OK) {
    vm_exit_during_initialization("Failed to start tracing backend.");
  }
  if (CleanChunkPoolAsync) {
    Chunk::start_chunk_pool_cleaner_task();
  }
  if (EnableInvokeDynamic) {
    initialize_class(vmSymbols::java_lang_invoke_MethodHandle(), CHECK_0);
    initialize_class(vmSymbols::java_lang_invoke_MemberName(), CHECK_0);
    initialize_class(vmSymbols::java_lang_invoke_MethodHandleNatives(), CHECK_0);
  }
  if (HAS_PENDING_EXCEPTION) {
    vm_exit(1);
  }
  if (Arguments::has_profile())       FlatProfiler::engage(main_thread, true);
  if (MemProfiling)                   MemProfiler::engage();
  StatSampler::engage();
  if (CheckJNICalls)                  JniPeriodicChecker::engage();
  BiasedLocking::init();
  if (JDK_Version::current().post_vm_init_hook_enabled()) {
    call_postVMInitHook(THREAD);
    // The Java side of PostVMInitHook.run must deal with all
    // exceptions and provide means of diagnosis.
    if (HAS_PENDING_EXCEPTION) {
      CLEAR_PENDING_EXCEPTION;
    }
  }
  {
      MutexLockerEx ml(PeriodicTask_lock, Mutex::_no_safepoint_check_flag);
      WatcherThread::make_startable();
      if (PeriodicTask::num_tasks() > 0) {
          WatcherThread::start();
      }
  }
  os::init_3();
  create_vm_timer.end();
  return JNI_OK;
}
```
此方法真正创建的jvm 其中JNIEnv对象的初始化在 JavaThread* main_thread = new JavaThread()时， JavaThread()这里调用JavaThread(bool is_attaching_via_jni = false)的构造方法，

然后执行initialize()-》set_jni_functions(jni_functions());完成初始化，

jni_functions()方法返回thread.cpp中的一个全局变量
