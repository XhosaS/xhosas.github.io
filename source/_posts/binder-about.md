---
title: binder about
date: 2022-04-10 16:52:46
tags:
---

# 前言

Binder 绝对是 Android 开发中绝对无法轻视的关键组件。作为一名 Android 开发工程师，在工作中遇到过非常多 Binder 相关的问题，也无数次的翻看过 Binder 相关资料。回顾过去，我发现随着工作经验的渐涨，自己每次学习 Binder 都会有更深的理解。过去的自己对 Binder 的理解只是简单的停留在如何使用，以及追过源码而已。对于 Binder 为什么要这样设计不求甚解。于是我想从 Binder 实现的角度出发，深入浅出的分享一下 Android Binder 的设计原理。这次分享会尽量的不牵涉到源码阅读。

# 概述

Binder在Android系统中地位非常高。在Zygote孵化出system_server进程后，在system_server进程中出初始化支持整个Android framework的各种各样的Service，而这些Service从大的方向来划分，分为Java层Framework和Native Framework层(C++)的Service，几乎都是基于BInder IPC机制。

1. **Java framework：作为Server端继承(或间接继承)于Binder类，Client端继承(或间接继承)于BinderProxy类。**例如 ActivityManagerService(用于控制Activity、Service、进程等) 这个服务作为Server端，间接继承Binder类，而相应的ActivityManager作为Client端，间接继承于BinderProxy类。 当然还有PackageManagerService、WindowManagerService等等很多系统服务都是采用C/S架构；
2. **Native Framework层：这是C++层，作为Server端继承(或间接继承)于BBinder类，Client端继承(或间接继承)于BpBinder。**例如MediaPlayService(用于多媒体相关)作为Server端，继承于BBinder类，而相应的MediaPlay作为Client端，间接继承于BpBinder类。

# 为什么 Android 要采用 Binder 作为 IPC 机制？
## Linux 传统的 IPC 方式
### 信号
不适用于信息交换，更适用于进程中断控制，比如非法内存访问，杀死某个进程等。

### 管道
在创建时分配一个page大小的内存，缓存区大小比较有限。管道通信面向字节流，传递之后还要在接收端做解析。管道只允许单向通信。

### 套接字（socket)
作为更通用的接口，传输效率低，主要用于不同机器或跨网络的通信。

### 消息队列
信息复制两次，额外的CPU消耗；不合适频繁或信息量大的通信。

![消息队列](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/消息队列.png)	

### 共享内存
无须复制，共享缓冲区直接付附加到进程虚拟地址空间，速度快；但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决。

![共享内存](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/共享内存.png)

### 信号量

常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。


## Binder

**Android的内核也是基于Linux内核，为何不直接采用Linux现有的进程IPC方案呢，难道Linux社区那么多优秀人员都没有考虑到有Binder这样一个更优秀的方案，是google太过于牛B吗？事实是真相并非如此。**


### 从性能角度
Binder数据拷贝只需要一次，而管道、消息队列、Socket都需要2次，但共享内存方式一次内存拷贝都不需要；从性能角度看，Binder性能仅次于共享内存。

### 从稳定性角度
Binder是基于C/S架构的，而共享内存实现方式复杂，没有客户与服务端之别， 需要充分考虑到访问临界资源的并发同步问题，否则可能会出现死锁等问题；从这稳定性角度看，Binder架构优越于共享内存。

### 从安全的角度
传统Linux IPC的接收方只能由用户在数据包里填入UID/PID，无法获得对方进程可靠的UID/PID，从而无法鉴别对方身份；而Android作为一个开放的开源体系，拥有非常多的开发平台，App来源甚广，因此手机的安全显得额外重要；对于普通用户，绝不希望从App商店下载偷窥隐射数据、后台造成手机耗电等等问题，传统Linux IPC无任何保护措施，完全由上层协议来确保。
Android为每个安装好的应用程序分配了自己的UID，故进程的UID是鉴别进程身份的重要标志，Android系统中对外只暴露Client端，Client端将任务发送给Server端，Server端会根据权限控制策略，判断UID/PID是否满足访问权限。

### 从语言层面的角度
大家多知道Linux是基于C语言(面向过程的语言)，而Android是基于Java语言(面向对象的语句)，而对于Binder恰恰也符合面向对象的思想，将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法，而其独特之处在于Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中。可以从一个进程传给其它进程，让大家都能访问同一Server，就像将一个对象或引用赋值给另一个引用一样。Binder模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行于同一个面向对象的程序之中。从语言层面，Binder更适合基于面向对象语言的Android系统，对于Linux系统可能会有点“水土不服”。
另外，Binder是为Android这类系统而生，而并非Linux社区没有想到Binder IPC机制的存在，对于Linux社区的广大开发人员，我还是表示深深佩服，让世界有了如此精湛而美妙的开源系统。也并非Linux现有的IPC机制不够好，相反地，经过这么多优秀工程师的不断打磨，依然非常优秀，每种Linux的IPC机制都有存在的价值，同时在Android系统中也依然采用了大量Linux现有的IPC机制，根据每类IPC的原理特性，因时制宜，不同场景特性往往会采用其下最适宜的。比如在Android OS中的Zygote进程的IPC采用的是Socket（套接字）机制，Android中的Kill Process采用的signal（信号）机制等等。而Binder更多则用在system_server进程与上层App层的IPC交互。

### 从公司战略的角度
总所周知，Linux内核是开源的系统，所开放源代码许可协议GPL保护，该协议具有“病毒式感染”的能力，怎么理解这句话呢？受GPL保护的Linux Kernel是运行在内核空间，对于上层的任何类库、服务、应用等运行在用户空间，一旦进行SysCall（系统调用），调用到底层Kernel，那么也必须遵循GPL协议。 而Android 之父 Andy Rubin对于GPL显然是不能接受的，为此，Google巧妙地将GPL协议控制在内核空间，将用户空间的协议采用Apache-2.0协议（允许基于Android的开发商不向社区反馈源码），同时在GPL协议与Apache-2.0之间的Lib库中采用BSD证授权方法，有效隔断了GPL的传染性，仍有较大争议，但至少目前缓解Android，让GPL止步于内核空间，这是Google在GPL Linux下开源与商业化共存的一个成功典范。

### 从历史角度
Binder是基于开源的 OpenBinder实现的，OpenBinder是一个开源的系统IPC机制,最初是由 Be Inc. 开发，接着由Palm, Inc.公司负责开发，现在OpenBinder的作者在Google工作，既然作者在Google公司，在用户空间采用Binder作为核心的IPC机制，再用Apache-2.0协议保护，自然而然是没什么问题，减少法律风险，以及对开发成本也大有裨益的，那么从公司战略角度，Binder也是不错的选择。另外，再说一点关于OpenBinder，在2015年OpenBinder以及合入到Linux Kernel主线 3.19版本，这也算是Google对Linux的一点回馈吧。

# Binder 的设计

Binder数据拷贝只需要一次，充分考虑到访问临界资源的并发同步问题，能够获得对方进程可靠的UID/PID，面向对象，隔离 Linux。

## Binder Driver
这里我们先从 Linux 中进程间通信涉及的一些基本概念开始介绍，然后逐步展开，向大家说明传统的进程间通信的原理。
### 进程隔离

![kernel_ram0](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/kernel_ram0.jpg)


简单的说就是操作系统中，进程与进程间内存是不共享的。两个进程就像两个平行的世界，A 进程没法直接访问 B 进程的数据，这就是进程隔离的通俗解释。A 进程和 B 进程之间要进行数据交互就得采用特殊的通信机制：进程间通信（IPC)。


### 进程空间划分：用户空间(User Space)/内核空间(Kernel Space)
现在操作系统都是采用的虚拟存储器，拿 32 位系统举例，它的寻址空间（虚拟存储空间）就是 2 的 32 次方，也就是 4GB。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也可以访问底层硬件设备的权限。为了保护用户进程不能直接操作内核，保证内核的安全，操作系统从逻辑上将虚拟空间划分为用户空间（User Space）和内核空间（Kernel Space）。针对 Linux 操作系统而言，将最高的 1GB 字节供内核使用，称为内核空间；较低的 3GB 字节供各进程使用，称为用户空间。

内核空间（Kernel）是系统内核运行的空间，用户空间（User Space）是用户程序运行的空间。为了保证安全性，它们之间是隔离的

![kernel_ram1](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/kernel_ram1.png)

### 系统调用：用户态与内核态
虽然从逻辑上进行了用户空间和内核空间的划分，但不可避免的用户空间需要访问内核资源，比如文件操作、访问网络等等。为了突破隔离限制，就需要借助系统调用来实现。系统调用是用户空间访问内核空间的唯一方式，保证了所有的资源访问都是在内核的控制下进行的，避免了用户程序对系统资源的越权访问，提升了系统安全性和稳定性。Linux 使用两级保护机制：0 级供系统内核使用，3 级供用户程序使用。当一个任务（进程）执行系统调用而陷入内核代码中执行时，称进程处于内核运行态（内核态）。此时处理器处于特权级最高的（0级）内核代码中执行。当进程处于内核态时，执行的内核代码会使用当前进程的内核栈。每个进程都有自己的内核栈。当进程在执行用户自己的代码的时候，我们称其处于用户运行态（用户态）。此时处理器在特权级最低的（3级）用户代码中运行。系统调用主要通过如下两个函数来实现：
```
copy_from_user() //将数据从用户空间拷贝到内核空间
copy_to_user() //将数据从内核空间拷贝到用户空间
```
理解了上面的几个概念，我们再来看看传统的 IPC 方式中，进程之间是如何实现通信的。通常的做法是消息发送方将要发送的数据存放在内存缓存区中，通过系统调用进入内核态。然后内核程序在内核空间分配内存，开辟一块内核缓存区，调用 copyfromuser() 函数将数据从用户空间的内存缓存区拷贝到内核空间的内核缓存区中。同样的，接收方进程在接收数据时在自己的用户空间开辟一块内存缓存区，然后内核程序调用 copytouser() 函数将数据从内核缓存区拷贝到接收进程的内存缓存区。这样数据发送方进程和数据接收方进程就完成了一次数据传输，我们称完成了一次进程间通信。如下图：

![kernel_ram2](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/kernel_ram2.jpg)

这种传统的 IPC 通信方式有两个问题：

1. 性能低下，一次数据传递需要经历：内存缓存区 --> 内核缓存区 --> 内存缓存区，需要 2 次数据拷贝；
2. 接收数据的缓存区由数据接收进程提供，但是接收进程并不知道需要多大的空间来存放将要传递过来的数据，因此只能开辟尽可能大的内存空间或者先调用 API 接收消息头来获取消息体的大小，这两种做法不是浪费空间就是浪费时间。

### 动态内核可加载模块
正如前面所说，跨进程通信是需要内核空间做支持的。传统的 IPC 机制如管道、Socket 都是内核的一部分，因此通过内核支持来实现进程间通信自然是没问题的。但是 Binder 并不是 Linux 系统内核的一部分，那怎么办呢？这就得益于 Linux 的动态内核可加载模块（Loadable Kernel Module，LKM）的机制；模块是具有独立功能的程序，它可以被单独编译，但是不能独立运行。它在运行时被链接到内核作为内核的一部分运行。这样，Android 系统就可以通过动态添加一个内核模块运行在内核空间，用户进程之间通过这个内核模块作为桥梁来实现通信。
在 Android 系统中，这个运行在内核空间，负责各个用户进程通过 Binder 实现通信的内核模块就叫 Binder 驱动（Binder Dirver）。
那么在 Android 系统中用户进程之间是如何通过这个内核模块（Binder 驱动）来实现通信的呢？难道是和前面说的传统 IPC 机制一样，先将数据从发送方进程拷贝到内核缓存区，然后再将数据从内核缓存区拷贝到接收方进程，通过两次拷贝来实现吗？显然不是，否则也不会有开篇所说的 Binder 在性能方面的优势了。

### 内存映射
Binder IPC 机制中涉及到的内存映射通过 mmap() 来实现，mmap() 是操作系统中一种内存映射的方法。内存映射简单的讲就是将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。内存映射能减少数据拷贝次数，实现用户空间和内核空间的高效互动。两个空间各自的修改能直接反映在映射的内存区域，从而被对方空间及时感知。也正因为如此，内存映射能够提供对进程间通信的支持。

### Binder Driver 实现原理

Binder IPC 正是基于内存映射（mmap）来实现的。

![binder_physical_memory](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/binder_physical_memory.jpg)

虚拟进程地址空间(vm_area_struct)和虚拟内核地址空间(vm_struct)都映射到同一块物理内存空间。当Client端与Server端发送数据时，Client（作为数据发送端）先从自己的进程空间把IPC通信数据`copy_from_user`拷贝到内核空间，而Server端（作为数据接收端）与内核共享数据，不再需要拷贝数据，而是通过内存地址空间的偏移量，即可获悉内存地址，整个过程只发生一次内存拷贝。一般地做法，需要Client端进程空间拷贝到内核空间，再由内核空间拷贝到Server进程空间，会发生两次拷贝。

对于进程和内核虚拟地址映射到同一个物理内存的操作是发生在数据接收端，而数据发送端还是需要将用户态的数据复制到内核态。到此，可能有人会好奇，为何不直接让发送端和接收端直接映射到同一个物理空间，那样就连一次复制的操作都不需要了，0次复制操作那就与Linux标准内核的共享内存的IPC机制没有区别了，对于共享内存虽然效率高，但是对于多进程的同步问题比较复杂，而管道/消息队列等IPC需要复制2两次，效率较低。

下面这图是从Binder在进程间数据通信的流程图，从图中更能明了Binder的内存转移关系。

![binder_memory_map](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/binder_memory_map.jpg)




## Service Manager
一次完整的进程间通信必然至少包含两个进程，通常我们称通信的双方分别为客户端进程（Client）和服务端进程（Server），由于进程隔离机制的存在，通信双方必然需要借助 Binder Driver 来实现。对于 Android 来说，Binder 的 Client 端和 Server 端是多对多的关系。一个 App Client 端会同时绑定多个 Server 端，同样的，一个 Service 也会同时服务于多个 Client 端。为了降低开发和性能负担，我们需要一个管理类来管理所有 Binder 的 Server——ServiceManager ，这个类负责注册 Server，查询 Server 以及 Server 的死亡通知。需要注意的是此处的Service Manager是指Native层的ServiceManager（C++），并非指framework层的ServiceManager(Java)。

ServiceManager是Binder IPC通信过程中的守护进程，本身也是一个Binder Server，当Service Manager启动之后，Client端和Server端通信时都需要先获取Service Manager接口，才能开始通信服务。


细心的同学肯定可能会发现问题，ServierManager 是一个进程，Server 是另一个进程，Server 向 ServiceManager 中注册 Binder 必然涉及到进程间通信。当前实现进程间通信又要用到进程间通信，这就好像蛋可以孵出鸡的前提却是要先找只鸡下蛋！

先让我们来回想一下古老的电话机，如果A要给B打电话，必须先连接通话中心，说明给我接通B的电话，这个时候接线员帮他呼叫B，连接建立，就完成了A与B的通信。另外，光有电话和接线员是不可能完成通信的，没有基站的支持，信息根本就无法传达。所以，表面上是 A 与 B 的直接通信，但实际上除了通信双方以外还有两个隐藏角色：通话中心和基站。在 Binder 的架构中，Binder Driver 就是基站，而 Service Manager 就是通话中心。

整个通信步骤如下：

1. 装载 Binder Driver（建立基站）；在Kernel启动时，将驱动装载到 `/dev/binder`。

2. Service Manager 建立(建立通话中心)；首先有一个进程向驱动提出申请为 Service Manager，驱动同意之后，Service Manager 进程负责管理 Service。不过这时候通信录还是空的，一个号码也没有。
3. 各个 Service 向 Service Manager 注册（填充通信录）；每个 Server 端启动后，向 Service Manager 报告，我是张三，要找我请返回 1234 这个号码；其他Server进程依次如此；这样SM就建立了一张表，对应着各个Server的名字和地址。
4. Client想要与Server通信，首先询问Service Manager；请告诉我如何联系张三，Service Manager收到后给他一个号码1234；Client收到之后，开心的用这个号码拨通了Server的电话。

![IPC-Binder](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/IPC-Binder.jpg)

现在回到那个先有鸡还是先有蛋的话题，Service Manager 是一个进程，Server 是另一个进程，Server 向 Service Manager 中注册 Binder 时是如何与 Service Manager 建立起跨进程通信的呢？答案很简单，通话中心的电话号码是固定的。我们只需要（向基站）拨打固定的电话号码，就可以与通话中心建立通信。在源码中，不管是注册还是获取 Binder 服务，都需要先调用 `defaultServiceManager() ` 来获取 gDefaultServiceManager 这个单例对象，通过这个单例对象来实现注册或者获取 Binder 服务的功能。`defaultServiceManager()` 等价于 `new BpServiceManager(new BpBinder(0));`，0 就是 Service Manager 的固定地址。

Service Manager 没有采用 `libbinder` 中的多线程模型来与 Binder 驱动通信，而是自行编写了`binder.c`直接和Binder驱动来通信，并且只有一个循环`binder_loop`来进行读取和处理事务，这样的好处是简单而高效。



## 注册 Binder 服务（addService）

经过上面对基础服务的一系列介绍，接下来我们就要开始研究如何注册一个 Binder 服务了。

1. 首先，我们需要打开 `/dev/binder` 设备，建立与内核的 Binder 驱动的交互通道；
2. 紧接着，我们需要使用 `mmap()` 函数建立与内核空间的映射；
3. 我们还需要定义一个抽象接口类封装 Server 向外暴露的所有功能，由于这些函数需要跨进程调用，须为其一一编号，从而Server可以根据收到的编号决定调用哪个函数；
4. 然后采用继承方式以接口类和Binder抽象类为基类构建Binder在Server中的实体，集成接口类是为了实现我们定义的所有功能，继承 Binder 抽象类是为了让这个类能够在驱动传递过来消息后通过 `onTransact()`函数接收到事件。然后将这个实体类的指针传入驱动；
5. 创建一个线程池，用于接收Client端传入的命令，并交予Binder实体类处理。



值得注意的是，我们这里调用的 `mmap()` 与普通的 `mmap()` 不同。因为普通的 `mmap()` 通常用在有物理存储介质的文件系统上，比如说 `mmap()` 一个硬盘上的文件。而像 Binder 这样没有物理介质，纯粹用来通信的字符驱动设备通常没有必要支持 `mmap()`。Binder驱动当然不是为了在物理介质和用户空间做映射，而是用来创建数据接收的缓存空间。当 Server 调用 `mmap()` 时，会创建一个隶属于 Server 进程内存空间中的内存。

虽然这片内存是 Server 进程创建的，但是驱动会帮助管理这个缓存池，当数据从发送端复制过来时，驱动会根据发送数据包的大小，使用最佳匹配算法从缓存池中找一块大小合适的空间存放。当接收方处理完数据后，会通知驱动释放这个数据占用的内存。

![binder_server_ram](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/binder_server_ram.png)

事实上，内核并不会直接管理服务端 Binder 实体的指针，而是将它交给了 Service Manager 统一管理。

![media_player_service_ipc](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/media_player_service_ipc.png)



## 获取 Binder 服务（getService）

我们的客户端会继承服务端提供的抽象接口类并实现其公共函数。但客户端不是真正的实现了这个函数的功能，而是对远程调用的包装。将函数参数打包，通过Binder 驱动向服务端发送申请并等待返回值。

对此，客户端需要知道服务端Binder 实体的相关信息。还记得之前服务端交给 Service Manager 管理的 Binder 实体的指针吗？当我们调用 `getService(name)` 获取服务时，实际上我们是向 Service Manager 申请查询指定名字的服务并获取到服务端Binder 实体的指针的**引用**（有些时候又叫做句柄handle）。

为什么传递给客户端的是”指针的引用“而不是直接返回”指针“呢？第一，因为可能会有多个客户端同时绑定同一个服务端。每当一个客户端申请了指针的引用以后，驱动就会把这个指针的引用计数+1，意味着服务端在被使用，不可销毁。当所有客户端断开绑定后，指针的引用计数归0，则驱动就可以把服务端给回收掉了。第二，这也是为了安全性考虑。如果直接传递指针给驱动就可以调用到远程服务，那客户端是不是可以随便猜一个指针地址填进去呢？如果猜中了，既不是就可以直接调用调服务端？事实上，引用是由驱动生成的。驱动同时也在管理这些引用。只有当驱动检索到客户端曾经在他那申请过这个引用，驱动允许客户端远程调用服务端。

客户端在执行远程调用时，将会把这个指针的引用携带到发送给 Binder 驱动的数据包中。Binder 驱动通过这个指针的引用找到对应的 Binder 实体的指针，并通过 `reinterpret_cast` 转成 Binder 抽象类并调用 `onTransact()`  函数。由于这是个虚函数，不同的Binder实体中有各自的实现，从而可以调用到不同Binder实体提供的onTransact()。

![binder_trans](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/binder_trans.gif)

## 调试

在创建 Binder Driver 时，咱们会利用 Linux 的 debugfs 暴露的接口来创建一个用于调试的虚拟文件系统。我们所有途经 Binder 驱动的信息都会记录在这里。

![binder_debugfs](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/binder_debugfs.png)

当kernel中禁用debugfs的话，返回值是-%ENODEV。默认是禁用的。如果需要打开，在目录`/kernel/arch/arm64/configs/`下找到目标defconfig文件中添加一行`CONFIG_DEBUG_FS=y`，再重新编译版本，即可打开debug_fs。

## 实践

### Android App AIDL 编程

职责表述：

**IBinder** : IBinder 是一个接口，代表了一种跨进程通信的能力。只要实现了这个借口，这个对象就能跨进程传输。

**IInterface** :  IInterface 代表的就是 Server 进程对象具备什么样的能力（能提供哪些方法，其实对应的就是 AIDL 文件中定义的接口）

**Binder** : Java 层的 Binder 类，代表的其实就是 Binder 本地对象。BinderProxy 类是 Binder 类的一个内部类，它代表远程进程的 Binder 对象的本地代理；这两个类都继承自 IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder 驱动会自动完成这两个对象的转换。

**Stub** : AIDL 的时候，编译工具会给我们生成一个名为 Stub 的静态内部类；这个类继承了 Binder, 说明它是一个 Binder 本地对象，它实现了 IInterface 接口，表明它具有 Server 承诺给 Client 的能力；Stub 是一个抽象类，具体的 IInterface 的相关实现需要开发者自己实现。

![MyServer_java_binder](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/MyServer_java_binder.jpg)



### 其他

服务端能不能知道客户端死亡


# Android 高版本上对 Binder 的优化

## 多个 Binder 域

为了在框架（独立于设备）和供应商（特定于设备）代码之间彻底拆分 Binder 流量，Android 8 引入了“Binder 上下文”的概念。每个 Binder 上下文都有自己的设备节点和上下文（服务）管理器。您只能通过上下文管理器所属的设备节点对其进行访问，并且在通过特定上下文传递 Binder 节点时，只能由另一个进程从相同的上下文访问上下文管理器，从而确保这些域完全互相隔离。

在 Android 8 中，`/dev/binder` 设备节点成为 framework 进程的专有节点，这意味着供应商进程无法再访问此节点。供应商进程可以访问 `/dev/hwbinder`，但必须将其 AIDL 接口转为使用 HIDL。


## 分散-集中
在之前的 Android 版本中，Binder 调用中的每条数据都会被复制 3 次：

* 一次是在调用进程中将数据序列化为 Parcel
* 一次是在内核驱动程序中将 Parcel 复制到目标进程
* 一次是在目标进程中反序列化 Parcel
Android 8 使用分散-集中优化将副本数量从 3 减少到 1。数据保留其原始结构和内存布局，且 Binder 驱动程序会立即将数据复制到目标进程中，而不是先在 Parcel 中对数据进行序列化。在目标进程中，这些数据的结构和内存布局保持不变，并且，在无需再次复制的情况下即可读取这些数据。

| IPC域          | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| /dev/binder    | 框架/应用进程之间的 IPC，使用 AIDL 接口                      |
| /dev/hwbinder  | 框架/供应商进程之间的 IPC，使用 HIDL 接口<br/>供应商进程之间的 IPC，使用 HIDL 接口 |
| /dev/vndbinder | 供应商/供应商进程之间的 IPC，使用 AIDL 接口                  |

在 Android 10 中，HIDL 功能已整合到 AIDL 中。此后，HIDL 就被废弃了，并且仅供尚未转换为 AIDL 的子系统使用。

## 精细锁定
在之前的 Android 版本中，Binder 驱动程序使用全局锁来防范对重要数据结构的并发访问。虽然采用全局锁时出现争用的可能性极低，但主要的问题是，如果低优先级线程获得该锁，然后实现了抢占，会导致同样需要获得该锁的优先级较高的线程出现严重的延迟。这会导致平台卡顿。

原先尝试解决此问题的方法是在保留全局锁的同时禁止抢占。但是，这更像是一种临时应对手段而非真正的解决方案，最终被上游拒绝并舍弃。后来尝试的解决方法侧重于提升锁定的精细程度，自 2017 年 1 月以来，Pixel 设备上一直采用的是更加精细的锁定。虽然这些更改大部分已公开，但后续版本中还会有一些重大的改进。

在确定了精细锁定实现中的一些小问题后，我们使用不同的锁定架构设计了一种改进的解决方案，并在所有通用内核分支中提交了相关更改。我们会继续在大量不同的设备上测试这种实现方式；由于目前看来这个方案不存在什么问题，因此建议搭载 Android 8 的设备都使用这种实现方式。

## 实时优先级继承
Binder 驱动程序一直支持 nice 优先级继承。随着 Android 中以实时优先级运行的进程日益增加，现在出现以下这种情形也属正常：如果实时线程进行 Binder 调用，则处理该调用的进程中的线程同样会以实时优先级运行。为了支持这些用例，Android 8 现在在 Binder 驱动程序中实现了实时优先级继承。

除了事务级优先级继承之外，“节点优先级继承”允许节点（Binder 服务对象）指定执行对该节点的调用所需的最低优先级。之前版本的 Android 已经通过 nice 值支持节点优先级继承，但 Android 8 增加了对实时调度政策节点继承的支持。

## 用户空间变更
Android 8 包含在通用内核中使用当前 Binder 驱动程序所需的所有用户空间变更，但有一个例外：针对 /dev/binder 停用实时优先级继承的原始实现使用了 ioctl。由于后续开发将优先级继承的控制方法改为了更加精细的方法（根据 Binder 模式，而非上下文），因此，ioctl 并非存于 Android 通用分支中，而是提交到了我们的通用内核中。

此项变更的影响是，所有节点均默认停用实时优先级继承。Android 性能团队发现，为 hwbinder 域中的所有节点启用实时优先级继承会有一定好处。

# FAQ

**Q：为什么进程间的内存是隔离的，怎么做到的？**

A：在早期的计算机中，任何用户都可以直接或者间接的访问到硬件上的任意的内存地址。但是这存在以下三个问题：

1. 进程地址空间不隔离。由于程序都是直接访问物理内存，所以恶意程序可以随意修改别的进程的内存数据，以达到破坏的目的。有些非恶意的，但是有bug的程序也可能不小心修改了其它程序的内存数据，就会导致其它程序的运行出现异常。这种情况对用户来说是无法容忍的，因为用户希望使用计算机的时候，其中一个任务失败了，至少不能影响其它的任务。
2. 内存使用效率低。在A和B都运行的情况下，如果用户又运行了程序C，而程序C需要20M大小的内存才能运行，而此时系统只剩下8M的空间可供使用，所以此时系统必须在已运行的程序中选择一个将该程序的数据暂时拷贝到硬盘上，释放出部分空间来供程序C使用，然后再将程序C的数据全部装入内存中运行。可以想象得到，在这个过程中，有大量的数据在装入装出，导致效率十分低下。
3. 程序运行的地址不确定。当内存中的剩余空间可以满足程序C的要求后，操作系统会在剩余空间中随机分配一段连续的20M大小的空间给程序C使用，因为是随机分配的，所以程序运行的地址是不确定的。

为了解决上述问题，人们想到了一种变通的方法，就是增加一个中间层，利用一种间接的地址访问方法访问物理内存。按照这种方法，程序中访问的内存地址不再是实际的物理内存地址，而是一个虚拟地址，然后由操作系统将这个虚拟地址映射到适当的物理内存地址上。这样，只要操作系统处理好虚拟地址到物理内存地址的映射，就可以保证不同的程序最终访问的内存地址位于不同的区域，彼此没有重叠，就可以达到内存地址空间隔离的效果。

当然，这只解决了问题1和问题3，内存使用效率低的问题并没有解决，于是人们又引入了内存分页。分页的思想是程序运行时用到哪页就为哪页分配内存，没用到的页暂时保留在硬盘上。当用到这些页时再在物理地址空间中为这些页分配内存，然后建立虚拟地址空间中的页和刚分配的物理内存页间的映射。



**Q：Android中的智能指针？**

A： Android的智能指针是通过引用技术实现指针指向的对象的共享，对象的创建和删除（主要是删除）交给智能指针处理，而不用用户过分关心。

实现智能指针需要两步：一是为指向的对象关联引用计数，二是构造智能指针对象。引用计数是目标对象的属性，实现方法是：编写基类，实现引用计数的维护，然后让指向的类继承该基类，获得引用技术的属性。

智能指针本身是一个对象，负责维护引用计数并根据引用计数delete引用对象。

如果系统中有两个对象A和B，在对象A的内部引用了对象B，而在对象B的内部也引用了对象A。当两个对象A和B都不再使用时，系统会发现无法回收这两个对象的所占据的内存的，因为系统一次只能回收一个对象，而无论系统决定要收回对象A还是要收回对象B时，都会发现这个对象被其它的对象所引用，因而就都回收不了，类似于死锁现象，这样就造成了内存泄漏。

针对这个问题，可以采用对象的引用计数同时存在强引用和弱引用两种计数。例如A引用B则B的强引用计数和弱引用计数+1，而B引用A则A仅仅弱引用数+1，在回收时只要对象的强引用计数为0，则不管弱引用数是否为0都进行回收，类似于死锁解决中的强制释放资源，这样问题得到解决。



# 其他
## Binder 的数据传输

下图是 Binder 的通信模型。

![binder_trans](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/binder_trans.gif)


这个模型放在这里想要解释清楚还为时尚早，接下来我们将慢慢剖析。
考察一次Binder通信的全过程会发现，Binder存在于系统以下几个部分中：

* 用户进程：分别位于Server进程和Client进程中

* Binder驱动：分别管理为Server端的Binder实体和Client端的引用

* 传输数据：由于Binder可以跨进程传递，需要在传输数据中予以表述

在系统不同部分，Binder实现的功能不同，表现形式也不一样。接下来逐一探讨Binder在各部分所扮演的角色和使用的数据结构。
### 服务端

对于服务端而言，Binder是一个实体类，首先需要定义一个抽象接口类封装 Server 向外暴露的所有功能。由于这些函数需要跨进程调用，须为其一一编号，从而Server可以根据收到的编号决定调用哪个函数。然后需要实现一个 Binder 的抽象类用来处理来自 Client 端的 Binder 请求数据包。其中最重要的就是实现 `onTransact()` 函数，该函数分析收到的数据包，并调用相应的接口函数处理请求。函数执行完毕，如果需要返回数据就再构建一个返回数据包。服务端需要维护一个无限轮询的线程和接收事件的队列，用来接收由驱动传来的事件并响应。

### 客户端

对于客户端而言，需要基于服务端定义的抽象接口类实现一个 Binder （BinderProxy）。但 Client 端并不是真正的实现了这个函数，而只是对远程函数调用的包装。通过将函数参数打包，通过 Binder 驱动发送给 Server 端并等待返回值。为此Client端的Binder还要知道Binder实体的相关信息，即对Server 端的 Binder 实体的**引用**。

### 传输数据中

Binder可以塞在数据包的有效数据中越进程边界从一个进程传递给另一个进程，这些传输中的Binder用结构flat_binder_object表示：

```c++
struct flat_binder_object{
    /* 表明该Binder的类型，包括以下几种：
	 * BINDER_TYPE_BINDER：表示传递的是Binder实体，并且指向该实体的引用都是强类型；
     * BINDER_TYPE_WEAK_BINDER：表示传递的是Binder实体，并且指向该实体的引用都是弱类型；
     * BINDER_TYPE_HANDLE：表示传递的是Binder强类型的引用
     * BINDER_TYPE_WEAK_HANDLE：表示传递的是Binder弱类型的引用
     * BINDER_TYPE_FD：表示传递的是文件形式的Binder*/
    unsigned long type;
    
    /* 该域只对第一次传递Binder实体时有效，因为此刻驱动需要在内核中创建相应的实体节点，有些参数需要从该
     * 域取出：
     * 第0-7位：代码中用FLAT_BINDER_FLAG_PRIORITY_MASK取得，表示处理本实体请求数据包的线程的最低
     * 优先级。当一个应用程序提供多个实体时，可以通过该参数调整分配给各个实体的处理能力。
     * 第8位：代码中用FLAT_BINDER_FLAG_ACCEPTS_FDS取得，置1表示该实体可以接收其它进程发过来的文件
     * 形式的Binder。由于接收文件形式的Binder会在本进程中自动打开文件，有些Server可以用该标志禁止该功
     * 能，以防打开过多文件*/
    unsigned long flags;
    union {
        undefined void *binder;  // 当传递的是Binder实体时使用binder域，指向Binder实体在应用程序中的地址
        signed long handle; // 当传递的是Binder引用时使用handle域，存放Binder在进程中的引用号
    }
    void *cookie; // 该域只对Binder实体有效，存放与该Binder有关的附加信息
};
```

无论是Binder实体还是对实体的引用都从属与某个进程，所以该结构不能透明地在进程之间传输，必须经过驱动翻译。例如当Server把Binder实体传递给Client时，在发送数据流中，flat_binder_object中的type是BINDER_TYPE_BINDER，binder指向Server进程用户空间地址。如果透传给接收端将毫无用处，驱动必须对数据流中的这个Binder做修改：将type该成BINDER_TYPE_HANDLE；为这个Binder在接收进程中创建位于内核中的引用并将引用号填入handle中。对于发生数据流中引用类型的Binder也要做同样转换。经过处理后接收进程从数据流中取得的Binder引用才是有效的，才可以将其填入数据包binder_transaction_data的target.handle域，向Binder实体发送请求。

这样做也是出于安全性考虑：应用程序不能随便猜测一个引用号填入target.handle中就可以向Server请求服务了，因为驱动并没有为你在内核中创建该引用，必定会被驱动拒绝。唯有经过身份认证确认合法后，由‘权威机构’（Binder驱动）亲手授予你的Binder才能使用，因为这时驱动已经在内核中为你使用该Binder做了注册，交给你的引用号是合法的。

下表总结了当flat_binder_object结构穿过驱动时驱动所做的操作：

| Binder类型（type域）                           | 在发送方的操作                                               | 在接收方的操作                                               |
| ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BINDER_TYPE_BINDER<br/>BINDER_TYPE_WEAK_BINDER | 只有实体所在的进程能发送该类型的Binder。如果是第一次发送驱动将创建实体在内核中的节点，并保存binder，cookie，flag域。 | 如果是第一次接收该Binder则创建实体在内核中的引用；将handle域替换为新建的引用号；将type域替换为BINDER_TYPE_(WEAK_)HANDLE |
| BINDER_TYPE_HANDLE<br/>BINDER_TYPE_WEAK_HANDLE | 获得Binder引用的进程都能发送该类型Binder。驱动根据handle域提供的引用号查找建立在内核的引用。如果找到说明引用号合法，否则拒绝该发送请求。 | 如果收到的Binder实体位于接收进程中：将ptr域替换为保存在节点中的binder值；cookie替换为保存在节点中的cookie值；type替换为BINDER_TYPE_(WEAK_)BINDER。<br/><br/>如果收到的Binder实体不在接收进程中：如果是第一次接收则创建实体在内核中的引用；将handle域替换为新建的引用号 |
| BINDER_TYPE_FD                                 | 验证handle域中提供的打开文件号是否有效，无效则拒绝该发送请求。 | 在接收方创建新的打开文件号并将其与提供的打开文件描述结构绑定。 |



### 驱动

对于驱动来说，Server的 Binder 实体是一个节点，隶属于 Server 的进程。Client 的 Binder 引用。

### 传输协议
Binder协议基本格式是（命令+数据），使用ioctl(fd, cmd, arg)函数实现交互。命令由参数cmd承载，数据由参数arg承载，随cmd不同而不同。下表列举了所有命令及其所对应的数据：

| 命令                   | 含义                                                         | 参数                                                         |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BINDER_WRITE_READ      | 该命令向Binder写入或读取数据。参数分为两段：写部分和读部分。如果write_size不为0就先将write_buffer里的数据写入Binder；如果read_size不为0再从Binder中读取数据存入read_buffer中。write_consumed和read_consumed表示操作完成时Binder驱动实际写入或读出的数据个数。 | struct binder_write_read<br/> {<br/>undefined signed long write_size;<br/>signed long write_consumed;<br/>unsigned long write_buffer;<br/>signed long read_size;<br/>signed long read_consumed;<br/>unsigned long read_buffer;<br/>} |
| BINDER_SET_MAX_THREADS | 该命令告知Binder驱动接收方（通常是Server端）线程池中最大的线程数。由于Client是并发向Server端发送请求的，Server端必须开辟线程池为这些并发请求提供服务。告知驱动线程池的最大值是为了让驱动发现线程数达到该值时不要再命令接收端启动新的线程。 | int max_threads                                              |
| BINDER_SET_CONTEXT_MGR | 将当前进程注册为SMgr。系统中同时只能存在一个SMgr。只要当前的SMgr没有调用close()关闭Binder驱动就不能有别的进程可以成为SMgr。 | N/A                                                          |
| BINDER_THREAD_EXIT     | 通知Binder驱动当前线程退出了。Binder会为所有参与Binder通信的线程（包括Server线程池中的线程和Client发出请求的线程）建立相应的数据结构。这些线程在退出时必须通知驱动释放相应的数据结构。 | N/A                                                          |
| BINDER_VERSION         | 获得Binder驱动的版本号。                                     | N/A                                                          |

这其中最常用的命令是BINDER_WRITE_READ。该命令的参数包括两部分数据：一部分是向Binder写入的数据，一部分是要从Binder读出的数据，驱动程序先处理写部分再处理读部分。这样安排的好处是应用程序可以很灵活地处理命令的同步或异步。例如若要发送异步命令可以只填入写部分而将read_size置成0；若要只从Binder获得数据可以将写部分置空即write_size置成0；若要发送请求并同步等待返回数据可以将两部分都置上。
### BINDER_WRITE_READ 之写操作

Binder写操作的数据时格式同样也是（命令+数据）。这时候命令和数据都存放在binder_write_read 结构write_buffer域指向的内存空间里，多条命令可以连续存放。数据紧接着存放在命令后面，格式根据命令不同而不同。下表列举了Binder写操作支持的命令：

| cmd                                                       | 含义                                                         | arg                                                          |
| --------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BC_TRANSACTION<br/>BC_REPLY                               | BC_TRANSACTION用于Client向Server发送请求数据；BC_REPLY用于Server向Client发送回复（应答）数据。其后面紧接着一个binder_transaction_data结构体表明要写入的数据。 | struct binder_transaction_data                               |
| BC_ACQUIRE_RESULT<br/>BC_ATTEMPT_ACQUIRE                  | 暂未实现                                                     | N/A                                                          |
| BC_FREE_BUFFER                                            | 释放一块映射的内存。Binder接收方通过mmap()映射一块较大的内存空间，Binder驱动基于这片内存采用最佳匹配算法实现接收数据缓存的动态分配和释放，满足并发请求对接收缓存区的需求。应用程序处理完这片数据后必须尽快使用该命令释放缓存区，否则会因为缓存区耗尽而无法接收新数据。 | 指向需要释放的缓存区的指针；该指针位于收到的Binder数据包中   |
| BC_INCREFS<br/>BC_ACQUIRE<br/>BC_RELEASE<br/>BC_DECREFS   | 这组命令增加或减少Binder的引用计数，用以实现强指针或弱指针的功能。 | 32位Binder引用号                                             |
| BC_INCREFS_DONE<br/>BC_ACQUIRE_DONE                       | 第一次增加Binder实体引用计数时，驱动向Binder实体所在的进程发送BR_INCREFS， BR_ACQUIRE消息；Binder实体所在的进程处理完毕回馈BC_INCREFS_DONE，BC_ACQUIRE_DONE | void *ptr：Binder实体在用户空间中的指针</br>void *cookie：与该实体相关的附加数据 |
| BC_REGISTER_LOOPER<br/>BC_ENTER_LOOPER<br/>BC_EXIT_LOOPER | 这组命令同BINDER_SET_MAX_THREADS一道实现Binder驱动对接收方线程池管理。BC_REGISTER_LOOPER通知驱动线程池中一个线程已经创建了；BC_ENTER_LOOPER通知驱动该线程已经进入主循环，可以接收数据；BC_EXIT_LOOPER通知驱动该线程退出主循环，不再接收数据。 | N/A                                                          |
| BC_REQUEST_DEATH_NOTIFICATION                             | 获得Binder引用的进程通过该命令要求驱动在Binder实体销毁得到通知。虽说强指针可以确保只要有引用就不会销毁实体，但这毕竟是个跨进程的引用，谁也无法保证实体由于所在的Server关闭Binder驱动或异常退出而消失，引用者能做的是要求Server在此刻给出通知。 | uint32 *ptr; 需要得到死亡通知的Binder引用</br>void **cookie: 与死亡通知相关的信息，驱动会在发出死亡通知时返回给发出请求的进程。 |
| BC_DEAD_BINDER_DONE                                       | 收到实体死亡通知的进程在删除引用后用本命令告知驱动。         | void **cookie                                                |

在这些命令中，最常用的是BC_TRANSACTION/BC_REPLY命令对，Binder请求和应答数据就是通过这对命令发送给接收方。这对命令所承载的数据包由结构体struct binder_transaction_data定义。Binder交互有同步和异步之分，利用binder_transaction_data中flag域区分。如果flag域的TF_ONE_WAY位为1则为异步交互，即Client端发送完请求交互即结束， Server端不再返回BC_REPLY数据包；否则Server会返回BC_REPLY数据包，Client端必须等待接收完该数据包方才完成一次交互。

### BINDER_WRITE_READ之读操作

从Binder里读出的数据格式和向Binder中写入的数据格式一样，采用（消息ID+数据）形式，并且多条消息可以连续存放。下表列举了从Binder读出的命令字及其相应的参数：

| 消息                                                     | 含义                                                         | 参数                                                         |
| -------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BR_ERROR                                                 | 发生内部错误（如内存分配失败）                               | N/A                                                          |
| BR_OK<br/>BR_NOOP                                        | 操作完成                                                     | N/A                                                          |
| BR_SPAWN_LOOPER                                          | 该消息用于接收方线程池管理。当驱动发现接收方所有线程都处于忙碌状态且线程池里的线程总数没有超过BINDER_SET_MAX_THREADS设置的最大线程数时，向接收方发送该命令要求创建更多线程以备接收数据。 | N/A                                                          |
| BR_TRANSACTION<br/>BR_REPLY                              | 这两条消息分别对应发送方的BC_TRANSACTION和BC_REPLY，表示当前接收的数据是请求还是回复。 | binder_transaction_data                                      |
| BR_ACQUIRE_RESULT<br/>BR_ATTEMPT_ACQUIRE<br/>BR_FINISHED | 尚未实现                                                     | N/A                                                          |
| BR_DEAD_REPLY                                            | 交互过程中如果发现对方进程或线程已经死亡则返回该消息         | N/A                                                          |
| BR_TRANSACTION_COMPLETE                                  | 发送方通过BC_TRANSACTION或BC_REPLY发送完一个数据包后，都能收到该消息做为成功发送的反馈。这和BR_REPLY不一样，是驱动告知发送方已经发送成功，而不是Server端返回请求数据。所以不管同步还是异步交互接收方都能获得本消息。 | N/A                                                          |
| BR_INCREFS<br/>BR_ACQUIRE<br/>BR_RELEASE<br/>BR_DECREFS  | 这一组消息用于管理强/弱指针的引用计数。只有提供Binder实体的进程才能收到这组消息。 | void *ptr：Binder实体在用户空间中的指针<br/>void *cookie：与该实体相关的附加数据N/A |
| BR_DEAD_BINDER<br/>BR_CLEAR_DEATH_NOTIFICATION_DONE      | 向获得Binder引用的进程发送Binder实体死亡通知书；收到死亡通知书的进程接下来会返回BC_DEAD_BINDER_DONE做确认。 | void **cookie：在使用BC_REQUEST_DEATH_NOTIFICATION注册死亡通知时的附加参数。 |
| BR_FAILED_REPLY                                          | 如果发送非法引用号则返回该消息                               | N/A                                                          |

和写数据一样，其中最重要的消息是BR_TRANSACTION 或BR_REPLY，表明收到了一个格式为binder_transaction_data的请求数据包（BR_TRANSACTION）或返回数据包（BR_REPLY）。



### binder_transaction_data : 收发数据包结构 

该结构是Binder接收/发送数据包的标准格式，每个成员定义如下：

| 成员                                                         | 含义                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| union<br/>{<br/>undefined size_t handle;<br/>void *ptr;<br/>} target | 对于发送数据包的一方，该成员指明发送目的地。由于目的是在远端，所以这里填入的是对Binder实体的引用，存放在target.handle中。如前述，Binder的引用在代码中也叫句柄（handle）。<br/>当数据包到达接收方时，驱动已将该成员修改成Binder实体，即指向Binder对象内存的指针，使用target.ptr来获得。该指针是接收方在将Binder实体传输给其它进程时提交给驱动的，驱动程序能够自动将发送方填入的引用转换成接收方Binder对象的指针，故接收方可以直接将其当做对象指针来使用（通常是将其reinterpret_cast成相应类）。 |
| void *cookie                                                 | 发送方忽略该成员；接收方收到数据包时，该成员存放的是创建Binder实体时由该接收方自定义的任意数值，做为与Binder指针相关的额外信息存放在驱动中。驱动基本上不关心该成员。 |
| unsigned int code                                            | 该成员存放收发双方约定的命令码，驱动完全不关心该成员的内容。通常是Server端定义的公共接口函数的编号。 |
| unsigned int flags                                           | 与交互相关的标志位，其中最重要的是TF_ONE_WAY位。如果该位置上表明这次交互是异步的，Server端不会返回任何数据。驱动利用该位来决定是否构建与返回有关的数据结构。另外一位TF_ACCEPT_FDS是出于安全考虑，如果发起请求的一方不希望在收到的回复中接收文件形式的Binder可以将该位置上。因为收到一个文件形式的Binder会自动为数据接收方打开一个文件，使用该位可以防止打开文件过多。 |
| pid_t sender_pid<br/>uid_t sender_euid                       | 该成员存放发送方的进程ID和用户ID，由驱动负责填入，接收方可以读取该成员获知发送方的身份。 |
| size_t data_size                                             | 该成员表示data.buffer指向的缓冲区存放的数据长度。发送数据时由发送方填入，表示即将发送的数据长度；在接收方用来告知接收到数据的长度。 |
| size_t offsets_size                                          | 驱动一般情况下不关心data.buffer里存放什么数据，但如果有Binder在其中传输则需要将其相对data.buffer的偏移位置指出来让驱动知道。有可能存在多个Binder同时在数据中传递，所以须用数组表示所有偏移位置。本成员表示该数组的大小。 |
| union {undefined  struct <br/>{<br/>undefined const void *buffer;<br/>const void *offsets;<br/>} ptr;<br/>uint8_t buf[8];<br/>} data; | data.bufer存放要发送或接收到的数据；data.offsets指向Binder偏移位置数组，该数组可以位于data.buffer中，也可以在另外的内存空间中，并无限制。buf[8]是为了无论保证32位还是64位平台，成员data的大小都是8个字节。 |



![binder_transaction_data](https://raw.githubusercontent.com/XhosaS/ImageHosting/master/binder_transaction_data.gif)



### 匿名 Binder

并不是所有Binder都需要注册给 Service Manager 广而告之的。Server端可以通过已经建立的Binder连接将创建的Binder实体传给Client，当然这条已经建立的Binder连接必须是通过实名Binder实现。由于这个Binder没有向Service Manager注册名字，所以是个匿名Binder。Client将会收到这个匿名Binder的引用，通过这个引用向位于Server中的实体发送请求。匿名Binder为通信双方建立一条私密通道，只要Server没有把匿名Binder发给别的进程，别的进程就无法通过穷举或猜测等任何方式获得该Binder的引用，向该Binder发送请求。



### 文件 Binder

除了通常意义上用来通信的Binder，还有一种特殊的Binder：文件Binder。这种Binder的基本思想是：将文件看成Binder实体，进程打开的文件号看成Binder的引用。一个进程可以将它打开文件的文件号传递给另一个进程，从而另一个进程也打开了同一个文件，就象Binder的引用在进程之间传递一样。

一个进程打开一个文件，就获得与该文件绑定的打开文件号。从Binder的角度，linux在内核创建的打开文件描述结构struct file是Binder的实体，打开文件号是该进程对该实体的引用。既然是Binder那么就可以在进程之间传递，故也可以用flat_binder_object结构将文件Binder通过数据包发送至其它进程，只是结构中type域的值为BINDER_TYPE_FD，表明该Binder是文件Binder。而结构中的handle域则存放文件在发送方进程中的打开文件号。我们知道打开文件号是个局限于某个进程的值，一旦跨进程就没有意义了。这一点和Binder实体用户指针或Binder引用号是一样的，若要跨进程同样需要驱动做转换。驱动在接收Binder的进程空间创建一个新的打开文件号，将它与已有的打开文件描述结构struct file勾连上，从此该Binder实体又多了一个引用。新建的打开文件号覆盖flat_binder_object中原来的文件号交给接收进程。接收进程利用它可以执行read()，write()等文件操作。

传个文件为啥要这么麻烦，直接将文件名用Binder传过去，接收方用open()打开不就行了吗？其实这还是有区别的。首先对同一个打开文件共享的层次不同：使用文件Binder打开的文件共享linux VFS中的struct file，struct dentry，struct inode结构，这意味着一个进程使用read()/write()/seek()改变了文件指针，另一个进程的文件指针也会改变；而如果两个进程分别使用同一文件名打开文件则有各自的struct file结构，从而各自独立维护文件指针，互不干扰。其次是一些特殊设备文件要求在struct file一级共享才能使用，例如android的另一个驱动ashmem，它和Binder一样也是misc设备，用以实现进程间的共享内存。一个进程打开的ashmem文件只有通过文件Binder发送到另一个进程才能实现内存共享，这大大提高了内存共享的安全性，道理和Binder增强了IPC的安全性是一样的。

### 异步 Binder



## Binder 的多线程模型

Binder通信实际上是位于不同进程中的线程之间的通信。假如进程S是Server端，提供Binder实体，线程T1从Client进程C1中通过Binder的引用向进程S发送请求。S为了处理这个请求需要启动线程T2，而此时线程T1处于接收返回数据的等待状态。T2处理完请求就会将处理结果返回给T1，T1被唤醒得到处理结果。在这过程中，T2仿佛T1在进程S中的代理，代表T1执行远程任务，而给T1的感觉就是象穿越到S中执行一段代码又回到了C1。为了使这种穿越更加真实，驱动会将T1的一些属性赋给T2，特别是T1的优先级nice，这样T2会使用和T1类似的时间完成任务。很多资料会用‘线程迁移’来形容这种现象，容易让人产生误解。一来线程根本不可能在进程之间跳来跳去，二来T2除了和T1优先级一样，其它没有相同之处，包括身份，打开文件，栈大小，信号处理，私有数据等。

对于Server进程S，可能会有许多Client同时发起请求，为了提高效率往往开辟线程池并发处理收到的请求。怎样使用线程池实现并发处理呢？这和具体的IPC机制有关。拿socket举例，Server端的socket设置为侦听模式，有一个专门的线程使用该socket侦听来自Client的连接请求，即阻塞在accept()上。这个socket就象一只会生蛋的鸡，一旦收到来自Client的请求就会生一个蛋 – 创建新socket并从accept()返回。侦听线程从线程池中启动一个工作线程并将刚下的蛋交给该线程。后续业务处理就由该线程完成并通过这个单与Client实现交互。

可是对于Binder来说，既没有侦听模式也不会下蛋，怎样管理线程池呢？一种简单的做法是，不管三七二十一，先创建一堆线程，每个线程都用BINDER_WRITE_READ命令读Binder。这些线程会阻塞在驱动为该Binder设置的等待队列上，一旦有来自Client的数据驱动会从队列中唤醒一个线程来处理。这样做简单直观，省去了线程池，但一开始就创建一堆线程有点浪费资源。于是Binder协议引入了专门命令或消息帮助用户管理线程池，包括：

· INDER_SET_MAX_THREADS

· BC_REGISTER_LOOP

· BC_ENTER_LOOP

· BC_EXIT_LOOP

· BR_SPAWN_LOOPER

首先要管理线程池就要知道池子有多大，应用程序通过INDER_SET_MAX_THREADS告诉驱动最多可以创建几个线程。以后每个线程在创建，进入主循环，退出主循环时都要分别使用BC_REGISTER_LOOP，BC_ENTER_LOOP，BC_EXIT_LOOP告知驱动，以便驱动收集和记录当前线程池的状态。每当驱动接收完数据包返回读Binder的线程时，都要检查一下是不是已经没有闲置线程了。如果是，而且线程总数不会超出线程池最大线程数，就会在当前读出的数据包后面再追加一条BR_SPAWN_LOOPER消息，告诉用户线程即将不够用了，请再启动一些，否则下一个请求可能不能及时响应。新线程一启动又会通过BC_xxx_LOOP告知驱动更新状态。这样只要线程没有耗尽，总是有空闲线程在等待队列中随时待命，及时处理请求。

关于工作线程的启动，Binder驱动还做了一点小小的优化。当进程P1的线程T1向进程P2发送请求时，驱动会先查看一下线程T1是否也正在处理来自P2某个线程请求但尚未完成（没有发送回复）。这种情况通常发生在两个进程都有Binder实体并互相对发时请求时。假如驱动在进程P2中发现了这样的线程，比如说T2，就会要求T2来处理T1的这次请求。因为T2既然向T1发送了请求尚未得到返回包，说明T2肯定（或将会）阻塞在读取返回包的状态。这时候可以让T2顺便做点事情，总比等在那里闲着好。而且如果T2不是线程池中的线程还可以为线程池分担部分工作，减少线程池使用率。

通常数据传输的接收端有两个队列：数据包接收队列和（线程）等待队列，用以缓解供需矛盾。当超市里的进货（数据包）太多，货物会堆积在仓库里；购物的人（线程）太多，会排队等待在收银台，道理是一样的。在驱动中，每个进程有一个全局的接收队列，也叫to-do队列，存放不是发往特定线程的数据包；相应地有一个全局等待队列，所有等待从全局接收队列里收数据的线程在该队列里排队。每个线程有自己私有的to-do队列，存放发送给该线程的数据包；相应的每个线程都有各自私有等待队列，专门用于本线程等待接收自己to-do队列里的数据。虽然名叫队列，其实线程私有等待队列中最多只有一个线程，即它自己。

由于发送时没有特别标记，驱动怎么判断哪些数据包该送入全局to-do队列，哪些数据包该送入特定线程的to-do队列呢？这里有两条规则。规则1：Client发给Server的请求数据包都提交到Server进程的全局to-do队列。不过有个特例，就是Binder对工作线程启动的优化。经过优化，来自T1的请求不是提交给P2的全局to-do队列，而是送入了T2的私有to-do队列。规则2：对同步请求的返回数据包（由BC_REPLY发送的包）都发送到发起请求的线程的私有to-do队列中。如上面的例子，如果进程P1的线程T1发给进程P2的线程T2的是同步请求，那么T2返回的数据包将送进T1的私有to-do队列而不会提交到P1的全局to-do队列。

数据包进入接收队列的潜规则也就决定了线程进入等待队列的潜规则，即一个线程只要不接收返回数据包则应该在全局等待队列中等待新任务，否则就应该在其私有等待队列中等待Server的返回数据。还是上面的例子，T1在向T2发送同步请求后就必须等待在它私有等待队列中，而不是在P1的全局等待队列中排队，否则将得不到T2的返回的数据包。

这些潜规则是驱动对Binder通信双方施加的限制条件，体现在应用程序上就是同步请求交互过程中的线程一致性：1) Client端，等待返回包的线程必须是发送请求的线程，而不能由一个线程发送请求包，另一个线程等待接收包，否则将收不到返回包；2) Server端，发送对应返回数据包的线程必须是收到请求数据包的线程，否则返回的数据包将无法送交发送请求的线程。这是因为返回数据包的目的Binder不是用户指定的，而是驱动记录在收到请求数据包的线程里，如果发送返回包的线程不是收到请求包的线程驱动将无从知晓返回包将送往何处。

接下来探讨一下Binder驱动是如何递交同步交互和异步交互的。我们知道，同步交互和异步交互的区别是同步交互的请求端（client）在发出请求数据包后须要等待应答端（Server）的返回数据包，而异步交互的发送端发出请求数据包后交互即结束。对于这两种交互的请求数据包，驱动可以不管三七二十一，统统丢到接收端的to-do队列中一个个处理。但驱动并没有这样做，而是对异步交互做了限流，令其为同步交互让路，具体做法是：对于某个Binder实体，只要有一个异步交互没有处理完毕，例如正在被某个线程处理或还在任意一条to-do队列中排队，那么接下来发给该实体的异步交互包将不再投递到to-do队列中，而是阻塞在驱动为该实体开辟的异步交互接收队列（Binder节点的async_todo域）中，但这期间同步交互依旧不受限制直接进入to-do队列获得处理。一直到该异步交互处理完毕下一个异步交互方可以脱离异步交互队列进入to-do队列中。之所以要这么做是因为同步交互的请求端需要等待返回包，必须迅速处理完毕以免影响请求端的响应速度，而异步交互属于‘发射后不管’，稍微延时一点不会阻塞其它线程。所以用专门队列将过多的异步交互暂存起来，以免突发大量异步交互挤占Server端的处理能力或耗尽线程池里的线程，进而阻塞同步交互。

## 死亡通知机制

//TODO: 2023/05/22

# 参考

* [Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)
* [Android Binder设计与实现 - 设计篇](https://blog.csdn.net/universus/article/details/6211589)
* [Binder学习指南](https://weishu.me/2016/01/12/binder-index-for-newer/)
* [Android进程间通信（IPC）机制Binder简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/6618363)
