---
layout: post
categories: [Python]
description: none
keywords: Python
---
# Python发行版Anaconda

## Python的科学计算发行版——Anaconda
Anaconda是一个用于科学计算的Python发行版，支持Linux、Mac、Windows系统，它提供了包管理与环境管理的功能，可以很方便地解决多版本Python并存、切换以及各种第三方包安装问题。

Anaconda能让你在数据科学的工作中轻松安装经常使用的程序包。你还可以使用它创建虚拟环境，以便更轻松地处理多个项目。Anaconda简化了工作流程，并且解决了多个包和Python版本之间遇到的大量问题。

你可能已经安装了Python，并且想知道为何还需要Anaconda。首先，Anaconda附带了一大批常用的数据科学包，因此你可以立即开始处理数据，而不需要使用Python自带的pip命令下载一大堆的数据科学包。
其次，使用Anaconda的自带命令conda来管理包和环境能减少在处理数据过程中使用到的各种库与版本时遇到的问题。

Anaconda是一个集成的Python运行环境。除了包含Python本身的运行环境外，还集成了很多第三方模块，如本书后面要将的numpy、pandas、flask等模块都集成在了Anaconda中，也就是说，只要安装了Anaconda，这些模块都不需要安装了。

Anaconda的安装相当简单，首先进入Anaconda的下载页面，地址如下：https://www.anaconda.com/download
Anaconda的下载页面也会根据用户当前使用的操作系统自动切换到相应的Anaconda安装包。Anaconda是跨平台的，支持Windows、Mac OS X和Linux。不管是哪个操作系统平台的安装包，下载直接安装即可。
Anaconda的安装包分为Python3.x和Python2.x两个版本。

如果计算机上已经安装了 Python，安装不会对你有任何影响。实际上，脚本和程序使用的默认 Python 是 Anaconda 附带的 Python。
完成安装后，如果你是在windows上操作，打开 Anaconda Prompt （或者 Mac 下的终端）将Anaconda Prompt统一称为“终端”。可以在终端或命令提示符中键入 conda list，以查看你安装的内容。
安装了 Anaconda 之后，就可以很方便的管理包了（安装，卸载，更新）。

## conda
conda与pip类似，只不过conda的可用包专注于数据科学，而pip应用广泛。然而，conda并不像pip那样为Python量身打造，它也可以安装非Python包。它是任何软件堆栈的包管理器。

conda的其中一个功能是包和环境管理器，用于在计算机上安装库和其他软件。conda只能通过命令行来使用。

### 安装包
在终端中键入：
```python
conda install package_name
```
例如，要安装 pandas，在终端中输入：
```python
conda install pandas
```
你还可以同时安装多个包。类似 conda install pandas numpy 的命令会同时安装所有这些包。还可以通过添加版本号（例如 conda install numpy=1.10）来指定所需的包版本。
conda 还会自动为你安装依赖项。例如，scipy 依赖于 numpy，因为它使用并需要 numpy。如果你只安装 scipy (conda install scipy)，则 conda 还会安装 numpy（如果尚未安装的话）。

### 卸载包
在终端中键入 ：
```python
conda remove package_names
```
上面命令中的package_names是指你要卸载包的名称，例如你想卸载pandas包：conda remove pandas

### 更新包
在终端中键入：
```python
conda update package_name
```
如果想更新环境中的所有包（这样做常常很有用），使用：conda update --all。

### 列出已安装的包
列出已安装的包
```python
conda list
```

## Conda创建和使用环境
- 查看 Python 版本：
```shell
python --version
```
- 创建环境:
```shell
conda create -n my_env python=3.7.0
```
新的开发环境会被默认安装在你 conda 目录下的 envs 文件目录下。
- 激活环境
```shell
activate my_env
```
- 列出所有的环境
```shell
conda info -e
```
当前激活的环境会标*。
- 切换到另一个环境
```shell
activate my_env
```
- 注销当前环境
```
   deactivate
```
- 复制环境
```shell
   conda create -n my_env --clone my_env_2
```
- 删除环境
```shell
   conda remove -n my_env2 --all
```

## 第一个Python程序
下面编写第一个Python程序。这个程序定义了两个整数类型的变量n和m，并将两个变量相加，最后调用print函数输出这两个变量的和。
输入下面的Python代码。
```python
n = 20

m = 30

print("n + m =",n + m)
```
运行Python程序