# Lab4-3：使用宿主机设备

### 设计目的

本实验在Lab4-2的基础上，添加了对USB摄像头的支持，使得PL端上运行的PKE及Riscv用户程序能够访问并接收摄像头拍摄的图片数据，从而可以使智能小车实现避障等更加丰富多彩的功能。该实验设计有以下几个功能：

* 学生在完成该实验的过程中，能够切实体会到操作系统是怎么和用户程序协作的。
* 学生在编写用户程序的过程中，能够了解如何通过最基础的函数和系统调用操作设备。PKE上的用户程序稍加改动也可以在Linux操作系统中运行，使得学生学到的知识具有普适性。
* 该设计也提高了PKE系统的实用性，能够支持更复杂的用户程序。

### 设计思路

USB摄像头最基础的控制方法是使用读写设备文件的方式。拍摄一张照片包含以下过程：

* 打开设备文件，使用open函数
* 设置设备参数，使用ioctl函数
* 映射内存，由于USB摄像头对应的设备文件不支持直接用read函数进行读写，所以需要用mmap函数将文件映射到一段虚拟地址，通过虚拟地址进行读写
* 拍摄，使用ioctl函数控制
* 结束和清理，包含使用ioctl函数关闭设备，使用munmap函数解映射，使用close函数关闭设备文件

其中ioctl、mmap、munmap三个函数是PKE和riscv-fesvr不支持的，需要在本设计中添加，因此本设计的重点包含以下三个内容：

* 对riscv-fesvr的修改：riscv-fesvr是PS端（Arm）的一个程序，用于控制PL端（Riscv）程序的启动以及和通信。通过riscv-fesvr，PL端上的程序也可以访问PS端的文件，调用一些PS端系统的函数。原版的riscv-fesvr不支持ioctl和mmap等函数，而操控USB摄像头的用户程序必须使用这些函数，所以需要对riscv-fesvr进行修改，使得PL端上运行的PKE和用户程序能够通过riscv-fesvr这个中间层调用宿主机的系统函数从而控制摄像头；
* 对PKE内核代码的修改：需要为riscv-fesvr新增的函数调用提供用户层接口。
* 对用户代码的修改：有了对fesvr和内核的修改，用户程序就可以调用各类系统调用函数操控摄像机了。为了实现避障的功能，程序还需要对获得的图片信息进行解码和分析，根据前方是否为障碍物选择是否刹车。

### 具体实现

#### 对riscv-fesvr的修改

需要实现ioctl、mmap、unmap和readmmap四个函数。

* ioctl函数：

  ioctl函数相对较简单，只需要对PL端传给自己的各个参数调用PS端的ioctl函数就行了。但是ioctl的第三个参数是一个指针，从PL端传过来的时候，这个指针指向的是PL端的数据，因此需要现将这段数据从PL端读出，放在PS端的内存中，再将这段内存的指针传入PS端的ioctl函数；函数返回后，PS端的这段内存被修改了，所以需要将这段数据再写回PL端。所以需要根据ioctl的第二个参数决定这段数据的大小：

  ```cpp
  reg_t syscall_t::sys_ioctl(reg_t fd, reg_t request, reg_t data, reg_t a4, reg_t a5, reg_t a6, reg_t a7) {
    size_t size = 0;
    switch (request) { // 确定第三个参数指向的数据大小
        case VIDIOC_S_FMT:
            size = sizeof(v4l2_format);
            break;
        case VIDIOC_REQBUFS:
            size = sizeof(v4l2_requestbuffers);
            break;
        case VIDIOC_QUERYBUF:
        case VIDIOC_QBUF:
        case VIDIOC_DQBUF:
            size = sizeof(v4l2_buffer);
            break;
        case VIDIOC_STREAMON:
        case VIDIOC_STREAMOFF:
            size = sizeof(unsigned int);
            break;
        default:
            return -1;
    }
    std::vector<char> buf(size); // PS端数据
    memif->read(data, size, &buf[0]); // 将PL端数据写到PS端的缓冲区
    int r = ioctl(fds.lookup(fd), request, &buf[0]);
    if (r > -1) memif->write(data, size, &buf[0]); // 写回
    return r;
  }
  ```

* mmap函数：

  因为PS端上调用mmap函数实际上是将设备文件映射到一个PS端的虚拟地址，所以如果想在PL端的用户程序调用函数，则要将设备文件映射到PL端的虚拟地址，这需要PS端的虚拟地址作为中间者。fesvr中维护了一个数组，就用来存储这些地址。每当PL端的用户程序调用mmap时，fesvr会用PL端传来的mmap配置参数调用PS端的mmap参数获得一个地址，将这个地址存起来，然后将这个地址的数组索引返回给PL端。这样当PL端试图读取mmap映射的数据时，传给fesvr数组索引，fesvr就可以通过索引找到PS端的映射过的地址，把数据返回给PL端。

  所以，fesvr里mmap函数需要调用PS端的mmap函数，然后将返回的指针添加到数组中，最后返回索引：
  
  ```cpp
  reg_t syscall_t::sys_mmap(reg_t addr, reg_t length, reg_t prot, reg_t flags, reg_t fd, reg_t offset, reg_t a7) {
      char *t = (char *)mmap(NULL, length, prot, flags, fds.lookup(fd), offset);
      if (t == (char *)-1) return -1; else {
          mmap_mem.push_back(t); return mmap_mem.size() - 1;
      }
  }
  ```
  
* munmap函数：

  为了不影响其他的数组索引，函数不会删除数组元素，只需要调用PS端的munmap函数将第一个参数（数组索引）对应的指针解除映射即可：

  ```cpp
  reg_t syscall_t::sys_munmap(reg_t num, reg_t length, reg_t a3, reg_t a4, reg_t a5, reg_t a6, reg_t a7) {
      int r = munmap(mmap_mem[num], length);
      if (r >= 0) mmap_mem[num] = NULL;
      return r;
  }
  ```

* readmmap函数：

  readmmap函数不是PS端的系统函数，但为了能够从PL端读取PS端通过mmap映射的数据，所以仍需定义这个函数，做法就是根据第一个参数（数组索引）找到对应的指针，根据第二个参数（偏移量）和第三个参数（长度）将对应的数据写回PL端即可：

  ```cpp
  reg_t syscall_t::sys_readmmap(reg_t num, reg_t offset, reg_t length, reg_t addr, reg_t a5, reg_t a6, reg_t a7) {
      memif->write(addr, length, mmap_mem[num] + offset);
      return 0;
  }
  ```

#### 对PKE内核的修改

对PKE的修改也是围绕上面四个函数实现，这里只给出内核中实现具体功能的函数，不给出直接和用户程序交互的系统调用函数：

* ioctl函数：

  比较简单，直接把用户程序调用的参数通过frontend_syscall传给PS端的fesvr程序，这里不再赘述。

* mmap函数：

  mmap函数在PKE上的功能变成了把用户程序调用的配置参数通过frontend_syscall传给PS端的fesvr程序，获得PS端的数组索引保存起来，然后为用户程序申请虚拟地址。由于之后用户调用readmmap和munmap函数都是用这个虚拟地址作参数，所以需要保存数组索引和虚拟地址的对应关系。同时还要保存映射长度，用以在readmmap时判断访问的内存区域是否合法：

  ```c++
  char *do_mmap(char *addr, uint64 length, int prot, int flags, int fd, int64 offset) {
      for (int i = 0; i < MMAP_MEM_SIZE; i++) {
          if (mmap_mem[i].length == 0) { // 找到一个还没有用过的数组元素
              int64 r = frontend_syscall(HTIFSYS_mmap, (uint64)addr, length,
                      prot, flags, spike_file_get(fd)->kfd, offset, 0);
              if (r >= 0) {
                  mmap_mem[i].addr = g_ufree_page; // 申请虚拟地址并保存，PKE用的虚拟地址管理策略非常简单，即用一个g_ufree_page变量表示下一个空闲的虚拟页的地址
                  g_ufree_page += ROUNDUP(length, PGSIZE); // 更新g_ufree_page变量，这个变量需要按PGSIZE（4096）字节对齐，所以需要向上取整
                  mmap_mem[i].length = length; // 保存映射的内存长度
                  mmap_mem[i].num = r; // 保存返回的数组索引
                  return (char *)mmap_mem[i].addr; // 返回申请的虚拟地址
              } else return (char *)-1;
          }
      }
      return (char *)-1;
  }

* munmap函数：

  munmap函数需要对传入的地址参数在上面所述的数组进行查找，找到对应的索引，然后以索引作为参数调用frontend_syscall传给fesvr进行PS端的munmap，之后需要将对应的数组元素的length属性置为0，表示其“还没有用过”，下次可以重用：

  ```c++
  int do_munmap(char *addr, uint64 length) {
      for (int i = 0; i < MMAP_MEM_SIZE; i++) {
          if (mmap_mem[i].addr == (uint64)addr
                  && mmap_mem[i].length == length) { // 找到对应的数组元素
              int64 r = frontend_syscall(HTIFSYS_munmap, mmap_mem[i].num, length, 0, 0, 0, 0, 0);
              if (r >= 0) mmap_mem[i].length = 0; // 表示“还没有用过”
              return r;
          }
      }
      return -1;
  }
  
  ```

* readmmap函数：

  readmmap函数需要对传入的地址参数和长度进行查找，找到对应的索引，然后以索引和接受数据的缓冲区地址通过frontend_syscall传给fesvr：

  ```c++
  int read_mmap(char *addr, int length, char *buf) {
      for (int i = 0; i < MMAP_MEM_SIZE; i++) {
          if ((uint64)addr >= mmap_mem[i].addr
                  && length <= mmap_mem[i].length) { // 待获取的地址区间包含在映射的地址区间之中
              return frontend_syscall(HTIFSYS_readmmap, mmap_mem[i].num,
                      (uint64)addr - mmap_mem[i].addr, length,
                      (uint64)buf, 0, 0, 0);
          }
      }
      return -1;
  }
  ```

#### 对用户程序的修改

用户程序包含两个进程，主进程和实验4_2类似，负责接收蓝牙发送过来的数据，根据数据控制小车行动，子进程负责拍摄和分析，首先初始化摄像头设备，然后是个死循环：如果当前小车处于前进状态，则拍摄，然后检查数据，如果判断前面有障碍物则控制车轮停转（刹车），否则如果主进程退出了，则自己进行释放文件、内存、关闭设备等操作，再退出。

从摄像头设备获取过来的数据是YUYV格式，属于yuv422，即用灰度、蓝色色度、红色色度三个属性表示颜色，每个像素点都有灰度属性，相邻两个像素点共享蓝色色度和红色色度。由于我们分析障碍物只需要灰度图，所以取每个像素点的灰度属性即可。因此对于获取过来的数据删去奇数索引的数据，就可以得到灰度图。对于障碍物的判断，我们使用了一个比较简单的算法：计算灰度小于64的像素点个数，如果个数大于像素点总数的7/10，即认为前方是障碍物。因为当摄像头靠近障碍物时，前方视野往往会变暗，因此灰度较低（较黑）的像素点会变多。

```c++
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
    char *info = allocate_share_page(); // 分配一个共享页，方便父子进程之间传递信息（小车的状态）
    int pid = do_fork();
    if (pid == 0) { // 子进程
        int f = do_open("/dev/video0", O_RDWR), r; // 打开设备文件

        struct v4l2_format fmt;
        fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV;
        fmt.fmt.pix.width = 320;
        fmt.fmt.pix.height = 180;
        fmt.fmt.pix.field = V4L2_FIELD_NONE;
        r = do_ioctl(f, VIDIOC_S_FMT, &fmt); // 设置摄像头：图片大小为320*180，格式为YUYV
        printu("Pass format: %d\n", r);

        struct v4l2_requestbuffers req;
        req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        req.count = 1; req.memory = V4L2_MEMORY_MMAP;
        r = do_ioctl(f, VIDIOC_REQBUFS, &req); // 设置摄像头：使用mmap方式读取数据，缓冲区数量为1
        printu("Pass request: %d\n", r);

        struct v4l2_buffer buf;
        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        buf.memory = V4L2_MEMORY_MMAP; buf.index = 0;
        r = do_ioctl(f, VIDIOC_QUERYBUF, &buf); // 设置buf结构体对应的缓冲区索引
        printu("Pass buffer: %d\n", r);

        int length = buf.length;
        char *img = do_mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_SHARED, f, buf.m.offset); // mmap映射内存
        unsigned int type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        r = do_ioctl(f, VIDIOC_STREAMON, &type); // 开启摄像头
        printu("Open stream: %d\n", r);

        char *img_data = allocate_page(); // 分配内存，用来保存图片
        for (int i = 0; i < (length + 4095) / 4096 - 1; i++)
            allocate_page(); // 图片较大，需分配更多的页
        yield(); // 初始化完设备了，让主进程开始监听蓝牙

        for (;;) {
            if (*info == '1') { // 小车处于前进状态
                r = do_ioctl(f, VIDIOC_QBUF, &buf); // 缓冲入队
                printu("Buffer enqueue: %d\n", r);
                r = do_ioctl(f, VIDIOC_DQBUF, &buf); // 缓冲出队，这时拍摄完一张照片
                printu("Buffer dequeue: %d\n", r);
                r = read_mmap(img_data, img, length); // 把照片从PS端的映射内存读出来
                int num = 0;
                for (int i = 0; i < length; i += 2)
                    if (img_data[i] < DARK) num++; // 统计灰度小于DARK的像素数量
                printu("Dark num: %d > %d\n", num, length / 2 * RATIO);
                if (num > length / 2 * RATIO) { // 如果灰度小于DARK的像素数量大于像素总数的RATIO比例（注意length是yuyv图片数据的总大小，像素总数是该大小的一半）
                    *info = '0'; gpio_reg_write(0x00); // 刹车
                }
            } else if (*info == 'q') break;
        }

        for (char *i = img_data; i - img_data < length; i += 4096)
            free_page(i); // 释放内存
        r = do_ioctl(f, VIDIOC_STREAMOFF, &type); // 关闭摄像头
        printu("Close stream: %d\n", r);
        do_munmap(img, length); do_close(f); exit(0); // 关闭文件和解映射
    } else { // 主进程
        yield(); // 先让子进程初始化完摄像头
        for (;;) {
            char temp = (char)uartgetchar(); // 接受蓝牙信号
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

值得注意的是程序的前六行，该用户程序使用riscv64的工具链编译，因此设备配置相关的类型全都是按64位来编译的，即long类型占8个字节和结构体中的8字节数据按4字节对齐。但是我们的用户程序声明的这些类型最后需要传给arm32位架构下的宿主机调用，而宿主机的库函数里默认这些类型都是按32位编译的，因此会发生混乱。

为了避免这种情况，需要用`#pragma pack(4)`宏强制在编译时让结构体按4字节对齐。同时在本程序中，long的差异主要体现在标准库里的timeval结构体，该结果体被v4l2_format引用。所以需要通过定义`#define _SYS__TIMEVAL_H_`宏覆盖标准库里的定义，然后自己定义一个timeval结构体，将里面的所有属性由原来的long改为int，就和宿主机一致了。