---
layout: post
categories: [Linux]
description: none
keywords: Linux
---
# 文件系统和模块设备的Page cache问题

## 普通文件的address space
文件系统读取文件一般会使用do_generic_file_read()，mapping指向普通文件的address space。如果一个文件的某一块不在page cache中，在find_get_page函数中会创建一个page，并将这个page根据index插入到这个普通文件的address space中。这也是我们熟知的过程。
```
static ssize_t do_generic_file_read(struct file *filp, loff_t *ppos,
        struct iov_iter *iter, ssize_t written)
{
    struct address_space *mapping = filp->f_mapping;
    struct inode *inode = mapping->host;
    struct file_ra_state *ra = &filp->f_ra;
    pgoff_t index;
    pgoff_t last_index;
    pgoff_t prev_index;
    unsigned long offset;      /* offset into pagecache page */
    unsigned int prev_offset;
    int error = 0;
 
    index = *ppos >> PAGE_CACHE_SHIFT;
    prev_index = ra->prev_pos >> PAGE_CACHE_SHIFT;
    prev_offset = ra->prev_pos & (PAGE_CACHE_SIZE-1);
    last_index = (*ppos + iter->count + PAGE_CACHE_SIZE-1) >> PAGE_CACHE_SHIFT;
    offset = *ppos & ~PAGE_CACHE_MASK;
 
    for (;;) {
        struct page *page;
        pgoff_t end_index;
        loff_t isize;
        unsigned long nr, ret;
 
        cond_resched();
find_page:
        page = find_get_page(mapping, index);
        if (!page) {
            page_cache_sync_readahead(mapping,
                    ra, filp,
                    index, last_index - index);
            page = find_get_page(mapping, index);
            if (unlikely(page == NULL))
                goto no_cached_page;
        }
       ......//此处省略约200行
}
```

## 块设备的address space
但是在读取文件系统元数据的时候，元数据对应的page会被加入到底层模块设备的address space中。下面代码的bdev_mapping指向块设备的address space，调用find_get_page_flags()后，一个新的page（如果page不在这个块设备的address space）就被创建并且插入到这个块设备的address space。
```
static struct buffer_head *
__find_get_block_slow(struct block_device *bdev, sector_t block)
{
    struct inode *bd_inode = bdev->bd_inode;
    struct address_space *bd_mapping = bd_inode->i_mapping;
    struct buffer_head *ret = NULL;
    pgoff_t index;
    struct buffer_head *bh;
    struct buffer_head *head;
    struct page *page;
    int all_mapped = 1;
 
    index = block >> (PAGE_CACHE_SHIFT - bd_inode->i_blkbits);
    page = find_get_page_flags(bd_mapping, index, FGP_ACCESSED);
    if (!page)
        goto out;
    ......//此处省略几十行
}
```
## 两份缓存
前面提到的情况是正常的操作流程，属于普通文件的page放在文件的address space，元数据对应的page放在块设备的address space中，大家井水不犯河水，和平共处。但是世事难料，总有一些不按套路出牌的人。文件系统在块设备上欢快地跑着，如果有人绕过文件系统，直接去操作块设备上属于文件的数据块，这会出现什么情况？如果这个数据块已经在普通文件的address space中，这次直接的数据块修改能够立马体现到普通文件的缓存中吗？

答案是直接修改块设备上块会新建一个对应这个块的page，并且这个page会被加到块设备的address space中。也就是同一个数据块，在其所属的普通文件的address space中有一个对应的page。同时，在这块设备的address space中也会有一个与其对应的page，所有的修改都更新到这个块设备address space中的page上。除非重新从磁盘上读取这一块的数据，否则普通文件的文件缓存并不会感知这一修改。

## 实验
口说无凭，实践是检验真理的唯一标准。我在这里准备了一个实验，先将一个文件的数据全部加载到page cache中，然后直接操作设备修改这个文件的数据块，再读取文件的内容，看看有没有被修改。

为了确认一个文件的数据是否在page cache中，我先介绍一个有趣的工具---vmtouch，这个工具可以显示出一个文件有多少内容已经被加载到page cache。大家可以在github上获取到它的源码，并自行编译安装

现在开始我们的表演：

首先，我们找一个测试文件，就拿我家目录下的read.c来测试，这个文件的内容就是一些凌乱的c代码。
````
~ cat read.c 
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
 
char buf[4096] = {0};
 
int main(int argc, char *argv[])
{
	int fd;
	if (argc != 2) {
		printf("argument error.\n");
		return -1;
	}
 
	fd = open(argv[1], O_RDONLY);
	if (fd < 0) {
		perror("open failed:");
		return -1;
	}
 
	read(fd, buf, 4096);
	//read(fd, buf, 4096);
	close(fd);
}
➜  ~
````
接着运行vmtouch，看看这个文件是否在page cache中了，由于这个文件刚才被读取过，所以文件已经全部保存在page cache中了。
```
➜  ~ vmtouch read.c                   
           Files: 1
     Directories: 0
  Resident Pages: 1/1  4K/4K  100%
         Elapsed: 0.000133 seconds
➜  ~
```
然后我通过debugfs找到read.c的数据块，并且通过dd命令直接修改数据块。
```
Inode: 3945394   Type: regular    Mode:  0644   Flags: 0x80000
Generation: 659328746    Version: 0x00000000:00000001
User:     0   Group:     0   Project:     0   Size: 386
File ACL: 0
Links: 1   Blockcount: 8
Fragment:  Address: 0    Number: 0    Size: 0
 ctime: 0x5ad2f108:60154d80 -- Sun Apr 15 14:28:24 2018
 atime: 0x5ad2f108:5db2f37c -- Sun Apr 15 14:28:24 2018
 mtime: 0x5ad2f108:5db2f37c -- Sun Apr 15 14:28:24 2018
crtime: 0x5ad2f108:5db2f37c -- Sun Apr 15 14:28:24 2018
Size of extra inode fields: 32
EXTENTS:
(0):2681460
 
➜  ~ dd if=/dev/zero of=/dev/sda2 seek=2681460 bs=4096 count=1 
1+0 records in
1+0 records out
4096 bytes (4.1 kB, 4.0 KiB) copied, 0.000323738 s, 12.7 MB/s
```
修改已经完成，我们看看直接读取这个文件会怎么样。
```
➜  ~ cat read.c 
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
 
char buf[4096] = {0};
 
int main(int argc, char *argv[])
{
	int fd;
	if (argc != 2) {
		printf("argument error.\n");
		return -1;
	}
 
	fd = open(argv[1], O_RDONLY);
	if (fd < 0) {
		perror("open failed:");
		return -1;
	}
 
	read(fd, buf, 4096);
	//read(fd, buf, 4096);
	close(fd);
}
 
➜  ~ vmtouch read.c 
           Files: 1
     Directories: 0
  Resident Pages: 1/1  4K/4K  100%
         Elapsed: 0.00013 seconds
```
文件依然在page cache中，所以我们还是能够读取到文件的内容。然而当我们drop cache以后，再读取这个文件，会发现文件内容被清空。
```
➜  ~ vmtouch read.c 
           Files: 1
     Directories: 0
  Resident Pages: 1/1  4K/4K  100%
         Elapsed: 0.00013 seconds
➜  ~ echo 3 > /proc/sys/vm/drop_caches                        
➜  ~ vmtouch read.c                   
           Files: 1
     Directories: 0
  Resident Pages: 0/1  0/4K  0%
         Elapsed: 0.000679 seconds
➜  ~ cat read.c
➜  ~
```
普通文件的数据可以保存在它的地址空间中，同时直接访问块设备中此文件的块，也会将这个文件的数据保存在块设备的地址空间中。这两份缓存相互独立，kernel并不会为这种非正常访问同步两份缓存，从而避免了同步的开销。