---
layout: post
categories: [Dubbo]
description: none
keywords: Dubbo
---
# DubboBootstrap启动
提供者在启动时的服务发布是在 DubboBootstrap 中完成的。

## DubboBootstrap构造函数
DubboBootstrap 是 Dubbo 服务启动的核心类，在其中完成了Dubbo服务暴露的过程。

DubboBootstrap 是单例的，其构造函数如下：
```
    private DubboBootstrap() {
    	// 获取 配置管理
        configManager = ApplicationModel.getConfigManager();
        // 获取 环境
        environment = ApplicationModel.getEnvironment();

        DubboShutdownHook.getDubboShutdownHook().register();
        // 设置回调，当服务关闭时触发该回调，调用 DubboBootstrap#destroy 来销毁服务。
        ShutdownHookCallbacks.INSTANCE.addCallback(new ShutdownHookCallback() {
            @Override
            public void callback() throws Throwable {
                DubboBootstrap.this.destroy();
            }
        });
    }

```

## DubboBootstrap#start
DubboBootstrap#start 的实现如下：
```
    public DubboBootstrap start() {
    	// cas 保证只启动一次
        if (started.compareAndSet(false, true)) {
        	// 1. 服务配置初始化
            initialize();
            // 2. Dubbo服务导出
            exportServices();

            // Not only provider register
            // 3. 元数据中心服务暴露，当使用服务自省模式时才会执行该部分逻辑
            // 3.1 不仅仅提供者注册 || 存在导出的服务
            if (!isOnlyRegisterProvider() || hasExportedServices()) {
                // 3.2 导出元数据服务
                exportMetadataService();
                // 3.3 如果需要注册本地服务实例
                registerServiceInstance();
            }
			// 4. 服务引用流程
            referServices();
        }
        return this;
    }

```

下面我们按照注释顺序来看：

## 服务初始化
在 DubboBootstrap#initialize 中完成了 Dubbo的配置检查和初始化，其具体实现如下：
```
    private void initialize() {
    	//  CAS 防止多次调用
        if (!initialized.compareAndSet(false, true)) {
            return;
        }
		// 1. 初始化 FrameworkExt 扩展类。 FrameworkExt 是 SPI接口，这里获取所有的 实现类并且调用FrameworkExt#initialized 来初始化
        ApplicationModel.iniFrameworkExts();
		// 2. 启用配置中心并刷新本地配置
        startConfigCenter();
		// 3. 在默认是zk作为注册中心时，如果没有配置配置中心，则使用注册中心作为配置中心
        useRegistryAsConfigCenterIfNecessary();
		// 4. 启动元数据中心配置
        startMetadataReport();
		// 5. 加载远程配置
        loadRemoteConfigs();
		// 6. 检查本地配置是否合法
        checkGlobalConfigs();
		// 7. 初始化元数据中心 Service
        initMetadataService();
		// 8. 初始化元数据中心导出类
        initMetadataServiceExporter();
		// 9. 初始化监听器
        initEventListener();
    }

```
在 Spring 中 Dubbo 的初始化工作和 Main 方法启动基本也是类似的，只不过在执行过程中稍微有些区别。下面我们按照注释来具体说明

### ApplicationModel.iniFrameworkExts()
ApplicationModel.iniFrameworkExts() 是执行框架扩展实现类的初始化方法，其实现如下：
```
    public static void iniFrameworkExts() {
    	// 获取支持的框架扩展FrameworkExt 实现 
        Set<FrameworkExt> exts = ExtensionLoader.getExtensionLoader(FrameworkExt.class).getSupportedExtensionInstances();
        for (FrameworkExt ext : exts) {
        	// 执行初始化方法
            ext.initialize();
        }
    }

```
其中 FrameworkExt 是 Dubbo 提供的 框架扩展 SPI 接口，继承了 Lifecycle 接口，如下：

```
@SPI
public interface FrameworkExt extends Lifecycle {
}

...

// Lifecycle  接口实现如下
public interface Lifecycle {

    void initialize() throws IllegalStateException;

    void start() throws IllegalStateException;

    void destroy() throws IllegalStateException;
}

```
Lifecycle 是 Dubbo 组件的声明周期接口，SPI 扩展实现类时在对应的生命周期会调用对应的方法。

Dubbo 提供的FrameworkExt 有三个实现 ： ConfigManager、Environment、ServiceRepository。我们这里先来看 initialize 方法。只有 Environment#initialize 有具体操作，如下：
```
	// 加载了配置中心的配置并保存到了  org.apache.dubbo.common.config.Environment 的 map 中。
    @Override
    public void initialize() throws IllegalStateException {
    	// 获取 配置管理
        ConfigManager configManager = ApplicationModel.getConfigManager();
        // 获取默认的配置中心
        Optional<Collection<ConfigCenterConfig>> defaultConfigs = configManager.getDefaultConfigCenter();
        // 如果存在配置中心，则从配置中心获取配置，保存到本地。
        defaultConfigs.ifPresent(configs -> {
            for (ConfigCenterConfig config : configs) {
                this.setExternalConfigMap(config.getExternalConfiguration());
                this.setAppExternalConfigMap(config.getAppExternalConfiguration());
            }
        });
    }

```

