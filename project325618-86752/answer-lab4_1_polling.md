# Lab4-1：轮询

### 设计目的

本实验设计通过使用蓝牙驱动小车移动来帮助学生理解轮询是如何实现的。该实验在PYNQ-Z2开发板上运行，设计有以下几个功能：

- 学生在完成该实验的过程中，能够体会到MMIO、轮询的具体含义。
- 该设计扩展PKE到实际的应用场景中，能够支持更多样的应用程序。

### 设计思路

首先在设备树中查找设备uart，然后将UART、GPIO(小车)在内存的映射地址在内核页表中映射以方便在内核态、用户态对设备进行使用。本设计将设备与文件系统隔离开，采用系统调用的方式来进行设备管理，通过在内核态设计不同的系统调用来方便用户态程序进行设备的使用。

### 具体实现

#### 支持pke在开发板上开发

需要设置pmp。在kernel/machine/minit.c文件中，添加 setup_pmp(); ，修改m_start()函数

* setup_pmp函数：
  
  通过设置pmp，将所有内存的权限开放给M态和S态，以方便内核态和用户态的程序正常运行。
  
  ```cpp
  void setup_pmp(void) {
    // Set up a PMP to permit access to all of memory.
    // Ignore the illegal-instruction trap if PMPs aren't supported.
    uintptr_t pmpc = PMP_NAPOT | PMP_R | PMP_W | PMP_X;
    asm volatile(
        "la t0, 1f\n\t"
        "csrrw t0, mtvec, t0\n\t"
        "csrw pmpaddr0, %1\n\t"
        "csrw pmpcfg0, %0\n\t"
        ".align 2\n\t"
        "1: csrw mtvec, t0"
        :
        : "r"(pmpc), "r"(-1UL)
        : "t0");
  }
  ```
  
* 修改m_start函数：
  
  在添加调用setup_pmp函数，即可对PL端设置PMP机制。
  
  ```cpp
  157 setup_pmp();
  ```

#### 对PKE内核的修改

对PKE的修改也是围绕上述设计思路：

1. 为MMIO映射的内存在内核态页表中添加映射。
2. 添加对应设备的系统调用。

* 修改 kern_vm_init 函数：
  
  比较简单，直接把MMIO那一段内存在内核页表中进行映射即可。

  ```
  kern_vm_map(t_page_dir, (uint64)0x60000000, (uint64)0x60000000, (uint64)0x60020000 - (uint64)0x60000000,
           prot_to_type(PROT_READ | PROT_WRITE, 0));
  ```
  
* 添加系统调用 **sys_user_uart_putchar** ：
  
  由于lab4_1采用轮询的方式，因此只需要根据UART的使用手册，读取UART的状态，直到可用时，向指定的地址写要输出的字符ch。
  
  ```c++
  //add uart putchar getchar syscall
  //
  // implement the SYS_user_uart_putchar syscall
  //
  void sys_user_uart_putchar(uint8 ch) {
      volatile uint32 *status = (void*)(uintptr_t)0x60000008;
      volatile uint32 *tx = (void*)(uintptr_t)0x60000004;
      while (*status & 0x00000008);
      *tx = ch;
  }
  ```
  
* 添加系统调用sys_user_uart_getchar：
  
  由于lab4_1采用轮询的方式，因此只需要根据UART的使用手册，读取UART的状态，直到UART中有数据时时，读取指定的地址。
  
  ```c++
  ssize_t sys_user_uart_getchar() {
      
      
      volatile uint32 *rx = (void*)(uintptr_t)0x60000000;
      volatile uint32 *status = (void*)(uintptr_t)0x60000008;
      while (!(*status & 0x00000001));
      int32 ch = *rx;
      return ch;
  }
  ```
  
* 添加系统调用sys_user_gpio_reg_write：
  
  通过该系统调用来控制小车的移动，直接写入对应的地址即可。
  
  ```c++
  //car control
  ssize_t sys_user_gpio_reg_write(uint8 val) {
      volatile uint32_t *control_reg = (void*)(uintptr_t)0x60001004;
      volatile uint32_t *data_reg = (void*)(uintptr_t)0x60001000;
      *data_reg = (uint32_t)val;
      return 1;
  }
  ```

#### 对用户程序的修改

用户程序只有一个进程，是个死循环，不停的从uart中读取数据，根据数据控制小车的前进后退左转右转。当读取到指定的字符'q'时，程序正常退出。

```c++
/*
 * Below is the given application for lab4_1.
 * The goal of this app is to control the car via Bluetooth. 
 */

#include "user_lib.h"
#include "util/types.h"

int main(void) {
  printu("please input the instruction through bluetooth!\n");
  while(1)
  {
    char temp = (char)uartgetchar();
    uartputchar(temp);
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
  exit(0);
  return 0;
}
```
