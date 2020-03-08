# Lab1实验报告

## 【练习1.1】理解通过 make 生成执行文件的过程

### 运行结果分析

执行指令 `make clean` (清除之前的运行结果) 和指令 `make "V="` (查看详细的被执行指令)

* 通过执行`gcc xx -o xx`指令将一系列.c和.s文件编译成目标文件.o

* 随后通过`ld`指令将目标文件链接成可执行文件，在这过程中有两条ld指令

> + ld bin/kernel
> + ld bin/bootblock

分别生成了可执行文件`bin/kernel`和`bin/bootblock`

* 执行指令生成ucore.img, 并将kernel和bootblock也写入ucore中，具体的指令如下

> dd if=/dev/zero of=bin/ucore.img count=10000
>
>  dd if=bin/bootblock of=bin/ucore.img conv=notrunc
>
> dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc

### **Makefile具体分析**

#### 生成kernel

具体的代码为

```makefile
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)

​    @echo + ld $@

​    $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)

​    @$(OBJDUMP) -S $@ > $(call asmfile,kernel)

​    @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

为了生成`bin/kernel`所需要的目标文件，这些文件包括：`kernel.ld` `init.o` `readline.o` `stdio.o` `kdebug.o` `kmonitor.o` `panic.o` `clock.o` `console.o` `intr.o` `picirq.o` `trap.o` `trapentry.o` `vectors.o` `pmm.o`  `printfmt.o` `string.o`,其中`kernel.ld`已经存在

为了生成其他的目标文件，具体代码如下：

```makefile
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```

以 `init.o`为例子说明生成该目标文件指令的相关参数含义

生成init.o需要init.c，此时具体的指令为：

```make
gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
```

参数说明：

| 参数                 | 参数含义                                                     |
| -------------------- | ------------------------------------------------------------ |
| -l[dir]              | 添加搜索头文件                                               |
| -Wall                | 开启常用警告                                                 |
| -fno-builtin         | 不是用C语言的内建函数，使用自己定义的函数（当函数命名产生冲突的时候 |
| -ggdb                | 为gdb调试生成更丰富的信息                                    |
| -gstabs              | 以stab格式生成GDB调试信息                                    |
| -m32                 | 交叉编译选项,生成32位(x86)代码                               |
| -nostdinc            | 不在标准系统目录中搜索头文件，只在-I指定的目录中搜索         |
| -fno-stack-protector | 不生成用于检测缓冲区溢出的代码，检测缓冲区溢出主要给应用程序使用 |

根据目标文件，生成kernel的指令为：

```cmake
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
```

参数说明：

| 参数            | 参数含义               |
| --------------- | ---------------------- |
| -T <scriptfile> | 让链接器使用指定的脚本 |

#### 生成bootblock

具体代码为

```makefile
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```

第一行代码是为了生成需要的目标文件，包括`bootasm.o`、`bootmain.o`、`sign`,

其中生成`bootasm.o`、`bootmain.o`的具体指令和生成kernel的目标文件的具体指令类似，不再赘述。生成`sign`的makefile代码为

```makefile
# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

实际指令为

```
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```

具体参数已经在上面说明

随后根据目标文件生成bootblock的具体指令为

```
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```

参数说明：

| 参数           | 参数含义                   |
| -------------- | -------------------------- |
| -m <emulation> | 模拟为i386上的链接器       |
| -N             | 设置代码段和数据段均可读写 |
| -Ttext         | 指定代码段开始位置         |
| -e <entry>     | 指定入口                   |

#### 生成ucore.img,虚拟磁盘

具体代码为

```makefile
$(UCOREIMG): $(kernel) $(bootblock)
	#初始化ucore.img为5120000 bytes，内容为0的文件
	$(V)dd if=/dev/zero of=$@ count=10000
	#拷贝bin/bootblock到ucore.img第一个扇区
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	#拷贝bin/kernel到ucore.img第二个扇区往后的空间
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

参数说明：

| 参数          | 参数含义                                         |
| ------------- | ------------------------------------------------ |
| -if           | 输入文件                                         |
| -of           | 输出文件                                         |
| -count        | 拷贝的block的数量                                |
| -seek         | 数据拷贝起始block寻址                            |
| -conv=notrunc | 当dd到一个比源文件大的目标文件时，不缩小目标文件 |

## 【练习1.2】一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

`sign.c`文件完成了硬盘主引导扇区的特征标记，文件的主要内容如下：

>buf[510] = 0x55;
>buf[511] = 0xAA;

因此引导扇区只有512个字节，其中倒数第一个为0xAA，倒数第二个为0x55为特征

## 【练习 2】使用 qemu 执行并调试 lab1 中的软件

### 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行

* 首先修改文件`tools/gbinit`

  ```c++
  file bin/kernel
  set architecture i8086
  target remote :1234
  break kern_init
  #删除continue防止直接开始执行
  ```

* 在lab1目录下执行make debug

  可以看到	

  ```0x0000fff0 in ?? ()```

  我们设想的是0xFFFFFFF0，即第一条指令位置，这是因为gdb只显示了当前的PC寄存器的数值，没有考虑cs寄存器的数值，因此我们如果要查看CPU加电后的第一条指令，需要对当前的PC加上CS寄存器的数值

* 在gdb中输入指令

  ```x /i $cs*16 + $pc```

  显示的输出结果为

  ``` 0xffff0:      ljmp   $0x3630,$0xf000e05b```

  第一条指令是一条长跳转指令到`BIOS`代码中执行，这里屏幕输出的是`ljmp`是长跳转指令，因此正确

* 在gdb中进行单步跟踪，输入si或者stepi指令

  ```
  (gdb) si
  0x0000e05b in ?? ()
  (gdb) si
  0x0000e062 in ?? ()
  (gdb) stepi
  0x0000e066 in ?? ()
  ```

### 在初始化位置 0x7c00 设置实地址断点,测试断点正常

* 修改文件`tools/gbinit`

  ```C++
  file bin/kernel
  set architecture i8086
  target remote :1234
  #在0x7c00的地方加入断点
  b	*0x7c00
  #继续执行
  continue
  #输出两条指令
  x /2i $pc
  set architecture i386
  ```

* 在lab1目录下执行make debug

  可以看到

  ```
  Breakpoint 1 at 0x7c00
  
  Breakpoint 1, 0x00007c00 in ?? ()
  => 0x7c00:      cli
     0x7c01:      cld
  ```

  断点测试正常

### 从 0x7c00 开始跟踪代码运行,将单步跟踪反汇编得到的代码与 bootasm.S 和 bootblock.asm 进行比较。

* 修改文件`tools/gbinit`

  ```c++
  file bin/kernel
  set architecture i8086
  target remote :1234
  break kern_init	
  b	*0x7c00
  continue
  x /10i $pc
  ```

* 修改makefile文件

  ```makefile
  $(V)$(QEMU) -no-reboot -d in_asm -D q.log -parallel stdio -hda $< -serial null
  ```

  增加的参数用来将运行的汇编指令保存在q.log中

* 运行 make qemu获得结果

  打开q.log

  ```
  IN: 
  0x00007c00:  cli    
  0x00007c01:  cld    
  0x00007c02:  xor    %ax,%ax
  0x00007c04:  mov    %ax,%ds
  0x00007c06:  mov    %ax,%es
  0x00007c08:  mov    %ax,%ss
  
  ----------------
  IN: 
  0x00007c0a:  in     $0x64,%al
  
  ----------------
  IN: 
  0x00007c0c:  test   $0x2,%al
  0x00007c0e:  jne    0x7c0a
  
  ----------------
  IN: 
  0x00007c10:  mov    $0xd1,%al
  0x00007c12:  out    %al,$0x64
  0x00007c14:  in     $0x64,%al
  0x00007c16:  test   $0x2,%al
  0x00007c18:  jne    0x7c14
  
  ----------------
  IN: 
  0x00007c1a:  mov    $0xdf,%al
  0x00007c1c:  out    %al,$0x60
  0x00007c1e:  lgdtw  0x7c6c
  0x00007c23:  mov    %cr0,%eax
  0x00007c26:  or     $0x1,%eax
  0x00007c2a:  mov    %eax,%cr0
  
  ----------------
  IN: 
  0x00007c2d:  ljmp   $0x8,$0x7c32
  
  ----------------
  IN: 
  0x00007c32:  mov    $0x10,%ax
  0x00007c36:  mov    %eax,%ds
  
  ----------------
  IN: 
  0x00007c38:  mov    %eax,%es
  
  ----------------
  IN: 
  0x00007c3a:  mov    %eax,%fs
  0x00007c3c:  mov    %eax,%gs
  0x00007c3e:  mov    %eax,%ss
  
  ----------------
  IN: 
  0x00007c40:  mov    $0x0,%ebp
  
  ----------------
  IN: 
  0x00007c45:  mov    $0x7c00,%esp
  0x00007c4a:  call   0x7d0d
  ```

  与bootasm.S 和 bootblock.asm比较发现，指令都相同

### 自己找一个 bootloader 或内核中的代码位置，设置断点并进行测试

* 测试断点kern_init, 输出5条指令，最终输出结果为

  ```c++
  0x100000 <kern_init>:        push   %ebp
     0x100001 <kern_init+1>:      mov    %esp,%ebp
  ---Type <return> to continue, or q <return> to quit---
     0x100003 <kern_init+3>:      push   %ebx
     0x100004 <kern_init+4>:      sub    $0x14,%esp
     0x100007 <kern_init+7>:      call   0x100280 <__x86.get_pc_thunk.bx>
  ```

## 【练习 3】分析 bootloader 进入保护模式的过程

#### 代码分析，阅读`bootasm.S`中第12行开始的内容。

```c++
//从cs=0 && ip=0x7c00进入bootloader启动过程
.globl start
start:
//启动进入先要对环境进行清理
.code16                                             # Assemble for 16-bit mode
    //关闭了中断使能
    cli                                             # Disable interrupts
    //使方向标志复位
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    //清空段寄存器
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```

```C++
//开启A20，以便能够通过总线访问更大的内存空间
//这里通过使能全部的32条地址线，我们可以访问4G的内存空间
# Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
seta20.1:
//首先等待8042为空
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1
//然后发送写指令到8042 输入buffer
    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
//等待8042为空
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2
//打开A20
    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

```c++
//初始化gdt表
//bootasm.S中的lgdt gdtdesc把全局描述符表的大小和起始地址共8个字节加载到全局描述符表寄存器GDTR中
lgdt gdtdesc
    //进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```

```C++
    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
//通过长跳转更新cs的基地质
    ljmp $PROT_MODE_CSEG, $protcseg

.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
//设置保护模式的段寄存器，并建立堆栈
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
```

```C++
    //整个过程完成，进入boot主方法
    call bootmain
```

#### 为什么我们要开启A20：

如果我们要在实模式下要访问高位的内存区，要开启A20地址线；在保护模式下，如果A20恒等于0，那么系统只能访问奇数兆的内存，即只能访问`0-1M`、`2-3M`、`4-5M`，这样无法有效访问所有可用内存。所以在保护模式下，必须要开启A20。

#### 如何开启A20:

1、等待8042为空

2、发送写指令到8042 input buffer

3、等待8042为空

4、将对应字节写入8042

#### 如何初始化GDT表：

一个简单的GDT表和其描述符已经静态储存在引导区中，载入即可

#### 如何使能和进入保护模式：

将cr0寄存器的PE位（cr0寄存器的最低位）设置为1，便使能和进入保护模式了

## 【练习4】分析bootloader加载ELF格式的OS的过程。

### bootloader如何读取硬盘扇区

在`bootmain.c`中，用`readsect`函数实现了读取硬盘扇区的功能

```C++
/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    //waitdisk的作用：不断查询读0x1F7寄存器的最高两位，直到最高位为0，并且次高位为1（即磁盘空闲）返回
    waitdisk();
    //设置读取扇区的数目为1
    outb(0x1F2, 1);                         // count = 1
    //在这4个字节线联合构成的32位参数中，29-31位强制设为1，28位(=0)表示访问"Disk 0"，0-27位是28位的偏移量。
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    //读扇区对应的命令字为0x20，放在0x1F7寄存器中；
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors
	
   	//发出命令后再次等待磁盘可哦贡献
    // wait for disk to be ready
    waitdisk();

    // read a sector
    //开始从0xiF0寄存器中读取数据，调用了insl函数
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

### bootloader 是如何加载 ELF 格式的 OS？

```C++
void
bootmain(void) {
    // read the 1st page off disk
    // 首先从bin/kernel文件第一页内容加载到内存地址为0x10000的位置，目的是为了读取kernel文件的ELF Header信息
    //readseg函数包装了readsect，可以读取kernel中任意长度的内容写入虚拟地址中
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
   // is this a valid ELF?
    //校验ELF Header的e_magic字段，以确保这是一个ELF文件
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;
	// ELF头部有描述ELF文件应加载到内存什么位置的描述表
     // 先将描述表的头地址存在ph
    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    // 按照描述表将ELF文件中数据载入内存
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    //加载完毕，通过ELF Header的e_entry得到内核的入口地址，并跳转到该地址开始执行内核代码
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

### 运行调试

* 修改gdbinit为

```makefile
file obj/bootblock.o
target remote :1234
break bootmain
continue
```

* 调用 make debug，输入n直至readseg函数执行完毕，此时查询ELF_Header的e_magic的数值

```
(gdb) x/xw 0x10000
0x10000:        0x464c457f
```

校验成功

## 【练习5】实现函数调用堆栈跟踪函数

### 实现代码

根据注释一步一步实现

```C
 /* LAB1 2015010207 : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
     */
     uint32_t ebp = read_ebp();
      // (2) call read_eip() to get the value of eip. the type is (uint32_t);
      uint32_t eip = read_eip();
      // (3) from 0 .. STACKFRAME_DEPTH
      int i = 0;
      while(i < STACKFRAME_DEPTH && ebp != 0 && eip != 0) {
          // (3.1) printf value of ebp, eip
           cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);   
           // (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
            uint32_t *args = (uint32_t *)ebp + 2;                            
            int j = 0;
            while(j < 4){           
                cprintf("0x%08x ", args[j]);
                j ++;
            }
            //(3.3) cprintf("\n");
            cprintf("\n");   
            // (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
            print_debuginfo(eip - 1);
            //(3.5) popup a calling stackframe
            eip = ((uint32_t *)ebp)[1]; 
            ebp = ((uint32_t *)ebp)[0]; 
            i++;
      }
```



### 获得输出

```c++
Special kernel symbols:
  entry  0x00100000 (phys)
  etext  0x001037c3 (phys)
  edata  0x0010e950 (phys)
  end    0x0010fdc0 (phys)
Kernel executable memory footprint: 64KB
ebp:0x00007b38 eip:0x00100be3 args:0x00010094 0x0010e950 0x00007b68 0x001000a2 
    kern/debug/kdebug.c:298: print_stackframe+33
ebp:0x00007b48 eip:0x00100f6a args:0x00000000 0x00000000 0x00000000 0x0010008d 
    kern/debug/kmonitor.c:125: mon_backtrace+23
ebp:0x00007b68 eip:0x001000a2 args:0x00000000 0x00007b90 0xffff0000 0x00007b94 
    kern/init/init.c:48: grade_backtrace2+32
ebp:0x00007b88 eip:0x001000d1 args:0x00000000 0xffff0000 0x00007bb4 0x001000e5 
    kern/init/init.c:53: grade_backtrace1+37
ebp:0x00007ba8 eip:0x001000f8 args:0x00000000 0x00100000 0xffff0000 0x00100109 
    kern/init/init.c:58: grade_backtrace0+29
ebp:0x00007bc8 eip:0x00100124 args:0x00000000 0x00000000 0x00000000 0x001037c4 
    kern/init/init.c:63: grade_backtrace+37
ebp:0x00007be8 eip:0x00100066 args:0x00000000 0x00000000 0x00000000 0x00007c4f 
    kern/init/init.c:28: kern_init+101
ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d6d --
```

解释最后一行参数的含义

```C++
//整个栈的栈顶地址为0x00007c00，当前 ebp的值是kern_init函数的栈顶地址。ebp指向的栈位置存放调用者的ebp寄存器的值，ebp+4指向的栈位置存放返回地址的值。
ebp:0x00007bf8 
 //kernel_init的返回地址，即bootmain函数调用kern_init对应的指令的下一条指令的地址
eip:0x00007d6e 
//args存放的是bootloader指令的前16个字节    
args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
```

## 【练习6】完善中断初始化和处理