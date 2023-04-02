# （答案）挑战一：缺页异常

### ——对应PKE文档的4.4 lab2_challenge1

#### 实验目标

* 加深对缺页异常的理解
* 提高对进程控制块的理解与运用。

#### 实验原理

<img src="ans-pictures/user_vm_space.png" alt="fig1_2" style="zoom:100%;" />

上图是用户进程的逻辑地址空间，可以看到用户态栈是从USER_STACK_TOP由高地址向低地址扩展，根据lab2_3，由于各种原因导致的“爆栈”是合理的缺页异常，而非法访问其它地址空间则是非法的缺页异常，因此该挑战的目的就是实现区分不同情况的缺页异常，从而进行不同的处理。因此当触发缺页异常的地址刚好在原本用户态栈底旁边时，则进行扩展栈空间的处理，其余地址的访存则是非法的缺页异常，应直接终止进程的运行。

---



在lab2_3_pagefault实验的基础上，达到挑战一的实验需求，需要完成以下内容：

#### 修改对应中断处理函数：

修改kernel/strap.c中的handle_user_page_fault函数，当发生缺页异常时，会从stvec进入smode_trap.S、smode_trap_handler函数然后跳转至handle_user_page_fault来进行最终的处理。

```
 46 //
 47 // sepc: the pc when fault happens;
 48 // stval: the virtual address that causes pagefault when being accessed.
 49 //
 50 void handle_user_page_fault(uint64 mcause, uint64 sepc, uint64 stval) {
 51   sprint("handle_page_fault: %lx\n", stval);
 52   switch (mcause) {
 53     case CAUSE_STORE_PAGE_FAULT:
 54       // TODO (lab2_3): implement the operations that solve the page fault to
 55       // dynamically increase application stack. 
 56       // hint: first allocate a new physical page, and then, maps the new page to the
 57       // virtual address that causes the page fault.
 58       {
 59         if(stval - current->trapframe->regs.sp < 32){
 60           uint64 newpage = (uint64)alloc_page();
 61         user_vm_map((pagetable_t)current->pagetable, ROUNDDOWN(stval, PGSIZE), PGSIZE, newpage,
 62               prot_to_type(PROT_WRITE | PROT_READ, 1));
 63           
 64         }
 65         else{
 66           sprint("page fault happens at address 0x%lx is illegal!\n", stval);
 67           shutdown(-1);
 68         }
 69        }
 70       break;
 71     default:
 72       sprint("unknown page fault.\n");
 73       break;
 74   }
 75 }
```

在实现该函数时，首先判断发生缺页异常的地址是否是由于超过栈底而触发的缺页异常（第59行），若是则是合理的缺页异常，（第59-63行）可以扩大内核栈的大小，为其映射物理块，若不是，则是非法的访问地址，应该报错并且终止程序（第65-68行）。



#### 关于“标答”的说明

“标答”只是给出了一种实现的可能，学生可能给出不一样的解决方案。只要能实现挑战的设计目标，这些其他的解决方案就都是正确的。

从解决问题的方法的角度，以上给出的解决方案是较为科学和简洁的方法，所有解答至少应涉及以上部分。

