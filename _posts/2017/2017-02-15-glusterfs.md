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