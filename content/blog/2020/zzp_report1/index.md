---
title: "内核中的同步与任务调度"
date: 2020-12-3T16:50:24+08:00
author: "康华"
keywords: ["时钟"]
categories : ["内核同步"]
banner : "img/blogimg/1.png"
summary : "同步概念在多线程和多进程编程中已经被诠释得很全面。同步方法对于用户应用程序来讲使用简单，无需过多考虑它们产生的原因（唯一的原因就是线程或进程并发）。但是内核中的同步处理就要复杂得多，开发者必须知道内核中任务得调度方式，才能有效的控制内核中的同步。所以本文就将结合内核任务调度，分析内核中的同步措施，并结合一个实例讲述内核中如何综合运用各种同步方法。"
---

# 内核中的同步与任务调度

**本文作者**：

**康华**：计算机硕士，主要从事Linux操作系统内核、Linux技术标准、计算机安全、软件测试等领域的研究与开发工作，现就职于信息产业部软件与集成电路促进中心所属的MII-HP Linux软件实验室。如果需要可以联系通过kanghua151@msn.com联系他。

 

**摘要**：同步概念在多线程和多进程编程中已经被诠释得很全面。同步方法对于用户应用程序来讲使用简单，无需过多考虑它们产生的原因（唯一的原因就是线程或进程并发）。但是内核中的同步处理就要复杂得多，开发者必须知道内核中任务得调度方式，才能有效的控制内核中的同步。所以本文就将结合内核任务调度，分析内核中的同步措施，并结合一个实例讲述内核中如何综合运用各种同步方法。

## 并发，竞争与同步：

并发，竞争和同步的概念，我们假定大家都有所了解，本文不再重申。我们讨论的重点放在什么情况会发生内核并发上？如何防止内核并发？有那些同步方法？以及这些方法的行为有何特点和如何使用它们？

下面一段描述了上述几个概念之间的大致关系，这种关系在内核中同样适用。

对于多线程程序的开发者来说，往往会利用多线程访问共享数据，避免繁琐的进程间通讯。但是多线程对共享数据的并发访问有可能产生竞争，使得数据处于不一致状态，所以需要一些同步方法来保护共享数据。多线程的并发执行是由于线程被抢占式的调度——一个线程在对共享数据访问期间（还未完成）被调度程序中断，将另一个线程投入运行——如果新被调度的线程也要对这个共享数据进行访问，就将产生竞争。为了避免竞争产生，需要使线程串行地访问共享数据 ，也就是说访问需要同步——在一方对数据访问结束后，另一方才能对同一数据进行访问。

## 内核并发原因

上述情况是用户空间并发产生的普遍原因，对于内核来说并发原因也大致类似，但是情况要更多样，也更复杂。

对于单处理机器来说情况相对简单一些。在2.6版本内核之前，Linux内核是非抢占式的——在内核任务没有执行完之前不能被打断，这样的话，内核中程序并发执行的情况很少，准确地讲只有两种可能：

一 ：*中断发生* ，因为中断执行是异步的，而且中断是在非抢占式内核中打断当前运行内核代码的唯一方法，所以中断显然是可以和其它内核代码并发执行的。因此如果中断操作和被中断的那内核代码都访问同样的内核数据，那么就会发生竞争。

二 *：睡眠和再调度*, 处于进程上下文（下面会进行讲述）的内核任务可以睡眠（睡眠意味放弃处理器），这时调度程序会调度其它程序去执行（首先执行调度任务队列中的内核任务，然后执行软中断等，最后从运行队列中选择一个高优先级的用户进程运行）。显然这里也会造成内核并发访问，当睡眠的内核任务和新投入运行的内核任务访问同一共享数据时，就发生了竞争。请看参考资料 1

2.6版本的内核变成了抢占式内核——内核可能在任何时刻抢占正在运行的内核代码。所以内核中发生并发执行的情况大大增加了。*内核抢占*成为了内核程序并发的又一种可能，所以在开发抢占式内核代码时需要时刻警惕抢占产生的竞争。

单处理器上的并发是逻辑上的伪并发，事实上所谓并发的内核程序其实是交错地占用处理器。真正的并发执行程序，必须依靠对称多处理器。但无论是逻辑上的并发还是真正的并发，都会产生竞争，而且它们的处理也是相同的。但是对于对称多处理器来说，由于两个或多个处理器可以在同一时刻执行代码，所以会不可避免地给内核带来并发可能，如果分别在不同处理器上执行的内核代码同时访问同一共享数据，竞争就产生了。因此，不用说*对称多处理*是内核并发的又一种可能。 请看参考资料2

可以看到随着Linux内核不断演化，在内核对系统支持更加全面，对任务调度更加高效的同时，也给内核带来了更多的并发可能，更容易引起竞争。上面提到的各种并发情况在内核中都必须得到有效的处理，才能确保内核有高稳定性。

无论是中断产生的并发或是睡眠引起的并发，还是内核抢占引起的并发，要想在内核开发中很好地避免，就必须从本质上了解它们的并发原因。只有在掌握内核任务的调度机制后，才可以真正的达到对并发可能的预测，进而能够采取合适的同步方法——锁——来避免并发。

下面我们就对任务调度进行讨论。对比并发产生的条件，分析内核中的调度发生的条件。

## 内核中的任务调度：

我们这里所说的任务调度不同于常说的进程调度。进程调度是：内核中的调度程序在进程运行队列中选择合适的（优先级高的）进程执行。而我们所说的内核任务调度指的是，内核中的任务获得执行机会。对于内核并发来说，内核任务之间的关系尤为重要。

首先我们来看看内核有那些任务，各有什么特点。

### 内核任务种类

***硬中断操作：\***

硬中断是指那些由处理器以外的外设产生的中断，这些中断被处理器接收后交给内核中的中断处理程序处理。要注意的是：第一,硬中断是异步产生的，中断发生后立刻得到处理，也就是说中断操作可以抢占内核中正在运行的代码。这点非常重要。第二，中断操作是发生在中断上下文中的（所谓中断上下文指的是和任何进程无关的上下文环境）。中断上下文中，不可以使用进程相关的资源，也不能够进行调度。请看参考资料2

 

***软中断操作：\***

软中断是Linux中为了执行一些硬中断操作来不及完成的任务而采取的推后执行机制。因为硬中断操作期间的中断会被抛弃，所以硬中断是在不安全时间运行的。不安全时间应该尽量短，所以采用软中断来执行大部分任务，它会把硬中断做不完的耗时任务推后到安全时间执行（软中断期间不会丢弃中断信号）。

软中断不象硬中断那样时随时都能够被执行，笼统来讲软中断会在内核处理任务处理完毕后返回用户级程序前得到处理机会。*具体的讲有三个时刻它将被执行**(do_softirq())**：硬件中断操作完成后；内核调度程序中；系统调用返回时，（另外的内核线程ksoftirqd**周期执行软中断）*。需要说明的是软中断的执行也处于中断上下文中，所以中断上下文对它的限制是和硬中断一样的。

***Tasklet\*** ***和bottom half\***

 Tasklet和bottom half都是建立在软中断之上的两种延迟机制，其中具体不同在于软中断是静态分配的，而且同类软中断可以并发地在几个CPU上运行；Tasklet可以动态分配，并且不同种类的Tasklets可以并发地在几个CPU上运行，但同类的tasklets 不可以；bottom half只能静态分配，实质上下半部分是一个不能与其它下半部分并发执行的高优先级tasklet，即使它们类型不同，而且在不同CPU上运行。

***系统调用\***

  系统调用是用户程序通过门机制来进入内核执行的内核例程，它运行在内核态，处于进程上下文中（进程上下文包括进程的堆栈等等环境），所以系统的调用代码可以对进程相关数据进行访问，可以执行调度程序，也可以睡眠。

### 内核任务之间并发关系

上述内核任务很多情况是可以交错执行的，所以很有可能产生竞争（都要访问同一个数据结构时，就产生了竞争）。下面分析这些内核任务之间有那些可能的并发行为。

可以抽象出，程序（用户态和内核态一样）并发执行的总原因无非是正在运行中得程序被其它程序*抢占*，所以我们必须看看内核任务之间的抢占关系：

中断处理程序可以抢占内核中的所有程序（当没有锁保护时），包括软中断，tasklet，bottom half和系统的调用，甚至也包括中断处理程序。也就是说中断处理程序可以和这些所有的内核任务并发执行，如果被抢占的程序和中断处理程序都要访问同一个资源，就产生了竞争。

软件中断可以抢占硬中断处理程序以外的内核程序，所以内核代码（比如，系统调用）中有数据和软中断共享，就有会有竞争。此外要注意的是，软中断即使是同种类型的也可以并发的运行在不同处理器上，所以它们之间共享数据都会产生竞争。（如果在用一个处理器上软中断是不能相互抢占的）。

同类的tasklet不可能同时运行，所以对于同类tasklet不会产生并发；但两个不同种类的tasklet有可已在不同处理器上并发运行，如果之间有数据共享就会产生竞争（同类的tasklet在同一个处理器上运行的tasklet不发生相互抢占的情况）。

Bottom half 无论是否是同类的，即使在不同处理器上也都不能并发执行，它是绝对串行化的，所以它们之间永远不能产生竞争。

*注意：tasklet**和bottom half**是建立在软中断之上的，所以它们也都遵从软中断的调度规则——都可以打断进程上下问中的内核代码（系统调用），都可被硬中断打断——这些都可能产生并发。*

系统调用这种内核代码可能和各种内核代码并发，除了上面提到的中断（软，硬）抢占它产生并发外，它是有可能自发性地主动睡眠（比如在一些阻塞性的操作中），放弃处理器，重新调度其它任务，所以系统调用中并发情况更普遍，尤其当用户空间需要和内核空间共同操作全局数据时，一定要注意保护。

 

## 内核同步方法

为了避免并发，防止竞争。内核提供了一组同步方法来提供对共享数据的保护。 我们的重点不是介绍这些方法的详细用法，而是强调为什么使用这些方法和它们之间的差别。

Linux使用的同步机制可以说从2.0到2.6以来不断发展完善。从最初的原子操作，到后来的信号量，从大内核锁到今天的自旋锁。这些同步机制的发展伴随Linux从单处理器到对称多处理器的过度；伴随着从非抢占内核到抢占内核的过度。锁机制越来越有效，也越来越复杂。

目前来说内核中原子操作多用来做计数使用，其它情况最常用的是两重锁以及它们的变种，一个是自旋锁，另一个是信号量。我们下来就着重介绍一下这两中锁机制。

**自旋锁**

自旋锁最多只能被一个可执行线程持有，如果一个执行线程试图请求一个已被争用（已经被持有）的自旋锁，那么这个线程就会一直进行忙循环——旋转——等待锁重新可用。要是锁未被争用，请求它的执行线程便能立刻得到它并且继续进行。自旋锁可以在任何时刻防止多于一个的执行线程同时进入临界区。

事实上，自旋锁的初衷就是：在短期间内进行轻量级的锁定。一个被争用的自旋锁使得请求它的线程在等待锁重新可用期间进行自旋（特别浪费处理器时间），所以自旋锁不应该被持有时间过长。如果需要长时间锁定,最好使用信号量。

自旋锁的基本形式如下：

spin_lock(&mr_lock);

/*临界区*/

spin_unlock(&mr_lock);

因为自旋锁在同一时刻只能被最多一个执行线程持有，所以一个时刻只有一个线程允许存在于临界区中。这点很好的满足了对称多处理机器需要的锁定服务。在单处理器上，自旋锁仅仅当作一个设置内核抢占的开关。如果内核抢占也不存在，那么自旋锁会在编译时被完全剔除出内核。

自旋锁在内核中有许多变种，如对bottom half 而言，可以使用spin_lock_bh()用来获得特定锁并且关闭半底执行。相反的操作由spin_unlock_bh()来执行；如果临界区的访问逻辑可以被清晰的分为读和写这种模式，那么可以使用读者/写者自旋锁，调用形式为：

*读者的代码路径：*

*read_lock**(**&mr_rwlock);*

*/***只读临界区\*/*

*read_unlock**(**&mr_rwlock);*

*写者的代码路径：*

*write_lock**(**&mr_rwlock);*

*/***读写临界区\*/*

*write_unlock**(**&mr_rwlock);*

   简单的说，自旋锁在内核中主要用来防止多处理器中并发访问临界区，防止内核抢占造成的竞争。另外自旋锁不允许任务睡眠（持有自旋锁的任务睡眠会造成自死锁），它能够在中断上下文中使用。

***信号量\***

Linux中的信号量是一种睡眠锁。如果有一个任务试图获得一个已被持有的信号量时，信号量会将其推入等待队列，然后让其睡眠。这时处理器获得自由去执行其它代码。当持有信号量的进程将信号量释放后，在等待队列中的一个任务将被唤醒，从而便可以获得这个信号量。

信号量的睡眠特性，使得信号量适用于锁会被长时间持有的情况；只能在进程上下文中使用，因为中断上下文中是不能被调度的；另外当代码持有信号量时，不可以再持有自旋锁。

信号量基本使用形式为：

static DECLARE_MUTEX(mr_sem);//声明互斥信号量

…

if(down_interruptible(&mr_sem))

/*可被中断的睡眠，当信号来到，睡眠的任务被唤醒 */

/*临界区…*/

up(&mr_sem);

同自旋锁一样，信号量在内核中也有许多变种，比如读者－写者信号量等，这里不再做介绍了。

 

 

***信号量和自旋锁区别\***

虽然听起来两者之间使用条件复杂，其实在实际使用中信号量和自旋锁并不易混淆。注意以下原则。

如果代码需要睡眠——这往往是发生在和用户空间同步时——使用信号量是唯一的选择。由于不受睡眠的限制，使用信号量通常来说更加简单一些。如果需要在自旋锁和信号量中作选择，应该取决于锁被持有的时间长短。理想情况是所有的锁都应该尽可能短的被持有，但是如果锁的持有时间较长的话，使用信号量是更好的选择。另外，信号量不同于自旋锁，它不会关闭内核抢占，所以持有自旋锁的代码可以被抢占。这意味者信号量不会对影响调度反应时间带来负面影响。

 

## 实际应用举例

这些锁机制在内核中使用很频繁。我们举个简单例子来帮助理解它们的实质。

Netfilter是Linux 2.4.x内核中最流行的防火墙构建平台，我们这里不深入剖析Netfilter-iptables的组织结构，而是抽取一小部分代码来看看其中的锁定机制是如果工作的。iptables是专门针对2.4.x内核的Netfilter制作的用户空间配置工具，通过socket接口对Netfilter进行操作，创建socket的方式如下：

socket(TC_AF, SOCK_RAW, IPPROTO_RAW)

其中TC_AF就是AF_INET。用户空间程序可以通过创建一个"原始IP套接字"获得访问Netfilter的句柄，然后通过getsockopt()和setsockopt()系统调用来读取、更改Netfilter设置。请看参考资料 3

get/setsockopt()系统调用最终是依靠nf_sockopt()函数来进行用户空间对内核空间的数据进行操作的。这个函数属于系统调用，处于进程上下文。由于它有可能睡眠，造成内部数据被其它内核任务并发访问，从而引起不一致状态，所以这里需要使用锁——信号量——来防止竞争。（见netfilter.c）请看参考资料 4

static int nf_sockopt(struct sock *sk, int pf, int val,

​          char *opt, int *len, int get)

{

​    struct list_head *i;

​    struct nf_sockopt_ops *ops;

​    int ret;

 

​    if (**down_interruptible****(&nf_sockopt_mutex)** != 0)

​       return -EINTR;

 

​    for (i = nf_sockopts.next; i != &nf_sockopts; i = i->next) {

​       ops = (struct nf_sockopt_ops *)i;

​       if (ops->pf == pf) {

​           if (get) {

​              if (val >= ops->get_optmin

​                && val < ops->get_optmax) {

​                  ops->use++;

​                  **up(****&nf_sockopt_mutex);**

​                  ret = ops->get(sk, val, opt, len);

​                  goto out;

​              }

​           } else {

​              if (val >= ops->set_optmin

​                && val < ops->set_optmax) {

​                  ops->use++;

​                  **up(****&nf_sockopt_mutex);**

​                  ret = ops->set(sk, val, opt, *len);

​                  goto out;

​              }

​           }

​       }

​    }

​    up(&nf_sockopt_mutex);

​    return -ENOPROTOOPT;

   

 out:

​    **down(****&nf_sockopt_mutex);**

​    ops->use--;

​    if (ops->cleanup_task)

​       wake_up_process(ops->cleanup_task);

​    **up(****&nf_sockopt_mutex);**

​    return ret;

}

代码中为了防止ops->use++操作时产生不一致状态，使用了互斥信号量（static DECLARE_MUTEX(nf_sockopt_mutex)）来保护临界区。只有再down后才操作user数据，操作结束up信号量，解除锁定。

另外如果要给钩子点(hook)上注册操作，需要使用nf_register_hook（）函数，相反注销使用nf_unregister_hook（）函数。由于注册和注销主要任务是为给定的内核数据链表中加入或删除数据。链表在此处是一个共享结构，对它的访问路径就是一个典型的临界区，所以这里将使用轻量级的锁——自旋锁。又由于网络协议在内核中是在软中断中被处理的（老版本在bottom half中），所以这个链表数据是被注册（或注销）函数和软中断共享的。为了防止软中断抢占执行，访问这个链表必须使用关闭软中断的自旋锁。(static spinlock_t nf_hook_lock = SPIN_LOCK_UNLOCKED)

int nf_register_hook(struct nf_hook_ops *reg)

{

​    struct list_head *i;

 

​    spin_lock_bh(&nf_hook_lock);

​    list_for_each(i, &nf_hooks[reg->pf][reg->hooknum]) {

​       if (reg->priority < ((struct nf_hook_ops *)i)->priority)

​           break;

​    }

​    list_add_rcu(&reg->list, i->prev);

​    **spin_unlock_bh****(****&nf_hook_lock);**

 

​    synchronize_net();

​    return 0;

}

 

void nf_unregister_hook(struct nf_hook_ops *reg)

{

​    **spin_lock_bh****(****&nf_hook_lock);**

​    list_del_rcu(&reg->list);

​    **spin_unlock_bh****(****&nf_hook_lock);**

 

​    synchronize_net();

}

这个文件中还有种锁——读者－写者自旋锁（static rwlock_t queue_handler_lock = RW_LOCK_UNLOCKED）这个锁被用来在注册和注销协议对应的处理函数时保护协议队列不被并发访问，由于这种协议队列在检索时（读时）可以并发，而在写时只能有一个写者存在，所以利用读者－写者自旋锁时最优选择。

int nf_register_queue_handler(int pf, nf_queue_outfn_t outfn, void *data)

{   

​    int ret;

 

​    **write_lock_bh****(****&queue_handler_lock);**

​    if (queue_handler[pf].outfn)

​       ret = -EBUSY;

​    else {

​       queue_handler[pf].outfn = outfn;

​       queue_handler[pf].data = data;

​       ret = 0;

​    }

​    write_unlock_bh(&queue_handler_lock);

 

​    return ret;

}

 

**总结** 要用好Linux锁定机制必须深刻理解内核调度原理，要能清楚区分并发原因。另外Linux内核中锁机制的使用非常普遍，理解和使用锁机制是代码能够可靠运行的关键问题之一，并发产生的错误不可再现，调试困难，往往给开发带来很大麻烦，所以使用锁来保护临界区尤为重要。

即使开发环境不存在多处理器，不要求内核抢占，也最好不要放弃使用锁机制，因为使用恰当的锁机制可以方便将开发的代码向其它环境移植。另外要记住，不要指望在代码编写完毕后，再加入锁来保护资源——这样做是非常困难和愚蠢的，一定要在设计初期就要考虑竞争问题，设计锁来保护临界区。

 

 

参考资料

1 Linux Device Driver, O’Reilly

2 Robert Love, Linux Kernel Development，Sams Publishing，2003

3 Paul Russell Linux netfilter Hacking HOWTO v1.2，2002

4 Paul Russell iptables源码v1.2.1a ，2002

 

 