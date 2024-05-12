---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码daemon端对container create的处理

## r.postContainersCreate()
r.postContainersCreate()的实现位于moby/api/server/router/container/container_routes.go，代码的主要内容是：
```
func (s *containerRouter) postContainersCreate(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
	//从http的form表单中获取名字，应该就是"/containers/create"吧？
	name := r.Form.Get("name")
	//获取从client传过来的Config、hostConfig和networkingConfig配置信息
	config, hostConfig, networkingConfig, err := s.decoder.DecodeConfig(r.Body)
	if err != nil {
		return err
	}
	//传入配置信息，调用ContainerCreate进一步创建容器
	ccr, err := s.backend.ContainerCreate(types.ContainerCreateConfig{
		Name:             name,
		Config:           config,
		HostConfig:       hostConfig,
		NetworkingConfig: networkingConfig,
		AdjustCPUShares:  adjustCPUShares,
	})
	
	//给client返回结果
	return httputils.WriteJSON(w, http.StatusCreated, ccr)
}
```
其中出现的Config主要目的是基于容器的可移植性信息，与host相互独立，Config 包括容器的基本信息，名字，输入输出流等；非可移植性在 HostConfig 结构体中。

对于s.backend.ContainerCreate()，进一步查看代码可以看到，它会调用moby/api/server/router/container/backend.go,然后再调用到daemon.containerCreate(params, false)。
```
type stateBackend interface {
	ContainerCreate(config types.ContainerCreateConfig) (container.ContainerCreateCreatedBody, error)
	...
}


func (daemon *Daemon) ContainerCreate(params types.ContainerCreateConfig) (containertypes.ContainerCreateCreatedBody, error) {
	return daemon.containerCreate(params, false)
}
```
下面详细分析daemon.containerCreate(params, false)。

## daemon.containerCreate()
daemon.containerCreate(params, false)的实现位于moby/daemon/create.go，主要代码为：
```
func (daemon *Daemon) containerCreate(params types.ContainerCreateConfig, managed bool) (containertypes.ContainerCreateCreatedBody, error) {
	//验证HostConfig、Config的信息正确性。
	warnings, err := daemon.verifyContainerSettings(params.HostConfig, params.Config, false)
	//验证NetworkingConfig的正确性，如client的配置中是否为一个容器创建时配置了超过1个network，查看IPAMConfig是否有效
	err = daemon.verifyNetworkingConfig(params.NetworkingConfig)
	//如果配置hostConfig为空，则使用默认值
	if params.HostConfig == nil {
		params.HostConfig = &containertypes.HostConfig{}
	}
	//修改hostconfig的不正常值，例如CPUShares、Memory
	err = daemon.adaptContainerSettings(params.HostConfig, params.AdjustCPUShares)
	if err != nil {
		return containertypes.ContainerCreateCreatedBody{Warnings: warnings}, err
	}
	//进一步调用daemon的create(),这里managed为false，还不了解是这个值的作用，用到时再分析
	container, err := daemon.create(params, managed)
	//返回运行结果
	return containertypes.ContainerCreateCreatedBody{ID: container.ID, Warnings: warnings}, nil
}
```

## daemon.create()
daemon.create(params, managed)的实现位于moby/daemon/create.go，主要代码为：
```
func (daemon *Daemon) create(params types.ContainerCreateConfig, managed bool) (retC *container.Container, retErr error) {
	//定义一些全局变量
	var (
		container *container.Container
		img       *image.Image
		imgID     image.ID
		err       error
	)
	//查找Image
	if params.Config.Image != "" {
		img, err = daemon.GetImage(params.Config.Image)
		imgID = img.ID()
	}
	//将用户指定的Config参数与镜像json文件中的config合并并验证
	err := daemon.mergeAndVerifyConfig(params.Config, img)
	//如果没有指定container log driver，将daemon log config与container log config合并
	err := daemon.mergeAndVerifyLogConfig(&params.HostConfig.LogConfig)
	//进一步调用daemon包下的newContainer函数，下面再详细分析
	container, err = daemon.newContainer(params.Name, params.Config, params.HostConfig, imgID, managed)
	//挂载了labels后，为容器设置读写层
	err := daemon.setRWLayer(container)
	//retrieves the remapped root uid/gid pair from the set of maps
	//If the maps are empty, then the root uid/gid will default to "real" 0/0
	rootUID, rootGID, err := idtools.GetRootUIDGID(daemon.uidMaps, daemon.gidMaps)
	//以 root uid gid的属性创建目录，在/var/lib/docker/containers目录下创建容器文件，并在容器文件下创建checkpoints目录
	err := idtools.MkdirAs(container.Root, 0700, rootUID, rootGID)
	err := idtools.MkdirAs(container.CheckpointDir(), 0700, rootUID, rootGID)

	/*
	1.daemon.registerMountPoints(),注册所有挂载到容器的数据卷
	2.daemon.registerLinks(),记录父子以及别名之间的关系，将 hostconfig 写入文件 hostconfig.json		
	*/
	err := daemon.setHostConfig(container, params.HostConfig)
	//daemon.Mount()函数在 /var/lib/docker/aufs/mnt 目录下创建文件，以及设置工作目录
	daemon.createContainerPlatformSpecificSettings(container, params.Config, params.HostConfig)
	//如果没有设置网络，将网络模式设置为 default
	runconfig.SetDefaultNetModeIfBlank(container.HostConfig)
	//更新网络设置
	daemon.updateContainerNetworkSettings(container, endpointsConfigs)
	//将container对象json化后写入本地磁盘进行持久化
	container.ToDisk()
	//在Daemon中注册新建的container对象
	daemon.Register(container)
	//生成一个只有默认属性容器相关事件
	daemon.LogContainerEvent(container, "create")
	//将container对象返回
	return container, nil
}
```

## daemon.newContainer()
daemon.newContainer()主要生成一个container结构体，这里包括id和name的确定。它的主要代码为：
```
func (daemon *Daemon) newContainer(name string, config *containertypes.Config, hostConfig *containertypes.HostConfig, imgID image.ID, managed bool) (*container.Container, error) {
	var (
		id             string
		err            error
		noExplicitName = name == ""
	)
	//为容器生成name和id
	id, name, err = daemon.generateIDAndName(name)
	if hostConfig.NetworkMode.IsHost() {
		if config.Hostname == "" {
			如果network是host mode，而且配置中没有指定hostname,则使用宿主机的hostname
			config.Hostname, err = os.Hostname()
			if err != nil {
				return nil, err
			}
		}
	} else {
		//否则，根据id和config生成一个hostname
		daemon.generateHostname(id, config)
	}
	//获得entrypoint和args
	entrypoint, args := daemon.getEntrypointAndArgs(config.Entrypoint, config.Cmd)
	//生成一个最基础的container对象
	base := daemon.newBaseContainer(id)
	//对container一些属性进行初始化，包括网络方面的
	base.Created = time.Now().UTC()
	base.Managed = managed   //在daemon.containerCreate(params, false)中传入的，managed==false
	base.Path = entrypoint
	base.Args = args //FIXME: de-duplicate from config
	base.Config = config
	base.HostConfig = &containertypes.HostConfig{}
	base.ImageID = imgID
	base.NetworkSettings = &network.Settings{IsAnonymousEndpoint: noExplicitName}
	base.Name = name
	base.Driver = daemon.GraphDriverName()
	//将生成的container对象返回
	return base, err
}
```
到这里就已经完成了container对象的初始化，然后根据postContainersCreate()中的return httputils.WriteJSON(w, http.StatusCreated, ccr),将结果返回给client。根据docker源码阅读之一 2.2中，下一步client调用ContainerStart()，并向daemon发送”/containers/”+containerID+”/start”,最后由daemon调用r.postContainersStart()。

本文分析了postContainersCreate()，这里主要是生成一个container对象，根据config、hostConfig、networkingConfig进行初始化，再交给下一步启动使用。
