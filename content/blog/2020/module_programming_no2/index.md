---
title: "内核模块编程之入门（二）—必备知识"
date: 2008-11-09T16:50:24+08:00
author: "helight0"
keywords: ["内核模块"]
categories : ["走进内核"]
banner : "img/blogimg/3.png"
summary : "模块编程属于内核编程，因此，除了对内核相关知识有所了解外，还需要了解与模块相关的知识。"
---

 1．应用程序与内核模块的比较 为了加深对内核模块的了解，表一给出应用程序与内核模块程序的比较。 表一 应用程序与内核模块程序的比较

|          | C语言应用程序 | 内核模块程序      |
| -------- | ------------- | ----------------- |
| 使用函数 | Libc库        | 内核函数          |
| 运行空间 | 用户空间      | 内核空间          |
| 运行权限 | 普通用户      | 超级用户          |
| 入口函数 | main()        | module_init()     |
| 出口函数 | exit()        | module_exit()     |
| 编译     | Gcc –c        | Makefile          |
| 连接     | Gcc           | insmod            |
| 运行     | 直接运行      | insmod            |
| 调试     | Gdb           | kdbug, kdb,kgdb等 |

​		从表一我们可以看出，内核模块程序不能调用libc库中的函数，它运行在内核空间，且只有超级用户可以对其运行。另外，模块程序必须通过module_init()和module-exit()函数来告诉内核“我来了”和“我走了”。

2．内核符号表（如果对以下第2~4点理解上有困难，可以越过）

​		如前所述，Linux内核是一个整体结构，像一个圆球，而模块是插入到内核中的插件。尽管内核不是一个可安装模块，但为了方便起见，Linux把内核也看作 一个“母”模块。那么模块与模块之间如何进行交互呢，一种常用的方法就是共享变量和函数。但并不是模块中的每个变量和函数都能被共享，内核只把各个模块中 主要的变量和函数放在一个特定的区段，这些变量和函数就统称为**符号**。到低哪些符号可以被共享？ Linux内核有自己的规定。对于内核这个特殊的母模块，在kernel/ksyms.c中定义了从中可以“移出”的符号，例如进程管理子系统可以“移出”的符号定义如下：

```shell
/* 进程管理 */

EXPORT_SYMBOL(do_mmap_pgoff);

EXPORT_SYMBOL(do_munmap);

EXPORT_SYMBOL(do_brk);

EXPORT_SYMBOL(exit_mm);

…

EXPORT_SYMBOL(schedule);

EXPORT_SYMBOL(jiffies);

EXPORT_SYMBOL(xtime);

…
```

​		你可能对这些变量和函数已经很熟悉。其中宏定义EXPORT_SYMBOL（）本身的含义是“移出符号”。为什么说是“移出”呢？因为这些符号本来是内核内部的符号，通过这个宏放在一个公开的地方，使得装入到内核中的其他模块可以引用它们。

​		实际上，仅仅知道这些符号的名字是不够的，还得知道它们在内核地址空间中的地址才有意义。因此，内核中定义了如下结构来描述模块的符号：

```c
struct module_symbol

{

unsigned long value; ／*符号在内核地址空间中的地址*/

const char *name; /*符号名*/

};
```

​	我们可以从/proc/ksyms文件中读取所有内核模块“移出”的符号，这所有符号就形成内核符号表，其格式如下：

```c
内存地址 符号名 ［所属模块］
```

​		在模块编程中，可以根据符号名从这个文件中检索出其对应的地址，然后直接访问该地址从而获得内核数据。第三列“所属模块”指符号所在的模块名，对于从内核这一母模块移出的符号，这一列为空。

​		模块加载后，2.4内核下可通过 /proc/ksyms、 2.6 内核下可通过/proc/kallsyms查看模块输出的内核符号

3．模块依赖

​		如前所述，内核符号表记录了所有模块可以访问的符号及相应的地址。当一个新的模块被装入内核后，它所申明的某些符号就会被登记到这个表中，而这些符号可能被其他模块所引用，这就引出了模块依赖这个问题。

​		一个模块A引用另一个模块B所移出的符号，我们就说模块B被模块A引用，或者说模块A依赖模块B。如果要链接模块A，必须先链接模块B。这种模块间相互依赖的关系就叫**模块依赖。**

4．模块引用计数器

​		为了确保模块安全地卸载，每个模块都有一个引用计数器。当执行模块所涉及的操作时就递增计数器，在操作结束时就递减这个计数器；另外，当模块B被模块A引用 时，模块B的引用计数就递增，引用结束，计数器递减。什么时候可以卸载这个模块？当然只有这个计数器值为0的时候，例如，当一个文件系统还被安装在系统上 时就不能将其卸载，当这个文件系统不再被使用时，引用计数器就为0，于是可以卸载。

5．模块编译

​		Linux 中最重要的软件开发工具是 GCC。GCC 是 GNU 的 C 和 C++ 编译器。但是，在大型的开发项目中，通常有几十到上百个的源文件，如果每次均手工键入 gcc 命令进行编译的话，则会非常不方便。因此，人们通常利用 make 工具来自动完成编译工作。利用这种自动编译可大大简化开发工作，避免不必要的重新编译。这些工作包括：如果仅修改了某几个源文件，则只重新编译这几个源文件；如果某个头文件被修改了，则重新编译所有包含该头文件的源文件。

​		编译工具make：

​		实际上，make 工具通过一个称为 Makefile 的文件来完成并自动维护编译工作。Makefile 需要按照某种语法进行编写，其中说明了如何编译各个源文件并连接生成可执行文件，并定义了源文件之间的依赖关系。下面给出**2.6 内核模块的Makefile模板（请参看Makefile的写法）** 

```c
# Makefile2.6 
obj-m += hellomod.o     # 产生hellomod 模块的目标文件 
CURRENT_PATH := $(shell pwd)  #模块所在的当前路径 
LINUX_KERNEL := $(shell uname -r)   #Linux内核源代码的当前版本 
LINUX_KERNEL_PATH := /usr/src/linux-headers-$(LINUX_KERNEL) #Linux内核源代码的绝对路径 
all: 
	make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules  #编译模块了 
clean: 
	make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) clean   #清理
```

注意： 在每个命令前（例如make命令前）要键入一个制表符（按TAB键产生） 有了Makefile,执行make命令，会自动形成相关的后缀为.o和.ko文件。 到此，模块编译好了，该把它插入到内核了： 如：$insmod hellomod.ko  当然，要以系统员的身份才能把模块插入。  成功插入后，可以通过dmesg命令查看，屏幕最后几行的输出就是你程序中输出的内容：Hello,World! from the kernel space…  当模块不再需要时，可以通过rmmod命令移去，例如 $rmmod hellomod