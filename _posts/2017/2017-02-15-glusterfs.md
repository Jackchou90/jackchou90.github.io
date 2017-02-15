---
title: glusterfs
layout: post
categories: notes
---


## xlator结构体中钩子函数的初始化


xlator_set_type => xlator_dynload



## fuse_thread_proc线程的创建过程

fuse_thread_proc线程是glusterfs client端用于监听/dev/fuse事件的循环线程。

>xlators/protocol/client/src/client.c

init => client_init_rpc => client_rpc_notify => client_handshake => client_dump_version_cbk => client_setvolume => client_setvolume_cbk => ret = client_notify_dispatch (this, GF_EVENT_CHILD_CONNECTING, NULL); => default_notify => xlator_notify => this->ctx->master->notify(fuse xlator的notify函数)  =>  ret = gf_thread_create (&private->fuse_thread, NULL, fuse_thread_proc, this);（到此，完成了fuse_thread_proc线程的创建）  

上面init函数的调用栈如下:

main => glusterfs_volumes_init => glusterfs_process_volfp => glusterfs_graph_activate => glusterfs_graph_init(循环初始化每个xlator) => xlator_init => __xlator_init => ret = xl->init (xl);  





## fuse_thread_proc线程监听事件的处理

>xlators/mount/fuse/src/fuse-bridge.c

以mknod事件为例:
fuse_ops[finh->opcode] (this,finh,msg) => fuse_std_ops[mknode](实际调用这个表中的函数指针) => fuse_mknod => fuse_resolve_and_resume => fuse_resolve_all => fuse_resolve_done => fuse_fop_resume => fn(state) (实际调用fuse_mknod_resume) => fuse_mknod_resume(state) => FUSE_FOP（宏函数）=>  STACK_WIND (frame, ret, xl, xl->fops->fop, args); (其中xl = state->active_subvol 是最顶层xlator，从此，mknode事件通过STACK_WIND层层向下传递)  

上面xl = state->active_subvol的由来:
fuse_graph_sync函数中new_subvol = priv->active_subvol = priv->next_graph->top; => fuse_active_subvol函数中return priv->active_subvol; => 最终get_fuse_state函数中 active_subvol = fuse_active_subvol (state->this); ……; state->active_subvol = active_subvol;(所以，state->active_subvol实际上指向的是graph的top xlator)