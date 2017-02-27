---
title: linux DIO BIO 同步 异步 概念解析
layout: post
---

* content
{:toc}


# 基本概念

在linux MM子系统中，文件的打开方式有两种，Direct IO和 Buffer IO, 区别在于如果以 `O_DIRECT` 方式打开文件， 后续对该文件的访问都会绕page cache，而不以 `O_DIRECT` 方式打开的文件，则默认后续对该文件的访问都会通过page cache。


* DIO: 对文件的操作不经过page cache，绕过page cache。 
* BIO: 对文件的操作经过page cache。
* sync(同步): 通过sys_write, sys_read系统调用接口陷入内核上下文，该同步系统调用会一直等待，直到内核上下文返回。如果DIO内核上下文检测出是DIO&sync，则DIO内核上下文会一直等待，直到底层驱动操作完成返回。 如果DIO内核上下文检测出是DIO&async，则DIO会申请一个完成队列，然后直接以`-EIOCBQUEUED`返回。 如果是BIO上下文，则内核不做sync和async的检测，直接将用户端的读写请求转化为对page cache的操作后直接返回，之后内核中的页回写机制完成将page cache中的脏页同步到具体的存储介质中。
* async(异步): 通过sys_iosubmit系统调用完成的读写操作，都是异步IO操作.

# 以ext4文件系统写操作，画图说明  

![dio-bio-sync-async.png](/assets/2017/dio-bio-sync-async.png)

# 用于标记和判断同步异步操作的函数

init_sync_kiocb 标记kiocb为同步模式(实质上是不初始化 struct kiocb 的 ki_complete 字段，为NULL)

is_sync_kiocb 判断kiocb是否为同步模式(实质上是判断struct kiocb 的 ki_complete是否为NULL)

# 结束语

所以，上层使用同步和异步接口在下层的DIO处理流程中是区分处理的，而在BIO中都是一样的.

