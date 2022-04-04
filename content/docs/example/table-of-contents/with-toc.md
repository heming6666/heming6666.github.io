---
title: Linux 进程调度
weight: 1
---
<!-- TOC -->
# Linux 进程调度

## 前言

在计算机科学中，调度就是一种将任务（Work）分配给资源的方法。任务可能是虚拟的计算任务，例如线程、进程或者数据流，这些任务会被调度到硬件资源上执行，例如：处理器 CPU 等设备。调度器或调度算法的设计与实现最终都会归结到一个问题上，即如何对有限的资源进行分配以实现资源利用率的最大化并满足特定的需求。

调度器是操作系统中的重要组件，操作系统中有进程调度器（Process Scheduler）、网络调度器（Network Scheduler）和 I/O 调度器（I/O Scheduler）等组件，本文介绍的是进程调度器。

进程调度器负责给系统中的所有进程分配有限的 CPU 时间资源。只有通过合理的调度算法，系统资源才能最大限度地发挥作用，多进程才会有并发执行的效果。

进程调度算法总是追求达到以下目标：

- 公平：保证每个进程得到合理的 CPU 时间，避免进程的饥饿现象。
- 高效：尽量充分使用 CPU，使 CPU 保持忙碌状态。
- 快速的响应时间：使交互用户的响应时间应尽可能短。
- 周转时间：使批处理用户等待输出的时间尽可能短。
- 吞吐量：单位时间内处理的进程数量尽可能多。

但是很显然，这几个目标是相互冲突的，不可能同时达到。因此只能在这几个方面进行取舍，从而确定自己的调度算法。

进程调度器将进程分为三类：

- 交互式进程(Interactive process)：这些进程经常与用户进行交互，因此进程不断地处于睡眠状态，等待用户输入。典型的应用比如命令行 shell、文本编辑程序。此类进程对系统响应时间要求比较高，否则用户会感觉系统反应迟缓。

- 批处理进程(Batch process)：这些进程一般在后台运行，不必与用户交互，需要占用大量的系统资源。但是能够忍受响应延迟。典型的批处理程序如编译程序、数据库搜索引擎等。

- 实时进程(Real-time process)：这些进程对调度延迟的要求最高，往往执行非常重要的操作，要求立即响应并执行。典型的实时程序比如视频播放软件、或飞机飞行控制系统，很明显这类程序不能容忍长时间的调度延迟。

根据进程的不同分类 Linux 采用不同的调度策略。

对于实时进程，采用 FIFO 或者 Round Robin 的调度策略。

对于普通进程，则需要区分交互式和批处理式的不同。传统 Linux 调度器提高交互式应用的优先级，使得它们能更快地被调度。而 CFS 和 RSDL 等新的调度器的核心思想是“完全公平”。这个设计理念不仅大大简化了调度器的代码复杂度，还对各种调度需求的提供了更完美的支持。

以下列出了 Linux 不同版本调度器的历史：

- 初始调度器 · v0.01 ~ v2.4
  - 由几十行代码实现，功能非常简陋；
  - 同时最多处理 64 个任务；
- O(n) 调度器 · v2.4 ~ v2.6
  - 调度时需要遍历全部任务；
  - 当待执行的任务较多时，同一个任务两次执行的间隔很长，会有比较严重的饥饿问题；
- O(1) 调度器 · v2.6.0 ~ v2.6.22
  - 通过引入运行队列和优先级数组实现 O(1) 的时间复杂度;
  - 使用本地运行队列替代全局运行队列增强在对称多处理器的扩展性；
  - 通过负载均衡保证多个运行队列中任务的平衡；
- 完全公平调度器 · v2.6.23 ~ 至今
  - 引入红黑树和运行时间保证调度的公平性；
  - 引入调度类实现不同任务类型的不同调度策略；

本文会详细介绍从最初的调度器到今天复杂的完全公平调度器（Completely Fair Scheduler，CFS）的演变过程。

## Linux 初始的调度算法

Linux 最初的进程调度器仅由 sched.h 和 sched.c 两个文件构成。你可能很难想象 Linux 早期版本使用只有几十行的 schedule 函数负责了操作系统进程的调度：

```c
void schedule(void) {
  int i,next,c;
  struct task_struct ** p;
  for(p = &LAST_TASK ; p > &FIRST_TASK ; --p) {
      ...
 }
 while (1) {
    c = -1;
    next = 0;
    i = NR_TASKS;
    p = &task[NR_TASKS];
    while (--i) {
    if (!*--p) continue;
    if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
      c = (*p)->counter, next = i;
    }
    if (c) break;
    for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
    if (*p)
      (*p)->counter = ((*p)->counter >> 1) + (*p)->priority;
  }
  switch_to(next);
}
```

无论是进程还是线程，在 Linux 中都被看做是 `task_struct` 结构体，所有的调度进程都存储在上限仅为 64 的数组中，调度器能够处理的进程上限也只有 64 个。

上述函数会先唤醒获得信号的可中断进程，然后从队列倒序查找计数器 `counter` 最大的可执行进程，`counter` 是进程能够占用的时间切片数量，该函数会根据时间切片的值执行不同的逻辑：

- 如果最大的 `counter` 时间切片大于 0，调用汇编语言的实现的 switch_to 切换进程；
- 如果最大的 `counter` 时间切片等于 0，意味着所有进程的可执行时间都为 0，那么所有进程都会获得新的时间切片；

Linux 操作系统的计时器会每隔 10ms 触发一次 `do_timer` 将当前正在运行进程的 `counter` 减一，当前进程的计数器归零时就会重新触发调度。

## O(n) 调度算法

### 数据结构：进程描述符

每个进程都有一个 `task_struct` 结构。该结构定义在 `<include/linux/sched.h>` 文件中，其中部分与进程调度相关的字段说明如下：

- `need_resched`：调度标志，决定是否调用 `schedule()` 函数。

- `counter`：进程处于可运行状态时所剩于的时钟节拍数。每次时钟中断到来时，`update_process_times()`对该值减 1。

  - 创建新进程时，`do_fork()`以下列方式设置 `current`(父)和 `p`(子)进程的 `counter` 字段：`current->counter >>=1; p->counter = current->counter`. 也就是说，父进程剩余的节拍数被分为两部分，一部分给父进程，一部分给子进程。这样做时为了防止通过 fork 子进程的方法无限制地使用 CPU 的时间。

- `rt_priority`：实时进程的实时优先级。取值范围 1-99。

- `nice`: 进程的静态优先级，它的值决定了 `counter` 的初值。`nice` 的取值范围是-20(优先级高)~19(优先级低)，缺省为 0。该值可通过 `nice` 系统调用改变。

- `policy`： 本进程的调度策：

  - `SCHED_RR` 和 `SCHED_FIFO` 用于实时进程。`SCHED_RR` 和 `SCHED_FIFO` 的调度策略在 `rt_priority` 不同的时候，都是谁的优先级高谁先执行，唯一的不同是相同优先级的处理：
  - `SCHED_RR` 是时间片轮转的实时进程。当多个进程具有同一优先级时，采用时间片轮转轮流调度运行。适用于实时性要求较高但每次运行耗时较长的进程。
  - `SCHED_FIFO` 是先入先出的实时进程，先占有 CPU 的进程会持续执行，直到退出或者阻塞的时候才会让出 CPU。也只有这时候，其他同优先级的实时进程才有机会执行。适应于实时性要求比较强、而每次运行的耗时又比较短的进程。
  - `SCHED_OTHER` 用于普通的分时进程。

  除了上面描述的三种调度策略，`policy` 成员也可以设定 `SCHED_YIELD` 的标记，它和调度策略无关，主要处理 `sched_yield` 系统调用的。

- `state`: 表示进程当前运行状态：
  - `TASK_RUNNING` ：该状态表示这个进程可被调度执行而成为当前进程，是进程表达了希望被调度运行的意愿，内核会将该进程的 `task_struct` 结构加入可运行队列。
  - `TASK_INTERRUPTIBLE` 和 `TASK_UNINTERRUPTIBLE` ：这两个状态都表示进程处于睡眠状态。前者表示浅度睡眠，可被信号唤醒；后者表示深度睡眠。`sleep_on` 和 `wake_up` 用于深度睡眠；而 `interruptible_sleep_on` 和 `interruptible_wake_up` 则用于浅度睡眠。
  - `TASK_ZOMBIE` 表示已退出而暂时没有被父进程收回资源的"僵尸"进程。
  - `TASK_STOPPED` 主要用于调试目的。进程接收到一个 `SIGSTOP` 信号后就将运行状态改成 `TASK_STOPPED` 而进入挂起状态，然后在接收到一个 `SIGCONT` 信号时又恢复继续运行。

    ![](../../pic/2020-05-19-12-46-41.png)

### 调度的时机

Linux 的调度程序是一个叫 `schedule()`的函数，由它来执行具体的调度算法。调用 `schedule()`的时机主要包括：

#### 直接调用

- 进程入睡时主动调用 `schedule()`：当现运行进程请求资源被阻塞时，会调用 `sleep_on()`或 `interruptible_sleep_on()`进入睡眠状态，这时会执行以下步骤：

  - 把当前进程 `current` 插入到合适的等待队列中。

  - 把当前进程 `current` 的状态修改为 `TASK_INTERUPTIBLE` 或 `TASK_UNINTERUPTIBLE`。

  - 调用 `schedule()` 函数。

  - 检查那个资源是否可用。如果不，转到第 2 步。

  - 一旦那个资源成为可用的，把 `current` 从等待队列中删除。

- 进程终止时主动调用 `schedule()`：当现运行进程终止时，会调用 exit() 终止运行，这时会主动调用 `schedule()` 函数。

- 设备驱动程序执行长而重复的任务时，主动调用 `schedule()`：驱动程序在每次循环中，都会去检查调度标志 `need_resched` 的值，如果必要，就调用 `schedule()`主动放弃 CPU。

- 创建新进程：在 `do_fork()`中也会调用 `schedule()` 函数。

#### 延迟调用

延迟调用是指当系统需要调度时，通过置现运行进程 `need_resched` 标志为 1。然后在从中断、异常、系统调用等从内核返回用户态时，对该标志进行检测，如果该标志为 1，则调用 `schedule()`。主要包括以下情况：

1. 在中断处理过程中，发现 `current` 用完了时间片，置 `need_resched` 标志为 1。

2. 一个进程被唤醒，且它的优先级比现运行进程更高，置 `need_resched` 标志为 1。

3. 当父进程 fork 子进程时，其时间片会均分到父子进程。
    - 如果只剩下一个 tick，这个 tick 会分配给子进程，而父进程的时间片则被清零，这时候等同于情况 1。
    - 否则，父子进程的时间片都不为 0，这时等同于情况 2。

4.  一个进程通过系统调用改变调度政策(`sched_setscheduler`)或表示礼让(`sched_yield`)时，会设置 `need_resched` 标志为 1。

### 如何计算优先级：godness()

调度算法的核心是在可运行队列链表中的所有进程中确定优先级最高的进程。goodness() 就是用来计算进程优先级。它接受两个输入参数：`prev`(前一个运行进程)和 p(要评估的进程)，返回一个整数值 c，表示进程 p 的“值得运行的程度(goodness)”。

`goodness()`函数将实时进程和普通进程区分计算。流程如下：

1.  如果该进程的 `policy` 被置 `SCHED_YIELD` 标志为，直接返回 -1。表示进程愿意“礼让”。
2.  如果该进程是 `SCHED_FIFO` 或 `SCHED_RR`，直接返回：`1000+ p->rt_priority`。可以看出，实时进程的优先级很高，且与 counter 和 nice 无关，至少为 1000，保证了实时进程会完全优先于普通进程的调度。
3.  如果该进程是 `SCHED_OTHER`：
4.  如果 `p->counter` 为 0，直接返回 0。即该进程已用完时间片。
5.  否则，返回 `p->counter + 20 - p->nice`。
6.  此外，在 2 的情况下，如果候选进程 p 是内核进程（无用户空间），或者 p 的用户空间与当前进程 `prev` 的用户空间相同，则返回 `p->counter + 20 - p->nice + 1`。因为如果 p 正好在 `prev` 之后运行，它们将使用同一页表。

### 调度流程：schedule()

可以将 `schedule()`函数大致分为以下三个部分：
1. 初始化部分。
2. 确定优先级最高的进程 `next`。
3. 完成进程切换。
4. 进程切换之后的操作

具体过程如下：

#### 初始化部分

1.  `current` 的值保存在 `prev` 局部变量中, 将 `prev->need_resched` 字段设为 0
2.  判断 `prev` 是不是用完了时间片的 `SCHED_RR` 进程。如果是，给 `prev` 分配一个新的时间片，并把它放到运行队列链表的队尾。
    ```c
    prev->counter = NICE_TO_TICKS(prev->nice));
    ```
3.  检查 `prev` 的状态，
    -  如果是 `TASK_INTERUPTIBLE` 且有未处理的信号，就唤醒:
   ` prev->state = TASK_RUNNING`;
    -  如果是 `TASK_RUNNING`，则接着转到 7。
    -  否则，将该进程从运行队列中删除。因此 `prev` 进程此时要么在因等待外部资源而入睡，又或者在 exit()系统调用中，已经将状态改为 TASK_ZOMBIE。

#### 确定优先级最高的进程 next

4.  接下来确定优先级最高的进程。首先初始化候选进程 `next` 为 idle 进程，优先级 c 为-1000。
5.  如果 `prev` 的状态是 `TASK_RUNNING`，则设置 `next` 设为 `prev`，c 设为 `prev` 的 goodness 值。
6.  遍历可运行队列，搜索优先级最高的就绪进程。在搜索的时候，调用 goodness 计算进程 p 的运行资格 weight。比较大小，更新 `next` 和 c。
7.  循环结束之后，检查 c 是否为 0。c 为 0 说明运行队列中的所有进程都用完了时间片,这时给所有进程重新分配新时间片。都赋值完之后，回到 4，重新挑选候选进程。
8.  如果最终的候选进程就是当前进程，直接返回。

#### 完成进程切换

9.  否则执行进程切换。首先将 kstat.context_swtch++,统计上下文切换的次数。
10. 检查候选进程 `next`：
    -  如果候选进程 `next` 是内核进程，则借用 `prev` 的 active_mm(指针赋值，active_mm 结构共享引用计数加 1)。
    -  如果候选进程 `next` 是用户空间进程，则执行 switch_mm 切换用户空间（将 mm_struct 结构中 pgd 成员所指向的页目录表的物理地址设置进 CR3）。
11. 检查 `prev` 进程，如果是内核进程，归还借用的 active_mm(指针赋为 NULL，active_mm 结构共享引用计数减 1)。
12. 执行 `switch_to()`进行真正的进程切换，即堆栈及寄存器状态切换。

#### 进程切换之后的操作

13. 执行`__schedule_tail()`，置 `prev->police` 的 `SCHED_YIELD` 为 0。
14. 函数返回。

### 如何重新分配时间片

在上一小节提到，当 c 为 0 时，说明可运行队列中没有实时进程，只有普通进程，且队列中所有普通进程的时间片都已用完，这时需要重新系统中所有进程的时间片（而不仅仅是可运行队列中的进程），计算方法为：

```c
for_each_task(p)
    p->counter = (p->counter >> 1) + NICE_TO_TICKS(p->nice);
```

宏 `NICE_TO_TICKS` 的定义如下。以 HZ 为 200 为例，每秒中断 200 次，那么一个时钟滴答 tick 为 5ms，`20 - nice` 的取值为[1 ,40]，缺省为 20，将 20 右移 1 位即除以 2 为 10，10 个滴答即 50ms。当时钟频率 HZ 越高，每个滴答所代表的时间越短，`NICE_TO_TICKS` 分配的滴答数越多，但最大只是 20 – nice 的值左移 2 位即乘以 4，最大值为 160，仍小于实时进程的 1000。

```c
#if HZ < 200
#define TICK_SCALE(x) ((x) >> 2)
#elif HZ < 400
#define TICK_SCALE(x) ((x) >> 1)
#elif HZ < 800
#define TICK_SCALE(x) (x)
#elif HZ < 1600
#define TICK_SCALE(x) ((x) << 1)
#else
#define TICK_SCALE(x) ((x) << 2)
#endif
#define NICE_TO_TICKS(nice) (TICK_SCALE(20-(nice))+1)
```

可以发现，经过重新计算后，那些不在可运行队列中的普通进程，会获得较高的时间配额，在将来的调度中会占一定的优势。但即使无数次更新方之后，`counter` 的值也不会超过两倍的 `NICE_TO_TICKS`，也不会超过实时进程的优先级。

### SMP 系统下的调度程序

Linux 为了支持对称多处理器(SMP)体系结构，必须对 Linux 的调度程序稍作修改。实际上，每个处理器运行它自己的 `schedule()`函数，但是，处理器间必须交换信息以提高系统性能。

#### 数据结构

##### schedule_data 结构体

如下所示，`schedule_data` 结构体用来很快的获得当前进程的描述符，每个 CPU 上都有一个该结构体。该结构体包含：
-  该 CPU 上现运行进程的描述符 `task_struct`。
-  现运行进程上台的时刻，即 `schedule()`是什么时候选 curr 作为运行进程。
```c
struct schedule_data{
    struct task_struct *curr;
    unsigned long last_schedule;
}
struct schedule_data aligned_data[NR_CPUS];
```

##### task_struct 进程描述符

除了在上文提到的一些与进程调度相关的字段外，`task_struct` 还包含了几个与 SMP 相关的字段，包括：
-  `processor`：表示该进程上一次运行在哪个 CPU 上。
-  `avg_slice`：表示该进程的平均时间片，即每次运行时间的期望值。

##### cacheflush_time 变量
`cacheflush_time` 变量表示对硬件高速缓存内容全部重写所需要花费的时间。这个值只是一个估值，大约不到 100 微秒。计算公式为：`以 kHZ 为单位的 CPU 频率*以 KB 大小为单位的缓存容量/5000`
在后面将看到，当现运行进程的 `avg_slice – (time – last_schedule) < cacheflush_time`，就不会执行进行的抢占。

#### SMP 系统下调度流程：schedule()

具体过程如下：
1. `current` 的值保存在 `prev` 局部变量中。`prev->processor` 的值存放在 `this_cpu` 局部变量中。`aligned_data[this_cpu]`的值存放在 `sched_data` 局部变量中。将 `prev->need_resched` 字段设为 0。

    ```c
    prev = current;
    this_cpu = prev->processor;
    sched_data = aligned_data[this_cpu];
    ```

2. 判断 `prev` 是不是用完了时间片的 `SCHED_RR` 进程。如果是，给 `prev` 分配一个新的时间片，并把它放到运行队列链表的队尾。
    ```c
    prev->counter = NICE_TO_TICKS(prev->nice));
    ```

3. 检查 `prev` 的状态:
-  如果是 `TASK_INTERUPTIBLE` 且有未处理的信号，就唤醒:
`prev->state = TASK_RUNNING`;
-  如果是 `TASK_RUNNING`，则接着转到 7。
-  否则，将该进程从运行队列中删除。因此 `prev` 进程此时要么在因等待外部资源而入睡，又或者在 exit()系统调用中，已经将状态改为 TASK_ZOMBIE。

##### 确定优先级最高的进程 next

4. 接下来确定优先级最高的进程。首先初始化候选进程 `next` 为 idle 进程，优先级 c 为-1000。
5. 如果 `prev` 的状态是 `TASK_RUNNING`，则设置 `next` 设为 `prev`，c 设为 `prev` 的 goodness 值。
6. 遍历可运行队列，搜索优先级最高的就绪进程。在搜索的时候，调用 goodness 计算进程 p 的运行资格 weight。比较大小，更新 `next` 和 c。在 goodness 函数中，检查进程的 processor 字段，并对最后在 this_cpu CPU 上执行的进程给与一定奖赏(PROC_CHANGE_PENALTY，通常为 15).
7. 循环结束之后，检查 c 是否为 0。c 为 0 说明运行队列中的所有进程都用完了时间片,这时给所有进程重新分配新时间片。都赋值完之后，回到 4，重新挑选候选进程。
8. 如果最终的候选进程就是当前进程，直接返回。

##### 完成进程切换

9. 否则执行进程切换。首先将 `kstat.context_swtch++`,统计上下文切换的次数。除此之外，做一些跟 CPU 相关的操作：
-  把 `sched_data->curr` 置为 `next`。
-  `next->has_cpu` 置为 1，`next->processo`r 置为 `this_cpu`。
-  在 t 局部变量中存放 `current` 时间标记寄存器的值，并执行：
    ```c
    this_slice = t – sched_data->last_schedule;
    sched_data->last_schedule = t;
    prev->avg_slice = (prev->avg_slice + this_slice) / 2
    ```
10. 检查候选进程 `next`：
-  如果候选进程 `next` 是内核进程，则借用 `prev` 的 active_mm(指针赋值，active_mm 结构共享引用计数加 1)。
-  如果候选进程 `next` 是用户空间进程，则执行 switch_mm 切换用户空间（将 mm_struct 结构中 pgd 成员所指向的页目录表的物理地址设置进 CR3）。
11. 检查 `prev` 进程，如果是内核进程，归还借用的 active_mm(指针赋为 NULL，active_mm 结构共享引用计数减 1)。
12. 执行 `switch_to()`进行真正的进程切换，即堆栈及寄存器状态切换。
进程切换之后的操作
13. 当内核返回到这时，说明之前的进程又被调度程序选中，然而，这时的 prev 局部变量并不指向我们开始描述 `schedule()`时所要替换出去的原来那个进程(我们假设记为 prev_old)，而是指向 prev_old 被调度时所要替换出去的那个进程(假设记为 prev_new)。如果 `prev`(即为 prev_new)还依然是可运行的，并且不是这个 CPU 的空任务，那么，对 `prev` 调用 reschedule_idle()函数。
14. 把 `prev` 的 has_cpu 字段清 0。
15. 函数返回。

#### SMP 系统下的 reschedule_idle()

当进程 p 变为可运行时，执行 reschedule_idle()函数决定进程是否应该抢占某一 CPU 上的当前进程。具体过程如下：
1. 如果 p 是实时进程，总会试图抢占，转到 3。
2. 如果有一个 CPU 上的当前进程满足下列两个条件，则立即返回(不试图抢占)：
    -  `cacheflush_time` 大于当前进程的平均时间片。防止高速缓存变得太“脏”。
    -  为了存取某一临界内核数据结构，p 和当前进程都需要全局内核锁。
3. 接下来执行 CPU 选择算法：
    -  如果 p->processor (即 p 最后运行的 CPU)是空闲的，选它。
    -  遍历所有 CPU，对其上正在运行的任务 tsk，计算以下差值：`goodness(tsk, p) - goodness(tsk, tsk)`, 如果这个差值为正，就选择差值最大的 CPU。
4. 如果选择了某个 CPU，给选中的 CPU 的正在运行进程的 `need_resched` 字段置 1，并向这个 CPU 发处理器间中断：`RESCHEDULE_VECTOR interprocessor interrupt`。

## O(1) 调度算法

### O(n) 调度算法缺陷

Linux2.4 之前的版本，用较为简单的调度算法实现了进程调度。但是该算法存在以下问题：
1. 算法复杂度问题。遍历运行队列的算法复杂度为 O(n)，意味着队列越长，选中一个进程所需要的时间就越长。此外，每次调度周期结束后，为每一个进程计算其时间片的过程太耗费时间。
2. 多处理器问题。多个处理器上的进程放在一个就绪队列中，使得这个就绪队列成为临界资源，为了实现内核同步机制，需要对其上自旋锁，降低了系统效率。
3. CPU 空转问题。在 `runqueue` 队列中的全部进程时间片被耗尽之前，系统总会处于这样一个状态：最后的一组尚存时间片的进程分分别调度到各个 CPU 上去。我们以 4 个 CPU 为例，T0 ～ T3 分别运行在 CPU0~CPU3 上。随着系统的运行，CPU2 上的 T2 首先耗尽了其时间片，但是这时候，其实 CPU2 上也是无法进行调度的，因为遍历 `runqueue` 链表，找不到适合的进程调度运行，因此它只能是处于 idle 状态。也许随后 T0 和 T3 也耗尽其时间片，从而导致 CPU0 和 CPU3 也进入了 idle 状态。现在只剩下最后一个进程 T1 仍然在 CPU1 上运行，而其他系统中的处理器处于 idle 状态，白白的浪费资源。唯一能改变这个状态的是 T1 耗尽其时间片，从而启动一个重新计算时间片的过程，这时候，正常的调度就可以恢复了。随着系统中 CPU 数目的加大，资源浪费会越来越严重。

    ![](../../pic/2020-05-19-12-57-48.png)

4.  SMP 亲和力问题。在一个新的周期开后，`runqueue` 中的进程时间片都是满满的，在各个 CPU 上调度进程的时候，它可选择的比较多，再加上调度器倾向于调度上次运行在本 CPU 的进程，因此调度器有很大的机会把上次运行的进程调度到同一个处理器上。但是随着 `runqueue` 中的进程一个个的耗尽其时间片，cpu 可选择的余地在不断的压缩，从而导致进程执行在一个和它亲和性不大的处理器（例如上次该进程运行在 CPU0，但是这个将其调度到 CPU1 执行，但是实际上该进程和 CPU0 的亲和性更大些）。
5.  实时进程调度性能问题。实时进程和普通进程挂在一个链表中，当调度实时进程的时候，我们需要遍历整个 `runqueue` 列表，扫描并计算所有进程的优先级，再从中选择出最终要调度的实时进程，在这过程中，一些时间片已经耗完的进程在不可能参与调度的情况下，依然会参与调度选择的过程。此外，整个 linux 内核不是抢占式内核，对于一些比较耗时的系统调用或者中断处理，必须返回用户空间才启动调度，大大降低了实时进程的调度性能。
6.  交互式普通进程的调度延迟问题。O（n）并不区分交互式进程和批处理进程，它只是奖励经常睡眠的那些进程。但是有些批处理进程也属于 IO-bound 进程，例如数据库服务进程，它本身是一个后台进程，对调度延迟不敏感，但是由于它需要和磁盘打交道，因此也会经常阻塞在 disk IO 上。对这样的后台进程进行动态优先级的升高其实是没有意义的，会增大其他交互式进程的调度延迟。

虽然 O（n）调度器存在不少的问题，但是社区的人还是基本认可这套算法的，因 此在设计新的调度器的时候并不是完全推翻 O（n）调度器的设计，而是针对 O（n）调度器的问题进行改进。

从以上分析中可以看出，单运行队列是影响调度性能的主要问题之一，因此改进运行队列就成为改进调度算法的入口点。

基于此，O(1)调度器为每个 CPU 设置一个运行队列，并且为每个运行队列再设置两个队列：活动队列和时间片过期队列。每个队列中的元素以优先级再进行分类，相同优先级的进程为一个队列，最多可以有 140 个优先级。为了快速选中要运行的进程，设置以优先级为序的队列位图，位图的每一位对应一个队列，只要队列中有一个可运行进程，该位置 1，否则置 0。这样，无需遍历所有队列，而只要遍历位图，找到有可运行进程的队列，该队列的第一个进程就是被选中的进程。该算法的复杂度为 O(1). 如图所示：

![](../../pic/2020-05-19-12-58-08.png)

通过将单链表变成多个链表，可以解决上述大部分问题：
1. 算法复杂度问题：O(1)调度器算法通过优先级位图，以及活动队列与过期队列指针交换，实现了 O(1)的复杂度，而不是遍历运行队列，并重新计算所有进程的时间片。
2. 多处理器问题：由于每个 CPU 都有一个运行队列，因此 O(1)调度器就不需要全局运行队列的自旋锁，而只需要把这个自旋锁放入到每个 CPU 的运行队列数据结构中，通过把一个大锁细分成小锁，可以大大降低调度延迟，提升系统响应时间。。
3. CPU 空转问题：O(1)调度器每个 CPU 都有一个运行队列，当一个进程的时间片耗尽，在被移动到过期数组之前，会重新计算其时间片，而不是等到一个调度周期结束再重新计算进程时间片，因此解决了 CPU 空转问题。
4. SMP 亲和力问题：O(1)调度器设置了较为合理的负载均衡算法，只有在需要平衡任务队列大小时才在 CPU 之间移动进程。
5. 实时进程与交互式调度性能问题：为了提高交互时进程和实时进程的响应时间，当前进程的时间片为 0 时，判断当前进程的类型，如果时交互式进程或实时进程，则重置其时间片并重新插入活动队列，否则插入过期队列，这样交互式进程和实时进程总能优先获得 CPU。然而当这些进程已经占用 CPU 时间超过一个固定值后，也会被移到过期队列中，避免其他进程产生饥饿现象。

### 数据结构

#### runqueue 数据结构

`runqueue` 可执行队列是调度程序中最基本的数据结构。定义于 `kernel/sched.c` 中。如上所述，每个 CPU 包含一个可执行队列；每个就绪进程都唯一地归属于某一个可执行队列。此外，可执行队列中还包含着每个 CPU 的调度信息。其包含的字段如下图所示：

![](../../pic/2020-05-19-12-58-42.png)

#### 优先级数组

如上所述，每个运行队列有活动队列和过期队列两个优先级数组，每个数组是一个 prio_array 类型的结构体。优先级数组使得该调度算法复杂度为 O(1)。

![](../../pic/2020-05-19-12-59-23.png)

1.  计数器 nr_active 是一个计数器，保存可运行进程数目。
2.  bitmap 是优先级位图数组。其中 BITMAP_SIZE 是位图数组的大小，类型为 unsigned long 长整型，长 32 位，每一位包含一个优先级，140 个优先级需要 5 个长整型数表示。bitmap 一开始所有的位都被置 0。当某个进程状态变为 `TASK_RUNNING` 时，对应的位被置 1。这样，查找系统中最高的优先级就变成了查找位图中被设置的第一个位。
3.  queue 是优先级链表数组，一个链表对应一种优先级。每个链表包含了该 CPU 上相应优先级的全部可运行进程。其中 MAX_PRIO 定义了系统拥有的优先级个数，默认为 140。

#### 进程描述符

与 linux 2.4 的进程描述符有些差异，其与调度相关的字段如下：

![](../../pic/2020-05-19-12-59-47.png)
![](../../pic/2020-05-19-12-59-57.png)

### 进程的优先级

#### 普通进程

##### 静态优先级

普通进程的静态优先级保存在 static_prio 成员中，取值范围是 100（优先级最高）～ 139（优先级最低），分别对应 nice 值的-20 ～ 19。静态优先级本质上决定了进程的基本时间片，对应公式如下：

![](../../pic/2020-05-19-13-00-33.png)

由该公式得到一些普通进程优先级的典型值如下：

![](../../pic/2020-05-19-13-00-46.png)

##### 动态优先级

在实际调度的时候使用的是动态优先级。普通进程的动态优先级保存在进程描述符的 prio 成员中。取值范围是 100（优先级最高）～ 139（优先级最低），和静态优先级一致。动态优先级根据以下公式得出：

![](../../pic/2020-05-19-13-01-10.png)

其中，bonus 取值范围 0~10，值小于 5 表示惩罚，大于 5 表示奖赏。bonus 的取值与进程的平均睡眠时间相关，如下表。需要说明的是，平均睡眠时间是进程在睡眠状态所消耗的平均纳秒数，但不是对过去时间的求平均值操作。例如，在 `TASK_INTERRUPTIBLE` 状态与在 `TASK_UNINTERRUPTIBLE` 状态所计算出的平均睡眠时间是不同的。而且，平均睡眠时间永远不会大于 1s.

![](../../pic/2020-05-19-13-01-21.png)

平均睡眠时间也被调度程序用来确定一个给定进程是交互式进程还是批处理进程。如果满足下列公式，则被看作是交互式进程：

![](../../pic/2020-05-19-13-01-32.png)

其中，静态优先级/4-28 被称为交互时的 δ。可以推出，高优先级的进程比低优先级的进程更容易成为交互式进程。例如，最高静态优先级(100)的进程，当他的 bonus 值超过 2，即睡眠时间超过 200ms 时，就被看作是交互式进程。

#### 实时进程

实时进程的实时优先级，存在进程描述符的 `rt_priority` 成员中，取值范围是 1（优先级最低）～ 99（优先级最高）。需要注意的是，当系统调用 nice()和 setpriority()用于基于时间片轮转的实时进程时，不改变实时进程的优先级，而会改变其基本时间片的长度。也就是说，基于时间片轮转的实时进程的基本时间片的长度与实时进程的优先级无关，而依赖于进程的静态优先级，它们的关系同普通进程下的公式一样。

普通进程采用复杂的公式计算动态优先级，而实时进程不计算动态优先级，保证了给定优先级别的实时进程总能抢占优先级比它低的进程。

### 调度程序所使用的函数

调度程序使用几个函数来完成调度工作，其中最重要的函数说明如下：

#### scheduler_tick()

每次时钟节拍到来时，scheduler_tick()被调用，执行以下步骤：
1. 把转换为纳秒的 TSC 的当前值存到本地运行队列的 timestamp_last_tick 字段。这个时间戳是从 sched_clock()函数得到。
2. 检查当前进程是不是本地 CPU 的 swapper 进程，如果是，则检查本地运行队列除了 swagger 进程外，是不是还有另外的可运行进程，如果是，就设置当前进程的 `TIF_NEED_RESCHED` 字段，以强迫进行重新调度。之所以会出现这种情况，是因为如果内核支持超线程技术，那么只要一个逻辑 CPU 运行队列中的所有进程都比另一个逻辑 CPU 上已经在执行的进程优先级低得多，而两个逻辑 CPU 是对应同一个物理 CPU 的，因此，前一个逻辑 CPU 就可能空闲，即使它的运行队列中也有可运行的进程。执行完上述检查后，直接跳到第 7 步，因为不需要更新 swagger 进程的时间片。
3. 检查 current->array 是否指向本地运行队列的活动链表，如果不是，说明进程已经过期但还没有被替换，则设置当前进程的 `TIF_NEED_RESCHED` 字段，以强迫进行重新调度。然后直接跳到第 7 步。
4. 获得 this_rq()->lock 自旋锁
5. 根据进程不同类型执行不同操作：
-  如果当前进程是 FIFO 的实时进程，则什么也不做，跳到 6。
-  当前进程是基于时间片轮转的实时进程：递减当前进程时间片，如果时间片已经用完，则：
    - 重填进程的时间片：`current->time_slice = task_timeslice()`
    - `current->first_time_slice= 0`。该字段是在 fork()系统调用服务例程中的 `copy_process()`中设置，并在进程的第一个时间片用完时清 0。
    - 调用 `set_tsk_need_resched()`设置进程的 `TIF_NEED_RESCHED` 字段。
    iv. 把当前进程移到当前的运行队列尾部。
-  当前进程是普通进程：递减当前进程时间片，如果时间片已经用完：
    - 将当前进程从活动队列(this_rq()->active)中移除。
    - 调用 `set_tsk_need_resched()` 设置进程的 `TIF_NEED_RESCHED` 字段。
    - 更新当前进程的动态优先级。`current->prio = effective_prio(current)`.
    - 重填进程的时间片：`current->time_slice = task_timeslice()`
    - `current->first_time_slice` 清 0。
    - 如果` this_rq()->expired_timestamp` 字段为 0（表示过期队列为空），把当前进程的时钟节拍 `jiffies` 赋值给 `this_rq()->expired_timestamp`。
    - 把当前进程插入活动队列或过期队列：

    ```c
    if (!TASK_INTERACTIVE(current) || EXPIRED_STARVING(this_rq()))
        enqueue_task(current, this_rq()->expired);
    else
        enqueue_task(current, this_rq()->active);
    ```

    其中, `TASK_INTERACTIVE` 宏用于识别一个进程是不是交互式进程。`EXPIRED_STARVING` 宏负责检查过期队列中的进程是否处于饥饿状态：如果已经有相对较长时间没有发生数组切换了，那么再把当前的进程放置到活动数组，则会加重过期队列中进程的饥饿状态。

    否则，如果时间片没有用完，检查当前进程的剩余时间片是否太长：

    ```c
    if (TASK_INTERACTIVE(p) &&
    !((task_timeslice(p) – p->time_slice) %TIMESLICE_GRANULARITY(p)) &&
    (p->time_slice >= TIMESLICE_GRANULARITY(p)) &&
    (p->array == rq->active)) {
        list_del(&current->run_list);
        list_add_tail(&current -> run_list, this_rq()->active->queue+current->prio);
        set_tsk_need_resched(p);
    }
    ```

    其中，宏 TIMESLICE_GRANULARITY 产生两个数的乘积给当前进程的 bonus，其中一个数为系统中 CPU 的数量，另一个为成比例的常量。基本上，具有高静态优先级的交互式进程，其时间片被分成大小为 TIMESLICE_GRANULARITY 的几个片段，以使这些进程不会独占 CPU。

6. 释放 this_rq()->lock 自旋锁。
7. 调用 `rebalance_tick()`函数，保证不同 CPU 的运行队列包含数量基本相同的可运行进程。

从第 5 步中可以看出，对于 O(1)调度器，时间片的重新赋值是分散处理的，在各个进程耗尽其时间片之后立刻进行的。修正了 O(n)调度器一次性的遍历系统所有进程，重新为时间片赋值的过程。

#### 唤醒：try_to_wake_up()

该函数通过把进程状态设置为 `TASK_RUNNING`，并调用 `activate_task()`函数将此进程放入对应的可运行队列中来唤醒睡眠或停止的进程。

该函数接受的参数有：
1. 被唤醒进程的描述符指针 `p`.
2. 可以被唤醒的进程状态掩码(`state`)。
3. 一个标志(`syn`)，用来禁止被唤醒的进程抢占本地 CPU 上正在运行的进程。

该函数执行以下操作：
1. 禁本地中断，并获得最后执行该进程的 CPU 的运行队列的锁。
2. 检查进程状态 `p->state == state`。如果不是，直接跳到 9 终止函数。
3. 如果 `p->arrray != NULL` ,说明该进程已经属于某个运行队列，跳到 8.
4. 确定目标 CPU：
    -  如果有空闲的 CPU，就选空闲的 CPU。
    -  如果先前执行进程的 CPU 的工作量远小于本地 CPU 的工作量，选前者。
    -  如果进程最近被执行过，就选这个老的运行队列。
    - 如果把进程迁移到本地 CPU 可以缓解 CPU 之间的不平衡，则选本地 CPU。
5. 如果进程处于 `TASK_UNINTERRUPTIBLE` 状态，则递减目标运行队列的 `nr_uninterruptible` 字段，并把 `p->activated` 字段置为-1。
6. 调用 `activate_task()`函数，执行：
    -  调用 `sched_clock()`获取以纳秒为单位的当前时间戳。如果目标 CPU 不是本地 CPU，就要补偿本地时钟中断的偏差：`now=(shced_clock()–this_rq()->timesamp_last_tick)+rq->timestamp_last_tick`
    -  调用 `recalc_task_prio()`。
    -  设置 `p->activated` 字段的值。
    -  使用 now 设置 `p->timestamp` 字段。
    - 把进程描述符插入活动队列，且 `rq->nr_running++`。
7. 如果目标 CPU 不是本地 CPU，或者没有设置 sync 标志，则，如果该进程优先级更高 `p->prio > rq->curr->prio`，就调用 `resched_task()`抢占 `rq->curr`。
    -  单处理器系统，设置 `rq->curr` 进程的 `TIF_NEED_RESCHED` 标志。
    -  多处理器系统，如果 `TIF_NEED_RESCHED` 旧值为 0，且目标 CPU 没有轮询进程 `TIF_NEED_RESCHED` 标志的值，则发送处理器间中断 IPI，强制目标 CPU 重新调度。
8. 把当前进程 `p->state` 字段设为 `TASK_RUNNING`。
9. 开 `rq` 运行队列的锁，打开本地中断。
10. 如果成功唤醒返回 1，否则返回 0.

#### 计算动态优先级：recalc_task_prio()

`recalc_task_prio()` 函数更新进程的平均睡眠时间和动态优先级。

该函数接受的参数有：
1. 进程描述符指针 `p`。
2. 当前时间戳 `now`。

该函数执行以下操作：
1. 把 `min(now – p->timestamp, 109)`的结果赋值给局部变量 `sleep_time`，表示进程消耗在睡眠状态的纳秒数。如果超过 1 秒，就设为 1 秒.
2. 如果 `sleep_time` 不大于 0，直接跳到 8.
3. 检查进程是不是内核线程、进程是否从 `TASK_UNINTERRUPTIBLE` 状态（即 `p->activated` 为-1）被唤醒、进程连续睡眠的时间是否超过给定的睡眠时间期限。如果这三个条件都满足，把 `p->sleep_avg` 字段设置位相当于 900 个时钟节拍的值（用最大平均睡眠时间减去一个标准进程的基本时间片长度获得的一个经验值）。然后跳到 8.
4. 计算进程原来的平均睡眠时间的 `bonus` 值。如果 `10-bonus>0`,则把 `sleep_time` 乘以 `10-bonus`。所以原来的 `p->sleep_avg` 越小，bonus 值越小，`10-bonus` 越大，`sleep_time` 越大，最终的 `p->sleep_avg` 增加的就越快。
5. 如果进程处于 `TASK_UNINTERRUPTIBLE` 状态而且不是内核线程：
    -  检查平均睡眠时间 `p->sleep_avg` 是否大于等于进程的睡眠时间极限。如果是，把 `sleep_time` 置为 0，直接跳到 6.
    -  如果 `sleep_time + p->sleep_avg` 大于等于睡眠时间极限，把 `p->sleep_avg` 设为睡眠时间极限并把 `sleep_time` 置为 0
    通过对进程平均睡眠时间的轻微限制，函数不会对睡眠时间很长的批处理进程给与过多的奖赏。
6. 把 `sleep_time` 加到进程的平均睡眠时间 `p->sleep_avg` 上。
7. 检查 `p->sleep_avg` 是否超过 1000 个时钟节拍，如果是，就置为 1000 个时钟节拍。
8. 更新进程的动态优先级。`p->prio = effective_prio(p)`.

可以看到，在评估用户交互指数上，O(n)调度器仅仅考虑了睡眠进程的剩余时间片，而 O(1)调度器的“平均睡眠时间”算法考虑了更多的因素：在 cpu 上的执行时间、在 `runqueue` 中的等待时间、睡眠时间、睡眠时候的进程状态（是否可被信号打断），什么上下文唤醒（中断上下文唤醒还是在进程上下文中唤醒），因此 O(1)调度器更好的判断了进程是否属于交互式进程。

#### 调度流程：

`schedule()` 与 `O(n)` 调度器类似，可以将 `schedule()` 函数大致分为以下四个部分：

1.  初始化部分。
2.  确定优先级最高的进程 `next`。
3.  完成进程切换。
4.  进程切换之后的操作
    具体过程如下：
    初始化部分
5.  禁用内核抢占。`current` 的值保存在 `prev` 局部变量中，本地 CPU 的运行队列保存在 rq 局部变量中。
    ```c
    preempt_disable();
    prev = current;
    rq = this_rq();
    ```
6.  保证 `prev` 不占用大内核锁。通过进程切换会自动释放和重新获取大内核锁。
    ```c
    if (prev->lock_dept >= 0)
        up(&kernel_sem);
    ```
7.  计算 `prev` 所用的 CPU 时间片长度：
    ```c
    now = sched_clock();
    run_time = now – prev->timesamp;
    if (run_time > 1000000000)
        run_time = 1000000000
    run_time /= (CURRENT_BONUS(PREV) ? : 1)
    ```
    `run_time` 用来限制进程对 CPU 的使用，最多 1 秒。不过，当进程有较长睡眠时间时，`CURRENT_BONUS()`返回值越大，`run_time` 就会被降低。这是对较长平均睡眠时间的奖赏。
8.  关本地中断，获得运行队列的自旋锁：`spin_lock_irq(&rq->lock)`
9.  检查 `prev` 是不是一个正在被终止的进程：
    ```c
    if (prev->flags & PF_DEAD)
        prev->state = EXIT_DEAD;
    ```
10. 检查 `prev` 的状态，如果不是 `TASK_RUNNING` 可运行状态，而且没有在内核态被抢占：
    -  如果是 `TASK_INTERUPTIBLE` 且有未处理的信号，就让其变为可运行状态：`prev->state = TASK_RUNNING` 以唤醒这个进程。
    -  否则，调用 `deactivate_task()`函数从运行队列中删除 `prev` 进程,。同时，如果该进程状态是 `TASK_UNINTERUPTIBLE`，则 `rq->nr_uniterruptible++`.
    ```c
    rq->nr_running--;
    dequeue_task(p, p->array);
    p->array = NULL;
    ```
11. 检查运行队列中剩于的可运行进程数。如果有可运行进程，但是当前内核支持超线程技术，且可运行进程比在相同物理 CPU 的某个逻辑 CPU 上运行的兄弟进程优先级低，`p->sleep_avg`直接去执行 swapper 进程。
12. 如果没有可运行进程，函数调用 `idle_balance()`，从其他 CPU 迁移一些可运行进程到本地队列中。如果本地队列还是没有可运行进程，就重新调度空闲 CPU 的可运行进程。如果还是没有，则直接去执行 swapper 进程。
13. 到这里运行队列中一定有可运行进程。检查运行队列中是否至少有一个进程是活动的(`rq->active->nr_active>0`)。如果没有，交换活动队列和过期队列的指针。
##### 确定优先级最高的进程 next
14. 在优先级数组中查找第一个非 0 的位图对应的链表的第一个进程描述符，并赋值给 `next`。
15. 检查 `next->activate` 字段，该字段编码值表示进程在被唤醒时的状态，如下表：

    ![](../../pic/2020-05-19-13-04-34.png)

    如果 `next` 是一个普通进程，并且 activate 为 1 或 2，就把自从进程插入运行队列开始所经过的纳秒数加到进程的平均睡眠时间。但是 1 和 2 的情况还是有区别，在 2 的情况下，增加全部运行队列等待时间，在 1 的情况下，只增加等待时间的部分。这是因为交互式进程更可能被异步事件(如键盘)而不是同步事件唤醒。

    ```c
    if (next->prio >= 100 && next-> activate > 0){
        unsigned long long delta = now – next->timestamp;
        if ( next-> activate == 1)
        delta = (delta * 38) / 128;
        array = next->array;
        dequeue_task(next, array);
        recalc_task_prio(next, next->timestamp + delt- ;
        enqueue_task(next, array);
    }
    next-> activate = 0;
    ```

##### 完成进程切换
12. 如果最终的候选进程就是当前进程，释放自旋锁，不做进程切换，直接结束。
13. 否则执行进程切换：
    ```c
    next->timestamp = now;
    rq->nr_switches ++;
    rq->current = next;
    prev = context_switch(rq, prev, next)
    ```
    其中 context_switch()函数建立 next 的地址空间。
    -  如果 `next` 是内核进程，则借用 `prev` 的 active_mm。
        ```c
        if ( ! next_mm ){
        next->active_mm = prev->active_mm;
        atomic_inc(&prev->active_mm->mm_count);
        enter_lazy_tlb( prev->active_mm, next)
        }
        ```
    -  如果 `next` 是普通进程，则执行 `switch_mm` 切换用户空间，把虚拟内存从上一个进程映射切换到新进程中。
        ```c
        if ( next_mm ){
        switch_mm( prev->active_mm, next->mm, next);
        }
        ```
14. 如果 `prev` 是内核进程或正在退出的进程：
    ```c
    if ( ! prev_mm ){
        rq->prev_mm = prev->active_mm;
        prev->active_mm = NULL;
    }
    ```
15. 调用 `switch_to()`进行真正的进程切换，从上一个进程的处理器状态切换到新进程的处理器状态，包括保存、恢复栈信息和寄存器信息。
    ```c
    switch_to( prev, next, prev);
    ```

##### 进程切换之后的操作

16. 当内核返回到这时，说明之前的进程又被调度程序选中，然而，这时的 `prev` 局部变量并不指向我们开始描述 `p->sleep_avg`时所要替换出去的原来那个进程(我们假设记为 prev_old)，而是指向 prev_old 被调度时所要替换出去的那个进程(假设记为 prev_new)。进程切换后的第一部分指令是：
    ```c
    barrier();
    finish_task_switch( prev);
    ```
    其中 `finish_task_switch(prev)` 函数如下：
    ```c
    mm = this_rq()->prev_mm;
    this_rq()->prev_mm = NULL;
    prev_task_flags = prev->flags;
    spin_unlock_irq(& this_rq()->lock);
    if (mm)
        mmdrop(mm)
    if (prev_task_flags & PF_DEAD)
        put_task_struct( prev);
    ```
    其中 `mmdrop()`减少内存描述符的使用计数器。如果减到了 0，释放与页表相关的所有描述符和虚拟存储区。`put_task_struct()`释放进程描述符使用计数器，并撤销所有其余对该进程的引用。

17. `p->sleep_avg`函数最后一部分代码如下。包括在需要的时候重新获得大内核锁，重新启用内核抢占，并检查是否一些其他的进程已经设置了当前进程的 `TIF_NEED_RESCHED`。如果是，则整个 `p->sleep_avg`函数重新执行，否则函数结束。
    ```c
    prev = current;
    if (prev->lock_depth >= 0)
        __reacquire_kernel_lock();
    preempt_enable_no_resched();
    if (test_bit(TIF_NEED_RESCHED, &current_thread_info() -> flags))
        goto need_resched;
    return;
    ```

#### 多处理器系统中运行队列的平衡

从 Linux2.6.7 版本开始，Linux 提出一种基于“调度域”概念的复杂的运行队列平衡算法，从而能够容易适应各种已有的多处理器体系结构。并提供了以下函数：

1. `rebalance_tick()`函数会在每一次时钟节拍到来时由 `scheduler_tick()`调用，负责周期性、在需要的时候调用 `load_balance()`函数。

2. `load_balance()`函数检查调度域是否处于严重的不平衡状态，如果是，将会尝试调用 `move_task()`函数把一些进程从一个运行队列迁移到另一个运行队列。

3. `move_task()`函数负责把进程从源运行队列迁移到本地运行队列。

其中比较重要的 `load_balance()`函数可简单描述为如下操作：

1、 调用 `find_busiest_queue()`，找到最繁忙的可运行队列，即该队列中的进程数目最多。如果没有哪个可运行队列中进程的数目比当前队列中的数目多 25%或更多，就返回 NULL，并且 `load_balance()`函数也返回。否则返回最繁忙的可运行队列。

2、 从最繁忙的运行队列中选择一个优先级数组以便抽取进程，最好是过期数组，因为那里面的进程已经相当较长一段时间没有运行了，很可能不在 CPU 的高速缓存中。如果过期数组为空，那就只能选活动数组。

3、 找到含有进程并且优先级最高的链表。

4、 分析找到的所有这些优先级相同的进程，选择一个不是正在执行，也不会因为 CPU 相关性而不可移动，并且不在高速缓存中的进程。如果有进程满足以上条件，调用 `move_task()`将其从最繁忙的队列迁移到当前队列。

5、 只要可运行队列之间仍然不平衡，就重复上面两个步骤，最终达到平衡。此时，解除对当前运行队列的锁定，从 `load_balance()`返回。

### 抢占

#### 用户抢占

用户抢占是指在内核即将返回用户空间的时候，如果 `need_resched` 标志被设置，会导致 `p->sleep_avg`被调用，此时就发生了用户抢占。与延迟调用小节描述的一样，用户抢占在从系统调用或中断处理程序返回用户空间时发生。

#### 内核抢占

在不支持内核抢占的内核中，内核代码可以一直执行，直到完成返回用户空间或者明显的阻塞为止。也就是说，调度程序没办法在一个内核任务正在执行的时候发起调度。在 2.6 版本的内核中，引入了内核抢占能力，只要重新调度是安全的，即，只要没有持有锁，那么正在执行的代码就是可重新导入的，内核就可以在任何时间抢占正在执行的任务。

为了支持内核抢占，为每个进程的` thread_info` 引入了 `preempt_count` 计数器。该计数器初始值为 0，每当使用锁的时候加 1，释放锁的时候减 1.当该值为 0 时，表示内核可以抢占。

因此，从中断返回内核空间的时候，内核会检查 `need_resched` 和 preempt_count 的值。如果 `need_resched` 被设置，且 preempt_count 为 0，说明有一个更为重要的任务需要执行并且可以安全的抢占，此时，调度程序就会被调用。此外，如果当前进程持有的所有锁都被释放了，此时会去检查 `need_resched` 是否被设置，如果是就调用调度程序。

如果内核中的进程被阻塞了，或它显示地调用 `p->sleep_avg`，内核抢占就显式地发生，这种形式的内核抢占一直都是支持的，因为无需额外的逻辑来保证内核可以安全的被抢占。

## 楼梯调度算法与 RSDL 调度算法

O(1)调度器区分交互式进程和批处理进程的算法与以前虽大有改进，但仍然在很多情况下会失效。有一些著名的程序(如 `fiftyp.c`, `thud.c`, `chew.c`, `ring-test.c`)总能让该调度器性能下降，导致交互式进程反应缓慢。

为了解决这些问题，大量难以维护和阅读的复杂代码被加入 Linux2.6.0 的调度器模块，虽然很多性能问题因此得到了解决，可是另外一个严重问题始终困扰着许多内核开发者。那就是代码的复杂度问题。这些不足催生了 Con Kolivas 的楼梯调度算法 SD，为调度器设计提供了一个新的思路。之后的 RSDL 和 CFS 都基于 SD 的许多基本思想。

### 楼梯调度算法
O(1)调度器算法的主要复杂性来自动态优先级的计算，调度器根据平均睡眠时间和一些很难理解的经验公式来修正进程的优先级以及区分交互式进程。这样的代码很难阅读和维护。

楼梯调度算法(staircase scheduler)抛弃了动态优先级的概念，而采用了一种完全公平的思路。其思路虽然简单，但是实验证明它对应交互式进程的响应比 O(1)调度器更好，而且极大地简化了代码。

和 O(1)调度器一样，楼梯算法也同样为每一个优先级维护一个进程队列，并将这些队列组织在 active 数组中。当选取下一个被调度进程时，SD 算法也同样从 active 数组中直接读取。
与 O(1)算法不同在于，当进程用完了自己的时间片后，并不是被移到 expire 数组中。而是被加入 active 数组的低一优先级列表中，即将其降低一个级别。不过请注意这里只是将该任务插入低一级优先级任务列表中，任务本身的优先级并没有改变。当时间片再次用完，任务被再次放入更低一级优先级任务队列中。就像一部楼梯，任务每次用完了自己的时间片之后就下一级楼梯。

任务下到最低一级楼梯时，如果时间片再次用完，它会回到初始优先级的下一级任务队列中。比如某进程的优先级为 1，当它到达最后一级台阶 140 后，再次用完时间片时将回到优先级为 2 的任务队列中，即第二级台阶。不过此时分配给该任务的 `time_slice` 将变成原来的 2 倍。比如原来该任务的时间片 `time_slice` 为 10ms，则现在变成了 20ms。基本的原则是，当任务下到楼梯底部时，再次用完时间片就回到上次下楼梯的起点的下一级台阶。并给予该任务相同于其最初分配的时间片。总结如下：设任务本身优先级为 P，当它从第 N 级台阶开始下楼梯并到达底部后，将回到第 N+1 级台阶。并且赋予该任务 N+1 倍的时间片。

以上描述的是普通进程的调度算法，实时进程还是采用原来的调度策略，即 FIFO 或者 Round Robin。

楼梯算法能避免进程饥饿现象，高优先级的进程会最终和低优先级的进程竞争，使得低优先级进程最终获得执行机会。

对于交互式应用，当进入睡眠状态时，与它同等优先级的其他进程将一步一步地走下楼梯，进入低优先级进程队列。当该交互式进程再次唤醒后，它还留在高处的楼梯台阶上，从而能更快地被调度器选中，加速了响应时间。

从实现角度看，SD 基本上还是沿用了 O(1)的整体框架，只是删除了 O(1)调度器中动态修改优先级的复杂代码；还淘汰了 expire 数组，从而简化了代码。它最重要的意义在于证明了完全公平这个思想的可行性。

### RSDL 调度算法

RSDL（The Rotating Staircase Deadline Schedule）也是由 Con Kolivas 开发的，它是对 SD 算法的改进。核心的思想还是“完全公平”。没有复杂的动态优先级调整策略。

RSDL 重新引入了 expire 数组。它为每一个优先级都分配了一个 “组时间配额”， 我们将组时间配额标记为 Tg；同一优先级的每个进程都拥有同样的"优先级时间配额"，本文中用 Tp 表示，以便于后续描述。

当进程用完了自身的 Tp 时，就下降到下一优先级进程组中。这个过程和 SD 相同，在 RSDL 中这个过程叫做 minor rotation。请注意 Tp 不等于进程的时间片，而是小于进程的时间片。下图表示了 minor rotation。进程从 priority1 的队列中一步一步下到 priority140 之后回到 priority2 的队列中，这个过程如下图左边所示，然后从 priority 2 开始再次一步一步下楼，到底后再次反弹到 priority3 队列中。

![](../../pic/2020-05-19-13-05-55.png)

在 SD 算法中，处于楼梯底部的低优先级进程必须等待所有的高优先级进程执行完才能获得 CPU。因此低优先级进程的等待时间无法确定。RSDL 中，当高优先级进程组用完了它们的 Tg(即组时间配额)时，无论该组中是否还有进程 Tp 尚未用完，所有属于该组的进程都被强制降低到下一优先级进程组中。这样低优先级任务就可以在一个可以预计的未来得到调度。从而改善了调度的公平性。这就是 RSDL 中 Deadline 代表的含义。
进程用完了自己的时间片 `time_slice` 时（下图中 T2），将放入 expire 数组中它初始的优先级队列中(priority 1)。

![](../../pic/2020-05-19-13-06-07.png)

当 active 数组为空，或者所有的进程都降低到最低优先级时就会触发 major rotation：。Major rotation 交换 active 数组和 expire 数组，所有进程都恢复到初始状态，再一次从新开始 minor rotation 的过程。

和 SD 同样的道理，交互式进程在睡眠时间时，它所有的竞争者都因为 minor rotation 而降到了低优先级进程队列中。当它重新进入 RUNNING 状态时，就获得了相对较高的优先级，从而能被迅速响应。

## CFS 完全公平调度算法

CFS 是最终被内核采纳的调度器。它从 RSDL/SD 中吸取了完全公平的思想，不再跟踪进程的睡眠时间，也不再企图区分交互式进程。它将所有的进程都统一对待，这就是公平的含义。CFS 的算法和实现都相当简单，众多的测试表明其性能也非常优越。

按照作者 Ingo Molnar 的说法："CFS 百分之八十的工作可以用一句话概括：CFS 在真实的硬件上模拟了完全理想的多任务处理器"。在“完全理想的多任务处理器“下，每个进程都能同时获得 CPU 的执行时间。当系统中有两个进程时，CPU 的计算时间被分成两份，每个进程获得 50%。然而在实际的硬件上，当一个进程占用 CPU 时，其它进程就必须等待。这就产生了不公平。

假设 `runqueue` 中有 n 个进程，当前进程运行了 10ms。在“完全理想的多任务处理器”中，10ms 应该平分给 n 个进程(不考虑各个进程的 nice 值)，因此当前进程应得的时间是(10/n)ms，但是它却运行了 10ms。所以 CFS 将惩罚当前进程，使其它进程能够在下次调度时尽可能取代当前进程。最终实现所有进程的公平调度。下面将介绍 CFS 实现的一些重要部分，以便深入地理解 CFS 的工作原理[5]。

### CFS 如何选取下一个要调度的进程

CFS 抛弃了 active/expire 数组，而使用红黑树选取下一个被调度进程。所有状态为 RUNABLE 的进程都被插入红黑树。在每个调度点，CFS 调度器都会选择红黑树的最左边的叶子节点作为下一个将获得 cpu 的进程。

### tick 中断

在 CFS 中，tick 中断首先更新调度信息。然后调整当前进程在红黑树中的位置。调整完成后如果发现当前进程不再是最左边的叶子，就标记 `need_resched` 标志，中断返回时就会调用 scheduler()完成进程切换。否则当前进程继续占用 CPU。从这里可以看到 CFS 抛弃了传统的时间片概念。Tick 中断只需更新红黑树，以前的所有调度器都在 tick 中断中递减时间片，当时间片或者配额被用完时才触发优先级调整并重新调度。

### 红黑树键值计算

理解 CFS 的关键就是了解红黑树键值的计算方法。该键值由三个因子计算而得：一是进程已经占用的 CPU 时间；二是当前进程的 nice 值；三是当前的 cpu 负载。

进程已经占用的 CPU 时间对键值的影响最大，其实很大程度上我们在理解 CFS 时可以简单地认为键值就等于进程已占用的 CPU 时间。因此该值越大，键值越大，从而使得当前进程向红黑树的右侧移动。另外 CFS 规定，nice 值为 1 的进程比 nice 值为 0 的进程多获得 10%的 CPU 时间。在计算键值时也考虑到这个因素，因此 nice 值越大，键值也越大。

在本文中，我们将为每个进程维护的变量称为进程级变量，为每个 CPU 维护的称作 CPU 级变量，为每个 `runqueue` 维护的称为 `runqueue` 级变量。

CFS 为每个进程都维护两个重要变量：`fair_clock` 和 `wait_runtime`。进程插入红黑树的键值即为 `fair_clock – wait_runtime`。
-  `fair_clock` 从其字面含义上讲就是一个进程应获得的 CPU 时间，即等于进程已占用的 CPU 时间除以当前 `runqueue` 中的进程总数；
-  `wait_runtime` 是进程的等待时间。它们的差值代表了一个进程的公平程度。该值越大，代表当前进程相对于其它进程越不公平。

对于交互式任务，`wait_runtime` 长时间得不到更新，因此它能拥有更高的红黑树键值，更靠近红黑树的左边。从而得到快速响应。
红黑树是平衡树，调度器每次总最左边读出一个叶子节点，该读取操作的时间复杂度是 `O(LgN)`。

### 调度器管理器

为了支持实时进程，CFS 提供了调度器模块管理器。各种不同的调度器算法都可以作为一个模块注册到该管理器中。不同的进程可以选择使用不同的调度器模块。2.6.23 中，CFS 实现了两个调度算法，CFS 算法模块和实时调度模块。对应实时进程，将使用实时调度模块。对应普通进程则使用 CFS 算法。Ingo Molnar 还邀请 Con Kolivas 可以将 RSDL/SD 写成一个调度算法模块。

### CFS 源代码分析

每次时钟中断会调用 `scheduler_tick()`函数。它首先更新 `runqueue` 级变量 clock；然后调用 CFS 的 tick 处理函数 `task_tick_fair()`。`task_tick_fair` 主要工作是调用 `entity_tick()`。函数 `entiry_tick` 源代码如下：

```c
static void entity_tick(struct cfs_rq _cfs_rq, struct sched_entity _curr)
{
  struct sched_entity _next;
  dequeue_entity(cfs_rq, curr, 0);
  enqueue_entity(cfs_rq, curr, 0);
  next = **pick_next_entity(cfs_rq);
  if (next == curr)
      return;
  __check_preempt_curr_fair(cfs_rq, next, curr,
  sched_granularity(cfs_rq));
}
```

首先调用 `dequeue_entity()`函数将当前进程从红黑树中删除，再调用 `enqueue_entity()`重新插入。这两个动作就调整了当前进程在红黑树中的位置。`_pick_next_entity()`返回红黑树中最左边的节点，如果不再是当前进程，就调用`_check_preempt_curr_fair`。该函数设置调度标志，当中断返回时就会调用 `p->sleep_avg`进行调度。
函数 `enqueue_entity()`的源码如下:

```c
enqueue_entity(struct cfs_rq _cfs_rq, struct sched_entity _se, int wakeup)
{
    update_curr(cfs_rq);
    if (wakeup)
        enqueue_sleeper(cfs_rq, se);
    update_stats_enqueue(cfs_rq, se);
    __enqueue_entity(cfs_rq, se);
}
```

它的第一个工作是更新调度信息。然后将进程插入红黑树中。其中 `update_curr()`函数是核心。完成调度信息的更新:

```
static void update_curr(struct cfs_rq _cfs_rq)
{
    struct sched_entity _curr = cfs_rq_curr(cfs_rq);
    unsigned long delta_exec;
    if (unlikely(!curr))
        return;
    delta_exec = (unsigned long)(rq_of(cfs_rq)->clock - curr->exec_start);
    curr->delta_exec += delta_exec;
    if (unlikely(curr->delta_exec > sysctl_sched_stat_granularity)) {
        __update_curr(cfs_rq, curr);
        curr->delta_exec = 0;
    }
    curr->exec_start = rq_of(cfs_rq)->clock;
}
```

该函数首先统计当前进程所获得的 CPU 时间，`rq_of(cfs_rq)->clock` 值在 tick 中断中被更新，`curr->exec_start` 就是当前进程开始获得 CPU 时的时间戳。两值相减就是当前进程所获得的 CPU 时间。将该变量存入 `curr->delta_exec` 中。然后调用`__update_curr()`:

```c
__update_curr(struct cfs_rq _cfs_rq, struct sched_entity _curr)
{
    unsigned long delta, delta_exec, delta_fair, delta_mine;
    struct load_weight _lw = &cfs_rq-load;
    unsigned long load = lw->weight;
    delta_exec = curr->delta_exec;
    schedstat_set(curr->exec_max, max((u64)delta_exec, curr->exec_max));
    curr->sum_exec_runtime += delta_exec;
    cfs_rq->exec_clock += delta_exec;
    if (unlikely(!load)) return;
    delta_fair = calc_delta_fair(delta_exec, lw);
    delta_mine = calc_delta_mine(delta_exec, curr->load.weight, lw);
    if (cfs_rq->sleeper_bonus > sysctl_sched_min_granularity) {
        delta = min((u64)delta_mine, cfs_rq->sleeper_bonus);
        delta = min(delta, (unsigned long)((long)sysctl_sched_runtime_limit - curr->wait_runtime));
        cfs_rq->sleeper_bonus -= delta;
        delta_mine -= delta;
    }
    cfs_rq->fair_clock += delta_fair;
    add_wait_runtime(cfs_rq, curr, delta_mine - delta_exec);
}
```

`__update_curr()`的主要工作就是更新前面提到的 `fair_clock` 和 `wait_runtime`。这两个值的差值就是后面进程插入红黑树的键值。变量 `Delta_exec` 保存了前面获得的当前进程所占用的 CPU 时间。函数 `calc_delta_fair()`根据 cpu 负载（保存在 lw 变量中），对 `delta_exec` 进行修正，然后将结果保存到 `delta_fair` 变量中，随后将 `fair_clock` 增加 `delta_fair`。函数 `calc_delta_mine()`根据 `nice` 值（保存在 `curr->load.weight` 中）和 cpu 负载修正 `delta_exec`，将结果保存在 `delta_mine` 中。根据源代码中的注释，`delta_mine` 就表示当前进程应该获得的 CPU 时间。

随后将 `delta_fair` 加给 `fair_clock` 而将 `delta_mine-delta_exec` 加给 `wait_runtime`。函数 `add_wait_runtime` 中两次将 `wait_runtime` 减去 `delta_mine-delta_exec`。由于 `calc_delt_xx()`函数对 `delta_exec` 仅做了较小的修改，为了讨论方便，我们可以忽略它们对 `delta_exec` 的修改。最终的结果可以近似看成 `fair_clock` 增加了一倍的 `delta_exec`，而 `wait_runtime` 减小了两倍的 `delta_exec`。因此键值 `fair_clock-wait_runtime` 最终增加了一倍的 `delta_exec` 值。键值增加，使得当前进程再次插入红黑树中就向右移动了。

### CFS 小结

以上的讨论看出 CFS 对以前的调度器进行了很大改动。用红黑树代替优先级数组；用完全公平的策略代替动态优先级策略；引入了模块管理器；它修改了原来 Linux2.6.0 调度器模块 70%的代码。结构更简单灵活，算法适应性更高。相比于 RSDL，虽然都基于完全公平的原理，但是它们的实现完全不同。相比之下，CFS 更加清晰简单，有更好的扩展性。

CFS 还有一个重要特点，即调度粒度小。CFS 之前的调度器中，除了进程调用了某些阻塞函数而主动参与调度之外，每个进程都只有在用完了时间片或者属于自己的时间配额之后才被抢占。而 CFS 则在每次 tick 都进行检查，如果当前进程不再处于红黑树的左边，就被抢占。在高负载的服务器上，通过调整调度粒度能够获得更好的调度性能。

在最新版本的 CFS 实现中，内核使用虚拟运行时间 vruntime 替代了等待时间，但是基本的调度原理和排序方式没有太多变化。

## 参考资料

[1] DanielP.Bovet, MarcoCesati, Bovet,等. 深入理解 LINUX 内核[M]. 中国电力出版社, 2001.

[2] DanielP.Bovet, MarcoCesati, Bovet,等. 深入理解 LINUX 内核(第三版)[M]. 中国电力出版社, 2007.

[3] 陈莉君, 康华. Linux 操作系统原理与应用[M]. 清华大学出版社, 2006.

[4] RobertLove, 洛夫, 陈莉君, et al. Linux 内核设计与实现[M]. 机械工业出版社, 2006.

[5] Linux 调度器发展简述
<https://www.ibm.com/developerworks/cn/linux/l-cn-scheduler/>

[6] O(n)、O(1)和 CFS 调度器
<http://www.wowotech.net/process_management/scheduler-history.html>

[7] 调度系统设计精要
<https://draveness.me/system-design-scheduler/>

[8] 进程调度之 6：进程的调度与切换
<https://my.oschina.net/u/3857782/blog/1857556>