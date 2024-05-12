---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码cgroup分析

## Manager 接口
libcontainer/cgroups/cgroups.go 中接口 Manager 定义了操作 cgroup 的方法
```
type Manager interface {
	// Applies cgroup configuration to the process with the specified pid
	Apply(pid int) error
 
	// Returns the PIDs inside the cgroup set
	GetPids() ([]int, error)
 
	// Returns the PIDs inside the cgroup set & all sub-cgroups
	GetAllPids() ([]int, error)
 
	// Returns statistics for the cgroup set
	GetStats() (*Stats, error)
 
	// Toggles the freezer cgroup according with specified state
	Freeze(state configs.FreezerState) error
 
	// Destroys the cgroup set
	Destroy() error
 
	// The option func SystemdCgroups() and Cgroupfs() require following attributes:
	// 	Paths   map[string]string
	// 	Cgroups *configs.Cgroup
	// Paths maps cgroup subsystem to path at which it is mounted.
	// Cgroups specifies specific cgroup settings for the various subsystems
 
	// Returns cgroup paths to save in a state file and to be able to
	// restore the object later.
	GetPaths() map[string]string
 
	// Sets the cgroup as configured.
	Set(container *configs.Config) error
}
```
## cgroup raw 子系统
libcontainer/cgroups 中有三个文件夹 fs，rootless，systemd，主要分析 fs 目录下的子系统，其中 libcontainer/cgroups/fs/apply_raw.go 中定义了子系统，包括 cpu，cpuset，memory，pid 等等
```
var (
	subsystems = subsystemSet{
		&CpusetGroup{},
		&DevicesGroup{},
		&MemoryGroup{},
		&CpuGroup{},
		&CpuacctGroup{},
		&PidsGroup{},
		&BlkioGroup{},
		&HugetlbGroup{},
		&NetClsGroup{},
		&NetPrioGroup{},
		&PerfEventGroup{},
		&FreezerGroup{},
		&NameGroup{GroupName: "name=systemd", Join: true},
	}
	HugePageSizes, _ = cgroups.GetHugePageSize()
)
```
## raw 子系统接口
libcontainer/cgroups/fs/apply_raw.go 中定义接口 subsystem
```
type subsystem interface {
	// Name returns the name of the subsystem.
	Name() string
	// Returns the stats, as 'stats', corresponding to the cgroup under 'path'.
	GetStats(path string, stats *cgroups.Stats) error
	// Removes the cgroup represented by 'cgroupData'.
	Remove(*cgroupData) error
	// Creates and joins the cgroup represented by 'cgroupData'.
	Apply(*cgroupData) error
	// Set the cgroup represented by cgroup.
	Set(path string, cgroup *configs.Cgroup) error
}
```
raw 子系统数据结构
libcontainer/cgroups/fs/apply_raw.go 中结构体 Manager 实现了第一章节 Manager 接口，其中 configs.Cgroup 定义在 libcontainer/configs/cgroup_linux.go 中
```
type Manager struct {
	mu       sync.Mutex
	Cgroups  *configs.Cgroup
	Rootless bool // ignore permission-related errors
	Paths    map[string]string
}
```
Cgroup 定义在 libcontainer/configs/cgroup_linux.go 中
```
 
type Cgroup struct {
	// Deprecated, use Path instead
	Name string `json:"name,omitempty"`
 
	// name of parent of cgroup or slice
	// Deprecated, use Path instead
	Parent string `json:"parent,omitempty"`
 
	// Path specifies the path to cgroups that are created and/or joined by the container.
	// The path is assumed to be relative to the host system cgroup mountpoint.
	Path string `json:"path"`
 
	// ScopePrefix describes prefix for the scope name
	ScopePrefix string `json:"scope_prefix"`
 
	// Paths represent the absolute cgroups paths to join.
	// This takes precedence over Path.
	Paths map[string]string
 
	// Resources contains various cgroups settings to apply
	*Resources
}
```

```
type cgroupData struct {
	root      string
	innerPath string
	config    *configs.Cgroup
	pid       int
}
```

## raw 子系统方法
cgroupRoot 定义了 cgroup 的全局根路径，getCgroupRoot 返回根路径，如果全局变量存在则返回，不存在则调用 FindCgroupMountpointDir 在 /proc/self/mountinfo 查找并验证路径是否存在
```
var cgroupRoot string
 
// Gets the cgroupRoot.
func getCgroupRoot() (string, error) {
	cgroupRootLock.Lock()
	defer cgroupRootLock.Unlock()
 
	if cgroupRoot != "" {
		return cgroupRoot, nil
	}
 
	root, err := cgroups.FindCgroupMountpointDir()
	if err != nil {
		return "", err
	}
 
	if _, err := os.Stat(root); err != nil {
		return "", err
	}
 
	cgroupRoot = root
	return cgroupRoot, nil
}
```

## Apply函數
libcontainer/cgroups/fs/apply_raw.go 中方法 Apply

getCgroupData: 获得 cgroup 根路径封装成 cgroupData 结构体
```
func (m *Manager) Apply(pid int) (err error) {
	if m.Cgroups == nil {
		return nil
	}
	m.mu.Lock()
	defer m.mu.Unlock()
 
	var c = m.Cgroups
 
	d, err := getCgroupData(m.Cgroups, pid)
	if err != nil {
		return err
	}
```
m.Paths: 将 cgroup name 和 path 加入到 map 中
```
	m.Paths = make(map[string]string)
	if c.Paths != nil {
		for name, path := range c.Paths {
			_, err := d.path(name)
			if err != nil {
				if cgroups.IsNotFound(err) {
					continue
				}
				return err
			}
			m.Paths[name] = path
		}
		return cgroups.EnterPid(m.Paths, pid)
	}
```
对各个子系统执行 Apply 方法，各个子系统稍后分析
```
	for _, sys := range subsystems {
		// TODO: Apply should, ideally, be reentrant or be broken up into a separate
		// create and join phase so that the cgroup hierarchy for a container can be
		// created then join consists of writing the process pids to cgroup.procs
		p, err := d.path(sys.Name())
		if err != nil {
			// The non-presence of the devices subsystem is
			// considered fatal for security reasons.
			if cgroups.IsNotFound(err) && sys.Name() != "devices" {
				continue
			}
			return err
		}
		m.Paths[sys.Name()] = p
 
		if err := sys.Apply(d); err != nil {
			// In the case of rootless (including euid=0 in userns), where an explicit cgroup path hasn't
			// been set, we don't bail on error in case of permission problems.
			// Cases where limits have been set (and we couldn't create our own
			// cgroup) are handled by Set.
			if isIgnorableError(m.Rootless, err) && m.Cgroups.Path == "" {
				delete(m.Paths, sys.Name())
				continue
			}
			return err
		}
 
	}
```
blkio -- 这个子系统为块设备设定输入/输出限制，比如物理设备（磁盘，固态硬盘，USB 等）。
cpu -- 这个子系统使用调度程序提供对 CPU 的 cgroup 任务访问。
cpuacct -- 这个子系统自动生成 cgroup 中任务所使用的 CPU 报告。
cpuset -- 这个子系统为 cgroup 中的任务分配独立 CPU（在多核系统）和内存节点。
devices -- 这个子系统可允许或者拒绝 cgroup 中的任务访问设备。
freezer -- 这个子系统挂起或者恢复 cgroup 中的任务。
memory -- 这个子系统设定 cgroup 中任务使用的内存限制，并自动生成由那些任务使用的内存资源报告。

cgroup cpu 子系统
libcontainer/cgroups/fs/cpu.go 文件中 cpu 子系统数据机构
```

type CpuGroup struct {
}
```
Apply - cpu
libcontainer/cgroups/fs/cpu.go 中 Apply 获得 cpu 的路径并创建路径，将数据以及 pid 写入到相应的文件中
```
func (s *CpuGroup) Apply(d *cgroupData) error {
	// We always want to join the cpu group, to allow fair cpu scheduling
	// on a container basis
	path, err := d.path("cpu")
	if err != nil && !cgroups.IsNotFound(err) {
		return err
	}
	return s.ApplyDir(path, d.config, d.pid)
}
```
ApplyDir將pid吸入到cpu文件中
```
func (s *CpuGroup) ApplyDir(path string, cgroup *configs.Cgroup, pid int) error {
	// This might happen if we have no cpu cgroup mounted.
	// Just do nothing and don't fail.
	if path == "" {
		return nil
	}
	if err := os.MkdirAll(path, 0755); err != nil {
		return err
	}
	// We should set the real-Time group scheduling settings before moving
	// in the process because if the process is already in SCHED_RR mode
	// and no RT bandwidth is set, adding it will fail.
	if err := s.SetRtSched(path, cgroup); err != nil {
		return err
	}
	// because we are not using d.join we need to place the pid into the procs file
	// unlike the other subsystems
	return cgroups.WriteCgroupProc(path, pid)
}
```

## GetStats - cpu
GetStates 主要是读取 cpu.stat 文件，获得 nr_periods，nr_throtted，throttled_time 三项数据

nr_periods: 总共经过的周期
nr_throtted: 受限制的周期
throttled_time: 就是总共被控制组掐掉的 cpu 使用时间
```
func (s *CpuGroup) GetStats(path string, stats *cgroups.Stats) error {
	f, err := os.Open(filepath.Join(path, "cpu.stat"))
	if err != nil {
		if os.IsNotExist(err) {
			return nil
		}
		return err
	}
	defer f.Close()
 
	sc := bufio.NewScanner(f)
	for sc.Scan() {
		t, v, err := getCgroupParamKeyValue(sc.Text())
		if err != nil {
			return err
		}
		switch t {
		case "nr_periods":
			stats.CpuStats.ThrottlingData.Periods = v
 
		case "nr_throttled":
			stats.CpuStats.ThrottlingData.ThrottledPeriods = v
 
		case "throttled_time":
			stats.CpuStats.ThrottlingData.ThrottledTime = v
		}
	}
	return nil
}
```

## cgroup memory 子系统
cgroup.clone_children： subsystem会读取这个配置文件，如果这个的值是1(默认是0)，子cgroup才会继承父cgroup的配置
cgroup.event_control： 监视状态变化和分组删除事件的配置文件
cgroup.procs： 属于分组的进程 PID 列表。仅包括多线程进程的线程 leader 的 TID，这点与 tasks 不同
memory.failcnt:  申请内存失败次数计数
memory.usage_in_bytes: 当前已用的内存
memory.limit_in_bytes: 当前限制的内存额度
memory.max_usage_in_bytes: 历史内存最大使用量
memory.memsw.limit_in_bytes: 设置内存加上交换分区的总量，通过设置这个值，可以防止进程将交换分区用完
memory.swappiness: 该文件的值默认和全局的swappiness（/proc/sys/vm/swappiness）一样，修改该文件只对当前cgroup生效，其功能和全局的swappiness一样 (有一点和全局的swappiness不同，那就是如果这个文件被设置成0，就算系统配置的有交换空间，当前cgroup也不会使用交换空间。)

## 调用函数
libcontainer/process_linux.go 中 start 函数在容器启功进程设置 cgroup 各个子系统，创建文件路径并将配置与 pid 写入相应的文件
```
func (p *initProcess) start() error {
       ......
       // Do this before syncing with child so that no children can escape the
       // cgroup. We don't need to worry about not doing this and not being root
       // because we'd be using the rootless cgroup manager in that case.
       if err := p.manager.Apply(p.pid()); err != nil {
              return newSystemErrorWithCause(err, "applying cgroup configuration for process")
       }
       ......
       return nil

```
/var/run/runc/container-bbbb/state.json 中纪录运行容器的状态信息，其中 cgroup 子系统对应的路径
```
 "cgroup_paths": {
        "blkio": "/sys/fs/cgroup/blkio/user.slice/container-bbbb",
        "cpu": "/sys/fs/cgroup/cpu,cpuacct/user.slice/container-bbbb",
        "cpuacct": "/sys/fs/cgroup/cpu,cpuacct/user.slice/container-bbbb",
        "cpuset": "/sys/fs/cgroup/cpuset/container-bbbb",
        "devices": "/sys/fs/cgroup/devices/user.slice/container-bbbb",
        "freezer": "/sys/fs/cgroup/freezer/container-bbbb",
        "hugetlb": "/sys/fs/cgroup/hugetlb/container-bbbb",
        "memory": "/sys/fs/cgroup/memory/user.slice/container-bbbb",
        "name=systemd": "/sys/fs/cgroup/systemd/user.slice/user-1000.slice/session-c2.scope/container-bbbb",
        "net_cls": "/sys/fs/cgroup/net_cls,net_prio/container-bbbb",
        "net_prio": "/sys/fs/cgroup/net_cls,net_prio/container-bbbb",
        "perf_event": "/sys/fs/cgroup/perf_event/container-bbbb",
        "pids": "/sys/fs/cgroup/pids/user.slice/user-1000.slice/container-bbbb"
    }

```