---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL日志管理redolog

## 概述
redo日志中记录了Innodb引擎对于数据页的修改，主要作用是用来在崩溃恢复过程中，保证数据的完整性。

Innodb引擎采用WAL机制来记录数据，即修改数据时，将redo日志优先记录到磁盘中，真正的数据修改会在后续刷脏过程中记录到磁盘。如果此时数据库意外退出，可以通过redo日志来恢复修改的数据。

## 源码
后续结合源码来介绍redo日志的各个模块。
文件
redo log的文件以格式ib_logfile[N]来命名，通过如下参数控制：
```
innodb_log_file_size = 48M 						// 每个redo文件的大小
innodb_log_files_in_group = 2					// redo文件的个数
innodb_log_group_home_dir=./					// redo文件的存放路径
```
redo log的文件通常在数据库初始化时创建：
```
create_log_files
   // 删除遗留文件，如果存在的话
| for (unsigned i = 0; i <= num_old_files; i++)
    sprintf(logfilename + dirnamelen, "ib_logfile%u", i);
    unlink(logfilename);
    
   // 创建新的redo文件， 并设置文件的大小
| for (unsigned i = 0; i < srv_n_log_files; i++) {
    err = create_log_file(&files[i], logfilename);

  // 为所有的redo log创建一个tablespace
| fil_space_create(  "innodb_redo_log", dict_sys_t::s_log_space_first_id, ..

```
对于innodb来说，所有的redo文件被认为是一个文件，只创建了一个名为"innodb_redo_log"的tablespace，即所有的redo文件的space_id是相同的。

## Block、LSN 、 SN、物理位置
redo文件是循环进行写入，每次操作文件的Block大小为固定大小的512个字节。在这512个字节中，其中header占用12个字节，tailer暂用4个字节，剩余的空间用来记录redo的日志内容
```
#define OS_FILE_LOG_BLOCK_SIZE 512

| header(12 byte) |  redo log record | tailer(4 byte) |
```
LSN为逻辑的日志序列号，是单调递增的。每次有新的redo日志记录，就会相应的增加。

在Innodb中会发现LSN的作用较多，包括用来计算redo日志的位置，用来确认redo日志是否可以重用，用来确认数据页是否可以刷脏等等。起到了一个逻辑时序的作用，LSN越小，代表本次操作越早。比如脏页落盘之前要先保证redo已经落盘，就是通过该脏页对应的LSN和已经落盘的LSN进行对比。

由于redo log是循环写入，总的日志文件大小是不变的，所以可以通过LSN和日志文件的大小，确定该LSN在redo日志中的位置。

SN 和 LSN、物理位置是相互对应的，LSN可以理解为redo日志中近似物理位置的相对位置，SN便可以理解为真正的redo日志内容的相对位置，三者可以相互转换：
```
SN为redo 记录内容的递增
LSN = HEADER *N + SN + TAILER * N
物理位置 = LSN + 文件头(4个BLOCK) * N
```
SN 和 LSN 转换
```
// 将sn转换为lsn，将每一个block添加header_size和tailer_size
log_translate_sn_to_lsn
  // LOG_BLOCK_DATA_SIZE = OS_FILE_LOG_BLOCK_SIZE（512）- header_size - tailer_size
| (sn / LOG_BLOCK_DATA_SIZE * OS_FILE_LOG_BLOCK_SIZE + sn % LOG_BLOCK_DATA_SIZE + LOG_BLOCK_HDR_SIZE)

// 将lsn转换为sn， 将每一个block删除header_size和tailer_size
log_translate_lsn_to_sn
|  sn = lsn / OS_FILE_LOG_BLOCK_SIZE * LOG_BLOCK_DATA_SIZE;
|  sn = sn + lsn % OS_FILE_LOG_BLOCK_SIZE - header_size
```
LSN和物理位置转换
```
// offset 为物理位置
lsn = offset - LOG_FILE_HDR_SIZE * (1 + offset / log.file_size

// lsn为与redo size总大小取模计算后的结果
offset = lsn + LOG_FILE_HDR_SIZE *
                       (1 + lsn / (log.file_size - LOG_FILE_HDR_SIZE)
```
后续会以LSN来描述redo的位置。

## redo日志的写入
redo日志的写入流程如下：
- mtr（Mini transaction）写入函数，将redo 日志内容记录到mtr的buffer中
- mtr commit，将mtr buffer中的redo日志，记录到redo的log buffer中
- 将log buffer中的日志内容写入到系统缓存
- 进行一次flush_sync将系统缓存中的日志内容刷新到系统磁盘
- lsn对应的脏页如果已经刷新到磁盘，对redo log进行checkpoint
- checkpoint成功后，之前的redo日志便可以重复使用

其中mtr为innodb内部最小的原子事务操作，不再展开讨论。上述过程中分别对应redo日志的几个状态，关联几个可见的LSN（show engine innodb status 查看）如下：
```
---
LOG
---
Log sequence number          164495037            // 当前最大的LSN
Log buffer assigned up to    164495037
Log buffer completed up to   164495037            
Log written up to            164495037            // 当前已经写入到系统缓存的LSN
Log flushed up to            164495037            // 当前已经刷新到磁盘的LSN
Added dirty pages up to      164495037             
Pages flushed up to          161538614            // 当前脏页刷新到的LSN
Last checkpoint at           161538614            // 当前checkpoint的LSN
```
log_sys
log_sys为log_t类的对象，为全局唯一的对象，redo操作的相关信息均存在该对象中。
```
log_t *log_sys;
| sn					// 当前最大的LSN(SN)
| write_lsn				// 当前已经写入到系统缓存的LSN
| flushed_to_disk_lsn	// 当前已经刷新到系统磁盘的LSN
| last_checkpoint_lsn	// 当前已经checkpoint的LSN
| ...
| ...
```
该对象在系统启动时创建，同时初始化其中的部分变量。
```
srv_start
| log_sys_init
| | log_sys = UT_NEW_NOKEY(...);
| | ...
| | log.m_first_file_lsn = LOG_START_LSN;
```
写入log buffer
log buffer为redo日志在系统中的缓存，redo日志先暂存在log buffer中，由事务commit时，或者后台线程统一写入到系统缓存中。

定义
log buffer的定义如下，定义在log_sys中。
```
log_t *log_sys;
| buf					// log buffer的buf指针
| buf_size				// log buffer的大小
| buf_size_sn			// 忽略header 和 tailer的log buffer的大小(同SN)

| buf_limit_sn          // 当前buffer 允许写入的最大SN

| recent_written        // 保证log buffer写入系统缓存时，是连续的
```
初始化
在log_sys对象初始化时同时初始化log buffer。
```
// 参数innodb_log_buffer_size控制log_buffer的大小
innodb_log_buffer_size = 16M

srv_start
| log_sys_init
| | ...
| | log_allocate_buffer(log)
| | | log.buf.create(srv_log_buffer_size);
```
log_buffer的大小并不是固定的，如果需要写入的长度大于log_buffer的长度，会自适应的堆log_buffer进行扩容。

## 写入
log buffer的写入流程如下：
- 分配start_len，根据写入的长度预留空间
- 根据mtr中存放的redo log，写入到log buffer中
- 写入完成后，将start_lsn 和 end_lsn 写入 recent_written
- 写入入口在mtr.commit时：
```
mtr_t::commit()   <==> mtr_t::Command::execute
| 1. 根据写入的长度len，来预留空间，并分配start_lsn
| log_buffer_reserve(*log_sys, len)
| | sn_t start_sn = log.sn.fetch_add(len)        		// 将start_sn开始 长度为len的 空间预留下来
| | if (unlikely(end_sn > log.buf_limit_sn.load())
| |   log_wait_for_space_after_reserving(log, handle);  // 空间不足，等待足够的空间
|
| 2. 根据LSN，将redo内容写入log buffer中
| log_buffer_write 
| | byte *ptr = log.buf + (start_lsn % log.buf_size);	// 根据LSN定位地址
| | if (ptr >= buf_end)  ptr -= log.buf_size;           // buffer循环使用
| | std::memcpy(ptr, str, len);
|
| 3. 将start_lsn 和 end_lsn 写入recent_written
| log_buffer_write_completed
| | log.recent_written.add_link_advance_tail(start_lsn, end_lsn);
```
log buffer的使用，同redo日志的文件使用类似，以Block作为写入的单位（同redo日志的block对齐，为512个字节），逻辑上上是无限的空间，内存中是循环使用。地址定位使用LSN进行定位，在操作log buffer时，只需要考虑循环使用的覆盖即可。

log buffer的空间是否不足，是通过log.write_lsn来判断。log.write_lsn之前的数据是已经写入到系统缓存的数据，这部分buffer是可以覆盖复用的。只需要保证 log.write_lsn + buf_size_sn 大于 end_lsn即可。

## 写入系统缓存
log buffer是循环使用，需要将log buffer中的内容，持续的写入到系统缓存中。有较多的地方会将log buffer中的内容写入系统缓存，比如commit、比如后台线程持续的写入等等。

相关变量定义
```
log_t *log_sys
| write_lsn 					// 当前写入系统缓存的最大的LSN

| current_file_lsn  			// 通过LSN计算对应物理文件的位置
| current_file_real_offset
| current_file_end_offset		// 当前文件的最大OFFSET

| recent_written     			// 用户判断当前可以写入的最大LSN
```
写入
判断可以写入的最大LSN
在将log buffer写入到系统缓存时，我们需要写入连续的内存，不能留有空洞，这样才能保证write_lsn之前的log buffer是可以复用的。redo日志是并发写入的，这里引入recent_written来保证连续的redo日志写入。
recent_written的定义如下：
```
Link_buf<lsn_t> recent_written
| m_capacity; 					// 该Link_buf最大的lsn长度
| m_links						
| m_tail						// 当前可以write的最大lsn
```
流程如下：
- 每次写入redo日志到log buffer时，会将该日志的start_lsn 和 end_lsn记录到recent_written中，使用m_links[start_lsn] = end_lsn来记录。LSN是连续的，那么end_lsn为下一次redo日志的起始位置。
- 当查找时，如果m_links[curr_lsn]不为0，则表示这一段lsn还没有写入到log buffer中。当前允许写入到系统缓存的最大lsn就是curr_lsn。
源码如下：
```
Link_buf<Position>::add_link (Position from, Position to)
| index = from & (m_capacity - 1)
| m_links[index].store(to)


Link_buf<Position>::advance_tail
| from = m_tail.load()
| next = m_links[from]
  // mlinks 没有被更新 (0 或者 遗留的旧值) 则表示还没有写入
  // 如果有值则继续推进，一直到第一个没有写入
| if (next < from) return
  else m_tail.store(from )
```
举例说明：
三条redo日志的lsn为(10,15)、(15,20)、(20,25)。当第一条和第三条写入完成时，recent_written中的记录为:
```
recent_written.m_tail = 10

m_links[10] = 15
m_links[15] = 0
m_links[20] = 25

// 调用该函数后，m_tail更新为15，表示LSN为15之前的log buffer可以写入系统缓存
Link_buf<Position>::advance_tail()
```

### 后台线程 log_writter 写入流程

后台线程，log buffer写入系统缓存的流程如下：
- 获取本次写入能写到的最大LSN
- 通过输入的LSN计算本次写入的起始位置和结束位置
- 自适应调整结束位置：不跨文件边界，使用的redo日志空间可以被覆盖
- 将LSN转换为redo日志文件的位置，并进行插入
- 更新write_lsn等元数据
- log buffer写入系统缓存的接口如下
```
 1. 通过recent_written获得当能能写入的最大的LSN
ready_lsn = log_buffer_ready_for_write_lsn(log);
| return log.recent_written.tail()


log_writer_write_buffer(log, ready_lsn)
| 2. 计算起始和结束位置start_offset、 end_offset
| size_t start_offset = write_lsn % log.buf_size;
| size_t end_offset = ready_lsn % log.buf_size;

| 3. 自适应的调整写入的最大LSN
| if (start_offset >= end_offset)  end_offset = log.buf_size;
| if (checkpoint_limited_lsn < ready_lsn)  ready_lsn = checkpoint_limited_lsn;

| 4. 将lsn转换为物理文件的位置并进行插入
| log_files_write_buffer( log, buf_begin, buf_end - buf_begin, ...
| | real_offset = compute_real_offset(log, start_lsn)
| | ...
| | write_blocks(log, write_buf, write_size, real_offset);
| |
| | 5. 更新write_lsn、buf_limit_sn(log buffer写入上限)等原数据
| | log.write_lsn.store(new_write_lsn);
| | log_update_buf_limit(log, new_write_lsn);
```
commit 写入流程
这里包括autocommit、ddl 等操作，不局限于某一种操作。写入流程如下：
- 等待可写入lsn 超过输入的参数 lsn
- 循环写入，直到write_lsn 到达或者超过 lsn
写入接口如下：
```
// 参数flush决定log buffer的redo写入系统缓存后，是否刷新内容到磁盘，这里暂时不做讨论。

log_write_up_to(*log_sys, lsn, flush) <==> log_self_write_up_to
  1. 等待ready_lsn 超过输入的lsn
| while (i < srv_n_spin_wait_rounds && ready_lsn < end_lsn) {
|  log.recent_written.advance_tail();
   ready_lsn = log_buffer_ready_for_write_lsn(log);

  2. 等待写入的lsn到ready_lsn
| while (write_lsn < ready_lsn) 
|  // 这里的写入同后台线程相同
|  log_writer_write_buffer(log, log_buffer_ready_for_write_lsn(log));

```

写入系统磁盘
redo日志刷盘是将所有的redo文件进行fsync，确保系统缓存中的redo日志内容存储到磁盘上。

redo刷盘由用户线程执行、或者系统后台线程执行。刷盘接口为统一的接口、

相关变量定义
```
log_t *log_sys
| flushed_to_disk_lsn		//当前磁盘刷新到的位置
```
刷新磁盘
刷新磁盘的源码如下：
```
log_flush_low
| const lsn_t last_flush_lsn = log.flushed_to_disk_lsn.load();
| const lsn_t flush_up_to_lsn = log.write_lsn.load();

  // 如果flush_lsn 小于 write_lsn 就进行刷盘
| if (last_flush_lsn < flush_up_to_lsn)
|  fil_flush_file_redo()
|  | // 对所有的redo文件进行刷盘
|  | for (auto &file : space->files) 
|  |   os_file_flush(file.handle);

```

redo的checkpoint
redo日志的checkpoint机制是为了保证redo日志的复用。在WAL机制下，redo日志优先落盘，数据页后续异步后台线程落盘。当数据页落盘之后，redo日志便可以回收复用。
```
log_t *log_sys
| last_checkpoint_lsn		//当前磁盘checkpoint到的位置

| checkpoint_buf            // 用来更新redo的header的buffer

| recent_closed				//判断当前可以刷脏的最大LSN
```
checkpoint
判断可以checkpoint的最大LSN
```
log_update_available_for_checkpoint_lsn
  1. 当前可以刷脏的最大LSN， 
| log_buffer_dirty_pages_added_up_to_lsn(log);
  | recent_closed.tail()

  2. flush list中最小的lsn向后退一个recent_closed.capacity()
| lsn_t lwm_lsn = buf_pool_get_oldest_modification_lwm();

  3. flush_lsn 和 checkpoint_lsn取小
| log.flushed_to_disk_lsn.load() // log.last_checkpoint_lsn.load()

  4. 取三项中最小的LSN最为本次可以 checkpoint的LSN
| log.available_for_checkpoint_lsn = oldest_lsn;

```
recent_closed是在将数据页加入flush_list时使用，原理同write_recnet类似，用以保证recent_closed.tail()之前的数据页都已经加入到flush_list中，在刷脏时不会有遗漏的情况。

checkpoint
```
log_consider_checkpoint(log);
  // 先将dynamic_metadata
| dict_persist_to_dd_table_buffer();
| log_checkpoint(log);
| | log_files_write_checkpoint(log, checkpoint_lsn);
  | | // 生成一个checkpoint_no
  | | checkpoint_no = log.next_checkpoint_no.load();
  
      // 将checkpoint相关信息写入buffer
  | | mach_write_to_8(buf + LOG_CHECKPOINT_NO, checkpoint_no);
  | | mach_write_to_8(buf + LOG_CHECKPOINT_LSN, next_checkpoint_lsn);
	
	  // 写入系统缓存并刷盘
  | | err = fil_redo_io( IORequestLogWrite,
  | | log_fsync();


```
redo使用两个BLOCK来记录checkpoint相关信息，根据checkpoint_no来决定使用哪一个，分别记录在redo日志第2个和第4个BLOCK中，两个checkpoint的block交替使用。