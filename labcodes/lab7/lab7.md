# Lab7 实验报告

## 实验涉及知识点

- 信号量的概念、实现与实际应用
- 管程与条件变量的概念、实现与实际应用(实现上与原理课讲解有很大不同)
- 哲学家就餐问题及其实现

## OS原理中很重要，但实验中未出现的知识点

- 生产者-消费者问题及其实现
- 读者-写者问题及其实现

## 练习1：理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题
### 1.1 请给出内核级信号量的设计描述，并说明其大致执行流流程。

#### 设计描述

- ucore中信号量的数据结构如下（`kern/sync/sem.h`）：

```C
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```

其中`value`代表信号量的值，而`wait_queue`是该信号量对应的等待队列（等待队列的相应实现在`kern/sync/wait.h`中）。

- 信号量的重要函数实现在`kern/sync/sem.c`中：
  - **down函数：对应P操作**
  - 观察__down()函数，可看出它遵循如下步骤，与OS原理一致：

  - - 屏蔽中断，若信号量>0，则将信号量减1，完成P操作。
    - 否则说明资源已占满，则将当前线程加入等待队列，恢复中断，并重新调度。
    - 待满足条件，重新唤醒、返回此线程后，在互斥环境下将当前线程出队。
    - 最后检查等待原因是否为“等待信号量”（`WT_KSEM`），排除异常情况。完成P操作。

  - **up函数：对应V操作**
  - 观察__up()函数，可看出它遵循如下步骤，与OS原理基本一致：
    - 屏蔽中断，若该信号量的等待队列为空，则将信号量加1，完成V操作。
    - 若有等待队列，则将队首的线程唤醒。

  - **trydown函数：辅助函数**
  - 屏蔽中断，尝试将信号量减1，若成功则返回true，否则返回false。

#### 执行流程
在`check_sync.c`中实现了信号量机制下哲学家问题的测试。

每个哲学家有三种状态：`THINKING、HUNGRY、EATING`；

两种行为：尝试取得左右两把叉子（`phi_take_forks_sema`）、放下左右两把叉子（`phi_put_forks_sema`）；

一个信号量：`s[i]`，只会取0或1，用于让拿到叉子的哲学家正常就餐，让得不到叉子的哲学家进入阻塞状态。具体而言，信号量的变化流程如下：

- `s[i]`的初始值均为0。
- 对于尝试获得叉子的哲学家i，若`phi_test_sema()`成功，则执行`up(&s[i])`让`s[i]`加一变为1，此时代表这个哲学家成功拿到叉子。接下来执行的`down(&s[i])`让`s[i]`减一变回0，正常返回外层函数`philosopher_using_semaphore()`，开始吃饭。
- 若测试不成功，则下面的`down(&s[i])`让该线程陷入阻塞（`s[i]`仍为0），等待之后资源满足时被唤醒。
- 对于要放下叉子的哲学家k，他将在自己的线程中帮助两个相邻哲学家LEFT、RIGHT做“取叉尝试”`phi_test_sema()`，尝试成功时，说明LEFT或RIGHT可以就餐了，就会执行`up(&s[i])`。由于之前线程i处于阻塞，故此时up()将唤醒线程i（`s[i]`依旧为0），并结束down函数，正常返回外层函数`philosopher_using_semaphore()`，开始吃饭。

为了保证临界区代码（修改`state_sema[i]`、取叉尝试`phi_test_sema()`）不出错，又加了一个临界区的锁：mutex。它也是一个二值信号量。


### 1.2 请给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

#### 设计方案

由于内核中已有完整的信号量机制，故用户态进程的信号量机制，只需将内核态的信号量机制进行封装，并提供系统调用接口。

具体而言，可以在内核中维护一个信号量的数组，用户态进程通过`sem_init()`申请信号量，并得到信号量的id（下标编号）。P、V操作也可以由相应的系统调用，进一步调用`down()`和`up()`来修改内核中的信号量，或实现进程的阻塞与唤醒。

#### 与内核级信号量机制的异同

相同之处在于都实现了完整的信号量机制，包括信号量的数据结构与相关P、V操作的函数。

不同之处在于内核级可以直接调用down、up函数来进行信号量的操作，但用户级需要通过系统调用来间接进行内核中的信号量操作。

## 练习2 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题
### 2.1 请给出内核级条件变量的设计描述，并说其大致执行流流程。
#### 设计描述

- ucore中管程和条件变量的数据结构如下：

```C
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;

typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```

这与OS原理中基本一致。

值得注意的是管程结构中的`next`信号量。当一个线程（B）调用`cond_signal()`唤醒其他线程（A）时，它自己会睡在管程的`next`信号量上。这样，被唤醒的线程（A）就能很方便地唤醒`next`上的线程（B）。

采用`next`机制的管程实际上就是Hoare管程。

- 条件变量的重要函数实现在`kern/sync/monitor.c`中：

  - **cond_signal函数：**

  `cond_signal()`函数的主要作用是唤醒睡在条件变量`cvp`上的线程，同时让当前线程自己睡在`next`信号量上，即退出管程，将管程的使用权交给刚唤醒的进程。

  - **cond_wait函数：**

  `cond_wait()`函数的主要作用是看是否有进程在`next`信号量的等待队列中，如果有，则唤醒它；否则，自己退出管程。同时，让自己睡在条件变量`cvp`上。两个动作不能颠倒顺序，否则会因为down的阻塞而发生死锁。

  - **monitor_init函数：**

  主要作用是为管程做初始化，创建n个条件变量。

#### 执行流程

在`check_sync.c`中实现了条件变量机制下哲学家问题的测试。

这里的管程一共有5个条件变量（`mt->cv[i]`），以及管程入口的信号量互斥锁`mt->mutex`。

哲学家的依旧有两个动作：取叉子和放叉子。

- 取叉子时，当前进程先进入管程，并测试是否可以得到叉子。如果能得到，则将状态改为EATING，`cond_signal`将不做任何操作；如果得不到，则退出管程或唤醒睡在`next`上的进程，并让自己睡在`cv[i]`条件变量上。
- 放叉子时，当前进程先进入管程，并将状态改为THINKING。此后分别帮左右的哲学家测试是否可以拿到叉子。此时如果左邻居取叉子成功，则`cond_signal`会将它唤醒，并让陷入THINKING的自己睡在`next`上。

我的实现与答案基本一致。

### 2.2 请给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。
#### 方案简述

基本思路与用户态的信号量机制一样，是对内核级条件变量进行封装。

具体来说，在内核中维护条件变量数组，通过`get_cond`系统调用可获取条件变量的id，再使用`signal`和`wait`系统调用对条件变量进行相应的操作。注意这里需要传递的参数不再是`condvar_t*`，而变为了条件变量的id，这是因为用户空间的地址与内核不同。

#### 与内核级条件变量机制的异同

相同之处在于都实现了完整的条件变量机制。

不同之处在于内核级可以直接调用cond_signal、cond_wait函数来进行条件变量的操作，但用户级需要通过系统调用来间接进行内核中的条件变量操作。

### 2.3 能否不用基于信号量机制来完成条件变量？如果不能，请给出理由；如果能，请给出设计说明和具体实现。
能。

只需按照OS原理中的代码，在一定的互斥锁机制的基础上实现即可。伪代码如下：

```C++
class Condition {
    WaitQueue q;
}

Condition::Wait(lock){
    local_intr_save(intr_flag);
    release(lock);
    Add this thread t  to q;
    local_intr_restore(intr_flag);
    schedule(); //need mutex
    local_intr_save(intr_flag);
    remove this thread t  from q;
    local_intr_restore(intr_flag);
    require(lock);
}

Condition::Signal(){
    local_intr_save(intr_flag);
    if (q.notEmpty()){
        Remove one thread t from q's head;
        wakeup(t);
    }
    local_intr_restore(intr_flag);
}

```
wait中的前两条语句——关中断和释放锁的顺序不能够互换，否则释放锁之后，另外一个进程可能在当前进程被放入等待队列之前执行signal操作，当前进程无法收到，从而导致其一直在等待队列中睡眠。

从伪代码看出，条件变量的wait操作与信号量的down操作十分类似，条件变量的signal操作与信号量的up操作十分类似。区别在于，条件变量在signal时，若等待队列为空会忽略本次操作，而信号量up操作时，若队列为空会把计数器加一，从而让之后的down操作直接通过。同时，条件变量还需要维护一个锁。
