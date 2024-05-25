# Linux System Calls

## 按抽象层次分类
广义的Linux syscall按照抽象层次可分三类：
1. Kernel call，也就是Linux man pages volume 2的API，查源码就能发现，他们都只是thin wrappers of ABIs（API是编程接口，也就是平时写代码调用的东西；ABI是二进制接口），这些ABI是kernel提供的真正的接口。因此，容易理解为什么这些API的参数都不超过6个———x86-64里，syscall指令用%rax传syscall number和返回值，并用六个寄存器传参；此外，这些API的返回值有个convention（volume 3的API基本上也遵循这个）——返回值小于零表示出错，大于等于零表示成功。
2. Library functions，也就是Linux man pages volume 3的API，他们是基于volume 2的API实现的。例如，execv(3)就是基于execve(2)实现的（注：Linux里，API后接括号里的数字指代这一API属于man pages的哪一个volume）。还有一些API甚至就不需要trap into kernel，例如用于字节序转换的htons。编程时区分这两个volume的意义不大。
3. 高级编程语言里的syscall。前面两类都是C接口，这一类是编程语言基于C接口实现的高层次接口，编程的时候尽量采用这一种（做同样的事情尽可能在高层次做，省心，thinking low-level, writing high-level）。

## 资源
1. Man Pages
    - 两种获取途径：shell里的man指令，或者上网查。网上有很多提供man pages的网站，我看得最多的是man7.org，此外die.net也是一个常见的靠谱渠道，只不过相对比较老。如果要查最新的man pages，那么最新版Linux的man指令一定是最新的，或者Arch Linux的也比较新。
2. Michael Kerrisk, The Linux Programming Interface
    - 这是一本工具书，非常详尽地列举了Linux方方面面的syscall。有需要的时候可以直接查阅特定的章节。
    - 还有很多人推荐Advanced Programming in the UNIX Environment，不过那本书相对比较老（epoll这种2002年就有的API在那本书里都没有）。
    - 肯定还有很多不错的书，因为我自己没看过就不推荐了。
3. 各种编程语言的文档。
    - 前两种资源只针对C接口，但是我们绝大多数时候只会使用各种编程语言里的高级接口，因此这些文档才是我们用得最多的。
4. 找Copilot/ChatGPT要example code。
    - 对于常用的API，他们能给出不错的example code，但是对于新API（比如io_uring），这招就不管用了。
5. GitHub
    - 对于一些很新的、文档缺乏的API（比如io_uring），去它的GitHub repo提issue/搜索别人提过的issue也是一个非常好的途径！即便只是不知道一件事情怎么做，也可以提问！并不是只有发现bug才能提issue。不过，对于新API，如果用得多了，发现bug也是很正常的事:)
    - 有些时候，如果实在没有办法，稍微读一点源码也是个办法……

## 按功能分类
这是本文的主体部分。这一部分不求全，而是评述一些比较重要、我用得比较多的类别。

### Thread
这是我用得最多的一类。现代编程语言还会基于kernel thread实现出user thread/coroutine还有各种user-space synchronization。这是很大的话题。本文就不展开了。

除了出题以外，我从来没有用pthread接口写过代码。其实pthread接口虽然看着有点恐怖，但是总体上设计风格还是比较统一， 比如：
- 根据syscall convention，返回值表示是否成功（负数失败，非负数成功）。
- OOP风格：很多函数的第一个参数可以理解为OOP里的对象，这样一转换，就会发现pthread接口就和现代语言里的接口接近很多了。
- pthread经常有一个attribute参数，这确实看着让人害怕，但是绝大多数时候这个参数传个null就行，毕竟default行为一般够了。

我反对用pthread进行多线程编程教学。我一贯的理念是，初学者应该学习抽象层次最高的接口，这样才容易抓住重点。考虑到C++是北大同学比较熟悉的语言，它是比较适合给初学者教授多线程的；尽可能使用C++的async或jthread而不是thread。

Python的GIL目前还没去掉（3.13在今年十月正式发布，那时候才有去掉GIL的版本），因此不建议采用Python教授多线程。此外，任何涉及性能的编程都应该尽可能避开Python。

Rust的多线程和synchronization比较难学，做好心理准备。

### Files

#### Files in Local Disk
文件读写也是非常常用的（大概大多数人第一次接触syscall就是文件读写，如果standard I/O不算的话）。

POSIX的文件读写接口设计得不错，功能也十分强大，但是尽可能还是不要直接用，因为缺少user-space buffer。C++的fstream是个不错的东西。相比之下，Java的文件读写值得吐槽——打开一个文件要写三行，打开、节点流、处理流（John Ousterhout把这称为overexposure，就是像用户暴露了过多不必要的信息）。

Linux里与file相关的有三个table：file descriptor table, open file table, vnode table。这提供了很大的灵活性，但是作为用户其实并不需要接触到这些。

#### Databases
其实很多文件访问（可能大多数时候都是）是通过各种数据库进行的。关系型数据库我比较推荐PostgreSQL。此外MongoDB、Redis也是常用的DB。

#### Cloud Storage
在这个云计算的时代，各种cloud storage和cloud DB层出不穷（在这里DB和file system其实没有明确的边界）。这方面可以参考AWS、Databricks等公司的各种产品，他们也产出了很多很不错的paper，比如Dynamo、Photon（CMU的advanced database systems课有给出几个这样的例子）。

此外，与过去所教授的不同的是，现代数据中心里网络比SSD更快（无论是延迟还是带宽），因此disaggregated storage才是更常见的形式。也就是说，虽然local disk仍然在用，但是这一般只是一个cache，真正persistently存数据的storage一般和做计算的机器并不放在一起。

对于数据中心的硬件条件介绍是当下教学中非常缺失的内容。

### Networking
John Ousterhout表示，socket接口是UNIX接口里他最不喜欢的部分。我深有同感。UNIX sockets可以说是极其痛苦的一套接口。

socket诞生于八十年代初的Berkeley，那个时候的人们对于网络编程的应用场景接近一无所知。socket（尤其是TCP socket）提供的抽象是一个stream，而且是一个没有message boundary的stream，这应该是模仿UNIX pipe的设计。然而，现代网络编程一般基于HTTP或RPC，这些都是message-based的编程模型，而非streaming。

C++至今没有网络接口，Boost里的ASIO的网络接口也是让人感到痛苦的。

Python里的socket接口基本上照搬UNIX socket。所幸它提供了一些高层次的抽象比如`socket.create_connection`和`socket.create_server`。

不过，就学习建议而言，我认为最值得学习的是RPC。目前生态最好的RPC是gRPC，不过那对于初学者可能稍微有难度。很多编程语言里有一些相对简单的RPC库，比如Python的xmlrpc，go的`net/rpc`，这些都可以作为入门。

RPC之外值得学的主要就是HTTP了。作为用户，并不需要操心DNS（一般来说尽可能用域名就行），至于MAC地址是什么那根本不需要关心（诚然，UNIX socket允许我们在链路层发包，但是那真的基本上不会用到，我只见过Zmap用了这个）。

UNIX socket其实是一套非常灵活而强大的接口，`setsockopt`和各种API里的flag提供了各种各样的可能性，但是，这些还是等老道一些再来琢磨吧，高层次的编程语言一般能用舒服得多的方式实现同样的功能。

Rust的tokio是我见过最好的socket接口。但是，尽量别直接用socket。Rust各类网络编程库非常丰富。

### Processes
CSAPP基于process讲授thread是不恰当的。现在，thread主要的目的是concurrency，相比之下，process有很浓厚的virtualization色彩，多数时候他们的应用场景不会重合（当然，不是绝对的）。

multiprocessing的C接口我只用过几次，更多时候是Python脚本里用`multiprocessing`和`subprocess`去执行一个shell命令，要不然就是写分布式系统的时候用多个process模拟多台机器。

UNIX process API总体上设计得不错，但是CSAPP讲得并不容易懂；此外，CSAPP也不应该把fork和exec拆散并且拿前者做太多的文章（各种烧脑练习题）。此外，对于初学者，execvp足以，可以暂时忽略环境变量。

不过，我并不认为初学者有必要弄清fork和exec。理解process这个模型是重要的，但是如何创建它，需要的时候再看就行（实际上也记不住）。

### IPC
The Linux Programming Interface 43.1有一个很好的概述。它把IPC分为三类：communication，synchronization，signal。我这里着重想讨论一下signal。

#### Signal
曾经教同学们如何避开signal的各种天坑的我，在多年以后对signal进行编程的时候，不出意外地自己也进坑里去了。

这是我见过最tricky的一种control flow———signal handler可以在我完全意料不到的地方开始执行，并且它没执行完之前我的程序还不能执行。关键是，它还会在我不想他来的时候来啊！

当年我自学CSAPP的时候，看到signal就十分头疼，明明一个`while(1);`死循环，怎么这程序就还能结束了呢？

UNIX signal实际上是一个极其强大而灵活的机制，signal handler的参数也可以获得大量的信息，很多syscall里也有signal有关的参数（就是可以选择性地block一些signal）。但是，我想说，随着技术发展，signal的应用场景越来越少了，它在绝大多数时候都有更好的替代。例如，signal很多时候被用来indicate timeout（或者说提供timer interrupt），但是现在很多syscall都接受timeout参数，后者的control flow明显更干净。

ICS的shell lab几乎可以说是体现了唯一需要signal的场景（只考虑用户态）——对用户从键盘发来的信息(如ctrl+C)做出反应。

一个比较常见的应用场景是注册一个SIGTERM handler，这样用户kill当前process的时候可以做一些清理现场的工作。但是，这个应用十分简单，以至于根本不需要专门学signal。我觉得学习Linux programming的时候真的没必要专门学signal，需要的时候迅速看一下就行了。

### Asynchronous Programming
#### High-level: Coroutines
coroutine是现代编程语言里用于asynchronous programming的抽象，它主要分为stackful coroutine和stackless coroutine，前者的编程模型和thread完全一样，后者就是async-await风格。这是一个很大的坑，本文不展开。
#### Low-level: Mechanisms Provided by Linux
本文着重讨论一下OS提供的async机制。The Linux Programming Interface第63章提供了很多介绍，美中不足的是缺少最新的io_uring。

1. I/O multiplexing: select/poll/epoll
    - 前两个已经被第三个取代，因此现在只关注epoll就行。
    - 这些接口的本质是：进行I/O之前，先得知这个操作是不是ready，例如，当socket的另一端发数据过来的时候，这个socket就会变为read ready。
        - 这些接口不适用于local file，因为它们永远是ready。而且，io_uring出现之前，file I/O并不支持non-blocking。
    - 这是一种event-driven风格的编程方式，拥有难以理解的control flow，写的时候需要十分小心。
    - 实际使用的时候不一定要直接写epoll，`libevent`和`libev`是不错的替代。
    - 不建议初学者学这个。
2. Signal
    - 没错，signal结合non-blocking API也是可以做async programming的。至于有多阴间，可想而知。
3. aio
    - 已淘汰。不讨论。
4. io_uring
    - 最新的机制，2019年才出现，文档严重不足。
    - 最高效的机制。
    - 编程接口、control flow比其他几种友好得多。
    - file和socket都支持。
    - 很强大的机制，就不展开了，但是很推荐学。推荐[Lord of the io_uring](https://unixism.net/loti/)。

## 哪些编程语言的syscall值得推荐？
首先，这取决于我们实际的项目需求。如果从学习的角度，Rust大概是设计得最好的，但是问题是Rust本身比较难学。其他主流编程语言总体上做得也都不错，C++、Java、C#、Python、Go的接口都可以学，以上只是一个不完整的语言列表。

## 附录0：特别推荐
John Ousterhout有一本小册子A Philosophy of Software Design，里面详细探讨了各种系统设计的原则和实例。这是我愿以名义担保推荐的一本书。

## 附录1：如何使用最新版Linux kernel
如果你是Windows用户，那么恭喜，WSL2支持自己编译并运行最新版的Linux kernel，并且不需要修改任何源码。方法是：
1. 从Linus Torvalds的[仓库](https://github.com/torvalds/linux)clone源码，还可以随心所欲修改。
2. 从WSL官方的Linux发行版复制一个[配置文件](https://github.com/microsoft/WSL2-Linux-Kernel/blob/linux-msft-wsl-5.15.y/arch/x86/configs/config-wsl)到前述源码仓库里。
3. build。具体过程参见[微软的教程](https://learn.microsoft.com/en-us/community/content/wsl-user-msft-kernel-v6)。其实挺容易。

## 附录2：如何使用最新版的接口？
这需要以下几部分：
1. 最新版的Linux kernel。参见附录1.
2. 最新版的compiler和library，这每个语言不一样。
3. 最新版的Glibc。
    - 这一点比较tricky。不要轻易自己安装glibc！这有可能导致连shell都启动不了（因为shell和shell里一些很基本的命令如ls都需要动态链接glibc，所以搞坏了这个东西后果比较严重）。
    - 一般来说，glibc版本取决于Linux发行版（不是kernel版本，而是例如Ubuntu版本）。因此，如果想要尝鲜，使用non-LTS版的Ubuntu可能是必要的。