---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码Client
客户端模块负责与MySQL服务器进行通信，接收和发送SQL语句。

## 客户端库
client。

客户端库包括mysql.cc（mysql可执行文件的源代码）和其他实用程序。 MySQL参考手册中提到了大多数实用程序。通常，这些是独立的C程序，它们在“客户端模式”下运行，即它们称为服务器。

目录中的C程序文件为：
```
get_password.c-从控制台要求输入密码
mysql.cc ---“ MySQL命令工具”
mysqladmin.cc ---维护MySQL数据库
mysqlcheck.c ---检查所有数据库，检查连接等。
mysqldump.c-将表的内容作为SQL语句转储，适合于备份MySQL数据库
mysqlimport.c ---将不同格式的文本文件导入表中
mysqlmanager-pwgen.c --- pwgen代表“密码生成”（当前未维护）
mysqlmanagerc.c --- mysql管理器的入口点（当前未维护）
mysqlshow.c ---显示数据库，表或列
mysqltest.c --- mysql-test套件使用的测试程序，mysql-test-run
```

## 客户端入口
client目录下的mysql.cc是MySQL客户端的主要入口点。对于 mysql 客户端，源码保存在 client/mysql.cc 文件中，下面是 main() 函数的主要执行流程。
```
main()
 |-sql_connect()
 | |-sql_real_connect()
 |   |-mysql_init()                             # 调用MySQL初始化
 |   |-mysql_options()                          # 设置链接选项
 |   |-mysql_real_connect()                     # sql-common/client.c
 |     |-connect_sync_or_async()                # 通过该函数尝试链接
 |     | |-my_connect()                         # 实际通过该函数建立链接
 |     |-cli_safe_read()                        # 等待第一个handshake包
 |     |-run_plugin_auth()                      # 通过插件实现认证
 |
 |-put_info()                                   # 打印客户端的欢迎信息
 |-read_and_execute()                           # 开始等待输入、执行SQL
```
客户端最终会调用 mysql_real_connect()，实际调用的是 cli_mysql_real_connect()，通过该函数建立链接，其中认证方式可以通过 run_plugin_auth() 时用插件实现。

然后，会输出一系列的欢迎信息，并通过 read_and_execute() 执行 SQL 命令。

在 MySQL 客户端执行时，并非所有的命令都是需要发送到服务端的，其中有一个数组定义了常见的命令。
```
static COMMANDS commands[] = {
  { "?",      '?', com_help,   1, "Synonym for `help'." },
  { "clear",  'c', com_clear,  0, "Clear the current input statement."},
  ... ...
};
```
每次读取一行都会通过 find_command() 函数进行检测，如果满足对应的命令，且对应的函数变量非空，则直接执行，如 clear，此时不需要输入分号即可；如果没有找到，则必须要等待输入分号。
```
int read_and_execute(bool interactive)
{
    while (!aborted) {
        if (!interactive) {                               // 是否为交互模式
            ... ...   // 非交互模式，直接执行
        } else {                                          // 交互模式
            char *prompt = ...;                           // 首先会设置提示符
            line = readline(prompt);                      // 从命令行读取
            if ( ... && (com= find_command(line))) {      // 从commands[]中查找
                (*com->func)(&glob_buffer,line);          // 如果是help、edit等指令，则直接执行
            }
            add_line(...);                                // 常见的SQL，最终在此执行
        }
    }
}
 
int com_go(String *buffer,char *line)
{
    timer=start_timer();                                                // 设置时间
    error= mysql_real_query_for_lazy(buffer->ptr(),buffer->length());   // 执行查询SQL
    do {
        // 获取结果
    } while(!(err= mysql_next_result(&mysql)));
}
```
在 add_line() 函数中，最终会调用 com_go() 函数，该函数是执行的主要函数，会最终调用 MySQL API 执行相应的 SQL、返回结果、输出时间等统计信息。

## 服务端
服务端通过 network_init() 执行一系列初始化之后，会阻塞在 handle_connections_sockets() 函数的 select()/poll() 函数处。

对于 one_thread_per_connection 这种方式，会新建一个线程执行 handle_one_connection() 。
```
handle_one_connection()
 |-thd_prepare_connection()
   |-login_connection()
     |-check_connection()
       |-acl_authenticate()
```
源码内容如下。
```
/* sql/sql_connect.cc */
int check_connection(THD *thd)
{
    if (!thd->main_security_ctx.host) {  // 通过TCP/IP连接，或者本地用-h 127.1
         if (acl_check_host(...))        // 检查hostname
    } else {                             // 使用unix sock连接，不会进行检测
        ... ...
    }
    return acl_authenticate(thd, connect_errors, 0)
}
 
/* sql/sql_acl.cc */
bool acl_authenticate(THD *thd, uint connect_errors, uint com_change_user_pkt_len)
{
    if (command == COM_CHANGE_USER) {
 
    } else {
        do_auth_once()                      // 执行认证模式
 
    }
}
```
在 acl_check_host() 会检查两个对象，一个是 hash 表 acl_check_hosts；另一个是动态数组 acl_wild_hosts 。这2个对象是在启动的时候，通过 init_check_host() 从 mysq.users 表里读出并加载的，其中 acl_wild_hosts 用于存放有统配符的主机，acl_check_hosts 存放具体的主机。

最终会调用 acl_authenticate() 这是主要的认证函数。

## 插件实现
MySQL 的认证授权采用插件实现。

默认采用 mysql_native_password 插件，也就是使用 native_password_auth_client() 作用户端的认证，实际有效的函数是 scramble()。

上述的函数通过用户输入的 password、服务器返回的 scramble 生成 reply，返回给服务端；可以通过 password('string') 查看加密后的密文。

以 plugin/auth/ 目录下的插件为例，在启动服务器时，可添加 --plugin-load=auth_test_plugin.so参数自动加载相应的授权插件。
```
----- 获得foobar的加密格式
mysql> select password('foobar');
----- 旧的加密格式
mysql> select old_password('foobar');
----- 默认方式
mysql> create user 'foobar2'@'localhost' identified via mysql_native_password using 'xxx';
 
----- 也可以动态加载
mysql> install plugin test_plugin_server soname 'auth_test_plugin.so';
----- 查看当前支持的插件
mysql> select * from information_schema.plugins where plugin_type='authentication';
 
mysql> create user 'foobar'@'localhost' identified with test_plugin_server;
mysql> SET PASSWORD FOR 'foobar'@'localhost'=PASSWORD('new_password');
mysql> DROP USER 'foobar'@'localhost';
mysql> FLUSH PRIVILEGES;
mysql> SELECT host, user, password, plugin FROM mysql.user;
```
在 plugin 目录下有很多 auth 插件可供参考，详细可参考官网 Writing Authentication Plugins 。

总结
在如下列举客户端与服务端的详细交互过程，其中客户端代码在 client 目录下。
```
### Client(mysql)  ###                       ### Server(mysqld) ###
----------------------------------------     --------------------------------------------------
main()                                       mysqld_main()
 |-sql_connect()                              |-init_ssl()
 | |-sql_real_connect() {for(;;)}             |-network_init()
 |   |-mysql_init()                           |-handle_connections_sockets()
 |   |-init_connection_options()                |-select()/poll()
 |   |-mysql_real_connect()                     |
 |     |-cli_mysql_real_connect()               |
 |       |-socket()                             |
 |       |-vio_new()                            |
 |       |-vio_socket_connect()                 |
 |       | |-mysql_socket_connect()             |
 |       |   |-connect()                        |
 |       |   |                                  |
 |       |   |        [Socket Connect]          |
 |       |   |>>==========>>==========>>======>>|
 |       |                                      |-accept()
 |       |-vio_keepalive()                      |-vio_new()
 |       |-my_net_set_read_timeout()            |-my_net_init()
 |       |-my_net_set_write_timeout()           |-create_new_thread()
 |       |-vio_io_wait()                          |-handle_one_connection()    {新线程}
 |       |                                          |-thd_prepare_connection() {for(;;)}
 |       |                                          | |-login_connection()
 |       |                                          |   |-check_connection()
 |       |                                          |     |-acl_check_host()
 |       |                                          |     |-vio_keepalive()
 |       |                                          |     |-acl_authenticate()
 |       |                                          |       |-do_auth_once()
 |       |                                          |       | |-native_password_authenticate()  {插件实现}
 |       |                                          |       |   |-create_random_string()
 |       |                                          |       |   |-send_server_handshake_packet()
 |       |                                          |       |   |
 |       |              [Handshake Initialization]  |       |   |
 |       |<<==<<==========<<==========<<==========<<==========<<|
 |       |-cli_safe_read()                          |       |   |-my_net_read()
 |       |-run_plugin_auth()                        |       |   |
 |       | |-native_password_auth_client()          |       |   |
 |       |   |-scramble()                           |       |   |
 |       |     |-my_net_write()                     |       |   |
 |       |     |                                    |       |   |
 |       |     |            [Client Authentication] |       |   |
 |       |     |>>==========>>==========>>==========>>========>>|
 |       |                                          |       |   |-check_scramble()
 |       |                                          |       |-mysql_change_db()
 |       |                                          |       |-my_ok()
 |       |                      [OK]                |       |
 |       |<<==========<<==========<<==========<<==========<<|
 |       |-cli_safe_read()                          |
 |                                                  |
 |                                                  |
 |                                                  |
 |                                                  |
 |-put_info() {Welcome Info}                        |
 |-read_and_execute() [for(;;)]                     |
                                                    |-thd_is_connection_alive()  [while()]
                                                    |-do_command()
```








