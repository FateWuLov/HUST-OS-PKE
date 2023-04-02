# 第五章．实验3：进程管理

### 目录 

- [5.1 lab3_challenge1 进程等待和数据段复制（难度：&#9733;&#9733;&#9734;&#9734;&#9734;）](#lab3_challenge1_wait) 
  - [给定应用](#lab3_challenge1_app)
  - [实验内容](#lab3_challenge1_content)
  - [实验指导](#lab3_challenge1_guide)
- [5.2 lab3_challenge2 实现信号量（难度：&#9733;&#9733;&#9733;&#9734;&#9734;）](#lab3_challenge2_semaphore) 
  - [给定应用](#lab3_challenge2_app)
  - [实验内容](#lab3_challenge2_content)
  - [实验指导](#lab3_challenge2_guide)

<a name="lab3_challenge1_wait"></a>

## 5.1 lab3_challenge1 进程等待和数据段复制（难度：&#9733;&#9733;&#9734;&#9734;&#9734;）

<a name="lab3_challenge1_app"></a>

#### **给定应用**

- user/app_wait.c

```c
  1 /*                                                                             
  2  * This app fork a child process, and the child process fork a grandchild process.
  3  * every process waits for its own child exit then prints.                     
  4  * Three processes also write their own global variables "flag"
  5  * to different values.
  6  */
  7 
  8 #include "user/user_lib.h"
  9 #include "util/types.h"
 10 
 11 int flag;
 12 int main(void) {
 13     flag = 0;
 14     int pid = fork();
 15     if (pid == 0) {
 16         flag = 1;
 17         pid = fork();
 18         if (pid == 0) {
 19             flag = 2;
 20             printu("Grandchild process end, flag = %d.\n", flag);
 21         } else {
 22             wait(pid);
 23             printu("Child process end, flag = %d.\n", flag);
 24         }
 25     } else {
 26         wait(-1);
 27         printu("Parent process end, flag = %d.\n", flag);
 28     }
 29     exit(0);
 30     return 0;
 31 }
```

wait系统调用是进程管理中一个非常重要的系统调用，它主要有两大功能：

* 当一个进程退出之后，它所占用的资源并不一定能够立即回收，比如该进程的内核栈目前就正用来进行系统调用处理。对于这种问题，一种典型的做法是当进程退出的时候内核立即回收一部分资源并将该进程标记为僵尸进程。由父进程调用wait函数的时候再回收该进程的其他资源。
* 父进程的有些操作需要子进程运行结束后获得结果才能继续执行，这时wait函数起到进程同步的作用。

在以上程序中，父进程把flag变量赋值为0，然后fork生成一个子进程，接着通过wait函数等待子进程的退出。子进程把自己的变量flag赋值为1，然后fork生成孙子进程，接着通过wait函数等待孙子进程的退出。孙子进程给自己的变量flag赋值为2并在退出时输出信息，然后子进程退出时输出信息，最后父进程退出时输出信息。由于fork之后父子进程的数据段相互独立（同一虚拟地址对应不同的物理地址），子进程对全局变量的赋值不影响父进程全局变量的值，因此结果如下：

```bash
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080009000, PKE kernel size: 0x0000000000009000 .
free physical memory address: [0x0000000080009000, 0x0000000087ffffff] 
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switch to user mode...
in alloc_proc. user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000 
User application is loading.
Application: obj/app_wait
CODE_SEGMENT added at mapped info offset:3
DATA_SEGMENT added at mapped info offset:4
Application program entry point (virtual address): 0x00000000000100b0
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User call fork.
will fork a child from parent 0.
in alloc_proc. user frame 0x0000000087fae000, user stack 0x000000007ffff000, user kstack 0x0000000087fad000 
do_fork map code segment at pa:0000000087fb2000 of parent to child at va:0000000000010000.
going to insert process 1 to ready queue.
going to insert process 0 to ready queue.
going to schedule process 1 to run.
User call fork.
will fork a child from parent 1.
in alloc_proc. user frame 0x0000000087fa1000, user stack 0x000000007ffff000, user kstack 0x0000000087fa0000 
do_fork map code segment at pa:0000000087fb2000 of parent to child at va:0000000000010000.
going to insert process 2 to ready queue.
going to insert process 1 to ready queue.
going to schedule process 0 to run.
going to insert process 0 to ready queue.
going to schedule process 2 to run.
Grandchild process end, flag = 2.
User exit with code:0.
going to schedule process 1 to run.
Child process end, flag = 1.
User exit with code:0.
going to schedule process 0 to run.
Parent process end, flag = 0.
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

<a name="lab3_challenge1_content"></a>

####  实验内容

本实验为挑战实验，基础代码将继承和使用lab3_3完成后的代码：

- （先提交lab3_3的答案，然后）切换到lab3_challenge1_wait、继承lab3_3中所做修改：

```bash
//切换到lab3_challenge1_wait
$ git checkout lab3_challenge1_wait

//继承lab3_3以及之前的答案
$ git merge lab3_3_rrsched -m "continue to work on lab3_challenge1"
```

注意：**不同于基础实验，挑战实验的基础代码具有更大的不完整性，可能无法直接通过构造过程。**
同样，不同于基础实验，我们在代码中也并未专门地哪些地方的代码需要填写，哪些地方的代码无须填写。这样，我们留给读者更大的“想象空间”。

- 本实验的具体要求为：
  - 通过修改PKE内核和系统调用，为用户程序提供wait函数的功能，wait函数接受一个参数pid：
    - 当pid为-1时，父进程等待任意一个子进程退出即返回子进程的pid；
    - 当pid大于0时，父进程等待进程号为pid的子进程退出即返回子进程的pid；
    - 如果pid不合法或pid大于0且pid对应的进程不是当前进程的子进程，返回-1。
  - 补充do_fork函数，实验3_1实现了代码段的复制，你需要继续实现数据段的复制并保证fork后父子进程的数据段相互独立。
- 注意：最终测试程序可能和给出的用户程序不同，但都只涉及wait函数、fork函数和全局变量读写的相关操作。

<a name="lab3_challenge1_guide"></a>

####  实验指导

* 你对内核代码的修改可能包含添加系统调用、在内核中实现wait函数的功能以及对do_fork函数的完善。

**注意：完成实验内容后，请读者另外编写应用，对自己的实现进行检测。**

**另外，后续的基础实验代码并不依赖挑战实验，所以读者可自行决定是否将自己的工作提交到本地代码仓库中（当然，提交到本地仓库是个好习惯，至少能保存自己的“作品”）。**

<a name="lab3_challenge2_semaphore"></a>

## 5.2 lab3_challenge2 实现信号量（难度：&#9733;&#9733;&#9733;&#9734;&#9734;）

<a name="lab3_challenge2_app"></a>

#### **给定应用**

- user/app_semaphore.c

```c
  1 /*                                                                                                                                       
  2 * This app create two child process.
  3 * Use semaphores to control the order of
  4 * the main process and two child processes print info. 
  5 */
  6 #include "user/user_lib.h"
  7 #include "util/types.h"
  8 
  9 int main(void) {
 10     int main_sem, child_sem[2];
 11     main_sem = sem_new(1);
 12     for (int i = 0; i < 2; i++) child_sem[i] = sem_new(0);
 13     int pid = fork();
 14     if (pid == 0) {
 15         pid = fork();
 16         for (int i = 0; i < 10; i++) {
 17             sem_P(child_sem[pid == 0]);
 18             printu("Child%d print %d\n", pid == 0, i);
 19             if (pid != 0) sem_V(child_sem[1]); else sem_V(main_sem);
 20         }
 21     } else {
 22         for (int i = 0; i < 10; i++) {
 23             sem_P(main_sem);
 24             printu("Parent print %d\n", i);
 25             sem_V(child_sem[0]);
 26         }
 27     }
 28     exit(0);
 29     return 0;
 30 }
```

如图：

![](pictures/fig_ans_5_6.png)

以上程序通过信号量的增减，控制主进程和两个子进程依次轮转，让输出按主进程，第一个子进程，第二个子进程，主进程，第一个子进程，第二个子进程……这样的顺序轮流输出，如上面的应用预期输出如下：

```bash
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080009000, PKE kernel size: 0x0000000000009000 .
free physical memory address: [0x0000000080009000, 0x0000000087ffffff] 
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switch to user mode...
in alloc_proc. user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000 
User application is loading.
Application: obj/app_semaphore
CODE_SEGMENT added at mapped info offset:3
DATA_SEGMENT added at mapped info offset:4
Application program entry point (virtual address): 0x00000000000100b0
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User call fork.
will fork a child from parent 0.
in alloc_proc. user frame 0x0000000087fae000, user stack 0x000000007ffff000, user kstack 0x0000000087fad000 
do_fork map code segment at pa:0000000087fb2000 of parent to child at va:0000000000010000.
going to insert process 1 to ready queue.
Parent print 0
going to schedule process 1 to run.
User call fork.
will fork a child from parent 1.
in alloc_proc. user frame 0x0000000087fa2000, user stack 0x000000007ffff000, user kstack 0x0000000087fa1000 
do_fork map code segment at pa:0000000087fb2000 of parent to child at va:0000000000010000.
going to insert process 2 to ready queue.
Child0 print 0
going to schedule process 2 to run.
Child1 print 0
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Parent print 1
going to insert process 1 to ready queue.
going to schedule process 1 to run.
Child0 print 1
going to insert process 2 to ready queue.
going to schedule process 2 to run.
Child1 print 1
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Parent print 2
going to insert process 1 to ready queue.
going to schedule process 1 to run.
Child0 print 2
going to insert process 2 to ready queue.
going to schedule process 2 to run.
Child1 print 2
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Parent print 3
going to insert process 1 to ready queue.
going to schedule process 1 to run.
Child0 print 3
going to insert process 2 to ready queue.
going to schedule process 2 to run.
Child1 print 3
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Parent print 4
going to insert process 1 to ready queue.
going to schedule process 1 to run.
Child0 print 4
going to insert process 2 to ready queue.
going to schedule process 2 to run.
Child1 print 4
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Parent print 5
going to insert process 1 to ready queue.
going to schedule process 1 to run.
Child0 print 5
going to insert process 2 to ready queue.
going to schedule process 2 to run.
Child1 print 5
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Parent print 6
going to insert process 1 to ready queue.
going to schedule process 1 to run.
Child0 print 6
going to insert process 2 to ready queue.
going to schedule process 2 to run.
Child1 print 6
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Parent print 7
going to insert process 1 to ready queue.
going to schedule process 1 to run.
Child0 print 7
going to insert process 2 to ready queue.
going to schedule process 2 to run.
Child1 print 7
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Parent print 8
going to insert process 1 to ready queue.
going to schedule process 1 to run.
Child0 print 8
going to insert process 2 to ready queue.
going to schedule process 2 to run.
Child1 print 8
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Parent print 9
going to insert process 1 to ready queue.
User exit with code:0.
going to schedule process 1 to run.
Child0 print 9
going to insert process 2 to ready queue.
User exit with code:0.
going to schedule process 2 to run.
Child1 print 9
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

<a name="lab3_challenge2_content"></a>

####  实验内容

本实验为挑战实验，基础代码将继承和使用lab3_3完成后的代码：

- （先提交lab3_3的答案，然后）切换到lab3_challenge2_semaphore、继承lab3_3中所做修改：

```bash
//切换到lab3_challenge2_semaphore
$ git checkout lab3_challenge2_semaphore

//继承lab3_3以及之前的答案
$ git merge lab3_3_rrsched -m "continue to work on lab3_challenge1"
```

**注意：不同于基础实验，挑战实验的基础代码具有更大的不完整性，可能无法直接通过构造过程。**
同样，不同于基础实验，我们在代码中也并未专门地哪些地方的代码需要填写，哪些地方的代码无须填写。这样，我们留给读者更大的“想象空间”。

- 本实验的具体要求为：通过修改PKE内核和系统调用，为用户程序提供信号量功能。
- 注意：最终测试程序可能和给出的用户程序不同，但都只涉及信号量的相关操作。

<a name="lab3_challenge2_guide"></a>

####  实验指导

* 你对内核代码的修改可能包含以下内容：
  * 添加系统调用，使得用户对信号量的操作可以在内核态处理
  * 在内核中实现信号量的分配、释放和PV操作，当P操作处于等待状态时能够触发进程调度

**注意：完成实验内容后，请读者另外编写应用，对自己的实现进行检测。**

**另外，后续的基础实验代码并不依赖挑战实验，所以读者可自行决定是否将自己的工作提交到本地代码仓库中（当然，提交到本地仓库是个好习惯，至少能保存自己的“作品”）。**