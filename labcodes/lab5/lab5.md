# **Lab5 Report**

### **练习1 加载应用程序并执行**

- ### 实现过程

  此部分的工作只需要正确设置trapframe即可，之前lab1完成特权级转换时设置过一次。需要将cs设为USER_CS，将ds、ss、es设置为USER_DS，即将当前的代码、数据、堆栈的访问权限设为了用户态。esp设为USTACKTOP，eip设为elf->e_entry，即程序的入口地址，eflags设为FL_IF使能中断。

  ```C
  tf->tf_cs = USER_CS;
  tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
  tf->tf_esp = USTACKTOP;
  tf->tf_eip = elf->e_entry;
  tf->tf_eflags = FL_IF;
  ```

- ### 其他改变

  有几处根据注释得知需要在前几个lab的基础上修改的地方：

  1. 时间片

     需要设定每一个TICK_NUM的时间流逝，就需要将当前进程的need_resched设为1。

     ```C
     case IRQ_OFFSET + IRQ_TIMER:
     		ticks ++;
         if(ticks == TICK_NUM){
             print_ticks();
             ticks = 0;
             current->need_resched = 1;
         }
         break;
     ```

  2. alloc_proc

     需要初始化lab5中proc_struct新增的成员：uint32_t wait_state; struct proc_struct *cptr, *optr, *yptr。

     ```C
     proc->wait_state = 0;
     proc->optr = proc->yptr = proc->cptr = NULL;
     ```

  3. do_fork

     需新增对父进程wait_state状态的检查，新增对进程关系的初始化(利用set_links函数，cptr为children，optr为older sibling，yptr为younger sibling，set_links函数中，将当前进程的optr设为父进程的cptr，将父进程的cptr设为当前进程，将当前进程的optr的yptr设为当前进程，这样就完成了新进程关系的建立)。同时由于set_links内部完成了进程列表的插入和进程数的++，所以应在lab4基础上删除上述两行。

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
         assert(current->wait_state == 0);  // new line
         if(setup_kstack(proc) != 0)
             goto bad_fork_cleanup_proc;
         if(copy_mm(clone_flags, proc) != 0)
             goto bad_fork_cleanup_kstack;
         copy_thread(proc, stack, tf);
         
         bool intr_flag;
         local_intr_save(intr_flag);
     
         proc->pid = get_pid();
         hash_proc(proc);
         set_links(proc);  // new line
     
         local_intr_restore(intr_flag);
     
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

- ### 描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。

  首先调用proc_run，将current设置为该进程，将ts的esp设置为该进程的esp，再调用函数lcr3切换到当前页目录，最后调用switch_to进行现场状态的交接。由于堆栈、代码段信息的相关寄存器值已经还原，所以接下来就开始了该进程的执行。



### **练习2 父进程复制自己的内存空间给子进程**

- ### 实现过程

  只需要完成copy_range的实现即可。copy_range的整体思路是将一段线性地址空间相等(start—end)，从from地址空间复制到to地址空间，同时to这边需要分配页，具体过程是一个页一个页的复制，直到复制页的尾地址大于end为止。有了线性地址及页目录项的地址，可以很容易地得到其物理页帧。再用宏page2kva得到两个物理页帧的内核虚拟地址，用memcpy进行复制，最后将将to地址空间新申请的物理页帧与线性地址相关联即可。

  ```C
  void * src_kvaddr = page2kva(page);
  void * ds                                                                                                   t_kvaddr = page2kva(npage);
  memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
  ret = page_insert(to, npage, start, perm);
  ```

- 简要说明如何设计实现”Copy on Write 机制“

  copy on write机制不给新进程分配物理地址，而是只为它分配一段虚拟地址，使其指向父进程的相应的物理空间，而当父进程修改了相应的段时，再为子进程分配空间。

  大致实现可以在fork时设计子进程的虚拟地址空间使其指向父进程的物理地址空间，并将该进程作上相应标记，表示它没有独立的物理地址空间，若父进程本就没有独立的物理地址空间，处理方式一样，只需要找到父进程的虚拟地址空间指向的物理地址空间即可。当某一进程对相应的段进行修改时，应该查询它所有的子进程，若查到没有独立物理地址空间的，就为它分配物理地址空间，此时需要递归地往下查找没有独立物理空间的子进程，并将其虚拟地址空间修改为父进程的物理地址空间。



### **练习3 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现**       

do_fork：用当前进程复制一个新的进程，为其分配内核栈空间，复制父进程的内存空间，分配新的pid号，将其加入进程链表。

do_execve：将当前进程清空，然后调用load_icode载入指定的ELF文件代码。

do_wait：查询当前进程的是否有处于ZOMBIE状态的子进程，若有，则将该子进程的资源释放，若没有，则将当前进程的状态设为SLEEPING，设置其wait_state为WT_CHILD，并立马进行schedule。该函数可以传入一个pid参数，应该是若有子进程的pid为传入的pid，且它是当前进程的子进程，还是ZOMBIE状态的话，优先将其释放。

do_exit：释放当前进程的资源，并把其状态设置为ZOMBIE，并将其父进程的等待状态设置为WT_CHILD，并将其唤醒，再将其所有子进程与它断绝父子关系，并全部认init_proc为新父亲。

状态转换图可见proc.c:32-40的注释，我觉得老师已经画得很好了。