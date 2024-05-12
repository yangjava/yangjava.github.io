---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码API容器操作

## 获取容器列表
```
func (cli *Client) ContainerList(ctx context.Context, options ContainerListOptions) ([]Container, error)
```
语法示例
```
containers, err := Cli.ContainerList(context.Background(), types.ContainerListOptions{All: true})
```
完整示例
```
package main

import (
	"bufio"
	"context"
	"fmt"
	"github.com/docker/docker/api/types"
	"github.com/docker/docker/client"
	"io"
	"log"
	"os"
)

func main() {
	cli, err := ConnectDocker()
	if err != nil {
		fmt.Println(err)
	} else {
		fmt.Println("docker 链接成功")
	}
	err = GetContainers(cli)
	if err != nil {
		fmt.Println(err)
	}
}

// ConnectDocker
// 链接docker
func ConnectDocker() (cli *client.Client, err error) {
	cli, err = client.NewClientWithOpts(client.WithAPIVersionNegotiation(), client.WithHost("tcp://10.10.239.32:2375"))
	if err != nil {
		fmt.Println(err)
		return nil, err
	}

	return cli, nil
}

// GetContainers
// 获取容器列表
func GetContainers(cli *client.Client) error {
	//All-true相当于docker ps -a
	containers, err := cli.ContainerList(context.Background(), types.ContainerListOptions{All: true})
	if err != nil {
		fmt.Println(err)
		return err
	}

	for _, container := range containers {
		fmt.Printf("%s %s\n", container.ID[:10], container.Image)
	}
	return nil
}

```

## 查看指定容器信息
查看方法
```
func (cli *Client) ContainerInspect(ctx context.Context, containerID string) (ContainerJSON, error)
```
语法示例
```
	ctx := context.Background()
	containerJson, err := cli.ContainerInspect(ctx, containerId)
```
返回结构体types.ContainerJSON
```
type ContainerJSON struct {
    *ContainerJSONBase
    Mounts          []MountPoint
    Config          *Config
    NetworkSettings *NetworkSettings
}

```