**一、设计目的和内容要求**

 1. 设计目的
	Linux 提供的模块机制能动态扩充 Linux功能而无须重新编译内核，已经广泛应用在Linux内核的许多功能的实现中。在本实验中将学习模块的基本概念、原理及实现技术，然后利用内核模块编程访问进程的基本信息，加深对进程概念的理解，掌握基本的模块编程技术。
 2. 内容要求
	1. 设计一个模块，要求列出系统中所有内核线程的程序名、PID、进程状态、进程优先级、父进程的PID。
	2. 设计一个带参数的模块，其参数为某个进程的-PID号，模块的功能是列出该进程的家族信息，包括父进程、兄弟进程和子进程的程序名、PID号及进程状态。
	3. 请根据自身情况，进一步阅读分析程序中用到的相关内核函数的源码实现。
 3. 开发平台
	Ubuntu 24.04 华为ECS弹性云服务器


### 前置工作
```
$ mkdir os_1 #新建文件夹
$ cd os_1
$ mkdir tasks tasks_Addition
```


## 模块一（无参类型）
要求：列出系统中所有内核线程的程序名，PID，进程状态和进程优先级，父进程的PID。

```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/init_task.h>

static int all_task_init(void)
{
    struct task_struct *p;

    printk(KERN_INFO "Listing kernel threads:\n");
    printk(KERN_INFO "COMMAND\tPID\tSTATE\tPRIO\tPPID\n");

    for_each_process(p) {
        if (p->mm == NULL) {  // 内核线程检测
            // 安全获取父进程PID
            pid_t ppid = p->parent ? p->parent->pid : 0;

            printk(KERN_INFO "%-16s\t%-6d\t%-4ld\t%-4d\t%-6d\n",
                  p->comm,
                  p->pid,
                  p->__state,
                  p->prio,
                  ppid);
        }
    }
    return 0;
}

static void all_task_exit(void)
{
    printk(KERN_INFO "all_task module unloaded\n");
}

module_init(all_task_init);
module_exit(all_task_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("zxyhiatus");
MODULE_DESCRIPTION("Kernel thread lister module");
MODULE_VERSION("0.1");



```

## 对应的Makefile文件

```
obj-m := tasks.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
default:
        make -C $(KDIR) M=$(PWD) modules
clean:
        make -C $(KDIR) M=$(PWD) clean
~                                         
```

### 编译模块

```
$ make
```
![[file-20250313154221580.png]]
图一 编译无参文件


### 加载模块


```
$ insmod tasks.ko
$ modinfo tasks.ko #查看模块详细信息
```
![[file-20250313154438717.png]]
图二 加载模块并查看模块详细信息

### 查看结果

```
$ dmesg
```
![[file-20250313154558054.png]]
图三 模块执行结果

### 卸载模块
```
$ rmmod tasks.ko
$ dmesg  #查看是否卸载成功
```
![[file-20250313154806312.png]]
图四 模块卸载输出

## 检验结果
```
$ ps aux #列出所有线程/进程
```
![[file-20250321101604507.png]]
图五 检验结果
## 模块二（有参）
要求:
其参数为某个进程的-PID号，模块的功能是列出该进程的家族信息，包括父进程、兄弟进程和子进程的程序名、PID号及进程状态。

```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/init_task.h>
#include <linux/moduleparam.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("zxyhiatus");
MODULE_DESCRIPTION("Process Family Tree Viewer");
MODULE_VERSION("1.0");

static int pid = 1;  // 默认查看 init 进程
module_param(pid, int, 0644);

static int family_task_init(void)
{
    struct task_struct *target = NULL;
    struct task_struct *parent = NULL;
    int found = 0;

    printk(KERN_INFO "=== Process Family Tree for PID: %d ===\n", pid);

    // 查找目标进程
    for_each_process(target) {
        if (target->pid == pid) {
            found = 1;
            break;
        }
    }

    if (!found) {
        printk(KERN_WARNING "Process with PID %d not found!\n", pid);
        return -ESRCH;
    }

    // 打印目标进程信息
    printk(KERN_INFO "[Target Process]\n");
    printk(KERN_INFO "%-16s %-6s %-6s\n", "COMMAND", "PID", "STATE");
    printk(KERN_INFO "%-16s %-6d %-6ld\n",
           target->comm, target->pid, target->__state);

    // 处理父进程
    parent = target->parent;
    printk(KERN_INFO "\n[Parent Process]\n");
    if (parent) {
        printk(KERN_INFO "%-16s %-6d %-6ld\n",
               parent->comm, parent->pid, parent->__state);
    } else {
        printk(KERN_INFO "No parent (kernel thread)\n");
    }

    // 处理子进程
    printk(KERN_INFO "\n[Children Processes]\n");
    if (!list_empty(&target->children)) {
        struct task_struct *child;
        list_for_each_entry(child, &target->children, sibling) {
            printk(KERN_INFO "%-16s %-6d %-6ld\n",
                   child->comm, child->pid, child->__state);
        }
    } else {
        printk(KERN_INFO "No children\n");
    }

    // 处理兄弟进程
    printk(KERN_INFO "\n[Sibling Processes]\n");
    if (parent && !list_empty(&parent->children)) {
        struct task_struct *sibling;
        list_for_each_entry(sibling, &parent->children, sibling) {
            if (sibling->pid != pid) {
                printk(KERN_INFO "%-16s %-6d %-6ld\n",
                       sibling->comm, sibling->pid, sibling->__state);
            }
        }
    } else {
        printk(KERN_INFO "No siblings\n");
    }

    return 0;
}

static void family_task_exit(void)
{
    printk(KERN_INFO "tasks_Addition module unloaded\n");
}

module_init(family_task_init);
module_exit(family_task_exit);
               
```

### 编辑Makefile

```
obj-m := tasks_Addition.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
default:
        make -C $(KDIR) M=$(PWD) modules
clean:
        make -C $(KDIR) M=$(PWD) clean
```

### 编译模块
![[file-20250313155552569.png]]

### 加载模块&卸载模块
![[file-20250313155740527.png]]




Q：
1.如何判断出这个进程是内核级进程？
2.父子进程逻辑树



A：
1.通过tasks结构中的mm链表头，如果不为空，则为内核级进程
2.![[file-20250529215339002.png]]