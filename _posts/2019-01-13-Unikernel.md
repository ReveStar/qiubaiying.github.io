﻿---
layout:     post
title:      Unikernels
subtitle:   Unikernel介绍
date:       2019-01-13
author:     ZHD SL LFS
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Unikernel
    - homework
    - KylinX
    - IncludeOS
    - LibOS
---

# 引  言

随着云计算的兴起，越来越多的人们使用云平台来运行自己的服务与应用。在云平台上如何快速的部署应用和有效的利用资源以满足应用需求是研究的一大热点。为了提高资源的有效利用，云提供商常常会使用虚拟化技术将同一台物理机器划分给多个不同的云租户使用, 云租户可以在虚拟的云环镜上部署自己的虚拟机以运行自己的应用。随着云计算的发展，云应用数量的急剧增加，虚拟机的安全成为一个重要的问题，如何为多租户云提供能够快速部署运行应用，高效利用资源，安全的云环境成为研究的热点。

Docker是一种有效的应用管理技术，与虚拟机相比，容器有着轻量级，启动速度快，隔离性性好等特点，在一定程度上缓解了云环境快速部署运行应用，高效利用资源，安全的问题。然而，容器不能真正确保云环境更加安全。更好的解决方案就是使用Unikernel。

本文将首先探讨现有云环境模型的缺点，然后具体介绍 unikernel，再介绍一些现有的unikernel工作，并分析它们之间的差异，分析其优缺点。然后讨论 unikernel 在云环境中的应用会出现的一些问题及解决方案，讨论解决方案的可行性。

## 云环境背景 

云环境的使用者在云环境中部署自己的虚拟机以运行自己的应用来提供服务，他们在虚拟机中运行通用操作系统来支撑自己应用的执行。通用操作系统，顾名思义，就是为多用户提供广泛的库和功能，应用场景。各种各样不同的应用和服务都可以在通用操作系统上运行。对于多数用户来说这是一件令人愉悦的事情，用户不必每次都要为自己使用的应用或者服务安装其所依赖的库或者某些设备的驱动。但是对于其他某些特定的用户来说，这是一件令人苦恼的事情。对于部分使用云服务器的租客来说，他们只是希望在服务器上运行其特定的服务，致力于一些特定的在线应用程序，例如高度专业化的大数据分析应用或者游戏服务器，这些特定的服务和应用只需要通用操作系统中的很少一部分库和功能。这些无用的功能可能会对这部分用户造成的一定麻烦和困扰。
首先，特定的应用和服务与通用操作系统不匹配，造成了资源的浪费，而且导致了在两者之间存在大概率的安全漏洞，降低了其应用的安全性。由于多余库或者功能的存在导致了这些特定的应用在云服务器上部署时更加繁琐，并且容易受到不必要的功能和库的影响。其次通用操作系统需要更多的内存和磁盘的支持，而这些额外的资源并不能提高特定应用和服务的性能，而仅仅是为了满足通用操作系统的需要而已。
仅仅为了提供兼容性，便在为每一个用户提供的虚拟机中运行完整的操作系统不再适用于现在的云平台。随着云平台的快速发展，其用户越来越多，由于虚拟机中存在重复和残留的操作系统组件，这使得每台虚拟机的成本都非常高，用户越多其消耗越大，资源的浪费越严重，一些微不足道的资源浪费在整个云平台汇集之后，所造成的浪费将是巨大的。比如，每个虚拟计算机需要模拟定时器中断，这就导致了虚拟计算机空闲的时候也需要消耗能量，但在云服务的背景下，所有的虚拟计算机消耗叠加在一起就成了一项巨大的资源浪费，同时也限制了虚拟机管理程序的承载量。

## Unikernels

### Unikernel 架构

Unikernel是使用libOS(library os)构建的具有专门用途的单地址空间机器镜像。为了支撑程序的运行，开发者从模块栈中选择最小的类库集合，构建对应的OS。类库和应用代码、配置文件一起构建成固定用途的镜像，可以直接运行在hypervisor或者硬件上而无需Linux或者Windows这样的操作系统。

![通用操作系统与Unikernel架构对比](http://plazfv8ew.bkt.clouddn.com/%E9%80%9A%E7%94%A8%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E4%B8%8EUnikernel%E6%9E%B6%E6%9E%84%E5%AF%B9%E6%AF%94%E5%9B%BE.png)
通用操作系统与Unikernel架构对比


相比于传统通用操作系统的架构，其架构优化的设计思路就是尽可能减少抽象层次，允许应用程序直接访问硬件，比如直接请求特定的物理内存空间，所以kernel内核非常小，只负责保护和分配系统资源。应用程序过来请求资源， kernel看看资源是否空闲，如果空闲，直接交给应用，至于应用怎么访问是它自己的事，kernel并不关心这些。而解决这些问题的办法就是Library Operation System。
首先对硬件设备的驱动进行抽象，也就是定义一组接口，再针对不同硬件设备提供实现的库，在编译的时候，根据硬件配置引入合适的library，应用和library一起构成了整个系统。由于减少了抽象层，可以直接操作硬件资源，并且所有操作都在同一个地址空间里，libOS可以实现更好的性能。而对于libOS而言却有新的问题：如何进行资源控制以及硬件适配。因为libOS仅仅是硬件资源的抽像，硬件资源的分配可以由应用直接请求，但是缺少对资源的控制。并且在硬件更新后，libOS的对硬件资源的抽象并不一定依然有效，可能需要对接口进行更新，也就不能做到对硬件的适配。而随着虚拟化的兴起，这些问题得到了解决。虚拟机管理程序可以提供硬件的适配以及资源的控制，并且在每一个虚拟机中都只运行一个应用程序，即前文提到的高度专业化的特定应用程序，而且在一定程度上保证了不同应用之间的隔离性。此时所形成的架构体系便如图3.1右边所示的specialised unikernel。

### Unikernel 的优势

Unikernel的出现很好的解决了之前的所存在的问题，而且优点很明显。

1. 性能更好。相比于Linux/Windows这种通用操作系统，Unikernel减少了复杂的软件抽象层。由于“内核”和应用程序没有隔离，运行在同一个地址空间中，消除了用户态和内核态转换以及数据复制的开销。最后，构建时可以采用全局优化技术。

2. 除了性能外，Linux/Windows里通常包含了太多东西，不论你需不需要，都会包含在系统里，比如USB驱动，在云环境中肯定是用不到的，比如各种文件系统，实际上用到的也就一两种。Unikernel里只包含了程序真正依赖到的东西，无论镜像，还是启动后所占用的资源都非常小。 
3. 启动快，这个就很好理解了，因为包含的东西少，系统层初始化非常快，通常是几ms到十几ms，真正的时间会用在应用程序本身的启动上。 
4. 更安全，因为Unikernel里运行的内容少，减少了潜在漏洞数量，相对攻击面就很小。

### IncludeOS

IncludeOS是一个单任务的操作系统，使用C++语言编写，支持开发者在编译时直接构建c++代码。它拥有极小的磁盘和内存占用，高效的异步I/O，只包含服务需要的OS库，且只有一个默认的设备驱动程序，能够适应绝大多数的虚拟平台。

#### IncludeOS 架构设计

IncludeOS的架构设计包含

1. 遵循零开销原则，只包含服务所必须的东西，实现最小化。 
2. 使用静态链接库和定制的GCC工具链OS的每一部分会被编译到一个目标文件中，之后通过ar结合在一起形成静态库。当程序链接到这个库时，只有需要的东西会被链接提取出来并写入最终二进制文件中。程序没有传统的main函数，相对地为用户提供Service::start接口。 
3. 使用相对较小的能够编译成静态链接库的RedHat's newlib作为C标准库，选择包含了大部分实现的EASTL作为C++标准库。
4. 使用单个设备驱动程序VirtioNet Device driver。 
5. 实现了模块化、面向对象的网络栈。IP栈被封装在Inet class中，不同的IP栈可以通过重用或替换对象进行构建。模块间仅靠委托链接，委托与普通C指针性能相同，每个模块仅会知道与它连接的模块的成员函数的函数签名。而inet class 对连接有着充分的控制。
6. 实现异步I/O和延迟中断请求。目前中断请求不管有没有时间处理，都会更新counter，并在主事件循环中推迟进一步处理，减少了切换上下文的时间和并行相关的问题。CPU通过所有I/O异步保持忙碌，因此没有阻塞情况发生

#### IncludeOS 的优缺点

如表所示，与传统的操作系统相比，IncludeOS 的硬盘内存占用极小，支持高效的异步I/O，在空闲时不使用CPU,几乎没有主机、软件依赖，具有更快的启动时间，没有系统调用，没有不必要的时钟终端开销，VM exit 数量更少，性能总体上更好。

|               IncludeOS               |      传统操作系统      |
| :-----------------------------------: | :--------------------: |
|                单任务                 |    多个程序同时运行    |
|           单个设备驱动程序            |    支持多种硬件设备    |
|           磁盘内存占用极小            | 需要很多磁盘和内存占用 |
|    高效的异步i/o，空闲时不使用cpu     | 产生稳定cpu和i/o使用流 |
|             性能总体更好              |      性能总体一般      |
|        几乎没有主机、软件依赖         |        大量依赖        |
|            更快的启动时间             |      启动时间较慢      |
|             没有系统调用              |       有系统调用       |
|       没有不必要的时钟中断开销        |    时钟中断开销较大    |
| 通过保持受保护指令数量减少VM exit数量 |    VM exit数量较多     |


IncludeOS的缺点很显然，由于它遵循了零开销原则，最大化地删去了不必须的东西，因此会造成功能不够完备的问题出现，比如：为了减小C++标准库的大小，研究者使用了EASTL库，包含了主要的string,steams,vector,map等实现，但是依旧缺少许多功能；网络栈的功能不够完备，尚未支持TCP和IPV6等标准，未实现RFC标准和socketAPI标准，将工作留给了应用开发者。

由于IncludeOS推迟所有的中断，会导致其无法适应所有任务，由于推迟中断会使CPU工作量很大时VM看上去没有任何响应。如果是需要好几秒CPU处理请求的服务，ICMP包会排队，直到virtio队满了之后，被丢弃，造成未响应的服务的效果。

QEMU为了运行IncludeOS需要更多的时间，导致在host之外消耗的时间更久，原因尚未知晓，可能是Linux的virtio driver更加优化，使得vm exits更少，与host driver互动更加高效

### Unikernel 其他相关项目总结

自从2013年，MirageOS的研究者首次提出了Unikernel的概念之后，越来越多的研究团队开始加入到Unikernel的研发工作中，对主要的Unikernel研究项目的特点进行比较，如表所示。

|       开源项目       |     特点及优势  |   语言支持   | 适用场景  | 支持平台          |
| -------------------- | ----------- | --------- | --- | ------------------ |
|   HaLVM unikernel    | 实现Xen上的高级轻量级虚拟机     |   Haskell    |   单用途、轻量级虚拟机   |            Xen            |
|      Drawbridge      |           将沙箱机制和库操作系统结合；能够运行未修改的Windows程序           |      C       |              Windows桌面应用程序               |          Windows          |
|  Rumprun unikernel   | 基于组件化、具有内核特性驱动程序的rump kernels；支持现有的未修改的POSIX软件 |   各种语言   |               嵌入式系统和云环境               |       Xen,KVM,裸机        |
|      IncludeOS       |                               支持完全虚拟化                                |     C++      |                     云计算                     |      KVM,virtualBox       |
|  MirageOS unikernel  |                高级语言OCaml编写；支持安全、高效网络应用服务                |    OCaml     |                云计算和移动应用                | Xen,kFreeBSD,POSIX,www/js |
|  Graphene unikernel  |                   支持构建Linux多进程应用服务；全隔离性强                   |      C       |                 Linux应用程序                  |           Linux           |
|  ClickOS unikernel   |                            高性能的软件中间平台                             |     C++      |             实现网络功能虚拟化NFV              |            Xen            |
|        Clive         |                                 Go语言编写                                  |      Go      |                 分布式和云计算                 |          Xen,KVM          |
|    Osv unikernel     |               零操作系统管理；快速VM构建和部署；通用框架集成                | 多种编程语言 |              快速构建的单应用程序              | Xen,KVM,Vmware,VirtualBox |
|         LING         | 支持并发、分发和容错；删除大多数vcetor file，只运行三个外部库；文件系统只读 |    Erlang    | 构建具有高可用性要求的大规模可扩展的软实时系统 |            Xen            |
| Runtime.js Unikernel |                        实现CPU和内存等低级资源的管理                        |  Javascript  |               轻量级虚拟机云服务               |            KVM            |
|      HermitCore      |              支持扩展成多核处理器虚拟机系统；支持极限规模运算               | 支持多种语言 |              高性能计算的应用服务              |    Xem,KVM,x86_64架构     |



### Unikernel的争议

Unikernel虽然有很多优点但是没有一个东西可以说是完美无缺的，Unikernel依然存在着一些瑕疵，一些争议。在传统意义上，Unikernel并没有应用程序，应用程序已经被放入了操作系统内核中，而在操作系统内核中实现功能的主要原因是为了提高性能：通过避免跨用户内核边界的上下文切换，可以加快依赖跨越该边界的传输的操作。在unikernel的情况下，这些论点是似是而非的：在现代平台运行时的复杂性和现代微处理器的性能之间，人们通常不会发现应用程序受到用户内核上下文切换的限制。虽然它们可能是不稳定的，但这些论点进一步受到以下事实的破坏：unikernel非常依赖硬件虚拟化来实现多租户，然而在硬件层进行虚拟化会带来不可阻挡的性能税：通过让系统能够实际看到与实际可以看到应用程序的系统隔离的硬件（即虚拟机管理程序）（客户操作系统）在硬件利用率（例如，DRAM，NIC，CPU，I / O）方面的效率会丢失。

由unikernel支持者给出的另一个原因是unikernel是“更安全的”，但目前还不清楚这个论点的理论基础究竟是什么。尽管unikernel通常运行较少的软件（因此可能具有较少的攻击面），但原则上没有任何关于unikernel的导致更少的软件的理论基础。unikernel经常运行新的或不同的软件（因此不容易受到OpenSSL本身的漏洞攻击），但这种安全性通过这样的论点可以用来适用于任何新的，深奥的系统。安全性的论点似乎也超过了unikernel非常依赖的保护边界（底层管理程序提供的客户操作系统之间的保护边界）的上限。管理程序漏洞着重存在;每个人都不能将Linux内核漏洞视为一种威胁，而将虚拟机管理程序漏洞视而不见。相反，通过剥夺应用程序开发人员使用用户保护边界的工具，违反了最小权限原则：应用程序中的任何漏洞都在Unikernel的核心，在基于容器的部署的世界中，这是一个棘手的问题。

因为unikernel作为guestOS运行，由管理程序为应用程序分配的DRAM即使应用程序本身没有使用它，guestOS也会全部消费。由于内存不足仍然是最有害的应用程序故障模式之一（特别是在动态环境中），而Unikernel内存大小往往被过度设计，因为需求通常会盲目地加倍或以其他方式上升。在unikernel模型中，任何这样的内存溢出都会造成浪费，没有其他人可以使用它，因为管理程序不知道它实际上并没有被使用。这与其他应用程序未使用的内存可供其他容器或系统本身使用的容器形成鲜明对比。

unikernel的缺点始于应用程序本身的机制。当操作系统边界被消除时，可能已经消除了应用程序与网络或持久存储的真实世界交互的接口，但是没有人放弃对这种交互的需求！一些unikernel（如OSv和Rumprun）采用实现“POSIX-like”接口的方法来最小化对应用程序的干扰。好消息是应用程序有点工作！但坏消息是是否需要这些接口。同时希望应用程序的“POSIX-likeness”不会扩展到像创建进程这样的旧概念：unikernel中没有进程，所以如果你的应用程序依赖于这个构造，那么便无法在Unikernel中运行。

如果应用程序是运行在单内核的，那么就会发现unikernel不适合生产的最深层次的原因，而且在部署任何真实的生产环境时，Unikernel完全是不可取的。没有进程，所以当然没有ps，没有htop，没有strace，但是也没有netstat，没有tcpdump，没有ping。没有像DTrace或MDB那样的现代产品。从调试的角度来看Unikernel是拒绝调试的。当系统出现不当的非致命行为时，系统将会进行重启。而系统重启是系统在高负载时遭到严重破坏后所进行的迫不得已的行为，而不应该是因为对异常情况无法及时处理所产生的问题挤压造成的系统重启。

### 本章小结

本章节介绍了Unikernel的架构，分析了其架构的优点。其次还介绍IncludeOS项目，通过与传统操作系统进行对比，分析了IncludeOS的优缺点。此外还总结了Unikernel的相关项目，对比了它们之间的差异。最后讨论了现存的对Unikernel的一些争议。

## KylinX: Unikernels 对多进程的支持

### Unikernel 多进程

Unikernel是单用户单进程的，这就意味着Unikernel 无法支持动态的 fork, 而这是传统UNIX应用程序常用的多进程抽象;且编译时确定的不变性也排除了运行时管理的可能性，如库的动态更新和地址空间随机化等。这些缺陷降低了Unikernel的适用性和性能。

KylinX扩展了Unikernel，来实现以前仅适用于进程的功能，KylinX基于MiniOS，这是一个C风格的Unikernel LibOS，用于在Xen管理程序上运行的用户VM域（domU）。 MiniOS使用其前端驱动程序访问硬件，硬件连接到特权dom0中的相应后端驱动程序或专用驱动程序域。 MiniOS具有单个地址空间，没有内核和用户空间分离，以及一个没有抢占式的简单调度程序。 MiniOS虽然很小但是可以在Xen上实现简洁有效的LibOS设计。基于MiniOS的KylinX设计包括Dom0中基于Xen工具堆栈的（受限制的）动态页面/库映射扩展，以及DomU中的进程抽象支持（包括动态pVM fork / IpC和运行时pVM库链接）。KylinX通过利用Xen的共享内存和授权表来执行跨域页面映射，支持进程 fork和进程间通信。

### pVM fork

fork API是实现pVM的传统多进程抽象的基础。 KylinX将每个用户域（pVM）视为一个进程，当应用程序调用fork时，将生成一个新的pVM。

主要利用Xen的内存共享机制来实现fork操作，通过复制xc_dom_image结构和调用Xen的unpause（）API来fork父进程并返回其ID来创建子pVM。当在父进程中调用fork（）时，我们使用内联汇编来获取CPU寄存器的当前状态并将它们传递给子寄存器。控制域（dom0）负责fork和启动子进程。我们修改libxc以便在创建父进程时将xc_dom_image结构保留在内存中，这样当调用fork（）时，此结构可以直接映射到子进程的虚拟地址空间，然后父进程可以与子进程通过使用特权表来共享特定的内存空间，可写数据则以写时复制（CoW）的方式在进程间共享。

通过unpause（）启动子进程之后，它接受来自其父级的共享页面，恢复CPU寄存器并在fork之后跳转到下一条指令，开始作为子进程运行。

在fork（）完成后，KylinX异步初始化一个事件通道，并在父进程和子进程之间共享专用页面以开始它们进程之间的通讯。

### Inter-pVM Communication (IpC)

KylinX提供了一个多进程应用程序，其所有进程都在OS管理程序上协同运行。 目前，KylinX遵循严格的隔离模型，其中只有相互信任的进程才能相互通信，只有属于同一族的进程才被认为是可信的并且允许彼此通信。 KylinX拒绝不受信任的进程之间的通信请求。如图所示的是KylinX中进程间通讯的的一些API。
![KylinX进程通信API](http://plazfv8ew.bkt.clouddn.com/KylinX%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1API.png)



### pVM Library Update

KylinX要求新旧lib是二进制兼容的，在新的lib中KylinX允许向其中添加新的函数和变量，但不允许更改函数的接口，删除函数/变量或更改结构。 对于正在使用的库的状态，我们期望所有状态都存储为变量（或动态分配的结构），这些变量将在更新期间保存和恢复。KylinX提供了更新库的API，用于为domU中进程ID为domid的 pVM动态替换旧的lib为新的lib，并且更新库状态。同时还提供了一个更新命令“update domid，new lib，old lib”，用于解析参数和调用update（）API。

进程库的动态更新的难点在于操纵VM中的符号表。这里利用dom0来解决这个问题。在调用更新API时，dom0将新库映射到dom0的虚拟地址空间; 与domU共享加载的lib; 通过要求domU检查domU的每个内核线程的调用堆栈来验证旧库是否静止; 等到旧库未使用并暂停执行;将受影响的符号条目修改为适当的地址;最后释放旧lib。

在执行动态库映射时，与静态的Unikernel相比，KylinX应该隔离任何新的漏洞。 主要威胁是攻击者可能会将恶意库加载到pVM的地址空间中，将要更新的库替换为具有相同名称和符号的受损库，或者将共享库的符号表中的条目修改为伪符号/功能。

为了解决这些威胁，KylinX强制限制库的身份以及库的加载器。 KylinX支持开发人员指定对动态库的签名，版本和加载器的限制，这些限制存储在pVM映像的标头中，并在链接库之前对更新的库进行验证。

有了这些限制，与静态的Unikernel相比，KylinX没有引入任何新的威胁。例如，对签名（作为可信开发人员），版本（作为特定版本号）和加载器（作为pVM本身）已进行检查后的pVM的在进行运行时库的动态更新时将具有与重新编译和重新启动相同的安全性保证

## KylinX 的总结与思考

### 要点总结

上文介绍了KylinX在进程fork以及动态更新库时一些具体细节和流程。该小节就是展示一下KylinX扩展Unikernel时实现的一些要点。

1. KylinX将hypervisor作为OS，Unikernel appliance 作为process。
2. 在hypervisor中实现动态映射lib而不是在libOS。 
3. 将hypervisor和dom0作为TCB可信计算基础。 
4. Fork和IPC通过共享内存和特权表实现并且存在跨域的页面映射。 
5. 不同的进程间通过事件通道和专门的内存页进行通讯。 
6. 在Xen control library中实现动态库映射机制。 
7. 为了快速启动一个PVM而在一个应用中以动态lib的方式来启动。在一个应用的空间中允许其他应用的PVM以动态lib的方式启动。
8. IPC仅仅允许相互信任的从同一根节点复制的同一族的pVMs进行通讯。
9. 库的动态更新，允许添加新的函数或者变量到库中。但是不允许改变函数的接口，移除函数或者变量，或者改变结构。 10) pVM的循环使用是重用内存中空的域，并将应用如动态库动态映射到那个域。应用本身被编译成为共享lib并使用app_entry作为它的入口。 11) 动态更新lib时对相关库的检查。

### Unikernel 多线程的优劣

libOS 是 Unikernel 的中间层，是构建 Unikernel 的核心。libOS的通用架构为kernel base和function libraries，libOS的kernel base中主要包含的功能一般为线程管理，cpu调度，虚拟内存映射以及引导初始化等功能。显而易见，在静态Unikernel中并没有多进程的抽象。而拥有多进程又有那些优势和劣势。

首先谈一下在系统中添加多进程的优势，进程因为其具有独立的内存空间的特点所以其本身具有一定的隔离性和封闭性。每一个进程都是独立的，这在一定程度上给予了系统某些容错性，在上文提到的Unikernel的缺点之一是当某个不安全的具有漏洞的应用运行在Unikernel时，当这个应用的漏洞被不怀好意的人利用之后，可以直接获得root权限，这个系统的最高权限，即在使用Unikernel时root权限极易丢失，因为租户的应用程序也许存在着大量的漏洞，而每一个漏洞都有可能丢失root权限，因为在Unikernel中没有内核态和用户态的隔离。应用程序就在内核态运行，它本身就具有root权限，对于传统系统而言，让用户具有root权限本身就是一件很危险的事情。这同样也是Unikernel极其不安全的一个体现。而在Unikernel中使用多进程也不是说能够防止root权限的丢失，而是可以通过在一个进程出现漏洞时将其及时清理掉从而在一定程度上保证其安全性。因为此时在Unikernel系统存在多个进程，一个进程被清理掉可能并不影响其他进程的正常运行。其次是在初始的Unikernel模型中，只有一个进程，而如果发生未知的异常错误导致进程崩溃，那么在Unikernel中所提供的单一服务也同时崩溃。这在一些应用中是绝对不希望发生的。而多进程在一定程度上也避免了这种情况的发生，即使是Unikernel中的某一个进程因为某个错误或者某些攻击导致了该进程的崩溃，也不会影响整个服务的运行，从而避免了上述的单进程Unikernel崩溃的现象。

在之前讨论Unikernel缺陷时，曾提及在Unikernel中无法进行调试，在对Unikernel相关内容进行详细了解之后认为Unikernel不能进行调试的原因可能是Unikernel仅仅是单进程无法进行调试工作，而在进行多进程扩展之后，开发者就可以在Unikernel中进行应用程序的调试。

以上是说明在对Unikernel实现多进程扩展之后所具有的一些优势，但是同样有一些可能问题，在论文中并没有对这些问题进行说明。

在扩展多进程之后所必然会提及到的问题是进程之间的调度问题。在论文中主要说明的是进程如何fork一个新的进程，以及如何快速启动一个进程，但是没有提及在KylinX中进程是如何调度的。在论文中介绍lib的动态更新是通过Xen中的库实现。由于KylinX论文中并没有说进程调度在哪里实现，所以这里只能猜测进程调度是如何实现的。Lib的动态更新可能是因为其机制问题所以才需要在Xen的库中实现，而进程的调度应该如libOS自身所具有的线程调度库一般，将进程调度的相关函数或者是相关库放入libOS中。其次是多进程抢占资源时锁的问题。在Linux系统中实现了RCU和读写锁，在一般情况下，RCU相对于读写锁有更好的性能。而在KylinX中是否实现了相关的管理机制，还是需要使用者自己去实现相关进程调度管理。

最后尽管KylinX实现了多进程，但是依然没有解决Unikernel的一个重大的缺陷，即对于开发人员非常高的要求，应用开发人员需要自己实现与硬件资源交互的功能。对于应用开发人员是比较大的负载。

## Unikernel在云环境中的应用

### Unikernel 在云环境中的应用

在多租户的云环境中，虚拟化是维持不同租户之间保持隔离性的标准形式。所谓的虚拟化，就是使用虚拟化技术将物理的计算机虚拟为多台逻辑计算机。不同的逻辑计算机可以运行在独立的空间内而不互相影响，从而提高资源的利用率。虚拟化技术的核心是虚拟机监视器（VMM）, 是一种运行在物理服务和操作系统之间的中间层软件，它是一种在虚拟环境中的“元”操作系统。虚拟机监视器可以访问服务器上包括CPU、内存、硬盘、网卡在内的所有的物理设备，协调虚拟机与硬件资源的访问，在各个虚拟机之间施加防护。虚拟化技术可以分为全虚拟化技术和半虚拟化技术。在传统的云环境中，虚拟机运行的操作系统是通用的操作系统，通用的操作系统常常包含额外的，租户并不需要的功能，不仅造成了资源的浪费，还引入了潜在的安全问题。Unikernel 作为一种使用libOs定制的具有专门用途的镜像，具有轻量级的特点，可以用来代替传统通用的操作系统，将Unikernel作为虚拟机在云环境中运行，为用户提供服务。
![使用虚化技术获取隔离性典型的 unikernel 架构](http://plazfv8ew.bkt.clouddn.com/%E4%BD%BF%E7%94%A8%E8%99%9A%E5%8C%96%E6%8A%80%E6%9C%AF%E8%8E%B7%E5%8F%96%E9%9A%94%E7%A6%BB%E6%80%A7%E5%85%B8%E5%9E%8B%E7%9A%84%20unikernel%20%E6%9E%B6%E6%9E%84.png)


如上图所示，是使用虚拟化硬件来获取隔离性一个典型的unikernel架构。运行在 linux 内核之中的 KVM 负责与虚拟化硬件（例如 VT-x）交互，并将其暴露给运行于用户空间的 unikernel 监视进程。Unikernel 监视进程的作用主要是在初始化的时候为 unikernel 虚拟机分配内存、cpu，通过系统调用与运行在内核中的KVM模块交互以正确的加载和启动 unikernel 虚拟机。

当使用虚拟化技术的时候，unikernel 访问硬件资源需要通过使用超级调用，由虚拟机监视进程通过KVM访问硬件资源，在将访问结果返回给 unikernel 虚拟机。在这个环境下， unikernel 虚拟机的隔离性来源于监视进程与 unikernel 虚拟机之间超级调用接口的数量大小。相比于 Linux 的系统调用接口，监视进程提供给 unikernel 的接口数量相当少。如下图所示的ukvm, 其提供给 unikernel的超级接口仅10个，与超过300的Linux系统调用数量相比，可以说是安全的多了。由此可见，使用 unikernel 代替传统通用的操作系统作为虚拟机，可以大大提高租户之间的隔离性。

![超级调用与系统调用的对映关系](http://plazfv8ew.bkt.clouddn.com/%E8%B6%85%E7%BA%A7%E8%B0%83%E7%94%A8%E4%B8%8E%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%E7%9A%84%E5%AF%B9%E6%98%A0%E5%85%B3%E7%B3%BB.png)

### 云环境中使用硬件虚拟化的限制

尽管使用 unikernel 代替传统通用的操作系统，可以精简系统接口，大大提高不同租户之间的隔离性，但是使用硬件辅助虚拟化也引入了不少缺点。
随着虚拟机成为运行在云端的主要单元，常见的进程级的工具，例如调试工具等，都可以在客户端操作系统上使用。然而，unikernel是不可调试的，常见的调试工具都失去了作用。尽管有一些通常应用于内核开发的调试技术，例如在监视进程中连接 gdb 的客户端，可以实现调试，但这些技术不适用于应用开发。

对于 unikernel客户机来说，不论是否使用同样的只读系统文件，所有的unikernel客户机都独立的执行文件缓存，这样就造成了内存的浪费。为提高内存的有效利用率，可以使用内核同页合并（KSM）技术来合并内存中副本页面，释放这些页面以供它用。但是KSM操作肯定会消耗额外的CPU时钟周期，且会提高复杂性。

在性能方面，当 unikernel 虚拟机每次执行一个超级调用的时候都会引起监视进程上下文和虚拟CPU上下文的切换。执行一个超级调用的时钟开销是执行一个方法调用的两倍之多。
最后，尽管虚拟化硬件的物理可用性不再是现代数据中心的关注点，但现有的基础设施机服务云常常是作为构建新的云服务的基础。例如OpenWhisk无服务设施就是一个搭建在虚拟机集群上。遗憾的是，云提供商很少支持嵌套虚拟化，也就意味着unikernel 虚拟机在云上的虚拟化没有硬件支持。

### 将Unikernel 当作进程运行

当unikernel 运行在监视进程之上的时候，unikernel 更像是一个应用而不是一个内核。因此本文的思路就是把 unikernel 当作一个进程运行，来避免使用虚拟化硬件所带来的限制。将 unikernel 当作进程运行时的隔离性由系统调用白名单机制解决。将 unikernel 作为进程运行的架构图如4.3所示。

![Unikernel 作为进程运行架构](http://plazfv8ew.bkt.clouddn.com/Unikernel%20%E4%BD%9C%E4%B8%BA%E8%BF%9B%E7%A8%8B%E8%BF%90%E8%A1%8C%E6%9E%B6%E6%9E%84.png)

在将 unikernel 当作进程运行时，使用 tender 进程代替监视进程，tender 进程的作用与监视进程的作用类似，同样包含两个功能：初始的设置功能和退出处理。与监视进程不同的是，在初始设置功能中，tender 进程除了进行 I/O 和文件的初始化外，tender进程将 unikernel 动态地加载进自己的地址空间，并为 unikernel 分配堆空间。unikernel 会继承 tender 的寄存器状态和栈， 因此它不需要分配额外的寄存器和临时栈。在加载 unikernel后，初始化设置还将配置 seccomp 过滤器，以限制 unikernel 进程对系统调用的访问。
在初始化配置后，tender进程调用 unikernel 的入口函数启动 unikernel。当 unikernel 启动后，unikernel 就像通常一样执行，与作为虚拟机运行时不同的是，当它需要执行超级调用时，不用再进行虚拟 CPU 和 unikernel 之间的上下文的切换，只需要像调用方法函数那样调用 tender 中的实现超级调用，tender通过执行系统调用来实现超级调用。作为进程，由于 unikernel 是运行在 tender 之中的，通过 tender调用底层的系统调用来实现超级调用。因此 unikernel 的隔离性不再来源于它与 tender 进程之间接口，而是 tender 与 unikernel 构成的进程与底层系统之间系统调用接口。在图3架构中，通过Linux内核的安全保护机制 seccomp, 配置tender 进程系统调用的白名单来限制对系统调用的访问，以此来获取与典型架构同样的隔离性。

### 将 unikernel 作为进程的增益与代价

在将 unikernel 作为进程运行的时候，主系统为进程提供的许多特性都可以应用在 unikernel 进程上。在没有修改的情况下，主系统可以为unikernel进程提供如下特性与增益：

1. 动态地址空间布局。每一次执行，Unikernel 会被动态加载器加载到不同的地址，使攻击者难以确定正确的目标地址。
2. 常用工具的支持。当需要对 unikernel 进程进行调试的时候，常用的基于进程的调试工具，例如gdb, perf 和 valgrind 可以不经修改，处理动态库的调试。
3. 内存共享。当多个unikernel的副本在同一台机器上运行的时候， tender 进程能够保证包含 unikernel 库的内存能够被主机共享。
4. 与位置无关的独立架构。当unikernel作为虚拟机运行时，需要与位置相关的代码来启动，配置内存管理或者处理中断等，作为进程运行时，编译时只需把 unikernel 编译成与位置无关的应用即可，从而在多种架构中运行
5. 性能的提升。作为进程运行时，unikernel 将超级调用的执行转化成对同处于一个进程tender中方法的调用，避免了unikernel和监视器进程间上下文的切换。

为了将unikernel作为进程运行，也要付出一定的代价。Unikernel 本身是作为虚拟机运行的，在编译时是静态编译的，因此编译出来的的版本包含与位置有关的代码。为了能够将unikernel动态的加载进监视器，作为进程运行，需要重新将 unikernel 编译成与位置无关的版本。在重新编译 unikernel 时，不同的 unikernel会面临不同的问题。如有些unikernel的库操作系统，例如 Rumprun，包含非嵌入式的汇编代码，需要重构这部分代码才能将 unikernel 编译成位置无关的代码。

### 对Unikernel作为进程运行的思考

俗话说，所有问题都可以通过增加一层抽象或减少一层抽象解决，将 unikernel 作为进程运行就是减少了虚拟机监视器这一层抽象，将 unikernel 与虚拟机监视器融为一层。但是减少一层抽象就要付出相应的代价，为了实现将 unikernel 作为进程运行，现有的虚拟机监视器不能实现，需要为虚拟机监视器增加新的特性才能支持 unikernel 作为进程运行。将unikernel作为进程运行时，将隔离性保障从unikernel与监视器之间的接口转化为依赖 seccomp 机制保障unikernel进程与主系统之间的接口，看似与原有的隔离性一致，但是这是建立在信任seccomp的机制上的。当 unikernel 作为虚拟机运行时，虚拟机监视程序可以同时管理多个unikernel虚拟机，如果使unikernel作为进程运行，虚拟机监视器是将unikernel 加载到自身地址空间中，监视器与 unikernel共同构成一个进程，为了保持隔离性，是否意味着虚拟监视器与unikernel一一对应。即需要多个虚拟机监视器来对应不同租户的unikernel。

## 总  结

在商用云服务中，大多虚拟机的操作系统是通用的操作系统例如Linux 等，通用操作系统为用户提供了广泛的库和功能。而对于云端租户来说，他们在云端运行的是特定目的的服务或应用，这些特定服务或应用只需通用操作系统少部分库和功能。通用操作系统的多余库和功能对于这些租户来说是一种累赘，不仅占用了多余的空间，浪费了资源，还增大了系统的攻击面。本文介绍了一种可以有效解决这些问题的方案：Unikernel。Unikernel 消除了用户态和系统态之间的转换及数据复制的开销，提高了系统的性能，同时由于Unikernel非常小，相对的攻击面也更加小，因此 Unikernel 也更加安全。但Unikernel 也存在一些争议，如 unikernel 无法调试等，不支持多进程。针对这些争议，本文介绍了两项工作，一项解决了unikernel 不支持多进程的缺陷，一项工作解决了 unikernel在云环境应用中受到的限制。并对两项工作进行了总结与思考。

## 参考文献

1. Unikernel的研究及其进展
2. 库操作系统的研究及其进展
3. Include OS：A minimal，resource efficient unikernel for cloud services
4. Unikernels as processes
5. KylinX: A Dynamic Library Operating System for Simplified and Efficient Cloud Virtualization
6. Unikernels: Library operating systems for the cloud
7. https://www.hpe.com/us/en/insights/articles/what-is-a-unikernel-and-why-does-it-matter-1710.html
8. https://www.joyent.com/blog/unikernels-are-unfit-for-production?spm=a2c4e.11153940.blogcont55911.9.42ff110eWmSvRV
9. https://cloud.tencent.com/info/43aef446698e2e7a126500880f1b4652.html
10. http://gaocegege.com/Blog/%E5%AE%89%E5%88%A9/unikernel-book
11. Unikernels





