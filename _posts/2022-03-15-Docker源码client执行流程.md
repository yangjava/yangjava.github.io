---
layout: post
categories: [Docker]
description: none
keywords: Docker
---
# Docker源码docker client命令行执行流程 

## docker client的入口main
源码
docker client的main函数位于cli/cmd/docker/docker.go，代码的主要内容是：
```
func main() {
	dockerCli, err := command.NewDockerCli()
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	logrus.SetOutput(dockerCli.Err())

	if err := runDocker(dockerCli); err != nil {
		if sterr, ok := err.(cli.StatusError); ok {
			if sterr.Status != "" {
				fmt.Fprintln(dockerCli.Err(), sterr.Status)
			}
			// StatusError should only be used for errors, and all errors should
			// have a non-zero exit status, so never exit with 0
			if sterr.StatusCode == 0 {
				os.Exit(1)
			}
			os.Exit(sterr.StatusCode)
		}
		fmt.Fprintln(dockerCli.Err(), err)
		os.Exit(1)
	}
}
```
这部分代码的主要工作是：

生成一个带有输入输出的客户端对象
根据dockerCli客户端对象，解析命令行参数，生成带有命令行参数及客户端配置信息的cmd命令行对象
根据输入参数args完成命令执行

我们需要追踪的是docker run的执行流程，从流程图中可以看到，从cmd := newDockerCommand(dockerCli)的一步步进行下去，可以追踪到runContainer(dockerCli,opts,copts,contianerConfig)，而这里就是run命令的执行函数。 在具体了解runContainer()的具体执行之前，先了解一下cobra.Command结构体。

## cobra.Command
首先介绍cobra这个库的简单使用:
```
package main import("fmt""github.com/spf13/cobra") 
func main(){//1.定义主命令varVersionboolvarrootCmd=&cobra.Command{Use:"root [sub]",Short:"My root command",//命令执行的函数Run:func(cmd*cobra.Command,args[]string){fmt.Printf("Inside rootCmd Run with args: %v\n",args)ifVersion{fmt.Printf("Version:1.0\n")}},}//2.定义子命令varsubCmd=&cobra.Command{Use:"sub [no options!]",Short:"My subcommand",//命令执行的函数Run:func(cmd*cobra.Command,args[]string){fmt.Printf("Inside subCmd Run with args: %v\n",args)},}//添加子命令rootCmd.AddCommand(subCmd)//3.为命令添加选项flags:=rootCmd.Flags()flags.BoolVarP(&Version,"version","v",false,"Print version information and quit")//执行命令_=rootCmd.Execute()}
```
基本用法大概就是四步：

定义一个主命令（包含命令执行函数等）
定义若干子命令（包含命令执行函数等，根据需要可以为子命令定义子命令），并添加到主命令
为命令添加选项
执行命令

docker中cobra.Command的使用
docker中Command的使用就体现在commands.AddCommands()。在commands.AddCommands(cmd, dockerCli)中，将run、build等一些命令的执行函数添加到commands中。

## runContainer()
源码
runContainer()的代码位于cli/command/container/run.go代码的主要部分为：
```
// nolint: gocyclo
func runContainer(dockerCli command.Cli, opts *runOptions, copts *containerOptions, containerConfig *containerConfig) error {
	config := containerConfig.Config
	hostConfig := containerConfig.HostConfig
	stdout, stderr := dockerCli.Out(), dockerCli.Err()
	client := dockerCli.Client()

	config.ArgsEscaped = false

	if !opts.detach {
		if err := dockerCli.In().CheckTty(config.AttachStdin, config.Tty); err != nil {
			return err
		}
	} else {
		if copts.attach.Len() != 0 {
			return errors.New("Conflicting options: -a and -d")
		}

		config.AttachStdin = false
		config.AttachStdout = false
		config.AttachStderr = false
		config.StdinOnce = false
	}

	// Telling the Windows daemon the initial size of the tty during start makes
	// a far better user experience rather than relying on subsequent resizes
	// to cause things to catch up.
	if runtime.GOOS == "windows" {
		hostConfig.ConsoleSize[0], hostConfig.ConsoleSize[1] = dockerCli.Out().GetTtySize()
	}

	ctx, cancelFun := context.WithCancel(context.Background())
	defer cancelFun()

	createResponse, err := createContainer(ctx, dockerCli, containerConfig, &opts.createOptions)
	if err != nil {
		reportError(stderr, "run", err.Error(), true)
		return runStartContainerErr(err)
	}
	if opts.sigProxy {
		sigc := ForwardAllSignals(ctx, dockerCli, createResponse.ID)
		defer signal.StopCatch(sigc)
	}

	var (
		waitDisplayID chan struct{}
		errCh         chan error
	)
	if !config.AttachStdout && !config.AttachStderr {
		// Make this asynchronous to allow the client to write to stdin before having to read the ID
		waitDisplayID = make(chan struct{})
		go func() {
			defer close(waitDisplayID)
			fmt.Fprintln(stdout, createResponse.ID)
		}()
	}
	attach := config.AttachStdin || config.AttachStdout || config.AttachStderr
	if attach {
		if opts.detachKeys != "" {
			dockerCli.ConfigFile().DetachKeys = opts.detachKeys
		}

		close, err := attachContainer(ctx, dockerCli, &errCh, config, createResponse.ID)

		if err != nil {
			return err
		}
		defer close()
	}

	statusChan := waitExitOrRemoved(ctx, dockerCli, createResponse.ID, copts.autoRemove)

	//start the container
	if err := client.ContainerStart(ctx, createResponse.ID, types.ContainerStartOptions{}); err != nil {
		// If we have hijackedIOStreamer, we should notify
		// hijackedIOStreamer we are going to exit and wait
		// to avoid the terminal are not restored.
		if attach {
			cancelFun()
			<-errCh
		}

		reportError(stderr, "run", err.Error(), false)
		if copts.autoRemove {
			// wait container to be removed
			<-statusChan
		}
		return runStartContainerErr(err)
	}

	if (config.AttachStdin || config.AttachStdout || config.AttachStderr) && config.Tty && dockerCli.Out().IsTerminal() {
		if err := MonitorTtySize(ctx, dockerCli, createResponse.ID, false); err != nil {
			fmt.Fprintln(stderr, "Error monitoring TTY size:", err)
		}
	}

	if errCh != nil {
		if err := <-errCh; err != nil {
			if _, ok := err.(term.EscapeError); ok {
				// The user entered the detach escape sequence.
				return nil
			}

			logrus.Debugf("Error hijack: %s", err)
			return err
		}
	}

	// Detached mode: wait for the id to be displayed and return.
	if !config.AttachStdout && !config.AttachStderr {
		// Detached mode
		<-waitDisplayID
		return nil
	}

	status := <-statusChan
	if status != 0 {
		return cli.StatusError{StatusCode: status}
	}
	return nil
}
```
在ContainerCreate()和ContainerStart()中分别向daemon发送了create和start命令。下一步，就需要到docker daemon中分析daemon对create和start的处理。









