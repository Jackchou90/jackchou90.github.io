---
title: fadvise64_64中POSIX_FADV_DONTNEED选项作用及流程分析
layout: post
---


* content
{:toc}




# fadvise64_64 

>mm/fadvise.c

```c
SYSCALL_DEFINE4(fadvise64_64, int, fd, loff_t, offset, loff_t, len, int, advice)

```


该有多个选项，用于通知内核采取合适的策略来应对当前进程的IO模式. 这里只说POSIX_FADV_DONTNEED选项，其他选项可以查看该系统调用。  

POSIX_FADV_DONTNEED选项是应用通知内核，与文件描述符 fd 关联的文件的指定范围 (offset 和 len 描述)的 page cache 都不需要了，脏页可以刷到盘上，然后直接丢弃了。   

调用流程:   
    
    SYSCALL_DEFINE4(fadvise64_64, int, fd, loff_t, offset, loff_t, len, int, advice)
     => __filemap_fdatawrite_range
      => do_writepages          //该函数很重要，它是内核页缓存回写机制的底层通用调用函数。
         1) mapping->a_ops->writepages如果初始化，调用该writepages钩子函数回写页面
         2) mapping->a_ops->writepages没有初始化，调用generic_writepages回写页面
       
      generic_writepages
        => write_cache_pages
         => __writepage
          => int ret = mapping->a_ops->writepage(page, wbc); //最终调用具体文件系统address_space_operations->writepage钩子函数，将page中的数据写到介质中.
   

延伸阅读:  

* [linux4.0页缓存回写机制](https://jackchou90.github.io/2017/02/24/linux40-page-cache-writeback.html)  
