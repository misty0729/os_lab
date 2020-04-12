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

### 【练习2】为新创建的内核线程分配资源

kernel_thread函数采用了局部变量`tf`来放置保存内核线程的临时中断帧，并把中断帧的指针传递给do_fork函数，而`do_fork`函数会调用`copy_thread`函数来在新创建的进程内核栈上专门给进程的中断帧分配一块空间。

根据注释实现了以下代码：

```C++
// 分配进程控制块（此时PCB还没有初始化）
proc = alloc_proc();
if (proc == NULL) {
  goto fork_out;
}
// 为PCB指定父进程
proc->parent = current;
// 分配两个页的内核栈空间给这个新的进程控制块
if (setup_kstack(proc) != 0) {
  goto bad_fork_cleanup_proc;
}
// 拷贝mm_struct，建立新进程的地址映射关系
if (copy_mm(clone_flags, proc) != 0) {
  goto bad_fork_cleanup_kstack;
}
// 拷贝父进程的trapframe，并为子进程设置返回值为0
copy_thread(proc, stack, tf);
// 中断可能由时钟产生，会使得调度器工作，为了避免产生错误，需要屏蔽中断
bool intr_flag;
local_intr_save(intr_flag);
{
  // 建立新的哈希链表
    proc->pid = get_pid();
    hash_proc(proc);
    list_add(&proc_list, &(proc->list_link));
    nr_process ++;
}
local_intr_restore(intr_flag);
// 唤醒进程，转为PROC_RUNNABLE状态
wakeup_proc(proc);
// 父进程应该返回子进程的pid
ret = proc->pid;
```

* 清说明ucore是否做到给每个新fork的县城一个唯一的id？

  ucore能做到给每个新fork线程一个唯一的id。通过函数get_pid来实现分配唯一的pid

  将last_pid设置为1，next_pid设置位MAX_PID,这两个变量表示进程序列号的区间范围

  遍历进程列表，执行以下操作：

  - 如果pid == last_pid，那么last_pid++
  - pid在last_pid和next之间，那么将next = pid
  - 如果last_pid++后比next大，那么将next = MAX_PID，last_pid不变，继续从头开始遍历

### 【练习3】理解proc_run函数和它调用的函数如何完成进程切换的

proc_run的执行过程如下：

* 保存IF位并禁止中断
* 将current指针指向将要执行的过程
* 更新TSS中的栈顶指针
* 加载新的页表
* 调用switch_to进行上下文切换
* 当执行proc_run的进程恢复执行之后，需要恢复IF位

```C++
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {	// 判断需要运行的线程是否已经运行着了
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag); // 关闭中断
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);	//设置TSS
            lcr3(next->cr3);	// 修改当前的cr3寄存器成需要运行线程（进程）的页目录表
            switch_to(&(prev->context), &(next->context));	//切换到新的线程
        }
        local_intr_restore(intr_flag);
    }
}
```

首先需要关闭中断，以免在进程切换过程中产生新的中断发生新的切换。随后进入switch_to函数执行上下文切换

进入`switch_to`之后，`esp`指向的是压入的返回地址，`esp+4`为第一个参数（`from`），`esp+8`为第二个参数（`to`）。然后把各个寄存器的值储存到`from`指向的地址处，即当前`proc`的`context`中，再将`to`指向的地址中保存的各寄存器的值恢复。之后`pushl 0(%eax)，eax`指向的位置存储的是`to`的`eip（eip`是`context`中的第一个变量），栈顶`（esp）`的位置就是要切换的进程的`eip`。`ret`指令会`pop`出栈顶的值并将这个值设置为当前的`eip`，于是就完成了进程的切换。

* 在本实验执行过程中，创建且运行了几个内核线程

  运行了一个内核进程，2个内核线程：idle init

* 语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag);`在这里有何作用?请说明理由

  在执行省略号部分的代码时关闭中断，执行完毕后再打开中断。作用是避免产生其他的陷入，例如产生另一个进程切换任务，这样会产生错误，因为省略号处代码的执行需要保证原子性。

### 与参考答案的区别

按照注释实现，无明显区别

### 涉及到的重点知识

* 进程控制块的结构

  如果要让内核线程运行，我们首先要创建内核线程对应的进程控制块，还需把这些进程控制块通过链表连在一起，便于随时进行插入，删除和查找操作等进程管理事务。

* 进程切换的过程

  实验四中实现了FIFO调度器，核心是schedule函数，执行逻辑是，首先设置当前线程的need_resched=0，在队列中找到下一个处于就绪的线程||进程，然后调用proc_run函数，进行切换。

### 未涉及的知识点

* 进程在不同状态之间转换时的操作
* 父子进程之间的关系