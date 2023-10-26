# Android跨进程通信IPC之Linux基础

由于Android系统是基于Linux系统的，所以有必要简单的介绍下Linux的跨进程通信，对大家后续了解Android的跨进程通信是有帮助的，本篇的主要内容如下：

> - 1、Linux介绍
> 1.1、Unix操作系统
> 1.2、GNU
> 1.3、Linux的诞生
> 1.4、开源发展实验室和Linux基金
> 1.5、Linux的全局图
> 1.6、Linux的源码目录结构
> - 2、内核态与用户态
> 2.1、内核态与用户态简介
> 3.2、为什么要有用户态和内核态
> 2.3、用户态与内核态的切换
> - 3、红黑树
> 3.1、二叉搜索树
> 3.2、红黑树
> 3.3、数据结构设计
> 3.4、树的旋转知识
> - 4、Linux的跨进程通信
>   4.1、匿名管道(pipe)
>   4.2、命名管道(FIFO)
>   4.3、信号(signal)
>   4.4、信号量(semaphore)
>   4.5、消息队列(message queue)
>   4.6、共享内存(share memory)
>   4.7、套接字(Socket)
>   4.8、Linux的几种跨进程通信的方式的比较的旋转知识

## 一、Linux介绍

说到Linux操作系统，不的不说下Unix系统

### (一)、Unix操作系统

Unix因为其安全可靠，高效强大的特点在服务器领域得到了广发的应用。直到GNU/Linux流行开始前，Unix也是科学计算、大型机、超级电脑等所用操作系统的主流。

**1、Unix的诞生**

> 汤普逊和里奇最早是在贝尔实验室开发Unix，此后10年，Unix在学术机构和大型企业中得到了广泛的应用，当Unix拥有者AT&T公司以低廉甚至免费的许可将Unix源码授权给学术机构做研究或教学之用，许多机构在此源码的基础加以扩展和优化，形成了所谓的"Unix变种"，这种变种反过来也促进了Unix的发展，其中最著名的变种就是BSD产品。BSD在后续的发展中也衍生出了3个主要分支:FresBSD、OpenBSD和NetBSD。

**2、Unix名字的由来**

> Unix的诞生和Multics(Multiplexed Information android Computing System) 是有一定渊源的。Multics是由麻省理工学院，AT&T贝尔实验室和通用电气合作进行的操作系统项目。由于Multics的失败，AT&T撤出了Multics投入的资源，其中一位开发者——肯.汤普逊则继续开发软件，并最终编写一个太空旅行游戏，但是他发现游戏速度很慢，并且成本高，在丹尼斯.里奇的帮助下，汤普逊用PDP-7的汇编语言重写了这个游戏，并让其再DEC PDP-7上运行起来。这次经历加上Multics项目的经验，汤普逊和里奇领导一组开发者开发了一个新的多任务操作系统，这个系统包括命令解释器和一些实用程序。1970年那部PDP-7却只能支持两个用户，所以他们就开玩笑的称他们的系统其实是"UNiplexed Information and Computing System"，缩写为"UNICS"，于是这个项目被称为UnICS(Uniplexed Information and Computing System)。后来，大家取其谐音这个名字就改为UNIX

**3、Unix的文化**

> UNIX is not jus an operating system , but a way of life. (UNIX 不仅仅是一个操作系统，更是一种生活方式。) 经过几十年的发展，UNIX在技术上日臻成熟的过程中，他独特的设计哲学和美学也深深地吸引了一大批技术人员，他们在维护、开发、使用UNIX的同时，UNIX也影响了他们的思考方式和看待世界的角度。这些人自然而然地形成了一个社团

**4、Unix的重要设计原则：**

> - 简洁至上
> - 提供机制而非策略

### (二)、GNU

GNU又称革奴计划，是由查理德.斯托曼在1983年9月27日发起的，它的目标是创建一套完全自由的操作系统。
 《GNU宣言》是解释为什么发起该计划的文章，其中一个重要理由就是要"重现当年软件界合作互助的团结精神"。为了保证GNU软件可以自由地"使用、复制、修改和发布"，所有GNU软件都有一份在禁止其他人添加任何限制的情况下授权所所有权利了给任何人的协议条款，GNU通用的公共许可证(GNU General Public License GPL)。即"反版权"(或者称copyleft)概念。

> GNU是“GNU is Not Unix”的缩写。斯托曼宣布GNU应当发音为Guh-NOO以避免与new这个单词混淆（注：Gnu在英文中原意为非洲牛羚，发音与new相同）。UNIX是一种广泛使用的商业操作系统的名称。由于GNU将要实现UNIX系统的接口标准，因此GNU计划可以分别开发不同的操作系统部件。GNU计划采用了部分当时已经可自由使用的软件。不过GNU计划也开发了大批其他的自由软件。

### (三)、Linux的诞生

**1、Linux的诞生**

> 1991年，在赫尔辛基，[Linus Toral](https://link.jianshu.com?t=http://baike.baidu.com/item/林纳斯·托瓦兹)开始写了一个项目，目的是用来访问大学里面的大型Unix服务器的虚拟终端。他专门写了一个用于他当时正在用的硬件的，与系统操作系统无关的程序，开发是在Minix，用的编译器是GCC来完成的，这个项目后面逐渐转变为Linux内核。

**2、Linux的诞生**

> Linus Torvalds本要把他的发时叫做Freax——“fread”，“free”和“x”（暗指Unix）的合成词。在开发系统的前半年里，他把文件以文件名“Freax”存储。Torvalds考虑过Linux这个名字，但是因为觉得它过于自我本位而放弃了使用它。为便于开发，后面，他把那些文件上传到了赫尔辛基工业大学（HUT）的FTP服务器。Torvalds在HUT负责管理那个服务器的同事，觉得“Freax”这个名字不是很好，就在不咨询Torvalds的情况下，把项目的名字改成了“Linux”。但是之后，Torvalds也同意“Linux”这个名字了：“经过多次讨论，他承认Linux这个名字更好。在0.01版本Linux的源代码的makefile里仍然使用‘Freax’这个名字，在之后‘Linux’这个名字才被使用。所以，Linux这个名字并不是预先想好的，只是它被广泛接受了而已”。

### (四)、开源发展实验室和Linux基金

开源码发展实验室（Open Source Development Lab）创立于2000年。它是一个独立的非营利性组织。它的目标是优化Linux以应用于数据中心和运营商的领域。它是Linus Torvalds和Andrew Morton工作的赞助来源。2006年年中，Morton去了Google（Google也是使用Linux内核的）；Torvalds全职为OSDL开发Linux内核。非商业性运营机制的资金主要来源于Red Hat，Novell，三菱，英特尔, IBM ，戴尔和惠普等几家大公司。

2007年1月22日，OSDL和自由标准组织合并为Linux基金会，把它们的工作焦点集中在改进GNU/Linux以与Windows竞争。

### (五)、Linux的全局图

下面是一幅Linux kernel map：



![img](https:////upload-images.jianshu.io/upload_images/5713484-64910e3e959d8bbb.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Linux kernel map.png



这是makeLinux网站提供的一幅非常经典的Linux内核图，涵盖了内核最为核心的方法，Linux除了驱动开发外，还有很多通用子系统，比如CPU，memory，file system等核心模块，即便不做底层驱动开发，掌握这些模块对于加深理解整个系统运转还是很有帮助的。

### (六)、Linux的源码目录结构

| 目录          | 解释                      | 部分子子目录                  |
| ------------- | ------------------------- | ----------------------------- |
| kernel        | 内核管理相关，进程调度等  | sched/fork等                  |
| fs            | 文件子系统                | ext4/f2fs/fuse/debugfs/proc等 |
| mm            | 内存子系统                |                               |
| drivers       | 设备驱动                  | staging/cpufreq/gpu 等        |
| arch          | 所有CPU系统结构相关的代码 | arm/x86等                     |
| include       | 头文件                    | linux/uapi/asm_generic等      |
| lib           | 标准通用的C库             |                               |
| ipc           | 进程通信相关              |                               |
| init          | 初始化过程(非系统引导)    |                               |
| block         | 块设备驱动程序            |                               |
| crypto        | 加密、解密、校验算法      |                               |
| Documentation | 说明文档                  |                               |

## 二、内核态与用户态

这部分是临时加进来的，是在后面的Binder驱动里面会用到，原来是打算加到"Android跨进程通信IPC之1——Linux基础"里面，不过由于简书的篇幅限制，我加到这里来了。

### 1、内核态和用户态的简介

> - **内核态**：CPU可以访问内存所有数据，包括外围设备，例如硬盘、网卡，CPU可以将自己从一个程序切换到另外一个程序。
> - **用户态**:   只能受限的访问内存，且不允许访问外围设备，占用CPU的能力被剥削，CPU资源可以被其他程序获取。

### 2、为什么要有用户态和内核态

> 由于需要限制不同的程序之间的访问能力，防止他们获取别的程序的内存数据，或者获取外围设备的数据，并发送网络，CPU划分出两个权限等级  ----用户态  和  内核态。

### 3、用户态与内核态的切换

**3.1 切换简介**

所有用户程序都是运行在用户态的，但是有时候程序确实需要做一些内核态的事情，例如从硬盘读取数据，或者从键盘获取输入等。而唯一可以这这些事情的就是 **操作系统** ，所以这时候 **程序** 就需要先向 **操作系统** 请求，以 **程序** 的名字来执行这些操作。这时候就需要一个这样的机制：用户态 切换到 内核态，但是不能控制内核态中执行的执行这种机制叫做** 系统调用 **，在CPU中的实现称之为 "陷阱指令(Trap Instruction)"

**3.2 系统调用机制流程：**

> - 1、用户态程序将一些数据值放在寄存器中，或者使用参数创建一个堆栈(stack frame)，以表明需要操作系统提供的服务。
> - 2、用户态程序执行陷阱指令
> - 3、CUP切换到内核态，并跳到内存指定位置的指令，这些指令是操作系统的一部分，他们具有内存保护，不可被用户态程序访问
> - 4、这些指令称之为 陷阱 (trap) 或者新系统调用处理器 ( system call hanlder )。他们会读取程序放入内存的数据参数，并执行程序请求的服务。
> - 5、系统调用完成后，操作系统会重置CPU为用户态并返回系统调用的结果。

## 三、红黑树

> 红黑树是60年代中期计算机科学界寻找一种算法复杂度稳定，容易实现的数据存储算法的产物。红黑树是常用的一种数据结构，它使得对数据的索引，插入和删除操作都能保持在O(lgn)的时间复杂度。在优先级队列、字典等实用领域都有广泛地应用，更是70年代提出的关系数据库模型——B树的鼻祖。当然，相比于一般的数据结构，红黑树实现的难度有所增加。** 关键是后面要讲解的Binder驱动里面用到了红黑树 **

### (一)二叉搜索树

> 在具体实现红黑树之前， 必须弄清楚它的基本含义。红黑树本质上是一颗二叉搜索树，它满足二叉搜索树的基本性质——即树中的任何节点的值大于它的左子节点，且小于它的右子节点。

![img](https:////upload-images.jianshu.io/upload_images/5713484-169f0fef8703f997.png?imageMogr2/auto-orient/strip|imageView2/2/w/798/format/webp)

二叉搜索树.png

按照二叉搜索树组织数据，使得对元素的查找非常快捷。比如上图的中的二叉搜索树，如果查询值为48的节点，只需要遍历4个节点即可完成。理论上，一颗平衡的二叉树搜索树的任意节点平均查找效率为树的高度h，即O(lgn)。但是如果二叉搜索树的失去平衡(元素在一侧)，搜索效率就退化为O(n)，因此二叉搜索树的平衡是搜索效率的关键所在。为了维护树的平衡性，数据结构内出现了各种各样的树，比如AVL树通过维持任何节点的左右子树的高度差 不大于1保持树的平衡，而红黑树使用颜色维持树的平衡，使二叉搜索树的左右子树的高度差 保持在固定的范围。相比于其他二叉搜索树，红黑树对二叉搜索树的平衡性维持着自身的优势

### (二) 红黑树

红黑树，顾名思义，红黑树的节点是有颜色概念的，即非红即黑，通过颜色的语速，红黑树为支持着二叉搜索树的平衡性。一个红黑树必须有下面5个特征

> - 1、节点是红色或黑色
> - 2、根是黑色
> - 3、所有叶子是黑色(叶子是NIL节点)
> - 4、每个红色节点的两个子节点都是黑色的(从每个叶子到跟的所有路径不能有两个连续的红色节点)
> - 5、从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-5ae2df1a02353724.png?imageMogr2/auto-orient/strip|imageView2/2/w/450/format/webp)

红黑树图1.png

这些特征强制约束了红黑树的关键性质：从跟到叶子的最长可能路径不多于最短可能路径的两倍长(特征4保证了路径最长的情况为1红1黑，最短的情况为全黑，再结合特征5，可以推导出)。结果是这个树大致上是平衡的。因为比如插入、删除和查找操作中，操作某个值的最坏情况的时间都要求与树的高度成比例，这个高度上的理论上限允许红黑树在最坏的情况都是高效的，而不同于普通的(二叉搜索树)

### (三) 数据结构设计

> - 和一般的数据结构设计类似，我们用抽象数据类型表示红黑树的节点，使用指针保存节点之间的相互关系。
> - 作为红黑树的节点，其基本属性有：节点的颜色，左节点的指针，右节点的指针、父节点的指针、节点的值

如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-56df9358ea73b7b0.png?imageMogr2/auto-orient/strip|imageView2/2/w/709/format/webp)

红河书节点.png

为了方便红黑树关键算法的实现，还定义了一些简单的操作（都是内联函数）。



```cpp
//红黑树节点
template<class T>
class rb_tree_node
{
    typedef rb_tree_node_color node_color;
    typedef rb_tree_node<T> node_type;
public:
    node_color color;//颜色
    node_type*parent;//父节点
    node_type*left;//左子节点
    node_type*right;//右子节点
    T value;//值
    rb_tree_node(T&v);//构造函数
    inline node_type*brother();//获取兄弟节点
    inline bool on_left();//自身是左子节点
    inline bool on_right();//自身是右子节点
    inline void set_left(node_type*node);//设置左子节点
    inline void set_right(node_type*node);//设置左子节点
};
```

为了表示红黑树节点的颜色，我们定义了一个简单的枚举类型。



```rust
//红黑树节点颜色
enum rb_tree_node_color
{
    red=false,
    black=true
};
```

有了节点，剩下的就是实现红黑树的构造、插入、搜索、删除等关键算法了。



```cpp
//红黑树
template<class T>
class rb_tree
{
public:
    typedef rb_tree_node<T> node_type;
    rb_tree();
    ~rb_tree();
    void clear();
    void insert(T v);//添加节点
    bool insert_unique(T v);//添加唯一节点
    node_type* find(T v);//查询节点
    bool  remove(T v);//删除节点
    inline node_type* maximum();//最大值
    inline node_type* minimum();//最小值
    inline node_type* next(node_type*node);//下一个节点
    inline node_type* prev(node_type*node);//上一个节点
    void print();//输出
    int height();//高度
    unsigned count();//节点数
    bool validate();//验证
    unsigned get_rotate_times();//获取旋转次数
private:
    node_type*root;//树根
    unsigned rotate_times;//旋转的次数
    unsigned node_count;//节点数
    void __clear(node_type*sub_root);//清除函数
    void __insert(node_type*&sub_root,node_type*parent,node_type*node);//内部节点插入函数
    node_type* __find(node_type*sub_root,T v);//查询
    inline node_type* __maximum(node_type*sub_root);//最大值
    inline node_type* __minimum(node_type*sub_root);//最小值
    void __rebalance(node_type*node);//新插入节点调整平衡
    void __fix(node_type*node,node_type*parent,bool direct);//删除节点调整平衡
    void __rotate(node_type*node);//自动判断类型旋转
    void __rotate_left(node_type*node);//左旋转    
    void __rotate_right(node_type*node);//右旋转
    void __print(node_type*sub_root);//输出
    int  __height(node_type*&sub_root);//高度
    bool __validate(node_type*&sub_root,int& count);//验证红黑树的合法性
};
```

> 在红黑树类中，定义了** 树根(root) ** 和** 节点数 (count) **，其中还记录了红黑树插入、删除时执行的旋转次数 rotate_times。其中核心操作有** 插入操作(insert) **，** 搜索操作 (find) **， \** 删除操作(remove) \**，\** 递减操作(prev) \** ——寻找比当前节点较小的节点，** 递增操作(next) ** ——寻找比当前节点比较大的节点， ** 最大值(maximum) ** 和 ** 最小值(minimum) ** 。** 其中验证操作(__ validate) ** 通过递归操作红黑树，验证红黑树的基本颜色约束，用于操纵红黑树验证红黑树是否保持平衡。

### (四) 树的旋转知识

当我们在对红黑树进行插入和删除等操作时，对树做了修改，那么可能会违背红黑树的性质。所以为了继续保持红黑树的性质，我们可以通过对节点进行重新着色，以及对树进行相关的旋转操作，即修改树中某些节点的颜色及指针结构，来达到对红黑树进行插入和删除结点等操作后，继续保持它的性质或平衡。

树的旋转，分为左旋和右旋，借助下图来做介绍：

**1、左旋**

![img](https:////upload-images.jianshu.io/upload_images/5713484-87720bc1badcb933.png?imageMogr2/auto-orient/strip|imageView2/2/w/564/format/webp)

左旋.png

如上图所示：

> - 当在某个结点pivot上，做左旋操作时，我们假设它的右孩子不是NIL，pivot可以为任何不是不是NIL的左孩子的结点。
> - 左旋以pivot到y之间的链为"支轴"进行，它使y成为该孩子树新的根，而y的左孩子b则成为pivot的右孩子。

**2、右旋**

![img](https:////upload-images.jianshu.io/upload_images/5713484-6e4e06618bf4d200.png?imageMogr2/auto-orient/strip|imageView2/2/w/571/format/webp)

右旋.png

对于树的旋转，能保持不变的只有原树和搜索性质，而原树的红黑性质则不能保持，在红黑树的数据插入和删除后可利用旋转和颜色重涂来恢复树的红黑性质。

这样大家对红黑树就有了初步了解。这里就不详细介绍了，如果大家有兴趣，可以自行去了解。

## 四、Linux的跨进程通信(IPC)概述

**(一)、跨进程通信(IPC)的目的**

跨进程通信(IPC)的目的主要如下：

> - 数据传递
>    一个进程需要将它的数据发送给另外一个进程，发送的数据量在一个字节到几M字节之间
> - 共享数据
>    多个进程想要操作共享数据。
> - 通知事件
>    一个进程需要向另一个或一组进程发送消息，通知它(它们)发生了某种事件(如进程终止时要通知父进程)
> - 资源共享
>    多个进程之间共享资源。为了做到这一点，需要内核提供锁和同步机制
> - 进程控制
>    有些进程希望完全控制另一个进程的执行(如debug进程)，此时控制进程希望能够拦截另一个进程的所有步骤和异常，并能够及时知道它的状态改变。

**(二)、Linux 进程间通信(IPC)的发展**

> **Linux 下的跨进程通信手段基本上是从Unix平台上的进程通信手段继承而来。而对Unix发展做出大量贡献的量大主力AT&T的贝尔实验室及BSD(加州大学伯克利分校伯克利软件发布中心)在进程间通信方面的侧重点有所不同。前者对Unix早期的进程间通信手段进行了系统的改进和扩充，形成了"system V IPC"，通信进程局限在单个计算机内；而后者则跳过了这个限制，形成了基于套接字(socket)的进程间通信机制。
>  Linux 则把两者继承了下来**。

所以可以把Linux中的进程间通信大体分为4类

> - 基于早期Unix的进程间通信：管道和信号
> - 基于System V的进程间通信：System V消息队列、System V 信号灯、System V 共享内存
> - 基于Socket 的进程间通信：socket
> - POSIX进程间通信：posix 消息队列、posix信号灯、posix共享内存

这里说下[PSOIX](https://link.jianshu.com?t=http://baike.baidu.com/item/POSIX):

> 由于Unix版本的多样性，电子电器工程协会(IEEE) 开发了一个独立的Unix标准，这个心的ANSI Unix标准被称为计算机环境的可移植性操作系统。现有的大部分Unix和流行版本都是遵循POSIX标准的，而Linux从一开始就是遵循POSIX标准。

## 五、Linux的跨进程通信详解

在Linux下进程通信有以下七种：

> - 1、匿名管道(pipe)
> - 2、命名管道(FIFO)
> - 3、信号(signal)
> - 4、信号量(semaphore)
> - 5、消息队列(message queue)
> - 6、共享内存(share memory)
> - 7、套接字(Socket)

那我们就来详细的了解下相关的内容

### (一)、匿名管道(pipe)

**1、什么是匿名管道？**

匿名管道(pipe)是Linux支持的最初Unix IPC形式之一，具有以下特点：

> - 匿名管道是半双工的，数据只能向一个方向流动；需要双方通信时，需要建立两个管道；
> - 只能作用于父子进程或者兄弟进程之间(具有亲缘关系的进程)
> - 单独构成的一种独立的文件系统：匿名管道对于管道两端的进程而言，就是一个文件，但它不是普通文件，它不属于某种文件系统，而是自理门户，单独构成一种文件系统，去并且只存在于内存中。
> - 数据的读出和写入：一个进程向管道中写的内容被管道另一端的进程读出。写入的内容每次都添加在管道缓冲区的末尾，并且每次都是从缓存区的头部读出数据。

**2、匿名管道的实现机制**

匿名管道是右内核管理的一个缓冲区，相当于我们放入内存的中一个纸条。匿名管道的一端连接一个进程的输出。这个进程会向管道中放入信息。匿名管道的另一端连接一个进程的输入，这个进程取出被放入管道的信息。一个缓存区不需要很大，它被设计成为唤醒的数据结构，以便管道可以被循环利用。当管道中没有信息的话，从管道中读取的进程会等待，直到另一端的进程放入信息。当管道被放满信息的时候，尝试放入信息的进程就会等待，直到另一端的进程取出信息。两个进程都终结的时候，管道也会自动消失。如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-1d562f8269b81345.png?imageMogr2/auto-orient/strip|imageView2/2/w/620/format/webp)

image.png

从原理上，匿名管道利用fork机制建立，从而让两个进程可以连接到同一个PIPE上。最开始的时候，上面的两个箭头都连接到同一个进程Process 1上(连接在Process 1上的两个箭头)。当fork复制进程的时候，会将这两个连接也复制到新的进程(Process 2)。随后，每个进程关闭在自己不需要的一个连接(两个黑色的箭头被关闭；Process 1关闭从PIPE来的输入连接，Process 2关闭输出到PIPE的连接)，这样，剩下的红色连接就构成了上图的PIPE。
 示例图如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-cfdbad83904e2279.png?imageMogr2/auto-orient/strip|imageView2/2/w/636/format/webp)

image.png

**3、匿名管道实现细节**

在Linux中，匿名管道的实现并没有使用专门的数据结构，而是借助了文件系统的file结构，和VFS的索引节点inode。通过将两个file结构指向同一个临时的VFS节点，而这个VFS索引节点又指向了一个物理页面而实现的。如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-7d91e2fab54d6c95.png?imageMogr2/auto-orient/strip|imageView2/2/w/470/format/webp)

管道实现细节.png

**4、关于匿名管道的读写**

匿名管道的实现的源代码在fs/pipe.c中，在pipe.c中有很多函数，其中有两个函数比较重要，即匿名管道pipe_read()读函数和匿名管道写函数pipe_write()。匿名管道写函数通过将字节复制到VFS索引节点指向物理内存而写入数据，而匿名管道读函数则通过复制物理内存而读出数据。当然，内核必须利用一定的 ***同步机制\*** 对管道的访问，为此内核使用了 ***锁\*** 、***等待队列\***、和 ***信号\***。

当写入进程向匿名管道中写入时，它利用标准的库函数write()，系统根据库函数传递的文件描述符，可找到该文件的file结构。file结构中制定了用来进行写操作的函数(即写入函数)地址，于是，内核调用该函数完成写操作。写入函数在向内存中写入数据之前，必须首先检查VFS索引节点中的信息，同时满足如下条件时，才能进行实际的内存复制工作：

> - 内存中有足够的空间可以容纳所有要写入的数据。
> - 内存没有被读程序锁定。

如果同时满足上述条件，写入函数首先会锁定内存，然后从写进程的地址空间中复制数据到内存。否则，写进程就休眠在VFS索引节点的等待队列中，接下来，内核将调用调度程序，而调度程序会选择其他进程运行。写进程实际处于可中断的等待状态，当内存中有足够的空间可以容纳写入数据，或内存被解锁时，读取进程会唤醒写入进程，这时，写入进程将接受到信号。当数据写入内存之后，内存被解锁，而所有休眠在索引节点的读取进程会被唤醒。

匿名管道的读取过程和写入过程类型。但是，进程可以在没有数据或者内存被锁定时立即返回错误信息，而不是阻塞该进程，这一来于文件或管道的打开模式。反之，进程可以休眠在索引节点的等待队列中等待写入进程写入数据。当所有的进程完成了管道操作之后，管道的索引节点被丢弃，而共享数据页被释放。

PS:有些同学可能不清楚VFS，我这里就简单介绍下；

> VFS(virtual File System/虚拟文件系统):是Linux文件系统对外的接口。任何要使用文件系统的程序都必须经由这层接口来使用它。它是采用标准的Unix系统调用读写位于不同物理介质上的不同文件系统。VFS是一个可以让open()、read()、write()等系统调用不用关系底层的存储介质和文件系统类型就可以工作的粘合层。在Linux中，VFS采用的是面向对象的编程方法。

### (二)、命名管道(FIFO/named PIPE)

在上面，我们介绍了匿名管道(pipe)，我们知道了如何匿名管道在进程之间传递数据，同时也是看到了这个方式的一个缺陷，就是这些就进程都是由一个共同的祖先进程启动，这给我们在不相关的进程之间交换数据带来了不方便。这里我将会介绍另一种通信方式——命名管道，来解决不相关进程之间的问题。

**1、什么是命名管道?**

> - 命名管道也被称为FIFO或者named pipe，它是一种特殊类型的文件，它在文件系统中以文件名的形式存在，但是它的行为却和之前所讲的匿名管道类似。
> - FIFO(First in, First out) 为一种特殊的文件类型，它在文件系统中有对应的路径。当一个进程以读(r)的方式打开该文件，而另一个进程以写(w)的方式打开该文件，那么内核就会在两个进程之间建立管道，所以FIFO实际上也由内核管理，不与硬盘打交道。之所以叫FIFO，因为管道本质上是一个**先进先出的队列数据结构 ，最早放入的数据被最先读出来，从而保证信息交流的顺序。FIFO只是借用了文件系统(file system，命名管道是一种特殊类型的文件，因为Linux中所有事物都是文件，它是在文件系统中以文件名的形式存在。)来为管道命名。写模式的进程向FIFO中写入，而读模式的进程从FIFO文件中读出。当删除FIFO文件时，管道连接也随之小时。FIFO的好处在于我们可以通过 文件的路径来识别管道，从而让没有亲缘关系的进程之间建立连接**。

**2、命名管道的读写规则：**

> - 1、从FIFO中读取数据的约定：如果一个进程为了从FIFO中读取数据而阻塞打开了FIFO，那么该进程内的读操作 为设置了阻塞标志的读操作。
> - 2、从FIFO中写入数据的约定：如果一个进程为了想FIFO中写入数据而阻塞打开了FIFO，那么该进程内的写操作 为设置了阻塞标志的写操作。

**3、命名管道的安全问题：**

> 大家想一下，只使用一个FIFO文件，如果有多个进程同时向同一个FIFO文件写数据，而只有一个读FIFO进程在同一个FIFO文件读取数据时，会发生怎么样的情况呢，会发生数据块的相互交错是很正常的？而且个人认为多个不同进程向一个FIFO读取进程发送数据是很正常的情况。

为了解决这个问题，就是让写操作原子化，怎么才能使写操作原子化呢？答案其实很简单：系统规定，在一个以O_WRONLY(即阻塞方式)打开的FIFO中，如果写入的数据长度小于等于PIPE_BUF，那么或者写入全部字节，或者一个字节都不写入。如果所有的写请求都是发往一个阻塞的FIFO的，并且每个写请求的数据长度小于等于PIPE_BUF字节，系统就可以确保数据绝不会交错在一起。

### (三)、信号(Signal)

**1、什么是信号？**

> 信号是比较复杂的通信方式，用于通知接受进程有某种事件发生，除了用于进程间通信外，进程还可以发送信号给进程本身；Linux除了支持Unix早期信号语义函数sigal外，还支持语义服务Posix.1标准的信号函数sigaction(实际上，该函数是基于BSD的，BSD为了实现可靠信号机制，有能够统一对外接口，用sigaction函数重新实现了signal函数)

**2、信号的种类**

如下图:



![img](https:////upload-images.jianshu.io/upload_images/5713484-b703c7840590eaeb.png?imageMogr2/auto-orient/strip|imageView2/2/w/506/format/webp)

信号种类.png



每种信号类型都有对应的信号处理程序(也叫信号的操作)，就好像每个中断都有一个中断服务例程一样。大多数信号的默认操作是结束接受信号的进程；然而一个进程通常可以请求系统采取某些代替的操作，各种代替操作是：

> - 忽略信号。随着这一选项的设置，进程将忽略信号的出现。有两个信号不可以被忽略：SIGKILL，它将结束进程：SIGSTOP，它是作业控制机制的一部分，将挂起作业的执行。
> - 恢复信号的默认操作
> - 执行一个预先安排的信号处理函数。进程可以登记特殊的信号处理函数。当进程收到信号时，信号处理函数将像中断服务例程一样被调用，当从信号处理函数返回时，控制被返回给主程序，并且继续正常执行。

但是，信号和中断有所不同。中断的响应和处理都发生在内核空间，而信号的响应发生在内核空间，信号处理程序的执行却发生在用户空间。
 那么什么时候检测和响应信号？通常发生在两种情况下：

> - 当前进程由于系统调用、中断或异常而进入内核空间以后，从内核空间返回到用户空间前戏
> - 当前进程在内核进入睡眠以后刚被唤醒的时候，由于检测到信号的存在而提前返回到用户空间

**3、信号的本质**

信号是在软件层次上对中断机制的一种模拟，在原理上，一个进程收到一个信号与处理器收到一个中断请求可以说是一样的。信号是异步的，一个进程不必通过任何操作来等待信号的到达，事实上，进程也不知道信号到底什么时候到达。

信号是进程间通信机制中唯一的异步通信机制，可以看作是异步通知，通知接收信号的进程有哪些事情发生了。信号机制经过POSIX实时扩展后，功能更加强大，除了基本通知功能外，还可以传递附加信息。

**4、信号来源**

信号事件的发生有两个来源：硬件来源(比如我们按下键盘或者其他硬件故障)；润健来源，最常用发送信号的系统函数是kill，raise,alarm和setitimer以及sigqueue函数，软件来源还包括一些非法运算等操作。

**5、关于信号处理机制的原理(内核角度)**

内核给一个进程发送中断信号，是在进程所在的进程表项的信号域设置对应于该信号的位。这里要补充的是，如果信号发送给一个正在睡眠的进程，那么要看该进程进入睡眠的优先级，如果进程睡眠在可被中断的优先级上，则唤醒正在睡眠的进程；否则仅设置进程表中信号域相应的位，而不是唤醒进程。这一点比较重要，因为进程检查是否收到信号的时机是：一个进程在即将从内核态返回到用户态时；或者，在一个进程进入或离开一个适当的低调度优先级睡眠状态时。

内核处理一个进程吸收的信号的时机是在一个进程从内核态返回用户态时。所以，当一个进程在内核态下运行时，软中断信号并不立即起作用，要等到将返回用户态时才处理。进程只有处理完信号才会返回用户态，进程在用户态下不会有未处理完的信号。

内核处理一个进程收到的软中断信号是在该进程的上下文中。因此，进程必须处于运行状态。如果进程收到一个要捕捉的信号，那么进程从内核态返回用户态时执行用户定义的函数。而且执行用户定义的函数的方法很巧妙，内核是在用户栈上创建一个新的层，该层中将返回地址的值设置成用户定义的处理函数的地址，这样进程从内核返回弹出栈顶时就返回到用户定义的函数出，从函数返回再弹出栈顶时，才返回原先进入内核的地方，接着原来的地方继续运行。这样做的原因是用户定义的处理函数不能切不允许在内核态下执行。

**6、信号的生命周期**

![img](https:////upload-images.jianshu.io/upload_images/5713484-025dbc4847a838ef.png?imageMogr2/auto-orient/strip|imageView2/2/w/594/format/webp)

信号的生命周期.png

### (四)、信号量(semaphore)

**1、什么是信号量**

> 信号量又称为信号灯，它用来协调不用进程间的数据对象，而最主要的应用是共享内存方式的进程间通信。本质上，信号量时一个计数器，他用来记录某个资源(如共享内存)的存取状况。信号量的使用，主要是用来保护共享资源，使得资源在一个时刻只有一个进程(线程)所拥有。

信号量的值为正的时候，说明它空闲。所有的线程可以锁定而使用它。若为0，说明它被占用，测试的线程要进入睡眠队列中，等待被唤醒。

**2、信号量的注意事项**

> - 为了防止出现因多个程序同时访问一个共享资源而引发的一系列问题，我们需要这一种方法，它可以通过生成并使用令牌来授权，在任一时刻只能由一个执行线程访问代码的临界区域
> - 临界区域是指执行数据更新的代码需要独占式地执行。而信号量就可以提供这样的一种访问机制，让一个临界区同一时间只有一个线程在访问它，也就说信号量临界区是指执行数据更新的代码需要独占式地执行。而信号量就可以提供这样的一种访问机制，让一个临界区同一时间只有一个线程在访问它，也就说信号量是用来协调进程对共享资源的访问。
> - 信号量时一个特殊的变量，程序对其访问都是原子操作，且只允许对它进行等待(即P-信号变量)和发送(即V信号变量)信息操作。
> - 最简单的信号量只能取0和1的变量，这也是信号量最常见的一种形式，叫做二进制信号量。而可以取多个正整数的信号量被称为通用信号量

**3、信号量的原理**

由于信号量只能进行两种操作即"等待"和"发送"，即P(sv)和V(sv)，他们的行为是这样：

> - P(sv):如果sv的值大于零，就给它减1；如果它的值为零，就挂起该进程的执行
> - V(sv):如果有其他进程因等待sv而被挂起，就让它恢复运行，如果没有进程因等待sv而挂起，就给它加1。

举个例子，就是两个进程共享信号量sv，一旦其中一个进程执行了P(sv)操作，他将得到信号量，并可以进如临界区，使sv减1。而第二个进程将被阻止进入临界区，因为当它试图执行P(sv)时，sv为0，它会挂起以等待第一个进程离开临界区并执行(sv)释放信号。

**3、信号量的分类**

Linux提供两种信号量

> - 内核信号量：由内核控制路径使用
> - 用户态进程使用的信号量：这种信号量又分为POSIX信号量和SYSTEM V信号量

POSIX辛信号量又分为有名信号量和无名信号量

> - 有名信号量：其值保存在文件中，所以它可以用于线程也可以用于进程间同步
> - 无名信号量：其值保存在内存。

POSIX信号量和SYSTEM V信号量的比较

> - 对POSIX来说，信号量是个非负数。常用于线程间同步
>    而SYSTEM V信号量则是一个或者多个信号量集合，他对应的是一个信号量的结构体，这个结构体为SYSTEM V IPC服务的，信号量只不过是它的一部分。常用语进程间同步。
> - POSIX信号量的引用头文件是<semaphore,h>，而SYSTEM V信号量的引用头文件是<sys/sem.h>

- 从使用的角度，System V信号量是简单的。比如，POSIX信号量的创建和初始化PV操作就很方便。

### (五)、消息队列(message)

**1、消息队列也称为报文队列：**

> 消息队列也成为报文队列，消息队列是随内核持续的，只有在内核重其或者显示删除一个消息队列时，该消息队列才会真正删除系统中记录消息队列的数据结构体 struct  ipc_ids_msg_ids位于内核中，系统中所有消息队列都可以在结构msg_ids中找到访问入口。

**2、消息队列的原理及注意事项：**

> - 消息队列其实就是一个消息的链表，每个消息都有一个队列头，称为struct_msg_queue，这个队列头描述了消息队列的key值，用户ID，组ID等信息，但它存于内核中而结构体struct msqid_ds能够返回或设置消息队列的信息，这个结构体位于用户空间中，与msg_queue结构相似的消息队列允许一个或多个进程向它写入或读取消息，消息队列是消息的链表。
> - 消息是按消息类型访问，进程必须指定消息类型来读取消息，同样，当向消息队列中写入消息事业必须给出消息的类型，如果读队列使用消息的类型为0，则读取队列中的第一条消息。
> - 内核空间的结构体msg_queue描述了对应key值消息队列的情况，而对应用户空间的msqid_ds这个结构体，因此，可以操作msgid_ds这个结构体来操作消息队列。

### (六)、共享内存(share memory)

共享内存是进程间通信中最简单的方式之一。

**1、什么是共享内存?**

> 共享内存是系统处于多个进程之间通讯的考虑，而预留的一块内存区。共享内存允许两个或更多的进程访问同一块内存，就如同malloc()函数向不同进程返回了指向同一个物理内存区域的指针。当一个进程改变了这块地址中的内容的时候，其他进程都会觉察到这个更改。

**2、关于共享内存**

> - 当一个程序加载进内存后，它就被分成叫做页的块。通信将存在内存的两个页之间或者两个独立的进程之间。总之，当一个程序想和另外一个程序通信的时候，那内存将会为这两个程序生成一块公共的内存区域。这块被两个进程分享的内存区域叫做共享内存。
> - 由于所有进程共享同一块内存，共享内存在各种进程间通信方式中具有最高的效率。访问共享内存区域和访问进程独有的内存区域一样快，并不需要通过系统调用或者其他需要切入内核的过程来完成。同时它也也避免了对数据的跟中不必要的复制。
> - 如果没有共享内存的概念，那一个进程不能存取另外一个进程的内存部分，因而导致共享数据或者通信失效。因为系统内核没有对访问共享内存进行同步，开发者必须提供自己的同步措施。
> - 解决了这些问题的常用方法是是通过信号量进行同步。不过通常我们程序只有一个进程访问了共享内存，因此在集中展示了共享内存机制的同时，我们避免了让代码被同步逻辑搞的混乱不堪。

为了简化共享数据的完整性和避免同时存取数据，内核提供了一种专门存取共享内存资源的机制。这称为互斥体或者Mutex对象。

**3、Mutex对象**

> 例如，在数据被写入前不允许进程从共享内存中读取信息、不允许两个进程同时向一个共享内存地址写入数据等。

当一个基础想和两一个进程通信的时候，它将按以下顺序运行：

> - 1、获取Mutex对象，锁定共享区域
> - 2、将要通信的数据写入共享区域
> - 3、释放Mutex对象

当一个进程从这个区域读取数据的时候，它将重复同样的步骤，只是将第二步变成读取。

**4、内存模型**

要使用一块共享内存

> - 进程必须首先分配它
> - 随后需要访问这个共享内存块的每一个进程都必须将这个共享内存绑定到自己的地址空间中。
> - 当完成通信之后，所有进程都脱离共享内存，并且由一个进程释放该共享内存块。

在/proc/sys/kernel/目录下，记录着共享内存的一些限制，如一个共享内存区的最大字节数shmmax，系统范围内最大的共享内存区标志符数shmmni等。

**5、Linux系统内存模型**

> - 在Linux系统中，每个进程的虚拟内存是被分为许多页面的。这些内存页面中包含了实际的数据。每个进程都会维护一个从内存地址到虚拟内存页面之间的映射关系。尽管每个进程都有自己的内存地址，不同的进程可以同时将同一个页面页面映射到自己的地址空间，从而达到共享内存的目的。
> - 分配一个新的共享内存块会创建新的内存页面。因为所有进程都希望共享对同一块内存的访问，只应由一个进程创建一块新的共享内存。再次分配一块已经存在的内存块不会创建新的页面，而只是会返回一个标示该内存块的标识符。
> - 一个进程如需使用这个共享内存块，则首先需要将它绑定到自己的地址空间中。
> - 这样会创建一个从进程本身虚拟地址到共享页面的映射关系。当对共享内存的使用结束之后，这个映射关系将被删除。
> - 当再也没有京城需要使用这个共享内存块的时候，必须有一个(有且只有一个)进程负责释放这个被共享的内存页面。
> - 所有共享内存块的大小必须是系统页面大小的整数倍。系统页面大小指的是系统中单个内存页面包含的字节数。在Linux系统中，内存页面大小是4KB，不过您仍然应高通过调用getPageSize获取这个值。

**6、Linux共享内存的实现步骤**

共享内存的实现分为两个步骤:

> - 创建共享内存，使用shmget函数
> - 映射共享内存，将这段创建的共享内存映射到具体的进程空间中，使用shmat函数

### (七)、套接字(socket)

套接字也是一种进程间通信机制，与其他通信机制不同的是，它可用于不同机器间的进程通信。
 更为一般的进程间通信机制，可用于不同机器之间的进程间通信。起初是右Unix系统的BSD分之开发出来的，但是现在一般可以移植其他类Unix系统上。比如Linux和System V的变种都支持套接字。

### (八)  Linux的几种跨进程通信的方式的比较

**1、效率比较**

| 类型             | 无连接 | 可靠 | 流控制 | 优先级 |
| ---------------- | ------ | ---- | ------ | ------ |
| 匿名PIPE         | N      | Y    | Y      | N      |
| 命名PIPE(FIFO)   | N      | Y    | Y      | N      |
| 信号量           | N      | Y    | Y      | Y      |
| 消息队列         | N      | Y    | Y      | Y      |
| 共享内存         | N      | Y    | Y      | Y      |
| UNIX流SOCKET     | N      | Y    | Y      | N      |
| UNIX数据包SOCKET | Y      | Y    | N      | N      |

> PS：无连接是指无需调用某种行动是OPEN，就有发送消息的能力流控制，如果系统资源短缺或者不能接受更多的消息，则发送进程能进进行流量控制。

**2、优缺点比较**

> - 匿名管道(pipe)：速度慢，容量有限，只有父子进程能通讯
> - 有名管道(FIFO)： 任务进程都能通讯，但速度慢
> - 消息队列(message queue)：容量收到系统限制，且要注意第一次读的时候，要考虑上一次没有读完数据问题。
> - 信号量：不能传递复杂消息，只能用来同步
> - 共享内存区：能够容易控制容量，速度快，但要保持同步，比如一个进程在写的时候，另一个进程要注意读写的问题。相当于线程中的线程安全，当然，共享内存区同样可以做线程间通讯，不过没有这个必要，线程间本来就已经共享了同一进程内的一块内存

**3、使用场景**

> - 如果用户传递的信息较少或是需要通过信号来出发某些行为，上面提到的软中断限号机制不失为一种简洁有效的一种进程间通信方式。但若是进程间要求传递的信息量比较大或者进程间存在交换数据的要求，那就需要考虑别的通信方式。
> - 匿名管道简单方便，但局限于单向通信的工作方式，并且只能创建它的进程及其子孙进程之间实现管道的共享。

- 有名管道虽然可以提供给任意关系的进程使用，但是由于其长期存在于系统之中，使用不当容易出错。所以不建议初级开发者使用。

> - 消息缓存可以不再局限于父子进程，而允许任意进程间通过共享消息队列来实现进程间通信，并由系统调用函数来实现消息发送和接受方之间的同步，从而使得用户在使用消息缓冲进行通信时不再需要考虑同步问题，使用方便，但是信息的复制需要额外的消耗CPU的时间，不适宜信息量大或操作频繁的场合。
> - 共享内存针对消息缓存的缺点而改进，利用了内存缓存区直接交换信息，无需复制，快捷、信息量大的是其优点。但是共享内存的通信方式是通过将共享内存缓存直接附加到进程的虚拟地址空间中来实现的。因此这些进程之间的读写操作的同步问题操作系统无法实现。必须由各进程利用其它同步工具解决。另外， 由于内存实体存在于计算机系统中，所以只能由处于同一个计算机系统中的其它进程共享，不方便网络通信。

**补充一点:**
 共享内存块提供了在任意数量的进程之间进行高效双向通信的机制。每个使用者都可以读取写入数据，但是所有程序之间必须达成并遵守一定的协议，以防止诸如在读取信息之前覆写内存空间等竞争状态的出现。不行的是，Linux无法严格保证提供对共享内存块的独占访问，同时，多个使用共享内存块的进程之间必须协调使用同一个键值。





# Android跨进程通信IPC之Bionic

## 一、为什么要学习Bionic

>  

Bionic库是Android的基础库之一，也是连接Android系统和Linux系统内核的桥梁，Bionic中包含了很多基本的功能模块，这些功能模块基本上都是源于Linux，但是就像青出于蓝而胜于蓝，它和Linux还是有一些不一样的的地方。同时，为了更好的服务Android，Bionic中也增加了一些新的模块，由于本次的主题是Androdi的跨进程通信，所以了解Bionic对我们更好的学习Android的跨进行通信还是很有帮助的。

Android除了使用ARM版本的内核外和传统的x86有所不同，谷歌还自己开发了Bionic库，那么谷歌为什么要这样做那?

## 二、谷歌为什么使用Bionic库

谷歌使用Bionic库主要因为以下三点:

>  

- 1、谷歌没有使用Linux的GUN Libc，很大一部分原因是因为GNU Libc的授权方式是GPL 授权协议有限制，因为一旦软件中使用了GPL的授权协议，该系统所有代码必须开元。
- 2、谷歌在BSD的C库上的基础上加入了一些Linux特性从而生成了Bionic。Bionic名字的来源就是BSD和Linux的混合。而且不受限制的开源方式，所以在现代的商业公司中比较受欢迎。
- 3、还有就是因为性能的原因，因为Bionic的核心设计思想就是"简单"，所以Bionic中去掉了很多高级功能。这样Bionic库仅为200K左右，是GNU版本体积的一半，这意味着更高的效率和低内存的使用，同时配合经过优化的Java VM  Dalvik才可以保证高的性能。

## 三、Bionic库简介

>  

Bionic 音标为 bīˈänik，翻译为"仿生"

Bionic包含了系统中最基本的lib库，包括libc，libm，libdl，libstd++，libthread_db，以及Android特有的链接器linker。

## 四、Bionic库的特性

Bionic库的特性很多，受篇幅限制，我挑几个和大家平时接触到的说下

### (一)、架构

Bionic 当前支持ARM、x86和MIPS执行集，理论上可以支持更多，但是需要做些工作，ARM相关的代码在目录arch-arm中，x86相关代码在arch-x86中，mips相关的代码在arch-mips中。

### (二)、Linux核心头文件

Bionic自带一套经过清理的Linxu内核头文件，允许用户控件代码使用内核特有的声明(如iotcls，常量等)这些头文件位于目录：

>  

bionic/libc/kernel/common
 bionic/libc/kernel/arch-arm
 bionic/libc/kernel/arch-x86
 bionic/libc/kernel/arch-mips

### (三)、DNS解析器

虽然Bionic 使用NetBSD-derived解析库，但是它也做了一些修改。

- 1、不实现name-server-switch特性
- 2、读取/system/etc/resolv.conf而不是/etc/resolv.config
- 3、从系统属性中读取服务器地址列表，代码中会查找'net.dns1'，'net.dns2'，等属性。每个属性都应该包含一个DNS服务器的IP地址。这些属性能被Android系统的其它进程修改设置。在实现上，也支持进程单独的DNS服务器列表，使用属性'net.dns1.<pid>'、'net.dns2.<pid>'等，这里<pid> 表示当前进程的ID号。
- 4、在执行查询时，使用一个合适的随机ID(而不是每次+1)，以提升安全性。
- 5、在执行查询时，给本地客户socket绑定一个随机端口号，以提高安全性。
- 6、删除了一些源代码，这些源代码会造成了很多线程安全的问题

### (四)、二进制兼容性

由于Bionic不与GNU C库、ucLibc，或者任何已知的Linux C相兼容。所以意味着不要期望使用GNU C库头文件编译出来的模块能够正常地动态链接到Bionic

### (五)、Android特性

Bionict提供了少部分Android特有的功能

**1、访问系统特性**

Android 提供了一个简单的"共享键/值 对" 空间给系统的中的所有进程，用来存储一定数量的"属性"。每个属性由一个限制长度的字符串"键"和一个限制长度的字符串"值"组成。
 头文件<sys/system_properties.h>中定义了读系统属性的函数，也定义了键/值对的最大长度。

**2、Android用户/组管理**

在Android中没有etc/password和etc/groups 文件。Android使用扩展的Linux用户/组管理特性，以确保进程根据权限来对不同的文件系统目录进行访问。
 Android的策略是：

- 1、每个已经安装的的应用程序都有自己的用户ID和组ID。ID从10000（一万）开始，小于10000（一万）的ID留给系统的守护进程。
- 2、tpwnam()能识别一些硬编码的子进程名(如"radio")，能将他们翻译为用户id值，它也能识别"app_1234"，这样的组合名字，知道将后面的1234和10000(一万)相加，得到的ID值为11234.getgrname()也类似。
- 3、getservent() Android中没有/etc/service，C库在执行文件中嵌入只读的服务列表作为代替，这个列表被需要它的函数所解析。所见文件bionic/libc/netbsd/net/getservent.c和bionic/libc/netbsd/net/service.h。
   这个内部定义的服务列表，未来可能有变化，这个功能是遗留的，实际很少使用。getservent()返回的是本地数据，getservbyport()和getservbyname()也按照同样的方式实现。
- 4、getprotoent() 在Android中没有/etc/protocel，Bionic目前没有实现getprotocent()和相关函数。如果增加的话，很可能会以getervent()相同的方式。

## 五、Bionic库的模块简介

Bionic目录下一共有5个库和一个linker程序
 5个库分别是:

- 1、libc
- 2、libm
- 3、libdl
- 4、libstd++
- 5、libthread_db

**(一)、Libc库**

Libc是C语言最基础的库文件，它提供了所有系统的基本功能，这些功能主要是对系统调用的封装，是Libc是应用和Linux内核交流的桥梁，主要功能如下：

- 进程管理：包括进程的创建、调度策略和优先级的调整
- 线程管理：包括线程的创建和销毁，线程的同步/互斥等
- 内存管理：包括内存分配和释放等
- 时间管理：包括获取和保存系统时间、获取当前系统运行时长等
- 时区管理：包括时区的设置和调整等
- 定时器管理：提供系统的定时服务
- 文件系统管理：提供文件系统的挂载和移除功能
- 文件管理：包括文件和目录的创建增删改
- 网络套接字：创建和监听socket，发送和接受
- DNS解析：帮助解析网络地址
- 信号：用于进程间通信
- 环境变量：设置和获取系统的环境变量
- Android Log：提供和Android Log驱动进行交互的功能
- Android 属性：管理一个共享区域来设置和读取Android的属性
- 标准输入/输出：提供格式化的输入/输出
- 字符串：提供字符串的移动、复制和比较等功能
- 宽字符：提供对宽字符的支持。

**(二)、Libm库**

Libm 是数学函数库，提供了常见的数学函数和浮点运算功能，但是Android浮点运算时通过软件实现的，运行速度慢，不建议频繁使用。

**(三)、libdl库**

libdl库原本是用于动态库的装载。很多函数实现都是空壳，应用进程使用的一些函数，实际上是在linker模块中实现。

(四)、Libm库

libstd++ 是标准的C++的功能库，但是，Android的实现是非常简单的，只是new，delete等少数几个操作符的实现。

**(五)、libthread_db库**

libthread_db 用来支持对多线程的中动态库的调试。

(**六)、Linker模块**

Linux系统上其实有两种并不完全相同的可执行文件

- 一种是静态链接的可执行程序。静态可执行程序包含了运行需要的所有函数，可以不依赖任何外部库来运行。
- 另一种是动态链接的可执行程序。动态链接的可执行程序因为没有包含所需的库文件，因此相对于要小很多。

静态可执行程序用在一些特殊场合，例如，系统初始化时，这时整个系统还没有准备好，动态链接的程序还无法使用。系统的启动程序Init就是一个静态链接的例子。在Android中，会给程序自动加上两个".o"文件，分别是"crtbegin_static.c"和"certtend_android.o"，这两个".o"文件对应的源文件位于bionic/libc/arch-common/bionic目录下，文件分别是crtbegin.c和certtend.S。_start()函数就位于cerbegin.c中。

在动态链接时，execuve()函数会分析可执行文件的文件头来寻找链接器，Linux文件就是ld.so，而Android则是Linker。execuve()函数将会将Linker载入到可执行文件的空间，然后执行Linker的_start()函数。Linker完成动态库的装载和符号重定位后再去运行真正的可执行文件的代码。

## 六、Bionic库的内存管理函数

**(一)内存管理函数**

对于32位的操作系统，能使用的最大地址空间是4GB，其中地址空间03GB分配给用户进程使用，地址空间3GB4GB由内核使用，但是用户进程并不是在启动时就获取了所有的0~3GB地址空间的访问权利，而是需要事先向内核申请对模块地址空间的读写权利。而且申请的只是地址空间而已，此时并没有分配真是的物理地址。只有当进程访问某个地址时，如果该地址对应的物理页面不存在，则由内核产生缺页中断，在中断中才会分配物理内存并建立页表。如果用户进程不需要某块空间了，可以通过内核释放掉它们，对应的物理内存也释放掉。

但是由于缺页中断会导致运行缓慢，如果频繁的地由内核来分配和释放内存将会降低整个体统的性能，因此，一般操作系统都会在用户进程中提供地址空间的分配和回收机制。用户进程中的内存管理会预先向内核申请一块打的地址空间，称为堆。当用户进程需要分配内存时，由内存管理器从堆中寻找一块空闲的内存分配给用户进程使用。当用户进程释放某块内存时，内存管理器并不会立刻将它们交给内核释放，而是放入空闲列表中，留待下次分配使用。

内存管理器会动态的调整堆的大小，如果堆的空间使用完了，内存管理器会向堆内存申请更多的地址空间，如果堆中空闲太多，内存管理器也会将一部分空间返给内核。

**(二) Bionic的内存管理器——dlmalloc**

dlmalloc是一个十分流行的内存分配器。dlmalloc位于bionic/libc/upstream-dlmalloc下，只有一个C文件malloc.c。由于本次主题是跨进程通信，后续有时间就Android的内存回收单独作为一个课题去讲解，今天就不详细说了，就简单的说下原理。
 dlmalloc的原理:

>  

- dlmalloc内部是以链表的形式将"堆"的空闲空间根据尺寸组织在一起。分配内存时通过这些链表能快速地找到合适大小的空闲内存。如果不能找到满足要求的空闲内存，dlmalloc会使用系统调用来扩大堆空间。
- dlmalloc内存块被称为"trunk"。每块大小要求按地址对齐(默认8个字节)，因此，trunk块的大小必须为8的倍数。
- dlmalloc用3种不同的的链表结构来组织不同大小的空闲内存块。小于256字节的块使用malloc_chunk结构，按照大小组织在一起。由于尺寸小于的块一共有256/8=32，所以一共使用了32个malloc_chunk结构的环形链表来组织小于256的块。大小大于256字节的块由结构malloc_tree_chunk组成链表管理，这些块根据大小组成二叉树。而更大的尺寸则由系统通过mmap的方式单独分配一块空间，并通过malloc_segment组成的链表进行管理。
- 当dlmalloc分配内存时，会通过查找这些链表来快速找到一块和要求的尺寸大小最匹配的空闲内存块(这样做事为了尽量避免内存碎片)。如果没有合适大小的块，则将一块大的分成两块，一块分配出去，另一块根据大小再加入对应的空闲链表中。
- 当dlmalloc释放内存时，会将相邻的空闲块合并成一个大块来减少内存碎片。如果空闲块过多，超过了dlmaloc内存的阀值，dlmalloc就开始向系统返回内存。
- dlmalloc除了能管理进程的"堆"空间，还能提供私有堆管理，就是在堆外单独分配一块地址空间，由dlmalloc按照同样的方式进行管理。dlmalloc中用来管理进程的"堆"空间的函数，都带有"dl"前缀，如"dlmalloc"，"dlfree"等，而私有堆的管理函数则带有前缀"msspace_"，如"msspace_malloc"

Dalvk虚拟机中使用了dlmalloc进行私有堆管理。

## 七、线程

Bionic中的线程管理函数和通用的Linux版本的实现有很多差异，Android根据自己的需要做了很多裁剪工作。

### (一)、Bionic线程函数的特性

**1、pthread的实现基于Futext，同时尽量使用简单的代码来实现通用操作，特征如下：**

- pthread_mutex_t，pthread_cond_t类型定义只有4字节。
- 支持normal,recursive and error-check 互斥量。考虑到通常大多数的时候都使用normal，对normal分支下代码流程做了很细致的优化
- 目前没有支持读写锁，互斥量的优先级和其他高级特征。在Android还不需要这些特征，但是在未来可能会添加进来。

**2、Bionic不支持pthread_cancel()，因为加入它会使得C库文件明显变大，不太值得，同时有以下几点考虑**

- 要正确实现pthread_cancel()，必须在C库的很多地方插入对终止线程的检测。
- 一个好的实现，必须清理资源，例如释放内存，解锁互斥量，如果终止恰好发生在复杂的函数里面(比如gthosbyname())，这会使许多函数正常执行也变慢。
- pthread_cancel()不能终止所有线程。比如无穷循环中的线程。
- pthread_cancel()本身也有缺点，不太容易移植。
- Bionic中实现了pthread_cleanup_push()和pthread_cleanup_pop()函数，在线程通过调用pthread_exit()退出或者从它的主函数中返回到时候，它们可以做些清理工作。

**3、不要在pthread_once()的回调函数中调用fork()，这么做会导致下次调用pthread_once()的时候死锁。而且不能在回调函数中抛出一个C++的异常。**

**4、不能使用_thread关键词来定义线程本地存储区。**

### (二)、创建线程和线程的属性

#### **1、创建线程**

函数[pthread_create()](https://link.jianshu.com?t=http://baike.baidu.com/item/pthread_create)用来创建线程，原型是：



```cpp
int  pthread_create（（pthread_t  *thread,  pthread_attr_t  *attr,  void  *（*start_routine）（void  *）,  void  *arg）
```

其中，pthread_t在android中等同于long



```cpp
typedef long pthread_t;
```

- 参数thread是一个指针，pthread_create函数成功后，会将代表线程的值写入其指向的变量。
- 参数 args 一般情况下为NULL，表示使用缺省属性。
- 参数start_routine是线程的执行函数
- 参数arg是传入线程执行函数的参数

若线程创建成功，则返回0，若线程创建失败，则返回出错编号。
 PS:要注意的是，pthread_create调用成功后线程已经创建完成，但是不会立刻发生线程切换。除非调用线程主动放弃执行，否则只能等待线程调度。

#### 2、线程的属性

结构 pthread_atrr_t用来设置线程的一些属性，定义如下：

```cpp
typedef struct
{
    uint32_t        flags;                
    void *    stack_base;              //指定栈的起始地址
    size_t    stack_size;               //指定栈的大小
    size_t    guard_size;  
    int32_t   sched_policy;           //线程的调度方式
    int32_t   sched_priority;          //线程的优先级
}
```

使用属性时要先初始化，函数原型是：



```cpp
int  pthread_attr_init(pthread_attr_t*  attr)
```

通过pthread_attr_init()函数设置的缺省属性值如下：



```php
int pthread_attr_init(pthread_attr_t* attr){
     attr->flag=0;
     attr->stack_base=null;
     attr->stack_szie=DEFAULT_THREAD_STACK_SIZE;  //缺省栈的尺寸是1MB
     attr0->quard_size=PAGE_SIZE;                                    //大小是4096
     attr0->sched_policy=SCHED_NORMAL;                      //普通调度方式
     attr0->sched_priority=0;                                                 //中等优先级
     return 0；
}
```

下面介绍每项属性的含义。

>  

- 1、flag  用来表示线程的分离状态
   Linux线程有两种状态:分离(detch)状态和非分离(joinable)状态，如果线程是非分离状态(joinable)状态，当线程函数退出时或者调用pthread_exit()时都不会释放线程所占用的系统资源。只有当调用了pthread_join()之后这些资源才会释放。如果是分离(detach)状态的线程，这些资源在线程函数退出时调用pthread_exit()时会自动释放
- 2、stack_base: 线程栈的基地址
- 3、stack_size: 线程栈的大小。基地址和栈的大小。
- 4、guard_size: 线程的栈溢出保护区大小。
- 5、sched_policy：线程的调度方式。
   线程一共有3中调度方式：SCHED_NORMAL，SCHED_FIFO，SCHED_RR。其中SCHED_NORMAL代表分时调度策略，SCHED_FIFO代表实时调度策略，先到先服务，一旦占用CPU则一直运行，一直运行到有更高优先级的任务到达，或者自己放弃。SCHED_RR代表实时调度策略:时间片轮转，当前进程时间片用完，系统将重新分配时间片，并置于就绪队尾。
- 6、sched_priority:线程的优先级。

Bionic虽然也实现了pthread_attr_setscope()函数，但是只支持PTHREAD_SCOP_SYSTEM属性，也就意味着Android线程将在全系统的范围内竞争CPU资源。

#### 3、退出线程的方法

**(1)、调用pthread_exit函数退出**

一般情况下，线程运行函数结束时线程才退出。但是如果需要，也可以在线程运行函数中调用pthread_exit()函数来主动退出线程运行。函数原型如下：



```cpp
 void pthread_exit( void * retval) ;
```

其中参数retval用来设置返回值

**(2)、设备布尔的全局变量**

但是如果希望在其它线程中结束某个线程？前面介绍了Android不支持pthread_cancel()函数，因此，不能在Android中使用这个函数来结束线程。通俗的方法是，如果线程在一个循环中不停的运行，可以在每次循环中检查一个初始值为false的全局变量，一旦这个变量的值为ture，则主动退出，这样其它线程就可以铜鼓改变这个全局变量的值来控制线程的退出，示例如下：



```cpp
bool g_force_exit =false;
void * thread_func(void *){
         for(;;){
             if(g_force_exit){
                    break;
             }
             .....
         }
         return NULL;
}
int main(){
     .....
     q_force_exit=true；       //青坡线程退出
}
```

这种方法实现起来简单可靠，在编程中经常使用。但它的缺点是：如果线程处于挂起等待状态，这种方法就不适用了。
 另外一种方式是使用pthread_kill()函数。pthread_kill()函数的作用不是"杀死"一个线程，而是给线程发送信号。函数如下：



```cpp
  int pthread_kill(pthread tid,int sig);
```

即使线程处于挂起状态，也可以使用pthead_kill()函数来给线程发送消息并使得线程执行处理函数，使用pthread_kill()函数的问题是：线程如果在信号处理函数中退出，不方便释放在线程的运行函数中分配的资源。

**(3)、通过管道**

更复杂的方法是：创建一个管道，在线程运行函数中对管道"读端"用select()或epoll()进行监听，没有数据则挂起线程，通过管道的"写端"写入数据，就能唤起线程，从而释放资源，主动退出。

#### 4、线程的本地存储TLS

线程本地存储(TLS)用来保存、传递和线程有关的数据。例如在前面说道的使用pthread_kill()函数关闭线程的例子中，需要释放的资源可以使用TLS传递给信号处理函数。

**(1)、TLS介绍**

TLS在线程实例中是全局可见的，对某个线程实例而言TLS是这个线程实例的私有全局变量。同一个线程运行函数的不同运行实例，他们的TLS是不同的。在这个点上TLS和线程的关系有点类似栈变量和函数的关系。栈变量在函数退出时会消失，TLS也会在线程结束时释放。Android实现了TLS的方式是在线程栈的顶开辟了一块区域来存放TLS项，当然这块区域不再受线程栈的控制。

TLS内存区域按数组方式管理，每个数组元素称为一个slot。Android 4.4中的TLS一共有128 slot，这和Posix中的要求一致(Android 4.2是64个)

**(2)、TLS注意事项**

- TLS变量的数量有限，使用前要申请一个key，这个key和内部的slot关联一起，使用完需要释放。
   申请一个key的函数原型：

```cpp
int  pthread_key_create(pthread_key_t *key，void (*destructor_function) (void *) );
```

pthread_key_create()函数成功返回0，参数key中是分配的slot，如果将来放入slot中的对象需要在线程结束的时候由系统释放，则需要提供一个释放函数，通过第二个函数destructor_function传入。

- 释放 TLS key的函数原型是：

```cpp
 int  pthread_key_delete ( pthread_key_t) ;
```

pthread_key_delete()函数并不检查当前是否还有线程正在使用这个slot，也不会调用清理函数，只是将slot释放以供下次调用pthread_key_create()使用。

- 利用TLS保存数据中函数原型：

```cpp
  int pthread_setspecific(pthread_key_t key，const void *value) ;
```

- 读取TLS保存数据中的函数原型：

```cpp
 void * pthread_getsepcific (pthread_key_t key);
```

#### 5、线程的互斥量(Mutex)函数

Linux线程提供了一组函数用于线程间的互斥访问，Android中的Mutex类实质上是对Linux互斥函数的封装，互斥量可以理解为一把锁，在进入某个保护区域前要先检查是否已经上锁了。如果没有上锁就可以进入，否则就必须等待，进入后现将锁锁上，这样别的线程就无法再进入了，退出保护区后腰解锁，其它线程才可以继续使用

**(1)、Mutex在使用前需要初始化**

初始化函数是:

```cpp
int pthread_mutex_init(pthread_mutext_t *mutex, const pthread_mutexattr_t *attr);
```

成功后函数返回0，metex被初始化成未锁定的状态。如果参数attr为NULL，则使用缺省的属性MUTEX_TYPE-BITS_NORMAL。
 互斥量的属性主要有两种，类型type和范围scope，设置和获取属性的函数如下：



```cpp
int  pthread_mutexattr_settype (pthread_mutexattr_t * attr, type);
int  pthread_mutexattr_gettype (const pthread_mutexattr_t * attr, int *type);
int  pthread_mutexattr_getpshared(pthread_mutexattr_t *attr, int *pshared );
int  pthread_mutexattrattr_ setpshared (pthread_mutexattr_t *attr,int  pshared);
```

互斥量Mutex的类型(type) 有3种

>  

- PTHREAD_MUTEX_NORMAL:该类型的的互斥量不会检测死锁。如果线程没有解锁(unlock)互斥量的情况下再次锁定该互斥量，会产生死锁。如果线程尝试解锁由其他线程锁定的互斥量会产生不确定的行为。如果尝试解锁未锁定的互斥量，也会产生不确定的行为。** 这是Android目前唯一支持的类型 **。
- PTHREAD_MUTEX_ERRORCHECK:此类型的互斥量可提供错误检查。如果线程在没有解锁互斥量的情况下尝试重新锁定该互斥量，或者线程尝试解锁的互斥量由其他线程锁定。** Android目前不支持这种类型 ** 。
- PTHREAD_MUTEX_RECURSIVE。如果线程没有解锁互斥量的情况下重新锁定该互斥量，可成功锁定该互斥量，不会产生死锁情况，但是多次锁定该互斥量需要进行相同次数的解锁才能释放锁，然后其他线程才能获取该互斥量。如果线程尝试解锁的互斥量已经由其他线程锁定，则会返回错误。如果线程尝试解锁还未锁定的互斥量，也会返回错误。** Android目前不支持这种类型 ** 。

互斥量Mutex的作用范围(scope) 有2种

>  

- PTHREAD_PROCESS_PRIVATE:互斥量的作用范围是进程内，这是缺省属性。
- PTHREAD_PROCESS_SHARED:互斥量可以用于进程间线程的同步。Android文档中说不支持这种属性，但是实际上支持，在audiofliger和surfacefliger都有用到，只不过在持有锁的进程意外死亡的情况下，互斥量(Mutex)不能释放掉，这是目前实现的一个缺陷。

#### 6、线程的条件量(Condition)函数

**(1)为什么需要条件量Condition函数**

条件量Condition是为了解决一些更复杂的同步问题而设计的。考虑这样的一种情况，A和B线程不但需要互斥访问某个区域，而且线程A还必须等待线程B的运行结果。如果仅使用互斥量进行保护，在线程B先运行的的情况下没有问题。但是如果线程A先运行，拿到互斥量的锁，往下忘无法进行。

>  

条件量就是解决这类问题的。在使用条件量的情况下，如果线程A先运行，得到锁以后，可以使用条件量的等待函数解锁并等待，这样线程B得到了运行的机会。线程B运行完以后通过条件量的信号函数唤醒等待的线程A，这样线程A的条件也满足了，程序就能继续执行力额。

**(2)Condition函数**

1️⃣ 条件量在使用前需要先初始化，函数原型是：

```cpp
int  pthread_cond_init(pthread_cond_t *cond, const pthread_condattr *attr);
```

使用完需要销毁，函数原型是：

```cpp
int  pthread_cond_destroy(pthread_cond_t *cond);
```

条件量的属性只有 "共享(share)" 一种，下面是属性相关函数原型，下面是属性相关的函数原型：

```cpp
int  pthread_condattr_init(pthread_condattr_t *attr);
int  pthread_condattr_getpshared(pthread_condattr_t *attr,int *pshared);
int pthread_condattr_setpshared(pthread_condattr_t *attr,int pshared) 
int pthread_condattr_destroy (pthread_condattr_t *__attr);
```

"共享(shared)" 属性的值有两种

- PTHREAD_PROCESS_PRIVATE：条件量的作用范围是进程内，这是缺省的属性。
- PTHREAD_PROCESS_SHARED：条件量可以用于进程间线程同步。

2️⃣条件量的等待函数的原型如下：

```cpp
 int pthread_cond_wait (pthread_cond_t *__restrict __cond,pthread_mutex_t *__restrict __mutex);

 int pthread_cond_timedwait (pthread_cond_t *__restrict __cond,pthread_mutex_t *__restrict __mutex, __const struct timespec *__restrict __abstime);
```

条件量的等待函数会先解锁互斥量，因此，使用前一定要确保mutex已经上锁。锁上后线程将挂起。pthread_cond_timedwait()用在希望线程等待一段时间的情况下，如果时间到了线程就会恢复运行。

3️⃣ 可以使用函数pthread_cond_signal()来唤醒等待队列中的一个线程，原型如下：



```cpp
 int pthread_cond_signal (pthread_cond_t *__cond);
```

也可以通过pthread_cond_broadcast()唤醒所有等待的线程



```cpp
 int pthread_cond_broadcast (pthread_cond_t *__cond);
```

### (三)、Futex同步机制

- Futex 是 fast userspace mutext的缩写，意思是快速用户控件互斥体。这里讨论Futex是因为在Android中不但线程函数使用了Futex，甚至一些模块中也直接使用了Futex作为进程间同步手段，了解Futex的原理有助于我们理解这些模块的运行机制。
- Linux从2.5.7开始支持Futex。在类Unix系统开发中，传统的进程同步机制都是通过对内核对象进行操作来完成，这个内核对象在需要同步的进程中都是可见的。这种同步方法因为涉及用户态和内核态的切换，效率比较低。使用了传统的同步机制时，进入临界区即使没有其他进程竞争也会切到内核态检查内核同步对象的状态，这种不必要的切换明显降低了程序的执行效率。
- Futex就是为了解决这个问题而设计的。Futex是一种用户态和内核态混合的同步机制，使用Futex同步机制，如果用于进程间同步，需要先调用mmap()创建一块共享内存，Futex变量就位于共享区。同时对Futex变量的操作必须是原子的，当进程驶入进入临界区或者退出临界区的时候，首先检查共享内存中的Futex变量，如果没有其他进程也申请了使用临界区，则只修改Futex变量而不再执行系统调用。如果同时有其他进程也申请使用临界区，还是需要通过系统调用去执行等待或唤醒操作。这样通过用户态的Futex变量的控制，减少了进程在用户态和内核态之间切换的次数，从而最大程度的降低了系统同步的开销。

**1、Futex的系统调用**

在Linux中，Futex系统调用的定义如下：

```cpp
#define _NR_futex    240
```

(1) Fetex系统调用的原型是：

```csharp
int  futex(int *uaddr, int cp, int val, const struct timespec *timeout, int *uaddr2, int val3);
```

- uaddr是Futex变量，一个共享的整数计数器。
- op表示操作类型，有5中预定义的值，但是在Bionic中只使用了下面两种：① FUTEX_WAIT,内核将检查uaddr中家属器的值是否等于val，如果等于则挂起进程，直到uaddr到达了FUTEX_WAKE调用或者超时时间到。②FUTEXT_WAKE：内核唤醒val个等待在uaddr上的进程。
- val存放与操作op相关的值
- timeout用于操作FUTEX_WAIT中，表示等待超时时间。
- uaddr2和val3很少使用。

**(1) 在Bionic中，提供了两个函数来包装Futex系统调用：**

```csharp
extern int  _futex_wait(volatile void *ftx,int val, const struct timespec *timespec );
extern int _futex_wake(volatile void *ftx, int count);
```

**(2) Bionic还有两个类似的函数，它们的原型如下：**

```cpp
extern int  _futex_wake_ex(volatile void *ftx,int pshared,int val);
extern int  _futex_wait_ex(volatile void *fex,int pshared,int val, const stuct timespec *timeout);
```

这两个函数多了一个参数pshared,pshared的值为true 表示wake和wait操作是用于进程间的挂起和唤醒；值为false表示操作于进程内线程的挂起和唤醒。当pshare的值为false时，执行Futex系统调用的操作码为

```undefined
FUTEX_WAIT|FUTEX_PRIVATE_FLAG
```

内核如何检测到操作有FUTEX_PRIVATE_FLAG标记，能以更快的速度执行七挂起和唤醒操作。
 _futex_wait 和_futex_wake函数相当于pshared等于true的情况。

**(3) 在Bionic中，提供了两个函数来包装Futex系统调用：**

```csharp
extern int  _futex_syscall3(volatile void *ftx,int pshared,int val);
extern int  _futex_syscall4(volatile void *ftx,int pshared,int val, const struct timespec *timeout);
```

_futex_syscall3()相当于 _futex_wake()，而 _futex_system4()相当于 _futex_wait()。这两个函数与前面的区别是能指定操作码op作为参数。操作码可以是FUTEX_WAIT_FUTEX_WAKE或者它们和FUTEX_PRIVATE_FLAG的组合。

**2、Futex的用户态操作**

Futex的系统调用FUTEX_WAIT和FUTEX_WAKE只是用来挂起或者唤醒进程，Futex的同步机制还包括用户态下的判断操作。用户态下的操作没有固定的函数调用，只是一种检测共享变量的方法。Futex用于临界区的算法如下：

- 首先创建一个全局的整数变量作为Futex变量，如果用于进程间的同步，这个变量必须位于共享内存。Futex变量的初始值为0。
- 当进程或线程尝试持有锁的时候，检查Futex变量的值是否为0，如果为0，则将Futex变量的值设为1，然后继续执行；如果不为0，将Futex的值设为2以后，执行FUTEX_WAIT 系统调用进入挂起等待状态。
- Futex变量值为0表示无锁状态，1表示有锁无竞争的状态，2表示有竞争的状态。
- 当进程或线程释放锁的时候，如果Futex变量的值为1，说明没有其他线程在等待锁，这样讲Futex变量的值设为0就结束了；如果Futex变量的值2，说明还有线程等待锁，将Futex变量值设为0，同时执行FUTEX_WAKE()系统调用来唤醒等待的进程。

对Futex变量操作时，比较和赋值操作必须是原子的。



# Android跨进程通信IPC之关于"JNI"的那些事

在分析IPC基于Android 6.0)的过程中，里面的核心部分是Native的，并且还包含一些linux kernel，而作为Android开发者看到的代码大部分都是Java层，所以这就一定会存在Java与C/C++代码的来回跳转，那么久很有必要来先说一下JNI，本文主要内容如下：

> - 1、相关代码
> - 2、JNI简介
> - 3、Android中应用程序框架
> - 4、JNI查找方式
> - 5、loadLibrary源码分析
> - 6、JNI资源
> - 7、总结

## 一、相关代码

### (一)、代码位置

```csharp
frameworks/base/core/jni/AndroidRuntime.cpp

libcore/luni/src/main/java/java/lang/System.java
libcore/luni/src/main/java/java/lang/Runtime.java
libnativehelper/JNIHelp.cpp
libnativehelper/include/nativehelper/jni.h

frameworks/base/core/java/android/os/MessageQueue.java
frameworks/base/core/jni/android_os_MessageQueue.cpp

frameworks/base/core/java/android/os/Binder.java
frameworks/base/core/jni/android_util_Binder.cpp

frameworks/base/media/java/android/media/MediaPlayer.java
frameworks/base/media/jni/android_media_MediaPlayer.cpp
```

### (二)、代码链接

> - [AndroidRuntime.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/AndroidRuntime.cpp)
> - [System.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/lang/System.java)
> - [Runtime.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/lang/Runtime.java)
> - [JNIHelp.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libnativehelper/JNIHelp.cpp)
> - [jni.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libnativehelper/include/nativehelper/jni.h)
> - [MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)
> - [android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)
> - [Binder.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Binder.java)
> - [android_util_Binder.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_util_Binder.cpp)
> - [MediaPlayer.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/media/java/android/media/MediaPlayer.java)
> - [android_media_MediaPlayer.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/media/jni/android_media_MediaPlayer.cpp)

## 二、JNI简介

### (一)、JNI介绍

JNI(Java Native Interface，Java本地接口)，用于打通Java层与Native(C/C++)层。这不是Android系统独有的，而是Java所有。众所周知，Java语言是是跨平台的语言，而这跨平台的背后都是一开Java虚拟机，虚拟机采用C/C++编写，适配各个系统，通过JNI为上层Java提供各种服务，保证跨平台性。

其实不少的Java的程序员，享受着其跨平台性，可能全然不知JNI的存在。在Android平台，让JNI大放异彩，为更多的程序员所数值，往往为了提供效率或者其他功能需求，就需要在NDK上开发。本文的主要目的是介绍android上层中Java与Native的纽带JNI。

![img](https:////upload-images.jianshu.io/upload_images/5713484-e97d1c6d6d94e4a4.png?imageMogr2/auto-orient/strip|imageView2/2/w/325/format/webp)

JNI.png

### (二)、Java/JNI/C的关系

**1、C与Java的侧重**

> - C语言：C语言中重要的是函数 fuction
> - Java语言：Java中最重要的是JVM，class类，以及class中的方法

**2、C与Java的"面向"**

> - C语言：是面向函数的语言
> - Java语言：是面向对象的语言

**3、C与Java如何交流**

> - JNI规范：C语言与Java语言交流需要一个适配器，中间件，即JNI，JNI提供一共规范
> - C语言中调用Java的方法：可以让我们在C代码中找到Java代码class的方法，并且调用该方法
> - Java语言中调用C语言方法：同时也可以在Java代码中，将一个C语言的方法映射到Java的某个方法上
> - JNI的桥梁作用：JNI提供了一个桥梁，打通了C语言和Java语言的之间的障碍。

**4、JNI中的一些概念**

> - natvie：Java语言中修饰本地方法的修饰符(也可以理解为关键字)，被该修饰符修饰的方法没有方法体
> - Native方法：在Java语言中被native关键字修饰的方法是Native方法
> - JNI层：Java声明Native方法的部分
> - JNI函数：JNIEnv提供的函数，这些函数在jni.h中进行定义
> - JNI方法：Native方法对应JNI实现的C/C++方法，即在jni目录中实现的那些C语言代码

**5、JNI接口函数和指针**

平台相关代码是通过调用JNI函数来访问Java虚拟机功能的。JNI函数可以通过接口指针来获得。接口指针是指针的指针，它指向一个指针数组，而指针数组中的每个元素又指向一个接口函数。每个接口函数都处在数组的某个预定偏移量中。下图说明了接口指针的组织结构。

![img](https:////upload-images.jianshu.io/upload_images/5713484-441a0f5a36037683.png?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)

接口指针.png

JNI接口的组织类似于C++虚拟函数表或COM接口。使用接口表而不实用硬性编入的函数表的好处是使JNI名字空间与平台代码分开。虚拟机可以很容易地提供了多个版本的JNI寒暑表。例如，虚拟机 可以支持以下两个JNI函数表：

> - 一个表对非法参数进行全面检查，适用于调试程序
> - 另一个表只进行JNI规范所要求的最小程度的检查，因此效率较高。

JNI接口指针只在当前线程中有效。因此，本地方法不能讲接口指针从一个线程传递到另一个线程中。实现JNI虚拟机可能将本地线程的数据分配和储存在JNI接口指针所指向区域中。本地方法将JNI接口指针当做参数来接受。虚拟机在从相同的Java线程中对本地方法进行多次调用时，保证传递给本地方法的接口指针是相同的。但是，一个本地方方可以被不同的Java线程所调用，因此可以接受不同的JNI接口指针。

本地方法将JNI接口指针当参数来接受。虚拟机在从相同的Java线程对本地方法进行多次调用时，保证传递给本地方法的接口指针是相同的。但是，一个本地方法可被不同的Java线程所调用，因此可以接受不同的JNI接口指针。

![img](https:////upload-images.jianshu.io/upload_images/5713484-e584e725ea6a6308.png?imageMogr2/auto-orient/strip|imageView2/2/w/554/format/webp)

image.png

**6、JavaVM和JNIEnv**

**(1)、JavaVM**

> 代表Java虚拟机。所有的工作都是从获取虚拟机接口开始的。有两种方式：第一种方式，在加载动态链接库时，JVM会调用JNI_OnLoad(JavaVM * jvm, void * reserved)(如果定了该函数)。第一个参数会传入JavaVM指针；第二种方式，在native_code中调用JNI_CreateJavaVM(&jvm,(void*)&env,&vm_args) 可以得到JavaVM指针。两种方式都可以用全局变量，比如JavaVM * g_jvm来保存获取到的指针以便在任意上下文中使用。Android系统是利用第二种方式Invocation interface来创建JVM

**(2)、JNIEnv**

> JNIEnv，即JNIEnvironment；字面意思就是JNI环境。其实它是一个与线程相关的JNI环境结构体。所以JNIEnv类型实际代表了Java环境。通过这个JNIEnv*指针，就可以对Java端代码进行操作。与线程相关，不同线程的JNIEnv相互独立。 JNIEnv只在当前线程中有效。Native方法不能将JNIEnv从一个线程传递到另一个线程中。相同的Java线程对Native方法多次调用时，传递给Native方法的JNIEnv是相同的。但是，一个本地方法可能会被不同的Java线程调用，因此可以接受不同的JNIEnv。

和JNIEnv相比，JavaVM可以在进程中各个线程间共享。理论上一个进程可以有多个JavaVM，但Android只允许一个JavaVM。需要强调在Android SDK中强调了额 **" do not  cache JNIEnv \* "**，要用的时候在不同的线程中通过JavaVM * jvm的方法来获取与当前线程相关的JNIEnv *。

> 在Java里，每一个一个process可以产生多个JavaVM对象，但是在android上，每一个process只有一个Dalvik虚拟机对象，也就是在android进程中是通过有且只有一个虚拟机对象来服务所有Java和C/C++代码。Java的dex字节码和C/C++的 xxx.so 同时运行Dalvik虚拟机之内，共同使用一个进程空间。之所以可以相互调用，也是因为有Dalvik虚拟机。当Java代码需要C/C++代码时，Dalvik虚拟机加载xxx.so库时，会先调用JNI_Onload()，此时会把Java对象的指针存储于C层JNI组件的全局环境中，在Java层调用C层的Native函数时，调用Native函数线程必然通过Dalvik虚拟机来调用C层的Native函数。此时，虚拟机会为Native的C组件是实例化一个JNIEnv指针，该指针指向Dalvik虚拟机的具体函数列表，当JNI的C组件调用Java层的方法或属性时，需要JNIEnv指针来进行调用。当本地C/C++想获的当前线程所要使用的JNIEnv时，可以使用Dalvik虚拟机对象的JavaVM * jvm—>GetEnv()返回当前线程所在的JNIEnv*。

## 三、Android中应用程序框架

### (一)、正常情况下的Android框架

最顶层是** Android应用程序代码 **，上层的 ** 应用层 ** 和 ** 应用框架层 ** 主要是Java代码，中间有一层的 ** Framework框架层代码  ** 是C/C++代码，通过Framework进行系统调用，调用底层的库和 Linux内核

![img](https:////upload-images.jianshu.io/upload_images/5713484-dafea18493c8d790?imageMogr2/auto-orient/strip|imageView2/2/w/553/format/webp)

正常情况下的Android框架

### (二)、使用JNI的Android框架

使用JNI时的Android框架：绕过Framework提供的底层代码，直接调用自己的写的C代码，该代码最终会编译成一个库，这个库通过JNI提供的一个Stable的ABI 调用Linux kernel，ABI是二进制程序接口 application binary interface

![img](https:////upload-images.jianshu.io/upload_images/5713484-7f05249cc1ea3d4e.png?imageMogr2/auto-orient/strip|imageView2/2/w/554/format/webp)

使用JNI的Android框架.png

### (三)、Android框架中的JNI

**1、纽带**

JNI是连接框架层(Framework - C/C++) 和应用框架层(Application Framework - Java )的纽带

**2、JNI在Android中的作用**

JNI可以调用本地代码库(即C/C++代码)，并通过Dalvik虚拟机与应用层和应用框架层进行交互，Android 中的JNI主要位于应用层和应用框架层之间

> - 应用层：该层是由JNI开发，主要使用标准的JNI编程模型
> - 应用框架层： 使用的是Android中自定义的一套JNI编程模型，该自定义的JNI编程模型弥补了标准的JNI编程模型的不足

**3、NDK与JNI区别：**

> - NDK：NDK是Google开发的一套开发和编译工具集，主要用于AndroidJNI开发
> - JNI：JNI十套编程接口，用来实现Java代码与本地C/C++代码进行交互

## 四、JNI查找方式

Android系统在启动的过程中，先启动Kernel创建init进程，紧接着由init进程fork第一个横穿Java和C/C++的进程，即Zygote进程。Zygote启动过程会在 **AndroidRuntime.cpp** 中的startVM创建虚拟机，VM创建完成后，紧接着调用 **startReg** 完成虚拟机中的JNI方法注册。

### (一)、startReg

```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp      1440行
/*
 * Register android native functions with the VM.
 * 在虚拟机上注册Android的native方法
 */
 int AndroidRuntime::startReg(JNIEnv*env) {
    /*
     * This hook causes all future threads created in this process to be
     * attached to the JavaVM.  (This needs to go away in favor of JNI
     * Attach calls.)
     * 此钩子将导致在此过程中创建的所有未来线程 附加到JavaVM。 （这需要消除对
     * JNI的支持附加调用。）
     * 说白了就是设置线程的创建方法为 javaCreateThreadEtc
     */
        androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);
        ALOGV("--- registering native functions ---\n");
    /*
     * Every "register" function calls one or more things that return
     * a local reference (e.g. FindClass).  Because we haven't really
     * started the VM yet, they're all getting stored in the base frame
     * and never released.  Use Push/Pop to manage the storage.
     * 每个“注册”函数调用一个或多个返回的东西本地引用（例如FindClass）。
     * 因为我们没有真的启动虚拟机，它们都被存储在基本框架中并没有发布。 
     * 使用Push / Pop管理存储。
     */
        env -> PushLocalFrame(200);
        // 进程JNI注册函数
        if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
            env -> PopLocalFrame(NULL);
            return -1;
        }
        env -> PopLocalFrame(NULL);
        //createJavaThread("fubar", quickTest, (void*) "hello");
        return 0;
       }
```

startReg()函数里面调用了register_jni_procs()函数，这个函数是真正的注册，那我们就来看下这个register_jni_procs()函数

**1、register_jni_procs()**



```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp      1283行
    static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv*env) {
        for (size_t i = 0; i < count; i++) {
            if (array[i].mProc(env) < 0) {
            #ifndef NDEBUG
                ALOGD("----------!!! %s failed to load\n", array[i].mName)
            #endif
                return -1;
            }
        }
        return 0;
    }
```

> 发现上面的代码很简单，register_jni_procs(gRegJNI, NELEM(gRegJNI), env)函数的的作用就是循环调用gRegJNI数组成员所对应的方法

上面提到了gRegJNI数组，gRegJNI是什么，我们一起来看下

**2、gRegJNI数组**



```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp      1296行
static const RegJNIRec gRegJNI[] = {
   REG_JNI(register_com_android_internal_os_RuntimeInit),
    REG_JNI(register_android_os_SystemClock),
    REG_JNI(register_android_util_EventLog),
    REG_JNI(register_android_util_Log),
    REG_JNI(register_android_content_AssetManager),
    REG_JNI(register_android_content_StringBlock),
    REG_JNI(register_android_content_XmlBlock),
    REG_JNI(register_android_emoji_EmojiFactory),
    REG_JNI(register_android_text_AndroidCharacter),
    REG_JNI(register_android_text_StaticLayout),
    REG_JNI(register_android_text_AndroidBidi),
    REG_JNI(register_android_view_InputDevice),
    REG_JNI(register_android_view_KeyCharacterMap),
    REG_JNI(register_android_os_Process),
    REG_JNI(register_android_os_SystemProperties),
    REG_JNI(register_android_os_Binder),
    REG_JNI(register_android_os_Parcel),
    REG_JNI(register_android_nio_utils),
    REG_JNI(register_android_graphics_Graphics),
    REG_JNI(register_android_view_DisplayEventReceiver),
    REG_JNI(register_android_view_RenderNode),
    REG_JNI(register_android_view_RenderNodeAnimator),
    REG_JNI(register_android_view_GraphicBuffer),
    REG_JNI(register_android_view_DisplayListCanvas),
    REG_JNI(register_android_view_HardwareLayer),
    REG_JNI(register_android_view_ThreadedRenderer),
    REG_JNI(register_android_view_Surface),
    REG_JNI(register_android_view_SurfaceControl),
    REG_JNI(register_android_view_SurfaceSession),
    REG_JNI(register_android_view_TextureView),
    REG_JNI(register_com_android_internal_view_animation_NativeInterpolatorFactoryHelper),
    REG_JNI(register_com_google_android_gles_jni_EGLImpl),
    REG_JNI(register_com_google_android_gles_jni_GLImpl),
    REG_JNI(register_android_opengl_jni_EGL14),
    REG_JNI(register_android_opengl_jni_EGLExt),
    REG_JNI(register_android_opengl_jni_GLES10),
    REG_JNI(register_android_opengl_jni_GLES10Ext),
    REG_JNI(register_android_opengl_jni_GLES11),
    REG_JNI(register_android_opengl_jni_GLES11Ext),
    REG_JNI(register_android_opengl_jni_GLES20),
    REG_JNI(register_android_opengl_jni_GLES30),
    REG_JNI(register_android_opengl_jni_GLES31),
    REG_JNI(register_android_opengl_jni_GLES31Ext),

    REG_JNI(register_android_graphics_Bitmap),
    REG_JNI(register_android_graphics_BitmapFactory),
    REG_JNI(register_android_graphics_BitmapRegionDecoder),
    REG_JNI(register_android_graphics_Camera),
    REG_JNI(register_android_graphics_CreateJavaOutputStreamAdaptor),
    REG_JNI(register_android_graphics_Canvas),
    REG_JNI(register_android_graphics_CanvasProperty),
    REG_JNI(register_android_graphics_ColorFilter),
    REG_JNI(register_android_graphics_DrawFilter),
    REG_JNI(register_android_graphics_FontFamily),
    REG_JNI(register_android_graphics_Interpolator),
    REG_JNI(register_android_graphics_LayerRasterizer),
    REG_JNI(register_android_graphics_MaskFilter),
    REG_JNI(register_android_graphics_Matrix),
    REG_JNI(register_android_graphics_Movie),
    REG_JNI(register_android_graphics_NinePatch),
    REG_JNI(register_android_graphics_Paint),
    REG_JNI(register_android_graphics_Path),
    REG_JNI(register_android_graphics_PathMeasure),
    REG_JNI(register_android_graphics_PathEffect),
    REG_JNI(register_android_graphics_Picture),
    REG_JNI(register_android_graphics_PorterDuff),
    REG_JNI(register_android_graphics_Rasterizer),
    REG_JNI(register_android_graphics_Region),
    REG_JNI(register_android_graphics_Shader),
    REG_JNI(register_android_graphics_SurfaceTexture),
    REG_JNI(register_android_graphics_Typeface),
    REG_JNI(register_android_graphics_Xfermode),
    REG_JNI(register_android_graphics_YuvImage),
    REG_JNI(register_android_graphics_pdf_PdfDocument),
    REG_JNI(register_android_graphics_pdf_PdfEditor),
    REG_JNI(register_android_graphics_pdf_PdfRenderer),

    REG_JNI(register_android_database_CursorWindow),
    REG_JNI(register_android_database_SQLiteConnection),
    REG_JNI(register_android_database_SQLiteGlobal),
    REG_JNI(register_android_database_SQLiteDebug),
    REG_JNI(register_android_os_Debug),
    REG_JNI(register_android_os_FileObserver),
    REG_JNI(register_android_os_MessageQueue),
    REG_JNI(register_android_os_SELinux),
    REG_JNI(register_android_os_Trace),
    REG_JNI(register_android_os_UEventObserver),
    REG_JNI(register_android_net_LocalSocketImpl),
    REG_JNI(register_android_net_NetworkUtils),
    REG_JNI(register_android_net_TrafficStats),
    REG_JNI(register_android_os_MemoryFile),
    REG_JNI(register_com_android_internal_os_Zygote),
    REG_JNI(register_com_android_internal_util_VirtualRefBasePtr),
    REG_JNI(register_android_hardware_Camera),
    REG_JNI(register_android_hardware_camera2_CameraMetadata),
    REG_JNI(register_android_hardware_camera2_legacy_LegacyCameraDevice),
    REG_JNI(register_android_hardware_camera2_legacy_PerfMeasurement),
    REG_JNI(register_android_hardware_camera2_DngCreator),
    REG_JNI(register_android_hardware_Radio),
    REG_JNI(register_android_hardware_SensorManager),
    REG_JNI(register_android_hardware_SerialPort),
    REG_JNI(register_android_hardware_SoundTrigger),
    REG_JNI(register_android_hardware_UsbDevice),
    REG_JNI(register_android_hardware_UsbDeviceConnection),
    REG_JNI(register_android_hardware_UsbRequest),
    REG_JNI(register_android_hardware_location_ActivityRecognitionHardware),
    REG_JNI(register_android_media_AudioRecord),
    REG_JNI(register_android_media_AudioSystem),
    REG_JNI(register_android_media_AudioTrack),
    REG_JNI(register_android_media_JetPlayer),
    REG_JNI(register_android_media_RemoteDisplay),
    REG_JNI(register_android_media_ToneGenerator),

    REG_JNI(register_android_opengl_classes),
    REG_JNI(register_android_server_NetworkManagementSocketTagger),
    REG_JNI(register_android_ddm_DdmHandleNativeHeap),
    REG_JNI(register_android_backup_BackupDataInput),
    REG_JNI(register_android_backup_BackupDataOutput),
    REG_JNI(register_android_backup_FileBackupHelperBase),
    REG_JNI(register_android_backup_BackupHelperDispatcher),
    REG_JNI(register_android_app_backup_FullBackup),
    REG_JNI(register_android_app_ActivityThread),
    REG_JNI(register_android_app_NativeActivity),
    REG_JNI(register_android_view_InputChannel),
    REG_JNI(register_android_view_InputEventReceiver),
    REG_JNI(register_android_view_InputEventSender),
    REG_JNI(register_android_view_InputQueue),
    REG_JNI(register_android_view_KeyEvent),
    REG_JNI(register_android_view_MotionEvent),
    REG_JNI(register_android_view_PointerIcon),
    REG_JNI(register_android_view_VelocityTracker),

    REG_JNI(register_android_content_res_ObbScanner),
    REG_JNI(register_android_content_res_Configuration),

    REG_JNI(register_android_animation_PropertyValuesHolder),
    REG_JNI(register_com_android_internal_content_NativeLibraryHelper),
    REG_JNI(register_com_android_internal_net_NetworkStatsFactory),
};
```

gRegJNI数组，有138个成员变量，定义在**AndroidRuntime.cpp**中，该数组中每一个成员都代表一类文件的jni映射，其中REG_JNI是一个宏定义，让我们来看下

**3、REG_JNI 宏定义**



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

> 其中 **mProc**，就等价于调用其参数名所指向的函数，例如
>  **REG_JNI(register_com_android_internal_os_RuntimeInit).mProc** 也就是指进入 **register_com_android_internal_os_RuntimeInit**的方法，以此为例，看下面的代码



```cpp
int register_com_android_internal_os_RuntimeInit(JNIEnv* env)
{
    return jniRegisterNativeMethods(env, "com/android/internal/os/RuntimeInit",
        gMethods, NELEM(gMethods));
}

//gMethods：java层方法名与jni层的方法的一一映射关系
static JNINativeMethod gMethods[] = {
    { "nativeFinishInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeFinishInit },
    { "nativeZygoteInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeZygoteInit },
    { "nativeSetExitWithoutCleanup", "(Z)V",
        (void*) com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup },
};
```

所以REG_JNI就是一个宏定义，该宏的作用就是调用相应的方法。

### (二)、如何查找native方法

> 当大家在看framework层代码时，经常会看到native方法，这往往需要查看所对应的C++方法在那个文件，对应那个方法？下面从一个实例出发带大家如何查看Java层方法所对应的Native方法位置

#### 1、实例(一)

在后面分析Android消息机制源码，遇到MessageQueue.java中有多个native方法，比如：

```java
private native void nativePollOnce(long ptr, int timeoutMillis);
```

这样要怎么查找那？主要分为两个步骤

> - 第一步：MessageQueue.java的全限定名为android.os.MessageQueue.java。方法名为android.os.MessageQueue.nativePollOnce()，而相对应的native层方法名只是将点号替换为下划线，所以可得android_os_MessgaeQueue_nativePollOnce()。所以：nativePollOnce---->android_os_MessageQueue_nativePollOnce()
> - 第二步：有了对应的native方法，接下来就需要这个native方法在那个文件中，上面已经说了，Android系统启动的时候已经注册了大量的JNI方法。

在AndroidRumtime.cpp的gRegJNI数组。这些注册方法命名方式如下：



```css
register_[包名]_[类名]
```

那么MessageQueue.java所定义的JNI注册方法名应该是
 ** register_android_os_MessageQueue ** ，的确存在于 ** gRegJNI ** 数组，说明这次JNI注册过程是开机过程完成的。该方法是在 ** AndroidRuntime.cpp ** 申明为extern方法:



```cpp
extern int register_android_os_MessageQueue(JNIEnv* env);
```

这些extern方法绝大多数位于 ** /framework/base/core/jni ** 目录，大多数情况下 native命名方式为



```css
[包名]_[类名].cpp
[包名]_[类名].h
```

所以 MessageQueue.java--->android_os_MessageQueue.cpp。
 打开android_os_MessageQueue.cpp文件，搜索android_os_MessageQueue_nativePollOnce方法，这便找到了目标方法：



```cpp
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```

到这里完成了一次从Java层方法搜索到所对应的C++方法的过程。

#### 2、实例(二)

对于native文件命名方式，有时并非[包名]_[类名].cpp，比如Binder.java，Binder.java所对应的native文件：android_util_Binder.cpp。

比如：Binder.java的native方法，代码如下：



```java
public static final native int getCallingPid();
```

> 根据实例(一)的方式，找到getCallingPid--->android_os_Binder_getCallingPid()，并且在AndroidRumtime.cpp中的gRegJNI数组找到了register_android_os_Binder。按照实例(一)方式则native中的文件名为android_os_Binder.cpp，可是在/framwork/base/core/jni/目录下找不到该文件，这是例外的情况。其实真正的文件名为android_util_Binder.cpp，这就是例外，这一点有些费劲，不明白为何google打破之前的规律。



```rust
//frameworks/base/core/jni/android_util_Binder.cpp    761行
static jint android_os_Binder_getCallingPid(JNIEnv* env, jobject clazz)
{
    return IPCThreadState::self()->getCallingPid();
}
```

有人会问，以后遇到打破常规的文件命名的文件怎么办？其实很简单，首先，先尝试在/framework/base/core/jni/中搜索，对于Binder.java，可以直接搜索Binder关键字，其他类似。如果这里找不到，可以通过grep全局搜索android_os_Binder_getCallingPid()这个方法在那个文件上。

jni存在的常见目录：

> - /framework/base/core/jni
> - /framework/base/services/core/jni
> - /framework/base/media/jni

#### 3、实例(三)

前面两种都是在Android系统启动之初，便已经注册过JNI所对应的方法。那么如果程序自己定义的JNI方法，该如何查看JNI方法所在的位置？下面以MediaPlayer.java为例，其包名为android.media:



```java
public class MediaPlayer{
    static {
        System.loadLibrary("media_jni");
        native_init();
    }

    private static native final void native_init();
}
```

> 通过static 静态代码块中的System.loadLibrary()方法来加载动态库，库名为media_jni，Android平台则会自动扩展成所对应的libmedia_jni.so库。接着通过关键字native加载native_init方法之前，便可以在java层直接使用native层方法。

接下来便要查看libmedia_jni.so库定义所在文件，一般都是通过android.mk文件中定义的LOCAL_MODULE:= libmedia_jni，可以采用grep或者mgrep来搜索包含libmedia_jni字段的Android.mk所在路径。

搜索可知，libmedia_jni.so位于/framework/base/media/jni/Android.mk。用前面的实例(一)中的知识来查看相应的文件和方法分别为:



```css
android_media_MediaPlayer.cpp
android_media_MediaPlayer_native_init()
```

再然后，你会发现果然在该Android.mk所在目录/frameworks/base/media/jni中找到android_media_MediaPlayer.cpp文件。并在文件中存在相应的方法



```php
//frameworks/base/media/jni/android_media_MediaPlayer.cpp     820行
    // This function gets some field IDs, which in turn causes class initialization.
    // It is called from a static block in MediaPlayer, which won't run until the
    // first time an instance of this class is used.
    // 当类初始化的时候此函数获取一些字段ID。因为它是在MediaPlayer中的静态块中调用的，所以除非是第一次使用此类的实例，否则它将不会运行。
    static void android_media_MediaPlayer_native_init(JNIEnv *env)
    {
        jclass clazz;
        clazz = env->FindClass("android/media/MediaPlayer");
        if (clazz == NULL) {
            return;
        }
        fields.context = env->GetFieldID(clazz, "mNativeContext", "J");
        if (fields.context == NULL) {
            return;
        }
        fields.post_event = env->GetStaticMethodID(clazz, "postEventFromNative",
                "(Ljava/lang/Object;IIILjava/lang/Object;)V");
        if (fields.post_event == NULL) {
            return;
        }
        fields.surface_texture = env->GetFieldID(clazz, "mNativeSurfaceTexture", "J");
        if (fields.surface_texture == NULL) {
            return;
        }
        env->DeleteLocalRef(clazz);
        clazz = env->FindClass("android/net/ProxyInfo");
        if (clazz == NULL) {
            return;
        }
        fields.proxyConfigGetHost =
                env->GetMethodID(clazz, "getHost", "()Ljava/lang/String;");
        fields.proxyConfigGetPort =
                env->GetMethodID(clazz, "getPort", "()I");
        fields.proxyConfigGetExclusionList =
                env->GetMethodID(clazz, "getExclusionListAsString", "()Ljava/lang/String;");
        env->DeleteLocalRef(clazz);
        gPlaybackParamsFields.init(env);
        gSyncParamsFields.init(env);
    }
```

所以 MediaPlayer.java中的native_init()方法位于/frameworks/base/media/jni/目录下的android_media_MediaPlayer.cpp文件中的android_media_MediaPlayer_native_init方法。

### (三)、总结

> JNI作为连接Java世界和C/C++世界的桥梁，很有必要掌握。看完本文后，一定要掌握在分析Android源码过程汇总如何查找native方法。首先明白native方法名和文件名的命名规律，其次要懂得该如何去搜索代码。JNI方式注册无非是Android系统启动中Zygote注册以及通过System.loadLibrary方式注册，对于系统启动过程注册的，可以通过查询AndroidRuntime.cpp中的gRegJNI是否存在对应的register方法，如果不存在，则大多数通过LoadLibrary方式来注册。

## 五、loadLibrary源码分析

> 再来进一步分析，Java层与native层方法是如何注册并映射的，继续以MediaPlayer为例，进一步分析。

在MediaPlayer.java中调用System.loadLibrary("media_jni"),把libmedia_jni.so动态库加载到内存。接下来以loadLibrary为起点展开JNI注册流程的过程分析。

### (一) loadLibrary() 流程

**1、loadLibrary()方法**

```dart
//libcore/luni/src/main/java/java/lang/System.java     1075行
    /**
     * Loads the system library specified by the <code>libname</code>
     * argument. The manner in which a library name is mapped to the
     * actual system library is system dependent.
     *
     * 加载由 libname 参数指定的系统库， library库名是通过系统依赖映射到实际系统库的。
     *
     * <p>
     * The call <code>System.loadLibrary(name)</code> is effectively
     * equivalent to the call
     * <blockquote><pre>
     * Runtime.getRuntime().loadLibrary(name)
     * </pre></blockquote>
     *
     * 调用 System.loadLibrary(name)实际上等价于调用
     * Runtime.getRuntime().loadLibrary（name）
     *
     * @param      libname   the name of the library.      lib库的名字
     * @exception  SecurityException  if a security manager exists and its
     *             <code>checkLink</code> method doesn't allow
     *             loading of the specified dynamic library
     * 如果存在安全管理员，并且其 checkLink 方法不允许 加载指定的动态库，则会抛出SecurityException
     * @exception  UnsatisfiedLinkError  if the library does not exist.
     * 如果库不存在则抛出UnsatisfiedLinkError
     * @exception  NullPointerException if <code>libname</code> is
     *             <code>null</code>
     * 如果libname是null则抛出NullPointerException
     * @see        java.lang.Runtime#loadLibrary(java.lang.String)
     * @see        java.lang.SecurityManager#checkLink(java.lang.String)
     */
    public static void loadLibrary(String libname) {
         Runtime.getRuntime().loadLibrary(libName, VMStack.getCallingClassLoader());
    }
```

通过代码和上面的注释，我们知道了，loadLibrary(String)其本质是调用了 Runtime.getRuntime().loadLibrary(String,ClassLoader)
 那我们就来跟踪一下

**2、Runtime.getRuntime().loadLibrary(String,ClassLoader)方法**

```dart
     /*
      * Searches for and loads the given shared library using the given ClassLoader.
      * 使用指定的ClassLoader搜索并加载给定的共享库
      */
    void loadLibrary(String libraryName, ClassLoader loader) {
         //如果load不为null 则进入该分支 
        if (loader != null) {
            //查找库所在的路径
            String filename = loader.findLibrary(libraryName);
            // 如果路径为null则说明找不到库
            if (filename == null) {
                // It's not necessarily true that the ClassLoader used
                // System.mapLibraryName, but the default setup does, and it's
                // misleading to say we didn't find "libMyLibrary.so" when we
                // actually searched for "liblibMyLibrary.so.so".
                // 当我们搜索liblibMyLibrary.so.so的时候，可能会提示我们没有找
                // 到"ibMyLibrary.so"。是因为ClassClassLoader不一定使用System，
                 // 但是默认设置又是这样的，所以会有一定的误导性
                throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                        System.mapLibraryName(libraryName) + "\"");
            }
            //找到路径，则加载库
            String error = doLoad(filename, loader);
            if (error != null) {
                throw new UnsatisfiedLinkError(error);
            }
            return;
        }
        // 如果loader为null，则执行下面的代码
        // 其中 System.mapLibraryName(String) 是"根据库名返回本地的库"
        String filename = System.mapLibraryName(libraryName);
        List<String> candidates = new ArrayList<String>();
        String lastError = null;
        for (String directory : mLibPaths) {
            String candidate = directory + filename;
            candidates.add(candidate);
            if (IoUtils.canOpenReadOnly(candidate)) {
                //加载库
                String error = doLoad(candidate, loader);
                if (error == null) {
                    return; // We successfully loaded the library. Job done.
                }
                lastError = error;
            }
        }
        if (lastError != null) {
            throw new UnsatisfiedLinkError(lastError);
        }
        throw new UnsatisfiedLinkError("Library " + libraryName + " not found; tried " + candidates);
    }
```

通过上面的代码，我们知道，无论loader是否为null，最后都是通过doLoad(String, ClassLoader)来真正的加载。那我们来一起看一下

**3、doLoad(String name, ClassLoader loader) 方法**

```tsx
    //libcore/luni/src/main/java/java/lang/Runtime.java            401行
    private String doLoad(String name, ClassLoader loader) {
        // Android apps are forked from the zygote, so they can't have a custom LD_LIBRARY_PATH,
        // which means that by default an app's shared library directory isn't on LD_LIBRARY_PATH.
       // Android应用程序从zygote分支fork出来的，所以他们无法自定义
       // LD_LIBRARY_PATH，这意味着默认情况下，应用程序的共享库目录不在
       // LD_LIBRARY_PATH上。

        // The PathClassLoader set up by frameworks/base knows the appropriate path, so we can load
        // libraries with no dependencies just fine, but an app that has multiple libraries that
        // depend on each other needed to load them in most-dependent-first order.

        // We added API to Android's dynamic linker so we can update the library path used for
        // the currently-running process. We pull the desired path out of the ClassLoader here
        // and pass it to nativeLoad so that it can call the private dynamic linker API.

        // We didn't just change frameworks/base to update the LD_LIBRARY_PATH once at the
        // beginning because multiple apks can run in the same process and third party code can
        // use its own BaseDexClassLoader.

        // We didn't just add a dlopen_with_custom_LD_LIBRARY_PATH call because we wanted any
        // dlopen(3) calls made from a .so's JNI_OnLoad to work too.

        // So, find out what the native library search path is for the ClassLoader in question...
        // 由framework / base设置的PathClassLoader知道适具体的路径，因此我们可以
        // 加载没有依赖关系的库，但是具有依赖于彼此的多个库的应用程序需要以大多
        // 数依赖的顺序加载它们。 
        // 为了让我们可以在正在运行的进程中更新库路径，所以我们向Android的动态链
        // 接器添加了API。
        // 我们将所需的路径从ClassLoader中拉出，并将其传递给nativeLoad，便可以  
        // 调用私有动态链接器API。
        // 我们不仅仅是更改框架/基础来更新LD_LIBRARY_PATH一次，因为多个apk
        // 可以在同一进程中运行，第三方代码可以使用自己的BaseDexClassLoader。
        // 我们没有添加dlopen_with_custom_LD_LIBRARY_PATH调用，因为我们希望
        // 使用.so的JNI_OnLoad进行的任何dlopen（3）调用也可以工作。
        //因此，找出本机库搜索路径对于有问题的ClassLoader

        String ldLibraryPath = null;
        String dexPath = null;
        if (loader == null) {
            // We use the given library path for the boot class loader. This is the path
            // also used in loadLibraryName if loader is null.
            ldLibraryPath = System.getProperty("java.library.path");
        } else if (loader instanceof BaseDexClassLoader) {
            BaseDexClassLoader dexClassLoader = (BaseDexClassLoader) loader;
            ldLibraryPath = dexClassLoader.getLdLibraryPath();
        }
        // nativeLoad should be synchronized so there's only one LD_LIBRARY_PATH in use regardless
        // of how many ClassLoaders are in the system, but dalvik doesn't support synchronized
        // internal natives.
        // nativeLoad应该同步，所以使用中只有一个LD_LIBRARY_PATH，无论系统中有多少个ClassLoaders，dalvik不支持native的同步。
        synchronized (this) {
            return nativeLoad(name, loader, ldLibraryPath);
        }
    }
```

nativeLoad()这是一个native方法，再进去ART虚拟机java_lang_Runtime.cc，再细讲就要深入剖析虚拟机内部，这里就不往下深入了。后续有时间单独讲解下虚拟机。这里直接说结论

> - 调用dlopen()函数，打开一个so文件并创建一个handle;
> - 调用dlsym()函数，查看相应so文件的JNIOnLoad()函数指针，并执行相应函数。

**4、总结**

所以说，System.loadLibrary()的作用就是调用相应库中的JNI_OnLoad()方法。那我们就来看下JNI_OnLoad()过程

### (二) JNI_OnLoad流程

**1、JNI_OnLoad()**

```cpp
jint JNI_OnLoad(JavaVM* vm, void* reserved)
{
    JNIEnv* env = NULL;
    //frameworks/base/media/jni/android_media_MediaPlayer.cpp    1132行
    // 注册JNI
    if (register_android_media_MediaPlayer(env) < 0) {
        goto bail;
    }
    ...
}
```

这里面主要通过调用register_android_media_MediaPlayer()函数来进行注册的。

**2、register_android_media_MediaPlayer()**

```cpp
//frameworks/base/media/jni/android_media_MediaPlayer.cpp 1086行
// This function only registers the native method
// 这个函数仅仅是用来注册native方法的
static int register_android_media_MediaPlayer(JNIEnv *env)
{
    return AndroidRuntime::registerNativeMethods(env,
                "android/media/MediaPlayer", gMethods, NELEM(gMethods));
}
```

我们看到register_android_media_MediaPlayer()函数里面，实际上是调用的AndroidRuntime的registerNativeMethods()函数。

在看AndroidRuntime的registerNativeMethods()函数之前，先说下gMethods

**3、gMethods**

```cpp
//frameworks/base/media/jni/android_media_MediaPlayer.cpp    1036行
static JNINativeMethod gMethods[] = {
    {
        "nativeSetDataSource",
        "(Landroid/os/IBinder;Ljava/lang/String;[Ljava/lang/String;"
        "[Ljava/lang/String;)V",
        (void *)android_media_MediaPlayer_setDataSourceAndHeaders
            
    },
    {"_setDataSource",      "(Ljava/io/FileDescriptor;JJ)V",    (void *)android_media_MediaPlayer_setDataSourceFD},
    {"_setDataSource",      "(Landroid/media/MediaDataSource;)V",(void *)android_media_MediaPlayer_setDataSourceCallback },
    {"_setVideoSurface",    "(Landroid/view/Surface;)V",        (void *)android_media_MediaPlayer_setVideoSurface},
    {"_prepare",            "()V",                              (void *)android_media_MediaPlayer_prepare},
    {"prepareAsync",        "()V",                              (void *)android_media_MediaPlayer_prepareAsync},
    {"_start",              "()V",                              (void *)android_media_MediaPlayer_start},
    {"_stop",               "()V",                              (void *)android_media_MediaPlayer_stop},
    {"getVideoWidth",       "()I",                              (void *)android_media_MediaPlayer_getVideoWidth},
    {"getVideoHeight",      "()I",                              (void *)android_media_MediaPlayer_getVideoHeight},
    {"setPlaybackParams", "(Landroid/media/PlaybackParams;)V", (void *)android_media_MediaPlayer_setPlaybackParams},
    {"getPlaybackParams", "()Landroid/media/PlaybackParams;", (void *)android_media_MediaPlayer_getPlaybackParams},
    {"setSyncParams",     "(Landroid/media/SyncParams;)V",  (void *)android_media_MediaPlayer_setSyncParams},
    {"getSyncParams",     "()Landroid/media/SyncParams;",   (void *)android_media_MediaPlayer_getSyncParams},
    {"seekTo",              "(I)V",                             (void *)android_media_MediaPlayer_seekTo},
    {"_pause",              "()V",                              (void *)android_media_MediaPlayer_pause},
    {"isPlaying",           "()Z",                              (void *)android_media_MediaPlayer_isPlaying},
    {"getCurrentPosition",  "()I",                              (void *)android_media_MediaPlayer_getCurrentPosition},
    {"getDuration",         "()I",                              (void *)android_media_MediaPlayer_getDuration},
    {"_release",            "()V",                              (void *)android_media_MediaPlayer_release},
    {"_reset",              "()V",                              (void *)android_media_MediaPlayer_reset},
    {"_setAudioStreamType", "(I)V",                             (void *)android_media_MediaPlayer_setAudioStreamType},
    {"_getAudioStreamType", "()I",                              (void *)android_media_MediaPlayer_getAudioStreamType},
    {"setParameter",        "(ILandroid/os/Parcel;)Z",          (void *)android_media_MediaPlayer_setParameter},
    {"setLooping",          "(Z)V",                             (void *)android_media_MediaPlayer_setLooping},
    {"isLooping",           "()Z",                              (void *)android_media_MediaPlayer_isLooping},
    {"_setVolume",          "(FF)V",                            (void *)android_media_MediaPlayer_setVolume},
    {"native_invoke",       "(Landroid/os/Parcel;Landroid/os/Parcel;)I",(void *)android_media_MediaPlayer_invoke},
    {"native_setMetadataFilter", "(Landroid/os/Parcel;)I",      (void *)android_media_MediaPlayer_setMetadataFilter},
    {"native_getMetadata", "(ZZLandroid/os/Parcel;)Z",          (void *)android_media_MediaPlayer_getMetadata},
    {"native_init",         "()V",                              (void *)android_media_MediaPlayer_native_init},
    {"native_setup",        "(Ljava/lang/Object;)V",            (void *)android_media_MediaPlayer_native_setup},
    {"native_finalize",     "()V",                              (void *)android_media_MediaPlayer_native_finalize},
    {"getAudioSessionId",   "()I",                              (void *)android_media_MediaPlayer_get_audio_session_id},
    {"setAudioSessionId",   "(I)V",                             (void *)android_media_MediaPlayer_set_audio_session_id},
    {"_setAuxEffectSendLevel", "(F)V",                          (void *)android_media_MediaPlayer_setAuxEffectSendLevel},
    {"attachAuxEffect",     "(I)V",                             (void *)android_media_MediaPlayer_attachAuxEffect},
    {"native_pullBatteryData", "(Landroid/os/Parcel;)I",        (void *)android_media_MediaPlayer_pullBatteryData},
    {"native_setRetransmitEndpoint", "(Ljava/lang/String;I)I",  (void *)android_media_MediaPlayer_setRetransmitEndpoint},
    {"setNextMediaPlayer",  "(Landroid/media/MediaPlayer;)V",   (void *)android_media_MediaPlayer_setNextMediaPlayer},
};
```

gMethods，记录java层和C/C++层方法的一一映射关系。这里涉及到结构体JNINativeMethod，其定义在jni.h文件：



```cpp
/、libnativehelper/include/nativehelper/jni.h    129行
typedef struct {
    const char* name;  //Java层native函数名
    const char* signature;   //Java函数签名，记录参数类型和个数，以及返回值类型
    void*       fnPtr;  //Native层对应的函数指针
} JNINativeMethod;
```

下面让我们看一下AndroidRuntime的registerNativeMethods()函数

**4、AndroidRuntime的registerNativeMethods()函数**

```cpp
//frameworks/base/core/jni/AndroidRuntime.cpp  262行
/*
 * Register native methods using JNI.
 * 使用JNI注册native方法
 */
int AndroidRuntime::registerNativeMethods(JNIEnv* env,
    const char* className, const JNINativeMethod* gMethods, int numMethods)
{
    return jniRegisterNativeMethods(env, className, gMethods, numMethods);
}
```

jniRegisterNativeMethods该方法是由Android JNI帮助类JNIHelp.cpp来完成。

**5、jniRegisterNativeMethods()函数**

```cpp
//libnativehelper/JNIHelp.cpp     73行
extern "C" int jniRegisterNativeMethods(C_JNIEnv* env, const char* className,
    const JNINativeMethod* gMethods, int numMethods)
{
    JNIEnv* e = reinterpret_cast<JNIEnv*>(env);

    ALOGV("Registering %s's %d native methods...", className, numMethods);
    scoped_local_ref<jclass> c(env, findClass(env, className));
    //找不到native注册方法
    if (c.get() == NULL) {
        char* msg;
        asprintf(&msg, "Native registration unable to find class '%s'; aborting...", className);
        e->FatalError(msg);
    }
     //调用JNIEnv结构体的成员变量
    if ((*env)->RegisterNatives(e, c.get(), gMethods, numMethods) < 0) {
        //如果native方法注册失败
        char* msg;
        asprintf(&msg, "RegisterNatives failed for '%s'; aborting...", className);
        e->FatalError(msg);
    }

    return 0;
}
```

通过上面的代码我们发现jniRegisterNativeMethods()内部真正的注册函数是RegisterNatives()函数，那我们继续跟踪

**6、RegisterNatives()函数**

```kotlin
// libnativehelper/include/nativehelper/jni.h         976行
    jint RegisterNatives(jclass clazz, const JNINativeMethod* methods,
        jint nMethods)
    { return functions->RegisterNatives(this, clazz, methods, nMethods); }
}
```

其中functions是指向JNINativeInterface结构体指针，也就是将调用下面的方法:

```cpp
struct JNINativeInterface {
    jint (*RegisterNatives)(JNIEnv*, jclass, const JNINativeMethod*,jint);
}
```

再往下深入就到了虚拟机内部了，就不在深入了。

### (三) 总结

总之，这个过程完成了gMethods数组中的方法的映射关系，比如java层的native_init()方法，映射到native层的android_media_MediaPlayer_native_init()方法。

虚拟机相关的变量中有两个非常重要的变量JavaVM和JNIEnv:

> - JavaVM：是指进程虚拟机环境，每个进程有且只有一个JavaVM实例。
> - JNIEnv：是指线程上下文环境，每个线程有且只有一个JNIEnv实例。

## 六、JNI资源

JNINativeMethod结构体中有一个字段为signature(签名)，在介绍signature格式之前需要掌握各种数据类型在Java层、Native层以及签名所采用的签名格式。

### (一)、数据类型

**1、基本数据类型**

| Signature | Java    | Native   |
| --------- | ------- | -------- |
| B         | byte    | jbyte    |
| C         | char    | jchar    |
| D         | double  | jdouble  |
| F         | float   | jfloat   |
| I         | int     | jint     |
| S         | short   | jshort   |
| J         | long    | jlong    |
| Z         | boolean | jboolean |
| V         | void    | void     |

**2、数组数据类型**

数组简称则是在前面添加" **[ ** " ：

| Signature格式 | Java      | Native        |
| ------------- | --------- | ------------- |
| [B            | byte[]    | jbyteArray    |
| [C            | char[]    | jcharArray    |
| [D            | double[]  | jdoubleArray  |
| [F            | float[]   | jfloatArray   |
| [I            | int[]     | jintArray     |
| [S            | shor[]    | jshortArray   |
| [ J           | long[]    | jlongArray    |
| [ Z           | boolean[] | jbooleanArray |

**3、复杂数据类型**

对象类型简称：L+ ** classname **+;

| Signature格式        | Java      | Native       |
| -------------------- | --------- | ------------ |
| Ljava/lang/String    | String    | jstring      |
| L+class+ ;           | 所有对象  | jobject      |
| [ L + classname + ;  | Object[]  | jobjectArray |
| Ljava.lang.Class     | Class     | jclass       |
| Ljava.lang.Throwable | Throwable | jthrowable   |

**4、Signature**

有了前面的讲解，我们通过案例来说说函数签名:**  (入参) 返回值参数 **，这里用到的便是前面介绍的Signature格式

| Java格式                        | 对应的签名                |
| ------------------------------- | ------------------------- |
| void foo()                      | ()V                       |
| float foo(int i)                | (I) F                     |
| long foo(int[] i)               | ([ i) J                   |
| double foo (Class c)            | (Ljava/lang/Class;) D     |
| boolean foo (int [] i,String s) | ([ILjava/lang/String ;) Z |
| String foo(int i)               | (I) Ljava/lang/String ;   |

### (二)、其他注意事项

**1、垃圾回收**

> 对于Java而言，开发者是无需关心垃圾回收的，因为这完全由虚拟机GC来负责垃圾回收，而对于JNI开发人员，由于内存释放需要谨慎处理，需要的时候申请，使用完后记得释放内存，以免发生内存泄露。

所以JNI提供了了三种Reference类型:

> - Local Reference(本地引用)
> - Global Reference(全局引用)
> - Weak Global Reference (全局弱引用)

其中Global Reference 如果不主动释放，则一直不会释放；对于其他两个类型的引用都是释放的可能性，那是不是意味着不需要手动释放? 答案是否定的，不管这三种类型的那种引用，都尽可能在某个内存不需要时，立即释放，这对系统更为安全可靠，以减少不可预知的性能与稳定性问题。

**另外，ART虚拟机在GC算法有所优化，为了减少内存碎片化问题，在GC之后有可能会移动对象内存的位置，对于Java层程序并没有影响，但是对于JNI程序要注意了，对于通过指针来直接访问内存对象时，Dalvik能正确运行的程序，ART下未必能正常运行。**

**2、异常处理**

> Java层出现异常，虚拟机会直接抛出异常，这是需要try... catch 或者继续向外throw。但是对于JNI出现异常时，即执行到JNIEnv 中某个函数异常时，并不会立即抛出异常来中断程序的执行，还可以继续执行内存之类的清理工作，知道返回Java层才会抛出相应的异常。

另外，Dalvik虚拟机有些情况下JNI函数出错可能会返回NULL，但ATR虚拟机在出错时更多是抛出异常。这样导致的问题就可能是在Dalvik版本能正常运行的成员，在ART虚拟机上并没正确处理异常而崩溃。

## 七、总结

本文主要是通过实例，基于Android 6.0源码分析 JNI原理，讲述JNI核心功能：

> - 介绍了JNI的概念及如何查找JNI方法，让大家明白如何从Java层跳转到Native层
> - 分了JNI函数注册流程，进异步加深对JNI的理解
> - 列举Java与Native一级函数签名方式。





# Android跨进程通信IPC之AndroidIPC基础1

进程Process 是程序的一个运行实例，以区别于"程序" 这一个静态概念；而线程(Thread) 则是CPU调度的单位。当前大部分操作系统都是支持多任务运行，这一特性让用户感到计算机好像可以同时处理很多事情。显然在只有一个CPU核心的情况下，这种"同时"是一种假象。它是操作系统先采用分时的方法，为正在运行的多任务分配合理的、单独的CPU来实现的。比如当前系统有5个任务，如果采用"平均分配"法且时间为10ms的话，那么各个任务每隔40ms就能被执行一次。只要机器速度足够快，用户的感觉就是所有任务都在同步运行。

## 一、Android IPC简介

**(一) 什么是IPC**

> IPC是Inter-Process Communication的缩写，含义为进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。

这里我们不得不提下什么是进程，什么事线程，进程和线程是截然不同的歹念。按照操作系统中的描述，线程是CPU调度的最小单元，同时线程是一种有限的系统资源。而进程一般指一个执行单元，在PC和移动设备上指一个程序或者一个应用。一个进程可以包含多个线程，因此进程和线程是包含与被包含的关系。最简单的情况下，一个进中可以只有一个线程，即主线程，在Android里面主线程也叫UI线程，在UI线程里面才能操作界面。很多时候，一个进程中需要执行大量耗时任务，如果这些任务放到主线程中去执行就会造成界面无法响应，严重影响用户体验，这种情况在PC系统和移动系统中都存在，在Android中就叫ANR(Application Not Responding)，即应用无响应。解决这个问题就需要用到线程，把一些耗时的任务放在线程即可。

> IPC不是Android独有的，任何一个操作系统都需要响应的IPC机制，比如Windows上可以通过剪切板、管道等来进行进程间通信；Linux可以通过命名管道、共享内存、信号量等来进行进程间通信。可以看到不同的操作系统平台有着不同的进程间通信方式，对于Android来说，它是一种基于Linux内核的移动操作系统，它的进程间通信方式并不完全继承自Linux，相反它有自己的进程间通信方式。在Android中最有特色的进程间通信方式就是Binder了，通过Binder可以轻松地实现进程间通信。除了Binder，Android还支持Socket，通过Socket也可以实现任意两个终端之间的通信，当然同一个设备上的两个进程通过Socket自然也可以实现。

说到IPC的使用场景就必须提到多线程，只有面对多进程这种场景下，才需要考虑进程间通信。多进程的场景一般分为两种

> - 1、一个app因为某些原因子什么需要采用多进程模式来实现的，至于原因可能很多，比如为了加大一个应用可使用的内存所以需要多进程来获取更多份的内存空间。Android对单个应用的内存做了最大限制，早期有16M，后面也有64M，不同设备有不同的大小。
> - 2、一个app需要其他app提供的数据，由于是两个app，所以必须采用跨进程的方式来获取数据，甚至我们可以通过系统的ContentPorvider去查询数据的时候，其实也是一种跨进程通信方式，只不过通信的细节被系统内部屏蔽了，我们无法感知而已。后面会有单独的文章介绍ContentProvider。总之，不管由于何种原因，我么采用了多进程的设计方法，那么应用中就必须妥善地处理进程间通信的各种问题。

## 二、Android 多进程模式

我们先来了解下Android中的多进程模式。通过给四大组件指定**android:process** 属性，我们可以轻易地开启多进程模式，这看起来很简单，但是实际上使用中却问题很多，多进程远远没有我们想的那么简单，有些时候我们多进程中遇到的“坑”远远大于使用多进程带来的"好处"。那我们就来详细了解下

### (一)、如何开启多进程模式

> 正常情况下，在Android中多进程是指一个应用中存在多个进程的情况下，因此这里不讨论两个应用之间的多进程情况。首先，在Android中使用多进程只有一种方式，那就是给四大组件(Activity、Service、Receiver、ContentProvider)在AndroidMenifest中添加 **"android:process"** 属性，除此之外没有其他方法，也就是说我们无法给一个线程或者一个实体类指定其运行时所在的进程。其实还有一种非常规的多进程方法，那就是通过JNI在native层去fork一个新的进程(5.0之前，我们 **进程保活**常用的 伎俩)，但是这种方法比较特殊，也不是常用的创建多进程方式，因此我们暂时不考虑这种方式。

举例如下：



```xml
        <activity
            android:name="com.test.process.demo.TestActivity1"
            android:process=":process_test1"
            android:screenOrientation="portrait" />

        <activity
            android:name="com.test.process.demo.TestActivity2"
            android:process="com.test.demo.process_test2"
            android:screenOrientation="portrait" />
```

> 上面的两个案例分别为TestActivity1和TestActivity2设置了 **"android:process"** 属性。所以这意味着当前应用增加了两个新的进程，假设当前包名为："com.test.process.demo",当TestActivity1被启动的时候会，系统会为它创建一个单独的进程，进程名为"com.test.process.demo.process_test1"，当TestActivity2启动的时候，会为它创建一个单独的进程，进程名为"com.test.demo.process_test2"。如果入口Activity是MainActivity，没有为它设置"  android:process"所以默认它的进程名是包名。

大家仔细看代码会发现TestActivity1和TestActivity2在设置**"android:process"** 并不一样，那么这两种方式有区别吗？其实是有区别的，首先 **":" ** 代表的是 要在当前进程名附加上当前的包名，这是一种简写的方式，对TestActivity1他的进程名为"com.test.process.demo.process_test1"。对TestActivity2来说，它的声明方式是一种完整的命名方式，不会附加包的信息；其次，进程名以":"开头的进程属于当前应用的私有进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。

> 我们知道Android系统会为每一个应用分配一个唯一的UID，具有相同的UID应用才能共享数据。这里要说明的是，两个应用通过ShareUID跑在同一个进程中是有要求的，需要这两个应用具有相同的ShareUID并且签名相同才可以。在这种情况下，它们可以相互访问对它们跑在同一个进程中，那么除了能共享data目录、组件信息，还可以共享内存数据，或者说它们看来就像是一个应用的两个部分。

### (二)、多进程模式的运行机制

如果大家之前没有用过多进程模式，如果第一次运行多进程模式，会有很多"坑"。而且这些"坑"，打死大家也想不到。一般问题如下

> - 1、静态成员和单利模式完全失效
> - 2、线程同步机制完全失效
> - 3、SharedPreferences的可靠性下降
> - 4、Application会多次创建

咱们来简单说下

**1、静态成员和单利模式完全失效**

我们来简单说下静态成员，比如在平时开发中我们会切换环境，开发环境和测试环境，正式环境中相互切换。代码入下：



```cpp
public class URLConstant {
     // 1是开发环境，2是测试环境，3是正式环境
     public static String EnvironmentType = 1;  
}
```

假设默认是开发环境，我们在切换界面把环境变更为测试环境，在代码中把EnvironmentType设为2。这时候我们在TestActivity1里面获取EnvironmentType，我们会发现EnvironmentType仍然为1，而不是2。为什么会这样？

> 出现上面的问题是TestActivity1运行在一个单独的进程里面，我们知道Android系统为每一个应用分配了一个独立的虚拟机，或者说为每个进程都分配了一个独立的虚拟机，不同的虚拟机在内存上有不同的地址空间，这就导致在不同的虚拟机中访问同一个类对象会产生多分副本，我们上门的例子，进程com.test.process.demo.process_test1和进程com.test.demo.process_test2都存在一个类是URLConstant，且这两个类不相互影响，所以这就解释了为什么我们在切换界面修改了EnvironmentType，但是在TestActivity1里面取到的值还是1，因为TestActivity1里面的值还没有变化。

**2、线程同步机制完全失效**

> 这个问题本质和上一个问题类似，既然不是一块内存了，那么不管是所对象还是锁全局类都无法保证同步，因为不同进程锁的不是同一个对象。所以同步机制一定会出问题。

**3、SharedPreferences的可靠性下降**

> 是因为SharePreference不支持两个进程同时去执行写操作，否则会导致一定几率的数据丢失，这是因为SharedPreferences底层是通过读/写XML文件来实现的，并发写显然会出问题的，甚至并发读/写都有可能出问题。

**4、Application会多次创建**

> 大家知道，当一个组件跑在一个新的进程中，由于系统要再创建新的进程同时分配独立的虚拟机，所以这个过程其实就是启动一个应用的过程。因此，相当于系统又把这个应用重新启动了一遍，既然重新启动了，那么自认会创建新的Applicatiaon。

## 三、Serializable和Parcelable接口

本节主要讲解三方面的内容Serializable接口和Parcelable接口以及Binder，只有熟悉这这两个接口后，我们才能在后面更好地理解跨进程通信。Serializable和Parcelable接口可以完成对象的序列化的过程，当我们需要通过Intent和Binder传输数据时就需要使用Parcelable或者Serializable，有时候我们还需要把对象持久化到存储设备上或者通过网络传输给其他客户端，这个时候也需要使用Seriazable来完成对象的持久化

### (一) Serializable接口

**1、Serializable简介**

Serializable 是Java所提供的一个序列化接口，它是一个空接口，为对象提供标准的序列化和反序列化操作。使用Serializable来实现序列化相当简单，只需要类在声明中指定一个类似下面的标示即可实现默认的序列化过程。

```java
private static final long seriaVersionUID=1L;
```

所以如果想让一个类实现序列化，只需要将这个类实现Serializable接口，并声明一个seriaVersionUID即可，实际上，甚至这个seriaVersionUID也不是必需的，我们不声明这个serialVersionUID即可，实际上，甚至这个seriaVersionUID也不是必需的，我们不声明这个serialVersionUID，同样也可以实现反序列化，但是这将会对反序列化过程产生影响，具体影响我们后面介绍

**2、Serializable序列化和反序列化**

我们举一个例子吧，Person类是一个实现了Serializable接口的类，它有3个字段，name，sex,age

```java
public class Persion implements Serializable{
  private static final  long serialVersionUID=1L;
  public String name;
  public String sex;
  public int age;

  public Persion(String name,String sex,int age){
         this.name=name;
         this.sex=sex;
         this.age=age;
  }
}
```

通过Serializable方式来实现对象的序列化，实现起来非常简单，几乎所有工作都被系统自动完成了。如何进程对象的序列化和反序列化也非常简单，只需要采用ObjectOutputStream和ObjectInputStream即可轻松实现。代码如下：

```csharp
//序列化过程
Person person=new Person("张三","男",23);
ObjectOutputStream out=new ObjectOutputStream(new FileOutStream("cache.txt"));
out.writeObject(person);
out.close();

//反序列化过程
ObjectInputStream in=new ObjectInputStream(new FileInputStream("cache.txt"));
Person newPerson=(Person)in.readObejct();
in.close();
```

上面的代码演示了采用Serializable方式序列化对象的典型过程，很简单，只需要把实现了Serializable接口的User对象写到文件中就可以快速恢复了，恢复后的对象newPerson和person内容完全一样，但是两者并不是同一个对象。

**3、serialVersionUID的作用**

即使不指定serialVersionUID也可以实现序列化，那到底要不要指定呢？serialVersionUID后面的数字有什么含义？

> - 这个serialVersionUID是用来辅助序列化和反序列化的过程。原则上序列化后的数据中的serialVersionUID只有和当前类的serialVersionUID一致才能成功的反序列化。
> - serialVersionUID的详细工作机制是这样的：序列化的时候系统会把当前类的serialVersionUID写入序列化的文件中(也可能是其他中介)，当反序列化的时候系统会去检测文件中的serialVersionUID，看它是否和当前类的serialVersionUID一致，如果一致就说明序列化的类的版本和当前类的版本是相同的，这个时候可以成功反序列化；否则就说明当前类和序列化的类相比发生了某些变换，比如成员变量的数量、类型可能会发生变化，这时候就无法正常的反序列化。会报如下错误:



```css
java.io.InvalidClassException
```

所以一般来说，我们应该手动去指定serialVersionUID的值，比如"1L",也可以让IDE根据当前类的结构去生成对应的hash值，这样序列化和反序列化时两者的serialVersionUID是相同的，因此可以正常的进程反序列化。如果不不设置serialVersionUID，系统在序列化的时候默认会根据类的结构在生成对应的serialVersionUID，在反序列化的时候，如果当天类有变化，比如增加或者减少字段，这时候当前的类的serialVersionUID和序列化的时候的serialVersionUID就不一样了，就会出现反序列化失败，如果没有捕获异常会导致crash

**所以当我们手动制订了它之后，就可以很大程度上避免了泛学历化过程的失败。**

比如当版本升级以后，我们可能删除了某个成员变量也可能增加一些新的成员变量，这个时候我们的反序列化过程仍然可以成功，程序仍然能够最大限度地恢复数据。相反 如果我们没有指定serialVersionUID的话，程序就会挂掉。当然我们也要考虑到另外一种情况，如果类结构发生了给常规性的改变，比如修改了类名，修改了成员变量的类型，这个时候尽管serialVersionUID验证通过了，但是反序列化过程还是会失败，因为类的而机构有了重大改变，根本无法从老版本的数据还原出一个新的类结构对象。

**4、补充**

通过上面的分析，我们知道给serialVersionUID指定为1L或者采用IDE根据当前类结构去生成的hash值，这两者并没有本质区别，效果完全一样。再补充三点：

> - 1、静态成员变量属于类，不属于对象，所以不会参与序列化的过程
> - 2、用transient关键字编辑的成员变量不参与序列化的过程。
> - 3、可以通过重写writeObject()和readObject()两个方法来重写系统默认的序列化和反序列化的过程。不过本人并不推荐

### (二) Parcelable接口

Parcelable也是一个接口，只要实现了这个接口，一个类的对象就可以实现序列化和并且通过Intent和Binder传递。我们就借用上面Person类来看下代码：



```java
public class Person implements Parcelable{
    private static final  long serialVersionUID=1L;
    public String name;
    public String sex;
    public int age;

    public Person(String name,String sex,int age){
        this.name=name;
        this.sex=sex;
        this.age=age;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeString(sex);
        dest.writeInt(age);
    }
    
    public static final Parcelable.Creator<Person> CREATOR= new Creator<Person>() {
        @Override
        public Person createFromParcel(Parcel source) {
            return new Person(source);
        }

        @Override
        public Person[] newArray(int size) {
            return new Person[size];
        }
    };
    
    private Person(Parcel source){
        name=source.readString();
        sex=source.readString();
        age=source.readInt();
    }
}
```

这里先说一下Parcel，Parcel内部包装了可序列化的数据，可以在Binder中自由传输，从上面的代码我们可以看出，在序列化的过程中，需要实现的功能有序列化，反序列化的和内容描述。

> - 1、序列化功能由writeToParcel来完成，最终是通过Parcel中的一些列write方法来完成的。
> - 2、反序列化是由CREATOR来完成，其内部标明了如何创建序列化对象和数组，并通过Parcel的一些列read方法来完成反序列化过程。
> - 3、内容描述功能由describeContents方法来完成，几乎在所有情况下这个方法都应该返回0，仅当前对象中存在文件描述符时，此方法返回1。

Parcelable的方法说明：

| 方法                                  | 功能                                                         | 标记为                                   |
| ------------------------------------- | ------------------------------------------------------------ | ---------------------------------------- |
| createFromParcel(Parcel source)       | 从序列化后的对象中创建原始对象                               |                                          |
| newArray                              | 创建指定长度的原始对象数组                                   |                                          |
| Person(Parcel source)                 | 从序列化后的对象中创建原始对象                               |                                          |
| writeToParcel(Parcel dest, int flags) | 从当前对象吸入序列化结构中，其中flag标识有两种值0或者1，为1时标识当前对象需要作为返回值返回，不能立刻释放资源，几乎所有情况都为0 | Parcelable.PARCELABLE_WRITE_RETURN_VALUE |
| describeContents                      | 返回当前对象的内容描述，如果含有文件描述符，返回1，否则返回0，几乎所有情况都是返回0 | Parcelable.CONTENTS_FILE_DESCRIPTOR      |

系统已经为我们提供了许多实现了Parcelable接口的类，它们都是可以直接序列化的，比如Intent，Bundle，Bitmap等，同时List和Map页可以序列化，不过要求它们的每一个元素都是可以序列化的。

### (三) Serializable 和Parcelable的区别

**1、平台区别**

> - Serializable是属于 **Java** 自带的，表示一个对象可以转换成可存储或者可传输的状态，序列化后的对象可以在网络上进行传输，也可以存储到本地。
> - Parcelable 是属于 **Android** 专用。不过不同于Serializable，Parcelable实现的原理是将一个完整的对象进行分解。而分解后的每一部分都是Intent所支持的数据类型。

**2、编写上的区别**

> - Serializable代码量少，写起来方便
> - Parcelable代码多一些，略复杂

**3、选择的原则**

> - 1、如果是仅仅在内存中使用，比如activity、service之间进行对象的传递，强烈推荐使用Parcelable，因为Parcelable比Serializable性能高很多。因为Serializable在序列化的时候会产生大量的临时变量， 从而引起频繁的GC。
> - 2、如果是持久化操作，推荐Serializable，虽然Serializable效率比较低，但是还是要选择它，因为在外界有变化的情况下，Parcelable不能很好的保存数据的持续性。

**4、本质的区别**

> - 1、Serializable的本质是使用了反射，序列化的过程比较慢，这种机制在序列化的时候会创建很多临时的对象，比引起频繁的GC、
> - 2、Parcelable方式的本质是将一个完整的对象进行分解，而分解后的每一部分都是Intent所支持的类型，这样就实现了传递对象的功能了。

## 四、Parcel类详解

### (一)、故事

关于在进程间传递数据，举两个例子

**例子1：快递包裹**

比如从上海快递一件衣服，给北京的朋友，显眼途径很多，无论是陆海空的物流或者快递，在整个传递过程中"衣服"本身始终没有变过——朋友拿到的衣服还是原来的那件

**例子2：email发送图片**

假设你在上海通过电子邮件给北京的朋友，发送一张"衣服的图片"，那么当朋友看到它时，已经无法估计这张图片数据，在传输过程中被复制了多少次了——但可以肯定的是，他看到的图像和原始图像绝对是一模一样的。

很显然，进程间的数据传递和例2属于同一种情况，不过略有区别。

> 如果只是一个int型数值，不断复制知道目标进程即可。但是如果是某个对象呢？我们可以想象下，同一进程间的对象传递都是通过引用来做的，因而本质上就是传递了一个内存的地址。这种方式明显在跨进程情况下就不行了。由于采用了虚拟内存机制，两个进程都有自己独立的内存地址空间，所以进程间传递地址值是无效的。而在进程间传递数据是Binder机制的重要一环，大家不要着急，下一节就是要讲解Binder了。Android系统中负责跨进程通信的就是Parcel。那我们就来看下Parcel

### (二)、Parcel简介

> Parcel翻译过来就是打包的意思，其实就是包装了我们需要传输的数据，然后在Binder中传输，用于跨进程传输数据。就上问说的，直接传送内存地址是不行的，那么如果把进程A中的内存相关数据打包起来，然后寄送到进程B中，有B在自己的进程"复现"这个对象。Parcel在Android中扮演了跨进程传输的集装箱的角色，是序列化的一种手段。Parcel提供了一套机制，可以将序列化之后的数据写入一个共享内存中，其他进程通过Parcel可以从这块共享内存读出字节流，并反序列化成对象，如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-9123c36de14effb2.png?imageMogr2/auto-orient/strip|imageView2/2/w/731/format/webp)

Parcel模型.png

所以说：Parcel是一个存放读取数据的容器，系统中的binder进程间通信(IPC)就使用了Parcel类来进行客户端与服务端数据交互，而且AIDL的数据也是通过Parcel来交互的。在Java层和C++层都实现了Parcel，由于它在C/C++中，直接使用了内存来读取数据，因此，它更有效率。

> 其实Parcel是内存中的结构的是一块连续的内存，会自动根据需要自动扩展大小(这个设计比较赞，一些对性能要求不是太高或者小数据的地方，可以不用费脑向分配多大的空间)。Parcel传递数据，可以分为3种，传递方式也不一样：
>
> - 小型数据：从用户空间(源进程)copy到kernel空间(Binder驱动中)再写回用户空间(目标进程，binder驱动负责寻找目标进程)
> - 大型数据：使用android 的匿名共享内存 (Ashmem)传递
> - binder对象：Kernel binder驱动专门处理

### (三)、Parcel.java类介绍

由于篇幅限制，我之类就不详细讲解Parcel类，主要讲解两个部分，一个是类注释，一个是核心方法。

#### 1、Parcel.java类注释

我直接上注释了，就不贴出源码了

> - Parcel是一个消息(数据和对象的引用)的容器，IBinder是android用于进程间通信的方式(IPC)。Parcel可以携带序列化后(flattened/marshalled/serialized，通过使用多种类型的writing函数或者Parcelable接口)的数据，在IPC的另外一个反序列化数据(变回反序列化的对象)，Parcel也可以携带活的IBinder对象，传递到Parcel中与原本IBinder相连的代理Binder中。
> - Parcel不是通用的序列化机制(Android特有的，Java的序列化机制是Serilizable，在效率上不如Parcel)。这个class(以及相应的ParcelableAPI，用于将任意对象转换为Parcel)被设计为高性能的的IPC传输方式。因此将Parcel数据放置在持久化存储位置是不合适的，任何基于Parcel中数据前世的变化都会导致旧数据不可读。

后面就是关于支持的类型，总结一下如下：

> - 与Parcel相关的API有多种读写不同类型的方式，主要的6种类型如：
> - Primitives 基本数据类型
> - Primitives Arrays 基本数据类型数组
> - Parcelables 实现了Parcelables接口的对象
> - Bundle类
> - Active Objects 类 主要包括 Binder对象和FileDescriptor对象
> - Untyped Containers  容器，比如List、Map、SparseArray

#### 2、常用方法

**(1)、Parcel设置相关**

母庸质疑，存入的数据越多，Parcel所占用的内存空间越大。我们可以通过以下方法来进行相关设置。

> - dataSize()：得到当前Parcel对象的实际存储空间
> - setDataCapacity(int size)：设置Parcel的空间大小，显然存储的数据不能大于这个值
> - setDataPosition(int pos)：改变Parcel中的读写位置(我喜欢叫偏移量)，必须介于0和dataSize()间
> - dataAvail()：当前Parcel中的可读数据大小。
> - dataCapacity()：当前Parce的存储能力
> - dataPosition()：数据的当前位置值(偏移量)，有点类似于游标
> - dataSize()：当前Parcel所包含的数据大小

PS:如果写入数据时，系统发现已经超出了Parcel的存储能力，它会自动申请所需要的内存空间，并扩展dataCapacity；并且每次写入都是从dataPosition()开始的。

**(2)、Primitives**

原始类型数据的读写操作。比如：

> - writeInt(int)  ：写入一个整数
> - writeFloat(float) ：写入一个浮点数(单精度)
> - writeDouble(double)：写入一个双精度
> - writeString(string)：写入一个字符串
> - readInt(int)  ：读出一个整数
> - readFloat(float) ：读出一个浮点数(单精度)
> - readDouble(double)：写入一个双精度
> - readString(string)：写入一个字符串

从中也可以看出读写操作是配套的，用哪种方式写入的数据就是要用相应的方式正式读取。另外，数据是按照host cup的字节序来读写的

**(3)、Primitives Arrays**

原始数据类型的数组读写操作通常是先用4个字节表示数据的大小值，接着才写入数据本身。另外用户既可以选择将数据读入现有的数据空间中，也可以让Parcel返回一个新的数组，此类方法如下：

> - writeBooleanArray(boolean[])：写入布尔数组
> - readBooleanArray(boolean[])：读出布尔数组
> - boolean[] createBooleanArray()：读取并返回一个布尔数组
> - writeByteArray(byte[] ):写入字节数组
> - writeByteArray(byte[],int , int ) 和上面几个不同的是，这个函数最后面的两个参数分别表示数组中需要被写入的数据起点以及需要写入多少。
> - readByteArray(byte[])：读取字节数组
> - byte[] createByteArray()：读取并返回一个数组

**(4)、Parcelables**

遵循Parcelable协议的对象可以通过Parcel来存取，如开发人员经常用的的bundle就是实现Parcelable的，与此类对象相关的Parcel操作包括：

> - writeParcelable(Parcelable，int)：将这个Parcel类的名字和内容写入Parcel中，实际上它是通过回调此Parcelable的writeToParcel()方法来写入数据的。
> - readParcelable(ClassLoader)：读取并返回一个新的Parcelable对象
> - writeParcelableArray(T[]，int)：写入Parcelable对象数组。
> - readParcelable(ClassLoader)：读取并返回一个Parcelable数组对象

**(5)、Bundle**

上面已经提到了，Bundle继承自Parcelable，是一种特殊的type-safe的容器。Bundle的最大特点是采用键值对的方式存储数据，并在一定程度上优化了读取效率。这个类型的Parcel操作包括：

> - writeBundle(Bundle)：将Bundle写入parcel
> - readBundle()：读取并返回一个新的Bundle对象
> - readBundle(ClassLoader)：读取并返回一个新的Bundle对象，ClassLoader用于Bundle获取对应的Parcelable对象。

**(6)、Activity Object**

Parcel的另一个强大武器就是可以读写Active Object，什么是Active Object？通常我们存入Parcel的是对象的内容，而Active Object 写入的则是他们的特殊标志引用。所以在从Parcel中读取这些对象时，大家看到的并不是重新创建的对象实例，而是原来那个被写入的实例。可以猜想到，能够以这种方式传输的对象不会很多，目前主要有两类

> - 1、Binder：Binder一方面是Android系统IPC通信的核心机制之一，另一方面也是一个对象。利用Parcel将Binder写入，读取时就能得到原始的Binder对象，或者是它的特殊代理实现(最终操作的还是原始Binder对象)，与此相关的操作包括： 
>   - writeStrongBinder(IBinder)
>   - writeStrongInterface(IInterface)
>   - readStrongBinder()
> - 2、FileDescriptor：FileDescriptor是Linux中的文件描述符，可以通过Parcel如下方法进行传递： 
>   - writeFileDescriptor(FileDescriptor)
>   - readFileDescriptor()

因为传递后的对象仍然会基于和原对象相同的文件流进行操作，因而可以认为是Active Object的一种

**(7)、Untyped Containers**

它用于读写标准的任意类型的java容器。包括：

> - writeArray(Object[])
> - readArray(ClassLoader)
> - writeList(List)
> - readList(List, ClassLoader)
> - readArrayList(ClassLoader)
> - writeMap(Map)
> - readMap(Map, ClassLoader)
> - writeSparseArray(SparseArray)

- readSparseArray(ClassLoader)

**(8)、Parcel创建与回收**

> - obtain()：获取一个Parcel，可以理解new了一个对象，其实是有一个Parcel池
> - recyle()：清空，回收Parcel对象的内存

**(9)、异常的读写**

> - writeException()：在Parcel队头写入一个异常
> - readException()：在Parcel队头读取，若读取值为异常，则抛出该异常；否则程序正常运行

#### 3、创建Parcl对象

**(1)、obtain()方法**

app可以通过Parcel。obtain()接口来获取一个Parcel对象。

```java
    /**
     * Retrieve a new Parcel object from the pool.
     * 从Parcel池中取出一个新的Parcel对象
     */
    public static Parcel obtain() {
        //系统预先产出的一个Parcel池(其实就是一个Parcel数组)，大小为6
        final Parcel[] pool = sOwnedPool;
        synchronized (pool) {
            Parcel p;
            for (int i=0; i<POOL_SIZE; i++) {
                p = pool[i];
                if (p != null) {
                    //引用置为空，这样下次就知道这个Parcel已经被占用了
                    pool[i] = null;
                    if (DEBUG_RECYCLE) {
                        p.mStack = new RuntimeException();
                    }
                    return p;
                }
            }
        }
        //如果Parcel池中已经空，就直接新建一个。
        return new Parcel(0);
    }
```

> 在这里，我们要注意到这里的池其实是一个数组，从里面提取对象的时候，从头扫描到尾，找不到为null的手，直接new一个Parcle对象并返回。

那我们就继续看下他的构造函数

**(2)、Parcel构造函数**

我们来看下它的构造函数



```cpp
private Parcel(long nativePtr) {
        if (DEBUG_RECYCLE) {
            mStack = new RuntimeException();
        }
        //Log.i(TAG, "Initializing obj=0x" + Integer.toHexString(obj), mStack);
        init(nativePtr);
    }
```

通过上面代码，我们知道，构造函数里面什么都没做，只是调用了init()函数，注意传入的是nativePtr是0



```java
    private void init(long nativePtr) {
        if (nativePtr != 0) {
             //如果nativePtr不是0
            mNativePtr = nativePtr;
            mOwnsNativeParcelObject = false;
        } else {
            //如果nativePtr是0
            // nativeCreate() 为本地层代码准备的指针
            mNativePtr = nativeCreate();
            mOwnsNativeParcelObject = true;
        }
    }

  private long mNativePtr; // used by native code
  private static native long nativeCreate();
```

我们知道最终是调用了nativeCreate()方法

**(3)、nativeCreate()方法**

根据上篇文章讲解的关于JNI的方法，我们得知nativeCreate对应的是JNI层的/frameworks/base/core/jni中，实际上Parcel.java只是一个简单的中介，最终所有类型的读写操作都是通过本地代码实现的:
 在[android_os_Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_Parcel.cpp)中



```cpp
//frameworks/base/core/jni/android_os_Parcel.cpp    551行
static jlong android_os_Parcel_create(JNIEnv* env, jclass clazz)
{
    Parcel* parcel = new Parcel();
    return reinterpret_cast<jlong>(parcel);
}
```

所以说mNativePtr变量实际上是一个本底层的Parcel(C++)对象

这里顺便说一下Parcel-Native的构造函数
 代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp)  343行



```cpp
Parcel::Parcel()
{
    LOG_ALLOC("Parcel %p: constructing", this);
    initState();
}
```

构造函数很简单就是调用了 **initState()** 函数，让我们来看下 **initState()**函数。

代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp)  1901行



```cpp
void Parcel::initState()
{
    LOG_ALLOC("Parcel %p: initState", this);
    mError = NO_ERROR;
    mData = 0;
    mDataSize = 0;
    mDataCapacity = 0;
    mDataPos = 0;
    ALOGV("initState Setting data size of %p to %zu", this, mDataSize);
    ALOGV("initState Setting data pos of %p to %zu", this, mDataPos);
    mObjects = NULL;
    mObjectsSize = 0;
    mObjectsCapacity = 0;
    mNextObjectHint = 0;
    mHasFds = false;
    mFdsKnown = true;
    mAllowFds = true;
    mOwner = NULL;
    mOpenAshmemSize = 0;
}
```

初始话很简单，几乎都是初始化为 0（NULL） 的

**(4)、Parcel.h:**

> 其实每一个Parcel对象都有一个Native对象相对应(以后均简称Parcel-Native对象，而Parcel对象均指Java层)，该Native对象就是实际的写入读出的一个对象，java端对它的引用是上面**mNativePtr**。

对应的Native层的Parcel定义是在 [/frameworks/native/inlcude/binder/Parcel.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/include/binder/Parcel.h)



```cpp
class Parcel {
public:
    ...
    int32_t             readInt32() const; // 举个例子
    ...
    status_t            writeInt32(int32_t val);
    ...
private:
    uint8_t*            mData;
    size_t              mDataSize;
    size_t              mDataCapacity;
    mutable size_t      mDataPos;
}
```

> Parcel对象的数据读取、写入操作都是最终通过Parcel-Native对象搞定的，这里有几个需要注意的重要成员，mData就是数据存储的起始指针，mDataSize总数据大小，mDataCapacity是指mData总空间 (包括已用和可用)大小，这个空间是变长的，mDataPos是指当前数据可写入的内存其实位置，mData总是一块连续的内存地址，每一次其总空间大小增长都会通过realloc进行内存分配，如果数据量过大、内存碎片过多导致内存分配失败就会报错。

**(5)、总结:**

使用Parcel一般是通过Parcel.obtain()从对象池中获取一个新的Parcel对象，如果对象池中没有则直接new的Parcel则直接创建新的一个Parcel对象，并且会自动创建一个Parcel-Native对象。

那我们就来看下回收的工作

#### 4、Parcell对象回收

**(1)、 Parcel-Native对象的回收**

上面说了创建Parcel对象的时候，会自动创建一个Parcel-Native对象。那这个Parcel-Native是什么时候回收的那？答案是：在Parcel对象销毁时，即finalize()时回收的，代码如下：

```java
// Parcel.java
    @Override
    protected void finalize() throws Throwable {
        if (DEBUG_RECYCLE) {
            if (mStack != null) {
                Log.w(TAG, "Client did not call Parcel.recycle()", mStack);
            }
        }
        destroy();
    }

    private void destroy() {
        if (mNativePtr != 0) {
            if (mOwnsNativeParcelObject) {
                nativeDestroy(mNativePtr);
                updateNativeSize(0);
            }
            mNativePtr = 0;
        }
    }

    private static native void nativeDestroy(long nativePtr);
```

我们看到在finalize()方法里面调用了 destroy()方法，而在destroy()方法里面调用了nativeDestroy(long)方法，那我们就继续跟踪下

```cpp
//frameworks/base/core/jni/android_os_Parcel.cpp      567行
static void android_os_Parcel_destroy(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    delete parcel;
}
```

我们知道了直接delete了 parcel对象

**(2)、 recycle()方法**

> sOwneredPool算是一个Parcel对象池(sHolderPool也是，二者区别在于sOwnerPool用于保存Parcel-Native对象生命周期需要由自己管理的Parcel对象)，可以复用之前创建好的Parcel对象，我们在使用完Parcel对象后，可以通过recycle方法回收到对象池中

直接上代码

```dart
    /**
     * Put a Parcel object back into the pool.  You must not touch
     * the object after this call.
     * 将Parcel对象放回池中。 这次调用之后，你就无法继续联系他了
     */
    public final void recycle() {
        if (DEBUG_RECYCLE) mStack = null;
        //Parcel-Native 对象的数据清空
        freeBuffer();

        final Parcel[] pool;
        if (mOwnsNativeParcelObject) {
            //选择合适的对象池
            pool = sOwnedPool;
        } else {
            mNativePtr = 0;
            pool = sHolderPool;
        }

        synchronized (pool) {
            for (int i=0; i<POOL_SIZE; i++) {
                if (pool[i] == null) {
                   //如果有位置就直接加入对象池。
                    pool[i] = this;
                    return;
                }
            }
        }
    }
```

加入对象池之后的对象除非重新靠obtain()启动，否则不能直接使用，因为它时刻都可能被其他地方获取使用导致数据错误。

#### 5、存储空间

> Parcel内部的存储区域主要有两个，是mData和mObjects，mData存储是基本数据类型，mObjects存储Binder数据类型。Parcel提供了针对各种数据写入和读出的操作函数。这两块区域都是使用malloc分配出来的。

**5.1、flat_binder_object**

在Parcel的序列化中，Binder对象使用flat_binder_object结构体保存。同时提供了flatten_binder和unflatten_binder函数用于序列化和反序列化。
 结构体代码在[Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h)   68行

```rust
/*
 * This is the flattened representation of a Binder object for transfer
 * between processes.  The 'offsets' supplied as part of a binder transaction
 * contains offsets into the data where these structures occur.  The Binder
 * driver takes care of re-writing the structure type and data as it moves
 * between processes.
 */
struct flat_binder_object {
    struct binder_object_header hdr;
    __u32               flags;

    /* 8 bytes of data. */
    union {
        binder_uintptr_t    binder; /* local object */
        __u32           handle; /* remote object */
    };
    /* extra data associated with local object */
    binder_uintptr_t    cookie;
};
```

这里先说下，Binder实体或引用在传递时，被表示为flat_binder_object，flat_binder_object的type域表示
 表示传输Binder的类型。

> - 跨进程的时候：flat_binder_object的type为BINDER_TYPE_HANDLE
>    跨进程的时候
> - 非跨进程的时候：flat_binder_object的type为BINDER_TYPE_BINDER

#### 6、关于偏移量

**(1)、基本类型**

看到了了前面的内容，相信大家对Parcel有了初步的了解，那么Parcel内部存储机制是怎么样的？偏移量又是什么东西？我们一起回忆一下基本数据类型的取值范围：

| 类型    | bit数量 | 字节   |
| ------- | ------- | ------ |
| boolean | 1 bit   | 1字节  |
| char    | 16bit   | 2字节  |
| int     | 32bit   | 4字节  |
| long    | 64 bit  | 8 字节 |
| float   | 32 bit  | 4 字节 |
| double  | 64bit   | 8字节  |

如果大家对C语言熟悉的话，C语言中结构体的内存对齐和Parcel采用的内存存放机制一样，即读取最小字节为32bit，也即4个字节。高于4个字节的，以实际数据类型进行存放，但得为4byte的倍数。基本公式如下：

> - 实际存放字节：
>    辨别一：32bit (<=32bit)                   **例如：boolean，char，int**
>    辨别二：实际占用字节(>32bit)         **例如：long，float，String 等** 
> - 实际读取字节：
>    辨别一：32bit (<=32bit)                   **例如：boolean，char，int**
>    辨别二：实际占用字节(>32bit)         **例如：long，float，String 组等** 

所以，当我们写入/读取一个数据时，偏移量至少为4byte(32bit)，于是，偏移量的公式如下：

```undefined
f(x)= 4x (x=0,1,....n)
```

事实上，我们可以通过setDataPosition(int position)来直接操作我们欲读取数据时的偏移量。毫无疑问，你可以设置任何便宜量，但是所读取值的类型是有误的。因此设置便宜量读取值的时候，需要小心。

**(2)、注意事项**

> PS:
>  我们在writeXXX()和readXXX()时，导致的偏移量时共用的，例如我们在writeIn(23)后，此时的dataposition=4，如果我想读取它，简单的通过readInt()是不行的，只能得到0，这时我们只能通过setDataPosition(0)设置为起始偏移量，从起始位置读取四个字节，即可得23。因此，在读取某个值时，需要使用setDataPosition(int)使偏移量偏移到我们的指定位置。

**(3)、取值规范**

由于可能存在读取值的偏差，一个默认的取值规范为：

> - 1、读取复杂对象时：对象匹配时，返回当前偏移位置的对象；对象不匹配时，返回null。
> - 2、读取简单对象时：对象匹配时，返回当前偏移位置的对象：对象不匹配时，返回0。

**(4)、存放空间图**

下面，给出一张浅显的Parcel的存放空间图

![img](https:////upload-images.jianshu.io/upload_images/5713484-d0960176652b250f.png?imageMogr2/auto-orient/strip|imageView2/2/w/805/format/webp)

Parcel空间存放图.png

#### 7 、Int类型数据写入

我们以writeInt()为例进行数据写入的跟踪

时序图如下：



![img](https:////upload-images.jianshu.io/upload_images/5713484-177c06edd517c83f.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

writeInt.jpeg

**(1) Parcel.writeInt(int)**

```cpp
    /**
     * Write an integer value into the parcel at the current dataPosition(),
     * growing dataCapacity() if needed.
     */
    public final void writeInt(int val) {
        nativeWriteInt(mNativePtr, val);
    }
```

方法翻译如下：

> 在当前的dataPosition()中，将一个int的值写入Parcel中，如果空间不足，则扩容。

通过代码我们知道其实调用的nativeWriteInt(long nativePtr, int val)，我们知道nativeWriteInt(long nativePtr, int val)其实对应的是JNI的方法

**(2) android_os_Parcel_writeInt(JNIEnv*,jclass,jlong,jint)函数**

代码在[android_os_Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_Parcel.cpp)    233行

```cpp
static void android_os_Parcel_writeInt(JNIEnv* env, jclass clazz, jlong nativePtr, jint val) {
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        const status_t err = parcel->writeInt32(val);
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
```

我们看到实际上是调用的 Parcel-Native类的writeInt32(jint)函数

**(3) Parcel::writeInt32(int32_t val)函数**

代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp)    748行

```cpp
status_t Parcel::writeInt32(int32_t val)
{
    return writeAligned(val);
}
```

我们看到实际上是调用的 Parcel-Native类的writeAligned()函数

**(4) Parcel::writeAligned(T val)函数**

代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp)   1148行

```cpp
template<class T>
status_t Parcel::writeAligned(T val) {
    COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE_UNSAFE(sizeof(T)) == sizeof(T));
    if ((mDataPos+sizeof(val)) <= mDataCapacity) {
restart_write:
        //分支二
        *reinterpret_cast<T*>(mData+mDataPos) = val;
        return finishWrite(sizeof(val));
    }
    //分支一  刚刚创建Parcel-Native对象
    status_t err = growData(sizeof(val));
    if (err == NO_ERROR) goto restart_write;
    return err;
}
```

writeAligned看名字就知道是内存对齐，第一句我没太搞明白，好像是验证是否内存对齐，好像能够根据编译选项判断，应该是打开某个编译选项，如果size没有内存对齐，直接报错，内存对齐都是搞一些位运算(这里是4个字节对齐)。

> writeAligned()函数判断当前pos+要写入的数据占用的内存  是否比mDataCapacity大，如果打，就是空间不足，需要自动增长空间，走growData()

这里我们假设刚刚创建了 Parcel-Native对象。这时候mDataCapacity=0,mDataPos=0,mData=0,mDataSizeDataSize=0,所这时候不走**分支二**，走**分支一** ,在**分支一** 里面调用了growData()函数 分配内存

**(5) Parcel::growData(size_t len)函数**

代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp)   1683行

```cpp
status_t Parcel::growData(size_t len)
{
     //如果超过int的最大值
    if (len > INT32_MAX) {
        // don't accept size_t values which may have come from an
        // inadvertent conversion from a negative int.
        return BAD_VALUE;
    }

    size_t newSize = ((mDataSize+len)*3)/2;
    //通过上面的代码newSize一般来说都会大于mDataSize
    return (newSize <= mDataSize)
            ? (status_t) NO_MEMORY
            : continueWrite(newSize);
}
```

> PS:这里是parcel的增长算法，((mDataSize+len)*3/2)；带有一定预测性的增长，避免频繁的空间调整(每次调整都需要重新malloc内存的，频繁的话会影响效率)。然后这里有个newSize< mDataSize就认为NO_MEMORY。这是如果溢出了(是负数)，就认为申请不到内存了。

因为newSize一般来说都会大于mDataSize，所以函数最后走到了continueWrite(newSize)函数里面去了

**(6) Parcel::continueWrite(size_t desired)函数**

代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp)   1743行

```cpp
status_t Parcel::continueWrite(size_t desired)
{
    if (desired > INT32_MAX) {
        // don't accept size_t values which may have come from an
        // inadvertent conversion from a negative int.
        return BAD_VALUE;
    }

    // If shrinking, first adjust for any objects that appear
    // after the new data size.
    size_t objectsSize = mObjectsSize;
    if (desired < mDataSize) {
        if (desired == 0) {
            objectsSize = 0;
        } else {
            while (objectsSize > 0) {
                if (mObjects[objectsSize-1] < desired)
                    break;
                objectsSize--;
            }
        }
    }
    //分支一
    if (mOwner) {
        // If the size is going to zero, just release the owner's data.
        if (desired == 0) {
            freeData();
            return NO_ERROR;
        }

        // If there is a different owner, we need to take
        // posession.
        uint8_t* data = (uint8_t*)malloc(desired);
        if (!data) {
            mError = NO_MEMORY;
            return NO_MEMORY;
        }
        binder_size_t* objects = NULL;

        if (objectsSize) {
            objects = (binder_size_t*)calloc(objectsSize, sizeof(binder_size_t));
            if (!objects) {
                free(data);

                mError = NO_MEMORY;
                return NO_MEMORY;
            }

            // Little hack to only acquire references on objects
            // we will be keeping.
            size_t oldObjectsSize = mObjectsSize;
            mObjectsSize = objectsSize;
            acquireObjects();
            mObjectsSize = oldObjectsSize;
        }

        if (mData) {
            memcpy(data, mData, mDataSize < desired ? mDataSize : desired);
        }
        if (objects && mObjects) {
            memcpy(objects, mObjects, objectsSize*sizeof(binder_size_t));
        }
        //ALOGI("Freeing data ref of %p (pid=%d)", this, getpid());
        mOwner(this, mData, mDataSize, mObjects, mObjectsSize, mOwnerCookie);
        mOwner = NULL;

        LOG_ALLOC("Parcel %p: taking ownership of %zu capacity", this, desired);
        pthread_mutex_lock(&gParcelGlobalAllocSizeLock);
        gParcelGlobalAllocSize += desired;
        gParcelGlobalAllocCount++;
        pthread_mutex_unlock(&gParcelGlobalAllocSizeLock);

        mData = data;
        mObjects = objects;
        mDataSize = (mDataSize < desired) ? mDataSize : desired;
        ALOGV("continueWrite Setting data size of %p to %zu", this, mDataSize);
        mDataCapacity = desired;
        mObjectsSize = mObjectsCapacity = objectsSize;
        mNextObjectHint = 0;
    //分支二 
    } else if (mData) {
        if (objectsSize < mObjectsSize) {
            // Need to release refs on any objects we are dropping.
            const sp<ProcessState> proc(ProcessState::self());
            for (size_t i=objectsSize; i<mObjectsSize; i++) {
                const flat_binder_object* flat
                    = reinterpret_cast<flat_binder_object*>(mData+mObjects[i]);
                if (flat->type == BINDER_TYPE_FD) {
                    // will need to rescan because we may have lopped off the only FDs
                    mFdsKnown = false;
                }
                release_object(proc, *flat, this, &mOpenAshmemSize);
            }
            binder_size_t* objects =
                (binder_size_t*)realloc(mObjects, objectsSize*sizeof(binder_size_t));
            if (objects) {
                mObjects = objects;
            }
            mObjectsSize = objectsSize;
            mNextObjectHint = 0;
        }

        // We own the data, so we can just do a realloc().
        if (desired > mDataCapacity) {
            uint8_t* data = (uint8_t*)realloc(mData, desired);
            if (data) {
                LOG_ALLOC("Parcel %p: continue from %zu to %zu capacity", this, mDataCapacity,
                        desired);
                pthread_mutex_lock(&gParcelGlobalAllocSizeLock);
                gParcelGlobalAllocSize += desired;
                gParcelGlobalAllocSize -= mDataCapacity;
                pthread_mutex_unlock(&gParcelGlobalAllocSizeLock);
                mData = data;
                mDataCapacity = desired;
            } else if (desired > mDataCapacity) {
                mError = NO_MEMORY;
                return NO_MEMORY;
            }
        } else {
            if (mDataSize > desired) {
                mDataSize = desired;
                ALOGV("continueWrite Setting data size of %p to %zu", this, mDataSize);
            }
            if (mDataPos > desired) {
                mDataPos = desired;
                ALOGV("continueWrite Setting data pos of %p to %zu", this, mDataPos);
            }
        }
    //分支三
    } else {
        // This is the first data.  Easy!
        uint8_t* data = (uint8_t*)malloc(desired);
        if (!data) {
            mError = NO_MEMORY;
            return NO_MEMORY;
        }

        if(!(mDataCapacity == 0 && mObjects == NULL
             && mObjectsCapacity == 0)) {
            ALOGE("continueWrite: %zu/%p/%zu/%zu", mDataCapacity, mObjects, mObjectsCapacity, desired);
        }

        LOG_ALLOC("Parcel %p: allocating with %zu capacity", this, desired);
        pthread_mutex_lock(&gParcelGlobalAllocSizeLock);
        gParcelGlobalAllocSize += desired;
        gParcelGlobalAllocCount++;
        pthread_mutex_unlock(&gParcelGlobalAllocSizeLock);

        mData = data;
        mDataSize = mDataPos = 0;
        ALOGV("continueWrite Setting data size of %p to %zu", this, mDataSize);
        ALOGV("continueWrite Setting data pos of %p to %zu", this, mDataPos);
        mDataCapacity = desired;
    }
    return NO_ERROR;
}
```

PS: mOwner是个函数指针。
 其实这个方法写的不是很好，那么多分支，应该增加几个工具函数，这样函数代码量就不会这么大了。简单说下三个主要分支

> - 分支一：如果设置了release函数指针(即mOwener)，调用release函数进行处理
> - 分支二：没有设置release函数指针，但是mData中存在数据，需要在原来的数据的基础上扩展存储空间。
> - 分支三：没有设置release函数指针，并且mData不存在数据(就是注释上说的第一次使用)，调用malloc申请内存块，保存mData。设置相应的设置capacity、size、pos、object的值。

> 承接上文，这里应该走分支分支三，分支三很简单，主要是调用malloc()方法分配一块(mDataSize+size(val))*3/2大小的内存，然后让mData指向该内存，并且将这里可以归纳一下，growData()方法只是分配了一内存。

根据返回值，又回到了Parcel::writeAligned(T val)中，由于返回值是NO_ERROR，所以就走到了**goto restart_write** ,这样就又到了Parcel::writeAligned(T val) 分支二中

**(7) Parcel::writeAligned(T val)函数 分支二**

分之二代码就两行，如下

```cpp
        *reinterpret_cast<T*>(mData+mDataPos) = val;
        return finishWrite(sizeof(val));
```

> - 1、* reinterpret_cast<T*>(mData+mDataPos) = val; 这行代码是直接获取当前地址强制转化指针类型，然后赋值。
> - 2、调用finishWrite()函数

**(8) Parcel::finishWrite(size_t len)函数**

代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp)       642行

```kotlin
status_t Parcel::finishWrite(size_t len)
{
    if (len > INT32_MAX) {
        // don't accept size_t values which may have come from an
        // inadvertent conversion from a negative int.
        return BAD_VALUE;
    }

    //printf("Finish write of %d\n", len);
    mDataPos += len;
    ALOGV("finishWrite Setting data pos of %p to %zu", this, mDataPos);
    if (mDataPos > mDataSize) {
        mDataSize = mDataPos;
        ALOGV("finishWrite Setting data size of %p to %zu", this, mDataSize);
    }
    //printf("New pos=%d, size=%d\n", mDataPos, mDataSize);
    return NO_ERROR;
}
```

这个方法比较简单，主要内容如下：

> mDataPos增加到刚刚写入数据的末尾，并且进行一个判断，如果mDataPos>mDataSize的话，就将mDataSize=mDataPos;而实际上mDataSize在赋值前还是0，所以会进行这个赋值操作，因此我们可以知道，其实mDataSize是记录当前mData中写入数据的大小。

#### 8 、Int类型数据读出

我们以readInt()为例进行数据写入的跟踪

时序图如下：

![img](https://upload-images.jianshu.io/upload_images/5713484-bc98aa0511f16523.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp) 

readInt.jpeg

**(1) Parcel.readInt(int)**

```java
    /**
     * Read an integer value from the parcel at the current dataPosition().
     */
    public final int readInt() {
        return nativeReadInt(mNativePtr);
    }

    private static native int nativeReadInt(long nativePtr);
```

方法注释翻译如下：

> 从当前dataPosition()的位置上读取一个Interger值

通过代码我们知道其实是调用的nativeReadInt(long nativePtr)，我们知道nativeReadInt(long nativePtr)其实对应的是JNI的方法

**(2) android_os_Parcel_readInt(JNIEnv* env, jclass clazz, jlong nativePtr) 函数**

代码在[android_os_Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_Parcel.cpp) 379行

```cpp
static jint android_os_Parcel_readInt(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        return parcel->readInt32();
    }
    return 0;
}
```

这个代码也简单，主要是直接调用了Parcel-Native的readInt32()函数

**(3) Parcel::readInt32() 函数**

代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp) 1168行

```cpp
int32_t Parcel::readInt32() const
{
    return readAligned<int32_t>();
}
```

这个代码也很简单，内部调用了readAligned<int32_t>() 函数

**(4) readAligned<int32_t>() 函数**

代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp) 1138行

```cpp
template<class T>
T Parcel::readAligned() const {
    T result;
    if (readAligned(&result) != NO_ERROR) {
        result = 0;
    }
    return result;
}
```

其实内部是有调用了Parcel::readAligned(T *pArg)函数

> 注意：Parcel::readAligned(T *pArg)和 Parcel::readAligned()的区别，一个是有入参的，一个是无入参的。

**(5) readAligned<int32_t>() 函数**

代码在[Parcel.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/libs/binder/Parcel.cpp) 1124行

```cpp
template<class T>
status_t Parcel::readAligned(T *pArg) const {
    COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE_UNSAFE(sizeof(T)) == sizeof(T));

    if ((mDataPos+sizeof(T)) <= mDataSize) {
        const void* data = mData+mDataPos;
        mDataPos += sizeof(T);
        *pArg =  *reinterpret_cast<const T*>(data);
        return NO_ERROR;
    } else {
        return NOT_ENOUGH_DATA;
    }
```

这个就是根据 mData+mDataPos 和具体的类型，进行强制类型转化获取对应的值。

**(6) 注意事项:**

同进程情况下，数据读取过程跟写入几乎一致，由于使用的是同一个Parcel对象，mDataPos首先要调整到0之后才能读取，同进程数据写入/读取并不会有什么效率提高，仍然会进行内存的考虑和分配，所以一般来说尽量避免同进程使用Parcel传递大数据。







# Android跨进程通信IPC之Binder的三大接口

本片文章的主要目的是让大家对Binder有个初步的了解，既然是初步了解，肯定所是以源码上的注释为主，让大家对Binder有一个更直观的认识。PS:大部分注释我是写在类里面了， 重要的我会单独的拿出来。
 主要内容如下：

> 1、IInterface
>  2、IBinder
>  3、Binder与BinderProxy类
>  4、总结

## 一、IInterface



```csharp
/**
 * Base class for Binder interfaces.  When defining a new interface,
 * you must derive it from IInterface.
 */
public interface IInterface
{
    /**
     * Retrieve the Binder object associated with this interface.
     * You must use this instead of a plain cast, so that proxy objects
     * can return the correct result.
     */
    public IBinder asBinder();
}
```

**(一)、类注释**

简单的翻译一下：

> IInterface是Binder中相关接口的基类。 定义新接口的时候，你必须从IInterface派生。

**(二)、抽象方法注释**

> 如果想获取和该接口关联的Binder对象。你必须使用这个方法来而不是使用一个简单的类型转化。这样代理对象才能返回正确的结果

**(三)、总结**

> 所以可以这样说IInterface接口提供了类型转化的功能，将服务或者服务代理类转为IBinder类型。其实实际的类型转换是由BnInterface、BpInterface两个类完成的，BnInterface将服务类转换成IBinder类型，而BpInterface则将代理服务类转换成IBinder类型。在通过Binder Driver传递Binder对象时，必须进行类型转换，比如在想系统注册服务时，需要先将服务类型转换成IBinder，在其传递给Context Manager。

## 二、IBinder

```dart
/**
 * Base interface for a remotable object, the core part of a lightweight
 * remote procedure call mechanism designed for high performance when
 * performing in-process and cross-process calls.  This
 * interface describes the abstract protocol for interacting with a
 * remotable object.  Do not implement this interface directly, instead
 * extend from {@link Binder}.
 * 
 * <p>The key IBinder API is {@link #transact transact()} matched by
 * {@link Binder#onTransact Binder.onTransact()}.  These
 * methods allow you to send a call to an IBinder object and receive a
 * call coming in to a Binder object, respectively.  This transaction API
 * is synchronous, such that a call to {@link #transact transact()} does not
 * return until the target has returned from
 * {@link Binder#onTransact Binder.onTransact()}; this is the
 * expected behavior when calling an object that exists in the local
 * process, and the underlying inter-process communication (IPC) mechanism
 * ensures that these same semantics apply when going across processes.
 * 
 * <p>The data sent through transact() is a {@link Parcel}, a generic buffer
 * of data that also maintains some meta-data about its contents.  The meta
 * data is used to manage IBinder object references in the buffer, so that those
 * references can be maintained as the buffer moves across processes.  This
 * mechanism ensures that when an IBinder is written into a Parcel and sent to
 * another process, if that other process sends a reference to that same IBinder
 * back to the original process, then the original process will receive the
 * same IBinder object back.  These semantics allow IBinder/Binder objects to
 * be used as a unique identity (to serve as a token or for other purposes)
 * that can be managed across processes.
 * 
 * <p>The system maintains a pool of transaction threads in each process that
 * it runs in.  These threads are used to dispatch all
 * IPCs coming in from other processes.  For example, when an IPC is made from
 * process A to process B, the calling thread in A blocks in transact() as
 * it sends the transaction to process B.  The next available pool thread in
 * B receives the incoming transaction, calls Binder.onTransact() on the target
 * object, and replies with the result Parcel.  Upon receiving its result, the
 * thread in process A returns to allow its execution to continue.  In effect,
 * other processes appear to use as additional threads that you did not create
 * executing in your own process.
 * 
 * <p>The Binder system also supports recursion across processes.  For example
 * if process A performs a transaction to process B, and process B while
 * handling that transaction calls transact() on an IBinder that is implemented
 * in A, then the thread in A that is currently waiting for the original
 * transaction to finish will take care of calling Binder.onTransact() on the
 * object being called by B.  This ensures that the recursion semantics when
 * calling remote binder object are the same as when calling local objects.
 * 
 * <p>When working with remote objects, you often want to find out when they
 * are no longer valid.  There are three ways this can be determined:
 * <ul>
 * <li> The {@link #transact transact()} method will throw a
 * {@link RemoteException} exception if you try to call it on an IBinder
 * whose process no longer exists.
 * <li> The {@link #pingBinder()} method can be called, and will return false
 * if the remote process no longer exists.
 * <li> The {@link #linkToDeath linkToDeath()} method can be used to register
 * a {@link DeathRecipient} with the IBinder, which will be called when its
 * containing process goes away.
 * </ul>
 * 
 * @see Binder
 */
public interface IBinder {
    /**
     * The first transaction code available for user commands.
     * 第一个可用于用户命令的事务代码。
     */
    int FIRST_CALL_TRANSACTION  = 0x00000001;
    /**
     * The last transaction code available for user commands.
     * 最后一个可用于用户命令的事务代码。
     */
    int LAST_CALL_TRANSACTION   = 0x00ffffff;
    
    /**
     * IBinder protocol transaction code: pingBinder().
     * IBinder协议事物码：在pingBinder()会用到
     */
    int PING_TRANSACTION        = ('_'<<24)|('P'<<16)|('N'<<8)|'G';
    
    /**
     * IBinder protocol transaction code: dump internal state.
     * IBinder协议事物码: 代表清除内部状态 
     */
    int DUMP_TRANSACTION        = ('_'<<24)|('D'<<16)|('M'<<8)|'P';
    
    /**
     * IBinder protocol transaction code: execute a shell command.
     * IBinder协议事物码：代表执行一个shell命令
     * @hide
     */
    int SHELL_COMMAND_TRANSACTION = ('_'<<24)|('C'<<16)|('M'<<8)|'D';

    /**
     * IBinder protocol transaction code: interrogate the recipient side
     * of the transaction for its canonical interface descriptor.
     *  IBinder协议事物码：代表询问被调用方的接口描述符号
     */
    int INTERFACE_TRANSACTION   = ('_'<<24)|('N'<<16)|('T'<<8)|'F';

    /**
     * IBinder protocol transaction code: send a tweet to the target
     * object.  The data in the parcel is intended to be delivered to
     * a shared messaging service associated with the object; it can be
     * anything, as long as it is not more than 130 UTF-8 characters to
     * conservatively fit within common messaging services.  As part of
     * {@link Build.VERSION_CODES#HONEYCOMB_MR2}, all Binder objects are
     * expected to support this protocol for fully integrated tweeting
     * across the platform.  To support older code, the default implementation
     * logs the tweet to the main log as a simple emulation of broadcasting
     * it publicly over the Internet.
     * 
     * <p>Also, upon completing the dispatch, the object must make a cup
     * of tea, return it to the caller, and exclaim "jolly good message
     * old boy!".
     * IBinder协议事物码:向目标对象发出一个呼叫。在parcel中的数据是用于
     * 分发一个与对象结合的共享信息服务。它可以是任何东西，只要它不超
     * 过130个UTF-8字符，保证适用于常见的信息服务。 作为 
     * Build.VERSION_CODES#HONEYCOMB_MR2的一部分，所有的binder
     * 对象都支持这个协议，为了在跨平台间完整推送同时也为了支持旧的交
     * 互码，一个默认的实现是在主日志中记录了一个推送，作为广播的一个
     * 简单模仿。
     * 并且，为了完成消息的派送，对象必须返回给调用者一个说明，
     * 表明是旧的信息。
     */
    int TWEET_TRANSACTION   = ('_'<<24)|('T'<<16)|('W'<<8)|'T';

    /**
     * IBinder protocol transaction code: tell an app asynchronously that the
     * caller likes it.  The app is responsible for incrementing and maintaining
     * its own like counter, and may display this value to the user to indicate the
     * quality of the app.  This is an optional command that applications do not
     * need to handle, so the default implementation is to do nothing.
     * 
     * <p>There is no response returned and nothing about the
     * system will be functionally affected by it, but it will improve the
     * app's self-esteem.
     * IBinder协议事物码:异步地告诉app 有一个调用者在呼叫它。这个app负
     * 责计算和维护自己的呼叫者数目。 并且可以展示这个值来告诉用户app
     * 的状态。这个是可选的命令，app不需要掌管它，所以默认是实现是什
     * 么都不做。
     * 这是没有响应的，并且不会对系统带来影响，但是它提高了app的自控
     * (我真的不知道怎么翻译，elf-esteem其实是自尊的意思)
     */
    int LIKE_TRANSACTION   = ('_'<<24)|('L'<<16)|('I'<<8)|'K';

    /** @hide */
    //隐藏的API
    int SYSPROPS_TRANSACTION = ('_'<<24)|('S'<<16)|('P'<<8)|'R';

    /**
     * Flag to {@link #transact}: this is a one-way call, meaning that the
     * caller returns immediately, without waiting for a result from the
     * callee. Applies only if the caller and callee are in different
     * processes.
     * 用于transact()方法中的flag，表示单向RPC，表明呼叫者会马上返回，
     * 不必等结果从被呼叫者返回。只有当呼叫者和被呼叫者在不同的进程中
     * 才有效.
     */
    int FLAG_ONEWAY             = 0x00000001;

    /**
     * Limit that should be placed on IPC sizes to keep them safely under the
     * transaction buffer limit.
     * @hide
     * 为了让其安全地保持在事物缓冲区限制之下，应该限制IPC的大小
     *  隐藏的API
     */
    public static final int MAX_IPC_SIZE = 64 * 1024;

    /**
     * Get the canonical name of the interface supported by this binder.
     *  获取一个支持binder的接口规范名称
     */
    public String getInterfaceDescriptor() throws RemoteException;

    /**
     * Check to see if the object still exists.
     * 
     * @return Returns false if the
     * hosting process is gone, otherwise the result (always by default
     * true) returned by the pingBinder() implementation on the other
     * side.
     * 检查对象是否仍然存在。
     * 如果返回false，主机进程消失，否则结果为true（始终默认true）由对
     * 手方来实现pingBinder()这个方法。
     */
    public boolean pingBinder();

    /**
     * Check to see if the process that the binder is in is still alive.
     *
     * @return false if the process is not alive.  Note that if it returns
     * true, the process may have died while the call is returning.
     * 检查该binder所在的进程是否仍然存在
     * 如果进程不存在，则返回false。 请注意，如果返回true，则调用返回时
     * 进程可能已经死机。
     */
    public boolean isBinderAlive();
    
    /**
     * Attempt to retrieve a local implementation of an interface
     * for this Binder object.  If null is returned, you will need
     * to instantiate a proxy class to marshall calls through
     * the transact() method.
     */
    public IInterface queryLocalInterface(String descriptor);

    /**
     * Print the object's state into the given stream.
     * 
     * @param fd The raw file descriptor that the dump is being sent to.
     * @param args additional arguments to the dump request.
     */
    public void dump(FileDescriptor fd, String[] args) throws RemoteException;

    /**
     * Like {@link #dump(FileDescriptor, String[])} but always executes
     * asynchronously.  If the object is local, a new thread is created
     * to perform the dump.
     *
     * @param fd The raw file descriptor that the dump is being sent to.
     * @param args additional arguments to the dump request.
     */
    public void dumpAsync(FileDescriptor fd, String[] args) throws RemoteException;

    /**
     * Execute a shell command on this object.  This may be performed asynchrously from the caller;
     * the implementation must always call resultReceiver when finished.
     *
     * @param in The raw file descriptor that an input data stream can be read from.
     * @param out The raw file descriptor that normal command messages should be written to.
     * @param err The raw file descriptor that command error messages should be written to.
     * @param args Command-line arguments.
     * @param resultReceiver Called when the command has finished executing, with the result code.
     * @hide
     */
    public void shellCommand(FileDescriptor in, FileDescriptor out, FileDescriptor err,
            String[] args, ResultReceiver resultReceiver) throws RemoteException;

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

    /**
     * Interface for receiving a callback when the process hosting an IBinder
     * has gone away.
     * 
     * @see #linkToDeath
     */
    public interface DeathRecipient {
        public void binderDied();
    }

    /**
     * Register the recipient for a notification if this binder
     * goes away.  If this binder object unexpectedly goes away
     * (typically because its hosting process has been killed),
     * then the given {@link DeathRecipient}'s
     * {@link DeathRecipient#binderDied DeathRecipient.binderDied()} method
     * will be called.
     * 
     * <p>You will only receive death notifications for remote binders,
     * as local binders by definition can't die without you dying as well.
     * 
     * @throws RemoteException if the target IBinder's
     * process has already died.
     * 
     * @see #unlinkToDeath
     */
    public void linkToDeath(DeathRecipient recipient, int flags)
            throws RemoteException;

    /**
     * Remove a previously registered death notification.
     * The recipient will no longer be called if this object
     * dies.
     * 
     * @return {@code true} if the <var>recipient</var> is successfully
     * unlinked, assuring you that its
     * {@link DeathRecipient#binderDied DeathRecipient.binderDied()} method
     * will not be called;  {@code false} if the target IBinder has already
     * died, meaning the method has been (or soon will be) called.
     * 
     * @throws java.util.NoSuchElementException if the given
     * <var>recipient</var> has not been registered with the IBinder, and
     * the IBinder is still alive.  Note that if the <var>recipient</var>
     * was never registered, but the IBinder has already died, then this
     * exception will <em>not</em> be thrown, and you will receive a false
     * return value instead.
     */
    public boolean unlinkToDeath(DeathRecipient recipient, int flags);
}
```

注释有点多，我们一个一个来

### (一)、类注释

简单的翻译一下：

> - 一个远程对象的基接口，是为高性能而设计的轻量级远程调用机制的核心部分，它不仅用于远程调用，也可以用于进程内调用。这个接口描述了与远程对象进行交互的抽象协议。建议继承Binder类，而不是直接实现这个接口。
>
>    IBinder的主要API是transact()，与它对应另一个方法是Binder.onTransact。第一个方法使你可以向远端的IBinder对象发送调用，第二个方法使你自己的远程对象能够响应接收到的调用。IBinder的API都是同步执行的，比如transact()直到对方的Binder.onTransact()方法调用完成后才返回。而在跨进程的时候，在IPC的帮助下，也是同样的效果。
>
>    通过transact()发送的数据是Parcel，Parcel是一种一般的缓冲区，除了有数据外还带有一些描述它内容的元数据。元数据用于管理IBinder对象的引用，这样就能在缓冲去从一个进程移动到另一个进程时，保存这些引用。这样就保证了当一个IBinder被写入到Parcel并发送到另一个进程中，如果另一个进程把同一个IBinder的引用回发到原来的进程，那么这个原来的进程就能接收到发出的那个IBinder的引用。这种机制使IBinder和Binder像唯一标志符那样在进程间管理。
>
>    系统为每一个进程维持一个存放交互的线程池。这些交互的线程用于派发所有从其他进程发来的IPC调用。例如：当一个IPC从进程A发到进程B，A中那个发出调用的线程就阻塞在transact()中了。进程B中的交互线程池的一个线程池接收了这个调用，它调用Binder.onTransact()，完成后用一个Parcel来作为结果返回。然后进程A中的那个等待线程在收到返回的Parcel才能继续执行。实际上，另一个进程看起来就像当前进程的一个线程，但不是当前进程创建的。
>
>    Binder机制还支持进程间的递归调用。例如，进程A执行自己的IBinder的transact()调用进程B的Binder,而进程B在其Binder.onTransact()中又用transact()向进程A发起调用，那么进程A在等待它发布出的调用返回的同时，还会用Binder.onTransact()响应进程B的transact()。总之，Binder造成的结果就是让我们感觉到跨进程的调用与进程内的调用没有什么区别。
>
>    当操作远程对象的时候，你需要经常查看它们是否有效，有3种方法可以使用： 
>
>   - 1 transact()方法将在IBinder所在的进程不存在时抛出RemoteException异常
>   - 2 如果目标进程不存在，那么调用pingBinder()时返回false\
>   - 3 可以用linkToDeath()方法向IBinder注册一个IBinder.DeathRecipient，在IBinder代表的进程退出时被调用。

### (二)、重要方法注释

**1、queryLocalInterface(String descriptor) 方法**

```dart
    /**
     * Attempt to retrieve a local implementation of an interface
     * for this Binder object.  If null is returned, you will need
     * to instantiate a proxy class to marshall calls through
     * the transact() method.
     */
    public IInterface queryLocalInterface(String descriptor);
```

翻译如下：

> 用于接收一个用于当前binder对象的本地接口的实现。如果返回null，你需要通过transact()方法去实例化一个代理类。

**2、dump(FileDescriptor fd, String[] args)  方法**

```java
    /**
     * Print the object's state into the given stream.
     * 
     * @param fd The raw file descriptor that the dump is being sent to.
     * @param args additional arguments to the dump request.
     */
    public void dump(FileDescriptor fd, String[] args) throws RemoteException;
```

翻译如下：

> 将对象状态打印入给定的数据流中.
>
> - 入参fd:转储发送到的原始文件描述符。
> - 入参args: 转储请求的附加参数

**3、dumpAsync(FileDescriptor fd, String[] args)  方法**

```dart
    /**
     * Like {@link #dump(FileDescriptor, String[])} but always executes
     * asynchronously.  If the object is local, a new thread is created
     * to perform the dump.
     *
     * @param fd The raw file descriptor that the dump is being sent to.
     * @param args additional arguments to the dump request.
     */
    public void dumpAsync(FileDescriptor fd, String[] args) throws RemoteException;
```

翻译如下：

> 类似dump(FileDescriptor, String[])方法，但是总是异步执行。如果对象在本地， 一个新的线程将会被创建去执行这个操作。
>
> - 入参fd:转储发送到的原始文件描述符。
> - 入参args: 转储请求的附加参数

**4、shellCommand(FileDescriptor in, FileDescriptor out, FileDescriptor err,String[] args, ResultReceiver resultReceiver) 方法**

```php
    /**
     * Execute a shell command on this object.  This may be performed asynchrously from the caller;
     * the implementation must always call resultReceiver when finished.
     *
     * @param in The raw file descriptor that an input data stream can be read from.
     * @param out The raw file descriptor that normal command messages should be written to.
     * @param err The raw file descriptor that command error messages should be written to.
     * @param args Command-line arguments.
     * @param resultReceiver Called when the command has finished executing, with the result code.
     * @hide
     */
    public void shellCommand(FileDescriptor in, FileDescriptor out, FileDescriptor err,String[] args, ResultReceiver resultReceiver) throws RemoteException;
```

翻译如下：

> 对此对象执行shell命令。 可以异步执行; 执行完毕后必须始终调用resultReceiver。
>
> - 入参in:可以读取输入数据流的原始文件描述符。
> - 入参out: 正常命令消息应写入的原始文件描述符。
> - 入参err:命令错误消息应写入的原始文件描述符。
> - 入参args: 命令行参数。
> - 入参resultReceiver: 当命令执行结束后，使用结果代码调用。

**5、transact(int code, Parcel data, Parcel reply, int flags)方法**

```dart
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

翻译如下：

> 对对象执行一个通用的操作
>
> - 入参code:操作码必须在FIRST_CALL_TRANSACTION和LAST_CALL_TRANSACTION之间
> - 入参data: 传输给目标的数据。如果你没有传输任何数据，你必须创建一个空的Parcel
> - 入参reply:从目标接收的数据。可以是null如果你对返回的值不感兴趣。
> - 入参flags: 附加操作标志。0是指普通的RPC。或者FLAG_ONEWAY，指单向RPC。

**6、linkToDeath(DeathRecipient recipient, int flags)方法**

```dart
    /**
     * Register the recipient for a notification if this binder
     * goes away.  If this binder object unexpectedly goes away
     * (typically because its hosting process has been killed),
     * then the given {@link DeathRecipient}'s
     * {@link DeathRecipient#binderDied DeathRecipient.binderDied()} method
     * will be called.
     * 
     * <p>You will only receive death notifications for remote binders,
     * as local binders by definition can't die without you dying as well.
     * 
     * @throws RemoteException if the target IBinder's
     * process has already died.
     * 
     * @see #unlinkToDeath
     */
    public void linkToDeath(DeathRecipient recipient, int flags)
            throws RemoteException;
```

翻译如下：

> 注册一个recipient用于提示binder消失。如果binder对象异常地消失(例如由于主进程被杀)，DeathRecipient中的binderDied会被调用.

**7、unlinkToDeath(DeathRecipient recipient, int flags)方法**

```dart
    /**
     * Remove a previously registered death notification.
     * The recipient will no longer be called if this object
     * dies.
     * 
     * @return {@code true} if the <var>recipient</var> is successfully
     * unlinked, assuring you that its
     * {@link DeathRecipient#binderDied DeathRecipient.binderDied()} method
     * will not be called;  {@code false} if the target IBinder has already
     * died, meaning the method has been (or soon will be) called.
     * 
     * @throws java.util.NoSuchElementException if the given
     * <var>recipient</var> has not been registered with the IBinder, and
     * the IBinder is still alive.  Note that if the <var>recipient</var>
     * was never registered, but the IBinder has already died, then this
     * exception will <em>not</em> be thrown, and you will receive a false
     * return value instead.
     */
    public boolean unlinkToDeath(DeathRecipient recipient, int flags);
}
```

翻译如下：

> 删除以前注册的死亡通知。 如果对象死亡，则recipient不再会被调用

### (三)、内部接口注释

**1、DeathRecipient(int code, Parcel data, Parcel reply, int flags)方法**

```csharp
   /**
     * Interface for receiving a callback when the process hosting an IBinder
     * has gone away.
     * 
     * @see #linkToDeath
     */
    public interface DeathRecipient {
        public void binderDied();
    }
```

翻译如下：

> 当持有IBinder进程消失，会回调这个接口

### (四)、总结：

通过上面对IBinder注释，我们大概可以知道以下信息

> - 1、IBindre是远程对象的基接口，不仅可以在跨进程可以调用，也可以在进程内部调用
> - 2、在远程调用的时候，一端用IBinder.transact()发送，另一端用Binder的Binder.onTransact()接受，并且是同步的
> - 3、transact()方法发送的是Parcel。
> - 4、系统为每个进程维护一个进行跨进程调用的线程池。
> - 5、可以使用pingBinder()方法来检测目标进程是否存在
> - 6、可以调用linkToDeath()来向IBinder注册一个IBinder.DeathRecipien。用于目标进程退出时候的提醒。
> - 7、建议继承Binder类，而不是直接实现这个接口。

## 三、Binder与BinderProxy类

### (一)、Binder

```dart
/**
 * Base class for a remotable object, the core part of a lightweight
 * remote procedure call mechanism defined by {@link IBinder}.
 * This class is an implementation of IBinder that provides
 * standard local implementation of such an object.
 *
 * <p>Most developers will not implement this class directly, instead using the
 * <a href="{@docRoot}guide/components/aidl.html">aidl</a> tool to describe the desired
 * interface, having it generate the appropriate Binder subclass.  You can,
 * however, derive directly from Binder to implement your own custom RPC
 * protocol or simply instantiate a raw Binder object directly to use as a
 * token that can be shared across processes.
 *
 * <p>This class is just a basic IPC primitive; it has no impact on an application's
 * lifecycle, and is valid only as long as the process that created it continues to run.
 * To use this correctly, you must be doing so within the context of a top-level
 * application component (a {@link android.app.Service}, {@link android.app.Activity},
 * or {@link android.content.ContentProvider}) that lets the system know your process
 * should remain running.</p>
 *
 * <p>You must keep in mind the situations in which your process
 * could go away, and thus require that you later re-create a new Binder and re-attach
 * it when the process starts again.  For example, if you are using this within an
 * {@link android.app.Activity}, your activity's process may be killed any time the
 * activity is not started; if the activity is later re-created you will need to
 * create a new Binder and hand it back to the correct place again; you need to be
 * aware that your process may be started for another reason (for example to receive
 * a broadcast) that will not involve re-creating the activity and thus run its code
 * to create a new Binder.</p>
 *
 * @see IBinder
 */
public class Binder implements IBinder {
    /*
     * Set this flag to true to detect anonymous, local or member classes
     * that extend this Binder class and that are not static. These kind
     * of classes can potentially create leaks.
     * 通过设置这个标记来检测是否是Binder的匿名内部类，或者Binder的
     * 本地内部类，因为这些类可能会存在潜在的内存泄露  handler里面也
     * 有这段话哦~
     */
    private static final boolean FIND_POTENTIAL_LEAKS = false;
    private static final boolean CHECK_PARCEL_SIZE = false;
    static final String TAG = "Binder";

    /** @hide */
    public static boolean LOG_RUNTIME_EXCEPTION = false; // DO NOT SUBMIT WITH TRUE

    /**
     * Control whether dump() calls are allowed.
     * 控制是否允许dump()方法的调用。
     */
    private static String sDumpDisabled = null;

    /**
     * Global transaction tracker instance for this process.
     * 进程的全局事务跟踪器实例。
     */
    private static TransactionTracker sTransactionTracker = null;

    // Transaction tracking code.
    // 事务跟踪代码

    /**
     * Flag indicating whether we should be tracing transact calls.
     * 标志， 表示我们是否应该跟踪事务调用。
     */
    private static boolean sTracingEnabled = false;

    /**
     * Enable Binder IPC tracing.
     * 启动 Binder IPC 跟踪
     * @hide
     */
    public static void  enableTracing() {
        sTracingEnabled = true;
    };

    /**
     * Disable Binder IPC tracing.
     * 关闭 Binder IPC 跟踪
     * @hide
     */
    public static void  disableTracing() {
        sTracingEnabled = false;
    }

    /**
     * Check if binder transaction tracing is enabled.
     * 检查 Binder的事务跟踪是否已启用
     * @hide
     */
    public static boolean isTracingEnabled() {
        return sTracingEnabled;
    }

    /**
     * Get the binder transaction tracker for this process.
     * 获取此进程的 Binder 事务跟踪器。
     * @hide
     */
    public synchronized static TransactionTracker getTransactionTracker() {
        if (sTransactionTracker == null)
            sTransactionTracker = new TransactionTracker();
        return sTransactionTracker;
    }

    /* mObject is used by native code, do not remove or rename */
    // mObject 会被Native 代码调用，不要删除或重命名
    private long mObject;
    private IInterface mOwner;
    private String mDescriptor;

    /**
     * Return the ID of the process that sent you the current transaction
     * that is being processed.  This pid can be used with higher-level
     * system services to determine its identity and check permissions.
     * If the current thread is not currently executing an incoming transaction,
     * then its own pid is returned.
     * 返回向您发送正在处理的当前事务的进程的ID。 该pid可以与更高级
     * 别的系统服务一起使用，以确定其身份和检查权限。 如果当前线程当
     * 前没有执行传入事务，则返回其自己的pid。
     */
    public static final native int getCallingPid();
    
    /**
     * Return the Linux uid assigned to the process that sent you the
     * current transaction that is being processed.  This uid can be used with
     * higher-level system services to determine its identity and check
     * permissions.  If the current thread is not currently executing an
     * incoming transaction, then its own uid is returned.
     * 将分配给您的Linux uid返回给向您发送正在处理的当前事务的进程。
     * 这个uid可以与更高级别的系统服务一起使用，以确定其身份和检查
     * 权限。 如果当前线程当前没有执行传入事务，则返回其自己的uid。
     */
    public static final native int getCallingUid();

    /**
     * Return the UserHandle assigned to the process that sent you the
     * current transaction that is being processed.  This is the user
     * of the caller.  It is distinct from {@link #getCallingUid()} in that a
     * particular user will have multiple distinct apps running under it each
     * with their own uid.  If the current thread is not currently executing an
     * incoming transaction, then its own UserHandle is returned.
     * 返回分配给发送给正在处理的当前事务的进程的UserHandle。 这是
     * 呼叫者的用户。 与getCallingUid()不同，特定用户将具有多个不同的
     * 应用程序，每个应用程序都具有自己的uid。 如果当前线程当前没有
     * 执行传入事务，则返回其自己的UserHandle。
     */
    public static final UserHandle getCallingUserHandle() {
        return UserHandle.of(UserHandle.getUserId(getCallingUid()));
    }

    /**
     * Reset the identity of the incoming IPC on the current thread.  This can
     * be useful if, while handling an incoming call, you will be calling
     * on interfaces of other objects that may be local to your process and
     * need to do permission checks on the calls coming into them (so they
     * will check the permission of your own local process, and not whatever
     * process originally called you).
     * 重置当前线程的IPC的身份。 如果在处理来电时，您将会调用
     * 其他对象的接口，这些对象可能在本地进程中，并且需要对其中的调
     * 用进行权限检查（不管原来的进程是如何调用你的，他们都将将检查
     * 您自己进程的的访问权限）。
     *  //最后一句话我实在是翻译不好
     * @return Returns an opaque token that can be used to restore the
     * original calling identity by passing it to
     * {@link #restoreCallingIdentity(long)}.
     *
     *  返回一个不透明的token，通过restoreCallingIdentity(long)这个方法
     *  可以恢复原始呼叫的身份标识
     * @see #getCallingPid()
     * @see #getCallingUid()
     * @see #restoreCallingIdentity(long)
     */
    public static final native long clearCallingIdentity();

    /**
     * Restore the identity of the incoming IPC on the current thread
     * back to a previously identity that was returned by {@link
     * #clearCallingIdentity}.
     *  恢复之前当前线程上的传入IPC的身份标识。这个身份标识是由
     *  clearCallingIdentity()方法改变的
     * @param token The opaque token that was previously returned by
     * {@link #clearCallingIdentity}.
     * token 参数是  以前由{@link #clearCallingIdentity}返回的 tocken
     *
     * @see #clearCallingIdentity
     */
    public static final native void restoreCallingIdentity(long token);

    /**
     * Sets the native thread-local StrictMode policy mask.
     *  设置 native层线程的StrictMode策略掩码。
     * <p>The StrictMode settings are kept in two places: a Java-level
     * threadlocal for libcore/Dalvik, and a native threadlocal (set
     * here) for propagation via Binder calls.  This is a little
     * unfortunate, but necessary to break otherwise more unfortunate
     * dependencies either of Dalvik on Android, or Android
     * native-only code on Dalvik.
     *
     *  StrictMode设置保存在两个地方：Java级别的 本地线程  在libcore / Dalvik中进行设置，
     * 和 native的 本地线程则通过Binder调用来设置。 这有点儿不幸，但总
     * 比依赖于Android上的Dalvik或者Android上Dalvik的native-only代码要好
     * @see StrictMode
     * @hide
     */
    public static final native void setThreadStrictModePolicy(int policyMask);

    /**
     * Gets the current native thread-local StrictMode policy mask.
     * 获取当前native 的StrictMode策略掩码
     * @see #setThreadStrictModePolicy
     * @hide
     */
    public static final native int getThreadStrictModePolicy();

    /**
     * Flush any Binder commands pending in the current thread to the kernel
     * driver.  This can be
     * useful to call before performing an operation that may block for a long
     * time, to ensure that any pending object references have been released
     * in order to prevent the process from holding on to objects longer than
     * it needs to.
     * 将当前线程中的Binder命令刷新到内核驱动程序中。这在执行可能会
     * 长时间阻塞的操作之前调用是有用的，因为这样可以确保已经释放了
     * 任何挂起的对象引用，以防止进程持续到比需要的对象更长的时间。
     */
    public static final native void flushPendingCommands();
    
    /**
     * Add the calling thread to the IPC thread pool.  This function does
     * not return until the current process is exiting.
    * 将调用线程添加到IPC线程池。 此方法在当前进程退出之前不返回。
     */
    public static final native void joinThreadPool();

    /**
     * Returns true if the specified interface is a proxy.
     * 如果指定的接口是代理，则返回true。
     * @hide
     */
    public static final boolean isProxy(IInterface iface) {
        return iface.asBinder() != iface;
    }

    /**
     * Call blocks until the number of executing binder threads is less
     * than the maximum number of binder threads allowed for this process.
     * 调用块直到执行绑定线程数量小于此进程允许的绑定线程的最大数量。
     * @hide
     */
    public static final native void blockUntilThreadAvailable();

    /**
     * Default constructor initializes the object.
     */
    public Binder() {
        init();

        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Binder> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Binder class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
    }
    
    /**
     * Convenience method for associating a specific interface with the Binder.
     * After calling, queryLocalInterface() will be implemented for you
     * to return the given owner IInterface when the corresponding
     * descriptor is requested.
     */
    public void attachInterface(IInterface owner, String descriptor) {
        mOwner = owner;
        mDescriptor = descriptor;
    }
    
    /**
     * Default implementation returns an empty interface name.
     */
    public String getInterfaceDescriptor() {
        return mDescriptor;
    }

    /**
     * Default implementation always returns true -- if you got here,
     * the object is alive.
     */
    public boolean pingBinder() {
        return true;
    }

    /**
     * {@inheritDoc}
     *
     * Note that if you're calling on a local binder, this always returns true
     * because your process is alive if you're calling it.
     */
    public boolean isBinderAlive() {
        return true;
    }
    
    /**
     * Use information supplied to attachInterface() to return the
     * associated IInterface if it matches the requested
     * descriptor.
     */
    public IInterface queryLocalInterface(String descriptor) {
        if (mDescriptor.equals(descriptor)) {
            return mOwner;
        }
        return null;
    }

    /**
     * Control disabling of dump calls in this process.  This is used by the system
     * process watchdog to disable incoming dump calls while it has detecting the system
     * is hung and is reporting that back to the activity controller.  This is to
     * prevent the controller from getting hung up on bug reports at this point.
     * @hide
     *
     * @param msg The message to show instead of the dump; if null, dumps are
     * re-enabled.
     */
    public static void setDumpDisabled(String msg) {
        synchronized (Binder.class) {
            sDumpDisabled = msg;
        }
    }

    /**
     * Default implementation is a stub that returns false.  You will want
     * to override this to do the appropriate unmarshalling of transactions.
     *
     * <p>If you want to call this, call transact().
     */
    protected boolean onTransact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
        if (code == INTERFACE_TRANSACTION) {
            reply.writeString(getInterfaceDescriptor());
            return true;
        } else if (code == DUMP_TRANSACTION) {
            ParcelFileDescriptor fd = data.readFileDescriptor();
            String[] args = data.readStringArray();
            if (fd != null) {
                try {
                    dump(fd.getFileDescriptor(), args);
                } finally {
                    IoUtils.closeQuietly(fd);
                }
            }
            // Write the StrictMode header.
            if (reply != null) {
                reply.writeNoException();
            } else {
                StrictMode.clearGatheredViolations();
            }
            return true;
        } else if (code == SHELL_COMMAND_TRANSACTION) {
            ParcelFileDescriptor in = data.readFileDescriptor();
            ParcelFileDescriptor out = data.readFileDescriptor();
            ParcelFileDescriptor err = data.readFileDescriptor();
            String[] args = data.readStringArray();
            ResultReceiver resultReceiver = ResultReceiver.CREATOR.createFromParcel(data);
            try {
                if (out != null) {
                    shellCommand(in != null ? in.getFileDescriptor() : null,
                            out.getFileDescriptor(),
                            err != null ? err.getFileDescriptor() : out.getFileDescriptor(),
                            args, resultReceiver);
                }
            } finally {
                IoUtils.closeQuietly(in);
                IoUtils.closeQuietly(out);
                IoUtils.closeQuietly(err);
                // Write the StrictMode header.
                if (reply != null) {
                    reply.writeNoException();
                } else {
                    StrictMode.clearGatheredViolations();
                }
            }
            return true;
        }
        return false;
    }

    /**
     * Implemented to call the more convenient version
     * {@link #dump(FileDescriptor, PrintWriter, String[])}.
     */
    public void dump(FileDescriptor fd, String[] args) {
        FileOutputStream fout = new FileOutputStream(fd);
        PrintWriter pw = new FastPrintWriter(fout);
        try {
            doDump(fd, pw, args);
        } finally {
            pw.flush();
        }
    }

    void doDump(FileDescriptor fd, PrintWriter pw, String[] args) {
        final String disabled;
        synchronized (Binder.class) {
            disabled = sDumpDisabled;
        }
        if (disabled == null) {
            try {
                dump(fd, pw, args);
            } catch (SecurityException e) {
                pw.println("Security exception: " + e.getMessage());
                throw e;
            } catch (Throwable e) {
                // Unlike usual calls, in this case if an exception gets thrown
                // back to us we want to print it back in to the dump data, since
                // that is where the caller expects all interesting information to
                // go.
                pw.println();
                pw.println("Exception occurred while dumping:");
                e.printStackTrace(pw);
            }
        } else {
            pw.println(sDumpDisabled);
        }
    }

    /**
     * Like {@link #dump(FileDescriptor, String[])}, but ensures the target
     * executes asynchronously.
     */
    public void dumpAsync(final FileDescriptor fd, final String[] args) {
        final FileOutputStream fout = new FileOutputStream(fd);
        final PrintWriter pw = new FastPrintWriter(fout);
        Thread thr = new Thread("Binder.dumpAsync") {
            public void run() {
                try {
                    dump(fd, pw, args);
                } finally {
                    pw.flush();
                }
            }
        };
        thr.start();
    }

    /**
     * Print the object's state into the given stream.
     * 
     * @param fd The raw file descriptor that the dump is being sent to.
     * @param fout The file to which you should dump your state.  This will be
     * closed for you after you return.
     * @param args additional arguments to the dump request.
     */
    protected void dump(FileDescriptor fd, PrintWriter fout, String[] args) {
    }

    /**
     * @param in The raw file descriptor that an input data stream can be read from.
     * @param out The raw file descriptor that normal command messages should be written to.
     * @param err The raw file descriptor that command error messages should be written to.
     * @param args Command-line arguments.
     * @param resultReceiver Called when the command has finished executing, with the result code.
     * @throws RemoteException
     * @hide
     */
    public void shellCommand(FileDescriptor in, FileDescriptor out, FileDescriptor err,
            String[] args, ResultReceiver resultReceiver) throws RemoteException {
        onShellCommand(in, out, err, args, resultReceiver);
    }

    /**
     * Handle a call to {@link #shellCommand}.  The default implementation simply prints
     * an error message.  Override and replace with your own.
     * <p class="caution">Note: no permission checking is done before calling this method; you must
     * apply any security checks as appropriate for the command being executed.
     * Consider using {@link ShellCommand} to help in the implementation.</p>
     * @hide
     */
    public void onShellCommand(FileDescriptor in, FileDescriptor out, FileDescriptor err,
            String[] args, ResultReceiver resultReceiver) throws RemoteException {
        FileOutputStream fout = new FileOutputStream(err != null ? err : out);
        PrintWriter pw = new FastPrintWriter(fout);
        pw.println("No shell command implementation.");
        pw.flush();
        resultReceiver.send(0, null);
    }

    /**
     * Default implementation rewinds the parcels and calls onTransact.  On
     * the remote side, transact calls into the binder to do the IPC.
     */
    public final boolean transact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
        if (false) Log.v("Binder", "Transact: " + code + " to " + this);

        if (data != null) {
            data.setDataPosition(0);
        }
        boolean r = onTransact(code, data, reply, flags);
        if (reply != null) {
            reply.setDataPosition(0);
        }
        return r;
    }
    
    /**
     * Local implementation is a no-op.
     */
    public void linkToDeath(DeathRecipient recipient, int flags) {
    }

    /**
     * Local implementation is a no-op.
     */
    public boolean unlinkToDeath(DeathRecipient recipient, int flags) {
        return true;
    }
    
    protected void finalize() throws Throwable {
        try {
            destroy();
        } finally {
            super.finalize();
        }
    }

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

    private native final void init();
    private native final void destroy();

    // Entry point from android_util_Binder.cpp's onTransact
    private boolean execTransact(int code, long dataObj, long replyObj,
            int flags) {
        Parcel data = Parcel.obtain(dataObj);
        Parcel reply = Parcel.obtain(replyObj);
        // theoretically, we should call transact, which will call onTransact,
        // but all that does is rewind it, and we just got these from an IPC,
        // so we'll just call it directly.
        boolean res;
        // Log any exceptions as warnings, don't silently suppress them.
        // If the call was FLAG_ONEWAY then these exceptions disappear into the ether.
        try {
            res = onTransact(code, data, reply, flags);
        } catch (RemoteException|RuntimeException e) {
            if (LOG_RUNTIME_EXCEPTION) {
                Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
            }
            if ((flags & FLAG_ONEWAY) != 0) {
                if (e instanceof RemoteException) {
                    Log.w(TAG, "Binder call failed.", e);
                } else {
                    Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
                }
            } else {
                reply.setDataPosition(0);
                reply.writeException(e);
            }
            res = true;
        } catch (OutOfMemoryError e) {
            // Unconditionally log this, since this is generally unrecoverable.
            Log.e(TAG, "Caught an OutOfMemoryError from the binder stub implementation.", e);
            RuntimeException re = new RuntimeException("Out of memory", e);
            reply.setDataPosition(0);
            reply.writeException(re);
            res = true;
        }
        checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
        reply.recycle();
        data.recycle();

        // Just in case -- we are done with the IPC, so there should be no more strict
        // mode violations that have gathered for this thread.  Either they have been
        // parceled and are now in transport off to the caller, or we are returning back
        // to the main transaction loop to wait for another incoming transaction.  Either
        // way, strict mode begone!
        StrictMode.clearGatheredViolations();

        return res;
    }
}
```

**1、类注释**

> - 远程对象的基类，由IBinder定义的轻量级远程调用机制的核心部分。这个类是IBinder的实现类。它提供了这种对象的标准本地实现。
> - 大多数开发人员不会直接使用这个类，而是使用 AIDL工具来实现这个接口，使其生成适当的Binder子类。 但是，您可以直接从Binder派生自己的自定义RPC协议，也可以直接实例化一个原始的Binder对象，把它当做一个token，来进行跨进程通信。
> - 这个Binder类是一个基础的IPC原生类，它对application的生命周期没有影响的，它仅当创建它的进程还活着的时候才有效。所以为了正确的使用它，你必须在一个顶级的app组件里明确地让系统知道，
>    这个Binder类是一个基础的IPC原生类，它对applicant的生命周期没有影响，它仅当创建它的进程还活着的时候才有效。所以为了正确地使用它，你必须在一个顶级app组件(例如service、activity或者ContentProvider)里明确地让系统知道，您的进程应该保持运行。
> - 你必须牢记你的进程可能会消失的情况，如果发生了这种情况，你必须在进程重启的时候创建一个新的Binder，并且关联这个进程。例如，如果你在Activity里面使用了Binder，你的Activity进程可能会被杀死，过了一会后，如果activity比重新启动了，这时候你要重新创建的一个新的Binder，并且把这个心的Binder放回之前的位置。你也要注意到是，你的进程可能因为一些原因(比如接收broadcast)而启动，在这种情况下，是不需要重新创建Activity的，这时候就需要运行其他的一些代码去创建Binder对象。

**2、关于Binder的构造函数**

Binder就提供一个默认的构造函数,代码如下

```java
    /**
     * Default constructor initializes the object.
     */
    public Binder() {
        init();

        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Binder> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Binder class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
    }
    private native final void init();
```

> - 首先是执行init()方法，init()是native层的，这里就暂不跟踪了。
> - 判断是否是匿名内部类，或者内部类，如果是的话，打印日志。

通过上面我们知道Binder这个类的核心构造函数是在native实现的。

**3、Binder的重要方法**

**(1)、attachInterface(IInterface, String)方法**

```dart
    /**
     * Convenience method for associating a specific interface with the Binder.
     * After calling, queryLocalInterface() will be implemented for you
     * to return the given owner IInterface when the corresponding
     * descriptor is requested.
     * 将特定接口与Binder相关联的快捷方法。 调用之后，将会实现
     * queryLocalInterface(), 当你请求相应的描述符时，queryLocalInterface()
     * 将返回给定的所有者IInterface。
     */
    public void attachInterface(IInterface owner, String descriptor) {
        mOwner = owner;
        mDescriptor = descriptor;
    }
```

**(2)、getInterfaceDescriptor()方法**

```cpp
    /**
     * Default implementation returns an empty interface name.
     * 默认实现返回一个空的接口名称。
     */
    public String getInterfaceDescriptor() {
        return mDescriptor;
    }
```

**(3)、pingBinder()方法**

```cpp
    /**
     * Default implementation always returns true -- if you got here,
     * the object is alive.
     * 默认实现总是返回true - 如果你走到这里，对象是活着的。
     */
    public boolean pingBinder() {
        return true;
    }
```

**(4)、isBinderAlive()方法**

```dart
    /**
     * {@inheritDoc}
     *
     * Note that if you're calling on a local binder, this always returns true
     * because your process is alive if you're calling it.
     * 请注意，如果您正在调用本地的Binder，则始终返回true
     * 如果你能调用他，则你的进程一定是活着的。
     */
    public boolean isBinderAlive() {
        return true;
    }
```

**(5)、queryLocalInterface(String)方法**

```dart
    /**
     * Use information supplied to attachInterface() to return the
     * associated IInterface if it matches the requested
     * descriptor.
     * 如果提供的描述符和之前关联的IInterface(通过attachInterface()
     * 方法进行关联)的描述符一致，则返回相对应的IInterface
     */
    public IInterface queryLocalInterface(String descriptor) {
        if (mDescriptor.equals(descriptor)) {
            return mOwner;
        }
        return null;
    }
```

**(6)、setDumpDisabled(String)方法**

```dart
   /**
     * Control disabling of dump calls in this process.  This is used by the system
     * process watchdog to disable incoming dump calls while it has detecting the system
     * is hung and is reporting that back to the activity controller.  This is to
     * prevent the controller from getting hung up on bug reports at this point.
     * @hide
     *
     * @param msg The message to show instead of the dump; if null, dumps are
     * re-enabled.
     */
    public static void setDumpDisabled(String msg) {
        synchronized (Binder.class) {
            sDumpDisabled = msg;
        }
    }
```

### (二)、BinderProxy

```java
final class BinderProxy implements IBinder {
    public native boolean pingBinder();
    public native boolean isBinderAlive();

    public IInterface queryLocalInterface(String descriptor) {
        return null;
    }

    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        if (Binder.isTracingEnabled()) { Binder.getTransactionTracker().addTrace(); }
        return transactNative(code, data, reply, flags);
    }

    public native String getInterfaceDescriptor() throws RemoteException;
    public native boolean transactNative(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException;
    public native void linkToDeath(DeathRecipient recipient, int flags)
            throws RemoteException;
    public native boolean unlinkToDeath(DeathRecipient recipient, int flags);

    public void dump(FileDescriptor fd, String[] args) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeFileDescriptor(fd);
        data.writeStringArray(args);
        try {
            transact(DUMP_TRANSACTION, data, reply, 0);
            reply.readException();
        } finally {
            data.recycle();
            reply.recycle();
        }
    }
    
    public void dumpAsync(FileDescriptor fd, String[] args) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeFileDescriptor(fd);
        data.writeStringArray(args);
        try {
            transact(DUMP_TRANSACTION, data, reply, FLAG_ONEWAY);
        } finally {
            data.recycle();
            reply.recycle();
        }
    }

    public void shellCommand(FileDescriptor in, FileDescriptor out, FileDescriptor err,
            String[] args, ResultReceiver resultReceiver) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeFileDescriptor(in);
        data.writeFileDescriptor(out);
        data.writeFileDescriptor(err);
        data.writeStringArray(args);
        resultReceiver.writeToParcel(data, 0);
        try {
            transact(SHELL_COMMAND_TRANSACTION, data, reply, 0);
            reply.readException();
        } finally {
            data.recycle();
            reply.recycle();
        }
    }

    BinderProxy() {
        mSelf = new WeakReference(this);
    }
    
    @Override
    protected void finalize() throws Throwable {
        try {
            destroy();
        } finally {
            super.finalize();
        }
    }
    
    private native final void destroy();
    
    private static final void sendDeathNotice(DeathRecipient recipient) {
        if (false) Log.v("JavaBinder", "sendDeathNotice to " + recipient);
        try {
            recipient.binderDied();
        }
        catch (RuntimeException exc) {
            Log.w("BinderNative", "Uncaught exception from death notification",
                    exc);
        }
    }
   
    final private WeakReference mSelf;
    private long mObject;
    private long mOrgue;
}
```

## 四、总结

所以大体的结构类图图下图:



![img](https:////upload-images.jianshu.io/upload_images/5713484-37292c20da8b0195.png?imageMogr2/auto-orient/strip|imageView2/2/w/1012/format/webp)

Java层的Binder对象模型.png







# Android跨进程通信IPC之Binder框架

本文的主要内容如下：

> - 1 Android整体架构
> - 2 IPC原理
> - 3 Binder简介
> - 4 Binder通信机制
> - 5 Binder协议

## 一、Android整体架构

### (一) Android的整体架构

为了让大家更好的理解Binder机制，我们先来看下Android的整体架构。因为这样大家就知道在Android架构中Binder出于什么地位。
 用一下官网上的图片



![img](https:////upload-images.jianshu.io/upload_images/5713484-bd60fdee4dc65c71.png?imageMogr2/auto-orient/strip|imageView2/2/w/486/format/webp)

Android架构.png

从下往上依次为：

> - 内核层：Linux内核和各类硬件设备的驱动，这里需要注意的是，Binder IPC驱动也是这一层的实现，比较特殊。
> - 硬件抽象层：封装"内核层"硬件驱动，提供可供"系统服务层"调用的统一硬件接口
> - 系统服务层：提供核心服务，并且提供可供"应用程序框架层"调用的接口
> - Binder IPC 层：作为"系统服务层"与"应用程序框架层"的IPC桥梁，相互传递接口调用的数据，实现跨进程的通信。
> - 应用程序框架层：这一层可以理解为Android SDK，提供四大组件，View绘制等平时开发中用到的基础部件

### (二) Android的架构解析

> 在一个大的项目里面，分层是非常重要的，处于最底层的接口最具有"通用性"，接口颗粒度最细，越往上层通用性降低。理论上来说上面每一层都可以"开放"给开发者调用，例如开发者可以直接调用硬件抽象层的接口去操作硬件，或者直接调用系统服务层中的接口去直接操作系统服务，甚至像Windows开发一样，开发者可以在内核层写程序，运行在内核中。不过开放带来的问题就是开发者权利太大，对于系统的稳定性是没有任何好处的，一个病毒制作者搞一个内核层的病毒出来，系统可能就永远起不来。所以谷歌的做法是将开发者的权利收拢到"应用程序框架层"，开发者只能调用这一层的接口。

在上面的层次中，内核层与硬件抽象层均用C/C++实现，系统服务层是以Java实现，硬件抽象层编译为so文件，以JNI的形式供系统服务层使用。系统服务层中的服务随着系统启动而启动，只要不关机，就会一直运行。这些服务干什么事情？其实很简单，就是完成一个手机有的核心功能如短信的收发管、电话的接听、挂断以及应用程序的包管理、Activity的管理等等。每一个服务均运行在一个独立的进程中，因此是以Java实现，所以本质上来说是运行在一个独立进程的Dalvik虚拟机中。那么问题来了，开发者的App也运行在一个独立的进程空间中，如果调用到系统的服务层中的接口？答案是IPC(Inter-Process Communication)，进程间通讯是和RPC(Remote Procedure Call)不一样的，实现原理也不一样。每一个系统服务在应用框架层都有一个Manager与之对应，方便开发者调用其相关功能，具体关系如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-f914d2f622dcf2bc.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

调用关系.png

### (三)、总结一下：

> - Android 从下而上分了内核层、硬件抽象层、系统服务层、Binder IPC 层、应用程序框架层
> - Android 中"应用程序框架层"以 SDK 的形式开放给开发者使用，"系统服务层" 中的核心服务随系统启动而运行，通过应用层序框架层提供的 Manager 实时为应用程序提供服务调用。系统服务层中每一个服务运行在自己独立的进程空间中，应用程序框架层中的 Manager 通过 Binder IPC 的方式调用系统服务层中的服务。

## 二、IPC原理

从进程的角度来看IPC机制

![img](https:////upload-images.jianshu.io/upload_images/5713484-b12db11921797839.png?imageMogr2/auto-orient/strip|imageView2/2/w/643/format/webp)

image.png



每个Android进程，只能运行在自己的进程所拥有的虚拟地址空间，如果是32位的系统，对应一个4GB的虚拟地址空间，其中3GB是用户空，1GB是内核空间，而内核空间的大小是可以通过参数配置的。对于用户空间，不同进程之间彼此是不能共享的，而内核空间确实可以共享的。Client进程与Server进程通信，恰恰是利用进程间可共享的内核内空间来完成底层通信工作的，Client端与Server端进程往往采用ioctl等方法跟内核空间的驱动进行。

## 三、Binder综述

### (一) Binder简介

**1、Binder的由来**

> 简单的说，Binder是Android平台上的一种跨进程通信技术。该技术最早并不是谷歌公司提出的，它前身是Be Inc公司开发的OpenBinder，而且在Palm中也有应用。后来OpenBinder的作者Dianne Hackborn加入了谷歌公司，并负责Android平台开发的工作，所以把这项技术也带进了Android。

我们知道，在Android的应用层次上，基本上已经没有了进程的概念了，然后在具体的实现层次上，它毕竟还是要构建一个个进程之上的。实际上，在Android内部，哪些支持应用的组件往往会身处不同的继承，那么应用的底层必然会涉及大批的跨进程通信，为了保证了通信的高效性，Android提供了Binder机制。

**2、什么是Binder**

让我们从四个维度来看Binder，这样会让大家对理解Binder机制更有帮助

> - 1 从来类的角度来说，Binder就是Android的一个类，它继承了IBinder接口
> - 2 从IPC的角度来说，Binder是Android中的一个中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux中没有(由于耦合性太强，而Linux没有接纳)
> - 3 从Android Framework角度来说，Binder是ServiceManager连接各种Manager(ActivityManager、WindowManager等)和相应的ManagerService的桥梁
> - 4 从Android应用层的角度来说，Binder是客户端和服务端进行通信的媒介，当你bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

**3、Binder机制的意义**

Binder机制具有两层含义：

> - 1 是一种跨进程通信的方式(IPC)
> - 2 是一种远程过程调用方式(PRC)

而从实现的角度来说，Binder核心被实现成一个Linux驱动程序，并运行于内核态。这样它才能具有强大的跨进程访问能力。

**4、和传统IPC机制相比，谷歌为什么采用Binder**

我们先看下Linux中的IPC通信机制:

> - 1、传统IPC：匿名管道(PIPE)、信号(signal)、有名管道(FIFO)
> - 2、AT&T Unix：共享内存，信号量，消息队列
> - 3、BSD Unix：Socket

关于这块如果大家不了解，请看前面的文章。

虽然Android继承Linux内核，但是Linux与Android通信机制是不同的。Android中有大量的C/S(Client/Server)应用方式，这就要求Android内部提供IPC方法，而如果采用Linux所支持的进程通信方式有两个问题：性能和安全性。那

> - 性能：目前Linux支持的IPC包括传统的管道，System V IPC(包括消息队列/共享内存/信号量)以及socket，但是只有socket支持Client/Server的通信方式，由于socket是一套通用当初网络通信方式，其效率低下，且消耗比较大(socket建立连接过程和中断连接过程都有一定的开销)，明显在手机上不适合大面积使用socket。而消息队列和管道采用"存储-转发" 方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存中拷贝到接收方缓存中，至少有两次拷贝过程。共享内存虽然无需拷贝，但控制复杂，难以使用。
> - 安全性：在安全性方面，Android作为一个开放式，拥有众多开发者的平台，应用程序的来源广泛，确保智能终端的安全是非常重要的。终端用户不希望从网上下载的程序在不知情的情况下偷窥隐私数据，连接无线网络，长期操作底层设备导致电池很快耗尽的情况。传统IPC没有任何安全措施，完全依赖上层协议来去报。首先传统IPC的接受方无法获取对方进程可靠的UID/PID(用户ID/进程ID)，从而无法鉴别对方身份。Android为每个安装好的应用程序分配了自己的UID，故进程的UID是鉴别进程的身份的重要标志。使用传统IPC只能由用户在数据包里填入UID/PID，但这样不可靠，容易被恶意程序利用。可靠的身份标记只由IPC机制本身在内核中添加。其次传统IPC访问接入点是开放的，无法建立私有通道。比如命名管道、system V的键值，socket的ip地址或者文件名都是开放的，只要知道这些接入点的程序都可以对端建立连接，不管怎样都无法阻止恶意程序通过接收方地址获得连接。

基于以上原因，Android需要建立一套新的IPC机制来满足系统对通信方式，传输性能和安全性的要求，所以就有了Binder。Binder基于Client/Server通信模式，传输过程只需要一次拷贝，为发送发添加UID/PID身份，鸡翅实名Binder也支持匿名Binder，安全性高。下图为Binder通信过程示例：

![img](https:////upload-images.jianshu.io/upload_images/5713484-235dbffbd3b4e3fa.png?imageMogr2/auto-orient/strip|imageView2/2/w/785/format/webp)

Binder通信过程.png

> - 相比于传统的跨进程通信手段，通信双方必须要处理线程同步，内存管理等问题，工作量大，而且问题多，就像我们前面介绍的传统IPC 命名管道(FIFO) 信号量(semaphore) 消息队列已经从Android中去掉了，同其他IPC相比，Socket是一种比较成熟的通信手段了，同步控制也很容易实现。Socket用于网络通信非常合适，但是用于进程间通信就效率很低。
> - Android在架构上一直希望模糊进程的概念，取而代之以组件的概念。应用也不需要关心组件存放的位置、组件运行在那个进程中、组件的生命周期等问题。随时随地的，只要拥有Binder对象，就能使用组件的功能。Binder就像一张网，将整个系统的组件，跨进程和线程的组织在一起。

Binder是整个系统的运行的中枢。Android在进程间传递数据使用共享内存的方式，这样数据只需要复制一次就能从一个进程到达另一个进城了(前面文章说了，一般IPC都需要两步，第一步用户进程复制到内核，第二步再从内核复制到服务进程。)

> PS: 整个Androdi系统架构中，虽然大量采用了Binder机制作为IPC(进程间通信)方式，但是也存在部分其他的IPC方式，比如Zygote通信就是采用socket。

**5、Binder在Service服务中的作用**

在Android中，有很多Service都是通过Binder来通信的，比如MediaService名下的众多Service：

> - AudioFlinger音频核心服务
> - AudioPolicyService：音频策略相关的重要服务
> - MediaPlayerService:多媒体系统中的重要服务
> - CarmeraService:有关摄像/照相的重要服务

那具体是怎么应用或者通信机制是什么那？那就让我们来详细了解下

### (二)、总结

> Android Binder 是在OpenBinder上定制实现的。原先的OpenBinder 框架现在已经不再继续开发，所以也可以说Android让原先的OpenBinder得到了重生。Binder是Android上大量使用的IPC(Inter-process communication，进程间通讯)机制。无论是应用程序对系统服务的请求，还是应用程序自身提供对外服务，都需要使用Binder。

**整体架构**

![img](https:////upload-images.jianshu.io/upload_images/5713484-b6d551adc5cd602c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

整体架构.png

从图中可以看出，Binder的实现分为这几层，按照大的框架理解是

> - framewor层 
>   - java   层
>   - jni 层
>   - native/ C++层
> - linux驱动层  c语言

让我们来仔细研究下。

> - 其中Linux驱动层位于Linux内核中，它提供了最底层的数据传递，对象标示，线程管理，通过调用过程控制等功能。驱动层其实是Binder机制的核心。
> - Framework层以Linux驱动层为基础，提供了应用开发的基础设施。Framework层既包含了C++部分的实现，也包含了Java基础部分的实现。为了能将C++ 的实现复用到Java端，中间通过JNI进行衔接。

开发者可以在Framework之上利用Binder提供的机制来进行具体的业务逻辑开发。其实不仅仅是第三方开发者，Android系统中本身也包含很多系统服务都是基于Binder框架开发的。其中Binder框架是典型的C/S架构。所以在后面中， 我们把服务的请求方称为Client，服务的实现方称之Server。Clinet对于Server的请求会经由Binder驱动框架由上至下传递到内核的Binder驱动中，请求中包含了Client将要调用的命令和参数。请求到了Binder驱动以后，在确定了服务的提供方之后，再讲从下至上将请求传递给具体的服务。如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/5713484-61c89cf6753c0f2a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder调用.png

如果大家对网络协议有所了解的话，其实会发现整个数据的传递过程和网络协议如此的相似。

## 四、Binder通信机制

Android内部采用C/S架构。而Binder通信也是采用C/S架构。那我们来看下Binder在C/S的中的流程。

### (一) Binder在C/S中的流程

如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-27da6c30d6c832af.png?imageMogr2/auto-orient/strip|imageView2/2/w/435/format/webp)

Binder流程.png



具体流程如下：

> - 1、相应的Service需要注册服务。Service作为很多Service的拥有者，当它想向Client提供服务时，得先去Service Manager(以后缩写成SM)那儿注册自己的服务。Server可以向SM注册一个或者多个服务。
> - 2、Client申请服务。Client作为Service的使用者，当他想使用服务时，得向SM申请自己所需要的服务。Client可以申请一个或者多个服务。
> - 3、当Client申请服务成功后，Client就可以使用服务了。

SM一方面管理Server所提供的服务，同时又响应Client的请求并为之分配响应的服务。扮演角色相当于月老，两边前线。这种通信方式的好处是：一方面，service和Client请求便于管理，另一方面在应用程序开发时，只需要为Client建立到Server的连接即可，这样只需要花很少的时间成本去实现Server的相应功能。那么Binder与这个通信有什么关系？其实三者的通信方式就是Binder机制(比如Server向SM注册服务，使用Binder通信；Client申请请求也是Binder通信。)

> PS:注意这里的ServiceManager是指Nativie层的ServiceManager(C++)，并非是framework层的ServiceManager(Java)。ServiceManager是整个Binder通信机制的大管家，是Android进程间通信机制的守护进程。

### (二)Binder通信整体框架

这里先提前和大家说下，后面我们会不断的提及两个概念，一个是Server，还有一个是Service，我这里先强调下，Server是Server，Service是Service，大家不要混淆，一个Server下面可能有很多Service，但是一个Servcie也只能隶属于一个Server。下面我们将从三个角度来看Binder框架，这样

**1、从内核和用户空间的角度来看**

Binder通信模型如下：



![img](https:////upload-images.jianshu.io/upload_images/5713484-a2f22475b603fe44.png?imageMogr2/auto-orient/strip|imageView2/2/w/798/format/webp)

Binder通信整体框架.png



我们可以发现：

> - 1、Client和Server是存在于用户空间
> - 2、Client和Server通信实现是由Binder驱动在内核的实现
> - 3、SM作为守护进程，处理客户端请求，管理所有服务

如果大家不好理解上面的意思，我们可以把SM理解成为DNS服务器，那么Binder Driver就相当于路由的功能。这里就涉及到Client和Server是如何通信的问题。

**2、从Android的层级的角度**

如下图：(注意图片的右边)

![img](https:////upload-images.jianshu.io/upload_images/5713484-a967860ae78d6e7b.png?imageMogr2/auto-orient/strip|imageView2/2/w/921/format/webp)

Binder原理.png

图中Client/Server/ServiceManager之间的相互通信都是基于Binder机制。图中Clinet/Server/ServiceManager之间交互都是虚线表示，是由于他们彼此之间不直接交互，都是通过Binder驱动进行交互，从而实现IPC通信方式。其中Binder驱动位于内核空间，Client/Server/ServiceManager可以看做是Android平台的基础架构。而Client和Server是Android的应用层，开发人员只需要自定义实现client、Server端，借助Android的基本平台架构就可以直接进行IPC通信。

**3、从Binder的架构角度来看**

如下图:



![img](https:////upload-images.jianshu.io/upload_images/5713484-8beb526a979a9fbb.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder架构.png

同样

> Binder IPC 属于 C/S 结构，Client 部分是用户代码，用户代码最终会调用 Binder Driver 的 transact 接口，Binder Driver 会调用 Server，这里的 Server 与 service 不同，可以理解为 Service 中 onBind 返回的 Binder 对象，请注意区分下。

Client端：用户需要实现的代码，如 AIDL 自动生成的接口类
 Binder Driver：在内核层实现的 Driver
 Server端：这个 Server 就是 Service 中 onBind 返回的 IBinder 对象
 需要注意的是，上面绿色的色块部分都是属于用户需要实现的部分，而蓝色部分是系统去实现了。也就是说 Binder Driver 这块并不需要知道，Server 中会开启一个线程池去处理客户端调用。为什么要用线程池而不是一个单线程队列呢？试想一下，如果用单线程队列，则会有任务积压，多个客户端同时调用一个服务的时候就会有来不及响应的情况发生，这是绝对不允许的。

对于调用 Binder Driver 中的 transact 接口，客户端可以手动调用，也可以通过 AIDL 的方式生成的代理类来调用，服务端可以继承 Binder 对象，也可以继承 AIDL 生成的接口类的 Stub 对象。这些细节下面继续接着说，这里暂时不展开。

> 切记，这里 Server 的实现是线程池的方式，而不是单线程队列的方式，区别在于，单线程队列的话，Server 的代码是线程安全的，线程池的话，Server 的代码则不是线程安全的，需要开发者自己做好多线程同步。

### (三)、Binder流程

**1、Server向SM注册服务**

![img](https:////upload-images.jianshu.io/upload_images/5713484-4b933c042f2712d7.png?imageMogr2/auto-orient/strip|imageView2/2/w/682/format/webp)

注册.png

> - 1、首先  XXServer(XXX代表某个)在自己的进程中向Binder驱动申请创建一个XXXService的Binder实体。
> - 2、Binder驱动为这个XXXService创建位于内核中的Binder实体节点以及Binder的引用，注意，是将名字和新建的引用打包传递给SM(实体没有传给SM)，通知SM注册一个名叫XXX的Service。
> - 3、SM收到数据包后，从中取出XXXService名字和引用，填入一张查找表中
> - 4、此时，如果有Client向SM发送申请服务XXXService的请求，那么SM就可以查找表中该Service的Binder引用，并把BInder引用(XXXBpBinder返回给Client)

在进一步了解Binder通信机制之前，我们先弄清楚几个概念。

> - 引用和实体。这里，对于一个用于通信的实体(可以理解为真实空间的Object)，可以额有多个该实体的引用(没有真实空间，可以理解成实体的一个链接，操作引用就可以操作对应链接上的实体)。如果一个进程持有某个实体，其他进程也想操作该实体，最高效的做法是去获取该实体的引用，再去操作这个引用。
> - 有些资源也把实体成本本地对象，应用称为远程对象。所以也可以这么理解：应用是从本地进程发送给其他进程操作实体之用，所以有本地和远程对象之名。

为了大家在后面更好的理解，这里补充几个概念

> -  **Binder实体对** :Binder实体对象就是Binder实体对象就是Binder服务的提供者。一个提供Binder服务的类必须继承BBinder类，因此，有时为了强调对象类型，也用"BBinder对象"来代替"Binder实体对象"。
> -  **Binder引用对象** :Binder引用对象是Binder实体对象在客户进程的代表，每个引用对象的类型都是BpBiner类，同样可以用名称"BpBinder对象"来代替"Binder引用对象"。
> -  **Binder代理对象** ：代理对象也成为接口对象，它主要是为了客户端的上层应用提供接口服务，从IInterface类派生。它实现了Binder服务的函数接口，当然只是一个转调的空壳。通过代理对象，应用能像使用本地对象一样使用远端实体对象提供服务。
> -  **IBiner对象** ：BBinder和BpBinder类是从IBinder类中继承来。在很多场合，不需要刻意地去区分实体对象和引用对象，这时候也可以统一使用"IBinder对象"来统一称呼他们。
> -  **Binder代理对象** 主要是和应用程序打交道，将Binder代理对象和Binder引用对应(BpBinder对象)分开的好处是代理对象可以有很多实例，但是它们包含的是同一个引用对象，这样方便了应用层的使用。如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/5713484-16da386aeb274c42.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder对象关系图.png

这样应用层可以直接抛开接口对象直接使用Binder的引用对象，但是这样开发的程序兼容性不好。也正是客户端将引用对象和代理对象分离，Android才能用一套架构来同时为Java和native层提供Binder服务。隔离后，Binder底层不需要关系上层的实现细节，只需要和Binder实体对象和引用对象进行交互。

> PS:BpBinder(Binder引用对象,在客户端)和BBinder(Binder实体,在服务端)都是Android中Binder通信相关的代表，它们都是从IBiner类中派生而来(BpBinder和BBinder在Native层，不在framework层)，关系图如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-f7abd498f0cec6b6.png?imageMogr2/auto-orient/strip|imageView2/2/w/361/format/webp)

image.png

client端：BpBinder通过调用transact()来发送事物请求
 server端：BBinder通过onTransact()会接受到相应的事物

这时候再来看下这个图，然后大家思考一下，就会明白很多事情。



![img](https:////upload-images.jianshu.io/upload_images/5713484-e7b48b84550c62a2.png?imageMogr2/auto-orient/strip|imageView2/2/w/921/format/webp)

Binder原理.png

**2、如何获得一个SM的远程接口**

![img](https:////upload-images.jianshu.io/upload_images/5713484-421f20db093018e9.png?imageMogr2/auto-orient/strip|imageView2/2/w/626/format/webp)

获取.png



如果你足够细心，你会发现这里有一个问题:

> SM和Server都是进程，Server向SM注册Binder需要进程间通信，当前实现的是进程间通信，却又用到进程间通信。有点晕是不，就好像先有鸡还是先有蛋这个问题。

其实Binder是这么解决这个问题的：

> - 针对Binder的通信机制，Server端拥有的是Binder的实体(BBinder)；Client拥有的是Binder的引用(BpBinder)。
> - 如果把SM看做Server端，让它在Binder驱动一起运行起来时就有自己的实体(BBinder)(代码中设置ServiceManager的Binder其handle的值恒为0)。这个Binder实体没有名字也不需要注册，所有的Client都认为handle值为0的binder引用(BpBinder)是用来与SM通信的。那么这个问题就解决了。
> - 但是问题又来了，Client和Server中handle的值为0(值为0的引用是专门与SM通信用的)，还不行，还需要让SM的handle值为0的实体(BBinder)为0才算大功告成。怎么实现的? 当一个进程调用Binder驱动时，使用** "BINDER_SET_CONTEXXT_MGR" ** 命名(在binder_ioctl中)将自己注册成SM时，Binder驱动会自动为她创建Binder实体。这个Binder的引用对所有Client都为0。

**3、Client从SM中获得Service的远程接口**

> Server向SM注册了Binder实体及其名字后，Client就可以Service的名字在SM在查找表中获得了该Binder的引用(BpBinder)了。Client也利用了保留的handle值为0的引用向SM请求访问某个Service:当申请访问XXXService的引用。SM就会从请求数据包中获得XXXService的名字，在查找表中找到名字对应的条目，取出Binder的引用打包回复给Client。然后，Client就可以利用XXXService的引用使用XXXService的服务了。如果有更多的Client请求该Service，系统中就会有更多的Client获得这个引用。

如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-306663b15530322c.png?imageMogr2/auto-orient/strip|imageView2/2/w/569/format/webp)

获取远程接口.png

**4、建立C/S连接后**

首先要明白一个事情：

> Client要拥有自己的自己的Binder实体，以及Server的Binder的应用；Server有用自己的Binder的实体，以及Client的Binder引用。

我们也可以按照网络请求的方式来分析：

> - 从Client向Server发送数据：Client为发送方，拥有Binder实体；Server为接收方，拥有Binder引用。
> - 从Server向Client发送数据：Server为发送方，拥有Binder实体：Client为接收方，拥有Binder引用。

其实，在我们建立C/S连接后，无需考虑谁是Client，谁是Server。只要理清谁是发送方，谁是接收方，就能知道Binder的实体和应用在那边。

那我们看下建立C/S连接后的，具体流程,如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-4c7fcd2e70d997c0.png?imageMogr2/auto-orient/strip|imageView2/2/w/991/format/webp)

建立C/S连接后的流程.png

那我们说下具体的流程:

> - 第一步，发送方通过Binder实体请求发送操作
> - 第二步，Binder驱动会处理这个操作请求，把发送方的数据放入写缓存(binder_write_read.write_buffer)(对于接受方来说为读缓存区)，并把read_size(接收方读数据)置为数据大小。
> - 第三步，接收方之前一直在阻塞状态中，当写缓存又数据，则会读取数据，执行命令操作
> - 第四步，接收方执行完后，会把返回结果同样采用binder_transaction_data结构体封装，写入缓冲区(对于发送方，为读缓冲区)

### (四) 匿名Binder

在Android中Binder还可以建立点对点的私有通道，匿名Binder就是这种方式。在Binder通信中，并不是所有通信的Binder实体都需要注册给SM的，Server可以通过已建立的实体Binder连接将创建的Binder实体传给Client。而这个Binder没有向SM注册名字。这样Server和Client通信就有很高的隐私性和安全性。

如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-32f314f85b38a4f3.png?imageMogr2/auto-orient/strip|imageView2/2/w/352/format/webp)

匿名Binder.png

### 五、 Binder的层次

从代码上看，Binder设计的类可以分成4个层级，如下图所示

![img](https:////upload-images.jianshu.io/upload_images/5713484-c4e04f1f98aad092.png?imageMogr2/auto-orient/strip|imageView2/2/w/361/format/webp)

Binder的层次图.png

> - 最上层的是位于Framewok中的各种Binder服务类和它们的接口类。这一层的类非常多，比如常见的ActivityManagerService(缩写叫AMS)、WindowManagerService(缩写叫WMS)、PackageManagerService(缩写是PMS)等，它们为应用程序提供了各种各样的服务。
> - 中间则分为为两层，上面是用于服务类和接口开发的基础，比如IBinder、BBinder、BpBinder等。下层是和驱动交互的IPCThreadState和ProcessState类。
> - 这里刻意把中间的libbinder中的类划分为两个层次的原因，是在这4层中，第一层的和第二层联系很紧密，第二层中的 各种Binder类用来支撑服务类和代理类的开发。但是第三层的IPCThread和第四层之间耦合得很厉害，单独理解IPCThread或者是驱动都是一件很难的事，必须把它们结合起来理解，这一点正是Binder架构被人诟病的地方，驱动和应用层之间过于耦合，违反了Linux驱动设计的原则，因此，主流的Linux并不愿意接纳Binder。

下面我们就来详细的看来Binder

## 六、Binder协议

Biner协议格式基本是"命令+数据"，使用ioctl(fd,cmd,arg)函数实现交互。命令由参数cmd承载，数据由参数arg，随着cmd不同而不动。下表列了所有命令及其对应的数据：

| 命令                   |                             含义                             |                                                    参数(arg) |
| ---------------------- | :----------------------------------------------------------: | -----------------------------------------------------------: |
| BINDER_WRITE_READ      | 该命令向Binder写入或读取数据。参数分为两段：写部分和读部分。如果write_size不为0，就将write_buffer里的数据写入Binder；如果read_ size不为0再从Binder中读取数据存入read_buffer中。write_consumered和read_consumered表示操作完成时Binder驱动实际写入或者读出数据的个数 | struct binder_write_read{  singed long write_size;singed long write_consumed; unsigned long write_buffer; signed long read_size; signed long read_consumed; unsigned long read_buffer; } ; |
| BINDER_SET_MAX_THREADS | 该命令告知Binder驱动接收方(通常是Server端)线程池中最大的线程数。由于Client是并发向Server端发送请求的，Server端必须开辟线程池为这些并发请求提供服务 。告知驱动线程池的最大值是为了让驱动发现线程数达到该线程池的最大值是为了让驱动发现线程数达到该值时，不要再命令接收端启动先的线程。 |                                             int max_threads; |
| BINDER_SET_CONTEXT_MGR | 当前进程注册为SM。系统中只能存在一个SM，只要当前的SM没有调用close()关闭，Binder驱动就不能有别的进程变成SM |                                                           无 |
| BINDER_TREAD_EXIT      | 通知Binder驱动当前线程退出了。Binder会为所有参与的通信线程(包括Server线程池中的线程和Client发出的请求的线程) 建立相应的数据结构。这些线程在退出时必须通知驱动释放相应的数据结构 |                                                           无 |
| BINDER_VERSION         |                    获取Binder驱动的版本号                    |                                                           无 |

这其中最常用的命令是 BINDER_WRITE_READ。该命令的参数包括两个部分:

> - 1、是向Binder写入数据
> - 2、是向Binder读出数据

驱动程序先处理写部分再处理读部分。这样安排的好处是应用程序可以很灵活的地处理命令的同步或者异步。例如若要发送异步命令可以只填入写部分而将read_size设置为0，若要只从Binder获得的数据可以将写部分置空，即write_size置0。如果想要发送请求并同步等待返回数据可以将两部分都置上。

### (一)、BINDER_WRITE_READ 之写操作

Binder写操作的数据时格式同样也是(命令+数据)。这时候命令和数据都存放在binder_write_read结构中的write_buffer域指向的内存空间里，多条命令可以连续存放。数据紧接着存放在命令后面，格式根据命令不同而不同。下表列举了Binder写操作支持的命令：
 我提供两套，一套是图片，方便手机用户，一部分是文字，方便PC用户



![img](https:////upload-images.jianshu.io/upload_images/5713484-ffb7c05eaa6f6eaa.png?imageMogr2/auto-orient/strip|imageView2/2/w/881/format/webp)

写操作.png



上面图片，下面是文字

| 命令                                                  |                             含义                             |                                                  参数(arg) |                                                              |
| ----------------------------------------------------- | :----------------------------------------------------------: | ---------------------------------------------------------: | ------------------------------------------------------------ |
| BC_TRANSACTION     BC_REPLY                           | BC_TRANSACTION用于Client向Server发送请求数据；BC_REPLY用于Server向Client发送回复（应答）数据。其后面紧接着一个binder_transaction_data结构体表明要写入的数据。 |                            struct  binder_transaction_data |                                                              |
| BC_ACQUIRE_RESULT                                     |                           暂未实现                           |                                                            |                                                              |
| BC_FREE_BUFFER                                        | 释放一块映射内存。Binder接受方通过mmap()映射一块较大的内存空间，Binder驱动基于这片内存采用最佳匹配算法实现接受数据缓存的动态分配和释放，满足并发请求对接受缓存区的需求。应用程序处理完这篇数据后必须尽快使用费改命令释放缓存区，否则会因为缓存区耗尽而无法接受新数据 | 指向需要释放的缓存区的指针；该指针位于收到的Binder数据包中 |                                                              |
| BC_INCREFS  BC_ACQUIRE   BC_RELEASE  BC_DECREFS       | 这组命令增加或减少Binder的引用计数，用以实现强指针或弱指针的功能 |                                           32位Binder引用号 |                                                              |
| BC_REGISTER_LOOPER   BC_ENTER_LOOPER   BC_EXIT_LOOPER | 这组命令同BINDER_SET_MAX_THREADS 一并实现Binder驱动对接收方线程池的管理。BC_REGISTER_LOOPER通知驱动线程池中的一个线程已经创建了；BC_ENTER_LOOPER通知该驱动线程已经进入主循环，可以接受数据；BC_EXIT_LOOPER通知驱动该线程退出主循环，不在接受数据。 |                                                      ----- |                                                              |
| BC_REQUEST_DEATH_NOTIFICATION                         | 获得Binder引用的进程通过该命令要求驱动在Binder实体销毁得到通知。虽说强指针可以确保只要有引用就不会销毁实体，但这毕竟是个跨进程的引用，谁也无法保证实体由于所在的Server关闭Binder驱动或异常退出而消失，引用者能做的就是要求Server在此刻给出通知 |                   uint32 *ptr;需要得到死亡的通知Binder引用 | void **cookie：与死亡通知相关的信息，驱动会在发出死亡通知时返回给发出请求的进程。 |
| BC_DEAD_BINDER                                        |     收到实体死亡通知书的进程在删除引用后用本命令告知驱动     |                                              void * cookie |                                                              |

> 在这些命令中，最常用的h是BC_TRANSACTION/BC_REPLY命令对，Binder请求和应答数据就是通过这对命令发送给接受方。这对命令所承载的数据包由结构体struct binder_transaction_data定义。Binder交互有同步和异步之分。利用binder_transcation_data中的flag区域划分。如果flag区域的TF_ONE_WAY位为1，则为异步交互，即client发送完请求交互即结束，Server端不再返回BC_REPLY数据包；否则Server会返回BC_REPLY数据包，Client端必须等待接受完数据包后才能完成一次交互。

### (二)、BINDER_WRITE_READ:从Binder读出数据

在Binder里读出数据格式和向Binder中写入数据格式一样，采用(消息ID+数据)形式，并且多条消息可以连续存放。下面列举从Binder读出命令及相应的参数。
 为了照顾手机端的朋友，先发图片

![img](https:////upload-images.jianshu.io/upload_images/5713484-2a7b20599b2e4bc1.png?imageMogr2/auto-orient/strip|imageView2/2/w/863/format/webp)

Binder读出数据.png

Binder读操作消息ID

| 消息                                                  |                             含义                             |                                                    参数(arg) |
| ----------------------------------------------------- | :----------------------------------------------------------: | -----------------------------------------------------------: |
| BR_ERROR                                              |                 发生内部错误(如内存分配失败)                 |                                                         ---- |
| BR_OK   BR_NOOP                                       |                           操作完成                           |                                                         ---- |
| BR_SPAWN_LOOPER                                       | 该消息用于接受方线程池管理。当驱动发现接收方所有线程都处于忙碌状态且线程池中的线程总数没有超过BINDER_SET_MAX_THREADS设置的最大线程时，向接收方发送该命令要求创建更多的线程以备接受数据。 |                                                        ----- |
| BR_TRANSCATION   BR_REPLY                             | 这两条消息分别对应发送方的 BC_TRANSACTION 和BC_REPLY，表示当前接受的数据是请求还是回复 |                                      binder_transaction_data |
| BR_ACQUIRE_RESULT   BR_ATTEMPT_ACQUIRE    BR_FINISHED |                           尚未实现                           |                                                        ----- |
| BR_DEAD_REPLY                                         |     交互过程中如果发现对方进程或线程已经死亡则返回该消息     |                                                        ----- |
| BR_TRANSACTION_COMPLETE                               | 发送方通过BC_TRRANSACTION或BC_REPLY发送完一个数据包后，都能收到该消息作为成功发送的反馈。这和BR_REPLY不一样，是驱动告知发送方已经发送成功，而不是Server端返回数据。所以不管同步还是异步交互接收方都能获得本消息。 |                                                        ----- |
| BR_INCREFS    BR_ACQUIRE   BR_RFLEASE   BR_DECREFS    | 这组消息用于管理强/弱指针的引用计数。只有提供Binder实体的进程才能收到这组消息 | void *ptr : Binder实体在用户空间中的指针     void **cookie：与该实体相关的附加数据 |
| BR_DEAD_BINDER      BR_CLEAR_DEATH_NOTIFICATION_DONE  | 向获得Binder引用的进程发送Binder实体死亡通知书：收到死亡通知书的进程接下来会返回   BC_DEAD_BINDER_DONE 确认 | void *cookie  在使用BC_REQUEST_DEATH_NOTIFICATION注册死亡通知时的附加参数 |
| BR_FAILED_REPLY                                       |                如果发送非法引用号则返回该消息                |                                                        ----- |

和写数据一样，其中最重要的消息是BR_TRANSACTION或BR_REPLY，表明收到一个格式为binder_transaction_data的请求数据包(BR_TRANSACTION或返回数据包(BR_REPLY))

### (三)、struct binder_transaction_data ：收发数据包结构

该结构是Binder接收/发送数据包的标准格式，每个成员定义如下：
 下图是Binder

![img](https:////upload-images.jianshu.io/upload_images/5713484-4dc6ef249047d46b.png?imageMogr2/auto-orient/strip|imageView2/2/w/835/format/webp)

Binder数据包.png

| 成员                                                         |                             含义                             |
| ------------------------------------------------------------ | :----------------------------------------------------------: |
| union{ size_t handle; void *ptr;}  target；                  | 对于发送数据包的一方，该成员指明发送目的地。由于目的地是远端，所以在这里填入的是对Binder实体的引用，存放在target.handle中。如前述，Binder的引用在代码中也叫句柄(handle)。     当数据包到达接收方时，驱动已将该成员修改成Binder实体，即指向 Binder对象内存的指针，使用target.ptr来获取。该指针是接受方在将Binder实体传输给其他进程时提交给驱动的，驱动程序能够自动将发送方填入的引用转换成接收方的Binder对象的指针，故接收方可以直接将其当对象指针来使用(通常是将其reinpterpret_cast相应类) |
| void *cookie；                                               | 发送方忽略该成员；接收方收到数据包时，该成员存放的是创建Binder实体时由该接收方自定义的任意数值，做为与Binder指针相关的额外信息存放在驱动中。驱动基本上不关心该成员 |
| unsigned int code ;                                          | 该成员存放收发双方约定的命令码，驱动完全不关心该成员的内容。通常是Server端的定义的公共接口函数的编号 |
| unsigned int code;                                           | 与交互相关的标志位，其中最重要的是TF_ONE_WAY位。如果该位置上表明这次交互是异步的，Server端不会返回任何数据。驱动利用该位决定是否构建与返回有关的数据结构。另外一位TF_ACCEPT_FDS是处于安全考虑，如果发起请求的一方不希望在收到回复中接收文件的Binder可以将位置上。因为收到一个文件形式的Binder会自动为接收方打开一个文件，使用该位可以防止打开文件过多 |
| pid_t send_pid     uid_t sender_euid                         | 该成员存放发送方的进程ID和用户ID，由驱动负责填入，接收方可以读取该成员获取发送方的身份。 |
| size_t data_size                                             | 驱动一般情况下不关心data.buffer里存放了什么数据。但如果有Binder在其中传输则需要将其对应data.buffer的偏移位置指出来让驱动知道。有可能存在多个Binder同时在数据中传递，所以须用数组表示所有偏移位置。本成员表示该数组的大小。 |
| union{  struct{ const void *buffer;  const void * offset; } ptr; uint8_t buf[8];} data; | data.buffer存放要发送或接收到的数据；data.offsets指向Binder偏移位置数组，该数组可以位于data.buffer中，也可以在另外的内存空间中，并无限制。buf[8]是为了无论保证32位还是64位平台，成员data的大小都是8字节。 |

> PS:这里有必要强调一下offsets_size和data.offsets两个成员，这是Binder通信有别于其他IPC的地方。就像前面说说的，Binder采用面向对象的设计思想，一个Binder实体可以发送给其他进程从而建立许多跨进程的引用；另外这些引用也可以在进程之间传递，就像java将一个引用赋值给另外一个引用一样。为Binder在不同进程中创建引用必须有驱动参与，由驱动在内核创建并注册相关的数据结构后接收方才能使用该引用。而且这些引用可以是强类型的，需要驱动为其维护引用计数。然后这些跨进程传递的Binder混杂在应用程序发送的数据包里，数据格式由用户定义，如果不把他们一一标记出来告知驱动，驱动将无法从数据中将他们提取出来。于是就是使用数组data.offsets存放用户数据中每个Binder相对于data.buffer的偏移量，用offersets_size表示这个数组的大小。驱动在发送数据包时会根据data.offsets和offset_size将散落于data.buffer中的Binder找出来并一一为它们创建相关的数据结构。

## 七、Binder的整体架构

![img](https:////upload-images.jianshu.io/upload_images/5713484-b1f3856acf1e2384.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder的整体架构.png







# Android跨进程通信IPC之Binder相关结构体简介



##  一、结构体binder_work

**1、位置**

位置在
 [Linux的binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)  240行

**2、代码注释**

**binder_work**代表binder驱动中进程要处理的工作项



```rust
struct binder_work {
    struct list_head entry;  //用于实现一个双向链表，存储的所有binder_work队列
    enum {
        BINDER_WORK_TRANSACTION = 1,
        BINDER_WORK_TRANSACTION_COMPLETE,
        BINDER_WORK_NODE,
        BINDER_WORK_DEAD_BINDER,
        BINDER_WORK_DEAD_BINDER_AND_CLEAR,
        BINDER_WORK_CLEAR_DEATH_NOTIFICATION,
    } type;  //描述工作项的类型
};
```

## 二、结构体binder_thread

**1、代码位置**

位置在
 [Linux的binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)  368行

2、**代码注释**

**binder_thread** 代表Binder线程池中的每个线程信息



```cpp
struct binder_thread {
    //宿主进程，即线程属于那么Binder进程
    struct binder_proc *proc;  
    //红黑树的一个节点，binder_proc使用红黑树来组织Binder线程池中的线程
    struct rb_node rb_node;  
    // 当前线程的PID
    int pid;
    // 当前线程的状态信息
    int looper;
    //事务堆栈，将事务封装成binder_transaction，添加到事务堆栈中，
    //定义了要接收和发送的进程和线程消息
    struct binder_transaction *transaction_stack;
    //队列，当有来自Client请求时，将会添加到to_do队列中
    struct list_head todo;
    // 记录阅读buf事务时出现异常错误情况信息
    uint32_t return_error; /* Write failed, return error code in read buf */
    // 记录阅读事务时出现异常错误情况信息
    uint32_t return_error2; /* Write failed, return error code in read */
        /* buffer. Used when sending a reply to a dead process that */
        /* we are also waiting on */
     // 等待队列，当Binder处理事务A依赖于其他Binder线程处理事务B的情况
     // 则会在sleep在wait所描述的等待队列中，知道B事物处理完毕再唤醒
    wait_queue_head_t wait;
     // Binder线程相关统计数据
    struct binder_stats stats;
};
```

这里面说下上面looper对应的值



```rust
enum {
    // Binder驱动 请求创建该线程，通过BC_REGISTER_LOOPER协议通知
    //Binder驱动，注册成功设置此状态
    BINDER_LOOPER_STATE_REGISTERED  = 0x01,
    //该线程是应用程序主动注册的  通过 BC_ENTER_LOOPER 协议
    BINDER_LOOPER_STATE_ENTERED     = 0x02,
    // Binder线程退出
    BINDER_LOOPER_STATE_EXITED      = 0x04,
    // Binder线程处于无效
    BINDER_LOOPER_STATE_INVALID     = 0x08,
    // Binder线程处于空闲状态
    BINDER_LOOPER_STATE_WAITING     = 0x10,
     // Binder线程处于需要返回用户控件
     // 使用场景是：1、线程注册为Binder线程后，还没有准备好去处理进程间通信
     //，需要返回用户空间做其他初始化准备；2、调用flush来刷新Binder线程池                                                                                                                 
    BINDER_LOOPER_STATE_NEED_RETURN = 0x20
};
```

## 三、结构体binder_stats

**binder_stats** 代表d的是Binder线程相关统计数据

**1、代码位置**

位置在
 [Linux的binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)  173行

**2、代码注释**

```cpp
struct binder_stats {
        // 统计各个binder响应码的个数
    int br[_IOC_NR(BR_FAILED_REPLY) + 1];
        // 统计各个binder请求码的个数
    int bc[_IOC_NR(BC_REPLY_SG) + 1];
        // 统计各种obj创建的个数
    int obj_created[BINDER_STAT_COUNT];
        // 统计各种obj删除个数
    int obj_deleted[BINDER_STAT_COUNT];
};
```

## 四、结构体binder_proc

> **binder_proc** 代表的是一个正在使用Binder进程通信的进程，binder_proc为管理其信息的记录体，当一个进程open /dev/binder 时，Binder驱动程序会为其创建一个binder_proc结构体，用以记录进程的所有相关信息，并把该结构体保存到一个全局的hash表中

**1、代码位置**

位置在
 [Linux的binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)  322行

**2、代码注释**

```cpp
struct binder_proc {
        /** 进程相关参数 */

        //上述全局hash表中一个节点，用以标记该进程
    struct hlist_node proc_node;
        // 进程组ID
    int pid;
         // 任务控制模块 
    struct task_struct *tsk; 
        // 文件结构体数组
    struct files_struct *files;

       /**  Binder线程池每一个Binder进程都有一个线程池，由Binder驱动来维护，Binder线程池中所有线程由一个红黑树来组织，RB树以线程ID为关键字  */
        //上述红黑树的根节点
    struct rb_root threads;
        
       /** 一系列Binder实体对象(binder_node)和Binder引用对象(binder_ref) */
       /** 在用户控件：运行在Server端称为Binder本地对象，运行在Client端称为Binder代理对象*/
       /**  在内核空间：Binder实体对象用来描述Binder本地对象，Binder引用对象来描述Binder代理对象 */
         // Binder实体对象列表(RB树)，关键字 ptr
    struct rb_root nodes;
         // Binder引用对象，关键字  desc
    struct rb_root refs_by_desc;
         // Binder引用对象，关键字  node
    struct rb_root refs_by_node;
         // 这里有两个引用对象，是为了方便快速查找 


     /**  进程可以调用ioctl注册线程到Binder驱动程序中，当线程池中没有足够空闲线程来处理事务时，Binder驱动可以主动要求进程注册更多的线程到Binder线程池中 */
        // Binder驱动程序最多可以请求进程注册线程的最大数量
    int max_threads;
        // Binder驱动每主动请求进程添加注册一个线程的时候，requested_threads+1
    int requested_threads;
        // 进程响应Binder要求后，requested_thread_started+1，request_threads-1，表示Binder已经主动请求注册的线程数目
    int requested_threads_started;

        // 进程当前空闲线程的数目
    int ready_threads;
        // 线程优先级，初始化为进程优先级
    long default_priority;
        //进程的整个虚拟地址空间
    struct mm_struct *vma_vm_mm;

        /** mmap 内核缓冲区*/
        // mmap——分配的内核缓冲区  用户控件地址(相较于buffer)
    struct vm_area_struct *vma; 
        // mmap——分配内核缓冲区，内核空间地址(相交于vma)  两者都是虚拟地址
    void *buffer;
        // mmap——buffer与vma之间的差值
    ptrdiff_t user_buffer_offset;

        /** buffer指向的内核缓冲区，被划分为很多小块进行性管理；这些小块保存在列表中，buffer就是列表的头部 */
         // 内核缓冲列表
    struct list_head buffers;
         // 空闲的内存缓冲区(红黑树)
    struct rb_root free_buffers;
         // 正在使用的内存缓冲区(红黑树)
    struct rb_root allocated_buffers;
        // 当前可用来保存异步事物数据的内核缓冲区大小
    size_t free_async_space;
        //  对应用于vma 、buffer虚拟机地址，这里是他们对应的物理页面
    struct page **pages;
        //  内核缓冲区大小
    size_t buffer_size;
        // 空闲内核缓冲区大小
    uint32_t buffer_free;

         /** 进程每接收到一个通信请求，Binder将其封装成一个工作项，保存在待处理队列to_do中  */
        //待处理队列
    struct list_head todo;
        // 等待队列，存放一些睡眠的空闲Binder线程
    wait_queue_head_t wait;
        // hash表，保存进程中可以延迟执行的工作项
    struct hlist_node deferred_work_node;
        // 延迟工作项的具体类型
    int deferred_work;
    
        //统计进程相关数据，具体参考binder_stats结构体
    struct binder_stats stats;
        // 表示 Binder驱动程序正在向进程发出死亡通知
    struct list_head delivered_death;
        // 用于debug
    struct dentry *debugfs_entry;
        // 连接 存储binder_node和binder_context_mgr_uid以及name
    struct binder_context *context;
};
```

## 五、结构体binder_node

**binder_node** 代表的是Binder实体对象，每一个service组件或者ServiceManager在Binder驱动程序中的描述，Binder驱动通过强引用和弱引用来维护其生命周期，通过node找到空间的Service对象

**1、代码位置**

位置在
 [Linux的binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)  252行

**2、代码注释**

```cpp
struct binder_node {
        // debug调试用的
    int debug_id;
    struct binder_work work;  //binder驱动中进程要处理的工作项 

        /** 每一个binder进程都由一个binder_proc来描述，binder进程内部所有Binder实体对象，
        由一个红黑树来进行组织(struct rb_root nodes)  ； rb_node 则对应nodes的一个节点 */
    union {
        //用于本节点连接红黑树
        struct rb_node rb_node;
        // 如果Binder 实体对象对应的进程死亡，销毁节点时需要将rb_node从红黑树中删除，
        //如果本节点还没有引用切断，则用dead_node将其隔离到另一个链表中，
        //直到通知所有进程切断与该节点的引用后，该节点才能销毁
        struct hlist_node dead_node;
    };

    // 指向该Binder实体对象 对应的进程，进程由binder_proc描述
    struct binder_proc *proc;
    // 该 Binder实体对象可能同时被多个Client组件引用，所有指向本实体对象的引用都
    //保存在这个hash队列中refs表示队列头部；这些引用可能隶属于不同进程，遍历该
    //hash表能够得到这些Client组件引用了这些对象
    struct hlist_head refs;

    /** 引用计数 
     * 1、当一个Binder实体对象请求一个Service组件来执行Binder操作时。会增加该Service
     * 组件的强/弱引用计数同时，Binder实体对象将会has_strong_ref与has_weak_ref置为1 
     *2、当一个Service组件完成一个Binder实体对象请求的操作后，Binder对象会请求减少该
     * Service组件的强/弱引用计数
     * 3、Binder实体对象在请求一个Service组件增加或减少强/弱引用计数的过程中，
     * 会将pending_strong_ref和pending_weak_ref置为1，当Service组件完成增加
     * 或减少计数时，Binder实体对象会将这两个变量置为0
     */

    //远程强引用 计数
    int internal_strong_refs;  //实际上代表了一个binder_node与多少个binder_ref相关联
    //本地弱引用技数
    int local_weak_refs;
     //本地强引用计数
    int local_strong_refs;

    unsigned has_strong_ref:1;
    unsigned pending_strong_ref:1;
    unsigned has_weak_ref:1;
    unsigned pending_weak_ref:1;

     /** 用来描述用户控件中的一个Service组件 */
    // 描述用户控件的Service组件，对应Binder实体对应的Service在用户控件的(BBinder)的引用
    binder_uintptr_t ptr;
    // 描述用户空间的Service组件，Binder实体对应的Service在用户控件的本地Binder(BBinder)地址
    binder_uintptr_t cookie;

     // 异步事务处理，单独讲解
    unsigned has_async_transaction:1;
    struct list_head async_todo;
    // 表示该Binder实体对象能否接收含有该文件描述符的进程间通信数据。当一个进程向
    //另一个进程发送数据中包含文件描述符时，Binder会在目标进程中打开一个相同的文件
    //故设为accept_fds为0 可以防止源进程在目标进程中打开文件
    unsigned accept_fds:1;
     // 处理Binder请求的线程最低优先级
    unsigned min_priority:8;

};
```

这里说下binder_proc和binder_node关系：

> 可以将binder_proc理解为一个进程，而将binder_noder理解为一个Service。binder_proc里面有一个红黑树，用来保存所有在它所描述的进程里面创建Service。而每一个Service在Binder驱动里面都有一个binder_node来描述。

**3、异步事物处理**

> 异步事务处理，目的在于为同步交互让路，避免长时间阻塞发送送端
>  异步事务定义：(相对于同步事务)单向进程间通信要求，即不需要等待应答的进程间通信请求
>  Binder驱动程序认为异步事务的优先级低于同步事务，则在同一时刻，一个Binder实体对象至多只有一个异步事物会得到处理。而同步事务则无此限制。
>  Binder将事务保存在一个线程binder_thread的todo队列中，表示由该线程来处理该事务。每一个事务都关联Binder实体对象(union target)，表示该事务的目标处理对象，表示要求该Binder实体对象对应的Service组件在制定线程中处理该事务，而如果Binder发现一个事务时异步事务，则会将其保存在目标Binder对象的async_todo的异步事务中等待处理

## 六、结构体binder_ref

**binder_ref** 代表的是Binder的引用对象，每一个Clinet组件在Binder驱动中都有一个Binder引用对象，用来描述它在内核中的状态。

**1、代码位置**

位置在
 [Linux的binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)  281行

**2、代码注释**

```cpp
struct binder_ref {
    /* Lookups needed: */
    /*   node + proc => ref (transaction) */
    /*   desc + proc => ref (transaction, inc/dec ref) */
    /*   node => refs + procs (proc exit) */
        //debug 调试用的
        int debug_id;
     
        /** binder_proc中使用红黑树(对应两个rb_root变量) 来存储器内部所有引用对象，
         *下面的rb_node则是红黑树中的节点
         */
        //Binder引用的宿主进程
        struct binder_proc *proc;
        //对应 refs_by_desc，以句柄desc索引  关联到binder_proc->refs_by_desc红黑树 
        struct rb_node rb_node_desc;
         //对应refs_by_node，以Binder实体对象地址作为关键字关联到binder_proc->refs_by_node红黑树
        struct rb_node rb_node_node;

        /** Client通过Binder访问Service时，仅需指定一个句柄，Binder通过该desc找到对应的binder_ref，
         *  再根据该binder_ref中的node变量得到binder_node(实体对象)，进而找到对应的Service组件
         */
        // 对应Binder实体对象中(hlist_head) refs引用对象队列中的一个节点
        struct hlist_node node_entry;
        // 引用对象所指向的Binder实体对象
        struct binder_node *node;
        // Binder引用的句柄值，Binder驱动为binder驱动引用分配一个唯一的int型整数（进程范围内唯一）
        // ，通过该值可以在binder_proc->refs_by_desc中找到Binder引用，进而可以找到Binder引用对应的Binder实体
        uint32_t desc;

        // 强引用 计数
        int strong;
        // 弱引用 计数
        int weak;
      
        //  表示Service组件接受到死亡通知
        struct binder_ref_death *death;
};
```

## 七、结构体binder_ref_death

**binder_ref_death** 一个死亡通知的结构体
 我们知道Client组件无法控制它所引用的Service组件的生命周期，由于Service组件所在的进程可能意外崩溃。Client进程需要能够在它所引用的Service组件死亡时获的通知，进而进行响应。则Client进程就需要向Binder驱动注册一个用来接收死亡通知的对象地址(这里的cookie)

**1、代码位置**

位置在
 [Linux的binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)  276行

**2、代码注释**

```cpp
struct binder_ref_death {
         //标志该通知具体的死亡类型
        struct binder_work work;   
         // 保存负责接收死亡通知的对象地址
        binder_uintptr_t cookie;
};
```

这里随带说一下Binder驱动向Client进程发送死亡通知的情况：

> - 1、Binder驱动检测到Service组件死亡时，会找到对应Serivce实体对象(binder_node)，再通过refs变量找到引用它的所有Client进程(binder_ref)，再通过death变量找到Client进程向Binder注册的死亡通知接收地址；Binder将死亡通知binder_ref_death封装成工作项，添加到Client进程to_do队列中等待处理。这种情况binder_work类型为BINDER_WORK_DEAD_BINDER
> - 2、Client进程向Binder驱动注册一个死亡接收通知时，如果它所引用的Service组件已经死亡，Binder会立即发送通知给Client进程。这种情况binder_work类型为BINDER_WORK_DEAD_BINDER
> - 3、当Client进程向Binder驱动注销一个死亡通知时，也会发送通知，来响应注销结果
> - ①当Client注销时，Service组件还未死亡：Binder会找到之前Client注册的binder_ref_death，当binder_work修改为BINDER_CLEAR_NOTIFICATION，并将通知按上述步骤添加到Client的to_do队列中
> - @当Client注销时，Service已经死亡，Binder同理将binder_work修改为WORK_DEAD_BINDER_AND_CLEAR，然后添加到todo中

## 八、结构体binder_state

**binder_state** 代表着binder设备文件的状态

**1、代码位置**

位置在
 [/frameworks/native/cmds/servicemanager/binder.c](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/native/cmds/servicemanager/binder.c)  89行

2、代码注释

```cpp
struct binder_state
{
    //打开 /dev/binder之后得到的文件描述符
    int fd;
    //mmap将设备文件映射到本地进程的地址空间，映射后的到地址空间中，映射后得到的地址空间地址，及大小。
    void *mapped;
    // 分配内存的大小，默认是128K
    size_t mapsize;
};
```

## 九、结构体binder_buffer

**binder_buffer**  内核缓冲区，用以在进程间传递数据。binder驱动程序管理这个内存映射地址空间方法，即管理buffer~（buffer+buffer_size）这段地址空间的，这个地址空间被划分为一段一段来管理，每一段是结构体struct binder_buffer来描述。每一个binder_buffer通过其成员entry从低到高地址连入到struct binder_proc中的buffers表示链表中去

**1、代码位置**

位置在
 [Linux的binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 298行

**2、代码注释**

```cpp
struct binder_buffer {
        //entry对应内核缓冲区列表的buffers(内核缓冲区列表)
        struct list_head entry; /* free and allocated entries by address */
        //结合free,如果free=1，则rb_node对应free_buffers中一个节点(内核缓冲区)
        //如果free!=1，则对应allocated_buffers中的一个节点
        struct rb_node rb_node; /* free entry by size or allocated entry */
                /* by address */
        unsigned free:1;

         /**  Binder将事务数据保存到一个内核缓冲区(binder_transaction.buffer)，然后交由Binder
          * 实体对象(target_node) 处理，而target_node会将缓冲区的内容交给对应的Service组件
          * (proc) 来处理，Service组件处理完事务后，若allow_user_free=1，则请求Binder释放该
          * 内核缓冲区
          */
        unsigned allow_user_free:1;
         // 描述一个内核缓冲区正在交给那个事务transaction，用以中转请求和返回结果
        struct binder_transaction *transaction;
         // 描述该缓冲区正在被那个Binder实体对象使用
        struct binder_node *target_node;

        //表示事务时异步的；异步事务的内核缓冲区大小是受限的，这样可以保证事务可以优先放到缓冲区
        unsigned async_transaction:1;
         //调试专用
        unsigned debug_id:29;

        /** 
         *  存储通信数据，通信数据中有两种类型数据：普通数据与Binder对象
         *  在数据缓冲区最后，有一个偏移数组，记录数据缓冲区中每一个Binder
         *  对象在缓冲区的偏移地址
         */
        //  数据缓冲区大小
        size_t data_size;
        // 偏移数组的大小(其实也是偏移位置)
        size_t offsets_size;
        // 用以保存通信数据，数据缓冲区，大小可变
        uint8_t data[0];
        // 额外缓冲区大小
        size_t extra_buffers_size;
};
```

## 十、结构体binder_transaction

** binder_transaction **  描述Binder进程中通信过程，这个过程称为一个transaction(事务)，用以中转请求和返回结果，并保存接受和要发送的进程信息

**1、代码位置**

位置在
 [Linux的binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c) 383行

2、代码注释

```cpp
struct binder_transaction {
        //调试调用
        int debug_id;
        // 用来描述的处理的工作事项，这里会将type设置为BINDER_WORK_TRANSACTION，具体结构看binder_work
        struct binder_work work;

        /**   源线程 */
        // 源线程，即发起事务的线程
        struct binder_thread *from;
        // 源线程的优先级
        long    priority;
        //源 线程的用户 ID
        kuid_t  sender_euid;

        /**  目标线程*/
        //  目标进程:处理该事务的进程
        struct binder_proc *to_proc;
        // 目标线程：处理该事务的线程
        struct binder_thread *to_thread;
   
         // 表示另一个事务要依赖事务(不一定要在同一个线程中)
        struct binder_transaction *from_parent;
         // 目标线程下一个需要处理的事务
        struct binder_transaction *to_parent;

        // 标志事务是同步/异步；设为1表示同步事务，需要等待对方回复；设为0异步
        unsigned need_reply:1;
        /* unsigned is_dead:1; */   /* not used at the moment */
     
        /* 参考binder_buffer中解释，指向Binder为该事务分配内核缓冲区
         *  code与flag参见binder_transaction_data
         */
        struct binder_buffer *buffer;
        unsigned int    code;
        unsigned int    flags;

        /**  目标线程设置事务钱，Binder需要修改priority；修改前需要将线程原来的priority保存到
         *    saved_priority中，用以处理完事务回复到原来优先级
         *   优先级设置：目标现场处理事务时，优先级应不低于目标Serivce要求的线程优先级，也
         *   不低于源线程的优先级，故设为两者的较大值。
         */
        long    saved_priority;

};
```

## 十一、结构体binder_transaction_data

**binder_transaction_data**  代表进程间通信所传输的数据：Binder对象的传递时通过binder_transaction_data来实现的，即Binder对象实际是封装在binder_transaction_data结构体中

**1、代码位置**

位置在
 [Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h) 217行

**2、代码注释**

```cpp
struct binder_transaction_data {
         //target很重要，我下面重点介绍
         /* The first two are only used for bcTRANSACTION and brTRANSACTION,
         * identifying the target and contents of the transaction.
         */
         union {
               /* target descriptor of command transaction */
               __u32    handle;
               /* target descriptor of return transaction */
               binder_uintptr_t ptr;
         } target;
      
         //Binder实体带有的附加数据
         binder_uintptr_t   cookie; /* target object cookie */
         // code是一个命令，描述了请求Binder对象执行的操作，表示要对目标对象请求的命令代码
         __u32      code;       /* transaction command */

         /* General information about the transaction. */
         // 事务标志，详细看transaction_flag结构体
         __u32          flags;
         // 发起请求的进程PID
         pid_t      sender_pid;
         // 发起请求的进程UID
         uid_t      sender_euid;
         // data.buffer缓冲区的大小，data见最下面的定义；命令的真正要传输的数据就保存data.buffer缓冲区
         binder_size_t  data_size;  /* number of bytes of data */
          // data.offsets缓冲区的大小
         binder_size_t  offsets_size;   /* number of bytes of offsets */

         /* If this transaction is inline, the data immediately
         * follows here; otherwise, it ends with a pointer to
         * the data buffer.
         */
         union {
               struct {
                        /* transaction data */
                        binder_uintptr_t    buffer;
                        /* offsets from buffer to flat_binder_object structs */
                        binder_uintptr_t    offsets;
               } ptr;
               __u8 buf[8];
         } data;
};
```

这里重点说两个共用体target和data
 target

> 一个共用体target，当这个BINDER_WRITE_READ命令的目标对象是本地Binder的实体时，就用ptr来表示这个对象在本进程的地址，否则就使用handle来表示这个Binder的实体引用。

只有目标对象是Binder实体时，cookie成员变量才有意义，表示一些附加数据。
 详解解释一下：传输的数据是一个复用数据联合体，对于BINDER类型，数据就是一个Binder本地对象。如果是HANDLE类型，这个数据就是远程Binder对象。很多人会说怎么区分本地Binder对象和远程Binder对象，主要是角度不同而已。本地对象还可以带有额外数据，保存在cookie中。

data

> - 命令的真正要传输的数据就保存在data.buffer缓冲区中，前面的一成员变量都是一些用来描述数据的特征。data.buffer所表示的缓冲区数据分为两类，一类是普通数据，Binder驱动程序不关心，一类是Binder实体或者Binder引用，这需要Binder驱动程序介入处理。为什么?因为如果一个进程A传递了一个Binder实体或Binder引用给进程B，那么，Binder驱动程序就需要介入维护这个Binder实体或者引用引用计数。防止B进程还在使用这个Binder实体时，A却销毁这个实体，这样的话，B进程就会crash了。所以在传输数据时，如果数据中含有Binder实体和Binder引用，就需要告诉Binder驱动程序他们的具体位置，以便Binder驱动程序能够去维护它们。data.offsets的作用就在这里，它指定在data.buffer缓冲区中，所以Binder实体或者引用的偏移位置。
> - 进程间传输的数据被称为Binder对象(Binder Object)，它是一个flat_binder_object。Binder对象的传递时通过binder_transaction_data来实现的，即Binder对象实际是封装在binder_transaction_data结构体中。

## 十二、结构体transaction_flags

**transaction_flags**  描述传输方式，比如同步或者异步等

**1、代码位置**

位置在
 [Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h) 210行

**2、代码注释**

```rust
enum transaction_flags {
         //当前事务异步，不需要等待
        TF_ONE_WAY  =  0x01,    /* this is a one-way call: async, no return */
        // 包含内容是根对象
        TF_ROOT_OBJECT  =  0x04,    /* contents are the component's root object */
        // 表示data所描述的数据缓冲区内 同时一个4bit的状态码
        TF_STATUS_CODE  =  0x08,    /* contents are a 32-bit status code */
        // 允许数据中包含文件描述
        TF_ACCEPT_FDS   =  0x10,    /* allow replies with file descriptors */
};
```

## 十三、结构体flat_binder_object

**flat_binder_object**  描述进程中通信过程中传递的Binder实体/引用对象或文件描述符

**1、代码位置**

位置在
 [Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h) 68行

**2、代码注释**

```rust
/*
 * This is the flattened representation of a Binder object for transfer
 * between processes.  The 'offsets' supplied as part of a binder transaction
 * contains offsets into the data where these structures occur.  The Binder
 * driver takes care of re-writing the structure type and data as it moves
 * between processes.
 */
struct flat_binder_object {
    struct binder_object_header hdr;
    __u32               flags;

    /* 8 bytes of data. */
    union {
        binder_uintptr_t    binder; /* local object */
        __u32           handle; /* remote object */
    };

    /* extra data associated with local object */
    binder_uintptr_t    cookie;
};
```

这个比较重要我们就一个一个来说，首先看下**struct binder_object_header   hdr;**
 里面涉及到一个结构体binder_object_header，这个结构体在[Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h) 57行，代码如下：



```rust
/**
 * struct binder_object_header - header shared by all binder metadata objects.
 * @type:   type of the object
 */
struct binder_object_header {
    __u32        type;
};
```

flat_binder_object通过type来区分描述类型，可选的描述类型如下：
 代码在
 [Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h) 30行



```rust
enum {
        //强类型Binder实体对象
    BINDER_TYPE_BINDER  = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),
        //弱类型Binder实体对象
    BINDER_TYPE_WEAK_BINDER = B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),
         // 强类型引用对象
    BINDER_TYPE_HANDLE  = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),
         // 弱类型引用对象
    BINDER_TYPE_WEAK_HANDLE = B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),
        // 文件描述符 
    BINDER_TYPE_FD      = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),
    BINDER_TYPE_FDA     = B_PACK_CHARS('f', 'd', 'a', B_TYPE_LARGE),
    BINDER_TYPE_PTR     = B_PACK_CHARS('p', 't', '*', B_TYPE_LARGE),
};
```

当描述实体对象时，cookie表示Binder实体对应的Service在用户空间的本地Binder(BBinder)地址，binder表示Binder实体对应的当描述引用对象时，handle表示该引用的句柄值。

PS:

> 关于BINDER_TYPE_FDA和BINDER_TYPE_PTR我也不是很清楚，有懂的兄弟，在下面留言，谢谢

## 十四、结构体binder_write_read

**binder_write_read**  描述进程间通信过程中所传输的数据

**1、代码位置**

位置在
 [Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h) 165行

**2、代码注释**

```cpp
/*
 * On 64-bit platforms where user code may run in 32-bits the driver must
 * translate the buffer (and local binder) addresses appropriately.
 */

struct binder_write_read {

        /** 输入数据 从用户控件传输到Binder驱动程序的数据
         *  数据协议代码为命令协议码，由binder_driver_command_protocol定义
         */
        // 写入的大小
        binder_size_t       write_size; /* bytes to write */
        // 记录了从缓冲区取了多少字节的数据
        binder_size_t       write_consumed; /* bytes consumed by driver */
        // 指向一个用户控件缓冲区的地址，里面的内容即为输入数据，大小由write_size指定
        binder_uintptr_t    write_buffer;
        
         /** 输出数据，从Binder驱动程序，返回给用户空间的数据
          *  数据协议代码为返回协议代码，由binder_driver_return_protocol定义
          */
        //读出的大小
        binder_size_t       read_size;  /* bytes to read */
        // read_buffer中读取的数据量
        binder_size_t       read_consumed;  /* bytes consumed by driver */
        // 指向一个用户缓冲区一个地址，里面保存输出的数据
        binder_uintptr_t    read_buffer;
};
```

里面涉及到两个协议，我们就在这里详细讲解下

**3、binder_driver_command_protocol协议**

代码在[Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h) 366行

```rust
enum binder_driver_command_protocol {
      
        /**  下面这两个命令数据类型为binder_transaction_data，是最常用到的 */
        /**一个Client进程请求目标进程执行某个事务时，会使用BC_TRANSACTION请求Binder驱
         *动程序将通信数据传递到Server目标进程
         * 使用者：Client进程   用处：传递数据
         */ 
        BC_TRANSACTION = _IOW('c', 0, struct binder_transaction_data),
         /** 当Server目标进程处理完事务后，会使用BC_REPLY请求Binder将结果返回给Client源进程
          * 使用者：Server进程  用处：返回数据
          */
        BC_REPLY = _IOW('c', 1, struct binder_transaction_data),


        /*
         * binder_transaction_data: the sent command.
         */
        //当前版本not support  在Linux中的binder.c就是这么写的
        BC_ACQUIRE_RESULT = _IOW('c', 2, __s32),

        /*
         * not currently supported
         * int:  0 if the last BR_ATTEMPT_ACQUIRE was not successful.
         * Else you have acquired a primary reference on the object.
         */

        // 数据类型为int类型，指向Binder内部一块内核缓冲区
        // 目标进程处理完源进程事务后，会使用BC_FREE_BUFFER来释放缓冲区
        BC_FREE_BUFFER = _IOW('c', 3, binder_uintptr_t),
    /*
     * void *: ptr to transaction data received on a read
     */

        //通信类型为int类型，表示binder_ref的句柄值handle
        // 增加弱引用数
        BC_INCREFS = _IOW('c', 4, __u32),
        // 减少弱引用数
        BC_DECREFS = _IOW('c', 7, __u32),
        // 增加强引用数
        BC_ACQUIRE = _IOW('c', 5, __u32),
        // 减少强引用数
        BC_RELEASE = _IOW('c', 6, __u32),
    
        /*
         * int: descriptor
         */
        /** Service进程完成增加强/弱引用的计数后，会使用这两个命令通知Binder */
        // 增加强引用计数后
        BC_INCREFS_DONE = _IOW('c', 8, struct binder_ptr_cookie),
        //增加弱引用计数后
        BC_ACQUIRE_DONE = _IOW('c', 9, struct binder_ptr_cookie),

        /*
         * void *: ptr to binder
         * void *: cookie for binder
         */
        //当前版本不支持 
        BC_ATTEMPT_ACQUIRE = _IOW('c', 10, struct binder_pri_desc),

        /*
         * not currently supported
         * int: priority
         * int: descriptor
         */
        // Binder驱动程序 请求进程注册一个线程到它的线程池中，新建立线程会使用
        //BC_REGISTER_LOOPER来通知Binder准备就绪
        BC_REGISTER_LOOPER = _IO('c', 11),

        /*
         * No parameters.
         * Register a spawned looper thread with the device.
         */
        //一个线程自己注册到Binder驱动后，会使用BC_ENTER_LOOPER通知Binder准备就绪
        BC_ENTER_LOOPER = _IO('c', 12),
        // 线程发送退出请求
        BC_EXIT_LOOPER = _IO('c', 13),

        /*
         * No parameters.
         * These two commands are sent as an application-level thread
         * enters and exits the binder loop, respectively.  They are
         * used so the binder can have an accurate count of the number
         * of looping threads it has available.
         */
        // 进程向Binder注册一个死亡通知
        BC_REQUEST_DEATH_NOTIFICATION = _IOW('c', 14,
                        struct binder_handle_cookie),
        /*
         * int: handle
         * void *: cookie
         */
         // 进程取消之前注册的死亡通知
        BC_CLEAR_DEATH_NOTIFICATION = _IOW('c', 15,
                        struct binder_handle_cookie),
        /*
         * int: handle
         * void *: cookie
         */
        // 数据指向死亡通知binder_ref_death的地址，进程获得Service组件的死亡通知，
        // 会使用该命令通知Binder其已经处理完死亡通知
        BC_DEAD_BINDER_DONE = _IOW('c', 16, binder_uintptr_t),
        /*
         * void *: cookie
         */

        BC_TRANSACTION_SG = _IOW('c', 17, struct binder_transaction_data_sg),
        BC_REPLY_SG = _IOW('c', 18, struct binder_transaction_data_sg),
        /*
         * binder_transaction_data_sg: the sent command.
         */
};
```

在上述枚举命令成员中，最重要的是BC_TRANSACTION和BC_REPLY命令，被作为发送操作的命令，其数据参数都是binder_transaction_data结构体。其中前者用于翻译和解析将要被处理的事务数据，而后者则是事务处理完成之后对返回"结果数据"的操作命令。

**4、binder_driver_return_protocol协议**

Binder驱动的响应（返回，BR_）协议，定义了Binder命令的数据返回格式。

代码在[Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h) 278行

```rust
enum binder_driver_return_protocol {
         // Binder驱动程序处理进程发送的请求时，发生异常，在返回BR_ERROR通知该进程
         // 数据类型为int，表示错误代码
        BR_ERROR = _IOR('r', 0, __s32),
    /*
     * int: error code
     */

        // 表示通知进程成功处理了该事务
        BR_OK = _IO('r', 1),
    /* No parameters! */
        
        // 与上面的 BC_ 相对应
        //  Client进程向Server进程发送通信请求(BC_) ，Binder使用BR_TRANSACTION通知Server
        // 使用者 :   Binder驱动程序     用途：通知Server
        BR_TRANSACTION = _IOR('r', 2, struct binder_transaction_data),
        // Server处理完 请求 使用 BC_ 通知Binder，Binder使用BR_REPLY通知Client
        // 使用者：Binder驱动程序，用途：通知Client
        BR_REPLY = _IOR('r', 3, struct binder_transaction_data),
    /*
     * binder_transaction_data: the received command.
     */

         // 当前不支持 
        BR_ACQUIRE_RESULT = _IOR('r', 4, __s32),
    /*
     * not currently supported
     * int: 0 if the last bcATTEMPT_ACQUIRE was not successful.
     * Else the remote object has acquired a primary reference.
     */

        // Binder处理请求时，发现目标进程或目标线程已经死亡，通知源进程
        BR_DEAD_REPLY = _IO('r', 5),
    /*
     * The target of the last transaction (either a bcTRANSACTION or
     * a bcATTEMPT_ACQUIRE) is no longer with us.  No parameters.
     */

        // Binder 接收到BC_TRANSACATION或BC_REPLY时，会返回 BR_TRANSACTION_COMPLETE通知源进程命令已经接收
        BR_TRANSACTION_COMPLETE = _IO('r', 6),
    /*
     * No parameters... always refers to the last transaction requested
     * (including replies).  Note that this will be sent even for
     * asynchronous transactions.
     */

        // 增加弱引用计数
        BR_INCREFS = _IOR('r', 7, struct binder_ptr_cookie),
        // 增加强引用计数
        BR_ACQUIRE = _IOR('r', 8, struct binder_ptr_cookie),
        // 减少强引用计数
        BR_RELEASE = _IOR('r', 9, struct binder_ptr_cookie),
         // 减少弱引用计数
        BR_DECREFS = _IOR('r', 10, struct binder_ptr_cookie),
    /*
     * void *:  ptr to binder
     * void *: cookie for binder
     */

         //当前不支持
        BR_ATTEMPT_ACQUIRE = _IOR('r', 11, struct binder_pri_ptr_cookie),
    /*
     * not currently supported
     * int: priority
     * void *: ptr to binder
     * void *: cookie for binder
     */

        // Binder通过源进程执行了一个空操作，用以可以替换为BR_SPAWN_LOOPER
        BR_NOOP = _IO('r', 12),
    /*
     * No parameters.  Do nothing and examine the next command.  It exists
     * primarily so that we can replace it with a BR_SPAWN_LOOPER command.
     */

         // Binder发现没有足够的线程处理请求时，会返回BR_SPAWN_LOOPER请求增加新的新城到Binder线程池中
        BR_SPAWN_LOOPER = _IO('r', 13),
    /*
     * No parameters.  The driver has determined that a process has no
     * threads waiting to service incoming transactions.  When a process
     * receives this command, it must spawn a new service thread and
     * register it via bcENTER_LOOPER.
     */

         // 当前暂不支持
        BR_FINISHED = _IO('r', 14),
    /*
     * not currently supported
     * stop threadpool thread
     */


        /** Binder检测到Service组件死亡时，使用BR_DEAD_BINDER通知Client进程，Client请求
         *  注销之前的死亡通知，Binder完成后，返回BR_CLEAR_DEATH_NOTIFACTION_DONE
         */
        // 告诉发送方对象已经死亡
        BR_DEAD_BINDER = _IOR('r', 15, binder_uintptr_t),

    /*
     * void *: cookie
     */
        //清理死亡通知
        BR_CLEAR_DEATH_NOTIFICATION_DONE = _IOR('r', 16, binder_uintptr_t),
    /*
     * void *: cookie
     */

         // 发生异常，通知源进程
        BR_FAILED_REPLY = _IO('r', 17),
    /*
     * The the last transaction (either a bcTRANSACTION or
     * a bcATTEMPT_ACQUIRE) failed (e.g. out of memory).  No parameters.
     */
};
```

**5、Binder通信协议流程**

单独看上面的协议可能很难理解，这里我们以一次Binder请求为过程来详细看一下Binder协议是如何通信的，就比较好理解了。这幅图的说明如下：

> - 1 Binder是C/S架构的，通信过程牵涉到Client、Server以及Binder驱动三个角色
> - 2 Client对于Server的请求以及Server对于Client的回复都需要通过Binder驱动来中转数据
> - 3 BC_XXX命令是进程发送给驱动命令
> - 4 BR_XXX命令是驱动发送给进程的命令
> - 5 整个通信过程由Binder驱动控制

![img](https:////upload-images.jianshu.io/upload_images/5713484-ece60dd742daca84.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder通信过程.png

PS:这里补充说明一下，通过上面的Binder协议的说明，我们看到，Binder协议的通信过程中，不仅仅是发送请求和接收数据这些命令。同时包括了对于引用计数的管理和对于死亡通知的管理(告知一方，通讯的另外一方已经死亡)。这个功能的流程和上述的功能大致一致。

## 十五、结构体binder_ptr_cookie

**binder_ptr_cookie**  用来描述一个Binder实体对象或一个Service组件的死亡通知

**1、代码位置**

位置在
 [Linux的binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h) 257行

**2、代码注释**

```cpp
struct binder_ptr_cookie {
    binder_uintptr_t ptr;
    binder_uintptr_t cookie;
};
```

> - 当描述Binder实体对象：ptr,cookie见binder_node
> - 当描述死亡通知：ptr指向一个Binder引用对象的句柄值，cookie指向接收死亡通知的对象地址

## 十六、总结

结构体就是这样的



![img](https:////upload-images.jianshu.io/upload_images/5713484-ce28e70308d4a24b.png?imageMogr2/auto-orient/strip|imageView2/2/w/478/format/webp)

如果以结构体为标的来看整个Binder传输过程则如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-ad87a4845fed2cb1.png?imageMogr2/auto-orient/strip|imageView2/2/w/856/format/webp)

结构体为标的.png







# Android跨进程通信IPC之Binder驱动

原本是没有这篇文章的，因为原来写Binder的时候没打算写Binder驱动，不过我发现后面大量的代码都涉及到了Binder驱动，如果不讲解Binder驱动，可能会对大家理解Binder造成一些折扣，我后面还是加上了这篇文章。主要内容如下：

> - 1、Binder驱动简述
> - 2、Binder驱动的核心函数
> - 3、Binder驱动的结构体
> - 4、Binder驱动通信协议
> - 5、Binder驱动内存
> - 6、附录:关于misc

驱动层的原路径(这部分代码不在AOSP中，而是位于Linux内核代码中)



```php
/kernel/drivers/android/binder.c
/kernel/include/uapi/linux/android/binder.h
```

> PS:我主要上面的源代码来分析。

或者



```swift
/kernel/drivers/staging/android/binder.c
/kernel/drivers/staging/android/uapi/binder.h
```

## 一、Binder驱动简述

### (一)、 简述

Binder驱动是Android专用的，但底层的驱动架构与Linux驱动一样。Binder驱动在misc设备上进行注册，作为虚拟字符设备，没有直接操作硬件，只对设备内存做处理。主要工作是：

> - 1、驱动设备的初始化(binder_init)
> - 2、打开(binder_open)
> - 3、映射(binder_mmap)
> - 4、数据操作(binder_ioctl)。

如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-a63bf619f32338a3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Binder驱动简述.png

## 二、系统调用

用户态的程序调用Kernel层驱动是需要陷入内核态，进行系统调用(system call，后面简写syscall)，比如打开Binder驱动方法的调用链为：open——>  _open()——> binder_open() 。open() 为用户态的函数，_open()便是系统调用(syscall)中的响应的处理函数，通过查找，调用内核态中对应的驱动binder_open()函数，至于其他的从用户态陷入内核态的流程也基本一致。

![img](https:////upload-images.jianshu.io/upload_images/5713484-13ac71586128bd39.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png

简单的说，当用户空间调用open()函数，最终会调用binder驱动的binder_open()函数；mmap()/ioctl()函数也是同理，Binder的系统中的用户态进入内核态都依赖系统调用过程。

## 三 Binder驱动的四个核心方法

### **3、binder_init()函数**

代码在[/kernel/drivers/android/binder.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/android/binder.c)
 代码如下：

```cpp
//kernel/drivers/android/binder.c      4216行
static int __init binder_init(void)
{
    int ret;
    //创建名为binder的工作队列
    binder_deferred_workqueue = create_singlethread_workqueue("binder");
    //   ****  省略部分代码  ****
    binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);
    if (binder_debugfs_dir_entry_root)
        binder_debugfs_dir_entry_proc = debugfs_create_dir("proc",
                         binder_debugfs_dir_entry_root);

    if (binder_debugfs_dir_entry_root) {
        //在debugfs文件系统中创建一系列的问题件
        //   ****  省略部分代码  ****
    }
     //   ****  省略部分代码  ****
    while ((device_name = strsep(&device_names, ","))) {
       //binder设备初始化
        ret = init_binder_device(device_name);
        if (ret)
            //binder设备初始化失败
            goto err_init_binder_device_failed;
    }
    return ret;
}
```

debugfs_create_dir是指在debugfs文件系统中创建一个目录，返回的是指向dentry的指针。当kernel中禁用debugfs的话，返回值是 -%ENODEV。默认是禁用的。如果需要打开，在目录/kernel/arch/arm64/configs/下找到目标defconfig文件中添加一行CONFIG_DEBUG_FS=y，再重新编译版本，即可打开debug_fs。

#### **3.1 init_binder_device()函数解析**

```rust
//kernel/drivers/android/binder.c      4189行
static int __init init_binder_device(const char *name)
{
    int ret;
    struct binder_device *binder_device;

    binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
    if (!binder_device)
        return -ENOMEM;

    binder_device->miscdev.fops = &binder_fops;
    binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;
    binder_device->miscdev.name = name;

    binder_device->context.binder_context_mgr_uid = INVALID_UID;
    binder_device->context.name = name;
        //注册 misc设备
    ret = misc_register(&binder_device->miscdev);
    if (ret < 0) {
        kfree(binder_device);
        return ret;
    }
    hlist_add_head(&binder_device->hlist, &binder_devices);
    return ret;
}
```

这里面主要就是通过调用misc_register()函数来注册misc设备，miscdevice结构体，便是前面注册misc设备时传递进去的参数

**3.1.1 binder_device的结构体**

这里要说一下binder_device的结构体

```cpp
//kernel/drivers/android/binder.c      234行
struct binder_device {
    struct hlist_node hlist;
    struct miscdevice miscdev;
    struct binder_context context;
};
```

在binder_device里面有一个miscdevice

然后看下

```php
//设备文件操作结构，这是file_operation结构
binder_device->miscdev.fops = &binder_fops;   
// 次设备号 动态分配
binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;
// 设备名
binder_device->miscdev.name = name;
// uid
binder_device->context.binder_context_mgr_uid = INVALID_UID;
//上下文的名字
binder_device->context.name = name;
```

如果有人对[miscdevice](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/linux/miscdevice.h)结构体有兴趣 ,可以自行研究。

**3.1.2   file_operations的结构体**

file_operations 结构体，指定相应文件操作的方法

```swift
/kernel/drivers/android/binder.c       4173行
static const struct file_operations binder_fops = {
    .owner = THIS_MODULE,
    .poll = binder_poll,
    .unlocked_ioctl = binder_ioctl,
    .compat_ioctl = binder_ioctl,
    .mmap = binder_mmap,
    .open = binder_open,
    .flush = binder_flush,
    .release = binder_release,
};
```

#### 3.2 binder_open()函数解析

开打binder驱动设备

```rust
/kernel/drivers/android/binder.c     3456行
static int binder_open(struct inode *nodp, struct file *filp)
{
        // binder进程
        struct binder_proc *proc;
        struct binder_device *binder_dev;
        binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d\n",
             current->group_leader->pid, current->pid);
        //为binder_proc结构体在再分配kernel内存空间
        proc = kzalloc(sizeof(*proc), GFP_KERNEL);
        if (proc == NULL)
             return -ENOMEM;
        get_task_struct(current);
        //当前线程的task保存在binder进程的tsk
        proc->tsk = current;
        proc->vma_vm_mm = current->mm;
        //初始化todo列表
        INIT_LIST_HEAD(&proc->todo);
         //初始化wait队列
        init_waitqueue_head(&proc->wait);
        // 当前进程的nice值转化为进程优先级
        proc->default_priority = task_nice(current);
        binder_dev = container_of(filp->private_data, struct binder_device,miscdev);
        proc->context = &binder_dev->context;
        //同步锁,因为binder支持多线程访问
        binder_lock(__func__);
        //BINDER_PROC对象创建+1
        binder_stats_created(BINDER_STAT_PROC);
        //将proc_node节点添加到binder_procs为表头的队列
        hlist_add_head(&proc->proc_node, &binder_procs);
        proc->pid = current->group_leader->pid;
        INIT_LIST_HEAD(&proc->delivered_death);
        //file文件指针的private_data变量指向binder_proc数据
        filp->private_data = proc;
        //释放同步锁
        binder_unlock(__func__);
        if (binder_debugfs_dir_entry_proc) {
        char strbuf[11];
        snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
/*
* proc debug entries are shared between contexts, so
* this will fail if the process tries to open the driver
* again with a different context. The priting code will
* anyway print all contexts that a given PID has, so this
* is not a problem.
*/
        proc->debugfs_entry =debugfs_create_file(strbuf,S_IRUGO,binder_debugfs_dir_entry_proc,(void *)(unsigned long)proc->pid,&binder_proc_fops);
    }
    return 0;
}
```

创建binder_proc对象，并把当前进程等信息保存到binder_proc对象，该对象管理IPC所需的各种新并有用其他结构体的跟结构体；再把binder_proc对象保存到文件指针filp，以及binder_proc加入到全局链表 **binder_proc**。如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-40182d6c6cda6724.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

binder_proc.png

Binder驱动通过static HIST_HEAD(binder_procs)；，创建了全局的哈希链表binder_procs，用于保存所有的binder_procs队列，每次创建的binder_proc对象都会加入binder_procs链表中。

#### 3.3 binder_mmap()函数解析

binder_mmap(文件描述符，用户虚拟内存空间)
 主要功能：首先在内核虚拟地址空间，申请一块与用户虚拟内存相同的大小的内存；然后申请1个page大小的物理内存，再讲同一块物理内存分别映射到内核虚拟内存空间和用户虚拟内存空间，从而实现了用户空间的buffer与内核空间的buffer同步操作的功能。

```php
/kernel/drivers/android/binder.c      3357行
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;
    //内核虚拟空间
    struct vm_struct *area;
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;
    struct binder_buffer *buffer;

    if (proc->tsk != current)
        return -EINVAL;
    //保证映射内存大小不超过4M
    if ((vma->vm_end - vma->vm_start) > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;

    binder_debug(BINDER_DEBUG_OPEN_CLOSE,
             "binder_mmap: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
             proc->pid, vma->vm_start, vma->vm_end,
             (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
             (unsigned long)pgprot_val(vma->vm_page_prot));

    if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
        ret = -EPERM;
        failure_string = "bad vm_flags";
        goto err_bad_arg;
    }
    vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;

    mutex_lock(&binder_mmap_lock);
    if (proc->buffer) {
        ret = -EBUSY;
        failure_string = "already mapped";
        goto err_already_mapped;
    }
     //分配一个连续的内核虚拟空间，与进程虚拟空间大小一致
    area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
    if (area == NULL) {
        ret = -ENOMEM;
        failure_string = "get_vm_area";
        goto err_get_vm_area_failed;
    }
     //指向内核虚拟空间的地址
    proc->buffer = area->addr;
      //地址便宜量=用户空间地址-内核空间地址
    proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
      // 释放锁
    mutex_unlock(&binder_mmap_lock);

#ifdef CONFIG_CPU_CACHE_VIPT
    if (cache_is_vipt_aliasing()) {
        while (CACHE_COLOUR((vma->vm_start ^ (uint32_t)proc->buffer))) {
            pr_info("binder_mmap: %d %lx-%lx maps %p bad alignment\n", proc->pid, vma->vm_start, vma->vm_end, proc->buffer);
            vma->vm_start += PAGE_SIZE;
        }
    }
#endif
        //分配物理页的指针数组，大小等于用户虚拟内存/4K
    proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
    if (proc->pages == NULL) {
        ret = -ENOMEM;
        failure_string = "alloc page array";
        goto err_alloc_pages_failed;
    }
    proc->buffer_size = vma->vm_end - vma->vm_start;

    vma->vm_ops = &binder_vm_ops;
    vma->vm_private_data = proc;
    // 分配物理页面，同时映射到内核空间和进程空间，目前只分配1个page的物理页
    if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {
        ret = -ENOMEM;
        failure_string = "alloc small buf";
        goto err_alloc_small_buf_failed;
    }
     // binder_buffer对象，指向proc的buffer地址
    buffer = proc->buffer;
     //创建进程的buffers链表头
    INIT_LIST_HEAD(&proc->buffers);
     //将binder_buffer地址  加入到所属进程的buffer队列
    list_add(&buffer->entry, &proc->buffers);
    buffer->free = 1;
     //将空闲的buffer放入proc->free_buffer中
    binder_insert_free_buffer(proc, buffer);
      // 异步可用空间大小为buffer总体大小的一半
    proc->free_async_space = proc->buffer_size / 2;
    barrier();
    proc->files = get_files_struct(current);
    proc->vma = vma;
    proc->vma_vm_mm = vma->vm_mm;

    /*pr_info("binder_mmap: %d %lx-%lx maps %p\n",
         proc->pid, vma->vm_start, vma->vm_end, proc->buffer);*/
    return 0;
//错误跳转
err_alloc_small_buf_failed:
    kfree(proc->pages);
    proc->pages = NULL;
err_alloc_pages_failed:
    mutex_lock(&binder_mmap_lock);
    vfree(proc->buffer);
    proc->buffer = NULL;
err_get_vm_area_failed:
err_already_mapped:
    mutex_unlock(&binder_mmap_lock);
err_bad_arg:
    pr_err("binder_mmap: %d %lx-%lx %s failed %d\n",
           proc->pid, vma->vm_start, vma->vm_end, failure_string, ret);
    return ret;
}
```

binder_mmap通过加锁，保证一次只有一个进程分享内存，保证多进程间的并发访问。其中user_buffer_offset是虚拟进程地址与虚拟内核地址的差值，也就是说同一物理地址，当内核地址为kernel_addr，则进程地址为proc_addr=kernel_addr+user_buffer_offset。

这里面重点说下binder_update_page_range()函数

**3.3.1  binder_update_page_range()函数解析**

```cpp
/kernel/drivers/android/binder.c      567行
static int binder_update_page_range(struct binder_proc *proc, int allocate,
                    void *start, void *end,
                    struct vm_area_struct *vma)
{
    void *page_addr;
    unsigned long user_page_addr;
    struct page **page;
    struct mm_struct *mm;

    binder_debug(BINDER_DEBUG_BUFFER_ALLOC,
             "%d: %s pages %p-%p\n", proc->pid,
             allocate ? "allocate" : "free", start, end);

    if (end <= start)
        return 0;

    trace_binder_update_page_range(proc, allocate, start, end);

    if (vma)
        mm = NULL;
    else
        mm = get_task_mm(proc->tsk);

    if (mm) {
        down_write(&mm->mmap_sem);
        vma = proc->vma;
        if (vma && mm != proc->vma_vm_mm) {
            pr_err("%d: vma mm and task mm mismatch\n",
                proc->pid);
            vma = NULL;
        }
    }

    if (allocate == 0)
        goto free_range;

    if (vma == NULL) {
        pr_err("%d: binder_alloc_buf failed to map pages in userspace, no vma\n",
            proc->pid);
        goto err_no_vma;
    }
    //  **************** 这里是重点********************
    for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
        int ret;

        page = &proc->pages[(page_addr - proc->buffer) / PAGE_SIZE];

        BUG_ON(*page);
         // 分配物理内存
        *page = alloc_page(GFP_KERNEL | __GFP_HIGHMEM | __GFP_ZERO);
        if (*page == NULL) {
            pr_err("%d: binder_alloc_buf failed for page at %p\n",
                proc->pid, page_addr);
            goto err_alloc_page_failed;
        }
        // 物理空间映射到内核空间
        ret = map_kernel_range_noflush((unsigned long)page_addr,
                    PAGE_SIZE, PAGE_KERNEL, page);
        flush_cache_vmap((unsigned long)page_addr,
                (unsigned long)page_addr + PAGE_SIZE);
        if (ret != 1) {
            pr_err("%d: binder_alloc_buf failed to map page at %p in kernel\n",
                   proc->pid, page_addr);
            goto err_map_kernel_failed;
        }
        user_page_addr =
            (uintptr_t)page_addr + proc->user_buffer_offset;
         // 物理空间映射到虚拟进程空间
        ret = vm_insert_page(vma, user_page_addr, page[0]);
        if (ret) {
            pr_err("%d: binder_alloc_buf failed to map page at %lx in userspace\n",
                   proc->pid, user_page_addr);
            goto err_vm_insert_page_failed;
        }
        /* vm_insert_page does not seem to increment the refcount */
    }
    if (mm) {
        up_write(&mm->mmap_sem);
        mmput(mm);
    }
    return 0;

free_range:
     
    for (page_addr = end - PAGE_SIZE; page_addr >= start;
         page_addr -= PAGE_SIZE) {
        page = &proc->pages[(page_addr - proc->buffer) / PAGE_SIZE];
        if (vma)
            zap_page_range(vma, (uintptr_t)page_addr +
                proc->user_buffer_offset, PAGE_SIZE);
err_vm_insert_page_failed:
        unmap_kernel_range((unsigned long)page_addr, PAGE_SIZE);
err_map_kernel_failed:
        __free_page(*page);
        *page = NULL;
err_alloc_page_failed:
        ;
    }
err_no_vma:
    if (mm) {
        up_write(&mm->mmap_sem);
        mmput(mm);
    }
    return -ENOMEM;
}
```

主要工作可用下面的图来表达：

![img](https:////upload-images.jianshu.io/upload_images/5713484-3893b4d3f7e47b8a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

binder_update_page_range.png

binder_update_page_range 主要完成工作：分配物理内存空间，将物理空间映射到内核空间，将物理空间映射到进程空间。当然binder_update_page_range 既可以分配物理页面，也可以释放物理页面

#### **3.4 binder_ioctl()函数解析**

在分析binder_ioctl()函数之前，建议大家看下我的上篇文章[Android跨进程通信IPC之7——Binder相关结构体简介](https://www.jianshu.com/p/5740a8447324)了解相关的结构体，为了便于查找，这些结构体之间都留有字段的存储关联的结构，下面的这幅图描述了这里说到的内容


![img](https:////upload-images.jianshu.io/upload_images/5713484-1e999ff8f503c4e7.png?imageMogr2/auto-orient/strip|imageView2/2/w/1042/format/webp)

相关结构体.png



现在让我们详细分析下binder_ioctl()函数

> - binder_ioctl()函数负责在两个进程间收发IPC数据和IPC reply数据
> - ioctl(文件描述符，ioctl命令，数据类型)
> - ioctl文件描述符，是通过open()方法打开Binder Driver后返回值。
> - ioctl命令和数据类型是一体，不同的命令对应不同的数据类型

下面这些命令中BINDER_WRITE_READ使用最为频繁，也是ioctl的最为核心的命令。



![img](https:////upload-images.jianshu.io/upload_images/5713484-d78303d96113b1f4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

ioctl.png

代码如下：



```cpp
//kernel/drivers/android/binder.c       3239行
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
        //binder线程
    struct binder_thread *thread;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;

    /*pr_info("binder_ioctl: %d:%d %x %lx\n",
            proc->pid, current->pid, cmd, arg);*/

    if (unlikely(current->mm != proc->vma_vm_mm)) {
        pr_err("current mm mismatch proc mm\n");
        return -EINVAL;
    }
    trace_binder_ioctl(cmd, arg);
        //进入休眠状态，直到中断唤醒
    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret)
        goto err_unlocked;

    binder_lock(__func__);
        //获取binder_thread
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    switch (cmd) {
        //进行binder的读写操作
    case BINDER_WRITE_READ:
        ret = binder_ioctl_write_read(filp, cmd, arg, thread);
        if (ret)
            goto err;
        break;
        //设置binder的最大支持线程数
    case BINDER_SET_MAX_THREADS:
        if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
            ret = -EINVAL;
            goto err;
        }
        break;
        //设置binder的context管理者，也就是servicemanager称为守护进程
    case BINDER_SET_CONTEXT_MGR:
        ret = binder_ioctl_set_ctx_mgr(filp);
        if (ret)
            goto err;
        break;
        //当binder线程退出，释放binder线程
    case BINDER_THREAD_EXIT:
        binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit\n",
                 proc->pid, thread->pid);
        binder_free_thread(proc, thread);
        thread = NULL;
        break;
         //获取binder的版本号
    case BINDER_VERSION: {
        struct binder_version __user *ver = ubuf;

        if (size != sizeof(struct binder_version)) {
            ret = -EINVAL;
            goto err;
        }
        if (put_user(BINDER_CURRENT_PROTOCOL_VERSION,
                 &ver->protocol_version)) {
            ret = -EINVAL;
            goto err;
        }
        break;
    }
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
err:
    if (thread)
        thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
    binder_unlock(__func__);
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret && ret != -ERESTARTSYS)
        pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
    trace_binder_ioctl_done(ret);
    return ret;
}
```

这里面重点说两个函数binder_get_thread()函数和binder_ioctl_write_read()函数

**3.4.1 binder_get_thread()函数解析**

> 从binder_proc中查找binder_thread，如果当前线程已经加入到proc的线程队列则直接返回，如果不存在则创建binder_thread，并将当前线程添加到当前的proc

```rust
//kernel/drivers/android/binder.c      3026行
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
    struct binder_thread *thread = NULL;
    struct rb_node *parent = NULL;
    struct rb_node **p = &proc->threads.rb_node;
        //根据当前进程的pid，从binder_proc中查找相应的binder_thread
    while (*p) {
        parent = *p;
        thread = rb_entry(parent, struct binder_thread, rb_node);

        if (current->pid < thread->pid)
            p = &(*p)->rb_left;
        else if (current->pid > thread->pid)
            p = &(*p)->rb_right;
        else
            break;
    }
    if (*p == NULL) {
                // 新建binder_thread结构体
        thread = kzalloc(sizeof(*thread), GFP_KERNEL);
        if (thread == NULL)
            return NULL;
        binder_stats_created(BINDER_STAT_THREAD);
        thread->proc = proc;
                //保持当前进程(线程)的pid
        thread->pid = current->pid;
        init_waitqueue_head(&thread->wait);
        INIT_LIST_HEAD(&thread->todo);
        rb_link_node(&thread->rb_node, parent, p);
        rb_insert_color(&thread->rb_node, &proc->threads);
        thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;
        thread->return_error = BR_OK;
        thread->return_error2 = BR_OK;
    }
      return thread;
}
```

**3.4.2 binder_ioctl_write_read()函数解析**

> 对于ioctl()方法中，传递进来的命令是cmd = BINDER_WRITE_READ时执行该方法，arg是一个binder_write_read结构体

```objectivec
/kernel/drivers/android/binder.c      3134行
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
         //把用户控件数据ubuf拷贝到bwr
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
    binder_debug(BINDER_DEBUG_READ_WRITE,
             "%d:%d write %lld at %016llx, read %lld at %016llx\n",
             proc->pid, thread->pid,
             (u64)bwr.write_size, (u64)bwr.write_buffer,
             (u64)bwr.read_size, (u64)bwr.read_buffer);
        //如果写缓存中有数据
    if (bwr.write_size > 0) {
                //执行binder写操作
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        trace_binder_write_done(ret);
                //如果写失败，再将bwr数据写回用户空间，并返回
        if (ret < 0) {
            bwr.read_consumed = 0;
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
        //当读缓存有数据
    if (bwr.read_size > 0) {
                 //执行binder读操作
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
                     bwr.read_size,
                     &bwr.read_consumed,
                     filp->f_flags & O_NONBLOCK);
        trace_binder_read_done(ret);
        if (!list_empty(&proc->todo))
            wake_up_interruptible(&proc->wait);
                 //如果读失败
        if (ret < 0) {
                        //当读失败，再将bwr数据写回用户空间，并返回
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
         //将内核数据bwr拷贝到用户空间ubuf
    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}
```

对于binder_ioctl_write_read流程图如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-a3647098d270c5e0.png?imageMogr2/auto-orient/strip|imageView2/2/w/1124/format/webp)

binder_ioctl_write_read流程图.png

流程：

> - 1 首先把用户空间的数据拷贝到内核空间bwr
> - 2 其次当bwr写缓存中有数据，则执行binder写操作。如果写失败，则再将bwr数据写回用户控件，并退出。
> - 3 再次当bwr读缓存中有数据，则执行binder读缓存；当读失败，再将bwr数据写会用户空间，并退出。
> - 4 最后把内核数据拷贝到用户空间。

#### 3.5 总结

本章主要讲解了binder驱动的的四大功能点

> - 1 binder_init ：初始化字符设备
> - 2 binder_open：打开驱动设备，过程需要持有binder_main_lock同步锁
> - 3 binder_mmap:  申请内存空间，该过程需要持有binder_mmap_lock同步锁；
> - 4 binder_ioctl：执行相应的io操作，该过程需要持有binder_main_lock同步锁；当处于binder_thread_read过程，却读缓存无数据则会先释放该同步锁，并处于wait_event_freezable过程，等有数据到来则唤醒并尝试持有同步锁。

下面我们重点介绍下binder_thread_write()函数和binder_thread_read()函数

## 四 Binder驱动通信

### (一) Binder驱动通信简述

> Client进程通过RPC(Remote Procedure Call Protocol) 与Server通信，可以简单地划分为三层:  1、驱动层  2、IPC层  3、业务层。**下图的doAction()**便是Client与Server共同协商好的统一方法；其中handle、PRC数据、代码、协议、这4项组成了IPC层的数据，通过IPC层进行数据传输；而真正在Client和Server两端建立通信的基础是Binder Driver

模型如下图:

![img](https:////upload-images.jianshu.io/upload_images/5713484-dae563576b4d9821.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

binder通信模型.png

### (二) Binder驱动通信协议

先来一发完整的Binder通信过程，如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-68a3eda6f5f95cc3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

binder流程.png

Binder协议包含在IPC数据中，分为两类：

> - BINDER_COMMAND_PROTOCOL:binder请求码，以"BC_" 开头，简称"BC码"，从IPC层传递到 Binder Driver层；
> - BINDER_RETURN_PROTOCOL: binder响应码，以"BR_"开头，简称"BR码"，用于从BinderDirver层传递到IPC层

Binder IPC 通信至少是两个进程的交互:

> - client进程执行binder_thread_write，根据BC_XXX 命令，生成相应的binder_work；
> - server进程执行binder_thread_read，根据binder_work.type类型，生成BR_XXX，发送用户处理。

如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-9b6b2b8f120c1b60.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

进程.png

其中binder_work.type共有6种类型



```rust
//kernel/drivers/android/binder.c      240行
struct binder_work {
    struct list_head entry;
    enum {
        BINDER_WORK_TRANSACTION = 1
        BINDER_WORK_TRANSACTION_COMPLETE,
        BINDER_WORK_NODE,
        BINDER_WORK_DEAD_BINDER,
        BINDER_WORK_DEAD_BINDER_AND_CLEAR,
        BINDER_WORK_CLEAR_DEATH_NOTIFICATION,
    } type;
};
```

这里用到了上面提到两个函数一个是binder_thread_write()函数，另一个是binder_thread_read函数骂我们就来详细研究下：

### (三)、通信函数

#### 1、binder_thread_write() 函数详解

> 请求处理过程是通过binder_thread_write()函数，该方法用于处理Binder协议的请求码。当binder_buffer存在数据，binder线程的写操作循环执行
>  代码在   kernel/drivers/android/binder.c   2248行。

代码太多了就不全部粘贴了，只粘贴重点部分，代码如下：



```cpp
static int binder_thread_write(){
    while (ptr < end && thread->return_error == BR_OK) {
        //获取IPC数据中的Binder协议的BC码
        if (get_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        switch (cmd) {
            case BC_INCREFS: ...
            case BC_ACQUIRE: ...
            case BC_RELEASE: ...
            case BC_DECREFS: ...
            case BC_INCREFS_DONE: ...
            case BC_ACQUIRE_DONE: ...
            case BC_FREE_BUFFER: ...
            
            case BC_TRANSACTION:
            case BC_REPLY: {
                struct binder_transaction_data tr;
                //拷贝用户控件tr到内核
                copy_from_user(&tr, ptr, sizeof(tr))；
                binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
                break;
            case BC_REGISTER_LOOPER: ...
            case BC_ENTER_LOOPER: ...
            case BC_EXIT_LOOPER: ...
            case BC_REQUEST_DEATH_NOTIFICATION: ...
            case BC_CLEAR_DEATH_NOTIFICATION:  ...
            case BC_DEAD_BINDER_DONE: ...
            }
        }
    }
}
```

对于 请求码为BC_TRANSCATION或BC_REPLY时，会执行binder_transaction()方法，这是最频繁的操作。那我们一起来看下binder_transaction()函数。

**1.1、binder_transaction() 函数详解**

这块的代码依旧很多，我就只粘贴重点了，全部代码在/kernel/drivers/android/binder.c 1827行。

```csharp
static void binder_transaction(struct binder_proc *proc,
               struct binder_thread *thread,
               struct binder_transaction_data *tr, int reply){
    //根绝各种判断，获取如下信息：
    //目标进程
    struct binder_proc *target_proc；  
    // 目标线程
    struct binder_thread *target_thread； 
     // 目标binder节点
    struct binder_node *target_node；    
    //目标TODO队列
    struct list_head *target_list；
    // 目标等待队列     
    wait_queue_head_t *target_wait；    
   //*** 省略部分代码 ***
   
    //分配两个结构体内存
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    if (t == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_alloc_t_failed;
    }
    binder_stats_created(BINDER_STAT_TRANSACTION);
    *tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    //*** 省略部分代码 ***
    //从target_proc分配一块buffer
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
     //*** 省略部分代码 ***
    for (; offp < off_end; offp++) {
     //*** 省略部分代码 ***
        switch (fp->type) {
        case BINDER_TYPE_BINDER: ...
        case BINDER_TYPE_WEAK_BINDER: ...
        case BINDER_TYPE_HANDLE: ...
        case BINDER_TYPE_WEAK_HANDLE: ...
        case BINDER_TYPE_FD: ...
        }
    }
    //分别给target_list和当前线程TODO队列插入事务
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
}
```

replay的过程会找到 target_thread，非reply则一般找到target_proc，对于特殊的嵌套binder call会根据transaction_stack来决定是否插入事物到目标进程。

**1.2、BC_PROTOCOL  请求码**

> Binder请求码，在[binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h)里面的366行。是用**binder_driver_command_protocol** 来定义的，是用于应用程序向Binder驱动设备发送请求消息，应用程序包含Client端和Server端，以BC_开头，供给17条 ( - 表示目前不支持请求码)

![img](https:////upload-images.jianshu.io/upload_images/5713484-ff408b3f4cf683ee.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

BC请求码.png

重点说几个：

> - BC_FREE_BUFFER:通过mmap()映射内存，其中ServiceMananger映射的空间大小为128K，其他Binder应用的进程映射的内存大小为8K-1M，Binder驱动基于这块映射的内存采用最佳匹配算法来动态分配和释放，通过**binder_buffer**结构体中**free**字段来表示相应的buffer是空闲还是已分配状态。对于已分配的buffer加入binder_proc中的allocated_buffers红黑树；对于空闲的buffers加入到binder_proch中的free_buffer红黑树。当应用程序需要内存时，根据所需内存大小从free_buffers中找到最合适的内存，并放入allocated_buffers树；当应用程序处理完后，必须尽快用BC_FREE_BUFFER命令来释放该buffer，从而添加回free_buffers树中。
> - BC_INCREFS、BC_ACQUIRE、BC_RELEASE、BC_DECREFS等请求码的作用是对binder的 强弱引用的技术操作，用于实现强/弱 指针的功能。
> - 对于参数类型 binder_ptr_cookie是由binder指针和cookie组成
> - Binder线程创建与退出：
> - BC_ENTER_LOOPER: binder主线程(由应用层发起)的创建会向驱动发送该消息；joinThreadPool()过程创建binder主线程；
> - BC_REGISTER_LOOPER: Binder用于驱动层决策而创建新的binder线程；joinThreadPool()过程，创建非binder主线程；
> - BC_EXIT_LOOPER:退出Binder线程，对于binder主线程是不能退出的；joinThreadPool的过程出现timeout，并且非binder主线程，则会退出该binder线程。

**1.3、binder_thread_read()函数详解**

响应处理过程是通过binder_thread_read()函数，该方法根据不同的binder_work->type以及不同的状态生成不同的响应码
 代码太多了，我只截取了部分代码，具体代码在/kernel/drivers/android/binder.c  2650行

```php
static int binder_thread_read（）{
     //*** 省略部分代码 ***
    // 根据 wait_for_proc_work 来决定wait在当前线程还是进程的等待队列
    if (wait_for_proc_work) {
         //*** 省略部分代码 ***
        ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
          //*** 省略部分代码 ***
    } else {
        //*** 省略部分代码 ***
        ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
       //*** 省略部分代码 ***
    }
   //*** 省略部分代码 ***
    while (1) {
         //*** 省略部分代码 ***
         //当&thread->todo和&proc->todo都为空时，goto到retry标志处，否则往下执行：
    if (!list_empty(&thread->todo)) {
        w = list_first_entry(&thread->todo, struct binder_work,entry);
    } else if (!list_empty(&proc->todo) && wait_for_proc_work) {
        w = list_first_entry(&proc->todo, struct binder_work,entry);
    } else {
        /* no data added */
        if (ptr - buffer == 4 &&
             !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN))
                goto retry;
        break;
    }
        struct binder_transaction_data tr;
        struct binder_transaction *t = NULL;
        switch (w->type) {
          case BINDER_WORK_TRANSACTION: ...
          case BINDER_WORK_TRANSACTION_COMPLETE: ...
          case BINDER_WORK_NODE: ...
          case BINDER_WORK_DEAD_BINDER: ...
          case BINDER_WORK_DEAD_BINDER_AND_CLEAR: ...
          case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: ...
        }
        ...
    }
done:
    *consumed = ptr - buffer;
        //当满足请求线程  加上 已准备线程数 等于0  并且 其已启动线程小于最大线程数15，并且looper状态为已注册或已进入时创建新的线程。
    if (proc->requested_threads + proc->ready_threads == 0 &&
        proc->requested_threads_started < proc->max_threads &&
        (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
         /*spawn a new thread if we leave this out */) {
        proc->requested_threads++;
        binder_debug(BINDER_DEBUG_THREADS,
                 "%d:%d BR_SPAWN_LOOPER\n",
                 proc->pid, thread->pid);
                //生成BR_SPAWN_LOOPER命令，用于创建新的线程
        if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
            return -EFAULT;
        binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
    }
    return 0;
}
```

当transaction堆栈为空，且线程todo链表为空，且non_block=false时，意味着没有任务事物需要处理，会进入等待客户端请求的状态。当有事物需要处理时便会进入循环处理过程，并生成相应的响应码。

> 在Binder驱动层面，只有进入binder_thread_read()方法时，同时满足以下条件，才会生成BR_SPAWN_LOOPER命令，当用户态进程收到该命令则会创新新线程：
>
> - binder_proc的requested_threads线程数为0
> - binder_proc的ready_threads线程数为0
> - binder_proc的requested_threads_started个数小于15(即最大线程个数)
> - binder_thread的looper状态为BINDER_LOOPER_STATE_REGISTERED或者BINDER_LOOPER_STATE_ENTERED

那么问题来了，什么时候处理的响应码？通过上面的Binder通信协议图，可以知道处理响应码的过程是在用户态处理，下一篇文章会讲解到用户控件IPCThreadState类中IPCThreadState::waitForResponse()和IPCThreadState::executeCommand()两个方法共同处理Binder协议的18个响应码

**1.3、BR_PROTOCOL  响应码**

> binder响应码，在[binder.h](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/include/uapi/linux/android/binder.h)里面的278行是用enum binder_driver_return_protocol来定义的，是binder设备向应用程序回复的消息，应用程序包含Client端和Serve端，以BR_开头，合计18条。

![img](https:////upload-images.jianshu.io/upload_images/5713484-18c1965c5e32c306.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

BR响应码.png

说几个难点：

> - BR_SPAWN_LOOPER:binder驱动已经检测到进程中没有线程等待即将到来的事物。那么当一个进程接收到这条命令时，该进程必须创建一条新的服务线程并注册该线程，在接下来的响应过程会看到合何时生成该响应嘛
> - BR_TRANSACTION_COMPLETE:当Client端向Binder驱动发送BC_TRANSACTION命令后，Client会受到BR_TRANSACTION_COMPLETE命令，告知Client端请求命令发送成功能；对于Server向Binder驱动发送BC_REPLY命令后，Server端会受到BR_TRANSACTION_COMPLETE命令，告知Server端请求回应命令发送成功。
> - BR_READ_REPLY: 当应用层向Binder驱动发送Binder调用时，若Binder应用层的另一个端已经死亡，则驱动回应BR_DEAD_BINDER命令。
> - BR_FAILED_REPLY:当应用层向Binder驱动发送Binder调用是，若transcation出错，比如调用的函数号不存在，则驱动回应BR_FAILED_REPLY。

## 五、Binder内存

为了让大家更好的理解Binder机制，特意插播一条"广告"，主要是讲解Binder内存，Binder内存我主要分3个内容来依次讲解，分别为：

> - 1、mmap机制
> - 2、内存分配
> - 3、内存释放

### (一) mmap机制

上面从代码的角度阐释了binder_mmap，也是Binder进程通信效率高的核心机制所在，如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-e431534704111fb7.png?imageMogr2/auto-orient/strip|imageView2/2/w/1196/format/webp)

mmap机制1.png

![img](https:////upload-images.jianshu.io/upload_images/5713484-b3b1f8e3bdb5f402.png?imageMogr2/auto-orient/strip|imageView2/2/w/669/format/webp)

mmap机制2.png

虚拟进程地址空间(vm_area_struct)和虚拟内核地址空间(vm_struct)都映射到同一块物理内存空间。当Client端与Server端发送数据时，Client(作为数据发送端)先从自己的进程空间把IPC通信数据copy_from_user拷贝到内核空间，而Server端(作为数据接收端)与内核共享数据，不再需要拷贝数据，而是通过内存地址空间的偏移量，即可获悉内存地址，整个过程只要发生一次内存拷贝。一般地做法，需要Client端进程空间拷贝到内核空间，再由内核空间拷贝到Server进程，会发生两次拷贝。

> 对于进程和内核虚拟地址映射到同一个物理内存的操作是发生在数据接收端，而数据发送端还是需要将用户态的数据复制到内核态。到此，可能会有同学会问，为什么不直接让发送端和接收端直接映射到同一个物理空间，这样不就是连一次复制操作都不需要了，0次复制操作就是与Linux标准内核的共享内存你的IPC机制没有区别了，对于共享内存虽然效率高，但是对于多进程的同步问题比较复杂，而管道/消息队列等IPC需要复制2次，效率低。关于Linux效率的对比，请看前面的文章。总之Android选择Binder是基于速度和安全性的考虑。

下面这图是从Binder在进程间数据通信的流程图，从图中更能明白Binder的内存转移关系。

![img](https:////upload-images.jianshu.io/upload_images/5713484-cb5801de7e750c73.png?imageMogr2/auto-orient/strip|imageView2/2/w/956/format/webp)

binder内存转移.png

### (二) 内存分配

> Binder内存分配方法通过binder_alloc_buf()方法，内存管理单元为binder_buffer结构体，只有在binder_transaction过程中才需要分配buffer。具体代码在/kernel/drivers/android/binder.c 678行，我选取了重点部分

```rust
static struct binder_buffer *binder_alloc_buf(struct binder_proc *proc,
                          size_t data_size, size_t offsets_size, int is_async)
{ 
     //*** 省略部分代码 ***
      //如果不存在虚拟地址空间为，则直接返回
    if (proc->vma == NULL) {
        pr_err("%d: binder_alloc_buf, no vma\n",
               proc->pid);
        return NULL;
    }
        
    data_offsets_size = ALIGN(data_size, sizeof(void *)) +
        ALIGN(offsets_size, sizeof(void *));
        //非法的size，直接返回
    if (data_offsets_size < data_size || data_offsets_size < offsets_size) {
        binder_user_error("%d: got transaction with invalid size %zd-%zd\n",
                proc->pid, data_size, offsets_size);
        return NULL;
    }
    size = data_offsets_size + ALIGN(extra_buffers_size, sizeof(void *));
        //非法的size，直接返回
    if (size < data_offsets_size || size < extra_buffers_size) {
        binder_user_error("%d: got transaction with invalid extra_buffers_size %zd\n",
                  proc->pid, extra_buffers_size);
        return NULL;
    }
        //如果 剩余的异步空间太少，以至于满足需求，也直接返回
    if (is_async &&
        proc->free_async_space < size + sizeof(struct binder_buffer)) {
        binder_debug(BINDER_DEBUG_BUFFER_ALLOC,
                 "%d: binder_alloc_buf size %zd failed, no async space left\n",
                  proc->pid, size);
        return NULL;
    }
    //从binder_buffer的红黑树丛中，查找大小相等的buffer块
        while (n) {
        buffer = rb_entry(n, struct binder_buffer, rb_node);
        BUG_ON(!buffer->free);
        buffer_size = binder_buffer_size(proc, buffer);

        if (size < buffer_size) {
            best_fit = n;
            n = n->rb_left;
        } else if (size > buffer_size)
            n = n->rb_right;
        else {
            best_fit = n;
            break;
        }
    }
   //如果内存分配失败，地址为空
    if (best_fit == NULL) {
        pr_err("%d: binder_alloc_buf size %zd failed, no address space\n",
            proc->pid, size);
        return NULL;
    }
    if (n == NULL) {
        buffer = rb_entry(best_fit, struct binder_buffer, rb_node);
        buffer_size = binder_buffer_size(proc, buffer);
    }
    binder_debug(BINDER_DEBUG_BUFFER_ALLOC,
             "%d: binder_alloc_buf size %zd got buffer %p size %zd\n",
              proc->pid, size, buffer, buffer_size);

    has_page_addr =
        (void *)(((uintptr_t)buffer->data + buffer_size) & PAGE_MASK);
    if (n == NULL) {
        if (size + sizeof(struct binder_buffer) + 4 >= buffer_size)
            buffer_size = size; /* no room for other buffers */
        else
            buffer_size = size + sizeof(struct binder_buffer);
    }
    end_page_addr =
        (void *)PAGE_ALIGN((uintptr_t)buffer->data + buffer_size);
    if (end_page_addr > has_page_addr)
        end_page_addr = has_page_addr;
    if (binder_update_page_range(proc, 1,
        (void *)PAGE_ALIGN((uintptr_t)buffer->data), end_page_addr, NULL))
        return NULL;

    rb_erase(best_fit, &proc->free_buffers);
    buffer->free = 0;
    binder_insert_allocated_buffer(proc, buffer);
    if (buffer_size != size) {
        struct binder_buffer *new_buffer = (void *)buffer->data + size;

        list_add(&new_buffer->entry, &buffer->entry);
        new_buffer->free = 1;
        binder_insert_free_buffer(proc, new_buffer);
    }
 binder_debug(BINDER_DEBUG_BUFFER_ALLOC,
             "%d: binder_alloc_buf size %zd got %p\n",
              proc->pid, size, buffer);
    buffer->data_size = data_size;
    buffer->offsets_size = offsets_size;
    buffer->extra_buffers_size = extra_buffers_size;
    buffer->async_transaction = is_async;
    if (is_async) {
        proc->free_async_space -= size + sizeof(struct binder_buffer);
        binder_debug(BINDER_DEBUG_BUFFER_ALLOC_ASYNC,
                 "%d: binder_alloc_buf size %zd async free %zd\n",
                  proc->pid, size, proc->free_async_space);
    }
    return buffer;
}
```

### (三)、内存释放

内存释放相关函数：

> - binder_free_buf() :  在/kernel/drivers/android/binder.c     847行
> - binder_delete_free_buffer()：在/kernel/drivers/android/binder.c   802行
> - binder_transaction_buffer_release()：在/kernel/drivers/android/binder.c    1430行

大家有兴趣可以自己去研究下，这里就不详细解说了。

## 六、附录:关于misc

### (一)、linux子系统-miscdevice

> 杂项设备(miscdevice) 是嵌入式系统中用的比较多的一种设备类型。在Linux驱动中把无法归类的五花八门的设备定义为混杂设备(用miscdevice结构体表述)。Linux内核所提供的miscdevice有很强的包容性，各种无法归结为标准字符设备的类型都可以定义为miscdevice，譬如NVRAM,看门狗，实时时钟，字符LCD等，就像一组大杂烩。

在Linux内核里把所有misc设备组织在一起，构成一个子系统(subsys)，统一进行管理。在这个子系统里的所有miscdevice类型的设备共享一个主设备号MISC_MAJOR(即10)，这次设备号不同。

在内核中用struct  miscdevice表示miscdevice设备，具体的定义在内核头文件"include/linux/miscdevice.h"中

```cpp
//https://github.com/torvalds/linux/blob/master/include/linux/miscdevice.h    63行
struct miscdevice  {
    int minor;
    const char *name;
    const struct file_operations *fops;
    struct list_head list;
    struct device *parent;
    struct device *this_device;
    const char *nodename;
    mode_t mode;
};
```

miscdevice的API的实现在[drivers/char/misc.c](https://link.jianshu.com?t=https://github.com/torvalds/linux/blob/master/drivers/char/misc.c)中，misc子系统的初始化，以及misc设备的注册，注销等接口都实现在这个文件中。通过阅读这个文件，miscdevice类型的设备实际上就是对字符设备的简单封装，最直接的证据可以看misdevice子系统的初始化函数misc_init()，在这个函数里，撇开其他代码不看，其中有如下两行关键代码:

```cpp
//drivers/char/misc.c  279行
...
misc_class = class_create(THIS_MODULE, "misc");
...
if (register_chrdev(MISC_MAJOR,"misc",&misc_fops))
```

代码解析：

> - 第一行，创建了一个名字叫misc的类，具体表现是在/sys/class目录下创建一个名为misc的目录。以后美注册一个自己的miscdevice都会在该目录下新建一项。
> - 第二行，调用register_chrdev为给定的主设备号MISC_MAJOR(10)注册0~255供256个次设备号，并为每个设备建立一个对应的默认的cdev结构，该函数是2.6内核之前的老函数，现在已经不建议使用了。由此可见misc设备其实也就是主设备号是MISC_MAJOR(10)的字符设备。从面相对象的角度来看，字符设备类是misc设备类的父类。同时我们也主要注意到采用这个函数注册后实际上系统最多支持有255个驱动自定义的杂项设备，因为杂项设备子系统模块自己占用了一个次设备号0。查看源文件可知，目前内核里已经被预先使用的子设备号定义在include/linux/miscdevice.h的开头

在这个子系统中所有的miscdevice设备形成了一个链表，他们共享一个主设备号10，但它们有不同的次设备号。对设备访问时内核根据次设备号查找对应的miscdevice设备。这一点我们可以通过阅读misc子系统提供的注册接口函数misc_register()和注销接口函数misc_deregister()来理解。这就不深入讲解了，有兴趣的可以自己去研究。

在这个子系统中所有的miscdevice设备不仅共享了主设备号，还共享了一个misc_open()的文件操作方法。



```cpp
//drivers/char/misc.c       161行
static const struct file_operations misc_fops = {
    .owner        = THIS_MODULE,
    .open        = misc_open,
};
```

该方法结构在注册设备时通过register_chrdev(MISC_MAJOR,"misc",&misc_fops)传递给内核。但不要以为所有的miscdevice都使用相同的文件open方法，仔细阅读misc_open()我们发现该函数内部会检查驱动模块自已定义的miscdevice结构体对象的fops成员，如果不为空将其替换掉缺省的，参考函数中的new_fops = fops_get(c->fops);以及file->f_op = new_fops;语句。如此这般，以后内核再调用设备文件的操作时就会调用该misc设备自己定义的文件操作函数了。这种实现方式有点类似java里面函数重载的概念。

### (二)采用miscdevice开发设备驱动的方法

大致的步骤如下：(大家联想下binder驱动)

- 第一步，定义自己misc设备的文件操作函数以及file_operations结构体对象。如下：

```cpp
static const struct file_operations my_misc_drv_fops = {
    .owner    = THIS_MODULE,
    .open    = my_misc_drv_open,
    .release = my_misc_drv_release,
    //根据实际情况扩充 ...
};
```

- 第二步，定义自己的misc设备对象，如下：

```cpp
static struct miscdevice my_misc_drv_dev = {
    .minor    = MISC_DYNAMIC_MINOR,
    .name    = KBUILD_MODNAME,
    .fops    = &my_misc_drv_fops,
};
```

> - .minor 如果填充MISC_DYNAMIC_MINOR，则由内核动态分配次设备号，否则根据你自己定义的指定。
> - .name 给定设备的名字，也可以直接利用内核编译系统的环境变量KBUILD_MODNAME。
> - .fops设置为第一步定义的文件操作结构体的地址

- 第三步，注销和销毁

驱动模块一般在模块初始化函数中调用misc_register()注册自己的misc设备。实例代码如下：

```cpp
ret = misc_register(&my_misc_drv_dev);
if (ret < 0) {
    //失败处理
}
```

注意在misc_register()函数中有如下语句

```php
misc->this_device = device_create(misc_class, misc->parent, dev,
                  misc, "%s", misc->name);
```

这句话配合前面在misc_init()函数中的misc_class = class_create(THIS_MODULE, "misc");

> 这样，class_create()创建了一个类，而device_create()就创建了该类的一个设备，这些都涉及linux内核的设备模型和sys文件系统额概念，暂不展开，我们只需要知道，如此这般，当该驱动模块被加载(insmod)时，和内核态的设备模型配套运行的用户态有个udev的后台服务会自动在/dev下创建一个驱动模块中注册的misc设备对应的设备文件节点，其名字就是misc->name。这样就省去了自己创建设备文件的麻烦。这样也有助于动态设备的管理。

驱动模块可以在模块卸载函数中调用misc_deregister()注销自己的misc设备。实例代码如下：

```undefined
misc_deregister(&my_misc_drv_dev);
```

在这个函数中会自动删除`/dev下的同名设备文件。

### (三)总结

> 杂项设备作为字符设备的封装，为字符设备提供简单的编程接口，如果编写新的字符驱动，可以考虑使用杂项设备接口，简单粗暴，只需要初始化一个miscdevice的结构体设备对象，然后调用misc_register注册就可以了，不用的时候，调用misc_deregister进行卸载。
