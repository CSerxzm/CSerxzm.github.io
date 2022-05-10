---
title: FPGA下载
comments: false
date: 2021-12-30 21:02:48
categories:
  - verilog
tags:
  - PL编程
  - FPGA
---

本文主要描述了如何在 Linux 系统启动以后，在线将比特流文件更新到 ZYNQ PL 的过程及方法。

ZYNQ 的开发板是由 PS 部分的双核 arm 和 PL 部分的可编程逻辑组成。可以通过 SDK 进行下载比特流进行编程，但是这样会导致 ZYNQ 重启，后面更新，支持热加载对 FPGA 部分进行编程。

<!-- more -->

其前提是 PS 部分已经运行 arm 版本的 unix 系统，让后进行下面的操作步骤：

1. Vivado 2018.2 生成.bit 比特流，进入到/runs/impl_1/ 查看是否生成.bit 文件。

2. 在/runs/impl_1/ 中新建 Full_Bitstream.bif ,并将在此文件下输入以下内容：

```shell
all:
{
      E:/ZYNQ/project_2/project_2.runs/impl_1/design_1_wrapper.bit
}
```

其中`E:/ZYNQ/project_2/project_2.runs/impl_1/design_1_wrapper.bit`为我的.bit 的文件路径，这里建议使用全路径。

3. 在 Vivado Tcl Shell 命令行中运行 Full_Bitstream.bif 生成.bit.bin 文件，运行命令如下所示：

```shell
bootgen -image E:/ZYNQ/project_2/project_2.runs/impl_1/Full_Bitstream.bif -arch zynq -process_bitstream bin
```

其中`E:/ZYNQ/project_2/project_2.runs/impl_1/Full_Bitstream.bif`为我的 Full_Bitstream.bif 文件所在位置，这里建议使用全路径。

4. 最后会在在/.runs/impl_1/文件夹中生成.bit.bin 文件，将此文件拷贝到 zynq 的文件系统，之后加载比特流到 FPGA 系统如下：

```shell
echo 0 > /sys/class/fpga_manager/fpga0/flags
mkdir -p /lib/firmware
cp /media/design_1_wrapper.bit.bin /lib/firmware/
echo design_1_wrapper.bit.bin > /sys/class/fpga_manager/fpga0/firmware
```

没有报错，则说明比特流加载成功。

## 其他

对于使用 ZYNQ 的时候，需要访问对应内存映射下的寄存器状态，使用下面的 devmem.c 编译成可执行文件。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <signal.h>
#include <fcntl.h>
#include <ctype.h>
#include <termios.h>
#include <sys/types.h>
#include <sys/mman.h>

#define FATAL do { fprintf(stderr, "Error at line %d, file %s (%d) [%s]\n", \
  __LINE__, __FILE__, errno, strerror(errno)); exit(1); } while(0)

#define MAP_SIZE 4096UL
#define MAP_MASK (MAP_SIZE - 1)

int main(int argc, char **argv) {
    int fd;
    void *map_base, *virt_addr;
    unsigned long read_result, writeval;
    off_t target;
    int access_type = 'w';

    if(argc < 2) {
        fprintf(stderr, "\nUsage:\t%s { address } [ type [ data ] ]\n"
            "\taddress : memory address to act upon\n"
            "\ttype    : access operation type : [b]yte, [h]alfword, [w]ord\n"
            "\tdata    : data to be written\n\n",
            argv[0]);
        exit(1);
    }
    target = strtoul(argv[1], 0, 0);

    if(argc > 2)
        access_type = tolower(argv[2][0]);


    if((fd = open("/dev/mem", O_RDWR | O_SYNC)) == -1) FATAL;
    printf("/dev/mem opened.\n");
    fflush(stdout);

    /* Map one page */ //将内核空间映射到用户空间
    map_base = mmap(0, MAP_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, target & ~MAP_MASK);
    if(map_base == (void *) -1) FATAL;
    printf("Memory mapped at address %p.\n", map_base);
    fflush(stdout);

    virt_addr = map_base + (target & MAP_MASK);
    //针对不一样的参数获取不一样类型内存数据
    switch(access_type) {
        case 'b':
            read_result = *((unsigned char *) virt_addr);
            break;
        case 'h':
            read_result = *((unsigned short *) virt_addr);
            break;
        case 'w':
            read_result = *((unsigned long *) virt_addr);
            break;
        default:
            fprintf(stderr, "Illegal data type '%c'.\n", access_type);
            exit(2);
    }
    printf("Value at address 0x%X (%p): 0x%X\n", target, virt_addr, read_result);
    fflush(stdout);
    //若参数大于3个，则说明为写入操做，针对不一样参数写入不一样类型的数据
    if(argc > 3) {
        writeval = strtoul(argv[3], 0, 0);
        switch(access_type) {
            case 'b':
                *((unsigned char *) virt_addr) = writeval;
                read_result = *((unsigned char *) virt_addr);
                break;
            case 'h':
                *((unsigned short *) virt_addr) = writeval;
                read_result = *((unsigned short *) virt_addr);
                break;
            case 'w':
                *((unsigned long *) virt_addr) = writeval;
                read_result = *((unsigned long *) virt_addr);
                break;
        }
        printf("Written 0x%X; readback 0x%X\n", writeval, read_result);
        fflush(stdout);
    }

    if(munmap(map_base, MAP_SIZE) == -1) FATAL;
    close(fd);
    return 0;
}
```

它会使用虚拟字符设备`/dev/mem`，将物理空间映射到用户空间上的虚拟地址，就可以使用为寄存器分配的逻辑地址访问寄存器的地址。
如下图我使用该程序读写挂载到`0x43c00000`的 AXI-Lite 的寄存器的值，可见读写正确，在实际的使用中，可将该寄存器中的值映射到 PL 部分的引脚，完成外部数据的获取或输入。

```shell
root@pynq:/home/xilinx# ./devmem 0x43c00000 w 0xfffffff0
/dev/mem opened.
Memory mapped at address 0xb6ff3000.
Value at address 0x43C00000 (0xb6ff3000): 0xFFFFFFF0
Written 0xFFFFFFF0; readback 0xFFFFFFF0
root@pynq:/home/xilinx# ./devmem 0x43c00000
/dev/mem opened.
Memory mapped at address 0xb6f35000.
Value at address 0x43C00000 (0xb6f35000): 0xFFFFFFF0
```
