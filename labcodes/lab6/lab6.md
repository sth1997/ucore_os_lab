# Lab6 实验报告

## 实验涉及知识点

- **时间片轮转算法**
- **调度器框架**
- **Stride Scheduling调度算法**

## OS原理中很重要，但实验中未出现的知识点

- **调度算法的评价指标**
- **实时操作系统**
- **优先级反置**

## 练习1 使用 Round Robin 调度算法
### 1.1 请理解并分析sched_calss中各个函数指针的用法，并结合Round Robin调度算法描述ucore的调度执行过程。

`sched_class`中的函数指针定义如下：

```C
    // Init the run queue
    void (*init)(struct run_queue *rq);
    // put the proc into runqueue, and this function must be called with rq_lock
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    // get the proc out runqueue, and this function must be called with rq_lock
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    // choose the next runnable task
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    // dealer of the time-tick
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
```

- `init()`

  主要完成run queue的初始化，即`list_entry`和`proc_num`的初始化。

- `enqueue()`

  将进程proc（的控制块）加入run queue队列，同时完成相应参数的修改（如`time_slice`）。

- `dequeue()`

  将进程proc（的控制块）移出run queue队列，同时完成相应参数的修改（如`time_slice`）。

- `pick_next()`

  调度算法的核心。即在run queue就绪队列中**选择**下一个要运行的进程。

- `proc_tick()`

  在时钟中断时被调用，作用是每隔一段时间，维护当前进程的调度相关参数（如`time_slice、need_resched`）。

要使用这些函数指针，首先需在如`default_sched.c`中完成对这些函数指针的具体实现，其次在`sched_init()`函数中将`sched_class`指向实现好的调度算法类，如：

  ```C
  sched_class = &default_sched_class;
  ```

值得说明的是，这些函数指针不会直接被调用，而是被`sched.c`封装到了不同的功能函数里，如`wakeup_proc()、schedule()`等。

调度的执行过程主要由两部分组成：**时钟中断时**和**发生调度时**。

- **时钟中断时**

  即在时钟中断发生时，对`proc_tick()`函数的调用。

  以RR算法为例，每次时钟中断发生时，都将调用封装好的`RR_proc_tick()`，对当前进程的`time_slice`减一；当`time_slice==0`时（说明当前进程的时间片已用尽），又设置`need_resched`为1，以便下一次中断时（或其他触发源发生时）进行调度。

- **发生调度时**

  即`schedule()`函数被调用时。

  此时将先调用`sched_class_enqueue(current)`让当前进程入队，再`next = sched_class_pick_next()`选出下一个进程，并调用`sched_class_dequeue(next)`让他出队，最后调用`proc_run()`完成进程切换。

  在RR算法中，选择的下一个进程即为队首的进程，而新入队的进程则放在队尾。

补充：在需要时（`do_fork()中唤醒新进程、do_exit()中唤醒父进程、do_kill()中唤醒要杀死的进程`），还会调用`wakeup_proc()`，在唤醒相应进程的同时让其入队。

### 1.2 请简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计。

多级反馈队列算法要维护多个不同优先级的队列，优先级越高的队列分配的**用于调度其进程的时间片**越长，每个队列可采用不同的调度算法。

每个进程将动态维护一个“进程优先级”，并根据自己的优先级插入到不同队列中。

为了实现方便，这里对每个队列多采用时间片轮转的调度算法。

为此，我们可以将`struct run_queue`中现有的`run_list`改为一个数组`runlists[]`，代表不同优先级的队列，并为每个队列设置其用于调度其进程的时间片大小`MAX_TIME_SLICE`（为多少个时钟中断的时长）。

为每个`struct proc_struct`维护一个优先级变量`priority`，在进程入队时根据它的优先级`p`加入到相应的`runlists[p]`中。

对于队列`runlists[p]`，将使用它对应的`MAX_TIME_SLICE`用时间片轮转法进行调度。

`schedule()`选择下一个进程时，尽可能从优先级最高的队列里选择执行进程。

若一个进程在它的时间片中未执行完，则将其移出当前队列，而加入优先级低一级的队列。

## 练习2 实现 Stride Scheduling 调度算法
Stride调度算法的核心在于根据优先级分配时间资源：优先级越高，分到时间片的次数越多。

为每个进程维护一个stride值，和一个步长pass = BIG_STRIDE / priority。每次调度时，选择stride值最小的进程来执行，并将其stride值加上pass的步长。在极限意义下，此算法会让每个进程分得的执行总时间∝1/pass，即与priority成正比。

在实现上，可以大体沿用Round Robin算法的实现方法，不同点在于如下两点：

- Stride算法采用Skew Heap的数据结构。

  相比于普通的双向链表，斜堆将插入、查找（stride最小的节点）、删除的时间分别优化到了O(log(n))、O(1)、O(log(n))，而链表的相应时间复杂度为O(1)、O(n)、O(1)。由于Stride每次调度时都会查找stride最小的进程，这样的优化是很有必要的。在代码中，对应了`skew_heap.h`中的一系列操作。

- Stride算法要维护各进程的stride值。

  stride值仅在**刚选出下一个进程时**更新，对应代码的`stride_pick_next()`函数。

对`BIG_STRIDE=0x7FFFFFFF`的解释：

`0x7FFFFFFF`是32位整型的最大值，使`BIG_STRIDE=0x7FFFFFFF`，可以保证Stride算法的正确性。

与标准答案相比，一开始我忽略了priority为0的情况。

事实上，若用户进程不调用`lab6_set_priority()`函数，则进程默认的`priority`是`0`，此时应使Stride算法退化为RR算法，否则`BIG_STRIDE / priority`会产生除零异常。

因此在`stride_pick_next()`中应判断`priority`是否为`0`：

```C
  if(p->lab6_priority == 0)
  	p->lab6_stride += BIG_STRIDE;
  else
  	p->lab6_stride += BIG_STRIDE / p->lab6_priority;
```