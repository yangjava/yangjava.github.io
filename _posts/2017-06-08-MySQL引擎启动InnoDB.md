---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL引擎启动InnoDB


## InnoDB启动
在MySql中，InnoDB的启动流程其实是很重要的。一些更细节的问题，就藏在了这其中。在前面分析过整个数据库启动的流程，本篇就具体分析一下InnoDB引擎启动所做的各种动作。在这期间，分析一下对数据库索引的处理过程。在前面的分析中已经探讨过，今天重点分析一下数据引擎的启动和加载流程。

在MySql中，方向是朝着插件化发展，所以InnoDB本身也是做为一个插件进行引用的。通过学习已经知道，handlerton这个数据结构体是插件加载的一个具体的实例的方式。它其实可以理解成一个MySql和数据库引擎的一个中间层，通过其可以动态的控制相关引擎插件的载入和应用。

那么，本次启动流程的分析就要从InnoDB引擎做为插件加载的那一刻进行源码分析。

## 源码分析
Innodb启动过程如下：
- 初始化innobase_hton，它是一个handlerton类型的指针，以便在server层能够调用存储引擎的接口。
- Innodb相关参数的检车和初始化，包括系统表空间，临时表空间，undo表空间，redo文件，doublewrite文件等。
- innobase_start_or_create_for_mysql()创建或者启动 innobase。

插件加载第一件事当然是对插件相关内容进行定义：ha_innodb.cc,这是MySQL中申明存储引擎插件的标准过程。
```
mysql_declare_plugin(innobase){
    MYSQL_STORAGE_ENGINE_PLUGIN,
    &innobase_storage_engine,
    innobase_hton_name,
    PLUGIN_AUTHOR_ORACLE,
    "Supports transactions, row-level locking, and foreign keys",
    PLUGIN_LICENSE_GPL,
    innodb_init,   /* Plugin Init */
    nullptr,       /* Plugin Check uninstall */
    innodb_deinit, /* Plugin Deinit */
    INNODB_VERSION_SHORT,
    innodb_status_variables_export, /* status variables */
    innobase_system_variables,      /* system variables */
    nullptr,                        /* reserved */
    0,                              /* flags * /
},
    i_s_innodb_trx, i_s_innodb_cmp, i_s_innodb_cmp_reset, i_s_innodb_cmpmem,
    i_s_innodb_cmpmem_reset, i_s_innodb_cmp_per_index,
    i_s_innodb_cmp_per_index_reset, i_s_innodb_buffer_page,
    i_s_innodb_buffer_page_lru, i_s_innodb_buffer_stats,
    i_s_innodb_temp_table_info, i_s_innodb_metrics,
    i_s_innodb_ft_default_stopword, i_s_innodb_ft_deleted,
    i_s_innodb_ft_being_deleted, i_s_innodb_ft_config,
    i_s_innodb_ft_index_cache, i_s_innodb_ft_index_table, i_s_innodb_tables,
    i_s_innodb_tablestats, i_s_innodb_indexes, i_s_innodb_tablespaces,
    i_s_innodb_columns, i_s_innodb_virtual, i_s_innodb_cached_indexes,
    i_s_innodb_session_temp_tablespaces

    mysql_declare_plugin_end;

```
而在/include/mysql/plugin.h定义了：
```
#define mysql_declare_plugin(NAME)                                        \
  __MYSQL_DECLARE_PLUGIN(NAME, builtin_##NAME##_plugin_interface_version ,\
                         builtin_##NAME##_sizeof_struct_st_plugin,        \
                         builtin_##NAME##_plugin)

#define mysql_declare_plugin_end                 \
  , { 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 } \
  }

```
如果实在不明白，可以用GCC编译时采用预编译指令保留中间结果文件（.i）就可以看到这段代码是什么了，结果基本如下：
```
int builtin_innobase_plugin_interface_version= 0x0104;
int builtin_innobase_sizeof_struct_st_plugin= sizeof(struct st_mysql_plugin);
struct st_mysql_plugin builtin_innobase_plugin[]= {
{
  1,
  &innobase_storage_engine,
  innobase_hton_name,
  plugin_author,
  "Supports transactions, row-level locking, and foreign keys",
  1,
  innobase_init,
  __null,
  (5 << 8 | 6),
  innodb_status_variables_export,
  innobase_system_variables,
  __null,
  0,
},
i_s_innodb_trx,
i_s_innodb_locks,
i_s_innodb_lock_waits,
i_s_innodb_cmp,
i_s_innodb_cmp_reset,
i_s_innodb_cmpmem,
i_s_innodb_cmpmem_reset,
i_s_innodb_cmp_per_index,
i_s_innodb_cmp_per_index_reset,
i_s_innodb_buffer_page,
i_s_innodb_buffer_page_lru,
i_s_innodb_buffer_stats,
i_s_innodb_metrics,
i_s_innodb_ft_default_stopword,
i_s_innodb_ft_deleted,
i_s_innodb_ft_being_deleted,
i_s_innodb_ft_config,
i_s_innodb_ft_index_cache,
i_s_innodb_ft_index_table,
i_s_innodb_sys_tables,
i_s_innodb_sys_tablestats,
i_s_innodb_sys_indexes,
i_s_innodb_sys_columns,
i_s_innodb_sys_fields,
i_s_innodb_sys_foreign,
i_s_innodb_sys_foreign_cols,
i_s_innodb_sys_tablespaces,
i_s_innodb_sys_datafiles

,{0,0,0,0,0,0,0,0,0,0,0,0,0}};
_____

```
这个宏定义其实有两个主要的函数，即innodb_init和innodb_deinit，看名字就可以明白，前者是加载数据库引擎插件初始化用的，后者是制裁插件时使用的。主要看一下引擎初始化的函数：
```
/*********************************************************************//**
初始化Innodb 插件
Opens an InnoDB database.
@return 0 on success, 1 on failure */
static
int
innobase_init(
/*==========*/
    void    *p) /*!< in: InnoDB handlerton */
{
    static char current_dir[3];     /*!< Set if using current lib */
    int     err;
    char        *default_path;
    uint        format_id;
    ulong       num_pll_degree;
        // 初始化 innobase_hton，以便在server层能够调用Innodb的接口
    DBUG_ENTER("innobase_init");
    handlerton* innobase_hton= (handlerton*) p;
    innodb_hton_ptr = innobase_hton;
      
    innobase_hton->state = SHOW_OPTION_YES;
    innobase_hton->db_type = DB_TYPE_INNODB;
    innobase_hton->savepoint_offset = sizeof(trx_named_savept_t);
    innobase_hton->close_connection = innobase_close_connection;
    innobase_hton->kill_connection = innobase_kill_connection;
    innobase_hton->savepoint_set = innobase_savepoint;
    innobase_hton->savepoint_rollback = innobase_rollback_to_savepoint;
 
    innobase_hton->savepoint_rollback_can_release_mdl =
                innobase_rollback_to_savepoint_can_release_mdl;
 
    innobase_hton->savepoint_release = innobase_release_savepoint;
    innobase_hton->commit = innobase_commit;
    innobase_hton->rollback = innobase_rollback;
    innobase_hton->prepare = innobase_xa_prepare;
    innobase_hton->recover = innobase_xa_recover;
    innobase_hton->commit_by_xid = innobase_commit_by_xid;
    innobase_hton->rollback_by_xid = innobase_rollback_by_xid;
    innobase_hton->create = innobase_create_handler;
    innobase_hton->alter_tablespace = innobase_alter_tablespace;
    innobase_hton->drop_database = innobase_drop_database;
    innobase_hton->panic = innobase_end;
    innobase_hton->partition_flags= innobase_partition_flags;
 
    innobase_hton->start_consistent_snapshot =
        innobase_start_trx_and_assign_read_view;
 
    innobase_hton->flush_logs = innobase_flush_logs;
    innobase_hton->show_status = innobase_show_status;
    innobase_hton->fill_is_table = innobase_fill_i_s_table;
    innobase_hton->flags =
        HTON_SUPPORTS_EXTENDED_KEYS | HTON_SUPPORTS_FOREIGN_KEYS |
        HTON_SUPPORTS_TABLE_ENCRYPTION;
 
    innobase_hton->release_temporary_latches =
        innobase_release_temporary_latches;
        innobase_hton->replace_native_transaction_in_thd =
                innodb_replace_trx_in_thd;
    innobase_hton->data = &innodb_api_cb;
    innobase_hton->is_reserved_db_name= innobase_check_reserved_file_name;
 
    innobase_hton->is_supported_system_table=
        innobase_is_supported_system_table;
 
    innobase_hton->rotate_encryption_master_key =
        innobase_encryption_key_rotation;
 
    ut_a(DATA_MYSQL_TRUE_VARCHAR == (ulint)MYSQL_TYPE_VARCHAR);
 
#ifndef NDEBUG
    static const char   test_filename[] = "-@";
    char            test_tablename[sizeof test_filename
                + sizeof(srv_mysql50_table_name_prefix) - 1];
    if ((sizeof(test_tablename)) - 1
            != filename_to_tablename(test_filename,
                         test_tablename,
                         sizeof(test_tablename), true)
            || strncmp(test_tablename,
                   srv_mysql50_table_name_prefix,
                   sizeof(srv_mysql50_table_name_prefix) - 1)
            || strcmp(test_tablename
                  + sizeof(srv_mysql50_table_name_prefix) - 1,
                  test_filename)) {
 
        sql_print_error("tablename encoding has been changed");
        DBUG_RETURN(innobase_init_abort());
    }
#endif /* NDEBUG */
 
    /* Check that values don't overflow on 32-bit systems. */
    if (sizeof(ulint) == 4) {
        if (innobase_buffer_pool_size > UINT_MAX32) {
            sql_print_error(
                "innodb_buffer_pool_size can't be over 4GB"
                " on 32-bit systems");
 
            DBUG_RETURN(innobase_init_abort());
        }
    }
 
    os_file_set_umask(my_umask);
 
    /* Setup the memory alloc/free tracing mechanisms before calling
    any functions that could possibly allocate memory. */
    ut_new_boot();
 
    /* First calculate the default path for innodb_data_home_dir etc.,
    in case the user has not given any value.
 
    Note that when using the embedded server, the datadirectory is not
    necessarily the current directory of this program. */
 
    if (mysqld_embedded) {
        default_path = mysql_real_data_home;
    } else {
        /* It's better to use current lib, to keep paths short */
        current_dir[0] = FN_CURLIB;
        current_dir[1] = FN_LIBCHAR;
        current_dir[2] = 0;
        default_path = current_dir;
    }
 
    ut_a(default_path);
 
    fil_path_to_mysql_datadir = default_path;
    folder_mysql_datadir = fil_path_to_mysql_datadir;
 
    /* Set InnoDB initialization parameters according to the values
    read from MySQL .cnf file */
 
    /* The default dir for data files is the datadir of MySQL
           默认的数据文件目录
        */
    srv_data_home = innobase_data_home_dir
        ? innobase_data_home_dir : default_path;
 
    /*--------------- Shared tablespaces -------------------------
          共享表空间, 分为系统表空间和临时共享表空间
        */
 
    /* Check that the value of system variable innodb_page_size was
    set correctly.  Its value was put into srv_page_size. If valid,
    return the associated srv_page_size_shift. */
        // 检查系统变量 innodb_page_size 的值。
    srv_page_size_shift = innodb_page_size_validate(srv_page_size);
    if (!srv_page_size_shift) {
        sql_print_error("InnoDB: Invalid page size=%lu.\n",
                srv_page_size);
        DBUG_RETURN(innobase_init_abort());
    }
 
    /* Set default InnoDB temp data file size to 12 MB and let it be
    auto-extending.
        设置默认的 Innodb 数据文件大小为12MB，并设置其自动增长。
        */
    if (!innobase_data_file_path) {
        innobase_data_file_path = (char*) "ibdata1:12M:autoextend";
    }
 
    /* This is the first time univ_page_size is used.
    It was initialized to 16k pages before srv_page_size was set
        univ_page_size 被初始化为 16k.
        */
    univ_page_size.copy_from(
        page_size_t(srv_page_size, srv_page_size, false));
        // 设置系统表空间的 space_id
    srv_sys_space.set_space_id(TRX_SYS_SPACE);
 
    /* Create the filespace flags.
           设置系统表空间 filespace_flags\name\path
        */
    ulint   fsp_flags = fsp_flags_init(
        univ_page_size, false, false, false, false);
    srv_sys_space.set_flags(fsp_flags);
         
    srv_sys_space.set_name(reserved_system_space_name);
    srv_sys_space.set_path(srv_data_home);
 
    /* Supports raw devices
           支持 raw devices
        */
    if (!srv_sys_space.parse_params(innobase_data_file_path, true)) {
        ib::error() << "Unable to parse innodb_data_file_path="
                << innobase_data_file_path;
        DBUG_RETURN(innobase_init_abort());
    }
 
    /* Set default InnoDB temp data file size to 12 MB and let it be
    auto-extending.
           设置默认的 Innodb temp 数据文件大小为 12MB 并自动增长。
        */
    if (!innobase_temp_data_file_path) {
        innobase_temp_data_file_path = (char*) "ibtmp1:12M:autoextend";
    }
 
    /* We set the temporary tablspace id later, after recovery.
    The temp tablespace doesn't support raw devices.
    Set the name and path.
        在这里设置临时表空间 name 和 path，临时表空间不支持原始设备。
        在 recovery 之后设置临时表空间id。
        */
    srv_tmp_space.set_name(reserved_temporary_space_name);
    srv_tmp_space.set_path(srv_data_home);
 
    /* Create the filespace flags with the temp flag set.
           设置临时表空间的 filespace_flags.
        */
    fsp_flags = fsp_flags_init(
        univ_page_size, false, false, false, true);
    srv_tmp_space.set_flags(fsp_flags);
         
    if (!srv_tmp_space.parse_params(innobase_temp_data_file_path, false)) {
        ib::error() << "Unable to parse innodb_temp_data_file_path="
                << innobase_temp_data_file_path;
        DBUG_RETURN(innobase_init_abort());
    }
 
    /* Perform all sanity check before we take action of deleting files*/
        // 检查系统表空间和临时表空间是否有公共 data file.
    if (srv_sys_space.intersection(&srv_tmp_space)) {
        sql_print_error("%s and %s file names seem to be the same.",
            srv_tmp_space.name(), srv_sys_space.name());
        DBUG_RETURN(innobase_init_abort());
    }
 
    /* ------------ UNDO tablespaces files ---------------------
           undo 表空间。
        */
        // undo表空间dir
    if (!srv_undo_dir) {
        srv_undo_dir = default_path;
    }
        // 规范 undo 表空间目录
    os_normalize_path(srv_undo_dir);
         
    if (strchr(srv_undo_dir, ';')) {
        sql_print_error("syntax error in innodb_undo_directory");
        DBUG_RETURN(innobase_init_abort());
    }
 
    /* -------------- All log files ---------------------------
           所有的日志文件
        */
 
    /* The default dir for log files is the datadir of MySQL
           默认redo log 目录
        */
        // 默认 redo log group dir
    if (!srv_log_group_home_dir) {
        srv_log_group_home_dir = default_path;
    }
        // 规范目录
    os_normalize_path(srv_log_group_home_dir);
 
    if (strchr(srv_log_group_home_dir, ';')) {
        sql_print_error("syntax error in innodb_log_group_home_dir");
        DBUG_RETURN(innobase_init_abort());
    }
         
    if (!innobase_large_prefix) {
        ib::warn() << deprecated_large_prefix;
    }
 
    if (!THDVAR(NULL, support_xa)) {
        ib::warn() << deprecated_innodb_support_xa_off;
        THDVAR(NULL, support_xa) = TRUE;
    }
 
    if (innobase_file_format_name != innodb_file_format_default) {
        ib::warn() << deprecated_file_format;
    }
 
    /* Validate the file format by animal name
           校验 innodb_file_format_max； innodb文件格式
        */
    if (innobase_file_format_name != NULL) {
 
        format_id = innobase_file_format_name_lookup(
            innobase_file_format_name);
 
        if (format_id > UNIV_FORMAT_MAX) {
 
            sql_print_error("InnoDB: wrong innodb_file_format.");
 
        DBUG_RETURN(innobase_init_abort());
        }
    } else {
        /* Set it to the default file format id. Though this
        should never happen. */
        format_id = 0;
    }
 
    srv_file_format = format_id;
 
    /* Given the type of innobase_file_format_name we have little
    choice but to cast away the constness from the returned name.
    innobase_file_format_name is used in the MySQL set variable
    interface and so can't be const. */
 
    innobase_file_format_name =
        (char*) trx_sys_file_format_id_to_name(format_id);
 
    /* Check innobase_file_format_check variable
           检查 innodb_file_format_check 变量；
        */
    if (!innobase_file_format_check) {
        ib::warn() << deprecated_file_format_check;
 
        /* Set the value to disable checking. */
        srv_max_file_format_at_startup = UNIV_FORMAT_MAX + 1;
 
    } else {
 
        /* Set the value to the lowest supported format. */
        srv_max_file_format_at_startup = UNIV_FORMAT_MIN;
    }
 
    if (innobase_file_format_max != innodb_file_format_max_default) {
        ib::warn() << deprecated_file_format_max;
    }
 
    /* Did the user specify a format name that we support?
    As a side effect it will update the variable
    srv_max_file_format_at_startup */
    if (innobase_file_format_validate_and_set(
            innobase_file_format_max) < 0) {
 
        sql_print_error("InnoDB: invalid"
                " innodb_file_format_max value:"
                " should be any value up to %s or its"
                " equivalent numeric id",
                trx_sys_file_format_id_to_name(
                    UNIV_FORMAT_MAX));
 
        DBUG_RETURN(innobase_init_abort());
    }
        /**
           Innodb change buffer
        */
    if (innobase_change_buffering) {
        ulint   use;
 
        for (use = 0;
             use < UT_ARR_SIZE(innobase_change_buffering_values);
             use++) {
            if (!innobase_strcasecmp(
                    innobase_change_buffering,
                    innobase_change_buffering_values[use])) {
                ibuf_use = (ibuf_use_t) use;
                goto innobase_change_buffering_inited_ok;
            }
        }
 
        sql_print_error("InnoDB: invalid value"
                " innodb_change_buffering=%s",
                innobase_change_buffering);
        DBUG_RETURN(innobase_init_abort());
    }
 
innobase_change_buffering_inited_ok:
        // Innodb_change_buffering = ALL
    ut_a((ulint) ibuf_use < UT_ARR_SIZE(innobase_change_buffering_values));
    innobase_change_buffering = (char*)
        innobase_change_buffering_values[ibuf_use];
 
    /* Check that interdependent parameters have sane values.
           对相互依赖的参数进行检查。
           srv_max_buf_pool_modified_pct & srv_max_dirty_pages_pct_lwm
           srv_max_io_capacity & srv_io_capacity & SRV_MAX_IO_CAPACITY_DUMMY_DEFAULT
        */
    if (srv_max_buf_pool_modified_pct < srv_max_dirty_pages_pct_lwm) {
        sql_print_warning("InnoDB: innodb_max_dirty_pages_pct_lwm"
                  " cannot be set higher than"
                  " innodb_max_dirty_pages_pct.\n"
                  "InnoDB: Setting"
                  " innodb_max_dirty_pages_pct_lwm to %lf\n",
                  srv_max_buf_pool_modified_pct);
 
        srv_max_dirty_pages_pct_lwm = srv_max_buf_pool_modified_pct;
    }
 
    if (srv_max_io_capacity == SRV_MAX_IO_CAPACITY_DUMMY_DEFAULT) {
 
        if (srv_io_capacity >= SRV_MAX_IO_CAPACITY_LIMIT / 2) {
            /* Avoid overflow. */
            srv_max_io_capacity = SRV_MAX_IO_CAPACITY_LIMIT;
        } else {
            /* The user has not set the value. We should
            set it based on innodb_io_capacity. */
            srv_max_io_capacity =
                ut_max(2 * srv_io_capacity, 2000UL);
        }
 
    } else if (srv_max_io_capacity < srv_io_capacity) {
        sql_print_warning("InnoDB: innodb_io_capacity"
                  " cannot be set higher than"
                  " innodb_io_capacity_max.\n"
                  "InnoDB: Setting"
                  " innodb_io_capacity to %lu\n",
                  srv_max_io_capacity);
 
        srv_io_capacity = srv_max_io_capacity;
    }
        // 检查 innodb_buffer_pool_filename 配置
    if (!is_filename_allowed(srv_buf_dump_filename,
                 strlen(srv_buf_dump_filename), FALSE)) {
        sql_print_error("InnoDB: innodb_buffer_pool_filename"
            " cannot have colon (:) in the file name.");
        DBUG_RETURN(innobase_init_abort());
    }
 
    /* --------------------------------------------------
           innodb_file_flush_method & innobase_log_file_size & innodb_log_write_ahead_size
           innodb_log_buffer_size & innodb_buffer_pool_size & innodb_read_io_threads & innodb_write_io_threads
           innodb_doublewrite & innodb_log_checksums & innodb_rollback_on_timeout & innobase_locks_unsafe_for_binlog
           innodb_open_files & innodb_monitor 配置 & innodb_old_blocks_pct & innodb_undo_logs &
            
        */
 
    srv_file_flush_method_str = innobase_file_flush_method;
 
    srv_log_file_size = (ib_uint64_t) innobase_log_file_size;
 
    if (UNIV_PAGE_SIZE_DEF != srv_page_size) {
        ib::warn() << "innodb-page-size has been changed from the"
            " default value " << UNIV_PAGE_SIZE_DEF << " to "
            << srv_page_size << ".";
    }
 
    if (srv_log_write_ahead_size > srv_page_size) {
        srv_log_write_ahead_size = srv_page_size;
    } else {
        ulong   srv_log_write_ahead_size_tmp = OS_FILE_LOG_BLOCK_SIZE;
 
        while (srv_log_write_ahead_size_tmp
               < srv_log_write_ahead_size) {
            srv_log_write_ahead_size_tmp
                = srv_log_write_ahead_size_tmp * 2;
        }
        if (srv_log_write_ahead_size_tmp
            != srv_log_write_ahead_size) {
            srv_log_write_ahead_size
                = srv_log_write_ahead_size_tmp / 2;
        }
    }
 
    srv_log_buffer_size = (ulint) innobase_log_buffer_size;
 
    srv_buf_pool_size = (ulint) innobase_buffer_pool_size;
 
    srv_n_read_io_threads = (ulint) innobase_read_io_threads;
    srv_n_write_io_threads = (ulint) innobase_write_io_threads;
 
    srv_use_doublewrite_buf = (ibool) innobase_use_doublewrite;
 
    if (!innobase_use_checksums) {
        ib::warn() << "Setting innodb_checksums to OFF is DEPRECATED."
            " This option may be removed in future releases. You"
            " should set innodb_checksum_algorithm=NONE instead.";
        srv_checksum_algorithm = SRV_CHECKSUM_ALGORITHM_NONE;
    }
 
    innodb_log_checksums_func_update(innodb_log_checksums);
 
#ifdef HAVE_LINUX_LARGE_PAGES
    if ((os_use_large_pages = my_use_large_pages)) {
        os_large_page_size = opt_large_page_size;
    }
#endif
 
    row_rollback_on_timeout = (ibool) innobase_rollback_on_timeout;
 
    srv_locks_unsafe_for_binlog = (ibool) innobase_locks_unsafe_for_binlog;
    if (innobase_locks_unsafe_for_binlog) {
        ib::warn() << "Using innodb_locks_unsafe_for_binlog is"
            " DEPRECATED. This option may be removed in future"
            " releases. Please use READ COMMITTED transaction"
            " isolation level instead; " << SET_TRANSACTION_MSG;
    }
 
    if (innobase_open_files < 10) {
        innobase_open_files = 300;
        if (srv_file_per_table && table_cache_size > 300) {
            innobase_open_files = table_cache_size;
        }
    }
 
    if (innobase_open_files > (long) open_files_limit) {
        ib::warn() << "innodb_open_files should not be greater"
                       " than the open_files_limit.\n";
        if (innobase_open_files > (long) table_cache_size) {
            innobase_open_files = table_cache_size;
        }
    }
 
    srv_max_n_open_files = (ulint) innobase_open_files;
    srv_innodb_status = (ibool) innobase_create_status_file;
 
    srv_print_verbose_log = mysqld_embedded ? 0 : 1;
 
    /* Round up fts_sort_pll_degree to nearest power of 2 number */
    for (num_pll_degree = 1;
         num_pll_degree < fts_sort_pll_degree;
         num_pll_degree <<= 1) {
 
        /* No op */
    }
 
    fts_sort_pll_degree = num_pll_degree;
 
    /* Store the default charset-collation number of this MySQL
    installation
        MySQL默认的 charset-collation.
        */
    data_mysql_default_charset_coll = (ulint) default_charset_info->number;
        // 初始化 innodb_commit_concurrency[限制并发提交] 的默认值
    innobase_commit_concurrency_init_default();
 
        // 初始化 os_event 对象。
    os_event_global_init();
 
    /* Set buffer pool size to default for fast startup when mysqld is
    run with --help --verbose options. */
    ulint   srv_buf_pool_size_org = 0;
    if (opt_help && opt_verbose
        && srv_buf_pool_size > srv_buf_pool_def_size) {
        ib::warn() << "Setting innodb_buf_pool_size to "
            << srv_buf_pool_def_size << " for fast startup, "
            << "when running with --help --verbose options.";
        srv_buf_pool_size_org = srv_buf_pool_size;
        srv_buf_pool_size = srv_buf_pool_def_size;
    }
 
    /* Since we in this module access directly the fields of a trx
    struct, and due to different headers and flags it might happen that
    ib_mutex_t has a different size in this module and in InnoDB
    modules, we check at run time that the size is the same in
    these compilation modules. */
        // 启动或直接创建 innobase
    err = innobase_start_or_create_for_mysql();
        // innobase_buffer_pool_size
    if (srv_buf_pool_size_org != 0) {
        /* Set the original value back to show in help. */
        srv_buf_pool_size_org =
            buf_pool_size_align(srv_buf_pool_size_org);
        innobase_buffer_pool_size =
            static_cast<long long>(srv_buf_pool_size_org);
    } else {
        innobase_buffer_pool_size =
            static_cast<long long>(srv_buf_pool_size);
    }
 
    if (err != DB_SUCCESS) {
        DBUG_RETURN(innobase_init_abort());
    }
 
    /* Create mutex to protect encryption master_key_id. */
    mutex_create(LATCH_ID_MASTER_KEY_ID_MUTEX, &master_key_id_mutex);
 
    /* Adjust the innodb_undo_logs config object
           调整 innodb_undo_logs
        */
    innobase_undo_logs_init_default_max();
 
    innobase_old_blocks_pct = static_cast<uint>(
        buf_LRU_old_ratio_update(innobase_old_blocks_pct, TRUE));
 
    ibuf_max_size_update(srv_change_buffer_max_size);
 
    innobase_open_tables = hash_create(200);
    mysql_mutex_init(innobase_share_mutex_key.m_value,
             &innobase_share_mutex,
             MY_MUTEX_INIT_FAST);
    mysql_mutex_init(commit_cond_mutex_key.m_value,
             &commit_cond_m, MY_MUTEX_INIT_FAST);
    mysql_cond_init(commit_cond_key.m_value, &commit_cond);
 
    innodb_inited= 1;
#ifdef MYSQL_DYNAMIC_PLUGIN
    if (innobase_hton != p) {
        innobase_hton = reinterpret_cast<handlerton*>(p);
        *innobase_hton = *innodb_hton_ptr;
    }
#endif /* MYSQL_DYNAMIC_PLUGIN */
 
    /* Get the current high water mark format. */
    innobase_file_format_max = (char*) trx_sys_file_format_max_get();
 
    /* Currently, monitor counter information are not persistent.
           Innodb monitor
        */
    memset(monitor_set_tbl, 0, sizeof monitor_set_tbl);
 
    memset(innodb_counter_value, 0, sizeof innodb_counter_value);
 
    /* Do this as late as possible so server is fully starts up,
    since  we might get some initial stats if user choose to turn
    on some counters from start up */
    if (innobase_enable_monitor_counter) {
        innodb_enable_monitor_at_startup(
            innobase_enable_monitor_counter);
    }
 
    /* Turn on monitor counters that are default on */
    srv_mon_default_on();
 
 
    /* Unit Tests */
#ifdef UNIV_ENABLE_UNIT_TEST_GET_PARENT_DIR
    unit_test_os_file_get_parent_dir();
#endif /* UNIV_ENABLE_UNIT_TEST_GET_PARENT_DIR */
 
#ifdef UNIV_ENABLE_UNIT_TEST_MAKE_FILEPATH
    test_make_filepath();
#endif /*UNIV_ENABLE_UNIT_TEST_MAKE_FILEPATH */
 
#ifdef UNIV_ENABLE_DICT_STATS_TEST
    test_dict_stats_all();
#endif /*UNIV_ENABLE_DICT_STATS_TEST */
 
#ifdef UNIV_ENABLE_UNIT_TEST_ROW_RAW_FORMAT_INT
# ifdef HAVE_UT_CHRONO_T
    test_row_raw_format_int();
# endif /* HAVE_UT_CHRONO_T */
#endif /* UNIV_ENABLE_UNIT_TEST_ROW_RAW_FORMAT_INT */
 
#ifndef UNIV_HOTBACKUP
#ifdef _WIN32
    if (ut_win_init_time()) {
        DBUG_RETURN(innobase_init_abort());
    }
#endif /* _WIN32 */
#endif /* !UNIV_HOTBACKUP */
 
    DBUG_RETURN(0);
}
```
在handlerton的定义中有这行代码“innobase_hton->ddse_dict_init = innobase_ddse_dict_init;”，后面的函数指针就是调用InnoDB引擎创建的主要函数，如果查找他的调用堆栈则可以看到，其一直向上延伸到数据库的启动，这里不再赘述，只向下分析：
```
/** Initialize InnoDB for being used to store the DD tables.
Create the required files according to the dict_init_mode.
Create strings representing the required DDSE tables, i.e.,
tables that InnoDB expects to exist in the DD,
and add them to the appropriate out parameter.

@param[in]	dict_init_mode	How to initialize files

@param[in]	version		Target DD version if a new server
                                is being installed.
                                0 if restarting an existing server.

@param[out]	tables		List of SQL DDL statements
                                for creating DD tables that
                                are needed by the DDSE.

@param[out]	tablespaces	List of meta data for predefined
                                tablespaces created by the DDSE.

@retval	true			An error occurred.
@retval	false			Success - no errors. */
static bool innobase_ddse_dict_init(
    dict_init_mode_t dict_init_mode, uint version,
    List<const dd::Object_table> *tables,
    List<const Plugin_tablespace> *tablespaces) {
  DBUG_TRACE;

  LogErr(SYSTEM_LEVEL, ER_IB_MSG_INNODB_START_INITIALIZE);

  assert(tables && tables->is_empty());
  assert(tablespaces && tablespaces->is_empty());

  if (dblwr::enabled) {
    if (innobase_doublewrite_dir != nullptr && *innobase_doublewrite_dir != 0) {
      dblwr::dir.assign(innobase_doublewrite_dir);
      switch (dblwr::dir.front()) {
        case '#':
        case '.':
          break;
        default:
          if (!Fil_path::is_absolute_path(dblwr::dir)) {
            dblwr::dir.insert(0, "#");
          }
      }
      ib::info(ER_IB_MSG_DBLWR_1325)
          << "Using " << dblwr::dir << " as doublewrite directory";
    } else {
      dblwr::dir.assign(".");
    }
    ib::info(ER_IB_MSG_DBLWR_1304) << "Atomic write enabled";
  } else {
    ib::info(ER_IB_MSG_DBLWR_1305) << "Atomic write disabled";
  }

  if (innobase_init_files(dict_init_mode, tablespaces)) {
    return true;
  }

  /* Instantiate table defs only if we are successful so far. */
  dd::Object_table *innodb_dynamic_metadata =
      dd::Object_table::create_object_table();
  innodb_dynamic_metadata->set_hidden(true);
  dd::Object_table_definition *def =
      innodb_dynamic_metadata->target_table_definition();
  def->set_table_name("innodb_dynamic_metadata");
  def->add_field(0, "table_id", "table_id BIGINT UNSIGNED NOT NULL");
  def->add_field(1, "version", "version BIGINT UNSIGNED NOT NULL");
  def->add_field(2, "metadata", "metadata BLOB NOT NULL");
  def->add_index(0, "index_pk", "PRIMARY KEY (table_id)");
  /* Options and tablespace are set at the SQL layer. */

  /* Changing these values would change the specification of innodb statistics
  tables. */
  static constexpr size_t DB_NAME_FIELD_SIZE = 64;
  static constexpr size_t TABLE_NAME_FIELD_SIZE = 199;

  static_assert(DB_NAME_FIELD_SIZE == dict_name::MAX_DB_CHAR_LEN,
                "dict_name::MAX_DB_CHAR_LEN mismatch with db column");

  static_assert(TABLE_NAME_FIELD_SIZE == dict_name::MAX_TABLE_CHAR_LEN,
                "dict_name::MAX_TABLE_CHAR_LEN mismatch with table column");

  /* Set length for database name field. */
  std::ostringstream db_name_field;
  db_name_field << "database_name VARCHAR(" << DB_NAME_FIELD_SIZE
                << ") NOT NULL";
  std::string db_field = db_name_field.str();

  /* Set length for table name field. */
  std::ostringstream table_name_field;
  table_name_field << "table_name VARCHAR(" << TABLE_NAME_FIELD_SIZE
                   << ") NOT NULL";
  std::string table_field = table_name_field.str();

  dd::Object_table *innodb_table_stats =
      dd::Object_table::create_object_table();
  innodb_table_stats->set_hidden(false);
  def = innodb_table_stats->target_table_definition();
  def->set_table_name("innodb_table_stats");
  def->add_field(0, "database_name", db_field.c_str());
  def->add_field(1, "table_name", table_field.c_str());
  def->add_field(2, "last_update",
                 "last_update TIMESTAMP NOT NULL \n"
                 "  DEFAULT CURRENT_TIMESTAMP \n"
                 "  ON UPDATE CURRENT_TIMESTAMP");
  def->add_field(3, "n_rows", "n_rows BIGINT UNSIGNED NOT NULL");
  def->add_field(4, "clustered_index_size",
                 "clustered_index_size BIGINT UNSIGNED NOT NULL");
  def->add_field(5, "sum_of_other_index_sizes",
                 "sum_of_other_index_sizes BIGINT UNSIGNED NOT NULL");
  def->add_index(0, "index_pk", "PRIMARY KEY (database_name, table_name)");
  /* Options and tablespace are set at the SQL layer. */

  dd::Object_table *innodb_index_stats =
      dd::Object_table::create_object_table();
  innodb_index_stats->set_hidden(false);
  def = innodb_index_stats->target_table_definition();
  def->set_table_name("innodb_index_stats");
  def->add_field(0, "database_name", db_field.c_str());
  def->add_field(1, "table_name", table_field.c_str());
  def->add_field(2, "index_name", "index_name VARCHAR(64) NOT NULL");
  def->add_field(3, "last_update",
                 "last_update TIMESTAMP NOT NULL"
                 "  DEFAULT CURRENT_TIMESTAMP"
                 "  ON UPDATE CURRENT_TIMESTAMP");
  /*
          There are at least: stat_name='size'
                  stat_name='n_leaf_pages'
                  stat_name='n_diff_pfx%'
  */
  def->add_field(4, "stat_name", "stat_name VARCHAR(64) NOT NULL");
  def->add_field(5, "stat_value", "stat_value BIGINT UNSIGNED NOT NULL");
  def->add_field(6, "sample_size", "sample_size BIGINT UNSIGNED");
  def->add_field(7, "stat_description",
                 "stat_description VARCHAR(1024) NOT NULL");
  def->add_index(0, "index_pk",
                 "PRIMARY KEY (database_name, table_name, "
                 "index_name, stat_name)");
  /* Options and tablespace are set at the SQL layer. */

  dd::Object_table *innodb_ddl_log = dd::Object_table::create_object_table();
  innodb_ddl_log->set_hidden(true);
  def = innodb_ddl_log->target_table_definition();
  def->set_table_name("innodb_ddl_log");
  def->add_field(0, "id", "id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT");
  def->add_field(1, "thread_id", "thread_id BIGINT UNSIGNED NOT NULL");
  def->add_field(2, "type", "type INT UNSIGNED NOT NULL");
  def->add_field(3, "space_id", "space_id INT UNSIGNED");
  def->add_field(4, "page_no", "page_no INT UNSIGNED");
  def->add_field(5, "index_id", "index_id BIGINT UNSIGNED");
  def->add_field(6, "table_id", "table_id BIGINT UNSIGNED");
  def->add_field(7, "old_file_path",
                 "old_file_path VARCHAR(512) COLLATE UTF8_BIN");
  def->add_field(8, "new_file_path",
                 "new_file_path VARCHAR(512) COLLATE UTF8_BIN");
  def->add_index(0, "index_pk", "PRIMARY KEY(id)");
  def->add_index(1, "index_k_thread_id", "KEY(thread_id)");
  /* Options and tablespace are set at the SQL layer. */

  tables->push_back(innodb_dynamic_metadata);
  tables->push_back(innodb_table_stats);
  tables->push_back(innodb_index_stats);
  tables->push_back(innodb_ddl_log);

  LogErr(SYSTEM_LEVEL, ER_IB_MSG_INNODB_END_INITIALIZE);

  return false;
}
/** Open or create InnoDB data files.
@param[in]	dict_init_mode	whether to create or open the files
@param[in,out]	tablespaces	predefined tablespaces created by the DDSE
@return 0 on success, 1 on failure * /
static int innobase_init_files(dict_init_mode_t dict_init_mode,
                               List<const Plugin_tablespace> * tablespaces) {
  DBUG_TRACE;

  ut_ad(dict_init_mode == DICT_INIT_CREATE_FILES ||
        dict_init_mode == DICT_INIT_CHECK_FILES ||
        dict_init_mode == DICT_INIT_UPGRADE_57_FILES);

  bool create = (dict_init_mode == DICT_INIT_CREATE_FILES);

  /* Check if the data files exist or not. * /
  dberr_t err =
      srv_sys_space.check_file_spec(create, MIN_EXPECTED_TABLESPACE_SIZE);

  if (err != DB_SUCCESS) {
    return innodb_init_abort();
  }

  srv_is_upgrade_mode = (dict_init_mode == DICT_INIT_UPGRADE_57_FILES);

  /* Start the InnoDB server. */
  err = srv_start(create);

  if (err != DB_SUCCESS) {
    return innodb_init_abort();
  }

  if (srv_is_upgrade_mode) {
    if (!dict_sys_table_id_build()) {
      return innodb_init_abort();
    }

    if (trx_sys->found_prepared_trx) {
      ib::error(ER_DD_UPGRADE_FOUND_PREPARED_XA_TRANSACTION);
      return innodb_init_abort();
    }

    /* Disable AHI when we start loading tables for purge.
    These tables are evicted anyway after purge. */

    bool old_btr_search_value = btr_search_enabled;
    btr_search_enabled = false;

    /* Load all tablespaces upfront from InnoDB Dictionary.
    This is needed for applying purge and ibuf from 5.7 */
    dict_load_tablespaces_for_upgrade();

    /* Start purge threads immediately and wait for purge to
    become empty. All table_ids will be adjusted by a fixed
    offset during upgrade. So purge cannot load a table by
    table_id later. Also InnoDB dictionary will be dropped
    during the process of upgrade. So apply all the purge
    now. */
    srv_start_purge_threads();

    uint64_t rseg_history_len;
    while ((rseg_history_len = trx_sys->rseg_history_len.load()) != 0) {
      ib::info(ER_IB_MSG_547)
          << "Waiting for purge to become empty:"
          << " current purge history len is " << rseg_history_len;
      sleep(1);
    }

    srv_upgrade_old_undo_found = false;

    buf_flush_sync_all_buf_pools();

    dict_upgrade_evict_tables_cache();

    dict_stats_evict_tablespaces();

    btr_search_enabled = old_btr_search_value;
  }

  bool ret;

  // For upgrade from 5.7, create mysql.ibd
  create |= (dict_init_mode == DICT_INIT_UPGRADE_57_FILES);
  ret = create ? dd_create_hardcoded(dict_sys_t::s_space_id,
                                     dict_sys_t::s_dd_space_file_name)
               : dd_open_hardcoded(dict_sys_t::s_space_id,
                                   dict_sys_t::s_dd_space_file_name);

  /* Once hardcoded tablespace mysql is created or opened,
  prepare it along with innodb system tablespace for server.
  Tell server that these two hardcoded tablespaces exist.  */
  if (!ret) {
    const size_t len =
        30 + sizeof("id=;flags=;server_version=;space_version=;state=normal");
    const char *fmt =
        "id=%u;flags=%u;server_version=%u;space_version=%u;state=normal";
    static char se_private_data_innodb_system[len];
    static char se_private_data_dd[len];
    snprintf(se_private_data_innodb_system, len, fmt, TRX_SYS_SPACE,
             predefined_flags, DD_SPACE_CURRENT_SRV_VERSION,
             DD_SPACE_CURRENT_SPACE_VERSION);
    snprintf(se_private_data_dd, len, fmt, dict_sys_t::s_space_id,
             predefined_flags, DD_SPACE_CURRENT_SRV_VERSION,
             DD_SPACE_CURRENT_SPACE_VERSION);

    static Plugin_tablespace dd_space(dict_sys_t::s_dd_space_name, "",
                                      se_private_data_dd, "",
                                      innobase_hton_name);
    static Plugin_tablespace::Plugin_tablespace_file dd_file(
        dict_sys_t::s_dd_space_file_name, "");
    dd_space.add_file(&dd_file);
    tablespaces->push_back(&dd_space);

    static Plugin_tablespace innodb(dict_sys_t::s_sys_space_name, "",
                                    se_private_data_innodb_system, "",
                                    innobase_hton_name);
    Tablespace::files_t::const_iterator end = srv_sys_space.m_files.end();
    Tablespace::files_t::const_iterator begin = srv_sys_space.m_files.begin();
    for (Tablespace::files_t::const_iterator it = begin; it != end; ++it) {
      innobase_sys_files.push_back(UT_NEW_NOKEY(
          Plugin_tablespace::Plugin_tablespace_file(it->name(), "")));
      innodb.add_file(innobase_sys_files.back());
    }
    tablespaces->push_back(&innodb);

  } else {
    return innodb_init_abort();
  }

  /* Create mutex to protect encryption master_key_id. */
  mutex_create(LATCH_ID_MASTER_KEY_ID_MUTEX, &master_key_id_mutex);

  innobase_old_blocks_pct = static_cast<uint>(
      buf_LRU_old_ratio_update(innobase_old_blocks_pct, TRUE));

  ibuf_max_size_update(srv_change_buffer_max_size);

  innobase_open_tables = hash_create(200);
  mysql_mutex_init(innobase_share_mutex_key.m_value, &innobase_share_mutex,
                   MY_MUTEX_INIT_FAST);
  mysql_mutex_init(commit_cond_mutex_key.m_value, &commit_cond_m,
                   MY_MUTEX_INIT_FAST);
  mysql_cond_init(commit_cond_key.m_value, &commit_cond);
  mysql_mutex_init(resume_encryption_cond_mutex_key.m_value,
                   &resume_encryption_cond_m, MY_MUTEX_INIT_FAST);
  mysql_cond_init(resume_encryption_cond_key.m_value, &resume_encryption_cond);
  innodb_inited = true;
#ifdef MYSQL_DYNAMIC_PLUGIN
  if (innobase_hton != p) {
    innobase_hton = reinterpret_cast<handlerton *>(p);
    *innobase_hton = *innodb_hton_ptr;
  }
#endif /* MYSQL_DYNAMIC_PLUGIN */

  /* Do this as late as possible so server is fully starts up,
  since  we might get some initial stats if user choose to turn
  on some counters from start up */
  if (innobase_enable_monitor_counter) {
    innodb_enable_monitor_at_startup(innobase_enable_monitor_counter);
  }

  /* Turn on monitor counters that are default on */
  srv_mon_default_on();

  /* Unit Tests */
#ifdef UNIV_ENABLE_UNIT_TEST_GET_PARENT_DIR
  unit_test_os_file_get_parent_dir();
#endif /* UNIV_ENABLE_UNIT_TEST_GET_PARENT_DIR */

#ifdef UNIV_ENABLE_UNIT_TEST_MAKE_FILEPATH
  test_make_filepath();
#endif /*UNIV_ENABLE_UNIT_TEST_MAKE_FILEPATH */

#ifdef UNIV_ENABLE_DICT_STATS_TEST
  test_dict_stats_all();
#endif /*UNIV_ENABLE_DICT_STATS_TEST */

#ifdef UNIV_ENABLE_UNIT_TEST_ROW_RAW_FORMAT_INT
#ifdef HAVE_UT_CHRONO_T
  test_row_raw_format_int();
#endif /* HAVE_UT_CHRONO_T */
#endif /* UNIV_ENABLE_UNIT_TEST_ROW_RAW_FORMAT_INT * /

  return 0;
}

```
在这个函数里调用srv_start这个函数，它在storage\innobase\src\srv0start.cc这个文件中：
```
dberr_t srv_start(bool create_new_db) {
  lsn_t flushed_lsn;

  /* just for assertions */
  lsn_t previous_lsn;

  /* output from call to create_log_files(...) */
  lsn_t new_checkpoint_lsn = 0;

  page_no_t sum_of_data_file_sizes;
  page_no_t tablespace_size_in_header;
  dberr_t err;
  uint32_t srv_n_log_files_found = srv_n_log_files;
  mtr_t mtr;
  purge_pq_t *purge_queue;
  char logfilename[10000];
  char *logfile0 = nullptr;
  size_t dirnamelen;
  unsigned i = 0;

  assert(srv_dict_metadata == nullptr);
  /* Reset the start state. */
  srv_start_state = SRV_START_STATE_NONE;

#ifdef UNIV_LINUX
#ifdef HAVE_FALLOC_PUNCH_HOLE_AND_KEEP_SIZE
  ib::info(ER_IB_MSG_1107);
#else
  ib::info(ER_IB_MSG_1108);
#endif /* HAVE_FALLOC_PUNCH_HOLE_AND_KEEP_SIZE */
#endif /* UNIV_LINUX */

  if (sizeof(ulint) != sizeof(void *)) {
    ib::error(ER_IB_MSG_1109, sizeof(ulint), sizeof(void *));
  }

  //对升级模式中的只读和强制恢复进行处理
  if (srv_is_upgrade_mode) {
    if (srv_read_only_mode) {
      ib::error(ER_IB_MSG_1110);
      return (srv_init_abort(DB_ERROR));
    }
    if (srv_force_recovery != 0) {
      ib::error(ER_IB_MSG_1111);
      return (srv_init_abort(DB_ERROR));
    }
  }

#ifdef UNIV_DEBUG
  ib::info(ER_IB_MSG_1112) << "!!!!!!!! UNIV_DEBUG switched on !!!!!!!!!";
#endif

#ifdef UNIV_IBUF_DEBUG
  ib::info(ER_IB_MSG_1113) << "!!!!!!!! UNIV_IBUF_DEBUG switched on !!!!!!!!!";
#ifdef UNIV_IBUF_COUNT_DEBUG
  ib::info(ER_IB_MSG_1114)
      << "!!!!!!!! UNIV_IBUF_COUNT_DEBUG switched on !!!!!!!!!";
  ib::error(ER_IB_MSG_1115)
      << "Crash recovery will fail with UNIV_IBUF_COUNT_DEBUG";
#endif
#endif

#ifdef UNIV_LOG_LSN_DEBUG
  ib::info(ER_IB_MSG_1116)
      << "!!!!!!!! UNIV_LOG_LSN_DEBUG switched on !!!!!!!!!";
#endif /* UNIV_LOG_LSN_DEBUG */

#if defined(COMPILER_HINTS_ENABLED)
  ib::info(ER_IB_MSG_1117) << "Compiler hints enabled.";
#endif /* defined(COMPILER_HINTS_ENABLED) */

  ib::info(ER_IB_MSG_1119) << MUTEX_TYPE;
  ib::info(ER_IB_MSG_1120) << IB_MEMORY_BARRIER_STARTUP_MSG;

  if (srv_force_recovery > 0) {
    ib::info(ER_IB_MSG_1121) << "!!! innodb_force_recovery is set to "
                             << srv_force_recovery << " !!!";
  }

#ifndef HAVE_MEMORY_BARRIER
#if defined __i386__ || defined __x86_64__ || defined _M_IX86 || \
    defined _M_X64 || defined _WIN32
#else
  ib::warn(ER_IB_MSG_1122);
#endif /* IA32 or AMD64 */
#endif /* HAVE_MEMORY_BARRIER */

#ifdef UNIV_ZIP_DEBUG
  ib::info(ER_IB_MSG_1123, ZLIB_VERSION) << " with validation";
#else
  ib::info(ER_IB_MSG_1123, ZLIB_VERSION);
#endif /* UNIV_ZIP_DEBUG */

#ifdef UNIV_ZIP_COPY
  ib::info(ER_IB_MSG_1124) << "and extra copying";
#endif /* UNIV_ZIP_COPY */

  /* Since InnoDB does not currently clean up all its internal data
  structures in MySQL Embedded Server Library server_end(), we
  print an error message if someone tries to start up InnoDB a
  second time during the process lifetime. */

  if (srv_start_has_been_called) {
    ib::error(ER_IB_MSG_1125);
  }

  srv_start_has_been_called = true;

  srv_is_being_started = true;

#ifdef HAVE_PSI_STAGE_INTERFACE
  /* Register performance schema stages before any real work has been
  started which may need to be instrumented. */
  mysql_stage_register("innodb", srv_stages, UT_ARR_SIZE(srv_stages));
#endif /* HAVE_PSI_STAGE_INTERFACE */

  /* Switch latching order checks on in sync0debug.cc, if
  --innodb-sync-debug=false (default) */
  ut_d(sync_check_enable());
  //启动INNODB服务，进行相关参数及插件的初始化
  srv_boot();

  ib::info(ER_IB_MSG_1126) << (ut_crc32_cpu_enabled ? "Using" : "Not using")
                           << " CPU crc32 instructions";

  os_create_block_cache();

  fil_init(srv_max_n_open_files);

  /* This is the default directory for IBD and IBU files. Put it first
  in the list of known directories. */
  fil_set_scan_dir(MySQL_datadir_path.path());

  /* Add --innodb-data-home-dir as a known location for IBD and IBU files
  if it is not already there. */
  ut_ad(srv_data_home != nullptr && *srv_data_home != '\0');
  fil_set_scan_dir(Fil_path::remove_quotes(srv_data_home));

  /* Add --innodb-directories as known locations for IBD and IBU files. */
  if (srv_innodb_directories != nullptr && *srv_innodb_directories != 0) {
    fil_set_scan_dirs(Fil_path::remove_quotes(srv_innodb_directories));
  }

  /* Note whether the undo path is different (not the same or under)
  from all other known directories. If so, this will allow us to keep
  IBD files out of this unique undo location.*/
  MySQL_undo_path_is_unique = !fil_path_is_known(MySQL_undo_path.path());

  /* For the purpose of file discovery at startup, we need to scan
  --innodb-undo-directory also if it is different from the locations above. */
  if (MySQL_undo_path_is_unique) {
    fil_set_scan_dir(Fil_path::remove_quotes(MySQL_undo_path));
  }

  ib::info(ER_IB_MSG_378) << "Directories to scan '" << fil_get_dirs() << "'";

  /* Must replace clone files before scanning directories. When
  clone replaces current database, cloned files are moved to data files
  at this stage. */
  err = clone_init();

  if (err != DB_SUCCESS) {
    return (srv_init_abort(err));
  }

  err = fil_scan_for_tablespaces();

  if (err != DB_SUCCESS) {
    return (srv_init_abort(err));
  }

  //非只读模式的处理即innodb monitor相关处理
  if (!srv_read_only_mode) {
    mutex_create(LATCH_ID_SRV_MONITOR_FILE, &srv_monitor_file_mutex);

    if (srv_innodb_status) {
      srv_monitor_file_name = static_cast<char *>(ut_malloc_nokey(
          MySQL_datadir_path.len() + 20 + sizeof "/innodb_status."));

      sprintf(srv_monitor_file_name, "%s/innodb_status." ULINTPF,
              static_cast<const char *>(MySQL_datadir_path),
              os_proc_get_number());

      srv_monitor_file = fopen(srv_monitor_file_name, "w+");

      if (!srv_monitor_file) {
        ib::error(ER_IB_MSG_1127, srv_monitor_file_name, strerror(errno));

        return (srv_init_abort(DB_ERROR));
      }
    } else {
      srv_monitor_file_name = nullptr;
      srv_monitor_file = os_file_create_tmpfile(nullptr);

      if (!srv_monitor_file) {
        return (srv_init_abort(DB_ERROR));
      }
    }

    mutex_create(LATCH_ID_SRV_MISC_TMPFILE, &srv_misc_tmpfile_mutex);

    srv_misc_tmpfile = os_file_create_tmpfile(nullptr);

    if (!srv_misc_tmpfile) {
      return (srv_init_abort(DB_ERROR));
    }
  }

  srv_n_file_io_threads = srv_n_read_io_threads;

  srv_n_file_io_threads += srv_n_write_io_threads;

  //增加日志 缓冲相关的IO线程
  if (!srv_read_only_mode) {
    /* Add the log and ibuf IO threads. */
    srv_n_file_io_threads += 2;
  } else {
    ib::info(ER_IB_MSG_1128);
  }

  ut_a(srv_n_file_io_threads <= SRV_MAX_N_IO_THREADS);

  //初始化异步IO
  if (!os_aio_init(srv_n_read_io_threads, srv_n_write_io_threads,
                   SRV_MAX_N_PENDING_SYNC_IOS)) {
    ib::error(ER_IB_MSG_1129);

    return (srv_init_abort(DB_ERROR));
  }

  double size;
  char unit;

  if (srv_buf_pool_size >= 1024 * 1024 * 1024) {
    size = ((double)srv_buf_pool_size) / (1024 * 1024 * 1024);
    unit = 'G';
  } else {
    size = ((double)srv_buf_pool_size) / (1024 * 1024);
    unit = 'M';
  }

  double chunk_size;
  char chunk_unit;

  if (srv_buf_pool_chunk_unit >= 1024 * 1024 * 1024) {
    chunk_size = srv_buf_pool_chunk_unit / 1024.0 / 1024 / 1024;
    chunk_unit = 'G';
  } else {
    chunk_size = srv_buf_pool_chunk_unit / 1024.0 / 1024;
    chunk_unit = 'M';
  }

  ib::info(ER_IB_MSG_1130, size, unit, srv_buf_pool_instances, chunk_size,
           chunk_unit);

  //创建引擎的内存缓冲池，内存不足时，报错。
  err = buf_pool_init(srv_buf_pool_size, srv_buf_pool_instances);

  if (err != DB_SUCCESS) {
    ib::error(ER_IB_MSG_1131);

    return (srv_init_abort(DB_ERROR));
  }

  ib::info(ER_IB_MSG_1132);

#ifdef UNIV_DEBUG
  /* We have observed deadlocks with a 5MB buffer pool but
  the actual lower limit could very well be a little higher. */

  if (srv_buf_pool_size <= 5 * 1024 * 1024) {
    ib::info(ER_IB_MSG_1133, ulonglong{srv_buf_pool_size / 1024 / 1024});
  }
#endif /* UNIV_DEBUG */

  //初始化FSP系统和重做日志
  fsp_init();
  pars_init();
  //创建Recover系统
  recv_sys_create();
  recv_sys_init(buf_pool_get_curr_size());
  //事务锁创建
  trx_sys_create();
  //锁创建
  lock_sys_create(srv_lock_table_size);
  //设置线程集合的状态
  srv_start_state_set(SRV_START_STATE_LOCK_SYS);

  /* Create i/o-handler threads: */

  /* For read only mode, we don't need ibuf and log I/O thread.
  Please see innobase_start_or_create_for_mysql() */
  ulint start = (srv_read_only_mode) ? 0 : 2;

  //创建线程并启动
  for (ulint t = 0; t < srv_n_file_io_threads; ++t) {
    IB_thread thread;
    if (t < start) {
      if (t == 0) {
        thread = os_thread_create(io_ibuf_thread_key, io_handler_thread, t);
      } else {
        ut_ad(t == 1);
        thread = os_thread_create(io_log_thread_key, io_handler_thread, t);
      }
    } else if (t >= start && t < (start + srv_n_read_io_threads)) {
      thread = os_thread_create(io_read_thread_key, io_handler_thread, t);

    } else if (t >= (start + srv_n_read_io_threads) &&
               t < (start + srv_n_read_io_threads + srv_n_write_io_threads)) {
      thread = os_thread_create(io_write_thread_key, io_handler_thread, t);
    } else {
      thread = os_thread_create(io_handler_thread_key, io_handler_thread, t);
    }
    thread.start();
  }

  /* Even in read-only mode there could be flush job generated by
  intrinsic table operations.
  //初始化页Cleaner
  buf_flush_page_cleaner_init(srv_n_page_cleaners);

  srv_start_state_set(SRV_START_STATE_IO);

  srv_startup_is_before_trx_rollback_phase = !create_new_db;

  if (create_new_db) {
    recv_sys_free();
  }

  /* Open or create the data files. */
  page_no_t sum_of_new_sizes;

  //打开OR创建数据文件IBDATA，并获取flushed_lsn
  err = srv_sys_space.open_or_create(false, create_new_db, &sum_of_new_sizes,
                                     &flushed_lsn);

  /* FIXME: This can be done earlier, but we now have to wait for
  checking of system tablespace. */
  dict_persist_init();

  switch (err) {
    case DB_SUCCESS:
      break;
    case DB_CANNOT_OPEN_FILE:
      ib::error(ER_IB_MSG_1134);
      /* fall through */
    default:

      /* Other errors might come from
      Datafile::validate_first_page() */

      return (srv_init_abort(err));
  }

  dirnamelen = strlen(srv_log_group_home_dir);
  ut_a(dirnamelen < (sizeof logfilename) - 10 - sizeof "ib_logfile");
  memcpy(logfilename, srv_log_group_home_dir, dirnamelen);

  /* Add a path separator if needed. */
  if (dirnamelen && logfilename[dirnamelen - 1] != OS_PATH_SEPARATOR) {
    logfilename[dirnamelen++] = OS_PATH_SEPARATOR;
  }

  srv_log_file_size_requested = srv_log_file_size;

  if (create_new_db) {
    ut_a(buf_are_flush_lists_empty_validate());

    flushed_lsn = LOG_START_LSN;

    err = create_log_files(logfilename, dirnamelen, flushed_lsn, 0, logfile0,
                           new_checkpoint_lsn);

    if (err != DB_SUCCESS) {
      return (srv_init_abort(err));
    }

    flushed_lsn = new_checkpoint_lsn;

    ut_a(new_checkpoint_lsn == LOG_START_LSN + LOG_BLOCK_HDR_SIZE);

  } else {
    for (i = 0; i < SRV_N_LOG_FILES_CLONE_MAX; i++) {
      os_offset_t size;
      os_file_stat_t stat_info;

      sprintf(logfilename + dirnamelen, "ib_logfile%u", i);
      //获得日志文件状态
      err = os_file_get_status(logfilename, &stat_info, false,
                               srv_read_only_mode);

      if (err == DB_NOT_FOUND) {
        if (i == 0) {
          if (flushed_lsn < static_cast<lsn_t>(1000)) {
            ib::error(ER_IB_MSG_1135);
            return (srv_init_abort(DB_ERROR));
          }

          err = create_log_files(logfilename, dirnamelen, flushed_lsn,
                                 SRV_N_LOG_FILES_CLONE_MAX, logfile0,
                                 new_checkpoint_lsn);

          if (err != DB_SUCCESS) {
            return (srv_init_abort(err));
          }

          create_log_files_rename(logfilename, dirnamelen, new_checkpoint_lsn,
                                  logfile0);

          /* Suppress the message about
          crash recovery. */
          flushed_lsn = new_checkpoint_lsn;
          ut_a(log_sys != nullptr);
          goto files_checked;
        } else if (i < 2) {
          /* must have at least 2 log files */
          ib::error(ER_IB_MSG_1136);
          return (srv_init_abort(err));
        }

        /* opened all files */
        break;
      }

      if (!srv_file_check_mode(logfilename)) {
        return (srv_init_abort(DB_ERROR));
      }

      err = open_log_file(&files[i], logfilename, &size);

      if (err != DB_SUCCESS) {
        return (srv_init_abort(err));
      }

      ut_a(size != (os_offset_t)-1);

      if (size & ((1 << UNIV_PAGE_SIZE_SHIFT) - 1)) {
        ib::error(ER_IB_MSG_1137, logfilename, ulonglong{size});
        return (srv_init_abort(DB_ERROR));
      }

      if (i == 0) {
        srv_log_file_size = size;
#ifndef UNIV_DEBUG_DEDICATED
      } else if (size != srv_log_file_size) {
#else
      } else if (!srv_dedicated_server && size != srv_log_file_size) {
#endif /* UNIV_DEBUG_DEDICATED */
        ib::error(ER_IB_MSG_1138, logfilename, ulonglong{size},
                  srv_log_file_size);

        return (srv_init_abort(DB_ERROR));
      }
    }

    //设置日志文件数量
    srv_n_log_files_found = i;

    /* Create the in-memory file space objects. */

    sprintf(logfilename + dirnamelen, "ib_logfile%u", 0);

    /* Disable the doublewrite buffer for log files. */
    fil_space_t *log_space = fil_space_create(
        "innodb_redo_log", dict_sys_t::s_log_space_first_id,
        fsp_flags_set_page_size(0, univ_page_size), FIL_TYPE_LOG);

    ut_ad(fil_validate());
    ut_a(log_space != nullptr);

    /* srv_log_file_size is measured in bytes */
    ut_a(srv_log_file_size / UNIV_PAGE_SIZE <= PAGE_NO_MAX);

    for (unsigned j = 0; j < i; j++) {
      sprintf(logfilename + dirnamelen, "ib_logfile%u", j);

      const ulonglong file_pages = srv_log_file_size / UNIV_PAGE_SIZE;

      if (fil_node_create(logfilename, static_cast<page_no_t>(file_pages),
                          log_space, false, false) == nullptr) {
        return (srv_init_abort(DB_ERROR));
      }
    }

    if (!log_sys_init(i, srv_log_file_size, dict_sys_t::s_log_space_first_id)) {
      return (srv_init_abort(DB_ERROR));
    }

    /* Read the first log file header to get the encryption
    information if it exist. */
    if (srv_force_recovery < SRV_FORCE_NO_LOG_REDO && !log_read_encryption()) {
      return (srv_init_abort(DB_ERROR));
    }
  }

  ut_a(log_sys != nullptr);

  /* Open all log files and data files in the system
  tablespace: we keep them open until database shutdown.

  When we use goto files_checked; we don't need the line below,
  because in such case, it's been already called at the end of
  create_log_files_rename(). */

  fil_open_log_and_system_tablespace_files();

files_checked:

  if (dblwr::enabled && ((err = dblwr::open(create_new_db)) != DB_SUCCESS)) {
    return (srv_init_abort(err));
  }

  arch_init();

  mtr_t::s_logging.init();

  if (create_new_db) {
    ut_a(!srv_read_only_mode);

    ut_a(log_sys->last_checkpoint_lsn.load() ==
         LOG_START_LSN + LOG_BLOCK_HDR_SIZE);

    ut_a(flushed_lsn == LOG_START_LSN + LOG_BLOCK_HDR_SIZE);

    log_start(*log_sys, 0, flushed_lsn, flushed_lsn);

    log_start_background_threads(*log_sys);

    err = srv_undo_tablespaces_init(true);

    if (err != DB_SUCCESS) {
      return (srv_init_abort(err));
    }

    mtr_start(&mtr);

    bool ret = fsp_header_init(0, sum_of_new_sizes, &mtr, false);

    mtr_commit(&mtr);

    if (!ret) {
      return (srv_init_abort(DB_ERROR));
    }

    /* To maintain backward compatibility we create only
    the first rollback segment before the double write buffer.
    All the remaining rollback segments will be created later,
    after the double write buffers haves been created. */
    trx_sys_create_sys_pages();

    purge_queue = trx_sys_init_at_db_start();

    /* The purge system needs to create the purge view and
    therefore requires that the trx_sys is inited. */

    trx_purge_sys_create(srv_threads.m_purge_workers_n, purge_queue);

    err = dict_create();

    if (err != DB_SUCCESS) {
      return (srv_init_abort(err));
    }

    srv_create_sdi_indexes();

    previous_lsn = log_get_lsn(*log_sys);

    buf_flush_sync_all_buf_pools();

    log_stop_background_threads(*log_sys);

    flushed_lsn = log_get_lsn(*log_sys);

    ut_a(flushed_lsn == previous_lsn);

    err = fil_write_flushed_lsn(flushed_lsn);
    ut_a(err == DB_SUCCESS);

    create_log_files_rename(logfilename, dirnamelen, new_checkpoint_lsn,
                            logfile0);

    log_start_background_threads(*log_sys);

    ut_a(buf_are_flush_lists_empty_validate());

    /* We always create the legacy double write buffer to preserve the
    expected page ordering of the system tablespace.
    FIXME: Try and remove this requirement. */
    err = dblwr::v1::create();

    if (err != DB_SUCCESS) {
      return srv_init_abort(err);
    }

  } else {
    /* Load the reserved boundaries of the legacy dblwr buffer, this is
    requird to check for stray reads and writes trying to access this
    reserved region in the sys tablespace.
    FIXME: Try and remove this requirement. */
    err = dblwr::v1::init();

    if (err != DB_SUCCESS) {
      return srv_init_abort(err);
    }

    /* Invalidate the buffer pool to ensure that we reread
    the page that we read above, during recovery.
    Note that this is not as heavy weight as it seems. At
    this point there will be only ONE page in the buf_LRU
    and there must be no page in the buf_flush list. */
    buf_pool_invalidate();

    /* We always try to do a recovery, even if the database had
    been shut down normally: this is the normal startup path */
    /**
从 checkpoint  flushed_lsn 位置开始恢复。
1. 初始化红黑树, 以便在恢复的过程中快速插入 flush 列表。
2. 在 log groups 中查找 latest checkpoint
3. 读取 latest checkpoint 所在的 redo log 页到 log_sys->checkpoint_buf中
4. 获取 checkpoint_lsn 和 checkpoint_no
5. 从 checkpoing_lsn 读取 redo log 到 hash 表中。
6. 检查 crash recovery 所需的表空间, 处理并删除double write buf 中的数据页, 这里会检查double write buf 中页对应的真实数据页的
完整性, 如果有问题, 则使用 double write buf 中页进行恢复。同时, 生成后台线程 recv_writer_thread 以清理缓冲池中的脏页。
7. 将日志段从最新的日志组复制到其他组, 我们目前只有一个日志组。
*/
    err = recv_recovery_from_checkpoint_start(*log_sys, flushed_lsn);

    if (err == DB_SUCCESS) {
      arch_page_sys->post_recovery_init();

      /* Initialize the change buffer. */
      err = dict_boot();
    }

    if (err != DB_SUCCESS) {
      return (srv_init_abort(err));
    }

    ut_ad(clone_check_recovery_crashpoint(recv_sys->is_cloned_db));

    /* We need to start log threads before asking to flush
    all dirty pages. That's because some dirty pages could
    be dirty because of ibuf merges. The ibuf merges could
    have written log records to the log buffer. The redo
    log has to be flushed up to the newest_modification of
    a dirty page, before the page might be flushed to disk.
    Hence we need the log_flusher thread which will flush
    log records related to the ibuf merges, allowing to
    flush the modified pages. That's why we need to start
    the log threads before flushing dirty pages. */

    if (!srv_read_only_mode) {
      log_start_background_threads(*log_sys);
    }

    if (srv_force_recovery < SRV_FORCE_NO_LOG_REDO) {
      /* Apply the hashed log records to the
      respective file pages, for the last batch of
      recv_group_scan_log_recs(). */

      /* Don't allow IBUF operations for cloned database
      recovery as it would add extra redo log and we may
      not have enough margin. */
      if (recv_sys->is_cloned_db) {
        recv_apply_hashed_log_recs(*log_sys, false);

      } else {
        recv_apply_hashed_log_recs(*log_sys, true);
      }

      if (recv_sys->found_corrupt_log) {
        err = DB_ERROR;
        return (srv_init_abort(err));
      }

      DBUG_PRINT("ib_log", ("apply completed"));

      /* Check and print if there were any tablespaces
      which had redo log records but we couldn't apply
      them because the filenames were missing. */
    }

    if (srv_force_recovery < SRV_FORCE_NO_LOG_REDO) {
      /* Recovery complete, start verifying the
      page LSN on read. */
      recv_lsn_checks_on = true;
    }

    /* We have gone through the redo log, now check if all the
    tablespaces were found and recovered. */

    if (srv_force_recovery == 0 && fil_check_missing_tablespaces()) {
      ib::error(ER_IB_MSG_1139);

      /* Set the abort flag to true. */
      /*
			完成 recovery 操作。
			1. 确保 recv_writer 线程已完成
			2. 等待 flush 操作完成, flush脏页操作已经完成
			3. 等待 recv_writer 线程终止
			4. 释放 flush 红黑树
			5. 回滚所有的数据字典表的事务，以便数据字典表没有被锁定。数据字典 latch 应保证一次只有一个数据字典事务处于活跃状态。
		*/
      auto p = recv_recovery_from_checkpoint_finish(*log_sys, true);

      ut_a(p == nullptr);

      return (srv_init_abort(DB_ERROR));
    }

    /* We have successfully recovered from the redo log. The
    data dictionary should now be readable. */

    if (recv_sys->found_corrupt_log) {
      ib::warn(ER_IB_MSG_1140);
    }

    if (!srv_force_recovery && !srv_read_only_mode) {
      buf_flush_sync_all_buf_pools();
    }

    srv_dict_metadata = recv_recovery_from_checkpoint_finish(*log_sys, false);

    /* We need to save the dynamic metadata collected from redo log to DD
    buffer table here. This is to make sure that the dynamic metadata is not
    lost by any future checkpoint. Since DD and data dictionary in memory
    objects are not fully initialized at this point, the usual mechanism to
    persist dynamic metadata at checkpoint wouldn't work. */

    if (srv_dict_metadata != nullptr && !srv_dict_metadata->empty()) {
      /* Open this table in case srv_dict_metadata should be applied to this
      table before checkpoint. And because DD is not fully up yet, the table
      can be opened by internal APIs. */

      fil_space_t *space = fil_space_acquire_silent(dict_sys_t::s_space_id);
      if (space == nullptr) {
        dberr_t error =
            fil_ibd_open(true, FIL_TYPE_TABLESPACE, dict_sys_t::s_space_id,
                         predefined_flags, dict_sys_t::s_dd_space_name,
                         dict_sys_t::s_dd_space_file_name, true, false);
        if (error != DB_SUCCESS) {
          ib::error(ER_IB_MSG_1142);
          return (srv_init_abort(DB_ERROR));
        }
      } else {
        fil_space_release(space);
      }

      dict_persist->table_buffer = UT_NEW_NOKEY(DDTableBuffer());
      /* We write redo log here. We assume that there should be enough room in
      log files, supposing log_free_check() works fine before crash. */
      srv_dict_metadata->store();

      /* Flush logs to persist the changes. */
      log_buffer_flush_to_disk(*log_sys);
    }

    if (!srv_force_recovery && !recv_sys->found_corrupt_log &&
        (srv_log_file_size_requested != srv_log_file_size ||
         srv_n_log_files_found != srv_n_log_files)) {
      /* Prepare to replace the redo log files. */

      if (srv_read_only_mode) {
        ib::error(ER_IB_MSG_1141);
        return (srv_init_abort(DB_READ_ONLY));
      }

      /* Prepare to delete the old redo log files */
      flushed_lsn = srv_prepare_to_delete_redo_log_files(i);

      log_stop_background_threads(*log_sys);

      /* Prohibit redo log writes from any other
      threads until creating a log checkpoint at the
      end of create_log_files(). */
      ut_d(log_sys->disable_redo_writes = true);

      ut_ad(!buf_pool_check_no_pending_io());

      RECOVERY_CRASH(3);

      /* Stamp the LSN to the data files. */
      err = fil_write_flushed_lsn(flushed_lsn);
      ut_a(err == DB_SUCCESS);

      RECOVERY_CRASH(4);

      /* Close and free the redo log files, so that
      we can replace them. */
      fil_close_log_files(true);

      RECOVERY_CRASH(5);

      log_sys_close();

      /* Finish clone file recovery before creating new log files. We
      roll forward to remove any intermediate files here. */
      clone_files_recovery(true);

      ib::info(ER_IB_MSG_1143);

      srv_log_file_size = srv_log_file_size_requested;

      err =
          create_log_files(logfilename, dirnamelen, flushed_lsn,
                           srv_n_log_files_found, logfile0, new_checkpoint_lsn);

      if (err != DB_SUCCESS) {
        return (srv_init_abort(err));
      }

      create_log_files_rename(logfilename, dirnamelen, new_checkpoint_lsn,
                              logfile0);

      ut_d(log_sys->disable_redo_writes = false);

      flushed_lsn = new_checkpoint_lsn;

      log_start(*log_sys, 0, flushed_lsn, flushed_lsn);

      log_start_background_threads(*log_sys);

    } else if (recv_sys->is_cloned_db) {
      /* Reset creator for log */

      log_stop_background_threads(*log_sys);

      log_files_header_read(*log_sys, 0);

      lsn_t start_lsn;
      start_lsn =
          mach_read_from_8(log_sys->checkpoint_buf + LOG_HEADER_START_LSN);

      log_files_header_read(*log_sys, LOG_CHECKPOINT_1);

      log_files_header_flush(*log_sys, 0, start_lsn);

      log_start_background_threads(*log_sys);
    }

    if (sum_of_new_sizes > 0) {
      /* New data file(s) were added */
      mtr_start(&mtr);

      fsp_header_inc_size(0, sum_of_new_sizes, &mtr);

      mtr_commit(&mtr);

      /* Immediately write the log record about
      increased tablespace size to disk, so that it
      is durable even if mysqld would crash
      quickly */

      log_buffer_flush_to_disk(*log_sys);
    }

    err = srv_undo_tablespaces_init(false);

    if (err != DB_SUCCESS && srv_force_recovery < SRV_FORCE_NO_UNDO_LOG_SCAN) {
      return (srv_init_abort(err));
    }

    purge_queue = trx_sys_init_at_db_start();

    if (srv_is_upgrade_mode) {
      if (!purge_queue->empty()) {
        ib::info(ER_IB_MSG_1144);
        srv_upgrade_old_undo_found = true;
      }
      /* Either the old or new undo tablespaces will
      be deleted later depending on the value of
      'failed_upgrade' in dd_upgrade_finish(). */
    } else {
      /* New undo tablespaces have been created.
      Delete the old undo tablespaces and the references
      to them in the TRX_SYS page. */
      srv_undo_tablespaces_upgrade();
    }

    DBUG_EXECUTE_IF("check_no_undo", ut_ad(purge_queue->empty()););

    /* The purge system needs to create the purge view and
    therefore requires that the trx_sys and trx lists were
    initialized in trx_sys_init_at_db_start(). */
    trx_purge_sys_create(srv_threads.m_purge_workers_n, purge_queue);
  }

  /* Open temp-tablespace and keep it open until shutdown. */
  err = srv_open_tmp_tablespace(create_new_db, &srv_tmp_space);
  if (err != DB_SUCCESS) {
    return (srv_init_abort(err));
  }

  err = ibt::open_or_create(create_new_db);
  if (err != DB_SUCCESS) {
    return (srv_init_abort(err));
  }

  /* Here the double write buffer has already been created and so
  any new rollback segments will be allocated after the double
  write buffer. The default segment should already exist.
  We create the new segments only if it's a new database or
  the database was shutdown cleanly. */

  /* Note: When creating the extra rollback segments during an upgrade
  we violate the latching order, even if the change buffer is empty.
  We make an exception in sync0sync.cc and check srv_is_being_started
  for that violation. It cannot create a deadlock because we are still
  running in single threaded mode essentially. Only the IO threads
  should be running at this stage. */

  ut_a(srv_rollback_segments > 0);
  ut_a(srv_rollback_segments <= TRX_SYS_N_RSEGS);

  /* Make sure there are enough rollback segments in each tablespace
  and that each rollback segment has an associated memory object.
  If any of these rollback segments contain undo logs, load them into
  the purge queue */
  if (!trx_rseg_adjust_rollback_segments(srv_rollback_segments)) {
    return (srv_init_abort(DB_ERROR));
  }

  /* Any undo tablespaces under construction are now fully built
  with all needed rsegs. Delete the trunc.log files and clear the
  construction list. */
  srv_undo_tablespaces_mark_construction_done();

  /* Now that all rsegs are ready for use, make them active. */
  undo::spaces->s_lock();
  for (auto undo_space : undo::spaces->m_spaces) {
    if (!undo_space->is_empty()) {
      undo_space->set_active();
    }
  }
  undo::spaces->s_unlock();

  /* Undo Tablespaces and Rollback Segments are ready. */
  srv_startup_is_before_trx_rollback_phase = false;

  if (!srv_read_only_mode) {
    if (create_new_db) {
      srv_buffer_pool_load_at_startup = FALSE;
    }

    /* Create the thread which watches the timeouts
    for lock waits */
    srv_threads.m_lock_wait_timeout =
        os_thread_create(srv_lock_timeout_thread_key, lock_wait_timeout_thread);

    srv_threads.m_lock_wait_timeout.start();

    /* Create the thread which warns of long semaphore waits */
    srv_threads.m_error_monitor = os_thread_create(srv_error_monitor_thread_key,
                                                   srv_error_monitor_thread);

    srv_threads.m_error_monitor.start();

    /* Create the thread which prints InnoDB monitor info */
    srv_threads.m_monitor =
        os_thread_create(srv_monitor_thread_key, srv_monitor_thread);

    srv_threads.m_monitor.start();

    srv_start_state_set(SRV_START_STATE_MONITOR);
  }

  srv_sys_tablespaces_open = true;

  /* Rotate the encryption key for recovery. It's because
  server could crash in middle of key rotation. Some tablespace
  didn't complete key rotation. Here, we will resume the
  rotation. */
  if (!srv_read_only_mode && !create_new_db &&
      srv_force_recovery < SRV_FORCE_NO_LOG_REDO) {
    size_t fail_count = fil_encryption_rotate();
    if (fail_count > 0) {
      ib::info(ER_IB_MSG_1146)
          << "During recovery, fil_encryption_rotate() failed for "
          << fail_count << " tablespace(s).";
    }
  }

  srv_is_being_started = false;

  ut_a(trx_purge_state() == PURGE_STATE_INIT);

  /* wake main loop of page cleaner up */
  os_event_set(buf_flush_event);

  sum_of_data_file_sizes = srv_sys_space.get_sum_of_sizes();
  ut_a(sum_of_new_sizes != FIL_NULL);

  tablespace_size_in_header = fsp_header_get_tablespace_size();

  if (!srv_read_only_mode && !srv_sys_space.can_auto_extend_last_file() &&
      sum_of_data_file_sizes != tablespace_size_in_header) {
    ib::error(ER_IB_MSG_1147, ulong{tablespace_size_in_header},
              ulong{sum_of_data_file_sizes});

    if (srv_force_recovery == 0 &&
        sum_of_data_file_sizes < tablespace_size_in_header) {
      /* This is a fatal error, the tail of a tablespace is
      missing */

      ib::error(ER_IB_MSG_1148);

      return (srv_init_abort(DB_ERROR));
    }
  }

  if (!srv_read_only_mode && srv_sys_space.can_auto_extend_last_file() &&
      sum_of_data_file_sizes < tablespace_size_in_header) {
    ib::error(ER_IB_MSG_1149, ulong{tablespace_size_in_header},
              ulong{sum_of_data_file_sizes});

    if (srv_force_recovery == 0) {
      ib::error(ER_IB_MSG_1150);

      return (srv_init_abort(DB_ERROR));
    }
  }

  /* Finish clone files recovery. This call is idempotent and is no op
  if it is already done before creating new log files. */
  clone_files_recovery(true);

  ib::info(ER_IB_MSG_1151, INNODB_VERSION_STR,
           ulonglong{log_get_lsn(*log_sys)});

  return (DB_SUCCESS);
}


```
好好分析一下这个函数，老复杂了。一千行的大函数，不得不说，写这代码的人是真牛逼还是假牛逼，还是逼不得已干这种事。
第一个需要分析是srv_boot这个函数，这玩意儿还跳过去，在srv0srv.cc这在同一个目录下：
```
/** Boots the InnoDB server. */
void srv_boot(void) {
  /* Initialize synchronization primitives, memory management, and thread
  local storage */

  srv_general_init();

  /* Initialize this module */

  srv_init();
}
/** Initializes the synchronization primitives, memory system, and the thread
 local storage. */
static void srv_general_init() {
  sync_check_init(srv_max_n_threads);
  /* Reset the system variables in the recovery module. */
  recv_sys_var_init();
  os_thread_open();
  trx_pool_init();
  que_init();
  row_mysql_init();
  undo_spaces_init();
}

/** Initializes the server. */
static void srv_init(void) {
  ulint n_sys_threads = 0;
  ulint srv_sys_sz = sizeof(*srv_sys);

  mutex_create(LATCH_ID_SRV_INNODB_MONITOR, &srv_innodb_monitor_mutex);

  ut_d(srv_threads.m_shutdown_cleanup_dbg = os_event_create());

  srv_threads.m_master_ready_for_dd_shutdown = os_event_create();

  srv_threads.m_purge_coordinator = {};

  srv_threads.m_purge_workers_n = srv_n_purge_threads;

  srv_threads.m_purge_workers =
      UT_NEW_ARRAY_NOKEY(IB_thread, srv_threads.m_purge_workers_n);

  if (!srv_read_only_mode) {
    /* Number of purge threads + master thread */
    n_sys_threads = srv_n_purge_threads + 1;

    srv_sys_sz += n_sys_threads * sizeof(*srv_sys->sys_threads);
  }

  srv_threads.m_page_cleaner_coordinator = {};

  srv_threads.m_page_cleaner_workers_n = srv_n_page_cleaners;

  srv_threads.m_page_cleaner_workers =
      UT_NEW_ARRAY_NOKEY(IB_thread, srv_threads.m_page_cleaner_workers_n);

  srv_sys = static_cast<srv_sys_t *>(ut_zalloc_nokey(srv_sys_sz));

  srv_sys->n_sys_threads = n_sys_threads;

  /* Even in read-only mode we flush pages related to intrinsic table
  and so mutex creation is needed. */
  {
    mutex_create(LATCH_ID_SRV_SYS, &srv_sys->mutex);

    mutex_create(LATCH_ID_SRV_SYS_TASKS, &srv_sys->tasks_mutex);

    srv_sys->sys_threads = (srv_slot_t *)&srv_sys[1];

    for (ulint i = 0; i < srv_sys->n_sys_threads; ++i) {
      srv_slot_t *slot = &srv_sys->sys_threads[i];

      slot->event = os_event_create();

      slot->in_use = false;

      ut_a(slot->event);
    }

    srv_error_event = os_event_create();

    srv_monitor_event = os_event_create();

    srv_buf_dump_event = os_event_create();

    buf_flush_event = os_event_create();

    buf_flush_tick_event = os_event_create();

    UT_LIST_INIT(srv_sys->tasks, &que_thr_t::queue);
  }

  srv_buf_resize_event = os_event_create();

  ut_d(srv_master_thread_disabled_event = os_event_create());

  /* page_zip_stat_per_index_mutex is acquired from:
  1. page_zip_compress() (after SYNC_FSP)
  2. page_zip_decompress()
  3. i_s_cmp_per_index_fill_low() (where SYNC_DICT is acquired)
  4. innodb_cmp_per_index_update(), no other latches
  since we do not acquire any other latches while holding this mutex,
  it can have very low level. We pick SYNC_ANY_LATCH for it. */
  mutex_create(LATCH_ID_PAGE_ZIP_STAT_PER_INDEX,
               &page_zip_stat_per_index_mutex);

  /* Create dummy indexes for infimum and supremum records */

  dict_ind_init();

  /* Initialize some INFORMATION SCHEMA internal structures * /
  trx_i_s_cache_init(trx_i_s_cache);

  ut_crc32_init();

  dict_mem_init();
}

```
srv_general_init这个函数是对线程本地参数、互斥体和内存进行初始化，在此函数内部可以看到很多相关的初始化的函数，这里有一个技巧，如果只是跳到inclue中的头文件中去，那就在本路径下的相关同名的文件件下找相关的同名文件.cc即可找到相关的内容。
再下面就是创建缓存的函数os_create_block_cache：
```
/** Creates and initializes block_cache. Creates array of MAX_BLOCKS
and allocates the memory in each block to hold BUFFER_BLOCK_SIZE
of data.

This function is called by InnoDB during srv_start().
It is also called by MEB while applying the redo logs on TDE tablespaces,
the "Blocks" allocated in this block_cache are used to hold the decrypted
page data. */
void os_create_block_cache() {
  ut_a(block_cache == nullptr);

  block_cache = UT_NEW_NOKEY(Blocks(MAX_BLOCKS));

  for (Blocks::iterator it = block_cache->begin(); it != block_cache->end();
       ++it) {
    ut_a(!it->m_in_use);
    ut_a(it->m_ptr == nullptr);

    /* Allocate double of max page size memory, since
    compress could generate more bytes than original
    data. * /
    it->m_ptr = static_cast<byte * >(ut_malloc_nokey(BUFFER_BLOCK_SIZE));

    ut_a(it->m_ptr != nullptr);
  }
}

```
这个比较简单，就是一个缓冲池的创建。这个块也可用来存储TDE表空间中的数据，保存的为解密后的页数据。
然后对表空间的相关缓存进行初始化：
```
/** Initializes the tablespace memory cache.
@param[in]	max_n_open	Maximum number of open files */
void fil_init(ulint max_n_open) {
  static_assert((1 << UNIV_PAGE_SIZE_SHIFT_MAX) == UNIV_PAGE_SIZE_MAX,
                "(1 << UNIV_PAGE_SIZE_SHIFT_MAX) != UNIV_PAGE_SIZE_MAX");

  static_assert((1 << UNIV_PAGE_SIZE_SHIFT_MIN) == UNIV_PAGE_SIZE_MIN,
                "(1 << UNIV_PAGE_SIZE_SHIFT_MIN) != UNIV_PAGE_SIZE_MIN");

  ut_a(fil_system == nullptr);

  ut_a(max_n_open > 0);

  fil_system = UT_NEW_NOKEY(Fil_system(MAX_SHARDS, max_n_open));
}

```
clone_init是对新增的数据库克隆的支持，接下来进入InnoDB引擎的文件打开阶段，这个没啥可说，直接用的C的文件库，主要是异常处理部分比较复杂，然后os_file_create_tmpfile创建临时文件。
需要关注的是异步通信的处理：
```
/** Initializes the asynchronous io system. Creates one array each for ibuf
and log i/o. Also creates one array each for read and write where each
array is divided logically into n_readers and n_writers
respectively. The caller must create an i/o handler thread for each
segment in these arrays. This function also creates the sync array.
No i/o handler thread needs to be created for that
@param[in]	n_readers	number of reader threads
@param[in]	n_writers	number of writer threads
@param[in]	n_slots_sync	number of dblwr slots in the sync aio array */
bool os_aio_init(ulint n_readers, ulint n_writers, ulint n_slots_sync) {
  /* Maximum number of pending aio operations allowed per segment * /
  ulint limit = 8 * OS_AIO_N_PENDING_IOS_PER_THREAD;

#ifdef _WIN32
  if (srv_use_native_aio) {
    limit = SRV_N_PENDING_IOS_PER_THREAD;
  }
#endif /* _WIN32 * /

  / * Get sector size for DIRECT_IO. In this case, we need to
  know the sector size for aligning the write buffer. * /
#if !defined(NO_FALLOCATE) && defined(UNIV_LINUX)
  os_fusionio_get_sector_size();
#endif /* !NO_FALLOCATE && UNIV_LINUX * /

  return (AIO::start(limit, n_readers, n_writers, n_slots_sync));
}

```
AIO::start就是AIO库的内容了，这里不再深入分析。它就是Linux提供的一个异步IO子系统。
接下来创建缓冲池：
```
/** Creates the buffer pool.
@param[in]  total_size    Size of the total pool in bytes.
@param[in]  n_instances   Number of buffer pool instances to create.
@return DB_SUCCESS if success, DB_ERROR if not enough memory or error */
dberr_t buf_pool_init(ulint total_size, ulint n_instances) {
  ulint i;
  const ulint size = total_size / n_instances;

  ut_ad(n_instances > 0);
  ut_ad(n_instances <= MAX_BUFFER_POOLS);
  ut_ad(n_instances == srv_buf_pool_instances);

  NUMA_MEMPOLICY_INTERLEAVE_IN_SCOPE;

  /* Usually buf_pool_should_madvise is protected by buf_pool_t::chunk_mutex-es,
  but at this point in time there is no buf_pool_t instances yet, and no risk of
  race condition with sys_var modifications or buffer pool resizing because we
  have just started initializing the buffer pool.*/
  buf_pool_should_madvise = innobase_should_madvise_buf_pool();

  buf_pool_resizing = false;

  buf_pool_ptr =
      (buf_pool_t *)ut_zalloc_nokey(n_instances * sizeof *buf_pool_ptr);

  buf_chunk_map_reg = UT_NEW_NOKEY(buf_pool_chunk_map_t());

  std::vector<dberr_t> errs;

  errs.assign(n_instances, DB_SUCCESS);

#ifdef UNIV_LINUX
  ulint n_cores = sysconf(_SC_NPROCESSORS_ONLN);

  /* Magic nuber 8 is from empirical testing on a
  4 socket x 10 Cores x 2 HT host. 128G / 16 instances
  takes about 4 secs, compared to 10 secs without this
  optimisation.. */

  if (n_cores > 8) {
    n_cores = 8;
  }
#else
  ulint n_cores = 4;
#endif /* UNIV_LINUX */

  dberr_t err = DB_SUCCESS;

  for (i = 0; i < n_instances; /* no op */) {
    ulint n = i + n_cores;

    if (n > n_instances) {
      n = n_instances;
    }

    std::vector<std::thread> threads;

    std::mutex m;

    for (ulint id = i; id < n; ++id) {
      threads.emplace_back(std::thread(buf_pool_create, &buf_pool_ptr[id], size,
                                       id, &m, std::ref(errs[id])));
    }

    for (ulint id = i; id < n; ++id) {
      threads[id - i].join();

      if (errs[id] != DB_SUCCESS) {
        err = errs[id];
      }
    }

    if (err != DB_SUCCESS) {
      for (size_t id = 0; id < n; ++id) {
        if (buf_pool_ptr[id].chunks != nullptr) {
          buf_pool_free_instance(&buf_pool_ptr[id]);
        }
      }

      buf_pool_free();

      return (err);
    }

    /* Do the next block of instances */
    i = n;
  }

  buf_pool_set_sizes();
  buf_LRU_old_ratio_update(100 * 3 / 8, FALSE);

  btr_search_sys_create(buf_pool_get_curr_size() / sizeof(void *) / 64);

  buf_stat_per_index =
      UT_NEW(buf_stat_per_index_t(), mem_key_buf_stat_per_index_t);

  return (DB_SUCCESS);
}

```
下面几个初始化和创建函数只分析一下事务锁的创建：
```

/** Creates the trx_sys instance and initializes purge_queue and mutex. */
void trx_sys_create(void) {
  ut_ad(trx_sys == nullptr);

  trx_sys = static_cast<trx_sys_t *>(ut_zalloc_nokey(sizeof(*trx_sys)));

  mutex_create(LATCH_ID_TRX_SYS, &trx_sys->mutex);

  UT_LIST_INIT(trx_sys->serialisation_list, &trx_t::no_list);
  UT_LIST_INIT(trx_sys->rw_trx_list, &trx_t::trx_list);
  UT_LIST_INIT(trx_sys->mysql_trx_list, &trx_t::mysql_trx_list);

  trx_sys->mvcc = UT_NEW_NOKEY(MVCC(1024));

  trx_sys->min_active_id = 0;

  ut_d(trx_sys->rw_max_trx_no = 0);

  new (&trx_sys->rw_trx_ids)
      trx_ids_t(ut_allocator<trx_id_t>(mem_key_trx_sys_t_rw_trx_ids));

  new (&trx_sys->rw_trx_set) TrxIdSet();

  new (&trx_sys->rsegs) Rsegs();
  trx_sys->rsegs.set_empty();

  new (&trx_sys->tmp_rsegs) Rsegs();
  trx_sys->tmp_rsegs.set_empty();
}

```
purge系统，包括代码中的trx和thread等，都是用来做最后清理动作的或者说真正的操作删除是在这一部分来做。前面的删除都是一个标记或者预处理的过程 。
在接下的线程创建中，使用的同样的POSIX中的线程库，看一下相关的宏定义：
```
#ifdef UNIV_PFS_THREAD
#define os_thread_create(...) create_detached_thread(__VA_ARGS__)
#else
#define os_thread_create(k, ...) create_detached_thread(0, __VA_ARGS__)
#endif /* UNIV_PFS_THREAD */

```
这都没啥可讲的，如果不知道，只好去看线程创建的相关知识了。
```
/** Initialize page_cleaner.
@param[in]	n_page_cleaners	Number of page cleaner threads to create */
void buf_flush_page_cleaner_init(size_t n_page_cleaners) {
  ut_ad(page_cleaner == nullptr);

  page_cleaner =
      static_cast<page_cleaner_t *>(ut_zalloc_nokey(sizeof(*page_cleaner)));

  mutex_create(LATCH_ID_PAGE_CLEANER, &page_cleaner->mutex);

  page_cleaner->is_requested = os_event_create();
  page_cleaner->is_finished = os_event_create();

  page_cleaner->n_slots = static_cast<ulint>(srv_buf_pool_instances);

  page_cleaner->slots = static_cast<page_cleaner_slot_t *>(
      ut_zalloc_nokey(page_cleaner->n_slots * sizeof(*page_cleaner->slots)));

  ut_d(page_cleaner->n_disabled_debug = 0);

  page_cleaner->is_running = true;

  srv_threads.m_page_cleaner_coordinator =
      os_thread_create(page_flush_coordinator_thread_key,
                       buf_flush_page_coordinator_thread, n_page_cleaners);

  srv_threads.m_page_cleaner_workers[0] =
      srv_threads.m_page_cleaner_coordinator;

  srv_threads.m_page_cleaner_coordinator.start();

  /* Make sure page cleaner is active. */
  ut_a(buf_flush_page_cleaner_is_active());
}

```
buf_flush_page_cleaner_init这个函数是在5.7之后增加的，用来处理脏数据的一个机制，类似于一个工作者的线程分配机制，能更好更快的刷新处理脏数据。
然后又是对表空内的数据文件的处理，看是否存在，打开或者新建立它：
```
/** Open or Create the data files if they do not exist.
@param[in]	is_temp	whether this is a temporary tablespace
@return DB_SUCCESS or error code */
dberr_t Tablespace::open_or_create(bool is_temp) {
  fil_space_t *space = nullptr;
  dberr_t err = DB_SUCCESS;

  ut_ad(!m_files.empty());

  files_t::iterator begin = m_files.begin();
  files_t::iterator end = m_files.end();

  for (files_t::iterator it = begin; it != end; ++it) {
    if (it->m_exists) {
      err = it->open_or_create(m_ignore_read_only ? false : srv_read_only_mode);
    } else {
      err = it->open_or_create(m_ignore_read_only ? false : srv_read_only_mode);

      /* Set the correct open flags now that we have
      successfully created the file. */
      if (err == DB_SUCCESS) {
        file_found(*it);
      }
    }

    if (err != DB_SUCCESS) {
      break;
    }

    bool atomic_write;

#if !defined(NO_FALLOCATE) && defined(UNIV_LINUX)
    if (!dblwr::enabled) {
      atomic_write = fil_fusionio_enable_atomic_write(it->m_handle);
    } else {
      atomic_write = false;
    }
#else
    atomic_write = false;
#endif /* !NO_FALLOCATE && UNIV_LINUX */

    /* We can close the handle now and open the tablespace
    the proper way. */
    it->close();

    if (it == begin) {
      /* First data file. */

      uint32_t flags = fsp_flags_set_page_size(0, univ_page_size);

      /* Create the tablespace entry for the multi-file
      tablespace in the tablespace manager. */
      space =
          fil_space_create(m_name, m_space_id, flags,
                           is_temp ? FIL_TYPE_TEMPORARY : FIL_TYPE_TABLESPACE);
    }

    ut_ad(fil_validate());

    /* Create the tablespace node entry for this data file. */
    if (!fil_node_create(it->m_filepath, it->m_size, space, false,
                         atomic_write)) {
      err = DB_ERROR;
      break;
    }
  }

  return (err);
}

```
dict_persist_init这个函数，用来管理动态元数据的相关信息持久化的初始化。
```
/** Inits the structure for persisting dynamic metadata */
void dict_persist_init(void) {
  dict_persist =
      static_cast<dict_persist_t *>(ut_zalloc_nokey(sizeof(*dict_persist)));

  mutex_create(LATCH_ID_DICT_PERSIST_DIRTY_TABLES, &dict_persist->mutex);

#ifndef UNIV_HOTBACKUP
  UT_LIST_INIT(dict_persist->dirty_dict_tables,
               &dict_table_t::dirty_dict_tables);
#endif /* !UNIV_HOTBACKUP */

  dict_persist->num_dirty_tables = 0;

  dict_persist->persisters = UT_NEW_NOKEY(Persisters());
  dict_persist->persisters->add(PM_INDEX_CORRUPTED);
  dict_persist->persisters->add(PM_TABLE_AUTO_INC);

#ifndef UNIV_HOTBACKUP
  dict_persist_update_log_margin();
#endif /* !UNIV_HOTBACKUP */
}

```
然后创建日志文件：
```
/** Creates all log files.
@param[in,out]  logfilename	    buffer for log file name
@param[in]      dirnamelen      length of the directory path
@param[in]      lsn             FIL_PAGE_FILE_FLUSH_LSN value
@param[in]      num_old_files   number of old redo log files to remove
@param[out]     logfile0	      name of the first log file
@param[out]     checkpoint_lsn  lsn of the first created checkpoint
@return DB_SUCCESS or error code */
static dberr_t create_log_files(char *logfilename, size_t dirnamelen, lsn_t lsn,
                                uint32_t num_old_files, char *&logfile0,
                                lsn_t &checkpoint_lsn) {
  dberr_t err;

  if (srv_read_only_mode) {
    ib::error(ER_IB_MSG_1064);
    return (DB_READ_ONLY);
  }

  if (num_old_files < INIT_LOG_FILE0) {
    num_old_files = INIT_LOG_FILE0;
  }

  /* Remove any old log files. */
  for (unsigned i = 0; i <= num_old_files; i++) {
    sprintf(logfilename + dirnamelen, "ib_logfile%u", i);

    /* Ignore errors about non-existent files or files
    that cannot be removed. The create_log_file() will
    return an error when the file exists. */
#ifdef _WIN32
    DeleteFile((LPCTSTR)logfilename);
#else
    unlink(logfilename);
#endif /* _WIN32 */
    /* Crashing after deleting the first
    file should be recoverable. The buffer
    pool was clean, and we can simply create
    all log files from the scratch. */
    RECOVERY_CRASH(6);
  }

  ut_ad(!buf_pool_check_no_pending_io());

  RECOVERY_CRASH(7);

  for (unsigned i = 0; i < srv_n_log_files; i++) {
    sprintf(logfilename + dirnamelen, "ib_logfile%u", i ? i : INIT_LOG_FILE0);

    err = create_log_file(&files[i], logfilename);

    if (err != DB_SUCCESS) {
      return (err);
    }
  }

  RECOVERY_CRASH(8);

  /* We did not create the first log file initially as
  ib_logfile0, so that crash recovery cannot find it until it
  has been completed and renamed. */
  sprintf(logfilename + dirnamelen, "ib_logfile%u", INIT_LOG_FILE0);

  /* Disable the doublewrite buffer for log files, not required */

  fil_space_t *log_space = fil_space_create(
      "innodb_redo_log", dict_sys_t::s_log_space_first_id,
      fsp_flags_set_page_size(0, univ_page_size), FIL_TYPE_LOG);

  ut_ad(fil_validate());
  ut_a(log_space != nullptr);

  /* Once the redo log is set to be encrypted,
  initialize encryption information. */
  if (srv_redo_log_encrypt) {
    if (!Encryption::check_keyring()) {
      ib::error(ER_IB_MSG_1065);

      return (DB_ERROR);
    }

    fsp_flags_set_encryption(log_space->flags);
    err = fil_set_encryption(log_space->id, Encryption::AES, nullptr, nullptr);
    ut_ad(err == DB_SUCCESS);
  }

  const ulonglong file_pages = srv_log_file_size / UNIV_PAGE_SIZE;

  logfile0 = fil_node_create(logfilename, static_cast<page_no_t>(file_pages),
                             log_space, false, false);

  ut_a(logfile0 != nullptr);

  for (unsigned i = 1; i < srv_n_log_files; i++) {
    sprintf(logfilename + dirnamelen, "ib_logfile%u", i);

    if (fil_node_create(logfilename, static_cast<page_no_t>(file_pages),
                        log_space, false, false) == nullptr) {
      ib::error(ER_IB_MSG_1066, logfilename);

      return (DB_ERROR);
    }
  }

  if (!log_sys_init(srv_n_log_files, srv_log_file_size,
                    dict_sys_t::s_log_space_first_id)) {
    return (DB_ERROR);
  }

  ut_a(log_sys != nullptr);

  fil_open_log_and_system_tablespace_files();

  /* Create the first checkpoint and flush headers of the first log
  file (the flushed headers store information about the checkpoint,
  format of redo log and that it is not created by mysqlbackup). */

  /* We start at the next log block. Note, that we keep invariant,
  that start lsn stored in header of the first log file is divisble
  by OS_FILE_LOG_BLOCK_SIZE. */
  lsn = ut_uint64_align_up(lsn, OS_FILE_LOG_BLOCK_SIZE);

  /* Checkpoint lsn should be outside header of log block. */
  lsn += LOG_BLOCK_HDR_SIZE;

  log_create_first_checkpoint(*log_sys, lsn);
  checkpoint_lsn = lsn;

  /* Write encryption information into the first log file header
  if redo log is set with encryption. */
  if (FSP_FLAGS_GET_ENCRYPTION(log_space->flags) &&
      !log_write_encryption(log_space->encryption_key, log_space->encryption_iv,
                            true)) {
    return (DB_ERROR);
  }

  /* Note that potentially some log files are still unflushed.
  However it does not matter, because ib_logfile0 is not present
  Before renaming ib_logfile101 to ib_logfile0, log files have
  to be flushed. We could postpone that to just before the rename,
  as we possibly will write some log records before doing the rename.

  However OS could anyway do the flush, and we prefer to minimize
  possible scenarios. Hence, to make situation more deterministic,
  we do the fsyncs now unconditionally and repeat the required
  flush just before the rename. */
  fil_flush_file_redo();

  return (DB_SUCCESS);
}

```
下来还有日志空间的创建，此处不再分析。
再其后就是相关节点的创建：
```
/** Attach a file to a tablespace. File must be closed.
@param[in]	name		file name (file must be closed)
@param[in]	size		file size in database blocks, rounded
                                downwards to an integer
@param[in,out]	space		space where to append
@param[in]	is_raw		true if a raw device or a raw disk partition
@param[in]	atomic_write	true if the file has atomic write enabled
@param[in]	max_pages	maximum number of pages in file
@return pointer to the file name
@retval nullptr if error */
char *fil_node_create(const char *name, page_no_t size, fil_space_t *space,
                      bool is_raw, bool atomic_write, page_no_t max_pages) {
  auto shard = fil_system->shard_by_id(space->id);

  fil_node_t *file;

  file = shard->create_node(name, size, space, is_raw,
                            IORequest::is_punch_hole_supported(), atomic_write,
                            max_pages);

  return file == nullptr ? nullptr : file->name;
}

```
启动mtr原子操作：
```
/** Start a mini-transaction.
@param sync		true if it is a synchronous mini-transaction
@param read_only	true if read only mini-transaction */
void mtr_t::start(bool sync, bool read_only) {
  ut_ad(m_impl.m_state == MTR_STATE_INIT ||
        m_impl.m_state == MTR_STATE_COMMITTED);

  UNIV_MEM_INVALID(this, sizeof(*this));

  UNIV_MEM_INVALID(&m_impl, sizeof(m_impl));

  m_sync = sync;

  m_commit_lsn = 0;

  new (&m_impl.m_log) mtr_buf_t();
  new (&m_impl.m_memo) mtr_buf_t();

  m_impl.m_mtr = this;
  m_impl.m_log_mode = MTR_LOG_ALL;
  m_impl.m_inside_ibuf = false;
  m_impl.m_modifications = false;
  m_impl.m_made_dirty = false;
  m_impl.m_n_log_recs = 0;
  m_impl.m_state = MTR_STATE_ACTIVE;
  m_impl.m_flush_observer = nullptr;
  m_impl.m_marked_nolog = false;

#ifndef UNIV_HOTBACKUP
  check_nolog_and_mark();
#endif /* !UNIV_HOTBACKUP */
  ut_d(m_impl.m_magic_n = MTR_MAGIC_N);

#ifdef UNIV_DEBUG
  auto res = s_my_thread_active_mtrs.insert(this);
  /* Assert there are no collisions in thread local context - it would mean
  reusing MTR without committing or destructing it. */
  ut_a(res.second);
#endif /* UNIV_DEBUG */
}


```
创建并初始化事物系统：
```
/** Creates and initializes the transaction system at the database creation. */
void trx_sys_create_sys_pages(void) {
  mtr_t mtr;

  mtr_start(&mtr);

  trx_sysf_create(&mtr);

  mtr_commit(&mtr);
}

```
还要创建SDI INDEXES序列化字典信息函数：
```
/** Create SDI Indexes in system tablespace. */
static void srv_create_sdi_indexes() {
  btr_sdi_create_index(SYSTEM_TABLE_SPACE, false);
}

```
还要点燃DICT：
```
/** Initializes the data dictionary memory structures when the database is
 started. This function is also called when the data dictionary is created.
 @return DB_SUCCESS or error code. */
dberr_t dict_boot(void) {
  dict_hdr_t *dict_hdr;
  mtr_t mtr;
  dberr_t err = DB_SUCCESS;

  mtr_start(&mtr);

  /* Create the hash tables etc. */
  dict_init();

  /* Get the dictionary header */
  dict_hdr = dict_hdr_get(&mtr);

  /* Because we only write new row ids to disk-based data structure
  (dictionary header) when it is divisible by
  DICT_HDR_ROW_ID_WRITE_MARGIN, in recovery we will not recover
  the latest value of the row id counter. Therefore we advance
  the counter at the database startup to avoid overlapping values.
  Note that when a user after database startup first time asks for
  a new row id, then because the counter is now divisible by
  ..._MARGIN, it will immediately be updated to the disk-based
  header. */

  dict_sys->row_id =
      DICT_HDR_ROW_ID_WRITE_MARGIN +
      ut_uint64_align_up(mach_read_from_8(dict_hdr + DICT_HDR_ROW_ID),
                         DICT_HDR_ROW_ID_WRITE_MARGIN);

  /* For upgrading, we need to load the old InnoDB internal SYS_*
  tables. */
  if (srv_is_upgrade_mode) {
    dict_table_t *table;
    dict_index_t *index;
    mem_heap_t *heap;

    /* Be sure these constants do not ever change.  To avoid bloat,
    only check the *NUM_FIELDS* in each table */
    ut_ad(DICT_NUM_COLS__SYS_TABLES == 8);
    ut_ad(DICT_NUM_FIELDS__SYS_TABLES == 10);
    ut_ad(DICT_NUM_FIELDS__SYS_TABLE_IDS == 2);
    ut_ad(DICT_NUM_COLS__SYS_COLUMNS == 7);
    ut_ad(DICT_NUM_FIELDS__SYS_COLUMNS == 9);
    ut_ad(DICT_NUM_COLS__SYS_INDEXES == 8);
    ut_ad(DICT_NUM_FIELDS__SYS_INDEXES == 10);
    ut_ad(DICT_NUM_COLS__SYS_FIELDS == 3);
    ut_ad(DICT_NUM_FIELDS__SYS_FIELDS == 5);
    ut_ad(DICT_NUM_COLS__SYS_FOREIGN == 4);
    ut_ad(DICT_NUM_FIELDS__SYS_FOREIGN == 6);
    ut_ad(DICT_NUM_FIELDS__SYS_FOREIGN_FOR_NAME == 2);
    ut_ad(DICT_NUM_COLS__SYS_FOREIGN_COLS == 4);
    ut_ad(DICT_NUM_FIELDS__SYS_FOREIGN_COLS == 6);

    heap = mem_heap_create(450);

    /* Insert into the dictionary cache the descriptions of the basic
    system tables */
    table = dict_mem_table_create("SYS_TABLES", DICT_HDR_SPACE, 8, 0, 0, 0, 0);

    dict_mem_table_add_col(table, heap, "NAME", DATA_BINARY, 0,
                           MAX_FULL_NAME_LEN, true);
    dict_mem_table_add_col(table, heap, "ID", DATA_BINARY, 0, 8, true);
    /* ROW_FORMAT = (N_COLS >> 31) ? COMPACT : REDUNDANT */
    dict_mem_table_add_col(table, heap, "N_COLS", DATA_INT, 0, 4, true);
    /* The low order bit of TYPE is always set to 1.  If ROW_FORMAT
    is not REDUNDANT or COMPACT, this field matches table->flags. */
    dict_mem_table_add_col(table, heap, "TYPE", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "MIX_ID", DATA_BINARY, 0, 0, true);
    /* MIX_LEN may contain additional table flags when
    ROW_FORMAT!=REDUNDANT.  Currently, these flags include
    DICT_TF2_TEMPORARY. */
    dict_mem_table_add_col(table, heap, "MIX_LEN", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "CLUSTER_NAME", DATA_BINARY, 0, 0,
                           true);
    dict_mem_table_add_col(table, heap, "SPACE", DATA_INT, 0, 4, true);

    table->id = DICT_TABLES_ID;

    dict_table_add_system_columns(table, heap);
    mutex_enter(&dict_sys->mutex);
    dict_table_add_to_cache(table, FALSE, heap);
    mutex_exit(&dict_sys->mutex);
    dict_sys->sys_tables = table;
    mem_heap_empty(heap);

    index = dict_mem_index_create("SYS_TABLES", "CLUST_IND", DICT_HDR_SPACE,
                                  DICT_UNIQUE | DICT_CLUSTERED, 1);

    index->add_field("NAME", 0, true);

    index->id = DICT_TABLES_ID;

    err = dict_index_add_to_cache(
        table, index,
        mtr_read_ulint(dict_hdr + DICT_HDR_TABLES, MLOG_4BYTES, &mtr), FALSE);
    ut_a(err == DB_SUCCESS);

    /*-------------------------*/
    index = dict_mem_index_create("SYS_TABLES", "ID_IND", DICT_HDR_SPACE,
                                  DICT_UNIQUE, 1);
    index->add_field("ID", 0, true);

    index->id = DICT_TABLE_IDS_ID;

    err = dict_index_add_to_cache(
        table, index,
        mtr_read_ulint(dict_hdr + DICT_HDR_TABLE_IDS, MLOG_4BYTES, &mtr),
        FALSE);
    ut_a(err == DB_SUCCESS);

    /*-------------------------*/
    table = dict_mem_table_create("SYS_COLUMNS", DICT_HDR_SPACE, 7, 0, 0, 0, 0);

    dict_mem_table_add_col(table, heap, "TABLE_ID", DATA_BINARY, 0, 8, true);
    dict_mem_table_add_col(table, heap, "POS", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "NAME", DATA_BINARY, 0, 0, true);
    dict_mem_table_add_col(table, heap, "MTYPE", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "PRTYPE", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "LEN", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "PREC", DATA_INT, 0, 4, true);

    table->id = DICT_COLUMNS_ID;

    dict_table_add_system_columns(table, heap);
    mutex_enter(&dict_sys->mutex);
    dict_table_add_to_cache(table, FALSE, heap);
    mutex_exit(&dict_sys->mutex);
    dict_sys->sys_columns = table;
    mem_heap_empty(heap);

    index = dict_mem_index_create("SYS_COLUMNS", "CLUST_IND", DICT_HDR_SPACE,
                                  DICT_UNIQUE | DICT_CLUSTERED, 2);

    index->add_field("TABLE_ID", 0, true);
    index->add_field("POS", 0, true);

    index->id = DICT_COLUMNS_ID;

    err = dict_index_add_to_cache(
        table, index,
        mtr_read_ulint(dict_hdr + DICT_HDR_COLUMNS, MLOG_4BYTES, &mtr), FALSE);
    ut_a(err == DB_SUCCESS);

    /*-------------------------*/
    table = dict_mem_table_create("SYS_INDEXES", DICT_HDR_SPACE,
                                  DICT_NUM_COLS__SYS_INDEXES, 0, 0, 0, 0);

    dict_mem_table_add_col(table, heap, "TABLE_ID", DATA_BINARY, 0, 8, true);
    dict_mem_table_add_col(table, heap, "ID", DATA_BINARY, 0, 8, true);
    dict_mem_table_add_col(table, heap, "NAME", DATA_BINARY, 0, 0, true);
    dict_mem_table_add_col(table, heap, "N_FIELDS", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "TYPE", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "SPACE", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "PAGE_NO", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "MERGE_THRESHOLD", DATA_INT, 0, 4,
                           true);

    table->id = DICT_INDEXES_ID;

    dict_table_add_system_columns(table, heap);
    mutex_enter(&dict_sys->mutex);
    dict_table_add_to_cache(table, FALSE, heap);
    mutex_exit(&dict_sys->mutex);
    dict_sys->sys_indexes = table;
    mem_heap_empty(heap);

    index = dict_mem_index_create("SYS_INDEXES", "CLUST_IND", DICT_HDR_SPACE,
                                  DICT_UNIQUE | DICT_CLUSTERED, 2);

    index->add_field("TABLE_ID", 0, true);
    index->add_field("ID", 0, true);

    index->id = DICT_INDEXES_ID;

    err = dict_index_add_to_cache(
        table, index,
        mtr_read_ulint(dict_hdr + DICT_HDR_INDEXES, MLOG_4BYTES, &mtr), FALSE);
    ut_a(err == DB_SUCCESS);

    /*-------------------------*/
    table = dict_mem_table_create("SYS_FIELDS", DICT_HDR_SPACE, 3, 0, 0, 0, 0);

    dict_mem_table_add_col(table, heap, "INDEX_ID", DATA_BINARY, 0, 8, true);
    dict_mem_table_add_col(table, heap, "POS", DATA_INT, 0, 4, true);
    dict_mem_table_add_col(table, heap, "COL_NAME", DATA_BINARY, 0, 0, true);

    table->id = DICT_FIELDS_ID;

    dict_table_add_system_columns(table, heap);
    mutex_enter(&dict_sys->mutex);
    dict_table_add_to_cache(table, FALSE, heap);
    mutex_exit(&dict_sys->mutex);
    dict_sys->sys_fields = table;
    mem_heap_free(heap);

    index = dict_mem_index_create("SYS_FIELDS", "CLUST_IND", DICT_HDR_SPACE,
                                  DICT_UNIQUE | DICT_CLUSTERED, 2);

    index->add_field("INDEX_ID", 0, true);
    index->add_field("POS", 0, true);

    index->id = DICT_FIELDS_ID;

    err = dict_index_add_to_cache(
        table, index,
        mtr_read_ulint(dict_hdr + DICT_HDR_FIELDS, MLOG_4BYTES, &mtr), FALSE);
    ut_a(err == DB_SUCCESS);

    mutex_enter(&dict_sys->mutex);
    dict_load_sys_table(dict_sys->sys_tables);
    dict_load_sys_table(dict_sys->sys_columns);
    dict_load_sys_table(dict_sys->sys_indexes);
    dict_load_sys_table(dict_sys->sys_fields);
    mutex_exit(&dict_sys->mutex);
  }

  mtr_commit(&mtr);

  /*-------------------------*/

  /* Initialize the insert buffer table, table buffer and indexes */

  ibuf_init_at_db_start();

  if (srv_force_recovery != SRV_FORCE_NO_LOG_REDO && srv_read_only_mode &&
      !ibuf_is_empty()) {
    ib::error(ER_IB_MSG_161) << "Change buffer must be empty when"
                                " --innodb-read-only is set!";

    err = DB_ERROR;
  }

  return (err);
}

```
写日志到硬盘：
```
void log_buffer_flush_to_disk(log_t &log, bool sync) {
  ut_a(!srv_read_only_mode);
  ut_a(!recv_recovery_is_on());

  const lsn_t lsn = log_get_lsn(log);

  log_write_up_to(log, lsn, sync);
}

```
后面还有很多的辅助性工作，如临时空间等的创建，recovery（recv_recovery_from_checkpoint_finish一个 checkpoint 位置完成 recovery 操作）的各种处理，等等。这个需要一个更详细的过程来分析，回头再分析和个细节时再一一展开。

## innobase_start_or_create_for_mysql() 函数解析如下：
```
dberr_t
innobase_start_or_create_for_mysql(void)
/*====================================*/
{
    bool        create_new_db = false;
    lsn_t       flushed_lsn;
    ulint       sum_of_data_file_sizes;
    ulint       tablespace_size_in_header;
    dberr_t     err;
    ulint       srv_n_log_files_found = srv_n_log_files;
    mtr_t       mtr;
    purge_pq_t* purge_queue;
    char        logfilename[10000];
    char*       logfile0    = NULL;
    size_t      dirnamelen;
    unsigned    i = 0;
 
    /* Reset the start state.
    重置 start state.
    */
    srv_start_state = SRV_START_STATE_NONE;
    // SRV_FORCE_NO_LOG_REDO: 不做 redo log 的前滚操作
    if (srv_force_recovery == SRV_FORCE_NO_LOG_REDO) {
        srv_read_only_mode = true;
    }
    // high_level_read_only:
    high_level_read_only = srv_read_only_mode
        || srv_force_recovery > SRV_FORCE_NO_TRX_UNDO;
    // 如果处于 read_only mode, 那么除了内部表之外，没有其他写操作，关闭两次写机制。
    if (srv_read_only_mode) {
        ib::info() << "Started in read only mode";
 
        /* There is no write except to intrinsic table and so turn-off
        doublewrite mechanism completely. */
        srv_use_doublewrite_buf = FALSE;
    }
 
 
#ifdef _WIN32
    srv_use_native_aio = TRUE;
 
#elif defined(LINUX_NATIVE_AIO)
 
    if (srv_use_native_aio) {
        ib::info() << "Using Linux native AIO";
    }
#else
    /* Currently native AIO is supported only on windows and linux
    and that also when the support is compiled in. In all other
    cases, we ignore the setting of innodb_use_native_aio. */
    srv_use_native_aio = FALSE;
#endif /* _WIN32 */
 
    /* Register performance schema stages before any real work has been
    started which may need to be instrumented. */
    mysql_stage_register("innodb", srv_stages, UT_ARR_SIZE(srv_stages));
    /**
    处理参数 innodb_flush_method
    通常情况下，innodb_flush_method 设置为 O_DIRECT | O_DIRECT_NO_FSYNC;
    */
    if (srv_file_flush_method_str == NULL) {
        /* These are the default options */
#ifndef _WIN32
        srv_unix_file_flush_method = SRV_UNIX_FSYNC;
    } else if (0 == ut_strcmp(srv_file_flush_method_str, "fsync")) {
        srv_unix_file_flush_method = SRV_UNIX_FSYNC;
 
    } else if (0 == ut_strcmp(srv_file_flush_method_str, "O_DSYNC")) {
        srv_unix_file_flush_method = SRV_UNIX_O_DSYNC;
 
    } else if (0 == ut_strcmp(srv_file_flush_method_str, "O_DIRECT")) {
        srv_unix_file_flush_method = SRV_UNIX_O_DIRECT;
 
    } else if (0 == ut_strcmp(srv_file_flush_method_str, "O_DIRECT_NO_FSYNC")) {
        srv_unix_file_flush_method = SRV_UNIX_O_DIRECT_NO_FSYNC;
 
    } else if (0 == ut_strcmp(srv_file_flush_method_str, "littlesync")) {
        srv_unix_file_flush_method = SRV_UNIX_LITTLESYNC;
 
    } else if (0 == ut_strcmp(srv_file_flush_method_str, "nosync")) {
        srv_unix_file_flush_method = SRV_UNIX_NOSYNC;
#else
        srv_win_file_flush_method = SRV_WIN_IO_UNBUFFERED;
    } else if (0 == ut_strcmp(srv_file_flush_method_str, "normal")) {
        srv_win_file_flush_method = SRV_WIN_IO_NORMAL;
        srv_use_native_aio = FALSE;
 
    } else if (0 == ut_strcmp(srv_file_flush_method_str, "unbuffered")) {
        srv_win_file_flush_method = SRV_WIN_IO_UNBUFFERED;
        srv_use_native_aio = FALSE;
 
    } else if (0 == ut_strcmp(srv_file_flush_method_str,
                  "async_unbuffered")) {
        srv_win_file_flush_method = SRV_WIN_IO_UNBUFFERED;
#endif /* _WIN32 */
    } else {
        ib::error() << "Unrecognized value "
            << srv_file_flush_method_str
            << " for innodb_flush_method";
        return(srv_init_abort(DB_ERROR));
    }
 
    /* Note that the call srv_boot() also changes the values of
    some variables to the units used by InnoDB internally */
 
    /* Set the maximum number of threads which can wait for a semaphore
    inside InnoDB: this is the 'sync wait array' size, as well as the
    maximum number of threads that can wait in the 'srv_conc array' for
    their time to enter InnoDB.
    设置 Innodb 内部可能等待信号量的最大线程数量: 这是 sync wait array 的大小, 以及
    在 srv_conc 数组中等待进入 Innodb的最大线程数。
    */
 
    srv_max_n_threads = 1   /* io_ibuf_thread */
                + 1 /* io_log_thread */
                + 1 /* lock_wait_timeout_thread */
                + 1 /* srv_error_monitor_thread */
                + 1 /* srv_monitor_thread */
                + 1 /* srv_master_thread */
                + 1 /* srv_purge_coordinator_thread */
                + 1 /* buf_dump_thread */
                + 1 /* dict_stats_thread */
                + 1 /* fts_optimize_thread */
                + 1 /* recv_writer_thread */
                + 1 /* trx_rollback_or_clean_all_recovered */
                + 128 /* added as margin, for use of
                  InnoDB Memcached etc. */
                + max_connections
                + srv_n_read_io_threads
                + srv_n_write_io_threads
                + srv_n_purge_threads
                + srv_n_page_cleaners
                /* FTS Parallel Sort */
                + fts_sort_pll_degree * FTS_NUM_AUX_INDEX
                  * max_connections;
    /**
        重置 innodb_buffer_pool_instances
    */
    if (srv_buf_pool_size >= BUF_POOL_SIZE_THRESHOLD) {
 
        if (srv_buf_pool_instances == srv_buf_pool_instances_default) {
#if defined(_WIN32) && !defined(_WIN64)
            /* Do not allocate too large of a buffer pool on
            Windows 32-bit systems, which can have trouble
            allocating larger single contiguous memory blocks. */
            srv_buf_pool_instances = ut_min(
                static_cast<ulong>(MAX_BUFFER_POOLS),
                static_cast<ulong>(srv_buf_pool_size
                           / (128 * 1024 * 1024)));
#else /* defined(_WIN32) && !defined(_WIN64) */
            /* Default to 8 instances when size > 1GB. */
            srv_buf_pool_instances = 8;
#endif /* defined(_WIN32) && !defined(_WIN64) */
        }
    } else {
        /* If buffer pool is less than 1 GiB, assume fewer
        threads. Also use only one buffer pool instance. */
        if (srv_buf_pool_instances != srv_buf_pool_instances_default
            && srv_buf_pool_instances != 1) {
            /* We can't distinguish whether the user has explicitly
            started mysqld with --innodb-buffer-pool-instances=0,
            (srv_buf_pool_instances_default is 0) or has not
            specified that option at all. Thus we have the
            limitation that if the user started with =0, we
            will not emit a warning here, but we should actually
            do so. */
            ib::info()
                << "Adjusting innodb_buffer_pool_instances"
                " from " << srv_buf_pool_instances << " to 1"
                " since innodb_buffer_pool_size is less than "
                << BUF_POOL_SIZE_THRESHOLD / (1024 * 1024)
                << " MiB";
        }
 
        srv_buf_pool_instances = 1;
    }
    // 调整 srv_buf_pool_chunk_unit 大小。
    if (srv_buf_pool_chunk_unit * srv_buf_pool_instances
        > srv_buf_pool_size) {
        /* Size unit of buffer pool is larger than srv_buf_pool_size.
        adjust srv_buf_pool_chunk_unit for srv_buf_pool_size. */
        srv_buf_pool_chunk_unit
            = static_cast<ulong>(srv_buf_pool_size)
              / srv_buf_pool_instances;
        if (srv_buf_pool_size % srv_buf_pool_instances != 0) {
            ++srv_buf_pool_chunk_unit;
        }
    }
    // 基于 srv_buf_pool_chunk_unit 对齐 srv_buf_pool_size
    srv_buf_pool_size = buf_pool_size_align(srv_buf_pool_size);
    // 根据 srv_buf_pool_instances 重置 innodb_page_cleaners
    if (srv_n_page_cleaners > srv_buf_pool_instances) {
        /* limit of page_cleaner parallelizability
        is number of buffer pool instances. */
        srv_n_page_cleaners = srv_buf_pool_instances;
    }
    /**
    启动innodb server, 进行相关参数和组件的初始化。
    */
    srv_boot();
 
    ib::info() << (ut_crc32_sse2_enabled ? "Using" : "Not using")
        << " CPU crc32 instructions";
    // innodb monitor 相关
    if (!srv_read_only_mode) {
 
        mutex_create(LATCH_ID_SRV_MONITOR_FILE,
                 &srv_monitor_file_mutex);
 
        if (srv_innodb_status) {
 
            srv_monitor_file_name = static_cast<char*>(
                ut_malloc_nokey(
                    strlen(fil_path_to_mysql_datadir)
                    + 20 + sizeof "/innodb_status."));
 
            sprintf(srv_monitor_file_name,
                "%s/innodb_status." ULINTPF,
                fil_path_to_mysql_datadir,
                os_proc_get_number());
 
            srv_monitor_file = fopen(srv_monitor_file_name, "w+");
 
            if (!srv_monitor_file) {
                ib::error() << "Unable to create "
                    << srv_monitor_file_name << ": "
                    << strerror(errno);
                return(srv_init_abort(DB_ERROR));
            }
        } else {
 
            srv_monitor_file_name = NULL;
            srv_monitor_file = os_file_create_tmpfile(NULL);
 
            if (!srv_monitor_file) {
                return(srv_init_abort(DB_ERROR));
            }
        }
 
        mutex_create(LATCH_ID_SRV_DICT_TMPFILE,
                 &srv_dict_tmpfile_mutex);
 
        srv_dict_tmpfile = os_file_create_tmpfile(NULL);
 
        if (!srv_dict_tmpfile) {
            return(srv_init_abort(DB_ERROR));
        }
 
        mutex_create(LATCH_ID_SRV_MISC_TMPFILE,
                 &srv_misc_tmpfile_mutex);
 
        srv_misc_tmpfile = os_file_create_tmpfile(NULL);
 
        if (!srv_misc_tmpfile) {
            return(srv_init_abort(DB_ERROR));
        }
    }
    /**
    file_io_threads
    */
    // innodb_read_io_threads & innodb_write_io_threads
    srv_n_file_io_threads = srv_n_read_io_threads;
 
    srv_n_file_io_threads += srv_n_write_io_threads;
    // 非 read only, 添加 log & ibuf io thread
    if (!srv_read_only_mode) {
        /* Add the log and ibuf IO threads. */
        srv_n_file_io_threads += 2;
    } else {
        ib::info() << "Disabling background log and ibuf IO write"
            << " threads.";
    }
 
    ut_a(srv_n_file_io_threads <= SRV_MAX_N_IO_THREADS);
    // 初始化异步IO子系统。
    if (!os_aio_init(srv_n_read_io_threads,
             srv_n_write_io_threads,
             SRV_MAX_N_PENDING_SYNC_IOS)) {
 
        ib::error() << "Cannot initialize AIO sub-system";
 
        return(srv_init_abort(DB_ERROR));
    }
    // 初始化各表空间的内存cache
    fil_init(srv_file_per_table ? 50000 : 5000, srv_max_n_open_files);
 
    double  size;
    char    unit;
    // innodb_buffer_pool_size 和 chunk_size
    if (srv_buf_pool_size >= 1024 * 1024 * 1024) {
        size = ((double) srv_buf_pool_size) / (1024 * 1024 * 1024);
        unit = 'G';
    } else {
        size = ((double) srv_buf_pool_size) / (1024 * 1024);
        unit = 'M';
    }
 
    double  chunk_size;
    char    chunk_unit;
 
    if (srv_buf_pool_chunk_unit >= 1024 * 1024 * 1024) {
        chunk_size = srv_buf_pool_chunk_unit / 1024.0 / 1024 / 1024;
        chunk_unit = 'G';
    } else {
        chunk_size = srv_buf_pool_chunk_unit / 1024.0 / 1024;
        chunk_unit = 'M';
    }
 
    ib::info() << "Initializing buffer pool, total size = "
        << size << unit << ", instances = " << srv_buf_pool_instances
        << ", chunk size = " << chunk_size << chunk_unit;
    // 创建 innodb_buffer_pool, 当没有足够的内存时会报错
    err = buf_pool_init(srv_buf_pool_size, srv_buf_pool_instances);
 
    if (err != DB_SUCCESS) {
        ib::error() << "Cannot allocate memory for the buffer pool";
 
        return(srv_init_abort(DB_ERROR));
    }
 
    ib::info() << "Completed initialization of buffer pool";
 
    // 初始化 fsp 系统 & redo log
    fsp_init();
    log_init();
    // 创建 recovery 系统, 针对一个 recovery 操作初始化 recovery 系统
    recv_sys_create();
    recv_sys_init(buf_pool_get_curr_size());
    // 数据库启动时创建锁系统
    lock_sys_create(srv_lock_table_size);
    // start lock-timeout thread
    srv_start_state_set(SRV_START_STATE_LOCK_SYS);
 
    /* Create i/o-handler threads:
    创建 io 线程
    */
    for (ulint t = 0; t < srv_n_file_io_threads; ++t) {
 
        n[t] = t;
 
        os_thread_create(io_handler_thread, n + t, thread_ids + t);
    }
 
    /* Even in read-only mode there could be flush job generated by
    intrinsic table operations.
    初始化 page_cleaner
    */
    buf_flush_page_cleaner_init();
    // 创建 buf_flush_page_cleaner_coordinator 线程
    os_thread_create(buf_flush_page_cleaner_coordinator,
             NULL, NULL);
    // 创建 buf_flush_page_cleaner_worker 线程
    for (i = 1; i < srv_n_page_cleaners; ++i) {
        os_thread_create(buf_flush_page_cleaner_worker,
                 NULL, NULL);
    }
 
    /* Make sure page cleaner is active.
    page_cleaner处于活跃状态
    */
    while (!buf_page_cleaner_is_active) {
        os_thread_sleep(10000);
    }
    // start io-thread
    srv_start_state_set(SRV_START_STATE_IO);
 
    // 对目录进行规范
    os_normalize_path(srv_data_home);
 
    /* Check if the data files exist or not.
    检查数据文件是否存在, ibdata1 ibdata2 等等，判断是否需要创建新的数据库
    */
    err = srv_sys_space.check_file_spec(
        &create_new_db, MIN_EXPECTED_TABLESPACE_SIZE);
 
    if (err != DB_SUCCESS) {
        return(srv_init_abort(DB_ERROR));
    }
    // 不是创建新的db, 则需要回滚未完成的事务
    srv_startup_is_before_trx_rollback_phase = !create_new_db;
 
    /* Check if undo tablespaces and redo log files exist before creating
    a new system tablespace
    检查是否存在 redo log file 和 undo 表空间
    */
    if (create_new_db) {
        err = srv_check_undo_redo_logs_exists();
        if (err != DB_SUCCESS) {
            return(srv_init_abort(DB_ERROR));
        }
        recv_sys_debug_free();
    }
 
    /* Open or create the data files.
    打开或者创建数据文件。
    */
    ulint   sum_of_new_sizes;
    // 打开或者创建数据文件[ibdata文件]，并从 ibdata1 文件中读取 flushed_lsn
    err = srv_sys_space.open_or_create(
        false, create_new_db, &sum_of_new_sizes, &flushed_lsn);
 
    switch (err) {
    case DB_SUCCESS:
        break;
    case DB_CANNOT_OPEN_FILE:
        ib::error()
            << "Could not open or create the system tablespace. If"
            " you tried to add new data files to the system"
            " tablespace, and it failed here, you should now"
            " edit innodb_data_file_path in my.cnf back to what"
            " it was, and remove the new ibdata files InnoDB"
            " created in this failed attempt. InnoDB only wrote"
            " those files full of zeros, but did not yet use"
            " them in any way. But be careful: do not remove"
            " old data files which contain your precious data!";
        /* fall through */
    default:
        /* Other errors might come from Datafile::validate_first_page() */
        return(srv_init_abort(err));
    }
 
    dirnamelen = strlen(srv_log_group_home_dir);
    ut_a(dirnamelen < (sizeof logfilename) - 10 - sizeof "ib_logfile");
    memcpy(logfilename, srv_log_group_home_dir, dirnamelen);
 
    /* Add a path separator if needed. */
    if (dirnamelen && logfilename[dirnamelen - 1] != OS_PATH_SEPARATOR) {
        logfilename[dirnamelen++] = OS_PATH_SEPARATOR;
    }
 
    srv_log_file_size_requested = srv_log_file_size;
     
    if (create_new_db) {
        /**
            如果是 create new db
        */
        // 从所有缓冲池实例的 flush list 的末尾同步的 flush dirty blocks.
        buf_flush_sync_all_buf_pools();
        // 获取 current lsn
        flushed_lsn = log_get_lsn();
        // 创建 redo log file
        err = create_log_files(
            logfilename, dirnamelen, flushed_lsn, logfile0);
 
        if (err != DB_SUCCESS) {
            return(srv_init_abort(err));
        }
    } else {
        // not create new db
        for (i = 0; i < SRV_N_LOG_FILES_MAX; i++) {
            os_offset_t size;
            os_file_stat_t  stat_info;
 
            sprintf(logfilename + dirnamelen,
                "ib_logfile%u", i);
            // 获取 logfile 文件状态
            err = os_file_get_status(
                logfilename, &stat_info, false,
                srv_read_only_mode);
 
            if (err == DB_NOT_FOUND) {
                if (i == 0) {
                    if (flushed_lsn
                        < static_cast<lsn_t>(1000)) {
                        ib::error()
                            << "Cannot create"
                            " log files because"
                            " data files are"
                            " corrupt or the"
                            " database was not"
                            " shut down cleanly"
                            " after creating"
                            " the data files.";
                        return(srv_init_abort(
                            DB_ERROR));
                    }
 
                    err = create_log_files(
                        logfilename, dirnamelen,
                        flushed_lsn, logfile0);
 
                    if (err != DB_SUCCESS) {
                        return(srv_init_abort(err));
                    }
 
                    create_log_files_rename(
                        logfilename, dirnamelen,
                        flushed_lsn, logfile0);
 
                    /* Suppress the message about
                    crash recovery. */
                    flushed_lsn = log_get_lsn();
                    goto files_checked;
                } else if (i < 2) {
                    /* must have at least 2 log files */
                    ib::error() << "Only one log file"
                        " found.";
                    return(srv_init_abort(err));
                }
 
                /* opened all files */
                break;
            }
            // 检查 log file mode
            if (!srv_file_check_mode(logfilename)) {
                return(srv_init_abort(DB_ERROR));
            }
            // 打开 redo log file
            err = open_log_file(&files[i], logfilename, &size);
 
            if (err != DB_SUCCESS) {
                return(srv_init_abort(err));
            }
 
            ut_a(size != (os_offset_t) -1);
             
            if (size & ((1 << UNIV_PAGE_SIZE_SHIFT) - 1)) {
 
                ib::error() << "Log file " << logfilename
                    << " size " << size << " is not a"
                    " multiple of innodb_page_size";
                return(srv_init_abort(DB_ERROR));
            }
 
            size >>= UNIV_PAGE_SIZE_SHIFT;
 
            if (i == 0) {
                srv_log_file_size = size;
            } else if (size != srv_log_file_size) {
 
                ib::error() << "Log file " << logfilename
                    << " is of different size "
                    << (size << UNIV_PAGE_SIZE_SHIFT)
                    << " bytes than other log files "
                    << (srv_log_file_size
                        << UNIV_PAGE_SIZE_SHIFT)
                    << " bytes!";
                return(srv_init_abort(DB_ERROR));
            }
        }
        // logfile的数量
        srv_n_log_files_found = i;
 
        /* Create the in-memory file space objects.
            创建 log file 内存中的文件空间对象。
        */
         
        sprintf(logfilename + dirnamelen, "ib_logfile%u", 0);
 
        /* Disable the doublewrite buffer for log files.
            log file 禁用两次写缓冲区。
        */
        fil_space_t*    log_space = fil_space_create(
            "innodb_redo_log",
            SRV_LOG_SPACE_FIRST_ID,
            fsp_flags_set_page_size(0, univ_page_size),
            FIL_TYPE_LOG);
 
        ut_a(fil_validate());
        ut_a(log_space);
 
        /* srv_log_file_size is measured in pages; if page size is 16KB,
        then we have a limit of 64TB on 32 bit systems */
        ut_a(srv_log_file_size <= ULINT_MAX);
        // 添加 log file文件到 log file space 中
        for (unsigned j = 0; j < i; j++) {
            sprintf(logfilename + dirnamelen, "ib_logfile%u", j);
 
            if (!fil_node_create(logfilename,
                         (ulint) srv_log_file_size,
                         log_space, false, false)) {
                return(srv_init_abort(DB_ERROR));
            }
        }
        // 初始化 redo log group
        if (!log_group_init(0, i, srv_log_file_size * UNIV_PAGE_SIZE,
                    SRV_LOG_SPACE_FIRST_ID)) {
            return(srv_init_abort(DB_ERROR));
        }
    }
 
files_checked:
    /* Open all log files and data files in the system
    tablespace: we keep them open until database
    shutdown */
    // 打开所有的日志文件和系统表数据文件。
    fil_open_log_and_system_tablespace_files();
    // 打开 undo 表空间, 在找到并打开所有的 undo 文件之后, 将他们全部加入文件管理系统
    err = srv_undo_tablespaces_init(
        create_new_db,
        srv_undo_tablespaces,
        &srv_undo_tablespaces_open);
 
    /* If the force recovery is set very high then we carry on regardless
    of all errors. Basically this is fingers crossed mode.
    接下来涉及到数据的恢复。
    */
 
    if (err != DB_SUCCESS
        && srv_force_recovery < SRV_FORCE_NO_UNDO_LOG_SCAN) {
 
        return(srv_init_abort(err));
    }
 
    /* Initialize objects used by dict stats gathering thread, which
    can also be used by recovery if it tries to drop some table */
    if (!srv_read_only_mode) {
        dict_stats_thread_init();
    }
    // 初始化 file_format_max变量。
    trx_sys_file_format_init();
    // 创建 trx_sys instance 并初始化 purge_queue 和 mutex
    trx_sys_create();
 
    if (create_new_db) {
 
        ut_a(!srv_read_only_mode);
 
        mtr_start(&mtr);
 
        bool ret = fsp_header_init(0, sum_of_new_sizes, &mtr);
 
        mtr_commit(&mtr);
 
        if (!ret) {
            return(srv_init_abort(DB_ERROR));
        }
 
        /* To maintain backward compatibility we create only
        the first rollback segment before the double write buffer.
        All the remaining rollback segments will be created later,
        after the double write buffer has been created. */
        trx_sys_create_sys_pages();
 
        purge_queue = trx_sys_init_at_db_start();
 
        DBUG_EXECUTE_IF("check_no_undo",
                ut_ad(purge_queue->empty());
                );
 
        /* The purge system needs to create the purge view and
        therefore requires that the trx_sys is inited. */
 
        trx_purge_sys_create(srv_n_purge_threads, purge_queue);
 
        err = dict_create();
 
        if (err != DB_SUCCESS) {
            return(srv_init_abort(err));
        }
 
        buf_flush_sync_all_buf_pools();
 
        flushed_lsn = log_get_lsn();
 
        fil_write_flushed_lsn(flushed_lsn);
 
        create_log_files_rename(
            logfilename, dirnamelen, flushed_lsn, logfile0);
 
    } else {
 
        /* Check if we support the max format that is stamped
        on the system tablespace.
        Note:  We are NOT allowed to make any modifications to
        the TRX_SYS_PAGE_NO page before recovery  because this
        page also contains the max_trx_id etc. important system
        variables that are required for recovery.  We need to
        ensure that we return the system to a state where normal
        recovery is guaranteed to work. We do this by
        invalidating the buffer cache, this will force the
        reread of the page and restoration to its last known
        consistent state, this is REQUIRED for the recovery
        process to work. */
        // 检查是否支持系统表空间上的 max 格式。
        err = trx_sys_file_format_max_check(
            srv_max_file_format_at_startup);
 
        if (err != DB_SUCCESS) {
            return(srv_init_abort(err));
        }
 
        /* Invalidate the buffer pool to ensure that we reread
        the page that we read above, during recovery.
        Note that this is not as heavy weight as it seems. At
        this point there will be only ONE page in the buf_LRU
        and there must be no page in the buf_flush list.
        使整个缓冲池无效, 来确保在 recovery的过程中我们重启读取之前读取的页。
        这是一个很轻量级的操作, 此时再 LRU 列表中只有一个数据页, 在 flush 列表中没有任何数据页。
        */
        buf_pool_invalidate();
 
        /* Scan and locate truncate log files. Parsed located files
        and add table to truncate information to central vector for
        truncate fix-up action post recovery.
        扫描并定位 truncate log file, 解析truncate log file.
        */
        err = TruncateLogParser::scan_and_parse(srv_log_group_home_dir);
        if (err != DB_SUCCESS) {
 
            return(srv_init_abort(DB_ERROR));
        }
 
        /* We always try to do a recovery, even if the database had
        been shut down normally: this is the normal startup path
        通常情况下, 需要做一个 recovery 操作, 即使 database 正常关闭。
        */
        /**
        从 checkpoint  flushed_lsn 位置开始恢复。
        1. 初始化红黑树, 以便在恢复的过程中快速插入 flush 列表。
        2. 在 log groups 中查找 latest checkpoint
        3. 读取 latest checkpoint 所在的 redo log 页到 log_sys->checkpoint_buf中
        4. 获取 checkpoint_lsn 和 checkpoint_no
        5. 从 checkpoing_lsn 读取 redo log 到 hash 表中。
        6. 检查 crash recovery 所需的表空间, 处理并删除double write buf 中的数据页, 这里会检查double write buf 中页对应的真实数据页的
        完整性, 如果有问题, 则使用 double write buf 中页进行恢复。同时, 生成后台线程 recv_writer_thread 以清理缓冲池中的脏页。
        7. 将日志段从最新的日志组复制到其他组, 我们目前只有一个日志组。
        */
        err = recv_recovery_from_checkpoint_start(flushed_lsn);
　　　　　　　　　 // 清除 double write buf 中的数据页
        recv_sys->dblwr.pages.clear();
        // 初始化 数据字典系统，并初始化change buffer
        if (err == DB_SUCCESS) {
            /* Initialize the change buffer. */
            err = dict_boot();
        }
 
        if (err != DB_SUCCESS) {
 
            /* A tablespace was not found during recovery. The
            user must force recovery. */
 
            if (err == DB_TABLESPACE_NOT_FOUND) {
 
                srv_fatal_error();
 
                ut_error;
            }
 
            return(srv_init_abort(DB_ERROR));
        }
        // 创建并初始化事务系统。
        purge_queue = trx_sys_init_at_db_start();
 
        if (srv_force_recovery < SRV_FORCE_NO_LOG_REDO) {
            /* Apply the hashed log records to the
            respective file pages, for the last batch of
            recv_group_scan_log_recs(). */
            // 应用 redo log, 完成 crash recovery 操作.
            recv_apply_hashed_log_recs(TRUE);
            DBUG_PRINT("ib_log", ("apply completed"));
 
            if (recv_needed_recovery) {
                /// Last MySQL binlog file position 0 894036112, file name mysql-bin.002128
                trx_sys_print_mysql_binlog_offset();
            }
        }
 
        if (recv_sys->found_corrupt_log) {
            ib::warn()
                << "The log file may have been corrupt and it"
                " is possible that the log scan or parsing"
                " did not proceed far enough in recovery."
                " Please run CHECK TABLE on your InnoDB tables"
                " to check that they are ok!"
                " It may be safest to recover your"
                " InnoDB database from a backup!";
        }
 
        /* The purge system needs to create the purge view and
        therefore requires that the trx_sys is inited. */
        // 创建 trx_purge_sys
        trx_purge_sys_create(srv_n_purge_threads, purge_queue);
 
        /* recv_recovery_from_checkpoint_finish needs trx lists which
        are initialized in trx_sys_init_at_db_start(). */
        /*
            完成 recovery 操作。
            1. 确保 recv_writer 线程已完成
            2. 等待 flush 操作完成, flush脏页操作已经完成
            3. 等待 recv_writer 线程终止
            4. 释放 flush 红黑树
            5. <em>回滚所有的数据字典表的事务，以便数据字典表没有被锁定。数据字典 latch 应保证一次只有一个数据字典事务处于活跃状态。</em>
        */
        recv_recovery_from_checkpoint_finish();
 
        /* Fix-up truncate of tables in the system tablespace
        if server crashed while truncate was active. The non-
        system tables are done after tablespace discovery. Do
        this now because this procedure assumes that no pages
        have changed since redo recovery.  Tablespace discovery
        can do updates to pages in the system tablespace.*/
        // 修复系统表空间中的表
        err = truncate_t::fixup_tables_in_system_tablespace();
 
        if (srv_force_recovery < SRV_FORCE_NO_IBUF_MERGE) {
            /* Open or Create SYS_TABLESPACES and SYS_DATAFILES
            so that tablespace names and other metadata can be
            found. */
            srv_sys_tablespaces_open = true;
            // 检查数据字典中每个表的表空间
            err = dict_create_or_check_sys_tablespace();
            if (err != DB_SUCCESS) {
                return(srv_init_abort(err));
            }
 
            /* The following call is necessary for the insert
            buffer to work with multiple tablespaces. We must
            know the mapping between space id's and .ibd file
            names.
 
            In a crash recovery, we check that the info in data
            dictionary is consistent with what we already know
            about space id's from the calls to fil_ibd_load().
 
            In a normal startup, we create the space objects for
            every table in the InnoDB data dictionary that has
            an .ibd file.
 
            We also determine the maximum tablespace id used.
 
            The 'validate' flag indicates that when a tablespace
            is opened, we also read the header page and validate
            the contents to the data dictionary. This is time
            consuming, especially for databases with lots of ibd
            files.  So only do it after a crash and not forcing
            recovery.  Open rw transactions at this point is not
            a good reason to validate. */
            bool validate = recv_needed_recovery
                && srv_force_recovery == 0;
 
            dict_check_tablespaces_and_store_max_id(validate);
        }
 
        /* Rotate the encryption key for recovery. It's because
        server could crash in middle of key rotation. Some tablespace
        didn't complete key rotation. Here, we will resume the
        rotation. */
        if (!srv_read_only_mode
            && srv_force_recovery < SRV_FORCE_NO_LOG_REDO) {
            fil_encryption_rotate();
        }
 
 
        /* Fix-up truncate of table if server crashed while truncate
        was active. */
        err = truncate_t::fixup_tables_in_non_system_tablespace();
 
        if (err != DB_SUCCESS) {
            return(srv_init_abort(err));
        }
 
        if (!srv_force_recovery
            && !recv_sys->found_corrupt_log
            && (srv_log_file_size_requested != srv_log_file_size
            || srv_n_log_files_found != srv_n_log_files)) {
 
            /* Prepare to replace the redo log files. */
 
            if (srv_read_only_mode) {
                ib::error() << "Cannot resize log files"
                    " in read-only mode.";
                return(srv_init_abort(DB_READ_ONLY));
            }
 
            /* Prepare to delete the old redo log files */
            flushed_lsn = srv_prepare_to_delete_redo_log_files(i);
 
            /* Prohibit redo log writes from any other
            threads until creating a log checkpoint at the
            end of create_log_files(). */
            ut_d(recv_no_log_write = true);
            ut_ad(!buf_pool_check_no_pending_io());
 
            RECOVERY_CRASH(3);
 
            /* Stamp the LSN to the data files. */
            fil_write_flushed_lsn(flushed_lsn);
 
            RECOVERY_CRASH(4);
 
            /* Close and free the redo log files, so that
            we can replace them. */
            fil_close_log_files(true);
 
            RECOVERY_CRASH(5);
 
            /* Free the old log file space. */
            log_group_close_all();
 
            ib::warn() << "Starting to delete and rewrite log"
                " files.";
 
            srv_log_file_size = srv_log_file_size_requested;
 
            err = create_log_files(
                logfilename, dirnamelen, flushed_lsn,
                logfile0);
 
            if (err != DB_SUCCESS) {
                return(srv_init_abort(err));
            }
 
            create_log_files_rename(
                logfilename, dirnamelen, flushed_lsn,
                logfile0);
        }
        // 回滚未提交的不完整的事务, 这是在一个后台线程中进行的。
        recv_recovery_rollback_active();
 
        /* It is possible that file_format tag has never
        been set. In this case we initialize it to minimum
        value.  Important to note that we can do it ONLY after
        we have finished the recovery process so that the
        image of TRX_SYS_PAGE_NO is not stale. */
        trx_sys_file_format_tag_init();
    }
 
    if (!create_new_db) {
        /* Check and reset any no-redo rseg slot on disk used by
        pre-5.7.2 redo resg with no data to purge. */
        trx_rseg_reset_pending();
    }
 
    if (!create_new_db && sum_of_new_sizes > 0) {
        /* New data file(s) were added */
        mtr_start(&mtr);
 
        fsp_header_inc_size(0, sum_of_new_sizes, &mtr);
 
        mtr_commit(&mtr);
 
        /* Immediately write the log record about increased tablespace
        size to disk, so that it is durable even if mysqld would crash
        quickly */
 
        log_buffer_flush_to_disk();
    }
 
    /* Open temp-tablespace and keep it open until shutdown. */
    // 打开临时表空间
    err = srv_open_tmp_tablespace(create_new_db, &srv_tmp_space);
 
    if (err != DB_SUCCESS) {
        return(srv_init_abort(err));
    }
 
    /* Create the doublewrite buffer to a new tablespace */
    if (buf_dblwr == NULL && !buf_dblwr_create()) {
        return(srv_init_abort(DB_ERROR));
    }
 
    /* Here the double write buffer has already been created and so
    any new rollback segments will be allocated after the double
    write buffer. The default segment should already exist.
    We create the new segments only if it's a new database or
    the database was shutdown cleanly. */
 
    /* Note: When creating the extra rollback segments during an upgrade
    we violate the latching order, even if the change buffer is empty.
    We make an exception in sync0sync.cc and check srv_is_being_started
    for that violation. It cannot create a deadlock because we are still
    running in single threaded mode essentially. Only the IO threads
    should be running at this stage. */
 
    /* Deprecate innodb_undo_logs.  But still use it if it is set to
    non-default and innodb_rollback_segments is default. */
    ut_a(srv_rollback_segments > 0);
    ut_a(srv_rollback_segments <= TRX_SYS_N_RSEGS);
    ut_a(srv_undo_logs > 0);
    ut_a(srv_undo_logs <= TRX_SYS_N_RSEGS);
    if (srv_undo_logs < TRX_SYS_N_RSEGS) {
        ib::warn() << deprecated_undo_logs;
        if (srv_rollback_segments == TRX_SYS_N_RSEGS) {
            srv_rollback_segments = srv_undo_logs;
        }
    }
 
    /* The number of rsegs that exist in InnoDB is given by status
    variable srv_available_undo_logs. The number of rsegs to use can
    be set using the dynamic global variable srv_rollback_segments. */
    // 创建回滚段
    srv_available_undo_logs = trx_sys_create_rsegs(
        srv_undo_tablespaces, srv_rollback_segments, srv_tmp_undo_logs);
 
    if (srv_available_undo_logs == ULINT_UNDEFINED) {
        /* Can only happen if server is read only. */
        ut_a(srv_read_only_mode);
        srv_rollback_segments = ULONG_UNDEFINED;
    } else if (srv_available_undo_logs < srv_rollback_segments
           && !srv_force_recovery && !recv_needed_recovery) {
        ib::error() << "System or UNDO tablespace is running of out"
                << " of space";
        /* Should due to out of file space. */
        return(srv_init_abort(DB_ERROR));
    }
 
    srv_startup_is_before_trx_rollback_phase = false;
 
    if (!srv_read_only_mode) {
        /* Create the thread which watches the timeouts
        for lock waits
            创建 lock_wait_timeout_thread watch 线程
        */
        os_thread_create(
            lock_wait_timeout_thread,
            NULL, thread_ids + 2 + SRV_MAX_N_IO_THREADS);
 
        /* Create the thread which warns of long semaphore waits
            创建 srv_error_monitor_thread 线程
        */
        os_thread_create(
            srv_error_monitor_thread,
            NULL, thread_ids + 3 + SRV_MAX_N_IO_THREADS);
 
        /* Create the thread which prints InnoDB monitor info
            创建 Innodb monitor info print 线程
        */
        os_thread_create(
            srv_monitor_thread,
            NULL, thread_ids + 4 + SRV_MAX_N_IO_THREADS);
 
        srv_start_state_set(SRV_START_STATE_MONITOR);
    }
 
    /* Create the SYS_FOREIGN and SYS_FOREIGN_COLS system tables */
    err = dict_create_or_check_foreign_constraint_tables();
    if (err != DB_SUCCESS) {
        return(srv_init_abort(err));
    }
 
    /* Create the SYS_TABLESPACES system table */
    err = dict_create_or_check_sys_tablespace();
    if (err != DB_SUCCESS) {
        return(srv_init_abort(err));
    }
    srv_sys_tablespaces_open = true;
 
    /* Create the SYS_VIRTUAL system table */
    err = dict_create_or_check_sys_virtual();
    if (err != DB_SUCCESS) {
        return(srv_init_abort(err));
    }
 
    srv_is_being_started = false;
 
    ut_a(trx_purge_state() == PURGE_STATE_INIT);
 
    /* Create the master thread which does purge and other utility
    operations
        创建 master 线程
    */
 
    if (!srv_read_only_mode) {
 
        os_thread_create(
            srv_master_thread,
            NULL, thread_ids + (1 + SRV_MAX_N_IO_THREADS));
 
        srv_start_state_set(SRV_START_STATE_MASTER);
    }
    // purge_coordinator 线程和 purge_worker 线程
    if (!srv_read_only_mode
        && srv_force_recovery < SRV_FORCE_NO_BACKGROUND) {
 
        os_thread_create(
            srv_purge_coordinator_thread,
            NULL, thread_ids + 5 + SRV_MAX_N_IO_THREADS);
 
        ut_a(UT_ARR_SIZE(thread_ids)
             > 5 + srv_n_purge_threads + SRV_MAX_N_IO_THREADS);
 
        /* We've already created the purge coordinator thread above. */
        for (i = 1; i < srv_n_purge_threads; ++i) {
            os_thread_create(
                srv_worker_thread, NULL,
                thread_ids + 5 + i + SRV_MAX_N_IO_THREADS);
        }
        // 等待 purge thread 启动
        srv_start_wait_for_purge_to_start();
 
        srv_start_state_set(SRV_START_STATE_PURGE);
    } else {
        purge_sys->state = PURGE_STATE_DISABLED;
    }
 
    /* wake main loop of page cleaner up
        唤醒 page cleaner 主循环
    */
    os_event_set(buf_flush_event);
 
    sum_of_data_file_sizes = srv_sys_space.get_sum_of_sizes();
    ut_a(sum_of_new_sizes != ULINT_UNDEFINED);
 
    tablespace_size_in_header = fsp_header_get_tablespace_size();
 
    if (!srv_read_only_mode
        && !srv_sys_space.can_auto_extend_last_file()
        && sum_of_data_file_sizes != tablespace_size_in_header) {
 
        ib::error() << "Tablespace size stored in header is "
            << tablespace_size_in_header << " pages, but the sum"
            " of data file sizes is " << sum_of_data_file_sizes
            << " pages";
 
        if (srv_force_recovery == 0
            && sum_of_data_file_sizes < tablespace_size_in_header) {
            /* This is a fatal error, the tail of a tablespace is
            missing */
 
            ib::error()
                << "Cannot start InnoDB."
                " The tail of the system tablespace is"
                " missing. Have you edited"
                " innodb_data_file_path in my.cnf in an"
                " inappropriate way, removing"
                " ibdata files from there?"
                " You can set innodb_force_recovery=1"
                " in my.cnf to force"
                " a startup if you are trying"
                " to recover a badly corrupt database.";
 
            return(srv_init_abort(DB_ERROR));
        }
    }
 
    if (!srv_read_only_mode
        && srv_sys_space.can_auto_extend_last_file()
        && sum_of_data_file_sizes < tablespace_size_in_header) {
 
        ib::error() << "Tablespace size stored in header is "
            << tablespace_size_in_header << " pages, but the sum"
            " of data file sizes is only "
            << sum_of_data_file_sizes << " pages";
 
        if (srv_force_recovery == 0) {
 
            ib::error()
                << "Cannot start InnoDB. The tail of"
                " the system tablespace is"
                " missing. Have you edited"
                " innodb_data_file_path in my.cnf in an"
                " InnoDB: inappropriate way, removing"
                " ibdata files from there?"
                " You can set innodb_force_recovery=1"
                " in my.cnf to force"
                " InnoDB: a startup if you are trying to"
                " recover a badly corrupt database.";
 
            return(srv_init_abort(DB_ERROR));
        }
    }
 
    if (srv_print_verbose_log) {
        ib::info() << INNODB_VERSION_STR
            << " started; log sequence number "
            << srv_start_lsn;
    }
 
    if (srv_force_recovery > 0) {
        ib::info() << "!!! innodb_force_recovery is set to "
            << srv_force_recovery << " !!!";
    }
 
    if (srv_force_recovery == 0) {
        /* In the insert buffer we may have even bigger tablespace
        id's, because we may have dropped those tablespaces, but
        insert buffer merge has not had time to clean the records from
        the ibuf tree. */
 
        ibuf_update_max_tablespace_id();
    }
 
    if (!srv_read_only_mode) {
        if (create_new_db) {
            srv_buffer_pool_load_at_startup = FALSE;
        }
 
        /* Create the buffer pool dump/load thread */
        os_thread_create(buf_dump_thread, NULL, NULL);
 
        /* Create the dict stats gathering thread */
        os_thread_create(dict_stats_thread, NULL, NULL);
 
        /* Create the thread that will optimize the FTS sub-system. */
        fts_optimize_init();
 
        srv_start_state_set(SRV_START_STATE_STAT);
    }
 
    /* Create the buffer pool resize thread */
    os_thread_create(buf_resize_thread, NULL, NULL);
 
    srv_was_started = TRUE;
    return(DB_SUCCESS);
}
```
## 总结
不同的版本的MySql，相关的细节还是有所不同的。引擎的数理启动过程，其实就一个初始化参数，配置参数，判断各种准备工作是否完成，然后启动相关的服务来进行引擎工作。你听这个名字，引擎，不就是发动机么，发动机转起来，才能让机器运动。