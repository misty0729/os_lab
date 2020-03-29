## Lab3实验报告

计65  钱姿

### 【练习0】填写已有代码

移植lab1、lab2相应的代码，移植完后运行make grade为10/45

### 【练习1】给未被映射的地址映射上物理页

完成`do_pgfault(mm/vmm.c)`函数，给未被映射的地址映射上物理页。设置访问权限的时候需要参考页面所在VMA的权限，同时需要注意映射物理页时需要操作内存控制结构所在指定的页表，而不是内核的页表

**设计思路**：

* 检测页表中是否有相应的表项，如果表项为空，则说明没有映射过，那么此时
  * 调用`get_pte`获取线性地址所对应的虚拟页的起始地址对应到的页表项
  * 如果查询到的PTE不为0，则表示对应的物理页可能在内存或者外存中，否则就表示对应的物理页尚未分配
* 为没有映射的虚拟页分配一个物理页，并确认分到的物理页不为空
  * 使用`pgdir_alloc_page`函数实现了内存分配功能，我们用这个函数来获取对应的物理页，设置映射关系。

```C++
//  //(1) try to find a pte, if pte's PT(Page Table) isn't existed, then create a PT.
if ((ptep = get_pte(mm->pgdir, addr, 1)) == NULL) {
        cprintf("get_pte in do_pgfault failed\n");
        return ret;
    }
   //(2) if the phy addr isn't exist, then alloc a page & map the phy addr with logical addr
    if (*ptep == 0) { 
        if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            cprintf("pgdir_alloc_page in do_pgfault failed\n");
            return ret;
        }
  }
```

* 请描述页目录项和页表项中组成部分对ucore实现页替换算法的潜在用处

  >页目录项和页表项的组成：两者组成相同，不同的是高20位的存储内容不同，页目录项存储的是二级页表的物理页帧号，页表项存储的是虚拟地址对应的页帧号
  >
  >* 页目录项为替换算法提供了被换出的页
  >* 页表村除了替换算法中被换入页的信息，替换后将其映射到物理地址
  >* 其中记录是否访问过、修改过的位可以提供替换算法换选的依据

* 如果ucore的缺页服务例程在执行过程中访问内存出现了页访问异常，请问硬件要做哪些事情

  >进入中断处理：CPU在执行完一个指令后开始读取中断信号，获取到中断index及相应的error code信息，接下来CPU会将相应的error code存储在trapframe中，并根据异常对应的IDT去寻找中断描述符
  >
  >缺页异常处理：发生缺页异常时，中断服务例程会将缺失的页调入到内存中
  >
  >完成中断处理

### 【练习2】补充完成基于FIFO的页面替换算法

设计思路:

FIFO算法：总是淘汰最先进入内存的页。

* 判断当前是否对交换机制进行了正确的初始化
* 将虚拟页对应的物理页从外存换入内存
* 给换入的物理页和虚拟页建立映射关系
* 将换入的物理页设置为允许被换出

根据练习1我们已经处理了pte=0的情况，如果pte不是0，那么我么需要从磁盘中调入相应的页面

首先调用swap_in函数根据mm，addr获取pte，然后根据pte中的内容去磁盘中调入该页面，存放在新分配的一个物理页面中。然后建立该地址到页面的映射。

根据注释完成以下的代码

```C++
if(swap_init_ok) {
            struct Page *page=NULL;
    //如果调入失败
            if ((ret = swap_in(mm, addr, &page)) != 0) {
                cprintf("swap_in in do_pgfault failed\n");
                return ret
            }    
            page_insert(mm->pgdir, page, addr, perm);
    //标记页面为swappable
            swap_map_swappable(mm, addr, page, 1);
            page->pra_vaddr = addr;
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
           return ret;
        }
```

由于FIFO基于双向链表实现，所以只需要将元素查到头节点之前

```C++
  list_add(head,entry);
```

选出一个victim换出,这里选择尾部节点

```C++
//(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
     list_entry_t *le = head->prev;
     assert(head!=le);
     struct Page *p = le2page(le, pra_page_link);
     list_del(le);
     assert(p !=NULL);
     //(2)  assign the value of *ptr_page to the addr of this page
      *ptr_page = p;
```

* 如果在ucore上实现extended clock页替换算法，请给出你的设计方案，现有的框架是否支持？

  >现有的框架可以支持
  >
  >extended clock：类似LRU，考虑页面是否被替换&修改
  >
  >* 利用PTE中记录的是否访问&修改
  >* 算法实现：我们考虑对FIFO算法进行修改，由于两种算法都是将所有可以换出的物理与诶安按照进入内存的顺序连城一个环状链表，因此初始化等函数不需要大幅度改动，需要改动的是替换物理页面的函数swap_out_victim，修改最近访问位和修改位
  >* 根据时钟算法即可找到合适的页面

* 需要被换出的页的特征

  每个页面都有两个标志位：修改位和使用位。换出页使用位为0，并且优先考虑替换修改位为0的页面

* 在ucore中如何判断具有特征的页

  PTE中包含了dirty和access，当页面被访问时，CPU中的MMU把访问位变成1，被修改时，修改位变成1

* 何时进行换入和换出操作

  当进程访问物理页没有存在内存缓存中而是存在磁盘的时候，需要进行换入操作

  当物理页的dirty从1改称0的时候进行换出操作。

### 与参考答案的区别

按照代码注释完成，有细微实现的区别，思想无区别。

### 实验中涉及的重点知识

* 虚拟内存管理

  虚拟内存单元不一定有实际的物理内存单元对应；虚拟内存单元和物理内存单元的地址不相等

* 页面置换算法
  * FIFO：该算法总是淘汰最先进入内存的页；存在belady现象
  * 时钟页替换算法：LRU近似实现，时钟替换算法把各个页面组织成环形链表结构，把指针指向最先进来的页面，PTE中记录该页面是否被访问
  * 改进的时钟替换算法：增加了一位是否修改位

### 未涉及的知识点

全局页面替换算法

* 工作集算法
* 缺页率算法



