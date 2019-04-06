# **Lab3 Report**

### **练习1 给未被映射的地址映射上物理页**

- ### 实现过程

  修改do_pgfault函数。do_pgfault函数接收发生缺页异常的虚拟地址addr，我们可以调用get_pte函数得到addr对应的页表项地址。再判断页表项的内容是否为空，若为空，则只需调用pgdir_alloc_page函数分配一个页并建立映射关系，若不为空，则需要用到页的替换算法。

  ```C
  if((ptep = get_pte(mm->pgdir, addr, 1)) == NULL)  // get_pte的create参数设置为1，即若存放页表项的页不存在，则创建，若创建失败，则goto failed
      goto failed;
  if(*ptep == 0){
      if(pgdir_alloc_page(mm->pgdir, addr, perm) == NULL)  // 为addr所在的页分配物理帧，并将pte映射到该物理帧，然后将该页加入可置换页链表
          goto failed;
  }
  ```

  

- ### 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

  pde可以让我们找到pte，而通过pte的内容我们可以判断是否需要进行换入换出算法。

  

- ### 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

  硬件需要保存缺页的虚拟地址，然后经过中断向量表再次进入一个缺页服务例程。但在一般情况下，缺页服务历程本使用操作系统的内核代码和数据，不会进行页的换入换出，所以不会出现页访问异常。

### **练习2 补充完成基于FIFO的页面替换算法**

- ### 实现过程

  首先需要完善do_pgfault函数页表项不为空的情况，此时应进行页的换入换出，调用swap_in函数，并将换入的物理帧、pte、缺页的线性地址相互对应，然后将该页加入可置换页链表。

  ```C
  else {
      if(swap_init_ok) {
          struct Page *page=NULL;
          ret = swap_in(mm, addr, &page);  // 将换入的页的地址存入page
          if (ret != 0) {  // ret为0时表示换入成功
              goto failed;
          }
          page_insert(mm->pgdir, page, addr, perm);  // 建立addr对应pte与新换入页的映射关系
          swap_map_swappable(mm, addr, page, 1);  // 将新换入页加入可置换页链表
          page->pra_vaddr = addr;
      }
      else
          goto failed;
  }
  ```

  其次需要完成FIFO算法。_fifo_map_swappable函数用于将一个页加入目前可置换页链表的最后一项。具体实现比较简单，由于ucore中实现的链表结构是一个循环双向链表。所以插入到最后一项相当于插入到第一项之前。

  ```c
  list_entry_t *head=(list_entry_t*) mm->sm_priv;
  list_entry_t *entry=&(page->pra_page_link);
   
  assert(entry != NULL && head != NULL);
  
  list_add_before(head, entry); // 插入第一项之前
  ```

  _fifo_swap_out_victim函数用于找到换出的受害者，受害者应该是最早被加入链表的页，因为每次都加入到链表的尾部，所以受害者应该是第一个页，也就是链表头的下一个元素。换出后需要受害者移出链表，并且将其地址返回*ptr_page。

  ```C
  list_entry_t *head=(list_entry_t*) mm->sm_priv;
  assert(head != NULL);
  assert(in_tick==0);
  
  list_entry_t *victim = list_next(head);  // 表头的下一项即是最早加入链表的页
  struct Page *victim_page = le2page(victim, pra_page_link);  // 利用宏le2page得到换出链表项对应的帧
  list_del(victim);  // 移出换出链表项
  *ptr_page = victim_page;
  ```

  