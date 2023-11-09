---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL系统变量variables

## 概述
mysql维护一系列系统变量，每个变量每个变量控制mysql server或者client的一种或几种行为。变量的具体细节可以参照：https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html

通常使用配置文件、命令行参数、客户端连接修改的方式来修改系统变量的值，后面会具体分析变量的定义、装载、更新等一系列流程。

## 源码分析
定义
mysql的系统变量以如下方式定义：
1. 定义一个类的对象或者全局结构体变量来存放系统变量的一些属性（sys_vars.cc）
2. 定义一个全局变量来存放系统变量的值

系统变量类
mysql为每种类型的系统变量均定义了一个系统的类，这里以bool型的系统变量为例
```
/* 所有的系统变量类均为类sys_var的派生类 */
class Sys_var_charptr: public sys_var
{
    /* 构造函数，将参数传入到sys_var的构造函数中 */
    Sys_var_mybool(...) :sys_var(&all_sys_vars...)

    /* 每个类重载的更新函数、校验函数等等 */
    bool session_update(THD *thd, set_var *var)
    bool global_update(THD *thd, set_var *var)
    bool do_check(THD *thd, set_var *var)
    ......
}

 /*所有系统变量的父类 */
class sys_var                                
{
   /* 初始化的更新函数和检查函数 */
   typedef bool (*on_check_function)(sys_var *self, THD *thd, set_var *var);
   typedef bool (*on_update_function)(sys_var *self, THD *thd, enum_var_type type);

   /* 存放变量的最大值、最小值、是否允许命令行等值 */
   /* 后续用于装载系统变量时使用的结构体 */
   my_option option;
   ......
   ......
}
```
系统变量（系统变量类对象）
```
/* 定义一个全局变量用于存放系统变量的值（mysqld.cc）*/
char *mysql_home_ptr;


/* 定义base_dir系统变量，定义一个char *型的系统变量（sys_vars.cc）*/
static Sys_var_charptr Sys_basedir(                                                
    "basedir",                                                                                                               /*变量名称 */
    "Path to installation directory. All paths are usually resolved relative to this",        /*变量说明 */
    READ_ONLY                                           /*设置变量的flag，设置只读           */                                                  
    GLOBAL_VAR(mysql_home_ptr),           /*设置变量的flag，设置为global， 同时设置该变量的值存放在变量mysql_home_ptr*/
    CMD_LINE(REQUIRED_ARG, 'b'),         /*设置命令行属性，指定可以使用命令行设定，同时可以使用-b来设置变量*/
    IN_FS_CHARSET,                                  /*字符集相关属性*/
    DEFAULT(0));                                         /*默认值*/
```
关于类的初始化不再具体描述，查看构造函数即可。

## 全局变量
静态的声明一些全局变量，类型与sys_var中option类型相同，这类系统变量是命令行使用的系统变量。
```
struct my_option my_long_early_options[]=                                        /*初始化需要加载的系统变量*/
{
   ...
  {"bootstrap", OPT_BOOTSTRAP, "Used by mysql installation scripts.", 0, 0, 0,
   GET_NO_ARG, NO_ARG, 0, 0, 0, 0, 0, 0},
   ...
}

struct my_option my_long_options[]=
{
  ...
  {"autocommit", 0, "Set default value for autocommit (0 or 1)",
   &opt_autocommit, &opt_autocommit, 0,
   GET_BOOL, OPT_ARG, 1, 0, 0, 0, 0, NULL},
  ...
}
```
对于一些类型的系统变量类不支持命令行，需要在这个全局变量中增加同名的变量。、
如Sys_var_bit类不允许执行命令行，那么使用这个类定义的对象autocommit，就需要在全局变量my_long_options中增加同名的变量来支持命令行参数。
```
class Sys_var_bit: public Sys_var_typelib                  
{
  Sys_var_bit(const char *name_arg,......
  {
    ...
    DBUG_ASSERT(getopt.id == -1); /* force NO_CMD_LINE  不支持命令行  */
    ...
  }
}
```
装载
系统变量类对象
系统变量类在定义一个对象时，在调用构造函数（父类sys_var的构造函数）的过程中，会将该变量存储到一个全局的链表中
```
class Sys_var_charptr: public sys_var
| Sys_var_charptr(const char *name_arg,
    | sys_var(&all_sys_vars, name_arg, comment, flag_args, off, getopt.id,
    |    if (chain->last)  chain->last->next= this;
    |    else  chain->first= this; chain->last= this;
```
静态全局变量
静态全局变量是不装载到系统变量链表中，只是在启动时将命令行参数的值更新到系统变量中。show variables是看不到这里声明的系统变量的。

初始化装载
创建一个全局的hash桶，用来存放所有的系统变量。

sever系统变量
```
mysql_main
| sys_var_init
      /* 初始化一个全局的系统变量hash桶*/
    | if (my_hash_init(&system_variable_hash, system_charset_info, 100, 0,
      /* 将所有的系统变量存放到全局hash桶中*/
    | mysql_add_sys_var_chain(all_sys_vars.first)
        for (var= all_sys_vars.first; var; var= var->next)
           my_hash_insert(&system_variable_hash, (uchar*) var)
```
插件系统变量
```
/* 启动或者装载插件*/
......                                                                              
| mysql_install_plugin                                                   /* 装载插件函数*/
    | plugin_add
       test_plugin_options
          mysql_add_sys_var_chain(plugin->system_vars)       /* 增加插件的所有变量*/

/* 卸载插件时*/
......                                                                              
| mysql_uninstall_plugin
    | reap_plugins
         mysql_del_sys_var_chain(plugin->system_vars)         /* 删除插件的所有变量*/

```
更新
配置文件和命令行更新
配置文件中和命令行中配置的系统变量如何生效这里不再重复介绍

客户端修改系统变量
以客户端设置变量为例，sql语句为set xxxx_var = ‘example’;

解析SQL语句，在解析过程中，判断变量名是否为系统变量
```
（sql_yacc.yy）
set 
SET start_option_value_list
option_value_no_option_type option_value_list_continued
internal_variable_name   equal    set_expr_or_default     
$$= NEW_PTN PT_internal_variable_name_1d($1);

/*定义类PT_internal_variable_name_1d的对象，然后依次嵌套调用函数contextualize */
class PT_internal_variable_name_1d : public PT_internal_variable_name
{
  virtual bool contextualize(Parse_context *pc)
  | if (find_sys_var_null_base(thd, &value))
      | find_sys_var(thd, tmp->base_name.str, tmp->base_name.length);
          | var= intern_find_sys_var(str, length)    
                /*在全局hansh桶system_variable_hash中查找变量 */
              | var= (sys_var*) my_hash_search(&system_variable_hash,......

}
```
解析后生成需要更新的系统变量的链表，这里的链表元素是使用类set_var的对象：
```
class set_var :public set_var_base                     /*用来操作系统变量的类 */
{
    ......
    sys_var *var;                                                   /* 系统变量    */
    Item *value;                                                     /* 解析出需要更新的值 */
    ...
    int check(THD *thd);                                        /* 校验函数 */
    int update(THD *thd);                                      /* 更新函数 */
}
```
使用类set_var对系统变量进行更新
```
......
mysql_parse
  /* 解析SQL语句，生成set的系统变量链表（见上） */
| err= parse_sql(thd, parser_state, NULL);
  /* 执行SQL语句 */
| mysql_execute_command
    case SQLCOM_SET_OPTION:
    if (!(res= sql_set_variables(thd, lex_var_list)))
        |if ((error= var->check(thd)))                    /* set_var的方法check */
            /* 这里的var是set_var的成员变量sys_var，即系统变量 */
            if (var->is_readonly())                       /* 判断系统变量是否为只读 */
            if ((type == OPT_GLOBAL && check_global_access(thd, SUPER_ACL)))  /*全局变量需要super权限 */
        if (var->check_update_type(value->result_type()))        /*校验更新的类型 */
            var->check(thd, this)                                                     /*调用变量的校验函数，变量定义时设置 */
        | error|= var->update(thd);                    /* set_var的方法update */
               /* 如果有设置的值则设置，没有设置的值则设置成默认 */
        | return value ? var->update(thd, this) : var->set_default(thd, this);
```