# （答案）挑战一：打印用户程序调用栈

### ——对应PKE文档的3.4 lab1_challenge1



在lab1_3_irq实验的基础上，达到挑战一的实验需求，需要完成以下4个方面的内容：



#### 添加系统调用：

user/app_print_backtrace.c程序中的print_backtrace()函数，以及其对应的系统调用并未在PKE用户函数库以及内核中实现。这样就要求在用户函数库文件（user/user_lib.c和user/user_lib.h），以及内核的系统调用实现（kernel/syscall.c和kernel/syscall.h）。

- 在user/user_lib.h文件末尾添加print_backtrace()函数的原型：

```
  8 // added for lab1_challenge1_backtrace.
  9 // predeclaration.
 10 int print_backtrace(int depth);
```



- 在user/user_lib.c文件末尾添加print_backtrace()函数的实现：

```
 53 //
 54 // added for lab1_challenge1_backtrace.
 55 // protype definition for print_backtrace().
 56 //
 57 int print_backtrace(int depth) {
 58   return do_user_call(SYS_user_backtrace, depth, 0, 0, 0, 0, 0, 0);
 59 }
```

- 在kernel/syscall.h文件添加一个SYS_user_backtrace的ID号：

```
11 #define SYS_user_backtrace (SYS_user_base + 2)
```

- 在kernel/syscall.c文件中添加SYS_user_backtrace系统调用的实现（以下的第105--106行）：

```C
 99 long do_syscall(long a0, long a1, long a2, long a3, long a4, long a5, long a6, long a7) {
100   switch (a0) {
101     case SYS_user_print:
102       return sys_user_print((const char*)a1, a2);
103     case SYS_user_exit:
104       return sys_user_exit(a1);
105     case SYS_user_backtrace:
106       return sys_user_backtrace(a1);
107     default:
108       panic("Unknown syscall %ld \n", a0);
109   }
110 }
```

这样，系统调用就添加好了，接下来，我们需要实现SYS_user_backtrace系统调用的具体实现函数sys_user_backtrace()。



#### 获取用户态栈：

```C
 59 //
 60 // added for lab1_challenge1_backtrace.
 61 // the function first locates the stack of user app by current->trapframe->regs.sp, and
 62 // then back traces the function calls via the return addresses stored in the user stack.
 63 // Note: as we consider only functions with no parameters in this challenge, we jump for
 64 // exactly 16 bytes during the back tracing.
 65 //
 66 ssize_t sys_user_backtrace(int64 depth) {
 67   // aquire the user stack, and bypass the latest call to print_backtrace().
 68   // 16 means length of leaf function (i.e., print_backtrace)'s call stack frame, where
 69   // no return address is available, 8 means length of the margin space in previous
 70   // function's call stack frame.
 71   uint64 user_sp = current->trapframe->regs.sp + 16 + 8;
 72
 73   // back trace user stack, lookup the symbol names of the return addresses.
 74   // Note: by priciple, we should traverse the stack via "fp" here. However, as we
 75   // consider the simple case where functions in the path have NO parameters, we can do
 76   // it by simply bypassing 16 bytes at each depth.
 77   // the traverse direction is from lower addresses to higher addressese.
 78   int64 actual_depth=0;
 79   for (uint64 p = user_sp; actual_depth<depth; ++actual_depth, p += 16) {
 80     // the return address is stored in (uint64*)p
 81     if (*(uint64*)p == 0) break; // end of user stack?
 82     // look up the symbol name of the given return address
 83     int symbol_idx = backtrace_symbol(*(uint64*)p);
 84     if (symbol_idx == -1) {
 85       sprint("fail to backtrace symbol %lx\n", *(uint64*)p);
 86       continue;
 87     }
 88     // print the function name.
 89     sprint("%s\n", &g_elfloader.strtb[g_elfloader.syms[symbol_idx].st_name]);
 90   }
 91
 92   return 0;
 93 }
```

在实现该函数时，首先（第71行）我们需要获得用户进程的栈底指针。由于sys_user_backtrace函数在执行时系统已处于操作系统中（即S模式），通过实验1的基础实验我们知道，PKE环境中应用在发出系统调用，系统进入内核态后会切换栈到“用户内核”栈，不再使用用户态栈，所以，此时获取用户态栈必须通过trapframe中存储的进程上下文中获取。所以，在第71行sys_user_backtrace函数通过current->trapframe->regs.sp寄存器中的内容获得用户态栈的栈底。

获得用户态栈后，我们需要进一步得到存储在栈中的应用函数调用的返回地址。这一步，我们在下面讨论。

#### 用户态栈中函数返回地址的获取：

通过PKE实验文档的[第一章](https://gitee.com/hustos/pke-doc/blob/master/chapter1_riscv.md#call_stack_structure)中的一个例子，我们知道，对于user/app_print_backtrace.c应用而言，它的执行会产生如下图的栈结构：



<img src="ans-pictures/stack_frame_of_app.png" alt="fig1_2" style="zoom:100%;" />

其中对于f8函数，由于是最后一个函数调用，所以它的栈帧里并未存放f8函数的返回地址。而其他函数的栈帧里，都存放了返回这些函数的地址ra。

函数调用回溯的基本思路是，通过用户态栈的结构，取得各函数调用的返回地址ra，并通过对比函数的逻辑地址范围（看ra是否落在某函数的逻辑地址范围），来反向解析ra对应的函数名称，即sys_user_backtrace函数的79--90行所作的工作。

另外，根据以上图示可知，回溯过程的第一个ra地址，距离sp的长度为16+8个字节，这也就是sys_user_backtrace函数在取得用户态栈时，会在第71行对trapframe中的sp进行+16+8的原因。



####  程序的符号表、字符串表以及逻辑地址到符号的解析

PKE的构造文件将应用的源代码编译成ELF文件时，会在ELF文件中加入符号表和字符串表，这些表能够通过riscv64-unknown-elf-readelf命令查看：

```bash
$ riscv64-unknown-elf-readelf -S ./obj/app_print_backtrace
There are 17 section headers, starting at offset 0x3998:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000081000000  00001000
       0000000000000316  0000000000000000  AX       0     0     2
  [ 2] .rodata.str1.8    PROGBITS         0000000081000318  00001318
       000000000000002b  0000000000000001 AMS       0     0     8
  [ 3] .rodata           PROGBITS         0000000081000344  00001344
       0000000000000058  0000000000000000   A       0     0     4
  [ 4] .debug_info       PROGBITS         0000000000000000  0000139c
       00000000000007b1  0000000000000000           0     0     1
  [ 5] .debug_abbrev     PROGBITS         0000000000000000  00001b4d
       000000000000027e  0000000000000000           0     0     1
  [ 6] .debug_aranges    PROGBITS         0000000000000000  00001dcb
       0000000000000090  0000000000000000           0     0     1
  [ 7] .debug_line       PROGBITS         0000000000000000  00001e5b
       000000000000099e  0000000000000000           0     0     1
  [ 8] .debug_str        PROGBITS         0000000000000000  000027f9
       00000000000001cc  0000000000000001  MS       0     0     1
  [ 9] .comment          PROGBITS         0000000000000000  000029c5
       0000000000000011  0000000000000001  MS       0     0     1
  [10] .riscv.attributes RISCV_ATTRIBUTE  0000000000000000  000029d6
       0000000000000035  0000000000000000           0     0     1
  [11] .debug_frame      PROGBITS         0000000000000000  00002a10
       0000000000000258  0000000000000000           0     0     8
  [12] .debug_loc        PROGBITS         0000000000000000  00002c68
       000000000000085e  0000000000000000           0     0     1
  [13] .debug_ranges     PROGBITS         0000000000000000  000034c6
       00000000000000b0  0000000000000000           0     0     1
  [14] .symtab           SYMTAB           0000000000000000  00003578
       00000000000002e8  0000000000000018          15    17     8
  [15] .strtab           STRTAB           0000000000000000  00003860
       000000000000007d  0000000000000000           0     0     1
  [16] .shstrtab         STRTAB           0000000000000000  000038dd
       00000000000000b9  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

输出结果中，.symtab就是符号节（section），.strtab就是字符串节。

符号节.symtab由一组符号记录组成，一个记录的结构如下（见标答的kernel/elf.h文件，该结构也可以从linux源代码的elf.h文件中获得）：

```C
 74 // added for lab1_challenge1_backtrace.
 75 // the symbol table record, each record stores information of a symbol
 76 typedef struct {
 77   // st_name is the offset in string table.
 78   uint32 st_name;         /* Symbol name (string tbl index) */
 79   unsigned char st_info;  /* Symbol type and binding */
 80   unsigned char st_other; /* Symbol visibility */
 81   uint16 st_shndx;        /* Section index */
 82   uint64 st_value;        /* Symbol value */
 83   uint64 st_size;         /* Symbol size */
 84 } elf_symbol_rec;
```

其中st_name并不是一个字符串,而是一个指向字符串表的索引值，在字符串表中对应位置上的字符串就是该符号名字的实际文本。如果此值为非 0，它代表符号名在字符串表中的索引值。如果此值为 0,那么此符号无名字。st_value是符号名对应的虚拟地址，st_size表示符号名对应的虚地址空间的大小。符号结构中的这几个成员，在对应用程序函数调用回溯时起到非常关键的作用。

为了实现对应用程序函数调用的回溯，标答中在kernel/elf.c文件中定义了elf_load_symbol函数，实现了对.symtab和.strtab两个节的加载。加载的目的地址放在用于elf加载的数据结构g_elfloader中。

```c
145 // added for lab1_challenge1_backtrace.
146 // load the symbol table section into memory.
147 //
148 elf_status elf_load_symbol(elf_ctx *ctx) {
149   elf_section_header sh;
150   int i, off;
151   int strsize = 0;
152   // load symbol table and string table
153   for (int i = 0, off = ctx->ehdr.shoff; i < ctx->ehdr.shnum;
154     ++i, off += sizeof(elf_section_header)) {
155     if (elf_fpread(ctx, (void *)&sh, sizeof(sh), off) != sizeof(sh)) return EL_EIO;
156     if (sh.sh_type == SHT_SYMTAB) {  // symbol table
157       // symbols are palced in sequential order
158       if (elf_fpread(ctx, &ctx->syms, sh.sh_size, sh.sh_offset) != sh.sh_size)
159         return EL_EIO;
160       ctx->syms_count = sh.sh_size / sizeof(elf_symbol_rec);
161     } else if (sh.sh_type == SHT_STRTAB) {  // string table
162       if (elf_fpread(ctx, &ctx->strtb + strsize, sh.sh_size, sh.sh_offset) != sh.sh_size)
163         return EL_EIO;
164       strsize += sh.sh_size;  //there may be several string tables
165     }
166   }
167
168   return EL_OK;
169 }
```

回到sys_user_backtrace函数，在它的79--90行就实现了对加载后符号节和字符串节中数据的查找过程。

#### 关于“标答”的说明

“标答”只是给出了一种实现的可能，学生可能给出不一样的解决方案。例如，将对符号表和字符串节的解析全部放在kernel/syscall.c中。只要能实现挑战的设计目标，这些其他的解决方案就都是正确的。

从解决问题的方法的角度，以上给出的解决方案是较为科学和简洁的方法，所有解答至少应涉及以上部分。

