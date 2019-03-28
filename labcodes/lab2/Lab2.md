# Lab2 Report

### 练习一 first-fit连续物理内存分配算法的实现

- ### 实现过程

  default_init和default_init_memmap无需修改。default_init用于初始化free_area，将free_list的prev和next都指向它自己，将nr_free空闲页数置0。

```c
static void default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
```

default_init_memmap用于初始化一段物理页帧，使其处于能够被映射的状态，将这段物理页帧的页描述符初始化，设置基址的property为页数，并使能property，nr_free加上分配的页数，最后将这个block加入free_list链表中（此时list_add_before和list_add_after没有区别，因为表中只有free_list一个节点）。

```c
static void default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;  // 初始化flag、property
        set_page_ref(p, 0);  // 置被引用数为0
    }
    base->property = n;  // 基址的property为改free_block页数
    SetPageProperty(base);  // 使能property
    nr_free += n;
    list_add(&free_list, &(base->page_link));
}
```

default_alloc_pages(size_t n)用于分配n页物理空间，实现过程：首先检查分配页数是否合法，若n比目前空前页数大，os将无能为力。对free_list进行查询，判断节点对应的free_block的页数是否大于n，找到第一个满足条件的节点（若找不到，则返回NULL），将这个节点从基址开始的n个页剥离出来，新的基地址为page+n，初始化新基址的property并使能property，将新的free_block加入原节点后面，再删除原节点，清除原基址property使能，返回原基址，便完成了页的分配。

```C
static struct Page * default_alloc_pages(size_t n) 
{
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;  // 找到则修改，没找到则返回NULL
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);  // 通过宏le2page可以通过list_entry找到物理帧描述符的基地址
        if (p->property >= n) {
            page = p;
            break;
        }
    }
    if (page != NULL) {
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;
            SetPageProperty(p);
            list_add_after(&(page->page_link), &(p->page_link));  // 将新节点加在老节点后面
        }
        nr_free -= n;
        list_del(&(page->page_link));  // 再删除老节点，便把老节点换成了新节点
        ClearPageProperty(page);
    }
    return page;
}
```

default_free_pages用于释放一段页帧，实现思路是先释放这段页帧为一个free_block，并对其进行初始化（类似default_init_memmap），查找free_list中地址在插入块前后的两个块，看是否存在可以合并，最后nr_free+=n。

```C
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base, *prev_p;
    list_entry_t *le;
    for (; p != base + n; p ++) {
        assert(!PageReserved(p) && !PageProperty(p));
        p->flags = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    if(list_empty(&free_list)){  // 若此时free_list为空，则直接将其加入链表中
        list_add(&free_list, &(base->page_link));
    }
    else{
        le = list_next(&free_list);   
        p = le2page(le, page_link);
        while (p < base && le != &free_list) {
            le = list_next(le);
            p = le2page(le, page_link);
        }  // 循环后，此时的p为地址刚好大于插入块的free_block或者是&free_list，而prev_p取p的前一块，则为地址刚好比插入块小的free_block
        prev_p = le2page(list_prev(le), page_link);
        if(prev_p + prev_p->property == base){  // 若前一块可以与插入块合并，则修改前一块的property，并取消插入块的property使能，再将base设为前一块的基页，以便后续统一的操作
            prev_p->property += n;
            ClearPageProperty(base);
            base = prev_p;
        }
        else  // 若前一块不能合并插入块，则需要将插入块插入
            list_add_before(le, &(base->page_link));
        if(base + base->property == p){  // 若后一块可以被插入块（若前一步发生合并，此时的base就是前一块基页）合并，则修改base->property，取消后一块的property使能，最后将后一块从链表中删除
            base->property += p->property;
            ClearPageProperty(p);
            list_del(&(p->page_link));
        }
    }
    nr_free += n;
}
```

- ### 是否有进一步的改进空间

思考此问题时发现有进一步改进空间，然后改进了。



### 练习二 实现寻找虚拟地址对应的页表项

- ### 实现过程

  get_pte中，给出了一级页表的地址与线性地址，线性地址中，前十位时一级页表的index，中十位是二级页表的index，后十二位是帧内偏移，所以我们可以根据前十位的值很容易的找到对应二级页表的地址。但需要考虑二级页表不存在的情况，此时需要用alloc_page来给这个二级页表分配内存。

  ```C
  pte_t * get_pte(pde_t *pgdir, uintptr_t la, bool create)
  {
  		pde_t *pdep = &pgdir[PDX(la)];  // 宏PDX可以得到线性地址前10位的值，也就是一级页表的index
      if(!(*pdep & PTE_P)){  // 如果该页表项不存在
          struct Page *page;
          if(create){  // 如果需要创建
              page = alloc_page();
              if(page != NULL){
                  set_page_ref(page, 1);  // 设置被引用值为1
                  uintptr_t pa = page2pa(page);
                  memset(KADDR(pa), 0, PGSIZE);  // 初始化该二级页表，KADDR可以从它的物理地址得到虚拟地址，才能被系统函数正确使用
                  *pdep = pa | PTE_P | PTE_W | PTE_U;  // 设置该一级页表项的参数，因为二级页表的地址是4k对齐，所以低12位都为0，可以用来存放参数
              }
              else
                  return NULL;
          }
          else
              return NULL;
      }
      pte_t *pd_base = KADDR(PDE_ADDR(*pdep));  // PDE_ADDR将一级页表项的后12位置0，即得到了二级页表的物理地址，用KADDR转换为操作系统能够使用的虚拟地址
      return &pd_base[PTX(la)];  // PTX得到线性地址的中10位，即二级页表的index，在二级页表基地址上偏移便可得到二级页表项
  }
  ```

  page_remove_pte用于释放二级页表项，释放时同时需要修改对应page的参数，主要是修改被引用数，被引用数到0时，需要释放该page。

  ```C
  static inline void page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep)
  {
      if(*ptep & PTE_P){
          struct Page *page = pte2page(*ptep);  // 因为二级页表项中前20位存放该项指向物理帧的物理地址的前二十位，物理地址的前二十位就是该页相对于pages的偏移，即可以找到该页的描述符
          page_ref_dec(page);
          if(page->ref == 0)  // 若无页表项再指向这个帧，则释放
              free_page(page);
          *ptep &= ~PTE_P;  // 置存在位为0
          tlb_invalidate(pgdir, la);  // 刷新TLB
      }
  }
  ```

- ### page与页表项、页目录项的关系

  page的偏移是该帧的物理地址的前20位，因为每一帧都是4k对齐，所以page的偏移即可以表示帧的物理地址，而页表项的前20位就是指向帧的物理地址的前二十位，也就是page的偏移值，page与页目录项没有直接关系，相应页目录项存放该page对应的页表的物理地址。

- ### 虚拟地址与物理地址相等

  ucore中虚拟地址经过对等的段映射得到的线性地址与虚拟地址相同，所以考虑线性地址与物理地址相等，而低12位的帧内偏移本来就对应，所以只用考虑高20位的，只需要页表项的前二十位等于线性地址的前二十位即可。