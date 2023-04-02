# （答案）挑战二：打印异常代码行

### ——对应PKE文档的3.6 lab1_challenge2

#### 实验目的

* 学习调用HTIF提供的函数读取宿主机文件

* 理解elf格式，懂得在elf文件中寻找自己需要的数据
* 大致了解调试信息格式DWARF
* 理解PKE的异常处理过程，知道指令是怎么和源代码对应起来的

---

在lab1_3_irq实验的基础上，达到挑战二的实验需求，需要完成以下几方面的内容：

#### elf和调试信息解析

对elf_load函数进行修改。一方面需要添加maxva变量存储程序各段虚拟地址的最大值，另一方面需要找到debug_line段，把它保存起来并调用构造表的函数。

```c
195 elf_status elf_load(elf_ctx *ctx) {
196   elf_prog_header ph_addr;
197   int i, off; uint64 maxva = 0;
198   // traverse the elf program segment headers
199   for (i = 0, off = ctx->ehdr.phoff; i < ctx->ehdr.phnum; i++, off += sizeof(ph_addr)) {
200     // read segment headers
201     if (elf_fpread(ctx, (void *)&ph_addr, sizeof(ph_addr), off) != sizeof(ph_addr)) return EL_EIO;
202 
203     if (ph_addr.type != ELF_PROG_LOAD) continue;
204     if (ph_addr.memsz < ph_addr.filesz) return EL_ERR;
205     if (ph_addr.vaddr + ph_addr.memsz < ph_addr.vaddr) return EL_ERR;
206 
207     // allocate memory before loading
208     void *dest = elf_alloc_mb(ctx, ph_addr.vaddr, ph_addr.vaddr, ph_addr.memsz);
209 
210     // actual loading
211     if (elf_fpread(ctx, dest, ph_addr.memsz, ph_addr.off) != ph_addr.memsz)
212       return EL_EIO;
213     if (ph_addr.vaddr + ph_addr.memsz > maxva) maxva = ph_addr.vaddr + ph_addr.memsz;
214   }
215 
216   char name[16]; ((elf_info *)ctx->info)->p->debugline = NULL;
217   elf_sect_header name_seg, tmp_seg;
218   // read name segment
219   if (elf_fpread(ctx, (void *)&name_seg, sizeof(name_seg),
220               ctx->ehdr.shoff + ctx->ehdr.shstrndx * sizeof(name_seg)) != sizeof(name_seg)) return EL_EIO;
221   // find ".debug_line" segment
222   for (i = 0, off = ctx->ehdr.shoff; i < ctx->ehdr.shnum; i++, off += sizeof(tmp_seg)) {
223       if (elf_fpread(ctx, (void *)&tmp_seg, sizeof(tmp_seg), off) != sizeof(tmp_seg)) return EL_EIO;
224       elf_fpread(ctx, (void *)name, 20, name_seg.offset + tmp_seg.name);
225       if (strcmp(name, ".debug_line") == 0) {
226           if (elf_fpread(ctx, (void *)maxva, tmp_seg.size, tmp_seg.offset) != tmp_seg.size) return EL_EIO;
227           make_addr_line(ctx, (char *)maxva, tmp_seg.size); break;
228       }
229   }
230                             
231   return EL_OK;
232 }
```

这里用maxva变量表示程序所有需映射的段数据的最大虚拟地址（213行）。接下来需要读取一个专门存储各段名称的段的段头（219到220行）。接下来了枚举所有段（222行），读取段头（223行），利用段头里包含的段名索引和前面得到的各段名称的段的段头，读取该段名称（224行），如果段名为.debug_line（225行），则把这个段的实际数据读下来放到maxva处（226行），对该数据调用make_addr_line函数（227行）。

#### 中断处理

在line_print函数中，首先根据文件索引找到文件名和文件夹索引，再根据文件夹索引找到文件夹名，文件夹名和文件名拼接成触发异常的源代码文件路径(35到37行），就可以利用htif功能读入源代码文件（41行到43行，需要利用file_stat函数获得文件大小），把对应行号的代码输出（44行到51行，需要根据换行符计算行号）：

```c
 27 char path[128], code[10240]; struct stat mystat;
 28 //
 29 // Parameter is the entry of array "process->line".
 30 // The function prints the code line the entry points to.
 31 //
 32 void line_print(addr_line *line) {
 33     int l = strlen(current->dir[current->file[line->file].dir]);
 34     // concat the directory and file to construct path
 35     strcpy(path, current->dir[current->file[line->file].dir]);
 36     path[l] = '/';
 37     strcpy(path + l + 1, current->file[line->file].file);
 38     // sprint(path);
 39     
 40     // read the code line and print
 41     spike_file_t *f = spike_file_open(path, O_RDONLY, 0);
 42     spike_file_stat(f, &mystat); spike_file_read(f, code, mystat.st_size);
 43     spike_file_close(f); int off = 0, cnt = 0;
 44     while (off < mystat.st_size) {
 45         int x = off; while (x < mystat.st_size && code[x] != '\n') x++;
 46         if (cnt == line->line - 1) {
 47             code[x] = '\0';
 48             sprint("Runtime error at %s:%d\n%s\n", path, line->line, code + off);
 49             break;
 50         } else cnt++, off = x + 1;
 51     }
 52 }
```

error_print函数根据mepc寄存器获得触发异常的指令地址（58行），根据地址找到地址-行号-文件索引表项（61行），调用line_print函数（62行）：

```c
 53 //
 54 // Find the "process->line" array entry according to
 55 // the exception intrustion address stores in epc register.
 56 //
 57 void error_print() {
 58     uint64 mepc = read_csr(mepc);
 59     for (int i = 0; i < current->line_ind; i++) {
 60         // find the exception line table entry
 61         if (mepc < current->line[i].addr) {
 62             line_print(current->line + i - 1); break;
 63         }
 64     }
 65 }
```

最后是在handle_mtrap中所有需要打印异常代码行的case下调用error_print函数。

```c
 70 void handle_mtrap() {
 71   uint64 mcause = read_csr(mcause);
 72   switch (mcause) {
 73     case CAUSE_MTIMER:
 74       handle_timer();
 75       break;
 76     case CAUSE_FETCH_ACCESS:
 77       error_print();
 78       handle_instruction_access_fault();
 79       break;
 80     case CAUSE_LOAD_ACCESS:
 81       error_print();
 82       handle_load_access_fault();
 83     case CAUSE_STORE_ACCESS:
 84       error_print();
 85       handle_store_access_fault();
 86       break;
 87     case CAUSE_ILLEGAL_INSTRUCTION:
 88       error_print();
 89       handle_illegal_instruction();
 90       break;
 91     case CAUSE_MISALIGNED_LOAD:    
  92       error_print();
 93       handle_misaligned_load();
 94       break;
 95     case CAUSE_MISALIGNED_STORE:
 96       error_print();
 97       handle_misaligned_store();
 98       break;
 99 
100     default:
101       sprint("machine trap(): unexpected mscause %p\n", mcause);
102       sprint("            mepc=%p mtval=%p\n", read_csr(mepc), read_csr(mtva
103       panic( "unexpected exception happened in M-mode.\n" );
104       break;
105   }
106 }
```

#### 关于“标答”的说明

“标答”只是给出了一种实现的可能，学生可能给出不一样的解决方案。例如，把各表存储在内核的静态数组中。只要能实现挑战的设计目标，这些其他的解决方案就都是正确的。

从解决问题的方法的角度，以上给出的解决方案是较为科学和简洁的方法，所有解答至少应涉及以上部分。