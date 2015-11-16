---
layout: page
title: Note On Are “Virtual-Machine Monitors Microkernels Done Right?”
categories:
    - cookies
---

这是两篇名字完全一样的论文，分别来自于Steven Hand（Xen小组）以及Gernot Heiser（L4小组）。两篇文章分别论证了各自持有的“为什么我比对方厉害”的观点，非常有趣。其中第二篇是一篇标准的反驳，所以相互对应着看，也比较清晰。

Hand作为第一篇的作者以及Xen的成员，要更加谦虚一些。他首先反对了“学术界的操作系统研究没有影响力”以及“操作系统研究毫无必要”的观点，认为VMM是对学术界的microkernel的一种approach，并且出于合理的对IPC的定位以及优秀的performance/工业界的流行，是一种micorkernel done right。

> We argue that modern VMMs present a practical platform which allows the development and deployment of innovative systems research: in essence, VMMs are microkernels done right. 

Hand一方面提到了microkernel在设计时的动机：一个更小的OS自然地带来易维护易拓展等好处，另一方面也提到现今的microkernel在performance上并不一定会输给商业化的unix的各种变体。同时也表示，VMM和microkernel虽有很多共同点，但是VMM在overhead上做的更好，并且VMM只需要修改少量guest OS代码即可使用（提供full virtualization的VMware甚至不需要修改guestOS的任何代码）的特点，使得VMM成为了一个十分有诱惑力的解决方案。

Heiser激烈地反对了Hand隐含的L4是“microkernel done false”的观点，他不认为这是对microkernel的两种不同的approach，而是在目标上就有一系列的不同。本质上来说，microkernel仍然试图在设计一种OS，而VMM则是对硬件资源的软件化，所以它的core primitives要更多，导致对应的问题也更多。而microkernel的核心问题就是IPC，所有的一切都是以IPC构建起的一些精妙的组件，从概念上讲，只要综合文中提到的三类IPC，microkernel就是一种非常干净的OS，任何操作都可以通过IPC的组合来解决。

看上去Heiser的反驳有点太理论，的确Hand的文章在后面也提到说IPC并不是VMM的关键，VMM对待GuestOS要更加粗放一些。但是我觉得Heiser在讲到Core primitives的最后，是比较有道理的。VMM很大的优点就在于GuestOS需要修改的代码很少，但是，为了支持所有合法的OS，它不得不从原先一个pure的虚拟化到为了不同的硬件提供不同的接口，这显著的提升了设计的复杂度。而L4就在九种不同的cpu上都可以运行，并且接口基本保持一致，这是VMM所做不到的。（然而好像paravirtualization有hypercall这种提高性能的东西，所以这一指责应该主要还是对设计的批评）

### **Architectural Lessons**

---

#### **Avoid Liability Inversion**

Hand和Heiser都认为对方存在这个问题。Hand以pager举例，他认为：microkernel是将原本OS内的功能分割成在user-space的组件，因此，用户级应用必须依赖于其他的用户级组件来运行，更有重要的是kernel本身也会依赖于用户级的组件，比如page fault handler(pager)。如果一个组件的pager发生了错误，依赖于这个组件的其他组件也会出错，相当于是说一个发生在小处的错误就扩散了。而VMM的设计则避免了这一点，它的内存管理系统不存在pager这一说，只需要将内存分割好之后分给不同的domain。VMM在isolation和sharing的取舍中做的更好。

Heiser指出Xen有相同的问题。为VMM提供存储的Parallax使用了一个外部的pager（不太明白..）声称避免了liability inversion，但是这和一个user-level的server所做的其实没有任何区别，所以在这个问题上microkernel和VMM没有什么区分。

#### **Make IPC Performance Irrelevant**

Hand在这一节里表示：与microkernel不同，IPC在VMM的设计中并不占有关键性的地位。存在一系列原因：
* VMM的核心问题是多台domain间的隔离，所以，多台虚拟机之间的synchronization and protected IPC其实是不必要的。
* 对于需要用到的IPC，VMM也做了一个优化：将消息分为data和control，其中data通过异步的比较不消耗系统资源的方式传递。

Heiser也从两个角度反驳了他：
* Xen所谓的simple asychronous event mechanism，本质上也就是Domain0和guestOS间的one round-trip communication，是一种asychronous IPC。并且这样的开销也很大，依据就是Dom0的CPU开销是和Xen的page-flipping的操作数成正比的。
* 尽管VMM调度的是整个OS，但是VMM中的interaction还是非常多的：比如异常和syscall都会导致一个进入VMM的trap，VMM重新再将调度权还给guest，这其实就是一种IPC。Xen所提供的shortcut也是受到限制的。从IPC的操作数上来说，L4和Xen基本上是一样的。

#### **Treat the OS as a Component**

Hand指出：VMM和microkernel关键性的不同就在于组件粒度（granularity of componentization）的不同。目前的microkernel项目都需要花很大的努力去修改API来支持已有的OS，而VMM（因为标题的缘故）只需要修改很少量的代码，，所以在工业界获得了流行（隐含的意思当然是microkernel并不流行）。其次，VMM都是基于Dom0（不是windows就是linux）的，用户可以在熟悉的环境里进行开发，作者举了Parallax这个存储系统的例子，表示它提供了和原先相近的block interface。

Heiser在这一节表示，其实microkernal也早有DROPS系统实现了差不多的功能并且被业界使用（感觉回应的比较无力..）。

#### **Conclusion**

Hand认为Xen是一种从microkernel走出的、对学界之外亦有重要影响的设计，隐含的意思当然是：Xen走了一条正确的道路。Heiser对文题坚定的说了No，主要在于他不认为VMM是一种microkernel，这是两个目的上就有不同的东西。