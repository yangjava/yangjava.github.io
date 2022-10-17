# Redis

[TOC]

## **Redis简介**

Redis:REmote DIctionary Server(远程字典服务)。

是由意大利人Salvatore Sanfilippo（网名：antirez）开发的一款内存高速**缓存**数据库。是完全开源免费的，用**C语言**编写的，遵守BSD协议，高性能的(**key/value**)**分布式内存数据库**，基于内存运行并支持**持久化**的**NoSQL**数据库。

Redis是一款开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存也可持久化的日志型、Key-Value高性能数据库。Redis与其他Key-Value缓存产品相比有以下三个特点：

- 支持数据持久化，可以将内存中的数据保存在磁盘中，重启可再次加载使用
- 支持简单的Key-Value类型的数据，同时还提供List、Set、Zset、Hash等数据结构的存储
- 支持数据的备份，即Master-Slave模式的数据备份

同时，我们再看下Redis有什么优势：

- 读速度为110000次/s，写速度为81000次/s，性能极高
- 具有丰富的数据类型，这个上面已经提过了
- Redis所有操作都是原子的，意思是要么成功执行要么失败完全不执行，多个操作也支持事务
- 丰富的特性，比如Redis支持publish/subscribe、notify、key过期等

## **Redis安装、启动**

在Linux系统上安装并启动Redis的步骤（我的Redis安装在/data/component/redis目录下，每一步使用的命令标红加粗）：

- 进入目录，**cd /data/component/redis**
- 下载Redis，**wget http://download.redis.io/releases/redis-3.2.11.tar.gz**，Redis地址为http://download.redis.io/releases/redis-4.0.9.tar.gz
- 解压下载下来的tar包，**tar -zxvf redis-3.2.11.tar.gz**，解压完毕的文件夹名称为redis-3.2.11
- 进入redis-3.2.11，**cd redis-3.2.11**
- 由于我们下载下来的是源文件，因此使用**make**命令对源文件进行一个构建，构建完毕我们会发现src目录下多出了redis-benchmark、redis-check-aof、redis-check-rdb、redis-cli、redis-sentinel、redis-server几个可执行文件，这几个可执行文件后面会说到
- 由于上述几个命令在/data/component/redis/redis-3.2.11/src目录下，为了更方便地使用这几个命令而不需要指定全路径，配置一下环境变量。这里我是以非root用户进行登录的，因此配置用户变量，先执行**cd**命令回到初始目录，再**vi ./.bash_profile**，在path这一行加入**PATH=PATH:PATH:HOME/.local/bin:$HOME/bin:/data/component/redis/redis-3.2.11/src**，使用**:wq**保存并退出
- 使环境变量生效，执行**source ./.bash_profile**
- 使用redis-server即可启动redis，**redis-server /data/component/redis/redis-3.2.11/redis.conf**

不过这个时候我们的启动稍微有点问题，不是后台启动的，即ctrl+c之后Redis就停了：

为了解决这个问题，我们需要修改一下redis.conf，将Redis设置为以守护进程的方式进行启动，打开redis.conf，找到daemonize，将其设置为yes即可：

这个时候先关闭一下再启动，Redis就在后台自动运行了，关闭Redis有两种方式：

- redis-cli shutdown，这是种安全关闭redis的方式，**但这种写法只适用于没有配置密码的场景**，比较不安全，配置密码下一部分会讲
- kill -9 pid，这种方式就是强制关闭，可能会造成数据未保存

重启后，我们可以使用**ps -ef | grep redis**，**netstat -ant | grep 6379**命令来验证Redis已经启动。

**Redis登录授权**

上面我们安装了Redis，但这种方式是非常不安全的，因为没有密码，这样任何连接上Redis服务器的用户都可以对Redis执行操作，所以这一部分我们来讲一下给Redis设置密码。

打开redis.conf，找到"requirepass"部分，打开原本关闭的注释，替换一下自己想要的密码即可：

重启Redis，授权登录有两种做法：

- 连接的时候直接指定密码，**redis-cli -h 127.0.0.1 -p 6379 -a 123456**
- 连接后授权，**redis-cli -h 127.0.0.1 -p 6379**，**auth 123456**

在配置了密码的情况下，没有进行授权，那么对Redis发送的命令，将返回"(error) NOAUTH Authentication required."。

## **Redis配置文件**

上面两小节，设置使用守护线程启动、设置密码，都需要修改redis.conf，说明redis.conf是Redis核心的配置文件，本小节我们来看一下redis.conf中一些常用配置：

| 配置                          | 作用                                                         | 默认                               |
| ----------------------------- | ------------------------------------------------------------ | ---------------------------------- |
| bind                          | 当配置了bind之后：只有bind指定的ip可以直接访问Redis,这样可以避免将Redis服务暴露于危险的网络环境中，防止一些不安全的人随随便便通过远程访问Redis如果bind选项为空或0.0.0.0的话，那会接受所有来自于可用网络接口的连接 | 127.0.0.1                          |
| protected-mode                | protected-mode是Redis3.2之后的新特性，用于加强Redis的安全管理，当满足以下两种情况时，protected-mode起作用：bind未设置，即接收所有来自网络的连接密码未设置当满足以上两种情况且protected-mode=yes的时候，访问Redis将报错，即密码未设置的情况下，无密码访问Redis只能通过安装Redis的本机进行访问 | yes                                |
| port                          | Redis访问端口，由于Redis是单线程模型，因此单机开多个Redis进程的时候会修改端口，不然一般使用大家比较熟悉的6379端口就可以了 | 6379                               |
| tcp-backlog                   | 半连接队列的大小，对半连接队列不熟的可以看我以前的文章[TCP：三次握手、四次握手、backlog及其他](http://www.cnblogs.com/xrq730/p/6910719.html) | 511                                |
| timeout                       | 指定在一个client空闲多少秒之后就关闭它，0表示不管            | 0                                  |
| tcp-keepalive                 | 设置tcp协议的keepalive，从Redis的注释来看，这个参数有两个作用：发现死的连接从中间网络设备的角度看连接是否存活 | 300                                |
| daemonize                     | 这个前面说过了，指定Redis是否以守护进程的方式启动            | no                                 |
| supervised                    | 这个参数表示可以通过upstart和systemd管理Redis守护进程，这个具体和操作系统相关，资料也不是很多，就暂时不管了 | no                                 |
| pidfile                       | 当Redis以守护进程的方式运行的时候,Redis默认会把pid写到pidfile指定的文件中 | /var/run/redis_6379.pid            |
| loglevel                      | 指定Redis的日志级别，Redis本身的日志级别有notice、verbose、notice、warning四种，按照文档的说法，这四种日志级别的区别是：debug，非常多信息，适合开发/测试verbose，很多很少有用的信息（直译，读着拗口，从上下文理解应该是有用信息不多的意思），但并不像debug级别这么混乱notice，适度的verbose级别的输出，很可能是生产环境中想要的warning，只记录非常重要/致命的信息 | notice                             |
| logfile                       | 配置log文件地址,默认打印在命令行终端的窗口上                 | ""                                 |
| databases                     | 设置Redis数据库的数量，默认使用0号DB                         | 16                                 |
| save                          | 把Redis数据保存到磁盘上，这个是在RDB的时候用的，介绍RDB的时候专门说这个 | save 900 1save 300 10save 60 10000 |
| stop-writes-on-bgsave-error   | 当启用了RDB且最后一次后台保存数据失败，Redis是否停止接收数据。这会让用户意识到数据没有正确持久化到磁盘上，否则没有人会注意到灾难（disaster）发生了。如果Redis重启了，那么又可以重新开始接收数据了 | yes                                |
| rdbcompression                | 是否在RBD的时候使用LZF压缩字符串，如果希望省点CPU，那就设为no，不过no的话数据集可能就比较大 | yes                                |
| rdbchecksum                   | 是否校验RDB文件，在RDB文件中有一个checksum专门用于校验       | yes                                |
| dbfilename                    | dump的文件位置                                               | dump.rdb                           |
| dir                           | Redis工作目录                                                | ./                                 |
| slaveof                       | 主从复制，使用slaveof让一个节点称为某个节点的副本，这个只需要在副本上配置 | 关闭                               |
| masterauth                    | 如果主机使用了requirepass配置进行密码保护，使用这个配置告诉副本连接的时候需要鉴权 | 关闭                               |
| slave-serve-stale-data        | 当一个Slave与Master失去联系或者复制正在进行中，Slave可能会有两种表现：如果为yes，Slave仍然会应答客户端请求，但返回的数据可能是过时的或者数据可能是空的如果为no，在执行除了INFO、SLAVEOF两个命令之外，都会应答"SYNC with master in progres"错误 | yes                                |
| slave-read-only               | 配置Redis的Slave实例是否接受写操作，即Slave是否为只读Redis   | yes                                |
| slave-priority                | 从站优先级是可以从redis的INFO命令输出中查到的一个整数。当主站不能正常工作时，redis sentinel使用它来选择一个从站并将它提升为主站。  低优先级的从站被认为更适合于提升，因此如果有三个从站优先级分别是10， 100， 25，sentinel会选择优先级为10的从站，因为它的优先级最低。  然而优先级值为0的从站不能执行主站的角色，因此优先级为0的从站永远不会被redis sentinel提升。 | 100                                |
| requirepass                   | 设置客户端认证密码                                           | 关闭                               |
| rename-command                | 命令重命名，对于一些危险命令例如：flushdb（清空数据库）flushall（清空所有记录）config（客户端连接后可配置服务器）keys（客户端连接后可查看所有存在的键）          作为服务端redis-server，常常需要禁用以上命令来使得服务器更加安全，禁用的具体做法是是：rename-command FLUSHALL ""也可以保留命令但是不能轻易使用，重命名这个命令即可：rename-command FLUSHALL abcdefg这样，重启服务器后则需要使用新命令来执行操作，否则服务器会报错unknown command | 关闭                               |
| maxclients                    | 设置同时连接的最大客户端数量，一旦达到了限制，Redis会关闭所有的新连接并发送一个"max number of clients reached"的错误 | 关闭，默认10000                    |
| maxmemory                     | 不要使用超过指定数量的内存，一旦达到了，Redis会尝试使用驱逐策略来移除键 | 关闭                               |
| maxmemory-policy              | 当达到了maxmemory之后Redis如何移除数据，有以下的一些策略：volatile-lru，使用LRU算法，移除范围为设置了失效时间的allkeys-lru，根据LRU算法，移除范围为所有的volatile-random，使用随机算法，移除范围为设置了失效时间的allkeys-random，使用随机算法，移除范围为所有的volatile-ttl，移除最近过期的数据noeviction，不过期，当写操作的时候返回错误注意，当写操作且Redis发现没有合适的数据可以移除的时候，将会报错 | 关闭，noeviction                   |
| appendonly                    | 是否开启AOF，关于AOF后面再说                                 | no                                 |
| appendfilename                | AOF文件名称                                                  | appendonly.aof                     |
| appendfsync                   | 操作系统实际写数据到磁盘的频率，有以下几个选项：always，每次有写操作都进行同步，慢，但是最安全everysec，对写操作进行累积，每秒同步一次，是一种折衷方案no，当操作系统flush缓存的时候同步，性能更好但是会有数据丢失的风险当不确定是使用哪种的时候，官方推荐使用everysec，它是速度与数据安全之间的一种折衷方案 | everysec                           |
| no-appendfsync-on-rewrite     | aof持久化机制有一个致命的问题，随着时间推移，aof文件会膨胀，当server重启时严重影响数据库还原时间，因此系统需要定期重写aof文件。重写aof的机制为bgrewriteaof（另外一种被废弃了，就不说了），即在一个子进程中重写从而不阻塞主进程对其他命令的处理，但是这依然有个问题。bgrewriteaof和主进程写aof，都会操作磁盘，而bgrewriteaof往往涉及大量磁盘操作，这样就会让主进程写aof文件阻塞。针对上述问题，可以使用此时可以使用no-appendfsync-on-rewrite参数做一个选择：no，最安全，不丢失数据，但是需要忍受阻塞yes，数据写入缓冲区，不造成阻塞，但是如果此时redis挂掉就会丢失数据，在Linux操作系统默认设置下，最坏场景下会丢失30秒数据 | no                                 |
| auto-aof-rewrite-percentage   | 本次aof文件超过上次aof文件该值的百分比时，才会触发rewrite    | 100                                |
| auto-aof-rewrite-min-size     | aof文件最小值，只有到达这个值才会触发rewrite，即rewrite由auto-aof-rewrite-percentage+auto-aof-rewrite-min-size共同保证 | 64mb                               |
| aof-load-truncated            | redis在以aof方式恢复数据时，对最后一条可能出问题的指令的处理方式： yes，log并继续no，直接恢复失败 | yes                                |
| slowlog-log-slower-than       | Redis慢查询的最低条件，单位微妙，即查询时间>这个值的会被记录 | 10000                              |
| slowlog-max-len               | Redis存储的慢查询最大条数，超过该值之后会将最早的slowlog剔除 | 128                                |
| lua-time-limit                | 一个lua脚本执行的最大时间，单位为ms                          | 5000                               |
| cluster-enabled               | 正常来说Redis实例是无法称为集群的一部分的，只有以集群方式启动的节点才可以。为了让Redis以集群方式启动，就需要此参数。 | 关闭                               |
| cluster-config-file           | 每个集群节点应该有自己的配置文件，这个文件是不应该手动修改的，它只能被Redis节点创建且更新，每个Redis集群节点需要不同的集群配置文件 | 关闭，nodes-6379.conf              |
| cluster-node-timeout          | 集群中一个节点向其他节点发送ping命令时，必须收到回执的毫秒数 | 关闭，15000                        |
| cluster-slave-validity-factor | 如果该项设置为0，不管Slave节点和Master节点间失联多久都会一直尝试failover。比如timeout为5，该值为10，那么Master与Slave之间失联50秒，Slave不会去failover它的Master | 关闭，10                           |
| cluster-migration-barrier     | 当一个Master拥有多少个好的Slave时就要割让一个Slave出来。例如设置为2，表示当一个Master拥有2个可用的Slave时，它的一个Slave会尝试迁移 | 关闭，1                            |
| cluster-require-full-coverage | 有节点宕机导致16384个Slot全部被覆盖，整个集群是否停止服务，这个值一定要改为no | 关闭，yes                          |

以上把redis.conf里面几乎所有的配置都写了一遍（除了ADVANCED CONFIG部分），给大家作为参考吧。

①、bind:绑定redis服务器网卡IP，默认为127.0.0.1,即本地回环地址。这样的话，访问redis服务只能通过本机的客户端连接，而无法通过远程连接。如果bind选项为空的话，那会接受所有来自于可用网络接口的连接。

　　②、port：指定redis运行的端口，默认是6379。由于Redis是单线程模型，因此单机开多个Redis进程的时候会修改端口。

　　③、timeout：设置客户端连接时的超时时间，单位为秒。当客户端在这段时间内没有发出任何指令，那么关闭该连接。默认值为0，表示不关闭。

　　④、tcp-keepalive ：单位是秒，表示将周期性的使用SO_KEEPALIVE检测客户端是否还处于健康状态，避免服务器一直阻塞，官方给出的建议值是300s，如果设置为0，则不会周期性的检测。

具体配置详解：

　　①、daemonize:设置为yes表示指定Redis以守护进程的方式启动（后台启动）。默认值为 no

　　②、pidfile:配置PID文件路径，当redis作为守护进程运行的时候，它会把 pid 默认写到 /var/redis/run/redis_6379.pid 文件里面

　　③、loglevel ：定义日志级别。默认值为notice，有如下4种取值：

　　　　　　　　　　debug（记录大量日志信息，适用于开发、测试阶段）

　　　　　　　　　　verbose（较多日志信息）

　　　　　　　　　　notice（适量日志信息，使用于生产环境）

　　　　　　　　　　warning（仅有部分重要、关键信息才会被记录）

　　④、logfile ：配置log文件地址,默认打印在命令行终端的窗口上

　　⑤、databases：设置数据库的数目。默认的数据库是DB 0 ，可以在每个连接上使用select <dbid> 命令选择一个不同的数据库，dbid是一个介于0到databases - 1 之间的数值。默认值是 16，也就是说默认Redis有16个数据库。

①、save：这里是用来配置触发 Redis的持久化条件，也就是什么时候将内存中的数据保存到硬盘。默认如下配置：

```
save 900 1：表示900 秒内如果至少有 1 个 key 的值变化，则保存
save 300 10：表示300 秒内如果至少有 10 个 key 的值变化，则保存
save 60 10000：表示60 秒内如果至少有 10000 个 key 的值变化，则保存
```

　　　　当然如果你只是用Redis的缓存功能，不需要持久化，那么你可以注释掉所有的 save 行来停用保存功能。可以直接一个空字符串来实现停用：save ""

　　②、stop-writes-on-bgsave-error ：默认值为yes。当启用了RDB且最后一次后台保存数据失败，Redis是否停止接收数据。这会让用户意识到数据没有正确持久化到磁盘上，否则没有人会注意到灾难（disaster）发生了。如果Redis重启了，那么又可以重新开始接收数据了

　　③、rdbcompression ；默认值是yes。对于存储到磁盘中的快照，可以设置是否进行压缩存储。如果是的话，redis会采用LZF算法进行压缩。如果你不想消耗CPU来进行压缩的话，可以设置为关闭此功能，但是存储在磁盘上的快照会比较大。

　　④、rdbchecksum ：默认值是yes。在存储快照后，我们还可以让redis使用CRC64算法来进行数据校验，但是这样做会增加大约10%的性能消耗，如果希望获取到最大的性能提升，可以关闭此功能。

　　⑤、dbfilename ：设置快照的文件名，默认是 dump.rdb

　　⑥、dir：设置快照文件的存放路径，这个配置项一定是个目录，而不能是文件名。使用上面的 dbfilename 作为保存的文件名。

①、slave-serve-stale-data：默认值为yes。当一个 slave 与 master 失去联系，或者复制正在进行的时候，slave 可能会有两种表现：

 　　　　1) 如果为 yes ，slave 仍然会应答客户端请求，但返回的数据可能是过时，或者数据可能是空的在第一次同步的时候 

 　　　　2) 如果为 no ，在你执行除了 info he salveof 之外的其他命令时，slave 都将返回一个 "SYNC with master in progress" 的错误

　　②、slave-read-only：配置Redis的Slave实例是否接受写操作，即Slave是否为只读Redis。默认值为yes。

　　③、repl-diskless-sync：主从数据复制是否使用无硬盘复制功能。默认值为no。

　　④、repl-diskless-sync-delay：当启用无硬盘备份，服务器等待一段时间后才会通过套接字向从站传送RDB文件，这个等待时间是可配置的。 这一点很重要，因为一旦传送开始，就不可能再为一个新到达的从站服务。从站则要排队等待下一次RDB传送。因此服务器等待一段 时间以期更多的从站到达。延迟时间以秒为单位，默认为5秒。要关掉这一功能，只需将它设置为0秒，传送会立即启动。默认值为5。

　　⑤、repl-disable-tcp-nodelay：同步之后是否禁用从站上的TCP_NODELAY 如果你选择yes，redis会使用较少量的TCP包和带宽向从站发送数据。但这会导致在从站增加一点数据的延时。 Linux内核默认配置情况下最多40毫秒的延时。如果选择no，从站的数据延时不会那么多，但备份需要的带宽相对较多。默认情况下我们将潜在因素优化，但在高负载情况下或者在主从站都跳的情况下，把它切换为yes是个好主意。默认值为no。

　①、rename-command：命令重命名，对于一些危险命令例如：

　　　　flushdb（清空数据库）

　　　　flushall（清空所有记录）

　　　　config（客户端连接后可配置服务器）

　　　　keys（客户端连接后可查看所有存在的键）          

　　作为服务端redis-server，常常需要禁用以上命令来使得服务器更加安全，禁用的具体做法是是：

- rename-command FLUSHALL ""

也可以保留命令但是不能轻易使用，重命名这个命令即可：

- rename-command FLUSHALL abcdefg

　　这样，重启服务器后则需要使用新命令来执行操作，否则服务器会报错unknown command。

　　②、requirepass:设置redis连接密码

　　比如: requirepass 123 表示redis的连接密码为123.

①、maxclients ：设置客户端最大并发连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件。 描述符数-32（redis server自身会使用一些），如果设置 maxclients为0 。表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息

①、maxmemory：设置Redis的最大内存，如果设置为0 。表示不作限制。通常是配合下面介绍的maxmemory-policy参数一起使用。

　　②、maxmemory-policy ：当内存使用达到maxmemory设置的最大值时，redis使用的内存清除策略。有以下几种可以选择：

　　　　1）volatile-lru  利用LRU算法移除设置过过期时间的key (LRU:最近使用 Least Recently Used ) 

　　　　2）allkeys-lru  利用LRU算法移除任何key 

　　　　3）volatile-random 移除设置过过期时间的随机key 

　　　　4）allkeys-random 移除随机ke

　　　　5）volatile-ttl  移除即将过期的key(minor TTL) 

　　　　6）noeviction noeviction  不移除任何key，只是返回一个写错误 ，默认选项

 　③、maxmemory-samples ：LRU 和 minimal TTL 算法都不是精准的算法，但是相对精确的算法(为了节省内存)。随意你可以选择样本大小进行检，redis默认选择3个样本进行检测，你可以通过maxmemory-samples进行设置样本数。

①、appendonly：默认redis使用的是rdb方式持久化，这种方式在许多应用中已经足够用了。但是redis如果中途宕机，会导致可能有几分钟的数据丢失，根据save来策略进行持久化，Append Only File是另一种持久化方式， 可以提供更好的持久化特性。Redis会把每次写入的数据在接收后都写入appendonly.aof文件，每次启动时Redis都会先把这个文件的数据读入内存里，先忽略RDB文件。默认值为no。

　　②、appendfilename ：aof文件名，默认是"appendonly.aof"

　　③、appendfsync：aof持久化策略的配置；no表示不执行fsync，由操作系统保证数据同步到磁盘，速度最快；always表示每次写入都执行fsync，以保证数据同步到磁盘；everysec表示每秒执行一次fsync，可能会导致丢失这1s数据

　　④、no-appendfsync-on-rewrite：在aof重写或者写入rdb文件的时候，会执行大量IO，此时对于everysec和always的aof模式来说，执行fsync会造成阻塞过长时间，no-appendfsync-on-rewrite字段设置为默认设置为no。如果对延迟要求很高的应用，这个字段可以设置为yes，否则还是设置为no，这样对持久化特性来说这是更安全的选择。  设置为yes表示rewrite期间对新写操作不fsync,暂时存在内存中,等rewrite完成后再写入，默认为no，建议yes。Linux的默认fsync策略是30秒。可能丢失30秒数据。默认值为no。

　　⑤、auto-aof-rewrite-percentage：默认值为100。aof自动重写配置，当目前aof文件大小超过上一次重写的aof文件大小的百分之多少进行重写，即当aof文件增长到一定大小的时候，Redis能够调用bgrewriteaof对日志文件进行重写。当前AOF文件大小是上次日志重写得到AOF文件大小的二倍（设置为100）时，自动启动新的日志重写过程。

　　⑥、auto-aof-rewrite-min-size：64mb。设置允许重写的最小aof文件大小，避免了达到约定百分比但尺寸仍然很小的情况还要重写。

　　⑦、aof-load-truncated：aof文件可能在尾部是不完整的，当redis启动的时候，aof文件的数据被载入内存。重启可能发生在redis所在的主机操作系统宕机后，尤其在ext4文件系统没有加上data=ordered选项，出现这种现象 redis宕机或者异常终止不会造成尾部不完整现象，可以选择让redis退出，或者导入尽可能多的数据。如果选择的是yes，当截断的aof文件被导入的时候，会自动发布一个log给客户端然后load。如果是no，用户必须手动redis-check-aof修复AOF文件才可以。默认值为 yes。

①、lua-time-limit：一个lua脚本执行的最大时间，单位为ms。默认值为5000.

①、cluster-enabled：集群开关，默认是不开启集群模式。

　　②、cluster-config-file：集群配置文件的名称，每个节点都有一个集群相关的配置文件，持久化保存集群的信息。 这个文件并不需要手动配置，这个配置文件有Redis生成并更新，每个Redis集群节点需要一个单独的配置文件。请确保与实例运行的系统中配置文件名称不冲突。默认配置为nodes-6379.conf。

　　③、cluster-node-timeout ：可以配置值为15000。节点互连超时的阀值，集群节点超时毫秒数

　　④、cluster-slave-validity-factor ：可以配置值为10。在进行故障转移的时候，全部slave都会请求申请为master，但是有些slave可能与master断开连接一段时间了， 导致数据过于陈旧，这样的slave不应该被提升为master。该参数就是用来判断slave节点与master断线的时间是否过长。判断方法是：比较slave断开连接的时间和(node-timeout * slave-validity-factor) + repl-ping-slave-period   如果节点超时时间为三十秒, 并且slave-validity-factor为10,假设默认的repl-ping-slave-period是10秒，即如果超过310秒slave将不会尝试进行故障转移

　　⑤、cluster-migration-barrier ：可以配置值为1。master的slave数量大于该值，slave才能迁移到其他孤立master上，如这个参数若被设为2，那么只有当一个主节点拥有2 个可工作的从节点时，它的一个从节点会尝试迁移。

　　⑥、cluster-require-full-coverage：默认情况下，集群全部的slot有节点负责，集群状态才为ok，才能提供服务。 设置为no，可以在slot没有全部分配的时候提供服务。不建议打开该配置，这样会造成分区的时候，小分区的master一直在接受写请求，而造成很长时间数据不一致。

**Redis性能测试**

之前说过Redis在make之后有一个redis-benchmark，这个就是Redis提供用于做性能测试的，它可以用来模拟N个客户端同时发出M个请求。首先看一下redis-benchmark自带的一些参数：

| 参数  | 作用                                                         | 默认值          |
| ----- | ------------------------------------------------------------ | --------------- |
| -h    | 服务器名称                                                   | 127.0.0.1       |
| -p    | 服务器端口                                                   | 6379            |
| -s    | 服务器Socket                                                 | 无              |
| -c    | 并行连接数                                                   | 50              |
| -n    | 请求书                                                       | 10000           |
| -d    | SET/GET值的字节大小                                          | 2               |
| -k    | 1表示keep alive，0表示重连                                   | 1               |
| -r    | SET/GET/INC使用随机Key而不是常量，在形式上key样子为mykey_ran:000000012456-r的值决定了value的最大值 | 无              |
| -p    | 使用管道请求                                                 | 1，即不使用管道 |
| -q    | 安静模式，只显示query/sec值                                  | 无              |
| --csv | 使用csv格式输出                                              | 无              |
| -l    | 循环，无限运行测试                                           | 无              |
| -t    | 只运行使用逗号分割的命令的测试                               | 无              |
| -I    | 空闲模式，只打开N个空闲线程并且等待                          | 无              |



## 数据类型

Redis 不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。

- 字符串（String）
- 哈希（Hash）
- 列表（List）
- 集合（Set）
- 有序集合（Sorted Set）
- 位图 ( Bitmaps )
- 基数统计 ( HyperLogLogs )
- 地理空间信息(Geo)
- 流（Stream）

![Redis-基本数据类型](png/Redis/Redis-基本数据类型.png)

### String(字符串)

string 是Redis的最基本的数据类型，是简单的 key-value 类型，一个key 对应一个 value。value 不仅可以是 String，也可以是数字（当数字类型用 Long 可以表示的时候encoding 就是整型，其他都存储在 sdshdr 当做字符串）。string 类型是二进制安全的，意思是 Redis 的 string 可以包含任何数据，比如图片或者序列化的对象，一个 redis 中字符串 value 最多可以是 512M。

#### String命令

下面用表格来看一下String操作的相关命令：

在这之外，专门提两点：

- Redis的命令不区分大小写
- Redis的Key区分大小写

| **命令** | **描述**                                                     | **用法**                                              |
| -------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| SET      | （1）将字符串值Value关联到Key（2）Key已关联则覆盖，无视类型（3）原本Key带有生存时间TTL，那么TTL被清除 | SET key value [EX seconds] [PX milliseconds] [NX\|XX] |
| GET      | （1）返回key关联的字符串值（2）Key不存在返回nil（3）Key存储的不是字符串，返回错误，因为GET只用于处理字符串 | GET key                                               |
| MSET     | （1）同时设置一个或多个Key-Value键值对（2）某个给定Key已经存在，那么MSET新值会覆盖旧值（3）如果上面的覆盖不是希望的，那么使用MSETNX命令，所有Key都不存在才会进行覆盖（4）**MSET是一个原子性操作**，所有Key都会在同一时间被设置，不会存在有些更新有些没更新的情况 | MSET key value [key value ...]                        |
| MGET     | （1）返回一个或多个给定Key对应的Value（2）某个Key不存在那么这个Key返回nil | MGET key [key ...]                                    |
| SETEX    | （1）将Value关联到Key（2）设置Key生存时间为seconds，单位为秒（3）如果Key对应的Value已经存在，则覆盖旧值（4）SET也可以设置失效时间，但是不同在于SETNX是一个原子操作，即关联值与设置生存时间同一时间完成 | SETEX key seconds value                               |
| SETNX    | （1）将Key的值设置为Value，当且仅当Key不存在（2）若给定的Key已经存在，SEXNX不做任何动作 | SETNX key value                                       |

　　①、上面的 ttl 命令是返回 key 的剩余过期时间，单位为秒。

　　②、mset和mget这种批量处理命令，能够极大的提高操作效率。因为一次命令执行所需要的时间=1次网络传输时间+1次命令执行时间，n个命令耗时=n次网络传输时间+n次命令执行时间，而批量处理命令会将n次网络时间缩减为1次网络时间，也就是1次网络传输时间+n次命令处理时间。

　　但是需要注意的是，Redis是单线程的，如果一次批量处理命令过多，会造成Redis阻塞或网络拥塞（传输数据量大）。

　　③、setnx可以用于实现分布式锁，具体实现方式后面会介绍。

#### **INCR/DECR**命令

前面介绍的是基本的Key-Value操作，下面介绍一种特殊的Key-Value操作即INCR/DECR，可以利用Redis自动帮助我们对一个Key对应的Value进行加减，用表格看一下相关命令：

| **命令** | **描述**                                                     | **用法**             |
| -------- | ------------------------------------------------------------ | -------------------- |
| INCR     | （1）Key中存储的数字值+1，返回增加之后的值（2）Key不存在，那么Key的值被初始化为0再执行INCR（3）如果值包含错误类型或者字符串不能被表示为数字，那么返回错误（4）值限制在64位有符号数字表示之内，即-9223372036854775808~9223372036854775807 | INCR key             |
| DECR     | （1）Key中存储的数字值-1（2）其余同INCR                      | DECR key             |
| INCRBY   | （1）将key所存储的值加上增量返回增加之后的值（2）其余同INCR  | INCRBY key increment |
| DECRBY   | （1）将key所存储的值减去减量decrement（2）其余同INCR         | DECRBY key decrement |

INCR/DECR在实际工作中还是非常管用的，

#### **典型使用场景**

- 计数：由于Redis单线程的特点，我们不用考虑并发造成计数不准的问题，通过 incrby 命令，我们可以正确的得到我们想要的结果。
- 限制次数：比如登录次数校验，错误超过三次5分钟内就不让登录了，每次登录设置key自增一次，并设置该key的过期时间为5分钟后，每次登录检查一下该key的值来进行限制登录。
- 秒杀库存：由于Redis本身极高的读写性能，一些秒杀的场景库存增减可以基于Redis来做而不是直接操作DB

### Hash(字典)

![Redis-Hash](png/Redis/Redis-Hash.png)

Hash 是一个键值对集合，是一个 string 类型的 key和 value 的映射表，key 还是key，但是value是一个键值对（key-value）。类比于 Java里面的 Map<String,Map<String,Object>> 集合。

#### Hash命令

接着我们用表格看一下Hash数据结构的相关命令：

| **命令** | **描述**                                                     | **用法**                                |
| -------- | ------------------------------------------------------------ | --------------------------------------- |
| HSET     | （1）将哈希表Key中的域field的值设为value（2）key不存在，一个新的Hash表被创建（3）field已经存在，旧的值被覆盖 | HSET key field value                    |
| HGET     | （1）返回哈希表key中给定域field的值                          | HGET key field                          |
| HDEL     | （1）删除哈希表key中的一个或多个指定域（2）不存在的域将被忽略 | HDEL key filed [field ...]              |
| HEXISTS  | （1）查看哈希表key中，给定域field是否存在，存在返回1，不存在返回0 | HEXISTS key field                       |
| HGETALL  | （1）返回哈希表key中，所有的域和值                           | HGETALL key                             |
| HINCRBY  | （1）为哈希表key中的域field加上增量increment（2）其余同INCR命令 | HINCRYBY key filed increment            |
| HKEYS    | （1）返回哈希表key中的所有域                                 | HKEYS key                               |
| HLEN     | （1）返回哈希表key中域的数量                                 | HLEN key                                |
| HMGET    | （1）返回哈希表key中，一个或多个给定域的值（2）如果给定的域不存在于哈希表，那么返回一个nil值 | HMGET key field [field ...]             |
| HMSET    | （1）同时将多个field-value对设置到哈希表key中（2）会覆盖哈希表中已存在的域（3）key不存在，那么一个空哈希表会被创建并执行HMSET操作 | HMSET key field value [field value ...] |
| HVALS    | （1）返回哈希表key中所有的域和值                             | HVALS key                               |

#### **典型使用场景**

- 缓存： 经典使用场景，把常用信息，字符串，图片或者视频等信息放到redis中，redis作为缓存层，mysql做持久化层，降低mysql的读写压力
- 计数器：redis是单线程模型，一个命令执行完才会执行下一个，同时数据可以一步落地到其他的数据源
- session：常见方案spring session + redis实现session共享

### List(列表)

![Redis-List](png\Redis\Redis-List.png)

list 列表，它是简单的字符串列表，链表（redis 使用双端链表实现的 List）。按照插入顺序排序，你可以添加一个元素到列表的头部（左边）或者尾部（右边），它的底层实际上是个链表。

列表有两个特点：

　　**一、有序**

　　**二、可以重复**

#### **List命令**

接着我们看一下Redis中的List，相关命令有：

| **命令**  | **描述**                                                     | **用法**                              |
| --------- | ------------------------------------------------------------ | ------------------------------------- |
| LPUSH     | （1）将一个或多个值value插入到列表key的表头（2）如果有多个value值，那么各个value值按从左到右的顺序依次插入表头（3）key不存在，一个空列表会被创建并执行LPUSH操作（4）key存在但不是列表类型，返回错误 | LPUSH key value [value ...]           |
| LPUSHX    | （1）将值value插入到列表key的表头，当且晋档key存在且为一个列表（2）key不存在时，LPUSHX命令什么都不做 | LPUSHX key value                      |
| LPOP      | （1）移除并返回列表key的头元素                               | LPOP key                              |
| LRANGE    | （1）返回列表key中指定区间内的元素，区间以偏移量start和stop指定（2）start和stop都以0位底（3）可使用负数下标，-1表示列表最后一个元素，-2表示列表倒数第二个元素，以此类推（4）start大于列表最大下标，返回空列表（5）stop大于列表最大下标，stop=列表最大下标 | LRANGE key start stop                 |
| LREM      | （1）根据count的值，移除列表中与value相等的元素（2）count>0表示从头到尾搜索，移除与value相等的元素，数量为count（3）count<0表示从从尾到头搜索，移除与value相等的元素，数量为count（4）count=0表示移除表中所有与value相等的元素 | LREM key count value                  |
| LSET      | （1）将列表key下标为index的元素值设为value（2）index参数超出范围，或对一个空列表进行LSET时，返回错误 | LSET key index value                  |
| LINDEX    | （1）返回列表key中，下标为index的元素                        | LINDEX key index                      |
| LINSERT   | （1）将值value插入列表key中，位于pivot前面或者后面（2）pivot不存在于列表key时，不执行任何操作（3）key不存在，不执行任何操作 | LINSERT key BEFORE\|AFTER pivot value |
| LLEN      | （1）返回列表key的长度（2）key不存在，返回0                  | LLEN key                              |
| LTRIM     | （1）对一个列表进行修剪，让列表只返回指定区间内的元素，不存在指定区间内的都将被移除 | LTRIM key start stop                  |
| RPOP      | （1）移除并返回列表key的尾元素                               | RPOP key                              |
| RPOPLPUSH | 在一个原子时间内，执行两个动作：（1）将列表source中最后一个元素弹出并返回给客户端（2）将source弹出的元素插入到列表desination，作为destination列表的头元素 | RPOPLPUSH source destination          |
| RPUSH     | （1）将一个或多个值value插入到列表key的表尾                  | RPUSH key value [value ...]           |
| RPUSHX    | （1）将value插入到列表key的表尾，当且仅当key存在并且是一个列表（2）key不存在，RPUSHX什么都不做 | RPUSHX key value                      |

理解Redis的List了，**操作List千万注意区分LPUSH、RPUSH两个命令，把数据添加到表头和把数据添加到表尾是完全不一样的两种结果**。

另外List还有BLPOP、BRPOP、BRPOPLPUSH三个命令没有说，它们是几个POP的阻塞版本，**即没有数据可以弹出的时候将阻塞客户端直到超时或者发现有可以弹出的元素为止**。

#### **典型使用场景**

- Stack(栈)：通过命令 lpush+lpop
- Queue（队列）：命令 lpush+rpop
- Capped Collection（有限集合）：命令 lpush+ltrim
- Message Queue（消息队列）：命令 lpush+brpop 消息队列，可以利用 List 的 *PUSH 操作，将任务存在 List 中，然后工作线程再用 POP 操作将任务取出进行执行。
-  TimeLine：使用 List 结构，我们可以轻松地实现最新消息排行等功能（比如新浪微博的 TimeLine ）。例如微博的时间轴，有人发布微博，用lpush加入时间轴，展示新的列表信息。

### Set(集合)

![Redis-Set](png/Redis/Redis-Set.png)

Set 就是一个集合，集合的概念就是一堆不重复值的组合。利用 Redis 提供的 Set 数据结构，可以存储一些集合性的数据。

Redis 的 set 是 String 类型的无序集合。

　　相对于列表，集合也有两个特点：

　　**一、无序**

​       **二、不可重复**

#### Set命令

接着我们看一下SET数据结构的相关操作：

| **命令**    | **描述**                                                     | **用法**                              |
| ----------- | ------------------------------------------------------------ | ------------------------------------- |
| SADD        | （1）将一个或多个member元素加入到key中，已存在在集合的member将被忽略（2）假如key不存在，则只创建一个只包含member元素做成员的集合（3）当key不是集合类型时，将返回一个错误 | SADD key number [member ...]          |
| SCARD       | （1）返回key对应的集合中的元素数量                           | SCARD key                             |
| SDIFF       | （1）返回一个集合的全部成员，该集合是第一个Key对应的集合和后面key对应的集合的差集 | SDIFF key [key ...]                   |
| SDIFFSTORE  | （1）和SDIFF类似，但结果保存到destination集合而不是简单返回结果集（2） destination如果已存在，则覆盖 | SDIFFSTORE destionation key [key ...] |
| SINTER      | （1）返回一个集合的全部成员，该集合是所有给定集合的交集（2）不存在的key被视为空集 | SINTER key [key ...]                  |
| SINTERSTORE | （1）和SINTER类似，但结果保存早destination集合而不是简单返回结果集（2）如果destination已存在，则覆盖（3）destination可以是key本身 | SINTERSTORE destination key [key ...] |
| SISMEMBER   | （1）判断member元素是否key的成员，0表示不是，1表示是         | SISMEMBER key member                  |
| SMEMBERS    | （1）返回集合key中的所有成员（2）不存在的key被视为空集       | SMEMBERS key                          |
| SMOVE       | （1）原子性地将member元素从source集合移动到destination集合（2）source集合中不包含member元素，SMOVE命令不执行任何操作，仅返回0（3）destination中已包含member元素，SMOVE命令只是简单做source集合的member元素移除 | SMOVE source desination member        |
| SPOP        | （1）移除并返回集合中的一个随机元素，如果count不指定那么随机返回一个随机元素（2）count为正数且小于集合元素数量，那么返回一个count个元素的数组且数组中的**元素各不相同**（3）count为正数且大于等于集合元素数量，那么返回整个集合（4）count为负数那么命令返回一个数组，数组中的**元素可能重复多次**，数量为count的绝对值 | SPOP key [count]                      |
| SRANDMEMBER | （1）如果count不指定，那么返回集合中的一个随机元素（2）count同上 | SRANDMEMBER key [count]               |
| SREM        | （1）移除集合key中的一个或多个member元素，不存在的member将被忽略 | SREM key member [member ...]          |
| SUNION      | （1）返回一个集合的全部成员，该集合是所有给定集合的并集（2）不存在的key被视为空集 | SUNION key [key ...]                  |
| SUNIONSTORE | （1）类似SUNION，但结果保存到destination集合而不是简单返回结果集（2）destination已存在，覆盖旧值（3）destination可以是key本身 | SUNION destination key [key ...]      |

#### **典型使用场景**

- 共同好友、二度好友。利用集合的交并集特性，比如在社交领域，我们可以很方便的求出多个用户的共同好友，共同感兴趣的领域等。比如在微博应用中，可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。因为 Redis 非常人性化的为集合提供了求交集、并集、差集等操作，那么就可以非常方便的实现如共同关注、共同喜好、二度好友等功能，对上面的所有集合操作，你还可以使用不同的命令选择将结果返回给客户端还是存集到一个新的集合中。
- 利用唯一性，可以统计访问网站的所有独立 IP
- 好友推荐的时候，根据 tag 求交集，大于某个 threshold 就可以推荐
- 标签（tag）,给用户添加标签，或者用户给消息添加标签，这样有同一标签或者类似标签的可以给推荐关注的事或者关注的人　
- 点赞，或点踩，收藏等，可以放到set中实现

### Sorted Set(有序集合)

![Redis-SortedSet](png/Redis/Redis-SortedSet.png)

有序集合（sorted set ），和上面的set 数据类型一样，也是 string 类型元素的集合，但是它是有序的。和Sets相比，Sorted Sets是将 Set 中的元素增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，比如一个存储全班同学成绩的 Sorted Sets，其集合 value 可以是同学的学号，而 score 就可以是其考试得分，这样在数据插入集合的时候，就已经进行了天然的排序。另外还可以用 Sorted Sets 来做带权重的队列，比如普通消息的 score 为1，重要消息的 score 为2，然后工作线程可以选择按 score 的倒序来获取工作任务。让重要的任务优先执行。

#### SortedSet命令

SortedSet顾名思义，即有序的Set，看下相关命令：

| **命令**         | **描述**                                                     | **用法**                                                  |
| ---------------- | ------------------------------------------------------------ | --------------------------------------------------------- |
| ZADD             | （1）将一个或多个member元素及其score值加入有序集key中（2）如果member已经是有序集的成员，那么更新member对应的score并重新插入member保证member在正确的位置上（3）score可以是整数值或双精度浮点数 | ZADD key score member [[score member] [score member] ...] |
| ZCARD            | （1）返回有序集key的元素个数                                 | ZCARD key                                                 |
| ZCOUNT           | （1） 返回有序集key中，score值>=min且<=max的成员的数量       | ZCOUNT key min max                                        |
| ZRANGE           | （1）返回有序集key中指定区间内的成员，成员位置按score从小到大排序（2）具有相同score值的成员按字典序排列（3）需要成员按score从大到小排列，使用ZREVRANGE命令（4）下标参数start和stop都以0为底，也可以用负数，-1表示最后一个成员，-2表示倒数第二个成员（5）可通过WITHSCORES选项让成员和它的score值一并返回 | ZRANGE key start stop [WITHSCORES]                        |
| ZRANK            | （1）返回有序集key中成员member的排名，有序集成员按score值从小到大排列（2）排名以0为底，即score最小的成员排名为0（3）ZREVRANK命令可将成员按score值从大到小排名 | ZRANK key number                                          |
| ZREM             | （1）移除有序集key中的一个或多个成员，不存在的成员将被忽略（2）当key存在但不是有序集时，返回错误 | ZREM key member [member ...]                              |
| ZREMRANGEBYRANK  | （1）移除有序集key中指定排名区间内的所有成员                 | ZREMRANGEBYRANK key start stop                            |
| ZREMRANGEBYSCORE | （1）移除有序集key中，所有score值>=min且<=max之间的成员      | ZREMRANGEBYSCORE key min max                              |

#### **典型使用场景**

- 带有权重的元素，比如一个游戏的用户得分排行榜。和set数据结构一样，zset也可以用于社交领域的相关业务，并且还可以利用zset 的有序特性，还可以做类似排行榜的业务。
- 比较复杂的数据结构，一般用到的场景不算太多
- 排行榜：有序集合经典使用场景。例如小说视频等网站需要对用户上传的小说视频做排行榜，榜单可以按照用户关注数，更新时间，字数等打分，做排行。

### Redis5.0新数据结构-Stream

Redis的作者在Redis5.0中，放出一个新的数据结构，Stream。Redis Stream 的内部，其实也是一个队列，每一个不同的key，对应的是不同的队列，每个队列的元素，也就是消息，都有一个msgid，并且需要保证msgid是严格递增的。在Stream当中，消息是默认持久化的，即便是Redis重启，也能够读取到消息。那么，stream是如何做到多播的呢？其实非常的简单，与其他队列系统相似，Redis对不同的消费者，也有消费者Group这样的概念，不同的消费组，可以消费同一个消息，对于不同的消费组，都维护一个Idx下标，表示这一个消费群组消费到了哪里，每次进行消费，都会更新一下这个下标，往后面一位进行偏移。

### 总结　　

在 Redis 中，常用的 5 种数据类型和应用场景如下：

- **String：** 缓存、计数器、分布式锁等
- **List：** 链表、队列、微博关注人时间轴列表等
- **Hash：** 用户信息、Hash 表等
- **Set：** 去重、赞、踩、共同好友等
- **Zset：** 访问量排行榜、点击量排行榜等

## **Redis的Key命令**

写完了Redis的数据结构，接着我们看下Redis的Key相关操作：

| **命令**  | **描述**                                                     | **用法**                                                     |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| DEL       | （1）删除给定的一个或多个key（2）不存在的Key将被忽略         | DEL key [key ...]                                            |
| EXISTS    | （1）检查给定key是否存在                                     | EXISTS key                                                   |
| EXPIRE    | （1）为给定key设置生存时间，key过期时它会被自动删除（2）对一个已经指定生存时间的Key设置执行EXPIRE，新的值会代替旧的值 | EXPIRE key seconds                                           |
| EXPIREAT  | （1）同EXPIRE，但此命令指定的是UNIX时间戳，单位为秒          | EXPIRE key timestamp                                         |
| KEYS      | （1）查找所有符合给定模式pattern的key，下面举一下例子（2）KEYS *匹配所有key（3）KEYS h?llo匹配hello、hallo、hxllo等（4）KEYS h*llo匹配hllo、heeeeello等（5）KEYS h[ae]llo匹配hello和hallo（6）特殊符号想当做查找内容经的使用\ | KEYS pattern                                                 |
| MIGRATE   | （1）原子性地将key从当前实例传送到目标实例指定的数据库上（2）原数据库Key删除，新数据库Key增加（3）阻塞进行迁移的两个实例，直到迁移成功、迁移失败、等待超时三个之一发生 | MIGRATE host port key destination-db timeout [COPY] [REPLACE] |
| MOVE      | （1）将当前数据库的key移动到给定数据库的db中（2）执行成功的条件为当前数据库有key，给定数据库没有key | MOVE key db                                                  |
| PERSIST   | （1）移除给定key的生存时间，将key变为持久的                  | PERSIST key                                                  |
| RANDOMKEY | （1）从当前数据库随机返回且不删除一个key，                   | RANDOMKEY                                                    |
| RENAME    | （1）将key改名为newkey（2）当key和newkey相同或key不存在，报错（3）newkey已存在，RENAME将覆盖旧值 | RENAME key newkey                                            |
| TTL       | （1）以秒为单位，返回给定的key剩余生存时间                   | TTL key                                                      |
| PTTL      | （1）以毫秒为单位，返回给定的key剩余生存时间                 | PTTL key                                                     |
| TYPE      | （1）返回key锁存储的值的类型                                 | TYPE key                                                     |

这里特别注意KEYS命令，虽然KEYS命令速度非常快，但是当Redis中百万、千万甚至过亿数据的时候，扫描所有Redis的Key，速度仍然会下降，由于Redis是单线程模型，这将导致后面的命令阻塞直到KEYS命令执行完。

因此**当Redis中存储的数据达到了一定量级（经验值从10W开始就值得注意了）的时候，必须警惕KEYS造成Redis整体性能下降**。

## **系统相关命令**

接着介绍一下部分系统相关命令：

| **命令**         | **描述**                                                     | **用法**                   |
| ---------------- | ------------------------------------------------------------ | -------------------------- |
| BGREWRITEAOF     | （1）手动触发AOF重写操作，用于减小AOF文件体积                | BGREWRITEAOF               |
| BGSAVE           | （1）后台异步保存当前数据库的数据到磁盘                      | BGSAVE                     |
| CLIENT KILL      | （1）关闭地址为ip:port的客户端（2）由于Redis为单线程设计，因此当当前命令执行完之后才会关闭客户端 | CLIENT KILL ip:port        |
| CLIENT LIST      | （1）以可读的格式，返回所有连接到服务器的客户端信息和统计数据 | CLIENT LIST                |
| CONFIG GET       | （1）取得运行中的Redis服务器配置参数（2）支持*               | CONFIG GET parameter       |
| CONFIG RESETSTAT | （1）重置INFO命令中的某些统计数据，例如Keyspace hits、Keyspace misses等 | CONFIG RESETSTAT           |
| CONFIG REWRITE   | （1）对**启动Redis时指定的redis.conf文件进行改写**           | CONFIG REWRITE             |
| CONFIG SET       | （1）动态调整Redis服务器的配置而无需重启（2）修改后的配置**立即生效** | CONFIG SET parameter value |
| SELECT           | （1）切换到指定数据库，数据库索引index用数字指定，以0作为起始索引值（2）默认使用0号数据库 | SELECT index               |
| DBSIZE           | （1）返回当前数据库的Key的数量                               | DBSIZE                     |
| DEBUG OBJECT     | （1）这是一个调试命令，不应当被客户端使用（2）key存在时返回有关信息，key不存在时返回错误 | DEBUG OBJECT key           |
| FLUSHALL         | （1）清空整个Redis服务器的数据                               | FLUSHALL                   |
| FLUSHDB          | （1）清空当前数据库中的所有数据                              | FLUSHDB                    |
| INFO             | （1）以一种易于解释且易于阅读的格式，返回Redis服务器的各种信息和统计数值（2）通过给定可选参数section，可以让命令只返回某一部分信息 | INFO [section]             |
| LASTSAVE         | （1）返回最近一次Redis成功将数据保存到磁盘上的时间，以UNIX时间戳格式表示 | LASTSAVE                   |
| MONITOR          | （1）实时打印出Redis服务器接收到的命令，调试用               | MONITOR                    |
| SHUTDOWN         | （1）停止所有客户端（2）如果至少有一个保存点在等待，执行SAVE命令（3）如果AOF选项被打开，更新AOF文件（4）关闭Redis服务器 | SHUTDOWN [SAVE\|NOSAVE]    |

## **Redis的事务**

最后，本文简单说一下Redis的事务机制，首先Redis的事务是由DISCARD、EXEC、MULTI、UNWATCH、WATCH五个命令来保证的：

| 命令    | 描述                                                         | 用法                |
| ------- | ------------------------------------------------------------ | ------------------- |
| DISCARD | （1）取消事务（2）如果正在使用WATCH命令监视某个/某些key，那么取消所有监视，等同于执行UNWATCH | DISCARD             |
| EXEC    | （1）执行所有事务块内的命令（2）如果某个/某些key正处于WATCH命令监视之下且事务块中有和这个/这些key相关的命令，那么**EXEC命令只在这个/这些key没有被其他命令改动的情况下才会执行并生效**，否则该事务被打断 | EXEC                |
| MULTI   | （1）标记一个事务块的开始（2）事务块内的多条命令会按照先后顺序被放入一个队列中，最后**由EXEC命令原子性地执行** | MULTI               |
| UNWATCH | （1）取消WATCH命令对所有key的监视（2）如果WATCH之后，EXEC/DISCARD命令先被执行了，UNWATCH命令就没必要执行了 | UNWATCH             |
| WATCH   | （1）监视一个/多个key，如果在事务执行之前这个/这些key被其他命令改动，那么事务将被打断 | WATCH key [key ...] |

看到开启事务之后，所有的命令返回的都是QUEUED，即放入队列，而不是直接执行。

接着简单说一下事务，和数据库类似的，事务保证的是两点：

- **隔离**，所有命令序列化、按顺序执行，事务执行过程中不会被其他客户端发来的命令打断
- **原子性**，事务中的命令要么全部执行，要么全部不执行

另外，Redis的事务并不支持回滚，这个其实网上已经说法挺多了，大致上是两个原因：

- Redis命令只会因为语法而失败（且这些问题不能再入队时被发现），或是命令用在了错误类型的键上面，也就是说，从实用性角度来说，失败的命令是由于编程错误造成的，而这些错误应该在开发的过程中被发现而不应该出现在生产环境中
- Redis内部可以保持简单且快速，因为不需要对回滚进行支持

总而言之，对Redis来说，回滚无法解决编程错误带来的问题，因此还不如更简单、更快速地无回滚处理事务。

## 特殊数据结构

### HyperLogLog(基数统计)

HyperLogLog 主要的应用场景就是进行基数统计。实际上不会存储每个元素的值，它使用的是概率算法，通过存储元素的hash值的第一个1的位置，来计算元素数量。HyperLogLog 可用极小空间完成独立数统计。命令如下：

| 命令                          | 作用                    |
| ----------------------------- | ----------------------- |
| pfadd key element ...         | 将所有元素添加到key中   |
| pfcount key                   | 统计key的估算值(不精确) |
| pgmerge new_key key1 key2 ... | 合并key至新key          |

#### **典型使用场景**

如何统计 Google 主页面每天被多少个不同的账户访问过？

对于 Google 这种访问量巨大的网页而言，其实统计出有十亿的访问量或十亿零十万的访问量其实是没有太多的区别的，因此，在这种业务场景下，为了节省成本，其实可以只计算出一个大概的值，而没有必要计算出精准的值。

对于上面的场景，可以使用`HashMap`、`BitMap`和`HyperLogLog`来解决。对于这三种解决方案，这边做下对比：

- `HashMap`：算法简单，统计精度高，对于少量数据建议使用，但是对于大量的数据会占用很大内存空间
- `BitMap`：位图算法，具体内容可以参考我的这篇，统计精度高，虽然内存占用要比`HashMap`少，但是对于大量数据还是会占用较大内存
- `HyperLogLog`：存在一定误差，占用内存少，稳定占用 12k 左右内存，可以统计 2^64 个元素，对于上面举例的应用场景，建议使用

### Geo(地理空间信息)

Geo主要用于存储地理位置信息，并对存储的信息进行操作（添加、获取、计算两位置之间距离、获取指定范围内位置集合、获取某地点指定范围内集合）。Redis支持将Geo信息存储到有序集合(zset)中，再通过Geohash算法进行填充。命令如下：

| 命令                                 | 作用                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| geoadd key latitude longitude member | 添加成员位置(纬度、经度、名称)到key中                        |
| geopos key member ...                | 获取成员geo坐标                                              |
| geodist key member1 member2 [unit]   | 计算成员位置间距离。若两个位置之间的其中一个不存在， 那返回空值 |
| georadius                            | 基于经纬度坐标范围查询                                       |
| georadiusbymember                    | 基于成员位置范围查询                                         |
| geohash                              | 计算经纬度hash                                               |



**GEORADIUS**

```properties
GEORADIUS key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count]
```

以给定的经纬度为中心， 返回键包含的位置元素当中， 与中心的距离不超过给定最大距离的所有位置元素。范围可以使用以下其中一个单位：

- **m** 表示单位为米
- **km** 表示单位为千米
- **mi** 表示单位为英里
- **ft** 表示单位为英尺

在给定以下可选项时， 命令会返回额外的信息：

- `WITHDIST`: 在返回位置元素的同时， 将位置元素与中心之间的距离也一并返回。 距离单位和范围单位保持一致
- `WITHCOORD`: 将位置元素的经度和维度也一并返回
- `WITHHASH`: 以 52 位有符号整数的形式， 返回位置元素经过原始 geohash 编码的有序集合分值。 这个选项主要用于底层应用或者调试， 实际中的作用并不大

命令默认返回未排序的位置元素。 通过以下两个参数， 用户可以指定被返回位置元素的排序方式：

- `ASC`: 根据中心的位置， 按照从近到远的方式返回位置元素
- `DESC`: 根据中心的位置， 按照从远到近的方式返回位置元素

在默认情况下， GEORADIUS 命令会返回所有匹配的位置元素。 虽然用户可以使用 **COUNT `<count>`** 选项去获取前 N 个匹配元素， 但是因为命令在内部可能会需要对所有被匹配的元素进行处理， 所以在对一个非常大的区域进行搜索时， 即使只使用 `COUNT` 选项去获取少量元素， 命令的执行速度也可能会非常慢。 但是从另一方面来说， 使用 `COUNT` 选项去减少需要返回的元素数量， 对于减少带宽来说仍然是非常有用的。



### Pub/Sub(发布订阅)

发布订阅类似于广播功能。redis发布订阅包括 发布者、订阅者、Channel。常用命令如下：

| 命令                    | 作用                               | 时间复杂度                                                   |
| ----------------------- | ---------------------------------- | ------------------------------------------------------------ |
| subscribe channel       | 订阅一个频道                       | O(n)                                                         |
| unsubscribe channel ... | 退订一个/多个频道                  | O(n)                                                         |
| publish channel msg     | 将信息发送到指定的频道             | O(n+m)，n 是频道 channel 的订阅者数量, M 是使用模式订阅(subscribed patterns)的客户端的数量 |
| pubsub CHANNELS         | 查看订阅与发布系统状态(多种子模式) | O(n)                                                         |
| psubscribe              | 订阅多个频道                       | O(n)                                                         |
| unsubscribe             | 退订多个频道                       | O(n)                                                         |



### Bitmap(位图)

Bitmap就是位图，其实也就是字节数组（byte array），用一串连续的2进制数字（0或1）表示，每一位所在的位置为偏移(offset)，位图就是用每一个二进制位来存放或者标记某个元素对应的值。通常是用来判断某个数据存不存在的，因为是用bit为单位来存储所以Bitmap本身会极大的节省储存空间。常用命令如下：

| 命令                        | 作用                                                | 时间复杂度 |
| --------------------------- | --------------------------------------------------- | ---------- |
| setbit key offset val       | 给指定key的值的第offset赋值val                      | O(1)       |
| getbit key offset           | 获取指定key的第offset位                             | O(1)       |
| bitcount key start end      | 返回指定key中[start,end]中为1的数量                 | O(n)       |
| bitop operation destkey key | 对不同的二进制存储数据进行位运算(AND、OR、NOT、XOR) | O(n)       |



**应用案例**

有1亿用户，5千万登陆用户，那么统计每日用户的登录数。每一位标识一个用户ID，当某个用户访问我们的网站就在Bitmap中把标识此用户的位设置为1。使用set集合和Bitmap存储的对比：

| 数据类型 | 每个 userid 占用空间                                         | 需要存储的用户量 | 全部占用内存量               |
| -------- | ------------------------------------------------------------ | ---------------- | ---------------------------- |
| set      | 32位也就是4个字节（假设userid用的是整型，实际很多网站用的是长整型） | 50,000,000       | 32位 * 50,000,000 = 200 MB   |
| Bitmap   | 1 位（bit）                                                  | 100,000,000      | 1 位 * 100,000,000 = 12.5 MB |



**应用场景**

- 用户在线状态
- 用户签到状态
- 统计独立用户



### BloomFilter(布隆过滤)

![Redis-BloomFilter](C:/Users/DELL/Downloads/lemon-guide-main/images/Middleware/Redis-BloomFilter.jpg)

当一个元素被加入集合时，通过K个散列函数将这个元素映射成一个位数组中的K个点（使用多个哈希函数对**元素key (bloom中不存value)** 进行哈希，算出一个整数索引值，然后对位数组长度进行取模运算得到一个位置，每个无偏哈希函数都会得到一个不同的位置），把它们置为1。检索时，我们只要看看这些点是不是都是1就（大约）知道集合中有没有它了：

- 如果这些点有任何一个为0，则被检元素一定不在
- 如果都是1，并不能完全说明这个元素就一定存在其中，有可能这些位置为1是因为其他元素的存在，这就是布隆过滤器会出现误判的原因



**应用场景**

- **解决缓存穿透**：事先把存在的key都放到redis的**Bloom Filter** 中，他的用途就是存在性检测，如果 BloomFilter 中不存在，那么数据一定不存在；如果 BloomFilter 中存在，实际数据也有可能会不存
- **黑名单校验**：假设黑名单的数量是数以亿计的，存放起来就是非常耗费存储空间的，布隆过滤器则是一个较好的解决方案。把所有黑名单都放在布隆过滤器中，再收到邮件时，判断邮件地址是否在布隆过滤器中即可
- **Web拦截器**：用户第一次请求，将请求参数放入布隆过滤器中，当第二次请求时，先判断请求参数是否被布隆过滤器命中，从而提高缓存命中率



#### 基于Bitmap数据结构

```java
import com.google.common.base.Preconditions;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.util.Collection;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@Service
public class RedisService {

    @Resource
    private RedisTemplate<String, Object> redisTemplate;

    /**
     * 根据给定的布隆过滤器添加值
     */
    public <T> void addByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
        Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            redisTemplate.opsForValue().setBit(key, i, true);
        }
    }

    /**
     * 根据给定的布隆过滤器判断值是否存在
     */
    public <T> boolean includeByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
        Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            if (!redisTemplate.opsForValue().getBit(key, i)) {
                return false;
            }
        }

        return true;
    }
}
```



#### 基于RedisBloom模块

RedisBloom模块提供了四种数据类型：

- **Bloom Filter (布隆过滤器)**
- **Cuckoo Filter（布谷鸟过滤器）**
- **Count-Mins-Sketch**
- **Top-K**

`Bloom Filter` 和 `Cuckoo` 用于确定（以给定的确定性）集合中是否存在某项。使用 `Count-Min Sketch` 来估算子线性空间中的项目数，使用 `Top-K` 维护K个最频繁项目的列表。

```shell
# 1.git 下载
[root@test ~]# git clone https://github.com/RedisBloom/RedisBloom.git
[root@test ~]# cd redisbloom
[root@test ~]# make

# 2.wget 下载
[root@test ~]# wget https://github.com/RedisBloom/RedisBloom/archive/v2.0.3.tar.gz
[root@test ~]# tar -zxvf RedisBloom-2.0.3.tar.gz
[root@test ~]# cd RedisBloom-2.0.3/
[root@test ~]# make

# 3.修改Redis Conf
[root@test ~]#vim /etc/redis.conf
# 在文件中添加下行
loadmodule /root/RedisBloom-2.0.3/redisbloom.so

# 4.启动Redis server
[root@test ~]# /redis-server /etc/redis.conf
# 或者启动服务时加载os文件
[root@test ~]# /redis-server /etc/redis.conf --loadmodule /root/RedisBloom/redisbloom.so

# 5.测试RedisBloom
[root@test ~]# redis-cli
127.0.0.1:6379> bf.add bloomFilter foo
127.0.0.1:6379> bf.exists bloomFilter foo
127.0.0.1:6379> cf.add cuckooFilter foo
127.0.0.1:6379> cf.exists cuckooFilter foo
```

## 底层数据结构

Redis 数据库里面的每个**键值对（key-value）** 都是由**对象（object）**组成的：

　　　　　　数据库键总是一个字符串对象（string object）;

　　　　　　数据库的值则可以是字符串对象、列表对象（list）、哈希对象（hash）、集合对象（set）、有序集合（sort set）对象这五种对象中的其中一种。

**Redis** 底层数据结构有一下数据类型：

- **简单动态字符串(SDS)**
- **链表(LINKEDLIST)**
- **字典**
- **跳跃表**
- **整数集合**
- **压缩列表**
- **对象**

当然是为了追求速度，不同数据类型使用不同的数据结构速度才得以提升。

![Redis数据类型与底层数据结构关系](png/Redis/Redis数据类型与底层数据结构关系.png)

### 底层类型

| 类型             | 编码                     |
| ---------------- | ------------------------ |
| STRING（字符串） | INT（整型）              |
| STRING（字符串） | EMBSTR（简单动态字符串） |
| STRING（字符串） | RAW（简单动态字符串）    |
| LIST（列表）     | QUICKLIST（快表）        |
| LIST（列表）     | LINKEDLIST（快表）       |
| SET（集合）      | INTSET（整数集合）       |
| SET（集合）      | HT（哈希表）             |
| ZSET（有序集合） | ZIPLIST（压缩列表）      |
| ZSET（有序集合） | SKIPLIST（跳表）         |
| HASH（哈希）     | ZIPLIST（压缩列表）      |
| HASH（哈希）     | HT（哈希表）             |

### SDS(简单动态字符)

![img](png/Redis/Redis-SDS简单动态字符.png)

Redis 是用 C 语言写的，但是对于Redis的字符串，却不是 C 语言中的字符串（即以空字符’\0’结尾的字符数组），它是自己构建了一种名为 简单动态字符串（simple dynamic string,SDS）的抽象类型，并将 SDS 作为 Redis的默认字符串表示。

**SDS 定义：**

```
struct sdshdr{
     //记录buf数组中已使用字节的数量
     //等于 SDS 保存字符串的长度
     int len;
     //记录 buf 数组中未使用字节的数量
     int free;
     //字节数组，用于保存字符串
     char buf[];
}
```

我们看上面对于 SDS 数据类型的定义：

　　1、len 保存了SDS保存字符串的长度

　　2、buf[] 数组用来保存字符串的每个元素

　　3、free j记录了 buf 数组中未使用的字节数量

　　上面的定义相对于 C 语言对于字符串的定义，多出了 len 属性以及 free 属性。为什么不使用C语言字符串实现，而是使用 SDS呢？这样实现有什么好处？

　　**①、常数复杂度获取字符串长度**

　　由于 len 属性的存在，我们获取 SDS 字符串的长度只需要读取 len 属性，时间复杂度为 O(1)。而对于 C 语言，获取字符串的长度通常是经过遍历计数来实现的，时间复杂度为 O(n)。通过 strlen key 命令可以获取 key 的字符串长度。

　　**②、杜绝缓冲区溢出**

　　我们知道在 C 语言中使用 strcat 函数来进行两个字符串的拼接，一旦没有分配足够长度的内存空间，就会造成缓冲区溢出。而对于 SDS 数据类型，在进行字符修改的时候，会首先根据记录的 len 属性检查内存空间是否满足需求，如果不满足，会进行相应的空间扩展，然后在进行修改操作，所以不会出现缓冲区溢出。

　　**③、减少修改字符串的内存重新分配次数**

　　C语言由于不记录字符串的长度，所以如果要修改字符串，必须要重新分配内存（先释放再申请），因为如果没有重新分配，字符串长度增大时会造成内存缓冲区溢出，字符串长度减小时会造成内存泄露。

　　而对于SDS，由于len属性和free属性的存在，对于修改字符串SDS实现了空间预分配和惰性空间释放两种策略：

　　1、空间预分配：对字符串进行空间扩展的时候，扩展的内存比实际需要的多，这样可以减少连续执行字符串增长操作所需的内存重分配次数。

　　2、惰性空间释放：对字符串进行缩短操作时，程序不立即使用内存重新分配来回收缩短后多余的字节，而是使用 free 属性将这些字节的数量记录下来，等待后续使用。（当然SDS也提供了相应的API，当我们有需要时，也可以手动释放这些未使用的空间。）

　　**④、二进制安全**

　　因为C字符串以空字符作为字符串结束的标识，而对于一些二进制文件（如图片等），内容可能包括空字符串，因此C字符串无法正确存取；而所有 SDS 的API 都是以处理二进制的方式来处理 buf 里面的元素，并且 SDS 不是以空字符串来判断是否结束，而是以 len 属性表示的长度来判断字符串是否结束。

　　**⑤、兼容部分 C 字符串函数**

　　虽然 SDS 是二进制安全的，但是一样遵从每个字符串都是以空字符串结尾的惯例，这样可以重用 C 语言库<string.h> 中的一部分函数。

一般来说，SDS 除了保存数据库中的字符串值以外，SDS 还可以作为缓冲区（buffer）：包括 AOF 模块中的AOF缓冲区以及客户端状态中的输入缓冲区。

**总结**

| C 字符串                                   | SDS                                    |
| ------------------------------------------ | -------------------------------------- |
| 获取字符串长度的复杂度为O（N)              | 获取字符串长度的复杂度为O(1)           |
| API 是不安全的，可能会造成缓冲区溢出       | API 是安全的，不会造成缓冲区溢出       |
| 修改字符串长度N次必然需要执行N次内存重分配 | 修改字符串长度N次最多执行N次内存重分配 |
| 只能保存文本数据                           | 可以保存二进制数据和文本文数据         |
| 可以使用所有<String.h>库中的函数           | 可以使用一部分<string.h>库中的函数     |

### linkedList(链表)

![quicklist](png/Redis/Redis-quicklist.png)



链表是一种常用的数据结构，C 语言内部是没有内置这种数据结构的实现，所以Redis自己构建了链表的实现。

链表定义：

```
typedef  struct listNode{
       //前置节点
       struct listNode *prev;
       //后置节点
       struct listNode *next;
       //节点的值
       void *value;  
}listNode
```

通过多个 listNode 结构就可以组成链表，这是一个双向链表，Redis还提供了操作链表的数据结构：

```
typedef struct list{
     //表头节点
     listNode *head;
     //表尾节点
     listNode *tail;
     //链表所包含的节点数量
     unsigned long len;
     //节点值复制函数
     void (*free) (void *ptr);
     //节点值释放函数
     void (*free) (void *ptr);
     //节点值对比函数
     int (*match) (void *ptr,void *key);
}list;
```

![Redis-链表](png\Redis\Redis-链表.png)

Redis链表特性：

　　①、双端：链表具有前置节点和后置节点的引用，获取这两个节点时间复杂度都为O(1)。

　　②、无环：表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL,对链表的访问都是以 NULL 结束。　　

　　③、带链表长度计数器：通过 len 属性获取链表长度的时间复杂度为 O(1)。

　　④、多态：链表节点使用 void* 指针来保存节点值，可以保存各种不同类型的值。

后续版本对列表数据结构进行了改造，使用 quicklist 代替了 ziplist 和 linkedlist。quicklist 是 ziplist 和 linkedlist 的混合体，它将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来。这也是为何 Redis 快的原因，不放过任何一个可以提升性能的细节。

### hash(字典)

![img](png/Redis/Redis全局hash字典.png)　



字典又称为符号表（symbol table）、关联数组（associative array）或映射（map），是一种用于保存键值对的抽象数据结构。字典中的每一个键 key 都是唯一的，通过 key 可以对值来进行查找或修改。C 语言中没有内置这种数据结构的实现，所以字典依然是 Redis自己构建的。Redis 整体就是一个 哈希表来保存所有的键值对，无论数据类型是 5 种的任意一种。哈希表，本质就是一个数组，每个元素被叫做哈希桶，不管什么数据类型，每个桶里面的 entry 保存着实际具体值的指针。

整个数据库就是一个全局哈希表，而哈希表的时间复杂度是 O(1)，只需要计算每个键的哈希值，便知道对应的哈希桶位置，定位桶里面的 entry 找到对应数据，这个也是 Redis 快的原因之一。

Redis 的字典使用哈希表作为底层实现

哈希表结构定义：

```
typedef struct dictht{
     //哈希表数组
     dictEntry **table;
     //哈希表大小
     unsigned long size;
     //哈希表大小掩码，用于计算索引值
     //总是等于 size-1
     unsigned long sizemask;
     //该哈希表已有节点的数量
     unsigned long used;
 
}dictht
```

哈希表是由数组 table 组成，table 中每个元素都是指向 dict.h/dictEntry 结构，dictEntry 结构定义如下：

```
typedef struct dictEntry{
     //键
     void *key;
     //值
     union{
          void *val;
          uint64_tu64;
          int64_ts64;
     }v;
 
     //指向下一个哈希表节点，形成链表
     struct dictEntry *next;
}dictEntry
```

key 用来保存键，val 属性用来保存值，值可以是一个指针，也可以是uint64_t整数，也可以是int64_t整数。

字典

```
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privedata;
    // 哈希表
    dictht  ht[2];
    // rehash 索引
    in trehashidx;

}
```

type 属性 和privdata 属性是针对不同类型的键值对，为创建多态字典而设置的。

ht 属性是一个包含两个项（两个哈希表）的数组

　　注意这里还有一个指向下一个哈希表节点的指针，我们知道哈希表最大的问题是存在哈希冲突，如何解决哈希冲突，有开放地址法和链地址法。这里采用的便是链地址法，通过next这个指针可以将多个哈希值相同的键值对连接在一起，用来解决**哈希冲突**。

![Redis-哈希表](png\Redis\Redis-哈希表.png)

**①、哈希算法：**Redis计算哈希值和索引值方法如下：

```
#1、使用字典设置的哈希函数，计算键 key 的哈希值``hash = dict->type->hashFunction(key);``#2、使用哈希表的sizemask属性和第一步得到的哈希值，计算索引值``index = hash & dict->ht[x].sizemask;
```

　　**②、解决哈希冲突：**这个问题上面我们介绍了，方法是链地址法。通过字典里面的 *next 指针指向下一个具有相同索引值的哈希表节点。

　　**③、扩容和收缩：**当哈希表保存的键值对太多或者太少时，就要通过 rerehash(重新散列）来对哈希表进行相应的扩展或者收缩。具体步骤：

　　　　　　1、如果执行扩展操作，会基于原哈希表创建一个大小等于 ht[0].used*2n 的哈希表（也就是每次扩展都是根据原哈希表已使用的空间扩大一倍创建另一个哈希表）。相反如果执行的是收缩操作，每次收缩是根据已使用空间缩小一倍创建一个新的哈希表。

　　　　　　2、重新利用上面的哈希算法，计算索引值，然后将键值对放到新的哈希表位置上。

　　　　　　3、所有键值对都迁徙完毕后，释放原哈希表的内存空间。

　　**④、触发扩容的条件：**

　　　　　　1、服务器目前没有执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且负载因子大于等于1。

　　　　　　2、服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且负载因子大于等于5。

　　　　ps：负载因子 = 哈希表已保存节点数量 / 哈希表大小。

　　**⑤、渐近式 rehash**

　　　　什么叫渐进式 rehash？也就是说扩容和收缩操作不是一次性、集中式完成的，而是分多次、渐进式完成的。如果保存在Redis中的键值对只有几个几十个，那么 rehash 操作可以瞬间完成，但是如果键值对有几百万，几千万甚至几亿，那么要一次性的进行 rehash，势必会造成Redis一段时间内不能进行别的操作。所以Redis采用渐进式 rehash,这样在进行渐进式rehash期间，字典的删除查找更新等操作可能会在两个哈希表上进行，第一个哈希表没有找到，就会去第二个哈希表上进行查找。但是进行增加操作，一定是在新的哈希表上进行的。

**那 Hash 冲突怎么办？**

当写入 Redis 的数据越来越多的时候，哈希冲突不可避免，会出现不同的 key 计算出一样的哈希值。Redis 通过链式哈希解决冲突：也就是同一个 桶里面的元素使用链表保存。但是当链表过长就会导致查找性能变差可能，所以 Redis 为了追求快，使用了两个全局哈希表。用于 rehash 操作，增加现有的哈希桶数量，减少哈希冲突。开始默认使用 hash 表 1 保存键值对数据，哈希表 2 此刻没有分配空间。

### skipList(跳跃表)

![skipList跳跃表](png/Redis/Redis-skipList跳跃表.png)

sorted set 类型的排序功能便是通过「跳跃列表」数据结构来实现。跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

跳跃表支持平均 O（logN）、最坏 O（N）复杂度的节点查找，还可以通过顺序性操作来批量处理节点。跳表在链表的基础上，增加了多层级索引，通过索引位置的几个跳转，实现数据的快速定位。

Redis 只在两个地方用到了跳跃表，一个是实现**有序集合键**，另外一个是在**集群节点**中用作**内部数据结构**。

跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其它节点的指针，从而达到快速访问节点的目的。具有如下性质：

　　1、由很多层结构组成；

　　2、每一层都是一个有序的链表，排列顺序为由高层到底层，都至少包含两个链表节点，分别是前面的head节点和后面的nil节点；

　　3、最底层的链表包含了所有的元素；

　　4、如果一个元素出现在某一层的链表中，那么在该层之下的链表也全都会出现（上一层的元素是当前层的元素的子集）；

　　5、链表中的每个节点都包含两个指针，一个指向同一层的下一个链表节点，另一个指向下一层的同一个链表节点；

![Redis-跳表](png\Redis\Redis-跳表.png)

Redis中跳跃表节点定义如下：

```
typedef struct zskiplistNode {
     //层
     struct zskiplistLevel{
           //前进指针
           struct zskiplistNode *forward;
           //跨度
           unsigned int span;
     }level[];
 
     //后退指针
     struct zskiplistNode *backward;
     //分值
     double score;
     //成员对象
     robj *obj;
 
} zskiplistNode
```

　　　　1、层：level 数组可以包含多个元素，每个元素都包含一个指向其他节点的指针。

　　　　2、前进指针：用于指向表尾方向的前进指针

　　　　3、跨度：用于记录两个节点之间的距离

　　　　4、后退指针：用于从表尾向表头方向访问节点

　　　　5、分值和成员：跳跃表中的所有节点都按分值从小到大排序。成员对象指向一个字符串，这个字符串对象保存着一个SDS值

多个跳跃表节点构成一个跳跃表：

```
typedef struct zskiplist{
     //表头节点和表尾节点
     structz skiplistNode *header, *tail;
     //表中节点的数量
     unsigned long length;
     //表中层数最大的节点的层数
     int level;
 
}zskiplist;
```

从结构图中我们可以清晰的看到，header，tail分别指向跳跃表的头结点和尾节点。level 用于记录最大的层数，length 用于记录我们的节点数量。      

​      ①、搜索：从最高层的链表节点开始，如果比当前节点要大和比当前层的下一个节点要小，那么则往下找，也就是和当前层的下一层的节点的下一个节点进行比较，以此类推，一直找到最底层的最后一个节点，如果找到则返回，反之则返回空。

　　②、插入：首先确定插入的层数，有一种方法是假设抛一枚硬币，如果是正面就累加，直到遇见反面为止，最后记录正面的次数作为插入的层数。当确定插入的层数k后，则需要将新元素插入到从底层到k层。

　　③、删除：在各个层中找到包含指定值的节点，然后将节点从链表中删除即可，如果删除以后只剩下头尾两个节点，则删除这一层。

### intset(整数数组)

当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis 就会使用整数集合作为集合键的底层实现，节省内存。整数集合（intset）是Redis用于保存整数值的集合抽象数据类型，它可以保存类型为int16_t、int32_t 或者int64_t 的整数值，并且保证集合中不会出现重复元素。

定义如下：

```
typedef struct intset{
     //用于定义整数集合的编码方式
     uint32_t encoding;
     //用于记录整数集合中变量的数量
     uint32_t length;
     //用于保存元素的数组，虽然我们在数据结构图中看到，intset将数组定义为int8_t，但实际上数组保存的元素类型取决于encoding
     int8_t contents[];
 
}intset;
```

整数集合的每个元素都是 contents 数组的一个数据项，它们按照从小到大的顺序排列，并且不包含任何重复项。

　　length 属性记录了 contents 数组的大小。

　　需要注意的是虽然 contents 数组声明为 int8_t 类型，但是实际上contents 数组并不保存任何 int8_t 类型的值，其真正类型有 encoding 来决定。

　　**①、升级**

　　当我们新增的元素类型比原集合元素类型的长度要大时，需要对整数集合进行升级，才能将新元素放入整数集合中。具体步骤：

　　1、根据新元素类型，扩展整数集合底层数组的大小，并为新元素分配空间。

　　2、将底层数组现有的所有元素都转成与新元素相同类型的元素，并将转换后的元素放到正确的位置，放置过程中，维持整个元素顺序都是有序的。

　　3、将新元素添加到整数集合中（保证有序）。

　　升级能极大地节省内存。

　　**②、降级**

　　整数集合不支持降级操作，一旦对数组进行了升级，编码就会一直保持升级后的状态。

### zipList(压缩列表)

![ZipList压缩列表](png/Redis/Redis-ZipList压缩列表.png)

压缩列表是 List 、hash、 sorted Set 三种数据类型底层实现之一。当一个列表只有少量数据的时候，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么 Redis 就会使用压缩列表来做列表键的底层实现。

ziplist 是由一系列特殊编码的连续内存块组成的顺序型的数据结构，ziplist 中可以包含多个 entry 节点，每个节点可以存放整数或者字符串。ziplist 在表头有三个字段 zlbytes、zltail 和 zllen，分别表示列表占用字节数、列表尾的偏移量和列表中的 entry 个数；压缩列表在表尾还有一个 zlend，表示列表结束。

压缩列表（ziplist）是Redis为了节省内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

　　**压缩列表的原理：压缩列表并不是对数据利用某种算法进行压缩，而是将数据按照一定规则编码在一块连续的内存区域，目的是节省内存。**

```go
struct ziplist<T> {
    int32 zlbytes;           // 整个压缩列表占用字节数
    int32 zltail_offset;  // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength;          // 元素个数
    T[] entries;              // 元素内容列表，挨个挨个紧凑存储
    int8 zlend;               // 标志压缩列表的结束，值恒为 0xFF
}
```

如果我们要查找定位第一个元素和最后一个元素，可以通过表头三个字段的长度直接定位，复杂度是 O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是 O(N)。

​		①、previous_entry_ength：记录压缩列表前一个字节的长度。previous_entry_ength的长度可能是1个字节或者是5个字节，如果上一个节点的长度小于254，则该节点只需要一个字节就可以表示前一个节点的长度了，如果前一个节点的长度大于等于254，则previous length的第一个字节为254，后面用四个字节表示当前节点前一个节点的长度。利用此原理即当前节点位置减去上一个节点的长度即得到上一个节点的起始位置，压缩列表可以从尾部向头部遍历。这么做很有效地减少了内存的浪费。

　　②、encoding：节点的encoding保存的是节点的content的内容类型以及长度，encoding类型一共有两种，一种字节数组一种是整数，encoding区域长度为1字节、2字节或者5字节长。

　　③、content：content区域用于保存节点的内容，节点内容类型和长度由encoding决定。

### 快速列表(quicklist)

一个由ziplist组成的双向链表。但是一个quicklist可以有多个quicklist节点，它很像B树的存储方式。是在redis3.2版本中新加的数据结构，用在列表的底层实现。

```
typedef struct quicklist {
    //指向头部(最左边)quicklist节点的指针
    quicklistNode *head;
 
    //指向尾部(最右边)quicklist节点的指针
    quicklistNode *tail;
 
    //ziplist中的entry节点计数器
    unsigned long count;        /* total count of all entries in all ziplists */
 
    //quicklist的quicklistNode节点计数器
    unsigned int len;           /* number of quicklistNodes */
 
    //保存ziplist的大小，配置文件设定，占16bits
    int fill : 16;              /* fill factor for individual nodes */
 
    //保存压缩程度值，配置文件设定，占16bits，0表示不压缩
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```

quicklist节点结构:

```
typedef struct quicklistNode {
    struct quicklistNode *prev;     //前驱节点指针
    struct quicklistNode *next;     //后继节点指针
 
    //不设置压缩数据参数recompress时指向一个ziplist结构
    //设置压缩数据参数recompress指向quicklistLZF结构
    unsigned char *zl;
 
    //压缩列表ziplist的总长度
    unsigned int sz;                  /* ziplist size in bytes */
 
    //ziplist中包的节点数，占16 bits长度
    unsigned int count : 16;          /* count of items in ziplist */
 
    //表示是否采用了LZF压缩算法压缩quicklist节点，1表示压缩过，2表示没压缩，占2 bits长度
    unsigned int encoding : 2;        /* RAW==1 or LZF==2 */
 
    //表示一个quicklistNode节点是否采用ziplist结构保存数据，2表示压缩了，1表示没压缩，默认是2，占2bits长度
    unsigned int container : 2;       /* NONE==1 or ZIPLIST==2 */
 
    //标记quicklist节点的ziplist之前是否被解压缩过，占1bit长度
    //如果recompress为1，则等待被再次压缩
    unsigned int recompress : 1; /* was this node previous compressed? */
 
    //测试时使用
    unsigned int attempted_compress : 1; /* node can't compress; too small */
 
    //额外扩展位，占10bits长度
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

**相关配置**

在redis.conf中的ADVANCED CONFIG部分：

```
list-max-ziplist-size -2
list-compress-depth 0
```

**list-max-ziplist-size参数**

我们来详细解释一下`list-max-ziplist-size`这个参数的含义。它可以取正值，也可以取负值。

当取正值的时候，表示按照数据项个数来限定每个quicklist节点上的ziplist长度。比如，当这个参数配置成5的时候，表示每个quicklist节点的ziplist最多包含5个数据项。

当取负值的时候，表示按照占用字节数来限定每个quicklist节点上的ziplist长度。这时，它只能取-1到-5这五个值，每个值含义如下：

-5: 每个quicklist节点上的ziplist大小不能超过64 Kb。（注：1kb => 1024 bytes）

-4: 每个quicklist节点上的ziplist大小不能超过32 Kb。

-3: 每个quicklist节点上的ziplist大小不能超过16 Kb。

-2: 每个quicklist节点上的ziplist大小不能超过8 Kb。（-2是Redis给出的默认值）

**list-compress-depth参数**

这个参数表示一个quicklist两端不被压缩的节点个数。注：这里的节点个数是指quicklist双向链表的节点个数，而不是指ziplist里面的数据项个数。实际上，一个quicklist节点上的ziplist，如果被压缩，就是整体被压缩的。

参数list-compress-depth的取值含义如下：

0: 是个特殊值，表示都不压缩。这是Redis的默认值。 1: 表示quicklist两端各有1个节点不压缩，中间的节点压缩。 2: 表示quicklist两端各有2个节点不压缩，中间的节点压缩。 3: 表示quicklist两端各有3个节点不压缩，中间的节点压缩。 依此类推…

Redis对于quicklist内部节点的压缩算法，采用的LZF——一种无损压缩算法。

### 总结

大多数情况下，Redis使用简单字符串SDS作为字符串的表示，相对于C语言字符串，SDS具有常数复杂度获取字符串长度，杜绝了缓存区的溢出，减少了修改字符串长度时所需的内存重分配次数，以及二进制安全能存储各种类型的文件，并且还兼容部分C函数。

　　通过为链表设置不同类型的特定函数，Redis链表可以保存各种不同类型的值，除了用作列表键，还在发布与订阅、慢查询、监视器等方面发挥作用（后面会介绍）。

　　Redis的字典底层使用哈希表实现，每个字典通常有两个哈希表，一个平时使用，另一个用于rehash时使用，使用链地址法解决哈希冲突。

　　跳跃表通常是有序集合的底层实现之一，表中的节点按照分值大小进行排序。

　　整数集合是集合键的底层实现之一，底层由数组构成，升级特性能尽可能的节省内存。

　　压缩列表是Redis为节省内存而开发的顺序型数据结构，通常作为列表键和哈希键的底层实现之一。

　　以上介绍的简单字符串、链表、字典、跳跃表、整数集合、压缩列表等数据结构就是Redis底层的一些数据结构，用来实现上一篇博客介绍的Redis五大数据类型，那么每种数据类型是由哪些数据结构实现的呢？

## 五大数据类型实现原理

在Redis中，并没有直接使用这些数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统，这些对象系统也就是前面说的五大数据类型，每一种数据类型都至少用到了一种数据结构。通过这五种不同类型的对象，Redis可以在执行命令之前，根据对象的类型判断一个对象是否可以执行给定的命令，而且可以针对不同的场景，为对象设置多种不同的数据结构，从而优化对象在不同场景下的使用效率。

### 对象的类型与编码

Redis使用前面说的五大数据类型来表示键和值，每次在Redis数据库中创建一个键值对时，至少会创建两个对象，一个是键对象，一个是值对象，而Redis中的每个对象都是由 redisObject 结构来表示：

```
typedef struct redisObject{
     //类型
     unsigned type:4;
     //编码
     unsigned encoding:4;
     //指向底层数据结构的指针
     void *ptr;
     //引用计数
     int refcount;
     //记录最后一次被程序访问的时间
     unsigned lru:22;
 
}robj
```

①、type属性

　　对象的type属性记录了对象的类型，这个类型就是前面讲的五大数据类型：

可以通过如下命令来判断对象类型：

```
type key
```

**注意：在Redis中，键总是一个字符串对象，而值可以是字符串、列表、集合等对象，所以我们通常说的键为字符串键，表示的是这个键对应的值为字符串对象，我们说一个键为集合键时，表示的是这个键对应的值为集合对象。**

②、encoding 属性和 *prt 指针

对象的 prt 指针指向对象底层的数据结构，而数据结构由 encoding 属性来决定。

![Redis-encoding](png\Redis\Redis-encoding.png)

而每种类型的对象都至少使用了两种不同的编码：

![Redis-Object](png\Redis\Redis-Object.png)

可以通过如下命令查看值对象的编码：

```
OBJECT ENCODING  key
```

比如 string 类型：（可以是 embstr编码的简单字符串或者是 int 整数值实现）

### 字符串对象

　字符串是Redis最基本的数据类型，不仅所有key都是字符串类型，其它几种数据类型构成的元素也是字符串。注意字符串的长度不能超过512M。

　　**①、编码**

　　字符串对象的编码可以是int，raw或者embstr。

　　1、int 编码：保存的是可以用 long 类型表示的整数值。

　　2、raw 编码：保存长度大于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）。

　　3、embstr 编码：保存长度小于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）。

　由上可以看出，int 编码是用来保存整数值，raw编码是用来保存长字符串，而embstr是用来保存短字符串。其实 embstr 编码是专门用来保存短字符串的一种优化编码，raw 和 embstr 的区别：

embstr与raw都使用redisObject和sds保存数据，区别在于，embstr的使用只分配一次内存空间（因此redisObject和sds是连续的），而raw需要分配两次内存空间（分别为redisObject和sds分配空间）。因此与raw相比，embstr的好处在于创建时少分配一次空间，删除时少释放一次空间，以及对象的所有数据连在一起，寻找方便。而embstr的坏处也很明显，如果字符串的长度增加需要重新分配内存时，整个redisObject和sds都需要重新分配空间，因此redis中的embstr实现为只读。

　　**ps：Redis中对于浮点数类型也是作为字符串保存的，在需要的时候再将其转换成浮点数类型。**

　　**②、编码的转换**

　　当 int 编码保存的值不再是整数，或大小超过了long的范围时，自动转化为raw。

　　对于 embstr 编码，由于 Redis 没有对其编写任何的修改程序（embstr 是只读的），在对embstr对象进行修改时，都会先转化为raw再进行修改，因此，只要是修改embstr对象，修改后的对象一定是raw的，无论是否达到了44个字节。

### 列表对象

list 列表，它是简单的字符串列表，按照插入顺序排序，你可以添加一个元素到列表的头部（左边）或者尾部（右边），它的底层实际上是个链表结构。

　　**①、编码**

　　列表对象的编码可以是 ziplist(压缩列表) 和 linkedlist(双端链表)。

　　比如我们执行以下命令，创建一个 key = ‘numbers’，value = ‘1 three 5’ 的三个值的列表。

```
rpush numbers 1 ``"three"` `5
```

　　**②、编码转换**

　　当同时满足下面两个条件时，使用ziplist（压缩列表）编码：

　　1、列表保存元素个数小于512个

　　2、每个元素长度小于64字节

　　不能满足这两个条件的时候使用 linkedlist 编码。

　　上面两个条件可以在redis.conf 配置文件中的 list-max-ziplist-value选项和 list-max-ziplist-entries 选项进行配置。

### 哈希对象

哈希对象的键是一个字符串类型，值是一个键值对集合。

　　**①、编码**

　　哈希对象的编码可以是 ziplist 或者 hashtable。

　　当使用ziplist，也就是压缩列表作为底层实现时，新增的键值对是保存到压缩列表的表尾。比如执行以下命令：

```
hset profile name ``"Tom"``hset profile age 25``hset profile career ``"Programmer"
```

hashtable 编码的哈希表对象底层使用字典数据结构，哈希对象中的每个键值对都使用一个字典键值对。

　　在前面介绍压缩列表时，我们介绍过压缩列表是Redis为了节省内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，相对于字典数据结构，压缩列表用于元素个数少、元素长度小的场景。其优势在于集中存储，节省空间。

　　**②、编码转换**

　　和上面列表对象使用 ziplist 编码一样，当同时满足下面两个条件时，使用ziplist（压缩列表）编码：

　　1、列表保存元素个数小于512个

　　2、每个元素长度小于64字节

　　不能满足这两个条件的时候使用 hashtable 编码。第一个条件可以通过配置文件中的 set-max-intset-entries 进行修改。

### 集合对象

集合对象 set 是 string 类型（整数也会转换成string类型进行存储）的无序集合。注意集合和列表的区别：集合中的元素是无序的，因此不能通过索引来操作元素；集合中的元素不能有重复。

　　**①、编码**

　　集合对象的编码可以是 intset 或者 hashtable。

　　intset 编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合中。

　　hashtable 编码的集合对象使用 字典作为底层实现，字典的每个键都是一个字符串对象，这里的每个字符串对象就是一个集合中的元素，而字典的值则全部设置为 null。这里可以类比Java集合中HashSet 集合的实现，HashSet 集合是由 HashMap 来实现的，集合中的元素就是 HashMap 的key，而 HashMap 的值都设为 null。

```
SADD numbers 1 3 5
```

**②、编码转换**

　　当集合同时满足以下两个条件时，使用 intset 编码：

　　1、集合对象中所有元素都是整数

　　2、集合对象所有元素数量不超过512

　　不能满足这两个条件的就使用 hashtable 编码。第二个条件可以通过配置文件的 set-max-intset-entries 进行配置。

### 有序集合对象

和上面的集合对象相比，有序集合对象是有序的。与列表使用索引下标作为排序依据不同，有序集合为每个元素设置一个分数（score）作为排序依据。

　　**①、编码**

　　有序集合的编码可以是 ziplist 或者 skiplist。

　　ziplist 编码的有序集合对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员，第二个节点保存元素的分值。并且压缩列表内的集合元素按分值从小到大的顺序进行排列，小的放置在靠近表头的位置，大的放置在靠近表尾的位置。

```
ZADD price 8.5 apple 5.0 banana 6.0 cherry
```

skiplist 编码的有序集合对象使用 zet 结构作为底层实现，一个 zset 结构同时包含一个字典和一个跳跃表：

```
typedef struct zset{
     //跳跃表
     zskiplist *zsl;
     //字典
     dict *dice;
} zset;
```

　　字典的键保存元素的值，字典的值则保存元素的分值；跳跃表节点的 object 属性保存元素的成员，跳跃表节点的 score 属性保存元素的分值。

　　这两种数据结构会通过指针来共享相同元素的成员和分值，所以不会产生重复成员和分值，造成内存的浪费。

　　说明：其实有序集合单独使用字典或跳跃表其中一种数据结构都可以实现，但是这里使用两种数据结构组合起来，原因是假如我们单独使用 字典，虽然能以 O(1) 的时间复杂度查找成员的分值，但是因为字典是以无序的方式来保存集合元素，所以每次进行范围操作的时候都要进行排序；假如我们单独使用跳跃表来实现，虽然能执行范围操作，但是查找操作有 O(1)的复杂度变为了O(logN)。因此Redis使用了两种数据结构来共同实现有序集合。

　　**②、编码转换**

　　当有序集合对象同时满足以下两个条件时，对象使用 ziplist 编码：

　　1、保存的元素数量小于128；

　　2、保存的所有元素长度都小于64字节。

　　不能满足上面两个条件的使用 skiplist 编码。以上两个条件也可以通过Redis配置文件zset-max-ziplist-entries 选项和 zset-max-ziplist-value 进行修改。

### 五大数据类型的应用场景

对于string 数据类型，因为string 类型是二进制安全的，可以用来存放图片，视频等内容，另外由于Redis的高性能读写功能，而string类型的value也可以是数字，可以用作计数器（INCR,DECR），比如分布式环境中统计系统的在线人数，秒杀等。

　　对于 hash 数据类型，value 存放的是键值对，比如可以做单点登录存放用户信息。

　　对于 list 数据类型，可以实现简单的消息队列，另外可以利用lrange命令，做基于redis的分页功能

　　对于 set 数据类型，由于底层是字典实现的，查找元素特别快，另外set 数据类型不允许重复，利用这两个特性我们可以进行全局去重，比如在用户注册模块，判断用户名是否注册；另外就是利用交集、并集、差集等操作，可以计算共同喜好，全部的喜好，自己独有的喜好等功能。

　　对于 zset 数据类型，有序的集合，可以做范围查找，排行榜应用，取 TOP N 操作等。

### 内存回收和内存共享

①、内存回收

　　前面讲 Redis 的每个对象都是由 redisObject 结构表示：

```
typedef struct redisObject{
     //类型
     unsigned type:4;
     //编码
     unsigned encoding:4;
     //指向底层数据结构的指针
     void *ptr;
     //引用计数
     int refcount;
     //记录最后一次被程序访问的时间
     unsigned lru:22;
 
}robj
```

　　其中关键的 type属性，encoding 属性和 ptr 指针都介绍过了，那么 refcount 属性是干什么的呢？

　　因为 C 语言不具备自动回收内存功能，那么该如何回收内存呢？于是 Redis自己构建了一个内存回收机制，通过在 redisObject 结构中的 refcount 属性实现。这个属性会随着对象的使用状态而不断变化：

　　1、创建一个新对象，属性 refcount 初始化为1

　　2、对象被一个新程序使用，属性 refcount 加 1

　　3、对象不再被一个程序使用，属性 refcount 减 1

　　4、当对象的引用计数值变为 0 时，对象所占用的内存就会被释放。

学过Java的应该知道，引用计数的内存回收机制其实是不被Java采用的，因为不能克服循环引用的例子（比如 A 具有 B 的引用，B 具有 C 的引用，C 具有 A 的引用，除此之外，这三个对象没有任何用处了），这时候 A B C 三个对象会一直驻留在内存中，造成内存泄露。那么 Redis 既然采用引用计数的垃圾回收机制，如何解决这个问题呢？

　　在前面介绍 redis.conf 配置文件时，在 MEMORY MANAGEMENT 下有个 maxmemory-policy 配置：

　　maxmemory-policy ：当内存使用达到最大值时，redis使用的清楚策略。有以下几种可以选择：

　　　　1）volatile-lru  利用LRU算法移除设置过过期时间的key (LRU:最近使用 Least Recently Used ) 

　　　　2）allkeys-lru  利用LRU算法移除任何key 

　　　　3）volatile-random 移除设置过过期时间的随机key 

　　　　4）allkeys-random 移除随机key

　　　　5）volatile-ttl  移除即将过期的key(minor TTL) 

　　　　6）noeviction noeviction  不移除任何key，只是返回一个写错误 ，默认选项

　　通过这种配置，也可以对内存进行回收。

②、内存共享 

　　refcount 属性除了能实现内存回收以外，还能用于内存共享。

　　比如通过如下命令 set k1 100,创建一个键为 k1，值为100的字符串对象，接着通过如下命令 set k2 100 ，创建一个键为 k2，值为100 的字符串对象，那么 Redis 是如何做的呢？

　　1、将数据库键的值指针指向一个现有值的对象

　　2、将被共享的值对象引用refcount 加 1

注意：Redis的共享对象目前只支持整数值的字符串对象。之所以如此，实际上是对内存和CPU（时间）的平衡：共享对象虽然会降低内存消耗，但是判断两个对象是否相等却需要消耗额外的时间。对于整数值，判断操作复杂度为O(1)；对于普通字符串，判断复杂度为O(n)；而对于哈希、列表、集合和有序集合，判断的复杂度为O(n^2)。

　　虽然共享对象只能是整数值的字符串对象，但是5种类型都可能使用共享对象（如哈希、列表等的元素可以使用）。

### 对象的空转时长

　在 redisObject 结构中，前面介绍了 type、encoding、ptr 和 refcount 属性，最后一个 lru 属性，该属性记录了对象最后一次被命令程序访问的时间。

　　使用 OBJECT IDLETIME 命令可以打印给定键的空转时长，通过将当前时间减去值对象的 lru 时间计算得到。

lru 属性除了计算空转时长以外，还可以配合前面内存回收配置使用。如果Redis打开了maxmemory选项，且内存回收算法选择的是volatile-lru或allkeys—lru，那么当Redis内存占用超过maxmemory指定的值时，Redis会优先选择空转时间最长的对象进行释放。

## 持久化

由于 Redis 是一个内存数据库，所谓内存数据库，就是将数据库中的内容保存在内存中，这与传统的MySQL，Oracle等关系型数据库直接将内容保存到硬盘中相比，内存数据库的读写效率比传统数据库要快的多（内存的读写效率远远大于硬盘的读写效率）。但是保存在内存中也随之带来了一个缺点，一旦断电或者宕机，那么内存数据库中的数据将会全部丢失。

　　为了解决这个缺点，Redis提供了将内存数据持久化到硬盘，以及用持久化文件来恢复数据库数据的功能。Redis 支持两种形式的持久化，一种是RDB快照（snapshotting），另外一种是AOF（append-only-file）。

通常来说，应该同时使用两种持久化方案，以保证数据安全：

- 如果数据不敏感，且可以从其他地方重新生成，可以关闭持久化
- 如果数据比较重要，且能够承受几分钟的数据丢失，比如缓存等，只需要使用RDB即可
- 如果是用做内存数据，要使用Redis的持久化，建议是RDB和AOF都开启
- 如果只用AOF，优先使用everysec的配置选择，因为它在可靠性和性能之间取了一个平衡

当RDB与AOF两种方式都开启时，Redis会优先使用AOF恢复数据，因为AOF保存的文件比RDB文件更完整

### RDB模式(内存快照)

RDB是Redis用来进行持久化的一种方式，是把当前内存中的数据集快照写入磁盘，也就是 Snapshot 快照（数据库中所有键值对数据）。恢复时是将快照文件直接读到内存里。

`RDB`（Redis Database Backup File，**Redis数据备份文件**）持久化方式：是指用数据集快照的方式半持久化模式记录 Redis 数据库的所有键值对，在某个时间点将数据写入一个临时文件，持久化结束后，用这个临时文件替换上次持久化的文件，达到数据恢复。

#### 触发方式

RDB 有两种触发方式，分别是自动触发和手动触发。

##### ①、自动触发

　　在 redis.conf 配置文件中的 SNAPSHOTTING 下

```
################################ SNAPSHOTTING  ################################
#
# Save the DB on disk:
# 在给定的秒数和给定的对数据库的写操作数下，自动持久化操作。
#  save <seconds> <changes>
# 
save 900 1
save 300 10
save 60 10000
#bgsave发生错误时是否停止写入，一般为yes
stop-writes-on-bgsave-error yes
#持久化时是否使用LZF压缩字符串对象?
rdbcompression yes
#是否对rdb文件进行校验和检验，通常为yes
rdbchecksum yes
# RDB持久化文件名
dbfilename dump.rdb
#持久化文件存储目录
dir ./
```

**①、save：**这里是用来配置触发 Redis的 RDB 持久化条件，也就是什么时候将内存中的数据保存到硬盘。比如“save m n”。表示m秒内数据集存在n次修改时，自动触发bgsave（这个命令下面会介绍，手动触发RDB持久化的命令）

　　默认如下配置：

```
save 900 1：表示900 秒内如果至少有 1 个 key 的值变化，则保存
save 300 10：表示300 秒内如果至少有 10 个 key 的值变化，则保存
save 60 10000：表示60 秒内如果至少有 10000 个 key 的值变化，则保存
```

　　　　当然如果你只是用Redis的缓存功能，不需要持久化，那么你可以注释掉所有的 save 行来停用保存功能。可以直接一个空字符串来实现停用：save ""

　　**②、stop-writes-on-bgsave-error ：**默认值为yes。当启用了RDB且最后一次后台保存数据失败，Redis是否停止接收数据。这会让用户意识到数据没有正确持久化到磁盘上，否则没有人会注意到灾难（disaster）发生了。如果Redis重启了，那么又可以重新开始接收数据了

　　**③、rdbcompression ；**默认值是yes。对于存储到磁盘中的快照，可以设置是否进行压缩存储。如果是的话，redis会采用LZF算法进行压缩。如果你不想消耗CPU来进行压缩的话，可以设置为关闭此功能，但是存储在磁盘上的快照会比较大。

　　**④、rdbchecksum ：**默认值是yes。在存储快照后，我们还可以让redis使用CRC64算法来进行数据校验，但是这样做会增加大约10%的性能消耗，如果希望获取到最大的性能提升，可以关闭此功能。

　　**⑤、dbfilename ：**设置快照的文件名，默认是 dump.rdb

　　**⑥、dir：**设置快照文件的存放路径，这个配置项一定是个目录，而不能是文件名。默认是和当前配置文件保存在同一目录。

　　也就是说通过在配置文件中配置的 save 方式，当实际操作满足该配置形式时就会进行 RDB 持久化，将当前的内存快照保存在 dir 配置的目录中，文件名由配置的 dbfilename 决定。

##### ②、手动触发

　　手动触发Redis进行RDB持久化的命令有两种：

　　1、save

　　该命令会阻塞当前Redis服务器，执行save命令期间，Redis不能处理其他命令，直到RDB过程完成为止。

　　显然该命令对于内存比较大的实例会造成长时间阻塞，这是致命的缺陷，为了解决此问题，Redis提供了第二种方式。

　　2、bgsave

　　执行该命令时，Redis会在后台异步进行快照操作，快照同时还可以响应客户端请求。具体操作是Redis进程执行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束。阻塞只发生在fork阶段，一般时间很短。

　　**基本上 Redis 内部所有的RDB操作都是采用 bgsave 命令。**

　　**ps:执行执行 flushall 命令，也会产生dump.rdb文件，但里面是空的.**

#### 恢复数据

将备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务即可，redis就会自动加载文件数据至内存了。Redis 服务器在载入 RDB 文件期间，会一直处于阻塞状态，直到载入工作完成为止。

　　获取 redis 的安装目录可以使用 config get dir 命令

#### 停止 RDB 持久化

有些情况下，我们只想利用Redis的缓存功能，并不像使用 Redis 的持久化功能，那么这时候我们最好停掉 RDB 持久化。可以通过上面讲的在配置文件 redis.conf 中，可以注释掉所有的 save 行来停用保存功能或者直接一个空字符串来实现停用：save ""

也可以通过命令：

```
redis-cli config set save " "
```

#### RDB 的优势和劣势

①、优势

　　1.RDB是一个非常紧凑(compact)的文件，它保存了redis 在某个时间点上的数据集。这种文件非常适合用于进行备份和灾难恢复。

　　2.生成RDB文件的时候，redis主进程会fork()一个子进程来处理所有保存工作，主进程不需要进行任何磁盘IO操作。可最大化Redis的的性能。在保存RDB文件，服务器进程只需要fork一个子进程来完成RDB文件创建，父进程不需要做IO操作

　　3.RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。

　　②、劣势

　　1、RDB方式数据没办法做到实时持久化/秒级持久化。因为bgsave每次运行都要执行fork操作创建子进程，属于重量级操作，如果不采用压缩算法(内存中的数据被克隆了一份，大致2倍的膨胀性需要考虑，这里评论区指出，确实有不妥，主进程 fork 出子进程，其实是共享一份真实的内存空间，但是为了能在记录快照的时候，也能让主线程处理写操作，采用的是 Copy-On-Write（写时复制）技术，只有需要修改的内存才会复制一份出来，所以内存膨胀到底有多大，看修改的比例有多大)，频繁执行成本过高(影响性能)

　　2、RDB文件使用特定二进制格式保存，Redis版本演进过程中有多个格式的RDB版本，存在老版本Redis服务无法兼容新版RDB格式的问题(版本不兼容)

　　3、在一定间隔时间做一次备份，所以如果redis意外down掉的话，就会丢失最后一次快照后的所有修改(数据有丢失)

#### RDB 自动保存的原理

Redis有个服务器状态结构：

```
struct redisService{
     //1、记录保存save条件的数组
     struct saveparam *saveparams;
     //2、修改计数器
     long long dirty;
     //3、上一次执行保存的时间
     time_t lastsave;
 
}
```

①、首先看记录保存save条件的数组 saveparam，里面每个元素都是一个 saveparams 结构：

```
struct saveparam{
     //秒数
     time_t seconds;
     //修改数
     int changes;
};
```

前面我们在 redis.conf 配置文件中进行了关于save 的配置：

```
save 900 1：表示900 秒内如果至少有 1 个 key 的值变化，则保存
save 300 10：表示300 秒内如果至少有 10 个 key 的值变化，则保存
save 60 10000：表示60 秒内如果至少有 10000 个 key 的值变化，则保存
```

那么服务器状态中的saveparam 数组将会是如下的样子：

![Redis-save](png\Redis\Redis-save.png)



②、dirty 计数器和lastsave 属性

　　dirty 计数器记录距离上一次成功执行 save 命令或者 bgsave 命令之后，Redis服务器进行了多少次修改（包括写入、删除、更新等操作）。

　　lastsave 属性是一个时间戳，记录上一次成功执行 save 命令或者 bgsave 命令的时间。

　　通过这两个命令，当服务器成功执行一次修改操作，那么dirty 计数器就会加 1，而lastsave 属性记录上一次执行save或bgsave的时间，Redis 服务器还有一个周期性操作函数 severCron ,默认每隔 100 毫秒就会执行一次，该函数会遍历并检查 saveparams 数组中的所有保存条件，只要有一个条件被满足，那么就会执行 bgsave 命令。

　　执行完成之后，dirty 计数器更新为 0 ，lastsave 也更新为执行命令的完成时间。

**① 创建**

当 Redis 持久化时，程序会将当前内存中的**数据库状态**保存到磁盘中。创建 RDB 文件主要有两个 Redis 命令：`SAVE` 和 `BGSAVE`。

![Redis-RDB-创建](png/Redis/Redis-RDB-创建.png)



**② 载入**

服务器在载入 RDB 文件期间，会一直处于**阻塞**状态，直到载入工作完成为止。

![Redis-RDB-载入](png/Redis/Redis-RDB-载入.png)



#### save同步保存

`save` 命令是同步操作，执行命令时，会 **阻塞** Redis 服务器进程，拒绝客户端发送的命令请求。

具体流程如下：

![Redis-RDB-Save命令](png/Redis/Redis-RDB-Save命令.png)

由于 `save` 命令是同步命令，会占用Redis的主进程。若Redis数据非常多时，`save`命令执行速度会非常慢，阻塞所有客户端的请求。因此很少在生产环境直接使用SAVE 命令，可以使用BGSAVE 命令代替。如果在BGSAVE命令的保存数据的子进程发生错误的时，用 SAVE命令保存最新的数据是最后的手段。

#### bgsave异步保存

`bgsave` 命令是异步操作，执行命令时，子进程执行保存工作，服务器还可以继续让主线程**处理**客户端发送的命令请求。

具体流程如下：

![Redis-RDB-BgSave命令](png/Redis/Redis-RDB-BgSave命令.png)

Redis使用Linux系统的`fock()`生成一个子进程来将DB数据保存到磁盘，主进程继续提供服务以供客户端调用。如果操作成功，可以通过客户端命令LASTSAVE来检查操作结果。

#### 自动保存

可通过配置文件对 Redis 进行设置， 让它在“ N 秒内数据集至少有 M 个改动”这一条件被满足时， 自动进行数据集保存操作:

```shell
# RDB自动持久化规则
# 当 900 秒内有至少有 1 个键被改动时，自动进行数据集保存操作
save 900 1
# 当 300 秒内有至少有 10 个键被改动时，自动进行数据集保存操作
save 300 10
# 当 60 秒内有至少有 10000 个键被改动时，自动进行数据集保存操作
save 60 10000

# RDB持久化文件名
dbfilename dump-<port>.rdb

# 数据持久化文件存储目录
dir /var/lib/redis

# bgsave发生错误时是否停止写入，通常为yes
stop-writes-on-bgsave-error yes

# rdb文件是否使用压缩格式
rdbcompression yes

# 是否对rdb文件进行校验和检验，通常为yes
rdbchecksum yes
```

#### 默认配置

RDB 文件默认的配置如下：

```properties
################################ SNAPSHOTTING  ################################
#
# Save the DB on disk:
# 在给定的秒数和给定的对数据库的写操作数下，自动持久化操作。
#  save <seconds> <changes>
# 
save 900 1
save 300 10
save 60 10000
#bgsave发生错误时是否停止写入，一般为yes
stop-writes-on-bgsave-error yes
#持久化时是否使用LZF压缩字符串对象?
rdbcompression yes
#是否对rdb文件进行校验和检验，通常为yes
rdbchecksum yes
# RDB持久化文件名
dbfilename dump.rdb
#持久化文件存储目录
dir ./
```



### AOF模式(日志追加)

#### AOF简介

RDB 持久化存在一个缺点是一定时间内做一次备份，如果redis意外down掉的话，就会丢失最后一次快照后的所有修改(数据有丢失)。对于数据完整性要求很严格的需求，怎么解决呢？

Redis的持久化方式之一RDB是通过保存数据库中的键值对来记录数据库的状态。而另一种持久化方式 AOF 则是通过保存Redis服务器所执行的写命令来记录数据库状态。

AOF（Append  Only File，**追加日志文件**）持久化方式：是指所有的命令行记录以 Redis 命令请求协议的格式完全持久化存储保存为 aof 文件。Redis 是先执行命令，把数据写入内存，然后才记录日志。因为该模式是**只追加**的方式，所以没有任何磁盘寻址的开销，所以很快，有点像 Mysql 中的binlog，AOF更适合做热备。

![Redis-AOF](png/Redis/Redis-AOF.png)

RDB 持久化方式就是将 str1,str2,str3 这三个键值对保存到 RDB文件中，而 AOF 持久化则是将执行的 set,sadd,lpush 三个命令保存到 AOF 文件中。

#### AOF 配置

AOF 文件默认的配置如下：

```properties
############################## APPEND ONLY MODE ###############################
#开启AOF持久化方式
appendonly no
#AOF持久化文件名
appendfilename "appendonly.aof"
#每秒把缓冲区的数据fsync到磁盘
appendfsync everysec
# appendfsync no
#是否在执行重写时不同步数据到AOF文件
no-appendfsync-on-rewrite no
# 触发AOF文件执行重写的增长率
auto-aof-rewrite-percentage 100
#触发AOF文件执行重写的最小size
auto-aof-rewrite-min-size 64mb
#redis在恢复时，会忽略最后一条可能存在问题的指令
aof-load-truncated yes
#是否打开混合开关
aof-use-rdb-preamble yes
```

①、**appendonly**：默认值为no，也就是说redis 默认使用的是rdb方式持久化，如果想要开启 AOF 持久化方式，需要将 appendonly 修改为 yes。

　　②、**appendfilename** ：aof文件名，默认是"appendonly.aof"

　　③、**appendfsync：**aof持久化策略的配置；

　　　　　　no表示不执行fsync，由操作系统保证数据同步到磁盘，速度最快，但是不太安全；

　　　　　　always表示每次写入都执行fsync，以保证数据同步到磁盘，效率很低；

　　　　　　everysec表示每秒执行一次fsync，可能会导致丢失这1s数据。通常选择 everysec ，兼顾安全性和效率。

　　④、no-appendfsync-on-rewrite：在aof重写或者写入rdb文件的时候，会执行大量IO，此时对于everysec和always的aof模式来说，执行fsync会造成阻塞过长时间，no-appendfsync-on-rewrite字段设置为默认设置为no。如果对延迟要求很高的应用，这个字段可以设置为yes，否则还是设置为no，这样对持久化特性来说这是更安全的选择。  设置为yes表示rewrite期间对新写操作不fsync,暂时存在内存中,等rewrite完成后再写入，默认为no，建议yes。Linux的默认fsync策略是30秒。可能丢失30秒数据。默认值为no。

　　⑤、auto-aof-rewrite-percentage：默认值为100。aof自动重写配置，当目前aof文件大小超过上一次重写的aof文件大小的百分之多少进行重写，即当aof文件增长到一定大小的时候，Redis能够调用bgrewriteaof对日志文件进行重写。当前AOF文件大小是上次日志重写得到AOF文件大小的二倍（设置为100）时，自动启动新的日志重写过程。

　　⑥、auto-aof-rewrite-min-size：64mb。设置允许重写的最小aof文件大小，避免了达到约定百分比但尺寸仍然很小的情况还要重写。

　　⑦、aof-load-truncated：aof文件可能在尾部是不完整的，当redis启动的时候，aof文件的数据被载入内存。重启可能发生在redis所在的主机操作系统宕机后，尤其在ext4文件系统没有加上data=ordered选项，出现这种现象 redis宕机或者异常终止不会造成尾部不完整现象，可以选择让redis退出，或者导入尽可能多的数据。如果选择的是yes，当截断的aof文件被导入的时候，会自动发布一个log给客户端然后load。如果是no，用户必须手动redis-check-aof修复AOF文件才可以。默认值为 yes。

#### 开启 AOF

将 redis.conf 的 appendonly 配置改为 yes 即可。

　　AOF 保存文件的位置和 RDB 保存文件的位置一样，都是通过 redis.conf 配置文件的 dir 配置：

#### AOF 文件恢复

重启 Redis 之后就会进行 AOF 文件的载入。

　　异常修复命令：redis-check-aof --fix 进行修复

#### AOF 重写

由于AOF持久化是Redis不断将写命令记录到 AOF 文件中，随着Redis不断的进行，AOF 的文件会越来越大，文件越大，占用服务器内存越大以及 AOF 恢复要求时间越长。为了解决这个问题，Redis新增了重写机制，当AOF文件的大小超过所设定的阈值时，Redis就会启动AOF文件的内容压缩，只保留可以恢复数据的最小指令集。可以使用命令 bgrewriteaof 来重新。

如果不进行 AOF 文件重写，那么 AOF 文件将保存四条 SADD 命令，如果使用AOF 重写，那么AOF 文件中将只会保留下面一条命令：

```
sadd animals ``"dog"` `"tiger"` `"panda"` `"lion"` `"cat"
```

　　**也就是说 AOF 文件重写并不是对原文件进行重新整理，而是直接读取服务器现有的键值对，然后用一条命令去代替之前记录这个键值对的多条命令，生成一个新的文件后去替换原来的 AOF 文件。**

 　AOF 文件重写触发机制：通过 redis.conf 配置文件中的 auto-aof-rewrite-percentage：默认值为100，以及auto-aof-rewrite-min-size：64mb 配置，也就是说默认Redis会记录上次重写时的AOF大小，**默认配置是当AOF文件大小是上次rewrite后大小的一倍且文件大于64M时触发。**

　　这里再提一下，我们知道 Redis 是单线程工作，如果 重写 AOF 需要比较长的时间，那么在重写 AOF 期间，Redis将长时间无法处理其他的命令，这显然是不能忍受的。Redis为了克服这个问题，解决办法是将 AOF 重写程序放到子程序中进行，这样有两个好处：

　　①、子进程进行 AOF 重写期间，服务器进程（父进程）可以继续处理其他命令。

　　②、子进程带有父进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据的安全性。

　　使用子进程解决了上面的问题，但是新问题也产生了：因为子进程在进行 AOF 重写期间，服务器进程依然在处理其它命令，这新的命令有可能也对数据库进行了修改操作，使得当前数据库状态和重写后的 AOF 文件状态不一致。

　　为了解决这个数据状态不一致的问题，Redis 服务器设置了一个 AOF 重写缓冲区，这个缓冲区是在创建子进程后开始使用，当Redis服务器执行一个写命令之后，就会将这个写命令也发送到 AOF 重写缓冲区。当子进程完成 AOF 重写之后，就会给父进程发送一个信号，父进程接收此信号后，就会调用函数将 AOF 重写缓冲区的内容都写到新的 AOF 文件中。

　　这样将 AOF 重写对服务器造成的影响降到了最低。

#### AOF的优缺点

**优点：**

　　①、AOF 持久化的方法提供了多种的同步频率，即使使用默认的同步频率每秒同步一次，Redis 最多也就丢失 1 秒的数据而已。数据更完整，安全性更高，秒级数据丢失（取决fsync策略，如果是everysec，最多丢失1秒的数据）

　　②、AOF 文件使用 Redis 命令追加的形式来构造，因此，即使 Redis 只能向 AOF 文件写入命令的片断，使用 redis-check-aof 工具也很容易修正 AOF 文件。

　　③、AOF 文件的格式可读性较强，这也为使用者提供了更灵活的处理方式。例如，如果我们不小心错用了 FLUSHALL 命令，在重写还没进行时，我们可以手工将最后的 FLUSHALL 命令去掉，然后再使用 AOF 来恢复数据。AOF文件是一个只进行追加的日志文件，且写入操作是以Redis协议的格式保存的，内容是可读的，适合误删紧急恢复

**缺点：**

　　①、对于具有相同数据的的 Redis，AOF 文件通常会比 RDF 文件体积更大。

　　②、虽然 AOF 提供了多种同步的频率，默认情况下，每秒同步一次的频率也具有较高的性能。但在 Redis 的负载较高时，RDB 比 AOF 具好更好的性能保证。

　　③、RDB 使用快照的形式来持久化整个 Redis 数据，而 AOF 只是将每次执行的命令追加到 AOF 文件中，因此从理论上说，RDB 比 AOF 方式更健壮。官方文档也指出，AOF 的确也存在一些 BUG，这些 BUG 在 RDB 没有存在。

 　那么对于 AOF 和 RDB 两种持久化方式，我们应该如何选择呢？

　　如果可以忍受一小段时间内数据的丢失，毫无疑问使用 RDB 是最好的，定时生成 RDB 快照（snapshot）非常便于进行数据库备份， 并且 RDB 恢复数据集的速度也要比 AOF 恢复的速度要快，而且使用 RDB 还可以避免 AOF 一些隐藏的 bug；否则就使用 AOF 重写。但是一般情况下建议不要单独使用某一种持久化机制，而是应该两种一起用，在这种情况下,当redis重启的时候会优先载入AOF文件来恢复原始的数据，因为在通常情况下AOF文件保存的数据集要比RDB文件保存的数据集要完整。Redis后期官方可能都有将两种持久化方式整合为一种持久化模型。

#### 持久化流程

![Redis-AOF持久化流程](png/Redis/Redis-AOF持久化流程.png)

**① 命令追加**

若 AOF 持久化功能处于打开状态，服务器在执行完一个命令后，会以协议格式将被执行的写命令**追加**到服务器状态的 `aof_buf` 缓冲区的末尾。

**② 文件同步**

服务器每次结束一个事件循环之前，都会调用 `flushAppendOnlyFile` 函数，这个函数会考虑是否需要将 `aof_buf` 缓冲区中的内容**写入和保存**到 AOF 文件里。`flushAppendOnlyFile` 函数执行以下流程：

- WRITE：根据条件，将 aof_buf 中的**缓存**写入到 AOF 文件；
- SAVE：根据条件，调用 fsync 或 fdatasync 函数，将 AOF 文件保存到**磁盘**中。

这个函数是由服务器配置的 `appendfsync` 的三个值：`always、everysec、no` 来影响的，也被称为三种策略：

- **always**：每条**命令**都会 fsync 到硬盘中，这样 redis 的写入数据就不会丢失。

  ![Redis-AOF-Always](png/Redis/Redis-AOF-Always.png)

- **everysec**：**每秒**都会刷新缓冲区到硬盘中(默认值)。

  ![Redis-AOF-Everysec](png/Redis/Redis-AOF-Everysec.png)

- **no**：根据当前**操作系统**的规则决定什么时候刷新到硬盘中，不需要我们来考虑。

  ![Redis-AOF-No](png/Redis/Redis-AOF-No.png)



**数据加载**

- 创建一个不带网络连接的伪客户端
- 从 AOF 文件中分析并读取出一条写命令
- 使用伪客户端执行被读出的写命令
- 一直执行步骤 2 和 3，直到 AOF 文件中的所有写命令都被**处理完毕**为止



#### 文件重写实现原理

**为何需要文件重写**

- 为了解决 AOF 文件**体积膨胀**的问题
- 通过重写创建一个新的 AOF 文件来替代现有的 AOF 文件，新的 AOF 文件不会包含任何浪费空间的冗余命令

**文件重写的实现原理**

- 不需要对现有的 AOF 文件进行任何操作
- 从数据库中直接读取键现在的值
- 用一条命令记录键值对，从而**代替**之前记录这个键值对的多条命令

**后台重写**

为不阻塞父进程，Redis将AOF重写程序放到**子进程**里执行。在子进程执行AOF重写期间，服务器进程需要执行三个流程：

- 执行客户端发来的命令
- 将执行后的写命令追加到 AOF 缓冲区
- 将执行后的写命令追加到 AOF 重写缓冲区

![Redis-AOF-后台重写](png/Redis/Redis-AOF-后台重写.png)

### RDB-AOF混合持久化

这里补充一个知识点，在Redis4.0之后，既上一篇文章介绍的RDB和这篇文章介绍的AOF两种持久化方式，又新增了RDB-AOF混合持久化方式。

　　这种方式结合了RDB和AOF的优点，既能快速加载又能避免丢失过多的数据。

　　具体配置为：

```
aof-use-rdb-preamble
```

　　设置为yes表示开启，设置为no表示禁用。

　　当开启混合持久化时，主进程先fork出子进程将现有内存副本全量以RDB方式写入aof文件中，然后将缓冲区中的增量命令以AOF方式写入aof文件中，写入完成后通知主进程更新相关信息，并将新的含有 RDB和AOF两种格式的aof文件替换旧的aof文件。

　　简单来说：混合持久化方式产生的文件一部分是RDB格式，一部分是AOF格式。

　　这种方式优点我们很好理解，缺点就是不能兼容Redis4.0之前版本的备份文件了。

## 过期策略

**过期策略用于处理过期缓存数据**。Redis采用过期策略：`惰性删除` + `定期删除`。memcached采用过期策略：`惰性删除`。

### 定时过期

每个设置过期时间的key都需要创建一个定时器，到过期时间就会立即对key进行清除。该策略可以立即清除过期的数据，对内存很友好；但是**会占用大量的CPU资源去处理过期的数据**，从而影响缓存的响应时间和吞吐量。

### 惰性过期

只有当访问一个key时，才会判断该key是否已过期，过期则清除。该策略可以最大化地节省CPU资源，却**对内存非常不友好**。极端情况可能出现大量的过期key没有再次被访问，从而不会被清除，占用大量内存。

### 定期过期

每隔一定的时间，会扫描一定数量的数据库的expires字典中一定数量的key，并清除其中已过期的key。该策略是前两者的一个折中方案。通过调整定时扫描的时间间隔和每次扫描的限定耗时，可以在不同情况下使得CPU和内存资源**达到最优**的平衡效果。expires字典会保存所有设置了过期时间的key的过期时间数据，其中 key 是指向键空间中的某个键的指针，value是该键的毫秒精度的UNIX时间戳表示的过期时间。键空间是指该Redis集群中保存的所有键。

## 淘汰策略

Redis淘汰机制的存在是为了更好的使用内存，用一定的缓存丢失来换取内存的使用效率。当Redis内存快耗尽时，Redis会启动内存淘汰机制，将部分key清掉以腾出内存。当达到内存使用上限超过`maxmemory`时，可在配置文件`redis.conf`中指定 `maxmemory-policy` 的清理缓存方式。

```properties
# 配置最大内存限制
maxmemory 1000mb
# 配置淘汰策略
maxmemory-policy volatile-lru
```

### LRU(最近最少使用)

- `volatile-lru`：从已设置过期时间的key中，挑选**最近最少使用(最长时间没有使用)**的key进行淘汰
- `allkeys-lru`：从所有key中，挑选**最近最少使用**的数据淘汰



### LFU(最近最不经常使用)

- `volatile-lfu`：从已设置过期时间的key中，挑选**最近最不经常使用(使用次数最少)**的key进行淘汰
- `allkeys-lfu`：从所有key中，选择某段时间内内**最近最不经常使用**的数据淘汰



### Random(随机淘汰)

- `volatile-random`：从已设置过期时间的key中，**任意选择**数据淘汰
- `allkeys-random`：从所有key中，**任意选择数**据淘汰



### TTL(过期时间)

- `volatile-ttl`：从已设置过期时间的key中，挑选**将要过期**的数据淘汰
- `allkeys-random`：从所有key中，**任意选择数**据淘汰



### No-Enviction(驱逐)

- `noenviction（驱逐）`：当达到最大内存时直接返回错误，不覆盖或逐出任何数据

**Java类型所占字节数（或bit数）**

| 类型    | 存储(byte) | bit数(bit) | 取值范围                                                     |
| ------- | ---------- | ---------- | ------------------------------------------------------------ |
| int     | 4字节      | 4×8位      | 即 (-2)的31次方 ~ (2的31次方) - 1                            |
| short   | 2字节      | 2×8位      | 即 (-2)的15次方 ~ (2的15次方) - 1                            |
| long    | 8字节      | 8×8位      | 即 (-2)的63次方 ~ (2的63次方) - 1                            |
| byte    | 1字节      | 1×8位      | 即 (-2)的7次方 ~ (2的7次方) - 1，-128~127                    |
| float   | 4字节      | 4×8位      | float 类型的数值有一个后缀 F（例如：3.14F）                  |
| double  | 8字节      | 8×8位      | 没有后缀 F 的浮点数值（例如：3.14）默认为 double             |
| boolean | 1字节      | 1×8位      | true、false                                                  |
| char    | 2字节      | 2×8位      | Java中，只要是字符，不管是数字还是英文还是汉字，都占两个字节 |

**注意**：

- 英文的数字、字母或符号：1个字符 = 1个字节数
- 中文的数字、字母或符号：1个字符 = 2个字节数
- 计算机的基本单位：bit 。一个bit代表一个0或1，1个字节是8个bit
- 1TB=1024GB，1GB=1024MB，1MB=1024KB，1KB=1024B（字节，byte），1B=8b（bit，位）



## 线程模型

![Redis线程模型](png/Redis/Redis线程模型.png)

Redis内部使用文件事件处理器`File Event Handler`，这个文件事件处理器是单线程的所以Redis才叫做单线程的模型。它采用`I/O`多路复用机制同时监听多个`Socket`，将产生事件的`Socket`压入到内存队列中，事件分派器根据`Socket`上的事件类型来选择对应的事件处理器来进行处理。文件事件处理器包含5个部分：

- **多个Socket**
- **I/O多路复用程序**
- **Scocket队列**
- **文件事件分派器**
- **事件处理器**（连接应答处理器、命令请求处理器、命令回复处理器）

![Redis文件事件处理器](png/Redis/Redis文件事件处理器.png)



### 通信流程

客户端与redis的一次通信过程：

![Redis请求过程](png/Redis/Redis请求过程.png)

- **请求类型1**：`客户端发起建立连接的请求`
  - 服务端会产生一个`AE_READABLE`事件，`I/O`多路复用程序接收到`server socket`事件后，将该`socket`压入队列中
  - 文件事件分派器从队列中获取`socket`，交给连接应答处理器，创建一个可以和客户端交流的`socket01`
  - 将`socket01`的`AE_READABLE`事件与命令请求处理器关联

- **请求类型2**：`客户端发起set key value请求`
  - `socket01`产生`AE_READABLE`事件，`socket01`压入队列
  - 将获取到的`socket01`与命令请求处理器关联
  - 命令请求处理器读取`socket01`中的`key value`，并在内存中完成对应的设置
  - 将`socket01`的`AE_WRITABLE`事件与命令回复处理器关联

- **请求类型3**：`服务端返回结果`
  - `Redis`中的`socket01`会产生一个`AE_WRITABLE`事件，压入到队列中
  - 将获取到的`socket01`与命令回复处理器关联
  - 回复处理器对`socket01`输入操作结果，如`ok`。之后解除`socket01`的`AE_WRITABLE`事件与命令回复处理器的关联



### 文件事件处理器

- **基于 Reactor 模式开发了自己的网络事件处理器（文件事件处理器，file event handler）**
- 文件事件处理器 **使用 I/O 多路复用（multiplexing）程序来同时监听多个套接字**，并根据套接字目前执行的任务来为套接字关联不同的事件处理器
- 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时， 与操作相对应的文件事件就会产生， 这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件
- 文件事件处理器以单线程方式运行， 但通过使用 I/O 多路复用程序来监听多个套接字， 文件事件处理器既实现了高性能的网络通信模型， 又可以很好地与 redis 服务器中其他同样以单线程方式运行的模块进行对接， 这保持了 Redis 内部单线程设计的简单性



### I/O多路复用

I/O多路复用的I/O是指网络I/O，多路指多个TCP连接(即socket或者channel），复用指复用一个或几个线程。意思说一个或一组线程处理多个TCP连接。最大优势是减少系统开销小，不必创建过多的进程/线程，也不必维护这些进程/线程。
I/O多路复用使用两个系统调用(select/poll/epoll和recvfrom)，blocking I/O只调用了recvfrom；select/poll/epoll 核心是可以同时处理多个connection，而不是更快，所以连接数不高的话，性能不一定比多线程+阻塞I/O好,多路复用模型中，每一个socket，设置为non-blocking,阻塞是被select这个函数block，而不是被socket阻塞的。



**select机制**
**基本原理**
客户端操作服务器时就会产生这三种文件描述符(简称fd)：writefds(写)、readfds(读)、和exceptfds(异常)。select会阻塞住监视3类文件描述符，等有数据、可读、可写、出异常 或超时、就会返回；返回后通过遍历fdset整个数组来找到就绪的描述符fd，然后进行对应的I/O操作。
**优点**

- 几乎在所有的平台上支持，跨平台支持性好

**缺点**

- 由于是采用轮询方式全盘扫描，会随着文件描述符FD数量增多而性能下降
- 每次调用 select()，需要把 fd 集合从用户态拷贝到内核态，并进行遍历(消息传递都是从内核到用户空间)
- 默认单个进程打开的FD有限制是1024个，可修改宏定义，但是效率仍然慢。



**poll机制**
基本原理与select一致，也是轮询+遍历；唯一的区别就是poll没有最大文件描述符限制（使用链表的方式存储fd）。



**epoll机制**
**基本原理**
没有fd个数限制，用户态拷贝到内核态只需要一次，使用时间通知机制来触发。通过epoll_ctl注册fd，一旦fd就绪就会通过callback回调机制来激活对应fd，进行相关的io操作。epoll之所以高性能是得益于它的三个函数：

- `epoll_create()`：系统启动时，在Linux内核里面申请一个B+树结构文件系统，返回epoll对象，也是一个fd
- `epoll_ctl()`：每新建一个连接，都通过该函数操作epoll对象，在这个对象里面修改添加删除对应的链接fd, 绑定一个callback函数
- `epoll_wait()`：轮训所有的callback集合，并完成对应的IO操作

**优点**

- 没fd这个限制，所支持的FD上限是操作系统的最大文件句柄数，1G内存大概支持10万个句柄
- 效率提高，使用回调通知而不是轮询的方式，不会随着FD数目的增加效率下降
- 内核和用户空间mmap同一块内存实现(mmap是一种内存映射文件方法，即将一个文件或其它对象映射到进程的地址空间)



例子：100万个连接，里面有1万个连接是活跃，我们可以对比 select、poll、epoll 的性能表现：

- `select`：不修改宏定义默认是1024，则需要100w/1024=977个进程才可以支持 100万连接，会使得CPU性能特别的差
- `poll`：  没有最大文件描述符限制，100万个链接则需要100w个fd，遍历都响应不过来了，还有空间的拷贝消耗大量资源
- `epoll`:  请求进来时就创建fd并绑定一个callback，主需要遍历1w个活跃连接的callback即可，即高效又不用内存拷贝



### 执行效率高

**Redis是单线程模型为什么效率还这么高？**

- `纯内存操作`：数据存放在内存中，内存的响应时间大约是100纳秒，这是Redis每秒万亿级别访问的重要基础
- `非阻塞的I/O多路复用机制`：Redis采用epoll做为I/O多路复用技术的实现，再加上Redis自身的事件处理模型将epoll中的连接，读写，关闭都转换为了时间，不在I/O上浪费过多的时间
- `C语言实现`：距离操作系统更近，执行速度会更快
- `单线程避免切换开销`：单线程避免了多线程上下文切换的时间开销，预防了多线程可能产生的竞争问题

## 部署架构

### 单节点(Single)

**优点**

- 架构简单，部署方便
- 高性价比：缓存使用时无需备用节点(单实例可用性可以用 supervisor 或 crontab 保证)，当然为了满足业务的高可用性，也可以牺牲一个备用节点，但同时刻只有一个实例对外提供服务
- 高性能

**缺点**

- 不保证数据的可靠性
- 在缓存使用，进程重启后，数据丢失，即使有备用的节点解决高可用性，但是仍然不能解决缓存预热问题，因此不适用于数据可靠性要求高的业务
- 高性能受限于单核CPU的处理能力(Redis是单线程机制)，CPU为主要瓶颈，所以适合操作命令简单，排序/计算较少场景



### 主从复制(Replication)

**基本原理**

主从复制模式中包含一个主数据库实例（Master）与一个或多个从数据库实例（Slave），如下图：

![Redis主从复制模式(Replication)](png\Redis\Redis主从复制模式(Replication).png)

优缺点

![Redis主从复制模式(Replication)优缺点](png\Redis\Redis主从复制模式(Replication)优缺点.png)

### 哨兵(Sentinel)

Sentinel主要作用如下：

- **监控**：Sentinel 会不断的检查主服务器和从服务器是否正常运行
- **通知**：当被监控的某个Redis服务器出现问题，Sentinel通过API脚本向管理员或者其他的应用程序发送通知
- **自动故障转移**：当主节点不能正常工作时，Sentinel会开始一次自动的故障转移操作，它会将与失效主节点是主从关系的其中一个从节点升级为新的主节点，并且将其他的从节点指向新的主节点

![Redis哨兵模式(Sentinel)](png\Redis\Redis哨兵模式(Sentinel).png)

哨兵模式优缺点

![Redis哨兵模式(Sentinel)优缺点](png\Redis\Redis哨兵模式(Sentinel)优缺点.png)

### 集群(Cluster)

集群模式

![Redis集群模式(Cluster)](png\Redis\Redis集群模式(Cluster).png)

集群模式优缺点

![Redis集群模式(Cluster)优缺点](png\Redis\Redis集群模式(Cluster)优缺点.png)

### 主从复制原理

Redis的复制功能分为同步（sync）和命令传播（command propagate）两个操作。

　　**①、旧版同步**

　　当从节点发出 SLAVEOF 命令，要求从服务器复制主服务器时，从服务器通过向主服务器发送 SYNC 命令来完成。该命令执行步骤：

　　1、从服务器向主服务器发送 SYNC 命令

　　2、收到 SYNC 命令的主服务器执行 BGSAVE 命令，在后台生成一个 RDB 文件，并使用一个缓冲区记录从开始执行的所有写命令

　　3、当主服务器的 BGSAVE 命令执行完毕时，主服务器会将 BGSAVE 命令生成的 RDB 文件发送给从服务器，从服务器接收此 RDB 文件，并将服务器状态更新为RDB文件记录的状态。

　　4、主服务器将缓冲区的所有写命令也发送给从服务器，从服务器执行相应命令。

　　**②、命令传播**

　　当同步操作完成之后，主服务器会进行相应的修改命令，这时候从服务器和主服务器状态就会不一致。

　　为了让主服务器和从服务器保持状态一致，主服务器需要对从服务器执行命令传播操作，主服务器会将自己的写命令发送给从服务器执行。从服务器执行相应的命令之后，主从服务器状态继续保持一致。

　　总结：通过同步操作以及命令传播功能，能够很好的保证了主从一致的特性。

　　但是我们考虑一个问题，如果从服务器在同步主服务器期间，突然断开了连接，而这时候主服务器进行了一些写操作，这时候从服务器恢复连接，如果我们在进行同步，那么就必须将主服务器从新生成一个RDB文件，然后给从服务器加载，这样虽然能够保证一致性，但是其实断开连接之前主从服务器状态是保持一致的，不一致的是从服务器断开连接，而主服务器执行了一些写命令，那么从服务器恢复连接后能不能只要断开连接的哪些写命令，而不是整个RDB快照呢？

　　同步操作其实是一个非常耗时的操作，主服务器需要先通过 BGSAVE 命令来生成一个 RDB 文件，然后需要将该文件发送给从服务器，从服务器接收该文件之后，接着加载该文件，并且加载期间，从服务器是无法处理其他命令的。

　　为了解决这个问题，Redis从2.8版本之后，使用了新的同步命令 **PSYNC** 来代替 SYNC 命令。该命令的部分重同步功能用于处理断线后重复制的效率问题。也就是说当从服务器在断线后重新连接主服务器时，主服务器只将断开连接后执行的写命令发送给从服务器，从服务器只需要接收并执行这些写命令即可保持主从一致。

## 环境搭建

**Redis安装及配置**

Redis的安装十分简单，打开redis的官网 [http://redis.io](http://redis.io/) 。

- 下载一个最新版本的安装包，如 redis-version.tar.gz
- 解压 `tar zxvf redis-version.tar.gz`
- 执行 make (执行此命令可能会报错，例如确实gcc，一个个解决即可)

如果是 mac 电脑，安装redis将十分简单执行`brew install redis`即可。安装好redis之后，我们先不慌使用，先进行一些配置。打开`redis.conf`文件，我们主要关注以下配置：

```shell
port 6379             # 指定端口为 6379，也可自行修改 
daemonize yes         # 指定后台运行
```



### 单节点(Single)

安装好redis之后，我们来运行一下。启动redis的命令为 :

```shell
$ <redishome>/bin/redis-server path/to/redis.config
```

假设我们没有配置后台运行（即：daemonize no）,那么我们会看到如下启动日志：

```shell
93825:C 20 Jan 2019 11:43:22.640 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
93825:C 20 Jan 2019 11:43:22.640 # Redis version=5.0.3, bits=64, commit=00000000, modified=0, pid=93825, just started
93825:C 20 Jan 2019 11:43:22.640 # Configuration loaded
93825:S 20 Jan 2019 11:43:22.641 * Increased maximum number of open files to 10032 (it was originally set to 256).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 5.0.3 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6380
 |    `-._   `._    /     _.-'    |     PID: 93825
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'   
```



### 主从复制(Replication)

Redis主从配置非常简单，过程如下(演示情况下主从配置在一台电脑上)：

**第一步**：复制两个redis配置文件（启动两个redis，只需要一份redis程序，两个不同的redis配置文件即可）

```shell
mkdir redis-master-slave
cp path/to/redis/conf/redis.conf path/to/redis-master-slave master.conf
cp path/to/redis/conf/redis.conf path/to/redis-master-slave slave.conf
```

**第二步**：修改配置

```shell
## master.conf
port 6379
 
## master.conf
port 6380
slaveof 127.0.0.1 6379
```

**第三步**：分别启动两个redis

```shell
redis-server path/to/redis-master-slave/master.conf
redis-server path/to/redis-master-slave/slave.conf
```

启动之后，打开两个命令行窗口，分别执行 `telnet localhost 6379` 和 `telnet localhost 6380`，然后分别在两个窗口中执行 `info` 命令,可以看到：

```shell
# Replication
role:master
 
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
```

主从配置没问题。然后在master 窗口执行 set 之后，到slave窗口执行get，可以get到，说明主从同步成功。这时，我们如果在slave窗口执行 set ，会报错:

```shell
-READONLY You can't write against a read only replica.
```

因为从节点是只读的。



### 哨兵(Sentinel)

Sentinel是用来监控主从节点的健康情况。客户端连接Redis主从的时候，先连接Sentinel，Sentinel会告诉客户端主Redis的地址是多少，然后客户端连接上Redis并进行后续的操作。当主节点挂掉的时候，客户端就得不到连接了因而报错了，客户端重新向Sentinel询问主master的地址，然后客户端得到了[新选举出来的主Redis]，然后又可以愉快的操作了。



**哨兵sentinel配置**

为了说明sentinel的用处，我们做个试验。配置3个redis（1主2从），1个哨兵。步骤如下：

```shell
mkdir redis-sentinel
cd redis-sentinel
cp redis/path/conf/redis.conf path/to/redis-sentinel/redis01.conf
cp redis/path/conf/redis.conf path/to/redis-sentinel/redis02.conf
cp redis/path/conf/redis.conf path/to/redis-sentinel/redis03.conf
touch sentinel.conf
```

上我们创建了 3个redis配置文件，1个哨兵配置文件。我们将 redis01设置为master，将redis02，redis03设置为slave。

```shell
vim redis01.conf
port 63791
 
vim redis02.conf
port 63792
slaveof 127.0.0.1 63791
 
vim redis03.conf
port 63793
slaveof 127.0.0.1 63791
 
vim sentinel.conf
daemonize yes
port 26379
sentinel monitor mymaster 127.0.0.1 63793 1   # 下面解释含义
```

上面的主从配置都熟悉，只有哨兵配置 sentinel.conf，需要解释一下：

```shell
mymaster        # 为主节点名字，可以随便取，后面程序里边连接的时候要用到
127.0.0.1 63793 # 为主节点的 ip,port
1               # 后面的数字 1 表示选举主节点的时候，投票数。1表示有一个sentinel同意即可升级为master
```



**启动哨兵**

上面我们配置好了redis主从，1主2从，以及1个哨兵。下面我们分别启动redis，并启动哨兵：

```shell
redis-server path/to/redis-sentinel/redis01.conf
redis-server path/to/redis-sentinel/redis02.conf
redis-server path/to/redis-sentinel/redis03.conf
 
redis-server path/to/redis-sentinel/sentinel.conf --sentinel
```

启动之后，可以分别连接到 3个redis上，执行info查看主从信息。



**模拟主节点宕机情况**

运行上面的程序（**注意，在实验这个效果的时候，可以将sleep时间加长或者for循环增多，以防程序提前停止，不便看整体效果**），然后将主redis关掉，模拟redis挂掉的情况。现在主redis为redis01,端口为63791

```shell
redis-cli -p 63791 shutdown
```



### 集群(Cluster)

上述所做的这些工作只是保证了数据备份以及高可用，目前为止我们的程序一直都是向1台redis写数据，其他的redis只是备份而已。实际场景中，单个redis节点可能不满足要求，因为：

- 单个redis并发有限
- 单个redis接收所有数据，最终回导致内存太大，内存太大回导致rdb文件过大，从很大的rdb文件中同步恢复数据会很慢

所以需要redis cluster 即redis集群。Redis 集群是一个提供在**多个Redis间节点间共享数据**的程序集。Redis集群并不支持处理多个keys的命令，因为这需要在不同的节点间移动数据，从而达不到像Redis那样的性能,在高负载的情况下可能会导致不可预料的错误。Redis 集群通过分区来提供**一定程度的可用性**，在实际环境中当某个节点宕机或者不可达的情况下继续处理命令.。Redis 集群的优势：

- 自动分割数据到不同的节点上
- 整个集群的部分节点失败或者不可达的情况下能够继续处理命令

为了配置一个redis cluster,我们需要准备至少6台redis，为啥至少6台呢？我们可以在redis的官方文档中找到如下一句话：

> Note that the minimal cluster that works as expected requires to contain at least three master nodes. 

因为最小的redis集群，需要至少3个主节点，既然有3个主节点，而一个主节点搭配至少一个从节点，因此至少得6台redis。然而对我来说，就是复制6个redis配置文件。本实验的redis集群搭建依然在一台电脑上模拟。



**配置 redis cluster 集群**

上面提到，配置redis集群需要至少6个redis节点。因此我们需要准备及配置的节点如下：

```shell
# 主：redis01  从 redis02    slaveof redis01
# 主：redis03  从 redis04    slaveof redis03
# 主：redis05  从 redis06    slaveof redis05
mkdir redis-cluster
cd redis-cluster
mkdir redis01 到 redis06 6个文件夹
cp redis.conf 到 redis01 ... redis06
# 修改端口, 分别配置3组主从关系
```



**启动redis集群**

上面的配置完成之后，分别启动6个redis实例。配置正确的情况下，都可以启动成功。然后运行如下命令创建集群：

```shell
redis-5.0.3/src/redis-cli --cluster create 127.0.0.1:6371 127.0.0.1:6372 127.0.0.1:6373 127.0.0.1:6374 127.0.0.1:6375 127.0.0.1:6376 --cluster-replicas 1
```

**注意**，这里使用的是ip:port，而不是 domain:port ，因为我在使用 localhost:6371 之类的写法执行的时候碰到错误：

```shell
ERR Invalid node address specified: localhost:6371
```

执行成功之后，连接一台redis，执行 cluster info 会看到类似如下信息：

```shell
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:1515
cluster_stats_messages_pong_sent:1506
cluster_stats_messages_sent:3021
cluster_stats_messages_ping_received:1501
cluster_stats_messages_pong_received:1515
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:3021
```

我们可以看到`cluster_state:ok`，`cluster_slots_ok:16384`，`cluster_size:3`。



## 拓展方案

### 分区(Partitioning)

指在面临**单机**的**存储空间**瓶颈时，即将全部数据分散在多个Redis实例中，每个实例不需要关联，可以是完全独立的。



**使用方式**

- 客户端处理
  和传统的数据库分库分表一样，可以从**key**入手，先进行计算，找到对应数据存储的实例在进行操作。
  - **范围角度**，比如orderId:1~orderId:1000放入实例1，orderId:1001~orderId:2000放入实例2
  - **哈希计算**，就像我们的**hashmap**一样，用hash函数加上位运算或者取模，高级玩法还有一致性Hash等操作，找到对应的实例进行操作
- 使用代理中间件
  我们可以开发独立的代理中间件，屏蔽掉处理数据分片的逻辑，独立运行。当然Redis也有优秀的代理中间件，譬如Twemproxy，或者codis，可以结合场景选择是否使用



**缺点**

- **无缘多key操作**，key都不一定在一个实例上，那么多key操作或者多key事务自然是不支持
- **维护成本**，由于每个实例在物理和逻辑上，都属于单独的一个节点，缺乏统一管理
- **灵活性有限**，范围分片还好，比如hash+MOD这种方式，如果想**动态**调整Redis实例的数量，就要考虑大量数据迁移



### 主从(Master-Slave)

分区暂时能解决**单点**无法容纳的**数据量问题**，但是一个Key还是只在一个实例上。主从则将数据从**主节点**同步到**从节点**，然后可做**读写分离**，将读流量均摊在各个从节点，可靠性也能提高。**主从**(Master-Slave)也就是复制(Replication)方式。



**使用方式**

- 作为主节点的Redis实例，并不要求配置任何参数，只需要正常启动
- 作为从节点的实例，使用配置文件或命令方式`REPLICAOF 主节点Host 主节点port`即可完成主从配置



**缺点**

- slave节点都是**只读**的，如果**写流量**大的场景，就有些力不从心
- **故障转移**不友好，主节点挂掉后，写处理就无处安放，需要**手工**的设定新的主节点，如使用`REPLICAOF no one` 晋升为主节点，再梳理其他slave节点的新主配置，相对来说比较麻烦



### 哨兵(Sentinel)

**主从**的手工故障转移，肯定让人很难接受，自然就出现了高可用方案-**哨兵**（Sentinel）。我们可以在主从架构不变的场景，直接加入**Redis Sentinel**，对节点进行**监控**，来完成自动的**故障发现**与**转移**。并且还能够充当**配置提供者**，提供主节点的信息，就算发生了故障转移，也能提供正确的地址。



**使用方式**

**Sentinel**的最小配置，一行即可：

```properties
sentinel monitor <主节点别名> <主节点host> <主节点端口> <票数>
```

只需要配置master即可，然后用``redis-sentinel <配置文件>`` 命令即可启用。哨兵数量建议在三个以上且为奇数。



**使用场景问题**

- 故障转移期间短暂的不可用，但其实官网的例子也给出了`parallel-syncs`参数来指定并行的同步实例数量，以免全部实例都在同步出现整体不可用的情况，相对来说要比手工的故障转移更加方便
- 分区逻辑需要自定义处理，虽然解决了主从下的高可用问题，但是Sentinel并没有提供分区解决方案，还需开发者考虑如何建设
- 既然是还是主从，如果异常的写流量搞垮了主节点，那么自动的“故障转移”会不会变成自动“灾难传递”，即slave提升为Master之后挂掉，又进行提升又被挂掉



### 集群(Cluster)

**Cluster**在分区管理上，使用了“**哈希槽**”(hash slot)这么一个概念，一共有**16384**个槽位，每个实例负责一部分**槽**，通过`CRC16（key）&16383`这样的公式，计算出来key所对应的槽位。



**使用方式**

配置文件

```properties
cluster-enabled yes
cluster-config-file "redis-node.conf"
```

启动命令

```bash
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
--cluster-replicas 1
```



**存在问题**

- 虽然是对分区良好支持，但也有一些分区的老问题。如如果不在同一个“槽”的数据，是没法使用类似mset的**多键操作**
- 在select命令页有提到, 集群模式下只能使用一个库，虽然平时一般也是这么用的，但是要了解一下
- 运维上也要谨慎，俗话说得好，“**使用越简单底层越复杂**”，启动搭建是很方便，使用时面对带宽消耗，数据倾斜等等具体问题时，还需人工介入，或者研究合适的配置参数

# Redis源码

## 如何阅读 Redis 源码？

在这篇文章中， 我将向大家介绍一种我认为比较合理的 Redis 源码阅读顺序， 希望可以给对 Redis 有兴趣并打算阅读 Redis 源码的朋友带来一点帮助。

**Redis简介**

redis全称REmote DIctionary Server，是一个由Salvatore Sanfilippo写的高性能key-value存储系统，其完全开源免费，遵守BSD协议。Redis与其他key-value缓存产品（如memcache）有以下几个特点。

- Redis支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。
- Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。
- Redis支持数据的备份，即master-slave模式的数据备份。

Redis的性能极高且拥有丰富的数据类型，同时，Redis所有操作都是原子性的，也支持对几个操作合并后原子性的执行。另外，Redis有丰富的扩展特性，它支持publish/subscribe, 通知,key 过期等等特性。Redis更为优秀的地方在于，它的代码风格极其精简，整个源码只有23000行，很有利于阅读和赏析！还在等什么呢？Start！

### 阅读数据结构实现

刚开始阅读 Redis 源码的时候， 最好从数据结构的相关文件开始读起， 因为这些文件和 Redis 中的其他部分耦合最少， 并且这些文件所实现的数据结构在大部分算法书上都可以了解到， 所以从这些文件开始读是最轻松的、难度也是最低的。

下表列出了 Redis 源码中， 各个数据结构的实现文件：

| 文件                                                         | 内容                        |
| :----------------------------------------------------------- | :-------------------------- |
| `sds.h` 和 `sds.c`                                           | Redis 的动态字符串实现。    |
| `adlist.h` 和 `adlist.c`                                     | Redis 的双端链表实现。      |
| `dict.h` 和 `dict.c`                                         | Redis 的字典实现。          |
| `redis.h` 中的 `zskiplist` 结构和 `zskiplistNode` 结构， 以及 `t_zset.c` 中所有以 `zsl` 开头的函数， 比如 `zslCreate` 、 `zslInsert` 、 `zslDeleteNode` ，等等。 | Redis 的跳跃表实现。        |
| `hyperloglog.c` 中的 `hllhdr` 结构， 以及所有以 `hll` 开头的函数。 | Redis 的 HyperLogLog 实现。 |

- 阅读Redis的数据结构部分，基本位于如下文件中：内存分配 zmalloc.c和zmalloc.h
- 动态字符串 sds.h和sds.c
- 双端链表 adlist.c和adlist.h
- 字典 dict.h和dict.c
- 跳跃表 server.h文件里面关于zskiplist结构和zskiplistNode结构，以及t_zset.c中所有zsl开头的函数，比如 zslCreate、zslInsert、zslDeleteNode等等。
- 基数统计 hyperloglog.c 中的 hllhdr 结构， 以及所有以 hll 开头的函数

### 阅读内存编码数据结构实现

在阅读完和数据结构有关的文件之后， 接下来就应该阅读内存编码（encoding）数据结构了。

和普通的数据结构一样， 内存编码数据结构基本上是独立的， 不和其他模块耦合， 但是区别在于：

- 上一步要读的数据结构， 比如双端链表、字典、HyperLogLog， 在算法书上或者相关的论文上都可以找到资料介绍。
- 而内存编码数据结构却不容易找到相关的资料， 因为这些数据结构都是 Redis 为了节约内存而专门开发出来的， 换句话说， 这些数据结构都是特制（adhoc）的， 除了 Redis 源码中的文档之外， 基本上找不到其他资料来了解这些特制的数据结构。

不过话又说回来， 虽然内存编码数据结构是 Redis 特制的， 但它们基本都和内存分配、指针操作、位操作这些底层的东西有关， 读者只要认真阅读源码中的文档， 并在有需要时， 画图来分析这些数据结构， 那么要完全理解这些内存编码数据结构的运作原理并不难， 当然这需要花一些功夫。

下表展示了 Redis 源码中， 各个内存编码数据结构的实现文件：

| 文件                       | 内容                           |
| :------------------------- | :----------------------------- |
| `intset.h` 和 `intset.c`   | 整数集合（intset）数据结构。   |
| `ziplist.h` 和 `ziplist.c` | 压缩列表（zip list）数据结构。 |

- 整数集合数据结构 intset.h和intset.c
- 压缩列表数据结构 ziplist.h和ziplist.c

### 阅读数据类型实现

在完成以上两个阅读步骤之后， 我们就读完了 Redis 六种不同类型的键（字符串、散列、列表、集合、有序集合、HyperLogLog）的所有底层实现结构了。

接下来， 为了知道 Redis 是如何通过以上提到的数据结构来实现不同类型的键， 我们需要阅读实现各个数据类型的文件， 以及 Redis 的对象系统文件， 这些文件包括：

| 文件                                             | 内容                           |
| :----------------------------------------------- | :----------------------------- |
| `object.c`                                       | Redis 的对象（类型）系统实现。 |
| `t_string.c`                                     | 字符串键的实现。               |
| `t_list.c`                                       | 列表键的实现。                 |
| `t_hash.c`                                       | 散列键的实现。                 |
| `t_set.c`                                        | 集合键的实现。                 |
| `t_zset.c` 中除 `zsl` 开头的函数之外的所有函数。 | 有序集合键的实现。             |
| `hyperloglog.c` 中所有以 `pf` 开头的函数。       | HyperLogLog 键的实现。         |

- 对象系统 object.c
- 字符串键 t_string.c
- 列表建 t_list.c
- 散列键 t_hash.c
- 集合键 t_set.c
- 有序集合键 t_zset.c中除 zsl 开头的函数之外的所有函数
- HyperLogLog键 hyperloglog.c中所有以pf开头的函数

### 阅读数据库实现相关代码

在读完了 Redis 使用所有底层数据结构， 以及 Redis 是如何使用这些数据结构来实现不同类型的键之后， 我们就可以开始阅读 Redis 里面和数据库有关的代码了， 它们分别是：

| 文件                                                   | 内容                             |
| :----------------------------------------------------- | :------------------------------- |
| `redis.h` 文件中的 `redisDb` 结构， 以及 `db.c` 文件。 | Redis 的数据库实现。             |
| `notify.c`                                             | Redis 的数据库通知功能实现代码。 |
| `rdb.h` 和 `rdb.c`                                     | Redis 的 RDB 持久化实现代码。    |
| `aof.c`                                                | Redis 的 AOF 持久化实现代码。    |

选读

Redis 有一些独立的功能模块， 这些模块可以在完成第 4 步之后阅读， 它们包括：

| 文件                                                         | 内容                                            |
| :----------------------------------------------------------- | :---------------------------------------------- |
| `redis.h` 文件的 `pubsubPattern` 结构，以及 `pubsub.c` 文件。 | 发布与订阅功能的实现。                          |
| `redis.h` 文件的 `multiState` 结构以及 `multiCmd` 结构， `multi.c` 文件。 | 事务功能的实现。                                |
| `sort.c`                                                     | `SORT` 命令的实现。                             |
| `bitops.c`                                                   | `GETBIT` 、 `SETBIT` 等二进制位操作命令的实现。 |

- 数据库实现 redis.h文件中的redisDb结构，以及db.c文件
- 通知功能 notify.c
- RDB持久化 rdb.c
- AOF持久化 aof.c

以及一些独立功能模块的实现

- 发布和订阅 redis.h文件的pubsubPattern结构，以及pubsub.c文件
- 事务 redis.h文件的multiState结构以及multiCmd结构，multi.c文件

### 阅读客户端和服务器的相关代码

在阅读完数据库实现代码， 以及 RDB 和 AOF 两种持久化的代码之后， 我们可以开始阅读客户端和 Redis 服务器本身的实现代码， 和这些代码有关的文件是：

| 文件                                                         | 内容                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `ae.c` ，以及任意一个 `ae_*.c` 文件（取决于你所使用的多路复用库）。 | Redis 的事件处理器实现（基于 Reactor 模式）。                |
| `networking.c`                                               | Redis 的网络连接库，负责发送命令回复和接受命令请求， 同时也负责创建/销毁客户端， 以及通信协议分析等工作。 |
| `redis.h` 和 `redis.c` 中和单机 Redis 服务器有关的部分。     | 单机 Redis 服务器的实现。                                    |

如果读者能完成以上 5 个阅读步骤的话， 那么恭喜你， 你已经了解了单机的 Redis 服务器是怎样处理命令请求和返回命令回复， 以及是 Redis 怎样操作数据库的了， 这是 Redis 最重要的部分， 也是之后继续阅读多机功能的基础。

选读

Redis 有一些独立的功能模块， 这些模块可以在完成第 5 步之后阅读， 它们包括：

| 文件          | 内容                 |
| :------------ | :------------------- |
| `scripting.c` | Lua 脚本功能的实现。 |
| `slowlog.c`   | 慢查询功能的实现。   |
| `monitor.c`   | 监视器功能的实现。   |

- 事件处理模块 ae.c/ae_epoll.c/ae_evport.c/ae_kqueue.c/ae_select.c
- 网路链接库 anet.c和networking.c
- 服务器端 redis.c
- 客户端 redis-cli.c
- 这个时候可以阅读下面的独立功能模块的代码实现
- lua脚本 scripting.c
- 慢查询 slowlog.c
- 监视 monitor.c

### 阅读多机功能的实现

在弄懂了 Redis 的单机服务器是怎样运作的之后， 就可以开始阅读 Redis 多机功能的实现代码了， 和这些功能有关的文件为：

| 文件            | 内容                        |
| :-------------- | :-------------------------- |
| `replication.c` | 复制功能的实现代码。        |
| `sentinel.c`    | Redis Sentinel 的实现代码。 |
| `cluster.c`     | Redis 集群的实现代码。      |

注意， 因为 Redis Sentinel 用到了复制功能的代码， 而集群又用到了复制和 Redis Sentinel 的代码， 所以在阅读这三个模块的时候， 记得先阅读复制模块， 然后阅读 Sentinel 模块， 最后才阅读集群模块， 这样理解起来就会更得心应手。

- 复制功能 replication.c
- Redis Sentinel sentinel.c
- 集群 cluster.c

如果你连这三个模块都读完了的话， 那么恭喜你， 你已经读完了 Redis 单机功能和多机功能的所有代码了！

其他代码文件介绍

关于测试方面的文件有：

- memtest.c 内存检测
- redis_benchmark.c 用于redis性能测试的实现。
- redis_check_aof.c 用于更新日志检查的实现。
- redis_check_dump.c 用于本地数据库检查的实现。
- testhelp.c 一个C风格的小型测试框架。

一些工具类的文件如下：

- bitops.c GETBIT、SETBIT 等二进制位操作命令的实现
- debug.c 用于调试时使用
- endianconv.c 高低位转换，不同系统，高低位顺序不同
- help.h  辅助于命令的提示信息
- lzf_c.c 压缩算法系列
- lzf_d.c  压缩算法系列
- rand.c 用于产生随机数
- release.c 用于发布时使用
- sha1.c sha加密算法的实现
- util.c  通用工具方法
- crc64.c 循环冗余校验
- sort.c SORT命令的实现
- 一些封装类的代码实现：
- bio.c background I/O的意思，开启后台线程用的
- latency.c 延迟类
- migrate.c 命令迁移类，包括命令的还原迁移等
- pqsort.c  排序算法类
- rio.c redis定义的一个I/O类
- syncio.c 用于同步Socket和文件I/O操作

整个Redis的源码分类大体上如上所述了，接下来就按照既定的几个阶段一一去分析Redis这款如此优秀的源代码吧！

**基础篇——Redis基础知识入门**

1.认识NoSql 

2.安装和运行Redis 

3.Redis常见数据结构和命令 

4.Jedis客户端 

5.SpringDataRedis

**实战篇——覆盖Redis 80% 应用场景**

1.基于Redis的短信登录方案（共享session）

2.Redis缓存解决方案（缓存、缓存更新策略、缓存雪崩、穿透、热点key）

3.Redis在秒杀中的各种应用（分布式锁、Lua脚本、消息队列）

4.Redis在社交APP的应用（好友、粉丝、消息推送）

5.Redis的GEO功能（附近的店铺）

6.Redis的BitMap功能（签到功能）

7.Redis的HyperLogLog功能（数据统计）

**高级篇——应对企业20%复杂工作**

1.Redis持久化  

2.Redis主从  

3.Redis集群 

4.Redis哨兵  

5.Redis最佳实践

**原理篇——面试能力提升**

1.Redis底层数据结构  

2.Redis的线程模型  

3.Redis的通信协议  

4.Redis的内存淘汰策略  

5.Redis的热点面试题

# 常见问题

**题目**：保证Redis 中的 20w 数据都是热点数据 说明是 被频繁访问的数据，并且要保证Redis的内存能够存放20w数据，要计算出Redis内存的大小。

- **保留热点数据：**对于保留 Redis 热点数据来说，我们可以使用 Redis 的内存淘汰策略来实现，可以使用**allkeys-lru淘汰策略，**该淘汰策略是从 Redis 的数据中挑选最近最少使用的数据删除，这样频繁被访问的数据就可以保留下来了

- **保证 Redis 只存20w的数据：**1个中文占2个字节，假如1条数据有100个中文，则1条数据占200字节，20w数据 乘以 200字节 等于 4000 字节（大概等于38M）;所以要保证能存20w数据，Redis 需要38M的内存



**题目：MySQL里有2000w数据，redis中只存20w的数据，如何保证redis中的数据都是热点数据?**

限定 Redis 占用的内存，Redis 会根据自身数据淘汰策略，加载热数据到内存。所以，计算一下 20W 数据大约占用的内存，然后设置一下 Redis 内存限制即可。



**题目：假如Redis里面有1亿个key，其中有10w个key是以某个固定的已知的前缀开头的，如果将它们全部找出来？**

使用 keys 指令可以扫出指定模式的 key 列表。对方接着追问：如果这个 Redis 正在给线上的业务提供服务，那使用 keys 指令会有什么问题？这个时候你要回答 Redis 关键的一个特性：Redis 的单线程的。keys 指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。这个时候可以使用 scan 指令，scan 指令可以无阻塞地提取出指定模式的 key 列表，但是会有一定的重复概率，在客户端做一次去重就可以了，但是整体所花费的时间会比直接用 keys 指令长。

