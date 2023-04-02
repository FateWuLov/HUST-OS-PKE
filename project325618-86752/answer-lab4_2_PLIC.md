# Lab4-2：中断

### 设计目的

本实验采用与Lab4-1轮询不同的设备处理方式——中断方式。

本实验设计通过使用蓝牙驱动小车移动来帮助学生理解中断是如何实现的。该实验设计有以上几个功能：

- 学生在完成该实验的过程中，能够体会到MMIO、中断的具体含义。
- 该设计扩展PKE到实际的应用场景中，能够支持更多样的应用程序。

### 设计思路

本实验与lab4_1不同之处在于添加了对外部中断的中断处理过程。而在pke中，我们设计将触发M态的外部中断转发到S态来进行处理，这样更方便使用内核态的代码来进行处理。而在对于设备的系统调用中，我们不再采用轮询的方式，当用户程序调用设备的系统调用，则将该进程进入阻塞状态，直到接收到来自设备的外部中断，则先将数据获取后将阻塞的进程唤醒，进而保证程序能够正常执行。

### 具体实现

#### 转发外部中断

修改kernel/machine/mtrap.c文件中的handle_mtrap()函数。

即通过向sip寄存器写入要进行处理的中断位。

```cpp
    case 0x800000000000000b: {
       write_csr(sip, SIE_SEIE);
     
      break;
```

#### 外部中断处理

由于M态将外部中断转发到了S态，所以我们需要在kernel/strap.c中修改smode_trap_handler()函数，以添加对来外部中断的中断处理。

我们首先从0xc201004L地址获取中断号，由于当前只有一个设备的进行外部中断，所以我们不需要多余的判断。接下来我们从UART中读取数据，并将其写入对应进程中存放，接下来重置UART的中断使能位以便于接收到下次的来自UART的外部中断。最后唤醒对应的进程。

```cpp
    case CAUSE_MEXTERNEL_S_TRAP:
      {
        int irq = *(uint32 *)0xc201004L;
        *(uint32 *)0xc201004L = irq;
        volatile uint32 *rx = (void*)(uintptr_t)0x60000000;
        int data = *rx;
        sprint("the rx is %c\n",(char)data);
        updateuartvalue(data);
        volatile int *ctrl_reg = (void *)(uintptr_t)0x6000000c;
        *ctrl_reg = *ctrl_reg | (1 << 4);
        // write_csr(sie, read_csr(sie) & ~(1L << 9));
        do_wake();
        break;
      }
```

#### 修改系统调用函数sys_user_uart_getchar：

当用户程序通过ecall调用该系统调用时，首先将进程设置为阻塞状态，等待外部中断来唤醒该进程，当被唤醒后，返回获取到的数据。

```cpp
ssize_t sys_user_uart_getchar() {
    
    sprint("in sleep!\n");
    do_sleep();
    sprint("we get what we want!\n");
    int32 ch = getuartvalue();
    return ch;
}
```









#### 对用户程序的修改

用户程序包含两个进程，主进程，负责接收蓝牙发送过来的数据，根据数据控制小车行动，子进程通过输出来提醒用户发送数据去控制小车。

```c++
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

值得注意的是程序的前六行，该用户程序使用riscv64的工具链编译，因此设备配置相关的类型全都是按64位来编译的，即long类型占8个字节和结构体中的8字节数据按4字节对齐。但是我们的用户程序声明的这些类型最后需要传给arm32位架构下的宿主机调用，而宿主机的库函数里默认这些类型都是按32位编译的，因此会发生混乱。

为了避免这种情况，需要用`#pragma pack(4)`宏强制在编译时让结构体按4字节对齐。同时在本程序中，long的差异主要体现在标准库里的timeval结构体，该结果体被v4l2_format引用。所以需要通过定义`#define _SYS__TIMEVAL_H_`宏覆盖标准库里的定义，然后自己定义一个timeval结构体，将里面的所有属性由原来的long改为int，就和宿主机一致了。