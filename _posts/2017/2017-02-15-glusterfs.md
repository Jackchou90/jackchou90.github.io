---
title: glusterfs
layout: post
categories: notes
---

* content
{:toc}

## 各xlator中init函数的执行

glusterfsd.c main => glusterfs_volumes_init => glusterfs_process_volfp => glusterfs_graph_activate => glusterfs_graph_init(循环初始化每个xlator) => xlator_init => __xlator_init => ret = xl->init (xl);  

## 各xlator结构体中钩子函数的初始化


xlator_set_type => xlator_dynload

## 各xlator结构体中private指针的初始化位置

统一在各xlator对应的init钩子函数中,

例如fuse xlator的init函数中:  
>xlators/mount/fuse/src/fuse-bridge.c

```c
this_xl->private = (void *) priv;
```

例如client xlator的init函数中:
>xlators/protocol/client/src/client.c

```c
this->private = conf;
```



## fuse_thread_proc线程的创建过程

fuse_thread_proc线程是glusterfs client端用于监听/dev/fuse事件的循环线程。

>xlators/protocol/client/src/client.c

init => client_init_rpc => client_rpc_notify => client_handshake => client_dump_version_cbk => client_setvolume => client_setvolume_cbk => ret = client_notify_dispatch (this, GF_EVENT_CHILD_CONNECTING, NULL); => default_notify => xlator_notify => this->ctx->master->notify(fuse xlator的notify函数)  =>  ret = gf_thread_create (&private->fuse_thread, NULL, fuse_thread_proc, this);（到此，完成了fuse_thread_proc线程的创建）  


## fuse_thread_proc线程监听事件的处理

>xlators/mount/fuse/src/fuse-bridge.c

以mknod事件为例:

fuse_ops[finh->opcode] (this,finh,msg) => fuse_std_ops[mknode](实际调用这个表中的函数指针) => fuse_mknod => fuse_resolve_and_resume => fuse_resolve_all => fuse_resolve_done => fuse_fop_resume => fn(state) (实际调用fuse_mknod_resume) => fuse_mknod_resume(state) => FUSE_FOP（宏函数）=>  STACK_WIND (frame, ret, xl, xl->fops->fop, args); (其中xl = state->active_subvol 是最顶层xlator，从此，mknode事件通过STACK_WIND层层向下传递)  

上面xl = state->active_subvol的由来:

fuse_graph_sync函数中new_subvol = priv->active_subvol = priv->next_graph->top; => fuse_active_subvol函数中return priv->active_subvol; => 最终get_fuse_state函数中 active_subvol = fuse_active_subvol (state->this); ……; state->active_subvol = active_subvol;(所以，state->active_subvol实际上指向的是graph的top xlator)


## client xlator（client端最底层叶子xlator）到 rpc库处理流程范例

同样以mknod事件为例,

1. 在client xlator初始化阶段初始化this->private->fops指针(该指针指向client事件处理函数表):init => client_init_rpc => client_rpc_notify => client_handshake => client_dump_version_cbk => select_server_supported_programs => conf = this->private(此处this指向的是client xlator); ……; conf->fops = &clnt3_3_fop_prog(初始化fops指针为clnt3_3_fop_prog); 
2. fops(client.c中) => client_mknod => proc = &conf->fops->proctable[GF_FOP_MKNOD]; ……; ret = proc->fn (frame, this, &args); => clnt3_3_fop_prog(最终调用该结构体中的事件表proctable中的钩子函数) => client3_3_mknod => client_submit_request => rpc_clnt_submit（至此mknod事件由client xlator传递到rpc库最终以请求的形式发送到server端）


## glusterfs创建新文件
  

### FUSE内核空间发送请求到用户空间  
  



    sys_open
     => do_sys_open
      => do_filp_open
       => path_openat
        => do_last
         => lookup_open
          => atomic_open   //FUSE 此处使用atomic_open而非vfs_create
           => error = dir->i_op->atomic_open(dir, dentry, file, open_flag, mode, need_lookup, opened);  //调用具体文件系统inode_operations方法
            => fuse_atomic_open  //该函数为上步实际调用的钩子函数(fuse中)
             => fuse_create_open
              => 
                 1) err = fuse_simple_request(fc, &args);  //发送request请求，并将返回结果保存在args.out中,下面的2),3),4),5),6)都是在该函数执行完成后才执行的, 该函数非常重要，函数的成功返回，意味着glusterfs已成功创建文件, 后面会重点讲该函数.
                 
                 2) ff->fh = outopen.fh;  //此处fh为glusterfs中对应文件对应的fd_t *fd类型指针变量的变量值(也即内存地址), ff为struct fuse_file *ff;, 后期在FUSE读/写操作时，FUSE会通过查询struct fuse_file结构体中的fh值，并将该值以fuse req的形式发送到glusterfs应用层，glusterfs根据该fh值计算得到对应的fd_t *fd指针变量，从而明确该对glusterfs中的哪个文件进行操作.
                 3) ff->nodeid = outentry.nodeid; //此处nodeid为glusterfs中文件对应的inode_t * inode指针变量的变量值(也即内存地址).
                 4) inode = fuse_iget(dir->i_sb, outentry.nodeid, outentry.generation, &outentry.attr, entry_attr_timeout(&outentry), 0); //为新文件生成新inode
                 5) d_instantiate(entry, inode); //将上步中新生成的inode和对应的entry关联。
                 6) 到此，新文件创建完毕.

#### 内核FUSE提交请求到pending队列  

  

**fuse_simple_request**函数非常重要，该函数的返回标志着glusterfs client处理完成并将结果通过参数返回, 细节如下,
   
    1) err = fuse_simple_request(fc, &args); //继续跟踪该函数到glusterfs的整个过程
     => fuse_request_send
      => fuse_request_send
       => __fuse_request_send
        => 
          1) queue_request(fiq, req);  //将新生成的fuse请求添加到fc->iq->pending队列中, 然后唤醒fud->fc->iq->waitq上的第一个等待节点
             =>
               1) list_add_tail(&req->list, &fiq->pending);
               2) wake_up_locked(&fiq->waitq);  //唤醒glusterfs的/dev/fuse监听线程

          2) request_wait_answer(fc, req);  
             => wait_event(req->waitq, test_bit(FR_FINISHED, &req->flags));  //调用kernel wait_event接口将当前进程睡眠在req->waitq(wait_queue_head_t)指向的等待队列上。

之后glusterfs client端监听/dev/fuse文件的监听线程的内核上下文被唤醒, 从而读取出fiq->pending队列中的req请求节点并反馈到glusterfs client端用户上下文，详情参见FUSE用户空间处理流程.

#### 内核FUSE接收并处理请求返回值

    2) ff->fh = outopen.fh;  //此处fh为glusterfs中对应文件对应的fd_t *fd类型指针变量的变量值(也即内存地址), ff为struct fuse_file *ff;, 后期在FUSE读/写操作时，FUSE会通过查询struct fuse_file结构体中的fh值，并将该值以fuse     req的形式发送到glusterfs应用层，glusterfs根据该fh值计算得到对应的fd_t *fd指针变量，从而明确该对glusterfs中的哪个文件进行操作.
    3) ff->nodeid = outentry.nodeid; //此处nodeid为glusterfs中文件对应的inode_t * inode指针变量的变量值(也即内存地址).
    4) inode = fuse_iget(dir->i_sb, outentry.nodeid, outentry.generation, &outentry.attr, entry_attr_timeout(&ou//为新文件生成新inode
    5) d_instantiate(entry, inode); //将上步中新生成的inode和对应的entry关联。
    6) 到此，新文件创建完毕.



### FUSE用户空间(glusterfs client端)处理请求并返回处理结果到内核

#### fuse_thread_proc 处理请求线程

fuse_thread_proc函数是FUSE用户空间(glusterfs client端)的事件处理函数, 该函数监听/dev/fuse文件上的内核事件，

    fuse_ops[finh->opcode] (this,finh,msg) 
     => fuse_std_ops[FUSE_CREATE] 
      => fuse_create 
       => fuse_resolve_and_resume (state, fuse_create_resume); 
        => fuse_resolve_all
         => fuse_resolve_done
          => fuse_fop_resume
           => fn(state) (实际调用在fuse_resolve_and_resume函数中注册的fuse_create_resume函数)
            => fuse_create_resume(state)
             => FUSE_FOP (state, fuse_create_cbk, GF_FOP_CREATE,
                  create, &state->loc, state->flags, state->mode,
                  state->umask, fd, state->xdata);  //该函数最终通过STACK_WIND层层调用子xlator钩子函数发送“创建文件请求”到服务器，并注册fuse_create_cbk为client端收到“创建文件请求返回”的glusterfs fuse层的回调函数。最终服务器端成功创建文件并将文件的属性信息(gfid等)反馈给client端，client端自下而上调用个层xlator注册的回调钩子函数，最终到达glusterfs fuse层，调用fuse_create_cbk回调函数.
  
此处略去FUSE_FOP向下层层调用，创建文件成功后，最终调用glusterfs fuse xlator注册的回调函数fuse_create_cbk的执行讲起,

#### fuse_create_cbk 文件创建完成回调函数

    fuse_create_cbk
     => 
       1) foo.fh = (uintptr_t) fd;  //将fd_t *fd指针变量的值(也即地址值)保存到foo.fh中，后面会传输到fuse内核空间中。
       2) feo.nodeid = inode_to_fuse_nodeid (linked_inode); //将fd_t *fd对应的inode_t *linked_inode指针变量值，保存到feo.nodeid中，后面会传输到fuse内核空间中。
       
       3) iov_out[0].iov_base = &fouh; // 保存struct fuse_out_header  fouh , 用于传输到fuse内核空间中。
       4) iov_out[1].iov_base = &feo; // 保存struct fuse_entry_out   feo , 用于传输到fuse内核空间中。
       5) iov_out[2].iov_base = &foo; // 保存struct fuse_open_out    foo , 用于传输到fuse内核空间中。
       
       6) send_fuse_iov (this, finh, iov_out, 3) // 返回处理结果到内核
        => 
          1) fouh->unique = finh->unique;  //以请求req中的finh->unique填充返回消息的fouh->unique，该值是消息的唯一ID识别码。
          2) res = writev (priv->fd, iov_out, count); //通过sys_writev系统调用，将iov_out中数据传入内核，供内核中的fuse文件系统处理。

到此，glusterfs fuse xlator将文件创建的处理结果通过" res = writev (priv->fd, iov_out, count);"反馈给了kernel, 下来继续跟踪sys_writev，

#### 用户空间FUSE将请求处理结果返回到内核空间

     
     2) res = writev (priv->fd, iov_out, count);
     => sys_writev
      => vfs_writev
       => do_readv_writev
        => ret = do_iter_readv_writev(file, &iter, pos, iter_fn);  //此处最终调用iter_fn参数指向的具体文件系统中的钩子函数()
         => ret = fn(&kiocb, iter); //fn为具体文件系统中file_operations结构体的.write_iter函数指针
          => fuse_dev_write //上步通过fn指针实际调用该函数
           => fuse_dev_do_write
            =>
              1) req = request_find(fpq, oh.unique); //以oh.unique为消息ID从FUSE processing 队列中找到之前的req.
              2) err = copy_out_args(cs, &req->out, nbytes); //将cs中保存的用户空间处理后的返回数据填充到req->out中.
              3) request_end(fc, req); //唤醒req->waitq等待队列, 此时fuse_simple_request函数所在的进程从阻塞态变成就绪态，继续后面的操作，后续操作参考"内核FUSE接收并处理请求返回值"小节.

              
到此，最开始的sys_open函数返回，完成了文件的创建并打开.   

### FUSE内核态及用户态重要数据结构

glusterfs内核态FUSE和用户态FUSE交互过程中，使用到的**重要数据结构**:  


    /** FUSE inode */
    struct fuse_inode {
	    /** Inode data */
	    struct inode inode; //内核态FUSE中文件对应的vfs inode变量.
    
	    /** Unique ID, which identifies the inode between userspace
	     * and kernel */
	    u64 nodeid; //glusterfs client端用户空间inode_t *inode变量值.
    
	    /** Number of lookups on this inode */
	    u64 nlookup;
    
	    /** The request used for sending the FORGET message */
	    struct fuse_forget_link *forget;
    
	    /** Time in jiffies until the file attributes are valid */
	    u64 i_time;
    
	    /** The sticky bit in inode->i_mode may have been removed, so
	        preserve the original mode */
	    umode_t orig_i_mode;
    
	    /** 64 bit inode number */
	    u64 orig_ino;
    
	    /** Version of last attribute change */
	    u64 attr_version;
    
	    /** Files usable in writepage.  Protected by fc->lock */
	    struct list_head write_files; //指struct fuse_inode对应的struct fuse_file结构体链表。
    
	    /** Writepages pending on truncate or fsync */
	    struct list_head queued_writes;
    
	    /** Number of sent writes, a negative bias (FUSE_NOWRITE)
	     * means more writes are blocked */
	    int writectr;
    
	    /** Waitq for writepage completion */
	    wait_queue_head_t page_waitq;
    
	    /** List of writepage requestst (pending or sent) */
	    struct list_head writepages;
    
	    /** Miscellaneous bits describing inode state */
	    unsigned long state;
    };
    
    
    
    
    
    /** FUSE specific file data */
    struct fuse_file {     //内核态中保存“用户空间文件”和“内核空间文件” “对应关系” 的结构体，在FUSE新创建文件时针对特定文件创建并初始化该结构体，后续FUSE读写操作都需从这里获取"用户空间文件"的唯一标识(fh, inodeid), 所以该结构体非常重要,     只有FUSE内核态有.    
    
	    /** Fuse connection for this file */
	    struct fuse_conn *fc;
    
	    /** Request reserved for flush and release */
	    struct fuse_req *reserved_req;
    
	    /** Kernel file handle guaranteed to be unique */
	    u64 kh;
    
	    /** File handle used by userspace */
	    u64 fh; //glusterfs client端用户空间fd_t *fd变量值.
    
	    /** Node id of this file */
	    u64 nodeid; //glusterfs client端用户空间inode_t *inode变量值.
    
	    /** Refcount */
	    atomic_t count;
    
	    /** FOPEN_* flags returned by open */
	    u32 open_flags;
    
	    /** Entry on inode's write_files list */
	    struct list_head write_entry;
    
	    /** RB node to be linked on fuse_conn->polled_files */
	    struct rb_node polled_node;
    
	    /** Wait queue head for poll */
	    wait_queue_head_t poll_wait;
    
	    /** Has flock been performed on this file? */
	    bool flock:1;
    };
    
    
    struct fuse_in_header { //FUSE内核态发送请求给用户态时消息头，FUSE内核态和glusterfs cliet端FUSE用户态均有该定义。
	    uint32_t	len;
	    uint32_t	opcode; //操作码
	    uint64_t	unique;  //消息ID
	uint64_t	nodeid; //从struct fuse_file *ff结构中取出的的nodeid值，该值实际是glusterfs用户空间成功创建文件后glusterfs client端用户空间的文件对应的inode    _t *inode指针变量的值，也即，glusterfs client中新创建的文件对应的inode_t类型变量在client端内存中的地址。  
	    uint32_t	uid;
	    uint32_t	gid;
	    uint32_t	pid;
	    uint32_t	padding;
    };    
    
    struct fuse_out_header { //glusterfs client端FUSE用户态返回请求给内核态时消息头，FUSE内核态和glusterfs cliet端FUSE用户态均有该定义。
	    uint32_t	len;
	    int32_t		error;
	    uint64_t	unique; //消息ID，与struct fuse_in_header中的unique保持一致.
    };
    
    struct fuse_entry_out { //glusterfs client端FUSE用户态返回给内核态数据，最终保存到内核中FUSE req->out中.
	    uint64_t	nodeid;	//glusterfs client端用户空间inode_t *inode变量值.
	    uint64_t	generation;	/* Inode generation: nodeid:gen must
					       be unique for the fs's lifetime */
	    uint64_t	entry_valid;	/* Cache timeout for the name */
	    uint64_t	attr_valid;	/* Cache timeout for the attributes */
	    uint32_t	entry_valid_nsec;
	    uint32_t	attr_valid_nsec;
	    struct fuse_attr attr;
    };
    
    struct fuse_open_out { //glusterfs client端FUSE用户态返回给内核态数据，最终保存到内核中FUSE req->out中.
	    uint64_t	fh; //glusterfs client端用户空间fd_t *fd变量值.
	    uint32_t	open_flags;
	    uint32_t	padding;
    };

    

## glusterfs写文件(以同步写&BIO为例来说明)

### FUSE 内核发送请求

    sys_write         
     => vfs_write
      => __vfs_write
       => new_sync_write
        => ret = filp->f_op->write_iter(&kiocb, &iter); //进入FUSE文件系统中
         => fuse_file_write_iter
          => fuse_perform_write //BIO流程
           => 
             1) fuse_fill_write_pages 
                => 
                  1)tmp = iov_iter_copy_from_user_atomic(page, ii, offset, bytes); //通过iov_iter_copy_from_user_atomic函数将用户空间数据拷贝到page中.
                  2) req->pages[req->num_pages] = page; //将page指针保存到req中.
                  3) 循环以上1)2),最终将所有用户空间数据都保存到page，并将page指针关联到req中.
             2) fuse_send_write_pages //发送写请求
                => 
                  1) fuse_wait_on_page_writeback(inode, req->pages[i]->index); //循环等待页缓存回写机制完成所有req中的page的回写操作，下面会详细讲该过程.
                  2) res = fuse_send_write(req, &io, pos, count, NULL); //上步所有req中page页回写完成，这里真正发送请求.
                     =>
                       1) fuse_write_fill(req, ff, pos, count); 
                          =>
                            1) inarg->fh = ff->fh; //传入fh, fh为FUSE一开始创建文件成功后，glusterfs client端中用户空间新创建文件的fd_t *fd指针值. inarg的类型为struct fuse_write_in *inarg, 最终会被glusterfs client端的fuse_write函数获取到:struct fuse_write_in *fwi = (struct fuse_write_in *) (finh + 1);, 此时glusterfs client端根据fwi->fh就能算出对应的fd_t *fd。 

                            2) req->in.h.opcode = FUSE_WRITE; //写操作码

                       2) fuse_request_send //同步发送请求接口, 最终阻塞在req->waitq队列上.

后续操作流程类似FUSE创建新文件，略去.   


## FUSE 页回写机制详解

#### FUSE 将用户空间数据写入页缓存  

写进程将数据写入页缓存并标记为脏页,

       同FUSE写操作流程
        ......
        => ret = filp->f_op->write_iter(&kiocb, &iter); //进入FUSE文件系统中
         => fuse_file_write_iter
          => fuse_perform_write //BIO流程
            =>
              1) fuse_fill_write_pages 
              =>
                 1)tmp = iov_iter_copy_from_user_atomic(page, ii, offset, bytes); //通过iov_iter_copy_from_user_atomic函数将用户空间数据拷贝到page中.
                 2) req->pages[req->num_pages] = page; //将page指针保存到req中.
                 3) 循环以上1)2),最终将所有用户空间数据都保存到page，并将page指针关联到req中.
                 4) 成功将所有用户空间数据拷贝到req关联的页缓存中.
             
               2) fuse_send_write_pages //发送写请求
               => 
                  1) fuse_wait_on_page_writeback(inode, req->pages[i]->index); //循环等待页缓存回写机制完成所有req中的page的回写操作.
                     => wait_event(fi->page_waitq, !fuse_page_is_writeback(inode, index)); //等待在fi->page_waitq上，该队列被唤醒意味页缓存回写完成.
                  2) fuse_send_write //发送请求实施函数，最终将req添加到pending队列上，然后将当前进程阻塞在req->waitq等待队列上.

后续操作流程类似FUSE创建新文件，略去, 下面重点讲写进程将数据写入页缓存并标记为脏页后，页缓存回写机制涉及到FUSE中的流程,

#### FUSE 页缓存回写flush进程，将页缓存中脏数据回写到底层

          触发并启动页缓存回写机制
           => fuse_writepages //参考本博客的linux4.0页缓存回写机制文章
            => write_cache_pages
             => fuse_writepages_fill
              => fuse_writepage_end
               => fuse_writepage_finish
                => wake_up(&fi->page_waitq); //唤醒等待在fi->page_wqitq队列上的进程.      
                 => 至此，完成页缓存回写动作。
                 
                  
