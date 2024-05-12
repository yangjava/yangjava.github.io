---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码namespace

## Config
libcontainer/configs/config.go 中结构体 Config 中使用 namespace，Config 为在容器环境中执行进行定义了配置选项
```
// Config defines configuration options for executing a process inside a contained environment.
type Config struct {
       // Namespaces specifies the container's namespaces that it should setup when cloning the init process
       // If a namespace is not provided that namespace is shared from the container's parent process
       Namespaces Namespaces `json:"namespaces"`
}


```
## namespace
libcontainer/configs/namespace_linux.go 中结构体 Namespace 定义了 namespace，Config 为在容器环境中执行进行定义了配置选项，只有两项内容，类型与路径
```
// Namespace defines configuration for each namespace.  It specifies an
// alternate path that is able to be joined via setns.
type Namespace struct {
	Type NamespaceType `json:"type"`
	Path string        `json:"path"`
}
```
namespace 类型
libcontainer/configs/namespace_linux.go 中定义类型如下所示，包括 net，pid，mnt，ipc，user，uts
```
const (
       NEWNET  NamespaceType = "NEWNET"
       NEWPID  NamespaceType = "NEWPID"
       NEWNS   NamespaceType = "NEWNS"
       NEWUTS  NamespaceType = "NEWUTS"
       NEWIPC  NamespaceType = "NEWIPC"
       NEWUSER NamespaceType = "NEWUSER"
)
```

## namespace 方法
libcontainer/configs/namespace_linux.go 中函数 IsNamespaceSupported 返回是否 namespace 可以使用，使用了全局变量 supportedNamespaces，key 为类型，value 为 true / false。NsName 转换 namespace 类型为相应的文件名，判断 /proc/self/ns/${type-to-file} 存在与否
```
// IsNamespaceSupported returns whether a namespace is available or
// not
func IsNamespaceSupported(ns NamespaceType) bool {
	nsLock.Lock()
	defer nsLock.Unlock()
	supported, ok := supportedNamespaces[ns]
	if ok {
		return supported
	}
	nsFile := NsName(ns)
	// if the namespace type is unknown, just return false
	if nsFile == "" {
		return false
	}
	_, err := os.Stat(fmt.Sprintf("/proc/self/ns/%s", nsFile))
	// a namespace is supported if it exists and we have permissions to read it
	supported = err == nil
	supportedNamespaces[ns] = supported
	return supported
}
```
libcontainer/configs/namespace_linux.go 中函数 GetPath 路径为 /proc/${pid}/ns/${type-to-file}
```
func (n *Namespace) GetPath(pid int) string {
       return fmt.Sprintf("/proc/%d/ns/%s", pid, NsName(n.Type))
}


```
libcontainer/configs/namespace_linux.go 中 Namespaces 定义的函数如下都比较简单
```
type Namespaces []Namespace
 
func (n *Namespaces) Remove(t NamespaceType) bool {
}
 
func (n *Namespaces) Add(t NamespaceType, path string) {
}
 
func (n *Namespaces) index(t NamespaceType) int {
}
 
func (n *Namespaces) Contains(t NamespaceType) bool {
}
 
func (n *Namespaces) PathOf(t NamespaceType) string {
}
```

## namespaces 系统调用参数
libcontainer/configs/namespace_syscall.go 中 namespaceInfo 定义了 namespce 系统调用参数
```
var namespaceInfo = map[NamespaceType]int{
	NEWNET:    unix.CLONE_NEWNET,
	NEWNS:     unix.CLONE_NEWNS,
	NEWUSER:   unix.CLONE_NEWUSER,
	NEWIPC:    unix.CLONE_NEWIPC,
	NEWUTS:    unix.CLONE_NEWUTS,
	NEWPID:    unix.CLONE_NEWPID,
	NEWCGROUP: unix.CLONE_NEWCGROUP,
}
```
CloneFlags 函数遍历数组中所有 namespace 类型，返回 clone flags
```
// CloneFlags parses the container's Namespaces options to set the correct
// flags on clone, unshare. This function returns flags only for new namespaces.
func (n *Namespaces) CloneFlags() uintptr {
       var flag int
       for _, v := range *n {
              if v.Path != "" {
                     continue
              }
              flag |= namespaceInfo[v.Type]
       }
       return uintptr(flag)
}
```
## namespace 调用函数
libcontainer/container_linux.go 中函数 newInitProcess 调用 CloneFlags 获得，传给 bootstrapData 函数使用
```
func (c *linuxContainer) newInitProcess(p *Process, cmd *exec.Cmd, parentPipe, childPipe *os.File) (*initProcess, error) {
	cmd.Env = append(cmd.Env, "_LIBCONTAINER_INITTYPE="+string(initStandard))
	nsMaps := make(map[configs.NamespaceType]string)
	for _, ns := range c.config.Namespaces {
		if ns.Path != "" {
			nsMaps[ns.Type] = ns.Path
		}
	}
	_, sharePidns := nsMaps[configs.NEWPID]
	data, err := c.bootstrapData(c.config.Namespaces.CloneFlags(), nsMaps)
	if err != nil {
		return nil, err
	}
	init := &initProcess{
		cmd:             cmd,
		childPipe:       childPipe,
		parentPipe:      parentPipe,
		manager:         c.cgroupManager,
		intelRdtManager: c.intelRdtManager,
		config:          c.newInitConfig(p),
		container:       c,
		process:         p,
		bootstrapData:   data,
		sharePidns:      sharePidns,
	}
	c.initProcess = init
	return init, nil
}
```
libcontainer/container_linux.go 中函数 bootstrapData 使用 netlink 请求与内核进行通信，添加的数据类型有 CloneFlagsAttr，NsPathAttr，UidmapAttr，GidmapAttr，OomScoreAdjAttr，RootlessAttr 等，将数据进行序列化
```
// bootstrapData encodes the necessary data in netlink binary format
// as a io.Reader.
// Consumer can write the data to a bootstrap program
// such as one that uses nsenter package to bootstrap the container's
// init process correctly, i.e. with correct namespaces, uid/gid
// mapping etc.
func (c *linuxContainer) bootstrapData(cloneFlags uintptr, nsMaps map[configs.NamespaceType]string) (io.Reader, error) {
	// create the netlink message
	r := nl.NewNetlinkRequest(int(InitMsg), 0)
 
	// write cloneFlags
	r.AddData(&Int32msg{
		Type:  CloneFlagsAttr,
		Value: uint32(cloneFlags),
	})
 
	// write custom namespace paths
	if len(nsMaps) > 0 {
		nsPaths, err := c.orderNamespacePaths(nsMaps)
		if err != nil {
			return nil, err
		}
		r.AddData(&Bytemsg{
			Type:  NsPathsAttr,
			Value: []byte(strings.Join(nsPaths, ",")),
		})
	}
 
	// write namespace paths only when we are not joining an existing user ns
	_, joinExistingUser := nsMaps[configs.NEWUSER]
	if !joinExistingUser {
		// write uid mappings
		if len(c.config.UidMappings) > 0 {
			if c.config.RootlessEUID && c.newuidmapPath != "" {
				r.AddData(&Bytemsg{
					Type:  UidmapPathAttr,
					Value: []byte(c.newuidmapPath),
				})
			}
			b, err := encodeIDMapping(c.config.UidMappings)
			if err != nil {
				return nil, err
			}
			r.AddData(&Bytemsg{
				Type:  UidmapAttr,
				Value: b,
			})
		}
 
		// write gid mappings
		if len(c.config.GidMappings) > 0 {
			b, err := encodeIDMapping(c.config.GidMappings)
			if err != nil {
				return nil, err
			}
			r.AddData(&Bytemsg{
				Type:  GidmapAttr,
				Value: b,
			})
			if c.config.RootlessEUID && c.newgidmapPath != "" {
				r.AddData(&Bytemsg{
					Type:  GidmapPathAttr,
					Value: []byte(c.newgidmapPath),
				})
			}
			if requiresRootOrMappingTool(c.config) {
				r.AddData(&Boolmsg{
					Type:  SetgroupAttr,
					Value: true,
				})
			}
		}
	}
 
	if c.config.OomScoreAdj != nil {
		// write oom_score_adj
		r.AddData(&Bytemsg{
			Type:  OomScoreAdjAttr,
			Value: []byte(fmt.Sprintf("%d", *c.config.OomScoreAdj)),
		})
	}
 
	// write rootless
	r.AddData(&Boolmsg{
		Type:  RootlessEUIDAttr,
		Value: c.config.RootlessEUID,
	})
 
	return bytes.NewReader(r.Serialize()), nil
}
```
/var/run/runc/container-bbbb/state.json 中纪录运行容器的状态信息，各个 namespace 对应的路径
```
 "namespace_paths": {
        "NEWIPC": "/proc/3193/ns/ipc",
        "NEWNET": "/proc/3193/ns/net",
        "NEWNS": "/proc/3193/ns/mnt",
        "NEWPID": "/proc/3193/ns/pid",
        "NEWUSER": "/proc/3193/ns/user",
        "NEWUTS": "/proc/3193/ns/uts"
    }

```