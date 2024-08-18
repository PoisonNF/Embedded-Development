# Crucis开发

## 检测没有问题的功能

### 接插电源板

深度计接口正常

### I.MX6ull板

### STM32板

AD4111正常

232输出正常

PWM输出正常

### 调试充电板

网口通信正常

jlink短距离烧录正常

## 需要修改的部分

1. 调试充电板少部分丝印，用起来不方便。232输出存在问题，考虑换回max3232

## 调试32遇到的无法4串口同时使用

现象为串口4接收并返回出现乱码，只能同时初始化两个串口

检查后发现启动文件中的堆空间上限为0x200也就是512字节，串口初始化默认使用200字节，其中包括发送和接收。当堆空间上限增加后问题解决。

```
Stack_Size      EQU     0x00000400

                AREA    STACK, NOINIT, READWRITE, ALIGN=3
Stack_Mem       SPACE   Stack_Size
__initial_sp
                                                  
; <h> Heap Configuration
;   <o>  Heap Size (in Bytes) <0x0-0xFFFFFFFF:8>
; </h>

Heap_Size       EQU     0x00000200		改到 0x1000

                AREA    HEAP, NOINIT, READWRITE, ALIGN=3
```



## I.MX6ull实际运用

I.MX6ull将采用本地的uboot、Linux zimage、Ubuntu-base

更新文件使用tftp的方式

1. 将需要下载的文件放入虚拟机Ubuntu的tftpboot文件夹中，该文件夹是tftp的共享文件夹

2. 板子输入命令 tftp 192.168.5.11
3. 输入get filename 获取文件
4. 输入q退出tftp模式
5. 可以使用ls -l查看文件时间，因为i.mx6ull的时间不一定是最新的时间，使用date命令查看