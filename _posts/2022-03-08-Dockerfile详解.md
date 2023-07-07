---
layout: post
categories: Docker
description: none
keywords: Docker
---
# Dockerfile详解
如果需要给Dockerfile下个定义，那么Dockerfile首先是一个文本文件，然后Dockerfile是一个Docker可以读懂的脚本文件，在这个脚本文件中记录着用户“创建”镜像过程当中需要执行的所有命令。

## Dockerfile有什么用
当Docker读取并执行Dockerfile中所定义的命令时，这些命令将会产生一些临时文件层。还记得我们提到过Docker使用的是AUFS文件系统吗？AUFS文件系统会将不同的文件层进行“叠加”从而形成一个完整的文件系统。这里在Dockerfile中执行命令后形成的临时文件层就是AUFS所使用的文件层。当Dockerfile所有命令都成功执行完之后，Docker会记录执行过程中所用到的所有文件层，并且会用一个名称来标记这一组文件层。

这一组文件层，就被称之为镜像。因此我们可以得出结论：镜像指的是一组特定的文件层。Docker中镜像的构建过程，就是Docker执行Dockerfile中所定义命令而形成这组文件层的过程。

Docker中所有的容器都是基于镜像而创建的，而所有的镜像又都是通过Dockerfile而形成的。通过这样的逻辑，我们就能够明白Docker整套虚拟生态圈的起点就是Dockerfile。

## Dockerfile的语法
Dockerfile的语法非常简单，所以无论是编写，还是修改Dockerfile，都是一件非常容易的事情。下面是Dockerfile典型的语法示例：
```
# 在Dockerfile语法中，#表示单行注释。
command argument argument ..

#Dockerfile语法，就是命令+参数+参数……
#例如，我们想要在镜像构建过程中输出一个hello world
RUN echo "Hello Docker!"

#RUN是Dockerfile内置命令，表示在镜像中执行后面的命令，详细用法会在后面介绍
```
在Dockerfile中，可以使用的内置命令及其作用
- FROM : 基础镜像，当前新镜像是基于哪个镜像的
- MAINTAINER : 镜像维护者的姓名和邮箱地址
- RUN : 容器构建时需要运行的命令
- EXPOSE : 当前容器对外暴露出的端口
- WORKDIR : 指定在创建容器后，终端默认登陆的进来工作目录，一个落脚点
- ENV : 用来在构建镜像过程中设置环境变量
ENV MY_PATH /usr/mytest 这个环境变量可以在后续的任何RUN指令中使用，这就如同在命令前面指定了环境变量前缀一样； 也可以在其它指令中直接使用这些环境变量，
比如：WORKDIR $MY_PATH
- UER : 为RUN CMD ENTRYPOINT 执行命令指定运行用户
- ADD : 将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包
- COPY : 类似ADD，拷贝文件和目录到镜像中。 将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置
- VOLUME : 容器数据卷，用于数据保存和持久化工作
- CMD : 指定一个容器启动时要运行的命令
注意: Dockerfile 中可以有多个 CMD 指令，但只有最后一个生效，CMD 会被 docker run 之后的参数替换
- ENTRYPOINT : 指定一个容器启动时要运行的命令;ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数,但是不会被 docker run 后面的参数替换,而是追加
- ONBUILD : 当构建一个被继承的Dockerfile时运行命令，父镜像在被子继承后父镜像的onbuild被触发

下面是Dockerfile典型示例：
```
############################################################
# Dockerfile to build MongoDB container images
# Based on Ubuntu
############################################################

# Set the base image to Ubuntu
FROM ubuntu

# File Author / Maintainer
MAINTAINER Example McAuthor

# Update the repository sources list
RUN apt-get update

################## BEGIN INSTALLATION ######################
# Install MongoDB Following the Instructions at MongoDB Docs
# Ref: http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/

# Add the package verification key
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10

# Add MongoDB to the repository sources list
RUN  echo  'deb  http://downloads-distro.mongodb.org/repo/ubuntu-upstart  dist  10gen'  |  tee
/etc/apt/sources.list.d/mongodb.list

# Update the repository sources list once more
RUN apt-get update

# Install MongoDB package (.deb)
RUN apt-get install -y mongodb-10gen

# Create the default data directory
RUN mkdir -p /data/db

##################### INSTALLATION END #####################

# Expose the default port
EXPOSE 27017

# Default port to execute the entrypoint (MongoDB)
CMD ["--port 27017"]

# Set default container command
ENTRYPOINT usr/bin/mongod
```
当我们编写好Dockerfile之后，就可以通过Docker CLI中的build命令来执行Dockerfile了。
```
#build命令，就是用来执行Dockerfile的，下面是其用法
# Example: sudo Docker build -t [name] .

Docker build -t my_ mongodb .
```
在等待Dockerfile中定义的所有命令都执行完毕之后，一个新的镜像my_mongodb就产生了。然后就可以通过Docker run或者Docker-compose工具来创建MongoDB的实例，从而对外提供服务了。

## 如何编写Dockerfile
我们将介绍这些内置命令如何使用。

### FROM命令
```
FROM <image>
FROM <image>:<tag>
FROM <image>@<digest>
```
FROM是Dockerfile内置命令中唯一一个必填项，其有上述三种用法。FROM用来指定后续指令执行的基础镜像，所以在一个有效的Dockerfile中，FROM永远是第一个命令（注释除外）。

FROM指定的基础镜像既可以是本地已经存在的镜像，也可以是远程仓库中的镜像。当Dockerfile执行时，如果本地没有其指定的基础镜像，那么就会从远程仓库中下载此镜像。

在Dockerfile的语法中，并没有规定只能出现一次FROM，因而如果需要在一个Dockerfile中同时构建多个镜像时，也可以出现多次FROM，例如：
```
# Nginx
#
# VERSION               0.0.1

FROM      ubuntu
MAINTAINER Victor Vieux <victor@Docker.com>

LABEL Description="This image is used to start the foobar executable" Vendor="ACME Products"
Version="1.0"
RUN apt-get update && apt-get install -y inotify-tools nginx apache2 openssh-server

# Firefox over VNC
#
# VERSION               0.3

FROM ubuntu
# Install vnc, xvfb in order to create a 'fake' display and firefox
RUN apt-get update && apt-get install -y x11vnc xvfb firefox
RUN mkdir ～/.vnc
# Setup a password
RUN x11vnc -storepasswd 1234 ～/.vnc/passwd
# Autostart firefox (might not be the best way, but it does the trick)
RUN bash -c 'echo "firefox" >> /.bashrc'

EXPOSE 5900
CMD    ["x11vnc", "-forever", "-usepw", "-create"]

# Multiple images example
#
# VERSION               0.1

FROM ubuntu
RUN echo foo > bar
# Will output something like ===> 907ad6c2736f

FROM ubuntu
RUN echo moo > oink
# Will output something like ===> 695d7793cbe4

# You᾿ll now have two images, 907ad6c2736f with /bar, and 695d7793cbe4 with
# /oink.
```
在上面的Dockerfile中一共出现了三次FROM命令，这就意味着当这个Dockerfile执行完毕之后，会同时生成三个镜像，但只会输出最后一个镜像的ID值，中间两个镜像只会被标记为<none>:<none>。所以是否需要在一个Dockerfile中同时生成多个镜像，需要用户根据情况自行决定。

在FROM用法中，tag和digest属于可选项，如果没有则默认取指定镜像的最新版，也就是latest版本。如果找不到latest版本，那么整个Dockerfile就会报错返回。

Docker 还存在一个特殊的镜像，名为 scratch 。这个镜像并不实际存在，它表示一个空白的镜像。如果以 scratch 为基础镜像的话，意味着不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。

## MAINTAINER命令
```
MAINTAINER authors_name
```
MAINTAINER是用来维护镜像作者的命令。一般情况下是放在FROM命令后面，当然也可以放在其他位置。

这里除了输入作者信息之外，还可以输入其他信息，同样不会报错。但入乡随俗，这里还是建议只输入作者信息，其他备注类的信息可以放到LABELS字段中。

## RUN命令
```
RUN <command>
RUN ["executable", "param1", "param2"]
```
RUN命令是用来在镜像中执行命令的命令，是一个完整Dockerfile中使用频率最高的命令，其有上面两种用法，最基本的用法就是RUN<command>。

当使用RUN<command>用法时，后面的命令其实是由/bin/sh–C来负责执行的。所以RUN的这种用法就会存在一个限制，那就是在镜像中必须要有/bin/sh。

如果基础镜像中没有/bin/sh时，该怎么办呢？

如果基础镜像没有/bin/sh，那么就需要使用RUN的另外一个用法了。`RUN ["executable","param1","param2"]`，可以执行基础镜像中任意一个二进制程序。比如我们需要使用bash来执行程序，就可以这样编写RUN指令：
```
RUN ["/bin/bash", "-c", "echo hello"]
```
在使用这种用法时，[]中的数据都会被按照JSON字符串的格式进行解析，因此只能使用双引号“”，不能使用单引号或者其他符号，这点请读者注意。

同时使用这种用法时，还需要注意到环境变量的使用问题：
```
RUN [ "echo", "$HOME" ]
```
基础镜像中即便存在$HOME变量，上述示例仍然会失败。因为此时RUN执行时，不会加载环境变量中的数据。如果需要使用环境变量，那么只能通过下面的方式：
```
RUN [ "sh", "-c", "echo", "$HOME" ]
```
当RUN命令执行完毕之后，就会产生一个新的文件层。这个新产生的文件层会被保存在缓存中，并且将作为下个指令的基础镜像而存在，例如：
```
RUN apt-get dist-upgrade -y
```
这条命令产生的数据将会被后续所有指令复用。如果不需要在缓存中保存这些数据，那么需要使用--no-cache来禁用缓存保存功能，例如：
```
Docker build --no-cache XXXXX
```
Docker目前版本中，存在一个已知的问题：所有镜像最多只能保存126个文件层。而执行一次RUN就会产生一个文件层，而且新产生的镜像会包括基础镜像的文件层。

所以使用RUN指令时，应尽量将多个命令放到一个RUN中去执行

### CMD命令
```
CMD ["executable","param1","param2"]
CMD ["param1","param2"]
CMD command param1 param2
```
CMD是用来设定镜像默认执行命令的命令。第一种用法是推荐的用法，当使用这种用法时，其设定的命令将作为容器启动时的默认执行命令，例如：
```
CMD  ["x11vnc", "-forever", "-usepw", "-create"]
```
当使用第二种用法时，其中的param将作为ENTERPOINT的默认参数使用。而第三种用法是将后面的命令作为shell命令，依靠/bin/sh–C来执行，例如：
```
CMD echo "This is a test." | wc -
```
如果用户需要脱离shell环境来执行命令，那么建议使用第一种用法来设定。

但无论使用哪种方法，都需要注意，在一个Dockerfile中可以同时出现多次CMD指令，可只有最后一次CMD命令会生效。同时在CMD中也只能出现双引号，不能使用单引号。与RUN命令一样，如果需要使用环境变量，请使用sh–C。

CMD与RUN都是执行命令的命令，但RUN是在镜像构建过程中执行的命令。而CMD命令只是设定好需要执行的命令，只有等容器启动时才会真正执行。

### LABEL命令
```
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```
LABEL是一个采用键值对的形式来向镜像中添加元数据的命令，例如：
```
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."
```
如果键值对中存在空格，那么就需要使用双引号来回避可能出现的错误，例如：
```
LABEL "com.example.vendor"="ACME Incorporated"
```
执行完LABEL命令之后，同样会产生一个新的文件层。因此为了避免出现126层的问题，我们建议将多个LABEL放到一起，统一执行，例如：
```
LABEL multi.label1="value1" multi.label2="value2" other="value3"
```
如果新添加的LAEBL在旧的镜像中已经存在，那么新添加的LABEL将会覆盖旧值。

当镜像构建成功之后，可以通过Docker CLI中的inspec命令查询到LABEL值，如下所示：
```
"Labels": {
    "com.example.vendor": "ACME Incorporated"
    "com.example.label-with-value": "foo",
    "version": "1.0",
    "description": "This text illustrates that label-values can span multiple lines.",
    "multi.label1": "value1",
    "multi.label2": "value2",
    "other": "value3"
}
```

### EXPOSE命令
```
EXPOSE <port> [<port>...]
```
EXPOSE命令是当容器运行时，来通知Docker这个容器中哪些端口是应用程序用来监听的。例如，在一个tomcat的Dockerfile中，一定会出现下面的EXPOSE指令：
```
EXPOSE 8080
CMD ["catalina.sh", "run"]
```
当Tomcat容器运行时，Tomcat应用就会开始监听8080端口了。

但是使用EXPOSE命令不等同于这些端口就可以被外部网络访问。只有在容器启动时，配合-p或者-P，外部网络才可以访问到这些端口。如果没有使用-p或者-P，那么这些端口只能被主机中的其他容器访问，无法被外部网络所访问到。

### ENV命令
```
ENV <key> <value>
ENV <key>=<value> ...
```
ENV命令用来在基础镜像中设定环境变量，有上面两种用法。同LABELS一样，ENV命令使用的也是key-value的方式存储数据。当使用第一种用法时，第一个字符串将被当作key来处理，后面的所有字符串将被当作value来处理，如下所示：
```
ENV myName John Doe
```
当采用第二种用法时，等号左边将是key，右边是value。如果value中存在空格时，需要使用“\”来进行转义，如下所示：
```
ENV myDog=Rex\ The\ Dog \
```
同时在使用第一种用法时，每个ENV命令只能维护一条环境变量，而第二种用法可以同时维护多条变量，如下所示：
```
##第一种用法
ENV myName John Doe
ENV myDog Rex The Dog
ENV myCat fluffy
##第二种用法
ENV myName="John Doe" myDog=Rex\ The\ Dog \
    myCat=fluffy
```
虽然在最终的镜像中，环境变量中的结果是一样的，但使用第二种用法时，这些环境变量会被保存在同一个文件层中。

当容器运行时，可以通过Docker inspect来查看，同时也可以在执行Docker run时通过-e来重新设定环境变量。

## COPY 与 ADD 命令
Dockerfile 中提供了两个非常相似的命令 COPY 和 ADD，总结其各自适合的应用场景。

## Build 上下文的概念
在使用 docker build 命令通过 Dockerfile 创建镜像时，会产生一个 build 上下文(context)。所谓的 build 上下文就是 docker build 命令的 PATH 或 URL 指定的路径中的文件的集合。在镜像 build 过程中可以引用上下文中的任何文件，比如我们要介绍的 COPY 和 ADD 命令，就可以引用上下文中的文件。

默认情况下 docker build -t testx . 命令中的 . 表示 build 上下文为当前目录。当然我们可以指定一个目录作为上下文，比如下面的命令：
```
$ docker build -t testx /home/nick/hc
```
我们指定 /home/nick/hc 目录为 build 上下文，默认情况下 docker 会使用在上下文的根目录下找到的 Dockerfile 文件。

### COPY 和 ADD 命令不能拷贝上下文之外的本地文件
对于 COPY 和 ADD 命令来说，如果要把本地的文件拷贝到镜像中，那么本地的文件必须是在上下文目录中的文件。其实这一点很好解释，因为在执行 build 命令时，docker 客户端会把上下文中的所有文件发送给 docker daemon。考虑 docker 客户端和 docker daemon 不在同一台机器上的情况，build 命令只能从上下文中获取文件。如果我们在 Dockerfile 的 COPY 和 ADD 命令中引用了上下文中没有的文件，就会收到类似下面的错误：

### ADD命令
ADD指令的功能是将主机构建环境（上下文）目录中的文件和目录、以及一个URL标记的文件 拷贝到镜像中。

其格式是：
```
ADD <src>... <dest>
ADD ["<src>",... "<dest>"] （包含空格的路径使用这种格式）
```
ADD命令是将src标记的文件，添加到容器中dest所标记的路径中去。src标记的文件可以是本地文件，也可以本地目录，甚至可以是远程URL链接。

当src标记的是本地文件或者目录时，其相对路径应该是相对于Dockerfile所在目录的路径，而dest则应该指向容器中的目录。如果这个目录不存在，那么当ADD命令执行时，将会在容器中自动创建此目录。

有如下注意事项：
```
FROM ubuntu
MAINTAINER hello

# 在src标记的路径中，允许使用通配符，例如：
ADD hom* /mydir/        # 添加所有以hom开头的文件
ADD hom?.txt /mydir/    # ?可以被任意单个字符所代替

# 而dest的路径则不允许使用通配符，并且其路径必须是绝对路径，或者相对于WORKDIR的相对路径，例如：
ADD test aDir/          #假设/是WORKDIR，那么就添加test到/aDir/

ADD test1.txt test1.txt
ADD test1.txt test1.txt.bak

# 如果源路径是个文件，且目标路径是以 / 结尾， 则docker会把目标路径当作一个目录，会把源文件拷贝到该目录下。
## 如果目标路径不存在，则会自动创建目标路径。
ADD test1.txt /mydir/

# 如果源路径是个文件，且目标路径是不是以 / 结尾，则docker会把目标路径当作一个文件。
## 如果目标路径不存在，会以目标路径为名创建一个文件，内容同源文件；
## 如果目标文件是个存在的文件，会用源文件覆盖它，当然只是内容覆盖，文件名还是目标文件名。
## 如果目标文件实际是个存在的目录，则会源文件拷贝到该目录下。 注意，这种情况下，最好显示的以 / 结尾，以避免混淆。
ADD data1  data1
ADD data2  data2

# 如果源文件是个归档文件（压缩文件），则docker会自动帮解压。
ADD zip.tar /myzip

# 如果源路径是个目录，且目标路径不存在，则docker会自动以目标路径创建一个目录，把源路径目录下的文件拷贝进来。
## 如果目标路径是个已经存在的目录，则docker会把源路径目录下的文件拷贝到该目录下。
ADD data/  test
ADD data/  test/
```
ADD 的最佳用途是将本地压缩包文件自动提取到镜像中：如下情况，自动解压缩的功能非常有用，比如官方镜像 ubuntu 中：
```
FROM scratch
ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /
...
```
提示：但在某些情况下，如果我们真的是希望复制个压缩文件进去，而不解压缩，这时就不可以使用 ADD 命令了。

由于镜像的体积很重要，所以强烈建议不要使用 ADD 从远程 URL 获取文件，下载文件我们应该使用 curl 或 wget 来代替。

对于不需要自动解压的文件或目录，应该始终使用 COPY。

### COPY命令
```
COPY <src>... <dest>
COPY ["<src>",... "<dest>"]
```
COPY指令和ADD指令功能和使用方式类似。只是COPY指令不会做自动解压工作。

COPY命令也是向容器中指定路径下添加文件。在添加文件时，同样需要确认此文件的确存在于Dockerfile所在目录中。与ADD命令相同，COPY命令也支持下例格式中的通配符：
```
COPY hom* /mydir/        # 添加所有以hom开头的文件
COPY hom?.txt /mydir/    # ?可以被任意单个字符所代替
```
在dest中的路径必须是全路径或者是相对于WORKDIR的相对路径。



### ENTRYPOINT命令
```
ENTRYPOINT ["executable", "param1", "param2"]
ENTRYPOINT command param1 param2
```
ENTRYPOINT是用来设定容器运行时默认执行程序的命令。第一种用法是Docker官方推荐的用法。通过第一种用法，读者可以自行设定需要执行的二进制程序和参数。而第二种方法则将所设定的二进制程序限制在/bin/sh–C下执行。

下面我们通过一个实例来演示ENTRYPOINT命令的用法：
```
##首先截取Nginx镜像部分Dockerfile
RUN sed -Ei 's/^(bind-address|log)/#&/' /etc/mysql/my.cnf \
  && echo 'skip-host-cache\nskip-name-resolve' | awk '{ print } $1 == "[mysqld]" && c == 0 { c = 1;
system("cat") }' /etc/mysql/my.cnf > /tmp/my.cnf \
  && mv /tmp/my.cnf /etc/mysql/my.cnf

VOLUME /var/lib/mysql

COPY Docker-entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]

EXPOSE 3306
CMD  ["mysqld"]
```
在MySQL官方提供的Dockerfile中，ENTRYPOINT命令使用的是第一种用法，设定含义为当mysql容器运行时，自动执行/entrypoint.sh，而参数则是mysqld。（为什么mysqld是参数，读者可参考CMD一节）

而/entrypoint.sh的功能就是启动mysql守护进程，并且创建默认用户。当entrypoint.sh执行成功之后，一个可供用户使用的MySQL数据库就准备好了。

Docker run mysql param1 param2所提供的两个参数，都将作为参数传入entrypoint.sh中。

下面我们再看一下示例：
```
##启动一个Nginx容器
Docker run -i -t --rm -p 80:80 nginx
```
当这个命令执行成功之后，一个监听80端口同时对外提供Web应用的Nginx容器就准备好了。但我们只是创建了容器，并未初始化Nginx应用，那么nginx应用的初始化工作是谁做的呢？

其实这些初始化工作就是依靠Nginx容器中ENTRYPOINT设定的脚本执行的。如果我们在run命令后面添加了其他参数，这些参数就会传入ECTRYPOINT设定的脚本中，同时CMD中所设定的参数也将会失效。

而ENTRYPOINT所设定的值，可以在run命令中通过-entrypoint来修改，例如：
```
Docker run –entrypoint="/bin/sh"  -d mysql
##将entrypoint.sh替换为/bin/sh
```
与CMD命令一样，ENTRYPOINT命令可以在Dockerfile中出现多次，但只有最后一次出现的ENTRYPOINT才会生效。

下面我们通过几个演示来加深对ENTRYPOINT命令的理解。

首先我们构建一个ubuntu容器，并且设定了ENTRYPOINT和CMD：
```
FROM ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]
```
当直接创建这个ubuntu容器之后，就可以直接查看到top–b–H执行的结果：
```
$ Docker run -it --rm --name test  top -H
top - 08:25:00 up  7:27,  0 users,  load average: 0.00, 0.01, 0.05
Threads:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.1 us,  0.1 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:   2056668 total,  1616832 used,   439836 free,    99352 buffers
KiB Swap:  1441840 total,        0 used,  1441840 free.  1324440 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
    1 root      20   0   19744   2336   2080 R  0.0  0.1   0:00.04 top
```
下面通过ps命令来验证结果：
```
$ Docker exec -it test ps aux

USER       PID %CPU %MEM    VSZ   RSS  TTY      STAT START   TIME COMMAND
root             1  2.6  0.1  19752     2352 ?        Ss+     08:24   0:00    top -b -H
root             7  0.0  0.1  15572     2164 ?        R+      08:25   0:00   ps aux
```
加黑的进程表示目前运行的进程就是top–b–H.

为什么是-H而不是-C呢？这是因为当run命令后没有添加其他参数时，CMD指定的-C将作为参数附加到top–b之后。如果run命令后面添加了其他参数，此时CMD指定的参数将会失效。

当我们使用ENTRYPOINT命令的第二种用法设定值时，又会怎样呢？

当使用第二种用法时，ENTRYPOINT命令设定的二进制程序将会忽略所有来自于CMD和RUN命令后面所添加的参数，只会运行ENTRYPOINT命令所设定的二进制程序。同时，为了确保容器可以正确处理stop命令发来的SIG信号，Docker建议使用exec来启动二进制程序。具体原因，我们看一下下面的示例：

首先我们构建一个同样执行top命令的ubuntu容器：
```
FROM ubuntu
ENTRYPOINT exec top –b
CMD ["-c"]
```
当我们运行这个容器时，会出现下面的情况：
```
$ Docker run -it --rm --name test top

Mem: 1704520K used, 352148K free, 0K shrd, 0K buff, 140368121167873K cached
CPU:   5% usr   0% sys   0% nic  94% idle   0% io   0% irq   0% sirq
Load average: 0.08 0.03 0.05 2/98 6
  PID  PPID USER     STAT   VSZ %VSZ %CPU COMMAND
    1     0 root     R     3164   0%   0% top -b
```
当前容器中只运行着top–b命令，CMD的参数和run后面添加的top都没有发挥作用。同时top–b变成了根进程，PID=1。

如果此时执行Docker stop，则可以正确关闭此容器。但如果我们在设定ENTRYPOINT时忘记使用exec了，那么就会出现下面的情况：
```
##Dockerfile
FROM ubuntu
ENTRYPOINT top –b

##运行容器
$ Docker run -it --name test
Mem: 1704184K used, 352484K free, 0K shrd, 0K buff, 140621524238337K cached
CPU:   9% usr   2% sys   0% nic  88% idle   0% io   0% irq   0% sirq
Load average: 0.01 0.02 0.05 2/101 7
  PID  PPID USER     STAT   VSZ %VSZ %CPU COMMAND
    1     0 root     S     3168   0%   0% /bin/sh -c top -b
    7     1 root     R     3164   0%   0% top -b
```
可以看到top–b不再是根进程了，而是变成了sh的子进程。此时执行Docker stop，因为sh不会处理Linux信号，所以容器不会正确关闭。只有过了所设定的超时时间后，通过SIGKILL信号才能强行关闭。

11. VOLUME命令

VOLUME ["/data"]
VOLUME可以在容器内部创建一个指定名称的挂载点。VOLUME命令参数可以采用类似VOLUME ["/var/log/"]这样的JSON格式数据（当使用JSON格式时，只能使用双引号）。也可以是使用空格分隔的字符串，例如：VOLUME/var/log/var/db。

如下所示：

FROM ubuntu
RUN mkdir /myvol
RUN echo "hello world" > /myvol/greeting
VOLUME /myvol
我们创建了一个新的挂载点，并且在此挂载点中生成了greeting文件。

在使用VOLUME命令时需要注意，如果在Dockerfile中已经声明了某个挂载点，那么以后对此挂载点中文件的操作将不会生效。因此，一般来说，只会在Dockerfile的结尾处声明挂载点。

12. USER命令

USER daemon
USER命令是用来切换用户身份的。当执行完USER命令后，其后面所有的命令都将以新用户的身份来执行。

13. WORKDIR命令

WORKDIR /path/to/workdir
WORKDIR是用来切换当前工作目录的指令。WORKDIR命令中所切换的工作目录，可以影响到后续的RUN、CMD、ENTRYPOINT、COPY和ADD指令中的路径。

WORKDIR可以在Dockerfile中出现多次，但最终生效的路径是所有WORKDIR指定路径的叠加，例如：

WORKDIR /a
##当前目录为/a
WORKDIR b
##当前工作目录为/a/b
WORKDIR c
##当前工作目录为/a/b/c
RUN pwd
##最终结果为/a/b/c
如果需要切换到其他工作目录，那么应该使用全路径进行切换。如果使用相对路径，则默认是在当前目录中切换。

在WORKDIR中只可以使用ENV设定的环境变量值，例如下例：

ENV DIRPATH /path
WORKDIR $DIRPATH/$DIRNAME
因为DIRPATH在环境变量中存在，所以最终结果为：/path/$DIRNAME

14. ONBUILD命令

ONBUILD [INSTRUCTION]
ONBUILD是用来创建触发命令集的命令。由ONBUILD创建的触发命令集在当前Dockerfile执行过程中不会执行，而当此镜像被其他镜像当作基础镜像使用时，将会被触发执行。

ONBUILD不挑食，所有只要是在Dockerfile中属于合法的内置命令，都可以在此设定。

这是一个非常有意思的功能，例如，当前镜像A包含一个特殊应用，当此镜像被其他镜像当作基础镜像而引入时，镜像A中的ONBUILD指令集就会自动运行，创建初始化用户，或者初始化环境变量，以及自动创建挂载点等操作。

再比如，镜像A是一个Python应用程序的编译环境。在使用镜像A编译自己的代码时，可能需要将源代码复制到一个特定目录，然后执行编译脚本。这些如果都需要用户来做，那么会相当费时费力。而镜像A在构建时，可以把这些工作都封装成ONBUILD指令集，自动运行。

下面我们看一下ONBUILD的运行机制：

（1）当Dockerfile执行时，如果遇到ONBUILD标记的命令，其将会把这些命令作为镜像的元数据保存起来，并且在本次执行时不再执行这些命令。

（2）在Dockefile所有的命令都执行成功后，标记为ONBUILD的命令将会被保存到镜像的manifest文件中，后续可以通过Docker inspect来查看。

（3）当此镜像被用作基础镜像时，Docker首先会取出这些标记为ONBUILD的命令，然后按照其当初被标记的顺序执行。如果有一条执行失败，则本次Dockefile整体失败返回。如果所有的ONBUILD命令执行成功，则FROM步骤成功，然后再执行后续的指令。

（4）这些命令不会被子镜像继承。

百闻不如一见，我们看一个使用ONBUILD命令的示例：

[...]
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
[...]
在ONBUILD命令中不允许嵌套，即ONBUILD ONBUILD是不允许的，同时在ONBUILD命令中也不允许执行FROM和MAINTAINER命令。

## Dockerfile的多阶段构建
从docker17.05版本开始，dockerfile中允许使用多个FROM指令(multistage),该特性可以使编译环境和发布环境分离。Docker 17.05版本以后，新增了Dockerfile多阶段构建。所谓多阶段构建，实际上是允许一个Dockerfile 中出现多个 FROM 指令。这样做有什么意义呢？

老版本Docker中为什么不支持多个 FROM 指令

Docker的各个层是有相关性的，在联合挂载的过程中，系统需要知道在什么样的基础上再增加新的文件。那么这就要求一个Docker镜像只能有一个起始层，只能有一个根。所以，Dockerfile中，就只允许一个 FROM指令。因为多个 FROM 指令会造成多根，则是无法实现的。但为什么 Docker 17.05 版本以后允许 Dockerfile支持多个 FROM 指令了呢，莫非已经支持了多根？

多个 FROM 指令的意义
多个 FROM 指令并不是为了生成多根的层关系，最后生成的镜像，仍以最后一条 FROM 为准，之前的 FROM 会被抛弃，那么之前的FROM 又有什么意义呢？

每一条 FROM 指令都是一个构建阶段，多条 FROM 就是多阶段构建，虽然最后生成的镜像只能是最后一个阶段的结果，但是，能够将前置阶段中的文件拷贝到后边的阶段中，这就是多阶段构建的最大意义。

最大的使用场景是将编译环境和运行环境分离，

编译阶段
```
FROM golang:1.10.3

COPY server.go /build/

WORKDIR /build

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GOARM=6 go build -ldflags ‘-w -s’ -o server
```
运行阶段
```
FROM scratch
```

从编译阶段的中拷贝编译结果到当前镜像中
```
COPY --from=0 /build/server /

ENTRYPOINT ["/server"]
```
这个 Dockerfile 的玄妙之处就在于 COPY 指令的 --from=0 参数，从前边的阶段中拷贝文件到当前阶段中，多个FROM语句时，0代表第一个阶段。除了使用数字，我们还可以给阶段命名，比如：
```
## 编译阶段 命名为 builder
FROM golang:1.10.3 as builder

## 从编译阶段的中拷贝编译结果到当前镜像中
COPY --from=builder /build/server /
```
更为强大的是，COPY --from 不但可以从前置阶段中拷贝，还可以直接从一个已经存在的镜像中拷贝。比如，
```
FROM ubuntu:16.04

COPY --from=quay.io/coreos/etcd:v3.3.9 /usr/local/bin/etcd /usr/local/bin/
```
我们直接将etcd镜像中的程序拷贝到了我们的镜像中，这样，在生成我们的程序镜像时，就不需要源码编译etcd了，直接将官方编译好的程序文件拿过来就行了。

有些程序要么没有apt源，要么apt源中的版本太老，要么干脆只提供源码需要自己编译，使用这些程序时，我们可以方便地使用已经存在的Docker镜像作为我们的基础镜像。但是我们的软件有时候可能需要依赖多个这种文件，我们并不能同时将 nginx 和 etcd 的镜像同时作为我们的基础镜像（不支持多根），这种情况下，使用 COPY --from 就非常方便实用了。




