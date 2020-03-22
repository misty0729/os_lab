## Lab2实验报告

计65 钱姿 2015010207

### 【练习0】填写已有实验

完成`debug/kdebug.c`中函数的合并

完成kern/trap/trap.c中函数的合并

### 【练习1】实现first-fit连续物理内存分配算法

#### First-fit 连续物理内存分配算法原理：

将空闲内存块按照地址从小到大的方式连起来，从低地址向高地址查找，找到大小能满足要求的第一个空间。

#### 数据结构：

在ucore中使用双向链表,具体的结构代码在`libs/list.h`中，如下所示

```C
struct list_entry {
    // 双向指针
    struct list_entry *prev, *next;
};
```

具有list_init, list_add, list_add_before, list_del, list_next等操作函数

使用`kern/mm/memlayout.h`中的free_area结构体对空闲块进行管理

```c
/* free_area_t - maintains a doubly linked list to record free (unused) pages */
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```

#### 需要修改的函数为在default_pmm.c文件中的四个函数

`default_init`,`default_init_memmap`,`default_alloc_pages`,`default_free_pages`

```C
static void
default_init(void) {
    //创建了一个用来存储块的链表
    list_init(&free_list);
    //记录此时的块的数量为0，即链表为空
    nr_free = 0;
}
```

```c
//根据注释可以知道，这个函数是根据基址和页码来初始化空闲块
//调用的关系如下：`kern_init` --> `pmm_init` --> `page_init` --> `init_memmap` --> `pmm_manager` --> `init_memmap`
static void
default_init_memmap(struct Page *base, size_t n) {
    //如果页码小于0，报错
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;
    list_add(&free_list, &(base->page_link));
}
```

以上是对现有代码的分析，随后添加自己的代码补充算法

```c
//根据注释得知，该代码块的作用是查找第一个满足条件的空闲块
static struct Page *
default_alloc_pages(size_t n) {
    //边界检查
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    //查找可用空闲块
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        //如果找到了就返回
        if (p->property >= n) {
            page = p;
            break;
        }
    }
    //-------------------------主要改动---------------------------------------//
  if (page != NULL) {
        list_del(&(page->page_link));
      //如果找到的空闲块大于n
        if (page->property > n) {
            struct Page *p = page + n;
            //那么需要将剩下的空间重新设置优先级别
            p->property = page->property - n;
            SetPageProperty(p);
            list_add_after(&(page->page_link), &(p->page_link));
    }
      //删除当前页
        list_del(&(page->page_link));
      //更新空闲块数量
        nr_free -= n;
        ClearPageProperty(page);
    }
    return page;
}
```

```c
//将空闲块重新放入空闲链表中去，并且将小空闲块（如果物理地址连续）合并成大空闲块
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(!PageReserved(p) && !PageProperty(p));
        p->flags = 0;
        set_page_ref(p, 0);
    }
    //设置释放的长度和page 和优先级别
    base->property = n;
    SetPageProperty(base);
    //找到插入的链表位置，和他的前一项（用于合并）
       list_entry_t *le = list_next(&free_list);
    list_entry_t *prev = &free_list;
    while(le != &free_list) {
        p = le2page(le,page_link);
        if(base < p) break;
        prev = le;
        le = list_next(le);
    }
    //检查是否可以和后一项合并
    p = le2page(prev, page_link);
    if(prev != & free_list && p + p->property ==base) {
        p->property += base->property;
        ClearPageProperty(base);
    }
    else {
        list_add_after(prev,&(base->page_link));
        p = base;
    }
    //检查是否可以和后一项中的空间可并
    struct Page *nextp  = le2page(le, page_link);
    if(le != & free_list && p + p->property ==nextp) {
        p->property += nextp->property;
        ClearPageProperty(nextp);
        list_del(le);
    }
    nr_free += n;
}
```

#### 对算法的改进

* 替换数据结构：

  目前我们使用的是双向链表的结构，可以使用一些树状结构替代，这样子就能优化一些操作例如查找操作

### 【练习2】实现寻找虚拟地址对应的页表项

需要补全在文件`kern/mm/pmm.c`中的函数get_pte

根据注释实现代码

实现思路如下：

首先查询一级页表项，在其中查找二级页表项，如果二级页表不存在，那么要根据参数来帕段是否分配二级页表的内存空间，若部分配则返回空。

具体的代码如下

```c
#if 0
    pde_t *pdep = NULL;   // (1) find page directory entry
//create为0，直接返回Null
    if (0) {              // (2) check if entry is not present
                          // (3) check if creating is needed, then alloc page for page table
                          // CAUTION: this page is used for page table, not for common data page
                          // (4) set page reference
        uintptr_t pa = 0; // (5) get linear address of page
                          // (6) clear page content using memset
                          // (7) set page directory entry's permission
    }
    return NULL;          // (8) return page table entry
#endif
//如果原本就有二级页表或者建立了新的页表，返回对应的地址
    pde_t *pdep = &pgdir[PDX(la)];
    if (!(*pdep & PTE_P)) {
        struct Page *page;
        //根据create的值来判定是否创建
        if (!create || (page = alloc_page()) == NULL) {
            return NULL;
        }
        set_page_ref(page, 1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa), 0, PGSIZE);
        *pdep = pa | PTE_U | PTE_W | PTE_P;
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
```

#### 请描述页目录项（PDE）和页表项（PTE）中每个组成部分的含义以及对ucore而言的潜在作用

* PDE
  * PDE_A，PDE_D，PDE_W：与高速缓存相关，记录该页面是否被访问过，是否执行了写穿透策略，如果ucore和硬件的cache交互，则需要用到这些位
  * PDE_U:当前页的访问权限
  * PDE_R:决定了当前页是否可写，权限控制
  * PDE_P：决定当前页是否存在
* PTE
  * 大多数与PDE相同
  * PTE_C: 与高速缓存相关，记录该页面是否被访问过，是否执行了写穿透策略，如果ucore和硬件的cache交互，则需要用到这些位
  * PTE_G:控制TLB地址的更新策略
  * PTE_D:该页是否被写过。

#### 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

* 保存引发异常页的地址在寄存器中
* 设置错误代码
* 引发错误page fault

### 【练习3】释放某虚地址所在的页并取消对应二级页表项的映射

根据注释实现代码，具体的实现思路如下所示:

首先释放la地址所指向的页，并设置对应的pte值（即找到对应的pte页，把所在页的ref-1）。如果该页的ref为0，则释放pte所在的页，再清空TLB

具体的代码如下：

```c
//确认页表项是否存在
if (*ptep & PTE_P) {
    //找到pte所在页
        struct Page *page = pte2page(*ptep);
    //减少引用
        page_ref_dec(page) 
            //判断是否为0
        if (page->ref== 0) {
            free_page(page);
        }
        *ptep = 0;
        tlb_invalidate(pgdir, la);
    }
```

#### 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，对应关系是什么？

页表中的页目录项保存的物理页面地址 & 页表项保存的物理页面地址都对应于Page数组中的某一页

#### 如果希望虚拟地址与物理地址相等，则需要如何修改lab2？

* 消除段机制启动前，虚拟地址和线性地址之间存在的偏差
* 消除页机制的线性偏差

所以应该

* 将虚拟地址改称0x100000
* 将基地址改称0x00000000
* 取消0-4M的页映射代码

### 与参考答案的不同

* 练习1实现有所不同，已经在报告中体现出来

### 本实验中重要的知识点

* 连续分配管理算法
  * 首次适应算法：空闲分区以地址递增的次序链接，分配内存时顺序查找，找到大小能满足要求的第一个空闲分区
  * 最佳适应算法：空闲分区按照容量递增的方式形成分区链，找到第一个能满足的空闲分区
  * 最坏适应算法：空闲分区按照容量递减的方式形成分区链，找到第一个能满足的空闲分区
  * 邻近适应算法：分配内存时从此次查找结束的地方开始，其余和首次适应算法一样

### 本实验中没有体现出来的重要的os知识点

* 碎片整理
  * 碎片整理是通过调整进程占用的分区位置来减少或者避免分区碎片