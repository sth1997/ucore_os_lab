# Lab1 实验报告

## 实验涉及知识点

与本实验有关的知识点有：  
```
    系统启动（包括bootloader、实模式转保护模式等）
    x86的异常和中断机制
    中断实现系统调用
    中断描述符表
```

## 练习1：理解通过make生成执行文件的过程

### 1.1 操作系统镜像文件UCORE.IMG是如何一步一步生成的？(需要比较详细地解释MAKEFILE中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

##### 【ucore.img的生成代码】

　　观察Makefile文件，首先找到ucore.img的生成代码：


```shell
UCOREIMG	:= $(call totarget,ucore.img)
```

```shell
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

　　可见，为生成ucore.img镜像文件，需首先生成kernel和bootclock。为了更好地理解Makefile的运行过程，我在Shell中执行了如下指令：

```shell
make V=
```

　　这条指令是Makefile的调试指令，显示出了Makefile的整个执行过程。以下将结合执行过程进行分析。

##### 【bootblock的生成代码】

```shell
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

　　在执行过程中的相应指令如下：

```shell
ld -melf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```

　　由此可见，要生成bootblock，首先需要生成相关的.o文件：bootasm.o、bootmain.o，以及sign工具。

###### bootasm.o，bootmain.o的生成代码

```shell
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
```

　　在执行过程中的相应指令如下：

```shell
【bootasm.o】
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
【bootmain.o】
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
```

　　其中关键的参数有：

-  -I(+dir)　添加搜索头文件的路径
-  -fno-builtin　除非__builtin_前缀出现，否则不进行內建函数优化
-  -ggdb　生成可供gdb使用的调试信息，以便使用qemu+gdb来调试bootloader和ucore
-  -m32　生成适用于X86-32环境的代码，这是为了使操作系统符合模拟硬件（32位的80386）的要求
-  -gstabs　生成stabs格式的调试信息，使用ucore的monitor可以显示出相应的信息
-  -nostdinc　不使用标准库（因为标准库是给应用程序用的）
-  -fno-stack-protector　不生成用于检测缓冲区溢出的代码
-  -Os　为减小代码大小而进行优化

###### sign工具的生成代码

```shell
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

　　在执行过程中的相应指令如下：

```shell
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```

　　其中主要的参数有：

- -g　利用操作系统的“原生格式”生成调试信息
- -Wall　打开警报（Warning）开关
- -O2　优化算法，提高代码的运行速度

###### 进一步生成bin/bootblock

　　在生成好bootasm.o、bootmain.o以及sign工具后，将继续执行如下步骤：

　　(a) 首先生成bootblock.o，执行过程中的相应指令为：

```shell
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```

　　其中关键的参数有：

- `-m elf_i386`　表示模拟i386上的连接器（在64位Linux机器上用gcc编译32位程序时需加-m32，ld指令则加-m elf_i386）
- -nostdlib　不使用标准库
- -N　设置代码段和数据段均可读写
- -e (+entry)　指定入口
- -Ttext　指定代码段开始位置

　　(b) 然后拷贝二进制代码bin/bootblock.o到obj/bootblock.out，提示信息为：

```shell
'obj/bootblock.out' size: 472 bytes
```

　　(c) 最后使用sign工具处理bootblock.out，生成最后的bootblock，提示信息为（注意到此时bootblock的大小恰为512字节）：

```shell
build 512 bytes boot sector: 'bin/bootblock' success!
```


##### 【kernel的生成代码】

```shell
$(kernel): tools/kernel.ld
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```

　　在执行过程中的相应指令如下：

```shell
ld -melf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o
obj/kern/libs/readline.o obj/kern/libs/stdio.o
obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o 
obj/kern/debug/panic.o obj/kern/driver/clock.o
obj/kern/driver/console.o obj/kern/driver/intr.o
obj/kern/driver/picirq.o obj/kern/trap/trap.o
obj/kern/trap/trapentry.o obj/kern/trap/vectors.o 
obj/kern/mm/pmm.o  obj/libs/printfmt.o 
obj/libs/string.o
```

　　其中新出现的重要参数为：

- -T (+scriptfile)　让连接器使用指定的脚本

　　可见，为生成kernel，首先需要生成kernel.ld和一系列相关的.o文件，其中kernel.ld已存在于tools/路径下。

　　生成相关.o文件的代码如下：
​    
```shell
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```

　　以trap.o为例（其他.o文件的生成过程类似），其执行过程中的相应指令如下：

```shell
gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32
-gstabs -nostdinc  -fno-stack-protector 
-Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ 
-c kern/trap/trap.c -o obj/kern/trap/trap.o
```


##### 【最后：将bootblock和kernel拷贝到ucore.img中】

　　dd是 Linux/UNIX 下的一个非常有用的命令，作用是用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。

　　首先，生成10000个块的文件，每个块默认512字节，用0填充。

```shell
dd if=/dev/zero of=bin/ucore.img count=10000
```

　　其中参数解释如下：

- if= 输入文件路径
- of= 输出文件路径
- count=(+blocks) 仅拷贝blocks个块，块大小等于ibs指定的字节数，这里默认512字节

　　随后，将bootblock的内容写入ucore.img的第一个块，再从第二个块开始写入kernel的全部内容：

```shell
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

　　其中参数解释如下：

- conv=(+conversion) 用指定参数转换文件
- notrunc 不截断输出文件
- seek=(+blocks) 从输出文件开头跳过blocks个块后再开始复制

　　**至此，便完成了操作系统镜像文件ucore.img的全部生成过程。**


### 1.2 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

　　硬盘主引导扇区（bootblock）的规范性检测和标注，是在sign.c中完成的。其对应的命令行指令为：

```shell
bin/sign obj/bootblock.out bin/bootblock
```

　　阅读sign.c的代码可知，其首先对bootblock.out文件进行了**大小检测**，以确保bootloader的原始代码小于510字节。

　　其次，它将bootblock.out读入512字节大小的char* buf中，并对buf末尾的两个字节进行了**特殊标记**：

```c
buf[510] = 0x55;
buf[511] = 0xAA;
```

　　最后，再将标记后的bootblock.out写入到最终的bootblock可执行文件中。

　　若能顺利执行到此，则系统保证了该硬盘主引导扇区是符合规范的。**其特征即为最后的特殊标记。**



## 练习2

### 2.1 从CPU加电后执行的第一条指令开始,单步跟踪BIOS的执行。  
打开gdbinit，删除以下两行，使得make debug时不自动运行：  

    break kern_init  
    continue  
之后新增一行：

    set	architecture	i8086  
执行make debug，开始调试。  
此时，qemu停在0xfffffff0，执行 x/i 0xfffffff0，可以得到开机后的第一条指令：ljmp $0xf000, $0xe05b。  
执行si，即可跳转到 0xe05b。  

### 2.2 在初始化位置0x7c00设置实地址断点,测试断点正常。  
继2.1，执行b *0x7c00，在0x7c00设置断点。  
执行c，程序将运行到0x7c00，说明断点设置成功。  

### 2.3 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较。  
继2.2，执行x /10i $pc，可以得到0x7c00后的汇编指令，将其与bootasm.s和bootblock.asm比较，大致相同，但稍有区别。  
bootblock.asm中的%eax在gdb内显示的都为%ax。  
bootasm.s中xorw在gdb内显示为xor，testb在gdb中显示为test。  

### 2.4 自己找一个bootloader或内核中的代码位置,设置断点并进行测试。  
可以使用在2.1中删除的两行指令：  

    break kern_init  
    continue  
这样就会在kern_init()函数停下来，之后单步调试即可。   

## 练习3  

### 分析bootloader进入保护模式的过程  

初始化，包括关终端、寄存器置零：  

    cli  
    cld  
    xorw %ax, %ax  
    movw %ax, %ds  
    movw %ax, %es  
    movw %ax, %ss  
由于历史原因，在进入保护模式前， 要开启A20来获得使用完整4G内存的能力。开启A20步骤：  

    seta20.1:
        inb $0x64, %al          #等待直到8042键盘控制器不忙  
        testb $0x2, %al  
        jnz seta20.1  
        movb $0xd1, %al         #向8042控制端口写入写P2端口命令（0xd1）  
        outb %al, $0x64  
    seta20.2:  
        inb $0x64, %al          #等待直到8042键盘控制器不忙  
        testb $0x2, %al  
        jnz seta20.2  
        movb $0xdf, %al         #向8042数据端口写入0xdf来打开A20  
        outb %al, $0x60  
初始化GDT表：  

    lgdt gdtdesc                #装载全局描述符的基地址和长度进入全局描述符表寄存器  
通过将cr0寄存器的PE位置1,开启保护模式：  

    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0  

执行一个long jump, 把保护模式下的段选择子（index)load到CS寄存器中，这样系统才真正地进入保 护模式。细心研究可以发现ljmp所跳转的地址实际上正是系统正要执行的下一条语句。但不同的是此时的寻址方式已经变了 系统切换到了32位模式，寻址空间不再有1MB的限制。CS中存放的也不再是段地址，而是GDT的index：  

    ljmp $PROT_MODE_CSEG, $protcseg  

初始化各个段寄存器：  

    movw $PROT_MODE_DSEG, %ax  
    movw %ax, %ds  
    movw %ax, %es            
    movw %ax, %fs            
    movw %ax, %gs            
    movw %ax, %ss  

初始化栈并跳转置bootmain：  

    movl $0x0, %ebp  
    movl $start, %esp  
    call bootmain  
转到保护模式完成。  

## 练习四

### 分析bootloader加载ELF格式的OS的过程  

按照代码逻辑自底向上分析，首先分析readsect函数，它实现了从设备的第secno个扇区读取数据到dst位置的功能。  
```
    static void
    readsect(void *dst, uint32_t secno) {
        waitdisk();     //等待磁盘控制器就绪

        outb(0x1F2, 1);                         // 读取扇区数量为1
        outb(0x1F3, secno & 0xFF);
        outb(0x1F4, (secno >> 8) & 0xFF);
        outb(0x1F5, (secno >> 16) & 0xFF);
        outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
        //以上这几行代码指定了扇区号，同时将29～31位强制置1,28位强制置0表示访问disk0
        outb(0x1F7, 0x20);                      // 读取扇区

        waitdisk();     //等待磁盘控制器就绪

        insl(0x1F0, dst, SECTSIZE / 4);     //读取扇区至dst位置，/4是因为以双字为单位
    }
```

其次是readseg函数，它可以实现读取任意长度内容的功能。
```
    static void
    readseg(uintptr_t va, uint32_t count, uint32_t offset) {
        uintptr_t end_va = va + count;
        va -= offset % SECTSIZE;
        uint32_t secno = (offset / SECTSIZE) + 1;   //+1是因为要跳过存放bootloader的0扇区

        for (; va < end_va; va += SECTSIZE, secno ++) {
            readsect((void *)va, secno);
        }
    }
```

最外层是bootmain函数，它实现了加载内核映像至内存的功能。
```
    void
    bootmain(void) {
        readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);//读取ELF头

        if (ELFHDR->e_magic != ELF_MAGIC) {//若不相等，则不是合法ELF文件
            goto bad;
        }

        struct proghdr *ph, *eph;

        ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);//ph为program header表的地址
        eph = ph + ELFHDR->e_phnum;
        for (; ph < eph; ph ++) {//将每个程序节读入到对应的内存
            readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
        }

        ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))(); //ELF头结构里面有内核的入口点地址，是一个整数，可以通过将此整数转换为函数指针然后调用来跳转到内核。

    bad:
        outw(0x8A00, 0x8A00);
        outw(0x8A00, 0x8E00);
        while (1);
    }
```

## 练习五

完全按照kdebug.c中print_stackframe()函数中的注释所描述的步骤来写即可。代码如下：
```
    void
    print_stackframe(void) {
        uint32_t ebp = read_ebp();
        uint32_t eip = read_eip();
        uint32_t * addr = (uint32_t *)ebp;
        print_debuginfo(eip - 1);
        while (ebp != 0){
            cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
            uint32_t * addr = (uint32_t *)ebp;
            addr += 2;
            for (int i = 0; i < 4; ++i, addr++)
                cprintf("0x%08x ", *addr);
            cprintf("\n");
            print_debuginfo(eip - 1);
            eip = *((uint32_t *)(ebp+4)); //return addr
            ebp = *((uint32_t *) ebp);
        }
    }
```
ebp位置存着上一个栈帧的ebp;  
ebp+4存着return address，也就是caller的eip;  
ebp+8,ebp+12,ebp+16,ebp+20分别存着四个可能的参数。  

还需注意的是，read_ebp定义为必须内联，read_eip定义为不得内联，这都是为了能够获得ebp和eip的真实的值。

栈中最高地址栈帧的输出内容为：  
```
    ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
        <unknow>: -- 0x00007d6d --
```
这应该是第一个使用栈的函数，bootmain.c中的bootmain()函数，由bootasm.S中的call bootmain调用。  
ebp为0x7bf8，这是bootmain栈帧基地址。  
在bootasm.S中，调用call bootmain之前，将esp设置为了start，也就是0x7c00，这也与ebp为0x7bf8相符（将return address和 old ebp压栈）。  
由于bootmain函数并没有参数，所以四个arg实际上是0x7c00~0x7c0f的指令。经验正，bootblock.asm中的这些指令（共9条）确实与栈中四个参数相同（需要注意大端小端）。  

## 练习六

### 6.1中断描述符表(也可简称为保护模式下的中断向量表)中一个表项占多少字节?其中哪几位代表中断处理代码的入口?  
一个表项8字节，其中，16～31位是段选择子，0～15位与48位～63位拼成offset（由于历史原因）。段选择子到GDT中找到相应段描述符，再与offset相加，即为服务程序入口。  

### 6.2 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。
对于中断向量表的每一项（idt[i]），使用SETGATE宏对其初始化。  
istrap项除了T_SYSCALL为1,其余全部为0，表示为中断门（interrupt-gate）。  
段选择子项均为GD_KTEXT，它是kernel代码段的基址。  
__vectors[i]里存着相应中断处理代码的入口，它是相对kernel代码段基址的偏移量，即offset。  
DPL段，对于用户态到内核态的特殊中断（T_SWITCH_TOK和T_SYSCALL），单独设置DPL为3，表示用户态（此中断为从用户态到内核态的转换，所以调用门的DPL为3可以保证用户态的应用程序仍可以触发此“中断”）；其余中断DPL置为0。  
最后，使用lidt指令，加载该中断向量表（idt），设定其线性基地址（base address）和界限（limit，即大小）。

### 6.3 请编程完善trap.c中的中断处理函数trap。
ticks每累加到100,就输出“100 ticks”，并将ticks置零即可。