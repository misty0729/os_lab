## Lab8实验报告

计65 钱姿

2015010207

### 【练习0】填写已有实验

只需要依次将实验1-7的代码填入本实验中就可，不需要其他改进

### 【练习1】完成读文件操作的实现（需要编码）

> 首先了解打开文件的处理流程，然后参考本实验后续的文件读写操作的过程分析，编写在sfs_inode.c中的sfs_io_nolock读文件中数据的实现代码

#### 打开文件的处理流程

打开文件的调用顺序为：file_open->vfs_open->vfs_lookup(vop_open)

`vfs_lookup`函数是一个针对目录的操作函数，需要通过`path`找到目标文件首先要 找 到根 目录` ”/”`， 然后又由接下 来的调用`vfs_lookup->vop_lookup->get_device->vfs_get_bootfs `找到根目录`”/”`对应的`inode`数据结构`node`，然`fs_lookup`会调用`vop_lookup`找到对应文件的索引节点。找到索引节点之后，再调用宏定义`vop_open`打开文件。

#### 读文件实现分析

在实现过程中，主要修改的是`sfs_io_nolock`的代码，该函数的功能是给定一个文件的`inode`以及需要读写的偏移量和大小，转换成数据块级别的读写操作。具体需要实现的如下：

* 根据文件系统和文件inode的结构体查询到要进行操作的块在磁盘上的块号
* 调用函数进行读操作

具体代码如下：

```C++
if ((blkoff = offset % SFS_BLKSIZE) != 0) {	//读取第一部分的数据
        size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);	//计算第一个数据块的大小
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {	//找到内存文件索引对应的block的编号
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0) {
            goto out;
        }
    	//完成实际的读写操作
        alen += size;
        if (nblks == 0) {
            goto out;
        }
        buf += size;
        blkno++;
        nblks--;
    }
	//读取中间部分的数据，将其分为size大学的块，然后一次读一块直至读完
    size = SFS_BLKSIZE;
    while (nblks != 0) {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_block_op(sfs, buf, ino, 1)) != 0) {
            goto out;
        }
        alen += size;
        buf += size;
        blkno++;
        nblks--;
    }
	//读取第三部分的数据
    if ((size = endpos % SFS_BLKSIZE) != 0) {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, 0)) != 0) {
            goto out;
        }
        alen += size;
    }
```

>请在实验报告中给出设计实现”UNIX的PIPE机制“的概要设方案，鼓励给出详细设计方案

管道是进程间通信的一个基础设施，管道缓存了需要传输的数据，接通了两个需要传输数据的进程。

实现的时候需要建立一个虚拟的设备，在内存中放置一个缓冲区，在fork进程的时候将其`stdin`和`stdout`两个设备文件正确地设置为指向该设备。其他具体的实现可以参考已有的`stdin`和`stdout`两个设备，包括合适的设置缓冲，以及等待输入时候的阻塞等待。

在`Linux`中，管道的实现并没有使用专门的数据结构，而是借助了文件系统的file结构和`VFS`的索引节点`inode`。通过将两个` file` 结构指向同一个临时的 `VFS `索引节点，而这个` VFS`索引节点又指向一个物理页面而实现的。

### 【练习2】完成基于文件系统的执行程序机制的实现（需要编码）

> 改写`proc.c`中的`load_icode`函数和其他相关函数，实现基于文件系统的执行程序机制。执行：`make qemu`。如果能看看到`sh`用户程序的执行界面，则基本成功了。如果在`sh`用户界面上可以执行`”ls”,”hello”`等其他放置在`sfs`文件系统中的其他执行程序，则可以认为本实验基本成功。

在alloc_proc()函数中：

增加对与filesp内容的初始化：

```C++
proc->filesp = NULL;
```

在do_fork函数中：

增加对于filesp的复制处理，直接调用实现好的函数即可:

```c++
if (copy_files(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_fs;
 }
```

load_icode()函数

根据注释完成下列步骤：

* 建立内存管理器
* 建立页目录
* 将文件逐个段加载到内存中
* 建立相应的虚拟内存映射表
* 建立并初始化用户堆栈
* 处理用户栈中传入的参数
* 设置用户进程的中断帧

具体的代码如下：

```C++
assert(argc >= 0 && argc <= EXEC_MAX_ARG_NUM);
//(1)建立内存管理器
if (current->mm != NULL) {	//要求当前内存管理器为空
    panic("load_icode: current->mm must be empty.\n");
}

int ret = -E_NO_MEM;
struct mm_struct *mm;	//建立内存管理器
if ((mm = mm_create()) == NULL) {
    goto bad_mm;
}
//(2)建立页目录
if (setup_pgdir(mm) != 0) {
    goto bad_pgdir_cleanup_mm;
}
struct Page *page;	//建立页表

//(3)从文件加载程序到内存
struct elfhdr __elf, *elf = &__elf;
if ((ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0)) != 0) {
    goto bad_elf_cleanup_pgdir;
}

if (elf->e_magic != ELF_MAGIC) {
    ret = -E_INVAL_ELF;
    goto bad_elf_cleanup_pgdir;
}

struct proghdr __ph, *ph = &__ph;
uint32_t vm_flags, perm, phnum;

struct proghdr *ph_end = ph + elf->e_phnum;
for (phnum = 0; phnum < elf->e_phnum; phnum ++) {
    off_t phoff = elf->e_phoff + sizeof(struct proghdr) * phnum;
    if ((ret = load_icode_read(fd, ph, sizeof(struct proghdr), phoff)) != 0) {
        goto bad_cleanup_mmap;
    }
    if (ph->p_type != ELF_PT_LOAD) {
        continue ;
    }
    if (ph->p_filesz > ph->p_memsz) {
        ret = -E_INVAL_ELF;
        goto bad_cleanup_mmap;
    }
    if (ph->p_filesz == 0) {
        continue ;
    }

    vm_flags = 0, perm = PTE_U;		//建立虚拟地址与物理地址之间的映射
    if (ph->p_flags & ELF_PF_X) vm_flags |= VM_EXEC;
    if (ph->p_flags & ELF_PF_W) vm_flags |= VM_WRITE;
    if (ph->p_flags & ELF_PF_R) vm_flags |= VM_READ;
    if (vm_flags & VM_WRITE) perm |= PTE_W;
    if ((ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;
    }
    off_t offset = ph->p_offset;
    size_t off, size;
    uintptr_t start = ph->p_va, end, la = ROUNDDOWN(start, PGSIZE);

    ret = -E_NO_MEM;

    //复制数据段和代码段
    end = ph->p_va + ph->p_filesz;	 //计算数据段和代码段终止地址
    while (start < end) {
        if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
            ret = -E_NO_MEM;
            goto bad_cleanup_mmap;
        }
        off = start - la, size = PGSIZE - off, la += PGSIZE;
        if (end < la) {
            size -= la - end;
        }
        //每次读取size大小的块，直至全部读完
        if ((ret = load_icode_read(fd, page2kva(page) + off, size, offset)) != 0) {
            goto bad_cleanup_mmap;
        }
        start += size, offset += size;
    }

    //建立BSS段
    end = ph->p_va + ph->p_memsz;	//同样计算终止地址
    if (start < la) {
        if (start == end) {
            continue ;
        }
        off = start + PGSIZE - la, size = PGSIZE - off;
        if (end < la) {
            size -= la - end;
        }
        memset(page2kva(page) + off, 0, size);
        start += size;
        assert((end < la && start == end) || (end >= la && start == la));
    }
    while (start < end) {
        if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
            ret = -E_NO_MEM;
            goto bad_cleanup_mmap;
        }
        off = start - la, size = PGSIZE - off, la += PGSIZE;
        if (end < la) {
            size -= la - end;
        }
        //每次操作size大小的块
        memset(page2kva(page) + off, 0, size);
        start += size;
    }
}
sysfile_close(fd);	//关闭文件，加载程序结束

//(4)建立相应的虚拟内存映射表
vm_flags = VM_READ | VM_WRITE | VM_STACK;
if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
    goto bad_cleanup_mmap;
}
assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);

//(5)设置用户栈
mm_count_inc(mm);
current->mm = mm;
current->cr3 = PADDR(mm->pgdir);
lcr3(PADDR(mm->pgdir));

//setup argc, argv
uint32_t argv_size=0, i;
for (i = 0; i < argc; i ++) {
    argv_size += strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
}
//(6)处理用户栈中传入的参数，其中argc对应参数个数，uargv[]对应参数的具体内容的地址
uintptr_t stacktop = USTACKTOP - (argv_size/sizeof(long)+1)*sizeof(long);
char** uargv=(char **)(stacktop  - argc * sizeof(char *));

argv_size = 0;
for (i = 0; i < argc; i ++) {
    uargv[i] = strcpy((char *)(stacktop + argv_size ), kargv[i]);
    argv_size +=  strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
}

stacktop = (uintptr_t)uargv - sizeof(int);	 //计算当前用户栈顶
*(int *)stacktop = argc;

//(7)设置进程的中断帧   
struct trapframe *tf = current->tf;
memset(tf, 0, sizeof(struct trapframe));	//初始化tf，设置中断帧
tf->tf_cs = USER_CS;
tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
tf->tf_esp = stacktop;
tf->tf_eip = elf->e_entry;
tf->tf_eflags = FL_IF;
ret = 0;
//(8)错误处理部分
out:
	return ret;	 
bad_cleanup_mmap:
	exit_mmap(mm);
bad_elf_cleanup_pgdir:
	put_pgdir(mm);
bad_pgdir_cleanup_mm:
	mm_destroy(mm);
bad_mm:
    goto out;
```

>请在实验报告中给出设计实现基于”`UNIX`的硬链接和软链接机制“的概要设方案，鼓励给出详细设计方案

硬链接：不同文件对应同一个磁盘inode，即不同的文件entry使用相同的number，对应磁盘上相同的inode。在磁盘的inode结构体中，记录有多少个文件连接到了同一个inode，当删除文件的时候，就递减这个计数，当减少到0的时候就将该文件inode本身删除

软链接：一个文件指向了另一个文件名。要在inode上增加标记位确认这个文件是普通文件还是软链接文件，在进行打开文件或是保存文件的时候，操作系统需要根据软链接指向的地址再次在文件目录中进行查询，寻找或创建相应的inode

运行make grade得分为190/190

### 与参考答案的区别

按照注释写，无实现思路的差别

### 实验中涉及到的知识点：

* 文件系统：文件、文件描述符、软链接、硬链接

### 实验中未涉及到的知识点

* 文件分配
* I/O子系统的控制