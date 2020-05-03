## LAB7实验报告

计65 钱姿 2015010207

### 【练习0】填写已有实验

将实验1/2/3/4/5/6中的代码移植到实验7中，并且根据注释修改trap.c

修改对时间中断的处理，修改函数为如下所示

```C++
run_timer_list();
```

### 【练习1】 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

>请在实验报告中给出内核级信号量的设计描述，并说明其大致执行流程

lab7在sem.c和sem.h中实现了信号量

信号量是一种同步互斥机制的软件实现。

* 数据结构

在sem.h中定义了一个基本记录信号量的数据结构，代码如下

```C++
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```

其中value是用于计数，wait_queue是等待进程队列，当value<0的时候，进程需要加入wait_queue等待被唤醒

* 功能函数

```C++
void sem_init(semaphore_t *sem, int value);
void up(semaphore_t *sem);
void down(semaphore_t *sem);
bool try_down(semaphore_t *sem);
```

定义了以上四个函数，其中sem_init初始化函数

up函数：调用了__up函数，实现v操作

down函数：调用了__down函数，实现了P操作

__up函数：

```C++
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    //为了保证操作的原子性不被打断，进入函数之后关闭中断，在退出函数之前再次打开中断。
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        // 如果在信号量的等待队列中并没有进程正在等待，那么对信号量的值+1，表示资源的释放
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            //如果有等待的进程，那么此时资源被释放，可以用来唤醒下一个进程
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
```

__down函数：

```C++
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
      //为了保证操作的原子性不被打断，进入函数之后关闭中断，在退出函数之前再次打开中断。
    bool intr_flag;
    local_intr_save(intr_flag);
    //在使用资源之前，需要检查资源是否剩余
    if (sem->value > 0) {
        //如果此时资源足够，那么对值减1，表示进行占用
        sem->value --;
        //打开中断
        local_intr_restore(intr_flag);
        return 0;
    }
    //如果此时资源不剩余，那么需要把该进程放入等待队列中，等待被唤醒
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);
	
    //重新调度进程，让其他进程执行
    schedule();

    //阻塞完毕，该进程被重新唤醒，开始执行
    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```

* 执行流程

所有哲学家共享临界区信号量mutex，这个信号量用于保证哲学家线程在操作`state_sema`数组时不产生冲突。因此在take_forks和put_forks函数里面都需要加锁。

使用信号量数组s,其中s[i]表示哲学家的状态：HUNGRY，EATING，THINKING，当尝试开始进餐的时候，哲学家会试图获取两个叉子，如果满足条件，则进行进餐；如果不能够取得，则使用信号量s[i]对其进行阻塞。而只有当周围的正在进餐的哲学家进餐完毕之后，才会去检查周围的哲学家是否能够进餐，如果可以进餐则解除信号量s[i]引起的阻塞，从而进行进餐。

>请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

由于操作系统需要保持`PV`操作的原子性，必须使用特权指令禁用中断。由于CLI和STI指令是特权指令，故在用户态中不能关闭中断来保证若干条操作的原子性。

由于用户进程的切换是由操作系统控制的，如果某用户进程在执行`_up`和`_down`操作的过程中被操作系统打断，改为由其他进程执行，就有可能出现错误。因此，关键是如何在用户态保证`_up`和`_down`操作的原子性。

可以使用`Peterson`算法和`Dekker`算法就是用来解决。

区别：用户态和内核态实现信号量的区别主要在于控制临界区访问的方法的不同

### 【练习2】完成内核级条件变量和基于内核级条件变量的哲学家就餐问题（需要编码）

>请在实验报告中给出内核级条件变量的设计描述，并说明其大致执行流程。

* 首先观察monitor.h中的数据结构

条件变量

```C++
typedef struct condvar{
    //用来管理这个条件变量上的等待进程
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    //在这个条件变量上等待的进程的数量
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;
```

管程管理器

```C++
typedef struct monitor{
    //控制区只能有一个进程进入
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    //条件变量
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```

* 实现函数

cond_signal：释放唤醒一个等待在该条件上的进程

首先进程判断`cvp.count`:

- 如果不大于0，则表示当前没有执行`cond_wait`而睡眠的进程，函数退出；

- 如果大于0，这表示当前有执行`cond_wait`而睡眠的进程A，因此需要唤醒等待在`cvp.sem`上睡眠的进程A。

  由于只允许一个进程在管程中执行，所以一旦进程B唤醒了别人（进程A），那么自己就需要睡眠。故让`monitor.next_count`加一，且让自己（进程B）睡在信号量`monitor.next`上。如果睡醒了，这让`monitor.next_count`减一。

```C++
monitor_t* mtp = cvp->owner;
    if (cvp-> count > 0){
        mtp->next_count++;
        up(&(cvp->sem));
        down(&(mtp->next));
        mtp->next_count --;
    }
```

cond_wait函数：等待某个条件的睡眠进程数量

* 首先该条件上的睡眠进程数量增加1
* 如果`next_count`大于0，说明有大于等于1个进程执行`cond_signal`且睡着了，所以唤醒`mtp`的`next`信号量，然后进程睡在`cv.sem`上，如果睡醒了，则让`cv.count`减一，表示等待此条件的睡眠进程个数少了一个，
* 如果`next_count`小于等于0，表示目前没有进程执行`cond_signal`函数且睡着了，那需要唤醒的是由于互斥条件限制而无法进入管程的进程，所以要唤醒睡在`monitor.mutex`上的进程。然后进程A睡在`cv.sem`上，如果睡醒了，则让`cv.count`减一，表示等待此条件的睡眠进程个数少了一个，可继续执行了。

```C++
   cvp->count ++;
    // 需要将自己换出，让出管程
    // 但是需要特别判断是否存在next优先进程。
    monitor_t* mtp = cvp->owner;
    if(mtp->next_count > 0){
       up(&(mtp->next));
    }else{
       up(&(mtp->mutex));
    }
    // 自己阻塞
    down(&(cvp->sem));
    cvp->count --;
```

phi_take_forks_condvar函数：哲学家获取叉子，能够获取就进行进餐，否则就在这个条件上睡眠

```C++
 state_condvar[i] = HUNGRY;		//记录下哲学家i是否饥饿，即处于等待状态拿叉子
     // try to get fork
     phi_test_condvar(i);
     while (state_condvar[i] != EATING) {
       cond_wait(&(mtp->cv[i]));	//如果得不到叉子就睡眠
     }
```

phi_put_forks_condvar函数:哲学家放下叉子，对左右哲学家试探能否进行进餐，能的话就唤醒

```C++
 state_condvar[i]=THINKING;	//记录进餐结束的状态
     // test left and right neighbors
     phi_test_condvar(LEFT);	//看一下左边哲学家现在是否能进餐
     phi_test_condvar(RIGHT);	//看一下右边哲学家现在是否能进餐
```

执行流程

初始化设置管程中有N个条件变量，每次进入管程获取管程的锁，退出的时候释放管程的锁（如果有处在阻塞的进程，则唤醒），和上课所说的简单的release略有不同。成功进入管程后调用phi_take_forks_condvar和phi_put_forks_condvar进行获取叉子、释放叉子。

> 请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

相同点:用户态条件变量机制与内核态实现基本一致，即由一个变量的操作来判断当前资源的使用情况，需要系统原子操作的支持，通过系统调用在用户态实现管程的处理函数。

不同点：内核态信号量可以直接调用内核的服务，而用户态信号量需要通过系统调用接口调用内核态的服务，涉及到栈切换等等。内核态信号量存储在内核态的内核栈上，而用户态信号量存储在内核中一段共享内存中。

用户态进程、线程的信号量机制依旧需要内核态的信号量机制支持，因此在内核部分沿用上面给出的内核态信号量实现。

`cond_signal`和`cond_wait`导致的进程阻塞与唤醒都可以由信号量的相关接口进行完成，因此我们可以设计用户态的管程机制，由语言内部维护或是用户手动维护这两个必要函数的运作，通过系统调用完成对信号量的`PV`操作。

我们可以参考`posix`信号量机制：`posix`为可移植的操作系统接口标准。信号量的概念在`system v `和`posix `中都有，但是它们两者的具体作用是有区别的。`system v`版本的信号量用于实现进程间的通信，而`posix`版本的信号量主要用于实现线程之间的通信，两者的主要区别在于信号量和共享内存。

我们可以模仿`posix`的信号量，提供接口：`sem_open,sem_close,sem_unlink`来支持.

> 请在实验报告中回答：能否不用基于信号量机制来完成条件变量？如果不能，请给出理由，如果能，请给出设计说明和具体实现。

可以，但是需要保证等待队列的操作是互斥的，需要在`cond_signal`,` cond_wait`函数前后需要加一个互斥锁。

### 与参考答案的区别

与注释实现基本一致

### 实验中涉及的重要知识点

* 信号量

  信号量是一种同步互斥机制的实现，普遍存在于现在的各种操作系统内核里。

* 管程和条件变量

  管程是为了将对共享资源的所有访问及其所需要的同步操作集中并封装起来。

* 哲学家就餐问题

### 未涉及知识点

* 生产者-消费者问题