---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码Libcontainer原理

## Libcontainer概述
用于容器管理的包，管理namespaces、cgroups、capabilities以及文件系统来对容器控制。可用Libcontainer创建容器，并对容器进行管理。pivot_root 用于改变进程的根目录，可以将进程控制在rootfs中。如果rootfs是基于ramfs的（不支持pivot_root），那会在mount时使用MS_MOVE标志位加上chroot来顶替。

Libcontainer通过接口的方式定义了一系列容器管理的操作，包括处理容器的创建（Factory）、容器生命周期管理（Container）、进程生命周期管理（Process）等一系列接口。

## 容器启动过程
在Libcontainer中，p.cmd.Start创建子进程，就进入了pipe wait等待父写入pipe，p.cmd.Start创建了新的Namespace，这时子进程就已经在新的Namespace里了。
daemon线程在执行p.manager.Apply，创建新的Cgroup，并把子进程放到新的Cgroup中。
daemon线程做一些网络配置，会把容器的配置信息通过管道发给子进程。同时让子进程继续往下执行。
daemon线程则进入pipe wait阶段，容器剩下的初始化由子进程完成了。
rootfs的切换在setupRootfs函数中。（首先子进程会根据config，把host上的相关目录mount到容器的rootfs中，或挂载到一些虚拟文件系统上，这些挂载信息可能是-v指定的volume、容器的Cgroup信息、proc文件系统等）。
完成文件系统操作，就执行syscall.PivotRoot把容器的根文件系统切换rootfs
再做一些hostname及安全配置，就可以调用syscall.Exec执行容器中的init进程了
容器完成创建和运行操作，同时通知了父进程，此时，daemon线程会回到Docker的函数中，执行等待容器进程结束的操作，整个过程完成

## 容器检查点保存Checkpoint
收集进程与其子进程构成的树，并冻结所有进程。
收集任务（包括进程和线程）使用的所有资源，并保存。
清理收集资源的相关寄生代码，并与进程分离。

## 容器检查点恢复Restore
读取快照文件并解析出共享的资源，对多个进程共享的资源优先恢复，其他资源则随后需要时恢复。
使用fork恢复整个进程树，注意此时并不恢复线程，在第4步恢复。
恢复所有基础任务（包括进程和线程）资源，除了内存映射、计时器、证书和线程。这一步主要打开文件、准备namespace、创建socket连接等。
恢复进程运行的上下文环境，恢复剩下的其他资源，继续运行进程。

## 配置结构体Config
```
// Config defines configuration options for executing a process inside a contained environment.
type Config struct {
       // NoPivotRoot will use MS_MOVE and a chroot to jail the process into the container's rootfs
       // This is a common option when the container is running in ramdisk
       NoPivotRoot bool `json:"no_pivot_root"`

       // ParentDeathSignal specifies the signal that is sent to the container's process in the case
       // that the parent process dies.
       ParentDeathSignal int `json:"parent_death_signal"`

       // PivotDir allows a custom directory inside the container's root filesystem to be used as pivot, when NoPivotRoot is not set.
       // When a custom PivotDir not set, a temporary dir inside the root filesystem will be used. The pivot dir needs to be writeable.
       // This is required when using read only root filesystems. In these cases, a read/writeable path can be (bind) mounted somewhere inside the root filesystem to act as pivot.
       PivotDir string `json:"pivot_dir"`

       // Path to a directory containing the container's root filesystem.
       Rootfs string `json:"rootfs"`

       // Readonlyfs will remount the container's rootfs as readonly where only externally mounted
       // bind mounts are writtable.
       Readonlyfs bool `json:"readonlyfs"`

       // Specifies the mount propagation flags to be applied to /.
       RootPropagation int `json:"rootPropagation"`

       // Mounts specify additional source and destination paths that will be mounted inside the container's
       // rootfs and mount namespace if specified
       Mounts []*Mount `json:"mounts"`

       // The device nodes that should be automatically created within the container upon container start.  Note, make sure that the node is marked as allowed in the cgroup as well!
       Devices []*Device `json:"devices"`

       MountLabel string `json:"mount_label"`

       // Hostname optionally sets the container's hostname if provided
       Hostname string `json:"hostname"`

       // Namespaces specifies the container's namespaces that it should setup when cloning the init process
       // If a namespace is not provided that namespace is shared from the container's parent process
       Namespaces Namespaces `json:"namespaces"`

       // Capabilities specify the capabilities to keep when executing the process inside the container
       // All capbilities not specified will be dropped from the processes capability mask
       Capabilities []string `json:"capabilities"`

       // Networks specifies the container's network setup to be created
       Networks []*Network `json:"networks"`

       // Routes can be specified to create entries in the route table as the container is started
       Routes []*Route `json:"routes"`

       // Cgroups specifies specific cgroup settings for the various subsystems that the container is
       // placed into to limit the resources the container has available
       Cgroups *Cgroup `json:"cgroups"`

       // AppArmorProfile specifies the profile to apply to the process running in the container and is
       // change at the time the process is execed
       AppArmorProfile string `json:"apparmor_profile,omitempty"`

       // ProcessLabel specifies the label to apply to the process running in the container.  It is
       // commonly used by selinux
       ProcessLabel string `json:"process_label,omitempty"`

       // Rlimits specifies the resource limits, such as max open files, to set in the container
       // If Rlimits are not set, the container will inherit rlimits from the parent process
       Rlimits []Rlimit `json:"rlimits,omitempty"`

       // OomScoreAdj specifies the adjustment to be made by the kernel when calculating oom scores
       // for a process. Valid values are between the range [-1000, '1000'], where processes with
       // higher scores are preferred for being killed.
       // More information about kernel oom score calculation here: https://lwn.net/Articles/317814/
       OomScoreAdj int `json:"oom_score_adj"`

       // UidMappings is an array of User ID mappings for User Namespaces
       UidMappings []IDMap `json:"uid_mappings"`

       // GidMappings is an array of Group ID mappings for User Namespaces
       GidMappings []IDMap `json:"gid_mappings"`

       // MaskPaths specifies paths within the container's rootfs to mask over with a bind
       // mount pointing to /dev/null as to prevent reads of the file.
       MaskPaths []string `json:"mask_paths"`

       // ReadonlyPaths specifies paths within the container's rootfs to remount as read-only
       // so that these files prevent any writes.
       ReadonlyPaths []string `json:"readonly_paths"`

       // Sysctl is a map of properties and their values. It is the equivalent of using
       // sysctl -w my.property.name value in Linux.
       Sysctl map[string]string `json:"sysctl"`

       // Seccomp allows actions to be taken whenever a syscall is made within the container.
       // A number of rules are given, each having an action to be taken if a syscall matches it.
       // A default action to be taken if no rules match is also given.
       Seccomp *Seccomp `json:"seccomp"`

       // NoNewPrivileges controls whether processes in the container can gain additional privileges.
       NoNewPrivileges bool `json:"no_new_privileges,omitempty"`

       // Hooks are a collection of actions to perform at various container lifecycle events.
       // CommandHooks are serialized to JSON, but other hooks are not.
       Hooks *Hooks

       // Version is the version of opencontainer specification that is supported.
       Version string `json:"version"`

       // Labels are user defined metadata that is stored in the config and populated on the state
       Labels []string `json:"labels"`

       // NoNewKeyring will not allocated a new session keyring for the container.  It will use the
       // callers keyring in this case.
       NoNewKeyring bool `json:"no_new_keyring"`
}

```

## 容器接口BaseContainer
```
// BaseContainer is a libcontainer container object.
//
// Each container is thread-safe within the same process. Since a container can
// be destroyed by a separate process, any function may return that the container
// was not found. BaseContainer includes methods that are platform agnostic.
type BaseContainer interface {
       // Returns the ID of the container
       ID() string

       // Returns the current status of the container.
       //
       // errors:
       // ContainerNotExists - Container no longer exists,
       // Systemerror - System error.
       Status() (Status, error)

       // State returns the current container's state information.
       //
       // errors:
       // SystemError - System error.
       State() (*State, error)

       // Returns the current config of the container.
       Config() configs.Config

       // Returns the PIDs inside this container. The PIDs are in the namespace of the calling process.
       //
       // errors:
       // ContainerNotExists - Container no longer exists,
       // Systemerror - System error.
       //
       // Some of the returned PIDs may no longer refer to processes in the Container, unless
       // the Container state is PAUSED in which case every PID in the slice is valid.
       Processes() ([]int, error)

       // Returns statistics for the container.
       //
       // errors:
       // ContainerNotExists - Container no longer exists,
       // Systemerror - System error.
       Stats() (*Stats, error)

       // Set resources of container as configured
       //
       // We can use this to change resources when containers are running.
       //
       // errors:
       // SystemError - System error.
       Set(config configs.Config) error

       // Start a process inside the container. Returns error if process fails to
       // start. You can track process lifecycle with passed Process structure.
       //
       // errors:
       // ContainerNotExists - Container no longer exists,
       // ConfigInvalid - config is invalid,
       // ContainerPaused - Container is paused,
       // SystemError - System error.
       Start(process *Process) (err error)

       // Run immediatly starts the process inside the conatiner.  Returns error if process
       // fails to start.  It does not block waiting for the exec fifo  after start returns but
       // opens the fifo after start returns.
       //
       // errors:
       // ContainerNotExists - Container no longer exists,
       // ConfigInvalid - config is invalid,
       // ContainerPaused - Container is paused,
       // SystemError - System error.
       Run(process *Process) (err error)

       // Destroys the container after killing all running processes.
       //
       // Any event registrations are removed before the container is destroyed.
       // No error is returned if the container is already destroyed.
       //
       // errors:
       // SystemError - System error.
       Destroy() error

       // Signal sends the provided signal code to the container's initial process.
       //
       // errors:
       // SystemError - System error.
       Signal(s os.Signal) error

       // Exec signals the container to exec the users process at the end of the init.
       //
       // errors:
       // SystemError - System error.
       Exec() error
}

```

## Factory接口
Factory对象为容器创建和初始化工作提供了一组抽象接口
```
type Factory interface {
       // Creates a new container with the given id and starts the initial process inside it.
       // id must be a string containing only letters, digits and underscores and must contain
       // between 1 and 1024 characters, inclusive.
       //
       // The id must not already be in use by an existing container. Containers created using
       // a factory with the same path (and file system) must have distinct ids.
       //
       // Returns the new container with a running process.
       //
       // errors:
       // IdInUse - id is already in use by a container
       // InvalidIdFormat - id has incorrect format
       // ConfigInvalid - config is invalid
       // Systemerror - System error
       //
       // On error, any partially created container parts are cleaned up (the operation is atomic).
       Create(id string, config *configs.Config) (Container, error)

       // Load takes an ID for an existing container and returns the container information
       // from the state.  This presents a read only view of the container.
       //
       // errors:
       // Path does not exist
       // Container is stopped
       // System error
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