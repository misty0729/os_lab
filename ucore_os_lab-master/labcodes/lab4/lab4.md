## Lab4实验报告

钱姿 计65 2015010207

### 【练习0】填写已有代码

将本实验依赖实验1/2/3的部分合并到了相应部分

### 【练习1】分配并初始化一个进程控制块

首先分析进程控制块中各个变量的含义

```C++
 *       enum proc_state state;                      // Process state
     *       int pid;                                    // Process ID
     *       int runs;                                   // the running times of Proces
     *       uintptr_t kstack;                           // Process kernel stack
     *       volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
     *       struct proc_struct *parent;                 // the parent process
     *       struct mm_struct *mm;                       // Process's memory management field
     *       struct context context;                     // Switch here to run process
     *       struct trapframe *tf;                       // Trap frame for current interrupt
     *       uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
     *       uint32_t flags;                             // Process flag
     *       char name[PROC_NAME_LEN + 1];               // Process name
```

主要变量如下：

* state：进程当前的状态
* mm：进程内存管理信息
* parent：用户进程的父进程
* context：进程的上下文，用于进程切换
* tf：中断帧的指针
* cr3：页表的基址
* kstack：每个线程都有一个内核栈

下面来分析初始化的数值

* proc_state表示了进程所处的状态，初始化的时候需要PROC_UNINIT状态
* pid = -1，表示进程pid尚未办好
* 内核线程没有用户空间，执行的只是内核的一小段代码，mm = NULL
* 在所有进程中，第一个进程没有父类，所以parent = NULL

完成代码如下：

```C++
	proc->state = PROC_UNINIT;
    proc->pid = -1;
    proc->runs = 0;
    proc->kstack = 0;
    proc->need_resched = 0;
    proc->parent = NULL;
    proc->mm = NULL;
    memset(&(proc->context), 0, sizeof(struct context));
    proc->tf = NULL;
    proc->cr3 = boot_cr3;
    proc->flags = 0;
    memset(proc->name, 0, PROC_NAME_LEN);
```

* 请说明proc_struct 中 struct context contect 和 struct trapframe *tf成员变量含义和在本实验中的作用是什么？

  * context的含义和作用

    进程的上下文，用于切换进程。

    context的定义如下

    ```c++
    struct context {
        uint32_t eip;	//指令寄存器
        uint32_t esp;	//堆栈指针寄存器
        uint32_t ebx;	//基址寄存器
        uint32_t ecx;	//计数器
        uint32_t edx;	//数据寄存器
        uint32_t esi;	//源地址指针寄存器
        uint32_t edi;	//目的地址指针寄存器
        uint32_t ebp;	//基址指针寄存器
    };
    ```

    利用context进行上下文切换的函数是switch_to

    ```void switch_to(struct context *from, struct context *to);```

  * trapframe含义和作用

    中断帧的指针，总是指向内核栈的某个位置：当进程从用户空间跳到内核空间时，中断帧记录了进程在被中断前的状态。当内核需要跳回用户空间时，需要调整中断帧以恢复让进程继续执行的各寄存器值。