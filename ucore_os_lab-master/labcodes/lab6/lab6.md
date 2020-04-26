## Lab6实验报告

钱姿 2015010207 计65

### 【练习0】填写已有实验

将之前代码移植到相应位置，并且根据注释进一步改进

* 修改proc.c

根据注释，我们需要在proc_struct中添加一些变量

```C++
 struct run_queue *rq;                       // 运行队列
 list_entry_t run_link;                      // 该进程的调度链表结构
int time_slice;                             //该进程剩余的时间片
skew_heap_entry_t lab6_run_pool;            // 该进程在优先队列中的节点
uint32_t lab6_stride;                       // 该进程的调度优先级
 uint32_t lab6_priority;                     // 该进程的调度步进值
```

因此需要对他们进行初始化：

```C++
 proc->rq = NULL;
list_init(&(proc->run_link));
proc->time_slice = 0;
proc->lab6_run_pool.left=proc->lab6_run_pool.right=proc->lab6_run_pool.parent=NULL;
proc->lab6_stride = 0;
proc->lab6_priority = 0;
```

* 修改trap.c

在每个时钟中断的时候需要给调度器通知，以更新各种相关的数据。

因此修改代码如下

```C++
 ticks++;
        if (ticks % TICK_NUM == 0) {
        }
        sched_class_proc_tick(current);
```

### 【练习1】使用Round Robin调度算法

执行make grade后得到163/170，不能通过priority中的大部分测试

执行部分的摘录如下

>priority:                (11.5s)
>  -check result:                             WRONG
>   -e !! error: missing 'sched class: stride_scheduler'
>   !! error: missing 'stride sched correct result: 1 2 3 4 5'
>
>  -check output:                             OK
>Total Score: 163/170

* #### 请理解并分析`sched_class`中各个函数指针的用法，并结合`Round Robin `调度算法描`ucore`的调度执行过程

>  在schedule/sched.h中，具体结构如下所示

```C++
  // 调度器的名字
    const char *name;
    // 初始化运行队列
    void (*init)(struct run_queue *rq);
    // 将进程p插入到队列rp中
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    // 将进程p从队列rq中删除
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    // 返回运行队列中下一个可执行的进程
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    //timetick处理函数
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
```

**调度器初始化void (*init)(struct run_queue *rq);**

* 在内核初始化的时候调用这个函数，负责选择调度器并且对选择的调度器进行初始化

将进程加入调度其 void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);

* 当进程被唤醒后，需要调用这个函数将其加入到调度其中

进程调度

调度函数`schedule`变化如下：

- 在切换进程之前调用`sched_class_enqueue`将当前进程加入到RR的调度链表中；
- 调用`sched_class_pick_next`获取RR算法选取的下一个进程；
- 调用`sched_class_dequeue`将即将运行的进程从RR算法调度链表中删除。

调度器时钟更新  void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);

当时间片用完，调用这个函数更新调度器中的时钟

>  Round Robin调度算法

`RR`调度算法的调度思想是让所有`runnable`态的进程分时轮流使用CPU时间。RR调度器维护当前`runnable`进程的有序运行队列。当前进程的时间片用完之后，调度器将当前进程放置到运行队列的尾部，再从其头部取出进程进行调度。

* 使用了sched_class来调用RR算法

  `RR`算法维护了一个就绪进程的队列。每次进行调度的时候，把当前进程（如果还是处于就绪）加入到队列的末尾，然后取出该队列头部的进程（同时从队列中删除），选择其调度执行。如果队列中没有可调度的进程，就选取`idleproc`运行。`wakeup_proc`函数中，也直接将要唤醒的那个进程加入就绪队列的末尾。

* #### 请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

* 在运行队列run_queue中增加多个队列，比如可以用数组实现
* 增加进程队列号管理优先级别
* 修改sched_class中的部分功能
  * enqueue:根据优先级别判断应该加入那个队列
* deuqeue
  *  从优先级高的队列开始查找，如果为空就往下查找优先级低一级的进程。根据`proc`中保存的信息找到对应的队列进行删除

### 【练习2】实现Stride Scheduling调度算法

首先需要换掉RR调度器的实现，即用`default_sched_stride_c`覆盖`default_sched.c`。然后根据此文件和后续文档对`Stride`度器的相关描述，完成`Stride`调度算法的实现。

#### 调度算法分析

该算法用一个小顶堆选择即将调入CPU执行的进程。初始化时，将堆设为空，进程数为0。出现新建进程时，将进程加入堆，并设置时间片和指向堆的指针，增加进程计数。当一个进程执行完毕后将其从堆中移除并减少进程计数。当需要调度时，从堆顶取出当前`stride`值最小的进程调入`CPU`执行，并将其`stride`增加`pass`大小，如果堆顶为空，则返回空指针。产生时钟中断的处理与其他调度算法相同，减少当前进程的执行时间，若剩余时间为0则进行调度。

需要编写的函数为：

```C++
//初始化
stride_init();
//入队
stride_enqueue();
//出队
stride_dequeue();
//下一个可执行进程
stride_pick_next();
//更新调度器时钟
stride_proc_tick();
```

* 设置` BIG_STRIDE`为`0x7FFFFFFF`

```
#define BIG_STRIDE   0x7FFFFFFF /* you should give a value, and is ??? */
```

* stride_init();

根据注释，初始化以下变量

```C++
//可调度的列表 
list_init(&(rq->run_list));
//正在运行的进程
     rq->lab6_run_pool = NULL;
//进程的数量
    rq->proc_num = 0;
```

* stride_enqueue();

  将进程插入到优先队列中

  更新进程剩余的时间片

  设置进程的队列指针

  增加进程的计数值

```C++
 rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &(proc->lab6_run_pool),
               proc_stride_comp_f);
     // Clamp time_slice to be valid value
     if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
         proc->time_slice = rq->max_time_slice;
     }
     proc->rq = rq;
     rq->proc_num += 1;
```

* stride_dequeue();

  将进程从优先队列中删除

  将进程计数-1

  ```C++
   rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &(proc->lab6_run_pool),
                 proc_stride_comp_f);
        rq->proc_num -= 1; 
  ```

* stride_pick_next();

扫描整个运行队列，返回其中`stride`值最小的对应进程。更新对应进程的`stride`值，即`pass = BIG_STRIDE / P->priority; P->stride += pass`。

​	如果队列为空，返回空指针；

​	从优先队列中获得一个进程（就是指针所指的进程控制块）；

​	更新`stride`值。

```C++
if (rq->lab6_run_pool == NULL) return NULL;
     struct proc_struct* min_proc = le2proc(rq->lab6_run_pool, lab6_run_pool);
     if (min_proc->lab6_priority == 0) {
          min_proc->lab6_stride += BIG_STRIDE;
     } else if (min_proc->lab6_priority > BIG_STRIDE) {
          min_proc->lab6_stride += 1;
     } else {
          min_proc->lab6_stride += BIG_STRIDE / min_proc->lab6_priority;
     }
     return min_proc;
```

* stride_proc_tick

  和RR调度算法的实现一致

  ```C++
    if(proc->time_slice > 0) proc->time_slice --;
      if(proc->time_slice == 0) proc->need_resched = 1;
  ```

  

### 和参考答案的区别

完全按照注释来写代码，由于已经定义了USE_SKEW_HEAP=1，因此没有考虑当其为0的情况，部分函数的实现和参考答案有区别

### 实验中涉及的知识点

* 进程调度实现
* RR算法
* Stride算法

### 实验中没有涉及的知识点

* FCFS算法
* 短进程优先算法