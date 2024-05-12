---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码start

## 问题
问题：Docker run 来执行一个容器，那么执行 Docker run 之后到底都做了什么工作呢？
首先，用户通过Docker client输入docker run来创建一个容器。Docker client 主要的工作是通过解析用户所提供的一系列参数后，分别发送了这样两条请求：

docker create 和 docker start，这篇文章分析 docker start 命令

## docker start container 源码分析
客户端 ContainerStart
函数位置 client/container_start.go，post 请求到 docker daemon 进程，路由为 /containers/${container-id}/start
```
func (cli *Client) ContainerStart(ctx context.Context, containerID string, options types.ContainerStartOptions) error {
       query := url.Values{}
       if len(options.CheckpointID) != 0 {
              query.Set("checkpoint", options.CheckpointID)
       }
       if len(options.CheckpointDir) != 0 {
              query.Set("checkpoint-dir", options.CheckpointDir)
       }

       resp, err := cli.post(ctx, "/containers/"+containerID+"/start", query, nil, nil)
       ensureReaderClosed(resp)
       return err
}

```

### daemon 端 ContainerStart
函数位置 daemon/start.go
```
// ContainerStart starts a container.
func (daemon *Daemon) ContainerStart(name string, hostConfig *containertypes.HostConfig, 
       checkpoint string, checkpointDir string) error {
       container, err := daemon.GetContainer(name)

       // check if hostConfig is in line with the current system settings.
       // It may happen cgroups are umounted or the like.
       if _, err = daemon.verifyContainerSettings(container.HostConfig, nil, false); err != nil {

       return daemon.containerStart(container, checkpoint, checkpointDir, true)
}
```

### GetContainer，可以根据全容器 ID，容器名，容器 ID 前缀
```
func (daemon *Daemon) GetContainer(prefixOrName string) (*container.Container, error) {
       if containerByID := daemon.containers.Get(prefixOrName); containerByID != nil {
              // prefix is an exact match to a full container ID
              return containerByID, nil
       }

       // GetByName will match only an exact name provided; we ignore errors
       if containerByName, _ := daemon.GetByName(prefixOrName); containerByName != nil {
              // prefix is an exact match to a full container Name
              return containerByName, nil
       }

       containerID, indexError := daemon.idIndex.Get(prefixOrName)
      
       return daemon.containers.Get(containerID), nil
}
```

```
func (daemon *Daemon) containerStart(container *container.Container, checkpoint string,
       checkpointDir string, resetRestartManager bool) (err error) {
       if err := daemon.conditionalMountOnStart(container); err != nil {

       if err := daemon.initializeNetworking(container); err != nil {
 
       spec, err := daemon.createSpec(container)

       createOptions, err := daemon.getLibcontainerdCreateOptions(container)

       if resetRestartManager {
              container.ResetRestartManager(true)
       }

       if checkpointDir == "" {
              checkpointDir = container.CheckpointDir()
       }

       if daemon.saveApparmorConfig(container); err != nil {
              return err
       }

       if err := daemon.containerd.Create(container.ID, checkpoint, checkpointDir, *spec, container.InitializeStdio, createOptions...); err != nil {

       return nil
}
```
conditionalMountOnStart 在创建的时候调用了一次，这次 start 又 mount 一次？有点蒙等了。
```
func (daemon *Daemon) conditionalMountOnStart(container *container.Container) error {
       return daemon.Mount(container)
}
```
initalizeNetworking 对网络进行初始化，docker 网络有三种，bridge模式（每个容器用户单独的网络栈），host模式（与宿主机共用一个网络栈），contaier 模式（与其他容器共用一个网络栈，猜测kubernate中的pod所用的模式）；根据 config 和 hostConfig 中的参数来确定容器的网络模式，然后调 libnetwork 包来建立网络，默认使用的是 bridge 模式
```
func (daemon *Daemon) initializeNetworking(container *container.Container) error {
       if container.HostConfig.NetworkMode.IsContainer() {
              // we need to get the hosts files from the container to join
              nc, err := daemon.getNetworkedContainer(container.ID, container.HostConfig.NetworkMode.ConnectedContainer())

              err = daemon.initializeNetworkingPaths(container, nc)
            
              container.Config.Hostname = nc.Config.Hostname
              container.Config.Domainname = nc.Config.Domainname
              return nil
       }

       if container.HostConfig.NetworkMode.IsHost() {
              if container.Config.Hostname == "" {
                     container.Config.Hostname, err = os.Hostname()
              }
       }

       if err := daemon.allocateNetwork(container); err != nil {

       return container.BuildHostnameFile()
}

```
allocateNetwork 函数，主要函数为 connectToNetwork
```
func (daemon *Daemon) allocateNetwork(container *container.Container) error {
       defaultNetName := runconfig.DefaultDaemonNetworkMode().NetworkName()
       if nConf, ok := container.NetworkSettings.Networks[defaultNetName]; ok {
              cleanOperationalData(nConf)
              if err := daemon.connectToNetwork(container, defaultNetName, nConf.EndpointSettings, updateSettings); err != nil {
       }

       // the intermediate map is necessary because "connectToNetwork" modifies "container.NetworkSettings.Networks"
       networks := make(map[string]*network.EndpointSettings)
       for n, epConf := range container.NetworkSettings.Networks {
              if n == defaultNetName {
                     continue
              }

              networks[n] = epConf
       }

       for netName, epConf := range networks {
              cleanOperationalData(epConf)
              if err := daemon.connectToNetwork(container, netName, epConf.EndpointSettings, updateSettings); err != nil {
       }

       if _, err := container.WriteHostConfig(); err != nil {
 
       return nil
}

```
Create 函数位于文件 libcontainerd/client_unix.go 中，创建目录，以及文件 config.json，调用 start 函数在 
```
func (clnt *client) Create(containerID string, checkpoint string, checkpointDir string, spec specs.Spec, attachStdio StdioCallback, options ...CreateOption) (err error) {
       container := clnt.newContainer(filepath.Join(dir, containerID), options...)
       if err := container.clean(); err != nil {

       if err := idtools.MkdirAllAndChown(container.dir, 0700, idtools.IDPair{uid, gid}); err != nil && !os.IsExist(err) {

       f, err := os.Create(filepath.Join(container.dir, configFilename))

       if err := json.NewEncoder(f).Encode(spec); err != nil {
    
       return container.start(&spec, checkpoint, checkpointDir, attachStdio)
}

```
start 函数位于 libcontainerd/container_unix.go 中，主要是发送 CreateContainerRequest RPC 请求给 docker-containerd 进程
```
func (ctr *container) start(spec *specs.Spec, checkpoint, checkpointDir string, attachStdio StdioCallback) (err error) {
       stdin := iopipe.Stdin
       iopipe.Stdin = ioutils.NewWriteCloserWrapper(stdin, func() error {
              var err error
              stdinOnce.Do(func() { // on error from attach we don't know if stdin was already closed
                     err = stdin.Close()
                     go func() {
                            select {
                            case <-ready:
                            case <-ctx.Done():
                            }
                            select {
                            case <-ready:
                                   if err := ctr.sendCloseStdin(); err != nil {
                                          logrus.Warnf("failed to close stdin: %+v", err)
                                   }
                            default:
                            }
                     }()
              })
              return err
       })

       r := &containerd.CreateContainerRequest{
              Id:            ctr.containerID,
              BundlePath:    ctr.dir,
              Stdin:         ctr.fifo(unix.Stdin),
              Stdout:        ctr.fifo(unix.Stdout),
              Stderr:        ctr.fifo(unix.Stderr),
              Checkpoint:    checkpoint,
              CheckpointDir: checkpointDir,
              // check to see if we are running in ramdisk to disable pivot root
              NoPivotRoot: os.Getenv("DOCKER_RAMDISK") != "",
              Runtime:     ctr.runtime,
              RuntimeArgs: ctr.runtimeArgs,
       }
       ctr.client.appendContainer(ctr)

       if err := attachStdio(*iopipe); err != nil {

       resp, err := ctr.client.remote.apiClient.CreateContainer(context.Background(), r)
  
       ctr.systemPid = systemPid(resp.Container)
       close(ready)

       return ctr.client.backend.StateChanged(ctr.containerID, StateInfo{
              CommonStateInfo: CommonStateInfo{
                     State: StateStart,
                     Pid:   ctr.systemPid,
              }})

}
```