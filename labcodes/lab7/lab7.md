# **Lab7 Report**

### **练习1: 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题**

- ### 内核级信号量的设计描述及实现

  semaphore_t成员变量

  > 信号量结构体有2个成员变量，int value用于表示该资源是否能够被获取，若大于0，则表示可以获取，若小于等于0，则需要等待资源释放。另一个成员变量wait_quene_t wait_quene用于存放该信号量的等待进程，wait_quene_t的本质是一个list_entry_t，作为进程等待队列的接口。

  初始化函数sem_init

  > 初始化value值，并调用wait_quene_init初始化wait_quene，实质是调用list_init初始化wait_quene旗下的wait_head。

  P()操作down

  > down函数调用\_\_down，并传入等待原因WT_KSEM(等待信号量)，由\_\_down函数完成进程对信号量成员变量的访问。首先需要关掉中断，使时钟中断失效，防止访问信号量成员变量时其他进程修改。此时若value大于0，则代表资源充足，可以访问，将value—后开启中断即可返回。若value等于0，表示资源处于繁忙状态，就需要创建一个存放等待信息的结构体wait_t，并把它加入信号量的wait_quene。然后开启中断，进入进程调度。等到进程再度进入执行时，说明等待的资源已经满足，需要在禁用中断的环境下将之前创建的wait_t结构体从wait_quene中删除，然后进程即可进入临界区执行代码。

  V()操作up

  > up函数调用\_\_up，并传入等待原因WT_KSEM，由\_\_up函数完成后续操作，在禁用中断的环境下，查看等待队列是否为空，若为空，则说明没有等待进程，只需要将value++变为1即可。若由等待进程，则找到队首的等待进程，调用wakeup_wait唤醒等待进程，wakeup_wait的具体实现是将该进程从等待队列中删除，进而调用wakeup_proc将进程结构体的状态置为PROC_RUNNABLE，将其wait_state清空，调用sched_class_enquen将其加入进程调度的等待队列。

  try_down函数：

  > try_down函数就是尝试着申请资源，若value大于0，则申请，若value等于0，也不等待，是个佛系函数。

- ### **给用户态进程/线程提供信号量机制的设计方案**

  由于信号量机制的实现需要开关中断这种原子操作，用户态没有权限进行这种操作，所以可以新建一个系统调用来实现信号量机制。用户在需要进行同步互斥操作时调用相应的系统调用，操作系统为其完成后续对信号量的操作。



### **练习2 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题**

### **条件变量的设计描述及实现**

condvar_t结构变量

> 条件变量结构体由3个成员变量组成，semaphore_t sem用信号量来表示该资源的忙闲状态和其等待队列，int count表示等待进程数，monitor_t *owner表示该条件变量归属的管程结构体指针。

条件变量的signal操作：cond_signal(condvar_t *cvp)

> 首先将owner->next_count++，count表示等待结束signal操作的进程数。由于结构变量的资源状态和等待队列都由信号量来实现，所以可以很方便的调用up函数来释放资源，在对owner->next调用down函数使进程进入等待，因为管程中只允许同时运行一个进程，唤醒等待进程后该进程通过信号量owner->next进入睡眠，保证互斥性。

```C
void 
cond_signal (condvar_t *cvp) {
    //LAB7 EXERCISE1: 2016012205
    cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);  
    if(cvp->count > 0){
        cvp->owner->next_count ++;
        up(&cvp->sem);
        down(&cvp->owner->next);
        cvp->owner->next_count --;
    }
    cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```

条件变量的wait操作：cond_wait(condvar_t *cvp)

> 首先将count++。然后判断owner->next_count 是否大于0，这一步是判断是否还有等待结束signal操作的进程，若有则通过释放owner->next信号量来获得条件变量的访问权，否则释放之前申请的管程互斥信号量owner->mutex来获取条件变量的访问权。然后对条件变量的资源进行请求，请求完毕后将count—，表示等待进程数-1。

```C
void
cond_wait (condvar_t *cvp) {
    //LAB7 EXERCISE1: 2016012205
    cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
    cvp->count ++;
    if(cvp->owner->next_count > 0)
        up(&cvp->owner->next);
    else
        up(&cvp->owner->mutex);
    down(&cvp->sem);
    cvp->count --;
    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```

- ### 能否不使用信号量来实现条件变量

  条件变量也需要一个value来表示资源的忙闲状态，一个wait_quene来表示等待的进程队列，即使不直接使用信号量来实现，其实现方法也必然与信号量大同小异

  