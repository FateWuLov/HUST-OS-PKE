# （答案）挑战二：实现信号量

### ——对应PKE文档的5.6 lab3_challenge2


#### 实验目的

* 学习信号量，通过实践了解其在内核中的实现
* 深化对PKE进程调度的理解
* 体会数据结构（链表和队列）在内核编写中的作用

---

在lab3_3_rrsched实验的基础上，达到挑战二的实验需求，需要完成以下几方面的内容：

#### 数据结构：

在process.h里添加信号量结构体：

```c
 92 typedef struct {   
 93     int value, occupied;
 94     process *wait_head, *wait_tail;
 95 } semaphore;       
```

value表示信号量的值，occupied表示当前信号量是否已被使用，在分配信号量时选取的信号量数组中occupied属性为0的信号量。wait_head和wait_tail分别为当前正在等待该信号量的进程队列的队列头指针和队列尾指针。

#### 实现函数

在process.c里添加信号量的申请、释放和增减操作函数：

```c
221 semaphore sems[NPROC];
222 int alloc_sem(int value) {
223     for (int i = 0; i < NPROC; i++)
224         if (! sems[i].occupied) {
225             sems[i].occupied = 1; sems[i].value = value;
226             sems[i].wait_head = sems[i].wait_tail = 0; return i;
227         }
228     return -1;
229 }
230 int inc_sem(int sem) {
231     if (sem < 0 || sem >= NPROC) return -1;
232     sems[sem].value++;
233     if (sems[sem].wait_head) {
234         process *t = sems[sem].wait_head;
235         sems[sem].wait_head = t->queue_next;
236         if (t->queue_next == 0) sems[sem].wait_tail = 0;
237         insert_to_ready_queue(t);
238     }
239     return 0;
240 }
241 int dec_sem(int sem) {
242     if (sem < 0 || sem >= NPROC) return -1;
243     sems[sem].value--;
244     if (sems[sem].value < 0) {
245         if (sems[sem].wait_head == 0) {
246             sems[sem].wait_head = sems[sem].wait_tail = current;
247             current->queue_next = 0;
248         } else {
249             sems[sem].wait_tail->queue_next = current->queue_next;
250             sems[sem].wait_tail = current;
251         }
252         current->status = BLOCKED; schedule();
253     }
254     return 0;
255 }                        
```

信号量的申请比较简单，重点是PV操作：

* 当信号量增加（V操作）时，首先更新信号量的值（232行），然后检查等待该信号量的进程队列（233行），如果队列不为空，则取出队头进程（234-236行），加入到调度队列中（237行）。
* 当信号量减小（P操作）时，首先更新信号量的值（243行），如果信号量的值小于0（244行），说明当前进程需阻塞并等待信号量增加，所以将其状态修改为BLOCKED（252行），加入到该信号量的等待进程队列中（245行到251行），然后触发CPU的调度（252行），由于此时该进程已不再调度队列中，所以CPU会执行其他用户程序。

#### 系统调用

为了把信号量相关操作添加为系统调用函数，还需修改user_lib.c、user_lib.h、syscall.c、syscall.h等文件，方法和前面的实验类似，此处不再赘述。

#### 关于“标答”的说明

“标答”只是给出了一种实现的可能，学生可能给出不一样的解决方案。例如，把循环等待子进程退出和执行其他用户进程的操作放在内核态的wait函数中。只要能实现挑战的设计目标，这些其他的解决方案就都是正确的。

从解决问题的方法的角度，以上给出的解决方案是较为科学和简洁的方法，所有解答至少应涉及以上部分。