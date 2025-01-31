---
title: Linux 设备驱动开发详解
categories:
  - Linux 学习
  - 内核层
abbrlink: 484892ff
date: 2024-10-24 15:00:00
updated: 2024-12-11 15:40:00
---

<meta name="referrer" content="no-referrer"/>

本书基于 Linux 4.0 内核编写，个人学习测试的内核版本是 5.15.167。

# 内核及内核编程

## Linux 2.6 后的内核特点

Linux 2.6 相对于 Linux 2.4 有着相当大的改进，主要表现为以下几个方面。

### 新的调度器

Linux 2.6 以后版本的 Linux 内核使用了新的进程调度算法，它在高负载的情况下有极其出色的性能，并且当有很多处理器时也可以很好地扩展。在 Linux 内核 2.6 的早期采用了 O（1）算法，之后转移到 CFS（Completely Fair Scheduler，完全公平调度）算法。在 Linux 3.14 中，也增加了一个新的调度类：SCHED_DEADLINE，它实现了 EDF（Earliest Deadline First，最早截止期限优先）调度算法。

<!-- more -->

### 内核抢占

在 Linux 2.6 以后版本的 Linux 内核中，一个内核任务可以被抢占，从而提高系统的实时性。这样做最主要的优势在于，可以极大地增强系统的用户交互性，用户将会觉得鼠标单击和击键的事件得到了更快速的响应。Linux 2.6 以后的内核版本还是存在一些不可抢占的区间，如中断上下文、软中断上下文和自旋锁锁住的区间。如果给 Linux 内核打上 RT-Preempt 补丁，则中断和软中断都被线程化了，自旋锁也被互斥体替换，Linux 内核变得能支持硬实时。

### 改进的线程模型

Linux 2.6 以后版本中的线程采用 NPTL（Native POSIX Thread Library，本地 POSIX 线程库）模型，操作速度得以极大提高，相比于 Linux 2.4 内核时代的 LinuxThreads 模型，它也更加遵循 POSIX 规范的要求。NPTL 没有使用 LinuxThreads 模型中采用的管理线程，内核本身也增加了 FUTEX（Fast Userspace Mutex，快速用户态互斥体），从而减小多线程的通信开销。

### 虚拟内存的变化

从虚拟内存的角度来看，新内核融合了 r-map（反向映射）技术，显著改善虚拟内存在一定大小负载下的性能。在 Linux 2.4 中，要回收页时，内核的做法是遍历每个进程的所有 PTE 以判断该 PTE 是否与该页建立了映射，如果建立了，则取消该映射，最后无 PTE 与该页相关联后才回收该页。在 Linux 2.6 后，则建立反向映射，可以通过页结构体快速寻找到页面的映射。

### 文件系统

Linux 2.6 版内核增加了对日志文件系统功能的支持，解决了 Linux 2.4 版本在这方面的不足。Linux 2.6 版内核在文件系统上的关键变化还包括对扩展属性及 POSIX 标准访问控制的支持。ext2/ext3/ext4 作为大多数 Linux 系统默认安装的文件系统，在 Linux 2.6 版内核中增加了对扩展属性的支持，可以给指定的文件在文件系统中嵌入元数据。 

在文件系统方面，基于 B 树的 Btrfs，称为是下一代 Linux 文件系统，它在扩展性、数据一致性、多设备管理和针对 SSD 的优化等方面都优于 ext4。

## 内核的组成

### 内核源码的目录结构

Linux 内核源代码包含以下目录，可通过网站 [https://elixir.bootlin.com/](https://elixir.bootlin.com/) 浏览。

- arch：包含和硬件体系结构相关的代码，每种平台占一个相应的目录，如 i386、arm、arm64、powerpc、mips 等。Linux 内核目前已经支持 30 种左右的体系结构。在 arch 目录下，存放的是各个平台以及各个平台的芯片对 Linux 内核进程调度、内存管理、中断等的支持，以及每个具体的 SoC 和电路板的板级支持代码。
- block：块设备驱动程序 I/O 调度。
- crypto：常用加密和散列算法（如 AES、SHA 等），还有一些压缩和 CRC 校验算法。
- documentation：内核各部分的通用解释和注释。
- drivers：设备驱动程序，每个不同的驱动占用一个子目录，如 char、block、net、mtd、i2c 等。
- fs：所支持的各种文件系统，如 EXT、FAT、NTFS、JFFS2 等。
- include：头文件，与系统相关的头文件放置在 include/linux 子目录下。
- init：内核初始化代码。著名的 start_kernel() 就位于 init/main.c 文件中。
- ipc：进程间通信的代码。
- kernel：内核最核心的部分，包括进程调度、定时器等，而和平台相关的一部分代码放在 arch/*/kernel 目录下。
- lib：库文件代码。
- mm：内存管理代码，和平台相关的一部分代码放在 arch/*/mm 目录下。
- net：网络相关代码，实现各种常见的网络协议。
- scripts：用于配置内核的脚本文件。
- security：主要是一个 SELinux 的模块。
- sound：ALSA、OSS 音频设备的驱动核心代码和常用设备驱动。
- usr：实现用于打包和压缩的 cpio 等。

内核一般要做到 drivers 与 arch 的软件架构分离，驱动中不包含板级信息，让驱动跨平台。同时内核的通用部分（如 kernel、fs、ipc、net 等）则与具体的硬件（arch 和 drivers）剥离。

### 内核的组成部分

Linux 内核主要由**进程调度（SCHED）、内存管理（MM）、虚拟文件系统（VFS）、网络接口（NET）和进程间通信（IPC）**5 个子系统组成，如图所示：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241024101324120.png" alt="image-20241024101324120" style="zoom:70%;" />

#### 进程调度

多个进程在 CPU 中“微观串行、宏观并行”的执行。进程调度处于系统的中心位置，内核中其他的子系统都依赖它，因为每个子系统都需要挂起或恢复进程。

Linux 的进程在几个状态间进行切换。在设备驱动编程中，当请求的资源不能得到满足时，驱动一般会调度其他进程执行，并使本进程进入睡眠状态，直到它请求的资源被释放，才会被唤醒而进入就绪状态。睡眠分成可中断的睡眠和不可中断的睡眠，两者的区别在于可中断的睡眠在收到信号的时候会醒。状态转化图如图：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241024102027520.png" alt="image-20241024102027520" style="zoom:70%;" />

完全处于 TASK_UNINTERRUPTIBLE 状态的进程甚至都无法被“杀死” ，所以 Linux 2.6.26 之后的内核 也存在一种 TASK_KILLABLE 的状态，它等于 TASK_WAKEKILL|TASK_UNINTERRUPTIBLE，可以响应致命信号。

在 Linux 内核中，**使用 task_struct 结构体来描述进程**。该结构体中包含描述该进程内存资源、文件系统资源、文件资源、tty 资源、信号处理等的指针。Linux 的线程采用轻量级进程模型来实现，在用户空间通过 pthread_create() API 创建线程的时候，本质上内核只是创建了一个新的 task_struct，并将新 task_struct 的所有资源指针都指向创建它的那个 task_struct 的资源指针。

绝大多数进程（以及进程中的多个线程）是由用户空间的应用创建的，当它们存在底层资源和硬件访问的需求时，会通过系统调用进入内核空间。有时候，在内核编程中，如果需要几个并发执行的任务，可以启动内核线程，这些线程没有用户空间。启动内核线程的函数如下：

```c
// #include <linux/sched/task.h>

pid_t kernel_thread(int (*fn)(void *), void *arg, unsigned long flags);
```

#### 内存管理

内存管理的主要作用是控制多个进程安全地共享主内存区域。当 CPU 提供内存管理单元（MMU）时，Linux 内存管理对于每个进程完成从虚拟内存到物理内存的转换。

一般而言，32 位处理器的 Linux 的每个进程享有 4GB 的内存空间，0~3GB 属于用户空间，3~4GB 属于内核空间，内核空间对常规内存、I/O 设备内存以及高端内存有不同的处理方式。当然，内核空间和用户空间的具体界限是可以调整的，在内核配置选项 Kernel Features → Memory split 下，可以设 置界限为 2GB 或者 3GB。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241024104326004.png" alt="image-20241024104326004" style="zoom:75%;" />

Linux 内核的内存管理总体比较庞大，包含底层的 Buddy 算法，它用于管理每个页的占用情况，内核空间的 slab 以及用户空间的 C 库的二次管理。另外，内核也提供了页缓存的支持，用内存来缓存磁盘，per-BDI flusher 线程用于刷回脏的页缓存到磁盘。Kswapd（交换进程）则是 Linux 中用于页面回收（包括 file-backed 的页和匿名页）的内核线程，它采用最近最少使用（LRU）算法进行内存回收。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241024104439338.png" alt="image-20241024104439338" style="zoom:65%;" />

#### 虚拟文件系统（VFS）

Linux 虚拟文件系统隐藏了各种硬件的具体细节，为所有设备提供了统一的接口。而且，它独立于各个具体的文件系统，是对各种文件系统的一个抽象。它为上层的应用程序提供了统一的 vfs_read()、vfs_write() 等接口，并调用具体底层文件系统或者设备驱动中实现的 file_operations 结构体的成员函数。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241024104730858.png" alt="image-20241024104730858" style="zoom:70%;" />

#### 网络接口

网络接口提供了对各种网络标准的存取和各种网络硬件的支持。在 Linux 中网络接口可分为网络协议和网络驱动程序，网络协议部分负责实现每一种可能的网络传输协议，网络设备驱动程序负责与硬件设备通信，每一种可能的硬件设备都有相应的设备驱动程序。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241024110756075.png" alt="image-20241024110756075" style="zoom:63%;" />

Linux 内核支持的协议栈种类较多，如 Internet、UNIX、CAN、NFC、Bluetooth、WiMAX、IrDA 等，上层的应用程序统一使用套接字接口。

#### 进程间通信

进程间通信支持进程之间的通信，Linux 支持进程间的多种通信机制，包含信号量、共享内存、消息队列、管道、UNIX 域套接字等，这些机制可协助多个进程、多资源的互斥访问、进程间的同步和消息传递。在实际的 Linux 应用中，人们更多地趋向于使用 UNIX 域套接字，而不是 System V IPC 中的消息队列等机制。

#### 组成部分的依赖关系

Linux 内核 5 个组成部分之间的依赖关系如下：

- 进程调度与内存管理之间的关系：这两个子系统互相依赖。在多程序环境下，程序要运行，则必须为之创建进程，而创建进程的第一件事情，就是将程序和数据装入内存。
- 进程间通信与内存管理的关系：进程间通信子系统要依赖内存管理支持共享内存通信机制，这种机制允许两个进程除了拥有自己的私有空间之外，还可以存取共同的内存区域。
- 虚拟文件系统与网络接口之间的关系：虚拟文件系统利用网络接口支持网络文件系统（NFS），也利用内存管理支持 RAMDISK 设备。
- 内存管理与虚拟文件系统之间的关系：内存管理利用虚拟文件系统支持交换，交换进程定期由调度程序调度，这也是内存管理依赖于进程调度的原因。当一个进程存取的内存映射被换出时，内存管理向虚拟文件系统发出请求，同时，挂起当前正在运行的进程。

除了这些依赖关系外，内核中的所有子系统还要依赖于一些共同的资源。这些资源包括所有子系统都用到的API，如分配和释放内存空间的函数、输出警告或错误消息的函数及系统提供的调试接口等。

### 内核空间与用户空间

在 Linux 系统中，内核可进行任何操作，而应用程序则被禁止对硬件的直接访问和对内存的未授权访问。

内核空间和用户空间这两个名词用来区分程序执行的两种不同状态，它们使用不同的地址空间。Linux 只能通过**系统调用**和**硬件中断**完成从用户空间到内核空间的控制转移。

## 内核的编译及加载

### 编译

既然要学习内核，首先肯定需要动手编译一个内核出来。在编译内核之前，首先下载想要编译安装的内核源代码到本地。然后就需要配置内核，可使用以下命令配置：

```bash
make config #（基于文本的最为传统的配置界面，不推荐使用）
make menuconfig #（基于文本菜单的配置界面）
make xconfig #（要求 QT 被安装）
make gconfig #（要求 GTK+ 安装）
```

Linux 内核的配置系统有三个部分组成：

- Makefile：分布在 Linux 内核源代码中，定义 Linux 内核的编译规则。
- 配置文件（Kconfig）：给用户提供配置选择的功能。
- 配置工具：包括配置命令解释器（对配置脚本中使用的配置命令进行解释）和配置用户界面（提供字符界面和图形界面）。这些配置工具使用的都是脚本语言，如用 Tcl/TK、Perl 等。

执行完配置命令以后，会生成一个 .config 配置文件，记录了哪些部分被编译入内核、哪些部分被编译为内核模块。当然一般情况下，我们可以沿用本机的设置，将其拷贝到想要想要安装的内核源代码目录，并重命名为 .config 文件：

```bash
cp -v /boot/config-`uname -r` .config
```

更完整的编译和调试内核的办法请参考文章 [https://ignotusjee.github.io/2024/10/18/Linux-Debug/](https://ignotusjee.github.io/2024/10/18/Linux-Debug/)。

另外的，在 Linux 内核中增加程序需要完成以下 3 项工作：

- 将编写的源代码复制到 Linux 内核源代码的相应目录中。
- 在目录的 Kconfig 文件中增加关于新源代码对应项目的编译配置选项。
- 在目录的 Makefile 文件中增加对新源代码的编译条目。

### Makefile

这里主要涉及内核专有的 Kbuild Makefile 编译系统。

1. 目标定义

定义哪些内容是编译并链接入内核，哪些是作为内核模块编译。

例如这段代码：

```makefile
obj-y += foo.o
```

表示需要编译 foo.c 或者 foo.s 文件得到 foo.o 并链接进内核。这是无条件编译，所以不需要 Kconfig 配置选项。

`obj-m` 表示作为内核模块编译，`obj-n` 表示不会被编译。

2. 多模块文件的定义

如果一个模块由多个文件组成，Makefile 会稍微复杂一点。这时应该采用模块名加 -y 或 -objs 后缀的形式定义模块的组成文件，例如：

```makefile
#
# Makefile for the linux ext2-filesystem routines.
#
obj-$(CONfiG_EXT2_FS) += ext2.o
ext2-y := balloc.o dir.o file.o fsync.o ialloc.o inode.o \
ioctl.o namei.o super.o symlink.o
ext2-$(CONfiG_EXT2_FS_XATTR) += xattr.o xattr_user.o xattr_trusted.o
ext2-$(CONfiG_EXT2_FS_POSIX_ACL) += acl.o
ext2-$(CONfiG_EXT2_FS_SECURITY) += xattr_security.o
ext2-$(CONfiG_EXT2_FS_XIP) += xip.o
```

模块的名字是 ext2，由 balloc.o dir.o file.o 等多个目标文件最终链接生成 ext2.o 或者 ext2.ko 的目标文件（当然 Linux 2.6 以后是 .ko）。其他的目标文件是否编译生成取决于配置文件。

3. 目录层次的迭代

当源代码比较多的时候，处于设计和美观的目的，按照目录分层次结构是有必要的。例如：

```makefile
obj-m += ext2/
```

这代表 Kbuild 会把 ext2/ 目录列入向下迭代的目标。ext2/ 目录中理应有自己的 Kbuild Makefile 的子构建系统。

### Kconfig

1. 配置选项

配置选项通过 config 关键字定义，例如：

```kconfig
config MODVERSIONS
    bool "Module versioning support"
    help
        Usually, you have to use modules compiled with your kernel.
        Saying Y here makes it ...
```

config 关键字定义新的配置选项，之后的几行代码定义了该配置选项的属性。配置选项的属性包括类型、数据范围、输入提示、依赖关系、选择关系及帮助信息、默认值等。

每个配置选项都必须指定类型，类型包括 bool、tristate、string、hex 和 int，其中 tristate 和 string 是两种基本类型，其他类型都基于这两种基本类型。类型定义后可以紧跟输入提示，下面两段代码是等价的：

```kconfig
bool “Networking support”

# 等价于

bool
prompt "Networking support"
```

输入提示使用 prompt 关键字，一般格式如下，其中可选 if 用于表示该提示的依赖关系。

```kconfig
prompt <prompt> [if <expr>]
```

默认值的格式如下。如果用户不设置对应的选项，配置选项的值就是默认值。

```kconfig
default <expr> [if <expr>]
```

依赖关系的格式如下。如果定义了多重依赖关系，它们之间用 `&&` 间隔。

```kconfig
depends on <expr>
```

依赖关系也可以应用到该菜单中所有的其他选项 （同样接受 if 表达式）内，因此下面两段脚本是等价的：

```kconfig
bool "foo" if BAR
default y if BAR


# 等价于

depends on BAR
bool "foo"
default y
```

选择关系，也成为反向依赖关系，格式如下。如果 A 选择了 B，那么 A 在被选中的情况下，B 也会自动被选中。

```kconfig
select <symbol> [if <expr>]
```

数据范围的格式为：

```kconfig
range <symbol> <symbol> [if <expr>]
```

expr 表达式定义为：

```kconfig
<expr> ::= <symbol>
            <symbol> '=' <symbol>
            <symbol> '!=' <symbol>
            '(' <expr> ')'
            '!' <expr>
            <expr> '&&' <expr>
            <expr> '||' <expr>
```

也就是说，expr 是由 symbol、两个 symbol 相等、两个 symbol 不等以及 expr 的赋值、非、与或运算构成。symbol 分为两类，一类是由菜单入口配置选项定义的非常数 symbol，另一类是作为 expr 组成部分的常数 symbol。

举个例子就明白了，如下 expr 表示依赖条件是 ARCH_R8A73A4 被选中以及 SH_DMAE 未被选中，才能出现 SHDMA_R8A73A4。

```kconfig
config SHDMA_R8A73A4
    bool y
    depends on ARCH_R8A73A4 && SH_DMAE == n
```

帮助信息的格式为，完全靠文本缩进识别结束。

```kconfig
help
# 或者：---help---
    开始
    …
    结束
```

2. 菜单结构

配置选项在菜单树结构的位置可以由两种方法决定。

第一种如下，所有处于 menu 和 endmenu 之间的配置选项都会成为 Network device support 的子菜单，而且，所有子菜单（config）选项都会继承父菜单（menu）的依赖关系。菜单 Network device support 对 NET 的依赖会加到配置选项 NETDEVICES 的依赖列表中。

```kconfig
menu "Network device support"
    depends on NET
config NETDEVICES

	...

endmenu
```

> 注：menu 后面跟的 Network device support 项仅仅是 1 个菜单，没有对应真实的配置选项，也不具备 3 种不同的状态。这是 menu 和 config 的区别。

另一种方式是通过分析依赖关系生成菜单结构。如果菜单项在一定程度上依赖于前面的选项，它就能成为该选项的子菜单。例如这里，config MODVERSIONS 直接依赖于 MODULES，只有当 MODULES 不为 n 的时候，MODVERSIONS 才可见。

```kconfig
config MODULES
    bool "Enable loadable module support"
config MODVERSIONS
    bool "Set version information on all module symbols"
    depends on MODULES
```

更详细的编写细节，请参考内核文档 Documentation 目录内的 kbuild 子目录下的 Kconfig-language.rst 和 Makefiles.rst 文件。

# 内核模块

## 内核模块简介

如果我们自己想要在 Linux 内核当中添加功能，有两种方法。

1. 一是把所有需要的功能都编译到 Linux 内核中。但这样会导致生成的内核很大，并且如果需要增加或删除功能，将不得不重新编译内核。
2. 二是让内核提供某种方法，使得原本的内核并不需要包含所有的功能，而是在需要使用的时候，将代码动态的加载到内核当中。

巧了，Linux 内核就提供了第二种方法，这样的机制成为模块。模块具有如下特点：

1. 模块本身不被编译入内核映像，从而控制了内核的大小。
2. 模块一旦被加载，它就和内核中的其他部分完全一样。

以一个最简单的 Hello World 模块为例，展示一下内核模块编程的大致认识：

本程序通过内核的 Makefile + Kbuild 系统编译以后会生成 hello.ko 目标文件，通过 insmod 命令加载到内核中，rmmod 命令卸载。

```c
#include <linux/init.h>
#include <linux/module.h>


MODULE_VERSION("v1.0.0");
MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("DavidingPlus");
MODULE_DESCRIPTION("A Simple Hello World Module");


static int __init hello_init(void)
{
    // 内核模块中用于输出的函数是内核空间的 printk() 而不是用户空间的 printf()，printk() 的用法和 printf() 基本相似，但前者可定义输出级别。
    printk(KERN_INFO "hello: Hello World\n");


    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "hello: Goodbye World\n");
}


module_init(hello_init);
module_exit(hello_exit);
```

在 Linux 中，可通过 lsmod 命令查看系统中已加载的所有模块以及之间的依赖关系：

```bash
sudo lsmod

# Module                  Size  Used by
# ufs                    81920  0
# qnx4                   16384  0
# hfsplus               110592  0
# hfs                    61440  0
# minix                  40960  0
# ntfs                  106496  0
# msdos                  20480  0
```

lsmod 实际上是读取并分析 `/proc/modules` 文件，对应结果如下：

```bash
cat /proc/modules

# ufs 81920 0 - Live 0x0000000000000000
# qnx4 16384 0 - Live 0x0000000000000000
# hfsplus 110592 0 - Live 0x0000000000000000
# hfs 61440 0 - Live 0x0000000000000000
# minix 40960 0 - Live 0x0000000000000000
# ntfs 106496 0 - Live 0x0000000000000000
# msdos 20480 0 - Live 0x0000000000000000
```

内核中已加载模块的信息也保存在 `/sys/modules/` 目录下。加载上面的 hello.ko 以后，内核中将包含 /sys/modules/hello/ 目录，该目录的结构图如下：

```markdown
.
├── coresize
├── holders
├── initsize
├── initstate
├── notes
│   ├── .note.gnu.build-id
│   └── .note.Linux
├── refcnt
├── sections
│   ├── .exit.data
│   ├── .exit.text
│   ├── .gnu.linkonce.this_module
│   ├── .init.data
│   ├── .init.text
│   ├── .note.gnu.build-id
│   ├── .note.Linux
│   ├── .rodata.str1.1
│   ├── .strtab
│   ├── .symtab
│   └── .text
├── srcversion
├── taint
├── uevent
└── version

3 directories, 21 files
```

除了 insmod 命令，还有一个 modprobe 命令。与 insmod 不同的是，**modprobe 在加载某模块时，会同时加载该模块所依赖的其他模块**。使用 modprobe 命令加载的模块若以 `modprobe -r filename` 的方式卸载，将同时卸载其依赖的模块。

模块之间的依赖关系存放在根文件系统的 `/lib/modules/<kernel-version>/modules.dep` 文件中，实际上是在整体编译内核的时候由 depmod 工具生成的，它的格式非常简单：

```markdown
kernel/arch/x86/events/intel/intel-cstate.ko:
kernel/arch/x86/events/rapl.ko:
kernel/arch/x86/kernel/cpu/mce/mce-inject.ko:
kernel/arch/x86/kernel/msr.ko:
kernel/arch/x86/kernel/cpuid.ko:
kernel/arch/x86/crypto/twofish-x86_64.ko: kernel/crypto/twofish_common.ko
kernel/arch/x86/crypto/twofish-x86_64-3way.ko: kernel/arch/x86/crypto/twofish-x86_64.ko kernel/crypto/twofish_common.ko
kernel/arch/x86/crypto/twofish-avx-x86_64.ko: kernel/crypto/crypto_simd.ko kernel/crypto/cryptd.ko kernel/arch/x86/crypto/twofish-x86_64-3way.ko kernel/arch/x86/crypto/twofish-x86_64.ko kernel/crypto/twofish_common.ko
...
```

使用 modinfo 命令可以获得模块的信息，包括模块作者、模块的说明、模块所支持的参数以及 vermagic：

```bash
modinfo hello.ko

# filename:       /home/lzx0626/DavidingPlus/linux-kernel-learning/build/linux/x86_64/debug/hello.ko
# description:    A Simple Hello World Module
# author:         DavidingPlus
# license:        Dual BSD/GPL
# version:        1.0.0
# srcversion:     533BB7E5866E52F63B9ACCB
# depends:
# retpoline:      Y
# name:           hello
# vermagic:       5.15.167 SMP mod_unload modversions
```

## 内核模块程序结构

一个 Linux 内核模块主要由以下几部分组成：

1. 模块加载函数

当通过 insmod 或 modprobe 命令加载内核模块时，模块的加载函数会自动被内核执行，完成本模块的相关初始化工作。

2. 模块卸载函数

当通过 rmmod 命令卸载某模块时，模块的卸载函数会自动被内核执行，完成与模块卸载函数相反的功能。

3. 模块许可证声明

许可证（LICENSE）声明描述内核模块的许可权限，如果不声明，模块被加载时，将收到内核被污染（Kernel Tainted）的警告。

4. 模块参数（可选）

模块参数是模块被加载的时候可以传递给它的值，它本身对应模块内部的全局变量。

5. 模块导出符号（可选）

内核模块可以导出的符号（symbol，对应于函数或变量）。若导出，其他模块则可以使用本模块中的变量或函数。

6. 模块作者等信息声明（可选）

## 模块加载函数

Linux 内核模块加载函数一般以 __init 标识声明，例如：

```c
static int __init initialization_function(void)
{
	...
}


module_init(initialization_function);
```

模块加载函数 initialization_function(void) 在内核被加载的时候，通过宏 module_init 进行调用。加载函数应该有一个返回值 int，成功返回 0，失败返回一个错误编码。该错误编码是一个接近 0 的负数，在 `<linux/errno.h>` 中定义，如 -ENODEV、-ENOMEM 等。强烈建议返回有意义的错误编码宏，因为这样用户可以通过 perror() 的方法将其转化为有意义的字符串。

在 Linux 内核程序代码中，可以通过 request_module(const char*fmt, …) 函数加载内核模块，例如：

```c
request_module(module_name);
```

在 Linux 内核中，所有标识为 __init 的函数如果直接编译进入内核，那么会成为内核镜像的一部分，连接的时候会放在 .init.text 区段内。

```c
#define __init		__section(".init.text") __cold  __latent_entropy __noinitretpoline __nocfi
```

所有的 `__init` 函数在区段 .initcall.init 中还保存了一份函数指针，在初始化时内核会通过这些函数指针调用这些 `__init` 函数，并在初始化完成后，释放 `__init` 区段（包括 .init.text、.initcall.init 等）的内存。

除了函数以外，数据也可以被定义为 __initdata。表示**只是初始化阶段需要的数据，初始化完成以后，内核会自动释放他们占用的内存。**

## 模块卸载函数

Linux 内核卸载加载函数一般以 __exit 标识声明，例如：

```c
static void __exit cleanup_function(void)
{
    ...
}


module_exit(cleanup_function);
```

模块卸载函数在模块卸载的时候执行，不返回任何值。通过宏 module_exit 的形式指定调用。通常来讲会完成模块加载函数相反的功能。

与模块加载函数对应的，可以使用 `__exit` 修饰模块卸载函数。如果告诉内核相关模块直接被编译进内核。这种情况下 `__exit` 修饰的模块卸载函数会被忽略，不会被链进最后的镜像。因为既然都编译进内核，被内置了，那么肯定不会被卸载了。

另外对应的，数据也可以被定义为 __exitdata。

## 模块参数

可以使用宏 `module_param(参数名，参数类型，参数读/写权限)` 为模块指定一个参数。宏函数参数分别对应模块参数名、参数类型以及参数读/写权限。

在装载内核模块的时候，可以向模块提供参数，格式如下。多个参数传递用空格隔开。如果不传递参数，将使用模块内定义的默认值。如果模块被内置，则无法被 insmod ，但是 bootloader 可以通过在 bootargs 里设置`“模块名.参数名=值”`的形式给该内置模块传递参数。

```bash
sudo insmod <param_name>=<param_value>
```

参数类型可以是 byte、short、ushort、int、uint、long、ulong、charp（字符指针）、bool 或 invbool（bool 的反）。模块在编译的时候会检查传入变量的类型和传入的类型是否一致。

另外，模块也可以拥有参数数组，通过宏 `module_param_array(数组名，数组类型，数组长，参 数读/写权限)` 指定，用法类似。

> 传递数组参数的时候需要注意一下格式。参数之间用 `,` 隔开，并且不加大括号，参数之间没有空格，例如传递一个 int 数组：
>
> ```bash
> sudo insmod hello.ko array=1,2,3,4
> ```

如果指定了模块参数，**并且参数读/写权限不为 0**，在装入模块以后会在 `/sys/modules/<module_name>` 下出现 parameters 子目录，用于记录内核模块的参数。

```markdown
.
...
├── parameters
│   ├── helloExitParam
│   └── helloInitParam
...
```

例如这里，就记录了内核模块参数 helloInitParam 和 helloExitParam，使用 cat 命令能够得到他们分别记录的值，其中数组的表示方法和上面提到的一致。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241105100931281.png" alt="image-20241105100931281" style="zoom:75%;" />

实例程序如下：

```c
#include <linux/init.h>
#include <linux/module.h>


MODULE_VERSION("1.0.0");
MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("DavidingPlus");
MODULE_DESCRIPTION("A Simple Hello World Module");


static int helloInitData __initdata = -114514;

static const char *helloExitData __exitdata = "foo";

static char *helloInitParam = "gee";
module_param(helloInitParam, charp, S_IRUGO);

static int helloExitParam[2] = {-10086, 10086};
int helloExitParamSize = sizeof(helloExitParam) / sizeof(int);
module_param_array(helloExitParam, int, &helloExitParamSize, S_IRUGO);


static int __init hello_init(void)
{
    printk(KERN_INFO "hello: Hello World %d %s\n", helloInitData, helloInitParam);


    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "hello: Goodbye World %s %d %d\n", helloExitData, helloExitParam[0], helloExitParam[1]);
}


module_init(hello_init);
module_exit(hello_exit);
```

## 导出符号

导出符号即内核可以将函数导出被其他模块使用。Linux 中 `/proc/kallsyms` 文件对应内核符号表，记录了符号以及符号所在的内存地址。

使用如下宏函数导出符号到内核符号表中：

```c
EXPORT_SYMBOL(符号名);

// EXPORT_SYMBOL_GPL() 只适用于包含 GPL 许可权的模块。
EXPORT_SYMBOL_GPL(符号名);
```

例如这里导出函数 add() 和 sub()：

```c
...

int add(int a, int b) { return a + b; }

int sub(int a, int b) { return a - b; }


EXPORT_SYMBOL(add);
EXPORT_SYMBOL(sub);

...
```

装载好内核模块以后查看 /proc/kallsyms 文件能得到对应的符号信息：

```bash
cat /proc/kallsyms | grep hello

# ...
# 0000000000000000 r __kstrtab_add        [hello]
# 0000000000000000 r __kstrtabns_add      [hello]
# 0000000000000000 r __ksymtab_add        [hello]
# 0000000000000000 r __kstrtab_sub        [hello]
# 0000000000000000 r __kstrtabns_sub      [hello]
# 0000000000000000 r __ksymtab_sub        [hello]
# 0000000000000000 T add  [hello]
# 0000000000000000 T sub  [hello]
```

## 模块声明与描述

在 Linux 内核模块中，我们可以用 MODULE_AUTHOR、MODULE_DESCRIPTION、MODULE_VERSION、MODULE_DEVICE_TABLE、MODULE_ALIAS 分别声明模块的作者、描述、版本、设备表和别名，例如：

```c
MODULE_AUTHOR(author);
MODULE_DESCRIPTION(description);
MODULE_VERSION(version_string);
MODULE_DEVICE_TABLE(table_info);
MODULE_ALIAS(alternate_name);
```

## 模块的使用计数

Linux 2.4 内核中，模块自身通过 MOD_INC_USE_COUNT、MOD_DEC_USE_COUNT 宏来管理自己被使用的计数。

Linux 2.6 以后的内核提供了模块计数管理接口 try_module_get(&module) 和 module_put(&module)，从而取代 Linux 2.4 内核中的模块使用计数管理宏。模块的使用计数一般不必由模块自身管理，而且模块计数管理还考虑了 SMP 与 PREEMPT 机制的影响。

```c
// 用于增加模块使用计数。若返回为0，表示调用失败，希望使用的模块没有被加载或正在被卸载中。
int try_module_get(struct module *module);

// 用于减少模块使用计数。
void module_put(struct module *module);
```

Linux 2.6 以后的内核为不同类型的设备定义了 struct module *owner 域，用来指向管理此设备的模块。当开始使用某个设备时，内核使用 try_module_get(dev->owner) 去增加管理此设备的 owner 模块的使用计数；当不再使用此设备时，内核使用 module_put(dev->owner) 减少对管理此设备的管理模块的使用计数。这样，当设备在使用时，管理此设备的模块将不能被卸载。只有当设备不再被使用时，模块才允许被卸载。

# 文件系统与设备文件

Linux 之下一些皆文件，并且由于字符设备和块设备都良好的体现了一切皆文件的思想，因此 Linux 文件系统与设备文件的基础知识就非常重要了。

## 文件操作

### 系统调用

1. 创建

```c
int creat(const char *pathname, mode_t mode);
```

参数 mode 代表创建文件的权限，和 umask 一起共同决定文件的最终权限（`mode & ~umask`）。umask 代表文件创建的时候需要去掉的一些权限，让最终的权限合理一些。可通过 umake() 系统调用改变：

```c
mode_t umask(mode_t mask);
```

umask() 函数将 umask 设置为 newmask，然后返回旧的 umask，它只影响读、写和执行权限。

2. 打开

```c
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

flags 参数表示文件的打开标志，可以是如下的一个或者几个的组合：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241105144109024.png" alt="image-20241105144109024" style="zoom:80%;" />

其中，O_RDONLY、O_WRONLY、O_RDWR 三者是互斥的，只可选择其一。

如果使用了 O_CREAT 标志，还需要提供第三个参数 mode 来表示新文件的权限，同前面的 creat() 函数。mode 权限如下图：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241105144550547.png" alt="image-20241105144550547" style="zoom:70%;" />

关于权限 mode，举个例子就明白了。Linux 下的文件权限是一个八进制数，例如 0777。3 个 7 分别表示对不同的用户（所有者，组成员，其他用户的权限）的权限。每一个都是 3 位 ，第一位表示读 R，第二位表示写 W，第三位表示可执行 X，7 对应就是 111 全有。

最后，open() 函数返回一个进程唯一的文件描述符，对文件的所有操作都通过该文件描述符实现。

3. 读写

打开文件获得文件描述符以后，通过 read() 和 write() 函数对文件进行读写。

```c
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

参数 buf 为存放读取数据的内存的地址指针，count 为长度。

函数 read() 从文件描述符 fd 所指定的文件中读取 count 个字节到 buf 所指向的内存中，返回值为实际读取的字节数。

函数 write() 实现把 count 个字节从 buf 中写到文件描述符 fd 所指向的文件中，返回值为实际写入的字节数。

4. 定位

对于随机文件，可以指定随机位置进行读写，使用 lseek() 函数：

```c
off_t lseek(int fd, off_t offset, int whence);
```

lseek() 函数将文件指针相对 whence 位置移动 offset 个字节，操作成功后返回文件指针现在的位置。

参数 whence 可使用如下值。

- SEEK_SET：相对文件开头。
- SEEK_CUR：相对文件读写指针的当前位置。
- SEEK_END：相对文件末尾。

当然 lseek() 的 offset 可以取负值，表示往前移动。一般可以通过此函数获得文件的大小：

```c
int size = lseek(fd, 0, SEEK_END);
```

5. 关闭

有始有终，在结束对文件的操作以后，需要关闭文件，释放文件描述符。通过 close() 实现：

```c
int close(int fd);
```

6. 实例 Demo

编写一个程序，在当前目录下创建用户可读写文件 hello.txt，在其中写入"He\nllo, software weekly."，关闭该文件。再次打开该文件，读取其中的内容并输出在屏幕上。最后删除该文件。

比较简单，cpp 程序如下：

```cpp
// 使用系统调用。
#include <iostream>
#include <cstring>

#include <unistd.h>
#include <fcntl.h>


// 故意搞一个不能一次读完的数组。
#define MAX_SIZE 10


int main()
{
    const char *filePath = "hello.txt";
    const char *writeStr = "He\nllo, software weekly.";


    int fd = open(filePath, O_RDWR | O_CREAT | O_TRUNC, 0755);
    if (-1 == fd)
    {
        perror("open");


        return -1;
    }

    write(fd, writeStr, strlen(writeStr));

    close(fd);


    int fd2 = open(filePath, O_RDONLY);
    if (-1 == fd2)
    {
        perror("open");


        return -1;
    }


    char readStr[MAX_SIZE] = {0};
    ssize_t len = 0;

    // 为保证读完，需循环读取。
    while (true)
    {
        bzero(readStr, sizeof(readStr));

        len = read(fd2, readStr, sizeof(readStr) - 1);
        if (-1 == len)
        {
            perror("read");


            return -1;
        }

        std::cout << readStr;

        if (0 == len) break;
    }

    std::cout << std::endl;

    close(fd2);


    unlink(filePath);


    return 0;
}
```

正确的输出预期应该如图：

![image-20241105160211266](https://image.davidingplus.cn/images/2025/02/01/image-20241105160211266.png)

### 标准 C 库函数

1. 创建和打开

这两个操作都使用 fopen() 函数：

```c
FILE *fopen(const char *pathname, const char *mode);
```

mode 表示打开模式，支持的模式如图所示：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20230907152718910.png" alt="image-20230907152718910" style="zoom: 80%;" />

其中，b 用于区分二进制文件和文本文件，这一点在 DOS、Windows 系统中是有区分的，但 Linux 不区分二进制文件和文本文件。

2. 读写

标准 C 库的 IO 函数会自动在用户层分配缓冲区，而系统调用不具有用户层缓冲区。二者都具有内核层的缓冲区。其中缓冲分为三种，**全缓冲、行缓冲和无缓冲**。

罗列部分 API 接口如下：

```cpp
int fgetc(FILE *stream);
int fputc(int c, FILE *stream);
char *fgets(char *s, int size, FILE *stream);
int fputs(const char *s, FILE *stream);
int fprintf(FILE *stream, const char *format, ...);
int fscanf(FILE *stream, const char *format, ...);
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```

另外，标准 C 库函数还提供了读写过程的定位能力，包括：

```cpp
int fgetpos(FILE *stream, fpos_t *pos);
int fsetpos(FILE *stream, const fpos_t *pos);
int fseek(FILE *stream, long offset, int whence);
```

更多的具体用法和细节请参考另一篇博客 [https://blog.davidingplus.cn/posts/9e44c3b9.html](https://blog.davidingplus.cn/posts/9e44c3b9.html) 的第五章。

3. 关闭

同上，是非常简单的操作。

```cpp
int fclose(FILE *stream);
```

4. 实例 Demo

将前面的实例 Demo 换成标准 C 库的实现方法如下：

```cpp
// 使用标准 C 库函数。
#include <iostream>
#include <cstring>


#define MAX_SIZE 10


int main()
{
    const char *filePath = "hello.txt";
    const char *writeStr = "He\nllo, software weekly.";

    FILE *file = fopen(filePath, "w");
    if (!file)
    {
        perror("fopen");


        return -1;
    }

    fputs(writeStr, file);

    fclose(file);


    FILE *file2 = fopen(filePath, "r");
    if (!file2)
    {
        perror("fopen");


        return -1;
    }

    char readStr[MAX_SIZE] = {0};

    while (true)
    {
        bzero(readStr, sizeof(readStr));

        fgets(readStr, sizeof(readStr) - 1, file2);
        if (ferror(file2))
        {
            perror("fgets");


            return -1;
        }

        std::cout << readStr;

        if (feof(file2)) break;
    }

    std::cout << std::endl;

    fclose(file2);


    remove(filePath);


    return 0;
}
```

同样输出和上面一样的结果。

## 文件系统

### 目录结构

进入 Linux 根目录（即 `/`，Linux 文件系统的入口，也是处于最高一级的目录），运行 `ls -la` 命令，包含以下目录：

1. /bin

包含基本命令，如 ls、cp、mkdir 等，这个目录中的文件都是可执行的。

2. /sbin

包含系统命令，如 modprobe、hwclock、ifconfig 等，大多是涉及系统管理的命令，这个目录中的文件都是可执行的。

3. /dev

设备文件存储目录，应用程序通过对这些文件的读写和控制以访问实际的设备。

4. /etc

系统配置文件的所在地，一些服务器的配置文件也在这里，如用户账号及密码配置文件。busybox 的启动脚本也存放在该目录。

5. /lib

系统库文件存放目录等。

6. /mnt

这个目录一般是用于存放挂载储存设备的挂载目录，比如含有 cdrom 等目录。可以参看 /etc/fstab 的定义。有时我们可以让系统开机自动挂载文件系统，并把挂载点放在这里。

7. /opt

opt 是可选的意思，有些软件包会被安装在这里。

8. /proc

操作系统运行时，进程及内核信息（比如 CPU、硬盘分区、内存信息等）存放在这里。/proc 目录为伪文件系统 proc 的挂载目录，proc 并不是真正的文件系统，它存在于内存之中。

9. /tmp

用户运行程序的时候，有时会产生临时文件，/tmp 用来存放临时文件。

10. /usr

这个是系统存放程序的目录，比如用户命令、用户库等。

11. /var

var 表示的是变化的意思，这个目录的内容经常变动，如 /var/log/ 目录被用来存放系统日志。

12. /sys

Linux 2.6 以后的内核所支持的 sysfs 文件系统被映射在此目录上。Linux 设备驱动模型中的总线、驱动 和设备都可以在 sysfs 文件系统中找到对应的节点。当内核检测到在系统中出现了新设备后，内核会在 sysfs 文件系统中为该新设备生成一项新的记录。

### 文件系统与设备驱动

如图所示是 Linux 下虚拟文件系统、磁盘/Flash 文件系统及一般的设备文件与设备驱动程序之间的关系。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241106093412285.png" alt="image-20241106093412285" style="zoom: 70%;" />

应用程序和 VFS 之间的接口是系统调用，而 VFS 与文件系统以及设备文件之间的接口是 file_operations 结构体成员函数，这个结构体包含对文件进行打开、关闭、读写、控制的一系列成员函数。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241106093650313.png" alt="image-20241106093650313" style="zoom:75%;" />

字符设备的上层没有类似于磁盘 ext2 等文件系统，所以字符设备的 file_operations 成员函数就直接由设备驱动提供了，并且是字符驱动的核心部分。

块设备有两种访问方法，一种方法是不通过文件系统直接访问裸设备，在 Linux 内核实现了统一的 def_blk_fops 这一 file_operations，它的源代码位于 fs/block_dev.c，所以当我们运行类似于 `dd if=/dev/sdb1of=sdb1.img` 的命令把整个 /dev/sdb1 裸分区复制到 sdb1.img 的时候，内核走的是 def_blk_fops 这个 file_operations；另外一种方法是通过文件系统来访问块设备，file_operations 的实现则位于文件系统内，文件系统会把针对文件的读写转换为针对块设备原始扇区的读写。ext2、fat、Btrfs 等文件系统中会实现针对 VFS 的 file_operations 成员函数，设备驱动层将看不到 file_operations 的存在。

#### file 结构体

file 结构体代表一个打开的文件。系统中每个打开的文件在内核空间都会对应一个 struct file。它由内核在打开文件的时候创建，并传递给文件上进行操作的任何函数，在文件的所有实例关闭以后，内核会释放这个数据结构。一般将 struct file 的指针命名为 file 或者 filp。

在内核 5.15 版本中如下定义：

```c
struct file {
	union {
		struct llist_node	fu_llist;
		struct rcu_head 	fu_rcuhead;
	} f_u;
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op; // 和文件关联的操作

	/*
	 * Protects f_ep, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	enum rw_hint		f_write_hint;
	atomic_long_t		f_count;
	unsigned int 		f_flags; // 文件标志，如 O_RDONLY、O_NONBLOCK、O_SYNC 等
	fmode_t			f_mode; // 文件读/写模式，如 FMODE_READ、FMODE_WRITE 等
	struct mutex		f_pos_lock;
	loff_t			f_pos; // 当前读写位置
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data; // 文件私有数据

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct hlist_head	*f_ep;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
	errseq_t		f_wb_err;
	errseq_t		f_sb_err; /* for syncfs */
} __randomize_layout
  __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */
```

文件读/写模式 f_mode、标志 f_flags 都是设备驱动关心的内容。私有数据指针 private_data 在设备驱动中被广泛应用，大多被指向设备驱动自定义以用于描述设备的结构体。

#### inode 结构体

inode 结构体包含文件访问权限、属主、组、大小、生成时间、访问时间、最后修改时间等信息。它是 Linux 管理文件系统的最基本单位，也是文件系统连接任何子目录、文件的桥梁。定义如下：

```c
struct inode {
	umode_t			i_mode; // inode 的权限
	unsigned short		i_opflags;
	kuid_t			i_uid; // inode 所有者 id
	kgid_t			i_gid; // inode 所属的群组 id
	unsigned int		i_flags;

#ifdef CONFIG_FS_POSIX_ACL
	struct posix_acl	*i_acl;
	struct posix_acl	*i_default_acl;
#endif

	const struct inode_operations	*i_op;
	struct super_block	*i_sb;
	struct address_space	*i_mapping;

#ifdef CONFIG_SECURITY
	void			*i_security;
#endif

	/* Stat data, not accessed from path walking */
	unsigned long		i_ino;
	/*
	 * Filesystems may only read i_nlink directly.  They shall use the
	 * following functions for modification:
	 *
	 *    (set|clear|inc|drop)_nlink
	 *    inode_(inc|dec)_link_count
	 */
	union {
		const unsigned int i_nlink;
		unsigned int __i_nlink;
	};
	dev_t			i_rdev; // 若是设备文件，此字段会记录设备的设备号
	loff_t			i_size; // inode 代表的文件大小
	struct timespec64	i_atime; // inode 最近一次的存取时间
	struct timespec64	i_mtime; // inode 最近一次的修改时间
	struct timespec64	i_ctime; // inode 的产生时间
	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
	unsigned short          i_bytes;
	u8			i_blkbits;
	u8			i_write_hint;
	blkcnt_t		i_blocks; // inode 使用的 block 数，一个 block 为 512 个字节

#ifdef __NEED_I_SIZE_ORDERED
	seqcount_t		i_size_seqcount;
#endif

	/* Misc */
	unsigned long		i_state;
	struct rw_semaphore	i_rwsem;

	unsigned long		dirtied_when;	/* jiffies of first dirtying */
	unsigned long		dirtied_time_when;

	struct hlist_node	i_hash;
	struct list_head	i_io_list;	/* backing dev IO list */
#ifdef CONFIG_CGROUP_WRITEBACK
	struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

	/* foreign inode detection, see wbc_detach_inode() */
	int			i_wb_frn_winner;
	u16			i_wb_frn_avg_time;
	u16			i_wb_frn_history;
#endif
	struct list_head	i_lru;		/* inode LRU list */
	struct list_head	i_sb_list;
	struct list_head	i_wb_list;	/* backing dev writeback list */
	union {
		struct hlist_head	i_dentry;
		struct rcu_head		i_rcu;
	};
	atomic64_t		i_version;
	atomic64_t		i_sequence; /* see futex */
	atomic_t		i_count;
	atomic_t		i_dio_count;
	atomic_t		i_writecount;
#if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
	atomic_t		i_readcount; /* struct files open RO */
#endif
	union {
		const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
		void (*free_inode)(struct inode *);
	};
	struct file_lock_context	*i_flctx;
	struct address_space	i_data;
	struct list_head	i_devices;
	union {
		struct pipe_inode_info	*i_pipe;
		struct cdev		*i_cdev; // 若是字符设备，对应其 cdev 结构体指针
		char			*i_link;
		unsigned		i_dir_seq;
	};

	__u32			i_generation;

#ifdef CONFIG_FSNOTIFY
	__u32			i_fsnotify_mask; /* all events this inode cares about */
	struct fsnotify_mark_connector __rcu	*i_fsnotify_marks;
#endif

#ifdef CONFIG_FS_ENCRYPTION
	struct fscrypt_info	*i_crypt_info;
#endif

#ifdef CONFIG_FS_VERITY
	struct fsverity_info	*i_verity_info;
#endif

	void			*i_private; /* fs or device private pointer */
} __randomize_layout;
```

对于表示设备文件的 inode 结构，i_rdev 字段包含设备编号。**Linux 内核设备编号分为主设备编号和次设备编号。设备号用 dev_t 类型表示，实际上是 `unsigned int`。前者为 dev_t 的高 12 位，后者为 dev_t 的低 20 位。**使用如下函数从 inode 获得主设备号和次设备号：

```c
static inline unsigned iminor(const struct inode *inode)
{
	return MINOR(inode->i_rdev);
}

static inline unsigned imajor(const struct inode *inode)
{
	return MAJOR(inode->i_rdev);
}
```

查看 /proc/devices 文件可以看到当前系统注册的设备，第一列为主设备号，第二列为设备名，例如：

```bash
cat /proc/devices

# Character devices:
#   1 mem
#   4 /dev/vc/0
#   4 tty
#   4 ttyS
#   5 /dev/tty
#   5 /dev/console
#   5 /dev/ptmx
#   5 ttyprintk
#   7 vcs
#  10 misc
#  13 input
#  21 sg
#  29 fb
#  89 i2c
# 108 ppp
# 128 ptm
# 136 pts
# 180 usb
# 189 usb_device
# 202 cpu/msr
# 204 ttyMAX
# 226 drm
# 241 aux
# 242 cec
# 243 hidraw
# 244 vfio
# 245 bsg
# 246 watchdog
# 247 ptp
# 248 pps
# 249 rtc
# 250 dax
# 251 dimmctl
# 252 ndctl
# 253 tpm
# 254 gpiochip

# Block devices:
#   7 loop
#   8 sd
#   9 md
#  11 sr
#  65 sd
#  66 sd
#  67 sd
#  68 sd
#  69 sd
#  70 sd
#  71 sd
# 128 sd
# 129 sd
# 130 sd
# 131 sd
# 132 sd
# 133 sd
# 134 sd
# 135 sd
# 253 device-mapper
# 254 mdp
# 259 blkext
```

查看 /dev 目录可以查看系统中在上述注册的设备上建立的设备文件。一个注册的设备可以有多个设备文件，日期前面的两列分别对应主设备号和次设备号。

主设备号是与驱动对应的概念。同一类设备一般使用相同的主设备号，不同类设备一般使用不同的主设备号（但是不排除在同一主设备号下包含有一定差异的设备）。**因为同一驱动可支持多个同类设备，因此用次设备号来描述使用该驱动的设备的序号**。序号一般从 0 开始。

## devfs

devfs（设备文件系统）是 Linux 内核 2.4 版本引入的，使得设备驱动程序能自主地管理自己的设备文件。具体来讲，有以下优点：

1. 可以通过程序在设备初始化的时候在 /dev 目录下创建设备文件，卸载设备的时候删除。
2. 设备驱动程序可以指定设备名、所有者和权限位，用户空间程序仍可以修改所有者和权限位。
3. 不再需要为设备驱动程序分配主设备号以及处理次设备号，在程序中可以直接给 register_chrdev() 传递 0 主设备号以获得可用的主设备号，并在 devfs_register() 中指定次设备号。

## udev

### udev 和 devfs 的区别

尽管 devfs 有很多的优点，但在 Linus 2.6 中，devfs 被认为是过时的方法，并最终被抛弃了，udev 取代了它。

Linux 设计中强调的一个基本观点是机制和策略的分离。机制是做某样事情的固定步骤、方法，而策略就是每一个步骤所采取的不同方式。机制是相对固定的，而每个步骤采用的策略是不固定、灵活的。在 Linux 内核中，应该实现机制而非策略。

例如，Linux 提供 API 可以让人把线程的优先级调高或者调低，或者调整调度策略为 SCHED_FIFO 什么的，但是 Linux 内核本身却不管谁高谁低。提供 API 属于机制，谁高谁低这属于策略，所以应该是应用程序自己去告诉内核要高或低，而内核不管这些杂事。属于策略的东西应该被移到用户空间中，谁爱给哪个设备创建什么名字或者想做更多的处理，谁自己去设定。内核只管把这些信息告诉用户就行了。这就是位于内核空间的 devfs 应该被位于用户空间的 udev 取代的原因，应该 devfs 管了一些它实际上不该管的事情。

udev 是 Linux 系统下的一套**设备管理系统**，能在 /dev 目录下动态创建和删除设备节点。

**udev 完全在用户态工作。它利用设备加入或移除时内核所发送的热插拔事件（Hotplug Event）来工作。在热插拔时，设备的详细信息会由内核通过 netlink 套接字发送出来，发出的事情叫 uevent。**udev 的设备命名策略、权限控制和事件处理都是在用户态下完成的，它利用从内核收到的信息来进行创建设备文件节点等工作。对于冷插拔的设备，在开机的时候就已经存在，在 udev 启动之前就已被插入。Linux 下提供了 sysfs 下面一个 uevent 节点。可以往该节点写一个 add，导致内核重新发送 netlink，之后 udev 就可以收到冷插拔的 netlink 消息了。

udev 和 devfs 另一个区别在于：采用 devfs，当一个并不存在的 /dev 节点**被打开**的时候，devfs 能自动加载对应的驱动，而 udev 则不这么做。udev 的设计者认为 Linux 应该在设备**被发现**的时候加载驱动模块，而不是当它被访问的时候。系统中所有的设备都应该产生热插拔事件并加载恰当的驱动，而不是在设备被打开的时候。udev 能注意到这点并且为它创建对应的设备节点。

### sysfs 文件系统与 Linux 设备模型

Linux 2.6 以后的内核引入了 sysfs 文件系统，是一个虚拟文件系统，可以产生**包括所有系统硬件的层级视图**，与**提供进程和状态信息**的 proc 非常类似。

sysfs 把连接在系统上的设备和总线组织成为一个分级的文件。它们可以由用户空间存取，向用户空间导出内核数据结构以及它们的属性。sysfs 的一个目的就是展示设备驱动模型中各组件的层次关系。

其目录结构类似如下。block 目录包含所有的块设备；devices 目录包含系统所有的设备，并根据设备挂载的总线类型组织成层次结构；bus 目录包含系统中所有的总线类型；class 目录包含系统中的设备类型，如网卡设备、声卡设备、输入设备等。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241107100021716.png" alt="image-20241107100021716" style="zoom:75%;" />

在 /sys/bus/ 的 pci 等子目录下，又会再分出 drivers 和 devices 目录，而 devices 目录中的文件是对 /sys/devices/ 目录中文件的符号链接。同样地，/sys/class/ 目录下也包含许多对 /sys/devices/ 下文件的链接。Linux 设备模型与设备、驱动、总线和类的现实状况是直接对应的，如图所示：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241107100653352.png" alt="image-20241107100653352" style="zoom:70%;" />

**在 Linux 内核中，使用 bus_type、device_driver、device 来描述总线、驱动和设备。**其定义如下：

```c
// include/linux/device/bus.h
struct bus_type {
	const char		*name;
	const char		*dev_name;
	struct device		*dev_root;
	const struct attribute_group **bus_groups;
	const struct attribute_group **dev_groups;
	const struct attribute_group **drv_groups;

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
	void (*sync_state)(struct device *dev);
	void (*remove)(struct device *dev);
	void (*shutdown)(struct device *dev);

	int (*online)(struct device *dev);
	int (*offline)(struct device *dev);

	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);

	int (*num_vf)(struct device *dev);

	int (*dma_configure)(struct device *dev);

	const struct dev_pm_ops *pm;

	const struct iommu_ops *iommu_ops;

	struct subsys_private *p;
	struct lock_class_key lock_key;

	bool need_parent_lock;
};

// include/linux/device/driver.h
struct device_driver {
	const char		*name;
	struct bus_type		*bus;

	struct module		*owner;
	const char		*mod_name;	/* used for built-in modules */

	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */
	enum probe_type probe_type;

	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;

	int (*probe) (struct device *dev);
	void (*sync_state)(struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;
	const struct attribute_group **dev_groups;

	const struct dev_pm_ops *pm;
	void (*coredump) (struct device *dev);

	struct driver_private *p;
};

// include/linux/device.h
struct device {
	struct kobject kobj;
	struct device		*parent;

	struct device_private	*p;

	const char		*init_name; /* initial name of the device */
	const struct device_type *type;

	struct bus_type	*bus;		/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this
					   device */
	void		*platform_data;	/* Platform specific data, device
					   core doesn't touch it */
	void		*driver_data;	/* Driver data, set and get with
					   dev_set_drvdata/dev_get_drvdata */
#ifdef CONFIG_PROVE_LOCKING
	struct mutex		lockdep_mutex;
#endif
	struct mutex		mutex;	/* mutex to synchronize calls to
					 * its driver.
					 */

	struct dev_links_info	links;
	struct dev_pm_info	power;
	struct dev_pm_domain	*pm_domain;

#ifdef CONFIG_ENERGY_MODEL
	struct em_perf_domain	*em_pd;
#endif

#ifdef CONFIG_GENERIC_MSI_IRQ_DOMAIN
	struct irq_domain	*msi_domain;
#endif
#ifdef CONFIG_PINCTRL
	struct dev_pin_info	*pins;
#endif
#ifdef CONFIG_GENERIC_MSI_IRQ
	raw_spinlock_t		msi_lock;
	struct list_head	msi_list;
#endif
#ifdef CONFIG_DMA_OPS
	const struct dma_map_ops *dma_ops;
#endif
	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask;/* Like dma_mask, but for
					     alloc_coherent mappings as
					     not all hardware supports
					     64 bit addresses for consistent
					     allocations such descriptors. */
	u64		bus_dma_limit;	/* upstream dma constraint */
	const struct bus_dma_region *dma_range_map;

	struct device_dma_parameters *dma_parms;

	struct list_head	dma_pools;	/* dma pools (if dma'ble) */

#ifdef CONFIG_DMA_DECLARE_COHERENT
	struct dma_coherent_mem	*dma_mem; /* internal for coherent mem
					     override */
#endif
#ifdef CONFIG_DMA_CMA
	struct cma *cma_area;		/* contiguous memory area for dma
					   allocations */
#endif
#ifdef CONFIG_SWIOTLB
	struct io_tlb_mem *dma_io_tlb_mem;
#endif
	/* arch specific additions */
	struct dev_archdata	archdata;

	struct device_node	*of_node; /* associated device tree node */
	struct fwnode_handle	*fwnode; /* firmware device node */

#ifdef CONFIG_NUMA
	int		numa_node;	/* NUMA node this device is close to */
#endif
	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	u32			id;	/* device instance */

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	struct class		*class;
	const struct attribute_group **groups;	/* optional groups */

	void	(*release)(struct device *dev);
	struct iommu_group	*iommu_group;
	struct dev_iommu	*iommu;

	enum device_removable	removable;

	bool			offline_disabled:1;
	bool			offline:1;
	bool			of_node_reused:1;
	bool			state_synced:1;
	bool			can_match:1;
#if defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_DEVICE) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU_ALL)
	bool			dma_coherent:1;
#endif
#ifdef CONFIG_DMA_OPS_BYPASS
	bool			dma_ops_bypass : 1;
#endif
};
```

device_driver 和 device 分别表示驱动和设备，而这两者都必须依附于一种总线，因此都包含 bus_type 指针。**在 Linux 内核中，设备和驱动是分开注册的。**注册 1 个设备的时候，并不需要驱动已经存在。而 1 个驱动被注册的时候，也不需要对应的设备已经被注册。设备和驱动各自涌向内核，而每个设备和驱动涌入内核的时候，都会去寻找自己的另一半。而正是 bus_type 的 match() 成员函数将两者捆绑在一起。一旦匹配成功，xxx_driver 的 probe() 函数就会被执行（xxx 是总线名， 如 platform、pci、i2c、spi、usb 等）。

**总线、驱动和设备最终都会落实为 sysfs 中的 1 个目录。**它们实际上都可以认为是 kobject 的派生类，**kobject 可看作是所有总线、设备和驱动的抽象基类**，1个 kobject 对应 sysfs 中的 1 个目录。

另外，**总线、驱动和设备中的各个 attribute 直接落实为 sysfs 中的一个文件。**结构体中含有 show() 和 store() 两个函数，分别用于读写对应的 sysfs 文件。这几个 attribute 相关结构体的定义如下：

```c
struct attribute {
	const char		*name;
	umode_t			mode;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	bool			ignore_lockdep:1;
	struct lock_class_key	*key;
	struct lock_class_key	skey;
#endif
};

struct bus_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct bus_type *bus, char *buf);
	ssize_t (*store)(struct bus_type *bus, const char *buf, size_t count);
};

struct driver_attribute {
	struct attribute attr;
	ssize_t (*show)(struct device_driver *driver, char *buf);
	ssize_t (*store)(struct device_driver *driver, const char *buf,
			 size_t count);
};

struct device_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct device *dev, struct device_attribute *attr,
			char *buf);
	ssize_t (*store)(struct device *dev, struct device_attribute *attr,
			 const char *buf, size_t count);
};
```

### udev 的组成

udev 目前和 systemd 项目已合并，地址：[https://github.com/systemd/systemd](https://github.com/systemd/systemd)

udev 在用户空间中执行，动态建立/删除设备文件，允许每个人都不用关心主/次设备号而提供 Linux 标准规范的名称，并且可以根据需要固定名称。

udev 的工作机制如下：

1. udev 以守护进程的方式运行，工作于用户空间，它监听一个 netlink socket。
2. 当有新设备接入或设备从系统中移除时，系统内核通过这个 socket 向 udev 发送 uevent。
3. udev 获取内核发送的信息，进行规则的匹配。匹配的事物包括 SUBSYSTEM、ACTION、atttribute、内核提供的名称（通过 KERNEL =）以及其他的环境变量。

### udev 规则文件

udev 规则文件以行为单位，一行代表一个规则。每个规则分为一个或多个匹配部分和赋值部分。二者带有不同的关键字。

匹配关键字包括：ACTION（行为）、KERNEL（匹配内核设备名）、BUS（匹配总线类型）、SUBSYSTEM（匹配子系统名）、ATTR（属性）等。

赋值关键字包括：NAME（创建的设备文件名）、SYMLINK（符号创建链接名）、OWNER（设置设备的所有者）、GROUP（设置设备的组）、IMPORT（调用外部程序）、MODE（节点访问权限）等。

举个例子看一下：

```markdown
SUBSYSTEM=="net", ACTION=="add", DRIVERS==" *", ATTR{address}=="08:00:27:35:be:ff", ATTR{dev_id}=="0x0", ATTR{type}=="1", KERNEL=="eth*", NAME="eth1"
```

匹配部分包括 SUBSYSTEM、ACTION、ATTR、KERNEL 等，赋值部分有一项，是 NAME。这个规则的意思是：当系统中出现的新硬件属于 net 子系统范畴，系统对该硬件采取的动作是 add 这个硬件，且这个硬件的 address 属性信息等于 "08:00:27:35:be:ff"，dev_id 属性等于 "0x0" 、type 属性为 1 等。此时，对这个硬件在 udev 层实行的动作是创建 /dev/eth1。

# 字符设备驱动

Linux 系统将设备分为3类：**字符设备、块设备、网络设备**。架构图如下：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20250201002122884.png" alt="image-20250201002122884" style="zoom:70%;" />

**字符设备是指只能一个字节一个字节读写的设备，不能随机读取设备内存中的某一数据，读取数据需要按照先后数据。**字符设备是面向流的设备。常见的字符设备有鼠标、键盘、串口、控制台和 LED 设备等。

**块设备是指可以从设备的任意位置读取一定长度数据的设备。**块设备包括硬盘、磁盘、U 盘和 SD 卡等。

每一个字符设备或块设备都在 /dev 目录下对应一个设备文件。Linux 用户程序通过设备文件（或称设备节点）来使用驱动程序操作字符设备和块设备。

## 字符设备驱动结构

### cdev 结构体

在 Linux 内核中，**使用 cdev 结构体描述字符设备**，定义如下：

```c
struct cdev {
	struct kobject kobj; // 内嵌的 kobject 对象
	struct module *owner; // 所属模块
	const struct file_operations *ops; // 文件操作结构体
	struct list_head list;
	dev_t dev; // 设备号
	unsigned int count;
} __randomize_layout;
```

cdev 的一个重要成员 file_operations 定义了字符设备驱动需要提供给虚拟文件系统 VFS 的接口函数。

cdev 的成员 dev_t 定义了设备号，包含主设备号和次设备号。dev_t 实际上就是 unsigned int，32 位。其中高 12 位是主设备号，低 20  位是次设备号。如下是设备号相关的宏函数：

```c
MAJOR(dev) // 通过 dev_t 获得主设备号。
MINOR(dev) // 通过 dev_t 获得次设备号。
MKDEV(major, minor) // 通过主设备号和次设备号构造 dev_t 的设备号。
```

Linux 提供了一系列函数用于操作 cdev 结构体：

```c
// 初始化 cdev 成员，并建立 cdev 和 file_operations 的连接。
void cdev_init(struct cdev *, const struct file_operations *);

// 用于动态申请 cdev 内存。
struct cdev *cdev_alloc(void);

void cdev_put(struct cdev *p);

// 向系统中添加 cdev，完成字符设备的注册。一般用于模块加载函数中。
int cdev_add(struct cdev *, dev_t, unsigned);

// 从系统中删除 cdev，完成字符设备的注销。一般用于模块卸载函数中。
void cdev_del(struct cdev *);
```

### 分配和释放设备号

在调用 cdev_add() 函数注册字符设备之前，需要先向系统申请设备号，即主设备号和次设备号。调用函数 register_chrdev_region() 或 alloc_chrdev_region() 实现：

```c
// 用于已知起始设备的设备号的情况。
int register_chrdev_region(dev_t from, unsigned count, const char *name);

// 用于起始设备号未知，向系统动态申请设备号的情况。
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);
```

注意函数中的参数 count，结合设备号的主设备号和次设备号的概念。我们发现整个流程中，首先需要向内核申请 dev_t 类型的设备号，包括主设备号和次设备号。同一类型的字符设备可共同同一个主设备号，而次设备号不同。因此，这样在添加注册字符设备的时候可以同时注册几个字符设备，这也是 count 的含义。

相应的，在初始释放的时候，除了需要从系统中删除 cdev 完成字符设备的注销，还需要释放原先申请的设备号，函数如下：

```c
void unregister_chrdev_region(dev_t from, unsigned count);
```

### file_operations 结构体

file_operations 结构体中的成员函数是字符设备驱动程序设计的主体内容，这些函数实际会在应用程序进行 Linux 的 open()、write()、read()、close() 等系统调用时最终被内核调用。其定义如下：

```c
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iopoll)(struct kiocb *kiocb, bool spin);
	int (*iterate) (struct file *, struct dir_context *);
	int (*iterate_shared) (struct file *, struct dir_context *);
	__poll_t (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	unsigned long mmap_supported_flags;
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
	ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
			loff_t, size_t, unsigned int);
	loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
				   struct file *file_out, loff_t pos_out,
				   loff_t len, unsigned int remap_flags);
	int (*fadvise)(struct file *, loff_t, loff_t, int);
} __randomize_layout;
```

下面对其进行简要分析：

1. llseek()：修改一个文件的当前读写位置，并返回新位置偏移指针。出错时函数返回负值。
2. read()：**从设备读取数据。**成功时返回读取的字节数，出错时返回一个负值。与用户空间的 `ssize_t read(int fd, void *buf, size_t count);` 函数对应。返回 0 代表 EOF（end of file）。
3. write()：**向设备发送数据。**成功时返回写入的字节数，出错时返回一个负值。如果未实现此函数，用户进行 write() 系统调用时，将得到 -EINVAL 返回值。与用户空间的 `ssize_t write(int fd, const void *buf, size_t count);` 函数对应。同样返回 0 代表 EOF（end of file）。
4. unlocked_ioctl()：提供**设备相关控制命令的实现**（既不是读操作，也不是写操作）。成功时返回一个非负值。与用户空间的 `int fcntl(int fd, int cmd, ... /* arg */ );` 函数对应。
5. mmap()：**将设备内存映射到进程的虚拟地址空间中。**如果未实现此函数，用户进行 mmap() 系统调用的时，会获得 -ENODEV 返回值。这个函数对于帧缓冲等设备特别有意义，帧缓冲被映射到用户空间后，应用程序可以直接访问它而无须在内核和应用间进行内存复制。与用户空间的 `void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);` 函数对应。
6. open()、release()：当用户空间调用 open() 打开设备文件时，设备驱动的 open() 函数最终被调用。驱动程序可以不实现这个函数，在这种情况下，设备的打开操作永远成功。与 open() 函数对应的是 release() 函数，也对应用户空间的 close() 函数。
7. poll()：一般用于询问设备是否可被非阻塞地立即读写。当询问的条件未触发时，用户空间进行 select() 和 poll() 系统调用将引起进程的阻塞。
8. aio_read()、aio_write()：对与文件描述符对应的设备进行**异步**读、写操作。设备实现这两个函数后，用户空间可以对该设备文件描述符执行 SYS_io_setup、SYS_io_submit、SYS_io_getevents、SYS_io_destroy 等系统调用进行读写。

### 字符设备驱动的组成

#### 模块加载与卸载函数

在模块加载函数中应实现设备号的申请和 cdev 的注册，在模块卸载函数中应实现 cdev 的注销和设备号的释放。

类似的模板如下：

```c
// 设备结构体
struct xxx_dev_t
{
    struct cdev cdev;

    ...
} xxx_dev;


// 设备驱动模块加载函数
static int __init xxx_init(void)
{
    ... 

    cdev_init(&xxx_dev.cdev, &xxx_fops); // 初始化cdev
    xxx_dev.cdev.owner = THIS_MODULE;

    // 获取字符设备号
    if (xxx_major)
    {
        register_chrdev_region(xxx_dev_no, 1, DEV_NAME);
    }
    else
    {
        alloc_chrdev_region(&xxx_dev_no, 0, 1, DEV_NAME);
    }

    res = cdev_add(&xxx_dev.cdev, xxx_dev_no, 1); // 注册设备

    ...
}

// 设备驱动模块卸载函数
static void __exit xxx_exit(void)
{
    unregister_chrdev_region(xxx_dev_no, 1); // 释放占用的设备号
    cdev_del(&xxx_dev.cdev);                 // 注销设备

    ...
}
```

#### file_operations 结构体的成员函数

file_operations 结构体的成员函数是字符设备驱动与内核虚拟文件系统的接口，是用户空间对 Linux 系统调用最终的落实者。

```c
// 读设备
// filp：文件结构体指针，buf：用户空间内存地址；count：要读的字节数；f_pos：读的位置相对文件开头的偏移。
ssize_t xxx_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
    ...

    copy_to_user(buf, ..., ...);

    ...
}

// 写设备
// filp：文件结构体指针，buf：用户空间内存地址；count：要写的字节数；f_pos：读的位置相对文件开头的偏移。
ssize_t xxx_write(struct file *filp, const char __user *buf, size_t count, loff_t *f_pos)
{
    ...

    copy_from_user(..., buf, ...);

    ...
}

// ioctl 函数
// cmd 参数为事先定义的 I/O 控制命令，而 arg 为对应于该命令的参数。
// 例如对于串行设备，若 SET_BAUDRATE 是一道设置波特率的命令，arg 就应该是波特率值。
long xxx_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    ...

    switch (cmd)
    {
        case XXX_CMD1:
        {
            ...

            break;
        }
        case XXX_CMD2:
        {
            ...

            break;
        }
        default:
            // 不能支持的命令
            return -ENOTTY;
    }


    return 0;
}
```

由于**用户空间不能直接访问内核空间的内存**，因此借助了函数 copy_from_user() 完成用户空间缓冲区到内核空间的复制，以及 copy_to_user() 完成内核空间到用户空间缓冲区的复制。这里注意一下语义，以读为例，注意主谓。xxx_read() 的含义是从内核空间中读，因此用户空间是接收的对象，因此使用 copy_to_user()，对应的也是用户空间的系统调用 read()。整个流程就是内核读 I/O（先不考虑用户区缓冲的事情），然后 copy_to_user() 到用户空间的内存。

这两个函数的声明如下：

```c
// 两个函数均返回不能被复制的字节数。如果成功，返回 0；如果失败，返回负值。
int copy_from_user(void *to, const void __user volatile *from, unsigned long n);
int copy_to_user(void __user volatile *to, const void *from, unsigned long n);
```

如果复制的内存是简单类型，如 char、int、long 等，可以使用简化的 put_user() 和 get_user()：

```c
int val; // 内核空间整型变量。

get_user(val, (int *) arg); // 用户 -> 内核，arg 是用户空间的地址。

put_user(val, (int *) arg); // 内核 -> 用户，arg 是用户空间的地址。
```

上面读和写函数的参数中 `__user` 是一个宏，代表后面的指针属于用户空间，用于假注释和提醒的作用。类似于模块加载函数的 `__init` 和模块卸载函数的 `__exit`。

内核空间虽然可以访问用户空间的数据，但是在访问之前，一般需要**检查合法性**。使用 `access_ok(addr, size)` 接口，以保证传入的数据的确属于用户空间。**此宏函数返回非 0 表示检查通过。**

另外，`put_user()` 和 `__put_user()` 的区别在于，`put_user()` 的实现中调用了 `__put_user()`，并且已经手动做了 `access_ok()` 的检查。所以一般推荐使用 `put_user()` 和 `get_user()`。同时，也存在 `copy_from_user()` 和 `__copy_from_user()`，区别同上。`copy_to_user()` 同理。但是，总的来讲为了保险，内核空间在访问用户空间数据以前，我们的驱动程序中应再次手动检查一下合法性。

在字符设备驱动中，需要定义一个 file_operations 的实例，将我们自己实现的函数注册给这些成员：

```c
// xxx_fops 在 cdev_init(&xxx_dev.cdev, &xxx_fops) 时同字符设备建立起连接。
struct file_operations xxx_fops = {
    .owner = THIS_MODULE,
    .read = xxx_read,
    .write = xxx_write,
    .unlocked_ioctl = xxx_ioctl,

    ...
};
```

下面是字符设备驱动的结构示意图：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241108100951215.png" alt="image-20241108100951215" style="zoom:75%;" />

## globalmem 虚拟设备

参考文章：[https://blog.csdn.net/weixin_45264425/article/details/130718602](https://blog.csdn.net/weixin_45264425/article/details/130718602)

自己实现的版本：[https://github.com/DavidingPlus/linux-kernel-learning/tree/globalmem](https://github.com/DavidingPlus/linux-kernel-learning/tree/globalmem)

## 小结

字符设备是 3 大类设备（字符设备、块设备和网络设备）中的一类。其驱动程序完成的主要工作是初始化、添加和删除 cdev 结构体，申请和释放设备号，以及填充 file_operations 结构体中的操作函数。实现 file_operations 结构体中的 read()、write() 和 ioctl() 等函数是驱动设计的主体工作。

# 设备驱动中的并发控制

Linux 设备驱动中必须要解决的问题是多个进程对共享资源的并发访问，并发会导致竞态，所以需要格外小心。

## 并发与竞态

并发（Concurrency）是指多个执行单元同时、并行被执行。而并发的执行单元对共享资源（硬件资源和软件上的全局变量、静态变量等）的访问则很容易导致竞态（Race Conditions）。

在 Linux 内核中，竞态主要分为以下几种情况：

1. 对称多处理器（SMP）的多个 CPU

SMP 是一种紧耦合、共享存储的系统模型，特点是多个 CPU 使用共同的系统总线，因此可访问共同的外设和储存器。体系结构如图：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241111152648934.png" alt="image-20241111152648934" style="zoom:80%;" />

在 SMP 的情形下，两个核(例如 CPU 0 和 CPU 1)的竞态可能发生在 CPU 0 和 CPU 1 的进程之间，CPU 0 的进程和 CPU 1 的中断之间，CPU 0 和 CPU 1 的中断之间等，如图所示：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241111152902140.png" alt="image-20241111152902140" style="zoom:80%;" />

2. 单 CPU 内进程与抢占它的进程

Linux 2.6 以后的内核支持内核抢占调度，一个进程在内核执行的时候可能耗完了自己的时间片（timeslice），也可能被另一个高优先级进程打断，进程与抢占它的进程访问共享资源的情况类似于 SMP 的多个 CPU。

3. 中断（硬中断、软中断、Tasklet、底半部）与进程之间

中断可以打断正在执行的进程，如果中断服务程序访问进程正在访问的资源，竞态也会发生。

另外，中断也有可能被新的更高优先级的中断打断，故多个中断之间本身也可能引起并发而导致竞态。但 Linux 2.6 之后，就取消了中断的嵌套。老版本的内核可以在申请中断时，设置标记 IRQF_DISABLED 以避免中断嵌套。但由于新内核直接就默认不嵌套中断，这个标记反而变得无用了。

上述三种并发的发生，除了 SMP 是真正的并行之外，其他的都是单核上的“宏观并行、微观串行”，但遇到的问题和 SMP 类似。CPU 核内和核间的并发示意图如下：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241111153624322.png" alt="image-20241111153624322" style="zoom:75%;" />

解决竞态问题的途径是保证对共享资源的互斥访问。互斥访问是指一个执行单元在访问共享资源的时候，其他的执行单元被禁止访问。

访问共享资源的代码区域称为临界区（Critical Sections），临界区需要被以某种互斥机制加以保护。中断屏蔽、原子操作、自旋锁、信号量、互斥体等都是 Linux 设备驱动中可采用的互斥途径。

## 编译乱序和执行乱序

编译乱序是编译器的行为，执行乱序则是处理器运行时的行为。

关于编译乱序，现代的高性能编译器在目标码优化上都具备对指令进行乱序优化的能力。编译器可以对访存的指令进行乱序，减少逻辑上不必要的访存，以及尽量提高 Cache 命中率和 CPU 的 Load/Store 单元的工作效率。因此在打开编译器优化（例如 gcc 的 -O2 优化）以后，看到生成的汇编码并没有严格按照代码的逻辑顺序，这是正常的。

更多编译乱序的细节，参考：[https://chonghw.github.io/blog/2016/09/05/compilermemoryreorder/](https://chonghw.github.io/blog/2016/09/05/compilermemoryreorder/)

关于执行乱序，是指即便编译的二进制指令的按照顺序排放，在处理器上执行时，后排放的指令还是可能先执行完，这是处理器的“乱序执行（Out-of-Order Execution）”策略。高级的 CPU 可以根据自己缓存的组织特性，将访存指令重新排序执行。连续地址的访问可能会先执行，因为这样缓存命中率高。有的还允许访存的非阻塞，即如果前面一条访存指令因为缓存不命中，造成长延时的存储访问时，后面的访存指令可以先执行，以便从缓存中取数。因此即使是从汇编上看顺序正确的指令，其执行的顺序也是不可预知的。

更多执行乱序的细节，参考：[https://chonghw.github.io/blog/2016/09/19/sourcecontrol/](https://chonghw.github.io/blog/2016/09/19/sourcecontrol/)

## 中断屏蔽

在单 CPU 范围内避免竞态的一种简单而有效的方法是在进入临界区之前屏蔽系统的中断，但在驱动编程中不值得推荐，驱动通常需要考虑跨平台特点而不假定自己在单核上运行。CPU 一般都具备屏蔽中断和打开中断的功能，这项功能可以保证正在执行的内核执行路径不被中断处理程序所抢占，防止某些竞态条件的发生。具体而言，**中断屏蔽将使得中断与进程之间的并发不再发生**，并且由于 Linux 内核的进程调度等操作都依赖中断来实现，内核抢占进程之间的并发也得以避免了。

**中断屏蔽的底层原理是让 CPU 本身不响应中断**，使用方法如下：

```c
local_irq_disable(); // 屏蔽中断
. . .
critical section // 临界区操作
. . .
local_irq_enable(); // 开中断
```

长时间屏蔽中断是很危险的，这有可能造成数据丢失乃至系统崩溃等后果。因此在屏蔽了中断之后，当前的内核执行路径应当尽快地执行完毕临界区的代码。

local_irq_disable() 和 local_irq_enable() 都只能禁止和激活本 CPU 内的中断，故并不能解决 SMP 多 CPU 引发的竞态。因此，单独使用中断屏蔽不是值得推荐的方法，它适合与自旋锁结合起来使用。

与 local_irq_disable() 不同的是，local_irq_save(flags) 除了进行禁止中断的操作以外，还保存目前 CPU 的中断位信息，local_irq_restore(flags) 进行的是与 local_irq_save(flags) 相反的操作。如果只是想禁止中断的底半部，应使用 local_bh_disable()，激活对应 local_bh_enable()。

## 原子操作

**原子操作可以保证对一个整型数据的修改是排他性的。**Linux 内核提供了一系列函数实现，分为两类，分别针对**位和整型变量**进行原子操作。它们都依赖于底层 CPU 的原子操作，与 CPU 架构密切相关。

### 整型原子操作

1. 设置原子变量的值

```c
void atomic_set(atomic_t *v, int i); // 设置原子变量的值为 i
atomic_t v = ATOMIC_INIT(0); // 定义原子变量 v 并初始化为 0
```

2. 获取原子变量的值

```c
atomic_read(const atomic_t *v); // 返回原子变量的值
```

3. 原子变量加/减

```c
void atomic_add(int i, atomic_t *v); // 原子变量增加 i
void atomic_sub(int i, atomic_t *v); // 原子变量减少 i
```

4. 原子变量自增/自减

```c
void atomic_inc(atomic_t *v); // 原子变量增加 1
void atomic_dec(atomic_t *v); // 原子变量减少 1
```

5. 操作并测试

这些操作对原子变量执行自增、自减和减操作后（**注意没有加**），测试其是否为 0，为 0 返回 true，否则返回 false。

```c
int atomic_inc_and_test(atomic_t *v);
int atomic_dec_and_test(atomic_t *v);
int atomic_sub_and_test(int i, atomic_t *v);
```

6. 操作并返回

这些操作对原子变量进行加/减和自增/自减操作，并返回新的值。

```c
int atomic_add_return(int i, atomic_t *v);
int atomic_sub_return(int i, atomic_t *v);
int atomic_inc_return(atomic_t *v);
int atomic_dec_return(atomic_t *v);
```

### 位原子操作

1. 设置位

```c
void set_bit(int nr, void *addr); // 设置 addr 地址的第 nr 位，所谓设置位即是将位写为 1
```

2. 清除位

```c
void clear_bit(int nr, void *addr); // 清除 addr 地址的第 nr 位，所谓设置位即是将位写为 0
```

3. 改变位

```c
void change_bit(int nr, void *addr); // 将 addr 地址的第 nr 位进行反置
```

4. 测试位

```c
int change_bit(int nr, void *addr); // 返回 addr 地址的第 nr 位
```

5. 测试并操作位

`test_and_xxx_bit(int nr，void *addr)` 操作等同于执行 `test_bit(int nr，void *addr)` 后再执行 `xxx_bit(int nr，void *addr)`。

```c
int test_and_set_bit(int nr, void *addr);
int test_and_clear_bit(int nr, void *addr);
int test_and_change_bit(int nr, void *addr);
```

## 自旋锁

### 自旋锁的使用

自旋锁是一种典型的对临界资源进行互斥访问的手段。为了获得一个自旋锁，进程需先进行一个原子操作，该操作测试并设置（Test-And-Set）一个内存变量。由于是原子操作，所以在该操作完成之前其他执行单元不可能访问这个内存变量。如果测试结果表明锁已经空闲，则程序获得这个自旋锁并继续执行；如果测试结果表明锁仍被占用，程序将在一个小的循环内重复这个“测试并设置”操作，即进行所谓的“自旋”，通俗地说就是“在原地打转”。当自旋锁的持有者通过重置该变量释放这个自旋锁后，某个等待的“测试并设置”操作 向其调用者报告锁已释放。

自旋锁可理解为一个变量。它把一个临界区标记为“我当前在运行，请稍等一会”或“我当前不在运行，可以被使用”。如果 A 执行单元首先进入例程，它将持有自旋锁。当 B 执行单元试图进入同一个例程时，将获知自旋锁已被持有，需等到 A 执行单元释放后才能进入。

自旋锁涉及的操作如下：

1. 定义自旋锁

```c
spinlock_t lock;
```

2. 初始化自旋锁

```c
spin_lock_init(lock); // 动态初始化自旋锁 lock。
```

3. 获得自旋锁

```c
spin_lock(lock); // 用于获得自旋锁 lock。如果能够立即获得，就马上返回。否则，将在那里自旋，直到该自旋锁的保持者释放。
spin_trylock(lock); // 尝试获得自旋锁 lock。如果能立即获得，它获得锁并返回 true，否则立即返回 false，实际上不再“在原地打转”。
```

4. 释放自旋锁

```c
spin_unlock(lock); // 释放自旋锁 lock，它与 spin_trylock() 或 spin_lock() 配对使用。
```

自旋锁主要针对 SMP 或单 CPU（内核可抢占）的情况，对于单 CPU（内核不支持抢占）的系统，自旋锁退化为空操作，因为此时不会出现并发和竞态。在单 CPU（内核可抢占）的系统中，自旋锁持有期间中内核的抢占将被禁止。由于单 CPU（内核可抢占）系统的行为实际上很类似于 SMP 系统，因此在这样的单 CPU 系统中使用自旋锁仍十分必要。另外，在多核 SMP 的情况下，任何一个核拿到了自旋锁，**该核**上的抢占调度也暂时禁止了，但是**没有禁止**另外一个核的抢占调度。

尽管自旋锁可以保证临界区不受别的 CPU 和本 CPU 内的抢占进程打扰，但是得到锁的代码路径在执行临界区的时候，还可能受到中断和底半部（BH）的影响。因此，需要用到自旋锁的衍生。这些函数为自旋锁的使用提供保障，防止突如其来的中断对系统造成伤害。

```c
spin_lock_irq() = spin_lock() + local_irq_disable()
spin_unlock_irq() = spin_unlock() + local_irq_enable()
spin_lock_irqsave() = spin_lock() + local_irq_save()
spin_unlock_irqrestore() = spin_unlock() + local_irq_restore()
spin_lock_bh() = spin_lock() + local_bh_disable()
spin_unlock_bh() = spin_unlock() + local_bh_enable()
```

在多核编程的时候，如果进程和中断可能访问同一片临界资源，**我们一般需要在进程上下文中调用 spin_lock_irqsave() 和 spin_unlock_irqrestore()，在中断上下文中调用 spin_lock() 和 spin_unlock()。**这样能避免一切核内和核间并发的可能性。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241112094336501.png" alt="image-20241112094336501" style="zoom:75%;" />

这样考虑以后，自旋锁一般这样使用：

```c
spinlock_t lock; // 定义一个自旋锁
spin_lock_init(&lock); // 初始化自旋锁

// 进程上下文
spin_lock_irqsave(&lock); // 获取自旋锁，保护临界区
... // 临界区
spin_unlock_irqrestore(&lock); // 解锁

// 中断上下文
spin_lock(&lock);
... // 临界区
spin_unlock(&lock);
```

在使用自旋锁的时候还需注意以下几个问题：

1. **自旋锁实际上是忙等锁。**当锁不可用时，CPU 一直循环执行“测试并设置”该锁直到可用而取得该锁，CPU 在等待自旋锁时不做任何有用的工作，仅仅是等待。因此，**只有在占用锁的时间极短的情况下，使用自旋锁才是合理的。**当临界区很大，或有共享设备的时候，需要较长时间占用锁，使用自旋锁会降低系统的性能。
2. **自旋锁可能导致系统死锁。**引发这个问题最常见的情况是递归使用一个自旋锁，即如果一个已经拥有某个自旋锁的 CPU 想第二次获得这个自旋锁，则该 CPU 将死锁。
3. **在自旋锁锁定期间不能调用可能引起进程调度的函数。**如果进程获得自旋锁之后再阻塞，如调用 copy_from_user()、copy_to_user()、kmalloc() 和 msleep() 等函数，则可能导致内核的崩溃。
4. **在单核情况下编程的时候，也应该认为自己的 CPU 是多核的。**在单 CPU 的情况下，中断中不调用 spin_lock() 也没有问题，因为进程中的 spin_lock_irqsave() 能保证同 CPU 的中断服务程序不可能执行，但如果是多核则无法屏蔽另一个核的中断，可能出现问题。因为无论如何在中断服务程序中都应加上 spin_lock()，同时也是为了编程的统一。

### 读写自旋锁

自旋锁对读写操作一视同仁。但对于读操作，多个执行单元同时读取是不会有问题的。因此衍生了读写自旋锁，它是一种比自旋锁粒度更小的锁机制。它保留了“自旋”的概念，但是在写操作方面，只能最多有 1 个写进程，在读操作方面，同时可以有多个读执行单元。当然，读和写也不能同时进行。

读写自旋锁涉及的操作如下：

1. 定义和初始化读写自旋锁

```c
rwlock_t my_rwlock;
rwlock_init(&my_rwlock); // 动态初始化
```

2. 读锁定

```c
void read_lock(rwlock_t *lock);
void read_lock_irqsave(rwlock_t *lock, unsigned long flags);
void read_lock_irq(rwlock_t *lock);
void read_lock_bh(rwlock_t *lock);
```

3. 读解锁

```c
void read_unlock(rwlock_t *lock);
void read_unlock_irqrestore(rwlock_t *lock, unsigned long flags);
void read_unlock_irq(rwlock_t *lock);
void read_unlock_bh(rwlock_t *lock);
```

4. 写锁定

```c
void write_lock(rwlock_t *lock);
void write_lock_irqsave(rwlock_t *lock, unsigned long flags);
void write_lock_irq(rwlock_t *lock);
void write_lock_bh(rwlock_t *lock);
int write_trylock(rwlock_t *lock);
```

5. 写解锁

```c
void write_unlock(rwlock_t *lock);
void write_unlock_irqrestore(rwlock_t *lock, unsigned long flags);
void write_unlock_irq(rwlock_t *lock);
void write_unlock_bh(rwlock_t *lock);
```

这些接口的含义类似于自旋锁，一般这样使用：

```c
rwlock_t lock; // 定义rwlock
rwlock_init(&lock); // 初始化rwlock

read_lock(&lock); // 读时获取锁
... // 临界资源
read_unlock(&lock); // 读后释放锁

write_lock_irqsave(&lock, flags);// 写时获取锁
... // 临界资源
write_unlock_irqrestore(&lock, flags); // 写后释放锁
```

### 顺序锁

顺序锁（seqlock）是对读写锁的一种优化。若使用顺序锁，**读执行单元不会被写执行单元阻塞**。读执行单元在写执行单元对被顺序锁保护的共享资源进行写操作时仍然可以继续读，而不必等待写执行单元完成写操作。同样写执行单元也不需要等待所有读执行单元完成读操作才去进行写操作。但是，写执行单元与写执行单元之间仍然是互斥的。如果有写执行单元在进行写操作，其他写执行单元必须自旋在那里，直到写执行单元释放了顺序锁。

但这样很容易衍生出一个问题。**尽管读写之间不互相排斥，但是如果读执行单元在读操作期间，写执行单元已经发生了写操作，那么，读执行单元必须重新读取数据，以便确保得到的数据是完整的。**在这种情况下，读端可能反复读多次同样的区域才能读到有效的数据。

由此可见，顺序锁的操作分为写执行单元和读执行单元，二者分别由不同的接口。

写执行单元涉及的操作如下：

1. 获得顺序锁

```c
void write_seqlock(seqlock_t *sl);
int write_tryseqlock(seqlock_t *sl);
write_seqlock_irqsave(lock, flags);
write_seqlock_irq(lock);
write_seqlock_bh(lock);
```

2. 释放顺序锁

```c
void write_sequnlock(seqlock_t *sl);
write_sequnlock_irqrestore(lock, flags);
write_sequnlock_irq(lock);
write_sequnlock_bh(lock);
```

使用写执行单元的模式如下：

```c
write_seqlock_irqsave(&seqlock_a, flags);
.../* 写操作代码块 */
write_sequnlock_irqrestore(&seqlock_a, flags);
```

读执行单元涉及的操作如下：

1. 读开始

读执行单元在对被顺序锁 s 保护的共享资源进行访问前需要调用该函数，该函数返回顺序锁 s 的当前顺序号。

```c
unsigned read_seqbegin(const seqlock_t *sl);
read_seqbegin_irqsave(lock, flags) = local_irq_save() + read_seqbegin()
```

2. 重读

读执行单元在访问完被顺序锁 s 保护的共享资源后需要调用该函数来检查，在读访问期间是否有写操作。如果有写操作，读执行单元就需要重新进行读操作，以保证读取的数据是最新的。

```c
int read_seqretry(const seqlock_t *sl, unsigned iv);
read_seqretry_irqrestore(lock, iv, flags) = read_seqretry() + local_irq_restore()
```

使用读执行单元的模式如下：

```c
do {
// 读操作代码块
seqnum = read_seqbegin(&seqlock_a);
...
} while (read_seqretry(&seqlock_a, seqnum))
```

### RCU

RCU，Read-Copy-Update，读-复制-更新。于 Linux 2.6 后正式包含在内核中。社区文档参考：[https://www.kernel.org/doc/ols/2001/read-copy.pdf](https://www.kernel.org/doc/ols/2001/read-copy.pdf)

RCU（Read-Copy-Update），是 Linux 中比较重要的一种同步机制。顾名思义就是“读，拷贝更新”，再直白点是**“随意读，但更新数据的时候，需要先复制一份副本，在副本上完成修改，再一次性地替换旧数据”**。这是 Linux 内核实现的一种针对**“读多写少”**的共享数据的同步机制。

**使用 RCU 的读端没有锁、内存屏障、原子指令类的开销，几乎可以认为是直接读（只是简单地标明读开始和读结束），而 RCU 的写执行单元在访问它的共享资源前首先复制一个副本，然后对副本进行修改，最后使用一个回调机制在适当的时机把指向原来数据的指针重新指向新的被修改的数据。这个时机就是所有引用该数据的 CPU 都退出对共享数据读操作的时候。等待适当时机的这一时期称为宽限期（Grace Period）。**

RCU 可以看作读写锁的高性能版本。相比读写锁，RCU 的优点在于既允许多个读执行单元同时访问被保护的数据，又允许多个读执行单元和多个写执行单元同时访问被保护的数据。但 RCU 不能替代读写锁，因为如果写比较多时，对读执行单元的性能提高不能弥补写执行单元同步导致的损失。因为使用 RCU 时，写执行单元之间的同步开销会比较大，它需要延迟数据结构的释放，复制被修改的数据结构，也必须使用某种锁机制来同步并发的其他写执行单元的修改操作。

**读者在访问被 RCU 保护的共享数据期间不能被阻塞，这是 RCU 机制得以实现的一个基本前提。**也就说当读者在引用被 RCU 保护的共享数据期间，读者所在的 CPU 不能发生上下文切换，spinlock 和 rwlock 都需要这样的前提。写者在访问被 RCU 保护的共享数据时不需要和读者竞争任何锁，只有在有多于一个写者的情况下需要获得某种锁以与其他写者同步。

RCU 涉及的操作如下：

1. 读锁定

```c
rcu_read_lock();
rcu_read_lock_bh();
```

2. 读解锁

```c
rcu_read_unlock();
rcu_read_unlock_bh();
```

使用 RCU 读的模式如下：

```c
rcu_read_lock();
... // 读临界区
rcu_read_unlock();
```

3. 同步 RCU

```c
synchronize_rcu();
```

该函数由 RCU 写执行单元调用，它将阻塞写执行单元，直到当前 CPU 上所有的已经存在的读执行单元完成读临界区，写执行单元才可以继续下一步操作。synchronize_rcu() 并不需要等待后续读临界区的完成。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241112104436256.png" alt="image-20241112104436256" style="zoom:80%;" />

4. 挂接回调

```c
void call_rcu(struct rcu_head *head, void (*func)(struct rcu_head *rcu));
```

该函数也由 RCU 写执行单元调用，与 synchronize_rcu() 不同的是，它不会使写执行单元阻塞，因而可以在中断上下文或软中断中使用。该函数把函数 func() 挂接到 RCU 回调函数链上，然后立即返回。挂接的回调函数会在一个宽限期结束（即所有已经存在的 RCU 读临界区完成）后被执行。

下列的函数用于实现内存屏障的作用。

给 RCU 保护的指针赋一个新的值：

```c
rcu_assign_pointer(p, v);// p 是被赋值指针，v 是赋值指针
```

读端使用 rcu_dereference() 获取一个 RCU 保护的指针，之后既可以安全地引用它（访问它指向的区域）。

```c
rcu_dereference(p);
```

读端使用 rcu_access_pointer() 获取一个 RCU 保护的指针，之后并不引用它。这种情况下，我们只关心指针本身的值，而不关心指针指向的内容。比如我们可以使用该 API 来判断指针是否为 NULL。

```c
rcu_access_pointer(p);
```

对于链表数据结构而言，Linux 内核专门增加了 RCU 保护的链表操作 API：

```c
// 把链表元素 new 插入 RCU 保护的链表 head 的开头。
static inline void list_add_rcu(struct list_head *new, struct list_head *head);

// 把链表元素 new 添加到被 RCU 保护的链表的末尾。
static inline void list_add_tail_rcu(struct list_head *new, struct list_head *head);

// 从 RCU 保护的链表中删除指定的链表元素 entry。
static inline void list_del_rcu(struct list_head *entry);

// 使用新的链表元素 new 取代旧的链表元素 old。
static inline void list_replace_rcu(struct list_head *old, struct list_head *new);

// 该宏用于遍历由 RCU 保护的链表 head。只要在读执行单元临界区使用该函数，它就可以安全地和其他 RCU 保护的链表操作函数（如 list_add_rcu() 等）并发运行。
list_for_each_entry_rcu(pos, head)
```

更多细节和原理，请参考：[https://www.cnblogs.com/schips/p/linux_cru.html](https://www.cnblogs.com/schips/p/linux_cru.html)

## 信号量

信号量（Semaphore）是操作系统中最典型的用于同步和互斥的手段，信号量的值可以是 0、1 或者 n。信号量与操作系统中的经典概念 PV 操作对应。

P(S)：

1. 将信号量 S 的值减 1，即 S=S-1。
2. 如果 S>=0，则该进程继续执行；否则该进程置为等待状态，排入等待队列。

V(S)：

1. 将信号量 S 的值加 1，即 S=S+1。
2. 如果 S>0，唤醒队列中等待信号量的进程。

信号量涉及的操作如下：

1. 定义信号量

```c
struct semaphore
{
    raw_spinlock_t lock;
    unsigned int count;
    struct list_head wait_list;
};
```

2. 初始化信号量

```c
void sema_init(struct semaphore *sem, int val); // 初始化信号量 sem 并设置为值 val。
```

3. 获得信号量

```c
// 获得信号量 sem，会导致睡眠，因此不能在中断上下文中使用。
void down(struct semaphore *sem);

// 功能与 down() 类似，不同之处为：down() 进入睡眠状态的进程不能被信号打断，down_interruptible() 进入睡眠状态的进程能被信号打断，信号也会导致该函数返回，这时候函数的返回值非 0。
// 使用 down_interruptible() 时，对返回值一般会进行检查。如果返回的非 0，通常立即返回 -ERESTARTSYS。
int down_interruptible(struct semaphore *sem); 

// 尝试获得信号量 sem，如果能够立刻获得，它就获得该信号量并返回 0，否则返回非 0 值。它不会导致调用者睡眠，可以在中断上下文中使用。
int down_trylock(struct semaphore *sem);
```

4. 释放信号量

```c
// 释放信号量 sem，唤醒等待者。
void up(struct semaphore *sem);
```

信号量可以保护临界区，它的使用方式和自旋锁类似。与自旋锁相同，只有得到信号量的进程才能执行临界区代码。但与自旋锁不同的是，**当获取不到信号量时，进程不会原地打转而是进入休眠等待状态。**

用作互斥时，信号量一般这样被使用：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241112141145150.png" alt="image-20241112141145150" style="zoom:75%;" />

关心具体数值的生产者/消费者问题，使用信号量较为合适。生产者/消费者问题也是一种同步问题。

## 互斥体

新的 Linux 内核倾向于直接使用互斥体 mutex 作为互斥手段，信号量作互斥不再被推荐使用，尽管信号量可以实现互斥的功能。

互斥体涉及的操作如下：

1. 定义互斥体

```c
struct mutex my_mutex;
mutex_init(&my_mutex);
```

2. 初始化互斥体

```c
// mutex_lock() 不可以被信号打断，mutex_lock_interruptible() 可以。
void mutex_lock(struct mutex *lock);
int mutex_lock_interruptible(struct mutex *lock); 
int mutex_trylock(struct mutex *lock);
```

3. 释放互斥体

```c
void mutex_unlock(struct mutex *lock);
```

互斥体 mutex 使用的模式如下：

```c
struct mutex my_mutex; // 定义 mutex
mutex_init(&my_mutex); // 初始化 mutex
mutex_lock(&my_mutex); // 获取 mutex
... // 临界资源
mutex_unlock(&my_mutex); // 释放 mutex
```

## 如何选择自旋锁和互斥体

自旋锁和互斥体都是解决互斥问题的基本手段，面对特定的情况，如何取舍这两种手段呢？选择的依据是临界区的性质和系统的特点。

从严格意义上说，**互斥体和自旋锁属于不同层次的互斥手段，前者的实现依赖于后者**。在互斥体本身的实现上，为了保证互斥体结构存取的原子性，需要自旋锁来互斥。所以自旋锁属于更底层的手段。

互斥体是进程级的，用于多个进程之间对资源的互斥，虽然也是在内核中，但是该内核执行路径是以进程的身份，代表进程来争夺资源的。如果竞争失败，会发生进程上下文切换，当前进程进入睡眠状态，CPU 将运行其他进程。鉴于进程上下文切换的开销也很大，因此，只有当进程占用资源时间较长时，用互斥体才是较好的选择。

当所要保护的临界区访问时间比较短时，用自旋锁是非常方便的，因为它可节省上下文切换的时间。但 CPU 得不到自旋锁会在那里空转直到其他执行单元解锁为止，所以要求锁不能在临界区里长时间停留，否则会降低系统的效率。

因此，可以总结出三大原则：

1. 当锁不能被获取到时，使用互斥体的开销是进程上下文切换时间，使用自旋锁的开销是等待获取自旋锁（由临界区执行时间决定）。**若临界区比较小，宜使用自旋锁，若临界区很大，应使用互斥体。**
2. 互斥体所保护的临界区可包含可能引起阻塞的代码，而**自旋锁则绝对要避免用来保护包含可能引起阻塞代码的临界区**。因为阻塞意味着要进行进程的切换，如果进程被切换出去后，另一个进程企图获取本自旋锁，死锁就会发生。
3. 互斥体存在于进程上下文，因此，如果被保护的共享资源需要在中断或软中断情况下使用，则在互斥体和自旋锁之间只能选择自旋锁。当然，如果一定要使用互斥体，则只能通过 mutex_trylock() 方式 进行，不能获取就立即返回以避免阻塞。

## 完成量

Linux 提供了完成量（completion），用于一个执行单元等待另一个执行单元执行完某事。

完成量涉及的操作如下：

1. 定义完成量

```c
struct completion my_completion;
```

2. 初始化完成量

```c
init_completion(&my_completion); // 初始化完成量的值为 0（未完成的状态）
reinit_completion(&my_completion); // 重新初始化
```

3. 等待完成量

```c
void wait_for_completion(struct completion *c);
```

4. 唤醒完成量

```c
void complete(struct completion *c); // 唤醒一个等待的执行单元。
void complete_all(struct completion *c); // 唤醒所有等待同一完成量的执行单元。
```

完成量用于同步的流程如下：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241112143006097.png" alt="image-20241112143006097" style="zoom:75%;" />

## 支持并发控制的 globalmem 设备驱动

TODO

## 小结

Linux 内核优化了自旋锁、信号量、互斥体、完成量等的管理，**并不需要显式销毁，在数据结构生命周期结束时自动释放**。因此只有初始化的相关接口，没有销毁的相关接口。

并发和竞态广泛存在，中断屏蔽、原子操作、自旋锁和互斥体都是解决并发问题的机制。中断屏蔽很少单独被使用，原子操作只能针对整数进行，因此自旋锁和互斥体应用最为广泛。

**自旋锁会导致死循环，锁定期间不允许阻塞，因此要求锁定的临界区小。互斥体允许临界区阻塞，可以适用于临界区大的情况。**

# 设备驱动中的阻塞与非阻塞 I/O

阻塞和非阻塞 I/O 是设备访问的两种不同模式，驱动程序可以灵活地支持这两种用户空间对设备的访问方式。

## 阻塞与非阻塞 I/O

阻塞操作是指在执行设备操作时，若不能获得资源，则挂起进程直到满足可操作的条件后再进行操作。被挂起的进程进入睡眠状态，被从调度器的运行队列移走，直到等待的条件被满足。而非阻塞操作的进程在不能进行设备操作时，并不挂起，它要么放弃，要么不停地查询（轮询），直至可以进行操作为止。

驱动程序应当提供阻塞与非阻塞的能力。当应用程序阻塞 read()、write() 的时候，若资源无法获取，驱动程序的 xxx_read()、xxx_write() 需要阻塞进程知道资源可以获取。同理，应用程序非阻塞的时候且资源无法获取的时候，驱动程序的 xxx_read()、xxx_write() 操作应立即返回，系统调用也随即返回，应用程序收到 -EAGAIN 返回值。

在阻塞访问时，不能获取资源的进程将进入休眠，它将 CPU 资源让给其他进程。由于阻塞的进程会进入休眠状态，所以必须确保有一个地方能够唤醒休眠的进程，否则，进程就真的醒不过来了。唤醒进程的地方最大可能发生在**中断**里面，因为在硬件资源获得的同时往往伴随着一个中断。而非阻塞的进程则不断尝试，直到可以进行 I/O。如图所示：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241112155206531.png" alt="image-20241112155206531" style="zoom:70%;" />

### 等待队列

在 Linux 驱动程序中，**可以使用等待队列（Wait Queue）来实现阻塞进程的唤醒**。它以队列为基础数据结构，与进程调度机制紧密结合，可以用来同步对系统资源的访问。

等待队列涉及的操作如下：

1. 定义等待队列头部

```c
struct wait_queue_head {
	spinlock_t		lock;
	struct list_head	head;
};
typedef struct wait_queue_head wait_queue_head_t;

wait_queue_head_t my_queue;
```

2. 初始化等待队列头部

```c
init_waitqueue_head(&my_queue);
```

DECLARE_WAIT_QUEUE_HEAD() 宏可作为定义并初始化等待队列头部的快捷方式。

```c
DECLARE_WAIT_QUEUE_HEAD(my_queue);
// 相当于
{ wait_queue_head_t my_queue; init_waitqueue_head(&my_queue); }
```

3. 定义等待队列元素

```c
DECLARE_WAITQUEUE(name, tsk); // 定义并初始化一个名为 name 的等待队列元素。
```

4. 添加/移除等待队列

```c
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
void remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
```

5. 等待事件

```c
wait_event(queue, condition); // 阻塞等待队列，直到满足条件 condition。
wait_event_interruptible(queue, condition); // 与上面的区别是可以被信号打断，上面不行。
wait_event_timeout(queue, condition, timeout); // timeout 是阻塞的超时时间，超时后不管 condition 是否满足，都返回。
wait_event_interruptible_timeout(queue, condition, timeout);
```

6. 唤醒队列

该函数会唤醒 queue 为头部的等待队列中的所有进程。

```c
void wake_up(wait_queue_head_t *queue);
void wake_up_interruptible(wait_queue_head_t *queue);
```

wake_up() 应该与 wait_event() 或 wait_event_timeout() 成对使用，而 wake_up_interruptible() 应与 wait_event_interruptible() 或 wait_event_interruptible_timeout() 成对使用。wake_up() 可唤醒处于 TASK_INTERRUPTIBLE 和 TASK_UNINTERRUPTIBLE 的进程，而 wake_up_interruptible() 只能唤醒处于 TASK_INTERRUPTIBLE 的进程。

7. 在等待队列上睡眠（已废弃）

老版本的接口，作用和 wait_event() 系列函数类似，因此使用前者即可。

```c
/*
 * These are the old interfaces to sleep waiting for an event.
 * They are racy.  DO NOT use them, use the wait_event* interfaces above.
 * We plan to remove these interfaces.
 */
extern void sleep_on(wait_queue_head_t *q);
extern long sleep_on_timeout(wait_queue_head_t *q, signed long timeout);
extern void interruptible_sleep_on(wait_queue_head_t *q);
extern long interruptible_sleep_on_timeout(wait_queue_head_t *q, signed long timeout);
```

以下是一个使用等待队列的模板，在写 I/O 的时候，检查设备是否可写。如果不可写，对于非阻塞 I/O，直接返回 -EAGAIN；对于阻塞 I/O，将进程睡眠并挂起至等待队列。

```c
static ssize_t xxx_write(struct file *file, const char *buffer, size_t count, loff_t *ppos)
{
    ...

    DECLARE_WAITQUEUE(wait, current); // 定义等待队列元素
    add_wait_queue(&xxx_wait, &wait); // 添加元素到等待队列

    // 等待设备缓冲区可写
    do
    {
        avail = device_writable(...);
        if (avail < 0)
        {
            if (file->f_flags & O_NONBLOCK)
            { // 非阻塞
                res = -EAGAIN;
                goto out;
            }

            __set_current_state(TASK_INTERRUPTIBLE); // 改变进程状态
            schedule();                              // 调度其他进程执行

            if (signal_pending(current)) // 如果是因为信号唤醒
            {

                res = -ERESTARTSYS;
                goto out;
            }
        }
    } while (avail < 0);

    // 写设备缓冲区
    device_write(...);

out:
    remove_wait_queue(&xxx_wait, &wait); // 将元素移出 xxx_wait 指引的队列
    set_current_state(TASK_RUNNING);     // 设置进程状态为 TASK_RUNNING


    return res;
}
```

这段代码里面有几个注意的点：

1. 对于非阻塞 I/O（O_NONBLOCK 标志位），设备忙时，返回 -EAGAIN。
2. 对于阻塞 I/O，设备忙时，调用 `__set_current_state(TASK_INTERRUPTIBLE)` 切换进程状态并使用 `schedule()` 调度其他进程执行。但由于使用的是 TASK_INTERRUPTIBLE，因此唤醒进程的可能是信号。需通过 `signal_pending(current)` 判断是否为信号唤醒，如果是，立即返回 -ERESTARTSYS。

DECLARE_WAITQUEUE()、add_wait_queue() 这两个动作加起来完成的效果如图所示。在 wait_queue_head_t 指向的链表上，新定义的 wait_queue 元素被插入，而这个新插入的元素绑定了一个 task_struct（当前做 xxx_write 的 current，这也是 DECLARE_WAITQUEUE 使用 current 作为参数的原因）。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241114100236690.png" alt="image-20241114100236690" style="zoom:70%;" />

### 支持阻塞操作的 globalfifo 设备驱动

TODO

## 轮询操作

在用户程序中，使用非阻塞 I/O 的程序通常会使用 select() 和 poll() 函数来进行 I/O 多路复用的操作，从而达到对设备非阻塞的忙轮询的优化，即用户将查询过程托管给内核，内核负责通知用户时机，从而达到对用户进程的优化，并且能在一个进程中同时监听多个文件描述符。

select() 和 poll() 系统调用最终会使设备驱动中的 poll() 函数被执行，在 Linux 2.5.45 内核中还引入了 epoll()，即扩展的 poll()。

### 应用程序中的轮询编程

此部分请参考博客 [https://blog.davidingplus.cn/posts/2adb5565.html](https://blog.davidingplus.cn/posts/2adb5565.html) 的第四章网络编程中 I/O 多路复用技术的部分。

### 设备驱动中的轮询编程

内核中 poll() 的定义为：

```c
// 第一个参数是 file 结构体指针，第二个是轮询表指针。
__poll_t (*poll) (struct file *, struct poll_table_struct *);
```

该 poll() 函数进行两项工作：

1. 对可能引起设备文件状态变化的等待队列调用 poll_wait() 函数。
2. 返回表示是否能对设备进行无阻塞读、写访问的掩码。

向 poll_table 注册等待队列的 poll_wait() 函数定义如下：

```c
void poll_wait(struct file *filp, wait_queue_head_t *wait_address, poll_table *p);
```

> poll_wait() 函数的名称非常容易让人产生误会，以为它和 wait_event() 等一样，会阻塞地等待某事件的发生，但其实这个函数并不会引起阻塞。poll_wait() 函数所做的工作是把当前进程添加到 p 参数指定的等待列表（poll_table）中，实际作用是让唤醒参数 wait_address 对应的等待队列可以唤醒用户层因 select()/poll()/epoll() 而睡眠的进程。

上面这段是书本上的原话，我个人不是特别理解，参考了博客 [https://blog.csdn.net/weixin_42462202/article/details/100017339](https://blog.csdn.net/weixin_42462202/article/details/100017339)，总结 poll 机制如下。

实际上 select()、poll()、epoll() 的实现都是利用等待队列机制，将执行这些函数的进程挂到每一个驱动程序的等待队列中，然后睡眠等待，直到被唤醒。

驱动程序的 poll() 函数至少会被这些系统调用给**调用两次**。

1. **第一次是 select()、poll()、epoll() 所在的进程调用驱动程序的 poll() 函数，将自己挂到驱动程序的等待队列中，通过内核函数 poll_wait() 实现。这也是驱动 poll() 函数的一个作用，当等待队列就能在满足条件 condition 的时候唤醒本进程。**

2. **第二次是 select()、poll()、epoll() 所在的进程被驱动程序唤醒，再次驱动程序的 poll() 函数，获取驱动程序满足的条件（可读或可写），返回对应的 mask，与用户进程对接起来，用户进程根据 mask 进行针对处理。**

驱动程序的 poll() 函数应该返回设备资源的可获取状态，即 POLLIN、POLLOUT、POLLPRI、POLLERR、POLLNVAL 等宏的位“或”结果。每个宏的含义都表明设备的一种状态，如 POLLIN（定义为 0x0001）意味着设备可以无阻塞地读，POLLOUT（定义为0x0004）意味着设备可以无阻塞地写。

因此，结合上述，在驱动中使用 poll() 函数的模板如下：

```c
static unsigned int xxx_poll(struct file *filp, poll_table *wait)
{
    unsigned int mask = 0;
    struct xxx_dev *dev = filp->private_data; // 获得设备结构体指针

    ...

    // 第一次被调用时，将调用 select()、poll()、epoll() 的进程加入驱动的等待队列中。
    poll_wait(filp, &dev->r_wait, wait); // 加入读等待队列
    poll_wait(filp, &dev->w_wait, wait); // 加入写等待队列

    // 第二次被调用时，如果条件满足，返回状态，之后 poll() 会返回。
    if (...)                         // 可读
        mask |= POLLIN | POLLRDNORM; // 标示数据可获得（对用户可读）

    if (...)                          // 可写
        mask |= POLLOUT | POLLWRNORM; // 标示数据可写入

    ...


    return mask;
}
```

> 代码中看到了状态 POLLIN 和 POLLRDNORM，这二者都是表示可读。区别在于 POLLIN 适用于所有的可读数据情况，不仅包括普通数据，还包括优先级数据，而 POLLRDNORM 只适用于普通数据的可读情况，不包括任何优先级数据。
>
> POLLOUT 和 POLLWRNORM 同理。

### 支持轮询操作的 globalfifo 设备驱动

TODO

## 小结

阻塞与非阻塞访问是 I/O 操作的两种不同模式，前者在暂时不可进行I/O操作时会让进程睡眠，后者则不然。

在设备驱动中阻塞 I/O 一般基于等待队列或者基于等待队列的其他 Linux 内核 API 来实现，等待队列可用于同步驱动中事件发生的先后顺序。使用非阻塞 I/O 的应用程序也可借助轮询函数来查询设备是否能立即被访问，用户空间调用 select()、poll() 或 epoll() 接口，设备驱动提供 poll() 函数。设备驱动的 poll() 本身不会阻塞，但是与 select()、poll() 和 epoll() 相关的系统调用则会阻塞地等待至少一个文件描述符集合，直到可访问或超时。

# 设备驱动中的异步通知与异步 I/O

异步通知的意思是：一旦设备就绪，则主动通知应用程序，这样应用程序根本就不需要查询设备状态，这一点非常类似于硬件上“中断”的概念，比较准确的称谓是“**信号驱动的异步 I/O**”。**信号是在软件层次上对中断机制的一种模拟。**在原理上，一个进程收到一个信号与处理器收到一个中断请求可以说是一样的。信号是异步的，一个进程不必通过任何操作来等待信号的到达，事实上，进程也不知道信号到底什么时候到达。

**阻塞 I/O 意味着一直等待设备可访问后再访问，非阻塞 I/O 中使用 poll() 意味着查询设备是否可访问，而异步通知则意味着设备通知用户自身可访问，之后用户再进行 I/O 处理。**

三种方式的示意图如下：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241118152443949.png" alt="image-20241118152443949" style="zoom:75%;" />

## 异步通知编程

### 关于信号

在 Linux 中，异步通信使用信号实现。同时，信号也是 Linux 进程间通信 IPC 的一种方式。Linux 常用信号如下表：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241118154442574.png" alt="image-20241118154442574" style="zoom:75%;" />

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241118154452629.png" alt="image-20241118154452629" style="zoom:75%;" />

除了 SIGSTOP 和 SIGKILL 两个信号外，进程能够忽略或捕获其他的全部信号。一个信号被捕获的意思是当一个信号到达时有相应的代码处理它。如果一个信号没有被这个进程所捕获，内核将采用默认行为处理。

### 信号的接收

在用户程序当中进行捕获信号。可使用 signal() 或 sigaction() 函数捕获信号。这两个函数的具体使用见博客 [https://blog.davidingplus.cn/posts/2adb5565.html](https://blog.davidingplus.cn/posts/2adb5565.html) 的第二章进程间通信的信号部分。

使用信号实现异步通知的用户层代码示例如下：

```cpp
#include <iostream>

#include <unistd.h>
#include <signal.h>
#include <fcntl.h>


#define MAX_SIZE BUFSIZ


void doIt(int num)
{
    char data[MAX_SIZE] = {0};

    // 读取并输出 STDIN_FILENO 上的输入
    int len = read(STDIN_FILENO, &data, MAX_SIZE);

    std::cout << "input data: " << data << std::endl;
}


int main()
{
    // 启动信号驱动机制
    struct sigaction act;
    act.sa_flags = 0;
    act.sa_handler = doIt;
    sigemptyset(&act.sa_mask);

    sigaction(SIGIO, &act, nullptr);

    // TODO 下面代码的含义是让当前进程设置为接受 SIGIO 信号，并且告诉内核是通过异步的方式接受。在这个程序中异步的感观不明显，但确实是这样用的。想办法让其感观明显一点。

    // 设置将接收 SIGIO 和 SIGURG 信号的进程 id 或进程组 id
    fcntl(STDIN_FILENO, F_SETOWN, getpid());
    // 修改为异步通知模式
    int flag = fcntl(STDIN_FILENO, F_GETFL);
    fcntl(STDIN_FILENO, F_SETFL, flag | FASYNC);

    // 最后进入一个死循环，仅为保持进程不终止，如果程序中没有这个死循会立即执行完毕
    for (;;)
    {
    }


    return 0;
}
```

由此可见，为了在用户空间中处理一个设备释放的信号，需要完成 3 项工作：

1. **通过 F_SETOWN IO 控制命令设置设备文件的拥有者为本进程，这样从设备驱动发出的信号才能被本进程接收到。**
2. **通过 F_SETFL IO 控制命令设置设备文件以支持 FASYNC，即异步通知模式。**
3. **通过 signal() 或者 sigaction() 函数连接信号和信号处理函数。**

### 信号的释放

在设备驱动和应用程序的异步通知交互中，仅仅在应用程序端捕获信号是不够的，因为信号的源头在设备驱动端。因此，应该在合适的时机让设备驱动释放信号，**在设备驱动程序中增加信号释放的相关代码**。

与用户程序对应的，驱动程序也涉及 3 项工作：

1. **支持 F_SETOWN 命令，能在这个控制命令处理中设置 filp->f_owner 为对应进程 ID。但此项工作已由内核完成，设备驱动无须处理。**
2. **支持 F_SETFL 命令的处理，每当 FASYNC 标志改变时，驱动程序中的 fasync() 函数将得以执行。因此，驱动中应该实现 fasync() 函数。**
3. **在设备资源可获得时，调用 kill_fasync() 函数激发相应的信号。**

这些与用户层的 3 项工作是一一对应的，关系图如下：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241120095605894.png" alt="image-20241120095605894" style="zoom:80%;" />

设备驱动中的异步通知编程涉及到一个数据结构和两个函数，具体如下：

1. 结构体 fasync_struct。

```c
struct fasync_struct {
	rwlock_t		fa_lock;
	int			magic;
	int			fa_fd;
	struct fasync_struct	*fa_next; /* singly linked list */
	struct file		*fa_file;
	struct rcu_head		fa_rcu;
};
```

2. 处理 FASYNC 标志变更的函数。

fasync_helper() 的作用是**将一个 fasync_struct 的对象注册进内核**，应用层执行 fcntl(fd, F_SETFL, oflags | FASYNC) 时会回调驱动的 fops.fasync()，所以通常将 fasync_helper() 放到 fasync() 的实现中。

```c
int fasync_helper(int fd, struct file *filp, int mode, struct fasync_struct **fa);
```

3. 产生信号用的函数。

```c
void kill_fasync(struct fasync_struct **fa, int sig, int band);
```

## 支持异步通知的 globalfifo 驱动

TODO

## 异步 IO

### 用户层 GNU C 库 AIO

Linux 中最常用的输入/输出（I/O）模型是同步 I/O。当请求发出之后，应用程序就会阻塞，直到请求满足为止。这是一种很好的解决方案，调用应用程序在等待 I/O 请求完成时不需要占用 CPU。但是在许多应用场景中，I/O 请求可能需要与 CPU 消耗产生交叠，以充分利用 CPU 和 I/O 提高吞吐率。

异步 I/O 的时序图如下。应用程序发起 I/O 动作以后，并不等待 I/O 结束，而是接着执行。它要么过一段时间来查询之前的 I/O 请求完成情况，要么 I/O 请求完成后自动被调用与 I/O 完成绑定的回调函数。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241120111926095.png" alt="image-20241120111926095" style="zoom:75%;" />

Linux 的 AIO 有多种实现。其中一种是**在用户空间的 glibc 库中实现的**。**它本质上是借用了多线程模型，用开启新的线程以同步的方法来做 I/O，新的 AIO 辅助线程与发起 AIO 的线程以 pthread_cond_signal() 的形式进行线程间的同步。**

其主要接口如下：

1. aio_read()

aio_read() 函数请求对一个有效的文件描述符进行异步读操作。这个文件描述符可以表示一个文件、套接字，甚至管道。

此函数在请求排队后就会立即返回，尽管读操作并未完成。如果执行成功，返回 0；如果出现错误，返回 -1，并设置 errno。

参数 aiocb（AIO I/O Control Block）结构体包含了传输的所有信息，以及为 AIO 操作准备的用户空间缓冲区。在产生 I/O 完成通知时，aiocb 结构就被用来唯一标识所完成的 I/O 操作。

```c
int aio_read(struct aiocb *aiocbp);
```

更进一步的，aiocb 结构体的定义如下：

```c
struct aiocb
{
    int aio_fildes;               // 文件描述符
    __off_t aio_offset;           // 文件偏移
    volatile void *aio_buf;       // 缓冲区地址
    size_t aio_nbytes;            // 传输的数据长度
    int aio_reqprio;              // 请求优先级
    struct sigevent aio_sigevent; // 通知方法
    int aio_lio_opcode;           // 仅被 lio_listio() 函数使用

	...
};
```

2. aio_write()

同 aio_read()，aio_write() 进行一个异步写操作。同样在请求排队后就会立即返回，返回值同上。

```c
int aio_write(struct aiocb *aiocbp);
```

3. aio_error()

aio_error() 函数用于确认请求的状态。

```c
int aio_error(const struct aiocb *aiocbp);
```

这个函数可以返回以下内容：

- EINPROGRESS：说明请求尚未完成。
- ECANCELED：说明请求被应用程序取消了。
- -1：说明发生了错误，具体错误原因由 errno 记录。

4. aio_return()

异步 I/O 和同步阻塞 I/O 方式之间的一个区别是不能立即访问这个函数的返回状态，因为异步 I/O 并没有阻塞在 aio_read() 调用上。在标准的同步阻塞 read() 调用中，返回状态是在该函数返回时提供的。但是在异步 I/O 中，我们要使用 aio_return() 函数，手动获取返回值。

```c
ssize_t aio_return(struct aiocb *aiocbp);
```

在实际使用的时候，只有在 aio_error() 调用确定请求已经完成（可能成功，也可能发生了错误）之后，才会调用这个函数。aio_return() 的返回值就等价于同步情况中 read() 或 write() 系统调用的返回值，即所传输的字节数。如果发生错误，返回值为负数。

下面是一个例子，展示了上面 API 的使用方法。注意使用的时候链接系统库 `rt`（见 man 文档）。

```cpp
#include <iostream>
#include <cstring>

#include <aio.h>
#include <fcntl.h>


#define PERROR(func)  \
    do                \
    {                 \
        perror(func); \
        return -1;    \
    } while (0)


int main()
{
    FILE *pFile = fopen("file.txt", "w");
    if (!pFile) PERROR("fopen");

    fputs("hello globalfifo module.", pFile);

    fclose(pFile);

    // 使用 AIO 读

    int fd = open("file.txt", O_RDONLY);
    if (-1 == fd) PERROR("open");

    struct aiocb aio;
    bzero(&aio, sizeof(aio));

    aio.aio_buf = malloc(BUFSIZ + 1);
    if (!aio.aio_buf) PERROR("malloc");

    aio.aio_fildes = fd;
    aio.aio_nbytes = BUFSIZ;
    aio.aio_offset = 0;

    int res = aio_read(&aio);
    if (-1 == res) PERROR("aio_read");

    while (aio_error(&aio) == EINPROGRESS) continue;

    res = aio_return(&aio);
    if (res < 0) PERROR("aio_return");

    std::cout << "res: " << res << std::endl;
    std::cout << "aio_read: " << (char *)aio.aio_buf << std::endl;

    remove("file.txt");


    return 0;
}
```

5. aio_suspend()

用户可以使用 aio_suspend() 函数来阻塞调用进程，直到异步请求完成为止。用户可以在参数中提供一个 aiocb 引用数组，其中任何一个完成都会导致 aio_suspend() 返回。

注意，第一个参数是**存放 aiocb 结构体指针的数组**，第二个参数是数组的大小，第三个参数是阻塞超时时长（nullptr 表示阻塞，非阻塞给超时时长）。

```c
int aio_suspend(const struct aiocb *const aiocb_list[], int nitems, const struct timespec *timeout);
```

当然，在 glibc 实现的 AIO 中，除了上述同步的等待方式以外，也可以使用信号或者回调机制来异步地标明 AIO 的完成。

经过修改后的示例程序如下：

```cpp
#include <iostream>
#include <cstring>

#include <aio.h>
#include <fcntl.h>


#define PERROR(func)  \
    do                \
    {                 \
        perror(func); \
        return -1;    \
    } while (0)


int main()
{
    FILE *pFile = fopen("file.txt", "w");
    if (!pFile) PERROR("fopen");

    fputs("hello globalfifo module.", pFile);

    fclose(pFile);

    int fd = open("file.txt", O_RDONLY);
    if (-1 == fd) PERROR("open");

    struct aiocb aio;
    std::memset(&aio, 0, sizeof(aio));

    struct aiocb *aioArr[1] = {&aio};

    aio.aio_buf = malloc(BUFSIZ + 1);
    if (!aio.aio_buf) PERROR("malloc");

    aio.aio_fildes = fd;
    aio.aio_nbytes = BUFSIZ;
    aio.aio_offset = 0;

    int res = aio_read(&aio);
    if (-1 == res) PERROR("aio_read");

    res = aio_suspend(aioArr, sizeof(aioArr) / sizeof(aioArr[0]), nullptr);
    if (-1 == res) PERROR("aio_suspend");

    res = aio_return(&aio);
    if (-1 == res) PERROR("aio_return");

    std::cout << "res: " << res << std::endl;
    std::cout << "aio_read: " << (char *)aio.aio_buf << std::endl;

    remove("file.txt");


    return 0;
}
```

6. aio_cancel()

aio_cancel() 函数允许用户取消对某个文件描述符执行的一个或所有 I/O 请求。

如果是**取消一个请求**，需提供 fd 和 aiocb 指针。如果请求被成功取消，会返回 AIO_CANCELED。如果请求已完成即取消失败，会返回 AIO_NOTCANCELED。

如果是取消对某个 fd 的**所有请求**。需提供这个 fd，并将 aiocb 参数设置为 NULL。如果所有的请求都取消了，会返回 AIO_CANCELED；如果至少有一个请求没有被取消，会返回 AIO_NOT_CANCELED；如果没有一个请求可以被取消，那么这个函数就会返回 AIO_ALLDONE。然后，可以使用 aio_error() 来验证每个 AIO 请求。如果某请求已经被取消了，那么 aio_error() 就会返回 -1，并且 errno 会被设置为 ECANCELED。

```c
int aio_cancel(int fd, struct aiocb *aiocbp);
```

7. lio_listio()

**lio_listio() 函数可用于同时发起多个传输，使得用户可以在一个系统调用中启动大量的 I/O 操作。**

mode 参数可以是 LIO_WAIT 或 LIO_NOWAIT。LIO_WAIT 会阻塞这个调用，直到所有的 I/O 都完成为止。但是若是 LIO_NOWAIT 模型，在 I/O 操作进行排队之后，该函数就会返回。list 是一个 aiocb 引用的指针的列表，最大元素的个数是由 nitems 定义的。如果 list 的元素为 nullptr，lio_listio() 会将其忽略。最后一个参数 sevp 暂时不用考虑，直接设置为 nullptr 即可。

```c
int lio_listio(int mode, struct aiocb *const aiocb_list[], int nitems, struct sigevent *sevp);
```

当使用这个函数的时候，aiocb 结构体中的成员 aio_lio_opcode 需要被设置。因为批量执行 I/O 的缘故，没有办法单独调用 aio_read() 或者 aio_write() 这些，因此需要设置标志以后托管给 lio_listio() 执行。aio_lio_opcode 值如下表：

|    值     |      含义      |
| :-------: | :------------: |
| LIO_READ  | 发起异步读操作 |
| LIO_WRITE | 发起异步写操作 |
|  LIO_NOP  |  忽略该 aiocb  |

经过修改后的示例程序如下：

```cpp
#include <iostream>
#include <cstring>

#include <aio.h>
#include <fcntl.h>


#define ERR_EXIT(func) \
    do                 \
    {                  \
        perror(func);  \
        exit(1);       \
    } while (0)


int main()
{
    FILE *pFile = fopen("file.txt", "w");
    if (!pFile) ERR_EXIT("fopen");

    fputs("hello globalfifo module.", pFile);

    fclose(pFile);

    int fd = open("file.txt", O_RDONLY);
    if (-1 == fd) ERR_EXIT("open");

    struct aiocb aio;
    std::memset(&aio, 0, sizeof(aio));

    aio.aio_buf = malloc(BUFSIZ + 1);
    if (!aio.aio_buf) ERR_EXIT("malloc");

    aio.aio_fildes = fd;
    aio.aio_nbytes = BUFSIZ;
    aio.aio_offset = 0;
    aio.aio_lio_opcode = LIO_READ; // 使用 lio_listio() 需要额外添加的一项。

    struct aiocb *aioArr[5] = {nullptr};
    aioArr[3] = &aio;

    int res = lio_listio(LIO_WAIT, aioArr, sizeof(aioArr) / sizeof(aioArr[0]), nullptr);
    if (-1 == res) ERR_EXIT("lio_listio");

    res = aio_return(&aio);
    if (-1 == res) ERR_EXIT("aio_return");

    std::cout << "res: " << res << std::endl;
    std::cout << "aio_read: " << (char *)aio.aio_buf << std::endl;

    remove("file.txt");


    return 0;
}
```

### 内核 AIO 与 libaio

Linux AIO 也可以**由内核空间实现**。异步 I/O 是 Linux 2.6 以后版本内核的一个标准特性。对于块设备而言，AIO 可以一次性发出大量的 read/write 调用并且通过通用块层的 I/O 调度来获得更好的性能，用户程序也可以减少过多的同步负载，还可以在业务逻辑中更灵活地进行并发控制和负载均衡。相较于 glibc 的用户空间多线程同步等实现也减少了线程的负载和上下文切换等。对于网络设备而言，在 socket 层面上，也可以使用 AIO，让 CPU 和网卡的收发动作充分交叠以改善吞吐性能。

在用户空间中，需要结合 libaio 库进行内核 AIO 的系统调用。首先需要安装 libaio 库：

```bash
sudo apt install libaio-dev
```

libaio 提供的内核 AIO 的 API 主要包括：

```c
int io_setup(int maxevents, io_context_t *ctxp);
int io_destroy(io_context_t ctx);
int io_submit(io_context_t ctx, long nr, struct iocb *ios[]);
int io_cancel(io_context_t ctx, struct iocb *iocb, struct io_event *evt);
int io_getevents(io_context_t ctx, long min_nr, long nr, struct io_event *events, struct timespec *timeout);
void io_set_callback(struct iocb *iocb, io_callback_t cb);
void io_prep_pread(struct iocb *iocb, int fd, void *buf, size_t count, long long offset);
void io_prep_pwrite(struct iocb *iocb, int fd, void *buf, size_t count, long long offset);
void io_prep_preadv(struct iocb *iocb, int fd, const struct iovec *iov, int iovcnt, long long offset);
void io_prep_pwritev(struct iocb *iocb, int fd, const struct iovec *iov, int iovcnt, long long offset);
```

AIO 的读写请求都用 io_submit() 下发。下发前通过 io_prep_pwrite() 和 io_prep_pread() 生成 iocb 的结构体。作为 io_submit() 的参数。这个结构体指定了读写类型、起始地址、长度和设备标志符等信息。读写请求下发之后，使用 io_getevents() 函数等待 I/O 完成事件。io_set_callback() 则可设置一个 AIO 完成的回调函数。

示例程序见：[https://github.com/DavidingPlus/linux-kernel-learning/tree/globalfifo/snippet/LibAioTest](https://github.com/DavidingPlus/linux-kernel-learning/tree/globalfifo/snippet/LibAioTest)

### AIO 与设备驱动

在 Linux 4.0 版本下，用户空间调用 io_submit() 后，对应于用户传递的每一个 iocb 结构，内核会生成一个与之对应的 kiocb 结构。file_operations 包含 3 个与 AIO 相关的成员函数。

io_submit() 系统调用间接引起了 file_operations 中的 aio_read() 和 aio_write() 的调用。

```c
ssize_t (*aio_read) (struct kiocb *iocb, const struct iovec *iov, unsigned long nr_segs, loff_t pos);
ssize_t (*aio_write) (struct kiocb *iocb, const struct iovec *iov, unsigned long nr_segs, loff_t pos);
int (*aio_fsync) (struct kiocb *iocb, int datasync);
```

Linux 5.0 版本之后的这三个接口可能已经改名（或删除并重新实现了一套逻辑），可能的对应关系如下。暂不用深究。

```c
ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
int (*iopoll)(struct kiocb *kiocb, bool spin);
```

**AIO 一般由内核空间的通用代码处理，对于块设备和网络设备而言，一般在Linux核心层的代码已经解决。字符设备驱动一般不需要实现 AIO 支持。**Linux 内核中对字符设备驱动实现 AIO 的特例包括 drivers/char/mem.c 里实现的 null、zero 等，由于 zero 这样的虚拟设备其实也不存在在要去读的时候读不到东西的情况，所以 aio_read_zero() 本质上也不包含异步操作。

## 小结

Linux 中的异步 I/O 使得应用程序在等待 I/O 操作的同时进行其他操作。

使用信号可以实现设备驱动与用户程序之间的异步通知。总体而言，设备驱动和用户空间要分别完成 3 项对应的工作，用户空间设置文件的拥有者、FASYNC 标志及捕获信号。内核空间响应对文件的拥有者、FASYNC 标志的设置并在资源可获得时释放信号。

Linux 2.6 以后的内核包含对 AIO 的支持，它为用户空间提供了统一的异步 I/O 接口。另外，glibc 也提供了一个不依赖于内核的，位于用户空间的 AIO 支持。

# 中断与时钟

**由于中断服务程序的执行并不存在于进程上下文中，所以要求中断服务程序的时间要尽量短。**故 Linux 在中断处理中引入了**顶半部和底半部分离**的机制。另外，内核对时钟的处理也采用中断方式，而内核软件定时器最终依赖于时钟中断。

## 中断与定时器

中断是指 CPU 在执行程序的过程中，出现了某些突发事件急待处理，CPU 必须暂停当前程序的执行，转去处理突发事件，处理完毕后又返回原程序被中断的位置继续执行。

根据中断的来源，中断可分为**内部中断和外部中断**。内部中断的中断源来自 CPU 内部（软件中断指令、溢出、除法错误，操作系统从用户态切换到内核态需借助 CPU 内部的软件中断等）。外部中断的中断源来自 CPU 外部，由外设提出请求。

根据中断是否可以屏蔽，中断可分为**可屏蔽中断与不可屏蔽中断**（NMI）。可屏蔽中断可以通过设置中断控制器寄存器等方法被屏蔽，屏蔽后，该中断不再得到响应。不可屏蔽中断不能被屏蔽。

根据中断入口跳转方法的不同，中断可分为**向量中断和非向量中断**。采用向量中断的 CPU 通常为不同的中断分配不同的中断号。当检测到某中断号的中断到来后，就自动跳转到与该中断号对应的地址执行。不同中断号的中断有不同的入口地址。非向量中断的多个中断共享一个入口地址，进入该入口地址后，再通过软件判断中断标志来识别具体是哪个中断。也就是说，**向量中断由硬件提供中断服务程序入口地址，非向量中断由软件提供中断服务程序入口地址。**

## 中断处理程序架构

设备的中断会打断内核进程中的正常调度和运行，系统对更高吞吐率的追求势必要求中断服务程序尽量短小精悍。但在大多数真实的系统中，当中断到来时，要完成的工作往往并不会是短小的，它可能要进行较大量的耗时处理。

为了在**中断执行时间尽量短和中断处理需完成的工作尽量大**之间找到一个平衡点，Linux 将中断处理程序分解为两个半部：顶半部（Top Half）和底半部（Bottom Half）。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241125093035072.png" alt="image-20241125093035072" style="zoom:75%;" />

**顶半部用于完成尽量少的比较紧急的功能。**它往往只是简单地读取寄存器中的中断状态，并在清除中断标志后就进行“登记中断”的工作。“登记中断”意味着将底半部处理程序挂到该设备的底半部执行队列中去。这样，顶半部执行的速度就会很快，从而可以服务更多的中断请求。

因此，**中断处理工作的重心就落在了底半部的头上，需用它来完成中断事件的绝大多数任务。**底半部几乎做了中断处理程序所有的事情，而且可以被新的中断打断，这也是底半部和顶半部的最大不同。顶半部往往被设计成不可中断。底半部相对来说并不是非常紧急的，而且相对比较耗时，不在硬件中断服务程序中执行。

但设计是这样，现实是灵活的。如果中断要处理的工作本身很少，完全可以直接在顶半部中全部完成。**真正的硬件中断服务程序都应该 尽量短。**许多操作系统都提供了中断上下文和非中断上下文相结合的机制，将中断的耗时工作保留到非中断上下文去执行。

在 Linux 中，查看 `/proc/interrupts` 文件可以获得系统中断的统计信息，知道每个中断号在 CPU 上发生的次数。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241125094233332.png" alt="image-20241125094233332" style="zoom:60%;" />

## 中断编程

### 申请和释放中断

1. 申请 irq

申请 irq 使用接口 request_irq()。

```c
// irq 是要申请的硬件中断号。
// handler 是向系统登记的顶半部的中断处理函数，是一个回调函数。中断发生时，系统调用这个函数，dev 参数将被传递给它。
// flags 是中断处理的属性，可以指定中断的触发方式以及处理方式。在触发方式上，可以是 IRQF_TRIGGER_RISING、IRQF_TRIGGER_FALLING、IRQF_TRIGGER_HIGH、IRQF_TRIGGER_LOW 等。在处理方式上，若设置了 IRQF_SHARED，则表示多个设备共享中断。
// 此函数返回 0 表示成功，返回 -EINVAL 表示中断号无效或处理函数指针为 NULL，返回 -EBUSY 表示中断已被占用且不能共享。
int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev);
```

当然，还有另一个接口 devm_request_irq()。与 request_irq() 的区别是 `devm_` 开头的 API 申请的是内核 managed 的资源，一般不需要在出错处理和 remove() 接口里再显式的释放。有点类似 Java 的垃圾回收机制。

顶半部 hander 类型 irq_handler_t 的定义为：

```c
typedef irqreturn_t (*irq_handler_t)(int, void *);

enum irqreturn {
	IRQ_NONE		= (0 << 0),
	IRQ_HANDLED		= (1 << 0),
	IRQ_WAKE_THREAD		= (1 << 1),
};

typedef enum irqreturn irqreturn_t;
```

2. 释放 irq

释放 irq 使用接口 free_irq()。

```c
const void *free_irq(unsigned int irq, void *dev);
```

### 使能和屏蔽中断

下面函数用于使能和屏蔽一个中断源。

如果在 n 号中断的顶半部调用 disable_irq(n)，会引起系统的死锁，这种情况下，只能调用 disable_irq_nosync(n)。

```c
void enable_irq(unsigned int irq);
void disable_irq(unsigned int irq);
void disable_irq_nosync(unsigned int irq); // 与上面的区别在于本函数会立即返回，disable_irq() 会等待目前的中断完成。
```

下面函数用于屏蔽本 CPU 内的所有中断。

区别在于 local_irq_save(flags) 会将目前的中断状态保存在 flags 中（注意 flags 为 unsigned long 类型，被直接传递，而不是通过指针）。后者禁止中断而不保存状态。

```c
#define local_irq_disable()                      \
    do                                           \
    {                                            \
        bool was_disabled = raw_irqs_disabled(); \
        raw_local_irq_disable();                 \
        if (!was_disabled)                       \
            trace_hardirqs_off();                \
    } while (0)

#define local_irq_save(flags)                \
    do                                       \
    {                                        \
        raw_local_irq_save(flags);           \
        if (!raw_irqs_disabled_flags(flags)) \
            trace_hardirqs_off();            \
    } while (0)
```

与上面对应的使能函数如下。

```c
#define local_irq_enable()      \
    do                          \
    {                           \
        trace_hardirqs_on();    \
        raw_local_irq_enable(); \
    } while (0)

#define local_irq_restore(flags)             \
    do                                       \
    {                                        \
        if (!raw_irqs_disabled_flags(flags)) \
            trace_hardirqs_on();             \
        raw_local_irq_restore(flags);        \
    } while (0)
```

**前缀 `local_` 表示作用范围在本 CPU 内。**

### 底半部机制

Linux 底半部机制主要涉及 tasklet、工作队列、软中断和线程化 irq。

#### tasklet

**tasklet 的执行上下文是软中断，执行时机通常是顶半部返回的时候。**我们只需定义 tasklet 及其处理函数，并关联二者即可。

```c
void my_tasklet_func(unsigned long);                // 定义一个处理函数
DECLARE_TASKLET(my_tasklet, my_tasklet_func, data); // 定义一个 tasklet 结构 my_tasklet，与 my_tasklet_func(data) 函数相关联
```

DECLARE_TASKLET(my_tasklet，my_tasklet_func，data) 实现了定义名称为 my_tasklet 的 tasklet，并将其与 my_tasklet_func() 这个函数绑定，并且传入函数的参数为 data。

需要调度 tasklet 时使用 tasklet_schedule() 函数就能使系统在适当的时候（例如在顶半部的代码中）进行调度运行。

```c
tasklet_schedule(&my_tasklet);
```

使用 tasklet 作为底半部处理中断的设备驱动程序代码模板如下：

```c
// 定义 tasklet 和底半部函数并将它们关联
void xxx_do_tasklet(unsigned long);
DECLARE_TASKLET(xxx_tasklet, xxx_do_tasklet, 0);

// 中断处理底半部
void xxx_do_tasklet(unsigned long)
{
    ...
}

// 中断处理顶半部
irqreturn_t xxx_interrupt(int irq, void *dev_id)
{
    ... 

    tasklet_schedule(&xxx_tasklet);

    ...
}

// 设备驱动模块加载函数
int __init xxx_init(void)
{
    ...

    // 申请中断
    result = request_irq(xxx_irq, xxx_interrupt, 0, "xxx", NULL);

    ...

    return IRQ_HANDLED;
}

// 设备驱动模块卸载函数
void __exit xxx_exit(void)
{
    ...

    // 释放中断
    free_irq(xxx_irq, xxx_interrupt);

    ...
}
```

上述程序在模块加载函数中申请中断，并在模块卸载函数中释放它。对应于 xxx_irq 的顶半部的中断处理程序被设置为 xxx_interrupt() 函数。在这个函数中，tasklet_schedule(&xxx_tasklet) 调度 tasklet 的函数 xxx_do_tasklet()，会在适当的时候执行，用于处理底半部。

#### 工作队列

工作队列的使用方法和 tasklet 很相似，但**工作队列的执行上下文是内核线程，因此可以调度和睡眠**。

与 tasklet 类似，下面用于定义工作队列和底半部执行函数：

```c
struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};

struct work_struct my_wq; // 定义一个工作队列
void my_wq_func(struct work_struct *work); // 定义一个处理函数
```

通过 INIT_WORK() 初始化工作队列并与处理函数绑定。

```c
INIT_WORK(&my_wq, my_wq_func); // 初始化工作队列并将其与处理函数绑定
```

与 tasklet_schedule() 对应的用于调度工作队列执行的函数为 schedule_work()。

```c
schedule_work(&my_wq); // 调度工作队列执行
```

同理可总结代码模板如下：

```c
// 定义工作队列和关联函数
struct work_struct xxx_wq;
void xxx_do_work(struct work_struct *work);

// 中断处理底半部
void xxx_do_work(struct work_struct *work)
{
    ...
}

// 中断处理顶半部
irqreturn_t xxx_interrupt(int irq, void *dev_id)
{
    ...

    schedule_work(&xxx_wq);
    
    ...

    return IRQ_HANDLED;
}

// 设备驱动模块加载函数
int xxx_init(void)
{
    ...

    // 申请中断
    result = request_irq(xxx_irq, xxx_interrupt, 0, "xxx", NULL);

    ...

    // 初始化工作队列
    INIT_WORK(&xxx_wq, xxx_do_work);

    ...
}

// 设备驱动模块卸载函数
void xxx_exit(void)
{
    ...

    // 释放中断
    free_irq(xxx_irq, xxx_interrupt);

    ...
}
```

与 tasklet 不同的是，在驱动模块加载函数中加入了初始化工作队列的部分。

工作队列早期的实现是在每个 CPU 核上创建一个 worker 内核线程，所有在这个核上调度的工作都在该 worker 线程中执行，其并发性显然差强人意。在 Linux 2.6 以后，转而实现了 Concurrency-managed workqueues，简称 cmwq。它会自动维护工作队列的线程池以提高并发性，同时保持了 API 的向后兼容。

#### 软中断

**软中断（softirq）也是一种传统的底半部处理机制，执行时机通常是顶半部返回的时候。tasklet 是基于软中断实现的，因此也运行于软中断上下文。**

在 Linux 内核中，用 softirq_action 结构体表示一个软中断，包含软中断处理函数指针和传递给该函数的参数。使用 open_softirq() 函数可以注册软中断对应的处理函数，而 raise_softirq() 函数可以触发一个软中断。

```c
struct softirq_action
{
    void (*action)(struct softirq_action *);
};

void open_softirq(int nr, void (*action)(struct softirq_action *));
void raise_softirq(unsigned int nr);
```

**软中断和 tasklet 运行于软中断上下文，仍属于原子上下文的一种，而工作队列则运行于进程上下文，即内核线程。故在软中断和 tasklet 处理函数中不允许睡眠，在工作队列处理函数中允许睡眠。**

内核中用于禁止和使能软中断及 tasklet 底半部机制的函数如下：

```c
void local_bh_disable(void);
void local_bh_enable(void)
```

内核中采用 softirq 的地方包括 HI_SOFTIRQ、TIMER_SOFTIRQ、NET_TX_SOFTIRQ、NET_RX_SOFTIRQ、SCSI_SOFTIRQ、TASKLET_SOFTIRQ 等。一般来说，驱动的编写者不会也不宜直接使用 softirq。因此，软中断稍微了解即可。

总结一下硬中断、软中断和信号的区别。**硬中断是外部设备对 CPU 的中断，软中断（softirq）是中断底半部的一种处理机制，而信号则是由内核或其他进程对某个进程的中断。在涉及系统调用的场合，人们也常说通过软中断陷入内核（其实就是计组里面背的那个软中断），此时软中断的概念是指由软件指令引发的中断，和这里的 softirq 是两个完全不同的概念。**

软中断以及基于软中断的 tasklet 如果在某段时间内大量出现的话，内核会把后续软中断放入 ksoftirqd 内核线程中执行。总的来说，**中断优先级高于软中断，软中断又高于任何一个线程**。软中断适度线程化，可以缓解高负载情况下系统的响应。

#### threaded_irq

在 Linux 内核中，除了可以通过 request_irq()、devm_request_irq() 申请中断以外，还可以通过 request_threaded_irq() 和 devm_request_threaded_irq() 申请。

这两个函数比上面函数多一个参数 thread_fn。用这两个 API 申请中断的时候，内核会为相应的中断号分配一个对应的内核线程，来执行一些东西。这个线程只针对这个中断号，如果其他中断也通过 request_threaded_irq() 申请，会得到新的内核线程。

```c
int request_threaded_irq(unsigned int irq, irq_handler_t handler, irq_handler_t thread_fn, unsigned long flags, const char *name, void *dev);

int devm_request_threaded_irq(struct device *dev, unsigned int irq, irq_handler_t handler, irq_handler_t thread_fn, unsigned long irqflags, const char *devname, void *dev_id);
```

**参数 handler 对应的函数执行于中断上下文，thread_fn 参数对应的函数则执行于内核线程。**如果顶半部 handler 结束的时候，返回值是 IRQ_WAKE_THREAD，内核会调度对应线程执行底半部 thread_fn 对应的函数。

我个人理解而言，handler 是处理顶半部机制的函数，thread_fn 是处理底半部机制的函数。二者要求传入的都是回调函数的函数指针。与工作队列相比，尽管二者的底层实现可能不一样，但给使用者的感受就是 threaded_irq 是对用户封装了一层的工作队列。

request_threaded_irq() 和 devm_request_threaded_irq() 支持在 irqflags 中设置 IRQF_ONESHOT 标记， 这样内核会自动帮助我们在中断上下文中屏蔽对应的中断号。在内核调度 thread_fn 执行后，重新使能该中断号。对于我们无法在上半部清除中断的情况，IRQF_ONESHOT 特别有用，避免了中断服务程序一退出，中断就洪泛的情况。

handler 参数可以设置为 NULL，这样内核会使用默认的 irq_default_primary_handler() 代替 handler，并会使用 IRQF_ONESHOT 标记。

```c
/*
 * Default primary interrupt handler for threaded interrupts. Is
 * assigned as primary handler when request_threaded_irq is called
 * with handler == NULL. Useful for oneshot interrupts.
 */
static irqreturn_t irq_default_primary_handler(int irq, void *dev_id)
{
	return IRQ_WAKE_THREAD;
}
```

## 中断共享

多个设备共享一根硬件中断线的情况在实际的硬件系统中广泛存在。Linux 也支持这种共享方法。

1. 共享中断的多个设备在申请中断时，都应该使用 IRQF_SHARED 标志，而且一个设备以 IRQF_SHARED 申请某中断成功的前提是该中断未被申请，或该中断虽然被申请了，但是之前申请该中断的所有设备也都以 IRQF_SHARED 标志申请该中断。
2. 尽管内核模块可访问的全局地址都可以作为 request_irq(..., void *dev) 的最后一个参数 dev，但是设备结构体指针显然是可传入的最佳参数。
3. 在中断到来时，会遍历执行共享此中断的所有中断处理程序，直到某一个函数返回 IRQ_HANDLED。在中断处理程序顶半部中，应根据硬件寄存器中的信息比照传入的 dev 参数迅速地判断是否为本设备的中断，若不是，应迅速返回 IRQ_NONE。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241125184336914.png" alt="image-20241125184336914" style="zoom:75%;" />

以下是使用共享中断的设备驱动程序模板：

```c
// 中断处理顶半部
irqreturn_t xxx_interrupt(int irq, void *dev_id)
{
    ...

    int status = read_int_status(); // 获知中断源
    if (!is_myint(dev_id, status))      // 判断是否为本设备中断
        return IRQ_NONE;                // 不是本设备中断，立即返回

    // 是本设备中断，进行处理
    ...

    return IRQ_HANDLED; // 返回IRQ_HANDLED表明中断已被处理
}

// 设备驱动模块加载函数
int xxx_init(void)
{
    ...

    // 申请共享中断
    result = request_irq(sh_irq, xxx_interrupt, IRQF_SHARED, "xxx", xxx_dev);

    ...
}

// 设备驱动模块卸载函数
void xxx_exit(void)
{
    ...

    // 释放中断
    free_irq(xxx_irq, xxx_interrupt);

    ...
}
```

## 内核定时器

### 内核定时器编程

软件意义上的定时器最终依赖硬件定时器来实现，内核在时钟中断发生后检测各定时器是否到期，到期后的定时器处理函数将作为软中断在底半部执行。实质上，**时钟中断处理程序会唤起 TIMER_SOFTIRQ 软中断，运行当前处理器上到期的所有定时器。**

Linux 内核提供了一组函数和数据结构用于定时触发工作和完成某周期性的任务。编程者在多数情况无需关心具体的内核和硬件行为。

书中内容基于 Linux 4.0 版本，在 Linux 5 中 API 已发生变动，参照文章 [https://blog.csdn.net/ZHONGCAI0901/article/details/120484815](https://blog.csdn.net/ZHONGCAI0901/article/details/120484815) 一同学习。

1. timer_list

timer_list 结构体的实例对应一个定时器。当定时器期满以后，function() 函数将被执行，expires 是定时器到期的时间。

```c
struct timer_list
{
    /*
     * All fields that change during normal runtime grouped to the
     * same cacheline
     */
    struct hlist_node entry;
    unsigned long expires;
    void (*function)(struct timer_list *);
    u32 flags;

#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;
#endif
};
```

2. 初始化定时器

`__init_timer` 是一个宏函数。用于初始化 timer_list 结构体，并设置其中的函数、flags。

```c
#define __init_timer(_timer, _fn, _flags)				\
	do {								\
		static struct lock_class_key __key;			\
		init_timer_key((_timer), (_fn), (_flags), #_timer, &__key);\
	} while (0)

#define __init_timer_on_stack(_timer, _fn, _flags)			\
	do {								\
		static struct lock_class_key __key;			\
		init_timer_on_stack_key((_timer), (_fn), (_flags),	\
					#_timer, &__key);		 \
	} while (0)
```

timer_setup 看定义应该是 `__init_timer` 的别名。

```c
#define timer_setup(timer, callback, flags)			\
	__init_timer((timer), (callback), (flags))

#define timer_setup_on_stack(timer, callback, flags)		\
	__init_timer_on_stack((timer), (callback), (flags))
```

`__TIMER_INITIALIZER(_function, _flags)` 宏用于赋值定时器结构体的 function、flags 成员。

DEFINE_TIMER 是定义并初始化名为 _name 的定时器快捷方式，从宏定义而知显而易见。

```c
#define __TIMER_INITIALIZER(_function, _flags) {		\
		.entry = { .next = TIMER_ENTRY_STATIC },	\
		.function = (_function),			\
		.flags = (_flags),				\
		__TIMER_LOCKDEP_MAP_INITIALIZER(		\
			__FILE__ ":" __stringify(__LINE__))	\
	}

#define DEFINE_TIMER(_name, _function)				\
	struct timer_list _name =				\
		__TIMER_INITIALIZER(_function, 0)
```

3. 添加定时器

此函数用于注册内核定时器，将定时器加入到内核动态定时器链表中。

```c
void add_timer(struct timer_list *timer);
```

4. 删除定时器

del_timer_sync() 是 del_timer() 的同步版，在删除一个定时器时需等待其被处理完，故其调用不能发生在中断上下文。

```c
int del_timer(struct timer_list * timer);
int del_timer_sync(struct timer_list *timer)
```

5. 修改定时器的到期时间 expires

此函数用于修改定时器的到期时间，在新的被传入的 expires 到来后才会执行定时器函数。

```c
int mod_timer(struct timer_list *timer, unsigned long expires);
```

定时器的时间基于 jiffies，在修改超时时间时，一般使用 2 种方法。

从下面的示例可以看出，**expires 的含义是过期的时间点，而非经过多少时间过期，故需要加上基准时间 jiffies。**

```c
mod_timer(&timer, jiffies + xxx); // xxx 表示多少个滴答后超时，也就是 xxx * 10ms
mod_timer(&timer, jiffies + 2 * HZ); // HZ 等于 CONFIG_HZ，2 * HZ 就相当于 2 秒
```

在 Linux 5 版本以后，**mod_timer() 的含义不仅仅是更改定时器的到期时间，同时会将其添加到内核动态定时器链表中。**与 add_timer() 不同的是 add_timer() 使用的到期时间是其结构体自己内部预先设置的时间，mod_timer() 相当于做了赋值操作后再添加。关于 mod_timer() 新的含义，通过源码也可以猜到：

```c
void add_timer(struct timer_list *timer)
{
	BUG_ON(timer_pending(timer));
	__mod_timer(timer, timer->expires, MOD_TIMER_NOTPENDING);
}
EXPORT_SYMBOL(add_timer);

int mod_timer(struct timer_list *timer, unsigned long expires)
{
	return __mod_timer(timer, expires, 0);
}
EXPORT_SYMBOL(mod_timer);
```

另外，**当定时器到期以后，执行完回调函数后，会从内核动态定时器链表中删除。如果需要周期性地执行，需要手动每次在过期以后重新添加。**从下面的模板中也可见一斑。

6. 使用模板

以下是一个使用内核定时器的模板：

```c
// xxx 设备结构体
struct xxx_dev
{
    struct cdev cdev;

    ...

    timer_list xxx_timer; // 设备要使用的定时器
};

// xxx 驱动中的加载函数
xxx_init(…)
{
    struct xxx_dev *dev = filp->private_data;
    ...

    // 初始化定时器
    timer_setup(&dev->xxx_timer, xxx_do_timer, 0);
    // 设备结构体指针作为定时器处理函数参数
    dev->xxx_timer.expires = jiffies + delay;
    // 添加（注册）定时器
    add_timer(&dev->xxx_timer);

    ...
}

// xxx 驱动中的卸载函数
xxx_exit(…)
{
    ...

    // 删除定时器
    del_timer(&dev->xxx_timer);

    ...
}

// 定时器处理函数
static void xxx_do_timer(unsigned long arg)
{
    struct xxx_device *dev = (struct xxx_device *)(arg);

    ...

    // 调度定时器再执行
    // 在实际使用中，定时器的到期时间往往是在目前 jiffies 的基础上添加一个时延，若为 Hz，则表示延迟 1s。
    dev->xxx_timer.expires = jiffies + delay;
    // 在定时器处理函数中，在完成相应的工作后，往往会延后 expires 并将定时器再次添加到内核定时器链表中，以便定时器能再次被触发。
    add_timer(&dev->xxx_timer);

    ...
}
```

Linux 内核在支持 tickless 和 NO_HZ 模式后，内核也包含对 hrtimer（高精度定时器）的支持，它可以支持到微秒级别的精度。内核也定义了 hrtimer 结构体，hrtimer_set_expires()、hrtimer_start_expires()、hrtimer_forward_now()、hrtimer_restart() 等类似的 API 来完成 hrtimer 的设置、时间推移以及到期回调。

### delayed_work

对于周期性的任务，除定时器以外，还可利用一套封装得很好的快捷机制，本质是利用工作队列和定时器实现，即 delayed_work。

```c
struct delayed_work {
	struct work_struct work;
	struct timer_list timer;

	/* target workqueue and CPU ->timer uses to queue ->work */
	struct workqueue_struct *wq;
	int cpu;
};

struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};
```

可以通过如下函数调度一个 delayed_work 在指定的延时以后运行：

```c
bool schedule_delayed_work(struct delayed_work *dwork, unsigned long delay);
```

当延时 delay 到来以后，delayed_work 结构体的 work_struct 类型成员 dwork 的 work_func_t 类型成员 func() 函数会被执行，work_func_t 定义为：

```c
void (*work_func_t)(struct work_struct *work);
```

另外，delay 参数的单位是 jiffies，一种常见的用法如下：

```c
// msecs_to_jiffies() 用于将毫秒转化为 jiffies。
schedule_delayed_work(&work, msecs_to_jiffies(poll_interval));
```

同定时器，如果需要周期性的执行任务，一般需要手动在工作函数再次调用 schedule_delayed_work()，周而复始。

以下函数用于取消 delay_work：

```c
bool cancel_delayed_work(struct delayed_work *dwork);
bool cancel_delayed_work_sync(struct delayed_work *dwork);
```

### 秒字符设备

编写一个字符设备 second（即“秒”）的驱动。它被打开的时候初始化一个定时器，并添加到内核定时器链表中，每秒在内核日志中输出一次当前的 jiffies，读取该字符设备的时候返回当前定时器的秒数。

自己实现的版本：[https://github.com/DavidingPlus/linux-kernel-learning/tree/second](https://github.com/DavidingPlus/linux-kernel-learning/tree/second)

## 内核延时

参考文章：

- [https://blog.csdn.net/liangzc1124/article/details/120835795](https://blog.csdn.net/liangzc1124/article/details/120835795)
- [https://www.cnblogs.com/xihong2014/p/6740876.html](https://www.cnblogs.com/xihong2014/p/6740876.html)

注：这两篇文章设计的 API 都不是 Linux 5 及其以后的，所以建议参考原理而非照搬 API。

1. 短延时

Linux 内核提供了三个函数分别进行纳秒、微妙和毫秒延迟，在 Linux 5 以后已改为宏函数。

这几个短延时的本质是**忙等待**，根据 CPU 频率进行一定次数的循环。

```c
#define ndelay(n) { ... }
#define udelay(n) { ... }
#define mdelay(n) { ... }
```

2. 中等延时

毫秒时延（以及更大的秒时延）已经比较大了。在内核中，最好不要直接使用 mdelay() 函数，因为忙等将耗费 CPU 资源。对于毫秒级以上的时延，内核提供了下述函数。

这几个中等延时期间，**程序进入睡眠状态**。并且受系统 Hz 以及进程调度的影响，msleep() 类似函数的精度是有限的。

```c
void msleep(unsigned int msecs); // 延时 msecs 毫秒，程序进入睡眠，且不可被打断。
unsigned long msleep_interruptible(unsigned int msecs); // 延时 msecs 毫秒，程序进入睡眠，但可以被打断。
void ssleep(unsigned int seconds); // 延时 seconds 秒，程序进入睡眠，且不可被打断。
```

3. 长延时

内核中延时的一个很直观的方法是比较当前的 jiffies 和目标 jiffies，直到未来的 jiffies 达到目标 jiffies。

从下面的使用方法看出，很类似于内核定时器的 expires，delay 也是表示延时到期的时间点。

```c
// 延时 100 个滴答
unsigned long delay = jiffies + 100;
while (time_before(jiffies, delay)) { }

// 延时 2s
unsigned long delay = jiffies + 2 * Hz;
while (time_before(jiffies, delay)) { }
```

长延时使用的宏函数 time_before() 和 time_after() 的定义如下，可见仅仅是对两个值做了一个大小比较。

time_before() 的第一个参数是当前 jiffies，第二个参数是目标 jiffies，time_after() 相反。

```c
#define time_after(a,b)		\
	(typecheck(unsigned long, a) && \
	 typecheck(unsigned long, b) && \
	 ((long)((b) - (a)) < 0))

#define time_before(a,b)	time_after(b,a)
```

结合上述可以发现，这样使用长延时是一个**忙等待**的过程。

4. 睡着延时

睡着延迟显然是比忙等待更好的方式，睡着延迟是在等待的时间到来之前进程处于睡眠状态，CPU 资源被其他进程使用。

schedule_timeout() 可以使当前任务休眠至指定的 jiffies 之后再重新被调度执行，msleep() 和 msleep_interruptible() 在本质上都是依靠包含了 schedule_timeout() 的 schedule_timeout_uninterruptible() 和 schedule_timeout_interruptible() 实现的。

schedule_timeout() 的原理是**向系统添加一个定时器，在定时器处理函数中唤醒与参数对应的进程**。

```c
void msleep(unsigned int msecs)
{
	unsigned long timeout = msecs_to_jiffies(msecs) + 1;

	while (timeout)
		timeout = schedule_timeout_uninterruptible(timeout);
}

unsigned long msleep_interruptible(unsigned int msecs)
{
	unsigned long timeout = msecs_to_jiffies(msecs) + 1;

	while (timeout && !signal_pending(current))
		timeout = schedule_timeout_interruptible(timeout);
	return jiffies_to_msecs(timeout);
}
```

另外，schedule_timeout_uninterruptible() 和 schedule_timeout_interruptible() 的定义如下，均使用 `__set_current_state` 设置进程状态。

```c
signed long __sched schedule_timeout_uninterruptible(signed long timeout)
{
	__set_current_state(TASK_UNINTERRUPTIBLE);
	return schedule_timeout(timeout);
}

signed long __sched schedule_timeout_interruptible(signed long timeout)
{
	__set_current_state(TASK_INTERRUPTIBLE);
	return schedule_timeout(timeout);
}
```

## 小结

Linux 的中断处理分为两个半部，顶半部处理紧急的硬件操作，底半部处理不紧急的耗时操作。tasklet 和工作队列都是调度中断底半部的良好机制。tasklet 基于软中断实现。内核定时器也依靠软中断实现。

内核中的延时可以采用忙等待或睡眠等待，为了充分利用 CPU 资源，使系统有更好的吞吐性能。在对延迟时间的要求并不是很精确的情况下，睡眠等待通常是值得推荐的，而 ndelay()、udelay() 忙等待机制在驱动中通常是为了配合硬件上的短时延迟要求。

# 内存与 I/O 访问

## CPU 与内存、I/O

### 内存空间与 I/O 空间

在 X86 处理器中存在着 I/O 空间的概念。I/O 空间是相对于内存空间而言的，它通过特定的指令 in、out 来访问。端口号标识了外设的寄存器地址。Intel 语法中的 in、out 指令格式如下：

```asm
IN 累加器, {端口号 │ DX}
OUT {端口号 │ DX}, 累加器
```

目前，大多数嵌入式微控制器（如 ARM、PowerPC 等）中并不提供 I/O 空间，而仅存在内存空间。**内存空间可以直接通过地址、指针来访问，程序及在程序运行中使用的变量和其他数据都存在于内存空间中。**

在 X86 处理器中，虽然提供了 I/O 空间，外设仍然可以只挂接在内存空间中。此时，CPU 可以像访问一个内存单元那样访问外设 I/O 端口，而不需要设立专门的 I/O 指令。**内存空间是必需的，而 I/O 空间是可选的。**如图是内存空间和 I/O 空间的对比。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241127111734557.png" alt="image-20241127111734557" style="zoom:75%;" />

在下面这段程序中，没有定义任何一个函数实体，但是程序却执行了函数调用 lpReset()。它实际上起到了“软重启”的作用，跳转到 CPU 启动后第一条要执行的指令的位置。因此，可以通过函数指针调用一个没有函数体的“函数”，本质上只是换一个地址开始执行。

```c
typedef void (*lpFunction)();                // 定义一个无参数、无返回类型的函数指针类型
lpFunction lpReset = (lpFunction)0xF000FFF0; // 定义一个函数指针，指向 CPU 启动后所执行的第一条指令的位置
lpReset();                                   // 调用函数
```

### 内存管理单元 MMU

高性能处理器一般会提供一个内存管理单元 MMU。**MMU 辅助操作系统进行内存管理，提供虚拟地址和物理地址的映射、内存访问权限保护和 Cache 缓存控制等硬件支持。**内核借助 MMU 可以让用户感觉到程序好像可以使用非常大的内存空间，从而使得编程人员在写程序时不用考虑计算机中物理内存的实际容量。

首先需明确几个概念：

1. **TLB（Translation Lookaside Buffer）：即转换旁路缓存。TLB 是 MMU 的核心部件，它缓存少量的虚拟地址与物理地址的转换关系，是转换表的 Cache，也经常被称为“快表”。**
2. **TTW（Translation Table walk）：即转换表漫游。当 TLB 中没有缓冲对应的地址转换关系时，需要通过对内存中转换表（大多数为多级页表）的访问来获得虚拟地址和物理地址的对应关系。TTW 成功后，结果应写入 TLB 中。**

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241127112853842.png" alt="image-20241127112853842" style="zoom:75%;" />

不同架构的 CPU 访问数据的流程有区别，但大致流程基本如下所示：

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241127142844768.png" alt="image-20241127142844768" style="zoom:80%;" />

MMU 具有虚拟地址和物理地址转换、内存访问权限保护等功能，使得 Linux 操作系统能单独为系统的每个用户进程分配独立的内存空间，并保证用户空间不能访问内核空间的地址，为操作系统的虚拟内存管理模块提供硬件基础。

在 Linux 2.6 之前，Linux 内核硬件无关层使用了三级页表 PGD、PMD 和 PTE。从 Linux 2.6 开始，为了配合 64 位 CPU 的体系结构，硬件无关层则使用了 4 级页表目录管理的方式，即 PGD、PUD、PMD 和 PTE。这仅仅是一种软件意义上的抽象，实际硬件的页表级数可能少于 4。

直到 Linux 5.0 的时候，都是如上的 4 级页表的结构。但在 Linux 5.15 版本中，在源代码中出现了一个叫 P4D 的东西，经查阅发现这是 5 级页表才需要的东西。源码如下：

```c
static int
pin_page_for_write(const void __user *_addr, pte_t **ptep, spinlock_t **ptlp)
{
	unsigned long addr = (unsigned long)_addr;
	pgd_t *pgd;
	p4d_t *p4d;
	pmd_t *pmd;
	pte_t *pte;
	pud_t *pud;
	spinlock_t *ptl;

	pgd = pgd_offset(current->mm, addr);
	if (unlikely(pgd_none(*pgd) || pgd_bad(*pgd)))
		return 0;

	p4d = p4d_offset(pgd, addr);
	if (unlikely(p4d_none(*p4d) || p4d_bad(*p4d)))
		return 0;

	pud = pud_offset(p4d, addr);
	if (unlikely(pud_none(*pud) || pud_bad(*pud)))
		return 0;

	pmd = pmd_offset(pud, addr);
	if (unlikely(pmd_none(*pmd)))
		return 0;

	/*
	 * A pmd can be bad if it refers to a HugeTLB or THP page.
	 *
	 * Both THP and HugeTLB pages have the same pmd layout
	 * and should not be manipulated by the pte functions.
	 *
	 * Lock the page table for the destination and check
	 * to see that it's still huge and whether or not we will
	 * need to fault on write.
	 */
	if (unlikely(pmd_thp_or_huge(*pmd))) {
		ptl = &current->mm->page_table_lock;
		spin_lock(ptl);
		if (unlikely(!pmd_thp_or_huge(*pmd)
			|| pmd_hugewillfault(*pmd))) {
			spin_unlock(ptl);
			return 0;
		}

		*ptep = NULL;
		*ptlp = ptl;
		return 1;
	}

	if (unlikely(pmd_bad(*pmd)))
		return 0;

	pte = pte_offset_map_lock(current->mm, pmd, addr, &ptl);
	if (unlikely(!pte_present(*pte) || !pte_young(*pte) ||
	    !pte_write(*pte) || !pte_dirty(*pte))) {
		pte_unmap_unlock(pte, ptl);
		return 0;
	}

	*ptep = pte;
	*ptlp = ptl;

	return 1;
}
```

为什么会出现 p4d 呢？参考了文章 [https://zhuanlan.zhihu.com/p/641479824](https://zhuanlan.zhihu.com/p/641479824) 才明白为什么。这里只做总结提炼，具体请参考原文。更多关于 Linux 内核页表管理的机制，请自行查阅资料。

Linux 修改了这部分代码，保证了目前的 4 级页表可用，又能对 5 级页表兼容。在 4 级页表索引中，虽然添加了 p4d，但是其对 p4d 实际上并未做任何有效操作，相当于将 p4d 折叠和忽略了。而 5 级页表需要的 p4d 就能正常工作，做到了设计统一。虽然 4 级页表能支持 256 TB 的地址空间，目前看来用不完。但大数据时代已经来临，未雨绸缪的考虑是值得肯定的。这样以来，针对使用 5 级页表的系统无需大量修改代码，直接可以使用现有的内核代码。

MMU 并不是对所有的处理器都是必需的。Linux 2.6 以后也支持不带 MMU 的处理器。Linux 为了更广泛地应用于嵌入式系统，融合了 mClinux，以支持这些无 MMU 系统。

## 内存管理

内核空间和用户空间的内存映射关系请参考文章 [https://blog.csdn.net/Luckiers/article/details/141750008](https://blog.csdn.net/Luckiers/article/details/141750008)

**每个进程的用户空间是完全独立、互不相干的，用户进程各自有不同的页表。而内核空间是由内核负责映射，它并不会跟着进程改变，是固定的。内核空间的虚拟地址到物理地址映射是被所有进程共享的，内核的虚拟空间独立于其他程序。**

通常 32 位 Linux 的内核地址空间划分为 0~3 G（3 G 的位置为 PAGE_OFFSET）的用户空间，3~4 G 的内核空间。

![image-20250201002213404](https://image.davidingplus.cn/images/2025/02/01/image-20250201002213404.png)

通常 64 位 Linux 系统的内核地址空间分为三个部分，256 TB 的用户空间、非规范区域和 256 TB 的内核空间。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20250201002304708.png" alt="image-20250201002304708" style="zoom:65%;" />

关于映射空间解析的部分，书中讲的甚是繁杂，没怎么看懂。参考上面文章以及其他资料吧。

## 内存存取

### 用户空间内存申请

在用户态中动态申请内存的函数是标准 C 库函数 malloc()，与之对应的释放函数是 free()。**对 Linux 而言，malloc() 一般通过 brk() 和 mmap() 两个系统调用从内核申请内存。**

**用户空间 C 库的 malloc 算法实际上具备一个二次管理能力，并不是每次申请和释放内存都一定伴随着对内核的系统调用。**下面的代码展示了这点，下面进行解释。

应用程序从内核拿到内存后，立即调用 free()，但由于 free() 之前调用了 mallopt(M_TRIM_THRESHOLD, -1) 和 mallopt(M_MMAP_MAX, 0)。

mallopt(M_TRIM_THRESHOLD, -1) 设置 M_TRIM_THRESHOLD 为 -1，关闭内存修剪功能，即使释放的内存块超过一定阈值，也不会通过 brk 归还给内核，而是保留在进程的地址空间中供分配器复用。

mallopt(M_MMAP_MAX, 0) 设置 M_MMAP_MAX 为 0，禁止分配器使用 mmap 分配内存，所有分配强制通过进程堆（sbrk）完成。释放的内存不会通过 munmap 归还给内核，而是由分配器管理。

如此以来，这个 free() 并不会把内存还给内核，而只是还给了 C 库的分配算法，内存仍然属于这个进程。后续的 malloc 调用会优先使用这块已经分配好的内存，而不需要进行系统调用（brk 或 mmap）。

```c
#include <malloc.h>
#include <sys/mman.h>

#define SOMESIZE (100 * 1024 * 1024) // 100MB

int main(int argc, char *argv[])
{
    unsigned char *buffer;
    int i;

    if (mlockall(MCL_CURRENT | MCL_FUTURE))
        mallopt(M_TRIM_THRESHOLD, -1);
    mallopt(M_MMAP_MAX, 0);

    buffer = malloc(SOMESIZE);
    if (!buffer)
        exit(-1);

    /*
     * Touch each page in this piece of memory to get it
     * mapped into RAM
     */
    for (i = 0; i < SOMESIZE; i += page_size)
        buffer[i] = 0;
    free(buffer);
    /* <do your RT-thing> */

    return 0;
}
```

另外，**Linux 内核总是按需调页（Demand Paging）**。当 malloc() 返回的时候，虽然是成功返回，但内核并没有真正给这个进程内存，这个时候如果去读申请的内存，内容全部是 0，并且这个页面的映射是只读的。只有当写到某个页面的时候，内核才会在页错误后，真正把这个页面给这个进程。

### 内核空间内存申请

在 Linux 内核空间中动态申请内存主要涉及 kmalloc()、`__get_free_pages()` 和 vmalloc() 等。

**kmalloc() 和 `__get_free_pages()`申请的内存位于 DMA 和常规区域的映射区，在物理上也是连续的，它们与真实的物理地址只有一个固定的偏移，存在较简单的转换关系。**

**vmalloc() 在虚拟内存空间给出一块连续的内存区。实质上，这片连续的虚拟内存在物理内存中并不一定连续，vmalloc() 申请的虚拟内存和物理内存之间也没有简单的换算关系。**

#### kmalloc()

```c
// 第一个参数是要分配的块的大小；第二个参数为分配标志，用于控制 kmalloc() 的行为。
void *kmalloc(size_t size, gfp_t flags)
```

**最常用的分配标志是 GFP_KERNEL，含义是在内核空间的进程中申请内存。kmalloc() 的底层是依赖 `__get_free_pages()` 实现的。使用 GFP_KERNEL 标志申请内存时，若暂时不能满足，进程会睡眠等待，即会引起阻塞，因此不能在中断上下文或持有自旋锁的时候使用 GFP_KERNEL 申请内存。**

**在中断处理函数、tasklet 和内核定时器等非进程上下文中不能阻塞，应当使用 GFP_ATOMIC 标志申请内存。**当使用 GFP_ATOMIC 标志申请内存时，若不存在空闲页，则不等待，直接返回，避免了睡眠阻塞的问题。

其他的申请标志包括：

- GFP_USER：用来为用户空间页分配内存，可能阻塞。
- GFP_HIGHUSER：类似 GFP_USER，但是它从高端内存分配。
- GFP_DMA：从 DMA 区域分配内存。
- GFP_NOIO：不允许任何 I/O 初始化。
- GFP_NOFS：不允许进行任何文件系统调用。
- `__GFP_HIGHMEM`：指示分配的内存可以位于高端内存。
- `__GFP_COLD`：请求一个较长时间不访问的页。
- `__GFP_NOWARN`：当一个分配无法满足时，阻止内核发出警告。
- `__GFP_HIGH`：高优先级请求，允许获得被内核保留给紧急状况使用的最后的内存页。
- `__GFP_REPEAT`：分配失败，则尽力重复尝试。
- `__GFP_NOFAIL`：标志只许申请成功，不推荐。
- `__GFP_NORETRY`：若申请不到，则立即放弃。

kmalloc() 申请的内存应使用 kfree() 函数释放，类似用户空间的标准 C 库函数 malloc() 和 free() 的关系。

#### __get_free_pages() 系列函数

**`__get_free_pages()` 系列函数/宏本质上是 Linux 内核最底层用于获取空闲内存的方法。底层的 buddy 算法以 2n 页为单位管理空闲内存，故最底层的内存申请总是以 2n 页为单位的。**

`__get_free_pages()` 系列函数/宏包括 get_zeroed_page()、`__get_free_page()` 和 `__get_free_pages()`。

```c
// 该函数返回一个指向新页的指针并且将该页清零。
// 注意这里的指针使用 unsigned long 类型表示，类似 LarkSDK 中 LLog 对指针的处理。
unsigned long get_zeroed_page(gfp_t gfp_mask);

// 该宏返回一个指向新页的指针但是该页不清零。
#define __get_free_page(gfp_mask) \
		__get_free_pages((gfp_mask), 0)

// 该函数可分配多个页并返回分配内存的首地址，分配的页数为 2 的 order 次方，分配的页不清零。order 允许的最大值是 10（1024 页）或者11（2048 页），这取决于具体的硬件平台。
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);
```

`__get_free_pages()` 和 get_zeroed_page() 在实现中调用了 alloc_pages() 函数。**alloc_pages() 既可以在内核空间分配，也可以在用户空间分配。**

```c
// 参数含义与 __get_free_pages() 类似，但它返回分配的第一个页的描述符而非首地址（前面的 unsigned long）。
// 这里的描述符使用 struct page * 指针描述的。
struct page *alloc_pages(gfp_t gfp_mask, unsigned int order);
```

该系列函数使用如下接口释放：

```c
void free_pages(unsigned long addr, unsigned int order);
#define free_page(addr) free_pages((addr), 0)
```

`__get_free_pages()` 系列函数在使用时，申请标志的值与 kmalloc() 完全一致，各标志的含义也与完全一致。显而易见，因为 kmalloc() 底层是基于 `__get_free_pages()` 实现的。

#### vmalloc()

**vmalloc() 一般只为存在于软件中（没有对应的硬件意义）的较大的顺序缓冲区分配内存，vmalloc() 远大于 `__get_free_pages()` 的开销。**为了完成 vmalloc()，新的页表项需要被建立。因此，调用 vmalloc() 来分配少量的内存（如 1 页以内的内存）是不妥的。

类似的，分配使用 vmalloc()，释放使用 vfree()。

```c
void *vmalloc(unsigned long size);
void vfree(const void *addr);
```

**vmalloc() 不能用在原子上下文中，因为内部实现使用了标志为 GFP_KERNEL 的 kmalloc()。**

**vmalloc() 在申请内存时，会进行内存的映射，改变页表项，不像 kmalloc() 实际用的是开机过程中就映射好的 DMA 和常规区域的页表项。**故 vmalloc() 的虚拟地址和物理地址不是一个简单的线性映射。

#### slab

**完全使用页为单元申请和释放内存容易导致浪费（若要申请少量字节，也需要用 1 页）。另外在操作系统的运作过程中，经常会涉及大量对象的重复生成、使用和释放内存问题。**在 Linux 系统中所用到的对象，比较典型的例子是 inode、task_struct 等。**若能用合适的方法使得对象在前后两次被使用时分配在同一块内存或同一类内存空间且保留了基本的数据结构，就可以大大提高效率。**slab 算法就是针对上述特点设计的。**实际上 kmalloc() 就是使用 slab 机制实现的。**

**slab 是建立在 buddy 算法之上的，它从 buddy 算法拿到 2n 页面后再次进行二次管理**，和用户空间的 C 库很像。slab 申请的内存以及基于 slab 的 kmalloc() 申请的内存，与物理内存之间也是一个简单的线性偏移。

1. 创建 slab 缓存

kmem_cache_create() 用于创建一个缓存，是一个可以保留任意数目且全部同样大小的后备缓存。

```c
// size：要分配的每个数据结构的大小。
// flags：控制如何进行分配的位掩码，如 SLAB_HWCACHE_ALIGN（每个数据对象被对齐到一个缓存行）、SLAB_CACHE_DMA（要求数据对象在 DMA 区域中分配）等。
struct kmem_cache *kmem_cache_create(const char *name, unsigned int size, unsigned int align, slab_flags_t flags, void (*ctor)(void *));
```

2. 分配 slab 缓存

kmem_cache_alloc() 在创建的 slab 后备缓存中分配一块并返回首地址指针。

```c
void *kmem_cache_alloc(struct kmem_cache *, gfp_t flags) __assume_slab_alignment __malloc;
```

3. 释放 slab 缓存

```c
void kmem_cache_free(struct kmem_cache *, void *);
```

4. 收回 slab 缓存

```c
void kmem_cache_destroy(struct kmem_cache *);
```

5. 使用模板

```c
// 创建 slab 缓存
static kmem_cache_t *xxx_cachep;
xxx_cachep = kmem_cache_create("xxx", sizeof(struct xxx), 0, SLAB_HWCACHE_ALIGN | SLAB_PANIC, NULL, NULL);

// 分配 slab 缓存
struct xxx *ctx;
ctx = kmem_cache_alloc(xxx_cachep, GFP_KERNEL);

// 使用 slab 缓存
...

// 释放 slab 缓存
kmem_cache_free(xxx_cachep, ctx);

// 收回 slab 缓存
kmem_cache_destroy(xxx_cachep);
```

系统中通过查看 /proc/slabinfo 可查看当前 slab 的分配和使用情况，结果可能如下：

```bash
cat /proc/slabinfo

# slabinfo - version: 2.1
# # name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
# AF_VSOCK              50     50   1280   25    8 : tunables    0    0    0 : slabdata      2      2      0
# ext4_groupinfo_4k    420    420    192   21    1 : tunables    0    0    0 : slabdata     20     20      0
# btrfs_delayed_node      0      0    312   26    2 : tunables    0    0    0 : slabdata      0      0      0
# btrfs_ordered_extent      0      0    424   19    2 : tunables    0    0    0 : slabdata      0      0      0
# btrfs_extent_map       0      0    144   28    1 : tunables    0    0    0 : slabdata      0      0      0
# btrfs_trans_handle      0      0    112   36    1 : tunables    0    0    0 : slabdata      0      0      0
# btrfs_inode            0      0   1208   27    8 : tunables    0    0    0 : slabdata      0      0      0
# fsverity_info          0      0    256   16    1 : tunables    0    0    0 : slabdata      0      0      0
# fscrypt_info           0      0    128   32    1 : tunables    0    0    0 : slabdata      0      0      0
# ip6-frags              0      0    184   22    1 : tunables    0    0    0 : slabdata      0      0      0
# PINGv6                 0      0   1216   26    8 : tunables    0    0    0 : slabdata      0      0      0
# RAWv6                288    416   1216   26    8 : tunables    0    0    0 : slabdata     16     16      0
# UDPv6                 96     96   1344   24    8 : tunables    0    0    0 : slabdata      4      4      0
# tw_sock_TCPv6          0      0    248   16    1 : tunables    0    0    0 : slabdata      0      0      0
# ...
```

> 注：slab 不是要代替 `__get_free_pages()`，其最底层仍然依赖于 `__get_free_pages()`，slab 在底层每次申请 1 页或多页，之后再分隔这些页为更小的单元进行管理，从而节省了内存，也提高了 slab 缓冲对象的访问效率。

#### 内存池

除 slab 以外，内存池也是一种非常经典的分配大量小对象的技术。

1. 创建内存池

```c
// min_nr：预分配对象的数目。
// alloc_fn：指向内存池机制提供的标准对象分配函数的指针。
// free_fn：指向内存池机制提供的标准对象回收函数的指针。
mempool_t *mempool_create(int min_nr, mempool_alloc_t *alloc_fn, mempool_free_t *free_fn, void *pool_data);
```

这两个函数指针定义如下：

```c
// pool_data：分配函数用到的指针。
// gfp_mask：分配标记。只有当 __GFP_WAIT 被指定时，分配函数才会休眠。
typedef void * (mempool_alloc_t)(gfp_t gfp_mask, void *pool_data);

// pool_data：回收函数用到的指针。
typedef void (mempool_free_t)(void *element, void *pool_data);
```

2. 分配和回收对象

mempool_alloc() 用于分配对象。若内存池分配器无法提供内存，可以使用预分配的池。mempool_free() 用于回收对象。

```c
void *mempool_alloc(mempool_t *pool, gfp_t gfp_mask) __malloc;
void mempool_free(void *element, mempool_t *pool);
```

3. 回收内存池

```c
void mempool_destroy(mempool_t *pool);
```

## 设备 I/O 端口与 I/O 内存的访问

**设备通常会提供一组寄存器来控制设备、读写设备和获取设备状态，即控制寄存器、数据寄存器和状态寄存器。**这些寄存器可能位于 I/O 空间中，也可能位于内存空间中。**当位于 I/O 空间时，通常被称为 I/O 端口；当位于内存空间时，对应的内存空间被称为 I/O 内存。二者的操作有非常多的相似之处。**

### I/O 端口和 I/O 内存访问接口

#### I/O 端口

Linux 内核提供了函数访问 I/O 空间的端口，具体如下：

1. 读写字节端口（8 位宽）。

```c
u8 inb(unsigned long port)
{
	return ioread8(ioport_map(port, 1));
}

void outb(u8 b, unsigned long port)
{
	iowrite8(b, ioport_map(port, 1));
}
```

2. 读写字端口（16 位宽）。

```c
u16 inw(unsigned long port)
{
	return ioread16(ioport_map(port, 2));
}

void outw(u16 b, unsigned long port)
{
	iowrite16(b, ioport_map(port, 2));
}
```

3. 读写长字节端口（32 位宽）。

```c
u32 inl(unsigned long port)
{
	return ioread32(ioport_map(port, 4));
}

void outl(u32 b, unsigned long port)
{
	iowrite32(b, ioport_map(port, 4));
}
```

4. 读写一串字节。

insb() 从端口 port 开始读 count 个字节端口，并将读取结果写入 dst 指向的内存；outsb() 将 addr 指向的内存中的 count 个字节连续写入以 port 开始的端口。

```c
void insb (unsigned long port, void *dst, unsigned long count);
void outsb (unsigned long port, const void *src, unsigned long count);
```

5. 读写一串字。

```c
void insw (unsigned long port, void *dst, unsigned long count);
void outsw (unsigned long port, const void *src, unsigned long count);
```

6. 读写一串长字。

```c
void insl (unsigned long port, void *dst, unsigned long count);
void outsl (unsigned long port, const void *src, unsigned long count);
```

上述函数 I/O 端口号 port 的类型高度依赖于具体的硬件平台，因此只写出了 unsigned。

#### I/O 内存

I/O 内存通常是芯片内部的各个 I2C、SPI、USB 等控制器的寄存器或者外部内存总线上的设备。**在内核中访问 I/O 内存之前，需首先使用 ioremap() 函数将设备的物理地址映射到虚拟地址上。**

```c
void __iomem *ioremap(unsigned long port, unsigned long size);
```

ioremap() 与 vmalloc() 类似，也需建立新的页表，**但并不进行 vmalloc() 中所执行的内存分配行为，类似于 reservation**。ioremap() 返回一个特殊的虚拟地址，该地址可用来存取特定的物理地址范围，这个虚拟地址位于 vmalloc 映射区域。

ioremap() 获得的虚拟地址用 iounmap() 函数释放。

```c
void iounmap(volatile void __iomem *addr);
```

ioremap() 有个变体 devm_ioremap()，类似其他以 devm_ 开头的函数，通过 devm_ioremap() 进行的映射通常不需要在驱动退出和出错处理的时候进行 iounmap()。

```c
void __iomem *devm_ioremap(struct device *dev, resource_size_t offset, resource_size_t size)
{
	return __devm_ioremap(dev, offset, size, DEVM_IOREMAP);
}
EXPORT_SYMBOL(devm_ioremap);
```

**在设备物理地址（一般都是寄存器）被映射到虚拟地址之后，尽管可以直接通过指针访问这些地址，但还是推荐用一组标准的 API 来完成这个虚拟地址的读写。**

读寄存器用 readb_relaxed()、readw_relaxed()、readl_relaxed()、readb()、readw()、readl() 这组 API，分别读 8 bit、16 bit、32 bit 的寄存器。有无 `_relaxed` 后缀的区别是没有 `_relaxed` 后缀的函数包含一个内存屏障。

```c
#define readb(c)		({ u8  __v = readb_relaxed(c); __iormb(); __v; })
#define readw(c)		({ u16 __v = readw_relaxed(c); __iormb(); __v; })
#define readl(c)		({ u32 __v = readl_relaxed(c); __iormb(); __v; })
// similar functions
#define readsb(p,d,l)		({ __raw_readsb(p,d,l); __iormb(); })
#define readsw(p,d,l)		({ __raw_readsw(p,d,l); __iormb(); })
#define readsl(p,d,l)		({ __raw_readsl(p,d,l); __iormb(); })
```

同理写操作如下：

```c
#define writeb(v,c)		({ __iowmb(); writeb_relaxed(v,c); })
#define writew(v,c)		({ __iowmb(); writew_relaxed(v,c); })
#define writel(v,c)		({ __iowmb(); writel_relaxed(v,c); })
// similar functions
#define writesb(p,d,l)		({ __iowmb(); __raw_writesb(p,d,l); })
#define writesw(p,d,l)		({ __iowmb(); __raw_writesw(p,d,l); })
#define writesl(p,d,l)		({ __iowmb(); __raw_writesl(p,d,l); })
```

### 申请与释放设备的 I/O 端口和 I/O 内存

#### I/O 端口申请

Linux 内核提供了函数用于申请和释放 I/O 端口，表明该驱动要访问这片区域。

```c
#define request_region(start,n,name)		__request_region(&ioport_resource, (start), (n), (name), 0)

// 函数向内核申请 n 个端口，这些端口从 start 开始，name 为设备的名称。若分配成功，返回值不是 NULL，若返回 NULL，意味着申请端口失败。
struct resource *__request_region(struct resource *parent, resource_size_t start, resource_size_t n, const char *name, int flags);
```

使用 request_region() 申请的 I/O 端口使用完毕后，通过 release_region() 归还给系统。

```c
#define release_region(start,n)	__release_region(&ioport_resource, (start), (n))

void __release_region(struct resource *parent, resource_size_t start, resource_size_t n);
```

request_region() 有变体，即不用考虑释放问题的 devm_request_region()。

```c
#define devm_request_region(dev,start,n,name) \
	__devm_request_region(dev, &ioport_resource, (start), (n), (name))

struct resource *__devm_request_region(struct device *dev, struct resource *parent, resource_size_t start, resource_size_t n, const char *name);
```

#### I/O 内存申请

同理，Linux 内核提供了函数用于申请和释放 I/O 内存。这里的申请代表该驱动要访问这片区域，但不会做任何内存映射的动作，类似于 reservation 的概念。

从源码中可以发现，不管是 I/O 端口还是 I/O 内存，底层都调用 __request_region() 函数，只是 parent 参数不同。这也恰好反映了二者操作上的类似和语义上的不同。

```c
#define request_mem_region(start,n,name) __request_region(&iomem_resource, (start), (n), (name), 0)
```

同理，使用 release_mem_region() 归还给系统。

```c
#define release_mem_region(start,n)	__release_region(&iomem_resource, (start), (n))
```

同理，存在变体 devm_request_mem_region()。

```c
#define devm_request_mem_region(dev,start,n,name) \
	__devm_request_region(dev, &iomem_resource, (start), (n), (name))
```

### 设备 I/O 端口和 I/O 内存访问流程

关于 I/O 端口的访问，在设备打开或驱动模块加载时申请 I/O 端口，之后使用 inb()、outb() 等进行端口访问，最后在设备关闭或驱动被卸载时释放 I/O 端口。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241209112015738.png" alt="image-20241209112015738" style="zoom:75%;" />

关于 I/O 内存的访问，首先调用 request_mem_region() 申请资源，接着将寄存器地址通过 ioremap() 映射到内核空间虚拟地址，之后通过 Linux 设备访问编程接口访问这些设备的寄存器。访问完成后，对 ioremap() 申请的虚拟地址进行释放，并释放 release_mem_region() 申请的 I/O 内存资源。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241209112025950.png" alt="image-20241209112025950" style="zoom:75%;" />

### 映射设备地址到内存空间

#### 内存映射与 VMA

**一般情况下，用户空间是不可能也不应该直接访问设备的，但设备驱动程序可实现 mmap() 函数，可使得用户空间能直接访问设备的物理地址。**实际上 mmap() 实现了这样的一个映射过程：**它将用户空间的一段内存与设备内存关联，当用户访问用户空间的这段地址范围时，实际上会转化为对设备的访问。**

这种能力对于显示适配器一类的设备非常有意义，若用户空间可直接通过内存映射访问显存的话，屏幕帧的各点像素不再需要一个从用户空间到内核空间复制的过程。

驱动的 mmap() 函数通过 file_operations 结构体注册。**mmap（）必须以 PAGE_SIZE 为单位进行映射。实际上，内存只能以页为单位进行映射，若要映射非 PAGE_SIZE 整数倍的地址范围，要先进行页对齐，强行以 PAGE_SIZE 的倍数大小进行映射。**

```c
// kernel space
struct file_operations
{
    ...

int (*mmap) (struct file *, struct vm_area_struct *);

    ...
};
```

用户空间进行系统调用 mmap() 的时候，最终会走到驱动的 mmap() 函数中。系统调用 mmap() 原型如下：

```c
// user space
// fd 为文件描述符，一般由 open() 返回。fd 可指定为 -1，此时需指定 flags 参数中的 MAP_ANON，表明是匿名映射。
// len 是映射到调用用户空间的字节数，从被映射文件开头 offset 个字节开始算起。offset 参数一般设为 0，表示从文件头开始映射。
// prot 参数指定访问权限，可取如下几个值的 "|"：PROT_READ（可读）、PROT_WRITE（可写）、PROT_EXEC（可执行）和 PROT_NONE（不可访问）。
// addr 指定文件应被映射到用户空间的起始地址，一般指定为 NULL，这样选择起始地址的任务将由内核完成，函数的返回值就是映射到用户空间的地址。
void *mmap(void *__addr, size_t __len, int __prot, int __flags, int __fd, __off_t __offset);
```

进行系统调用 mmap() 的时候，内核会如下处理：

1. **在进程的虚拟空间查找一块 VMA。**
2. **将这块 VMA 进行映射。**
3. **如果设备驱动程序或文件系统的 file_operations 定义了 mmap()，调用此函数。**
4. **将这个 VMA 插入进程的 VMA 链表中。**

file_operations 的 mmap() 的参数 struct vm_area_struct * 就是第一步找到的 VMA。

系统 mmap() 映射的内存通过 munmap() 解除映射。

```c
int munmap(void *__addr, size_t __len);
```

**驱动程序中 mmap() 的实现机制是建立页表，并填充 VMA 结构体的 vm_operations_struct 指针。VMA 就是 vm_area_struct，用于描述一个虚拟内存区域。**

```c
/*
 * This struct describes a virtual memory area. There is one of these
 * per VM-area/task. A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
	/* The first cache line has the info for VMA tree walking. */

	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;

	struct rb_node vm_rb;

	/*
	 * Largest free memory gap in bytes to the left of this VMA.
	 * Either between this VMA and vma->vm_prev, or between one of the
	 * VMAs below us in the VMA rbtree and its ->vm_prev. This helps
	 * get_unmapped_area find a free area of the right size.
	 */
	unsigned long rb_subtree_gap;

	/* Second cache line starts here. */

	struct mm_struct *vm_mm;	/* The address space we belong to. */

	/*
	 * Access permissions of this VMA.
	 * See vmf_insert_mixed_prot() for discussion.
	 */
	pgprot_t vm_page_prot;
	unsigned long vm_flags;		/* Flags, see mm.h. */

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap interval tree.
	 */
	struct {
		struct rb_node rb;
		unsigned long rb_subtree_last;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	struct list_head anon_vma_chain; /* Serialized by mmap_lock &
					  * page_table_lock */
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */

	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops;

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units */
	struct file * vm_file;		/* File we map to (can be NULL). */
	void * vm_private_data;		/* was vm_pte (shared mem) */

#ifdef CONFIG_SWAP
	atomic_long_t swap_readahead_info;
#endif
#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
	struct vm_userfaultfd_ctx vm_userfaultfd_ctx;
} __randomize_layout;
```

VMA 结构体描述的虚地址介于 vm_start 和 vm_end 之间。vm_ops 成员指向这个 VMA 的操作集。针对 VMA 的操作都被包含在 vm_operations_struct 结构体中。

```c
/*
 * These are the virtual MM functions - opening of an area, closing and
 * unmapping it (needed to keep files on disk up-to-date etc), pointer
 * to the functions called when a no-page or a wp-page exception occurs.
 */
struct vm_operations_struct {
	void (*open)(struct vm_area_struct * area);
	void (*close)(struct vm_area_struct * area);
	/* Called any time before splitting to check if it's allowed */
	int (*may_split)(struct vm_area_struct *area, unsigned long addr);
	int (*mremap)(struct vm_area_struct *area);
	/*
	 * Called by mprotect() to make driver-specific permission
	 * checks before mprotect() is finalised.   The VMA must not
	 * be modified.  Returns 0 if eprotect() can proceed.
	 */
	int (*mprotect)(struct vm_area_struct *vma, unsigned long start,
			unsigned long end, unsigned long newflags);
	vm_fault_t (*fault)(struct vm_fault *vmf);
	vm_fault_t (*huge_fault)(struct vm_fault *vmf,
			enum page_entry_size pe_size);
	vm_fault_t (*map_pages)(struct vm_fault *vmf,
			pgoff_t start_pgoff, pgoff_t end_pgoff);
	unsigned long (*pagesize)(struct vm_area_struct * area);

	/* notification that a previously read-only page is about to become
	 * writable, if an error is returned it will cause a SIGBUS */
	vm_fault_t (*page_mkwrite)(struct vm_fault *vmf);

	/* same as page_mkwrite when using VM_PFNMAP|VM_MIXEDMAP */
	vm_fault_t (*pfn_mkwrite)(struct vm_fault *vmf);

	/* called by access_process_vm when get_user_pages() fails, typically
	 * for use by special VMAs. See also generic_access_phys() for a generic
	 * implementation useful for any iomem mapping.
	 */
	int (*access)(struct vm_area_struct *vma, unsigned long addr,
		      void *buf, int len, int write);

	/* Called by the /proc/PID/maps code to ask the vma whether it
	 * has a special name.  Returning non-NULL will also cause this
	 * vma to be dumped unconditionally. */
	const char *(*name)(struct vm_area_struct *vma);

#ifdef CONFIG_NUMA
	/*
	 * set_policy() op must add a reference to any non-NULL @new mempolicy
	 * to hold the policy upon return.  Caller should pass NULL @new to
	 * remove a policy and fall back to surrounding context--i.e. do not
	 * install a MPOL_DEFAULT policy, nor the task or system default
	 * mempolicy.
	 */
	int (*set_policy)(struct vm_area_struct *vma, struct mempolicy *new);

	/*
	 * get_policy() op must add reference [mpol_get()] to any policy at
	 * (vma,addr) marked as MPOL_SHARED.  The shared policy infrastructure
	 * in mm/mempolicy.c will do this automatically.
	 * get_policy() must NOT add a ref if the policy at (vma,addr) is not
	 * marked as MPOL_SHARED. vma policies are protected by the mmap_lock.
	 * If no [shared/vma] mempolicy exists at the addr, get_policy() op
	 * must return NULL--i.e., do not "fallback" to task or system default
	 * policy.
	 */
	struct mempolicy *(*get_policy)(struct vm_area_struct *vma,
					unsigned long addr);
#endif
	/*
	 * Called by vm_normal_page() for special PTEs to find the
	 * page for @addr.  This is useful if the default behavior
	 * (using pte_page()) would not find the correct page.
	 */
	struct page *(*find_special_page)(struct vm_area_struct *vma,
					  unsigned long addr);
};
```

vm_operations_struct 结构体的实体会在 file_operations 的 mmap() 成员函数里被赋值给相应的 vm_ops。上述 open() 函数通常在驱动的 mmap() 里调用，close() 函数通常会在 munmap() 的时候被调用到。

一个 vm_operations_struct 的使用示例如下：

```c
static int xxx_mmap(struct file *filp, struct vm_area_struct *vma)
{
    // 创建页表项，以 VMA 结构体的成员作为 remap_pfn_range() 的参数，映射的虚拟地址范围是 vma->vm_start 至 vma->vm_end。
    if (remap_pfn_range(vma, vma->vm_start, vm->vm_pgoff, vma->vm_end - vma->vm_start, vma->vm_page_prot)) return -EAGAIN;

    vma->vm_ops = &xxx_remap_vm_ops;
    xxx_vma_open(vma);


    return 0;
}

// VMA 打开函数
static void xxx_vma_open(struct vm_area_struct *vma)
{
    ...

    printk(KERN_NOTICE "xxx VMA open, virt %lx, phys %lx\n", vma->vm_start, vma->vm_pgoff << PAGE_SHIFT);
}

// VMA 关闭函数
static void xxx_vma_close(struct vm_area_struct *vma)
{
    ...

    printk(KERN_NOTICE "xxx VMA close.\n");
}

// VMA 操作结构体
static struct vm_operations_struct xxx_remap_vm_ops = {
    .open = xxx_vma_open,
    .close = xxx_vma_close,
    ...
};
```

remap_pfn_range() 函数的原型如下：

```c
// addr：内存映射开始处的虚拟地址，虚拟地址范围是 [addr, addr + size]
// pfn：虚拟地址应该映射到的物理地址的页帧号。实际上是物理地址右移 PAGE_SHIFT 位。若 PAGE_SIZE 为 4 KB，则 PAGE_SHIFT 为 12，因为 PAGE_SIZE 等于 1 << PAGE_SHIFT。
// prot：新页所要求的保护属性。
int remap_pfn_range(struct vm_area_struct *vma, unsigned long addr, unsigned long pfn, unsigned long size, pgprot_t prot);
```

在驱动程序中，使用 remap_pfn_range() 能映射内存中的保留页、设备 I/O、framebuffer、camera 等内存。在此函数基础上可封装 io_remap_pfn_range()、vm_iomap_memory() 等 API。

```c
int io_remap_pfn_range(struct vm_area_struct *vma, unsigned long addr, unsigned long pfn, unsigned long size, pgprot_t prot)
{
	return remap_pfn_range(vma, addr, pfn, size, pgprot_decrypted(prot));
}

int vm_iomap_memory(struct vm_area_struct *vma, phys_addr_t start, unsigned long len);
```

**通常 I/O 内存被映射时是需要 nocache 的。这时应该设置 vma->vm_page_prot 为 nocache 标志再映射。**

```c
static int xxx_nocache_mmap(struct file *filp, struct vm_area_struct *vma)
{
    // 设置 nocache 标志
    vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);
    vma->vm_pgoff = ((u32)map_start >> PAGE_SHIFT);

    // 映射
    if (remap_pfn_range(vma, vma->vm_start, vma->vm_pgoff, vma->vm_end - vma->vm_start, vma->vm_page_prot)) return -EAGAIN;


    return 0;
}
```

pgprot_noncached() 是一个宏函数，实现高度依赖于 CPU 架构，另一个限制少一点的宏是 pgprot_writecombine()。pgprot_noncached() 实际禁止了相关页的 Cache 和写缓冲，pgprot_writecombine() 则没有禁止写缓冲。

#### fault() 函数

除 remap_pfn_range() 以外，在驱动程序中实现 VMA 的 fault() 函数可以为设备提供更加灵活的内存映射途径。当访问的页不在内存里，即发生缺页异常时，fault() 会被内核自动调用，而 fault() 的具体行为可以自定义。

当发生缺页异常的时候，系统会如下处理：

1. 找到缺页的虚拟地址所在的 VMA。
2. 如果必要，分配中间页目录表和页表。
3. 如果页表项对应的物理页面不存在，调用这个 VMA 的 fault() 方法，它返回物理页面的页描迏符。
4. 将物理页面的地址填充到页表中。

一个使用示例如下：

```c
static int xxx_fault(struct vm_area_struct *vma, struct vm_fault *vmf)
{
    unsigned long paddr;
    unsigned long pfn;
    pgoff_t index = vmf->pgoff;
    struct vma_data *vdata = vma->vm_private_data;

    ...

    pfn = paddr >> PAGE_SHIFT;

    vm_insert_pfn(vma, (unsigned long)vmf->virtual_address, pfn);

    return VM_FAULT_NOPAGE;
}
```

> 大多数设备驱动其实都不需要提供设备内存到用户空间的映射能力。因为对于串口等面向流的设备而言，实现这种映射毫无意义。而对于显示、视频等设备，建立映射可减少用户空间和内核空间之间的内存复制，这种情形下的映射是有意义的。

## DMA

**DMA 是一种无需 CPU 的参与就可以让外设与系统内存之间进行双向数据传输的硬件机制。DMA 使系统 CPU 从实际的 I/O 数据传输过程中摆脱出来，大大提高系统的吞吐率。**DMA 通常与硬件体系结构，特别是外设的总线技术密切相关。

DMA 的数据传输由 DMA 控制器（DMAC）控制，在传输期间，CPU 可以并发地执行其他任务。当 DMA 结束后，DMAC 通过中断通知 CPU 数据传输已经结束，然后由 CPU 执行相应的中断服务程序进行后处理。

### DMA 与 Cache 一致性

Cache 和 DMA 本身似乎是两个毫不相关的事物。Cache 被用作 CPU 针对内存的缓存，利用程序的空间和时间的局部性原理，达到较高的命中率，从而避免 CPU 每次都必须要与相对慢速的内存交互数据来提高数据的访问速率。DMA 可以作为内存与外设之间传输数据的方式，在这种传输方式之下，数据并不需要经过 CPU 中转。

若 DMA 针对内存的目的地址与 Cache 缓存的对象没有重叠区域，DMA 和 Cache 将相安无事。**若 DMA 的目的地址与 Cache 所缓存的内存地址有重叠，经过 DMA 操作，目的地址的内存已被修改，而 CPU 并不知道，它仍然认为 Cache 中的老数据就是内存中的数据，那在以后访问 Cache 映射的内存时，它仍然使用陈旧的 Cache 数据。这样就会发生 Cache 与内存之间数据“不一致性”的错误。**所谓 Cache 数据与内存数据的不一致性，是指在采用 Cache 的系统中，同样一个数据可能既存在于 Cache 中，也存在于主存中，Cache 与主存中的数据一样则具有一致性，数据若不一样则具有不一致性。

<img src="https://image.davidingplus.cn/images/2025/02/01/image-20241209172643859.png" alt="image-20241209172643859" style="zoom:65%;" />

> Cache 与内存的一致性问题经常被遗忘。在发生 Cache 与内存不一致性错误后，驱动将无法正常运行。Cache 的不一致性问题并不是只发生在 DMA 的情况下，实际上，还存在于 Cache 使能和关闭的时刻。

### DMA 编程

**DMA 本身不属于一种等同于字符设备、块设备和网络设备的外设，它只是一种外设与内存交互数据的方式。**故更合理的称呼方式是 DMA 编程而非 DMA 驱动。

**内存中用于与外设交互数据的区域称为 DMA 缓冲区。**在设备不支持 scatter/gather（分散/聚集，简称 SG）操作的情况下，DMA 缓冲区在物理上必须是连续的。

#### DMA 区域

对于 x86 架构的 ISA 设备而言，其 DMA 操作只能在 16 MB 以下的内存中进行，因此，在使用 kmalloc()、`__get_free_pages()` 及其类似函数申请 DMA 缓冲区时应使用 GFP_DMA 标志，这样能保证获得的内存位于 DMA 区域中，并具备 DMA 能力。

关于 ISA、PCI、PCIE 等总线协议的了解，参考：[https://blog.csdn.net/yinqiusheng/article/details/140387774](https://blog.csdn.net/yinqiusheng/article/details/140387774)

内核中定义了使用 GFP_DMA 标志的申请 DMA 缓冲区的快捷函数 `__get_dma_pages()`，定义如下：

```c
#define __get_dma_pages(gfp_mask, order) \
		__get_free_pages((gfp_mask) | GFP_DMA, (order))
```

如果不想使用参数 order 申请 DMA 内存，可使用另一个函数 dma_mem_alloc()，定义如下：

```c
static unsigned long dma_mem_alloc(unsigned long size)
{
    // get_order()：order = log2(size)
	return __get_dma_pages(GFP_KERNEL, get_order(size));
}
```

#### 虚拟地址、物理地址和总线地址

基于 DMA 的硬件使用的是总线地址而不是物理地址，总线地址是从设备角度上看到的内存地址，物理地址则是从 CPU MMU 控制器外围角度上看到的内存地址（从 CPU 核角度看到的是虚拟地址）。虽然在 PC 上，对于 ISA 和 PCI 而言，总线地址即为物理地址，但并不是每个平台都是如此。因为有时候接口总线通过桥接电路连接，桥接电路会将 I/O 地址映射为不同的物理地址。

内核提供了如下函数进行简单的虚拟地址/总线地址转换：

```c
unsigned long virt_to_bus(void *address)
{
	return (unsigned long)address;
}

void *bus_to_virt(unsigned long address)
{
	return (void *)address;
}
```

#### DMA 地址掩码

设备不一定能在所有的内存地址上执行 DMA 操作，在这种情况下应该通过下列函数执行 DMA 地址掩码：

```c
int dma_set_mask(struct device *dev, u64 mask);
```

这个函数的本质是修改 device 结构体中的 dma_mask 成员。在 device 结构体中，除了 dma_mask 以外，还有 coherent_dma_mask 成员。dma_mask 是设备 DMA 可寻址的范围，coherent_dma_mask 用作申请一致性 DMA 缓冲区。

```c
struct device {

    ...

	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask;/* Like dma_mask, but for
					     alloc_coherent mappings as
					     not all hardware supports
					     64 bit addresses for consistent
					     allocations such descriptors. */

    ...
};
```

#### 一致性 DMA 缓冲区

**DMA 映射包括两部分工作：分配一片 DMA 缓冲区；为这片缓冲区产生设备可访问的地址。**同时 DMA 映射也必须考虑 Cache 一致性问题。内核中提供了如下函数以分配一个 DMA 一致性的内存区域：

```c
// 返回申请到的 DMA 缓冲区的虚拟地址。
// 通过参数 dma_handle 返回 DMA 缓冲区的总线地址。
void *dma_alloc_coherent(struct device *dev, size_t size, dma_addr_t *dma_handle, gfp_t gfp)
{
	return dma_alloc_attrs(dev, size, dma_handle, gfp,
			(gfp & __GFP_NOWARN) ? DMA_ATTR_NO_WARN : 0);
}
```

dma_alloc_coherent() 申请一片 DMA 缓冲区，进行地址映射并保证该缓冲区的 Cache 一致性。对应的释放函数为 dma_free_coherent()。

```c
void dma_free_coherent(struct device *dev, size_t size, void *cpu_addr, dma_addr_t dma_handle)
{
	return dma_free_attrs(dev, size, cpu_addr, dma_handle, 0);
}
```

使用函数 dma_alloc_writecombine() 分配一个写合并（Writecombining）的 DMA 缓冲区，释放函数是 dma_free_coherent()。这两个接口在 Linux 5.15 以后已移除。

Linux 内核还提供了 PCI 设备申请 DMA 缓冲区的函数 pci_alloc_consistent()，释放函数是 pci_free_consistent()。

```c
void *pci_alloc_consistent(struct pci_dev *hwdev, size_t size, dma_addr_t *dma_handle)
{
	return dma_alloc_coherent(&hwdev->dev, size, dma_handle, GFP_ATOMIC);
}

void pci_free_consistent(struct pci_dev *hwdev, size_t size, void *vaddr, dma_addr_t dma_handle)
{
	dma_free_coherent(&hwdev->dev, size, vaddr, dma_handle);
}
```

> dma_alloc_xxx() 函数虽然以 `dma_alloc_` 开头，但是申请的区域不一定在 DMA 区域里。以 32 位 ARM 处理器为例，当 coherent_dma_mask 小于 0xffffffff 时，才会设置 GFP_DMA 标记，并从 DMA 区域申请内存。

#### 流式 DMA 映射

**并不是所有的 DMA 缓冲区都是驱动申请的，如果是驱动申请的，用一致性 DMA 缓冲区自然最方便，这直接考虑了 Cache 一致性问题。**但在许多情况下，缓冲区来自内核的较上层（如网卡驱动中的网络报文、块设备驱动中要写入设备的数据等），上层很可能用普通的 kmalloc()、`__get_free_pages()` 等方法申请，这时就要使用流式 DMA 映射。使用步骤一般如下：

1. 进行流式 DMA 映射。
2. 执行 DMA 操作。
3. 进行流式 DMA 去映射（去掉映射）。

**流式 DMA 映射操作本质上大多是进行 Cache 的使无效或清除操作，以解决 Cache 一致性问题。**

##### 单一缓冲区下的流式 DMA 映射

对于单个已经分配的缓冲区而言，使用 dma_map_single() 可实现流式 DMA 映射。

```c
#define dma_map_single(d, a, s, r) dma_map_single_attrs(d, a, s, r, 0)

// 映射成功返回总线地址，失败返回 NULL。
// dir：DMA 的方向，包括 DMA_TO_DEVICE、DMA_FROM_DEVICE、DMA_BIDIRECTIONAL 和 DMA_NONE 等。
dma_addr_t dma_map_single_attrs(struct device *dev, void *ptr, size_t size, enum dma_data_direction dir, unsigned long attrs)
{
	/* DMA must never operate on areas that might be remapped. */
	if (dev_WARN_ONCE(dev, is_vmalloc_addr(ptr),
			  "rejecting DMA map of vmalloc memory\n"))
		return DMA_MAPPING_ERROR;
	debug_dma_map_single(dev, ptr, size);
	return dma_map_page_attrs(dev, virt_to_page(ptr), offset_in_page(ptr),
			size, dir, attrs);
}
```

dma_map_single() 对应的去映射函数是 dma_unmap_single()。

```c
#define dma_unmap_single(d, a, s, r) dma_unmap_single_attrs(d, a, s, r, 0)

void dma_unmap_single_attrs(struct device *dev, dma_addr_t addr,
		size_t size, enum dma_data_direction dir, unsigned long attrs)
{
	return dma_unmap_page_attrs(dev, addr, size, dir, attrs);
}
```

通常情况下，设备驱动不应访问未映射的流式 DMA 缓冲区。如果一定要这么做，可使用如下函数获得 DMA 缓冲区的拥有权。

```c
void dma_sync_single_for_cpu(struct device *dev, dma_addr_t addr, size_t size, enum dma_data_direction dir);
```

在驱动访问完 DMA 缓冲区后，使用如下函数将其所有权返还给设备。

```c
void dma_sync_single_for_device(struct device *dev, dma_addr_t addr, size_t size, enum dma_data_direction dir);
```

##### SG 映射

如果设备要求较大的 DMA 缓冲区，在其支持 SG 模式的情况下，申请多个相对较小不连续的 DMA 缓冲区通常是防止申请太大的连续物理空间的方法。使用函数 dma_map_sg() 申请，对应的释放函数是 dma_unmap_sg()。

```c
#define dma_map_sg(d, s, n, r) dma_map_sg_attrs(d, s, n, r, 0)

// 函数返回 DMA 缓冲区的数量，可能小于 nents。
// nents：散列表（scatterlist）入口的数量。
// 对于 scatterlist 的每个项目，dma_map_sg() 为设备产生恰当的总线地址，它会合并物理上临近的内存区域。
unsigned int dma_map_sg_attrs(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction dir, unsigned long attrs)
{
	int ret;

	ret = __dma_map_sg_attrs(dev, sg, nents, dir, attrs);
	if (ret < 0)
		return 0;
	return ret;
}
EXPORT_SYMBOL(dma_map_sg_attrs);

#define dma_unmap_sg(d, s, n, r) dma_unmap_sg_attrs(d, s, n, r, 0)

void dma_unmap_sg_attrs(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction dir, unsigned long attrs)
{
	const struct dma_map_ops *ops = get_dma_ops(dev);

	BUG_ON(!valid_dma_direction(dir));
	debug_dma_unmap_sg(dev, sg, nents, dir);
	if (dma_map_direct(dev, ops) ||
	    arch_dma_unmap_sg_direct(dev, sg, nents))
		dma_direct_unmap_sg(dev, sg, nents, dir, attrs);
	else if (ops->unmap_sg)
		ops->unmap_sg(dev, sg, nents, dir, attrs);
}
EXPORT_SYMBOL(dma_unmap_sg_attrs);
```

scatterlist 结构体定义如下，包含与 scatterlist 对应的页结构体指针、缓冲区在页中的偏移 offset、缓冲区长度 length 以及总线地址 dma_address。

```c
struct scatterlist {
	unsigned long	page_link;
	unsigned int	offset;
	unsigned int	length;
	dma_addr_t	dma_address;
#ifdef CONFIG_NEED_SG_DMA_LENGTH
	unsigned int	dma_length;
#endif
};
```

执行 dma_map_sg() 后，通过 sg_dma_address() 返回 scatterlist 对应缓冲区的总线地址，sg_dma_len() 返回 scatterlist 对应缓冲区的长度。

```c
#define sg_dma_address(sg)	((sg)->dma_address)

#define sg_dma_len(sg)		((sg)->dma_length)
```

同单一缓冲区的情况，如果设备驱动一定要访问未映射的 SG 缓冲区，应先调用函数 dma_sync_sg_for_cpu()，归还所有权的函数是 dma_sync_sg_for_device()。

```c
void dma_sync_sg_for_cpu(struct device *dev, struct scatterlist *sg, int nelems, enum dma_data_direction dir);

void dma_sync_sg_for_device(struct device *dev, struct scatterlist *sg, int nelems, enum dma_data_direction dir);
```

#### dmaengine 标准 API

推荐使用 dmaengine 的驱动架构来编写 DMA 控制器的驱动，外设的驱动使用标准的 dmaengine API 进行 DMA 的准备、发起和完成时的回调工作。

和中断一样，在使用 DMA 前，设备驱动程序需首先向 dmaengine 系统申请 DMA 通道，申请 DMA 通道的函数如下：

```c
/* Deprecated, please use dma_request_chan() directly */
struct dma_chan * __deprecated dma_request_slave_channel(struct device *dev, const char *name)
{
	struct dma_chan *ch = dma_request_chan(dev, name);

	return IS_ERR(ch) ? NULL : ch;
}

struct dma_chan *dma_request_chan(struct device *dev, const char *name);
```

对应的释放通道的函数是 dma_release_channel()。

```c
void dma_release_channel(struct dma_chan *chan);
```

下面是利用 dmaengine API 发起一次 DMA 操作的示例：

```c
static void xxx_dma_fini_callback(void *data)
{
    struct completion *dma_complete = data;

    complete(dma_complete);
}

issue_xxx_dma(...)
{
    // 通过 dmaengine_prep_slave_single() 准备好一些 DMA 描述符。
    rx_desc = dmaengine_prep_slave_single(xxx->rx_chan, xxx->dst_start, t->len, DMA_DEV_TO_MEM, DMA_PREP_INTERRUPT | DMA_CTRL_ACK);

    // 填充完成回调为 xxx_dma_fini_callback()。
    rx_desc->callback = xxx_dma_fini_callback;
    rx_desc->callback_param = &xxx->rx_done;

    // 通过 dmaengine_submit() 把这个描述符插入队列。
    dmaengine_submit(rx_desc);

    // 通过 dma_async_issue_pending() 发起这次 DMA 动作。完成后 xxx_dma_fini_callback() 函数会被 dmaengine 驱动自动调用。
    dma_async_issue_pending(xxx->rx_chan);
}
```

## 小结

外设可处于 CPU 的内存空间和 I/O 空间。除 x86 外，嵌入式处理器一般只存在内存空间。Linux 为 I/O 内存和 I/O 端口的访问提高了一套统一的方法，访问流程一般为**申请资源->映射->访问->去映射->释放资源**。

对于有 MMU 的处理器而言，Linux 的内部布局比较复杂，可直接映射的物理内存称为常规内存，超出部分为高端内存。kmalloc() 和 `__get_free_pages()` 申请的内存在物理上连续，vmalloc() 申请的内存在物理上不连续。

DMA 操作可能导致 Cache 不一致性的问题，故对于 DMA 缓冲，应使用 dma_alloc_coherent() 等方法申请。在 DMA 操作中涉及总线地址、物理地址和虚拟地址等概念，区分这 3 类地址非常重要。

# 块设备驱动

见文章 [https://blog.davidingplus.cn/posts/69869e7f.html](https://blog.davidingplus.cn/posts/69869e7f.html)。

