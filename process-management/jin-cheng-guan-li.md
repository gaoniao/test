# 进程介绍

## 进程的组成 

一个进程由一个地址空间和内核内部的一组数据结构共同组成。

地址空间是由内核标记出来供进程使用的一组内存页面。它包含正在执行的代码和库、进程变量、进程栈以及在进程正在运行时内核所需要的各种其他信息。 因为 UNIX 和 Linux 都是采用虚拟内存的系统，因此一个内存页面在进程的地址空间中的位置和它在机器的物理内存或交换空间中的位置之间没有关系。

内核的内部数据结构记录了有关每个进程的各种信息，其中非常重要的信息有：

* 进程的地址空间映射。
* 进程的当前状态（睡眠状态、停止状态、可运行状态等）。
* 进程执行的优先级。
* 进程已用资源的信息。
* 进程打开的文件和网络端口的信息。
* 进程的信号掩码（一个记录，确定要封锁那些信号）。
* 进程的属主。

一个执行线程（execution thread）通常简称为一个线程（thread），它是在一个进程内执行一次 fork 的结果。线程继承包含它的进程的许多属性（例如，进程的地址空间），多个线程在同一个进程内按照一种称为多线程（multithreading）的机制并发执行。在老实单处理器系统的内核可以模拟并发执行（concurrent execution），而在多核和多 CPU 体系结构上，多个线程可以在不同的核心（core）上同时运行。

### PID：进程的 ID 号

内核给每个进程分配一个独一无二的 ID 号。

控制进程的大多数命令和系统调用需要用户指定 PID 来标识操作的目标。PID 按照创建进程的顺序来分配。

### PPID：父 PID

Linux 没有提供创建新进程去运行某个特定程序的系统调用，现有进程必须克隆自身去创建一个新进程。克隆出的进程能够把它在运行的那个程序替换成另一个不同的程序。

当一个进程被克隆时，原来的进程就叫做父进程，而克隆出的副本则叫做子进程。进程的 PPID 属性就是克隆它的父进程的 PID。

当遇到无法辨认（以及可能运行失常）的进程时，父进程 PID 就成了一项很有用的信息。回溯该进程的来源（是一个 shell 还是另一个程序），就能更好的了解它的目的和作用。

### UID 和 EUID：真实的和有效的用户 ID

进程的 UID 就是其创建者的用户标识号，或者更确切地说，就是复制了父进程的 UID 值。  
通常，只允许创建者（属主）和超级用户对进程进行操作。

EUID 是“有效的（effective）”用户 ID，这是一个额外的 UID，用来确定进程在任何给定的时刻对哪些资源和文件具有访问权限，使得运行程序的用户拥有该程序的有效用户权限。 对大多数进程来说，UID 和 EUID 是一样的，例外的情况是 setuid 程序。

为什么同时采用 UID 和 EUID 呢？这是因为要保持标识和权限之间的区别，还因为 setuid 程序可能不太希望一直以扩大了的权限运行。在大多数系统上，可以设置和重置进程的有效 UID 以便启用或限制它所共享有的额外权限。

因此，编写严谨的 setuid 程序可以让其大部分执行操作与它的特殊权限无关，只有在需要额外权限的特定时刻才用上它们。

以下通过一个典型问题和代码实例来说明 UID 和 EUID 的区别：

Linux 系统中每个普通用户都可以更改自己的密码，这是合理的设置。

然而，用户的信息保存在 /etc/passwd 文件中，用户的密码保存在 /etc/shadow 文件中。

```bash
[root@k8s-m1 opt]$ ll /etc/{passwd,shadow}
-rw-r--r-- 1 root root 905 Nov  7 20:44 /etc/passwd
-r-------- 1 root root 749 Nov  7 20:44 /etc/shadow
```

可以看出 /etc/passwd 文件只有 root 用户才有写权限，普通用户只有读权限，并没有写权限。/etc/shadow 文件只有 root 用户有读权限，普通用户没有任何权限。

所以，普通用户为什么可以更改自己的密码？

其实，用户可以更改密码不在于文件的权限，而在于更改密码的命令 passwd。

```bash
[root@k8s-m1 opt]$ ll /usr/bin/passwd 
-rwsr-xr-x. 1 root root 27832 Jun 10  2014 /usr/bin/passwd
```

发现，passwd 命令有一个特殊的权限标记 s， 这就是用户可以更改密码的原因所在。因为 passwd 程序的属主是 root，并且它被设置了 setuid 标志，这个标志表示，任何普通用户运行 passwd 程序时，其有效用户就是该程序的属主。

那么，根据有效用户的含义，任何运行 su 程序的普通用户都能访问 /etc/passwd 文件。有效用户为 root 进程称为特权进程（privileged processes）。类似的程序还有 su 、sudo 等。

> 注：在 Linux 中使用 passwd 设置或更改用户密码，是先写入到 /etc/passwd 文件然后通过 pwconv 命令转换到 /etc/shadow 文件，执行 pwunconv 命令可观察到转换前效果，会观察到 /etc/shadow 文件消失了（会生成备份文件 /etc/shadow-），而 /etc/passwd 文件中原来 x 的地方变成了真正的加密密码。
>
> ```bash
> # 查看 passwd 文件内的密码位为 x
> [root@k8s-m1 opt]$ cat /etc/passwd
> root:x:0:0:root:/root:/bin/bash
> ...
> gaopeng:x:1000:1000::/home/gaopeng:/bin/bash
>
> # 执行 pwunconv 命令观察转换前的效果
> # 密码加密后直接写入 passwd 文件
> [root@k8s-m1 opt]$ pwunconv && cat /etc/passwd | sed -n '1p;$p' 
> root:$6$4wYFKD3M$.fisd6w9t7UQ.5u9ep0G0uKxdAeKR3LOZolmxcrRRpx4QnlfESkSQqWYgZ/9DGpbYOLVvXvFb17OLsYb/skHu0:0:0:root:/root:/bin/bash
> gaopeng:$6$YrSGNUFu$6DVBtSC6D21CFhQqtiB2i8I2hIC3MCNL691WR6IlxTMxXiBxQEwzH3dcC5gji1SOno4Y7ekQ5Wiy85DzafHzo0:1000:1000::/home/gaopeng:/bin/bash
> # shadow 文件消失
> [root@k8s-m1 opt]$ ll /etc/shadow
> ls: cannot access /etc/shadow: No such file or directory
>
> # 执行 pwconv 生成 shadow 影子文件
> [root@k8s-m1 opt]$ pwconv && cat /etc/passwd | awk 'NR==1  { print } END { print }'
> root:x:0:0:root:/root:/bin/bash
> gaopeng:x:1000:1000::/home/gaopeng:/bin/bash
> [root@k8s-m1 opt]$ ll /etc/shadow
> -r-------- 1 root root 749 Nov  7 20:44 /etc/shadow
> ```

再举个例子：

以普通用户创建一个文件，这个文件的属主与属组都为普通用户。

```bash
[gaopeng@k8s-m1 tmp]$ touch test
-rw-rw-r-- 1 gaopeng gaopeng 0 Nov  7 21:58 test
```

如果将 touch 程序设置了 setuid 标志位，再进行创建呢？

```bash
# 为 touch 程序添加 setuid 标志
[root@k8s-m1 opt]$ chmod u+s /usr/bin/touch
# 查看 touch 程序的属主为 root，并且有 setuid 标志
[root@k8s-m1 opt]$ ll /usr/bin/touch
-rwsr-xr-x. 1 root root 62488 Apr 11  2018 /usr/bin/touch
# 切换到普通用户再创建文件
[root@k8s-m1 opt]$ su gaopeng
[gaopeng@k8s-m1 tmp]$ touch test_1
[gaopeng@k8s-m1 tmp]$ ll test_1
-rw-rw-r-- 1 root gaopeng 0 Nov  7 22:00 test_1
```

发现创建的文件的属主变成了 touch 程序的属主 root，所以程序添加了 setuid 标志位时，普通用户执行此程序时，有效用户实际是此程序的 EUID。通俗说，如果 touch 程序设置了 setuid 标志位，当普通用户执行 touch 时，那么普通用户就会变成 touch 的属主 root 用户。

所以进程中的 UID 就是用户的 UID，用来标识用户。进程中的 EUID 是有效的用户 ID，用来判断权限的问题 。 

所以一个进程如果没有设置 setuid 标志，那么 EUID = UID。如果设置了 setuid 标志，则 EUID = 程序的属主。

### GID 和 EGID：真实的和有效的组 ID

GID 就是进程的组标识号。

EGID 与 GID 的关系和 EUID 与 UID 的关系相同，EGID 也可以用 setgid 标志来固定为程序的属组。

为了规定访问权限，一个进程可以同时是多个组的成员。组的完整列表与 GID 和 EGID 分开保存。判断访问权限一般要考虑 EGID 和补充的组清单，不考虑 GID。

只有在一个进程要创建新文件的时候，GID 才会起作用。根据文件系统的权限设定情况，新文件可能要采用创建该文件的进程的 GID。

所以一个进程如果没有设置 setgid 标志，那么 EGID = GID。如果设置了 setgid 标志，则 EGID = 程序的属组。

### 谦让度

进程的调度优先级决定了它所接受到的 CPU 时间有多少。

内核使用动态算法来计算优先级：它考虑一个进程近来已经消耗 CPU 的时间量以及考虑该进程已经等待运行的时间因素。

内核还会关注为管理目的而设置的值，这种值通常叫做“谦让值（nice vaule）”或“谦让度（niceness）”，之所以这么叫是因为它表明了管理员计划对待系统其它用户的友好程度。

为了给低延迟的应用提供更好的支持，Linux 向传统的 UNIX 调度模型中增加了“调度类（scheduling class）”的概念。目前有三种调度类，每个进程都属于一种调度类。遗憾的是，实时类（real-time class）既没有得到广泛使用，命令行对它的支持也不好。系统进程都使用传统的（基于谦让度）调度机制。

### 控制终端

大多数不是守护进程（daemon）的进程都有一个与自己相关联的控制终端。

控制终端决定了标准输入、标准输出和标准错误通道的默认链接位置。当用户从 shell 启动一个命令时，它的终端通常就成为该进程的控制终端。

