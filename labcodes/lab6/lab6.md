# lab6 Report

### **练习1: 使用 Round Robin 调度算法**

- **理解并分析sched_class中各个函数指针的用法，并结合Round Robin 调度算法描ucore的调度执行过程**

  > init：用于初始化调度器
  >
  > enquene：用于将一个进程加入RUNNABLE队列中
  >
  > dequene：用于将一个进程从RUNNABLE队列中取出
  >
  > pick_next：从RUNNABLE队列中选择下一占用CPU执行的进程
  >
  > proc_tick：产生一个时钟中断时对相应进程的操作

  Round Robin是时间片算法，且在ucore中没有优先级。具体的调度过程如下：

  1. 当一个进程进入RUNNABLE状态后，调用enquene将其加入RUNNABLE队列中，然后为这个进程赋一个初始的时间片time_slice；
  2. 每产生一次时钟中断，便调用一次以current(当前进程)为参数的proc_tick，在proc_tick中将当前进程的剩余时间片time_slice减1，若time_slice减到了0，则标记该进程的need_resched为1，表示当前进程的时间片已用完，需要进行调度；
  3. 当前进程的need_resched为1时，就会调用schedule函数，schedule函数首先调用enquene将当前进程加入RUNNABLE队列中，然后再调用pick_next选取下一个占用cpu进行执行的进程，然后调用dequene将它从RUNNABLE队列中取出。
  4. schedule函数在最后会调用proc_run函数，proc_run负责进程的切换，其核心函数switch_to完成了最终的堆栈、寄存器值等的切换。

- **简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计**

  在run_quene结构体增加多个list_entry_t成员变量来表示多级队列，每一个队列有一个确定的时间片，优先级越高的队列时间片越短，但进程调度时高优先级的进程被执行的可能性大于低优先级的进程。

  每一个进程进来时都是最高优先级，若一个进程在当前优先级分配的时间片中没有完成其手头的任务(进入等待状态)，则下调其优先级。这样导致的结果是IO密集型进程处于高优先级，CPU密集型处于低优先级。这种结果是令人接受的，因为往往IO密集型需要的是进程间的并行性，CPU密集型需要的是更高的CPU利用率，而在这种调度算法中，高优先级时间片短，切换速度快，低优先级时间片长，切换速度慢，符合人的期望。

### **练习2: 实现 Stride Scheduling 调度算法**

- **BIG_STRIDE的值**

  由stride算法的原理，很容易证明不等式STRIDE_MAX - STRIDE_MIN <= PASS_MAX，而由于priority最小可以取1，PASS = BIG_STRIDE / priority，所以PASS_MAX最大可以取到BIG_STRIDE，于是不等式转变为STRIDE_MAX - STRIDE_MIN <= BIG_STRIDE。

  BIG_STRIDE是当stride值溢出时用于判断怎样相减才是合法的比较方式，若两个stride相减大于了BIG_STRIDE，可称为是假的差值，因为它并没有真正的反映出两个进程的stride的差距，而是的到了相反的结果。要求得BIG_STRIDE则需要找到溢出产生的最小的假的STRIDE_MAX - STRIDE_MIN。可以想向两个进程的stride同为UINT(2^32) - 1，此时一个进程passBIG_STRIDE步，得到stride为BIG_STRIDE - 1，则有不等式(UINT - 1) - (BIG_STRIDE - 1) > BIG_STRIDE，即满足该不等式，才能分辨真假差值。求得BIG_STRIDE最大为UINT/2即0x7fffffff。

- **实现过程**

  stride算法基本上是在default_sched_stride.c中实现的，完成init、enquene、dequene、pick_next、proc_tick五个关键函数

  init：将run quene的run_pool设为NULL，不使用skew_heap_init的原因是，在merge时调用比较函数会产生问题。把进程数proc_num设为0.

  ```C
  static void
  stride_init(struct run_queue *rq) {
       rq->lab6_run_pool = NULL;
       rq->proc_num = 0;
  }
  ```

  enquene：直接调用skew_heap_insert，然后需要设置入队的进程的时间片，若它自带的时间片合法，则沿用，若不合法，则设为max_time_slice。最后若进程的优先级为0，即没有预设优先级，需要将其设置为1，防止后面计算pass时出错。

  ```C
  static void
  stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
       rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &proc->lab6_run_pool, proc_stride_comp_f);
       if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
          proc->time_slice = rq->max_time_slice;
       }
       proc->rq = rq;
       rq->proc_num ++;
  
       if(proc->lab6_priority == 0)
            proc->lab6_priority = 1;
  }
  ```

  dequene：调用skew_heap_remove

  ```C
  static void
  stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
       rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &proc->lab6_run_pool, proc_stride_comp_f);
       rq->proc_num --;
  }
  ```

  pick_next：skew heap的merge操作保证了任何一个节点，都比其后代的值(stride)要小，所以在任何时刻，根节点(即rq->lab6_run_pool)则表示当前所有进程中stride最小的节点，使用宏le2proc即可得到，还需要修改它的stride+=pass，pass用BIG_STRIDE/priority来计算。

  ```C
  static struct proc_struct *
  stride_pick_next(struct run_queue *rq) {
       struct proc_struct *p = le2proc(rq->lab6_run_pool, lab6_run_pool);
       p->lab6_stride += BIG_STRIDE / p->lab6_priority;
       return p;
  }
  ```

  proc_tick：将当前进程的时间片-1tick，减到0就需要标记need_resched为1.

  ```C
  static void
  stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
       proc->time_slice --;
       if(proc->time_slice == 0)
            proc->need_resched = 1;
  }
  ```

- **其他改动**

  proc.c:alloc_proc中需要初始化lab6中新增的一些成员变量

  ```C
  // static struct alloc_struct * alloc_proc(void)
  proc->rq = NULL;
  proc->time_slice = 0;
  skew_heap_init(&proc->lab6_run_pool);
  proc->lab6_stride = 0;
  proc->lab6_priority = 0;
  ```

  trap.c:trap_dispatch需要重新设置对时钟中断的操作

  ```C
  //static void trap_dispatch(struct trapframe *)
  case IRQ_OFFSET + IRQ_TIMER:
  	ticks ++;
  	sched_class_proc_tick(current);
  ```

  