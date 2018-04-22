# Lab4 实验报告

## 实验涉及知识点
内核线程控制块（TCB）的构成  
创建内核线程的步骤  
复制创建新线程、切换到新线程的步骤  

实验中有，原理课没有的知识点：  
为线程分配唯一pid的具体方法  
完成线程切换的方法（通过中断机制）  

实验中未涉及，原理课讲到的知识点：  
进程与线程的区别（实验中对二者的处理未作区分）  
线程的就绪状态与等待状态的区别（实验中均为`state = RUNNABLE`）  

## 练习1 分配并初始化一个进程控制块
初始化TCB，将`proc_struct`的成员变量清零。  
其中以下变量除外：  
```
proc->state = PROC_UNINIT; //PROC_UNINIT实际上也是0
proc->pid = -1;
proc->cr3 = boot_cr3;
```
- `state`设置为`PROC_UNINIT`：表示当前线程刚创建，待执行`proc_init()`或`wakeup_proc()`变为运行状态（`PROC_RUNNABLE`）。
- `pid`设置为-1：表示此时进程尚不合法，没有合法的pid。
- `cr3`设置为`boot_cr3`：内核线程均共享同一个页表，即内核进程ucore_os的页表，其页目录表基地址存放在`boot_cr3`中。这里`cr3`变量替代了`mm->pgdir`的功能。

我的实现与答案一致。

### 1.1 请说明proc_struct中 struct context context 和 struct trapframe *tf 成员变量含义和在本实验中的作用是啥?
- `context`是当前线程的上下文，具体来说就是一系列通用寄存器的值（不包括`%eax`）：
```
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```
在本实验中，`context`主要用于在线程切换（switch.S）时，将当前进程寄存器的值压栈保存，同时将目标进程寄存器的值弹栈载入，以完成进程状态的切换。  

- `trapframe`一般是在该任务通过某种方式（比如中断或者系统调用）主动或被动进入内核时保存的，存于trapframe结构体中，存于该任务的内核栈中，此任务的控制块存放一个指向它的指针。若任务是内核线程，任务的栈与其内核栈是一个。其组成如下：
```
struct trapframe {
    /* A:  */
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* B:  below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* C:  below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```
如注释，主要分为三组变量：  
- A：由软件保存的，一系列段寄存器（ds、es、fs、gs）、通用寄存器的值，以及中断号`tf_trapno`。
- B：由硬件保存的，当前线程中断时，出现在内核栈上的信息（err、eip、cs、eflags）。
- C：由硬件保存的，从用户态切换到内核态时，需要额外存到内核栈上的信息（esp、ss）。

综上，本实验中`trapframe* tf`的主要功能是：在发生中断（或异常）时，保存当前线程的运行状态。

## 练习2 为新创建的内核线程分配资源
练习2实现`do_fork()`函数，完成对新内核线程的创建。

具体而言，需要为新线程分配TCB、内核栈`kstack`，复制原线程的中断帧`trapframe`和上下文`context`到新线程（在`initproc`的创建中，部分`trapframe`内容来自`kernel_thread()`函数中的临时变量`tf`，另一部分`trapframe`以及`context`则在`do_fork()--->copy_thread()`函数中设置。），添加新线程到进程链表和哈希表，唤醒新线程，并最终返回新线程号。

一开始我没有对“添加新线程到线程列表”的步骤关闭中断。屏蔽中断是必要的，因为新线程分配pid、将线程插入链表与哈希表、nr_process++这些操作应该具有原子性（都与线程个数有关），如果中途被打断，将导致进程队列的混乱，进而可能导致程序崩溃。

添加“关闭中断”的相关代码后，我与答案的实现一致。

### 2.1 请说明ucore是否做到给每个新fork的线程一个唯一的id?请说明你的分析和理由。
能做到。`get_pid()`函数保证了每次分配id的唯一性。  
该函数使用了静态变量`last_pid`表示上一次分配的pid，这次只需要从`last_pid`的下一个id(`++last_pid`)开始查看是否可以分配即可。  
最笨的方法当然就是每次查找链表所有进程，看是否有某进程的pid与当前打算分配的pid重复，若重复则`last_pid++`，重新从链表头开始查找所有进程，直到某次循环整个链表都没有找到重复，就表示当前的`last_pid`无重复、可分配。  
但是这样做的效率实在是太低了。函数中加了一个优化，使用静态变量`next_safe`表示链表中最小的大于`last_pid`的进程pid值。这样，即使我们在链表中找到了某个进程的pid与`last_pid`相同，也不需要`last_pid++`后重新从头开始查找链表每一个进程，只有当`++last_pid >= next_safe`时才会重新从头开始查找链表每一个进程，这是因为，我们只用`next_safe`来记录了最小的大于`last_pid`的进程pid值，但不知道次小的大于`last_pid`的进程pid值，所以一旦`last_pid >= next_safe`，就需要重置next_safe（`next_safe = MAX_PID`)并从头开始查找每一个进程（这样做其实是为了维护新的`next_safe`）。

## 练习3 阅读代码,理解 proc_run 函数和它调用的函数如何完成进程切换的。
`proc_run`函数被内核函数`schedule()`调用，负责让被处理机选中的下一个线程进入运行状态，完成线程的切换。  
具体而言，若目标线程不是当前线程，就在屏蔽中断的前提下，执行以下步骤：  
- 当前进程`current`设为目标进程；
- 加载相应的内核栈、页目录表（将栈顶指针指向新线程的栈空间，修改cr3寄存器）；
- 调用`switch_to`完成上下文的切换，并跳到`forkrets`（在trapentry.S中）

其中，`switch_to`的参数是切换前的进程（from）的context地址与切换后的进程（to）的context地址。在`switch_to`中，现将切换前进程的所有信息保存到from的context中，然后将切换后进程的context内容放入寄存器中（将eip放到栈上，作为return address）。

### 3.1 在本实验的执行过程中,创建且运行了几个内核线程?
本实验创建和运行了`idleproc`和`initoproc`两个内核线程。  
idle是一个循环，不断释放自己的时间片，内核在没有其他进程可以运行的情况下，会选择运行此进程。  
本实验的init进程只是打印字符串然后退出，之后的init进程还会进行更多的动作。  

### 3.2 语句 local_intr_save(intr_flag);....local_intr_restore(intr_flag);	 在这里有何作用?请说明理由  
两条语句分别负责关闭中断和打开中断。  
两条语句中间的代码完成了从当前线程到目标线程的切换，这4个步骤要么都进行，要么都不进行，这样才能保证内核栈顶指针、页表机制、寄存器等不发生错乱。因此，这两条语句保护了切换任务上下文的过程不被中断破坏。
