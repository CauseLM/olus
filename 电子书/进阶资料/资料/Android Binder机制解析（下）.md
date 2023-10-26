# Android跨进程通信IPC之Binder之Framework层C++篇

Framework是一个中间层，它对接了底层的实现，封装了复杂的内部逻辑，并提供外部使用接口。Framework层是应用程序开发的基础。Binder Framework层为了C++和Java两个部分，为了达到功能的复用，中间通过JNI进行衔接。Binder Framework的C++部分，头文件位于这个路径：/frameworks/native/include/binder/。实现位于这个路径：/frameworks/native/libs/binder/。binder库最终会编译成一个动态链接库：/libbinder.so，供其他进程连接使用。今天按照android Binder的流程来源码分析Binder，本篇主要是Framwork层里面C++的内容，里面涉及到的驱动层的调用，请看上一篇文章。我们知道要要想号获取相应的服务，服务必须现在ServiceManager中注册，那么问题来了，ServiceMamanger是什么时候启动的？所以本篇的主要内容如下：

> - 1、ServiceManager的启动
> - 2、ServiceManager的核心服务
> - 3、ServiceManager的获得
> - 4、注册服务
> - 5、获得服务

## 一、ServiceManager的启动

### (一) ServiceManager启动简述

ServiceManager(后边简称 SM) 是 Binder的守护进程。就像前面说的，它本身也是一个Binder的服务。是通过编写binder.c直接和Binder驱动来通信，里面含量一个循环binder_looper来进行读取和处理事务。因为毕竟是手机，只有这样才能达到简单高效。

经过前面几篇文章，大家也知道SM的工作也很简单，就是两个：

> - 1、注册服务
> - 2、查询

因为Binder里面的通信一般都是由BpBinder和BBinder来实现的，就像ActivityManagerProxy与ActivityManagerService之间的通信。

### (二)源码的位置

由于Binder中大部分的代码都是在C层，所以我特意把源码的地址发上来。里面涉及几个类，代码路径如下：



```swift
framework/native/cmds/servicemanager/
  - service_manager.c
  - binder.c
system/core/rootdir
   -/init.rc
kernel/drivers/ (不同Linux分支路径略有不同)
  - android/binder.c 
```

大家如果想看源码请点击下面的对应的类即可

> - [service_manager.c](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/cmds/servicemanager/service_manager.c)
> - [binder.c](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/cmds/servicemanager/binder.c)
> - [init.rc](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/rootdir/init.rc)

**kernel下binder.c这个文件已经不在android的源码里面了，在Linux源码里面**

> - [binder.c ](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)

**强调一下这里面有两个binder.c文件，一个是framework/native/cmds/servicemanager/binder.c，另外一个是kernel/drivers/android/binder.c ，绝对不是同一个东西，千万不要弄混了。**

### (三) 启动过程

在前面文章讲解Binder驱动的时候，我们就说到了：任何使用Binder机制的进程都必须要对**/dev/binder**设备进行open以及mmap之后才能使用，这部分逻辑是所有使用Binder机制进程通用的，SM也不例外。那我们就来看看

启动流程图下：



![img](https:////upload-images.jianshu.io/upload_images/5713484-b04df9be2959a66d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

启动整体流程图.png

> ServiceManager是由init进程通过解析[init.rc](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/rootdir/init.rc)文件而创建的，其所对应的可执行程序是/system/bin/servicemanager，所对应的源文件是service_manager.c，进程名为/system/bin/servicemanager。

代码如下：



```kotlin
// init.rc  602行
service servicemanager /system/bin/servicemanager
    class core
    user system
    group system
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart drm
```

#### **1、service_manager.c**

启动Service Manager的入口函数是service_manager.c的main()方法如下：

```cpp
 //service_manager.c    347行
int main(int argc, char **argv)
{
    struct binder_state *bs;
    //打开binder驱动，申请128k字节大小的内存空间
    bs = binder_open(128*1024);
    ...
    //省略部分代码
    ...
    //成为上下文管理者 
    if (binder_become_context_manager(bs)) {
        return -1;
    }

    selinux_enabled = is_selinux_enabled(); //selinux权限是否使能
    sehandle = selinux_android_service_context_handle();
    selinux_status_open(true);

    if (selinux_enabled > 0) {
        if (sehandle == NULL) {  
            abort(); //无法获取sehandle
        }
        if (getcon(&service_manager_context) != 0) {
            abort(); //无法获取service_manager上下文
        }
    }
    union selinux_callback cb;
    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);
    //进入无限循环，充当Server角色，处理client端发来的请求 
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```

**PS:svcmgr_handler是一个方向指针，相当于binder_loop的每一次循环调用到svcmgr_handler()函数。**
 这部分代码 主要分为3块

> - bs = binder_open(128*1024)：打开binder驱动，申请128k字节大小的内存空间
> - binder_become_context_manager(bs)：变成上下文的管理者
> - binder_loop(bs, svcmgr_handler)：进入轮询，处理来自client端发来的请求

下面我们就详细的来看下这三块的代码

**1.1、 binder_open(128*1024)**

这块代码在framework/native/cmds/servicemanager/binder.c中

```cpp
 // framework/native/cmds/servicemanager/binder.c   96行
struct binder_state *binder_open(size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }

    //通过系统调用进入内核，打开Binder的驱动设备
    bs->fd = open("/dev/binder", O_RDWR);
    if (bs->fd < 0) {
        //无法打开binder设备
        goto fail_open; 
    }
    
    //通过系统调用，ioctl获取binder版本信息
    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        //如果内核空间与用户空间的binder不是同一版本
        goto fail_open; 
    }

    bs->mapsize = mapsize;
    //通过系统调用，mmap内存映射，mmap必须是page的整数倍
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        //binder设备内存映射失败
        goto fail_map; // binder
    }

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
```

> - 1、打开binder相关操作，先调用open()打开binder设备，open()方法经过系统调用，进入Binder驱动，然后调用方法binder_open()，该方法会在Binder驱动层创建一个**binder_proc**对象，再将 **binder_proc** 对象赋值给fd->private_data，同时放入全局链表binder_proc。
> - 2、再通过ioctl检验当前binder版本与Binder驱动层的版本是否一致。
> - 3、调用mmap()进行内存映射，同理mmap()方法经过系统调用，对应Binder驱动层binde_mmap()方法，该方法会在Binder驱动层创建Binder_buffer对象，并放入当前binder_proc的** proc->buffers **  链表

PS:这里重点说下binder_state

```cpp
//framework/native/cmds/servicemanager/binder.c  89行
struct binder_state
{
    int fd;                           //dev/binder的文件描述
    void *mapped;             //指向mmap的内存地址 
    size_t mapsize;           //分配内存的大小，默认是128K
};
```

至此，整个binder_open就已经结束了。

#### **1.2、binder_become_context_manager()函数解析**

代码很简单，如下：

```csharp
 //framework/native/cmds/servicemanager/binder.c   146行
int binder_become_context_manager(struct binder_state *bs)
{
    //通过ioctl，传递BINDER_SET_CONTEXT_MGR执行
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```

变成上下文的管理者，整个系统中只有一个这样的管理者。通过ioctl()方法经过系统调用，对应的是Binder驱动的binder_ioctl()方法。

**1.2.1 binder_ioctl解析**

Binder驱动在Linux 内核中，代码在kernel中
 如下：

```cpp
//kernel/drivers/android/binder.c      3134行
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
     ...
    //省略部分代码
    ...
    switch (cmd) {
       ...
        //省略部分代码
       ...
       //3279行
      case BINDER_SET_CONTEXT_MGR:
          ret = binder_ioctl_set_ctx_mgr(filp);
          if (ret)
        goto err;
      break;
      }
       ...
        //省略部分代码
       ...
    }
    ...
    //省略部分代码
    ...
}
```

根据参数BINDER_SET_CONTEXT_MGR,最终调用binder_ioctl_set_ctx_mgr()方法，这个过程会持有binder_main_lock。

**1.2.2、binder_ioctl_set_ctx_mgr() 是属于Linux kernel的部分，代码**

```rust
//kernel/drivers/android/binder.c   3198行
static int binder_ioctl_set_ctx_mgr(struct file *filp)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    struct binder_context *context = proc->context;

    kuid_t curr_euid = current_euid();
       //保证binder_context_mgr_node对象只创建一次
    if (context->binder_context_mgr_node) {
        pr_err("BINDER_SET_CONTEXT_MGR already set\n");
        ret = -EBUSY;
        goto out;
    }
    ret = security_binder_set_context_mgr(proc->tsk);
    if (ret < 0)
        goto out;
    if (uid_valid(context->binder_context_mgr_uid)) {
        if (!uid_eq(context->binder_context_mgr_uid, curr_euid)) {
            pr_err("BINDER_SET_CONTEXT_MGR bad uid %d != %d\n",
                   from_kuid(&init_user_ns, curr_euid),
                   from_kuid(&init_user_ns,
                     context->binder_context_mgr_uid));
            ret = -EPERM;
            goto out;
        }
    } else {
                //设置当前线程euid作为Service Manager的uid
        context->binder_context_mgr_uid = curr_euid;
    }
        //创建ServiceManager的实体。
    context->binder_context_mgr_node = binder_new_node(proc, 0, 0);
    if (!context->binder_context_mgr_node) {
        ret = -ENOMEM;
        goto out;
    }
    context->binder_context_mgr_node->local_weak_refs++;
    context->binder_context_mgr_node->local_strong_refs++;
    context->binder_context_mgr_node->has_strong_ref = 1;
    context->binder_context_mgr_node->has_weak_ref = 1;
out:
    return ret;
}
```

进入Binder驱动，在Binder驱动中定义的静态变量

**1.2.3 binder_context 结构体**

```cpp
//kernel/drivers/android/binder.c   228行
struct binder_context {
         //service manager所对应的binder_node
    struct binder_node *binder_context_mgr_node;
        //运行service manager的线程uid
    kuid_t binder_context_mgr_uid;
    const char *name;
};
```

创建了全局的binder_node对象binder_context_mgr_node，并将binder_context_mgr_node的强弱引用各加1

这时候我们再来看下binder_new_node()方法里面

**1.2.4、binder_new_node()函数解析**

```php
//kernel/drivers/android/binder.c  
static struct binder_node *binder_new_node(struct binder_proc *proc,
                       binder_uintptr_t ptr,
                       binder_uintptr_t cookie)
{
    struct rb_node **p = &proc->nodes.rb_node;
    struct rb_node *parent = NULL;
    struct binder_node *node;
        //第一次进来是空
    while (*p) {
        parent = *p;
        node = rb_entry(parent, struct binder_node, rb_node);

        if (ptr < node->ptr)
            p = &(*p)->rb_left;
        else if (ptr > node->ptr)
            p = &(*p)->rb_right;
        else
            return NULL;
    }
        //给创建的binder_node 分配内存空间
    node = kzalloc(sizeof(*node), GFP_KERNEL);
    if (node == NULL)
        return NULL;
    binder_stats_created(BINDER_STAT_NODE);
        //将创建的node对象添加到proc红黑树
    rb_link_node(&node->rb_node, parent, p);
    rb_insert_color(&node->rb_node, &proc->nodes);
    node->debug_id = ++binder_last_id;
    node->proc = proc;
    node->ptr = ptr;
    node->cookie = cookie;
        //设置binder_work的type
    node->work.type = BINDER_WORK_NODE;
    INIT_LIST_HEAD(&node->work.entry);
    INIT_LIST_HEAD(&node->async_todo);
    binder_debug(BINDER_DEBUG_INTERNAL_REFS,
             "%d:%d node %d u%016llx c%016llx created\n",
             proc->pid, current->pid, node->debug_id,
             (u64)node->ptr, (u64)node->cookie);
    return node;
}
```

在Binder驱动层创建了binder_node结构体对象，并将当前的binder_pro加入到binder_node的node->proc。并创建binder_node的async_todo和binder_work两个队列

#### 1.3、binder_loop()详解

```cpp
 // framework/native/cmds/servicemanager/binder.c    372行
    void binder_loop(struct binder_state *bs, binder_handler func) {
        int res;
        struct binder_write_read bwr;
        uint32_t readbuf[ 32];

        bwr.write_size = 0;
        bwr.write_consumed = 0;
        bwr.write_buffer = 0;

        readbuf[0] = BC_ENTER_LOOPER;
        //将BC_ENTER_LOOPER命令发送给Binder驱动，让ServiceManager进行循环
        binder_write(bs, readbuf, sizeof(uint32_t));

        for (; ; ) {
            bwr.read_size = sizeof(readbuf);
            bwr.read_consumed = 0;
            bwr.read_buffer = (uintptr_t) readbuf;
            //进入循环，不断地binder读写过程
            res = ioctl(bs -> fd, BINDER_WRITE_READ, & bwr);

            if (res < 0) {
                ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
                break;
            }
            //解析binder信息
            res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
            if (res == 0) {
                ALOGE("binder_loop: unexpected reply?!\n");
                break;
            }
            if (res < 0) {
                ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
                break;
            }
        }
    }
```

进入循环读写操作，由main()方法传递过来的参数func指向svcmgr_handler。binder_write通过ioctl()将BC_ENTER_LOOPER命令发送给binder驱动，此时bwr只有write_buffer有数据，进入binder_thread_write()方法。 接下来进入for循环，执行ioctl()，此时bwr只有read_buffer有数据，那么进入binder_thread_read()方法。

主要是循环读写操作，这里有3个重点是

> - binder_thread_write结构体
> - binder_write函数
> - binder_parse函数

**1.3.1 binder_thread_write**

```cpp
//kernel/drivers/android/binder.c    2248行
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    struct binder_context *context = proc->context;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr < end && thread->return_error == BR_OK) {
        //获取命令
        get_user(cmd, (uint32_t __user *)ptr); 
        switch (cmd) {
              //**** 省略部分代码 ****
             case BC_ENTER_LOOPER:
             //设置该线程的looper状态
             thread->looper |= BINDER_LOOPER_STATE_ENTERED;
             break;
             //**** 省略部分代码 ****
    }
       //**** 省略部分代码 ****
    return 0;
}
```

主要是从bwr.write_buffer中拿出数据，此处为BC_ENTER_LOOPER，可见上层调用binder_write()方法主要是完成当前线程的looper状态为BINDER_LOOPER_STATE_ENABLE。

**1.3.2、 binder_write函数**

这块的函数在

```cpp
    // framework/native/cmds/servicemanager/binder.c     151行
    int binder_write(struct binder_state *bs, void *data, size_t len) {
        struct binder_write_read bwr;
        int res;

        bwr.write_size = len;
        bwr.write_consumed = 0;
        //此处data为BC_ENTER_LOOPER
        bwr.write_buffer = (uintptr_t) data;
        bwr.read_size = 0;
        bwr.read_consumed = 0;
        bwr.read_buffer = 0;
        res = ioctl(bs -> fd, BINDER_WRITE_READ, & bwr);
        if (res < 0) {
            fprintf(stderr, "binder_write: ioctl failed (%s)\n",
                    strerror(errno));
        }
        return res;
    }
```

根据传递进来的参数，初始化bwr，其中write_size大小为4,write_buffer指向缓冲区的起始地址，其内容为BC_ENTER_LOOPER请求协议号。通过ioctl将bwr数据发送给Binder驱动，则调用binder_ioctl函数

**1.3.3让我们来看下binder_ioctl函数**

```cpp
//kernel/drivers/android/binder.c     3239行
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
      //**** 省略部分代码 ****
     //获取binder_thread
    thread = binder_get_thread(proc); 
    switch (cmd) {
      case BINDER_WRITE_READ:  
          //进行binder的读写操作
          ret = binder_ioctl_write_read(filp, cmd, arg, thread); 
          if (ret)
              goto err;
          break;
          //**** 省略部分代码 ****
    }
}  
```

主要就是根据参数 BINDER_SET_CONTEXT_MGR，最终调用binder_ioctl_set_ctx_mgr()方法，这个过程会持有binder_main_lock。

**binder_ioctl_write_read()函数解析**

```objectivec
//kernel/drivers/android/binder.c    3134
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    if (size != sizeof(struct binder_write_read)) {
        ret = -EINVAL;
        goto out;
    }
        //把用户空间数据ubuf拷贝到bwr中
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
    binder_debug(BINDER_DEBUG_READ_WRITE,
             "%d:%d write %lld at %016llx, read %lld at %016llx\n",
             proc->pid, thread->pid,
             (u64)bwr.write_size, (u64)bwr.write_buffer,
             (u64)bwr.read_size, (u64)bwr.read_buffer);
        // “写缓存” 有数据
    if (bwr.write_size > 0) {
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        trace_binder_write_done(ret);
        if (ret < 0) {
            bwr.read_consumed = 0;
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
        // "读缓存" 有数据
    if (bwr.read_size > 0) {
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
                     bwr.read_size,
                     &bwr.read_consumed,
                     filp->f_flags & O_NONBLOCK);
        trace_binder_read_done(ret);
        if (!list_empty(&proc->todo))
            wake_up_interruptible(&proc->wait);
        if (ret < 0) {
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    binder_debug(BINDER_DEBUG_READ_WRITE,
             "%d:%d wrote %lld of %lld, read return %lld of %lld\n",
             proc->pid, thread->pid,
             (u64)bwr.write_consumed, (u64)bwr.write_size,
             (u64)bwr.read_consumed, (u64)bwr.read_size);
         //将内核数据bwr拷贝到用户控件bufd
    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}
```

此处代码就一个作用：就是讲用户空间的binder_write_read结构体 拷贝到内核空间。

**1.3.3、 binder_parse函数解析**

binder_parse在// framework/native/cmds/servicemanager/binder.c中

```cpp
  // framework/native/cmds/servicemanager/binder.c    204行
 int binder_parse(struct binder_state *bs, struct binder_io *bio,
                     uintptr_t ptr, size_t size, binder_handler func) {
        int r = 1;
        uintptr_t end = ptr + (uintptr_t) size;

        while (ptr < end) {
            uint32_t cmd = *(uint32_t *) ptr;
            ptr += sizeof(uint32_t);
            #if TRACE
            fprintf(stderr, "%s:\n", cmd_name(cmd));
            #endif
            switch (cmd) {
                case BR_NOOP:
                    //误操作，退出循环
                    break;
                case BR_TRANSACTION_COMPLETE:
                    break;
                case BR_INCREFS:
                case BR_ACQUIRE:
                case BR_RELEASE:
                case BR_DECREFS:
                    #if TRACE
                    fprintf(stderr, "  %p, %p\n", (void *)ptr, (void *)(ptr + sizeof(void *)));
                    #endif
                    ptr += sizeof(struct binder_ptr_cookie);
                    break;
                case BR_TRANSACTION: {
                    struct binder_transaction_data *txn = (struct binder_transaction_data *)ptr;
                    if ((end - ptr) < sizeof( * txn)){
                        ALOGE("parse: txn too small!\n");
                        return -1;
                    }
                    binder_dump_txn(txn);
                    if (func) {
                        unsigned rdata[ 256 / 4];
                        struct binder_io msg;
                        struct binder_io reply;
                        int res;

                        bio_init( & reply, rdata, sizeof(rdata), 4);
                        bio_init_from_txn( & msg, txn);
                        res = func(bs, txn, & msg, &reply);
                        binder_send_reply(bs, & reply, txn -> data.ptr.buffer, res);
                    }
                    ptr += sizeof( * txn);
                    break;
                }
                case BR_REPLY: {
                    struct binder_transaction_data *txn = (struct binder_transaction_data *)ptr;
                    if ((end - ptr) < sizeof( * txn)){
                        ALOGE("parse: reply too small!\n");
                        return -1;
                    }
                    binder_dump_txn(txn);
                    if (bio) {
                        bio_init_from_txn(bio, txn);
                        bio = 0;
                    } else {
                                        /* todo FREE BUFFER */
                    }
                    ptr += sizeof( * txn);
                    r = 0;
                    break;
                }
                case BR_DEAD_BINDER: {
                    struct binder_death *death = (struct binder_death *)
                    (uintptr_t) * (binder_uintptr_t *) ptr;
                    ptr += sizeof(binder_uintptr_t);
                    //binder死亡消息
                    death -> func(bs, death -> ptr);
                    break;
                }
                case BR_FAILED_REPLY:
                    r = -1;
                    break;
                case BR_DEAD_REPLY:
                    r = -1;
                    break;
                default:
                    ALOGE("parse: OOPS %d\n", cmd);
                    return -1;
            }
        }
        return r;
    }
```

主要是解析binder消息，此处参数ptr指向BC_ENTER_LOOPER，func指向svcmgr_handler，所以有请求来，则调用svcmgr

这里面我们重点分析BR_TRANSACTION里面的几个函数

> - bio_init()函数
> - bio_init_from_txn()函数

**1.3.3.1 bio_init()函数**

```rust
    // framework/native/cmds/servicemanager/binder.c      409行
    void bio_init_from_txn(struct binder_io *bio, struct binder_transaction_data *txn)
    {
        bio->data = bio->data0 = (char *)(intptr_t)txn->data.ptr.buffer;
        bio->offs = bio->offs0 = (binder_size_t *)(intptr_t)txn->data.ptr.offsets;
        bio->data_avail = txn->data_size;
        bio->offs_avail = txn->offsets_size / sizeof(size_t);
        bio->flags = BIO_F_SHARED;
    }
```

其中binder_io的结构体在 /frameworks/native/cmds/servicemanager/binder.h 里面
 [binder.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/cmds/servicemanager/binder.h)

```cpp
//frameworks/native/cmds/servicemanager/binder.h     12行
struct binder_io
{
    char *data;            /* pointer to read/write from */
    binder_size_t *offs;   /* array of offsets */
    size_t data_avail;     /* bytes available in data buffer */
    size_t offs_avail;     /* entries available in offsets array */

    char *data0;           //data buffer起点位置
    binder_size_t *offs0;  //buffer偏移量的起点位置
    uint32_t flags;
    uint32_t unused;
};
```

**1.3.3.2 bio_init_from_txn()函数**

```rust
// framework/native/cmds/servicemanager/binder.c    409行
void bio_init_from_txn(struct binder_io *bio, struct binder_transaction_data *txn)
{
    bio->data = bio->data0 = (char *)(intptr_t)txn->data.ptr.buffer;
    bio->offs = bio->offs0 = (binder_size_t *)(intptr_t)txn->data.ptr.offsets;
    bio->data_avail = txn->data_size;
    bio->offs_avail = txn->offsets_size / sizeof(size_t);
    bio->flags = BIO_F_SHARED;
}
```

其实很简单，就是将readbuf的数据赋给bio对象的data
 将readbuf的数据赋给bio对象的data

\####### 1.3.4 svcmgr_handler



```cpp
 //service_manager.c    244行
int svcmgr_handler(struct binder_state*bs,
                       struct binder_transaction_data*txn,
                       struct binder_io*msg,
                       struct binder_io*reply) {
        struct svcinfo*si;
        uint16_t * s;
        size_t len;
        uint32_t handle;
        uint32_t strict_policy;
        int allow_isolated;

        if (txn -> target.ptr != BINDER_SERVICE_MANAGER)
            return -1;

        if (txn -> code == PING_TRANSACTION)
            return 0;


        strict_policy = bio_get_uint32(msg);
        s = bio_get_string16(msg, & len);
        if (s == NULL) {
            return -1;
        }

        if ((len != (sizeof(svcmgr_id) / 2)) ||
                memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
            fprintf(stderr, "invalid id %s\n", str8(s, len));
            return -1;
        }

        if (sehandle && selinux_status_updated() > 0) {
            struct selabel_handle*tmp_sehandle = selinux_android_service_context_handle();
            if (tmp_sehandle) {
                selabel_close(sehandle);
                sehandle = tmp_sehandle;
            }
        }

        switch (txn -> code) {
            case SVC_MGR_GET_SERVICE:
            case SVC_MGR_CHECK_SERVICE:
                //获取服务名
                s = bio_get_string16(msg, & len);
                if (s == NULL) {
                    return -1;
                }
                //根据名称查找相应服务 
                handle = do_find_service(bs, s, len, txn -> sender_euid, txn -> sender_pid);
                if (!handle)
                    break;
                bio_put_ref(reply, handle);
                return 0;

            case SVC_MGR_ADD_SERVICE:
                //获取服务名
                s = bio_get_string16(msg, & len);
                if (s == NULL) {
                    return -1;
                }
                handle = bio_get_ref(msg);
                allow_isolated = bio_get_uint32(msg) ? 1 : 0;
                 //注册服务
                if (do_add_service(bs, s, len, handle, txn -> sender_euid,
                        allow_isolated, txn -> sender_pid))
                    return -1;
                break;

            case SVC_MGR_LIST_SERVICES: {
                uint32_t n = bio_get_uint32(msg);

                if (!svc_can_list(txn -> sender_pid)) {
                    ALOGE("list_service() uid=%d - PERMISSION DENIED\n",
                            txn -> sender_euid);
                    return -1;
                }
                si = svclist;
                while ((n-- > 0) && si)
                    si = si -> next;
                if (si) {
                    bio_put_string16(reply, si -> name);
                    return 0;
                }
                return -1;
            }
            default:
                ALOGE("unknown code %d\n", txn -> code);
                return -1;
        }
        bio_put_uint32(reply, 0);
        return 0;
    }
```

代码看着很多，其实主要就是servicemanger提供查询服务和注册服务以及列举所有服务。
 这里提一下svcinfo



```cpp
 //service_manager.c    128行
    struct svcinfo
    {
        struct svcinfo*next;
        uint32_t handle;
        struct binder_death death;
        int allow_isolated;
        size_t len;
        uint16_t name[ 0];
    };
```

每一个服务用svcinfo结构体来表示，该handle值是注册服务的过程中，又服务所在进程那一端所确定。

**1.3.4 总结**

ServiceManager集中管理系统内的所有服务，通过权限控制进程是否有权注册服务，通过字符串名称来查找对应的Service；由于ServiceManager进程建立跟所有向其注册服务的死亡通知，那么当前服务所在进程死亡后，会只需要告知ServiceManager。每个Client通过查询ServiceManager可获取Service进程的情况，降低所有Client进程直接检测导致负载过重。

让我们再次看这张图

![img](https:////upload-images.jianshu.io/upload_images/5713484-b04df9be2959a66d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

启动整体流程图.png

ServiceManager 启动流程：

> - 打开binder驱动，并调用mmap()方法分配128k内存映射空间：binder_open()
> - 通知binder驱动使其成为守护进程：binder_become_context_manager()；
> - 验证selinux权限，判断进程是否有权注册或查看指定服务;
> - 进入循环状态，等待Client端的请求
> - 注册服务的过程，根据服务的名称，但同一个服务已注册，然后调用binder_node_release。这个过程便会发出死亡通知的回调。

## 二、ServiceManager的核心服务

通过上面的代码我们知道service manager的核心服务主要有4个

> - do_add_service()函数：注册服务
> - do_find_service()函数：查找服务
> - binder_link_to_death()函数：结束服务
> - binder_send_reply()函数：将注册结果返回给Binder驱动

下面我们就挨个讲解一下

### (一)、do_add_service()函数

```cpp
//service_manager.c      194行
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid)
{
    struct svcinfo *si;

    if (!handle || (len == 0) || (len > 127))
        return -1;

    //权限检查
    if (!svc_can_register(s, len, spid)) {
        return -1;
    }

    //服务检索
    si = find_svc(s, len);
    if (si) {
        if (si->handle) {
            svcinfo_death(bs, si); //服务已注册时，释放相应的服务
        }
        si->handle = handle;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) { 
           //内存不足，无法分配足够内存
            return -1;
        }
        si->handle = handle;
        si->len = len;
        //内存拷贝服务信息
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        // svclist保存所有已注册的服务
        si->next = svclist; 
        svclist = si;
    }

    //以BC_ACQUIRE命令，handle为目标的信息，通过ioctl发送给binder驱动
    binder_acquire(bs, handle);
    //以BC_REQUEST_DEATH_NOTIFICATION命令的信息，通过ioctl发送给binder驱动，主要用于清理内存等收尾工作。
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```

注册服务部分主要分块内容：

> - svc_can_register：检查权限：检查selinux权限是否满足
> - find_svc：服务检索，根据服务名来查询匹配的服务；
> - svcinfo_death：释放服务，当查询到已存在的同名的服务，则先清理该服务信息，再讲当前的服务加入到服务列表svclist;

svc_can_register：检查权限，检查selinux权限是否满足；
 find_svc：服务检索，根据服务名来查询匹配的服务；
 svcinfo_death：释放服务，当查询到已存在同名的服务，则先清理该服务信息，再将当前的服务加入到服务列表svclist；

**1、svc_can_register()函数**

```cpp
//service_manager.c      110行
static int svc_can_register(const uint16_t *name, size_t name_len, pid_t spid)
{
    const char *perm = "add";
    //检查selinux权限是否满足
    return check_mac_perms_from_lookup(spid, perm, str8(name, name_len)) ? 1 : 0;
}
```

**2、svcinfo_death()函数**

```rust
//service_manager.c      153行
void svcinfo_death(struct binder_state *bs, void *ptr)
{
    struct svcinfo *si = (struct svcinfo* ) ptr;

    if (si->handle) {
        binder_release(bs, si->handle);
        si->handle = 0;
    }
}
```

**3、bio_get_ref()函数**

```cpp
// framework/native/cmds/servicemanager/binder.c     627行
uint32_t bio_get_ref(struct binder_io *bio)
{
    struct flat_binder_object *obj;

    obj = _bio_get_obj(bio);
    if (!obj)
        return 0;

    if (obj->type == BINDER_TYPE_HANDLE)
        return obj->handle;

    return 0;
}
```

### (二)、do_find_service()

```cpp
 //service_manager.c      170行
uint32_t do_find_service(struct binder_state *bs, const uint16_t *s, size_t len, uid_t uid, pid_t spid)
{
    //具体查询相应的服务
    struct svcinfo *si = find_svc(s, len);
    if (!si || !si->handle) {
        return 0;
    }

    if (!si->allow_isolated) {
        uid_t appid = uid % AID_USER;
         //检查该服务是否允许孤立于进程而单独存在
        if (appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END) {
            return 0;
        }
    }
     //服务是否满足于查询条件
    if (!svc_can_find(s, len, spid)) {
        return 0;
    }
   /返回结点中的ptr，这个ptr是binder中对应的binder_ref.des
    return si->handle;
}
```

主要就是查询目标服务，并返回该服务所对应的handle

**1、find_svc()函数**

```cpp
 //service_manager.c      140行
struct svcinfo *find_svc(const uint16_t *s16, size_t len)
{
    struct svcinfo *si;
    for (si = svclist; si; si = si->next) {
        //当名字完全一致，则返回查询到的结果
        if ((len == si->len) &&
            !memcmp(s16, si->name, len * sizeof(uint16_t))) {
            return si;
        }
    }
    return NULL;
}
```

在svclist服务列表中，根据服务名遍历查找是否已经注册。当服务已经存在svclist，则返回相应的服务名，否则返回null。

> 当找到服务的handle，则调用bio_put_ref(reply,handle)，将handle封装到reply。

在svcmgr_handler中当执行完do_find_service()函数后，会调用bio_put_ref()函数，让我们来一起研究下这个函数

**2、bio_put_ref()函数**

```cpp
// framework/native/cmds/servicemanager/binder.c    505行
void bio_put_ref(struct binder_io *bio, uint32_t handle)
{
    //构造了一个flat_binder_object
    struct flat_binder_object *obj;
    if (handle)
        obj = bio_alloc_obj(bio); 
    else
        obj = bio_alloc(bio, sizeof(*obj));
    if (!obj)
        return;
    obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    obj->type = BINDER_TYPE_HANDLE; //返回的是HANDLE类型
    //以service manager的身份回应给kernel driver，ptr就是handler对应的ref索引值 1，2，3，4，5，6等
    obj->handle = handle;
    obj->cookie = 0;
}
```

这个段代码也不复杂，就是根据handle来判断分别执行bio_alloc_obj()函数和bio_alloc()函数
 那我们就来好好研究和两个函数

**3、bio_alloc_obj()函数**

```rust
// framework/native/cmds/servicemanager/binder.c   468 行 
static struct flat_binder_object *bio_alloc_obj(struct binder_io *bio)
{
    struct flat_binder_object *obj;
    obj = bio_alloc(bio, sizeof(*obj));//[见小节3.1.4]

    if (obj && bio->offs_avail) {
        bio->offs_avail--;
        *bio->offs++ = ((char*) obj) - ((char*) bio->data0);
        return obj;
    }
    bio->flags |= BIO_F_OVERFLOW;
    return NULL;
}
```

**4、bio_alloc()函数**

```rust
// framework/native/cmds/servicemanager/binder.c   437 行 
static void *bio_alloc(struct binder_io *bio, size_t size)
{
    size = (size + 3) & (~3);
    if (size > bio->data_avail) {
        bio->flags |= BIO_F_OVERFLOW;
        return NULL;
    } else {
        void *ptr = bio->data;
        bio->data += size;
        bio->data_avail -= size;
        return ptr;
    }
}
```

### (三) 、 binder_link_to_death() 函数

```cpp
// framework/native/cmds/servicemanager/binder.c        305行
void binder_link_to_death(struct binder_state *bs, uint32_t target, struct binder_death *death)
{
    struct {
        uint32_t cmd;
        struct binder_handle_cookie payload;
    } __attribute__((packed)) data;

    data.cmd = BC_REQUEST_DEATH_NOTIFICATION;
    data.payload.handle = target;
    data.payload.cookie = (uintptr_t) death;
    binder_write(bs, &data, sizeof(data)); //[见小节3.3.1]
}
```

binder_write和前面的binder_write一样，进入Binder driver后，直接调用binder_thread_write，处理BC_REQUEST_DEATH_NOTIFICATION命令。其中binder_ioctl_write_read()函数，上面已经讲解过了。这里就不详细讲解了

**1、 binder_thread_write() 函数**

```cpp
//kernel/drivers/android/binder.c    2248行
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    struct binder_context *context = proc->context;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr < end && thread->return_error == BR_OK) {
        //获取命令
        get_user(cmd, (uint32_t __user *)ptr); 
        switch (cmd) {
              //**** 省略部分代码 ****
              // 注册死亡通知
             case BC_REQUEST_DEATH_NOTIFICATION:
         case BC_CLEAR_DEATH_NOTIFICATION: { 
              uint32_t target;
            void __user *cookie;
            struct binder_ref *ref;
            struct binder_ref_death *death;
            //获取taget
            get_user(target, (uint32_t __user *)ptr); 
            ptr += sizeof(uint32_t);
            /获取death
            get_user(cookie, (void __user * __user *)ptr); /
            ptr += sizeof(void *);
            //拿到目标服务的binder_ref
            ref = binder_get_ref(proc, target); 
            if (cmd == BC_REQUEST_DEATH_NOTIFICATION) {
                 //已设死亡通知
                if (ref->death) {
                    break; 
                }
                death = kzalloc(sizeof(*death), GFP_KERNEL);
                INIT_LIST_HEAD(&death->work.entry);
                death->cookie = cookie;
                ref->death = death;
                //当目标服务所在进程已死，则发送死亡通知
                if (ref->node->proc == NULL) {
                    //当前线程为binder线程，则直接添加到当前线程的TODO队列
                    ref->death->work.type = BINDER_WORK_DEAD_BINDER;
                    if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
                        list_add_tail(&ref->death->work.entry, &thread->todo);
                    } else {
                        list_add_tail(&ref->death->work.entry, &proc->todo);
                        wake_up_interruptible(&proc->wait);
                    }
                }
            } else {
                ...
            }
        } break;
       //**** 省略部分代码 ****
    }
       //**** 省略部分代码 ****
    return 0;
}
```

此方法中的proc，thread都是指当前的servicemanager进程信息，此时TODO队列有数据，则进入binder_thread_read。

那么问题来了，哪些场景会向队列增加BINDER_WORK_READ_BINDER事物？那边是当binder所在进程死亡后，会调用binder_realse方法，然后调用binder_node_release这个过程便会发出死亡通知的回调。

**2、binder_thread_read() 函数**

```php
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  binder_uintptr_t binder_buffer, size_t size,
                  binder_size_t *consumed, int non_block)
    ...
    //只有当前线程todo队列为空，并且transaction_stack也为空，才会开始处于当前进程的事务
    if (wait_for_proc_work) {
        ...
        ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
    } else {
        ...
        ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    }
    //加锁
    binder_lock(__func__);
    if (wait_for_proc_work)
        //空闲的binder线程减1
        proc->ready_threads--; 
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;
    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;
        //从todo队列拿出前面放入的binder_work, 此时type为BINDER_WORK_DEAD_BINDER
        if (!list_empty(&thread->todo)) {
            w = list_first_entry(&thread->todo, struct binder_work,
                         entry);
        } else if (!list_empty(&proc->todo) && wait_for_proc_work) {
            w = list_first_entry(&proc->todo, struct binder_work,
                         entry);
        }

        switch (w->type) {
            case BINDER_WORK_DEAD_BINDER: {
              struct binder_ref_death *death;
              uint32_t cmd;

              death = container_of(w, struct binder_ref_death, work);
              if (w->type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION)
                  ...
              else
              //进入此分支
                  cmd = BR_DEAD_BINDER; 
              //拷贝用户空间
              put_user(cmd, (uint32_t __user *)ptr);
              ptr += sizeof(uint32_t);

              //此处的cookie是前面传递的svcinfo_death
              put_user(death->cookie, (binder_uintptr_t __user *)ptr);
              ptr += sizeof(binder_uintptr_t);

              if (w->type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION) {
                  ...
              } else
                  list_move(&w->entry, &proc->delivered_death);
              if (cmd == BR_DEAD_BINDER)
                  goto done;
            } break;
        }
    }
    ...
    return 0;
}
```

将命令BR_DEAD_BINDER写到用户空间，此处的cookie是前面传递的svcinfo_death。当binder_loop下一次执行binder_parse的过程便会处理该消息。  binder_parse()函数和svcinfo_death()函数上面已经说明了，这里就不详细说明了。

**3、 binder_release() 函数**

```cpp
//frameworks/native/cmds/servicemanager/binder.c   297行
void binder_release(struct binder_state *bs, uint32_t target)
{
    uint32_t cmd[2];
    cmd[0] = BC_RELEASE;
    cmd[1] = target;
    binder_write(bs, cmd, sizeof(cmd));
}
```

向Binder Driver写入BC_RELEASE命令，最终进入Binder Driver后执行binder_dec_ref(ref，1) 来减少binder node的引用。

### (四)、 binder_send_reply() 函数

```kotlin
//frameworks/native/cmds/servicemanager/binder.c   170行
    void binder_send_reply(struct binder_state *bs,
                           struct binder_io *reply,
                           binder_uintptr_t buffer_to_free,
                           int status) {
        struct {
            uint32_t cmd_free;
            binder_uintptr_t buffer;
            uint32_t cmd_reply;
            struct binder_transaction_data txn;
        } __attribute__((packed)) data;
        //free buffer命令
        data.cmd_free = BC_FREE_BUFFER;
        data.buffer = buffer_to_free;
        //replay命令
        data.cmd_reply = BC_REPLY;
        data.txn.target.ptr = 0;
        data.txn.cookie = 0;
        data.txn.code = 0;
        if (status) {
            data.txn.flags = TF_STATUS_CODE;
            data.txn.data_size = sizeof(int);
            data.txn.offsets_size = 0;
            data.txn.data.ptr.buffer = (uintptr_t) & status;
            data.txn.data.ptr.offsets = 0;
        } else {
            data.txn.flags = 0;
            data.txn.data_size = reply -> data - reply -> data0;
            data.txn.offsets_size = ((char*)reply -> offs)-((char*)reply -> offs0);
            data.txn.data.ptr.buffer = (uintptr_t) reply -> data0;
            data.txn.data.ptr.offsets = (uintptr_t) reply -> offs0;
        }
        //向Binder驱动通信
        binder_write(bs, & data, sizeof(data));
    }
```

执行binder_parse方法，先调用svcmgr_handler()函数，然后再执行binder_send_reply过程，该过程会调用binder_write进入binder驱动后，将BC_FREE_BUFFER和BC_REPLY命令协议发送给Binder驱动，向Client端发送reply，其中data数据区中保存的是TYPE为HANDLE。

现在我们对ServiceManager有个初步的了解，那么我们怎么才能得到ServiceManager那？下面就让我们来看下如何获得ServiceManager。

## 三、ServiceManager的获得

### (一)、源码信息

代码位于

```java
framework/native/libs/binder/
  - ProcessState.cpp
  - BpBinder.cpp
  - Binder.cpp
  - IServiceManager.cpp
framework/native/include/binder/
  - IServiceManager.h
  - IInterface.h
```

链接为

> - [ProcessState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/ProcessState.cpp)
> - [BpBinder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/BpBinder.cpp)
> - [Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Binder.cpp)
> - [IServiceManager.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IServiceManager.cpp)
> - [IServiceManager.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/include/binder/IServiceManager.h)
> - [IInterface.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/include/binder/IInterface.h)

**这里重点提醒下framework/native/libs/binder/IServiceManager.cpp和 framework/native/include/binder/IServiceManager.h大家千万不要弄混了。**

### (二)、获取Service Manager简述

> 获取Service Manager是通过defaultServiceManager()方法来完成的。当进程 **注册服务** 与 **获取服务**之前，都需要调用defaultServiceManager()方法来获取gDefaultServiceManager对象。对于gDefaultServiceManager对象，如果存在直接返回。如果不存在直接创建该对象，创建过程包括调用open()打开binder驱动设备，利用mmap()映射内核的地址空间。

### (三)、流程图

![img](https:////upload-images.jianshu.io/upload_images/5713484-44dee73939fb85ed.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

获取ServiceManager流程图.png

### (四)、获取defaultServiceManager

代码如下

```php
//frameworks/native/libs/binder/IServiceManager.cpp      33行
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
    {
         //加锁
        AutoMutex _l(gDefaultServiceManagerLock); 
        while (gDefaultServiceManager == NULL) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                //这里才是关键和重点
                ProcessState::self()->getContextObject(NULL));
            if (gDefaultServiceManager == NULL)
                sleep(1);
        }
    }
    return gDefaultServiceManager;
}
```

获取ServiceManager 对象采用单例模式，当gDefaultServiceManager存在，则直接返回，否则创建一个新对象。这里的创建单利模式和咱们之前的java里面的单例不一样。它里面多了一层while循环，这是谷歌在2013年1月Todd Poynor提交的修改。因为当第一次尝试创建获取ServiceManager时，ServiceManager可能还未准备就绪，所以通过sleep1秒，实现延迟1秒，然后尝试去获取直到成功。

而gDefualtServiceManager的创建过程又可以分解为3个步骤

> - ProcessState：：self() ：用于获取ProcessState对象(也是单例模式)，每个进程有且只有一个ProcessState对象，存在则直接返回，不存在则创建。
> - getContextObject()： 用于获取BpBinder对象，对于hanle=0的BpBinder对象，存在则直接返回，不存在则创建。
> - interface_cast<IServiceManager>()：用于获取BpServiceManager对象。

所以下面的  (五)(六)(七) 依次讲解ProcessState、BpBinder对象和BpServiceManager对象

### (五)、获取ProcessState对象

**1、ProcessState::self**

我们先来看下这块代码

```cpp
//frameworks/native/libs/binder/ProcessState.cpp   70行
// 这又是一个进程单体
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    //实例化 ProcessState，首次创建
    gProcess = new ProcessState;
    return gProcess;
}
```

获取ProcessState对象：这也是一个单利模式，从而保证每一个进程只有一个ProcessState对象，其中gProccess和gProccessMutex是保持在Static.cpp的类全局变量。

那我们来一起看下ProccessState的构造函数

**2、ProccessState的构造函数**

```cpp
//frameworks/native/libs/binder/ProcessState.cpp       339行
ProcessState::ProcessState()
    //这里打开了打开了Binder驱动，也就是/dev/binder文件，返回文件描述符
    : mDriverFD(open_driver())
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        //采用内存映射函数mmap，给binder分配一块虚拟地址空间，涌来了接收事物
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            //没有足够空间分配给/dev/binder，则关闭驱动。
            close(mDriverFD); 
            mDriverFD = -1;
        }
    }
}
```

通过上面的构造函数我们知道

> - ProcessState的单利模式的唯一性，因此一个进程只打开binder设备一次，其中ProcessState的成员变量mDriverFD记录binder驱动的fd，用于访问binder设备。
> - BINDER_VM_SIZE=(1*1024*1024- (4096*2))，所以binder分配的默认内存大小是1024*1016也就是1M-8K(1M减去8k)
> - DEFAULT_MAX_BINDER_THREAD=15，binder默认的最大可并发的线程数为16。

这里面调用了open_driver()方法，那么让我们研究下这个方法

**3、open_driver()方法**

```cpp
//frameworks/native/libs/binder/ProcessState.cpp       311行
static int open_driver()
{
    // 打开/dev/binder设备，建立与内核的Binder驱动的交互通道
    int fd = open("/dev/binder", O_RDWR);
    if (fd >= 0) {
        fcntl(fd, F_SETFD, FD_CLOEXEC);
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        if (result == -1) {
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
            close(fd);
            fd = -1;
        }
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;

        // 通过ioctl设置binder驱动，能支持的最大线程数
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
    }
    return fd;
}
```

open_driver的作用就是打开/dev/binder设备，设定binder支持的最大线程数。binder驱动相应的内容请看上一篇文章。

### (六)、获取BpBiner对象

**1、getContextObject()方法**

```cpp
//frameworks/native/libs/binder/ProcessState.cpp       85行
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);  
}
```

我们发现这里面什么都没做，就是调用getStrongProxyForHandle()方法，**大家注意它的入参写死为0**，然后我们继续深入

**2、getStrongProxyForHandle()方法**

注释有点长，我把注释删除了

```php
//frameworks/native/libs/binder/ProcessState.cpp       179行
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;
    AutoMutex _l(mLock);
    //查找handle对应的资源项
    handle_entry* e = lookupHandleLocked(handle);
    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                Parcel data;
                //通过ping操作测试binder是否已经准备就绪
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }
           //当handle值所对应的IBinder不存在或弱引用无效时，则创建BpBinder对象
            b = new BpBinder(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}
```

**大家注意上面函数的入参handle=0**

> 当handle值所对应的IBinder不存在或弱引用无效时会创建BpBinder，否则直接获取。针对hande==0的特殊情况，通过PING_TRANSACTION来判断是否准备就绪。如果在context manager还未生效前，一个BpBinder的本地引用就已经被创建，那么驱动将无法提供context manager的引用。

在getStrongProxyForHandle()方法里面先后调用了lookupHandleLocked()方法和创建BpBinder对象，那我们就来详细研究下

**3、lookupHandleLocked()方法**

```cpp
//frameworks/native/libs/binder/ProcessState.cpp      166行
ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
{
    const size_t N=mHandleToObject.size();
    //当handle大于mHandleToObject的长度时，进入该分支
    if (N <= (size_t)handle) {
        handle_entry e;
        e.binder = NULL;
        e.refs = NULL;
        //从mHandleToObject的第N个位置开始，插入(handle+1-N)个e到队列中
        status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
        if (err < NO_ERROR) return NULL;
    }
    return &mHandleToObject.editItemAt(handle);
}
```

根据handle值来查找对应的handle_entry,handle_entry是一个结构体，里面记录了IBinder和weakref_type两个指针。当handle大于mHandleToObject的Vector长度时，则向Vector中添加(handle+1-N)个handle_entry结构体，然后再返回handle向对应位置的handle_entry结构体指针。

**4、创建BpBinder**

```cpp
//frameworks/native/libs/binder/BpBinder.cpp       89行
BpBinder::BpBinder(int32_t handle)
    : mHandle(handle)
    , mAlive(1)
    , mObitsSent(0)
    , mObituaries(NULL)
{
    //延长对象的生命时间
    extendObjectLifetime(OBJECT_LIFETIME_WEAK); 
    // handle所对应的bindle弱引用+1
    IPCThreadState::self()->incWeakHandle(handle); 
}
```

创建BpBinder对象中将handle相对应的弱引用+1

### (七)、获取BpServiceManager对象

**1、interface_cast()函数**

```cpp
//frameworks/native/include/binder/IInterface.h  42行
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj); 
}
```

这是一个模板函数，可得出，interface_cast<IServiceManager>()等价于IServiceManager::asInterface()。接下来，再说说asInterface()函数的具体功能。

**2、IServiceManager::asInterface()函数**

对于asInterface()函数，通过搜索代码，你会发现根本找不到这个方法是在哪里定义这个函数的，其实是通过模板函数来定义的，通过下面两个代码完成的

```cpp
// 位于IServiceManager.h     33行
DECLARE_META_INTERFACE(ServiceManager)
//位于IServiceManager.cpp    108行
IMPLEMENT_META_INTERFACE(ServiceManager,"android.os.IServiceManager")
```

那我们就来重点说下这两块代码的功能

**3、DECLARE_META_INTERFACE**

```php
//framework/native/include/binder/IInterface.h      74行
#define DECLARE_META_INTERFACE(INTERFACE)                               
   static const android::String16 descriptor;                          
   static android::sp<I##INTERFACE> asInterface(                       
          const android::sp<android::IBinder>& obj);                  
   virtual const android::String16& getInterfaceDescriptor() const;    
   I##INTERFACE();                                                     
   virtual ~I##INTERFACE();       
```

位于IServiceManager.h文件中,INTERFACE=ServiceManager展开即可得：

```rust
static const android::String16 descriptor;

static android::sp< IServiceManager > asInterface(const android::sp<android::IBinder>& obj)

virtual const android::String16& getInterfaceDescriptor() const;

IServiceManager ();
virtual ~IServiceManager();
```

该过程主要是声明asInterface()、getInterfaceDescriptor()方法。

**4、 IMPLEMENT_META_INTERFACE**

```php
//framework/native/include/binder/IInterface.h      83行
#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    const android::String16 I##INTERFACE::descriptor(NAME);             \
    const android::String16&                                            \
            I##INTERFACE::getInterfaceDescriptor() const {              \
        return I##INTERFACE::descriptor;                                \
    }                                                                   \
    android::sp<I##INTERFACE> I##INTERFACE::asInterface(                \
            const android::sp<android::IBinder>& obj)                   \
    {                                                                   \
        android::sp<I##INTERFACE> intr;                                 \
        if (obj != NULL) {                                              \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == NULL) {                                         \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    I##INTERFACE::I##INTERFACE() { }                                    \
    I##INTERFACE::~I##INTERFACE() { }                                   \
```

位于IServiceManager.cpp文件中，INTERFACE=ServiceManager，NAME="android.os.IServiceManager" 开展即可得：



```cpp
const 
 android::String16 
 IServiceManager::descriptor(“android.os.IServiceManager”);

const android::String16& IServiceManager::getInterfaceDescriptor() const
{
     return IServiceManager::descriptor;
}

 android::sp<IServiceManager> IServiceManager::asInterface(const android::sp<android::IBinder>& obj)
{
       android::sp<IServiceManager> intr;
        if(obj != NULL) {
           intr = static_cast<IServiceManager *>(
               obj->queryLocalInterface(IServiceManager::descriptor).get());
           if (intr == NULL) {
               intr = new BpServiceManager(obj);  //【见小节4.5】
            }
        }
       return intr;
}
IServiceManager::IServiceManager () { }
IServiceManager::~ IServiceManager() { }
```

**不难发现，上面说的IServiceManager::asInterface() 等价于new BpServiceManager()。在这里，更确切地说应该是new   BpServiceManager(BpBinder)。**

**4.1、 BpServiceManager实例化**

```kotlin
//frameworks/native/libs/binder/IServiceManager.cpp    126行
class BpServiceManager : public BpInterface<IServiceManager>
{
public:
    BpServiceManager(const sp<IBinder>& impl)
        : BpInterface<IServiceManager>(impl)
    {
    }

    virtual sp<IBinder> getService(const String16& name) const
    {
        unsigned n;
        for (n = 0; n < 5; n++){
            sp<IBinder> svc = checkService(name);
            if (svc != NULL) return svc;
            ALOGI("Waiting for service %s...\n", String8(name).string());
            sleep(1);
        }
        return NULL;
            }

    virtual sp<IBinder> checkService( const String16& name) const
    {
        Parcel data, reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeString16(name);
        remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
        return reply.readStrongBinder();
    }
    virtual status_t addService(const String16& name, const sp<IBinder>& service,
            bool allowIsolated)
    {
        Parcel data, reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeString16(name);
        data.writeStrongBinder(service);
        data.writeInt32(allowIsolated ? 1 : 0);
        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
        return err == NO_ERROR ? reply.readExceptionCode() : err;
    }

    virtual Vector<String16> listServices()
    {
        Vector<String16> res;
        int n = 0;

        for (;;) {
            Parcel data, reply;
            data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
            data.writeInt32(n++);
            status_t err = remote()->transact(LIST_SERVICES_TRANSACTION, data, &reply);
            if (err != NO_ERROR)
                break;
            res.add(reply.readString16());
        }
        return res;
    }
};
```

创建BpServiceManager对象的过程，会先初始化父类对象：

**4.2、 BpServiceManager实例化**

```cpp
//frameworks/native/include/binder/IInterface.h     135行
template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
   public: BpInterface(const sp<IBinder>& remote);
   protected:  virtual IBinder*            onAsBinder();
};
```

**4.3、BpRefBase初始化**

```kotlin
BpRefBase::BpRefBase(const sp<IBinder>& o)
    : mRemote(o.get()), mRefs(NULL), mState(0)
{
    extendObjectLifetime(OBJECT_LIFETIME_WEAK);

    if (mRemote) {
        mRemote->incStrong(this);
        mRefs = mRemote->createWeak(this);
    }
}
```

new BpServiceManager()，在初始化过程中，比较重要的类BpRefBase的mRemote指向new BpBinder(0)，从而BpServiceManager能够利用Binder进行通信。

### (八) 模板函数

C层的Binder架构，通过下面的两个宏，非常方便地创建了**new Bp##INTERFACE(obj)**
 代码如下：

```cpp
// 用于申明asInterface()，getInterfaceDescriptor()
#define DECLARE_META_INTERFACE(INTERFACE) 
// 用于实现上述两个方法
#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME) 
```

例如:

```cpp
// 实现BpServiceManager对象
IMPLEMENT_META_INTERFACE(ServiceManager,"android.os.IServiceManager")
```

等价于:

```cpp
const android::String16 IServiceManager::descriptor(“android.os.IServiceManager”);
const android::String16& IServiceManager::getInterfaceDescriptor() const
{
     return IServiceManager::descriptor;
}

 android::sp<IServiceManager> IServiceManager::asInterface(const android::sp<android::IBinder>& obj)
{
       android::sp<IServiceManager> intr;
        if(obj != NULL) {
           intr = static_cast<IServiceManager *>(
               obj->queryLocalInterface(IServiceManager::descriptor).get());
           if (intr == NULL) {
               intr = new BpServiceManager(obj);
            }
        }
       return intr;
}

IServiceManager::IServiceManager () { }
IServiceManager::~ IServiceManager() { }
```

### (九) 总结

> - defaultServiceManager 等价于new BpServiceManager(new BpBinder(0));
> - ProcessState:: self() 主要工作： 
>   - 调用open，打/dev/binder驱动设备
>   - 调用mmap(),创建大小为  1016K的内存地址空间
>   - 设定当前进程最大的并发Binder线程个数为16
> - BpServiceManager巧妙将通信层与业务层逻辑合为一体，通过继承接口IServiceManager实现接口中的业务逻辑函数;通过成员变量mRemote=new BpBinder(0) 进行Binder通信工作。BpBinder通过handle来指向所对应的BBinder，在整个Binder系统总handle=0代表ServiceManager所对应的BBinder。



## 四、注册服务

### (一) 源码位置：



```java
framework/native/libs/binder/
  - Binder.cpp
  - BpBinder.cpp
  - IPCThreadState.cpp
  - ProcessState.cpp
  - IServiceManager.cpp
  - IInterface.cpp
  - Parcel.cpp

frameworks/native/include/binder/
  - IInterface.h (包括BnInterface, BpInterface)

/frameworks/av/media/mediaserver/
  - main_mediaserver.cpp

/frameworks/av/media/libmediaplayerservice/
  - MediaPlayerService.cpp
```

对应的链接为

> - [Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Binder.cpp)
> - [BpBinder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/BpBinder.cpp)
> - [IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
> - [ProcessState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/ProcessState.cpp)
> - [IServiceManager.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IServiceManager.cpp)
> - [IInterface.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IInterface.cpp)
> - [IInterface.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/include/binder/IInterface.h)
> - [main_mediaserver.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/av/media/mediaserver/main_mediaserver.cpp)
> - [MediaPlayerService.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp)

### (二)、概述

由于服务注册会涉及到具体的服务注册，网上大多数说的都是Media注册服务，我们也说它。

> media入口函数是 “main_mediaserver.cpp”中的main()方法，代码如下：



```rust
frameworks/av/media/mediaserver/main_mediaserver.cpp      44行
int main(int argc __unused, char** argv)
{
    *** 省略部分代码  *****
    InitializeIcuOrDie();
    // 获得ProcessState实例对象 
    sp<ProcessState> proc(ProcessState::self());
    //获取 BpServiceManager
    sp<IServiceManager> sm = defaultServiceManager();
    AudioFlinger::instantiate();
    //注册多媒体服务
    MediaPlayerService::instantiate();
    ResourceManagerService::instantiate();
    CameraService::instantiate();
    AudioPolicyService::instantiate();
    SoundTriggerHwService::instantiate();
    RadioService::instantiate();
    registerExtensions();
    //启动Binder线程池
    ProcessState::self()->startThreadPool();
    //当前线程加入到线程池
    IPCThreadState::self()->joinThreadPool();
 }
```

所以在main函数里面

> - 首先 获得了一个ProcessState的实例
> - 其次 调用defualtServiceManager方法获取IServiceManager实例
> - 再次 进行重要服务的初始化
> - 最后调用startThreadPool方法和joinThreadPool方法。

**PS: (1)获取ServiceManager：我们上篇文章讲解了defaultServiceManager()返回的是BpServiceManager对象，用于跟servicemanger进行通信。**

### (三)、类图

我们这里主要讲解的是Native层的服务，所以我们以native层的media为例，来说一说服务注册的过程，先来看看media的关系图

![img](https:////upload-images.jianshu.io/upload_images/5713484-1048ad1cfa1b87af.png?imageMogr2/auto-orient/strip|imageView2/2/w/1014/format/webp)

media类关系图.png

图解

> - 蓝色代表的是注册MediaPlayerService
> - 绿色代表的是Binder架构中与Binder驱动通信
> - 紫色代表的是注册服务和获取服务的公共接口/父类

### (四)、时序图

先通过一幅图来说说，media服务启动过程是如何向servicemanager注册服务的。

![img](https:////upload-images.jianshu.io/upload_images/5713484-dc5d238e4ce2b686.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

注册.png

### (五)、流程介绍

#### **1、inistantiate()函数**

```cpp
// MediaPlayerService.cpp     269行
void MediaPlayerService::instantiate() {
    defaultServiceManager()->addService(String16("media.player"), new MediaPlayerService());
}
```

> - 1 创建一个新的Service——BnMediaPlayerService，想把它告诉ServiceManager。然后调用BnServiceManager的addService的addService来向ServiceManager中添加一个Service，其他进程可以通过字符串"media.player"来向ServiceManager查询此服务。
> - 2 注册服务MediaPlayerService：由defaultServiceManager()返回的是BpServiceManager，同时会创建ProcessState对象和BpBinder对象。故此处等价于调用BpServiceManager->addService。

#### **2、BpSserviceManager.addService()函数**

```kotlin
/frameworks/native/libs/binder/IServiceManager.cpp   155行
virtual status_t addService(const String16& name, const sp<IBinder>& service,
        bool allowIsolated)
{
    //data是送到BnServiceManager的命令包
    Parcel data, reply; 
    //先把interface名字写进去，写入头信息"android.os.IServiceManager"
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());   
    // 再把新service的名字写进去 ，name为"media.player"
    data.writeString16(name);  
    // MediaPlayerService对象
    data.writeStrongBinder(service); 
     // allowIsolated= false
    data.writeInt32(allowIsolated ? 1 : 0);
    //remote()指向的BpServiceManager中保存的BpBinder
    status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
    return err == NO_ERROR ? reply.readExceptionCode() : err;
}
```

服务注册过程：向ServiceManager 注册服务MediaPlayerService，服务名为"media.player"。这样别的进程皆可以通过"media.player"来查询该服务

这里我们重点说下writeStrongBinder()函数和最后的transact()函数

**2.1、writeStrongBinder()函数**

```kotlin
/frameworks/native/libs/binder/Parcel.cpp        872行
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}
```

里面调用flatten_binder()函数，那我们继续跟踪

**2.1.1、 flatten_binder()函数**

```cpp
/frameworks/native/libs/binder/Parcel.cpp        205行
status_t flatten_binder(const sp<ProcessState>& /*proc*/,
    const sp<IBinder>& binder, Parcel* out)
{
    flat_binder_object obj;

    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
     //本地Binder不为空
    if (binder != NULL) {
        IBinder *local = binder->localBinder(); 
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.type = BINDER_TYPE_HANDLE; 
            obj.binder = 0; 
            obj.handle = handle;
            obj.cookie = 0;
        } else { 
            // 进入该分支
            obj.type = BINDER_TYPE_BINDER; 
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    } else {
        ...
    }
    return finish_flatten_binder(binder, obj, out);
}
```

其实是将Binder对象扁平化，转换成flat_binder_object对象

> - 对于Binder实体，则用cookie记录binder实体的指针。
> - 对于Binder代理，则用handle记录Binder代理的句柄。

关于localBinder，代码如下：

```cpp
//frameworks/native/libs/binder/Binder.cpp      191行
BBinder* BBinder::localBinder()
{
    return this;
}

//frameworks/native/libs/binder/Binder.cpp      47行
BBinder* IBinder::localBinder()
{
    return NULL;
}
```

上面 最后又调用了finish_flatten_binder()让我们一起来看下

**2.1.1、 finish_flatten_binder()函数**

```csharp
//frameworks/native/libs/binder/Parcel.cpp        199行
inline static status_t finish_flatten_binder(
    const sp<IBinder>& , const flat_binder_object& flat, Parcel* out)
{
    return out->writeObject(flat, false);
}
```

**2.2、 transact()函数**

```cpp
//frameworks/native/libs/binder/BpBinder.cpp       159行
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    if (mAlive) {
        // code=ADD_SERVICE_TRANSACTION
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
    return DEAD_OBJECT;
}
```

Binder代理类调用transact()方法，真正的工作还是交给IPCThreadState来进行transact工作，先来，看见IPCThreadState:: self的过程。

Binder代理类调用transact()方法，真正工作还是交给IPCThreadState来进行transact工作。先来 看看IPCThreadState::self的过程。

**2.2.1、IPCThreadState::self()函数**

```cpp
//frameworks/native/libs/binder/IPCThreadState.cpp     280行
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        // new 了一个  IPCThreadState对象
        return new IPCThreadState; 
    }

    if (gShutdown) return NULL;

    pthread_mutex_lock(&gTLSMutex);
     //首次进入gHaveTLS为false
    if (!gHaveTLS) { 
        // 创建线程的TLS
        if (pthread_key_create(&gTLS, threadDestructor) != 0) { 
            pthread_mutex_unlock(&gTLSMutex);
            return NULL;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}
```

> TLS 是指Thread local storage(线程本地存储空间)，每个线程都拥有自己的TLS，并且是私有空间，线程空间是不会共享的。通过pthread_getspecific/pthread_setspecific函数可以设置这些空间中的内容。从线程本地存储空间中获得保存在其中的IPCThreadState对象。

说到 IPCThreadState对象，我们就来看看它的构造函数

**2.2.1、IPCThreadState的构造函数**

```rust
//frameworks/native/libs/binder/IPCThreadState.cpp     686行  
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mMyThreadId(gettid()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}
```

每个线程都有一个IPCThreadState，每个IPCThreadState中都有一个mIn，一个mOut。成员变量mProcess保存了ProccessState变量(每个进程只有一个)

> - mIn：用来接收来自Binder设备的数据，默认大小为256字节
> - mOut：用来存储发往Binder设备的数据，默认大小为256字节

**2.2.2、IPCThreadState::transact()函数**

```cpp
//frameworks/native/libs/binder/IPCThreadState.cpp     548行
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    //数据错误检查
    status_t err = data.errorCheck(); 
    flags |= TF_ACCEPT_FDS;
    //  ***  省略部分代码  ***
    if (err == NO_ERROR) {
        //传输数据
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    //  ***  省略部分代码  ***
    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            //等待响应 
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        //one waitForReponse(NULL,NULL)
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
```

IPCThreadState进行trancsact事物处理3部分：

> - errorCheck()  ：负责 数据错误检查
> - writeTransactionData()： 负责 传输数据
> - waitForResponse()： 负责 等待响应

那我们重点看下writeTransactionData()函数与waitForResponse()函数

**2.2.2.1、writeTransactionData)函数**

```cpp
//frameworks/native/libs/binder/IPCThreadState.cpp      904行
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags, int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer) 
{
            binder_transaction_data tr;
            tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */ 
            // handle=0
            tr.target.handle = handle;
             //code=ADD_SERVICE_TRANSACTION
            tr.code = code;
            // binderFlags=0
            tr.flags = binderFlags;
            tr.cookie = 0;
            tr.sender_pid = 0;
            tr.sender_euid = 0;
            // data为记录Media服务信息的Parcel对象
            const status_t err = data.errorCheck();
            if (err == NO_ERROR) {
                    tr.data_size = data.ipcDataSize();
                    tr.data.ptr.buffer = data.ipcData();
                    tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
                    tr.data.ptr.offsets = data.ipcObjects();
               } else if (statusBuffer) {
                  tr.flags |= TF_STATUS_CODE;
                    *statusBuffer = err;
                    tr.data_size = sizeof(status_t);
                    tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
                   tr.offsets_size = 0;
                    tr.data.ptr.offsets = 0;
              } else {
                   return (mLastError = err);
                }
           // cmd=BC_TRANSACTION
           mOut.writeInt32(cmd);
          // 写入binder_transaction_data数据
           mOut.write(&tr, sizeof(tr));
           return NO_ERROR;
        }
```

其中handle的值用来标示目的端，注册服务过程的目的端为service manager，此处handle=0所对应的是binder_context_mgr_node对象，正是service manager所对应的binder实体对象。其中 binder_transaction_data结构体是binder驱动通信的数据结构，该过程最终是把Binder请求码BC_TRANSACTION和binder_transaction_data写入mOut。

transact的过程，先写完binder_transaction_data数据，接下来执行waitForResponse。

**2.2.2.2、waitForResponse()函数**

```cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult) {
        uint32_t cmd;
        int32_t err;

        while (1) {
            if ((err = talkWithDriver()) < NO_ERROR) break;
            err = mIn.errorCheck();
            if (err < NO_ERROR) break;
            if (mIn.dataAvail() == 0) continue;

            cmd = (uint32_t) mIn.readInt32();

            IF_LOG_COMMANDS() {
                alog << "Processing waitForResponse Command: "
                        << getReturnString(cmd) << endl;
            }

            switch (cmd) {
                case BR_TRANSACTION_COMPLETE:
                    if (!reply && !acquireResult) goto finish;
                    break;

                case BR_DEAD_REPLY:
                    err = DEAD_OBJECT;
                                goto finish;

                case BR_FAILED_REPLY:
                    err = FAILED_TRANSACTION;
                                goto finish;

                case BR_ACQUIRE_RESULT: {
                    ALOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");
                                    const int32_t result = mIn.readInt32();
                    if (!acquireResult) continue;
                                   *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
                }
                            goto finish;

                case BR_REPLY: {
                    binder_transaction_data tr;
                    err = mIn.read( & tr, sizeof(tr));
                    ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                    if (err != NO_ERROR) goto finish;

                    if (reply) {
                        if ((tr.flags & TF_STATUS_CODE) == 0) {
                            reply -> ipcSetDataReference(
                                    reinterpret_cast <const uint8_t * > (tr.data.ptr.buffer),
                                    tr.data_size,
                                    reinterpret_cast <const binder_size_t * > (tr.data.ptr.offsets),
                                    tr.offsets_size / sizeof(binder_size_t),
                                    freeBuffer, this);
                        } else {
                            err = *reinterpret_cast<const status_t * > (tr.data.ptr.buffer);
                            freeBuffer(NULL,
                                    reinterpret_cast <const uint8_t * > (tr.data.ptr.buffer),
                                    tr.data_size,
                                    reinterpret_cast <const binder_size_t * > (tr.data.ptr.offsets),
                                    tr.offsets_size / sizeof(binder_size_t), this);
                        }
                    } else {
                        freeBuffer(NULL,
                                reinterpret_cast <const uint8_t * > (tr.data.ptr.buffer),
                                tr.data_size,
                                reinterpret_cast <const binder_size_t * > (tr.data.ptr.offsets),
                                tr.offsets_size / sizeof(binder_size_t), this);
                        continue;
                    }
                }
                            goto finish;

                default:
                    err = executeCommand(cmd);
                    if (err != NO_ERROR) goto finish;
                    break;
            }
        }

        finish:
        if (err != NO_ERROR) {
            if (acquireResult) *acquireResult = err;
            if (reply) reply -> setError(err);
            mLastError = err;
        }
        return err;
    }
```

在waitForResponse过程，首先执行BR_TRANSACTION_COMPLETE；另外，目标进程收到事物后，处理BR_TRANSACTION事物，然后送法给当前进程，再执行BR_REPLY命令。

这里详细说下talkWithDriver()函数

**2.2.2.3、talkWithDriver()函数**

```cpp
    status_t IPCThreadState::talkWithDriver(bool doReceive) {
        if (mProcess -> mDriverFD <= 0) {
            return -EBADF;
        }

        binder_write_read bwr;

        // Is the read buffer empty?
            const bool needRead = mIn.dataPosition() >= mIn.dataSize();

        // We don't want to write anything if we are still reading
        // from data left in the input buffer and the caller
        // has requested to read the next data.
            const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

        bwr.write_size = outAvail;
        bwr.write_buffer = (uintptr_t) mOut.data();

        // This is what we'll read.
        if (doReceive && needRead) {
            //接受数据缓冲区信息的填充，如果以后收到数据，就直接填在mIn中了。
            bwr.read_size = mIn.dataCapacity();
            bwr.read_buffer = (uintptr_t) mIn.data();
        } else {
            bwr.read_size = 0;
            bwr.read_buffer = 0;
        }
        IF_LOG_COMMANDS() {
            TextOutput::Bundle _b(alog);
            if (outAvail != 0) {
                alog << "Sending commands to driver: " << indent;
                            const void*cmds = (const void*)bwr.write_buffer;
                            const void*end = ((const uint8_t *)cmds)+bwr.write_size;
                alog << HexDump(cmds, bwr.write_size) << endl;
                while (cmds < end) cmds = printCommand(alog, cmds);
                alog << dedent;
            }
            alog << "Size of receive buffer: " << bwr.read_size
                    << ", needRead: " << needRead << ", doReceive: " << doReceive << endl;
        }

        // Return immediately if there is nothing to do.
        // 当读缓冲和写缓冲都为空，则直接返回
        if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;
        bwr.write_consumed = 0;
        bwr.read_consumed = 0;
        status_t err;
        do {
            IF_LOG_COMMANDS() {
                alog << "About to read/write, write size = " << mOut.dataSize() << endl;
            }
            #if defined(HAVE_ANDROID_OS)
            //通过ioctl不停的读写操作，跟Binder驱动进行通信
            if (ioctl(mProcess -> mDriverFD, BINDER_WRITE_READ, & bwr) >=0)
            err = NO_ERROR;
                    else
            err = -errno;
            #else
            err = INVALID_OPERATION;
            #endif
            if (mProcess -> mDriverFD <= 0) {
                err = -EBADF;
            }
            IF_LOG_COMMANDS() {
                alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
            }
        } while (err == -EINTR);

        IF_LOG_COMMANDS() {
            alog << "Our err: " << (void*)(intptr_t) err << ", write consumed: "
                    << bwr.write_consumed << " (of " << mOut.dataSize()
                    << "), read consumed: " << bwr.read_consumed << endl;
        }

        if (err >= NO_ERROR) {
            if (bwr.write_consumed > 0) {
                if (bwr.write_consumed < mOut.dataSize())
                    mOut.remove(0, bwr.write_consumed);
                else
                    mOut.setDataSize(0);
            }
            if (bwr.read_consumed > 0) {
                mIn.setDataSize(bwr.read_consumed);
                mIn.setDataPosition(0);
            }
            IF_LOG_COMMANDS() {
                TextOutput::Bundle _b(alog);
                alog << "Remaining data size: " << mOut.dataSize() << endl;
                alog << "Received commands from driver: " << indent;
                          const void*cmds = mIn.data();
                           const void*end = mIn.data() + mIn.dataSize();
                alog << HexDump(cmds, mIn.dataSize()) << endl;
                while (cmds < end) cmds = printReturnCommand(alog, cmds);
                alog << dedent;
            }
            return NO_ERROR;
        }
        return err;
    }
```

**binder_write_read结构体** 用来与Binder设备交换数据的结构，通过ioctl与mDriverFD通信，是真正与Binder驱动进行数据读写交互的过程。主要操作是mOut和mIn变量。

ioctl经过系统调用后进入Binder Driver
 大体流程如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-9025f7a168cd3333.png?imageMogr2/auto-orient/strip|imageView2/2/w/546/format/webp)

大体流程图.png

### (六)、Binder驱动

Binder驱动内部调用了流程

> ioctl——> binder_ioctl ——> binder_ioctl_write_read

#### 1、binder_ioctl_write_read()函数处理

```cpp
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    struct binder_proc *proc = filp->private_data;
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    //将用户空间bwr结构体拷贝到内核空间
    copy_from_user(&bwr, ubuf, sizeof(bwr));
    //  ***省略部分代码***
    if (bwr.write_size > 0) {
        //将数据放入目标进程
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        //  ***省略部分代码***
    }
    if (bwr.read_size > 0) {
        //读取自己队列的数据 
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
             bwr.read_size,
             &bwr.read_consumed,
             filp->f_flags & O_NONBLOCK);
        if (!list_empty(&proc->todo))
            wake_up_interruptible(&proc->wait);
           //  ***省略部分代码***
    }

    //将内核空间bwr结构体拷贝到用户空间
    copy_to_user(ubuf, &bwr, sizeof(bwr));
     //  ***省略部分代码***
}  
```

#### 2、binder_thread_write()函数处理

```cpp
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr < end && thread->return_error == BR_OK) {
        //拷贝用户空间的cmd命令，此时为BC_TRANSACTION
        if (get_user(cmd, (uint32_t __user *)ptr)) -EFAULT;
        ptr += sizeof(uint32_t);
        switch (cmd) {
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;
            //拷贝用户空间的binder_transaction_data
            if (copy_from_user(&tr, ptr, sizeof(tr)))   return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
            break;
        }
        //  ***省略部分代码***
    }
    *consumed = ptr - buffer;
  }
  return 0;
}
```

#### 3、binder_thread_write()函数处理

```php
static void binder_transaction(struct binder_proc *proc,
               struct binder_thread *thread,
               struct binder_transaction_data *tr, int reply){

    if (reply) {
        //  ***省略部分代码***
    }else {
        if (tr->target.handle) {
          //  ***省略部分代码***
        } else {
            // handle=0则找到servicemanager实体
            target_node = binder_context_mgr_node;
        }
        //target_proc为servicemanager进程
        target_proc = target_node->proc;
    }

    if (target_thread) {
         //  ***省略部分代码***
    } else {
        //找到servicemanager进程的todo队列
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }

    t = kzalloc(sizeof(*t), GFP_KERNEL);
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);

    //非oneway的通信方式，把当前thread保存到transaction的from字段
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;

    t->sender_euid = task_euid(proc->tsk);
    t->to_proc = target_proc; //此次通信目标进程为servicemanager进程
    t->to_thread = target_thread;
    t->code = tr->code;  //此次通信code = ADD_SERVICE_TRANSACTION
    t->flags = tr->flags;  // 此次通信flags = 0
    t->priority = task_nice(current);

    //从servicemanager进程中分配buffer
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));

    t->buffer->allow_user_free = 0;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;

    if (target_node)
        //引用计数+1
        binder_inc_node(target_node, 1, 0, NULL); 
    offp = (binder_size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));

    //分别拷贝用户空间的binder_transaction_data中ptr.buffer和ptr.offsets到内核
    copy_from_user(t->buffer->data,
        (const void __user *)(uintptr_t)tr->data.ptr.buffer, tr->data_size);
    copy_from_user(offp,
        (const void __user *)(uintptr_t)tr->data.ptr.offsets, tr->offsets_size);

    off_end = (void *)offp + tr->offsets_size;

    for (; offp < off_end; offp++) {
        struct flat_binder_object *fp;
        fp = (struct flat_binder_object *)(t->buffer->data + *offp);
        off_min = *offp + sizeof(struct flat_binder_object);
        switch (fp->type) {
            case BINDER_TYPE_BINDER:
            case BINDER_TYPE_WEAK_BINDER: {
              struct binder_ref *ref;
              struct binder_node *node = binder_get_node(proc, fp->binder);
              if (node == NULL) { 
                //服务所在进程 创建binder_node实体
                node = binder_new_node(proc, fp->binder, fp->cookie);
                 //  ***省略部分代码***
              }
              //servicemanager进程binder_ref
              ref = binder_get_ref_for_node(target_proc, node);
              ...
              //调整type为HANDLE类型
              if (fp->type == BINDER_TYPE_BINDER)
                fp->type = BINDER_TYPE_HANDLE;
              else
                fp->type = BINDER_TYPE_WEAK_HANDLE;
              fp->binder = 0;
              fp->handle = ref->desc; //设置handle值
              fp->cookie = 0;
              binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                       &thread->todo);
            } break;
            case :  //  ***省略部分代码***
    }

    if (reply) {
          //  ***省略部分代码***
    } else if (!(t->flags & TF_ONE_WAY)) {
        //BC_TRANSACTION 且 非oneway,则设置事务栈信息
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        thread->transaction_stack = t;
    } else {
          //  ***省略部分代码***
    }
    //将BINDER_WORK_TRANSACTION添加到目标队列，本次通信的目标队列为target_proc->todo
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);
    //将BINDER_WORK_TRANSACTION_COMPLETE添加到当前线程的todo队列
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    //唤醒等待队列，本次通信的目标队列为target_proc->wait
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
}
```

> - 注册服务的过程，传递的是BBinder对象，因此上面的writeStrongBinder()过程中localBinder不为空，从而flat_binder_object.type等于BINDER_TYPE_BINDER。
> - 服务注册过程是在服务所在进程创建binder_node，在servicemanager进程创建binder_ref。对于同一个binder_node，每个进程只会创建一个binder_ref对象。
> - 向servicemanager的binder_proc->todo添加BINDER_WORK_TRANSACTION事务，接下来进入ServiceManager进程。

这里说下这个函数里面涉及的三个重要函数

> - binder_get_node()
> - binder_new_node()
> - binder_get_ref_for_node()

**3.1、binder_get_node()函数处理**

```rust
//     /kernel/drivers/android/binder.c     904行
static struct binder_node *binder_get_node(struct binder_proc *proc,
             binder_uintptr_t ptr)
{
  struct rb_node *n = proc->nodes.rb_node;
  struct binder_node *node;

  while (n) {
    node = rb_entry(n, struct binder_node, rb_node);

    if (ptr < node->ptr)
      n = n->rb_left;
    else if (ptr > node->ptr)
      n = n->rb_right;
    else
      return node;
  }
  return NULL;
}
```

从binder_proc来根据binder指针ptr值，查询相应的binder_node

**3.2、binder_new_node()函数处理**

```php
//kernel/drivers/android/binder.c      923行
static struct binder_node *binder_new_node(struct binder_proc *proc,
                       binder_uintptr_t ptr,
                       binder_uintptr_t cookie)
{
    struct rb_node **p = &proc->nodes.rb_node;
    struct rb_node *parent = NULL;
    struct binder_node *node;
        //第一次进来是空
    while (*p) {
        parent = *p;
        node = rb_entry(parent, struct binder_node, rb_node);

        if (ptr < node->ptr)
            p = &(*p)->rb_left;
        else if (ptr > node->ptr)
            p = &(*p)->rb_right;
        else
            return NULL;
    }
        //给创建的binder_node 分配内存空间
    node = kzalloc(sizeof(*node), GFP_KERNEL);
    if (node == NULL)
        return NULL;
    binder_stats_created(BINDER_STAT_NODE);
        //将创建的node对象添加到proc红黑树
    rb_link_node(&node->rb_node, parent, p);
    rb_insert_color(&node->rb_node, &proc->nodes);
    node->debug_id = ++binder_last_id;
    node->proc = proc;
    node->ptr = ptr;
    node->cookie = cookie;
        //设置binder_work的type
    node->work.type = BINDER_WORK_NODE;
    INIT_LIST_HEAD(&node->work.entry);
    INIT_LIST_HEAD(&node->async_todo);
    binder_debug(BINDER_DEBUG_INTERNAL_REFS,
             "%d:%d node %d u%016llx c%016llx created\n",
             proc->pid, current->pid, node->debug_id,
             (u64)node->ptr, (u64)node->cookie);
    return node;
}
```

**3.3、binder_get_ref_for_node()函数处理**

```rust
//    kernel/drivers/android/binder.c      1066行
static struct binder_ref *binder_get_ref_for_node(struct binder_proc *proc,
              struct binder_node *node)
{
  struct rb_node *n;
  struct rb_node **p = &proc->refs_by_node.rb_node;
  struct rb_node *parent = NULL;
  struct binder_ref *ref, *new_ref;
  //从refs_by_node红黑树，找到binder_ref则直接返回。
  while (*p) {
    parent = *p;
    ref = rb_entry(parent, struct binder_ref, rb_node_node);

    if (node < ref->node)
      p = &(*p)->rb_left;
    else if (node > ref->node)
      p = &(*p)->rb_right;
    else
      return ref;
  }
  
  //创建binder_ref
  new_ref = kzalloc_preempt_disabled(sizeof(*ref));
  
  new_ref->debug_id = ++binder_last_id;
  //记录进程信息
  new_ref->proc = proc; 
  // 记录binder节点
  new_ref->node = node; 
  rb_link_node(&new_ref->rb_node_node, parent, p);
  rb_insert_color(&new_ref->rb_node_node, &proc->refs_by_node);

  //计算binder引用的handle值，该值返回给target_proc进程
  new_ref->desc = (node == binder_context_mgr_node) ? 0 : 1;
  //从红黑树最最左边的handle对比，依次递增，直到红黑树遍历结束或者找到更大的handle则结束。
  for (n = rb_first(&proc->refs_by_desc); n != NULL; n = rb_next(n)) {
    //根据binder_ref的成员变量rb_node_desc的地址指针n，来获取binder_ref的首地址
    ref = rb_entry(n, struct binder_ref, rb_node_desc);
    if (ref->desc > new_ref->desc)
      break;
    new_ref->desc = ref->desc + 1;
  }

  // 将新创建的new_ref 插入proc->refs_by_desc红黑树
  p = &proc->refs_by_desc.rb_node;
  while (*p) {
    parent = *p;
    ref = rb_entry(parent, struct binder_ref, rb_node_desc);

    if (new_ref->desc < ref->desc)
      p = &(*p)->rb_left;
    else if (new_ref->desc > ref->desc)
      p = &(*p)->rb_right;
    else
      BUG();
  }
  rb_link_node(&new_ref->rb_node_desc, parent, p);
  rb_insert_color(&new_ref->rb_node_desc, &proc->refs_by_desc);
  if (node) {
    hlist_add_head(&new_ref->node_entry, &node->refs);
  } 
  return new_ref;
}
```

handle值计算方法规律：

> - 每个进程binder_proc所记录的binder_ref的handle值是从1开始递增的
> - 所有进程binder_proc所记录的handle=0的binder_ref都指向service manager
> - 同一服务的binder_node在不同进程的binder_ref的handle值可以不同

### (七)、ServiceManager流程

关于ServiceManager的启动流程，我这里就不详细讲解了。启动后，就会循环在binder_loop()过程，当来消息后，会调用binder_parse()函数

**1、binder_parse()函数**

```cpp
// framework/native/cmds/servicemanager/binder.c      204行
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);
        switch(cmd) {
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            // *** 省略部分源码 ***
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg; 
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                //从txn解析出binder_io信息
                bio_init_from_txn(&msg, txn); 
                 // 收到Binder事务 
                res = func(bs, txn, &msg, &reply);
                // 发送reply事件
                binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
            }
            ptr += sizeof(*txn);
            break;
        }
        case :   // *** 省略部分源码 ***
    }
    return r;
}
```

**2、svcmgr_handler()函数**

```go
//frameworks/native/cmds/servicemanager/service_manager.c  
    244行  
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;
    // *** 省略部分源码 ***
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
     // *** 省略部分源码 ***
    switch(txn->code) {
      case SVC_MGR_ADD_SERVICE: 
          s = bio_get_string16(msg, &len);
          ...
          handle = bio_get_ref(msg); //获取handle
          allow_isolated = bio_get_uint32(msg) ? 1 : 0;
           //注册指定服务 
          if (do_add_service(bs, s, len, handle, txn->sender_euid,
              allow_isolated, txn->sender_pid))
              return -1;
          break;
       case : // *** 省略部分源码 ***
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```

**3、do_add_service()函数**

```cpp
// frameworks/native/cmds/servicemanager/service_manager.c     194行
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid)
{
    struct svcinfo *si;

    if (!handle || (len == 0) || (len > 127))
        return -1;

    //权限检查
    if (!svc_can_register(s, len, spid)) {
        return -1;
    }

    //服务检索
    si = find_svc(s, len);
    if (si) {
        if (si->handle) {
            //服务已经注册时，释放相应的服务
            svcinfo_death(bs, si); 
        }
        si->handle = handle;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        //内存不足时，无法分配足够的内存
        if (!si) { 
            return -1;
        }
        si->handle = handle;
        si->len = len;
         //内存拷贝服务信息
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t)); 
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        //svclist保存所有已注册的服务
        si->next = svclist; 
        svclist = si;
    }

    //以BC_ACQUIRE命令，handle为目标的信息，通过ioctl发送给binder驱动
    binder_acquire(bs, handle);
    //以BC_REQUEST_DEATH_NOTIFICATION命令的信息，通过ioctl发送给binder驱动，主要用于清理内存等收尾工作。
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```

svcinfo记录着服务名和handle信息

**4、binder_send_reply()函数**

```kotlin
// frameworks/native/cmds/servicemanager/binder.c   170行
void binder_send_reply(struct binder_state *bs,
                       struct binder_io *reply,
                       binder_uintptr_t buffer_to_free,
                       int status)
{
    struct {
        uint32_t cmd_free;
        binder_uintptr_t buffer;
        uint32_t cmd_reply;
        struct binder_transaction_data txn;
    } __attribute__((packed)) data;
     //free buffer命令
    data.cmd_free = BC_FREE_BUFFER; 
    data.buffer = buffer_to_free;
    // reply命令
    data.cmd_reply = BC_REPLY; 
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
     // *** 省略部分源码 ***
    } else {
        data.txn.flags = 0;
        data.txn.data_size = reply->data - reply->data0;
        data.txn.offsets_size = ((char*) reply->offs) - ((char*) reply->offs0);
        data.txn.data.ptr.buffer = (uintptr_t)reply->data0;
        data.txn.data.ptr.offsets = (uintptr_t)reply->offs0;
    }
    //向Binder驱动通信
    binder_write(bs, &data, sizeof(data));
}
```

binder_write进去binder驱动后，将BC_FREE_BUFFER和BC_REPLY命令协议发送给Binder驱动，向Client端发送reply
 binder_write进入binder驱动后，将BC_FREE_BUFFER和BC_REPLY命令协议发送给Binder驱动， 向client端发送reply.

### (八)、总结

服务注册过程(addService)核心功能：在服务所在进程创建的binder_node，在servicemanager进程创建binder_ref。其中binder_ref的desc在同一个进程内是唯一的：

> - 每个进程binder_proc所记录的binder_ref的handle值是从1开始递增的
> - 所有进程binder_proc所记录的bandle=0的binder_ref指向service manager
> - 同一个服务的binder_node在不同的进程的binder_ref的handle值可以不同

Media服务注册的过程设计到MediaPlayerService(作为Cliient进程)和Service Manager(作为Service 进程)，通信的流程图如下：

 ![img](https://upload-images.jianshu.io/upload_images/5713484-14140d892d92a261.png?imageMogr2/auto-orient/strip|imageView2/2/w/953/format/webp) 

Media服务注册流程.png

过程分析：

> - 1、MediaPlayerService进程调用 
>
>   ioctl()
>
>   向Binder驱动发送IPC数据，该过程可以理解成一个事物 
>
>   binder_transaction
>
>    (记为BT1)，执行当前操作线程的binder_thread(记为 thread1)，则BT1 ->from_parent=NULL， BT1 ->from=thread1，thread1 ->transaction_stack=T1。其中IPC数据内容包括： 
>
>   - Binder协议为BC_TRANSACTION
>   - Handle等于0
>   - PRC代码为ADD_SERVICE
>   - PRC数据为"media.player"
>
> - 2、Binder驱动收到该Binder请求。生成BR_TRANSACTION命令，选择目标处理该请求的线程，即ServiceManager的binder线程(记为thread2)，则T1->to_parent=NULL,T1 -> to_thread=thread2，并将整个binder_transaction数据(记为BT2)插入到目标线程的todo队列。
>
> - 3、Service Manager的线程thread收到BT2后，调用服务注册函数将服务“media.player”注册到服务目录中。当服务注册完成，生成IPC应答数据(BC_REPLY)，BT2->from_parent=BT1，BT2 ->from=thread2，thread2->transaction_stack=BT2。
>
> - 4、Binder驱动收到该Binder应答请求，生成BR_REPLY命令，BT2->to_parent=BT1，BT2->to_thread1，thread1->transaction_stack=BT2。在MediaPlayerService收到该命令后，知道服务注册完成便可以正常使用。

## 五、获取服务

### (一) 源码位置

```java
/frameworks/av/media/libmedia/
  - IMediaDeathNotifier.cpp

framework/native/libs/binder/
  - Binder.cpp
  - BpBinder.cpp
  - IPCThreadState.cpp
  - ProcessState.cpp
  - IServiceManager.cpp
```

对应的链接为

> - [Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Binder.cpp)
> - [BpBinder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/BpBinder.cpp)
> - [IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
> - [ProcessState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/ProcessState.cpp)
> - [IServiceManager.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IServiceManager.cpp)
> - [IInterface.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IServiceManager.cpp)
> - [IInterface.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IInterface.cpp)
> - [IInterface.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/include/binder/IInterface.h)
> - [main_mediaserver.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/av/media/mediaserver/main_mediaserver.cpp)
> - [MediaPlayerService.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp)
> - [IMediaDeathNotifier.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/av/media/libmedia/IMediaDeathNotifier.cpp)

在Native层的服务注册，我们依旧选择media为例展开讲解，先来看看media类关系图。

### (二)、类图

 ![img](https://upload-images.jianshu.io/upload_images/5713484-245b6f229fa02280.png?imageMogr2/auto-orient/strip|imageView2/2/w/1014/format/webp) 

类图.png

图解：

> - 蓝色:代表获取MediaPlayerService服务相关的类
> - 绿色:代表Binder架构中与Binder驱动通信过程中的最为核心的两个雷
> - 紫色:代表 **注册服务** 和 **获取服务** 的公共接口/父类

### (三)、获取服务流程

#### **1、getMediaPlayerService()函数**

```php
//frameworks/av/media/libmedia/IMediaDeathNotifier.cpp   35行
sp<IMediaPlayerService>&
IMediaDeathNotifier::getMediaPlayerService()
{
    Mutex::Autolock _l(sServiceLock);
    if (sMediaPlayerService == 0) {
         // 获取 ServiceManager
        sp<IServiceManager> sm = defaultServiceManager(); 
        sp<IBinder> binder;
        do {
            //获取名为"media.player"的服务
            binder = sm->getService(String16("media.player"));
            if (binder != 0) {
                break;
            }
            usleep(500000); // 0.5s
        } while (true);

        if (sDeathNotifier == NULL) {
            // 创建死亡通知对象
            sDeathNotifier = new DeathNotifier(); 
        }

        //将死亡通知连接到binder
        binder->linkToDeath(sDeathNotifier);
        sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
    }
    return sMediaPlayerService;
}
```

其中defaultServiceManager()过程在上面已经说了，返回的是BpServiceManager

> 在请求获取名为"media.player"的服务过程中，采用不断循环获取的方法。由于MediaPlayerService服务可能还没向ServiceManager注册完成或者尚未启动完成等情况，故则binder返回NULL，休眠0.5s后继续请求，知道获取服务为止。

#### **2、BpServiceManager.getService()函数**

```cpp
//frameworks/native/libs/binder/IServiceManager.cpp       134行
virtual sp<IBinder> getService(const String16& name) const
    {
        unsigned n;
        for (n = 0; n < 5; n++){
            sp<IBinder> svc = checkService(name); 
            if (svc != NULL) return svc;
            sleep(1);
        }
        return NULL;
    }
```

通过BpServiceManager来获取MediaPlayer服务：检索服务是否存在，当服务存在则返回相应的服务，当服务不存在则休眠1s再继续检索服务。该循环进行5次。为什么循环5次？这估计和Android的ANR的时间为5s相关。如果每次都无法获取服务，循环5次，每次循环休眠1s，忽略checkService()的时间，差不多是5s的时间。

#### **3、BpSeriveManager.checkService()函数**

```kotlin
//frameworks/native/libs/binder/IServiceManager.cpp    146行
virtual sp<IBinder> checkService( const String16& name) const
{
    Parcel data, reply;
    //写入RPC头
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
    //写入服务名
    data.writeString16(name);
    remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply); 
    return reply.readStrongBinder(); 
}
```

检索制定服务是否存在，其中remote()为BpBinder

#### **4、BpBinder::transact()函数**

```cpp
// /frameworks/native/libs/binder/BpBinder.cpp    159行
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
    return DEAD_OBJECT;
}
```

Binder代理类调用transact()方法，真正工作还是交给IPCThreadState来进行transact工作。

**4.1、IPCThreadState::self()函数**

```cpp
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        //初始化 IPCThreadState
        return new IPCThreadState;  
    }

    if (gShutdown) return NULL;
    pthread_mutex_lock(&gTLSMutex);
     //首次进入gHaveTLS为false
    if (!gHaveTLS) {
         //创建线程的TLS
        if (pthread_key_create(&gTLS, threadDestructor) != 0) { 
            pthread_mutex_unlock(&gTLSMutex);
            return NULL;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}
```

TLS是指Thread local storage(线程本地存储空间)，每个线程都拥有自己的TLS，并且是私有空间，线程之间不会共享。通过pthread_getspecific()/pthread_setspecific()函数可以获取/设置这些空间中的内容。从线程本地存储空间获的保存期中的IPCThreadState对象。

**以后面的流程和上面的注册流程大致相同，主要流程也是 IPCThreadState:: transact()函数、IPCThreadState::writeTransactionData()函数、IPCThreadState::waitForResponse()函数和IPCThreadState.talkWithDriver()函数，由于上面已经讲解过了，这里就不详细说明了。我们从IPCThreadState.talkWithDriver()  开始继讲解**

**4.2、IPCThreadState:: talkWithDriver()函数**

```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    ...
    binder_write_read bwr;
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();
     //接收数据缓冲区信息的填充。如果以后收到数据，就直接填在mIn中了。
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    //当读缓冲和写缓冲都为空，则直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        //通过ioctl不停的读写操作，跟Binder Driver进行通信
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        ...
       //当被中断，则继续执行
    } while (err == -EINTR); 
    ...
    return err;
}
```

**binder_write_read结构体** 用来与Binder设备交换数据的结构，通过ioctl与mDriverFD通信，是真正的与Binder驱动进行数据读写交互的过程。先向service manager进程发送查询服务的请求(BR_TRANSACTION)。当service manager 进程收到带命令后，会执行do_find_service()查询服务所对应的handle，然后再binder_send_reply()应发送者，发送BC_REPLY协议，然后再调用binder_transaction()，再向服务请求者的todo队列插入事务。接下来，再看看binder_transaction过程。

让我们继续看下binder_transaction的过程

**4.2.1、binder_transaction()函数**

```php
//kernel/drivers/android/binder.c     1827行
static void binder_transaction(struct binder_proc *proc,
               struct binder_thread *thread,
               struct binder_transaction_data *tr, int reply){
    //根据各种判定，获取以下信息：
    // 目标线程
    struct binder_thread *target_thread； 
    // 目标进程
    struct binder_proc *target_proc； 
     /// 目标binder节点   
    struct binder_node *target_node；    
    // 目标 TODO队列
    struct list_head *target_list；    
     // 目标等待队列
    wait_queue_head_t *target_wait；    
    ...
    
    //分配两个结构体内存
    struct binder_transaction *t = kzalloc(sizeof(*t), GFP_KERNEL);
    struct binder_work *tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    //从target_proc分配一块buffer
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,

    for (; offp < off_end; offp++) {
        switch (fp->type) {
        case BINDER_TYPE_BINDER: ...
        case BINDER_TYPE_WEAK_BINDER: ...
        
        case BINDER_TYPE_HANDLE: 
        case BINDER_TYPE_WEAK_HANDLE: {
          struct binder_ref *ref = binder_get_ref(proc, fp->handle,
                fp->type == BINDER_TYPE_HANDLE);
          ...
          //此时运行在servicemanager进程，故ref->node是指向服务所在进程的binder实体，
          //而target_proc为请求服务所在的进程，此时并不相等。
          if (ref->node->proc == target_proc) {
            if (fp->type == BINDER_TYPE_HANDLE)
              fp->type = BINDER_TYPE_BINDER;
            else
              fp->type = BINDER_TYPE_WEAK_BINDER;
            fp->binder = ref->node->ptr;
             // BBinder服务的地址
            fp->cookie = ref->node->cookie; 
            binder_inc_node(ref->node, fp->type == BINDER_TYPE_BINDER, 0, NULL);
            
          } else {
            struct binder_ref *new_ref;
            //请求服务所在进程并非服务所在进程，则为请求服务所在进程创建binder_ref
            new_ref = binder_get_ref_for_node(target_proc, ref->node);
            fp->binder = 0;
             //重新给handle赋值
            fp->handle = new_ref->desc; 
            fp->cookie = 0;
            binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);
          }
        } break;
        
        case BINDER_TYPE_FD: ...
        }
    }
    //分别target_list和当前线程TODO队列插入事务
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
}
```

这个过程非常重要，分两种情况来说：

> - 情况1 当请求服务的进程与服务属于不同的进程，则为请求服务所在进程创建binder_ref对象，指向服务进程中的binder_node
> - 当请求服务的进程与服务属于同一进程，则不再创建新对象，只是引用计数+1，并且修改type为BINDER_TYPE_BINER或BINDER_TYPE_WEAK_BINDER。

**4.2.2、binder_thread_read()函数**

```php
//kernel/drivers/android/binder.c    2650行
binder_thread_read（struct binder_proc *proc,struct binder_thread *thread,binder_uintptr_t binder_buffer, size_t size,binder_size_t *consumed, int non_block）{
    ...
    //当线程todo队列有数据则执行往下执行；当线程todo队列没有数据，则进入休眠等待状态
    ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    ...
    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;
        //先从线程todo队列获取事务数据
        if (!list_empty(&thread->todo)) {
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        // 线程todo队列没有数据, 则从进程todo对获取事务数据
        } else if (!list_empty(&proc->todo) && wait_for_proc_work) {
            ...
        }
        switch (w->type) {
            case BINDER_WORK_TRANSACTION:
                //获取transaction数据
                t = container_of(w, struct binder_transaction, work);
                break;
                
            case : ...  
        }

        //只有BINDER_WORK_TRANSACTION命令才能继续往下执行
        if (!t) continue;

        if (t->buffer->target_node) {
            ...
        } else {
            tr.target.ptr = NULL;
            tr.cookie = NULL;
            //设置命令为BR_REPLY
            cmd = BR_REPLY; 
        }
        tr.code = t->code;
        tr.flags = t->flags;
        tr.sender_euid = t->sender_euid;

        if (t->from) {
            struct task_struct *sender = t->from->proc->tsk;
            //当非oneway的情况下,将调用者进程的pid保存到sender_pid
            tr.sender_pid = task_tgid_nr_ns(sender, current->nsproxy->pid_ns);
        } else {
            ...
        }

        tr.data_size = t->buffer->data_size;
        tr.offsets_size = t->buffer->offsets_size;
        tr.data.ptr.buffer = (void *)t->buffer->data +
                    proc->user_buffer_offset;
        tr.data.ptr.offsets = tr.data.ptr.buffer +
                    ALIGN(t->buffer->data_size,
                        sizeof(void *));

        //将cmd和数据写回用户空间
        put_user(cmd, (uint32_t __user *)ptr);
        ptr += sizeof(uint32_t);
        copy_to_user(ptr, &tr, sizeof(tr));
        ptr += sizeof(tr);

        list_del(&t->work.entry);
        t->buffer->allow_user_free = 1;
        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
            ...
        } else {
            t->buffer->transaction = NULL;
            //通信完成则运行释放
            kfree(t); 
        }
        break;
    }
done:
    *consumed = ptr - buffer;
    if (proc->requested_threads + proc->ready_threads == 0 &&
        proc->requested_threads_started < proc->max_threads &&
        (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED))) {
        proc->requested_threads++;
        // 生成BR_SPAWN_LOOPER命令，用于创建新的线程
        put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer)；
    }
    return 0;
}
```

**4.3、readStrongBinder()函数**

```kotlin
//frameworks/native/libs/binder/Parcel.cpp   1334行
sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    unflatten_binder(ProcessState::self(), *this, &val);
    return val;
}
```

里面主要是调用unflatten_binder()函数
 那我们就来详细看下

**4.3.1、unflatten_binder()函数**

```cpp
status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);
    if (flat) {
        switch (flat->type) {
            case BINDER_TYPE_BINDER:
                // 当请求服务的进程与服务属于同一进程
                *out = reinterpret_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                //请求服务的进程与服务属于不同进程
                *out = proc->getStrongProxyForHandle(flat->handle);
                //创建BpBinder对象
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
```

如果服务的进程与服务属于不同的进程会调用getStrongProxyForHandle()函数，那我们就好好研究下

**4.3.2、getStrongProxyForHandle()函数**

```php
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);
    //查找handle对应的资源项[2.9.3]
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            ...
            //当handle值所对应的IBinder不存在或弱引用无效时，则创建BpBinder对象
            b = new BpBinder(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}
```

readStrong的功能是flat_binder_object解析并创建BpBinder对象

**4.3.2、getStrongProxyForHandle()函数**

```cpp
ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
{
    const size_t N=mHandleToObject.size();
    //当handle大于mHandleToObject的长度时，进入该分支
    if (N <= (size_t)handle) {
        handle_entry e;
        e.binder = NULL;
        e.refs = NULL;
        //从mHandleToObject的第N个位置开始，插入(handle+1-N)个e到队列中
        status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
        if (err < NO_ERROR) return NULL;
    }
    return &mHandleToObject.editItemAt(handle);
}
```

根据handle值来查找对应的handle_entry。

### (四)、死亡通知

死亡通知时为了让Bp端知道Bn端的生死情况

> - DeathNotifier是继承IBinder::DeathRecipient类，主要需要实现其binderDied()来进行死亡通告。
> - 注册：binder->linkToDeath(sDeathNotifier)是为了将sDeathNotifier死亡通知注册到Binder上。

Bp端只需要覆写binderDied()方法，实现一些后尾清楚类的工作，则在Bn端死掉后，会回调binderDied()进行相应处理

**1、linkToDeath()函数**

```cpp
// frameworks/native/libs/binder/BpBinder.cpp   173行
status_t BpBinder::linkToDeath(
    const sp<DeathRecipient>& recipient, void* cookie, uint32_t flags)
{
    Obituary ob;
    ob.recipient = recipient;
    ob.cookie = cookie;
    ob.flags = flags;

    {
        AutoMutex _l(mLock);
        if (!mObitsSent) {
            if (!mObituaries) {
                mObituaries = new Vector<Obituary>;
                if (!mObituaries) {
                    return NO_MEMORY;
                }
                getWeakRefs()->incWeak(this);
                IPCThreadState* self = IPCThreadState::self();
                self->requestDeathNotification(mHandle, this);
                self->flushCommands();
            }
            ssize_t res = mObituaries->add(ob);
            return res >= (ssize_t)NO_ERROR ? (status_t)NO_ERROR : res;
        }
    }
    return DEAD_OBJECT;
}
```

里面调用了requestDeathNotification()函数

**2、requestDeathNotification()函数**

```cpp
//frameworks/native/libs/binder/IPCThreadState.cpp    670行
status_t IPCThreadState::requestDeathNotification(int32_t handle, BpBinder* proxy)
{
    mOut.writeInt32(BC_REQUEST_DEATH_NOTIFICATION);
    mOut.writeInt32((int32_t)handle);
    mOut.writePointer((uintptr_t)proxy);
    return NO_ERROR;
}
```

向binder driver发送 BC_REQUEST_DEATH_NOTIFICATION命令。后面的流程和  **Service Manager** 里面的 ** binder_link_to_death()  ** 的过程。

**3、binderDied()函数**

```cpp
//frameworks/av/media/libmedia/IMediaDeathNotifier.cpp    78行
void IMediaDeathNotifier::DeathNotifier::binderDied(const wp<IBinder>& who __unused) {
    SortedVector< wp<IMediaDeathNotifier> > list;
    {
        Mutex::Autolock _l(sServiceLock);
        // 把Bp端的MediaPlayerService清除掉
        sMediaPlayerService.clear();   
        list = sObitRecipients;
    }

    size_t count = list.size();
    for (size_t iter = 0; iter < count; ++iter) {
        sp<IMediaDeathNotifier> notifier = list[iter].promote();
        if (notifier != 0) {
            //当MediaServer挂了则通知应用程序，应用程序回调该方法
            notifier->died(); 
        }
    }
}
```

客户端进程通过Binder驱动获得Binder的代理(BpBinder)，死亡通知注册的过程就是客户端进程向Binder驱动注册的一个死亡通知，该死亡通知关联BBinder，即与BpBinder所对应的服务端。

**4、unlinkToDeath()函数**

当Bp在收到服务端的死亡通知之前先挂了，那么需要在对象的销毁方法内，调用unlinkToDeath()来取消死亡通知；

```cpp
//frameworks/av/media/libmedia/IMediaDeathNotifier.cpp    101行
IMediaDeathNotifier::DeathNotifier::~DeathNotifier()
{
    Mutex::Autolock _l(sServiceLock);
    sObitRecipients.clear();
    if (sMediaPlayerService != 0) {
        IInterface::asBinder(sMediaPlayerService)->unlinkToDeath(this);
    }
}
```

**5、触发时机**

> 每当service进程退出时，service manager 会收到来自Binder驱动的死亡通知。这项工作在启动Service Manager时通过 binder_link_to_death(bs, ptr, &si->death)完成。另外，每个Bp端也可以自己注册死亡通知，能获取Binder的死亡消息，比如前面的IMediaDeathNotifier。

那Binder的死亡通知时如何被出发的？对于Binder的IPC进程都会打开/dev/binder文件，当进程异常退出的时候，Binder驱动会保证释放将要退出的进程中没有正常关闭的/dev/binder文件，实现机制是binder驱动通过调用/dev/binder文件所对应的release回调函数，执行清理工作，并且检查BBinder是否有注册死亡通知，当发现存在死亡通知时，那么久向其对应的BpBinder端发送死亡通知消息。

### (五)总结

在请求服务(getService)的过程，当执行到binder_transaction()时，会区分请求服务所属进程情况。

> - 当请求服务的进程与服务属于不同进程，则为请求服务所在进程创binder_ref对象，指向服务进程的binder_noder
> - 当请求服务的进程与服务属于同一进程， 则不再创建新对象，只是引用计数+1，并且修改type为BINDER_TYPE_BINDER或BINDER_TYPE_WEAK_BINDER。 
>   - 最终readStrongBinder()，返回的是BB对象的真实子类



#  Android跨进程通信IPC之Binder之Framework层Java篇

本篇主要分析Binder在framework中Java层的框架，相关源码



```csharp
framework/base/core/java/android/os/
  - IInterface.java
  - IServiceManager.java
  - ServiceManager.java
  - ServiceManagerNative.java(包含内部类ServiceManagerProxy)

framework/base/core/java/android/os/
  - IBinder.java
  - Binder.java(包含内部类BinderProxy)
  - Parcel.java

framework/base/core/java/com/android/internal/os/
  - BinderInternal.java

framework/base/core/jni/
  - AndroidRuntime.cpp
  - android_os_Parcel.cpp
  - android_util_Binder.cpp
```

链接如下：

> - [IInterface.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/IInterface.java)
> - [IServiceManager.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/IServiceManager.java)
> - [ServiceManager.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/ServiceManager.java)
> - [ServiceManagerNative.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/ServiceManagerNative.java)
> - [IBinder.java)](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/IBinder.java)
> - [Binder.java)](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Binder.java)
> - [Parcel.java)](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Parcel.java)
> - [AndroidRuntime.cppshi](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/AndroidRuntime.cpp)
> - [AndroidRuntime.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/AndroidRuntime.cpp)
> - [android_os_Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_Parcel.cpp)
> - [android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp)

## 一、概述

Binder在framework层，采用JNI技术来调用native(C/C++)层的binder架构，从而为上层应用程序提供服务。看过binder之前的文章，我们知道native层中，binder是C/S架构，分为Bn端(Server)和Bp端(Client)。对于Java层在命令与架构上非常相近，同时实现了一套IPC通信架构。

### (一)架构图

framework Binder架构图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-898607abe3f440cf.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder架构图.png

图解：

> - 图中紫色色代表整个framework层binder相关组件，其中Binder代表Server端，BinderProxy代表Client端
> - 图中黑色代表Native层Binder架构相关组件
> - 上层framework层的Binder逻辑是建立在Native层架构基础上的，核心逻辑都是交于Native层来处理
> - framework层的ServiceManager类与Native层的功能并不完全对应，framework层的ServiceManager的实现对最终是通过BinderProxy传递给Native层来完成的。

### (二)、类图

下面列举framework的binder类的关系图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-3f6fec4c87c91181.png?imageMogr2/auto-orient/strip|imageView2/2/w/1074/format/webp)

类的关系图.png

图解：
 其中蓝色都是interface,其余都是Class

> - ServiceManager：通过getIServiceManager方法获取的是ServiceManagerProxy对象。ServiceMnager的addService()，getService()实际工作都交给ServiceManagerProxy的相应方法来处理。
> - ServiceManagerProxy：其成员变量mRemote指向BinderProxy对象，ServiceManagerProxy的addService()，getService()方法最终是交给mRemote去完成。
> - ServiceManagerNative：其方法asInterface()返回的是ServiceManagerProxy对象，ServiceManager便是借助ServiceManagerNative类来找到ServiceManagerProxy。
> - Binder：其成员mObject和方法execTransact()用于native方法
> - BinderInternal：内部有一个GcWatcher类，用于处理和调试与Binder相关的拦击回收。
> - IBinder：接口中常量FLAG_ONEWAY:客户端利用binder跟服务端通信是阻塞式的，但如果设置了FLAG_ONEWAY，这成为非阻塞的调用方式，客户端能立即返回，服务端采用回调方式来通知客户端完成情况。另外IBinder接口有一个内部接口DeathDecipent(死亡通告)。

### (三)、Binder类分层

整个Binder从kernel至native，JNI，Framework层所涉及的全部类

![img](https:////upload-images.jianshu.io/upload_images/5713484-d6d1cb1c6a19b759.png?imageMogr2/auto-orient/strip|imageView2/2/w/944/format/webp)

Binder类分层.png

Android应用程序使用Java语言开发，Binder框架自然也少不了在Java层提供接口。前面的文章我们知道，Binder机制在C++层有了完整的实现。因此Java层完全不用重复实现，而是通过JNI衔接C++层以复用其实现。

关于Binder类中 从Binder Framework层到C++层的衔接关系如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-5d7a452830adeffb.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder衔接关系图.png

图解：

> - IInterface(interface) ：供Java层Binder服务接口继承的接口
> - IBinder(interface)：Java层IBinder类，提供了transaction方法来调用远程服务
> - Binder(class)：实现了IBinder接口，封装了JNI的实现。Java层Binder服务的基类
> - BInderProxy(class)：实现了IBinder接口，封装了JNI的实现。提供transac()方法调用远程服务
> - JavaBBinderHolder(class) ：内部存储了JavaBBinder
> - JavaBBinder(class)：将C++端的onTransact调用传递到Java端
> - Parcel(class)：Java层的数据包装器。

这里的IInterface，IBinder和C++层的两个类是同名的。这个同名并不是巧合：它们不仅仅是同名，它们所起到的作用，以及其中包含的接口几乎都是一样的，区别仅仅是一个在C++层，一个在Java层而已。而且除了IInterface，IBinder之外，这里Binder与BinderProxy类也是与C++的类对应的，下面列出了Java层和C++层类的对应关系。

| C++层      | Java层     |
| ---------- | ---------- |
| IInterface | IInterface |
| IBinder    | IBinder    |
| BBinder    | BBinder    |
| BpProxy    | BpProxy    |
| Parcel     | Parcel     |

### (四)、JNI的衔接

> JNI全称是Java Native Interface，这个是由Java虚拟机提供的机制。这个机制使得natvie代码可以和Java代码相互通讯。简单的来说就是：我们可以在C/C++端调用Java代码，也可以在Java端调用C/C++代码。

关于JNI的详细说明，可以参见Oracle的官方文档:[Java Native Interface](https://link.jianshu.com?t=http://docs.oracle.com/javase/8/docs/technotes/guides/jni/) ，这里就不详细说明了。

> 实际上，在Android中很多的服务或者机制都是在C/C++层实现的，想要将这些实现复用到Java层
>  就必须通过JNI进行衔接。Android Opne Source Projcet(以后简称AOSP)在源码中，/frameworks/base/core/jni/ 目录下的源码就是专门用来对接Framework层的JNI实现。其实大家看一下Binder.java的实现就会发现，这里面有不少的方法都是native 关键字修饰的，并且没有方法实现体，这些方法其实都是在C++中实现的。

以Binder为例：

**1、Java调用C++层代码**

```java
// Binder.java
public static final native int getCallingPid();

public static final native int getCallingUid();

public static final native long clearCallingIdentity();

public static final native void restoreCallingIdentity(long token);

public static final native void setThreadStrictModePolicy(int policyMask);

public static final native int getThreadStrictModePolicy();

public static final native void flushPendingCommands();

public static final native void joinThreadPool();
```

在 android_util_Binder.cpp文件中的下面的这段代码，设定了Java方法与C++方法对应关系



```cpp
//android_util_Binder      843行
static const JNINativeMethod gBinderMethods[] = {
    { "getCallingPid", "()I", (void*)android_os_Binder_getCallingPid },
    { "getCallingUid", "()I", (void*)android_os_Binder_getCallingUid },
    { "clearCallingIdentity", "()J", (void*)android_os_Binder_clearCallingIdentity },
    { "restoreCallingIdentity", "(J)V", (void*)android_os_Binder_restoreCallingIdentity },
    { "setThreadStrictModePolicy", "(I)V", (void*)android_os_Binder_setThreadStrictModePolicy },
    { "getThreadStrictModePolicy", "()I", (void*)android_os_Binder_getThreadStrictModePolicy },
    { "flushPendingCommands", "()V", (void*)android_os_Binder_flushPendingCommands },
    { "init", "()V", (void*)android_os_Binder_init },
    { "destroy", "()V", (void*)android_os_Binder_destroy },
    { "blockUntilThreadAvailable", "()V", (void*)android_os_Binder_blockUntilThreadAvailable }
};
```

这种对应关系意味着：当Binder.java中的getCallingPid()方法被调用的时候，真正的实现其实是android_os_Binder_getCallingPic，当getCallUid方法被调用的时候，真正的实现其实是android_os_Binder_getCallingUid，其他类同。

然后我们再看一下android_os_Binder_getCallingPid方法的实现就会发现，这里其实就是对街道了libbinder中了：



```rust
static jint android_os_Binder_getCallingPid(JNIEnv* env, jobject clazz)
{
    return IPCThreadState::self()->getCallingPid();
}
```

**2、C++层代码调用Java代码**

上面看到了Java端代码是如何调用libbinder中的C++方法的。那么相反的方向是如何调用的？关键，libbinder中的** BBinder::onTransacts **是如何能能够调用到Java中的Binder:: onTransact的？

这段逻辑就是android_util_Binder.cpp中JavaBBinder::onTransact中处理的了。JavaBBinder是BBinder的子类，其类的结构如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-bf7dc0450d0ab7b9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

JavaBBinder类结构.png

JavaBBinder:: onTransact关键代码如下：



```cpp
//android_util_Binder      247行
virtual status_t onTransact(
   uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
{
   JNIEnv* env = javavm_to_jnienv(mVM);

   IPCThreadState* thread_state = IPCThreadState::self();
   const int32_t strict_policy_before = thread_state->getStrictModePolicy();

   jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
       code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
   ...
}
```

注意这一段代码：



```cpp
jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
  code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
```

这一行代码其实是在调用mObject上offset为mExecTransact的方法。这里的几个参数说明下：

> - mObject指向了Java端的Binder对象
> - gBinderOffsets.mExecTransact指向了Binder类的exectTransac方法
> - data调用了exectTransac方法的参数
> - code，data，reply，flag都是传递给调用方法execTransact参数

而JNIEnv.callBooleanMethod这个方法是由虚拟机实现的。即：虚拟机提供native方法来调用一个Java Object上方法。

**这样，就在C++层的JavaBBinder::onTransact中调用了Java层 Binder::execTransact方法。而在Binder::execTransact方法中，又调用了自身的onTransact方法，由此保证整个过程串联起来。**

## 二、初始化

在Android系统开始过程中，Zygote启东时会有一个"虚拟机注册过程"，该过程调用AndroidRuntime::startReg()方法来完成jni方法的注册

### 1、startReg()函数

```php
//  frameworks/base/core/jni/AndroidRuntime.cpp   1440行
int AndroidRuntime::startReg(JNIEnv* env)
{
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);
    env->PushLocalFrame(200);
    //核心函数   register_jni_procs()  注册jni方法
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);
    return 0;
}
```

>  

注册 JNI方法，其中gRegJNI是一个数组，记录所有需要注册的jni方法，其中有一项便是REG_JNI(register_android_os_Binder)。

**1.1 gRegJNI数组**

REG_JNI(register_android_os_Binder)在 ** frameworks/base/core/jni/AndroidRuntime.cpp  ** 的1312行，大家自行去查看吧。



```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp   1296行。
static const RegJNIRec gRegJNI[] = {
    ......  
    REG_JNI(register_android_os_SystemProperties),
    // *****  重点部分  *****
    REG_JNI(register_android_os_Binder),
    // *****  重点部分  *****
    REG_JNI(register_android_os_Parcel),
    ......  
};
```

**1.2 register_jni_procs() 函数**

```cpp
//  frameworks/base/core/jni/AndroidRuntime.cpp   1283行
    static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv*env) {
        for (size_t i = 0; i < count; i++) {
            if (array[i].mProc(env) < 0) {
                #ifndef NDEBUG
                ALOGD("----------!!! %s failed to load\n", array[i].mName);
                #endif
                return -1;
            }
        }
        return 0;
    }
```

那让我们继续看

**1.3 RegJNIRec数据结构**



```cpp
#ifdef NDEBUG
    #define REG_JNI(name)      { name }
   struct RegJNIRec {
                int (*mProc)(JNIEnv*);
            };
#else
    #define REG_JNI(name)      { name, #name }
    struct RegJNIRec {
              int (*mProc)(JNIEnv*);
              const char* mName;
            };
#endif
```

所以这里最终调用了register_android_os_Binder()函数，下面说说register_android_os_Binder过程。

### 2、register_android_os_Binder()函数

```kotlin
// frameworks/base/core/jni/android_util_Binder.cpp    1282行
int register_android_os_Binder(JNIEnv* env)
{
    // 注册Binder类的 jin方法
    if (int_register_android_os_Binder(env) < 0)
        return -1;

    // 注册 BinderInternal类的jni方法
    if (int_register_android_os_BinderInternal(env) < 0)
        return -1;

    // 注册BinderProxy类的jni方法
    if (int_register_android_os_BinderProxy(env) < 0)
        return -1;
    ...
    return 0;
}
```

这里面主要是三个注册方法

> - int_register_android_os_Binder()：注册Binder类的JNI方法
> - int_register_android_os_BinderInternal()：注册BinderInternal的JNI方法
> - int_register_android_os_BinderProxy()：注册BinderProxy类的JNI方法

那么就让我们依次研究下

**2.1 int_register_android_os_Binder()函数**



```cpp
// frameworks/base/core/jni/android_util_Binder.cpp    589行
static int int_register_android_os_Binder(JNIEnv* env)
{
    //kBinderPathName="android/os/Binder"，主要是查找kBinderPathName路径所属类
    jclass clazz = FindClassOrDie(env, kBinderPathName);
    //将Java层Binder类保存到mClass变量上
    gBinderOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    //将Java层execTransact()方法保存到mExecTransact变量；
    gBinderOffsets.mExecTransact = GetMethodIDOrDie(env, clazz, "execTransact", "(IJJI)Z");
    //将Java层的mObject属性保存到mObject变量中
    gBinderOffsets.mObject = GetFieldIDOrDie(env, clazz, "mObject", "J");
    //注册JNI方法
    return RegisterMethodsOrDie(env, kBinderPathName, gBinderMethods,
        NELEM(gBinderMethods));
}
```

PS: 注册Binder的JNI方法，其中：

> - FindClassOrDie(env, kBinderPathName)  等于   env->FindClass(kBinderPathName)
> - MakeGlobalRefOrDie()  等于  env->NewGlobalRef()
> - GetMethodIDOrDie() 等于 env->GetMethodID()
> - GetFieldIDOrDie() 等于 env->GeFieldID()
> - RegisterMethodsOrDie()  等于 Android::registerNativeMethods();

上面代码提到了gBinderOffsets，它是一个什么东西？

**2.1.1  gBinderOffsets：**

gBinderOffsets是全局静态结构体(struct)，定义如下：

```cpp
// frameworks/base/core/jni/android_util_Binder.cpp    65行
static struct bindernative_offsets_t
{
    // Class state.
    //记录 Binder类
    jclass mClass; 
    // 记录execTransact()方法
    jmethodID mExecTransact; 
    // Object state.
    // 记录mObject属性
    jfieldID mObject; 
} gBinderOffsets;
```

> gBinderOffsets保存了Binder.java类本身以及其成员方法execTransact()和成员属性mObject，这为JNI层访问Java层提供通道。另外通过查询获取Java层 binder信息后保存到gBinderOffsets，而不再需要每次查找binder类信息的方式能大幅提高效率，是由于每次查询需要花费较多的CPU时间，尤其是频繁访问时，但用额外的结构体来保存这些信息，是以空间换时间的方法。

**2.1.2  gBinderMethods：**

```cpp
// frameworks/base/core/jni/android_util_Binder.cpp    843行
static const JNINativeMethod gBinderMethods[] = {
     /* 名称, 签名, 函数指针 */
    { "getCallingPid", "()I", (void*)android_os_Binder_getCallingPid },
    { "getCallingUid", "()I", (void*)android_os_Binder_getCallingUid },
    { "clearCallingIdentity", "()J", (void*)android_os_Binder_clearCallingIdentity },
    { "restoreCallingIdentity", "(J)V", (void*)android_os_Binder_restoreCallingIdentity },
    { "setThreadStrictModePolicy", "(I)V", (void*)android_os_Binder_setThreadStrictModePolicy },
    { "getThreadStrictModePolicy", "()I", (void*)android_os_Binder_getThreadStrictModePolicy },
    { "flushPendingCommands", "()V", (void*)android_os_Binder_flushPendingCommands },
    { "init", "()V", (void*)android_os_Binder_init },
    { "destroy", "()V", (void*)android_os_Binder_destroy },
    { "blockUntilThreadAvailable", "()V", (void*)android_os_Binder_blockUntilThreadAvailable }
};
```

>  

通过RegisterMethodsOrDie()，将为gBinderMethods数组中的方法建立了一一映射关系，从而为Java层访问JNI层提供了通道。

**2.1.3  总结：**

总结，int_register_android_os_Binder方法的主要功能：

> - 通过gBinderOffsets，保存Java层Binder类的信息，为JNI层访问Java层提供了通道
> - 通过RegisterMethodsOrDie，将gBinderMethods数组完成映射关系，从而为Java层访问JNI层提供通道

也就是说该过程建立了Binder在Native层与framework之间的相互调用的桥梁。

**2.2 int_register_android_os_BinderInternal()函数**

注册 BinderInternal



```cpp
// frameworks/base/core/jni/android_util_Binder.cpp    935行
static int int_register_android_os_BinderInternal(JNIEnv* env)
{
    //其中kBinderInternalPathName
    jclass clazz = FindClassOrDie(env, kBinderInternalPathName);
    gBinderInternalOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderInternalOffsets.mForceGc = GetStaticMethodIDOrDie(env, clazz, "forceBinderGc", "()V");
    return RegisterMethodsOrDie(
        env, kBinderInternalPathName,
        gBinderInternalMethods, NELEM(gBinderInternalMethods));
}
```

注册 Binderinternal类的jni方法，gBinderInternaloffsets保存了BinderInternal的forceBinderGc()方法。

下面是BinderInternal类的JNI方法注册



```cpp
// frameworks/base/core/jni/android_util_Binder.cpp    925号
static const JNINativeMethod gBinderInternalMethods[] = {
    { "getContextObject", "()Landroid/os/IBinder;", (void*)android_os_BinderInternal_getContextObject },
    { "joinThreadPool", "()V", (void*)android_os_BinderInternal_joinThreadPool },
    { "disableBackgroundScheduling", "(Z)V", (void*)android_os_BinderInternal_disableBackgroundScheduling },
    { "handleGc", "()V", (void*)android_os_BinderInternal_handleGc }
};
```

>  

该过程 和**注册Binder类 JNI**非常类似，也就是说该过程建立了是BinderInternal类在Native层与framework层之间的相互调用的桥梁。

**2.3 int_register_android_os_BinderProxy()函数**



```cpp
// frameworks/base/core/jni/android_util_Binder.cpp     1254行
static int int_register_android_os_BinderProxy(JNIEnv* env)
{
    //gErrorOffsets 保存了Error类信息
    jclass clazz = FindClassOrDie(env, "java/lang/Error");
    gErrorOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    //gBinderProxyOffsets保存了BinderProxy类的信息
    //其中kBinderProxyPathName="android/os/BinderProxy"
    clazz = FindClassOrDie(env, kBinderProxyPathName);
    gBinderProxyOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderProxyOffsets.mConstructor = GetMethodIDOrDie(env, clazz, "<init>", "()V");
    gBinderProxyOffsets.mSendDeathNotice = GetStaticMethodIDOrDie(env, clazz, "sendDeathNotice", "(Landroid/os/IBinder$DeathRecipient;)V");
    gBinderProxyOffsets.mObject = GetFieldIDOrDie(env, clazz, "mObject", "J");
    gBinderProxyOffsets.mSelf = GetFieldIDOrDie(env, clazz, "mSelf", "Ljava/lang/ref/WeakReference;");
    gBinderProxyOffsets.mOrgue = GetFieldIDOrDie(env, clazz, "mOrgue", "J");
    //gClassOffsets保存了Class.getName()方法
    clazz = FindClassOrDie(env, "java/lang/Class");
    gClassOffsets.mGetName = GetMethodIDOrDie(env, clazz, "getName", "()Ljava/lang/String;");
    return RegisterMethodsOrDie(
        env, kBinderProxyPathName,
        gBinderProxyMethods, NELEM(gBinderProxyMethods));
}
```

注册BinderPoxy类的JNI方法，gBinderProxyOffsets保存了BinderProxy的构造方法，sendDeathNotice()，mObject，mSelf，mOrgue信息。

我们来看下gBinderProxyOffsets

**2.3.1 gBinderProxyOffsets结构体**



```cpp
// frameworks/base/core/jni/android_util_Binder.cpp     95行
static struct binderproxy_offsets_t {
        // Class state.
        // 对应的是 class对象 android.os.BinderProxy
        jclass mClass;
        // 对应的是  BinderProxy的构造函数
        jmethodID mConstructor;
        // 对应的是  BinderProxy的sendDeathNotice方法
        jmethodID mSendDeathNotice;

        // Object state.
        // 对应的是 BinderProxy的 mObject字段
        jfieldID mObject;
        // 对应的是 BinderProxy的mSelf字段
        jfieldID mSelf;
        // 对应的是 BinderProxymOrgue字段
        jfieldID mOrgue;
}   gBinderProxyOffsets;
```

PS: 这里补充下BinderProxy类是Binder类的内部类
 下面BinderProxy类的JNI方法注册：



```dart
// frameworks/base/core/jni/android_util_Binder.cpp     1241行
static const JNINativeMethod gBinderProxyMethods[] = {
     /* 名称, 签名, 函数指针 */
    {"pingBinder",          "()Z", (void*)android_os_BinderProxy_pingBinder},
    {"isBinderAlive",       "()Z", (void*)android_os_BinderProxy_isBinderAlive},
    {"getInterfaceDescriptor", "()Ljava/lang/String;", (void*)android_os_BinderProxy_getInterfaceDescriptor},
    {"transactNative",      "(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z", (void*)android_os_BinderProxy_transact},
    {"linkToDeath",         "(Landroid/os/IBinder$DeathRecipient;I)V", (void*)android_os_BinderProxy_linkToDeath},
    {"unlinkToDeath",       "(Landroid/os/IBinder$DeathRecipient;I)Z", (void*)android_os_BinderProxy_unlinkToDeath},
    {"destroy",             "()V", (void*)android_os_BinderProxy_destroy},
};
```

该过程上面非常类似，也就是说该过程建立了是BinderProxy类在Native层与framework层之间的相互调用的桥梁。

## 三、注册服务

注册服务在ServiceManager里面



```java
//frameworks/base/core/java/android/os/ServiceManager.java     70行
public static void addService(String name, IBinder service, boolean allowIsolated) {
    try {
        //getIServiceManager()是获取ServiceManagerProxy对象
        // addService() 是执行注册服务操作
        getIServiceManager().addService(name, service, allowIsolated); 
    } catch (RemoteException e) {
        Log.e(TAG, "error in addService", e);
    }
}
```

### (一) 、先来看下getIServiceManager()方法

```csharp
//frameworks/base/core/java/android/os/ServiceManager.java     70行
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }
        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }
```

通过上面，大家的第一反应应该是sServiceManager是单例的。第二反映是如果想知道sServiceManager的值，必须了解**BinderInternal.getContextObject()的返回值**和 **ServiceManagerNative.asInterface()**方法的内部执行，那我们就来详细了解下

**1、先来看下BinderInternal.getContextObject()方法**

```dart
//frameworks/base/core/java/com/android/internal/os/BinderInternal.java  88行
    /**
     * Return the global "context object" of the system.  This is usually
     * an implementation of IServiceManager, which you can use to find
     * other services.
     */
    public static final native IBinder getContextObject();
```

可见BinderInternal.getContextObject()最终会调用JNI通过C层来实现，那我们就继续跟踪

**1.1、android_os_BinderInternal_getContextObject)函数**

```php
// frameworks/base/core/jni/android_util_Binder.cpp     899行
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);  
}
```

看到上面的代码 大家有没有熟悉的感觉，前面讲过了：对于ProcessState::self() -> getContextObject()
 对于ProcessState::self()->getContextObject()可以理解为new BpBinder(0)，那就剩下 **javaObjectForIBinder(env, b)** 那我们就来看下这个函数

**1.2、javaObjectForIBinder()函数**

```cpp
// frameworks/base/core/jni/android_util_Binder.cpp         547行
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    if (val == NULL) return NULL;
    //返回false
    if (val->checkSubclass(&gBinderOffsets)) { 
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        return object;
    }

    AutoMutex _l(mProxyLock);

    jobject object = (jobject)val->findObject(&gBinderProxyOffsets);
    //第一次 object为null
    if (object != NULL) { 
        //查找是否已经存在需要使用的BinderProxy对应，如果有，则返回引用。
        jobject res = jniGetReferent(env, object);
        if (res != NULL) {
            return res;
        }
        android_atomic_dec(&gNumProxyRefs);
        val->detachObject(&gBinderProxyOffsets);
        env->DeleteGlobalRef(object);
    }

    //创建BinderProxy对象
    object = env->NewObject(gBinderProxyOffsets.mClass, gBinderProxyOffsets.mConstructor);
    if (object != NULL) {
        // BinderProxy.mObject成员变量记录BpBinder对象
        env->SetLongField(object, gBinderProxyOffsets.mObject, (jlong)val.get());
        val->incStrong((void*)javaObjectForIBinder);

        jobject refObject = env->NewGlobalRef(
                env->GetObjectField(object, gBinderProxyOffsets.mSelf));
         //将BinderProxy对象信息附加到BpBinder的成员变量mObjects中
        val->attachObject(&gBinderProxyOffsets, refObject,
                jnienv_to_javavm(env), proxy_cleanup);

        sp<DeathRecipientList> drl = new DeathRecipientList;
        drl->incStrong((void*)javaObjectForIBinder);
         // BinderProxy.mOrgue成员变量记录死亡通知对象
        env->SetLongField(object, gBinderProxyOffsets.mOrgue, reinterpret_cast<jlong>(drl.get()));

        android_atomic_inc(&gNumProxyRefs);
        incRefsCreated(env);
    }
    return object;
}
```

上面的大致流程如下：

> - 1、第二个入参val在有些时候指向BpBinder，有些时候指向JavaBBinder
> - 2、至于是BpBinder还是JavaBBinder是通过if (val->checkSubclass(&gBinderOffsets)) 这个函数来区分的，如果是JavaBBinder，则为true，则就会通过成员函数object()，返回一个Java对象，这个对象就是Java层的Binder对象。由于我们这里是BpBinder，所以是 ** 返回false**
> - 3如果是BpBinder，会先判断是不是第一次，如果是第一次,下面的object为null。



```kotlin
jobject object = (jobject)val->findObject(&gBinderProxyOffsets);
```

如果不是第一次，就会先查找是否已经存在需要使用的BinderProxy对象，如果找到就会返回引用

> - 4、如果没有找到可用的引用，就new一个BinderProxy对象

所以主要是根据BpBinder(C++) 生成BinderProxy(Java对象)，主要工作是创建BinderProxy对象，并把BpBinder对象地址保存到BinderProxy.mObject成员变量。到此，可知ServiceManagerNative.asInterface(BinderInternal.getContextObject()) 等价于



```cpp
ServiceManagerNative.asInterface(new BinderProxy())
```

**2、再来看下ServiceManagerNative.asInterface()方法**

```csharp
//frameworks/base/core/java/android/os/ServiceManagerNative.java      33行
    /**
     * Cast a Binder object into a service manager interface, generating
     * a proxy if needed.
     * 将Binder对象转换service manager interface，如果需要，生成一个代理。
     */
    static public IServiceManager asInterface(IBinder obj)
    {
        //obj为 BpBinder
        // 如果 obj为null 则直接返回
        if (obj == null) {
            return null;
        }
        // 由于是BpBinder，所以BpBinder的queryLocalInterface(descriptor) 默认返回null
        IServiceManager in =
            (IServiceManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        return new ServiceManagerProxy(obj);
    }
```

我们看下这个obj.queryLocalInterface(descriptor)方法，其实他是调用的IBinder的native方法如下



```dart
public interface IBinder {
    .....
    /**
     * Attempt to retrieve a local implementation of an interface
     * for this Binder object.  If null is returned, you will need
     * to instantiate a proxy class to marshall calls through
     * the transact() method.
     */
    public IInterface queryLocalInterface(String descriptor);
    .....
}
```

> 通过注释我们知道,queryLocalInterface是查询本地的对象，我简单解释下什么是本地对象，这里的本地对象是指，如果进行IPC调用，如果是两个进程是同一个进程，即对象是本地对象；如果两个进程是两个不同的进程，则返回的远端的代理类。所以在BBinder的子类BnInterface中，重载了这个方法，返回this，而在BpInterface并没有重载这个方法。又因为queryLocalInterface 默认返回的是null，所以obj.queryLocalInterface=null。
>  所以最后结论是 **return new ServiceManagerProxy(obj);**

那我们来看下ServiceManagerProxy

**2.1、ServiceManagerProxy**

PS:ServiceManagerProxy是ServiceManagerNative类的内部类

```java
//frameworks/base/core/java/android/os/ServiceManagerNative.java    109行
class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
    }
}
```

> mRemote为BinderProxy对象，该BinderProxy对象对应于BpBinder(0)，其作为binder代理端，指向native的层的Service Manager。

所以说：

> ServiceManager.getIServiceManager最终等价于new ServiceManagerProxy(new BinderProxy())。所以

```css
 getIServiceManager().addService()
```

等价于

```css
ServiceManagerNative.addService();
```

framework层的ServiceManager的调用实际的工作确实交给了ServiceManagerProxy的成员变量BinderProxy；而BinderProxy通过JNI的方式，最终会调用BpBinder对象；可见上层binder结构的核心功能依赖native架构的服务来完成的。

### (二) addService()方法详解

上面已经知道了

```css
getIServiceManager().addService(name, service, allowIsolated); 
```

等价于

```css
ServiceManagerProxy..addService(name, service, allowIsolated);
```

> PS:上面的ServiceManagerProxy代表ServiceManagerProxy对象

所以让我们来看下ServiceManagerProxy的addService()方法

**1、ServiceManagerProxy的addService()**

```kotlin
//frameworks/base/core/java/android/os/ServiceManagerNative.java     142行
    public void addService(String name, IBinder service, boolean allowIsolated)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        //是个常量是 “android.os.IServiceManager"
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        data.writeStrongBinder(service);
        data.writeInt(allowIsolated ? 1 : 0);
        mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
        reply.recycle();
        data.recycle();
    }
```

里面代码都比较容易理解，这里重点说下**data.writeStrongBinder(service);** 和 **mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);**

**2、Parcel.writeStrongBinder()**

那我们就来看下Parcel的writeStrongBinder()方法

```cpp
//frameworks/base/core/java/android/os/Parcel.java     583行
    /**
     * Write an object into the parcel at the current dataPosition(),
     * growing dataCapacity() if needed.
     */
    public final void writeStrongBinder(IBinder val) {
        nativeWriteStrongBinder(mNativePtr, val);
    }
```

先看下注释，翻译一下

> 在当前的dataPosition()的位置上写入一个对象，如果空间不足，则增加空间

通过上面代码我们知道 **writeStrongBinder()** 方法里面实际是调用的 **nativeWriteStrongBinder()** 方法，那我们来看下 **nativeWriteStrongBinder()** 方法

**2.1  nativeWriteStrongBinder()方法**



```csharp
/frameworks/base/core/java/android/os/Parcel.java      265行
    private static native void nativeWriteStrongBinder(long nativePtr, IBinder val);
```

我们知道了nativeWriteStrongBinder是一个native方法，那我们继续跟踪

**2.2  android_os_Parcel_writeStrongBinder()函数**



```cpp
//frameworks/base/core/jni/android_os_Parcel.cpp    298行
static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object)
{
    //将java层Parcel转换为native层Parcel
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
```

这里主要涉及的两个重要的函数

> - writeStrongBinder()函数
> - ibinderForJavaObject()函数

那我们就来详细研究这两个函数

**2.2.1  ibinderForJavaObject()函数**



```php
// frameworks/base/core/jni/android_util_Binder.cpp   603行
sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;

    //Java层的Binder对象
    //mClass指向Java层中的Binder class
    if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env->GetLongField(obj, gBinderOffsets.mObject);
        //get()返回一个JavaBBinder，继承自BBinder
        return jbh != NULL ? jbh->get(env, obj) : NULL;
    }
    //Java层的BinderProxy对象
    // mClass 指向Java层的BinderProxy class
    if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        //返回一个 BpBinder，mObject 是它的地址值
        return (IBinder*)env->GetLongField(obj, gBinderProxyOffsets.mObject);
    }
    return NULL;
}
```

根据Binder(Java)生成JavaBBinderHolder(C++)对象，主要工作是创建JavaBBinderHolder对象，并把JavaBBinder对象保存在到Binder.mObject成员变量。

> - 这个函数，本质就是根据传进来的Java对象找到对应的C++对象，这里的obj可能会指向两种对象:Binder对象和BinderProxy对象。
> - 如果传进来的是Binder对象，则会把gBinderOffsets.mObject转化为JavaBBinderHolder，并从中获得一个JavaBBinder对象(JavaBBinder继承自BBinder)。
> - 如果是BinderProxy对象，会返回一个BpBinder，这个BpBinder的地址值保存在gBinderProxyOffsets.mObject中

在上面的代码里面调用了get()函数，如下图



```php
JavaBBinderHolder* jbh = (JavaBBinderHolder*)
env->GetLongField(obj, gBinderOffsets.mObject);
//get()返回一个JavaBBinder，继承自BBinder
return jbh != NULL ? jbh->get(env, obj) : NULL;
```

那我们就来研究下JavaBBinderHolder.get()函数

**2.2.1.1 JavaBBinderHolder.get()函数**



```java
// frameworks/base/core/jni/android_util_Binder.cpp      316行
sp<JavaBBinder> get(JNIEnv* env, jobject obj)
{
    AutoMutex _l(mLock);
    sp<JavaBBinder> b = mBinder.promote();
    if (b == NULL) {
        //首次进来，创建JavaBBinder对象
        b = new JavaBBinder(env, obj);
        mBinder = b;
    }
    return b;
}
```

> JavaBBinderHolder有一个成员变量mBinder，保存当前创建的JavaBBinder对象，这是一个wp类型的，可能会被垃圾回收器给回收的，所以每次使用前都需要先判断是否存在。

那我们再来看看下JavaBBinder的初始化

**2.2.1.2 JavaBBinder的初始化**



```kotlin
JavaBBinder(JNIEnv* env, jobject object)
    : mVM(jnienv_to_javavm(env)), mObject(env->NewGlobalRef(object))
{
    ALOGV("Creating JavaBBinder %p\n", this);
    android_atomic_inc(&gNumLocalRefs);
    incRefsCreated(env);
}
```

创建JavaBBinder，该对象继承于BBinder对象。

**2.2.1.3  总结**

> 所以说 data.writeStrongBinder(Service)最终等价于parcel->writeStringBinder(new JavaBBinder(env, obj));

**2.2.2、writeStrongBinder() 函数**



```kotlin
// frameworks/native/libs/binder/Parcel.cpp     872行
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}
```

我们看到writeStrongBinder()函数 实际上是调用的flatten_binder()函数

**2.2.2.1、 writeStrongBinder() 函数**



```cpp
//frameworks/native/libs/binder/Parcel.cpp    205行
status_t flatten_binder(const sp<ProcessState>& /*proc*/,
    const sp<IBinder>& binder, Parcel* out)
{
    flat_binder_object obj;
    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    if (binder != NULL) {
        IBinder *local = binder->localBinder();
        if (!local) {
            //如果不是本地Binder
            BpBinder *proxy = binder->remoteBinder();
            const int32_t handle = proxy ? proxy->handle() : 0;
            //远程Binder
            obj.type = BINDER_TYPE_HANDLE; 
            obj.binder = 0; 
            obj.handle = handle;
            obj.cookie = 0;
        } else {
            //如果是本地Binder
            obj.type = BINDER_TYPE_BINDER; 
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    } else {
        //本地Binder
        obj.type = BINDER_TYPE_BINDER;  
        obj.binder = 0;
        obj.cookie = 0;
    }
    return finish_flatten_binder(binder, obj, out);
}
```

将Binder对象扁平化，转换成flat_binder_object对象

> - 对于Binder实体，则cookie记录Binder实体指针
> - 对于Binder代理，则用handle记录Binder代理的句柄

关于localBinder，在Binder.cpp里面



```cpp
BBinder* BBinder::localBinder()
{
    return this;
}

BBinder* IBinder::localBinder()
{
    return NULL;
}
```

在最后面调用了finish_flatten_binder()函数，那我们再研究下finish_flatten_binder()函数

**2.2.2.2、 finish_flatten_binder() 函数**



```csharp
//frameworks/native/libs/binder/Parcel.cpp    199行
inline static status_t finish_flatten_binder(
    const sp<IBinder>& , const flat_binder_object& flat, Parcel* out)
{
    return out->writeObject(flat, false);
}
```

这个大家看明白了吧，就是写入一个object。

**Parcel.writeStrongBinder()整个流程已经结束了。下面让我回来看下**

ServiceManagerProxy的addService()中的mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);

**3、IBinder.transact()**

> ServiceManagerProxy的addService()中的mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);里面的mRemote的类型是BinderProxy的，所以调用是BinderProxy的transact()方法，那我们就进去看看

**3.1、BinderProxy.transact()**

**温馨提示：BinderProxy类是Binder类的内部类**

他其实是重写的IBinder的里面的transact()方法，那让我们看下IBinder里面



```dart
// frameworks/base/core/java/android/os/IBinder.java  223行
    /**
     * Perform a generic operation with the object.
     * 
     * @param code The action to perform.  This should
     * be a number between {@link #FIRST_CALL_TRANSACTION} and
     * {@link #LAST_CALL_TRANSACTION}.
     * @param data Marshalled data to send to the target.  Must not be null.
     * If you are not sending any data, you must create an empty Parcel
     * that is given here.
     * @param reply Marshalled data to be received from the target.  May be
     * null if you are not interested in the return value.
     * @param flags Additional operation flags.  Either 0 for a normal
     * RPC, or {@link #FLAG_ONEWAY} for a one-way RPC.
     */
    public boolean transact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException;
```

其实让大家看这个主要是向大家说下这个注释，(*__*) 嘻嘻……
 翻译一下

> - 用对象执行一个操作
> - 参数code 为操作码，是介于FIRST_CALL_TRANSACTION和LAST_CALL_TRANSACTION之间
> - 参数data 是要发往目标的数据，一定不能null，如果你没有数据要发送，你也要创建一个Parcel，哪怕是空的。
> - 参数reply 是从目标发过来的数据，如果你对这个数据没兴趣，这个数据是可以为null的。
> - 参数flags 一个操作标志位，要么是0代表普通的RPC，要么是FLAG_ONEWAY代表单一方向的RPC即不管返回值

这时候我们再回来看



```java
/frameworks/base/core/java/android/os/Binder.java   501行
final class BinderProxy implements IBinder {
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        if (Binder.isTracingEnabled()) { Binder.getTransactionTracker().addTrace(); }
        return transactNative(code, data, reply, flags);
    }
}
```

先来看下 Binder.checkParcel方法

**3.1.1、Binder.checkParcel()**



```go
/frameworks/base/core/java/android/os/Binder.java   415行
    static void checkParcel(IBinder obj, int code, Parcel parcel, String msg) {
        if (CHECK_PARCEL_SIZE && parcel.dataSize() >= 800*1024) {
            // Trying to send > 800k, this is way too much
            StringBuilder sb = new StringBuilder();
            sb.append(msg);
            sb.append(": on ");
            sb.append(obj);
            sb.append(" calling ");
            sb.append(code);
            sb.append(" size ");
            sb.append(parcel.dataSize());
            sb.append(" (data: ");
            parcel.setDataPosition(0);
            sb.append(parcel.readInt());
            sb.append(", ");
            sb.append(parcel.readInt());
            sb.append(", ");
            sb.append(parcel.readInt());
            sb.append(")");
            Slog.wtfStack(TAG, sb.toString());
        }
    }
```

这段代码很简单，主要是检查Parcel大小是否大于800K。

执行完Binder.checkParcel后，直接调用了transactNative()方法，那我们就来看看transactNative()方法

**3.1.2、transactNative()方法**



```java
// frameworks/base/core/java/android/os/Binder.java   507行
    public native boolean transactNative(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException;
```

我们看到他是一个native函数，后面肯定经过JNI调用到了native层，根据包名，它对应的方法应该是"android_os_BinderProxy_transact"函数，那我们继续跟踪

**3.1.3、android_os_BinderProxy_transact()函数**



```cpp
// frameworks/base/core/jni/android_util_Binder.cpp     1083行
   static jboolean android_os_BinderProxy_transact(JNIEnv*env, jobject obj,
                                                    jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
    {
        if (dataObj == NULL) {
            jniThrowNullPointerException(env, NULL);
            return JNI_FALSE;
        }
        // 将 java Parcel转化为native Parcel
        Parcel * data = parcelForJavaObject(env, dataObj);
        if (data == NULL) {
            return JNI_FALSE;
        }
        Parcel * reply = parcelForJavaObject(env, replyObj);
        if (reply == NULL && replyObj != NULL) {
            return JNI_FALSE;
        }
        // gBinderProxyOffsets.mObject中保存的是new BpBinder(0)对象
        IBinder * target = (IBinder *)
        env -> GetLongField(obj, gBinderProxyOffsets.mObject);
        if (target == NULL) {
            jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
            return JNI_FALSE;
        }

        ALOGV("Java code calling transact on %p in Java object %p with code %"PRId32"\n",
                target, obj, code);


        bool time_binder_calls;
        int64_t start_millis;
        if (kEnableBinderSample) {
            // Only log the binder call duration for things on the Java-level main thread.
            // But if we don't
            time_binder_calls = should_time_binder_calls();

            if (time_binder_calls) {
                start_millis = uptimeMillis();
            }
        }

        //printf("Transact from Java code to %p sending: ", target); data->print();
         // 此处便是BpBinder:: transact()，经过native层，进入Binder驱动。
        status_t err = target -> transact(code, * data, reply, flags);
        //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();

        if (kEnableBinderSample) {
            if (time_binder_calls) {
                conditionally_log_binder_call(start_millis, target, code);
            }
        }

        if (err == NO_ERROR) {
            return JNI_TRUE;
        } else if (err == UNKNOWN_TRANSACTION) {
            return JNI_FALSE;
        }
        signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data -> dataSize());
        return JNI_FALSE;
    }
```

通过上面的代码我们知道，Java层BinderProxy.transact()最终交由Native层的BpBinder::transact()完成。这部分之前代码讲解过了，我这里就不详细说明了。不过注意，该方法可能会抛出RemoteException。

### (三)、总结

所以 整个 addService的核心可以缩写为向下面的代码



```java
public void addService(String name, IBinder service, boolean allowIsolated)
        throws RemoteException {
    ...
     //此处还需要将Java层的Parcel转化为Native层的Parcel
    Parcel data = Parcel.obtain();
    data->writeStrongBinder(new JavaBBinder(env, obj));
    //与Binder驱动交互
    BpBinder::transact(ADD_SERVICE_TRANSACTION, *data, reply, 0);
    ...
}
```

说白了，注册服务过程就是通过BpBinder来发送ADD_SERVICE_TRANSACTION命令，与binder驱动进行数据交互。

## 四、获取服务

### (一)、ServiceManager.getService()方法

```kotlin
//frameworks/base/core/java/android/os/ServiceManager.java    49行
public static IBinder getService(String name) {
    try {
        //先从缓存中查看
        IBinder service = sCache.get(name); 
        if (service != null) {
            return service;
        } else {
            return getIServiceManager().getService(name); 
        }
    } catch (RemoteException e) {
        Log.e(TAG, "error in getService", e);
    }
    return null;
}
```

> - 1、先从缓存中取出，如果有，则直接return。其中sCache是以HashMap格式的缓存
> - 2、如果没有调用getIServiceManager().getService(name)获取一个，并且return
>    通过前面的内容我们知道



```undefined
getIServiceManager()
```

等价于



```cpp
new  ServiceManagerProxy(new BinderProxy())
```

那我们来下ServiceManagerProxy的getService()方法

**1、ServiceManagerProxy.getService(name)**

```kotlin
// frameworks/base/core/java/android/os/ServiceManagerNative.java     118行
    public IBinder getService(String name) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        //mRemote为BinderProxy
        mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
        //从replay里面解析出获取的IBinder对象
        IBinder binder = reply.readStrongBinder();
        reply.recycle();
        data.recycle();
        return binder;
    }
```

这里面有两个重点方法，一个是 mRemote.transact(),一个是 reply.readStrongBinder()。那我们就逐步研究下

**2、mRemote.transact()方法**

我们mRemote其实是BinderPoxy，那我们来看下BinderProxy的transact方法



```java
     //frameworks/base/core/java/android/os/Binder.java   501行
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        if (Binder.isTracingEnabled()) { Binder.getTransactionTracker().addTrace(); }
        return transactNative(code, data, reply, flags);
    }

     // frameworks/base/core/java/android/os/Binder.java   507行
    public native boolean transactNative(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException;
```

关于Binder.checkParcel()方法，上面已经说过了，就不详细说了。transact()方法其实是调用了natvie的transactNative()方法，这样就进入了JNI里面了

**2.1、mRemote.transact()方法**



```cpp
// frameworks/base/core/jni/android_util_Binder.cpp     1083行
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    if (dataObj == NULL) {
        jniThrowNullPointerException(env, NULL);
        return JNI_FALSE;
    }
    //Java的 Parcel 转为native的 Parcel
    Parcel* data = parcelForJavaObject(env, dataObj);
    if (data == NULL) {
        return JNI_FALSE;
    }
    Parcel* reply = parcelForJavaObject(env, replyObj);
    if (reply == NULL && replyObj != NULL) {
        return JNI_FALSE;
    }
    // gBinderProxyOffsets.mObject中保存的的是new BpBinder(0)对象
    IBinder* target = (IBinder*)
        env->GetLongField(obj, gBinderProxyOffsets.mObject);
    if (target == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
        return JNI_FALSE;
    }

    ALOGV("Java code calling transact on %p in Java object %p with code %" PRId32 "\n",
            target, obj, code);

    bool time_binder_calls;
    int64_t start_millis;
    if (kEnableBinderSample) {
        // Only log the binder call duration for things on the Java-level main thread.
        // But if we don't
        time_binder_calls = should_time_binder_calls();

        if (time_binder_calls) {
            start_millis = uptimeMillis();
       }
    }

    //printf("Transact from Java code to %p sending: ", target); data->print();
    //gBinderProxyOffseets.mObject中保存的是new BpBinder(0) 对象
    status_t err = target->transact(code, *data, reply, flags);
    //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();

    if (kEnableBinderSample) {
        if (time_binder_calls) {
            conditionally_log_binder_call(start_millis, target, code);
        }
    }

    if (err == NO_ERROR) {
        return JNI_TRUE;
    } else if (err == UNKNOWN_TRANSACTION) {
        return JNI_FALSE;
    }

    signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data->dataSize());
    return JNI_FALSE;
}
```

上面代码中，有一段重点代码



```cpp
status_t err = target->transact(code, *data, reply, flags);
```

现在 我们看一下他里面的事情

**2.2、BpBinder::transact()函数**



```cpp
/frameworks/native/libs/binder/BpBinder.cpp    159行
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
    return DEAD_OBJECT;
}
```

其实是调用的IPCThreadState的transact()函数

**2.3、BpBinder::transact()函数**



```cpp
//frameworks/native/libs/binder/IPCThreadState.cpp    548行
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck(); //数据错误检查
    flags |= TF_ACCEPT_FDS;
    ....
    if (err == NO_ERROR) {
         // 传输数据
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    ...

    // 默认情况下,都是采用非oneway的方式, 也就是需要等待服务端的返回结果
    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            //等待回应事件
            err = waitForResponse(reply);
        }else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
```

主要就是两个步骤

> - 首先，调用writeTransactionData()函数 传输数据
> - 其次，调用waitForResponse()函数来获取返回结果

那我们来看下waitForResponse()函数里面的重点实现

**2.4、IPCThreadState::waitForResponse函数**



```cpp
//frameworks/native/libs/binder/IPCThreadState.cpp    712行
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;
    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break; 
        ...
        cmd = mIn.readInt32();
        switch (cmd) {
          case BR_REPLY:
          {
            binder_transaction_data tr;
            err = mIn.read(&tr, sizeof(tr));
            if (reply) {
                if ((tr.flags & TF_STATUS_CODE) == 0) {
                    //当reply对象回收时，则会调用freeBuffer来回收内存
                    reply->ipcSetDataReference(
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t),
                        freeBuffer, this);
                } else {
                    ...
                }
            }
          }
          case :...
        }
    }
    ...
    return err;
}
```

这时候就在等待回复了，如果有回复，则通过cmd = mIn.readInt32()函数获取命令

**2.5、IPCThreadState::waitForResponse函数**



```kotlin
//
void binder_send_reply(struct binder_state *bs,
                       struct binder_io *reply,
                       binder_uintptr_t buffer_to_free,
                       int status)
{
    struct {
        uint32_t cmd_free;
        binder_uintptr_t buffer;
        uint32_t cmd_reply;
        struct binder_transaction_data txn;
    } __attribute__((packed)) data;
    //free buffer命令
    data.cmd_free = BC_FREE_BUFFER; 
    data.buffer = buffer_to_free;
    // reply命令
    data.cmd_reply = BC_REPLY; // reply命令
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
        ...
    } else {=
    
        data.txn.flags = 0;
        data.txn.data_size = reply->data - reply->data0;
        data.txn.offsets_size = ((char*) reply->offs) - ((char*) reply->offs0);
        data.txn.data.ptr.buffer = (uintptr_t)reply->data0;
        data.txn.data.ptr.offsets = (uintptr_t)reply->offs0;
    }
    //向Binder驱动通信
    binder_write(bs, &data, sizeof(data));
}
```

binder_write将BC_FREE_BUFFER和BC_REPLY命令协议发送给驱动，进入驱动。
 在驱动里面bingder_ioctl -> binder_ioctl_write_read ->binder_thread_write，由于是BC_REPLY命令协议，则进入binder_transaction，该方法会向请求服务的线程TODO队列插入事务。接来下，请求服务的进程在执行talkWithDriver的过程执行到binder_thread_read()，处理TODO队列的事物。

**3、Parcel.readStrongBinder()方法**

其实Parcel.readStrongBinder()的过程基本上就是writeStrongBinder的过程。
 我们先来看下它的源码



```java
//frameworks/base/core/java/android/os/Parcel.java    1686行
    /**
     * Read an object from the parcel at the current dataPosition().
     * 在当前的 dataPosition()位置上读取一个对象
     */
    public final IBinder readStrongBinder() {
        return nativeReadStrongBinder(mNativePtr);
    }


  private static native IBinder nativeReadStrongBinder(long nativePtr);
```

其实它内部是调用的是nativeReadStrongBinder()方法，通过上面的源码我们知道nativeReadStrongBinder是一个native的方法，所以通过JNI调用到android_os_Parcel_readStrongBinder这个函数

**3.1、android_os_Parcel_readStrongBinder()函数**



```cpp
//frameworks/base/core/jni/android_os_Parcel.cpp          429行
static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        return javaObjectForIBinder(env, parcel->readStrongBinder());
    }
    return NULL;
}
```

javaObjectForIBinder将native层BpBinder对象转换为Java层的BinderProxy对象。
 上面的函数中，调用了readStrongBinder()函数

**3.2、readStrongBinder()函数**



```kotlin
//frameworks/native/libs/binder/Parcel.cpp  1334行
sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    unflatten_binder(ProcessState::self(), *this, &val);
    return val;
}
```

这里面也很简单，主要是调用unflatten_binder()函数

**3.3、unflatten_binder()函数**



```cpp
//frameworks/native/libs/binder/Parcel.cpp  293行
status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);
    if (flat) {
        switch (flat->type) {
            case BINDER_TYPE_BINDER:
                *out = reinterpret_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                //进入该分支
                *out = proc->getStrongProxyForHandle(flat->handle);
                //创建BpBinder对象
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
```

>  

PS:在/frameworks/native/libs/binder/Parcel.cpp/frameworks/native/libs/binder/Parcel.cpp 里面有两个unflatten_binder()函数，其中区别点是，最后一个入参，一个是sp<IBinder>* out，另一个是wp<IBinder>* out。大家别弄差了。

在unflatten_binder里面进入 case BINDER_TYPE_HANDLE: 分支，然后执行getStrongProxyForHandle()函数。

**3.4、getStrongProxyForHandle()函数**



```php
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);
    //查找handle对应的资源项
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            ...
            //当handle值所对应的IBinder不存在或弱引用无效时，则创建BpBinder对象
            b = new BpBinder(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}
```

经过该方法，最终创建了指向Binder服务端的BpBinder代理对象。所以说javaObjectForIBinder将native层的BpBinder对象转化为Java层BinderProxy对象。也就是说通过getService()最终取得了指向目标Binder服务器的代理对象BinderProxy。

**4、总结**

所以说getService的核心过程：



```cpp
public static IBinder getService(String name) {
    ...
    //此处还需要将Java层的Parcel转化为Native层的Parcel
    Parcel reply = Parcel.obtain(); 
    // 与Binder驱动交互
    BpBinder::transact(GET_SERVICE_TRANSACTION, *data, reply, 0);  
    IBinder binder = javaObjectForIBinder(env, new BpBinder(handle));
    ...
}
```

javaObjectForIBinder作用是创建BinderProxy对象，并将BpBinder对象的地址保存到BinderProxy对象的mObjects中，获取服务过程就是通过BpBinder来发送GET_SERVICE_TRANSACTION命令，实现与binder驱动进行数据交互。





# Android跨进程通信IPC之AIDL

## 一、AIDL简介

> AIDL是一个缩写，全程是Android Interface Definition Language，也是android接口定义语言。准确的来说，它是用于定义客户端/服务器通信接口的一种描述语言。它其实一种IDL语言，可以拿来生成用于IPC的代码。从某种意义上说它其实是一个模板。为什么这么说？因为在我们使用中，实际起作用的并不是我们写的AIDL代码，而是系统根据它生成的一个IInterface的实例的代码。而如果大家都生成这样的几个实例，然后它们拿来比较，你会发现它们都是有套路的——都是一样的流程，一样的结构，只是根据具体的AIDL文件的不同由细微变动。所以其实AIDL就是为了避免我们一遍遍的写一些前篇一律的代码而出现的一个模板

在详解讲解AIDL之前，大家想一想下面几个问题？

> - 1、那么为什么安卓团队要定义这一种语言。
> - 2、如果使用AIDL
> - 3、AIDL的原理

那我们开始围绕这三个问题开始一次接待

## 二、为什么要设置AIDL

两个维度来看待这个问题：

### (一) IPC的角度

设计这门语言的目的是为了实现进程间通信，尤其是在涉及多进程并发情况的下的进程间通信IPC。每一个进程都有自己的Dalvik VM实例，都有自己的一块独立的内存，都在自己的内存上存储自己的数据，执行着自己的操作，都在自己的那个空间里操作。每个进程都是独立的，你不知我，我不知你。就像两座小岛之间的桥梁。通过这个桥梁，两个小岛可以进行交流，进行信息的交互。

> 通过AIDL，可以让本地调用远程服务器的接口就像调用本地接口那么接单，让用户无需关注内部细节，只需要实现自己的业务逻辑接口，内部复杂的参数序列化发送、接收、客户端调用服务端的逻辑，你都不需要去关心了。

### (二)方便角度

> 在Android process 之间不能用通常的方式去访问彼此的内存数据。他们把需要传递的数据解析成基础对象，使得系统能够识别并处理这些对象。因为这个处理过程很难写，所以Android使用AIDL来解决这个问题

## 三、ADIL的注意事项

在定义AIDL之前，请意识到调用这些接口是direct function call。请不要认为call这些接口的行为是发生在另外一个线程里面的。具体的不同因为这个调用是发生在local process还是 remote process而异。

> - 发生在local process 里面的调用会跑在这个local process的thread里面。如果这是你的UI主线程，那么AIDL接口的调用也会发生在这个UIthread里面。如果这是发生在另外一个thread，那么调用会发生在service里面。因此，如果仅仅是发生在local process的调用，则你可以完全控制这些调用，当然这样的话，就不需要AIDL了。因为你完全可以使用Bound Service的第一种方式去实现。
> - 发生在remote process 里面调用的会跑在你自己的process所维护的thread pool里面。那么你需要注意可能会在同一时刻接收到多个请求。所以AIDL的操作需要做到thread-safe。(每次请求，都交给Service，在线程池里面启动一个thread去执行哪些请求，所以那些方法需要是线程安全的)
> - oneway关键字改变了remote cal的行为。当使用这个关键字时，remote call 不会被阻塞住，它仅仅是发送交互数据后再立即返回。IBinder thread pool之后会把它当做一个通常的remote call。

## 四、AIDL的使用

### (一)、什么时候使用AIDL

前面我们介绍了，Binder机制，还有后面要讲解的Messager，以及现在说的AIDL等，Android系统中有事先IPC的很多中方式，到底什么情况下应该使用AIDL？Android官方文档给出了答案：

> Note: Using AIDL is necessary only if you allow clients from different applications to access your service for IPC and want to handle multithreading in your service. If you do not need to perform concurrent IPC across different applications, you should create your interface by implementing a Binder or, if you want to perform IPC, but do not need to handle multithreading, implement your interface using a Messenger. Regardless, be sure that you understand Bound Services before implementing an AIDL.

所以使用AIDL只有在你先允许来自不同应用的客户端跨进程通信访问你的Service，并且想要在你的Service处理多线程的时候才是必要的。简单的来说，就是多个客户端，多个线程并发的情况下要使用AIDL。官方文档还之处，如果你的IPC不需要适用多个客户端，就用Binder。如果你想要IPC，但是不需要多线程，那就选择Messager。

### (二)、建立AIDL

我们以在Android Studio为例进行讲解

**1、创建AIDL文件夹**

如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-3b0b60a5943f1c9e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1049/format/webp)

创建AIDL文件夹.png

**2、创建AIDL文件**

如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-639b4d559e1fa814.png?imageMogr2/auto-orient/strip|imageView2/2/w/942/format/webp)

创建AIDL文件.png

文件名为  **"IMyAidlInterface"**

**3、编辑和生成AIDL文件**

增加一行代码如下图

```cpp
   void connect();
```

![img](https:////upload-images.jianshu.io/upload_images/5713484-8cf233049248aeb8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1044/format/webp)

编辑.png

这时候可以点击"同步"按钮，或者rebuild以下项目。然后去看下是否生成了相应的文件

如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-90cd8fba73b019a1.png?imageMogr2/auto-orient/strip|imageView2/2/w/846/format/webp)

生成文件.png

**4、编写客户端代码**

设置一个单例模式的PushManager，代码如下：



```java
public class PushManager {

    private static final String TAG = "GEBILAOLITOU";
    private int id=1;

    //定义为单例模式
    private PushManager() {
    }

    private IMyAidlInterface iMyAidlInterface;

    private static PushManager instance = new PushManager();

    public static PushManager getInstance() {
        return instance;
    }


    public void init(Context context){
        //定义intent
        Intent intent = new Intent(context,PushService.class);
        //绑定服务
        context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
    }

    public void connect(){
        try {
            //通过AIDL远程调用
            Log.d(TAG,"pushManager ***************start Remote***************");
            iMyAidlInterface.connect();
        } catch (RemoteException e) {
            e.printStackTrace();
        }

    }


    private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //成功连接
            Log.d(TAG,"pushManager ***************成功连接***************");
            iMyAidlInterface = IMyAidlInterface.Stub.asInterface(service);

        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            //断开连接调用
            Log.d(TAG,"pushManager ***************连接已经断开***************");
        }
    };
}
```

还有一个Server



```java
public class PushService extends Service{

    private MyServer myServer=new MyServer();


    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return myServer;
    }
}
```

**5、编写服务器代码**

> 编写服务端实现connect()、sendInMessage()方法

我自己写一个MyServer.java类，代码如下:



```java
public class MyServer extends IMyAidlInterface.Stub {

    private static final String TAG = "GEBILAOLITOU";


    @Override
    public void connect() throws RemoteException {
        Log.i(TAG,"connect");
    }

    @Override
    public void connect() throws RemoteException {
        Log.i(TAG,"MyServer connect");
    }

    @Override
    public void sendInMessage(Message message) throws RemoteException {
        Log.i(TAG,"MyServer ** sendInMessage **"+message.toString());
    }
}
```

再写一个Service



```java
public class PushService extends Service{

    private MyServer myServer=new MyServer();


    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return myServer;
    }
}
```

**6、编写传递的"数据"**

> 之前讲解Binder的时候，说过，跨进程调试的时候需要实现Parcelable接口。因为AIDL默认支持的类型保罗Java基本类型（int、long等）和（String、List、Map、CharSequence），如果要传递自定义的类型要实现android.os.Parcelable接口。

假设在两个进程中传递的“数据”一般都是“消息”，我们编写具体的载体，写一个实体类public class Message implements Parcelable。代码如下：



```java
public class Message implements Parcelable {

    public long id;  //消息的id
    public String content; //消息的内容
    public long time;  //时间

    @Override
    public String toString() {
        return "Message{" +
                "id=" + id +
                ", content='" + content + '\'' +
                ", time=" + time +
                '}';
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
          dest.writeLong(id);
          dest.writeString(content);
          dest.writeLong(time);
    }

    public Message(Parcel source){
          id=source.readLong();
          content=source.readString();
          time=source.readLong();
    }

    public Message(){

    }

    public void readFromParcel(Parcel in){
        id = in.readLong();
        content = in.readString();
        time=in.readLong();
    }


    public static final Creator<Message> CREATOR=new Creator<Message>() {
        @Override
        public Message createFromParcel(Parcel source) {
            return new Message(source);
        }
        @Override
        public Message[] newArray(int size) {
            return new Message[size];
        }
    };
}
```

修改IMyAidlInterface.aidl，增加一个方法



```java
// IMyAidlInterface.aidl
package com.gebilaolitou.android.aidl;

// Declare any non-default types here with import statements
import com.gebilaolitou.android.aidl.Message;

interface IMyAidlInterface {

    //连接
    void connect();

    void sendMessage(Message message);
}
```

**7、进行调试**

这时候我们rebuild项目，或者同步，会报错



```ruby
Information:Gradle tasks [:app:generateDebugSources, :app:generateDebugAndroidTestSources, :app:mockableAndroidJar, :app:prepareDebugUnitTestDependencies]
/Users/gebilaolitou/Downloads/AIDLDemo/app/src/main/aidl/com/gebilaolitou/android/aidl/IMyAidlInterface.aidl
Error:(5) couldn't find import for class com.gebilaolitou.android.aidl.Message
```

这是因为自定义类型不仅要定义实现android.os.Parcelable接口类，还的为该实现类定义个aidl文件。

![img](https:////upload-images.jianshu.io/upload_images/5713484-f19b6f8697485083.png?imageMogr2/auto-orient/strip|imageView2/2/w/632/format/webp)

Message.AIDL.png

代码如下：



```go
// Message.aidl.aidl
package com.gebilaolitou.android.aidl;

// Declare any non-default types here with import statements
import com.gebilaolitou.android.aidl.Message;

parcelable Message ;
```

> 注意:自定义类型aidl文件名字、路径需要和自定义类名字，路径保持一致。

编译项目或者同步，还是报错



```ruby
Information:Gradle tasks [:app:generateDebugSources, :app:generateDebugAndroidTestSources, :app:mockableAndroidJar, :app:prepareDebugUnitTestDependencies]
Error:Execution failed for task ':app:compileDebugAidl'.
> java.lang.RuntimeException: com.android.ide.common.process.ProcessException: Error while executing process /Users/gebilaolitou/Library/Android/sdk/build-tools/25.0.2/aidl with arguments {-p/Users/gebilaolitou/Library/Android/sdk/platforms/android-25/framework.aidl -o/Users/gebilaolitou/Downloads/AIDLDemo/app/build/generated/source/aidl/debug -I/Users/gebilaolitou/Downloads/AIDLDemo/app/src/main/aidl -I/Users/gebilaolitou/Downloads/AIDLDemo/app/src/debug/aidl -I/Users/gebilaolitou/.android/build-cache/a1586d1e8ebbe7e662fad3571bc225fc0194aa31/output/aidl -I/Users/gebilaolitou/.android/build-cache/d52918cebdc5dd2a94851191b48afd011ca2b5c1/output/aidl -I/Users/gebilaolitou/.android/build-cache/a04d8c3e429534f503f1d5bbe860d0472d27613b/output/aidl -I/Users/gebilaolitou/.android/build-cache/83996cff7d35ba84168d1ed30e788df3aae82edd/output/aidl -I/Users/gebilaolitou/.android/build-cache/59c7b347cfeea50e258b0409d1f3d4eb91f58927/output/aidl -I/Users/gebilaolitou/.android/build-cache/7127c78b885aaac60b2b0712034851136a252f90/output/aidl -I/Users/gebilaolitou/.android/build-cache/d501962f472162456967742c491558ed0737c27c/output/aidl -I/Users/guochenli/.android/build-cache/13936966f3097ecab148b88871eeb79b0a9fe984/output/aidl -I/Users/gebilaolitou/.android/build-cache/fb883931c2e88ee11d0e77773aa01a2e67652940/output/aidl -I/Users/gebilaolitou/.android/build-cache/a0568698ab34df5d8fa827b197c443b9e1747f8e/output/aidl -d/var/folders/7d/z3snv_1n33xbf7l4vr2956fh0000gn/T/aidl2942334890792150517.d /Users/gebilaolitou/Downloads/AIDLDemo/app/src/main/aidl/com/gebilaolitou/android/aidl/IMyAidlInterface.aidl}
```

貌似什么看不出来，这时候我们看下Gradle Console，会发现



```ruby
Executing tasks: [:app:generateDebugSources, :app:generateDebugAndroidTestSources, :app:mockableAndroidJar, :app:prepareDebugUnitTestDependencies]

Configuration on demand is an incubating feature.
Incremental java compilation is an incubating feature.
:app:preBuild UP-TO-DATE
:app:preDebugBuild UP-TO-DATE
:app:checkDebugManifest
:app:preReleaseBuild UP-TO-DATE
:app:prepareComAndroidSupportAnimatedVectorDrawable2531Library
:app:prepareComAndroidSupportAppcompatV72531Library
:app:prepareComAndroidSupportConstraintConstraintLayout102Library
:app:prepareComAndroidSupportSupportCompat2531Library
:app:prepareComAndroidSupportSupportCoreUi2531Library
:app:prepareComAndroidSupportSupportCoreUtils2531Library
:app:prepareComAndroidSupportSupportFragment2531Library
:app:prepareComAndroidSupportSupportMediaCompat2531Library
:app:prepareComAndroidSupportSupportV42531Library
:app:prepareComAndroidSupportSupportVectorDrawable2531Library
:app:prepareDebugDependencies
:app:compileDebugAidl
aidl E 19367 21417307 type_namespace.cpp:130] In file /Users/guochenli/Downloads/AIDLDemo/app/src/main/aidl/com/gebilaolitou/android/aidl/IMyAidlInterface.aidl line 12 parameter message (argument 1):
aidl E 19367 21417307 type_namespace.cpp:130]     'Message' can be an out type, so you must declare it as in, out or inout.
```

看重点

> aidl E 19367 21417307 type_namespace.cpp:130] In file /Users/gebilaolitou/Downloads/AIDLDemo/app/src/main/aidl/com/gebilaolitou/android/aidl/IMyAidlInterface.aidl line 12 parameter message (argument 1):
>  **aidl E 19367 21417307 type_namespace.cpp:130]     'Message' can be an out type, so you must declare it as in, out or inout.**

这是因为AIDL不是Java。它是真的很接近，但是它不是Java。Java参数是没有方向的概念，AIDL参数是有方向，参数可以从客户端传到服务端，再返回来。

> 所以如果sendMessage()方法的message参数是纯粹的输入参，这意味着是从客户端到服务器的数据，你需要在ADIL声明：



```cpp
void sendMessage(in Message message);
```

> 如果sendMessage方法的message参数是纯粹的输出，这意味它的数据是通过服务器到客户端的，使用：



```csharp
void sendMessage(out Message message);
```

> 如果sendMessage方法的message参数是输入也是输出，客户端的值在服务器可能会被修改，使用



```cpp
void sendMessage(inout Message message);
```

> 所以，最终我们修改IMyAidlInterface.aidl文件如下



```java
// IMyAidlInterface.aidl
package com.gebilaolitou.android.aidl;

// Declare any non-default types here with import statements
import com.gebilaolitou.android.aidl.Message;

interface IMyAidlInterface {
    //连接
    void connect();
    //发送消息  客户端——> 服务器
    void sendInMessage(in Message message);
}
```

这时候重新编译就可以了

**8、补充完善**

设计一个Activity，如下



```java
public class MainActivity extends AppCompatActivity {

    private Button   btn,btnSend;
    private EditText et;
    private boolean isConnected=false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        btn=(Button)this.findViewById(R.id.btn);
        et=(EditText)this.findViewById(R.id.et);
        btnSend=(Button)this.findViewById(R.id.send);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                PushManager.getInstance().connect();
                isConnected=true;
            }
        });
        btnSend.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if(!isConnected){
                    Toast.makeText(MainActivity.this,"请连接",Toast.LENGTH_LONG).show();
                }
                if(et.getText().toString().trim().length()==0){
                    Toast.makeText(MainActivity.this,"请输入",Toast.LENGTH_LONG).show();
                }
                PushManager.getInstance().sendString(et.getText().toString());
            }
        });
    }
}
```

对应的xml如下



```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    >

    <Button
        android:id="@+id/btn"
        android:layout_width="368dp"
        android:layout_height="50dp"
        android:text="连接"
        />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="horizontal">
        <EditText
            android:layout_weight="1"
            android:id="@+id/et"
            android:layout_width="wrap_content"
            android:layout_height="50dp"
            />
        <Button
            android:id="@+id/send"
            android:text="发送"
            android:layout_width="50dp"
            android:layout_height="50dp" />
    </LinearLayout>

</LinearLayout>
```

别忘记manifest文件



```xml
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:name=".App"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <service
            android:name=".PushService"
            android:enabled="true"
            android:process=":push"
            android:exported="true" />
    </application>
```

**9、效果显示**

**(1) 没有任何操作下显示如下**

无操作情况下，**主进程**log输出如下

```undefined
08-14 15:57:57.491 9575-9575/com.gebilaolitou.android.aidl I/GEBILAOLITOU: PushManager ***************成功连接***************
```

**(2) 先点击"连接"按钮，主进程log输出如下**



```undefined
08-14 16:00:20.394 9575-9575/com.gebilaolitou.android.aidl I/GEBILAOLITOU: PushManager ***************start Remote***************
```

这时候看下**push进程**log输出如下



```undefined
08-14 16:00:20.394 9605-9627/com.gebilaolitou.android.aidl:push I/GEBILAOLITOU: MyServer connect
```

**(3) 在输入框 输入"123456"，然后点击"发送"按钮,主进程log输出如下**



```bash
08-14 16:02:23.042 9575-9575/com.gebilaolitou.android.aidl I/GEBILAOLITOU: PushManager ***************sendString***************Message{id=2, content='123456', time=1502697743041}
```

这时候看下**push进程**log输出如下



```bash
08-14 16:02:23.045 9605-9625/com.gebilaolitou.android.aidl:push I/GEBILAOLITOU: MyServer ** sendInMessage **Message{id=2, content='123456', time=1502697743041}
```

至此我们成功把一个Message对象通过AIDL传递到另外一个进程中。

## 五、源码跟踪

通过上面的内容，我们已经学会了AIDL的全部用法，接下来让我们透过现象看本质，研究一下究竟AIDL是如何帮助我们进行跨进程通信的。

> 我们上文已经提到，在写完AIDL文件后，编译器会帮我们自动生成一个同名的.java文件，大家已经发现了，在我们实际编写客户端和服务端代码的过程中，真正协助我们工作的其实就是这个文件，而.aidl文件从头到尾都没有出现过。大家会问：我们为什么要写这个.aidl文件。其实我们写这个.aidl文件就是为了生成这个对应的.java文件。事实上，就算我们不写AIDL文件，直接按照它生成的.java文件这样写一个.java文件出来。在服务端和客户端也可以照常使用这个.java类进行跨进程通信。所以说AIDL语言只是在简化我们写这个.java文件而已，而要研究AIDL是符合帮助我们进行跨境进程通信的，其实就是研究这个生成的.java文件是如何工作的

### (一) .java文件位置

位置如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-49c9805701f5e961.png?imageMogr2/auto-orient/strip|imageView2/2/w/910/format/webp)

位置.png

它的完整路径是：app->build->generated->source->aidl->debug->com->gebilaolitou->android->aidl->IMyAidlInterface.java（其中 com.gebilaolitou.android.aidl
 是包名，相对应的AIDL文件为 IMyAidlInterface.aidl ）。在AndroidStudio里面目录组织方式由默认的 android改为 Project 就可以直接按照文件夹结构访问到它。

### (二) IMyAidlInterface .java类分析

IMyAidlInterface .java 里面代码如下：



```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/gebilaolitou/Downloads/AIDLDemo/app/src/main/aidl/com/gebilaolitou/android/aidl/IMyAidlInterface.aidl
 */
package com.gebilaolitou.android.aidl;

public interface IMyAidlInterface extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.gebilaolitou.android.aidl.IMyAidlInterface {
        private static final java.lang.String DESCRIPTOR = "com.gebilaolitou.android.aidl.IMyAidlInterface";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.gebilaolitou.android.aidl.IMyAidlInterface interface,
         * generating a proxy if needed.
         */
        public static com.gebilaolitou.android.aidl.IMyAidlInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.gebilaolitou.android.aidl.IMyAidlInterface))) {
                return ((com.gebilaolitou.android.aidl.IMyAidlInterface) iin);
            }
            return new com.gebilaolitou.android.aidl.IMyAidlInterface.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_connect: {
                    data.enforceInterface(DESCRIPTOR);
                    this.connect();
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_sendInMessage: {
                    data.enforceInterface(DESCRIPTOR);
                    com.gebilaolitou.android.aidl.Message _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.gebilaolitou.android.aidl.Message.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.sendInMessage(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.gebilaolitou.android.aidl.IMyAidlInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }
//连接

            @Override
            public void connect() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_connect, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
//发送消息  客户端——> 服务器

            @Override
            public void sendInMessage(com.gebilaolitou.android.aidl.Message message) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((message != null)) {
                        _data.writeInt(1);
                        message.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_sendInMessage, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_connect = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_sendInMessage = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }
//连接

    public void connect() throws android.os.RemoteException;
//发送消息  客户端——> 服务器

    public void sendInMessage(com.gebilaolitou.android.aidl.Message message) throws android.os.RemoteException;
}
```

> 们可以看到编译的后**IMyAidlInterface.java**文件是一个接口，继承自**android.os.IInterface**，仔细观察IMyAidlInterface接口我们可以发现IMyAidlInterface内部代码主要分成两部分，一个是**抽象类Stub** 和 **原来aidl声明的connect()和sendInMessage()方法**

如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-142047d87be7e2e4.png?imageMogr2/auto-orient/strip|imageView2/2/w/764/format/webp)

IMyAidlInterface结构.png

> 重点在于**Stub类**，下面我们来分析一下。从**Stub类**中我们可以看到是继承自Binder，并且实现了IMyAidlInterface接口。如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-30a832bd1957002f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Stub类.png

Stub类基本结构如下：

> - 静态方法 asInterface(android.os.IBinder obj)
> - 静态内部类 Proxy
> - 方法 onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
> - 方法 asBinder()
> - private的String类型常量DESCRIPTOR
> - private的int类型常量TRANSACTION_connect
> - private的int类型常量TRANSACTION_sendInMessage

那我们就依次来分析下，我们先从 asInterface(android.os.IBinder obj) 方法入手

**1、静态方法 asInterface(android.os.IBinder obj)**

```csharp
        /**
         * Cast an IBinder object into an com.gebilaolitou.android.aidl.IMyAidlInterface interface,
         * generating a proxy if needed.
         */
        public static com.gebilaolitou.android.aidl.IMyAidlInterface asInterface(android.os.IBinder obj) {
            //非空判断
            if ((obj == null)) {
                return null;
            }
            // DESCRIPTOR是常量为"com.gebilaolitou.android.aidl.IMyAidlInterface"
            // queryLocalInterface是Binder的方法，搜索本地是否有可用的对象
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            //如果有，则强制类型转换并返回
            if (((iin != null) && (iin instanceof com.gebilaolitou.android.aidl.IMyAidlInterface))) {
                return ((com.gebilaolitou.android.aidl.IMyAidlInterface) iin);
            }
            //如果没有，则构造一个IMyAidlInterface.Stub.Proxy对象
            return new com.gebilaolitou.android.aidl.IMyAidlInterface.Stub.Proxy(obj);
        }
```

> 上面的代码可以看到，主要的作用就是根据传入的Binder对象转换成客户端需要的IMyAidlInterface接口。通过之前学过的Binder内容，我们知道：如果客户端和服务端处于同一进程，那么queryLocalInterface()方法返回就是服务端Stub对象本身；如果是跨进程，则返回一个封装过的Stub.Proxy，也是一个代理类，在这个代理中实现跨进程通信。那么让我们来看下Stub.Proxy类

**2、onTransact()方法解析**

onTransact()方法是根据code参数来处理，这里面会调用真正的业务实现类

> 在onTransact()方法中，根据传入的code值回去执行服务端相应的方法。其中常量TRANSACTION_connect和常量TRANSACTION_sendInMessage就是code值(在AIDL文件中声明了多少个方法就有多少个对应的code)。其中data就是服务端方法需要的的参数，执行完，最后把方法的返回结果放入reply中传递给客户端。如果该方法返回false，那么客户端请求失败。

**3、静态类Stub.Proxy**

代码如下：



```java
private static class Proxy implements com.gebilaolitou.android.aidl.IMyAidlInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }
            //Proxy的asBinder()返回位于本地接口的远程代理
            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }
//连接
            
            @Override
            public void connect() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_connect, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
//发送消息  客户端——> 服务器

            @Override
            public void sendInMessage(com.gebilaolitou.android.aidl.Message message) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((message != null)) {
                        _data.writeInt(1);
                        message.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_sendInMessage, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }
```

通过上面的代码，我们知道了几个重点

> - 1、Proxy 实现了 com.gebilaolitou.android.aidl.IMyAidlInterfac接口，所以他内部有IMyAidlInterface接口的两个抽象方法
> - 2、Proxy的asBinder()方法返回的mRemote，而这个mRemote是什么时候被赋值的？是在构造函数里面被赋值的。

**3.1、静态类Stub.Proxy的connect()方法和sendInMessage()方法**

**3.1.1connect()方法解析**

> - 1、 通过阅读静态类Stub.Proxy的connect()方法，我们容易分析出来里面的两个android.os.Parcel**_data**和**_reply**是用来进行跨进程传输的"载体"。而且通过字面的意思，很容易猜到，**_data**用来存储 **客户端流向服务端** 的数据，**_reply**用来存储 **服务端流向客户端** 的数据。
> - 2、通过mRemote. transact()方法，将**_data**和**_reply**传过去
> - 3、通过_reply.readException()来读取服务端执行方法的结果。
> - 4、最后通过finally回收l**_data**和**_reply** 

**3.1.2的相关参数**

关于Parcel，这部分我已经在前面，讲解过了，如果有不明白的，请看[Android跨进程通信IPC之4——AndroidIPC基础1](https://www.jianshu.com/p/f5e103674953)中的第四部分**Parcel类详解**。

关于 transact()方法：这是客户端和和服务端通信的核心方法，也是IMyAidlInterface.Stub继承android.os.Binder而重写的一个方法。调起这个方法之后，客户端将会挂起当前线程，等候服务端执行完相关任务后，通知并接收返回的**_reply**数据流。关于这个方法的传参，有注意两点

> - 1 方法ID：transact()方法第一个参数是一个方法ID，这个是客户端和服务端约定好的给方法的编码，彼此一一对应。在AIDL文件转话为.java时候，系统会自动给AIDL里面的每一个方法自动分配一个方法ID。而这个ID就是咱们说的常量**TRANSACTION_connect**和**TRANSACTION_sendInMessage**这些常量生成了递增的ID,是根据你在aidl文件的方法顺序而来，然后在IMyAidlInterface.Stub中的onTransact()方法里面switch根据第一个参数code即我们说的ID而来。
> - 2最后的一个参数：transact()方法最后一个参数是一个int值，代表是单向的还是双向的。具体大家请参考我们前面的文章[Android跨进程通信IPC之5——Binder的三大接口](https://www.jianshu.com/p/3c71473e7305)中关于IBinder部分。我这里直接说结论：0表示双向流通，即**_reply**可以正常的携带数据回来。如果为1的话，那么数据只能单向流程，从服务端回来的数据**_reply**不携带任何数据。注意：AIDL生成的.java文件，这个参数均为0

sendInMessage()方法和connect()方法大同小异，这里就不详细说明了。

**3.2  AIDL中关于定向tag的理解**

定向tag是AIDL语法的一部分，而in，out，intout，是三个定向的tag

**3.2.1  android官方文档中关于AIDL中定向tag的介绍**

[https://developer.android.com/guide/components/aidl.html](https://link.jianshu.com?t=https://developer.android.com/guide/components/aidl.html)中**1. Create the .aidl file** 里面，原文内容截图了一部分如下:

> All non-primitive parameters require a directional tag indicating which way the data goes . Either in , out , or inout . Primitives are in by default , and connot be otherwise .

翻译过来就是：**所有非基本参数都需要一个定向tag来指出数据流通的方向，不管是in，out，inout。基本参数的定向tag默认是并且只能是in**
 所以按照上面翻译的理解，in和out代表客户端与服务端两条单向的数据流向，而inout则表示两端可双向流通数据的。

**3.2.2 总结**

> AIDL中的定向tag表示在跨进程通信中数据 流向，其中in表示数据只能由客户端流向服务端，out表示数据只能由服务端流行客户端，而inout则表示数据可在服务端与客户端之间双向流通。其中，数据流向是针对客户端中的那个传入方法的对象而言。in为定向tag的话，表现为服务端将会接受到一个那个对象的完整数据，但是客户端的那个对象不会因为服务端传参修改而发生变动；out的话表现为服务端将会接收到那个对象的参数为空的对象，但是在服务端对接收到的空对象有任何修改之后客户端将会同步变动；inout为定向tag的情况下，服务端将会接收到客户端传来对象的完整信息，并且客户端将会同步服务对该对象的任何变动。

**4.asBinder()方法**

该方法就是返回当前的Binder方法

### (三) IMyAidlInterface .java流程分析

通过阅读上面源码，我们来看下AIDL的流程，以上面的例子为例，那么先从客户端开始

**1.客户端流程**

**1、获取IMyAidlInterface对象**

```java
    public void init(Context context){
        //定义intent
        Intent intent = new Intent(context,PushService.class);
        //绑定服务
        context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
    }

    private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //成功连接
            Log.i(TAG,"PushManager ***************成功连接***************");
            iMyAidlInterface = IMyAidlInterface.Stub.asInterface(service);

        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            //断开连接调用
            Log.i(TAG,"PushManager ***************连接已经断开***************");
        }
    };
```

客户端中通过Intent去绑定一个服务端的Service。在** onServiceConnected(ComponentName name, IBinder service)**方法中通过返回service可以得到AIDL接口的实例。这是调用了asInterface(android.os.IBinder) 方法完成的。

**1.1、asInterface(android.os.IBinder)方法**



```csharp
        /**
         * Cast an IBinder object into an com.gebilaolitou.android.aidl.IMyAidlInterface interface,
         * generating a proxy if needed.
         */
        public static com.gebilaolitou.android.aidl.IMyAidlInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.gebilaolitou.android.aidl.IMyAidlInterface))) {
                return ((com.gebilaolitou.android.aidl.IMyAidlInterface) iin);
            }
            return new com.gebilaolitou.android.aidl.IMyAidlInterface.Stub.Proxy(obj);
        }
```

在asInterface(android.os.IBinder)我们知道是调用的new com.gebilaolitou.android.aidl.IMyAidlInterface.Stub.Proxy(obj)构造的一个Proxy对象。

所以可以这么说在PushManager中的变量IMyAidlInterface其实是一个IMyAidlInterface.Stub.Proxy对象。

**2、调用connect()方法**

上文我们说过了PushManager类中的iMyAidlInterface其实IMyAidlInterface.Stub.Proxy对象，所以调用connect()方法其实是IMyAidlInterface.Stub.Proxy的connect()方法。代码如下：



```java
            @Override
            public void connect() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_connect, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
```

> - 1、这里面主要是生成了_data和_reply数据流，并向_data中存入客户端的数据。
> - 2、通过 transact()方法将他们传递给服务端，并请求服务指定的方法
> - 3、接收_reply数据，并且从中读取服务端传回的数据。

通过上面客户端的所有行为，我们会发现，其实通过ServiceConnection类中onServiceConnected(ComponentName name, IBinder service)中第二个参数service很重要，因为我们最后是滴啊用它的transact() 方法，将客户端的数据和请求发送给服务端去。从这个角度来看，这个service就像是服务端在客户端的代理一样，而IMyAidlInterface.Stub.Proxy对象更像一个二级代理，我们在外部通过调用这个二级代理来间接调用service这个一级代理

**2服务端流程**

在前面几篇文章中我们知道Binder传输中，客户端调用transact()对应的是服务端的onTransact()函数，我们在IMyAidlInterface.java中看到

**1、获取IMyAidlInterface.Stub的transact()方法**



```kotlin
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_connect: {
                    data.enforceInterface(DESCRIPTOR);
                    this.connect();
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_sendInMessage: {
                    data.enforceInterface(DESCRIPTOR);
                    com.gebilaolitou.android.aidl.Message _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.gebilaolitou.android.aidl.Message.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.sendInMessage(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }
```

可以看到，他在收到客户端的 transact()方法后，直接调用了switch选择，根据ID执行不同操作，因为我们知道是调用的connect()方法，所以对应的code是TRANSACTION_connect，所以我们下**case TRANSACTION_connect:**的内容，如下：



```kotlin
                case TRANSACTION_connect: {
                    data.enforceInterface(DESCRIPTOR);
                    this.connect();
                    reply.writeNoException();
                    return true;
                }
```

这里面十分简单了，就是直接调用服务端的connect()方法。

整体流程如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-298c59e740ccf7e3.png?imageMogr2/auto-orient/strip|imageView2/2/w/965/format/webp)

AIDL流程.png

## 六、AIDL的设计给我们的思考

通过研究我们知道AIDL是基于Binder实现的跨进程通信，但是为什么要设计成AIDL，这么麻烦？

> 在程序的设计领域，任何的解决方案，无非都是需求和性能两方面的综合考量。性能又包括可维护性和可拓展性等。

我们知道android已经有了Binder实现跨进程通信，但是里面涉及很多Service在启动的时候在ServiceManager中进行注册，但是在application层面做到这一步基本是无法实现的。所以很有以必要研究一套application层的IPC通信机制。因为android已经提供了Binder机制，如果能重复利用Binder机制岂不是更好，所以就有了现在的AIDL。android是操作系统，要利于开发者去方便操作，所以就应该是设计出一套模板。这样就很方便了。

> 由于是跨进程通信，所以我们就需要有一种途径去访问它们，在这时候，**代理—桩**的设计理念就初步成型了。为了达到我们的目的，我们可以在客户端建立一个服务端的代理，在服务端建立一个客户端的桩，这样一来，客户端有什么需求可以直接和代理通信，代理说你等下，代理就把信息发送给桩，桩把信息给服务端进行处理，处理结束后，服务器告诉桩，桩告诉代理，代理告诉客户端。这样一来，客户端以为代理就是服务端，并且事实是它也只是和代理进行交互，服务端也是也是如此。

类似的跨进程通信机制，我知道还有一个是[Hermes](https://link.jianshu.com?t=https://github.com/Xiaofei-it/Hermes)，大家有空可以去了解下。

## 七、总结

> AIDL是Android IPC机制中很重要的一部分，AIDL主要是通过Binder来实现进程通信的，其实另一种IPC方式的Message底层也是通过AIDL来实现的。所以AIDL的重要性就不言而喻了。后面我将讲解Message.

为了让大家更好的理解AIDL，我下面补上了三张图，分别是类图、流程图和原理图。

设计的类图如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-7789dc0525d8b8fb.png?imageMogr2/auto-orient/strip|imageView2/2/w/754/format/webp)

类图.png

![img](https:////upload-images.jianshu.io/upload_images/5713484-2c51ce26ad1e82a9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1113/format/webp)

AIDL流程图.png

![img](https:////upload-images.jianshu.io/upload_images/5713484-f640dfbc3d397ba5.png?imageMogr2/auto-orient/strip|imageView2/2/w/960/format/webp)





# android跨进程通信IPC之Binder的补充

## 一 、Binder中的线程池

> 客户端在使用Binder可以调用服务端的方法，这里面有一些隐含的问题，如果我们服务端的方法是一个耗时的操作，那么对于我们客户端和服务端都存在风险，如果有很多客户端都来调用它的方法，那么是否会造成ANR那？多个客户端调用，是否会有同步问题？如果客户端在UI线程中调用的这个是耗时方法，那么是不是它也会造成ANR？这些问题都是真实存在的，首先第一个问题是不会出现，因为服务端所有这些被调用方法都是在一个线程池中执行的，不在服务端的UI线程中，因此服务端不会ANR，但是服务端会有同步问题，因此我们提供的服务端接口方法应该注意同步问题。客户端会ANR很容易解决，就是我们不要在UI线程中就可以避免了。那我们一起来看下Binder的线程池

### (一) Binder线程池简述

> Android系统启动完成后，ActivityManager、PackageManager等各大服务都运行在system_server进程，app应用需要使用系统服务都是通过Binder来完成进程间的通信，那么对于Binder线程是如何管理的？又是如何创建的？其实无论是system_server进程还是app进程，都是在fork完成后，便会在新进程中执行onZygoteInit()的过程，启动Binder线程池。

从整体架构以及通信协议的角度来阐述了Binder架构。那对于binder线程是如何管理的呢，又是如何创建的呢？其实无论是system_server进程，还是app进程，都是在进程fork完成后，便会在新进程中执行onZygoteInit()的过程中，启动binder线程池。

### (二) Binder线程池创建

Binder 线程创建与其坐在进程的创建中产生，Java层进程的创建都是通过**Process.start()**方法，向Zygote进程发出创建进程的socket消息，Zygote收到消息后会调用Zygote.forkAndSpecialize()来fork出新进程，在新进程中调用**RuntimeInit.nativeZygoteInit()**方法，该方法经过JNI映射，最终会调用app_main.cpp中的onZygoteInit，那么接下来从这个方法开始。

**1、onZygoteInit()**

代码在[app_main.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/cmds/app_process/app_main.cpp) 的91行



```rust
    virtual void onZygoteInit()
    {
        // 获取 ProcessState对象
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }
```

ProcessState主要工作是调用open()打开/dev/binder驱动设备，再利用mmap()映射内核的地址空间，将Binder驱动的fd赋值ProcessState对象的变量mDriverFD，用于交互操作。startThreadPool()是创建一个型的Binder线程，不断进行talkWithDriver()。

**2、ProcessState.startThreadPool()**

代码在[ProcessState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/ProcessState.cpp) 的132行



```cpp
void ProcessState::startThreadPool()
{
     //多线程同步
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}
```

启动Binder线程池后，则设置mThreadPoolStarted=true，通过变量mThreadPoolStarted来保证每个应用进程只允许启动一个Binder线程池，且本次创建的是Binder主线程(isMain=true，isMain具体请看spawnPooledThread(true))。其余Binder线程池中的线程都是由Binder驱动来控制创建的。然后继续跟踪看下spawnPooledThread(true)函数

**3、ProcessState. spawnPooledThread()**

代码在[ProcessState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/ProcessState.cpp) 的286行



```cpp
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        //获取Binder线程名
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
         //这里注意isMain=true
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
```

**3.1、ProcessState. makeBinderThreadName()**

代码在[ProcessState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/ProcessState.cpp) 的279行



```cpp
String8 ProcessState::makeBinderThreadName() {
    int32_t s = android_atomic_add(1, &mThreadPoolSeq);
    String8 name;
    name.appendFormat("Binder_%X", s);
    return name;
}
```

获取Binder的线程名，格式为**Binder_X**，其中X为整数，每个进程中的Binder编码是从1开始，依次递增；只有通过**makeBinderThreadName()**方法来创建线程才符合这个格式，对于直接将当前线程通过**joinThreadPool()**加入线程池的线程名则不符合这个命名规则。另外，目前Android N中Binder命令已改为**Binder: _X**，则对于分析问题很有帮助，通过Binder名称的pid就可以很快定位到该Binder所属进程的pid

**3.2、PoolThread.run**

代码在[ProcessState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/ProcessState.cpp) 的52行



```cpp
class PoolThread : public Thread
{
public:
    PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }

protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain); 
        return false;
    }
    const bool mIsMain;
};
```

从函数名看起来是创建线程池，其实就只是创建一个线程，该PoolThread继承Thread类，t->run()函数最终会调用PoolThread的threadLooper()方法。

**4、IPCThreadState. joinThreadPool()**

代码在[IPCThreadState.cpp.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp) 的52行



```cpp
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());
     // 创建Binder线程
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    // This thread may have been spawned by a thread that was in the background
    // scheduling group, so first we will make sure it is in the foreground
    // one to avoid performing an initial transaction in the background.
    //设置前台调度策略
    set_sched_policy(mMyThreadId, SP_FOREGROUND);

    status_t result;
    do {
         //清楚队列的引用
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        // 处理下一条指令
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
            abort();
        }

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            //非主线程出现timeout则线程退出
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%p\n",
        (void*)pthread_self(), getpid(), (void*)result);
    // 线程退出循环
    mOut.writeInt32(BC_EXIT_LOOPER);
    //false代表bwr数据的read_buffer为空
    talkWithDriver(false);
}
```

> - 1 对于**isMain**=true的情况下，command为BC_ENTER_LOOPER，代表的是Binder主线程，不会退出线程。
> - 2 对于**isMain**=false的情况下，command为BC_REGISTER_LOOPER，表示的是binder驱动创建的线程。

joinThreadLoop()里面有一个do——while循环，这个thread里面主要的调用，也就是重点，里面主要就是调用了两个函数processPendingDerefs()和getAndExecuteCommand()函数，那我们依次来看下。

**4.1、IPCThreadState. processPendingDerefs()**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp) 的454行



```cpp
// When we've cleared the incoming command queue, process any pending derefs
void IPCThreadState::processPendingDerefs()
{
    if (mIn.dataPosition() >= mIn.dataSize()) {
        size_t numPending = mPendingWeakDerefs.size();
        if (numPending > 0) {
            for (size_t i = 0; i < numPending; i++) {
                RefBase::weakref_type* refs = mPendingWeakDerefs[i];
                //弱引用减一
                refs->decWeak(mProcess.get());
            }
            mPendingWeakDerefs.clear();
        }

        numPending = mPendingStrongDerefs.size();
        if (numPending > 0) {
            for (size_t i = 0; i < numPending; i++) {
                BBinder* obj = mPendingStrongDerefs[i];
                //强引用减一
                obj->decStrong(mProcess.get());
            }
            mPendingStrongDerefs.clear();
        }
    }
}
```

我们知道了processPendingDerefs()这个函数主要是将mPendingWeakDerefs和mPendingStrongDerefs中的指针解除应用，而且他的执行结果并不影响Loop的执行，那我们主要看下getAndExecuteCommand()函数里面做了什么。

**4.2、IPCThreadState. getAndExecuteCommand()**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp) 的414行



```php
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;
     //与binder进行交互
    result = talkWithDriver();
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32();
        IF_LOG_COMMANDS() {
            alog << "Processing top-level Command: "
                 << getReturnString(cmd) << endl;
        }

        pthread_mutex_lock(&mProcess->mThreadCountLock);
        mProcess->mExecutingThreadsCount++;
        pthread_mutex_unlock(&mProcess->mThreadCountLock);
        // 执行Binder响应吗
        result = executeCommand(cmd);

        pthread_mutex_lock(&mProcess->mThreadCountLock);
        mProcess->mExecutingThreadsCount--;
        pthread_cond_broadcast(&mProcess->mThreadCountDecrement);
        pthread_mutex_unlock(&mProcess->mThreadCountLock);

        // After executing the command, ensure that the thread is returned to the
        // foreground cgroup before rejoining the pool.  The driver takes care of
        // restoring the priority, but doesn't do anything with cgroups so we
        // need to take care of that here in userspace.  Note that we do make
        // sure to go in the foreground after executing a transaction, but
        // there are other callbacks into user code that could have changed
        // our group so we want to make absolutely sure it is put back.
        set_sched_policy(mMyThreadId, SP_FOREGROUND);
    }

    return result;
}
```

我们知道了getAndExecuteCommand()主要就是调用两个函数talkWithDriver()和executeCommand()，我们分别看一下

**4.2.1、ProcessState. talkWithDriver()**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp) 的803行



```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD <= 0) {
        return -EBADF;
    }

    binder_write_read bwr;

    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    IF_LOG_COMMANDS() {
        TextOutput::Bundle _b(alog);
        if (outAvail != 0) {
            alog << "Sending commands to driver: " << indent;
            const void* cmds = (const void*)bwr.write_buffer;
            const void* end = ((const uint8_t*)cmds)+bwr.write_size;
            alog << HexDump(cmds, bwr.write_size) << endl;
            while (cmds < end) cmds = printCommand(alog, cmds);
            alog << dedent;
        }
        alog << "Size of receive buffer: " << bwr.read_size
            << ", needRead: " << needRead << ", doReceive: " << doReceive << endl;
    }

    // Return immediately if there is nothing to do.
    //如果没有输入输出数据，直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        IF_LOG_COMMANDS() {
            alog << "About to read/write, write size = " << mOut.dataSize() << endl;
        }
#if defined(HAVE_ANDROID_OS)
        // ioctl执行binder读写操作，经过syscall，进入Binder驱动，调用Binder_ioctl
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD <= 0) {
            err = -EBADF;
        }
        IF_LOG_COMMANDS() {
            alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
        }
    } while (err == -EINTR);

    IF_LOG_COMMANDS() {
        alog << "Our err: " << (void*)(intptr_t)err << ", write consumed: "
            << bwr.write_consumed << " (of " << mOut.dataSize()
                        << "), read consumed: " << bwr.read_consumed << endl;
    }

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        IF_LOG_COMMANDS() {
            TextOutput::Bundle _b(alog);
            alog << "Remaining data size: " << mOut.dataSize() << endl;
            alog << "Received commands from driver: " << indent;
            const void* cmds = mIn.data();
            const void* end = mIn.data() + mIn.dataSize();
            alog << HexDump(cmds, mIn.dataSize()) << endl;
            while (cmds < end) cmds = printReturnCommand(alog, cmds);
            alog << dedent;
        }
        return NO_ERROR;
    }

    return err;
}
```

在这里调用的是isMain=true，也就是向mOut写入的是便是BC_ENTER_LOOPER。后面就是进入Binder驱动了，具体到binder_thread_write()函数的BC_ENTER_LOOPER的处理过程。

**4.2.1.1、binder_thread_write**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 的2252行



```cpp
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr < end && thread->return_error == BR_OK) {
        //拷贝用户空间的cmd命令，此时为BC_ENTER_LOOPER
        if (get_user(cmd, (uint32_t __user *)ptr)) -EFAULT;
        ptr += sizeof(uint32_t);
        switch (cmd) {
          case BC_REGISTER_LOOPER:
              if (thread->looper & BINDER_LOOPER_STATE_ENTERED) {
                //出错原因：线程调用完BC_ENTER_LOOPER，不能执行该分支
                thread->looper |= BINDER_LOOPER_STATE_INVALID;

              } else if (proc->requested_threads == 0) {
                //出错原因：没有请求就创建线程
                thread->looper |= BINDER_LOOPER_STATE_INVALID;

              } else {
                proc->requested_threads--;
                proc->requested_threads_started++;
              }
              thread->looper |= BINDER_LOOPER_STATE_REGISTERED;
              break;

          case BC_ENTER_LOOPER:
              if (thread->looper & BINDER_LOOPER_STATE_REGISTERED) {
                //出错原因：线程调用完BC_REGISTER_LOOPER，不能立刻执行该分支
                thread->looper |= BINDER_LOOPER_STATE_INVALID;
              }
              //创建Binder主线程
              thread->looper |= BINDER_LOOPER_STATE_ENTERED;
              break;

          case BC_EXIT_LOOPER:
              thread->looper |= BINDER_LOOPER_STATE_EXITED;
              break;
        }
        ...
    }
    *consumed = ptr - buffer;
  }
  return 0;
}
```

> 处理完BC_ENTER_LOOPER命令后，一般情况下成功设置thread->looper |= BINDER_LOOPER_STATE_ENTERED。那么binder线程的创建是在什么时候？那就当该线程有事务需要处理的时候，进入binder_thread_read()过程。

**4.2.1.2、binder_thread_read**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 的2654行



```php
binder_thread_read（）{
  ...
retry:
    //当前线程todo队列为空且transaction栈为空，则代表该线程是空闲的
    wait_for_proc_work = thread->transaction_stack == NULL &&
        list_empty(&thread->todo);

    if (thread->return_error != BR_OK && ptr < end) {
        ...
        put_user(thread->return_error, (uint32_t __user *)ptr);
        ptr += sizeof(uint32_t);
        //发生error，则直接进入done
        goto done; 
    }

    thread->looper |= BINDER_LOOPER_STATE_WAITING;
    if (wait_for_proc_work)
         //可用线程个数+1
        proc->ready_threads++; 
    binder_unlock(__func__);

    if (wait_for_proc_work) {
        if (non_block) {
            ...
        } else
            //当进程todo队列没有数据,则进入休眠等待状态
            ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
    } else {
        if (non_block) {
            ...
        } else
            //当线程todo队列没有数据，则进入休眠等待状态
            ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    }

    binder_lock(__func__);
    if (wait_for_proc_work)
        //可用线程个数-1
        proc->ready_threads--; 
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

    if (ret)
        //对于非阻塞的调用，直接返回
        return ret; 

    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        //先考虑从线程todo队列获取事务数据
        if (!list_empty(&thread->todo)) {
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        //线程todo队列没有数据, 则从进程todo对获取事务数据
        } else if (!list_empty(&proc->todo) && wait_for_proc_work) {
            w = list_first_entry(&proc->todo, struct binder_work, entry);
        } else {
            ... //没有数据,则返回retry
        }

        switch (w->type) {
            case BINDER_WORK_TRANSACTION: ...  break;
            case BINDER_WORK_TRANSACTION_COMPLETE:...  break;
            case BINDER_WORK_NODE: ...    break;
            case BINDER_WORK_DEAD_BINDER:
            case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
            case BINDER_WORK_CLEAR_DEATH_NOTIFICATION:
                struct binder_ref_death *death;
                uint32_t cmd;

                death = container_of(w, struct binder_ref_death, work);
                if (w->type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION)
                  cmd = BR_CLEAR_DEATH_NOTIFICATION_DONE;
                else
                  cmd = BR_DEAD_BINDER;
                put_user(cmd, (uint32_t __user *)ptr;
                ptr += sizeof(uint32_t);
                put_user(death->cookie, (void * __user *)ptr);
                ptr += sizeof(void *);
                ...
                if (cmd == BR_DEAD_BINDER)
                  goto done; //Binder驱动向client端发送死亡通知，则进入done
                break;
        }

        if (!t)
            continue; //只有BINDER_WORK_TRANSACTION命令才能继续往下执行
        ...
        break;
    }

done:
    *consumed = ptr - buffer;
    //创建线程的条件
    if (proc->requested_threads + proc->ready_threads == 0 &&
        proc->requested_threads_started < proc->max_threads &&
        (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED))) {
        proc->requested_threads++;
        // 生成BR_SPAWN_LOOPER命令，用于创建新的线程
        put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer)；
    }
    return 0;
}
```

放生一下三种情况中的任意一种，就会进入**done**

> - 当前线程return_error发生error的情况
> - 当Binder驱动向client端发送死亡通知的情况
> - 当类型为BINDER_WORK_TRANSACTION(即受到命令是BC_TRANSACTION或BC_REPLY)的情况

任何一个Binder线程当同事满足以下条件时，则会生成用于创建新线程的BR_SPAWN_LOOPER命令：

> - 1、当前进程中没有请求创建binder线程，即request_threads=0
> - 2、当前进程没有空闲可用binder线程，即ready_threads=0（线程进入休眠状态的个数就是空闲线程数）
> - 3、当前线程应启动线程个数小于最大上限(默认是15)
> - 4、当前线程已经接收到BC_ENTER_LOOPER或者BC_REGISTER_LOOPEE命令，即当前处于BINDER_LOOPER_STATE_REGISTERED或者BINDER_LOOPER_STATE_ENTERED状态。

**4.2.2、IPCThreadState. executeCommand()**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp) 的947行



```cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    status_t result = NO_ERROR;
    switch ((uint32_t)cmd) {
      ...
      case BR_SPAWN_LOOPER:
          //创建新的binder线程 
          mProcess->spawnPooledThread(false);
          break;
      ...
    }
    return result;
}
```

Binder主线程的创建时在其所在进程创建的过程中一起创建的，后面再创建的普通binder线程是由
 spawnPooledThread(false)方法所创建的。

### (三) Binder线程池流程

Binder设计架构中，只有第一个Binder主线程也就是Binder_1线程是由应用程序主动创建的，Binder线程池的普通线程都是Binder驱动根据IPC通信需求而创建，Binder线程的创建流程图如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-9996ac0e027d7cf4.png?imageMogr2/auto-orient/strip|imageView2/2/w/972/format/webp)

Binder线程的创建流程.png

每次由Zygote fork出新进程的过程中，伴随着创建binder线程池，调用spawnPooledThread来创建binder主线程。当线程执行binder_thread_read的过程中，发现当前没有空闲线程，没有请求创建线程，且没有达到上限，则创建新的binder线程。

Binder的transaction有3种类型：

> -call：发起进程的线程不一定是Binder线程，大多数情况下，接受者只指向进程，并不确定会有两个线程来处理，所以不指定线程。
>
> - reply：发起者一定是binder线程，并且接收者线程便是上此call时的发起线程(该线程不一定是binder线程，可以是任意线程)
> - async：与call类型差不多，唯一不同的是async是oneway方式，不需要回复，发起进程的线程不一定是在Binder线程，接收者只指向进程，并不确定会有那个线程来处理，所以不指定线程。

Binder系统中可分为3类binder线程：

> - Binder主线程：进程创建过程会调用startThreadPool过程再进入spawnPooledThread(true)，来创建Binder主线程。编号从1开始，也就是意味着binder主线程名为binder_1，并且主线程是不会退出的。
> - Binder普通线程：是由Binder Driver是根据是否有空闲的binder线程来决定是否创建binder线程，回调spawnPooledThread(false) ，isMain=false，该线程名格式为binder_x
> - Binder其他线程：其他线程是指并没有调用spawnPooledThread方法，而是直接调用IPCThreadState.joinThreadPool()，将当前线程直接加入binder线程队列。例如：mediaserver和servicemanager的主线程都是binder主线程，但是system_server的主线程并非binder主线程。

## 二、Binder的权限

### (一) 概述

前面关于Binder的文章，讲解了Binder的IPC机制。看过Android系统源代码的读者一定看到过**Binder.clearCallingIdentity()**，**Binder.restoreCallingIdentity()**，定义在**Binder.java**文件中



```java
 // Binder.java
    //清空远程调用端的uid和pid，用当前本地进程的uid和pid替代
    public static final native long clearCallingIdentity();
     // 作用是回复远程调用端的uid和pid信息，正好是"clearCallingIdentity"的饭过程
    public static final native void restoreCallingIdentity(long token);
```

这两个方法都涉及了uid和pid，每个线程都有自己独一无二***IPCThreadState\*对象，记录当前线程的pid和uid，可通过方法**Binder.getCallingPid()**和**Binder.getCallingUid()**获取相应的pic和uid。

> clearCallingIdentity()，restoreCallingIdentity()这两个方法使用过程都是成对使用的，这两个方法配合使用，用于权限控制检测功能。

### (二)  原理

从定义这两个方法是native方法，通过Binder的JNI调用，在android_util_Binder.cpp文件中定义了native两个方法所对应的jni方法。

**1、clearCallingIdentity**

代码在[android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp)  771行



```rust
static jlong android_os_Binder_clearCallingIdentity(JNIEnv* env, jobject clazz)
{
    return IPCThreadState::self()->clearCallingIdentity();
}
```

这里面代码混简单，就是调用了IPCThreadState的clearCallingIdentity()方法

**1.1、IPCThreadState::clearCallingIdentity()**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp)  356行



```cpp
int64_t IPCThreadState::clearCallingIdentity()
{
    int64_t token = ((int64_t)mCallingUid<<32) | mCallingPid;
    clearCaller();
    return token;
}
```

UID和PID是IPCThreadState的成员变量，都是32位的int型数据，通过移动操作，将UID和PID的信息保存到token，其中高32位保存UID，低32位保存PID。然后调用clearCaller()方法将当前本地进程pid和uid分别赋值给PID和UID，这个具体的操作在**IPCThreadState::clearCaller()**里面，最后返回token

**1.1.1、IPCThreadState::clearCaller()**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp)  356行



```cpp
void IPCThreadState::clearCaller()
{
    mCallingPid = getpid();
    mCallingUid = getuid();
}
```

**2、JNI:restoreCallingIdentity**

代码在[android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp)  776行



```cpp
static void android_os_Binder_restoreCallingIdentity(JNIEnv* env, jobject clazz, jlong token)
{
    // XXX temporary sanity check to debug crashes.
    //token记录着uid信息，将其右移32位得到的是uid
    int uid = (int)(token>>32);
    if (uid > 0 && uid < 999) {
        // In Android currently there are no uids in this range.
        //目前android系统不存在小于999的uid，所以uid<999则抛出异常。
        char buf[128];
        sprintf(buf, "Restoring bad calling ident: 0x%" PRIx64, token);
        jniThrowException(env, "java/lang/IllegalStateException", buf);
        return;
    }
    IPCThreadState::self()->restoreCallingIdentity(token);
}
```

这个方法主要是获取uid，然后调用IPCThreadState的restoreCallingIdentity(token)方法

**2.1、restoreCallingIdentity**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp)  383行



```cpp
void IPCThreadState::restoreCallingIdentity(int64_t token)
{
    mCallingUid = (int)(token>>32);
    mCallingPid = (int)token;
}
```

从token中解析出PID和UID，并赋值给相应的变量。该方法正好是clearCallingIdentity反过程。

**3、JNI:getCallingPid**

代码在[android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp)  761行



```rust
static jint android_os_Binder_getCallingPid(JNIEnv* env, jobject clazz)
{
    return IPCThreadState::self()->getCallingPid();
}
```

调用的是IPCThreadState的getCallingPid()方法

**3.1、IPCThreadState::getCallingPid**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp)  346行



```cpp
pid_t IPCThreadState::getCallingPid() const
{
    return mCallingPid;
}
```

直接返回mCallingPid

**4、JNI:getCallingUid**

代码在[android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp)  766行



```rust
static jint android_os_Binder_getCallingUid(JNIEnv* env, jobject clazz)
{
    return IPCThreadState::self()->getCallingUid();
}
```

调用的是IPCThreadState的getCallingUid()方法

**4.1、IPCThreadState::getCallingUid**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp)  346行



```cpp
uid_t IPCThreadState::getCallingUid() const
{
    return mCallingUid;
}
```

直接返回mCallingUid

**5、远程调用**

**5.1、binder_thread_read**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 的2654行



```php
binder_thread_read（）{
    ...
    while (1) {
      struct binder_work *w;
      switch (w->type) {
        case BINDER_WORK_TRANSACTION:
            t = container_of(w, struct binder_transaction, work);
            break;
        case :...
      }
      if (!t)
        continue; //只有BR_TRANSACTION,BR_REPLY才会往下执行
        
      tr.code = t->code;
      tr.flags = t->flags;
      tr.sender_euid = t->sender_euid; //mCallingUid

      if (t->from) {
          struct task_struct *sender = t->from->proc->tsk;
          //当非oneway的情况下,将调用者进程的pid保存到sender_pid
          tr.sender_pid = task_tgid_nr_ns(sender,current->nsproxy->pid_ns);
      } else {
          //当oneway的的情况下,则该值为0
          tr.sender_pid = 0;
      }
      ...
}
```

**5.2、IPCThreadState. executeCommand()**

代码在[IPCThreadState.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp) 的947行



```cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
        case BR_TRANSACTION:
        {
            const pid_t origPid = mCallingPid;
            const uid_t origUid = mCallingUid;
            // 设置调用者pid
            mCallingPid = tr.sender_pid; 
            // 设置调用者uid
            mCallingUid = tr.sender_euid;
            ...
            reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer,
                        &reply, tr.flags);
            // 恢复原来的pid
            mCallingPid = origPid; 
             // 恢复原来的uid
            mCallingUid = origUid; 
        }
        
        case :...
    }
}
```

关于mCallingPid、mCallingUid修改过程：是在每次Binder Call的远程进程在执行binder_thread_read()过程中，会设置pid和uid，然后在IPCThreadState的transact收到BR_TRANSACTION则会修改mCallingPid，mCallingUid。

> PS:当oneway的情况下：mCallingPid=0，不过mCallingUid可以拿到正确值

### (三)  思考

**1、场景分析：**

（1）场景：比如线程X通过Binder远程调用线程Y，然后线程Y通过Binder调用当前线程的另一个Service或者activity之类的组件。

（2）分析：

> - 1 线程X通过Binder远程调用线程Y：则线程Y的IPCThreadState中的mCallingUid和mCallingPid保存的就是线程X的UID和PID。这时在线程Y中调用Binder.getCallingPid()和Binder.getCallingUid()方法便可获取线程X的UID和PID，然后利用UID和PID进行权限对比，判断线程X是否有权限调用线程Y的某个方法
> - 2 线程Y通过Binder调用当前线程的某个组件：此时线程Y是线程Y某个组件的调用端，则mCallingUid和mCallingPid应该保存当前线程Y的PID和UID，故需要调用clearCallingIdentity()方法完成这个功能。当前线程Y调用完某个组件，由于线程Y仍然处于线程A的被用调用端，因此mCallingUidh和mCallingPid需要回复线程A的UID和PID，这时调用restoreCallingIdentity()即完成。

![img](https:////upload-images.jianshu.io/upload_images/5713484-050e3d696039d7b2.png?imageMogr2/auto-orient/strip|imageView2/2/w/607/format/webp)

场景分析.png

一句话：图中过程2(调用组件2开始之前)执行clearCallingIdentity()，过程3(调用组件2结束之后)执行restoreCallingIdentity()。

**2、实例分析：**

上述过程主要在system_server进程各个线程中比较常见(普通app应用很少出现)，比如system_server进程中的ActivityManagerService子线程

代码在[ActivityManagerService.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java)  6246行



```java
    @Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            //获取远程Binder调用端的pid
            int callingPid = Binder.getCallingPid();
            // 清除远程Binder调用端的uid和pid信息，并保存到origId变量
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            //通过origId变量，还原远程Binder调用端的uid和pid信息
            Binder.restoreCallingIdentity(origId);
        }
    }
```

> **attachApplication()**该方法一般是system_server进程的子线程调用远程进程时使用，而**attachApplicationLocked()**方法则在同一个线程中，故需要在调用该方法前清空远程调用该方法清空远程调用者的uid和pid，调用结束后恢复远程调用者的uid和pid。

## 三、Binder的死亡通知机制

### (一)、概述

> 死亡通知时为了让Bp端(客户端进程)能知晓Bn端(服务端进程)的生死情况，当Bn进程死亡后能通知到Bp端。
>
> - 定义：AppDeathRecipient是继承IBinder::DeathRecipient类，主要需要实现其binderDied()来进行死亡通告。
> - 注册：binder->linkToDeath(AppDeathRecipient)是为了将AppDeathRecipient死亡通知注册到Binder上

Bn端只需要重写binderDied()方法，实现一些后尾清楚类的工作，则在Bn端死掉后，会回调binderDied()进行相应处理。

### (二)、注册死亡通知

**1、Java层代码**

代码在[ActivityManagerService.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java) 6016行



```java
public final class ActivityManagerService {
    private final boolean attachApplicationLocked(IApplicationThread thread, int pid) {
        ...
        //创建IBinder.DeathRecipient子类对象
        AppDeathRecipient adr = new AppDeathRecipient(app, pid, thread);
        //建立binder死亡回调
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
        ...
        //取消binder死亡回调
        app.unlinkDeathRecipient();
    }

    private final class AppDeathRecipient implements IBinder.DeathRecipient {
        ...
        public void binderDied() {
            synchronized(ActivityManagerService.this) {
                appDiedLocked(mApp, mPid, mAppThread, true);
            }
        }
    }
}
```

这里面涉及两个方法linkToDeath和unlinkToDeath方法，实现如下：

**1.1、linkToDeath()与unlinkToDeath()**

代码在[ActivityManagerService.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Binder.java) 397行



```java
public class Binder implements IBinder {
    public void linkToDeath(DeathRecipient recipient, int flags) {
    }

    public boolean unlinkToDeath(DeathRecipient recipient, int flags) {
        return true;
    }
}
```

代码在[ActivityManagerService.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Binder.java) 509行



```java
final class BinderProxy implements IBinder {
    public native void linkToDeath(DeathRecipient recipient, int flags)
            throws RemoteException;
    public native boolean unlinkToDeath(DeathRecipient recipient, int flags);
}
```

可见，以上两个方法：

> - 当为Binder服务端，则相应的两个方法实现为空，没有实际功能；
> - 当为BinderProxy代理端，则调用native方法来实现相应功能，这是真实使用场景

**2、JNI及Native层代码**

native方法linkToDeath()和unlinkToDeath() 通过JNI实现，我们来依次了解。

**2.1 android_os_BinderProxy_linkToDeath()**

代码在[android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp) 397行



```php
static void android_os_BinderProxy_linkToDeath(JNIEnv* env, jobject obj,
        jobject recipient, jint flags)
{
    if (recipient == NULL) {
        jniThrowNullPointerException(env, NULL);
        return;
    }

    //第一步 获取BinderProxy.mObject成员变量值, 即BpBinder对象
    IBinder* target = (IBinder*)env->GetLongField(obj, gBinderProxyOffsets.mObject);
    ...

    //只有Binder代理对象才会进入该对象
    if (!target->localBinder()) {
        DeathRecipientList* list = (DeathRecipientList*)
                env->GetLongField(obj, gBinderProxyOffsets.mOrgue);
        //第二步 创建JavaDeathRecipient对象
        sp<JavaDeathRecipient> jdr = new JavaDeathRecipient(env, recipient, list);
        //第三步 建立死亡通知
        status_t err = target->linkToDeath(jdr, NULL, flags);
        if (err != NO_ERROR) {
            //如果添加失败，第四步 ， 则从list移除引用
            jdr->clearReference();
            signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/);
        }
    }
}
```

大体流程是:

> - 第一步，获取BpBinder对象
> - 第二步，构建JavaDeathRecipient对象
> - 第三步，调用BpBinder的linkToDeath，建立死亡通知
> - 第四步，如果添加死亡通知失败，则调用JavaDeathRecipient的clearReference移除

补充说明：

> - 获取DeathRecipientList：其成员变量mList记录该BinderProxy的JavaDeathRecipient列表信息(一个BpBinder可以注册多个死亡回调)
> - 创建JavaDeathRecipient：继承与IBinder::DeathRecipient

那我们就依照上面四个步骤依次详细了解下,获取BpBinder对象的过程和之前讲解Binder一样，这里就不详细说明了，直接从第二步开始。

**2.1.1 JavaDeathRecipient类**

代码在[android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp) 348行



```cpp
class JavaDeathRecipient : public IBinder::DeathRecipient
{
public:
    JavaDeathRecipient(JNIEnv* env, jobject object, const sp<DeathRecipientList>& list)
        : mVM(jnienv_to_javavm(env)), mObject(env->NewGlobalRef(object)),
          mObjectWeak(NULL), mList(list)
    {
        //将当前对象sp添加到列表DeathRecipientList
        list->add(this);
        android_atomic_inc(&gNumDeathRefs);
        incRefsCreated(env); 
    }
}
```

该方法主要功能：

> - 通过env->NewGloablRef(object)，为recipient创建相应的全局引用，并保存到mObject成员变量
> - 将当前对象JavaDeathRecipient强指针sp添加到DeathRecipientList

这里说下DeathRecipient关系图

![img](https:////upload-images.jianshu.io/upload_images/5713484-9b09d4a91503ea92.png?imageMogr2/auto-orient/strip|imageView2/2/w/877/format/webp)

DeathRecipient关系图.png

> 其中Java层的BinderProxy.mOrgue 指向DeathRecipientList，而DeathRecipientList记录JavaDeathRecipient对象

最后调用了incRefsCreated()函数，让我们来看下

**2.1.1.1 incRefsCreated()函数**

代码在[android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp) 144行



```cpp
static void incRefsCreated(JNIEnv* env)
{
    int old = android_atomic_inc(&gNumRefsCreated);
    if (old == 200) {
        android_atomic_and(0, &gNumRefsCreated);
        //出发forceGc
        env->CallStaticVoidMethod(gBinderInternalOffsets.mClass,
                gBinderInternalOffsets.mForceGc);
    } else {
        ALOGV("Now have %d binder ops", old);
    }
}
```

该方法的主要作用是增加引用计数incRefsCreated，每计数增加200则执行一次forceGC;
 会触发调用incRefsCreated()的场景有：

> - JavaBBinder 对象创建过程
> - JavaDeathRecipient对象创建过程
> - javaObjectForIBinder()方法：将native层的BpBinder对象转换为Java层BinderProxy对象的过程

**2.1.2 BpBinder::linkToDeath()**

代码在[BpBinder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/BpBinder.cpp) 173行



```cpp
status_t BpBinder::linkToDeath(
    const sp<DeathRecipient>& recipient, void* cookie, uint32_t flags)
{
    Obituary ob;
    //recipient 该对象为JavaDeathRecipient
    ob.recipient = recipient;
    // cookie 为null
    ob.cookie = cookie;
    // flags=0;
    ob.flags = flags;

    LOG_ALWAYS_FATAL_IF(recipient == NULL,
                        "linkToDeath(): recipient must be non-NULL");

    {
        AutoMutex _l(mLock);
        if (!mObitsSent) {
             // 没有执行过sendObituary，则进入该方法
            if (!mObituaries) {
                mObituaries = new Vector<Obituary>;
                if (!mObituaries) {
                    return NO_MEMORY;
                }
                ALOGV("Requesting death notification: %p handle %d\n", this, mHandle);
                getWeakRefs()->incWeak(this);
                IPCThreadState* self = IPCThreadState::self();
                //具体调用步骤1
                self->requestDeathNotification(mHandle, this);
                // 具体调用步骤2
                self->flushCommands();
            }
            // 将创新的Obituary添加到mbituaries
            ssize_t res = mObituaries->add(ob);
            return res >= (ssize_t)NO_ERROR ? (status_t)NO_ERROR : res;
        }
    }
    return DEAD_OBJECT;
}
```

这里面的核心代码的就是分别调用了** IPCThreadState的requestDeathNotification(mHandle, this)**函数和**flushCommands()**函数，那我们就一次来看下

**2.1.2.1 IPCThreadState::requestDeathNotification()函数**

代码在[BpBinder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp) 670行



```cpp
status_t IPCThreadState::requestDeathNotification(int32_t handle, BpBinder* proxy)
{
    mOut.writeInt32(BC_REQUEST_DEATH_NOTIFICATION);
    mOut.writeInt32((int32_t)handle);
    mOut.writePointer((uintptr_t)proxy);
    return NO_ERROR;
}
```

进入Binder Driver后，直接调用后进入binder_thread_write处理BC_REQUEST_DEATH_NOTIFICATION命令

**2.1.2.2 IPCThreadState::flushCommands()函数**

代码在[BpBinder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/IPCThreadState.cpp) 395行



```cpp
void IPCThreadState::flushCommands()
{
    if (mProcess->mDriverFD <= 0)
        return;
    talkWithDriver(false);
}
```

flushCommands就是把命令向驱动发出，此处参数是false，则不会阻塞等待读。向Linux Kernel层的Binder Driver发送 BC_REQEUST_DEATH_NOTIFACTION命令，经过ioctl执行到binder_ioctl_write_read()方法。

**2.1.3 clearReference()函数**

代码在[android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp) 412行



```cpp
void clearReference()
 {
     sp<DeathRecipientList> list = mList.promote();
     if (list != NULL) {
         // 从列表中移除
         list->remove(this); 
     }
 }
```

**3、Linux Kernel层代码**

**3.1、binder_ioctl_write_read()函数**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 3138行



```objectivec
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;
    // 把用户控件数据ubuf拷贝到bwr
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) { 
        ret = -EFAULT;
        goto out;
    }
    // 此时写入缓存数据
    if (bwr.write_size > 0) { 
        ret = binder_thread_write(proc, thread,
                  bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
         ...
    }
    //此时读缓存没有数据
    if (bwr.read_size > 0) {
      ...
    }
    // 将内核数据bwr拷贝到用户控件ubuf
    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) { 
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}
```

主要调用binder_thread_write来读写缓存数据，按我们来看下**binder_thread_write()**函数

**3.2、binder_thread_write()函数**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 2252行



```cpp
static int binder_thread_write(struct binder_proc *proc,
      struct binder_thread *thread,
      binder_uintptr_t binder_buffer, size_t size,
      binder_size_t *consumed)
{
  uint32_t cmd;
  //proc, thread都是指当前发起端进程的信息
  struct binder_context *context = proc->context;
  void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
  void __user *ptr = buffer + *consumed; 
  void __user *end = buffer + size;
  while (ptr < end && thread->return_error == BR_OK) {
    get_user(cmd, (uint32_t __user *)ptr); //获取BC_REQUEST_DEATH_NOTIFICATION
    ptr += sizeof(uint32_t);
    switch (cmd) {
        case BC_REQUEST_DEATH_NOTIFICATION:{ 
           //注册死亡通知
            uint32_t target;
            void __user *cookie;
            struct binder_ref *ref;
            struct binder_ref_death *death;
            //获取targe
            get_user(target, (uint32_t __user *)ptr); t
            ptr += sizeof(uint32_t);
             //获取BpBinder
            get_user(cookie, (void __user * __user *)ptr); 
            ptr += sizeof(void *);
            //拿到目标服务的binder_ref
            ref = binder_get_ref(proc, target); 

            if (cmd == BC_REQUEST_DEATH_NOTIFICATION) {
                //native Bp可注册多个，但Kernel只允许注册一个死亡通知
                if (ref->death) {
                    break; 
                }
                death = kzalloc(sizeof(*death), GFP_KERNEL);

                INIT_LIST_HEAD(&death->work.entry);
                death->cookie = cookie;
                ref->death = death;
                //当目标binder服务所在进程已死,则直接发送死亡通知。这是非常规情况
                if (ref->node->proc == NULL) { 
                    ref->death->work.type = BINDER_WORK_DEAD_BINDER;
                    //当前线程为binder线程,则直接添加到当前线程的todo队列. 
                    if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
                        list_add_tail(&ref->death->work.entry, &thread->todo);
                    } else {
                        list_add_tail(&ref->death->work.entry, &proc->todo);
                        wake_up_interruptible(&proc->wait);
                    }
                }
            } else {
                ...
            }
        } break;
      case ...;
    }
    *consumed = ptr - buffer;
  }   
 }
```

> 该方法在处理BC_REQUEST_DEATH_NOTIFACTION过程，正好遇到目标Binder进服务所在进程已死的情况，向todo队列增加BINDER_WORK_BINDER事务，直接发送死亡通知，但这属于非常规情况。

更常见的场景是binder服务所在进程死亡后，会调用binder_release方法，然后调用binder_node_release。这个过程便会发出死亡通知的回调。

### (三)、出发死亡通知

> 当Binder服务所在进程死亡后，会释放进程相关的资源，Binder也是一种资源。binder_open打开binder驱动/dev/binder，这是字符设备，获取文件描述符。在进程结束的时候会有一个关闭文件系统的过程会调用驱动close方法，该方法相对应的是release()方法。当binder的fd被释放后，此处调用相应的方法是binder_release()。

但并不是每个close系统调用都会出发调用release()方法。只有真正释放设备数据结构才调用release()，内核维持一个文件结构被使用多少次的技术，即便是应用程序没有明显地关闭它打开的文件也使用：内核在进程exit()时会释放所有内存和关闭相应的文件资源，通过使用close系统调用最终也会release binder。

**1、release**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 4172行



```cpp
static const struct file_operations binder_fops = {
  .owner = THIS_MODULE,
  .poll = binder_poll,
  .unlocked_ioctl = binder_ioctl,
  .compat_ioctl = binder_ioctl,
  .mmap = binder_mmap,
  .open = binder_open,
  .flush = binder_flush,
   //对应着release的方法
  .release = binder_release, 
};
```

那我们来看下binder_release

2、binder_release

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 3536行



```rust
static int binder_release(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc = filp->private_data;

    debugfs_remove(proc->debugfs_entry);
    binder_defer_work(proc, BINDER_DEFERRED_RELEASE);

    return 0;
}
```

我们看到里面调用了binder_defer_work()函数，那我们一起继续看下

**3、binder_defer_work**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 3739行



```rust
static void
binder_defer_work(struct binder_proc *proc, enum binder_deferred_state defer)
{
        //获取锁
    mutex_lock(&binder_deferred_lock);
        // 添加BINDER_DEFERRED_RELEASE
    proc->deferred_work |= defer;
    if (hlist_unhashed(&proc->deferred_work_node)) {
        hlist_add_head(&proc->deferred_work_node,
                &binder_deferred_list);
                //向工作队列添加binder_derred_work
        schedule_work(&binder_deferred_work);
    }
        // 释放锁
    mutex_unlock(&binder_deferred_lock);
}
```

这里面涉及到一个结构体**binder_deferred_workqueue**，那我们就来看下

**4、binder_deferred_workqueue**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 3737行



```cpp
static DECLARE_WORK(binder_deferred_work, binder_deferred_func);
```

代码在[workqueue.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/linux/workqueue.h) 183行



```csharp
#define DECLARE_WORK(n, f)                      \
    struct work_struct n = __WORK_INITIALIZER(n, f)
```

代码在[workqueue.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/linux/workqueue.h) 169行



```csharp
#define __WORK_INITIALIZER(n, f) {          \
  .data = WORK_DATA_STATIC_INIT(),        \
  .entry  = { &(n).entry, &(n).entry },        \
  .func = (f),              \
  __WORK_INIT_LOCKDEP_MAP(#n, &(n))        \
  }
```

上面看起来有点凌乱，那我们合起来看



```csharp
static DECLARE_WORK(binder_deferred_work, binder_deferred_func);

#define DECLARE_WORK(n, f)            \
  struct work_struct n = __WORK_INITIALIZER(n, f)

#define __WORK_INITIALIZER(n, f) {          \
  .data = WORK_DATA_STATIC_INIT(),        \
  .entry  = { &(n).entry, &(n).entry },        \
  .func = (f),              \
  __WORK_INIT_LOCKDEP_MAP(#n, &(n))        \
  }
```

那么 他是什么时候被初始化的？

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 4215行



```cpp
//全局工作队列
static struct workqueue_struct *binder_deferred_workqueue;
static int __init binder_init(void)
{
  int ret;
  //创建了名叫“binder”的工作队列
  binder_deferred_workqueue = create_singlethread_workqueue("binder");
  if (!binder_deferred_workqueue)
    return -ENOMEM;
  ...
}

device_initcall(binder_init);
```

在Binder设备驱动初始化过程中执行binder_init()方法中，调用**create_singlethread_workqueue(“binder”)**，创建了名叫**"binder"**的工作队列(workqueue)。workqueue是kernel提供的一种实现简单而有效的内核线程机制，可延迟执行任务。
 此处的binder_deferred_work的func为binder_deferred_func,接下来看该方法。

**5、binder_deferred_work**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 2697行



```rust
static void binder_deferred_func(struct work_struct *work)
{
    struct binder_proc *proc;
    struct files_struct *files;

    int defer;
    do {
        binder_lock(__func__);
                // 获取binder_main_lock
        mutex_lock(&binder_deferred_lock);
        if (!hlist_empty(&binder_deferred_list)) {
            proc = hlist_entry(binder_deferred_list.first,
                    struct binder_proc, deferred_work_node);
            hlist_del_init(&proc->deferred_work_node);
            defer = proc->deferred_work;
            proc->deferred_work = 0;
        } else {
            proc = NULL;
            defer = 0;
        }
        mutex_unlock(&binder_deferred_lock);

        files = NULL;
        if (defer & BINDER_DEFERRED_PUT_FILES) {
            files = proc->files;
            if (files)
                proc->files = NULL;
        }

        if (defer & BINDER_DEFERRED_FLUSH)
            binder_deferred_flush(proc);

        if (defer & BINDER_DEFERRED_RELEASE)
                         // 核心代码，调用binder_deferred_release()
            binder_deferred_release(proc); /* frees proc */

        binder_unlock(__func__);
        if (files)
            put_files_struct(files);
    } while (proc);
}
```

可见，binder_release最终调用的是binder_deferred_release；同理，binder_flush最终调用的是binder_deferred_flush。

**6、binder_deferred_release**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 3590行



```rust
static void binder_deferred_release(struct binder_proc *proc)
{
    struct binder_transaction *t;
    struct binder_context *context = proc->context;
    struct rb_node *n;
    int threads, nodes, incoming_refs, outgoing_refs, buffers,
        active_transactions, page_count;

    BUG_ON(proc->vma);
    BUG_ON(proc->files);

        //删除proc_node节点
    hlist_del(&proc->proc_node);

    if (context->binder_context_mgr_node &&
        context->binder_context_mgr_node->proc == proc) {
        binder_debug(BINDER_DEBUG_DEAD_BINDER,
                 "%s: %d context_mgr_node gone\n",
                 __func__, proc->pid);
        context->binder_context_mgr_node = NULL;
    }
  
        //释放binder_thread
    threads = 0;
    active_transactions = 0;
    while ((n = rb_first(&proc->threads))) {
        struct binder_thread *thread;

        thread = rb_entry(n, struct binder_thread, rb_node);
        threads++;
        active_transactions += binder_free_thread(proc, thread);
    }

        //释放binder_node
    nodes = 0;
    incoming_refs = 0;
    while ((n = rb_first(&proc->nodes))) {
        struct binder_node *node;

        node = rb_entry(n, struct binder_node, rb_node);
        nodes++;
        rb_erase(&node->rb_node, &proc->nodes);
        incoming_refs = binder_node_release(node, incoming_refs);
    }

        //释放binder_ref
    outgoing_refs = 0;
    while ((n = rb_first(&proc->refs_by_desc))) {
        struct binder_ref *ref;

        ref = rb_entry(n, struct binder_ref, rb_node_desc);
        outgoing_refs++;
        binder_delete_ref(ref);
    }

        //释放binder_work
    binder_release_work(&proc->todo);
    binder_release_work(&proc->delivered_death);

    buffers = 0;
    while ((n = rb_first(&proc->allocated_buffers))) {
        struct binder_buffer *buffer;

        buffer = rb_entry(n, struct binder_buffer, rb_node);

        t = buffer->transaction;
        if (t) {
            t->buffer = NULL;
            buffer->transaction = NULL;
            pr_err("release proc %d, transaction %d, not freed\n",
                   proc->pid, t->debug_id);
            /*BUG();*/
        }

                //释放binder_buf
        binder_free_buf(proc, buffer);
        buffers++;
    }

    binder_stats_deleted(BINDER_STAT_PROC);

    page_count = 0;
    if (proc->pages) {
        int i;

        for (i = 0; i < proc->buffer_size / PAGE_SIZE; i++) {
            void *page_addr;

            if (!proc->pages[i])
                continue;

            page_addr = proc->buffer + i * PAGE_SIZE;
            binder_debug(BINDER_DEBUG_BUFFER_ALLOC,
                     "%s: %d: page %d at %p not freed\n",
                     __func__, proc->pid, i, page_addr);
            unmap_kernel_range((unsigned long)page_addr, PAGE_SIZE);
            __free_page(proc->pages[i]);
            page_count++;
        }
        kfree(proc->pages);
        vfree(proc->buffer);
    }

    put_task_struct(proc->tsk);

    binder_debug(BINDER_DEBUG_OPEN_CLOSE,
             "%s: %d threads %d, nodes %d (ref %d), refs %d, active transactions %d, buffers %d, pages %d\n",
             __func__, proc->pid, threads, nodes, incoming_refs,
             outgoing_refs, active_transactions, buffers, page_count);

    kfree(proc);
}
```

此处proc是来自Bn端的binder_proc.
 binder_defered_release的主要工作有：

> - binder_free_thread(proc,thread)
> - binder_node_release(node,incoming_refs)
> - binder_delete_ref(ref)
>    -binder_release_work(&proc->todo)
>    -binder_release_work(&proc->delivered_death)
>    -binder_free_buff(proc,buffer)
>    -以及释放各种内存信息

**6.1、binder_free_thread**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 3065行



```rust
static int binder_free_thread(struct binder_proc *proc,
                  struct binder_thread *thread)
{
    struct binder_transaction *t;
    struct binder_transaction *send_reply = NULL;
    int active_transactions = 0;

    rb_erase(&thread->rb_node, &proc->threads);
    t = thread->transaction_stack;
    if (t && t->to_thread == thread)
        send_reply = t;
    while (t) {
        active_transactions++;
        binder_debug(BINDER_DEBUG_DEAD_TRANSACTION,
                 "release %d:%d transaction %d %s, still active\n",
                  proc->pid, thread->pid,
                 t->debug_id,
                 (t->to_thread == thread) ? "in" : "out");

        if (t->to_thread == thread) {
            t->to_proc = NULL;
            t->to_thread = NULL;
            if (t->buffer) {
                t->buffer->transaction = NULL;
                t->buffer = NULL;
            }
            t = t->to_parent;
        } else if (t->from == thread) {
            t->from = NULL;
            t = t->from_parent;
        } else
            BUG();
    }
        //发送失败回复
    if (send_reply)
        binder_send_failed_reply(send_reply, BR_DEAD_REPLY);
    binder_release_work(&thread->todo);
    kfree(thread);
    binder_stats_deleted(BINDER_STAT_THREAD);
    return active_transactions;
}
```

**6.2、binder_node_release**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 3546行



```php
static int binder_node_release(struct binder_node *node, int refs)
{
    struct binder_ref *ref;
    int death = 0;

    list_del_init(&node->work.entry);
    binder_release_work(&node->async_todo);

    if (hlist_empty(&node->refs)) {
                //引用为空，直接删除节点
        kfree(node);
        binder_stats_deleted(BINDER_STAT_NODE);

        return refs;
    }

    node->proc = NULL;
    node->local_strong_refs = 0;
    node->local_weak_refs = 0;
    hlist_add_head(&node->dead_node, &binder_dead_nodes);

    hlist_for_each_entry(ref, &node->refs, node_entry) {
        refs++;

        if (!ref->death)
            continue;

        death++;

        if (list_empty(&ref->death->work.entry)) {
                       //添加BINDER_WORK_DEAD_BINDER事务到todo队列
            ref->death->work.type = BINDER_WORK_DEAD_BINDER;
            list_add_tail(&ref->death->work.entry,
                      &ref->proc->todo);
            wake_up_interruptible(&ref->proc->wait);
        } else
            BUG();
    }
    binder_debug(BINDER_DEBUG_DEAD_BINDER,
             "node %d now dead, refs %d, death %d\n",
             node->debug_id, refs, death);
    return refs;
}
```

该方法会遍历该binder_node所有的binder_ref，当存在binder希望通知，则向相应的binder_ref所在进程的todo队列添加BINDER_WORK_DEAD_BINDER事务并唤醒处于proc->wait的binder线程。

**6.3、binder_delete_ref**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 1133行



```rust
static void binder_delete_ref(struct binder_ref *ref)
{
    binder_debug(BINDER_DEBUG_INTERNAL_REFS,
             "%d delete ref %d desc %d for node %d\n",
              ref->proc->pid, ref->debug_id, ref->desc,
              ref->node->debug_id);

    rb_erase(&ref->rb_node_desc, &ref->proc->refs_by_desc);
    rb_erase(&ref->rb_node_node, &ref->proc->refs_by_node);
    if (ref->strong)
        binder_dec_node(ref->node, 1, 1);
    hlist_del(&ref->node_entry);
    binder_dec_node(ref->node, 0, 1);
    if (ref->death) {
        binder_debug(BINDER_DEBUG_DEAD_BINDER,
                 "%d delete ref %d desc %d has death notification\n",
                  ref->proc->pid, ref->debug_id, ref->desc);
        list_del(&ref->death->work.entry);
        kfree(ref->death);
        binder_stats_deleted(BINDER_STAT_DEATH);
    }
    kfree(ref);
    binder_stats_deleted(BINDER_STAT_REF);
}
```

**6.4、binder_delete_ref**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 2980行



```php
static void binder_release_work(struct list_head *list)
{
    struct binder_work *w;

    while (!list_empty(list)) {
        w = list_first_entry(list, struct binder_work, entry);
                 //删除 binder_work
        list_del_init(&w->entry);
        switch (w->type) {
        case BINDER_WORK_TRANSACTION: {
            struct binder_transaction *t;

            t = container_of(w, struct binder_transaction, work);
            if (t->buffer->target_node &&
                !(t->flags & TF_ONE_WAY)) {
                                 //发送failed回复
                binder_send_failed_reply(t, BR_DEAD_REPLY);
            } else {
                binder_debug(BINDER_DEBUG_DEAD_TRANSACTION,
                    "undelivered transaction %d\n",
                    t->debug_id);
                t->buffer->transaction = NULL;
                kfree(t);
                binder_stats_deleted(BINDER_STAT_TRANSACTION);
            }
        } break;
        case BINDER_WORK_TRANSACTION_COMPLETE: {
            binder_debug(BINDER_DEBUG_DEAD_TRANSACTION,
                "undelivered TRANSACTION_COMPLETE\n");
            kfree(w);
            binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
        } break;
        case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
        case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: {
            struct binder_ref_death *death;

            death = container_of(w, struct binder_ref_death, work);
            binder_debug(BINDER_DEBUG_DEAD_TRANSACTION,
                "undelivered death notification, %016llx\n",
                (u64)death->cookie);
            kfree(death);
            binder_stats_deleted(BINDER_STAT_DEATH);
        } break;
        default:
            pr_err("unexpected work type, %d, not freed\n",
                   w->type);
            break;
        }
    }
}
```

**6.4、binder_delete_ref**

代码在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 2980行



```rust
static void binder_free_buf(struct binder_proc *proc,
                struct binder_buffer *buffer)
{
    size_t size, buffer_size;

    buffer_size = binder_buffer_size(proc, buffer);

    size = ALIGN(buffer->data_size, sizeof(void *)) +
        ALIGN(buffer->offsets_size, sizeof(void *)) +
        ALIGN(buffer->extra_buffers_size, sizeof(void *));

    binder_debug(BINDER_DEBUG_BUFFER_ALLOC,
             "%d: binder_free_buf %p size %zd buffer_size %zd\n",
              proc->pid, buffer, size, buffer_size);

    BUG_ON(buffer->free);
    BUG_ON(size > buffer_size);
    BUG_ON(buffer->transaction != NULL);
    BUG_ON((void *)buffer < proc->buffer);
    BUG_ON((void *)buffer > proc->buffer + proc->buffer_size);

    if (buffer->async_transaction) {
        proc->free_async_space += size + sizeof(struct binder_buffer);

        binder_debug(BINDER_DEBUG_BUFFER_ALLOC_ASYNC,
                 "%d: binder_free_buf size %zd async free %zd\n",
                  proc->pid, size, proc->free_async_space);
    }

    binder_update_page_range(proc, 0,
        (void *)PAGE_ALIGN((uintptr_t)buffer->data),
        (void *)(((uintptr_t)buffer->data + buffer_size) & PAGE_MASK),
        NULL);
    rb_erase(&buffer->rb_node, &proc->allocated_buffers);
    buffer->free = 1;
    if (!list_is_last(&buffer->entry, &proc->buffers)) {
        struct binder_buffer *next = list_entry(buffer->entry.next,
                        struct binder_buffer, entry);

        if (next->free) {
            rb_erase(&next->rb_node, &proc->free_buffers);
            binder_delete_free_buffer(proc, next);
        }
    }
    if (proc->buffers.next != &buffer->entry) {
        struct binder_buffer *prev = list_entry(buffer->entry.prev,
                        struct binder_buffer, entry);

        if (prev->free) {
            binder_delete_free_buffer(proc, buffer);
            rb_erase(&prev->rb_node, &proc->free_buffers);
            buffer = prev;
        }
    }
    binder_insert_free_buffer(proc, buffer);
}
```

### (四)、总结

> 对于Binder IPC进程都会打开/dev/binder文件，当进程异常退出时，Binder驱动会保证释放将要退出的进程中没有正常关闭的/dev/binder文件，实现机制是binder驱动通过调用/dev/binder文件所在对应的release回调函数，执行清理工作，并且检查BBinder是否注册死亡通知，当发现存在死亡通知时，就向其对应的BpBinder端发送死亡通知消息。

死亡回调DeathRecipient只有Bp才能正确使用，因为DeathRecipient用于监控Bn挂掉的情况，如果Bn建立跟自己的死亡通知，自己进程都挂了，就无法通知了。

清空引用，将JavaDeathRecipient从DeathRecipientList列表移除。







# Android跨进程通信IPC之Binder总结

## 一 、 Android为什么选用Binder作为最重要的IPC机制

我们知道在Linux系统中，进程间的通信方式有socket，named pipe，message queue， signal，sharememory等。这几种通信方式的优缺点如下：

> - name pipe：任何进程都能通讯，但速度慢
> - message queue：容量受到系统限制，且要注意第一次读的时候，要考虑上一次没有读完数据的问题。
> - signal：不能传递复杂消息，只能用来同步
> - shared memory：能够容易控制容量，速度快，但要保持同步，比如写一个进程的时候，另一个进程要注意读写的问题，相当于线程中的线程安全。当然，共享内存同样可以作为线程通讯，不过没有这个必要，线程间本来就已经共享了同一个进程内的一块内存。
> - socket：本机进程之间可以利用socket通信，跨进程之间也可利用socket通信，通常RPC的实现最底层都是通过socket通信。socket通信是一种比较复杂的通信方式，通常客户端需要开启单独的监听线程来接受从服务端发过来的数据，客户端线程发送数据给服务端，如果需要等待服务端的响应，并通过监听线程接受数据，需要进行同步，是一件很麻烦的事情。socket通信速度也不快。

Android中属性服务的实现和vold服务的实现采用了socket，getprop和setprop等命令都是通过socket和init进程通信来获的属性或者设置属性，vdc命令和mount service也是通过socket和vold服务通信来操作外接设备，比如SD卡

Message queue允许任意进程共享消息队列实现进程间通信，并由内核负责消息发送和接受之间的同步，从而使得用户在使用消息队列进行通信时不再需要考虑同步问题。这样使用方便，但是信息的复制需要额外消耗CPU时间，不适合信息量大或者操作频繁的场合。共享内存针对消息缓存的缺点改而利用内存缓冲区直接交换信息，无须复制，快递，信息量大是其优点。

共享内存块提供了在任意数量的进程之间进行高效的双向通信机制，每个使用者都可以读写数据，但是所有程序之间必须达成并遵守一定的协议，以防止诸如在读取信息之前覆盖内存空间等竞争状态的实现。不幸的是，Linux无法严格保证对内存块的独占访问，甚至是你通过使用IPC_PRIVATE创建新的共享内存块的时候，也不能保证访问的的独占性。同时，多个使用共享内存块的进程之间必须协调使用同一个键值。

Android应用程序开发者开发应用程序时，对系统框架的进程和线程运行机制不必了解，只需要利用四大组件开发，Android应用开发时可以轻易调用别的软件提供的功能，甚至可以调用系统App，在Android的世界里，所有应用都是平等的，但实质上应用进程被隔离在不同的沙盒里。

> Android平台的进程之间需要频繁的通信，比如打开一个应用便需要在Home应用程序进程和运行在system_server进程里的ActivityManagerService通信才能打开。正式由于Android平台的进程需要非常频繁的通信，故此对进程间通信机制要求比较高，速度要快，还要能进行复杂的数据的交换，应用开发时尽可能简单，并能提供同步调用。虽然共享内存的效率高，但是它需要复杂的同步机制，使用时很麻烦，故此不能采用。Binder能满足这些要求，所以Android选择了Binder作为最核心的进程间通信方式。Binder主要提供一下功能：
>
> - 1、用驱动程序来推进进程间的通信方式
> - 2、通过共享内存来提高性能
> - 3、为进程请求分配的每个进程的线程池，每个进程默认启动的两个Binder服务线程
> - 4、针对系统中的对象引入技术和跨进程对象的引用映射
> - 5、进程间同步调用。

## 二、Binder中相关的类简述

为了让大家更好的理解Binder机制，我这里把每个类都简单说下，设计到C层就是结构体。每个类/结构体都有一个基本的作用，还是按照之前的分类，如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-f87953c36119a382.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder类分布.png

关于其中的关系，比如继承，实现如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-b5009ac1ed8b141f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

类关系图.png

### (一)、Java层相关类

**1、IInterface**

> 类型：接口
>  作用：供Java层服务接口继承的接口，所有的服务提供者，必须继承这个接口

比如我们知道的ActivityManagerService继承自ActivityManagerNative，而ActivityManagerNative实现了IActivityManager，而IActivityManager继承自IInterface

**2、IBinder**

> 类型：接口
>  作用：Java层的IBinder类，定义了Java层Binder通信的一些规则；提供了transact方法来调用远程服务

比如我们知道的Binder类就是实现了IBinder

**3、Binder**

> 类型：类
>  作用：实现了IBinder接口，封装了JNI的实现。Java层Binder服务的基类。存在服务端的Binder对象

比如我们知道的ActivityManagerService继承自ActivityManagerNative，而ActivityManagerNative继承自Binder

**4、BinderProxy**

> 类型：类
>  作用：实现了IBinder接口，封装了JNI的实现，提供了transaction()方法提供进行远程调用

存在客户端的进程的服务端Binder的代理

**5、Parcel**

> 类型：类
>  作用：Java层的数据包装器，跨进程通信传递的数据的载体就是Parcel

我们经常的Parcelable其实就是将数据写入Parcel。具体可以看C++层的Parcel类

### (二)、JNI层相关类

**1、JavaBBinderHolder**

> 类型：类
>  作用：内部存储了JavaBBinder

**2、JavaBBinder**

> 类型：类
>  作用：将C++端的onTransact调用传递到Java端

JavaBBinder和JavaBBinderHolder相关的类类图如下所示(若看不清，请点击看大图)，JavaBBinder继承自本地框架的BBinder，代表Binder Service服务端的实体，而JavaBBinderHolder保存了JavaBBinder指针，Java层的Binder的mObject保存的JavaBBinderHolder指针的值，故此这里用聚合关系表示。BinderProxy的mObject保存的是BpBinder对象的指针的值，故此这里用聚合关系表示

![img](https:////upload-images.jianshu.io/upload_images/5713484-93f65aea9eb22bca.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder.png

**3、Java层Binder对象和NativeBinder对象相互转化的方法**

这里涉及两个重要的函数

> - 1、javaObjectForBinder(JNIEnv* env, const sp<IBinder>& val)——>将Native层的IBinder对象转化为Java层的IBinder对象。
> - 2、ibinderForJavaObject(JNIEnv* env, jobject obj)——> 将Java层的IBinder对象转化为Native层的IBinder对象

这样就实现了两层对象的转化

### (三)、Native层相关类

**1、BpRefBase**

> 类型：基类
>  作用：RefBase子类，提供remote方法获取远程Binder，Client端在查询ServiceManager获取所需的BpBinder后，BpRefBase负责管理当前获取的BpBinder实例。

**2、IInterface**

> 类型：基类
>  作用：Binder服务接口的基类，Binder服务通常需要同时提供本地接口和远程接口。它的子类分别声明了Client/Server能够实现所有的方法。

**3、BpInterface**

> 类型： 接口
>  作用：远程接口的积累，远程接口是供客户端调用的接口集
>  如通client端想要使用 Binder IPC与Service通信，那么首先会从SerrviceManager处查询并获得server端service的BpBinder，在client端，这个对象被认为是server端的远程代理。为了使Client能能够像本地一样调用一个远程server，server需要向client提供一个接口，client在这个接口的基础上创建一个BpInterface，使用这个对象，client的应用能够像本地一样直接调用server端的方法。而不是用去关心具体的Binder IPC实现

BpInterface的原型：



```cpp
template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
```

BpInterface是一个模板类，当server提供了INTERFACE接口(例如IXXXService),通常会继承BpInterface模板实现了一个BpXXXService



```ruby
class BpXXXService: public BpInterface<IXXXService>
```

> 实际上BpXXXService实现了双继承IXXXService和BpRefBase，这样既实现了Service中各方法的本地操作，将每个方法的参数以Parcel的形式发给Binder Driver；同时又将BpBinder作为自己的成员来管理，将BpBinder存储在mRemote中，通过调用BpRefBase的remote()函数来获取BpBinder的指针。

**4、BnInterface**

> 类型 ：接口
>  作用：本地接口的基类，本地接口是需要服务中真正实现的接口集合
>  BnInterface也是一个模板类。在定义Native端的Service时，基于server提供的INTERFACE接口(IXXXService)，通常会继承BnInterface模板类实现一个BnXXXService，而Native端的Service继承自BnXXXService。BnXXXService定义了一个onTransact函数，这个函数负责解包收到的Parcel并执行client端的请求方法。BnInterface的原型是：



```cpp
template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder
```

BnXXXService 例如：



```ruby
class BnXXXService: public BnInterface<IXXXService>
```

IXXXService为client端的代理接口BpXXXService和Server端的BnXXXServer的共同接口类，这个共同接口类的目的就是保证Service方法在C/S两端的一致性。

> 每个Service都可视为一个binder，而真正的Service端的Binder操作及状态的维护就是通过继承自BBinder来实现的。BBinder是Service作为BBinder的本质所在

那么，BBinder与BpBinder的区别是什么？BpBinder是Client端创建的用于向Server发送消息的代理，而BBinder是Server端用于接受消息的通道。他们代码中虽然均有transact方法，但两者的作用不同，BpBinder的transact方法时向IPCThreadStata实例发送消息，通知其有消息要发送给Binder Driver；而BBinder则当IPCThreadState实例收到Binder Driver消息时，通过BBinder的transact方法将其传递给它的子类BnXXXService的onTransact函数执行Server端的操作

**5、IBiner**

> 类型 接口
>  作用 Binder对象的基类，BBinder和BpBinder都是这个的类的子类

这个类比较重要，说一下他的几个方法

| 方法名                 | 说明                                                   |
| ---------------------- | ------------------------------------------------------ |
| localBinder            | 获取本地Binder对象                                     |
| remoteBinder           | 获取远程Binder对象                                     |
| transact               | 进行一次Binder操作                                     |
| queryLocalInterface    | 获取本地Binder，如果没有则返回NULL                     |
| getInterfaceDescriptor | 获取Binder的服务接口描述，其实就是Binder服务的唯一标识 |
| isBinderAlive          | 查询Binder服务是否还活着                               |
| pingBinder             | 发送PING_TRANSACTION给Binder服务                       |

**6、BpBinder**

> 类型 类
>  作用 远程Binder对象，这个类提供transact方法来发送请求，BpXXX中会用到。

BpBinder的实例代表了远程Binder，这个类的对象将被客户端调用。其中**handle()函数**将返回指向Binder服务实现者的句柄，这个类最重要的就是提供了**transact()函数**，这个函数将远程调用的参数封装好发送给Binder驱动。

**7、BBinder**

> 类型 基类
>  作用 本地Binder，服务实现方的基类，提供了onTransact接口来接收请求。

BBinder的实例代表了本地的Binder，它描述了服务的提供方，所有Binder服务的实现者都继承这个类(的子类)，在继承类中，最重要的就是实现**onTransact()函数**，因为这个方法是所有请求的入口。因此，这个方法是和BpBinder中的**transact()函数**对应的。

由于BBinder与BpBinder都是IBinder的子类，具体区别如下：



![img](https:////upload-images.jianshu.io/upload_images/5713484-dcd5b70996b24853.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

BBinder与BpBinder.png

**8、ProcessState**

> 类型 类
>  作用 代表Binder的进程

ProcessState是以单例模式设计的，每个进程在使用Binder机制进行通信时，均需要维护一个ProcessState实例来描述当前进程在Binder通信时Binder状态。

> ProcessState有两个主要功能：
>
> - 1、创建一个thread，该线程负责与内核中的Binder模块进行通信，该线程称为Poolthread；
> - 2、为指定的handle创建一个BpBinder对象，并管理该进程中所有的BpBinder对象。

在Binder IPC中，所有进程均会启动一个thread来负责与Binder Drive来通信，也就是不停的读写Binder Drive。Poolthread的启动方式为



```rust
ProcessState::self()->startThreadPool();
```

BpBinder的主要功能是负责Client向Binder Driver发送调用请求的数据，它是Client端Binder通信的核心对象，通过调用transact函数向Binder Driver发送调用请求数据。BpBinder的构造函数为



```cpp
BpBinder(int32_t handle);
```

该构造函数可见，BpBinder会将通信中的Server的handle记录下来，当有数据发送时，会通知Binder Driver数据的发送目标。
 ProcessState通过下下述方式来获取BpBinder对象。



```rust
ProcessState::self()->getContextObject(handle);
```

ProcessState创建的BpBinder实例，一般情况下会作为参数创建一个Client端的Service代理接口，例如BpXXX，在和ServiceManager通信时，Client会创建一个代理接口BpServieManager。

**9、IPCThreadState**

> 类型 类
>  作用 代表了使用Binder的线程，这个类中封装了与Binder驱动通信的逻辑，说白了就是负责与Binder Driver的驱动

IPCThreadState是以单例模式设计的。因为每个进程只维护了一个ProcessState实例，同时Process state只启动了一个Poolthread，因此每个进程只需要一个IPCThreadState即可。

Poolthread实际内容为：



```rust
IPCThreadState::self()->joinThreadPool();
```

> ProcessState中有两个Parcel对象，mIn和mOut。Poolthread会不停的查询Binder Driver中是否有数据可读，如果有，将其读出并保存到mIn;同时不听的查询mOut是否有数据要想Binder Driver发送，如果有，则将其内容写入Binder Driver中。总而言之，从Binder Driver读出的数据保存在mIn，待写入到Binder Driver中的数据保存在mOut中。

![img](https:////upload-images.jianshu.io/upload_images/5713484-316497e484da0247.png?imageMogr2/auto-orient/strip|imageView2/2/w/944/format/webp)

in与out.png

ProcessState中生成了BpBinder实例通过调用IPCThreadState的transact函数来向mOut中写入数据。

> IPCThreadState有两个重要的函数，talkWithDriver函数负责从Binder Driver续写数据，executeCommand函数负责解析并执行mIn中的数据

**10、Parcel**

> 类型 类
>  作用 在Binder上传递数据的包装器。

Parcel是Binder IPC中最基本的通信单元，它存储C/S间函数调用的参数。Parccel只能存储基本的数据类型，如果是复杂的数据类型的话，在存储时，需要将其拆分为基本的数据类型来存储。

> Binder通信的双方即可以作为Client，也可以作为Server，此时Binder通信是一个半双工的通信，操作的过程会比单工的情况复杂，但是基本原理是一样的。

**11、补充**

> 这里补充一下 这里的IInterface、IBinder和C++层的两个类是同名的。这个同名并不是巧合：它们不仅仅同名，它们所起的作用，以及其中包含的接口都是几乎一样的，区别仅仅是一个在C++层，一个是在Java层

**12、类关系**

下图描述了这些类之间的关系：

> PS:Binder的服务实现类(图中紫色部分)通常都会遵守下面的命名规则：
>
> - 服务的接口使用I字母作为前缀
> - 远程接口使用Bp作为前缀
> - 本地接口使用Bn作为前缀

![img](https:////upload-images.jianshu.io/upload_images/5713484-cd0f6c9e1f30b0d8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Native层类关系.png

**12、Native流程**

那我们来看下在Native层中的Binder流程，以MediaPlayer为例

![img](https:////upload-images.jianshu.io/upload_images/5713484-c0639c9c73ba7b98.png?imageMogr2/auto-orient/strip|imageView2/2/w/754/format/webp)

Native层的Binder流程.png



总结一下：

> - 1、在已知服务名的情况下，App通过getService()从ServiceManager获取服务的信息，该信息封装在Parcel里。
> - 2、应用程序收到返回的这个Parcel对象(通过Binder Drive)，从中读取出flat_binder_object对象，最终从对象中得到服务对应的服务号，mHandle。
> - 3、以该号码作为参数输入生成一个IBinder对象(实际上是BpBinder)。
> - 4、应用获取该对象后，通过asInterface(IBinder*)生成服务对应的Proxy对象(BpXXX)，并将其强转为接口对象(IXXX)，然后直接调用接口函数。
> - 5、所有接口对象调用最终会走到BpBinder->transact()函数，这个函数调用IPCThreadState->transact()并以Service号作为参数之一。
> - 6、最终通过系统调用ioctl()进入内核空间，Binder驱动根据传进来的Service号寻找该Service正处于等待状态的Binder Thread，唤醒它并在该线程内执行相应的函数，并返回结果给App。

强调一下：

> 1、从应用程序的角度来看，他只认识IBinder和IMediaPlayer这两个类，但真正的实现在BpBinder和BpMediaPlayer，这正式是设计模式中所推崇的“Programs to interface，not implementations”，可以说Android是一个严格遵循设计模式思想精心设计的系统。
>  2、客户端应该层层的封装，最终目的就是获取和传递这个mHandle值，从图中，我们看到，这个mHandle值来自与IServiceManager，他是一个管理其他服务的服务，通过服务的名字我们可以拿到这个服务对应的Handle号，类似网络域名服务系统。但是我们说了，IServiceManager也是服务，要访问他我们也需要一个Handle号，对了，就如同你必须为你的机器设置DNS服务器地址，你才能获得DNS服务。在Android系统里，默认的将ServiceManager的Handler号设为0，0就是DNS服务器的地址，这样，我们通过调用getStrongProxyForHandle(0)就可以拿到ServiceManager的IBinder对象，当然系统提供一个getService(char*)函数来完成这个过程。
>  3 Android Binder设计目的就是让访问远端服务就像调用本地函数一样简单，但是远端对象不再本地控制之内，我们必须保证调用过程中远端的对象不能被析构，否则本地应用程序将很可能崩溃。同时，万一远端服务异常退出，如Crash，本地对象必须知晓从而避免后续的错误。Android通过智能指针和DeathNotifacation来支持这两个请求。

Binder的Native层设计逻辑简单介绍完毕。我们接下来看看Binder的底层设计。

### (四)、Linux内核层的结构体

Binder驱动中有很多结构体，驱动中的结构体可以分为两类：

> - 与用户空间共用的，定义在[binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h)中
> - 仅在Binder Driver中使用的，定义在[binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)中

**1、与用户控件共用的结构体**

| 结构体名称              | 说明                                           |
| ----------------------- | ---------------------------------------------- |
| flat_binder_object      | 描述在Binder在IPC中传递的对象                  |
| binder_write_read       | 存储一次读写操作的数据                         |
| binder_version          | 存储Binder的版本号                             |
| transaction_flags       | 描述事务的flag，例如是否是异步请求，是否支持fd |
| binder_transaction_data | 存储一次事务的数据                             |
| binder_ptr_cookie       | 包含了一个指针和一个cookie                     |
| binder_handle_cookie    | 包含了一个句柄和一个cookie                     |

这里面**binder_write_read**和**binder_transaction_data**这两个结构体最为重要，它们存储了IPC调用过程中的数据

**2、仅在Binder驱动中使用**

| 结构体名称                   | 说明                                   |
| ---------------------------- | -------------------------------------- |
| binder_node                  | 描述了Binder实体节点，即对应一个Server |
| binder_ref                   | 描述对于Binder实体的引用               |
| binder_buffer                | 描述Binder通信过程中存储数据的Buffer   |
| binder_proc                  | 描述使用Binder的进程                   |
| binder_thread                | 描述Binder的线程                       |
| binder_work                  | 描述通信的一项任务                     |
| binder_transaction           | 描述一次事务的相关信息                 |
| binder_deferred_state        | 描述延迟任务                           |
| binder_ref_death             | 描述Binder实体死亡的信息               |
| binder_transaction_log       | debugfs日志                            |
| binder_transaction_log_entry | debugfs日志条目                        |

**3、总结**

> - 1 当一个service向Binder Driver注册时(通过flat_binder_object)，Binder Driver会创建一个binder_node，并挂载到service所在进程的nodes红黑树中。
> - 2 这个service的binder线程在proc->wait队列上进入睡眠等待。等待一个binder_work的到来。
> - 3 客户端的BpBinder创建的时候，它在Binder Driver内部也产生了一个binder_ref对象，并指向某个binder_node，在Binder Driver内部，将client和server关联起来。如果它需要或者Service的死亡状态，则会生成相应的binder_ref_death。
> - 4 客户端通过transact() (对应内核命令BC_TRANSACTION)请求远端服务，Driver 通过ref->node的映射，找到service所在进程，生产一个binder_buffer，binder_transaction和binder_work并插入proc->todo对下列，接着唤醒某个睡在proc->wait队列上的Binder_thread，与此同时，该客户端线程在其线程的wait队列上进入睡眠，等待返回值。
> - 5 这个binder thread 从proc->todo队列中读出一个binder_transaction，封装成transaction_data(命令为BR_TRANSACTION)并送到用户控件。Binder用户线程唤醒并最终执行对应的on_transact()函数
> - 6 Binder用户线程通过transact()向内核发送BC_REPLY命令，Driver收到后从其thread->transaction_stack中找到对应的binder_trannsaction，从而知道是哪个客户端线程正在等待这个返回
> - 7 Driver 生产新的binder_transaction(命令 BR_REPLY)，binder_buffer，binder_work，将其插入应用线程的todo队列，并将该线程唤醒。
> - 8 客户端的用户线程收到回复数据，该Transaction完成。
> - 9 当service所在进程发生异常，Driver的release函数被调用到，在某位内核work_queue线程里完成该service在内核态的清理工作(thread,buffer,node,work...)，并找到所有引用它的binder_ref，如果某个binder_ref有不在空的binder_ref_death，生成新的binder_work，送入其线程的todo队列，唤醒它来执行剩余工作，用户端的
>    DeathRecipient会最终调用来完成client端的清理工作。

西面这张时序图描述了上述一个transaction完成的过程。不同颜色代表不同的线程。注意的是，虽然Kernel和User space线程颜色是不一样的，但所有的系统调用都发生在用户进程的上下文李(所谓上下文，就是Kernel能通过某种方式找到关联的进程，并完成进程的相关操作，比如唤醒某个睡眠线程，或跟用户控件交换数据，copy_from，copy_to，与之相对应的是中断上下文，其完全异步出发， 因此无法做任何与进程相关的操作，比如睡眠，锁等).

![img](https:////upload-images.jianshu.io/upload_images/5713484-80862f6f574f543b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

transaction过程.png

### (五)、Binder整个通信个过程

![img](https:////upload-images.jianshu.io/upload_images/5713484-536d64d781904a95.png?imageMogr2/auto-orient/strip|imageView2/2/w/965/format/webp)

Binder整个通信个过程.png

## 三、Binder机制概述

前面几篇文章分别从驱动，Native，Framework层介绍了Binder，那我们就来总结一下：

> - 1、从IPC角度来说：Binder是Android中的一种跨进程通信方式，该方式在linux中没有，是Android独有的。
> - 2、从Android Driver层：Binder还可以理解为一种虚拟物理设备，它的设备驱动是/dev/binder。
> - 3、从Android Native层：Binder创建Service Manager以及BpBinder/BBinder模型，大推荐与binder驱动的桥梁
> - 4、从Android Framework层：Binder是各种Manager(ActivityManager,WindowManager等)和相应的xxManagerService的桥梁
> - 5、从Android App层：Binder是客户端和服务端进行通信的媒介，当binderService时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

## 四、Binder通信概述

Binder通信是一种C/S结构的通信结构。

> - 从表面上来看，是client通过获得一个server的代理接口，对server进行直接调用。
> - 实际上，代理接口中定义的方法与server中定义的方法是一一对应的。
> - Client端调用某这个代理接口中的方法时，代理接口的方法会将client传递的参数打包成Parcel对象。
> - 代理接口将该Parcel发送给内核中的Binder Driver。
> - Server端会读取Binder Driver中的请求的数据，如果是发送给自己的，解包Parcel对象，处理并将结果返回。
> - 整个的调用过程是一个同步过程，在server处理的时候，client会block住。

整体流程如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-cc505bcf669da946.png?imageMogr2/auto-orient/strip|imageView2/2/w/366/format/webp)

Binder通信概述.png

## 五、Binder协议

Binder协议可以分为控制协议和驱动协议两类

### (一) Binder控制协议

Binder控制协议是 进程通过 ioctl("/dev/binder") 与Binder设备进行通讯的协议，该协议包含以下几种命令：

| 命令                   |                            说明                             |          参数类型 |
| ---------------------- | :---------------------------------------------------------: | ----------------: |
| BINDER_WRITE_READ      | 读写操作，最常用的命令。IPC过程就是通过这个命令进行数据传递 | binder_write_read |
| BINDER_SET_MAX_THREADS |                 设置进程支持的最大线程数量                  |            size_t |
| BINDER_SET_CONTEXT_MGR |                  设置自身为ServiceManager                   |                无 |
| BINDER_THREAD_EXIT     |                   通知驱动Binder线程退出                    |                无 |
| BINDER_VERSION         |                   获取Binder驱动的版本号                    |    binder_version |

### (二) Binder驱动协议

Binder驱动协议描述了对Binder驱动的具体使用过程。驱动协议又可以分为两类：

> - binder_driver_command_protocol:描述了进程发送给Binder驱动的命令
> - binder_driver_return_protocol:描述了Binder驱动发送给进程的命令

**1、binder_driver_command_protocol 共包含17条命令，分别如下：**

| 命令                          |                  说明                  |                参数类型 |
| ----------------------------- | :------------------------------------: | ----------------------: |
| BC_TRANSACTION                | Binder事务，即：Client对于Server的请求 | binder_transaction_data |
| BC_REPLAY                     |  事务的应答，即Server对于Client的回复  | binder_transaction_data |
| BC_FREE_BUFFER                |           通知驱动释放Buffer           |        binder_uintptr_t |
| BC_ACQUIRE                    |              强引用技术+1              |                    _u32 |
| BC_RELEASE                    |              强引用技术-1              |                    _u32 |
| BC_INCREFS                    |                弱引用+1                |                    _u32 |
| BC_DECREFS                    |               弱引用 -1                |                    _u32 |
| BC_ACQUIRE_DONE               |            BR_ACQUIRE的回复            |       binder_ptr_cookie |
| BC_INCREFS_DONE               |            BR_INCREFS的回复            |       binder_ptr_cookie |
| BC_ENTER_LOOPER               |          通知驱动主线程ready           |                    void |
| BC_REGISTER_LOOPER            |          通知驱动子线程ready           |                    void |
| BC_EXIT_LOOPER                |          通知驱动线程已经退出          |                    void |
| BC_REQUEST_DEATH_NOTIFICATION |            请求接受死亡通知            |       binder_ptr_cookie |
| BC_CLEAR_DEATH_NOTIFICATION   |            去除接收死亡通知            |       binder_ptr_cookie |
| BC_DEAD_BINDER_DONE           |           已经处理完死亡通知           |        binder_uintptr_t |

**BC_ATTEMPT_ACQUIRE** 和 **BC_ACQUIRE_RESULT** 暂未实现。

**2、binder_driver_return_protocol 共包含18条命令，分别如下：**

| 返回类型                         |                  说明                  |                参数类型 |
| -------------------------------- | :------------------------------------: | ----------------------: |
| BR_OK                            |                操作完成                |                    void |
| BR_NOOP                          |                操作完成                |                    void |
| BR_ERROR                         |                发生错误                |                    _s32 |
| BR_TRANSACTION                   |  通知进程收到一次Binder请求(Server端)  | binder_transaction_data |
| BR_REPLAY                        | 通知进程收到Binder请求的回复(Client端) | binder_transaction_data |
| BR_TRANSACTION_COMPLETE          |       驱动对于接受请求的确认税负       |                    void |
| BR_FAILED_REPLAY                 |        告知发送发通信目标不存在        |                    void |
| BR_SPWAN_LOOPER                  |     通知Binder进程创建一个新的线程     |                    void |
| BR_ACQUIRE                       |              强引用计数+1              |       binder_ptr_cookie |
| BR_RELEASE                       |              强引用计数-1              |       binder_ptr_cookie |
| BR_INCREFS                       |              弱引用计数+1              |       binder_ptr_cookie |
| BR_DECREFS                       |              弱引用计数-1              |       binder_ptr_cookie |
| BR_DEAD_BINDER                   |              发送死亡通知              |        binder_uintptr_t |
| BR_CELAR_DEATH_NOTIFACATION_DONE |            清理死亡通知完成            |        binder_uintptr_t |
| BR_DEAD_REPLY                    |         告知发送方对方已经死亡         |                    void |

**BR_ACQUIRE_RESULT** 、**BR_FINISHED** 和 **BR_ATTEMPT_ACQUIRE** 暂未实现。

单独看上面的协议可能很难理解，这里我们可以将一次Binder请求过程来看一下Binder协议是如何通信的，就比较好理解了，如下图：

图说明：

> - Binder是C/S机构，通信从过程涉及到Client、Service以及Binder驱动三个角色
> - Client对于Server的请求以及Server对于Client回复都需要通过Binder驱动来中转数据
> - BC_XXX 命令是进城发送给驱动的命令
> - BR_XXX 命令是驱动发送给进程的命令
> - 整个通信过程有Binder驱动控制

![img](https:////upload-images.jianshu.io/upload_images/5713484-d4b05ca7e314bc2f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder通信.png

补充说明，通过上面的Binder协议，我们知道，Binder协议的通信过程中，不仅仅是发送请求和接收数据这些命令。同时包括了对于引用计数的管理和对于死亡通知管理的功能(告知一方，通信的另外一方已经死亡)。这些功能的通信过程和上面这幅图类似：

> 一方发送BC_XXX，然后由驱动控制通信过程，接着发送对应BR_XXX命令给通信的另外一方。因为这种相似性，我就不多说了。

上面提到了Binder的架构，那我们下面就来研究下Binder的结构

## 六、Binder架构

### (一)、Binder架构的思考

在说到Binder架构之前，先简单说说大家熟悉的TCP/IP五层通信系统结构

![img](https:////upload-images.jianshu.io/upload_images/5713484-fa7c194f6ac98235.png?imageMogr2/auto-orient/strip|imageView2/2/w/572/format/webp)

TCP/IP五层模型结构.png

> - 应用层：直接为用户提供服务
> - 传输层：传输的是报文(TCP数据)或者用户数据报(UDP数据)
> - 网络层：传输的是包(Paceket)，例如路由器
> - 数据链路层：传输的是帧(Frame)，例如以太网交互机
> - 物理层：相邻节点间传输bit，例如集线器，双绞线等

这是经典的五层TCP/IP协议体系，这样分层设计的思想，让每一个子问题都设计成一个独立的协议，这协议的设计/分析/实现/测试都变得更加简单：

> - 层与层具有独立性，例如应用层可以使用传输层提供的功能而无需知晓其原理原理。
> - 设计灵活，层与层之间都定义好接口，即便层内方法发生变化，只有接口不变，对这个系统便毫无影响。
> - 结构的解耦合，让每一层可以用更适合的技术方案，更适合的语言
> - 方便维护，可分层调试和定位问题

Binder架构也是采用分层架构设计，每一层都有其不同的功能，以大家平时用的startService为例子，AMP为ActivityManagerProxy，AMS为ActivityManagerSerivce 如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-35382c0eca87fb0e.png?imageMogr2/auto-orient/strip|imageView2/2/w/947/format/webp)

Binder分层.png

> - Java应用层：对于上层通过调用AMP.startService，完全可以不用去关心底层，经过层层调用，最终必然会调用到AMS.startService。
> - Java   IPC层：Binder采用的是C/S脚骨，Android系统的基础架构便已经设计好的Binder在Java Framework层的Binder 客户端BinderProxy和服务端Binder。
> - Native IPC层： 对于Native层，如果需要使用Binder，则可以直接使用BpBinder和BBinder(也包括JavaBBindder)即可，对于上一层Java IPC通信也是基于这个层面。
> - Kernel物理层：这里是Binder Driver，前面三层都跑在用户控件，对于用户控件内存资源是不共享的，每个Android的进程只能运行在自己基础讷航所拥有的虚拟地址空间，而内核空间却是可共享的。真正通信的核心环节还是Binder Driver。

### (二) 、Binder结构

![img](https:////upload-images.jianshu.io/upload_images/5713484-24090a9e885c0176.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder架构.png

> Binder在整个Android系统中有着举足轻重的地位，在Native层有一套完成的Binder通信的C/S架构（图中的蓝色），BpBinder作为客户端，BBinder作为服务端。基于native层的Binder框架，Java也有一套镜像功能的Binder C/S架构，通过JNI技术，与Native层的Binder对应，Java层的Binder功能最终都是交给native的Binder来完成。从kernel到native，jni，framwork层的架构所涉及的所有有关类和方法见下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-992dc03f9e3cfa0f.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/944/format/webp)

java_binder_framework.jpg

### (三) 、startService的流程

如下图:

![img](https:////upload-images.jianshu.io/upload_images/5713484-07ecf157e4f003ea.png?imageMogr2/auto-orient/strip|imageView2/2/w/1080/format/webp)

startService的流程.png

### (四)、SeviceManager自身的注册和其他service的注册

这里放一张图说明整个过程

![img](https:////upload-images.jianshu.io/upload_images/5713484-58d4e7afcd49bed7.png?imageMogr2/auto-orient/strip|imageView2/2/w/858/format/webp)





# Android跨进程通信IPC之其他IPC方式

## 一、Bundle

### (一)、IPC中的Bundle

> 我们知道，四大组件中三大组件(Activity，Service,Receiver)都是支持在Intent中传递Bundle数据的，由于Bundle中实现了Parcelable接口，所以它方便地在不同的进程间传输。基于这一点，当我们在一个进程中启动了另一个进程的Activity、Service和Receiver，我们就可以在Bundle中附加我们需要传输给远程进程的信息并通过Intent发送出去。当然，我们传输的数据必须能够序列化，比如基本类型、实现了Parcelable接口的对象、实现了Serializable接口的对象以及一些Android支持的特殊对象，具体内容可以看Bundle这个类，就是可以看到所有它支持的类型。Bundle不支持的类型我们无法通过他在进程间传递，这是最简单的进程间通信方式。

除了直接传递数据这种典型的使用场景，它还有一种特殊的使用场景。比如进程A正在进行一个计算，计算完成后它要启动B进程的一个组件并把计算结果传递给B进程，可是遗憾的是这个计算结果不支持放入Bundle中，因此无法通过Intent来传输，这个时候如果我们用其他IPC方式就会略显复杂。可以考虑如下方式：我们通过Intent启动进程B的一个Service组件(比如IntentService)，让Service在后台进行计算：计算完毕后再启动B进程中真正要启动的目标组件，由于Service也运行在B进程中，所以目标组件就可以接获取计算结果，这样一来就轻松解决了跨进程的问题。这种方式的核心思想在于将原本需要在A进程的计算任务转移到B进程的后台Service中去执行，这样就成功避免了进程间通信问题，而且只用了很小的代价

### (二)、Bundle类简介

> 根据google官方文档 [https://developer.android.com/reference/android/os/Bundle.html](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/Bundle.html)Bundle类是一个key-value对，**"A mapping from String keys to various Parcelable values."**

可以看出，它和Map类型有异曲同工之妙，其实它内部是使用ArrayMap来存储的，并且实现了Parcelable接口，那么它是支持进程间通信的。所以Bundle可以看做是一个特殊的Map类型，它支持进程间通信，保存了特定的数据。

PS:Bundle继承自[BaseBundle](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/BaseBundle.html) ，而在[BaseBundle](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/BaseBundle.html) 中有一个内部变量叫mMap就是**ArrayMap**类型

### (三)、Bundle类的重要方法

> - clear()：清楚此Bundle映射中的所有保存的数据
> - clone()：克隆当前Bundle
> - containKey(String key)：是否包含key
> - getString(String key)：返回key对应的String类型的value
> - hasFileDescriptors()：指示是否包含任何捆绑打包文件描述
> - isEmpty()：判断是否为空
> - putXxx(String key,Xxx value)：插入一个键值对
> - writeToParcel(Parcel parcel，int flags)，写入一个parcel
> - readFromParcel(Parcel parcel) 读取一个parcel
> - remove(String key)：移除特定key的值

## 二、文件共享

> 共享文件也是一种不错的进程间通信方式，两个进程通过读/写同一个文件来交换数据，比如A进程把数据写入文件，B进程通过读取这个文件来获取数据。由于Android是基于Linux的，所以并发读/写文件可以没有限制地进行，甚至两个线程同时对同一份文件进行读写操作都是允许的，尽管这可能出现问题。通过文件交换数据很方便使用，除了可以交换一些文本信息外，我们还可以序列化一个对象到文件系统中的同时从另一个进程中恢复这个对象。

PS:反序列化得到的对象只是在内容上和序列化之前的对象是一样的，但它们本质上还是两个对象。

通过文件共享这种方式共享数据对文件格式是没有具体要求的，比如可以是文本文件，也可以是XML文件，只要读/写双方约定数据格式即可。通过文件共享的方式也是有局限的，比如并发读/写的问题。如果并发读/写，那么我们读出的内容就有可能不是最新的，如果是并发写的话，那就更严重了。因此我们要尽量避免比规范法写这种情况发生或者考虑用线程同步来限制多个线程的写操作。通过上面的分析，我们可以知道，文件共享方式适合在对数据同步要求不高的进程之间进行通信，并且妥善处理并发读/写的问题。

> 说到文件共享又不得不说下SharedPreferences(后面简称SP)，当然SP是个特例，众所周知，SP是Android中提供的轻量级存储方案，它通过键值对的方式来存储数据，在底层实现上它采用XML文件来存储键值对，每个应用的SP文件都可以在当前包所在的data目录下查看到。一般来说，它的目录位于/data/data/package name/shared_prefs 目录下，其中package name表示的是当前应用的包名。从本质上来说，SP也属于文件的一种，但是由于系统对它的读/写有一定缓存策略，即在内存会有一份SP文件的缓存，因此在多进程模式下，系统对它的读/写就变的不可靠，当面对高并发的读/写访问，SP有很大几率会丢失数据，因此，不建议在进程间通信中使用SP。

## 三、Messenger

### (一)、概述

前面[Android跨进程通信IPC之11——AIDL](https://www.jianshu.com/p/375e3873b1f4)讲解了AIDL，用于Android进程间的通信。大家知道用编写AIDL比较麻烦，有没有比较"好的"AIDL。那我们就来介绍Messenger。

> Messenger是一种轻量级的IPC方案，其底层实现原理就是AIDL，它对AIDL做了一次封装，所以使用方法会比AIDL简单，由于它的效率比较低，一次只能处理一次请求，所以不存在线程同步的问题。

Messenger的官网地址是[https://developer.android.com/reference/android/os/Messenger.html](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/Messenger.html)
 看下官网描述

> Reference to a Handler, which others can use to send messages to it. This allows for the implementation of message-based communication across processes, by creating a Messenger pointing to a Handler in one process, and handing that Messenger to another process.

大概的意思是说，首先Messenger要与一个Handler相关联，才允许以message为基础的会话进行跨进程通讯。通讯创建一个messenger指向一个handler在同一个进程内，然后就可以在另一个进程处理messenger了。

这里的重点是



```undefined
This allows for the implementation of message-based communication across processes
```

**允许实现基于消息的进程间通讯方式。**
 那么什么是基于消息的进程间通信方式？看图理解下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-338c2db71a490dfc.png?imageMogr2/auto-orient/strip|imageView2/2/w/668/format/webp)

允许实现基于消息的进程间通讯方式.png

可以看到，我们可以在客户端发送一个Message给服务端，在服务端的handler会接受客户端的消息，然后进行队形的处理，处理完成后，在将结果等数据封装成Message，发送给客户端，客户端的handler中会接收到处理的结果。

这样对比AIDL就方便很多了

> - 基于 Message，详细大家都容易上手
> - 支持回调的方式，也就是服务端处理完任务可以和客户端交互
> - 不需要编写AIDL文件
>    此外，还支持，记录客户端对象的Messenger，然后可以实现一对多的通信；甚至作为一个转接处，任意两个进程都能通过服务端进行通信。

### (二)、构造函数

通过我们分析Messenger类的结构，如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-f42a7818db691871.png?imageMogr2/auto-orient/strip|imageView2/2/w/1152/format/webp)

Messenger类的结构.png



我们发现Messager有两个构造函数

**1 构造函数1**

代码在[Messenger.java)](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Messenger.java) 43行



```dart
    /**
     * Create a new Messenger pointing to the given Handler.  Any Message
     * objects sent through this Messenger will appear in the Handler as if
     * {@link Handler#sendMessage(Message) Handler.sendMessage(Message)} had
     * been called directly.
     *
     * @param target The Handler that will receive sent messages.
     */
    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
```

先翻译下注释

> 传入一个Handler作为参数创建一个新的Messenger，通过该Messenger发送的任何消息对象将会出现在这个Handler里面,就像Handler直接调用sendMessage(Message)一样。入参target接收这些已经发送的消息。

**1.1 getIMessenger()方法**

我们看到这个构造函数很简单，就是调用了getIMessenger()方法，我们跟踪一下，代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)  708行



```java
    final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl();
            return mMessenger;
        }
    }
```

这里使用了线程同步的懒汉式单例模式。并且这里面new了一个MessengerImpl()对象，MessengerImpl其实是Handler的内部类

**1.2 MessengerImpl类**

[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)  718行



```java
    private final class MessengerImpl extends IMessenger.Stub {
        public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid();
            Handler.this.sendMessage(msg);
        }
    }
```

> - 1、IMessenger,Stub 这样的结构大家有没有印象？是不是很想AIDL里面的形式，我们先搜一下IMessenger，发现没有对应的.java文件，只有一个[IMessenger.aidl](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/IMessenger.aidl)，所以说Messenger底层是基于AIDL的。
> - 2、当发送消息的时候，调用是**Handler.this.sendMessage(msg);**，其实就是自己调用的sendMessage(msg)方法而已。

**2 构造函数2**

代码在[Messenger.java)](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Messenger.java) 145行



```php
   /**
     * Create a Messenger from a raw IBinder, which had previously been
     * retrieved with {@link #getBinder}.
     *
     * @param target The IBinder this Messenger should communicate with.
     */
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
```

先来翻译一下注释

> 根据原始的IBinder来创建一个Messenger对象，可以通过**getBinder()**方法来获取这个IBinder对象。入参target其实就是这个Messenger与之通信的

这个更简单了，其中**IMessenger.Stub.asInterface(target);**这个方法简直就是AIDL方式里面讲到的方法，有没有？

### (三)、其他两个重要方法

**1、getBinder()方法**

代码在[Messenger.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Messenger.java)66行



```cpp
    /**
     * Retrieve the IBinder that this Messenger is using to communicate with
     * its associated Handler.
     * 
     * @return Returns the IBinder backing this Messenger.
     */
    public IBinder getBinder() {
        return mTarget.asBinder();
    }
```

> 注释翻译：返回这个Messenger使用Handler与之通讯的IBinder对象

返回一个IBinder对象，一般在服务端的onBinder()方法里面调用这个方法，返回给客户端一个IBinder对象。

**2、send(Message msg)方法**

代码在[Messenger.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Messenger.java)56行



```java
    /**
     * Send a Message to this Messenger's Handler.
     * 
     * @param message The Message to send.  Usually retrieved through
     * {@link Message#obtain() Message.obtain()}.
     * 
     * @throws RemoteException Throws DeadObjectException if the target
     * Handler no longer exists.
     */
    public void send(Message message) throws RemoteException {
        mTarget.send(message);
    }
```

> 注释翻译：发送消息给Messenger的Handler。入参message是要被发送的消息，通常通过Message.obtain()来获取，如果目标Handler不存在，就抛出RemoteException异常

发送一个message对象到 messagerHandler。这里我们传递的参数是一个Message对象

### (四)、举例说明

**1 服务端进程**

首先我们需要在服务端创建一个Service来处理客户端的连接请求，同时创建一个Handler并通过它来创建一个Messenger对象，然后在Service的onBinder中返回这个Messenger对象底层的Binder即可。
 代码如下



```java
public class MessageService extends Service {

    private  static class MessaengerHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what){
                case Constants.CLENT_MESSAGE:
                    Log.i("GELAOLITOU","收到消息："+msg.getData().get(Constants.KEY));
                    break;
                default:
                    break;
            }
        }
    }

    private Messenger messenger=new Messenger(new MessaengerHandler());

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}
```

当然别忘记在AndroidManifest.xml里面注册



```xml
        <service android:name=".MessageService"
                      android:process=".remote" />
```

**2 客户端进程**

客户端进程中，首先要绑定服务端的Service，绑定成功后用服务端返回的IBinder对象创建一个Messenger，通过这个Messenger就可以向服务端发送消息了，发消息类型为Message对象。如果需要服务端能够回应客户端，就和服务端一样，我们还要创建一个Handler并创建一个新的Messenger，并把这个Messenger对象通过Message的replyTo参数传递给服务端，服务端通过这个replyTo参数就可以回应客户端。这听起来可能还是有点抽象。那我们就直接上代码



```java
public class MainActivity extends AppCompatActivity {


    private Messenger mService;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent=new Intent(this,MessageService.class);
        bindService(intent,connection, Context.BIND_AUTO_CREATE);

    }


    private ServiceConnection connection=new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mService=new Messenger(service);
            //创建一个what值为Constants.CLENT_MESSAGE的message
            Message message=Message.obtain(null,Constants.CLENT_MESSAGE);
            Bundle bundle=new Bundle();
            bundle.putString(Constants.KEY,"我是来自客户端的信息");
            message.setData(bundle);
            try{
                mService.send(message);
            }catch (Exception e){
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onDestroy() {
        unbindService(connection);
        super.onDestroy();
    }
}
```

运行app，得出下面的截图



![img](https:////upload-images.jianshu.io/upload_images/5713484-28685c7b92313c9c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

结果.png

### **(五)、服务端响应客户端请求**

上面的例子演示了如何在服务端接收客户端中发送的消息，但是有时候我们还需要能回应客户端，下面就介绍如何实现这种效果。还是采用上面的例子，但是稍微做一下修改，每当客户端发送来一条消息，服务端就会回复一条"你的信息我已经收到，现在就回复你"。

**1、首先服务端的修改**

服务端只修改MessengerHandler，当收到消息后，会立即回复一条消息给客户端。



```java
public class MessageService extends Service {


    private  static class MessaengerHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what){
                case Constants.CLENT_MESSAGE:
                    Log.i("GELAOLITOU","收到消息："+msg.getData().get(Constants.KEY));
                    Messenger msgr_client= msg.replyTo;
                    Message mes_reply=Message.obtain(null,Constants.SERVER_MESSAGE);
                    Bundle bundle=new Bundle();
                    bundle.putString(Constants.REPLY_KEY,"你的信息我已经收到，现在就回复你");
                    mes_reply.setData(bundle);

                    try {
                        msgr_client.send(mes_reply);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
                default:
                    break;
            }
        }
    }

    private Messenger messenger=new Messenger(new MessaengerHandler());

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}
```

**2、其次客户端的修改**

为了接收服务端的回复，客户端也需要准备一个接收消息的Messenger和Handler。代码如下：



```java
public class MainActivity extends AppCompatActivity {


    private Messenger mService;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent=new Intent(this,MessageService.class);
        bindService(intent,connection, Context.BIND_AUTO_CREATE);

    }


    private ServiceConnection connection=new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mService=new Messenger(service);
            //创建一个what值为Constants.CLENT_MESSAGE的message
            Message message=Message.obtain(null,Constants.CLENT_MESSAGE);
            Bundle bundle=new Bundle();
            bundle.putString(Constants.KEY,"我是来自客户端的信息");
            message.setData(bundle);
            try{
                mService.send(message);
            }catch (Exception e){
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };


    private Messenger mGetReplyMessenger=new Messenger(new MessengerClientHandler());

    private static class MessengerClientHandler extends Handler{

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case Constants.SERVER_MESSAGE:
                    Log.i("GEBILAOLITOU","收到 服务端的消息，消息内容是"+msg.getData().getString(Constants.REPLY_KEY));
                    break;
            }

        }

    }


    @Override
    protected void onDestroy() {
        unbindService(connection);
        super.onDestroy();
    }
}
```

除了上述修改，还有很多关键的一点，当客户端发送消息的时候，需要把接收服务端的Messenger通过Message的replyTo参数传递给服务端，如下所示：



```java
    private ServiceConnection connection=new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mService=new Messenger(service);
            //创建一个what值为Constants.CLENT_MESSAGE的message
            Message message=Message.obtain(null,Constants.CLENT_MESSAGE);
            Bundle bundle=new Bundle();
            bundle.putString(Constants.KEY,"我是来自客户端的信息");
            message.setData(bundle);
            
            //新增，这是重点
            message.replyTo=mReplyMessenger;
            try{
                mService.send(message);
            }catch (Exception e){
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };
```

通过上述修改，我们再运行程序，然后看一下log，很显然，客户端收到了服务端的回复。请看截图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-aa78c72577ab055c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

结果截图.png

### (六)、总结

> 通过上面的例子可以看出，在Messenger中进行数据传递必须将数据放入Message中，而Messenger和Message都实现了Parcelable接口，因此可以跨进程传输。简单来说，Message中所支持的数据类型就是Messenger所支持的传输类型。实际上，通过Messenger来传输Message，Message中只能使用的载体只有what、arg1、arg2、Bunder以及replyTo。Messag中的另一个字段object在同一个进程中是很实用的，但是在进程间通信的时候，在Android2.2以前object字段不支持跨进程传输。即便是2.2以后，也仅仅是系统提供的实现了Parcelable接口的对象才能通过它来传输。这就意味着我们自定义的Parcelable对象是无法通过object字段来传输的，不过所幸我们还有Bundle，Bundle中可以支持大量的数据类型。

下面给出一张Messenger的工作原理图，以便大家更好地理解Meseenger.



![img](https:////upload-images.jianshu.io/upload_images/5713484-98d6b30a9a2e1c84.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Messenger的工作原理.png

## 四、ContentProvider

> ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，从这一点来看，它天生就适合进程间通信。和Messenger一样，ContentProvider的底层同样也是Binder，由此可见Binder在Android系统是何等重要。虽然Content Provider的底层实现是Binder，但是它的使用过程要比AIDL简单许多，这是因为系统已经为我们做了封装，使得我们无须关心底层细节即可轻松实现IPC。ContentProvider虽然使用起来很简单，包括自己创建一个ContentProvider也不是什么难事，尽管如此，它的细节还是很多的。比如CRUD操作等。

系统预置了许多ContentProvider，比如通讯录信息、日程表信息等，要跨进程访问这些信息，只需要通过ContentResolver的query、update、insert和delete方法即可。受篇幅的限制，我这里就不粘贴ContentProvicder的代码了，这块代码网上很多，大家可以自行去搜索

> PS:这里需要注意的是**query、update、insert、delete**四大方法是存在多线程并发访问的，因此方法内部要做好线程同步。而大多数android手机上采用的是SQLite，并且只有一个SQLiteDatabase的时候，要正确应对多线程的情况。因为SQLiteDatabase内部对数据库的操作有同步处理。如果ContentProvider的底层数据是一块内存的话，比如是List，在这种情况下同List遍历、插入、删除操作就需要进行线程同步，否则会一起并发错误。这点要特别注意。

## 五、Socket

**(一) Socket 简述**

> 我们也可以通过socket来实现进程间通信。Socket也称为"套接字"，是网络通信中的概念。它分为流式套接字和用户数据报套接字两种。分别对应网络传输控制层的TCP和UDP协议。TCP协议是面向连接的协议，提供稳定的双向通信功能，TCP连接的建立需要经过"三次握手"才能完成，为了提供稳定的数据传输功能，其本身提供的超时重传机制，因此具有很高的稳定性；而UDP是无连接的，提供不稳定的单向通信功能，当然UDP也可以实现双向通信功能。在性能上，UDP具有更好的效率，其确定啊是不保证数据一定能够正确传输，尤其是在网络拥塞的情况下。关于TCP和UDP的介绍就这么多，更详细的资料请查看相关网络资料。

PS：在使用Socket进行跨进程通信的时候，有两点需要注意

> - 1 首先要在AndroidManifest上声明权限
>    <uses-permission android:name="android.permission.INTERNET" />
>    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
> - 2 其次不能在主线程中访问网络，这回导致在我们在Android 4.0及其以上设备，会抛出异常NetworkOnMainThreadException异常。而且进行网络操作很可能是耗时的，如果放在主线程中，会影响响应效率，这方面来说也不应该在主线程中访问网络。

**(二) 举例说明**

这块的例子很多，大家上网搜一下，推荐这边博客[[Android IPC机制（五）用Socket实现跨进程聊天程序] [Android IPC机制（五）用Socket实现跨进程聊天程序](https://link.jianshu.com?t=http://blog.csdn.net/itachi85/article/details/50667740)

## 六、AIDL：

具体请参考[Android跨进程通信IPC之11——AIDL](https://www.jianshu.com/p/375e3873b1f4)

## 七、使用广播（Broadcast）

> 广播是一种被动跨进程的通讯方式。当某个程序向系统发送广播时，其他的应用程序只能被动的接受广播数据。就像电台进行广播一样，听众只能被动地收听，而不能主动的与电台进行沟通。

BroadcastReceiver本质上是一种系统界别的监听器，他专门监听各种程序发出的Broadcast，因此他拥有自己的进程，只要存在与之匹配的Intent被广播出来，BroadcastReceiver总会被激发。我们知道，只有先注册了某个广播之后，广播接收者才能收到该广播。广播注册的一个行为将是自己感兴趣的IntentFliter注册到Android系统的AMS(ActivityManagerService)中，里面保存了一个IntentFilter列表。广播发送者将自己的IntentFilter的action行为发送到AMS中，然后遍历AMS中的IntentFilter列表，看谁订阅了该广播，然后将消息遍历发送到注册了相应的IntentFilter的Activity或者Service中中，也就是会调用抽象方法onReceive()方法。其中AMS起到了中间桥梁的作用。

程序启动的BroadcastReceiver只需要两步：

> - 1 创建需需启动的BroadcastReceive的Intent
> - 2 调用Context的sendBroadcast()或sendOrderBroadcast()方法来启动指定的BroadcastReceiver。

每当Broadcast事件发生后，系统会创建对应的BroadcastReceiver实例，并自动触发onReceiver()方法。onReceiver()方法执行完后，BroadcastReceiver实例就会被销毁。

> 注意：onReceiver()方法中尽量不要做耗时操作，如果onReceiver()方法不能在10s内完成事件处理，Android会认为该程序无响应，也就弹出我们熟悉的ANR对话框。如果我们需要在接收到广播小猴进行一些耗时的操作，我们可以考虑通过Intent启动一个Server来完成操作，不应该启动一个新的线程来完成操作，因为BroadcastReceiver生命周期很短，可能新建线程还没有执行完，BroadcastReceiver已经销毁了，而如果BroadcastReceiver结束了，它所在的进程中虽然还有启动的新线程执行任务，可是由于该进程中已经没有任何组件，因此系统会在内存紧张的情况下回收该进程，这就导致BroadcastReceiver启动的子线程不能执行完成。

## 八、Binder连接池

> 上面我们介绍了不同的IPC方式，我们知道不同的IPC方式有不同特点和使用场景，这里还是要在说一下AIDL，因为AIDL是一种常见的进程间通信方式，是日常开发中设计进程通信时的首选。

如何使用AIDL在[Android跨进程通信IPC之11——AIDL](https://www.jianshu.com/p/375e3873b1f4)中已经详细介绍了，现在回顾一下大致流程：首先创建一个Service和AIDL接口，接着创建一个类继承自AIDL接口中的Stub类并实现Stub中的抽象方法，在Service的onBinder方法中返回这类的对象，然后客户端就可以绑定服务端的Service，建立连接后，就可以访问远程服务端的方法了。

上面的过程就是典型的AIDL的使用流程。这本来也没什么问题，但是现在考虑一种情况：公司项目越来越大了，现在有10个不同的业务模块都需要使用AIDL来进行进程间通信，那我们该怎么处理？或许有人会说按照AIDL昂视一个一个来吧，如果有100个地方需要用到AIDL，难道就要创建100个Service？大家应该明白随AIDL数量的增加，我们不能无限制的增加Service,Service是四大组件之一，本身就是一种系统资源。而且太多的Service会使得我们的应用看起来很重量级，因此正在运行的Service可以在应用详情页看到，而且让用户看到有10个服务正在运行，也很坑，针对上面的问题，我们需要减少Service的数量，将所有的AIDL放到同一个Service中去管理。

> 在这种某事下，整个工作机制是这样的：每个业务模块都创建自己的AIDL接口并实现此接口，这时候不同的业务模块之间是不能耦合的，所有实现细节我们都要单独来看，然后向服务端提供自己的唯一标示和对应的Binder对象；对于服务端来说，只需要一个Service就可以了，服务端提供一个queryBinder接口，这个接口能够根据业务模块的特征来返回相应的Binder对象给它们，不同的业务模块拿到所需的Bidner对象后就可以进行远程方法调用了。由此可见，Binder连接池的主要作用就是将每个业务模块的Binder请求转发到远程Service中去执行，从而避免了重复创建Service的过程。原理如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-1e233c6c80e94a48.png?imageMogr2/auto-orient/strip|imageView2/2/w/836/format/webp)

Binder连接池.png

上面是理论，也许不是很好理解，现在我们用代码的形式给大家弄一个demo。既然是demo，我们就不弄100个接口了，就弄3个好了，一个是登录的用户模块的IUser，一个是订单模块的IOrder，一个是给交易模块的IPay，其中IUser提供登录，和登出功能，IOrder提供下单和取消订单功能，IPay提供支付和取消支付功能，具体代码如下：

### 1、编写相应的AIDL文件

编写 IOrder.aidl文件



```csharp
interface IOrder {
    void doOrder(long orderId);
    void cancelOrder(long orderId);
}
```

编写IPay.aidl文件



```csharp
interface IPay {
    void doPay(long payId,int amount);
    void cancelPay(long payId);
}
```

编写IUser.aidl文件



```dart
interface IUser {


    void login(String userName,String passwork);
    void logout(String username);
}
```

### 2、编写具体的的AIDL中Stub的具体实现类

当然在具体写实现类的时候，别忘记同步项目，否则你是找不到相应的Stub类的

**(1) 编写IPayImpl.java类**



```java
/**
 * Created by gebilaolitou on 2017/8/26.
 * 支付模块的实现
 */

public class IPayImpl extends IPay.Stub  {
    @Override
    public void doPay(long payId, int amount) throws RemoteException {
        Log.i("GEBILAOLITOU","IPayImpl  doPay, payId="+payId+",amount="+amount);
    }

    @Override
    public void cancelPay(long payId) throws RemoteException {
        Log.i("GEBILAOLITOU","IPayImpl  doPay, payId="+payId);
    }
}
```

**(2) 编写IUserImpl.java类**



```java
/**
 * Created by gebilaolitou on 2017/8/26.
 * 用户模块的实现
 */

public class IUserImpl extends IUser.Stub {


    @Override
    public void login(String userName, String password) throws RemoteException {
        Log.i("GEBILAOLITOU","IUserImpl  login, userName="+userName+",password="+password);
    }

    @Override
    public void logout(String username) throws RemoteException {
        Log.i("GEBILAOLITOU","IUserImpl  logout, userName="+username);
    }
}
```

**(3) 编写IUserImpl.java类**



```java
/**
 * Created by gebilaolitou on 2017/8/26.
 * 订单模块的实现
 */

public class IOrderImpl extends IOrder.Stub {
    @Override
    public void doOrder(long orderId) throws RemoteException {
        Log.i("GEBILAOLITOU","IOrderImpl  doOrder, oderId="+orderId);
    }

    @Override
    public void cancelOrder(long orderId) throws RemoteException {
        Log.i("GEBILAOLITOU","IOrderImpl  cancelOrder, oderId="+orderId);
    }
}
```

### 3、定义IBinderPool接口并实现之。

**(1) 编写IBinderPool.aidl**



```csharp
interface IBinderPool {
    IBinder queryBinder(int binderCode);
}
```

**(2) 编写IBinderPool的具体实现类**



```java
/**
 * Created by gebilaolitou on 2017/8/26.
 * 连接池
 */

public class BinderPoolImpl extends IBinderPool.Stub {

    /**
     * query  方法就是根据请求功能模块的代码，返回相应模块的实现
     * @param binderCode
     * @return
     * @throws RemoteException
     */
    @Override
    public IBinder queryBinder(int binderCode) throws RemoteException {
        Log.d("GEBILAOLITOU", "BinderPoolImpl queryBinder  binderCode="+binderCode);
        switch (binderCode){
            case Constans.TYPE_ORDER:
                return new IOrderImpl();
            case Constans.TYPE_PAY:
                return new IPayImpl();
            case Constans.TYPE_USER:
                return new IUserImpl();
        }
        return null;
    }
}
```

### 4、编写相应的Service。



```java
public class BinderPoolService extends Service {

    private BinderPoolImpl mBinderPoolImp = new BinderPoolImpl();
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinderPoolImp;
    }
}
```

在AndroidManifest.xml作相应的配置



```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.gebilaolitou.binderpooldemo">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <service android:name=".BinderPoolService"
            android:process=".binderpoolservice"
            />
    </application>

</manifest>
```

### 5、客户端的实现

**(1)实现一个BinderPool的管理器**

为了操作方便，将所有的BinderPool操作放在同一个类里面。这个类的大体结构和标准的AIDL客户端基本相同。首先要有一个IBinderPool类型的私有变量，然后定义一个ServiceConnection类型的私有变量，并在它的onServiceConnected 方法里面给IBinderPool类型的变量赋值，然后定义一个bindBinderPoolService函数，在里面bindService。完整的代码如下：



```java
public class BinderPoolMananger {


    private Context mContext;
    private static BinderPoolMananger sInstance;
    private IBinderPool mBinderPool;
    private CountDownLatch mCountDownLatch;

    private BinderPoolMananger(Context context){
        this.mContext = context;
        bindBinderPoolService();
    }

    public static BinderPoolMananger getInstance(Context context){
        if(sInstance == null){
            sInstance = new BinderPoolMananger(context);
        }
        return sInstance;
    }


    private ServiceConnection mServiceConnection = new ServiceConnection(){
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d("GEBILAOLITOU", "onServiceConnected() 被调用了");
            mBinderPool = IBinderPool.Stub.asInterface(service);
            try {
                //死亡通知
                mBinderPool.asBinder().linkToDeath(new IBinder.DeathRecipient() {
                    @Override
                    public void binderDied() {
                        mBinderPool.asBinder().unlinkToDeath(this, 0);
                        mBinderPool = null;
                        bindBinderPoolService();
                    }
                }, 0);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            mCountDownLatch.countDown();
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    private void bindBinderPoolService(){
        Log.d("GEBILAOLITOU", "bindService()  被调用");
        mCountDownLatch = new CountDownLatch(1);
        Intent intent = new Intent(mContext,BinderPoolService.class);
        Log.d("GEBILAOLITOU", "开始 bind service");
        mContext.bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
        try {
            mCountDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Log.d("GEBILAOLITOU", "finish to bind service");
    }


    public IBinder query(int code){
        Log.d("GEBILAOLITOU", "query() 被调用, code = " + code);
        IBinder binder = null;
        try {
            binder = mBinderPool.queryBinder(code);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return binder;
    }
}
```

**(2) 调用BinderPool管理器**



```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread(new Runnable() {
            @Override
            public void run() {
                doBinder();
            }
        }).start();
    }

    private void doBinder() {
        BinderPoolMananger bm = BinderPoolMananger.getInstance(MainActivity.this);
        
        IOrder order = IOrder.Stub.asInterface(bm.query(Constans.TYPE_ORDER));
        try {
             order.doOrder(123L);
             order.cancelOrder(123L);
        } catch (RemoteException e) {
            e.printStackTrace();
            Log.d("GEBILAOLITOU", "order 模块出现异常，e="+e.getMessage());
        }

        IPay pay = IPay.Stub.asInterface(bm.query(Constans.TYPE_PAY));

        try {
            pay.doPay(123L,1000);
            pay.cancelPay(123L);
        } catch (RemoteException e) {
            e.printStackTrace();
            Log.d("GEBILAOLITOU", "pay 模块出现异常，e="+e.getMessage());
        }


        IUser user = IUser.Stub.asInterface(bm.query(Constans.TYPE_USER));

        try {
            user.login("gebilaolitou","123456");
            user.logout("gebilaolitou");
        } catch (RemoteException e) {
            e.printStackTrace();
            Log.d("GEBILAOLITOU", "user 模块出现异常，e="+e.getMessage());
        }
    }
}
```

### 6、其他代码



```java
public class Constans {

    public static final int TYPE_USER=1;  //用户模块
    public static final int TYPE_PAY=2;   //支付模块
    public static final int TYPE_ORDER=3;  //订单模块
}
```

### 7、项目结构

如下图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-aa69be7a5979942d.png?imageMogr2/auto-orient/strip|imageView2/2/w/604/format/webp)

项目结构.png

### 8、结果输出

![img](https:////upload-images.jianshu.io/upload_images/5713484-f72e52e3637cf810.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

结果输出.png

## 九、选择合适的IPC方式

receiver其实本质是用的Bundle来实现的，Binder连接池其实本质是AIDL，所以我们大概可以分为 Bundle、AIDL、Messenger、ContentProvider、Socket。

![img](https:////upload-images.jianshu.io/upload_images/5713484-e729c41516a8cca1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

