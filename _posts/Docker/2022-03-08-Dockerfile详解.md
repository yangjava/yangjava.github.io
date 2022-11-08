---
layout: post
categories: Docker
description: none
keywords: Docker
---
# Dockerfile详解
天生我材必有用，千金散尽还复来。——李白《将进酒》

### Dockerfile

镜像是多层存储，每一层是在前一层的基础上进行的修改；Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

我们所使用的镜像基本都是来自于 Docker Hub 的镜像，直接使用这些镜像是可以满足一定的需求，而当这些镜像无法直接满足需求时，我们就需要定制这些镜像。

编写 Dockerfile 时，主要会用到如下一些指令：

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

Dockerfile 中每一个指令都会建立一层，在其上执行后面的命令，执行结束后， commit 这一层的修改，构成新的镜像。对于有些指令，尽量将多个指令合并成一个指令，否则容易产生非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错。而且Union FS 是有最大层数限制的，比如 AUFS不得超过 127 层。因此镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

**① FROM**

一个 Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令。

格式：FROM <镜像名称>[:<TAG>]

Docker 还存在一个特殊的镜像，名为 scratch 。这个镜像并不实际存在，它表示一个空白的镜像。如果以 scratch 为基础镜像的话，意味着不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。

**② RUN**

RUN 指令是用来执行命令行命令的。由于命令行的强大能力， RUN 指令在定制镜像时是最常用的指令之一。

多个 RUN 指令尽量合并成一个，使用 && 将各个所需命令串联起来。Dockerfile 支持 Shell 类的行尾添加 \ 的命令换行方式，以及行首 # 进行注释的格式。

格式：

- shell 格式： RUN <命令> ，就像直接在命令行中输入的命令一样。
- exec 格式： RUN ["可执行文件", "参数1", "参数2"] ，这更像是函数调用中的格式。

**③ COPY**

COPY 指令将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置。<源路径> 可以是多个，甚至可以是通配符。<目标路径> 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径。

使用 COPY 指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。

格式：

- COPY <源路径>,... <目标路径>
- COPY ["<源路径1>",... "<目标路径>"]

**④ ENV**

设置环境变量，无论是后面的其它指令，如 RUN ，还是运行时的应用，都可以直接使用这里定义的环境变量。

格式：

- ENV <key> <value>
- ENV <key1>=<value1> <key2>=<value2>...

**⑤ USER**

USER 用于切换到指定用户，这个用户必须是事先建立好的，否则无法切换。

格式： USER <用户名>

**⑥ EXPOSE**

EXPOSE 指令是声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。

格式：EXPOSE <端口1> [<端口2>...]

**⑦ HEALTHCHECK**

HEALTHCHECK 指令是告诉 Docker 应该如何判断容器的状态是否正常。通过该指令指定一行命令，用这行命令来判断容器主进程的服务状态是否还正常，从而比较真实的反应容器实际状态。

当在一个镜像指定了 HEALTHCHECK 指令后，用其启动容器，初始状态会为 starting ，在 HEALTHCHECK 指令检查成功后变为 healthy ，如果连续一定次数失败，则会变为 unhealthy

格式：

- HEALTHCHECK [选项] CMD <命令> ：设置检查容器健康状况的命令，CMD 命令的返回值决定了该次健康检查的成功与否： 0 - 成功； 1 - 失败；。
- HEALTHCHECK NONE ：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令

HEALTHCHECK 支持下列选项：

- --interval=<间隔> ：两次健康检查的间隔，默认为 30 秒；
- --timeout=<时长> ：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
- --retries=<次数> ：当连续失败指定次数后，则将容器状态视为unhealthy ，默认 3 次。

**⑧ WORKDIR**

使用 WORKDIR 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，该目录需要已经存在， WORKDIR 并不会帮你建立目录。

在Dockerfile 中，两行 RUN 命令的执行环境是不同的，是两个完全不同的容器。每一个 RUN 都是启动一个容器、执行命令、然后提交存储层文件变更。因此如果需要改变以后各层的工作目录的位置，那么应该使用 WORKDIR 指令。

格式：WORKDIR <工作目录路径>

**⑨ VOLUME**

容器运行时应该尽量保持容器存储层不发生写操作，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中。为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 Dockerfile 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。

格式：

- VOLUME ["<路径1>", "<路径2>"...]
- VOLUME <路径>

这里的路径会在运行时自动挂载为匿名卷，任何向此路径中写入的信息都不会记录进容器存储层，从而保证了容器存储层的无状态化。

**⑩ CMD**

CMD 指令用于指定默认的容器主进程的启动命令。

格式：

- shell 格式： CMD <命令>
- exec 格式： CMD ["可执行文件", "参数1", "参数2"...]

在运行时可以指定新的命令来替代镜像设置中的这个默认命令，跟在镜像名后面的是 command ，运行时会替换 CMD 的默认值；例如：docker run -ti nginx /bin/bash，"/bin/bash" 就是替换命令。

在指令格式上，一般推荐使用 exec 格式，这类格式在解析时会被解析为 JSON数组，一定要使用双引号，而不要使用单引号。如果使用 shell 格式的话，实际的命令会被包装为 sh -c 的参数的形式执行，如：CMD echo $HOME 会被包装为 CMD [ "sh", "-c", "echo $HOME" ]，实际就是一个 shell 进程。

注意：Docker 不是虚拟机，容器中的应用都应该以前台执行。对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出。启动程序时，应要求其以前台形式运行。

**⑪ ENTRYPOINT**

ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数。 ENTRYPOINT 在运行时也可以替代，不过比 CMD 要略显繁琐，需要通过 docker run 的参数 --entrypoint 来指定。

格式：

- shell 格式： ENTRYPOINT <命令>
- exec 格式： ENTRYPOINT ["可执行文件", "参数1", "参数2"...]

当指定了 ENTRYPOINT 后， CMD 的含义就发生了改变，不再是直接的运行其命令，而是将 CMD 的内容作为参数传给 ENTRYPOINT 指令。ENTRYPOINT 中的参数始终会被使用，而 CMD 的额外参数可以在容器启动时动态替换掉。

```
例如：
ENTRYPOINT ["/bin/echo", "hello"]  

容器通过 docker run -ti <image> 启动时，输出为：hello

容器通过 docker run -ti <image> docker 启动时，输出为：hello docker

将Dockerfile修改为：

ENTRYPOINT ["/bin/echo", "hello"]  
CMD ["world"]

容器通过 docker run -ti <image> 启动时，输出为：hello world

容器通过 docker run -ti <image> docker 启动时，输出为：hello docker
```

### Docker执行Dockerfile的大致流程

1. docker从基础镜像运行一个容器
2. 执行一条指令并对容器作出修改
3. 执行类似docker commit的操作提交一个新的镜像层
4. docker再基于刚提交的镜像运行一个新容器
5. 执行dockerfile中的下一条指令直到所有指令都执行完成

总结:

从应用软件的角度来看，Dockerfile、Docker镜像与Docker容器分别代表软件的三个不同阶段，

- Dockerfile是软件的原材料
- Docker镜像是软件的交付品
- Docker容器则可以认为是软件的运行态。 Dockerfile面向开发，Docker镜像成为交付标准，Docker容器则涉及部署与运维，三者缺一不可，合力充当Docker体系的基石。

### 构建基础镜像

以构建 nginx 基础镜像为例看如何使用 Dockerfile 构建镜像。

① 编写 Dockerfile

```
# 基于 centos:8 构建基础镜像
FROM centos:8
# 作者
MAINTAINER jiangzhou.bo@vip.163.com
# 安装编译依赖的 gcc 等环境，注意最后清除安装缓存以减小镜像体积
RUN yum install -y gcc gcc-c++ make \
    openssl-devel pcre-devel gd-devel \
    iproute net-tools telnet wget curl \
    && yum clean all \
    && rm -rf /var/cache/yum/*
# 编译安装 nginx
RUN wget http://nginx.org/download/nginx-1.17.4.tar.gz \
    && tar zxf nginx-1.17.4.tar.gz \
    && cd nginx-1.17.4 \
    && ./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module \
    && make && make install \
    && rm -rf /usr/local/nginx/html/* \
    && echo "hello nginx !" >> /usr/local/nginx/html/index.html \
    && cd / && rm -rf nginx-1.17.4*
# 加入 nginx 环境变量
ENV PATH $PATH:/usr/local/nginx/sbin
# 拷贝上下文目录下的配置文件
COPY ./nginx.conf /usr/local/nginx/conf/nginx.conf
# 设置工作目录
WORKDIR /usr/local/nginx
# 暴露端口
EXPOSE 80
# daemon off: 要求 nginx 以前台进行运行
CMD ["nginx", "-g", "daemon off;"]
```

② 构建镜像

命令：**docker build -t <镜像名称>:<TAG> \**[-f <Dockerfile 路径>]\** <上下文目录>**

- **上下文目录**：镜像构建的上下文。Docker 是 C/S 结构，docker 命令是客户端工具，一切命令都是使用的远程调用形式在服务端(Docker 引擎)完成。当构建的时候，用户指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。这样在 COPY 文件的时候，实际上是复制上下文路径下的文件。一般来说，应该将 Dockerfile 以及所需文件置于一个空目录下，并指定这个空目录为上下文目录。
- Dockerfile 路径：可选，缺省时会寻找当前目录下的 Dockerfile 文件，如果是其它名称的 Dockerfile，可以使用 -f 参数指定文件路径。

在 nginx 目录下包含了构建需要的 Dockerfile 文件及配置文件，docker build 执行时，可以清晰地看到镜像的构建过程。构建成功后，就可以在本地看到已经构建好的镜像。

```
docker build 命令用于使用 Dockerfile 创建镜像。

语法
docker build [OPTIONS] PATH | URL | -
OPTIONS说明：

--build-arg=[] :设置镜像创建时的变量；

--cpu-shares :设置 cpu 使用权重；

--cpu-period :限制 CPU CFS周期；

--cpu-quota :限制 CPU CFS配额；

--cpuset-cpus :指定使用的CPU id；

--cpuset-mems :指定使用的内存 id；

--disable-content-trust :忽略校验，默认开启；

-f :指定要使用的Dockerfile路径；

--force-rm :设置镜像过程中删除中间容器；

--isolation :使用容器隔离技术；

--label=[] :设置镜像使用的元数据；

-m :设置内存最大值；

--memory-swap :设置Swap的最大值为内存+swap，"-1"表示不限swap；

--no-cache :创建镜像的过程不使用缓存；

--pull :尝试去更新镜像的新版本；

--quiet, -q :安静模式，成功后只输出镜像 ID；

--rm :设置镜像成功后删除中间容器；

--shm-size :设置/dev/shm的大小，默认值是64M；

--ulimit :Ulimit配置。

--squash :将 Dockerfile 中所有的操作压缩为一层。

--tag, -t: 镜像的名字及标签，通常 name:tag 或者 name 格式；可以在一次构建中为一个镜像设置多个标签。

--network: 默认 default。在构建期间设置RUN指令的网络模式
```

使用当前目录的 Dockerfile 创建镜像

```
docker build -t mysql .
```

使用URL **github.com/creack/docker-firefox** 的 Dockerfile 创建镜像。

```
docker build github.com/creack/docker-firefox
```

也可以通过 -f Dockerfile 文件的位置：

```
docker build -f Dockerfile .
```

在 Docker 守护进程执行 Dockerfile 中的指令前，首先会对 Dockerfile 进行语法检查，有语法错误时会返回：

```
Error response from daemon: dockerfile parse error line 1: unknown instruction: ://GITHUB.COM/ELASTIC/ELASTICSEARCH-DOCKER
```

基于构建的镜像启用容器并测试

③ 构建应用镜像

构件好基础镜像之后，以这个基础镜像来构建应用镜像。

首先编写 Dockerfile：将项目下的 html 目录拷贝到 nginx 目录下

```
FROM bojiangzhou/nginx:v1
COPY ./html /usr/local/nginx/html
```

项目镜像构建成功后，运行镜像，可以看到内容已经被替换了。我们构建好的项目镜像就可以传给其他用户去使用了。