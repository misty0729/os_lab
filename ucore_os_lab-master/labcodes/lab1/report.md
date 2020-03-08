# Lab1实验报告**

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

