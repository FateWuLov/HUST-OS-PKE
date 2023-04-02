# （答案）挑战一：进程等待和数据段复制

### ——对应PKE文档的5.5 lab3_challenge1

#### 实验目的

* 学习wait系统调用，通过实践了解其在内核中的实现
* 深化对PKE进程调度的理解
* 理解fork复制进程数据的过程，加深对虚拟内存机制的体会

---

在lab3_3_rrsched实验的基础上，达到挑战一的实验需求，需要完成以下几方面的内容：

#### 内核函数：

本答案将wait功能中负责阻塞的逻辑放在用户态，内核态的wait函数只负责查询等待的进程号是否合法和是否已退出。

在process.c里添加wait函数：

```c
236 // wait child process exit
237 // pid == -1 means waiting for any child to exit
238 // pid > 0 means waiting for the child whose pid
239 // equal to the parameter to exit
240 int wait(int pid) {
241     if (pid == -1) {
242         int f = 0;
243         for (int i = 0; i < NPROC; i++)
244             if (procs[i].parent == current) {
245                 f = 1;
246                 if (procs[i].status == ZOMBIE) {
247                     procs[i].status = FREE; return i;
248                 }
249             }
250         if (f) return -2; else return -1;
251     } else if (pid < NPROC) {
252         if (procs[pid].parent != current) return -1;
253         else {
254             if (procs[pid].status == ZOMBIE) {
255                 procs[pid].status = FREE; return pid;
256             } else  return -2;
257         }
258     } else return -1;
259 }
```

我们约定，对于该wait函数，返回值大于0表示等待结束，返回值为等待的子进程的pid；返回值为-2表示等待的子进程存在，但还没有运行结束；返回值为-1表示等待的子进程不存在。因此在该函数中：

* 对于pid为-1的情况（241行）：
  * 如果当前进程有一个子进程运行结束了（状态为ZOMBIE）（246到248行），则返回该进程的pid；
  * 否则如果当前进程有子进程但所有子进程都未运行结束，返回-2（f为1，即有进程满足244行的if条件但这些进程都不满足231行的if条件）；
  * 否则返回-1（f为0，即所有进程都不满足244行的if条件）。
* 对于pid大于0的情况（251行）：
  * 如果pid对应的进程为当前进程的子进程，且该进程运行已结束（状态为ZOMBIE），返回pid（254到255行）；
  * 如果pid对应的进程为当前进程的子进程，且该进程运行未结束，返回-2（256行）；
  * 如果pid对应的进程不为当前进程的子进程，或pid不合法，返回-1（252行和258行）。

在实验3_1，我们在do_fork函数中实现了代码段的复制，在本实验中需要实现数据段的复制。数据段复制的操作大体和代码段复制相同，但有一点不同，就是数据段复制时子进程的所有虚拟页帧都需要映射到新的物理页帧，而不能直接映射到父进程的物理页帧：

```c
210       case DATA_SEGMENT:
211         for( int j=0; j<parent->mapped_info[i].npages; j++ ){
212             uint64 addr = lookup_pa(parent->pagetable, parent->mapped_info[i].va+j*PGSIZE);
213             char *newaddr = alloc_page(); memcpy(newaddr, (void *)addr, PGSIZE);
214             map_pages(child->pagetable, parent->mapped_info[i].va+j*PGSIZE, PGSIZE,
215                     (uint64)newaddr, prot_to_type(PROT_WRITE | PROT_READ, 1));
216         }
217 
218         // after mapping, register the vm region (do not delete codes below!)
219         child->mapped_info[child->total_mapped_region].va = parent->mapped_info[i].va;
220         child->mapped_info[child->total_mapped_region].npages = 
221           parent->mapped_info[i].npages;
222         child->mapped_info[child->total_mapped_region].seg_type = DATA_SEGMENT;
223         child->total_mapped_region++;
224         break;
225     }
226   }
```

第213行新申请了一个物理页帧，把父进程物理页帧的内容复制过来，然后再将子进程的虚拟页帧映射到刚申请的这个物理页帧（214到215行）。

#### 用户函数

在user_lib.c里添加wait函数：

```c
 80 //
 81 // lib call to wait
 82 //
 83 int wait(int pid) {
 84     for (;;) {
 85         int r = do_user_call(SYS_user_wait, pid, 0, 0, 0, 0, 0, 0);
 86         if (r != -2) return r; else yield();
 87     }
 88 }     
```

即调用内核的wait函数，如果返回值为-2则通过yield系统调用执行其他用户进程，然后再继续循环，否则直接返回。

#### 系统调用

为了把wait添加为系统调用函数，还需修改syscall.c、syscall.h等文件，方法和前面的实验类似，此处不再赘述。

#### 关于“标答”的说明

“标答”只是给出了一种实现的可能，学生可能给出不一样的解决方案。例如，把循环等待子进程退出和执行其他用户进程的操作放在内核态的wait函数中。只要能实现挑战的设计目标，这些其他的解决方案就都是正确的。

从解决问题的方法的角度，以上给出的解决方案是较为科学和简洁的方法，所有解答至少应涉及以上部分。