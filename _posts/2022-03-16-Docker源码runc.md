---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码runc

## runC
根据官方定义：runC是一个根据OCI（Open Container Initiative）标准创建并运行容器的CLI tool

容器的工业级标准化组织OCI(Open Container Initiative)出炉，这是业界大佬为避免容器生态和docker耦合过紧做的努力，也是docker做出的妥协
随着docker等容器引擎自身功能越来越丰富，其逐渐呈现出组件化的趋势（将底层交给OCI，自己则专注网络，配置管理，集群，编排，安全等方面）
内核中关于容器的开发也如火如荼，包括 capabilities, fs, net, uevent等和容器相关的子系统

NAME:
docker-runc init - initialize the namespaces and launch the process (do not call it outside of runc)

USAGE:
docker-runc init [arguments...]

## Factory 接口
```
type Factory interface {
	// Creates a new container with the given id and starts the initial process inside it.
	Create(id string, config *configs.Config) (Container, error)
 
	// Load takes an ID for an existing container and returns the container information
	// from the state.  This presents a read only view of the container.
	Load(id string) (Container, error)
 
	// StartInitialization is an internal API to libcontainer used during the reexec of the
	// container.
	//
	// Errors:
	// Pipe connection error
	// System error
	StartInitialization() error
 
	// Type returns info string about factory type (e.g. lxc, libcontainer...)
	Type() string
}
```

## namespaces定义
```
	NEWNET    NamespaceType = "NEWNET"
	NEWPID    NamespaceType = "NEWPID"
	NEWNS     NamespaceType = "NEWNS"
	NEWUTS    NamespaceType = "NEWUTS"
	NEWIPC    NamespaceType = "NEWIPC"
	NEWUSER   NamespaceType = "NEWUSER"
	NEWCGROUP NamespaceType = "NEWCGROUP"
```
接着通过init管道将容器配置p.bootstrapData写入管道中。然后再调用parseSync()函数，通过init管道与容器初始化进程进行同步，待其初始化完成之后，执行PreStart Hook等回调。关闭init管道容器创建完成

## runc init 分析
初始化iniit command
```
func init() {
	if len(os.Args) > 1 && os.Args[1] == "init" {
		runtime.GOMAXPROCS(1)
		runtime.LockOSThread()
	}
}
 
var initCommand = cli.Command{
	Name:  "init",
	Usage: `initialize the namespaces and launch the process (do not call it outside of runc)`,
	Action: func(context *cli.Context) error {
		factory, _ := libcontainer.New("")
		if err := factory.StartInitialization(); err != nil {
			// as the error is sent back to the parent there is no need to log
			// or write it to stderr because the parent process will handle this
			os.Exit(1)
		}
		panic("libcontainer: container init failed to exec")
	},
}
```
New函数

libcontainer.New函数初始化linuxFactory，实现了Factory接口
```
// New returns a linux based container factory based in the root directory and
// configures the factory with the provided option funcs.
func New(root string, options ...func(*LinuxFactory) error) (Factory, error) {
	if root != "" {
		if err := os.MkdirAll(root, 0700); err != nil {
			return nil, newGenericError(err, SystemError)
		}
	}
	l := &LinuxFactory{
		Root:      root,
		InitPath:  "/proc/self/exe",
		InitArgs:  []string{os.Args[0], "init"},
		Validator: validate.New(),
		CriuPath:  "criu",
	}
	Cgroupfs(l)
	for _, opt := range options {
		if opt == nil {
			continue
		}
		if err := opt(l); err != nil {
			return nil, err
		}
	}
	return l, nil
}
```
StartInitialization函数
路径 libcontainer/factory_linux.go
```
// StartInitialization loads a container by opening the pipe fd from the parent to read the configuration and state
// This is a low level implementation detail of the reexec and should not be consumed externally
func (l *LinuxFactory) StartInitialization() (err error)
```
获得pipe管道
```
	var (
		pipefd, fifofd int
		consoleSocket  *os.File
		envInitPipe    = os.Getenv("_LIBCONTAINER_INITPIPE")
		envFifoFd      = os.Getenv("_LIBCONTAINER_FIFOFD")
		envConsole     = os.Getenv("_LIBCONTAINER_CONSOLE")
	)
 
	// Get the INITPIPE.
	pipefd, err = strconv.Atoi(envInitPipe)
	if err != nil {
		return fmt.Errorf("unable to convert _LIBCONTAINER_INITPIPE=%s to int: %s", envInitPipe, err)
	}
 
	var (
		pipe = os.NewFile(uintptr(pipefd), "pipe")
		it   = initType(os.Getenv("_LIBCONTAINER_INITTYPE"))
	)
	defer pipe.Close()
```
newContainerInit函数
如果类型为 initStandard 则结构体 linuxStandardInit 实现了 Init 方法，容器内初始化做了一大堆事
```
func newContainerInit(t initType, pipe *os.File, consoleSocket *os.File, fifoFd int) (initer, error) {
	var config *initConfig
	if err := json.NewDecoder(pipe).Decode(&config); err != nil {
		return nil, err
	}
	if err := populateProcessEnvironment(config.Env); err != nil {
		return nil, err
	}
	switch t {
	case initSetns:
		return &linuxSetnsInit{
			pipe:          pipe,
			consoleSocket: consoleSocket,
			config:        config,
		}, nil
	case initStandard:
		return &linuxStandardInit{
			pipe:          pipe,
			consoleSocket: consoleSocket,
			parentPid:     unix.Getppid(),
			config:        config,
			fifoFd:        fifoFd,
		}, nil
	}
	return nil, fmt.Errorf("unknown init type %q", t)
}
```
## linuxStandardInit
路径 libcontainer/standard_init_linux.go
```
// linuxSetnsInit performs the container's initialization for running a new process
// inside an existing container.
type linuxSetnsInit struct {
	pipe          *os.File
	consoleSocket *os.File
	config        *initConfig
}
```

Init函数
setupNetwork: 配置容器的网络，调用第三方 netlink.LinkSetup
setupRoute: 配置容器静态路由信息，调用第三方 netlink.RouteAdd
label.Init:   检查selinux是否被启动并将结果存入全局变量。
finalizeNamespace: 根据config配置将需要的特权capabilities加入白名单，设置user namespace，关闭不需要的文件描述符。
unix.Openat: 只写方式打开fifo管道并写入0，会一直保持阻塞，直到管道的另一端以读方式打开，并读取内容
syscall.Exec 系统调用来执行用户所指定的在容器中运行的程序
配置 hostname、apparmor、processLabel、sysctl、readonlyPath、maskPath。create 虽然不会执行命令，但会检查命令路径，错误会在 create 期间返回

setupNetWork函数

配置容器的网络，调用第三方 netlink.LinkSetup，相当于命令ip link set $link up

如果不指定任何网络，只有loopback
```
// setupNetwork sets up and initializes any network interface inside the container.
func setupNetwork(config *initConfig) error {
	for _, config := range config.Networks {
		strategy, err := getStrategy(config.Type)
		if err != nil {
			return err
		}
		if err := strategy.initialize(config); err != nil {
			return err
		}
	}
	return nil
}
```
setupRoute

配置容器静态路由信息，调用第三方 netlink.RouteAdd，相当于命令ip route add $route
```
func setupRoute(config *configs.Config) error {
	for _, config := range config.Routes {
		_, dst, err := net.ParseCIDR(config.Destination)
		if err != nil {
			return err
		}
		src := net.ParseIP(config.Source)
		if src == nil {
			return fmt.Errorf("Invalid source for route: %s", config.Source)
		}
		gw := net.ParseIP(config.Gateway)
		if gw == nil {
			return fmt.Errorf("Invalid gateway for route: %s", config.Gateway)
		}
		l, err := netlink.LinkByName(config.InterfaceName)
		if err != nil {
			return err
		}
		route := &netlink.Route{
			Scope:     netlink.SCOPE_UNIVERSE,
			Dst:       dst,
			Src:       src,
			Gw:        gw,
			LinkIndex: l.Attrs().Index,
		}
		if err := netlink.RouteAdd(route); err != nil {
			return err
		}
	}
	return nil
}
```
syncParentReady函数

syncParentReady函数发送ready到pipe，等待父进程下发exec命令
```
// syncParentReady sends to the given pipe a JSON payload which indicates that
// the init is ready to Exec the child process. It then waits for the parent to
// indicate that it is cleared to Exec.
func syncParentReady(pipe io.ReadWriter) error {
	// Tell parent.
	if err := writeSync(pipe, procReady); err != nil {
		return err
	}
 
	// Wait for parent to give the all-clear.
	return readSync(pipe, procRun)
}
```
只写方式打开fifo管道并写入0，会一直保持阻塞，直到管道的另一端以读方式打开，并读取内容
```
	// Wait for the FIFO to be opened on the other side before exec-ing the
	// user process. We open it through /proc/self/fd/$fd, because the fd that
	// was given to us was an O_PATH fd to the fifo itself. Linux allows us to
	// re-open an O_PATH fd through /proc.
	fd, err := unix.Open(fmt.Sprintf("/proc/self/fd/%d", l.fifoFd), unix.O_WRONLY|unix.O_CLOEXEC, 0)
	if err != nil {
		return newSystemErrorWithCause(err, "open exec fifo")
	}
	if _, err := unix.Write(fd, []byte("0")); err != nil {
		return newSystemErrorWithCause(err, "write 0 exec fifo")
	}
	// Close the O_PATH fifofd fd before exec because the kernel resets
	// dumpable in the wrong order. This has been fixed in newer kernels, but
	// we keep this to ensure CVE-2016-9962 doesn't re-emerge on older kernels.
	// N.B. the core issue itself (passing dirfds to the host filesystem) has
	// since been resolved.
	// https://github.com/torvalds/linux/blob/v4.9/fs/exec.c#L1290-L1318
	unix.Close(l.fifoFd)
```
系统调用来执行用户所指定的在容器中运行的程序
```
	if err := syscall.Exec(name, l.config.Args[0:], os.Environ()); err != nil {
		return newSystemErrorWithCause(err, "exec user process")
	}
```