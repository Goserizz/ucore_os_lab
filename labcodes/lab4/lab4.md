# **Lab4 Report**

### **练习1 分配并初始化一个进程控制块**

- ### **实现过程**

  实现过程非常简单，只需要将proc_struct结构体中相应的成员进行初始化即可。特别地，pid应初始化为-1，表示还未分配合法的pid值，state应初始化为PROC_UNINIT，表示该进程还未初始化完成，即该进程还不具备能够运行的条件，cr3应初始化为boot_cr3，即内核地址空间的页目录表起始地址。

  ```C
  static struct proc_struct *
  alloc_proc(void) {
      struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
      if (proc != NULL) {
        proc->state = PROC_UNINIT;
        proc->pid = -1;
        proc->runs = 0;
        proc->kstack = 0;
        proc->need_resched = 0;
        proc->parent = NULL;
        proc->mm = NULL;
        memset(&proc->context, 0, sizeof(struct context));
        proc->tf = NULL;
        proc->cr3 = boot_cr3;
        proc->flags = 0;
        memset(proc->name, 0, sizeof(char) * (PROC_NAME_LEN + 1));
        }
      return proc;
  }
  ```

- ### **请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？**

  context用于保存当前的数据寄存器中的内容，如eax、ebx等，tf用于保存中断现场的堆栈信息，如段寄存器cs，下一条指令位置eip等。这两个成员变量用于在该进程从运行状态转变到等待或者就绪状态时保存中断时的信息，在进程从就绪转变到运行状态时还原上一次中断时的状态。



### **练习2 为新创建的内核线程分配资源**

- ### **实现过程**

  1.调用alloc_proc将该进程的proc_struct，若分配失败，则直接跳转到fork_out返回

  2.将该进程的父进程设置为当前进程current

  3.调用setup_kstack为该进程分配内核栈空间，若失败，则跳转至bad_fork_cleanup_proc，释放proc_struct所占空间后返回

  4.调用copy_mm初始化proc_struct的mm成员变量，该阶段中进程全部处于内核态，内核堆栈空间不会出现缺页异常，所以其实copy_mm函数什么也没做。若失败，则跳转至bad_fork_cleanup_kstack，释放该进程之前分配的内核空间、该进程描述符(proc_struct)所占空间后返回。

  5.调用copy_thread初始化该进程的tf和context

  6.调用get_pid为该进程分配pid值。

  7.调用hash_proc和list_add将该进程的描述符加入hash_list和proc_list。值得注意的时，get_pid应在hash_proc之前完成，因为hash值的计算需要用到pid。

  8.此时初始化已基本完成，将所有进程数nr_process++，并调用wakeup_proc设置进程状态为runnable，最后返回该进程的pid。

  ```C
  int do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
      int ret = -E_NO_FREE_PROC;
      struct proc_struct *proc;
      if (nr_process >= MAX_PROCESS) {
          goto fork_out;
      }
      ret = -E_NO_MEM;
  
  if((proc = alloc_proc()) == NULL)
      goto fork_out;
  
  proc->parent = current;
  if(setup_kstack(proc) != 0)
      goto bad_fork_cleanup_proc;
  if(copy_mm(clone_flags, proc) != 0)
      goto bad_fork_cleanup_kstack;
  copy_thread(proc, stack, tf);
  proc->pid = get_pid();
  hash_proc(proc);
  list_add(&proc_list, &proc->list_link);
  nr_process ++;
  wakeup_proc(proc);
  ret = proc->pid;
  
  fork_out:
      return ret;
  bad_fork_cleanup_kstack:
      put_kstack(proc);
  bad_fork_cleanup_proc:
      kfree(proc);
      goto fork_out;
  }
  ```

- ### **请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。**

  可以。通过分析get_pid函数，其大致的算法流程概括如下：用last_pid表示当前查找的pid值，将proc_list中的描述符循环查找，每当查找到的进程的pid与last_pid相等，就将last_pid++，用next_safe表示last_pid在这次循环中能够++到的最大值，即当前查询过的pid值中最小但又大于last_pid的值，若last_pid达到了next_safe的值，则重新开始循环，因为next_safe只记录当前最小，之前记录过的可能重复的pid都已经丢失了，所以要重新开始循环，我觉得或许将next_safe改为有序链表结构更好，记录每一个比当前查询值大的已存在的pid值，若last_pid增加到了表头的值，便把表头删除，此时可以不用从头再开始循环。

- ### **练习3 阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。**

  1.local_intr_save(intr_flag);....local_intr_restore(intr_flag); 用于停止暂停中断，防止在中断切换的关键时刻整出夭蛾子；

  2.将当前进程指针current指向该进程；

  3.load_esp0(next->kstack + KSTACKSIZE); 在fork中调用copy_thread函数时将中断帧信息存放在了next->kstack + KSTACKSIZE处，此时load_esp0将该地址中esp成员变量位置对应的值载入esp寄存器中，即完成了栈的切换。(此处其实可以直接使用next->tf，将更加直观易懂)

  4.lcr3(next->cr3); 用于还原cr3寄存器，即页目录表基地址。

  5.switch_to(&(prev->context), &(next->context)); 用于存储当前的寄存器的值，然后还原要切换到的进程的寄存器的值，switch_to的本体在switch.S中，可以很容易的看出，switch_to就是先将当前各寄存器的值保存在了&prev->context对应的地址空间中，然后将&next->context地址空间中对应的寄存器的值加载到寄存器当中，值得注意的时，起初按张调用函数时形参压栈的规则，from的值存储在4(%esp)中，to的值存储在8(%esp)中，但后来通过popl 0(%eax)将eip的值存到prev->context中，esp的值便减少了4，所以此时to的值的位置为4(%esp)，有语句movl 4(%esp), %eax。

### 实现过程

实现过程非常简单，只需要将proc_struct里相应的成员初始化或分配空间即可。特别地，pid应初始化为-1，表示还未分配pid值，state应初始化为PROC_UNINIT，表示还未完成初始化，即该进程还没有准备好开始运行，cr3应初始化为boot_cr3，即内核地址空间的页目录项起始位置。

```C
static struct proc_struct * alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
        proc->state = PROC_UNINIT;
        proc->pid = -1;
        proc->runs = 0;
        proc->kstack = 0;
        proc->need_resched = 0;
        proc->parent = NULL;
        proc->mm = NULL;
        memset(&(proc->context), 0, sizeof(struct context));
        proc->tf = NULL;
        proc->cr3 = boot_cr3;
        proc->flags = 0;
        memset(proc->name, 0, PROC_NAME_LEN);
    }
    return proc;
}
```