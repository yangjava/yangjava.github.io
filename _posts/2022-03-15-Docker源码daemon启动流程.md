---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码daemon启动流程

## docker daemon的入口main
源码
docker daemon的main函数位于/moby/cmd/dockerd/docker.go，代码的主要内容是：
```
func main() {
	if reexec.Init() {
		return
	}

	// initial log formatting; this setting is updated after the daemon configuration is loaded.
	logrus.SetFormatter(&logrus.TextFormatter{
		TimestampFormat: jsonmessage.RFC3339NanoFixed,
		FullTimestamp:   true,
	})

	// Set terminal emulation based on platform as required.
	_, stdout, stderr := term.StdStreams()

	initLogging(stdout, stderr)

	onError := func(err error) {
		fmt.Fprintf(stderr, "%s\n", err)
		os.Exit(1)
	}

	cmd, err := newDaemonCommand()
	if err != nil {
		onError(err)
	}
	cmd.SetOutput(stdout)
	if err := cmd.Execute(); err != nil {
		onError(err)
	}
}

```

## newDaemonCommand()
newDaemonCommand()主代码
newDaemonCommand()包含daemon初始化的流程，主要代码为:
```
func newDaemonCommand() (*cobra.Command, error) {
	opts := newDaemonOptions(config.New())

	cmd := &cobra.Command{
		Use:           "dockerd [OPTIONS]",
		Short:         "A self-sufficient runtime for containers.",
		SilenceUsage:  true,
		SilenceErrors: true,
		Args:          cli.NoArgs,
		RunE: func(cmd *cobra.Command, args []string) error {
			opts.flags = cmd.Flags()
			return runDaemon(opts)
		},
		DisableFlagsInUseLine: true,
		Version:               fmt.Sprintf("%s, build %s", dockerversion.Version, dockerversion.GitCommit),
	}
	cli.SetupRootCommand(cmd)

	flags := cmd.Flags()
	//daemon command执行时，执行该函数returnrunDaemon(opts)
	flags.BoolP("version", "v", false, "Print version information and quit")
	defaultDaemonConfigFile, err := getDefaultDaemonConfigFile()
	if err != nil {
		return nil, err
	}
	flags.StringVar(&opts.configFile, "config-file", defaultDaemonConfigFile, "Daemon configuration file")
	opts.InstallFlags(flags)
	if err := installConfigFlags(opts.daemonConfig, flags); err != nil {
		return nil, err
	}
	installServiceFlags(flags)

	return cmd, nil
}
```

## runDaemon(opts)
runDaemon()是在daemon command使用时被调用，主要代码为：
```
funcrunDaemon(optsdaemonOptions)error{daemonCli:=NewDaemonCli()//创建daemon客户端对象daemonCli.start(opts)////启动daemonCli}
```
它包括了NewDaemonCli()和daemonCli.start(opts)两个部分，下面将详细分析这两个部分。

## NewDaemonCli()
NewDaemonCli()创建一个DaemonCli结构体对象，该结构体包含配置信息，配置文件，参数信息，APIServer,Daemon对象，authzMiddleware（认证插件），代码如下:
```
// DaemonCli represents the daemon CLI.
type DaemonCli struct {
	*config.Config  //配置信息
	configFile *string  //配置文件
	flags      *pflag.FlagSet //flag参数信息

	api             *apiserver.Server  //APIServer:提供api服务，定义在docker/api/server/server.go
	d               *daemon.Daemon  //Daemon对象,结构体定义在daemon/daemon.go文件中
	authzMiddleware *authorization.Middleware // authzMiddleware enables to dynamically reload the authorization plugins
}
```
其中APIServer在接下来的daemonCli.start()实现过程中具有非常重要的作用。
apiserver.Server的结构体为：
```
// Server contains instance details for the server
type Server struct {
	cfg           *Config  //apiserver的配置信息
	servers       []*HTTPServer //httpServer结构体对象，包括http.Server和net.Listener监听器。
	routers       []router.Router //路由表对象Route,包括Handler,Method, Path
	routerSwapper *routerSwapper  //路由交换器对象，使用新的路由交换旧的路由器
	middlewares   []middleware.Middleware //中间件
}
```

## daemonCli.start(opts)
daemonCli.start(opts)的主要代码以实现如下：
```
func (cli *DaemonCli) start(opts *daemonOptions) (err error) {
	stopc := make(chan bool)
	defer close(stopc)

	// warn from uuid package when running the daemon
	uuid.Loggerf = logrus.Warnf
    //1.设置默认可选项参数
	opts.SetDefaultOptions(opts.flags)
//2.根据opts对象信息来加载DaemonCli的配置信息config对象，并将该config对象配置到DaemonCli结构体对象中去
	if cli.Config, err = loadDaemonCliConfig(opts); err != nil {
		return err
	}

	if err := configureDaemonLogs(cli.Config); err != nil {
		return err
	}

	logrus.Info("Starting up")
    //3.对DaemonCli结构体中的其它成员根据opts进行配置
	cli.configFile = &opts.configFile
	//4.根据DaemonCli结构体对象中的信息定义APIServer配置信息结构体对象&apiserver.Config(包括tls传输层协议信息)
	cli.flags = opts.flags

	if cli.Config.Debug {
		debug.Enable()
	}

	if cli.Config.Experimental {
		logrus.Warn("Running experimental build")
		if cli.Config.IsRootless() {
			logrus.Warn("Running in rootless mode. Cgroups, AppArmor, and CRIU are disabled.")
		}
		if rootless.RunningWithRootlessKit() {
			logrus.Info("Running with RootlessKit integration")
			if !cli.Config.IsRootless() {
				return fmt.Errorf("rootless mode needs to be enabled for running with RootlessKit")
			}
		}
	} else {
		if cli.Config.IsRootless() {
			return fmt.Errorf("rootless mode is supported only when running in experimental mode")
		}
	}
	// return human-friendly error before creating files
	if runtime.GOOS == "linux" && os.Geteuid() != 0 {
		return fmt.Errorf("dockerd needs to be started with root. To see how to run dockerd in rootless mode with unprivileged user, see the documentation")
	}

	system.InitLCOW(cli.Config.Experimental)

	if err := setDefaultUmask(); err != nil {
		return err
	}

	// Create the daemon root before we create ANY other files (PID, or migrate keys)
	// to ensure the appropriate ACL is set (particularly relevant on Windows)
	if err := daemon.CreateDaemonRoot(cli.Config); err != nil {
		return err
	}

	if err := system.MkdirAll(cli.Config.ExecRoot, 0700, ""); err != nil {
		return err
	}

	potentiallyUnderRuntimeDir := []string{cli.Config.ExecRoot}

	if cli.Pidfile != "" {
		pf, err := pidfile.New(cli.Pidfile)
		if err != nil {
			return errors.Wrap(err, "failed to start daemon")
		}
		potentiallyUnderRuntimeDir = append(potentiallyUnderRuntimeDir, cli.Pidfile)
		defer func() {
			if err := pf.Remove(); err != nil {
				logrus.Error(err)
			}
		}()
	}

	if cli.Config.IsRootless() {
		// Set sticky bit if XDG_RUNTIME_DIR is set && the file is actually under XDG_RUNTIME_DIR
		if _, err := homedir.StickRuntimeDirContents(potentiallyUnderRuntimeDir); err != nil {
			// StickRuntimeDirContents returns nil error if XDG_RUNTIME_DIR is just unset
			logrus.WithError(err).Warn("cannot set sticky bit on files under XDG_RUNTIME_DIR")
		}
	}
   //这里是newAPIServerConfig内部实现
   //5.根据定义好的&apiserver.Config新建APIServer对象，并赋值到DaemonCli实例的对应属性中
	serverConfig, err := newAPIServerConfig(cli)
	if err != nil {
		return errors.Wrap(err, "failed to create API server")
	}
	cli.api = apiserver.New(serverConfig)
    //开启服务端监听
	hosts, err := loadListeners(cli, serverConfig)
	// 这里是loadListeners内部实现
	//6.解析host文件及传输协议（tcp）等内容
	//7.根据host解析内容初始化监听器
	//8.为建立好的APIServer设置我们初始化的监听器listener，可以监听该地址的连接
	//9.根据DaemonCli中的相关信息来新建libcontainerd对象
	if err != nil {
		return errors.Wrap(err, "failed to load listeners")
	}
    //下面是initContainerD的内部实现，启动containerd
	ctx, cancel := context.WithCancel(context.Background())
	waitForContainerDShutdown, err := cli.initContainerD(ctx)
	if waitForContainerDShutdown != nil {
		defer waitForContainerDShutdown(10 * time.Second)
	}
	if err != nil {
		cancel()
		return err
	}
	defer cancel()
 //  设置信号捕获,进入Trap()函数，可以看到os.Interrupt, syscall.SIGTERM, syscall.SIGQUIT, syscall.SIGPIPE
 //	四种信号，这里传入一个cleanup()函数，当捕获到这四种信号时，可以利用该函数进行shutdown善后处理
	signal.Trap(func() {
		cli.stop()
		<-stopc // wait for daemonCli.start() to return
	}, logrus.StandardLogger())
// 处于阻塞状态，等待stopc通道返回数据
//11.提前通知系统api可以工作了，但是要在daemon安装成功之后
//12.根据DaemonCli的配置信息，注册的服务对象及libcontainerd对象来构建Daemon对象
	// Notify that the API is active, but before daemon is set up.
	preNotifySystem()

	pluginStore := plugin.NewStore()

	if err := cli.initMiddlewares(cli.api, serverConfig, pluginStore); err != nil {
		logrus.Fatalf("Error creating middlewares: %v", err)
	}
    //13.将新建的Daemon对象与DaemonCli相关联cli.d=d
    
	d, err := daemon.NewDaemon(ctx, cli.Config, pluginStore)
	if err != nil {
		return errors.Wrap(err, "failed to start daemon")
	}

	d.StoreHosts(hosts)

	// validate after NewDaemon has restored enabled plugins. Don't change order.
	if err := validateAuthzPlugins(cli.Config.AuthorizationPlugins, pluginStore); err != nil {
		return errors.Wrap(err, "failed to validate authorization plugin")
	}

	cli.d = d

	if err := cli.startMetricsServer(cli.Config.MetricsAddress); err != nil {
		return err
	}
    //14.新建cluster对象
    //下面是createAndStartCluster内部实现，启动集群实例
    //15.重启Swarm容器d.RestartSwarmContainers()
    //16.初始化路由器initRouter(api,d,c)
    //17.新建goroutine来监听apiserver执行情况，当执行报错时通道serverAPIWait就会传出错误信息goapi.Wait(serveAPIWait)
    //18.通知系统Daemon已经安装完成，可以提供api服务了notifySystem()
    //19.等待apiserver执行出现错误，没有错误则会阻塞到该语句，直到server API完成errAPI:=<-serveAPIWait
    //20.执行到这一步说明，serverAPIWait有错误信息传出（一下均是），所以对cluster进行清理操作c.Cleanup()
    //21.关闭
	c, err := createAndStartCluster(cli, d)
	if err != nil {
		logrus.Fatalf("Error starting cluster component: %v", err)
	}

	// Restart all autostart containers which has a swarm endpoint
	// and is not yet running now that we have successfully
	// initialized the cluster.
	d.RestartSwarmContainers()

	logrus.Info("Daemon has completed initialization")

	routerOptions, err := newRouterOptions(cli.Config, d)
	if err != nil {
		return err
	}
	routerOptions.api = cli.api
	routerOptions.cluster = c

	initRouter(routerOptions)

	go d.ProcessClusterNotifications(ctx, c.GetWatchStream())

	cli.setupConfigReloadTrap()

	// The serve API routine never exits unless an error occurs
	// We need to start it as a goroutine and wait on it so
	// daemon doesn't exit
	serveAPIWait := make(chan error)
	go cli.api.Wait(serveAPIWait)

	// after the daemon is done setting up we can notify systemd api
	notifySystem()

	// Daemon is fully initialized and handling API traffic
	// Wait for serve API to complete
	errAPI := <-serveAPIWait
	c.Cleanup()

	shutdownDaemon(d)

	// Stop notification processing and any background processes
	cancel()

	if errAPI != nil {
		return errors.Wrap(errAPI, "shutting down due to ServeAPI error")
	}

	logrus.Info("Daemon shutdown complete")
	return nil
}
```
这部分代码实现了daemonCli、apiserver的初始化，其中apiserver的功能是处理client段发送请求，并将请求路由出去。所以initRouter(api, d, c)中包含了api路由的具体实现。

## 从daemon.NewDaemon()分析daemon启动过程中的initNetworkController()
从daemon.NewDaemon()到initNetworkController()的过程为：daemon.NewDaemon()—>d.restore()—>daemon.initNetworkController()

首先，从NewDaemon()中分析，主要代码如下：
```
func NewDaemon(config *config.Config, registryService registry.Service, containerdRemote libcontainerd.Remote, pluginStore *plugin.Store) (daemon *Daemon, err error) {
	...
	if err := d.restore(); err != nil {
		return nil, err
	}
	...
}
```
再到d.restore()的调用，主要代码如下，
```
func (daemon *Daemon) restore() error {
	...
	daemon.netController, err = daemon.initNetworkController(daemon.configStore, activeSandboxes)
	...
}
```
下面详细分析daemon启动过程中的网络初始化daemon.initNetworkController(),代码位于moby/daemon/daemon_unix.go#L727#L774。

函数主要做了以下两件事情：

初始化controller
初始化null(none的驱动)/host/bridge三个内置网络
主要代码为：
```
func (daemon *Daemon) initNetworkController(config *config.Config, activeSandboxes map[string]interface{}) (libnetwork.NetworkController, error) {
	netOptions, err := daemon.networkOptions(config, daemon.PluginStore, activeSandboxes)
	//生成controller对象
	controller, err := libnetwork.New(netOptions...)
	// Initialize default network on "null"
	if n, _ := controller.NetworkByName("none"); n == nil {
		_, err := controller.NewNetwork("null", "none", "", libnetwork.NetworkOptionPersist(true))
	}

	// Initialize default network on "host"
	if n, _ := controller.NetworkByName("host"); n == nil {
		_, err := controller.NewNetwork("host", "host", "", libnetwork.NetworkOptionPersist(true))
	}

	// Clear stale bridge network
	n, err := controller.NetworkByName("bridge"); err == nil {
		n.Delete(); 
	}

	if !config.DisableBridge {
		// Initialize default driver "bridge"
		err := initBridgeDriver(controller, config); err != nil {
			return nil, err
		}
	} else {
		removeDefaultBridgeInterface()
	}

	return controller, nil
}
```