---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL事务系统MVCC
线程本地的核心结构体为trx_t，其主要保存着该会话事务相关的所有信息，当然也包括本节需要讲解的read_view（一致性视图结构体）；innodb事务核心系统trx_sys_t，其维护innodb的事务系统，其中包含本节主要分析的mvcc。

MVCC是保证多事务并发处理过程中，实现事物之间隔离的一种技术。本文主要分析innodb引擎实现MVCC（多版本控制）细节。

## Innodb MVCC原理
innodb存储引擎会对每行记录添加三个字段：

6字节的事务ID（TRX_ID ）
该字段主要标识修改该行的事务id。用于多版本并发控制的可见性判断

7字节的回滚指针（ROLL_PTR）
该字段是标识指向undo回滚段该行以前的版本，这样同一行数据就形成了一个链表形成，越老的数据越在链表的后面。如下图所示。

隐藏的ROW_ID （无主键时）

对于同一行数据（ROW_ID=1），第一次修改是事务id号为2的事务进行了插入；第二次则是修改该行，是由事务id号为3的事务进行的，将原来的行记录放入undo回滚段，并修改本行的ROLL_PTR指针指向它；第三次修改该行的事务号为5，同样将原来的行记录放入undo回滚段，并修改本行的ROLL_PTR指针指向它，从而形成一个事务版本的链表。只有当历史版本没有活跃的事务关联时（也就是没有活跃事务需要查看），由清除线程进行历史版本清除工作。

因为事务id号是单调递增的，依照事物的版本来检查每行的版本号，从而判断行的可见性。下面通过源码分析来具体说明innodb mvcc的实现。

## 核心数据结构

### ReadView类
该类是每个THD线程结构持有的一致性视图类，是事务实现一致性读视图的基本结构，用于识别哪些行对本事务是可见的，哪些行对本事务是不可见的。

主要成员变量
m_low_limit_id	表示创建此视图结构时，系统此时最大的事务id+1（也即下一个事务的id，即trx_sys->max_trx_id值），从而如果行记录的事务id只要大于等于此值，则对其不可见
m_up_limit_id	这个变量保存着结构创建时正在运行中事务中的最小事物id，说明小于此事务id的行对本结构都是可见的。
m_creator_trx_id	初始化时，就是事物本身id
m_ids	视图创建时，系统所有活跃的事务id集合。通过该集合我们知道哪些事务，在我们开启视图时正在执行。
m_low_limit_no	小于该值的行不需要查看undo log，直接读取即可。该值初始化与m_up_limit_id一致，除非当事务trx->no更小时，初始化为trx->no。

从核心成员变量可以看出：

其实整个事物可见性就是通过在事务创建时构建readview结构来实现的，其中形成一个[m_up_limit_id, m_low_limit_id]区间.

如果row的事务id大于等于m_low_limit_id，对事物不可见，因为修改该行的事务是本身事务之后的未来事物;
如果小于m_up_limit_id，对事物可见，是因为事物开始时，该事务已经提交了。
如果在这之间，我们可以通过m_ids变量，如果row的事务id在m_ids中，说明事务开始时，该行还没有提交，不可见。如果row事务id不在m_ids中，说明事务开始时，该行已经提交，即可见。

### 主要成员函数
初始化创建函数
void prepare(trx_id_t id)
该行是进行初始化readview结构的函数，主要就是初始化上述变量，在创建了新的readview结构体时，需要调用。
void copy_prepare(const ReadView &other);

void copy_complete();
这对函数与上对函数功能相似，不同的是从另一个readview结构构建而已。需要成对出现。

上面的函数功能上是一致的，这里主要分析prepare函数，理解innodb是如何创建初始化readview结构体内相关变量的。
```
/*
该函数时初始化一个readview结构体成员。
参数：
    id  该事务的id
*/
void ReadView::prepare(trx_id_t id) {
  ut_ad(mutex_own(&trx_sys->mutex));
  m_creator_trx_id = id;  //为m_creator_trx_id赋值自身事务id
  /*
  初始化m_low_limit_no，m_low_limit_id，m_up_limit_id都为现在系统最大的事务id，当然后面还需要对
  这些值进行调整。
  */
  m_low_limit_no = m_low_limit_id = m_up_limit_id = trx_sys->max_trx_id;
  /*
  如果此时系统有活跃的读写事务
  1、调用copy_trx_ids成员函数将其id集合拷贝到成员变量m_ids中；
  2、修改m_up_limit_id为活跃事务中的最小id
  */
  if (!trx_sys->rw_trx_ids.empty()) { 
    copy_trx_ids(trx_sys->rw_trx_ids);
  } else {
    m_ids.clear();
  }

  ut_ad(m_up_limit_id <= m_low_limit_id);
  //如果事务系统的事务提交serialisation_list列表不为空，修改m_low_limit_no为最小的提交no
  if (UT_LIST_GET_LEN(trx_sys->serialisation_list) > 0) {
    const trx_t *trx;
    trx = UT_LIST_GET_FIRST(trx_sys->serialisation_list);
    if (trx->no < m_low_limit_no) {
      m_low_limit_no = trx->no;
    }
  }

  m_closed = false;
}
```
通过查看源代码，我们看到了核心成员的初始化过程。下面主要看可见性检测函数。

可见性检测函数
void check_trx_id_sanity(trx_id_t id, const table_name_t &name)
该函数就是检测表name中的行记录id是否合法，主要是与系统最大是事物id比较，如果比之大，说明这个事物id是无效的。主要被成员函数changes_visible调用，用于检测一致性视图事物id后是否合法有效。
bool changes_visible(trx_id_t id, const table_name_t &name) const
最主要的函数，用于判断参数给定的事务id，在此事务中是否可见。精简代码如下：
```
bool changes_visible(trx_id_t id, const table_name_t &name) const
      MY_ATTRIBUTE((warn_unused_result)) {
    ut_ad(id > 0);
    /*
    如果给定row的事务id小于m_up_limit_id，则说明该行是事务开始前就已经提交了
    如果给定row的事务id等于m_creator_trx_id，说明该行是自身修改的记录
    以上两种情况放回true，可见
    */
    if (id < m_up_limit_id || id == m_creator_trx_id) {
      return (true);
    }
    
    check_trx_id_sanity(id, name);  //检测给定row的事务id是否合法
    
    //如果id>= m_low_limit_id，说明该行是事务开始之后，修改的行，不可见
    if (id >= m_low_limit_id) {
      return (false);
    } 
    /*
    此时id只能在[m_up_limit_id，m_low_limit_id）区间，
    如果开始事务时，没有活跃状态的读写事务，则可见
    */
    else if (m_ids.empty()) { 
      return (true);
    }
    //最后检测id是否在活跃事务列表中，如果在，则不可见，如果不在，则可见
    const ids_t::value_type *p = m_ids.data();
    return (!std::binary_search(p, p + m_ids.size(), id));
  }
```

## MVCC类
MVCC是实现一致性读的核心结构，本质上是readview的管理类，主要管理所有的readview结构体。innodb事务管理类trx_sys_t通过MVCC类来实现一致性视图管理。下面对MVCC类的简要分析
m_free	用于维护空闲的ReadView对象，初始化时创建1024个ReadView对象（trx_sys_create），当释放一个活跃的视图时，会将其加到该链表上，以便下次重用
m_views	这里存储了两类视图，一类是当前活跃的视图，另一类是上次被关闭的只读事务视图。后者主要是为了减少视图分配开销。因为当系统的读占大多数时，如果在两次查询中间没有进行过任何读写操作，那我们就可以重用这个ReadView，而无需去持有trx_sys->mutex锁重新分配；

主要成员函数
get_view()	从m_free列表获得一个新的readview结构或者额外新建一个
view_open(view,trx)	通过调用get_view()，view-> prepare, view-> copy_trx_ids构建新的view，并初始化。
view_release(view)	释放一个readview，从m_views列表删除，移到m_free列表中
size()	返回m_views中，非关闭的视图个数

## Innodb MVCC实现过程
下面我们通过从视图的创建，可见性判断，以及隔离级别的影响三个方面来分析Innodb MVCC实现细节。

### 会话的视图创建
在innodb内部，通过trx_assign_read_view函数，为本身事务trx_t创建一个一致性视图结构readview。集体逻辑如下：
```
/*
该函数是为innodb层事务结构体trx_t创建一个一致性试图readview
*/
ReadView *trx_assign_read_view(trx_t *trx) /*!< in/out: active transaction */
{
  //确保事务状态是active  
  ut_ad(trx->state == TRX_STATE_ACTIVE);
  //如果服务器是只读状态，则不需要创建一致性视图
  if (srv_read_only_mode) {
    ut_ad(trx->read_view == NULL);
    return (NULL);
  } else if (!MVCC::is_view_active(trx->read_view)) {
    //从全局事务视图管理类mvcc获取一个一致性视图结构readview
    trx_sys->mvcc->view_open(trx->read_view, trx);
  }

  return (trx->read_view);
}
```
注意：上述函数的调用并不是服务层开启事务（如执行begin语句）时，或者innodb层事务激活（调用trx_start_low）时。而是在真正操作或者查看数据库数据时才为其创建该视图结构。

### 可见性判断
对于记录的可见性判断，是在函数lock_clust_rec_cons_read_sees中，该函数主要是判断给定的记录对本事务是否可见，如果不可见，则需要进一步查找更早的版本。
```
/*
该函数检测一条记录是否可见，返回true为可见，返回false为不可见，需进一步查找更早的版本
*/
bool lock_clust_rec_cons_read_sees(
    const rec_t *rec,     /*!< in: user record which should be read or
                          passed over by a read cursor */
    dict_index_t *index,  /*!< in: clustered index */
    const ulint *offsets, /*!< in: rec_get_offsets(rec, index) */
    ReadView *view)       /*!< in: consistent read view */
{
  ut_ad(index->is_clustered());
  ut_ad(page_rec_is_user_rec(rec));
  ut_ad(rec_offs_validate(rec, index, offsets));

  /*
  由于临时表是各个连接私有的，不同连接是不能相互访问各自的临时表，所以
  1、服务器为只读模式
  2、访问的是临时表
  以上情况都是可见的
  */
  if (srv_read_only_mode || index->table->is_temporary()) {
    ut_ad(view == 0 || index->table->is_temporary());
    return (true);
  }
  //获取记录行的事务id
  trx_id_t trx_id = row_get_rec_trx_id(rec, index, offsets);
  //通过本事务的readview视图的changes_visible函数检测该行的可见性
  return (view->changes_visible(trx_id, index->table->name));
}

```
## 隔离级别影响
现在我们来思考另一个问题？那事务隔离级别又是如何影响相关可见性的呢？对于不同的事务隔离级别：
tx_isolation='READ-COMMITTED'

语句级别的一致性：只要当前语句执行前已经提交的数据都是可见的。

tx_isolation='REPEATABLE-READ'

事务级别的一致性：只要是当前事务执行前已经提交的数据都是可见的。

从上面隔离级别的表述，我们也可以发现端倪：隔离级别不同，线程事务结构中的readview申请和释放时机不同，即可实现不同的事务隔离级别。

具体如下：在READ-COMMITTED隔离级别下，每次语句开始重新获取readview即可；在REPEATABLE-READ隔离级别下，事务初始化构建readview，事务结束才释放readview。

READ-COMMITTED隔离级别
ha_innobase::external_lock中，该函数是新语句开始的都会调用该函数，在该函数中，有如下代码：
```
//如果隔离级别为READ-COMMITTED，则关闭先前的一致性视图
} else if (trx->isolation_level <= TRX_ISO_READ_COMMITTED &&
               MVCC::is_view_active(trx->read_view)) {
      mutex_enter(&trx_sys->mutex);

      trx_sys->mvcc->view_close(trx->read_view, true);

      mutex_exit(&trx_sys->mutex);
}
```
重新获取是在row_search_for_mysql下的row_search_mvcc函数中，该函数执行语句搜索时调用。这样就可以根据当前的全局事务链表创建read_view的事务区间，实现read committed隔离级别。
```
//判断是否是语句开始的事务，true表示语句事务
if (!prebuilt->sql_stat_start) {
    /* No need to set an intention lock or assign a read view */

    if (!MVCC::is_view_active(trx->read_view) && !srv_read_only_mode &&
        prebuilt->select_lock_type == LOCK_NONE) {
      ib::error(ER_IB_MSG_1031) << "MySQL is trying to perform a"
                                   " consistent read but the read view is not"
                                   " assigned!";
      trx_print(stderr, trx, 600);
      fputc('\n', stderr);
      ut_error;
    }
  } else if (prebuilt->select_lock_type == LOCK_NONE) {
    /* This is a consistent read */
    /* Assign a read view for the query */
    //开启一个一致性视图
    if (!srv_read_only_mode) {
      trx_assign_read_view(trx);
    }

    prebuilt->sql_stat_start = FALSE;
  } else {
```
REPEATABLE-READ隔离级别
具体创建是在事务创建的是时候，一直维持到事务结束，具体函数为innobase_start_trx_and_assign_read_view，具体相关代码为
```
  //如果隔离级别是RR，则开启一个一致性视图
  if (trx->isolation_level == TRX_ISO_REPEATABLE_READ) {
    trx_assign_read_view(trx);
  } else {
    push_warning_printf(thd, Sql_condition::SL_WARNING, HA_ERR_UNSUPPORTED,
                        "InnoDB: WITH CONSISTENT SNAPSHOT"
                        " was ignored because this phrase"
                        " can only be used with"
                        " REPEATABLE READ isolation level.");
  }
```
结束是在trx_commit_in_memory事务提交中，具体代码如下：
```
    if (trx->read_view != NULL) {
      trx_sys->mvcc->view_close(trx->read_view, false);
    }
```

针对RR隔离级别，在第一次创建readview后，这个readview就会一直持续到事务结束，也就是说在事务执行过程中，数据的可见性不会变，所以在事务内部不会出现不一致的情况。
针对RC隔离级别，事务中的每个查询语句都单独构建一个readview，所以如果两个查询之间有事务提交了，两个查询读出来的结果就不一样。从这里可以看出，在InnoDB中，RR隔离级别的效率是比RC隔离级别的高。
针对RU隔离级别，由于不会去检查可见性，所以在一条SQL中也会读到不一致的数据。
针对串行化隔离级别，InnoDB是通过锁机制来实现的，而不是通过多版本控制的机制，所以性能很差。
