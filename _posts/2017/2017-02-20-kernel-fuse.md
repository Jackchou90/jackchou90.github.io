---
title: kernel中fuse机制
layout: post
categories: notes
---

>参考 Documentation/filesystems/fuse.txt

* content
{:toc}

## 流程图举例  

FUSE 内核中的流程基本大体上入下图所示，下面以unlink（rm操作）举例,

NOTE: everything in this description is greatly simplified   

    |"rm /mnt/fuse/file"                 |  FUSE filesystem daemon  
    |                                    |
    |                                    |  >sys_read()
    |                                    |    >fuse_dev_read()
    |                                    |      >request_wait()
    |                                    |        [sleep on fc->waitq]
    |                                    |
    |  >sys_unlink()                     |
    |    >fuse_unlink()                  |
    |      [get request from             |
    |       fc->unused_list]             |
    |      >request_send()               |
    |        [queue req on fc->pending]  |
    |        [wake up fc->waitq]         |        [woken up]
    |        >request_wait_answer()      |
    |          [sleep on req->waitq]     |
    |                                    |      <request_wait()
    |                                    |      [remove req from fc->pending]
    |                                    |      [copy req to read buffer]
    |                                    |      [add req to fc->processing]
    |                                    |    <fuse_dev_read()
    |                                    |  <sys_read()
    |                                    |
    |                                    |  [perform unlink]
    |                                    |
    |                                    |  >sys_write()
    |                                    |    >fuse_dev_write()
    |                                    |      [look up req in fc->processing]
    |                                    |      [remove from fc->processing]
    |                                    |      [copy write buffer to req]
    |          [woken up]                |      [wake up req->waitq]
    |                                    |    <fuse_dev_write()
    |                                    |  <sys_write()
    |        <request_wait_answer()      |
    |      <request_send()               |
    |      [add request to               |
    |       fc->unused_list]             |
    |    <fuse_unlink()                  |
    |  <sys_unlink()                     |


## 核心思想

FUSE 在内核中主要维护，

1) 两个 请求队列(用于保存请求)(fud-fc-iq->pending, fud->pq->processing)   
2) 多个(至少两个) 等待队列(用于同步事件)(fud->fc->iq->waitq, req1->waitq, req2->waitq, ……)





## read操作举例  

>fs/fuse/dev.c  

    const struct file_operations fuse_dev_operations = {
	    .owner		= THIS_MODULE,
	    .open		= fuse_dev_open,
	    .llseek		= no_llseek,
	    .read_iter	= fuse_dev_read, // /dev/fuse虚拟设备文件对应sys_read的内核钩子函数
	    .splice_read	= fuse_dev_splice_read,
	    .write_iter	= fuse_dev_write,
	    .splice_write	= fuse_dev_splice_write,
	    .poll		= fuse_dev_poll,
	    .release	= fuse_dev_release,
	    .fasync		= fuse_dev_fasync,
	    .unlocked_ioctl = fuse_dev_ioctl,
	    .compat_ioctl   = fuse_dev_ioctl,
    };
    
    fuse_dev_read       //fud->fc->iq->pending队列消费者, fud->pq->processing队列生产者
      => fuse_dev_do_read
        => err = wait_event_interruptible_exclusive_locked(fiq->waitq,
				!fiq->connected || request_pending(fiq));   //将当前进程阻塞在fud->fc->iq->waitq (wait_queue_head_t)上, 等待生产者产生新的请求并放入fc->iq->    pending队列中并最终唤醒fud->fc->iq->waitq等待队列上的线程。



>fs/read_write.c  

     sys_read   //fud->fc->iq->pending生产者
      =>filp->f_op->read_iter
   
>fs/fuse/file.c   

    static const struct file_operations fuse_file_operations = {
	   .llseek		= fuse_file_llseek,
	   .read_iter	= fuse_file_read_iter,   //sys_read最终调用该函数
	   .write_iter	= fuse_file_write_iter,
	   .mmap		= fuse_file_mmap,
	   .open		= fuse_open,
	   .flush		= fuse_flush,
	   .release	= fuse_release,
	   .fsync		= fuse_fsync,
	   .lock		= fuse_file_lock,
	   .flock		= fuse_file_flock,
	   .splice_read	= generic_file_splice_read,
	   .unlocked_ioctl	= fuse_file_ioctl,
	   .compat_ioctl	= fuse_file_compat_ioctl,
	   .poll		= fuse_file_poll,
	   .fallocate	= fuse_file_fallocate,
    };

    fuse_file_read_iter        //fud->fc->iq->pending生产者
      => generic_file_read_iter    //进入到filemap.c
        => do_generic_file_read   //filemap.c
          => mapping->a_ops->readpage(filp, page);  //通过 address_space_operations的 .readpage 从filemap.c跳转到具体的文件系统中
            => fuse_readpage   //fuse文件系统 .readpage
              => fuse_do_readpage  
                => fuse_send_read
                  => fuse_request_send
                    => __fuse_request_send
                      => 
                      1) queue_request(fiq, req);  //将新生成的fuse请求添加到fc->iq->pending队列中, 然后唤醒fud->fc->iq->waitq上的第一个等待节点
                           =>
                             1) list_add_tail(&req->list, &fiq->pending);
                             2) wake_up_locked(&fiq->waitq);
                      2) request_wait_answer(fc, req);  //
                           => wait_event(req->waitq, test_bit(FR_FINISHED, &req->flags));  //调用kernel wait_event接口将当前进程睡眠在req->waitq(wait_queue_head_t)指向的等待队列上。
    

>fs/fuse/dev.c 

    fuse_dev_write    fud->pq->processing队列消费者
      => fuse_dev_do_write
        => 
           1) req = request_find(fpq, oh.unique); //在fud->pq->processing队列中找到之前的req
           2) err = copy_out_args(cs, &req->out, nbytes); //拷贝用户空间返回值到req的out字段指向的buffer中
           3) list_del_init(&req->list); //将req从fud->pq->processing队列摘除
           3) request_end(fc, req); //唤醒req对应的等待队列(req->waitq)上的等待节点
               => wake_up(&req->waitq);

