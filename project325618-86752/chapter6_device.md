# 第六章．实验4：设备管理


### 目录  
- [6.1 实验4的基础知识](#fundamental)  
  - [6.1.1  内存映射I/O(MMIO)](#subsec_MMIO) 
  - [6.1.2  轮询I/O控制方式](#subsec_polling)
  - [6.1.3  中断驱动I/O控制方式](#subsec_plic)  
- [6.2 lab4_1 POLLING](#polling) 
  - [给定应用](#lab4_1_app)
  - [实验内容](#lab4_1_content)
  - [实验指导](#lab4_1_guide)
- [6.3 lab4_2 ](#) 
  - [给定应用](#lab4_2_app)
  - [实验内容](#lab4_2_content)
  - [实验指导](#lab4_2_guide)
- [6.4 lab4_3](#) 
  - [给定应用](#lab4_3_app)
  - [实验内容](#lab4_3_content)

<a name="fundamental"></a>

## 6.1 实验4的基础知识

完成前面所有实验后，PKE内核的整体功能已经得到完善。在实验四的设备实验中，我们将结合fpga-pynq板，在rocket chip上增加uart模块和蓝牙模块，并搭载PKE内核，实现蓝牙通信控制智能小车，设计设备管理的相关实验。

<a name="subsec_MMIO"></a>

### 6.1.1 内存映射I/O(MMIO)

内存映射(Memory-Mapping I/O)是一种用于设备驱动程序和设备通信的方式，它区别于基于I/O端口控制的Port I/O方式。RICSV指令系统的CPU通常只实现一个物理地址空间，这种情况下，外设I/O端口的物理地址就被映射到CPU中单一的物理地址空间，成为内存的一部分，CPU可以像访问一个内存单元那样访问外设I/O端口，而不需要设立专门的外设I/O指令。

在MMIO中，内存和I/O设备共享同一个地址空间。MMIO是应用得最为广泛的一种IO方法，它使用相同的地址总线来处理内存和I/O设备，I/O设备的内存和寄存器被映射到与之相关联的地址。当CPU访问某个内存地址时，它可能是物理内存，也可以是某个I/O设备的内存。此时，用于访问内存的CPU指令就可以用来访问I/O设备。每个I/O设备监视CPU的地址总线，一旦CPU访问分配给它的地址，它就做出响应，将数据总线连接到需要访问的设备硬件寄存器。为了容纳I/O设备，CPU必须预留给I/O一个地址映射区域。

用户空间程序使用mmap系统调用将IO设备的物理内存地址映射到用户空间的虚拟内存地址上，一旦映射完成，用户空间的一段内存就与IO设备的内存关联起来，当用户访问用户空间的这段内存地址范围时，实际上会转化为对IO设备的访问。

<a name="subsec_polling"></a>

### 6.1.2  轮询I/O控制方式

在实验四中，我们设备管理的主要任务是控制设备与内存的数据传递，具体为从蓝牙设备读取到用户输入的指令字符（或传递数据给蓝牙在手机端进行打印），解析为小车前、后、左、右、停止等动作来传输数据给电机实现对小车的控制。在前两个实验中，我们分别需要对轮询控制方式和中断控制方式进行实现。

首先，程序直接控制方式（又称循环测试方式），每次从外部设备读取一个字的数据到存储器，对于读入的每个字，CPU需要对外设状态进行循环检查，直到确定该数据已经传入I/O数据寄存器中。在轮询的控制方式下，由于CPU的高速性和I/O设备的低速性，导致CPU浪费绝大多数时间处于等待I/O设备完成数据传输的循环测试中，会造成大量资源浪费。

轮询I/O控制方式流程如图：

<img src="pictures/fig6_1_polling.png" alt="fig6_1" style="zoom:100%;" />

<a name="subsec_plic"></a>

### 6.1.3  中断驱动I/O控制方式

在前一种轮询的控制方式中，由于没有采用中断机制，CPU需要不断测试I/O设备的状态，造成CPU资源的极大浪费。中断驱动的方式是，允许I/O设备主动打断CPU的运行并请求相应的服务，请求I/O的进程首先会进入阻塞状态，PLIC将字符读取操作转化为s态中断进行处理，向进程传递读取的数据后，唤醒进程继续运行。

采用中断驱动的控制方式，在I/O操作过程中，CPU可以执行其他的进程，CPU与设备之间达到了部分并行的工作状态，从而提升了资源利用率。

中断驱动I/O方式流程如图：

<img src="pictures/fig6_2_plic.png" alt="fig6_2" style="zoom:100%;" />

### 6.1.4 设备树

设备树（Device Tree）是描述计算机的特定硬件设备信息的数据结构，以便于操作系统的内核可以管理和使用这些硬件，包括CPU或CPU，内存，总线和其他一些外设。

硬件的相应信息都会写在`.dts`为后缀的文件中，`dtc`是编译`dts`的工具，`dtb(Device Tree Blob)`，`dts`经过`dtc`编译之后会得到`dtb`文件，`dtb`通过`Bootloader`引导程序加载到内核。所以`Bootloader`需要支持设备树才行；Kernel也需要加入设备树的支持。

<img src="pictures/fig6_3.png" alt="fig6_3" style="zoom:130%;" />

在rocketchip中，设备即通过设备树的方式提供给pke使用。

<a name="polling"></a>

## 6.2 lab4_1 POLLING

<a name="lab4_1_app"></a>

#### **给定应用**
- user/app_poll.c

```
1	/*
2	 * Below is the given application for lab4_1.
3	 * The goal of this app is to control the car via Bluetooth. 
4	 */
5	
6	#include "user_lib.h"
7	#include "util/types.h"
8	
9	int main(void) {
10	  printu("please input the instruction through bluetooth!\n");
11	  while(1)
12	  {
13	    char temp = (char)uartgetchar();
14	    uartputchar(temp);
15	    switch (temp)
16	    {
17	      case '1' : gpio_reg_write(0x2e); break; //前进
18	      case '2' : gpio_reg_write(0xd1); break; //后退
19	      case '3' : gpio_reg_write(0x63); break; //左转
20	      case '4' : gpio_reg_write(0x9c); break; //右转
21	      case 'q' : exit(0);              break;
22	      default : gpio_reg_write(0x00); break;  //停止
23	    }
24	  }
25	  exit(0);
26	  return 0;
27	}
```

应用通过轮询的方式从蓝牙端获取指令，实现对小车的控制功能。

- 切换到lab4_1，继承lab3_3及之前实验所做的修改，并make后的直接运行结果：

```
//切换到lab4_1
$ git checkout lab4_1_poll

//继承lab3_3以及之前的答案
$ git merge lab3_3_rrsched -m "continue to work on lab4_1"

//重新构造
$ make clean; make

//运行构造结果
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_poll
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
please input the instruction through bluetooth!
You need to implement the uart_getchar function in lab4_1 here!

System is shutting down with exit code -1.

```

从结果上来看，蓝牙端端口获取用户输入指令的uartgetchar系统调用未完善，所以无法进行控制小车的后续操作。按照提示，我们需要实现蓝牙uart端口的获取和打印字符系统调用，以及传送驱动数据给小车电机的系统调用，实现对小车的控制。

<a name="lab4_1_content"></a>

#### **实验内容**

如输出提示所表示的那样，需要找到并完成对uartgetchar，uartputchar，gpio_reg_write的调用，并获得以下预期结果：

```
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_poll
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Ticks 0
please input the instruction through bluetooth!
Ticks 1
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

<a name="lab4_1_guide"></a>

#### **实验指导**

基于实验lab1_1，你已经了解和掌握操作系统中系统调用机制的实现原理。对于本实验的应用，我们发现user/app_poll.c文件中有三个函数调用：uartgetchar，uartputchar和gpio_reg_write。对代码进行跟踪，我们发现这三个函数都在user/user_lib.c中进行了实现，对应于lab1_1的流程，我们可以在kernel/syscall.h中查看新增的系统调用以及编号：

```
17	#define SYS_user_uart_putchar (SYS_user_base + 6)
18	#define SYS_user_uart_getchar (SYS_user_base + 7)
19	#define SYS_user_uart16550_putchar (SYS_user_base + 8)
20	#define SYS_user_uart16550_getchar (SYS_user_base + 9)
21	#define SYS_user_gpio_reg_write (SYS_user_base + 10)
```

继续追踪，我们发现在kernel/syscall.c的do_syscall函数中新增了对应系统调用编号的实现函数，对于新增系统调用，分别有如下函数进行处理：

```
136	    case SYS_user_uart_putchar:
137	      sys_user_uart_putchar(a1);
138	      return 1;
139	    case SYS_user_uart_getchar:
140	      return sys_user_uart_getchar();
141	    case SYS_user_gpio_reg_write:
142	      return sys_user_gpio_reg_write(a1);
```

读者的任务即为在kernel/syscall.c中追踪并完善对应的函数。对于uart的函数，我们给出uart端口的地址映射如图：

<img src="pictures/fig6_3_address.png" alt="fig6_3" style="zoom:80%;" />

我们可以看到配置uart端口的偏移地址为0x60000000，对应写地址为0x60000000，读地址为0x60000004，同时对0x60000008的状态位进行轮询，检测到信号时进行读写操作。

在kernel/syscall.c中找到函数实现空缺，并根据注释完成uart系统调用：

```
88	//
89	// implement the uart syscall
90	//
91	void sys_user_uart_putchar(uint8 ch) {
92	    panic( "You need to implement the uart_putchar function in lab4_1 here.\n" );
93	    //Getting status information from 0x60000008
94	    
95	    //Getting the character from 0x60000004
96	    
97	    //Gudge status
98	    
99	    //Transfer and display the character ch
100	    
101	}
102	
103	ssize_t sys_user_uart_getchar() {
104	    panic( "You need to implement the uart_getchar function in lab4_1 here.\n" );
105	    //Getting the character from 0x60000000
106	    
107	    //Getting status information from 0x60000008
108	    
109	    //Gudge status
110	    
111	    //Return the result character
112	    
113	    
114	}
```

和uart端口读写过程类似，其中电机连接端口gpio数据地址为0x60001000，根据用户程序app_poll中流程，我们需要将uart端口读到的驱动数据传递给电机。

在kernel/syscall.c中完善函数：

```
119	ssize_t sys_user_gpio_reg_write(uint8 val) {
120	    panic( "You need to implement the gpio_reg_write function in lab4_1 here.\n" );
121	    volatile uint32_t *control_reg = (void*)(uintptr_t)0x60001004;
122	    //Transfer the data to 0x60001000
123	    
124	    
125	    return 1;
126	}
```

安卓手机端验证：首先将HC-05蓝牙模块接入pynq板，接口对应关系为：

| pynq接口 | HC-05接口 |
| -------- | --------- |
| VCC      | VCC       |
| GND      | GND       |
| JA4      | RXD       |
| JA3      | TXD       |

接入时注意对应接口错位正确插入，然后在手机端下载BluetoothSerial，连接hc-05蓝牙模块，使用网线连接pynq板和电脑，打开开发板电源。

成功连接蓝牙模块后，启动连接：

```
$ ssh xilinx@192.168.2.99
```

随后使用scp指令将编译后的pke内核和用户app文件导入：

```
$ scp 文件名 xilinx@192.168.2.99:~
```

此时便成功进入pynq板环境，可对结果进行验证。

**实验完毕后，记得提交修改（命令行中-m后的字符串可自行确定），以便在后续实验中继承lab4_1中所做的工作**：

```
$ git commit -a -m "my work on lab4_1 is done."
```



<a name="PLIC"></a>

## 6.3 lab4_2 

<a name="lab4_2_app"></a>

#### **给定应用**

- user/app_PLIC.c

```
 /*
 * Below is the given application for lab4_2.
 * The goal of this app is to control the car via Bluetooth. 
 */

#include "user_lib.h"
#include "util/types.h"
void delay(unsigned int time){
  unsigned int a = 0xfffff ,b = time;
  volatile unsigned int i,j;
  for(i = 0; i < a; ++i){
    for(j = 0; j < b; ++j){
      ;
    }
  }
}
int main(void) {
  printu("Hello world!\n");
  int i;
  int pid = fork();
  if(pid == 0)
  {
    while (1)
    {
      delay(3);
      printu("waiting for you!\n");
    }
    
  }
  else
  {
    for (;;) {
      char temp = (char)uartgetchar();
      printu("%c\n", temp);
      switch (temp)
      {
        case '1' : gpio_reg_write(0x2e); break; //前进
        case '2' : gpio_reg_write(0xd1); break; //后退
        case '3' : gpio_reg_write(0x63); break; //左转
        case '4' : gpio_reg_write(0x9c); break; //右转
        case 'q' : exit(0);              break;
        default : gpio_reg_write(0x00); break;  //停止
      }
    }
  }
  

  exit(0);

  return 0;
}
```

应用通过中断的方式从蓝牙端获取指令，实现对小车的控制功能。

- 切换到lab4_2，继承lab3_3及之前实验所做的修改，并make后的直接运行结果：

```
//切换到lab4_2
$ git checkout lab4_2_PLIC

//继承lab3_3以及之前的答案
$ git merge lab3_3_rrsched -m "continue to work on lab4_2"

//重新构造
$ make clean; make

//运行构造结果
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_poll
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
please input the instruction through bluetooth!
You need to implement the uart_getchar function in lab4_2 here!

System is shutting down with exit code -1.

```



<a name="lab4_2_content"></a>

#### **实验内容**

如输出提示所表示的那样，需要找到并完成对uartgetchar，do_sleep，getuartvalue的调用，并获得以下预期结果：

```
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_polling
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Ticks 0
please input the instruction through bluetooth!
Ticks 1
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

<a name="lab4_2_guide"></a>

#### **实验指导**

对于本实验的应用，我们需要在lab4_1基础上实现基于中断的uartgetchar。对代码进行跟踪，我们可以在kernel/syscall.h中查看新增的系统调用以及编号：

```
17	#define SYS_user_uart_putchar (SYS_user_base + 6)
18	#define SYS_user_uart_getchar (SYS_user_base + 7)
21	#define SYS_user_gpio_reg_write (SYS_user_base + 10)
```

继续追踪，我们发现在kernel/syscall.c的do_syscall函数中新增了对应系统调用编号的实现函数，对于新增系统调用，分别有如下函数进行处理：

```
139	    case SYS_user_uart_getchar:
140	      return sys_user_uart_getchar();
```

你的任务即为在kernel/syscall.c中追踪并完善对应的函数。

在kernel/syscall.c中找到函数实现空缺，并根据注释完成uart系统调用：

```
88	//
89	// implement the uart syscall
90	//
103	ssize_t sys_user_uart_getchar() {
104	    panic( "You need to implement the uart_getchar function in lab4_2 here.\n" );
105	    //sleep
106	    
107	    //Wait for wake
108	    
109	    //get the value
110	    
111	    //Return the result character
112	    
113	    
114	}
```

当蓝牙有数据发送时，pke会收到外部中断，你需要完成接收到外部中断后的处理。

在kernel/strap.c中找到函数空缺，并根据注释完成中断处理函数：

```
100     case CAUSE_MEXTERNEL_S_TRAP:
101       {
102         panic( "You need to complete case CAUSE_MEXTERNEL_S_TRAP function in lab4_2 here.\n"
103         int irq = *(uint32 *)0xc201004L;
104         *(uint32 *)0xc201004L = irq;
105         volatile int *ctrl_reg = (void *)(uintptr_t)0x6000000c;
106         *ctrl_reg = *ctrl_reg | (1 << 4);
107         
108         // get the data from MMIO.
109         // send it to the process.
110         // call function to awake process[0]
111         
112         
113         
114         break;
115       }
```



在kernel/process.c中找到函数实现空缺，并根据注释完成do_sleep函数：

```
221 void do_sleep(){
222   panic( "You need to implement do_sleep function in lab4_2 here.\n"
223   // set the process BLOCKED.
224 }
```

在kernel/process.c中找到函数实现空缺，并根据注释完成do_wake函数：

```
226 void do_wake(){
227   panic( "You need to implement do_sleep function in lab4_2 here.\n"
228   //set the process READY.
229   //insert_to_ready_queue
230   
231   //schedule
232 }
```



**实验完毕后，记得提交修改（命令行中-m后的字符串可自行确定），以便在后续实验中继承lab4_2中所做的工作**：

```
$ git commit -a -m "my work on lab4_2 is done."
```

<a name="hostdevice"></a>

## 6.4 lab4_3 

<a name="lab4_3_app"></a>

#### **给定应用**
- user/app_host_device.c

```
 #pragma pack(4)
#define _SYS__TIMEVAL_H_
struct timeval {
    unsigned int tv_sec;
    unsigned int tv_usec;
};

#include "user_lib.h"
#include "videodev2.h"
#define DARK 64
#define RATIO 7 / 10

int main() {
    char *info = allocate_share_page();
    int pid = do_fork();
    if (pid == 0) {
        int f = do_open("/dev/video0", O_RDWR), r;

        struct v4l2_format fmt;
        fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV;
        fmt.fmt.pix.width = 320;
        fmt.fmt.pix.height = 180;
        fmt.fmt.pix.field = V4L2_FIELD_NONE;
        r = do_ioctl(f, VIDIOC_S_FMT, &fmt);
        printu("Pass format: %d\n", r);

        struct v4l2_requestbuffers req;
        req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        req.count = 1; req.memory = V4L2_MEMORY_MMAP;
        r = do_ioctl(f, VIDIOC_REQBUFS, &req);
        printu("Pass request: %d\n", r);

        struct v4l2_buffer buf;
        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        buf.memory = V4L2_MEMORY_MMAP; buf.index = 0;
        r = do_ioctl(f, VIDIOC_QUERYBUF, &buf);
        printu("Pass buffer: %d\n", r);

        int length = buf.length;
        char *img = do_mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_SHARED, f, buf.m.offset);
        unsigned int type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        r = do_ioctl(f, VIDIOC_STREAMON, &type);
        printu("Open stream: %d\n", r);

        char *img_data = allocate_page();
        for (int i = 0; i < (length + 4095) / 4096 - 1; i++)
            allocate_page();
        yield();

        for (;;) {
            if (*info == '1') {
                r = do_ioctl(f, VIDIOC_QBUF, &buf);
                printu("Buffer enqueue: %d\n", r);
                r = do_ioctl(f, VIDIOC_DQBUF, &buf);
                printu("Buffer dequeue: %d\n", r);
                r = read_mmap(img_data, img, length);
                int num = 0;
                for (int i = 0; i < length; i += 2)
                    if (img_data[i] < DARK) num++;
                printu("Dark num: %d > %d\n", num, length / 2 * RATIO);
                if (num > length / 2 * RATIO) {
                    *info = '0'; gpio_reg_write(0x00);
                }
            } else if (*info == 'q') break;
        }

        for (char *i = img_data; i - img_data < length; i += 4096)
            free_page(i);
        r = do_ioctl(f, VIDIOC_STREAMOFF, &type);
        printu("Close stream: %d\n", r);
        do_munmap(img, length); do_close(f); exit(0);
    } else {
        yield();
        for (;;) {
            char temp = (char)uartgetchar();
            printu("From bluetooth: %c\n", temp);
            *info = temp;
            switch (temp) {
                case '1': gpio_reg_write(0x2e); break; //前进
                case '2': gpio_reg_write(0xd1); break; //后退
                case '3': gpio_reg_write(0x63); break; //左转
                case '4': gpio_reg_write(0x9c); break; //右转
                case 'q': exit(0); break;
                default: gpio_reg_write(0x00); break;  //停止
            }
        }
    }
    return 0;
}
```



- 切换到lab4_3、继承lab4_2中所做修改，并make后的直接运行结果：

```
//切换到lab4_2
$ git checkout lab4_2_PLIC

//继承lab3_3以及之前的答案
$ git merge lab4_2_PLIC -m "continue to work on lab4_2"

//重新构造
$ make clean; make

//运行构造结果
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_PLIC
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
please input the instruction through bluetooth!
You need to implement the uart_getchar function in lab4_3 here!

System is shutting down with exit code -1.

```



<a name="lab4_3_content"></a>

####  实验内容

如输出提示所表示的那样，需要找到并完成对ioctl的调用。

预期结果：小车在前进过程中能够正常识别障碍物后停车。

**实验完毕后，记得提交修改（命令行中-m后的字符串可自行确定），以便在后续实验中继承lab4_3中所做的工作**：

```
$ git commit -a -m "my work on lab4_3 is done."
```
