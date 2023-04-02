# 第四章．实验2：内存管理


### 目录 

- [4.1 lab2_challenge1 复杂缺页异常（难度：&#9733;&#9734;&#9734;&#9734;&#9734;）](#lab2_challenge1_pagefault)
  - [给定应用](#lab2_challenge1_app)
  - [实验内容](#lab2_challenge1_content)
  - [实验指导](#lab2_challenge1_guide)

- [4.2 lab2_challenge2 堆空间管理（难度：&#9733;&#9733;&#9733;&#9733;&#9734;）](#lab2_challenge2_singlepageheap)
  - [给定应用](#lab2_challenge2_app)
  - [实验内容](#lab2_challenge2_content)
  - [实验指导](#lab2_challenge2_guide)

<a name="lab2_challenge1_pagefault"></a>

## 4.1 lab2_challenge1 复杂缺页异常（难度：&#9733;&#9734;&#9734;&#9734;&#9734;）

<a name="lab2_challenge1_app"></a>

#### 给定应用

- user/app_sum_sequence.c

  ```c
   1	/*
  2	 * The application of lab2_4.
  3	 * Based on application of lab2_3.
  4	 */
  5	
  6	#include "user_lib.h"
  7	#include "util/types.h"
  8	
  9	//
  10	// compute the summation of an arithmetic sequence. for a given "n", compute
  11	// result = n + (n-1) + (n-2) + ... + 0
  12	// sum_sequence() calls itself recursively till 0. The recursive call, however,
  13	// may consume more memory (from stack) than a physical 4KB page, leading to a page fault.
  14	// PKE kernel needs to improved to handle such page fault by expanding the stack.
  15	//
  16	uint64 sum_sequence(uint64 n, int *p) {
  17	  if (n == 0)
  18	    return 0;
  19	  else
  20	    return *p=sum_sequence( n-1, p+1 ) + n;
  21	}
  22	
  23	int main(void) {
  24	  // FIRST, we need a large enough "n" to trigger pagefaults in the user stack
  25	  uint64 n = 1024;
  26	
  27	  // alloc a page size array(int) to store the result of every step
  28	  // the max limit of the number is 4kB/4 = 1024
  29	
  30	  // SECOND, we use array out of bound to trigger pagefaults in an invalid address
  31	  int *ans = (int *)naive_malloc();
  32	
  33	  printu("Summation of an arithmetic sequence from 0 to %ld is: %ld \n", n, sum_sequence(n+1, ans) );
  34	
  35	  exit(0);
  36	}
  
  ```
  
  程序思路基本同lab2_3一致，对给定n计算0到n的和，但要求将每一步递归的结果保存在数组ans中。创建数组时，我们使用了当前的malloc函数申请了一个页面（4KB）的大小，对应可以存储的个数上限为1024。在函数调用时，我们试图计算1025求和，首先由于n足够大，所以在函数递归执行时会触发用户栈的缺页，你需要对其进行正确处理，确保程序正确运行；其次，1025在最后一次计算时会访问数组越界地址，由于该处虚拟地址尚未有对应的物理地址映射，因此属于非法地址的访问，这是不被允许的，对于这种缺页异常，应该提示用户并退出程序执行。如上的应用预期输出如下：
  
  ```bash
  In m_start, hartid:0
  HTIF is available!
  (Emulated) memory size: 2048 MB
  Enter supervisor mode...
  PKE kernel start 0x0000000080000000, PKE kernel end: 0x000000008000e000, PKE kernel size: 0x000000000000e000 .
  free physical memory address: [0x000000008000e000, 0x0000000087ffffff] 
  kernel memory manager is initializing ...
  KERN_BASE 0x0000000080000000
  physical address of _etext is: 0x0000000080004000
  kernel page table is on 
  User application is loading.
  user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000 
  Application: ./obj/app_sum_sequence
  Application program entry point (virtual address): 0x00000000000100da
  Switching to user mode...
  handle_page_fault: 000000007fffdff8
  handle_page_fault: 000000007fffcff8
  handle_page_fault: 000000007fffbff8
  handle_page_fault: 000000007fffaff8
  handle_page_fault: 000000007fff9ff8
  handle_page_fault: 000000007fff8ff8
  handle_page_fault: 000000007fff7ff8
  handle_page_fault: 000000007fff6ff8
  handle_page_fault: 0000000000401000
  this address is not available!
  System is shutting down with exit code -1.
  ```

根据结果可以看出：前八个缺页是由于函数递归调用引起的，而最后一个缺页是对动态申请的数组进行越界访问造成的，访问非法地址，程序报错并退出。

#### 实验内容

本实验为挑战实验，基础代码将继承和使用lab2_3完成后的代码：

- （先提交lab2_3的答案，然后）切换到lab2_challenge1_pagefaults、继承lab2_3中所做修改：

```bash
//切换到lab2_challenge1_pagefault
$ git checkout lab2_challenge1_pagefaults

//继承lab2_3以及之前的答案
$ git merge lab2_3_pagefault -m "continue to work on lab2_challenge1"
```

注意：**不同于基础实验，挑战实验的基础代码具有更大的不完整性，可能无法直接通过构造过程。**同样，不同于基础实验，我们在代码中也并未专门地哪些地方的代码需要填写，哪些地方的代码无须填写。这样，我们留给读者更大的“想象空间”。

- 本实验的具体要求为：通过修改PKE内核（包括machine文件夹下的代码），使得对于不同情况的缺页异常进行不同的处理。
- 文件名规范：需要包含路径，如果是用户源程序发生的错误，路径为相对路径，如果是调用的标准库内发生的错误，路径为绝对路径。

<a name="lab2_challenge1_guide"></a>

#### 实验指导

- 你对内核代码的修改可能包含以下内容：
  - 修改process.h中进程的数据结构以记录进程用户栈的栈底。
  - 修改kernel/strap.c中的异常处理函数。对于合理的缺页异常，扩大内核栈大小并为其映射物理块；对于非法地址缺页，报错并退出程序。

**注意：完成挑战任务对两种缺页进行实现后，读者可对任意n验证，由于目前的malloc函数是申请一个页面大小，所以对于n<1024，只会产生第一种缺页并打印正确的计算结果；对于n>=1024，则会因为访问非法地址退出，请读者验证自己的实现。**

**另外，后续的基础实验代码并不依赖挑战实验，所以读者可自行决定是否将自己的工作提交到本地代码仓库中（当然，提交到本地仓库是个好习惯，至少能保存自己的“作品”）。**

<a name="lab2_challenge2_singlepageheap"></a>

## 4.2 lab2_challenge2 堆空间管理 （难度：&#9733;&#9733;&#9733;&#9733;&#9734;）

<a name="lab2_challenge2_app"></a>

#### 给定应用

- user/app_singlepageheap.c

```c
 1	/*
2	 * Below is the given application for lab2_challenge2_singlepageheap.
3	 * This app performs malloc memory.
4	 */
5	
6	#include "user_lib.h"
7	#include "util/types.h"
8	#include "util/string.h"
9	int main(void) {
10	  
11	  char str[20] = "hello world.";
12	  char *m = (char *)better_malloc(100);
13	  char *p = (char *)better_malloc(50);
14	  if((uint64)p - (uint64)m > 512 ){
15	    printu("you need to manage the vm space precisely!");
16	    exit(-1);
17	  }
18	  better_free((void *)m);
19	
20	  strcpy(p,str);
21	  printu("%s\n",p);
22	  exit(0);
23	  return 0;
24	}
```

以上程序先利用better_malloc分别申请100和50个字节的一个物理页的内存，然后使用better_free释放掉100个字节，向50个字节中复制一串字符串，进行输出。原本的pke中malloc的实现是非常简化的（一次直接分配一个页面），你的挑战任务是**修改内核(包括machine文件夹下)的代码，使得应用程序的malloc能够在一个物理页中分配，并对各申请块进行合理的管理**，如上面的应用预期输出如下：

```bash
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080008000, PKE kernel size: 0x0000000000008000 .
free physical memory address: [0x0000000080008000, 0x0000000087ffffff]
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on
User application is loading.
user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000
Application: obj/app_singlepageheap
Application program entry point (virtual address): 0x00000000000100b0
Switch to user mode...
hello world.
User exit with code:0.
System is shutting down with exit code 0.
```

通过应用程序和对应的预期结果可以看出：两次申请的空间在同一页面，并且释放第一块时，不会释放整个页面，所以需要你设计合适的数据结构对各块进行管理，使得better_malloc申请的空间更加“紧凑”。

<a name="lab2_challenge2_content"></a>

####  实验内容

本实验为挑战实验，基础代码将继承和使用lab2_3完成后的代码：

- （先提交lab2_3的答案，然后）切换到lab2_challenge2、继承lab2_3中所做修改：

```bash
//切换到lab2_challenge2_singlepageheap
$ git checkout lab2_challenge2_singlepageheap

//继承lab2_challenge1以及之前的答案
$ git merge lab2_3_pagefault -m "continue to work on lab2_challenge2"
```

注意：**不同于基础实验，挑战实验的基础代码具有更大的不完整性，可能无法直接通过构造过程。**同样，不同于基础实验，我们在代码中也并未专门地哪些地方的代码需要填写，哪些地方的代码无须填写。这样，我们留给读者更大的“想象空间”。

- 本实验的具体要求为：通过修改PKE内核（包括machine文件下的代码），实现优化后的malloc函数，使得应用程序两次申请块在同一页面，并且能够正常输出存入第二块中的字符串"hello world"。
- 文件名规范：需要包含路径，如果是用户源程序发生的错误，路径为相对路径，如果是调用的标准库内发生的错误，路径为绝对路径。

<a name="lab2_challenge2_guide"></a>

#### 实验指导

- 为完成该挑战，你需要对进程的虚拟地址空间进行管理，建议参考Linux的malloc内存分配策略从而实现malloc和free。

- 你对内核代码的修改可能包含以下内容：

  - 增加内存控制块数据结构对分配的内存块进行管理。

  - 修改process的数据结构以扩展对虚拟地址空间的管理，后续对于heap的扩展，需要对新增的虚拟地址添加对应的物理地址映射。

  - 设计函数对进程的虚拟地址空间进行管理，借助以上内容具体实现heap扩展。

  - 设计malloc函数和free函数对内存块进行管理，要求实现一个内存池，当申请的空间小于等于内存池中空闲的内存块的大小时，能够直接使用空闲内存块，否则重新申请内存块，并放入内存池中。

  
    

**注意：本挑战的创新思路及难点就在于分配和回收策略的设计（对应malloc和free），读者应同时思考两个函数如何实现，基于紧凑性和高效率的设计目标，设计自己认为高效的分配策略。完成设计后，请读者另外编写应用，设计不同场景使用better_malloc和better_free函数，验证挑战目标以及对自己实现进行检测。**

**另外，后续的基础实验代码并不依赖挑战实验，所以读者可自行决定是否将自己的工作提交到本地代码仓库中（当然，提交到本地仓库是个好习惯，至少能保存自己的“作品”）。**
