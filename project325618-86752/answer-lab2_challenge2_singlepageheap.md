# （答案）挑战二：singlepageheap

### ——对应PKE文档的4.5 lab2_challenge2

#### 实验目标：

* 深化对虚拟地址、物理地址的理解。
* 深化对操作系统内存管理的认知。

该挑战通过实现malloc和free函数来要求学生通过对地址的查找、操纵来操纵数据结构，通过虚拟地址与物理地址的大量的转换，以及对地址的强制类型转换来提高学生对地址、指针的控制程度，同时还涉及到设计的数据结构在地址中的对齐，因此本挑战难度很高。

---

在lab2_3_pagefault实验的基础上，达到挑战二的实验需求，需要完成以下2个部分的内容：

#### 增加内存控制块：

修改kernel/vmm.h文件，添加mem_control_block数据结构如下。

```
  6 //
  7 // user memory control block.
  8 // is_available: 1 for usable, 0 for unusable.
  9 // size: this block size.
 10 //
 11 typedef struct mem_control_block{
 12     int is_available;
 13     int size;
 14     uint64 offset;
 15     struct mem_control_block *next;
 16 }mem_control_block;
```

通过mem_control_block管理已经申请的内存，即当你每调用一次malloc()申请n个字节时，实际物理空间中在n个字节之前会加上一个mem_control_block。

<img src="ans-pictures/heap_unit_block.png" alt="fig1_2" style="zoom:100%;" />



## PART 1：实现进程heap size的增加

#### 增加user_vm_malloc函数：

在kernel/vmm.c中，声明user_vm_malloc函数，目的是将进程newsz与oldsz之间的虚拟地址映射到实际的物理地址上。

```
168 //                                                
169 // Allocate PTEs and physical memory to grow process from oldsz to
170 // newsz, which need not be page aligned.  Returns new size or 0 on error.
171 //
172 uint64 user_vm_malloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
173 {
174   char *mem;
175   if(oldsz > newsz)
176     return oldsz;
177   oldsz = PGROUNDUP(oldsz);
178   for(uint64 old = oldsz; old < newsz; old += PGSIZE)
179   {
180     mem = (char *)alloc_page();
181     if(mem == 0)
182       panic("failed to user_vm_malloc .\n");
183     memset(mem,0,sizeof(uint8) * PGSIZE);
184     map_pages(pagetable, oldsz, PGSIZE, (uint64)mem, prot_to_type(PROT_READ | PROT_WRITE,1) );
185   }
186   return newsz;
187 }
```

注意177行，将oldsz向上对其一页，含义是对其后的地址之前的虚拟地址已经与实际的物理地址有了映射，所以此时的newsz是可能小于oldsz的，当newsz<oldsz时，无需对虚拟地址与物理地址之间的映射进行操作，直接返回即可。
当newsz > oldsz时，通过该函数实现对newsz与oldsz之间的虚拟地址添加对应的物理地址的映射，具体操作为（180行）申请一页物理页，（183行）初始化该页，（184行）将该页与虚拟地址oldsz之后的一页地址映射。该函数的实现可以方便后面直接通过系统调用来实现对进程heap大小的管理。

#### 利用user_vm_malloc构造growprocess函数

修改kernel/process.c,添加growprocess函数，实现对current指向的进程的heap_size的增加。

```
 30 int growprocess(uint64 n)
 31 {
 32   uint64 sz = current->heap_sz;
 33   if(n < 0) panic("failed in growprocess .\n");
 34   user_vm_malloc(current->pagetable, sz, sz + n);
 35   current->heap_sz = sz + n;
 36   return 0;
 37 }
```
该函数则是对进程的heap_sz进行实际的管理，当执行过user_vm_malloc之后，代表进程的pagetable已经对sz+n之前的地址进行了映射，所以（第35行）直接将heap_sz+n即可。
再进一步进行抽象，抽象成系统调用sys_user_sbrk函数。

#### 抽象growprocess函数成系统调用sys_user_sbrk

修改kernel/syscall.c,添加sys_user_sbrk函数。

```
 48 //
 49 // allocate n bytes. 
 50 //
 51 uint64 sys_user_sbrk(uint64 n)
 52 {
 53   uint64 addr = current->heap_sz;
 54   growprocess(n);
 55   return addr;
 56 }
```
该系统调用将申请的n个字节的首地址返回，并在虚拟地址空间中进行扩展。
## PART 2：管理内存池

#### 初始化内存池

在load_user_program后，对user_app对应的堆内存池应先进行初始化。修改kernel/vmm.c中的代码。

```
233     if(init_flag == 0)
234     {
235 
236       current->heap_sz = USER_FREE_ADDRESS_START;
237       uint64 addr = current->heap_sz;
238       growprocess(sizeof(mem_control_block));
239       pte_t *pte = page_walk(current->pagetable, addr, 0);
240       mem_control_block *first_control_block = (mem_control_block *) PTE2PA(*pte);
241       current->heap_memory_start = (uint64) first_control_block;
242       first_control_block->next = first_control_block;
243       first_control_block->size = 0;
244       current->heap_memory_last = (uint64)first_control_block;
245       init_flag = 1;
246     }
```

<img src="ans-pictures/process_heap_organize.png" alt="fig1_2" style="zoom:100%;" />

整个内存池如上图所示，用一个链表连接，所以在初始化进程时，也需要对其堆内存池进行初始化，上述代码仅仅是创建head和tail块。

#### 实现malloc函数

修改kernel/vmm.c代码，创建函数malloc(int n)，实现优先从内存池中获取申请的空间，若内存池中没有满足要求的，则在通过PART 1中的API从物理内存中创建内存块并放入进程的内存池中。

```
227 //
228 // malloc n bytes for user.
229 //
230 uint64 malloc(int n)
231 {
232     mem_control_block *head = (mem_control_block *)current->heap_memory_start;
233     mem_control_block *last = (mem_control_block *)current->heap_memory_last;
234 
235     while (1)
236     {
237         if(head->size >= n && head->is_available == 1)
238         {
239 
240             head->is_available = 0;
241             return head->offset + sizeof(mem_control_block);
242         }
243         if(head->next == last) break;
244         head = head->next;
245     }
246    uint64 alloacte_addr = current->heap_sz;
247    growprocess((uint64) (sizeof(mem_control_block) + n + 8));
248    pte_t *pte = page_walk(current->pagetable, alloacte_addr, 0);
249    mem_control_block *now = (mem_control_block *)(PTE2PA(*pte) + (alloacte_addr & 0xfff));
250    uint64 amo = (8 - ((uint64)now % 8))%8;
251    now = (mem_control_block *)((uint64)now + amo);
252 
253    now->is_available = 0;
254    now->offset = alloacte_addr;
255    now->size = n;
256    now->next = head->next;
257 
258    head->next = now;
259    head = (mem_control_block *)current->heap_memory_start;
260    return alloacte_addr + sizeof(mem_control_block);
261 
262 }
```

该函数实现了当用户程序申请n个字节的空间时，内核首先对该用户的内存池进行遍历（第235-245行），查找是否有可用的内存块（空闲且大小大于或者等于n），当有时，则将该内存块的内存返回，相当于上图中从head开始从第一个mem_control_block遍历至tail，查找是否有状态空闲，且其管理的size大于n的。若没有，（第246-259行）则向内核申请在heap中增加sizeof(mem_control_block) + n 个字节的大小，生成一个内存块并加入内存池中，**此处对于内存的操作比较复杂，再加上链表操作，需要认真理清思路！**

#### 实现free函数

修改kernel/vmm.c代码，创建free(void *)函数，实现根据参数提供的虚拟地址，寻找到对应的物理地址，并找到其控制块，将该内存块放入进程的内存池中。

```
265 //
266 // free the allocated memory into pool.
267 //
268 void free(void *firstaddr)
269 {
270     firstaddr = (void *)((uint64)firstaddr - sizeof(mem_control_block));
271     pte_t *pte = page_walk(current->pagetable, (uint64)(firstaddr), 0);
272     mem_control_block *now = (mem_control_block *)(PTE2PA(*pte) + ((uint64)firstaddr & 0xfff));
273     uint64 amo = (8 - ((uint64)now % 8))%8;
274    now = (mem_control_block *)((uint64)now + amo);
275     if(now->is_available == 1)
276         panic("in free function, the memory has been freed before! \n");
277     now->is_available = 1;
278 }
```

free函数首先要根据参数的虚拟地址，根据进程的pagetable，找到对应的物理地址（第270-274行），并找到其control_block，直接将内存块的状态修改为空闲（第275-277行）即可。





#### 关于“标答”的说明

“标答”只是给出了一种实现的可能，学生可能给出不一样的解决方案。例如，对内存池内存的组织可以有不同的结构。只要能实现挑战的设计目标，这些其他的解决方案就都是正确的。

从解决问题的方法的角度，以上给出的解决方案是较为科学和简洁的方法，所有解答至少应涉及以上部分。

