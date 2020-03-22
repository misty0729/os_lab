## Lab2实验报告

计65 钱姿 2015010207

### 【练习0】填写已有实验

完成`debug/kdebug.c`中函数的合并

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