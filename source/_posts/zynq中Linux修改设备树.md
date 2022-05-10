---
title: zynq中Linux修改设备树
comments: false
date: 2022-01-06 13:50:48
categories:
- verilog
tags:
- spi
- 设备树
---

​	在Zynq 7000系列中，对于驱动设备的修改，一般会使用xilinx官方的SDK Petalinux进行Linux的内核配置，然后进行重新打包成镜像，但是笔者在实验中使用的是Pynq（对Zynq的封装，集成了Python环境），所用的镜像是已有的镜像，在不想重新打包镜像文件的前提下，进行以下的实验操作。

​	`Open Firmware Device Tree`或者简单设备树，是一种描述硬件的一种数据结构和语言。特别的是，使用DTS描述操作系统只读的硬件信息，因此操作系统中不需要硬编码描述设备信息。DeviceTree发源于PowerPC架构，目的是为了为了消除代码中冗余的各种device注册代码而产生的，现在已经成为了linux的通用机制。设备树是一种用树状结构的方式来描述硬件信息，由Node(节点)、Property(属性)两种元素组成。

<!-- more -->

​	在Linux的默认的存在几乎所有的驱动设备的设备镜像，只是没有使能，处于`disable`的状态，在运行的Linux可以动态的修改，这就需要我们进行Overlay,以SPI为例，进行下面的操作，重写设备树。

* 准备文件`spi_overlay.dts`，后面将他转换成dtbo格式。

```properties
/* Device Tree Overlay for tl-emio-emac-demo test */

/dts-v1/;
/plugin/;

/ {

        fragment@0 {
                target = <&fpga_full>;
                overlay0: __overlay__ {
                        #address-cells = <1>;
                        #size-cells = <1>;
                        firmware-name = "design_1_wrapper.bit.bin";
                };

        };

        fragment@1 {
                target = <&spi0>;
                overlay1: __overlay__ {
                        status = "okay";
                        device@0 {
                                //compatible = "spidev";
                                compatible = "rohm,dh2228fv"; // Avoid warning info in dmesg
                                reg = <0>; //cs_0
                                spi-max-frequency = <2500000>;
                                #address-cells = <1>;
                                #size-cells = <1>;
                        }; 
                };
        };
};
```

* 转换成dtbo的文件格式使用的脚本,此后运行将会产生dtbo格式的文件。

```bash
#!/bin/sh
if [ $# -lt 1 ]; then
        echo "usage: $0 dts_source"
        exit
fi
filename=$1
name=${filename%.*}
DTC=dtc
#DTC=dtc
${DTC} -O dtb -o ${name}.dtbo -b 0 -@ ${filename}
```

> 如果没有dtc工具，可运行 apt-get install device-tree-compiler -y 进行安装后，再执行上面的脚本文件。

* 后面进行以下的指令执行。

```bash
root@pynq:/home/xilinx# mkdir /configfs
root@pynq:/home/xilinx# ls /configfs/
root@pynq:/home/xilinx# mount -t configfs configfs /configfs
root@pynq:/home/xilinx# ls /configfs/
device-tree
root@pynq:/home/xilinx# ls /configfs/device-tree/overlays/
root@pynq:/home/xilinx# mkdir /configfs/device-tree/overlays/spi
root@pynq:/home/xilinx# ls /configfs/device-tree/overlays/spi
dtbo  path  status
root@pynq:/home/xilinx# cp spi_overlay.dtbo /lib/firmware/
root@pynq:/home/xilinx# echo spi_overlay.dtbo > /configfs/device-tree/overlays/spi/path 
root@pynq:/home/xilinx# ls /dev/spidev*
/dev/spidev1.0
```

* 使用测试spi_test.c文件进行测试

```bash
#include <stdint.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <getopt.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/types.h>
#include <linux/spi/spidev.h>

#define ARRAY_SIZE(a) (sizeof(a) / sizeof((a)[0]))

static void pabort(const char *s)
{
    perror(s);
    abort();
}

static const char *device = "/dev/spidev1.0";
static uint32_t mode;
static uint8_t bits = 8;
static uint32_t speed = 2500000;
static uint16_t delay;
static int verbose;

uint8_t default_tx[] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
    0x40, 0x00, 0x00, 0x00, 0x00, 0x95,
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
    0xF0, 0x0D,
};

uint8_t default_rx[ARRAY_SIZE(default_tx)] = {0, };
char *input_tx;

static void hex_dump(const void *src, size_t length, size_t line_size, char *prefix)
{
    int i = 0;
    const unsigned char *address = src;
    const unsigned char *line = address;
    unsigned char c;

    printf("%s | ", prefix);
    while (length-- > 0) {
        printf("%02X ", *address++);
        if (!(++i % line_size) || (length == 0 && i % line_size)) {
            if (length == 0) {
                while (i++ % line_size)
                    printf("__ ");
            }
            printf(" | ");  /* right close */
            while (line < address) {
                c = *line++;
                printf("%c", (c < 33 || c == 255) ? 0x2E : c);
            }
            printf("\n");
            if (length > 0)
                printf("%s | ", prefix);
        }
    }
}

/*
 *  Unescape - process hexadecimal escape character
 *      converts shell input "\x23" -> 0x23
 */
static int unescape(char *_dst, char *_src, size_t len)
{
    int ret = 0;
    char *src = _src;
    char *dst = _dst;
    unsigned int ch;

    while (*src) {
        if (*src == '\\' && *(src+1) == 'x') {
            sscanf(src + 2, "%2x", &ch);
            src += 4;
            *dst++ = (unsigned char)ch;
        } else {
            *dst++ = *src++;
        }
        ret++;
    }
    return ret;
}

static void transfer(int fd, uint8_t const *tx, uint8_t const *rx, size_t len)
{
    int ret;

    struct spi_ioc_transfer tr = {
        .tx_buf = (unsigned long)tx,
        .rx_buf = (unsigned long)rx,
        .len = len,
        .delay_usecs = delay,
        .speed_hz = speed,
        .bits_per_word = bits,
    };

    if (mode & SPI_TX_QUAD)
        tr.tx_nbits = 4;
    else if (mode & SPI_TX_DUAL)
        tr.tx_nbits = 2;
    if (mode & SPI_RX_QUAD)
        tr.rx_nbits = 4;
    else if (mode & SPI_RX_DUAL)
        tr.rx_nbits = 2;
    if (!(mode & SPI_LOOP)) {
        if (mode & (SPI_TX_QUAD | SPI_TX_DUAL))
            tr.rx_buf = 0;
        else if (mode & (SPI_RX_QUAD | SPI_RX_DUAL))
            tr.tx_buf = 0;
    }

    ret = ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
    if (ret < 1)
        pabort("can't send spi message");

    if (verbose)
        hex_dump(tx, len, 32, "TX");
    hex_dump(rx, len, 32, "RX");
}

static void print_usage(const char *prog)
{
    printf("Usage: %s [-DsbdlHOLC3]\n", prog);
    puts("  -D --device   device to use (default /dev/spidev1.0)\n"
         "  -s --speed    max speed (Hz)\n"
         "  -d --delay    delay (usec)\n"
         "  -b --bpw      bits per word \n"
         "  -l --loop     loopback\n"
         "  -H --cpha     clock phase\n"
         "  -O --cpol     clock polarity\n"
         "  -L --lsb      least significant bit first\n"
         "  -C --cs-high  chip select active high\n"
         "  -3 --3wire    SI/SO signals shared\n"
         "  -v --verbose  Verbose (show tx buffer)\n"
         "  -p            Send data (e.g. \"1234\\xde\\xad\")\n"
         "  -N --no-cs    no chip select\n"
         "  -R --ready    slave pulls low to pause\n"
         "  -2 --dual     dual transfer\n"
         "  -4 --quad     quad transfer\n");
    exit(1);
}

static void parse_opts(int argc, char *argv[])
{
    while (1) {
        static const struct option lopts[] = {
            { "device",  1, 0, 'D' },
            { "speed",   1, 0, 's' },
            { "delay",   1, 0, 'd' },
            { "bpw",     1, 0, 'b' },
            { "loop",    0, 0, 'l' },
            { "cpha",    0, 0, 'H' },
            { "cpol",    0, 0, 'O' },
            { "lsb",     0, 0, 'L' },
            { "cs-high", 0, 0, 'C' },
            { "3wire",   0, 0, '3' },
            { "no-cs",   0, 0, 'N' },
            { "ready",   0, 0, 'R' },
            { "dual",    0, 0, '2' },
            { "verbose", 0, 0, 'v' },
            { "quad",    0, 0, '4' },
            { NULL, 0, 0, 0 },
        };
        int c;

        c = getopt_long(argc, argv, "D:s:d:b:lHOLC3NR24p:v", lopts, NULL);

        if (c == -1)
            break;

        switch (c) {
        case 'D':
            device = optarg;
            break;
        case 's':
            speed = atoi(optarg);
            break;
        case 'd':
            delay = atoi(optarg);
            break;
        case 'b':
            bits = atoi(optarg);
            break;
        case 'l':
            mode |= SPI_LOOP;
            break;
        case 'H':
            mode |= SPI_CPHA;
            break;
        case 'O':
            mode |= SPI_CPOL;
            break;
        case 'L':
            mode |= SPI_LSB_FIRST;
            break;
        case 'C':
            mode |= SPI_CS_HIGH;
            break;
        case '3':
            mode |= SPI_3WIRE;
            break;
        case 'N':
            mode |= SPI_NO_CS;
            break;
        case 'v':
            verbose = 1;
            break;
        case 'R':
            mode |= SPI_READY;
            break;
        case 'p':
            input_tx = optarg;
            break;
        case '2':
            mode |= SPI_TX_DUAL;
            break;
        case '4':
            mode |= SPI_TX_QUAD;
            break;
        default:
            print_usage(argv[0]);
            break;
        }
    }
    if (mode & SPI_LOOP) {
        if (mode & SPI_TX_DUAL)
            mode |= SPI_RX_DUAL;
        if (mode & SPI_TX_QUAD)
            mode |= SPI_RX_QUAD;
    }
}

int main(int argc, char *argv[])
{
    int ret = 0;
    int fd;
    uint8_t *tx;
    uint8_t *rx;
    int size;

    parse_opts(argc, argv);

    fd = open(device, O_RDWR);
    if (fd < 0)
        pabort("can't open device");

    /*
     * spi mode
     */
    ret = ioctl(fd, SPI_IOC_WR_MODE32, &mode);
    if (ret == -1)
        pabort("can't set spi mode");

    ret = ioctl(fd, SPI_IOC_RD_MODE32, &mode);
    if (ret == -1)
        pabort("can't get spi mode");

    /*
     * bits per word
     */
    ret = ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    if (ret == -1)
        pabort("can't set bits per word");

    ret = ioctl(fd, SPI_IOC_RD_BITS_PER_WORD, &bits);
    if (ret == -1)
        pabort("can't get bits per word");

    /*
     * max speed hz
     */
    ret = ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);
    if (ret == -1)
        pabort("can't set max speed hz");

    ret = ioctl(fd, SPI_IOC_RD_MAX_SPEED_HZ, &speed);
    if (ret == -1)
        pabort("can't get max speed hz");

    printf("spi mode: 0x%x\n", mode);
    printf("bits per word: %d\n", bits);
    printf("max speed: %d Hz (%d KHz)\n", speed, speed/1000);

    if (input_tx) {
        size = strlen(input_tx+1);
        tx = malloc(size);
        rx = malloc(size);
        size = unescape((char *)tx, input_tx, size);
        transfer(fd, tx, rx, size);
        free(rx);
        free(tx);
    } else {
        transfer(fd, default_tx, default_rx, sizeof(default_tx));
    }

    close(fd);

    return ret;
}
```

* 将SPI接口的MISO和MOSI进行短接，运行程序进行测试，可见程序测试成功。

```bash
root@pynq:/home/xilinx/code# gcc -o spi_test spi_test.c 
root@pynq:/home/xilinx/code# ./spi_test -D /dev/spidev1.0 -v
spi mode: 0x0
bits per word: 8
max speed: 2500000 Hz (2500 KHz)
TX | FF FF FF FF FF FF 40 00 00 00 00 95 FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF F0 0D  | ......@......................
RX | FF FF FF FF FF FF 40 00 00 00 00 95 FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF F0 0D  | ......@......................
```

