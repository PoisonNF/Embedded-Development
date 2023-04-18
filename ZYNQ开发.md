# 软件安装

[vivado2019.2安装+license添加教程_vivado license_原地打转的瑞哥的博客-CSDN博客](https://blog.csdn.net/weixin_47730622/article/details/125623165)

vivado许可证

链接: https://pan.baidu.com/s/1-2QL3MCYhUh-qANEuxVHIQ 提取码: vh3h 

# 一些配置

## 使用无线网卡

1. 插入无线网卡
2. 输入ifconfig -a 找到wlan0设备
3. 输入ifconfig wlan0 up 打开设备
4. 输入iwlist wlan0 scan 扫描附近的WiFi
5. 关闭PS网口 ifconfig eth0 down
6. 连接WiFi /opt/hardwareTest/usb_wifi_setup.sh WiFi名称 密码 wlan0
7. 测试 ping -I wlan0 -c 10 www.baidu.com

# 常见问题

- 启动模式选择

| BOOT_CFG |  1   |  2   |
| :------: | :--: | :--: |
|   JTAG   |  ON  |  ON  |
|   NAND   | OFF  |  ON  |
|   QSPI   |  ON  | OFF  |
| SD Card  | OFF  | OFF  |



# PL(FPGA)
## 开发流程

### 新建工程

![image-20230413180541788](./ZYNQ开发.assets/image-20230413180541788.png)

点击Create Project，然后点击Next。

输入工程名以及保存路径。默认勾选“Create project subdirectory”，该目录用于存放工程内的各种文件，方便管理。

![image-20230413181051157](./ZYNQ开发.assets/image-20230413181051157.png)

再点击Next，选择RTL Project，不勾选“Do not specify sources at this time”，将会出现添加源文件的界面。

接下来是添加约束文件，我们也是直接点击“Next”。

接下来选择开发板的芯片型号，直接在搜素框中输入完整的芯片型号，xc7z010clg400-1，一直点Next就行。

### 设计输入

在Source中点击'+'添加源文件，定好文件名，基本不用怎么设置，一路next。(使用verilog语言)

### RTL分析

点击RTL ANALYSIS的Open Elaborated Designed，生成原理图

### 管脚约束

在右上角选择I/O Planning，通过查看原理图进行约束。主要是I/O口和电平。

### 生成比特流

点击Generate bitstream

### 烧录到开发板

连接开发板，点击Open Target，选择Auto Connect

## AXI总线

![image-20230415233058699](./ZYNQ开发.assets/image-20230415233058699.png)

![image-20230415233123260](./ZYNQ开发.assets/image-20230415233123260.png)

## 硬件调试

ILA 和 VIO 

### 例化ILA核

调用IP核库，搜索ILA，选择ILA(IntergrateLogic Analyzer)

配置探针数量以及位宽，进行编译

在IP sources中找到例化模板（.veo结尾），即可复制使用

### 网表加入探针

必须先综合

右上角选择Debug，打开网表，在需要查看的信号名上右键 Mark Debug

输入信号后缀为IBUF 输出信号为OBUF

如果不让vivado进行优化，需要添加(\*mark_debug = "true"\*)

## Vivado Simulator仿真

TB文件结构

```verilog
`timescale 仿真单位/仿真精度

module test_bench();
//通常testbench没有输入和输出端口

信号或者变量定义声明

使用initial或者always语句产生激励波形

例化设计模块

endmodule
```

在simulation sources创建testbench文件

### 时序约束

在时序约束之前，先分配管脚，因为管脚位置会影响时序约束。

在IMPLEMENTATION下选择Edit timing constraints

## Modelsim安装

[Modelsim 安装步骤详解_兄弟抱一下~的博客-CSDN博客](https://blog.csdn.net/QWERTYzxw/article/details/115350715)

## Modelsim使用

\# 等待

$stop 系统任务暂停

$stop(n) n可取0、1、2

$finish 退出仿真任务

Initial 初始化语句



在开始仿真前，需要取消勾选Enable optimization，防止波形被优化

# PS(裸机)

## ZYNQ裸机开发流程

- 创建Vivado工程
- 使用IP Processor创建Processor System
- 生成顶层HDL
- 生成比特流导出到SDK（不使用FPGA就不用生成比特流）
- 在SDK中创建应用工程
- 板级验证

## ZYNQ最小系统

最小系统只需要ARM Cortex-A9加上DDR3和UART即可，不需要PL部分。

## 软件操作

在IP INTEGRATOR下中创建或者打开IP核，生成的文件后缀为.bd

### 配置外设

在DDR configuration中Memory Part选择需要更改。7010核心板选择MT41J128M16HA-125;7020核心板选择MT41J256M16RE-125

配置UART在peripheral I/O Pins中，根据板子的原理图选择对应的管脚，启明星是Bank14和15。

在PS-PL configuration中的General中配置串口的波特率

Bank500是bank0，Bank501是bank1，根据原理图选择IO电平

![image-20230413222737510](./ZYNQ开发.assets/image-20230413222737510.png)

### 去掉不需要的接口

在PS-PL configuration中的AXI Non Secure Enavlement中的AXI GP0 interface，取消选择。

在Clock Configuration中的PL Fabric Clocks中关闭FCLK_CLK0

在PS-PL configuration中的General中的Enable Clock Resets关闭

### Run block Automation

![image-20230414172535132](./ZYNQ开发.assets/image-20230414172535132.png)

按F6或者菜单栏上的检查按钮，检查设计是否有误。

### 生成IP核

在.bd文件上右键选择Generate Output Products

![image-20230414173113353](./ZYNQ开发.assets/image-20230414173113353.png)

### 生成顶层HDL包装

在.bd文件上右键选择Create HDL Wrapper

![image-20230414174019847](./ZYNQ开发.assets/image-20230414174019847.png)

### 导出硬件

在菜单栏File->Export Hardware，可以选择是否包含比特流

### 启动SDK

在菜单栏File->Launch SDK

### 在SDK中创建工程

在SDK菜单栏File->New->Application Project

### SDK烧录程序

编译之后，在工程目录下Binaries下的.elf文件右键，选择Run As->Launch on Hardware

# PS(Linux)



