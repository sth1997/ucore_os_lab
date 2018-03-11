# Lab1 实验报告

## 练习1

### 1.1 操作系统镜像文件ucore.img是如何一步一步生成的?(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义,以及说明命令导致的结果)  
注：由于makefile涉及一些语法，描述起来不方便，直接通过 make =V查看实际执行的命令，对其进行描述  

#### ①编译内核代码，生成obj文件  
gcc -I\*\*\*\* -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -c kern/\*\*\*.c -o obj/kern/\*\*\*.o  
参数：  

    -I\*\*\*\* 设定引用文件查找目录  
    -fno-builtin -nostdinc 关闭内建库  
    -Wall 开启所有警告  
    -ggdb -gstabs 添加调试信息  
    -m32 生成32位代码  
    -fno-stack-protector 不生成栈保护代码  

#### ②链接生成内核ELF映像  
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o  
参数：  

    -m    elf_i386生成32位ELF映像  
    -nostdlib 关闭内建库  
    -T tools/kernel.ld 使用链接器脚本tools/kernel.ld，这个脚本描述了代码和数据在内存中的布局，以及设定了内核入口地址  

#### ③编译bootloader代码，生成obj文件  
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o  
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o  
参数：  

    -Os 优化选项  
    其余参数同①  

#### ④链接生成bootloader ELF映像  
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o  
参数：  

    -N 设置代码段和数据段都可读可写，关闭动态链接  
    -e start 指定入口点符号为start  
    -Ttext 0x7C00设置代码段起始地址为0x7c00  
    其余参数同②

#### ⑤生成bootloader二进制代码  
objcopy -S -O binary obj/bootblock.o obj/bootblock.out  
参数：  

    -S 不拷贝重定位信息和调试信息  
    -O binary拷贝二进制代码  
    obj/bootblock.o obj/bootblock.out将ELF格式的obj/bootblock.o文件中的代码段拷贝到obj/bootblock.out

#### ⑥生成启动扇区  
首先，生成tools/sign工具  
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o  
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign  
之后，使用sign工具检查obj/bootblock.out文件的大小是否超过510字节，生成启动扇区bin/bootblock（该命令没有输出出来，所以从Makefile文件中复制过来）  
@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)  
参数： 

    -O2 优化选项  
    -Wall 开启所有警告  
    -g 生成可为gdb调试用的文件，包含调试信息  

#### ⑦初始化磁盘镜像文件  
dd if=/dev/zero of=bin/ucore.img count=10000  
参数：  

    if=/dev/zero 将一个全零的设备文件当作输入文件  
    of=bin/ucore.img 将bin/ucore.img当作输出文件  
    count=10000 共10000个block（每个block默认512B）  

#### ⑧将启动扇区映像写入镜像文件  
dd if=bin/bootblock of=bin/ucore.img conv=notrunc  
参数：  

    if=bin/bootblock 将⑥生成的启动扇区文件当作输入文件  
    of=bin/ucore.img 将bin/ucore.img当作输出文件  
    conv=notrunc 不截短输出文件  

#### ⑨将内核映像写入镜像文件  
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc  
参数：  

    seek=1 从输出文件开头跳过1个block后开始复制（最开头的1个block512B用来存放启动扇区映像）  
    其余参数同⑧

### 1.2 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?
从sign.c的代码可看出，一个符合规范的硬盘主引导扇区应有512B。  
其中，第511个字节为0x55，第512个字节为0xAA。（假设字节下标从1开始）

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

### 6.2&6.3  
给的注释都非常清楚，直接按照注释描述的步骤写即可，非常简单。  
只是有一点与答案不同，答案的6.2不知为何并没有考虑T_SYSCALL。