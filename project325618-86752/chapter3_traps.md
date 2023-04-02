# 第三章．实验1：系统调用、异常和外部中断

- [3.1 lab1_challenge1 打印用户程序调用栈（难度：&#9733;&#9733;&#9734;&#9734;&#9734;）](#lab1_challenge1_backtrace) 
  - [给定应用](#lab1_challenge1_app)
  - [实验内容](#lab1_challenge1_content)
  - [实验指导](#lab1_challenge1_guide)
- [3.2 lab1_challenge2 打印异常代码行（难度：&#9733;&#9733;&#9733;&#9733;&#9734;）](#lab1_challenge2_errorline) 
  - [给定应用](#lab1_challenge2_app)
  - [实验内容](#lab1_challenge2_content)
  - [实验指导](#lab1_challenge2_guide)

<a name="lab1_challenge1_backtrace"></a>

## 3.1 lab1_challenge1 打印用户程序调用栈（难度：&#9733;&#9733;&#9734;&#9734;&#9734;）

<a name="lab1_challenge1_app"></a>

#### **给定应用**

- user/app_print_backtrace.c

```c
  1 /*
  2  * Below is the given application for lab1_challenge1_backtrace.
  3  * This app prints all functions before calling print_backtrace().
  4  */
  5
  6 #include "user_lib.h"
  7 #include "util/types.h"
  8
  9 void f8() { print_backtrace(7); }
 10 void f7() { f8(); }
 11 void f6() { f7(); }
 12 void f5() { f6(); }
 13 void f4() { f5(); }
 14 void f3() { f4(); }
 15 void f2() { f3(); }
 16 void f1() { f2(); }
 17
 18 int main(void) {
 19   printu("back trace the user app in the following:\n");
 20   f1();
 21   exit(0);
 22   return 0;
 23 }
```

以上程序在真正调用系统调用print_backtrace(7)之前的函数调用关系比复杂，图示起来有以下关系：

main -> f1 -> f2 -> f3 -> f4 -> f5 -> f6 -> f7 -> f8

print_backtrace(7)的作用是将以上用户程序的函数调用关系，从最后的f8向上打印7层，预期的输出为：

```bash
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
Application: obj/app_print_backtrace
Application program entry point (virtual address): 0x0000000081000072
Switching to user mode...
back trace the user app in the following:
f8
f7
f6
f5
f4
f3
f2
User exit with code:0.
System is shutting down with exit code 0.
```

<a name="lab1_challenge1_content"></a>

####  实验内容

本实验为挑战实验，基础代码将继承和使用lab1_3完成后的代码：

- （先提交lab1_3的答案，然后）切换到lab1_challenge1_backtrace、继承lab1_3中所做修改：

```bash
//切换到lab1_challenge1_backtrace
$ git checkout lab1_challenge1_backtrace

//继承lab1_3以及之前的答案
$ git merge lab1_3_irq -m "continue to work on lab1_challenge1"
```

注意：**不同于基础实验，挑战实验的基础代码具有更大的不完整性，可能无法直接通过构造过程。**例如，由于以上的用户代码中print_backtrace()系统调用并未实现，所以构造时就会报错。同样，不同于基础实验，我们在代码中也并未专门地哪些地方的代码需要填写，哪些地方的代码无须填写。这样，我们留给读者更大的“想象空间”。

- 本实验的具体要求为：

通过修改PKE内核，来实现从给定应用（user/app_print_backtrace.c）到预期输出的转换。

对于print_backtrace()函数的实现要求：

应用程序调用print_backtrace()时，应能够通过控制输入的参数（如例子user/app_print_backtrace.c中的7）控制回溯的层数。例如，如果调用print_backtrace(5)则只输出5层回溯；如果调用print_backtrace(100)，则应只回溯到main函数就停止回溯（因为调用的深度小于100）。

- 讨论的简化

实验可以对中间函数，如user/app_print_backtrace.c中的f1到f8，进行讨论上的简化，可假设它们都不带参数。



<a name="lab1_challenge1_guide"></a>

####  实验指导

为完成该挑战，PKE内核的完善应包含以下内容：

- 系统调用路径上的完善，可参见[3.2](#syscall)中的知识；
- 在操作系统内核中获取用户程序的栈。这里需要注意的是，PKE系统中当用户程序通过系统调用陷入到内核时，会切换到S模式的“用户内核”栈，而不是在用户栈上继续操作。我们的print_backtrace()函数的设计目标是回溯并打印用户进程的函数调用情况，所以，进入操作系统内核后，需要找到用户进程的用户态栈来开始回溯；
- 找到用户态栈后，我们需要了解用户态栈的结构。实际上，这一点在我们的第一章就有[举例](chapter1_riscv.md#call_stack_structure)来说明，读者可以回顾一下第一章的例子。另外，由于我们可以对讨论进行简化（即假设中间函数无参数），那么单次函数调用的栈深度我们也可以进行相应的假设；
- 通过用户栈找到函数的返回地址后，需要将虚拟地址转换为源程序中的符号。这一点，读者需要了解ELF文件中的符号节（.symtab section），以及字符串节（.strtab section）的相关知识，了解这两个节（section）里存储的内容以及存储的格式等内容。对ELF的这两个节，网上有大量的介绍，例如[这里](https://blog.csdn.net/edonlii/article/details/8779075)。

**注意：完成实验内容后，请读者另外编写应用，通过调用print_backtrace()函数，并带入不同的深度参数，对自己的实现进行检测。**

**另外，后续的基础实验代码并不依赖挑战实验，所以读者可自行决定是否将自己的工作提交到本地代码仓库中（当然，提交到本地仓库是个好习惯，至少能保存自己的“作品”）。**



<a name="lab1_challenge2_errorline"></a>

## 3.2 lab1_challenge2 打印异常代码行（难度：&#9733;&#9733;&#9733;&#9733;&#9734;）

<a name="lab1_challenge2_app"></a>

#### **给定应用**

- user/app_errorline.c（和lab1_2的应用一致）

```c
  1 /*                                                                             
  2  * Below is the given application for lab1_challenge2 (same as lab1_2).
  3  * This app attempts to issue M-mode instruction in U-mode, and consequently raises an exception.
  4  */
  5 
  6 #include "user_lib.h"
  7 #include "util/types.h"
  8 
  9 int main(void) {
 10   printu("Going to hack the system by running privilege instructions.\n");
 11   // we are now in U(user)-mode, but the "csrw" instruction requires M-mode privilege.
 12   // Attempting to execute such instruction will raise illegal instruction exception.
 13   asm volatile("csrw sscratch, 0");
 14   exit(0);
 15 }
 16 
```

以上程序试图在用户态读取在内核态才能读取的寄存器sscratch，因此此处会触发illegal instruction异常，你的任务是**修改内核（包括machine文件夹下）的代码，使得用户程序在发生异常时，内核能够输出触发异常的用户程序的源文件名和对应代码行**，如上面的应用预期输出如下：

```bash
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
Application: obj/app_errorline
Application program entry point (virtual address): 0x0000000081000000
Switch to user mode...
Going to hack the system by running privilege instructions.
Runtime error at user/app_errorline.c:13
  asm volatile("csrw sscratch, 0");
Illegal instruction!
System is shutting down with exit code -1.
```

<a name="lab1_challenge2_content"></a>

####  实验内容

本实验为挑战实验，基础代码将继承和使用lab1_3完成后的代码：

- （先提交lab1_3的答案，然后）切换到lab1_challenge2_errorline、继承**lab1_3**（注意，不是继承lab1_challenge1_backtrace！**PKE的挑战实验之间无继承关联**）中所做修改：

```bash
//切换到lab1_challenge2_errorline
$ git checkout lab1_challenge2_errorline

//继承lab1_3以及之前的答案
$ git merge lab1_3_irq -m "continue to work on lab1_challenge1"
```

注意：**不同于基础实验，挑战实验的基础代码具有更大的不完整性，可能无法直接通过构造过程。**同样，不同于基础实验，我们在代码中也并未专门地哪些地方的代码需要填写，哪些地方的代码无须填写。这样，我们留给读者更大的“想象空间”。

- 本实验的具体要求为：通过修改PKE内核（包括machine文件夹下的代码），使得用户程序在发生异常时，内核能够输出触发异常的用户程序的源文件名和对应代码行。
- 注意：虽然在示例的app_errorline.c中只触发了非法指令异常，但最终测试时你的内核也应能够对其他会导致panic的异常和其他源文件输出正确的结果。
- 文件名规范：需要包含路径，如果是用户源程序发生的错误，路径为相对路径，如果是调用的标准库内发生的错误，路径为绝对路径。
- 为了降低挑战的难度，本实验在elf.c中给出了debug_line段的解析函数make_addr_line。这个函数接受三个参数，ctx为elf文件的上下文指针，这个可以参考文件中的其他函数；debug_line为指向.debug_line段数据的指针，你需要读取elf文件中名为.debug_line的段保存到缓冲区中，然后将缓冲区指针传入这个参数；length为.debug_line段数据的长度。
- 函数调用结束后，process结构体的dir、file、line三个指针会各指向一个数组，dir数组存储所有代码文件的文件夹路径字符串指针，如/home/abc/bcd的文件夹路径为/home/abc，本项目user文件夹下的app_errorline.c文件夹路径为user；file数组存储所有代码文件的文件名字符串指针以及其文件夹路径在dir数组中的索引；line数组存储所有指令地址，代码行号，文件名在file数组中的索引三者的映射关系。如某文件第3行为a = 0，被编译成地址为0x1234处的汇编代码li ax, 0和0x1238处的汇编代码sd 0(s0), ax。那么file数组中就包含两项，addr属性分别为0x1234和0x1238，line属性为3，file属性为“某文件”的文件名在file数组中的索引。
- 注意：dir、file、line三个数组会依次存储在debug_line数据缓冲区之后，dir数组和file数组的大小为64。所以如果你用静态数组来存储debug_line段数据，那么这个数组必须足够大；或者你也可以把debug_line直接放在程序所有需映射的段数据之后，这样可以保证有足够大的动态空间。

<a name="lab1_challenge2_guide"></a>

####  实验指导

* 为完成该挑战，需要利用用户程序编译时产生的**调试信息**，目前最广泛使用的调试信息格式是DWARF，可以参考[这里](https://wiki.osdev.org/DWARF)了解其格式，该网站的参考文献中也给出了DWARF的完整文档地址，必要时可参考。
* 你对内核代码的修改可能包含以下内容：
  * 修改读取elf文件的代码，找到包含调试信息的段，将其内容保存起来（可以保存在用户程序的地址空间中）
  * 在适当的位置调用debug_line段解析函数，对调试信息进行解析，构造指令地址-源代码行号-源代码文件名的对应表，注意，连续行号对应的不一定是连续的地址，因为一条源代码可以对应多条指令。
  * 在异常中断处理函数中，通过相应寄存器找到触发异常的指令地址，然后在上述表中查找地址对应的源代码行号和文件名输出

**另外，后续的基础实验代码并不依赖挑战实验，所以读者可自行决定是否将自己的工作提交到本地代码仓库中（当然，提交到本地仓库是个好习惯，至少能保存自己的“作品”）。**