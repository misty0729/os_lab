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

