# Lab1 Report

### 练习1

- ##### ucore.img是如何一步一步生成的

  1.生成可执行文件bin/kernel

  ​	被gcc编译成为.o的文件

  > - kern/init/init.c
  > - kern/libs/readline.c
  > - kern/libs/stdio.c
  > - kern/debug/kdebug.c
  > - kern/debug/kmonitor.c
  > - kern/debug/panic.c
  > - kern/driver/clock.c
  > - kern/driver/console.c
  > - kern/driver/intr.c
  > - kern/driver/picirq.c
  > - kern/trap/trap.c
  > - kern/trap/trapentry.S
  > - kern/trap/vectors.S
  > - kern/mm/pmm.c
  > - libs/printfmt/c
  > - libs/string.c

  ​	将上述生成的.o用ld命令链接成可执行文件bin/kernel，次即操作系统内核代码。

  2.生成bin/bootblock.o

  ​	被gcc编译成为.o的文件

  > - boot/bootasm.S
  > - boot/bootmain.c
  > - tools/sign.c

  ​	将上述生成的.o用ld命令链接成可执行文件bin/bootblock.o。

  3.将bin/bootblock.out改造成合格的主引导程序

  > bin/sign obj/bootblock.out bin/bootblock

  3.为bin/ucore.img开辟5120000bytes的空间:

  > dd if=/dev/zero of=bin/ucore.img count=10000
  >
  > /dev/zero可以提供任意多的0，count代表块数，每一块缺省为512bytes

  4.将主引导程序写入bin/ucore.img

  > dd if=bin/bootblock of=bin/ucore.img conv=notrunc
  >
  > notrunc：不截断输出文件

  5.将内核程序写入bin/ucore.img

  > dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
  >
  > seek=1：跳过第一个block，即bootloader所在的代码区



- ##### 一个被系统认为是符合规范的硬盘主引导扇区的特征

  大小只有512字节，并且第510个字节为0x55，第511个字节为0xaa。



### 练习3

- ##### 为何，如何开启A20

  ​	早期电脑的地址线只有20条，寻址空间为1M，超过1M的地址就会回卷0x0ffef。为了兼容早期电脑的这个特性，设置了A20信号，若A20信号为0，则高于20位的地址信号都会被抹去。我们当前的机器是32位，只有打开A20才能发挥其全部的寻址能力。。bootasm.S中依靠seta20.1和seta20.2两个汇编代码块实现A20的开启。

  ​	A20信号位于8042的键盘控制器上，其设备的命令端口为0x64，数据端口为0x60。bootasm.S中依靠seta20.1和seta20.2两个汇编代码块实现A20的开启。

  ```asm
  seta20.1:
  
  inb $0x64, %al
  
  testb $0x2, %al
  
  jnz seta20.1		# 等待8042空闲
  
  movb $0xd1, %al
  
  outb %al. $0x64	
  ```

  #0xd1为向8042的P2端口写指令

  ```assembly
  seta20.2:
  
  inb $0x64, %al
  
  testb $0x2, %al
  
  jnz seta20.2		#等待8042空闲
  
  movb $0xdf, %al
  
  outb %al, $0x60	#0xdf为将A20位置1
  ```

  

- ##### 如何初始化GDT

  ```assembly
  lgdt gdtdesc		#从gdtdesc加载gdt
  
  gdtdesc:
  
  	.word 0x17
  
  	.long gdt
  
  gdt:
  
  	SEG_NULLASM
  
  	SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)
  
  	SEG_ASM(STA_W, 0x0, 0xffffffff)
  
  					#由于还未加载操作系统，段表只有内核态的可读执行代码段和可写数据段
  ```

  

- ##### 如何使能和进入保护模式

  ```asm
  movl %cr0, %eax
  
  orl $CR0_PE_ON, %eax	# CR0_PE_ON = 0x1
  
  movl %eax, %cr0	# 将cr0寄存器的PE位置1
  
  ljmp &PROT_MODE_CSEG,  $protcseg	#长跳修改CS
  
  movw $PROT_MODE_DSEG, %ax
  
  movw %ax, %ds
  
  ……					# 设置各段寄存器为内核数据段的index
  
  movl $0x0, %ebp
  
  movl $start, %esp		#start = 0x7c00 建立堆栈
  ```

  



### 练习4

- ##### bootloader如何读取硬盘扇区

  ```asm
  waitdisk();	// 等待磁盘空闲，0x1f7为状态和命令命令寄存器，通过它判断磁盘是否空闲
  
  outb(0x1F2, 1);	// 向磁盘发出请求一个扇区的命令
  
  outb(0x1F3, secno & 0xFF);
  
  outb(0x1F4, (secno >> 8) & 0xFF);
  
  outb(0x1F5, (secno >> 16) & 0xFF);
  
  outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);	// 设置要读数据的LBA
  
  outb(0x1F7, 0x20);	// 发出读命令
  
  waitdisk();
  
  insl(0x1F0, dst, SECTSIZE/4); // 0x1F7空闲时，从ox1F0读取数据到dst
  ```

  

- ##### bootloader如何加载ELF格式的OS

  创建了elfhdr和proghdr两个结构体，elfhdr存放ELF格式的头，proghdr存放可执行文件的程序头部。

  ```c
  readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);  // 读取从第一个扇区开始的8个扇区，实际上只用上了主引导分区
  
  if (ELFHDR->e_magic != ELF_MAGIC) { goto bad; }  // 判断是否为合法的ELF头部
  
  struct proghdr *ph, *eph;
  
  ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
  
  eph = ph + ELFHDR->e_phnum;  // e_phoff为程序头的offset，ph为第一个程序头的地址，e_phnum为程序头的数量，eph为最后一个程序头的地址
  
  for (; ph < eph; ph ++) {                                                                                                                                                                                                                             
  
       readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
  
  }  // p_va为程序被放入内存中的虚拟地址，p_memsz为程序的字节数，p_offset为相对文件头的offset，用readseg将所有ELF程序读入内存
  
  ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();  // 调用入口程序
  ```

  

### 练习5

- ##### **实现过程**

  修改了print_debuginfo函数，在其打印出<unknow>时返回0，打印出有效info时返回1。

  调用函数read_ebp()和read_eip()得到当前域(print_stackframe)中ebp和eip的值。在递归循环中，首先打出当前ebp、eip。第一个参数的位置在(ebp+8)的位置，需打出四个可能的参数，则循环四次，运用内联汇编，打出四个参数。

  ```c
  for(int i = 0;i < 4; i ++){
          asm volatile("movl 8(%2, %1, 4), %0" : "=r"(arg):"r"(i), "r"(ebp));
          cprintf("0x%08x ", arg);
      }
  ```

  调用print_debuginfo(eip-1)，若<unknow>，则说明当前函数无调用函数，是最底层的函数，则break出递归循环。

  ```c
  if(!print_debuginfo(eip - 1))
      break;
  ```

  否则，修改ebp、eip的值，进入下一次循环。eip的值存在(ebp+4)的位置。

  ```c
  asm volatile("movl 4(%1), %0" : "=r" (eip) : "r"(ebp));
  
  asm volatile("movl (%1), %0" : "=r"(ebp) : "r"(ebp));
  ```

  

### 练习6

- ##### **中断描述符表一个表项占多少字节，哪几位代表中断处理代码的入口**

  8字节，8-15位为段选择子，0-7位、48-63位共同组成offset。

- ##### 载入中断向量表，完善时钟中断的实现过程

  外部调用__vectors[]，其在vector.S中，数组存放的是各种中断例程的入口地址。

  ```c
  extern uintptr_t __vectors[];
  ```

  共有256个中断，故循环256次，用SETGATE进行初始化。

  > ```c
  > for(int i = 0; i < 256; i ++){
  > 
  >     SETGATE(idt[i], 0, KERNEL_CS, __vectors[i], DPL_KERNEL);
  > 
  > }  // idt[i]表示当前的中断描述符号，0表示是interrupt，KERNEL_CS是中断服务例程的段选择子，__vectors[i]是中断服务例程在段中的偏移值，因为当前操作系统只有一个段，此时逻辑地址等于段偏移值，DPL_KERNEL是0特权级。
  > ```

  而有一个ID是特殊的，0x80号中断，即idt[128]。它是一个系统调用(trap)，所以应该修改它的type和dpl。

  ```c
  SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
  ```

  最后用lidt命令载入该idt。

  ```c
  lidt(&idt_pd);
  ```

  对于时钟中断，每次产生时钟中断(10ms)时，将ticks加1，判断是否为100，若是，则调用print_ticks()打出"100 ticks"，并重新置ticks为0，若否，则无动于衷。

  ```c
  ticks ++;
   if(ticks == 100){
       print_ticks();
       ticks = 0;
   }
  ```

  

### 扩展练习 challenge 1

- ##### 内核态到用户态

  内核态产生中断，故int无特权级转换，故保存中断现场的trapframe中没有esp和ss，但是在iret返回时，发生了特权级转换，故在出栈时会多出8字节，即iret认为的esp和ss所存的位置，所以如果对堆栈寄存器不加修改直接int势必会导致当前堆栈数据遭到破坏，于是在中断压栈前将esp加8，返回时就不会影响到现在的堆栈。

  ```c
  asm volatile (
      "subl $0x8, %%esp\n"
      "int %0\n"			//产生切换到用户态中断
      :: "i"(T_SWITCH_TOU)
  )
  ```

  由于中断时无特权级转换，所以trapframe位于内核堆栈，与中断前的数据在地址上应是连续的，故只用修改当前trapframe的各参数，trapframe出栈时便会修改影响特权级的关键寄存器的值(cs、ds、es、ss)。

  ```c
  case T_SWITCH_TOU:
  _t_switch_tou:  // 用于键盘中断跳转
      if(trap_in_kernel(tf)) {
          tf->tf_cs = USER_CS;
          tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
          tf->tf_eflags |= FL_IOPL_MASK;  // 降低调用IO所需要的权限，使用户态也可使用IO
          tf->tf_esp = (uint32_t)tf + sizeof(struct trapframe);  // 修改esp到最初的起点
      }
      break;
  ```

- 用户态到内核态

  ```c
  asm volatile("int %0" :: "i"(T_SWITCH_TOK));  // static void lab1_switch_to_kernel()
  ```

  用户态产生中断，中断时发生特权级的转变，此时trapframe所在地位于内核堆栈，且压入了esp和ss。而在中断返回时没有特权级的转变，则trapframe出栈时esp和ss不会弹出，所以不能在trapframe上修改esp引导返回地址回到最初的起点。中断产生后，经过向量描述符表和中断例程alltraps后来到trap函数，此时栈顶存放trapframe的首地址(目前是栈顶地址+4)，将来trap返回时，alltraps便是通过栈顶的值找到trapframe的位置，弹出段寄存器和错误码后再交由iret继续恢复工作，由于trapframe中的esp不会派上用场，我们只能让iret弹出各寄存器后使esp自然的回到最初的起点，故我们只需在用户堆栈上复制一份没有esp和ss的trapframe，并修改内核栈栈顶的值为该trapframe的地址，便可以引导iret在用户栈上操作，从而使得iret结束后esp自然的落在了中断开始前的位置。

  ```c
  case T_SWITCH_TOK:
  _t_switch_tok:
      if(!trap_in_kernel(tf)) {
          struct trapframe *temp_tf = (struct trapframe *)(tf->tf_esp - sizeof(struct trapframe) + 8);  // tf_esp存放中断前地址，复制的trapframe的首地址应该距它一个trapframe大小减8(esp、ss和padding)的位置
          __memcpy(temp_tf, tf, sizeof(struct trapframe) - 8);
          temp_tf->tf_cs = KERNEL_CS;
          temp_tf->tf_ds = temp_tf->tf_es = KERNEL_DS;  // iret并不弹出ss，所以修改ss没用，可能是在iret时根据特权级发生转变自动修改ss？
          temp_tf->tf_eflags &= ~FL_IOPL_MASK;  // 还愿IO调用权限
          *((uint32_t *)tf - 1) = (uint32_t)temp_tf;  // 将内核栈栈顶的值赋为用户栈上复制的trapframe的首地址
      }
      break;
  ```

  同时在初始化中断向量表时，应将T_SWITCH_TOK这一项的DPL赋为DPL_USER

  ```c
  SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
  ```

- ### 扩展练习 Challenge 2

  只需在键盘中断时判断是否是0或3，若是则goto相应的标签。

  ```c
  if (c == 48) goto _t_switch_tok;  // 0
  if (c == 51) goto _t_switch_tou;  // 3
  ```

  