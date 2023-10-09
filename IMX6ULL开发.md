# 环境搭建与开发板操作

## 三端配网

Ubuntu 192.168.5.11

![image-20230610211702839](./IMX6ULL开发.assets/image-20230610211702839.png)

Windows 192.168.5.10

![image-20230610211716810](./IMX6ULL开发.assets/image-20230610211716810.png)

开发板 192.168.5.9



使用命令`ifconfig eth0 192.168.5.9`设置开发板IP

可以通过修改interfaces文件固定IP

1. `vi /etc/network/interfaces`

2. 输入下面的内容

```shell
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet static
 address 192.168.5.9
 netmask 255.255.255.0
 gateway 192.168.5.1
```

3. 按下键盘 ESC，然后输入执行 :wq 保存并退出
4. 执行韦东山为`/etc/init.d/S40network restart `正点原子为`/etc/init.d/networking restart`重启网络服务。

## 下载BSP以及配置工具链

### 配置 repo

```shell
git config --global user.email "user@100ask.com"
git config --global user.name "100ask"

```

### 下载 BSP

```shell
git clone https://e.coding.net/codebug8/repo.git

mkdir -p 100ask_imx6ull-sdk && cd 100ask_imx6ull-sdk

../repo/repo init -u https://gitee.com/weidongshan/manifests.git -b linux-sdk -m imx6ull/100ask_imx6ull_linux4.9.88_release.xml --no-repo-verify

../repo/repo sync -j4

```

在~/100ask_imx6ull-sdk目录下执行`../repo/repo sync -c`更新代码。

### 配置交叉编译工具链

#### 永久生效（推荐）

```shell
vim ~/.bashrc 或者 gedit ~/.bashrc

export ARCH=arm
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
export PATH=$PATH:/home/book/100ask_imx6ull-sdk/ToolChain/arm-buildroot-linux-gnueabihf_sdk-buildroot/bin

source ~/.bashrc
arm-buildroot-linux-gnueabihf-gcc -v
```

#### 临时生效

只对当前终端有效，另开一个终端需要再次设置

```shell
export ARCH=arm
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
export PATH=$PATH:/home/book/100ask_imx6ull-sdk/ToolChain/arm-buildroot-linux-gnueabihf_sdk-buildroot/bin
```

### 压缩Linux源码传回Windows

使用命令对Linux源码进行压缩`tar cjf Linux-4.9.88.tar.bz2 Linux-4.9.88`

使用FileZilla传回Windows

### NFS挂载

在韦东山开发板中输入`mount -t nfs -o nolock,vers=3 192.168.5.11:/home/book/nfs_rootfs /mnt`

-t nfs 表示使用nfs服务 nolock,vers=3 不锁定，使用第三个版本 

将192.168.5.11:/home/book/nfs_rootfs 挂载到本地/mnt目录下

正点原子则为`mount -t nfs -o nolock,vers=3 192.168.5.11:/home/bcl/nfs /mnt`

## 第一个应用实验

使用例程01_hello，将文件拷贝到Ubuntu->nfs_rootfs目录下。

执行`arm-buildroot-linux-gnueabihf-gcc -o hello hello.c`即可编译代码。

在开发板上执行`./hello`，就可以看到实验效果。

在传统X86上使用的是gcc编译，在开发板ARM架构上需要使用交叉编译器，也就是开头这个arm-buildroot-linux-gnueabihf-gcc。

## 第一个驱动程序

### 配置编译：内核、设备树、驱动

#### 编译内核

```shell
cd /home/book/100ask_imx6ull-sdk/Linux-4.9.88	#移到内核源码下
make mrproper					#清除内核源代码目录下的所有已编译对象及配置文件，以便重新编译。
make 100ask_imx6ull_defconfig	#加载 i.MX6ULL 开发板的默认编译配置文件，以便编译正确的内核镜像。
make zImage -j4					#编译内核镜像 -j4 参数指定使用 4 个 CPU 核心
make dtbs						#编译设备树二进制文件
cp arch/arm/boot/zImage ~/nfs_rootfs
cp arch/arm/boot/dts/100ask_imx6ull-14x14.dtb ~/nfs_rootfs
```

#### 编译内核模块

`make modules`

#### 安装内核模块到 Ubuntu 某个目录下备用

`make ARCH=arm INSTALL_MOD_PATH=/home/book/nfs_rootfs modules_install`

### 放到开发板上

```shell
cp /mnt/zImage /boot
cp /mnt/100ask_imx6ull-14x14.dtb /boot
cp /mnt/lib/modules /lib -rfd
sync
```

### 编译测试驱动程序

进入 100ask_imx6ull_src_bin 目录 ， 修改Makefile 文 件 “KERN_DIR”为自己的内核所在路径。

`/home/book/100ask_imx6ull-sdk/Linux-4.9.88`

make编译生成驱动文件.ko和测试文件

`insmod hello_drv.ko`装载驱动

通过lsmod命令查看安装的模块

## 解压编译 bootloader

#### 编译u-boot

```shel
cd /home/book/100ask_imx6ull-sdk
cd Uboot-2017.03
make distclean
make mx6ull_14x14_evk_defconfig
make
```

生成u-boot-dtb.imx

#### 构建 IMX6ULL Pro 版的根文件系统

```shell
cd /home/book/100ask_imx6ull-sdk
cd Buildroot_2020.02.x
make clean
make 100ask_imx6ull_pro_ddr512m_systemV_qt5_defconfig
make all -j4
```

编译需要很久，编译成功后文件输出路径为 output/images 

100ask-imx6ull-pro512d-systemv-v1.img重命名为emmc.img(或sdcard.img)，把emmc.img(或sdcard.img) 放到“100ask_imx6ull 烧写工具”的 files 目录里，就可以使用工具烧写。

## 烧写EMMC、SD卡

NFS根文件系统、插USB OTG线配合软件烧录、用读卡器插入SD卡烧录

## 通过Uboot使用网络启动系统

需要在uboot中设置一下bootargs和bootcmd

```shell
#bootcmd 从网络tftp启动
setenv bootcmd 'tftp 80800000 zImage;tftp 83000000 imx6ull-alientek-emmc.dtb;bootz 80800000 - 83000000;'
saveenv
#bootargs 设置根文件系统放在EMMC的分区2下
setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'
saveenv
```

### 遇到的tftp报错

配置tftp，新建文件/etc/xinetd.d/tftp

加入以下内容：

```shell
service tftp
{
        socket_type = dgram
        wait = yes
        disable = no
        user = root
        protocol = udp
        server = /usr/sbin/in.tftpd
        server_args = -s /home/bcl/tftpboot
        disable = no
        per_source = 11
        cps =100 2
        flags =IPv4
}
```

如果u-boot 中能ping通主机，但是tftp下载失败：service tftpd-hpa restart 重启一下tftp服务器

[tftp的安装以及使用_xinetd 配置tftp_星火(star&fire)的博客-CSDN博客](https://blog.csdn.net/neuzhangno/article/details/126510177)

### 遇到的nfs报错

[u-boot NFS下载文件报错：Loading: *** ERROR: File lookup fail解决方法_uboot nfs loading:*_polaris_zgx的博客-CSDN博客](https://blog.csdn.net/polaris_zgx/article/details/103571725)

### 教程资料

[Imx6ull 开发板通过Uboot使用网络启动系统_正点原子imx6ull网络启动_TrueDei的博客-CSDN博客](https://blog.csdn.net/qq_17623363/article/details/120472730)

# **嵌入式Linux应用开发基础知识**

gcc适用于x86-64，不能直接在ARM开发板上使用。通过交叉编译生成的可执行文件才能在开发板上使用。

|    选项     |                     说明                     |              示例              |
| :---------: | :------------------------------------------: | :----------------------------: |
|     -E      |  预处理指定的源文件，不进行编译，生成.i文件  |  gcc -E circle.c -o circle.i   |
|     -S      | 编译指定的源文件，但是不进行汇编，生成.s文件 |                                |
|     -c      |       编译、汇编但是不连接，生成.o文件       | gcc -c test1.c test2.c test3.c |
|     -o      |                  编译，输出                  |  gcc main.c func.c -o app.out  |
|   -I dir    |       指定 include 包含文件的搜索目录        |    gcc test.c -I ./testdir/    |
|     -g      |     生成调试信息，该程序可以被调试器调试     |                                |
| -save-temps |             不删除生成的中间文件             |                                |



[Makefile教程（绝对经典，所有问题看这一篇足够了）_GUYUEZHICHENG的博客-CSDN博客](https://blog.csdn.net/weixin_38391755/article/details/80380786)

在路径01_all_series_quickstart\04_嵌入式Linux应用开发基础知识\source\05_general_Makefile中有通用Makefile的模板。

## 指定编码格式

-finput-charset=XXXX 指定输入编码格式

-fexec-charset=XXXX 指定可执行文件的编码格式

## **交叉编译程序的万能命令**

./configure --host=arm-buildroot-linux-gnueabihf --prefix=$PWD/tmp

## 输入系统

Linux 系统为了统一管理这些输入设备，实现了一套能兼容所有输入设备的框架：输入系统。

所谓输入事件就是一个“struct input_event”结构体

```c
struct input_event {
	struct timeval time;
	__u16 type;
	__u16 code;
	__s32 value;
};
```

type：表示哪类事件

code：表示该类事件下的哪一个事件

value：表示事件值

同步事件也是一个 input_event 结构体，它的 type、code、value 三项都是 0，APP 已经读到了完整的数据。

## 网络编程

TCP：可靠，重传

UDP：不可靠

[Linux下TCP/IP网络编程示例——实现服务器/客户端通信（一）_linux tcp/ip编程例子_Mr_XJC的博客-CSDN博客](https://blog.csdn.net/Mr_XJC/article/details/106788694)

## 串口

### TTY

在 Linux 或 UNIX 中，TTY 变为了一个抽象设备。有时它指的是一个物理输入设备，例如串口，有时它指的是一个允许用户和系统交互的虚拟 TTY。

TTY 是 Linux 或 UNIX 的一个子系统，它通过 TTY 驱动程序在内核级别实现进程管理、行编辑和会话管理。

在大多数 *发行版* 中，你可以使用以下键盘快捷键来得到 TTY 屏幕：

- `CTRL + ALT + F1` – 锁屏
- `CTRL + ALT + F2` – 桌面环境
- `CTRL + ALT + F3` – TTY3
- `CTRL + ALT + F4` – TTY4
- `CTRL + ALT + F5` – TTY5
- `CTRL + ALT + F6` – TTY6

TTY0指向前台程序。

使用/dev/tty找到当前程序所使用的终端

### Terminal和Console的区别

Terminal含有远端的意思，Console为控制台，具有更大的权限，能查看更多的信息。

- 某一个Terminal可以是Console
- 不是所有Terminal都是Console

可以通过内核的cmdline指定控制台，比如console=ttyS0 console=tty，当console有多个取值时，使用最后一个取值来判断。

可以在Ubuntu启动时按住Esc进入内核设置界面，选择第二项Advanced options for Ubuntu，按E进入。找到带有quiet splash 的Linux那行，quiet可以删掉，然后显示出完整的内核启动过程，可以加上console=ttyS0来选择控制台。按F10保存重启。



行规程Line discipline

`stty -a `命令显示终端行规程的配置

[Linux终端和Line discipline图解_dog250的博客-CSDN博客](https://blog.csdn.net/dog250/article/details/78818612#:~:text=行规程规定了键盘,给出相应的输出。)

## I2C

在命令行中使用`i2cdetect -l`可以显示所有I2C总线

```shell
[root@imx6ull:~]# i2cdetect -l
i2c-1   i2c             21a4000.i2c                             I2C adapter
i2c-0   i2c             21a0000.i2c                             I2C adapter
```

`i2cdetect -y 1`显示I2C1上挂载的设备

```shell
[root@imx6ull:~]# i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- UU -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- UU -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: 60 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --

```

在开发板上有AP3216C三合一（红外、光强、距离）传感器

- 复位：往寄存器0写入0x4
- 使能：往寄存器0写入0x3
- 读光强：读寄存器0xc、0xd得到2字节的数据
- 读距离：读寄存器0xe、0xf得到2字节的数据

AP3216C设备地址为0x1e，在i2c0上

- 使用SMBus协议

    ```shell
    [root@imx6ull:~]# i2cset -f -y 0 0x1e 0 0x4
    [root@imx6ull:~]# i2cset -f -y 0 0x1e 0 0x3
    [root@imx6ull:~]# i2cget -f -y 0 0x1e 0xc 2
    [root@imx6ull:~]# i2cget -f -y 0 0x1e 0xc w
    ```

- 使用I2C协议

    ```shell
    [root@imx6ull:~]# i2ctransfer -f -y 0 w2@0x1e 0 0x4
    [root@imx6ull:~]# i2ctransfer -f -y 0 w2@0x1e 0 0x3
    [root@imx6ull:~]# i2ctransfer -f -y 0 w1@0x1e 0xc r2
    [root@imx6ull:~]# i2ctransfer -f -y 0 w1@0x1e 0xe r2
    ```

    

# 正点原子IMX6ULL应用编程

## Poky 交叉编译工具链

1. 使用命令`source /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi`使能环境变量，切换终端或者用户，需要重新使能。
2. 可以使用env命令查看环境变量有无交叉编译工具链
3. `arm-poky-linux-gnueabi-gcc -v`指令可以查看 gcc 版本，表明环境变量已经生效

设置完环境变量后，可以使用${CC} -o xxx xxx.c进行编译C文件

## 通用交叉编译工具链

1. 在/usr/local/arm下放入gcc-linaro-4.9.4- 2017.01-x86_64_arm-linux-gnueabihf.tar.xz

2. 解压该工具链
3. 修改环境变量sudo vi /etc/profile，加入export PATH=$PATH:/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin
4. 重启系统，验证是否安装成功arm-linux-gnueabihf-gcc -v

工具链压缩包在资料5、开发工具->1、交叉编译器中。

额外的库sudo apt-get install lsb-core lib32stdc++6





如果出现这个问题说明ncurses库没装，不能使用图形配置Linux

```shell
HOSTCC  scripts/kconfig/mconf.o
<command-line>: fatal error: curses.h: No such file or directory
compilation terminated.
make[1]: *** [scripts/Makefile.host:108: scripts/kconfig/mconf.o] Error 1
```

需要sudo apt-get install libncurses5-dev安装

[Linux下make menuconfig命令出现[scripts/kconfig/mconf.o\] Error 1......错误的解决办法_make menuconfig报错_西岸贤的博客-CSDN博客](https://blog.csdn.net/weixin_42570192/article/details/120678754)

## 点亮LED实验

LED的驱动位于/sys/class/leds下

在/sys/class/leds/sys-led中有三个属性文件：

1. brightness：亮度，0为灭，数值越大越亮，对于led而言只有亮和灭的区别
2. max_brightness：用于获取 LED 设备的最大亮度等级
3. trigger：触发模式，通过 cat 命令查看该属性文件，可获取 LED 支持的所有触发模式以及 LED 当前被设置的触发模式

方括号（[heartbeat]）括起来的表示当前 LED 对应的触发模式，none 表示无触发，常用的触发模式包括 none（无触发）、mmc0（当对 mmc0 设备发起读写操作的时候 LED 会闪烁）、timer（LED 会有规律的一 亮一灭，被定时器控制住）、heartbeat（心跳呼吸模式，LED 模仿人的心跳呼吸那样亮灭变化）。

```shell
echo timer > trigger //将 LED 触发模式设置为 timer
echo none > trigger //将 LED 触发模式设置为 none
echo 1 > brightness //点亮 LED echo 0 > brightness//熄灭 LED
```

## GPIO实验

GPIO的驱动位于/sys/class/gpio下

为 GPIO1、GPIO2、GPIO3、GPIO4、GPIO5，在这里分别对应 gpiochip0、gpiochip32、 gpiochip64、gpiochip96、gpiochip128

export（只写）：用于将指定编号的 GPIO 引脚导出。会在目录下生成一个对应名字的文件夹。

unexport（只写）：将导出的 GPIO 引脚删除。文件夹会消失。

[poll函数详解_青季的博客-CSDN博客](https://blog.csdn.net/skypeng57/article/details/82743681)

## KEY实验

```c
struct input_event {
 struct timeval time;
 __u16 type;
 __u16 code;
 __s32 value;
};
```

分析 3 个成员变量 type、code、value。得到输入设备数据。

查看/proc/bus/input/devices，看key设备节点。

![image-20230814210900500](./IMX6ULL开发.assets/image-20230814210900500.png)

type 等于 1，表示按键类事件，code 等于 114、value 等于1，所以表示按键 KEY_VOLUMEDOWN 被按下。value 等于0表示松开。value 等于 2，表示长按状态。

EV_SYN 同步类事件（type=0）中的 SYN_REPORT 事件（code=0），表示本轮数 据已经完整、报告同步。

## 串口实验

终端的分类 

⚫ 本地终端：例如对于我们的个人 PC 机来说，PC 机连接了显示器、键盘以及鼠标等设备，这样的 一个显示器/键盘组合就是一个本地终端；同样对于开发板来说也是如此，开发板也可以连接一个 LCD 显示器、键盘和鼠标等，同样可以构成本地终端。 /dev/ttyX

⚫ 用串口连接的远程终端：对于嵌入式 Linux 开发来说，这是最常见的终端—串口终端。譬如我们的 开发板通过串口线连接到一个带有显示器和键盘的 PC 机，在 PC 机通过运行一个终端模拟程序， 譬如 Windows 超级终端、putty、MobaXterm、SecureCRT 等来获取并显示开发板通过串口发出的 数据、同样还可以通过这些终端模拟程序将用户数据通过串口发送给开发板 Linux 系统，系统接收 到数据之后便会进行相应的处理、譬如执行某个命令，这就是一种人机交互！ /dev/pts/X

⚫ 基于网络的远程终端：譬如我们可以通过 ssh、Telnet 这些协议登录到一个远程主机。/dev/ttymxcX

```c
struct termios
{
 tcflag_t c_iflag; /* input mode flags */
 tcflag_t c_oflag; /* output mode flags */
 tcflag_t c_cflag; /* control mode flags */
 tcflag_t c_lflag; /* local mode flags */
 cc_t c_line; /* line discipline */
 cc_t c_cc[NCCS]; /* control characters */
 speed_t c_ispeed; /* input speed */
 speed_t c_ospeed; /* output speed */
};
```

## 看门狗实验

看门狗实验只需要用到ioctl()函数。

|    ioctl指令     |          说明          |
| :--------------: | :--------------------: |
| WDIOC_GETSUPPORT | 获取看门狗支持哪些功能 |
| WDIOC_SETOPTIONS |  用于开启或关闭看门狗  |
| WDIOC_KEEPALIVE  |        喂狗操作        |
| WDIOC_SETTIMEOUT |   设置看门狗超时时间   |
| WDIOC_GETTIMEOUT |   获取看门狗超时时间   |

```c
struct watchdog_info {
 __u32 options; /* Options the card/driver supports */
 __u32 firmware_version; /* Firmware version of the card */
 __u8 identity[32]; /* Identity of the board */
};
```

# 正点原子IMX6ULL系统移植和根文件构建

## Uboot相关

### 编译Uboot

将uboot放入到一个文件夹中解压，执行make命令如下：

```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_14x14_ddr512_emmc_defconfig
make V=1 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j12
```

生成一个u-boot.bin，通过./tools/mkimage软件添加头部信息，生成u-boot.imx

### Uboot命令的使用

使用“？”查看所有命令的使用方法，使用“？ cmd”查询某个命令的信息

### Uboot更新Uboot

EMMC

```shell
mmc dev 1 0               //切换到 EMMC 分区 0
tftp 80800000 u-boot.imx  //下载 u-boot.imx 到 DRAM
mmc write 80800000 2 2EE  //烧写 u-boot.imx 到 EMMC 中 2EE由u-boot.imx的大小决定
mmc partconf 1 1 0 0      //分区配置，EMMC 需要这一步！
```

- `mmc dev 0` ：选择SD卡设备
- `mmc dev 1` ：选择EMMC 设备

千万不要写 SD 卡或者 EMMC 的前两个块(扇区)，里面保存着分区表！ 

### 将Uboot烧录到SD卡中

使用imxdownload下载uboot，使用命令 ./imxdownload u-boot.bin /dev/sdb (看具体名称)

[嵌入式Linux--U-Boot（九）通过TFT/NFS网络更新U-Boot、Kernel、DTB文件_liefyuan的博客-CSDN博客](https://blog.csdn.net/qq_28877125/article/details/111148590)

### 从EMMC启动Linux

首先查看EMMC中是否有系统，zImage和dtb。

mmc dev 1 //切换到EMMC

fatls mmc 1:1 //查看EMMC分区1中的文件

fatload mmc 1:1 80800000 zImage //下载镜像到ddr的0x80800000处

fatload mmc 1:1 83000000 dtb //下载设备树到ddr的0x83000000处

bootz 80800000 - 83000000	//启动内核



分区0 Uboot 分区1 zImage 和 dtb 分区2 根文件系统

### 使用网络启动Linux

bootz、bootm、boot

需要准备Linux镜像文件和设备树。（设备树需要有权限，比如777）

#### bootz

使用 tftp 命令 将zImage 下载到DRAM的0X80800000地址处，然后将设备树imx6ull-14x14-emmc-7-1024x600- c.dtb 下载到 DRAM 中的 0X83000000 地址处，最后之后命令 bootz 启动，命令如下：

```shell
tftp 80800000 zImage
tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb	#根据实际使用的设备树更改
bootz 80800000 - 83000000
```

#### bootm

用于uImage镜像启动，一般不用

#### boot

将bootz中的操作进行打包存放在bootcmd中

```shell
setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb; 
bootz 80800000 - 83000000'
saveenv
boot
```

#### run

使用run配合自己写的环境变量也可以启动系统

```shell
setenv mybootnet 'tftp 80800000 zImage; tftp 83000000imx6ull-14x14-emmc-7-1024x600-c.dtb; 
bootz 80800000 - 83000000'
saveenv
```

比如run mybootemmc 、run mytoobnand

### Uboot源码目录分析

1. 分析之前要先编译Uboot
2. arch/arm/cpu/u-boot.lds是整个Uboot的链接脚本
3. board/freescale/mx6ullevk是重点
4. config目录是uboot是默认配置文件目录。配置文件对应不同的板子。自己板子配置文件名字应该有所区分。
5. 移植Uboot需要重点关注./board/freescale ./configs

### NXP官方Uboot移植

[i.MX6ULL系统移植 | 移植NXP官方uboot 2016.03版本（2022.04.06更新）_nxp uboot仓库_Mculover666的博客-CSDN博客](https://blog.csdn.net/Mculover666/article/details/120537028)

拷贝官方的Uboot压缩包，解压到某个目录下

执行下面的命令

```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_14x14_evk_emmc_defconfig
make V=1 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16
```

或者参考之前编译写一个脚本文件

编译完成后，将.imx文件烧写进SD卡或者EMMC中，然后启动

1. uboot能正常启动
2. LCD驱动要更改，显示的是NXP官方的
3. NET初始化失败

#### 在Uboot中加入ALPHA开发板或者自己的开发板

#### 修改默认配置文件

借鉴NXP官方6ULL EVK开发板，在官方的配置文件上修改即可

所使用的是mx6ull_14x14_evk_emmc_defconfig，复制并修改名称

#### 添加板子对应的头文件

在自己的配置文件中加入头文件，比如

CONFIG_SYS_EXTRA_OPTIONS="IMX_CONFIG=board/freescale/mx6ull_alientek_ emmc/imximage.cfg,MX6ULL_EVK_EMMC_REWORK"

 CONFIG_TARGET_MX6ULL_ALIENTEK_EMMC=y

不同的板子有一些需要配置的信息，一般是在一个头文件里面配置，每个板子有一个

官方NXP的头文件路径为/home/bcl/uboot/uboot-imx-rel_imx_4.1.15_2.1.0_ga_alientek/include/configs/mx6sllevk.h

拷贝该头文件，并且改名mx6ull_alientek_emmc.h，更改条件编译宏定义

#### 添加板子对应的板级文件夹

使用cp mx6ullevk/ mx6ull_alientek_emmc -r命令复制一份板级文件夹来修改

修改内部的mx6ullevk.c文件的名字，同时要更改Makefile里面的变量名

修改imximage.cfg中PLUGIN，将文件夹路径修改

修改Kconfig中的目标文件名，与配置文件中相同，下面关于evk相关的都要更改

修改MAINTAINERS中的evk相关东西

#### 修改Uboot的配置界面

修改./arch/arm/cpu/armv7/mx6/Kconfig

在207行加入如下内容：

```shell
config TARGET_MX6ULL_ALIENTEK_EMMC
	bool "Support mx6ull_alientek_emmc"
	select MX6ULL
	select DM
	select DM_THERMAL
```

在最后一行的 endif 的前一行添加如下内容：

source "board/freescale/mx6ull_alientek_emmc/Kconfig"

### LCD驱动修改

Uboot修改驱动都是在xxxx.c和xxxx.h中修改，xxxx为板子名称

重点关注

1. LCD使用的GPIO，查看Uboot中的LCDIO配置是否正确 iomux_v3_cfg_t const lcd_pads[]
2. LCD背光引脚GPIO的配置
3. LCD配置参数是否正确 display_info_t const displays[]

还要修改头文件中panel对应的屏幕名称

pixclock像素时钟计算方法：pixclock=(1/lcdclk)*10^12=19531

以正点原子的 7 寸 1024*600 分辨率的屏幕(ATK7016)为例， 屏幕要求的像素时钟为 51.2MHz，因此：

pixclock=(1/51200000)*10^12=19531

### 网络驱动修改

重点：

1. ENET1 复位引脚初始化
2. LAN8720A 的器件 ID
3. LAN8720 驱动

#### 网络 PHY 地址修改

在mx6ull_alientek_emmc.h中修改ENET1 和 ENET2 的 PHY 地址和驱动

根据原理图正点原子的 I.MX6U-ALPHA 开发板 ENET1 的 PHY 地址为 0X0，ENET2 的 PHY 地址为 0X1，所以需要将第 335 行的宏 CONFIG_FEC_MXC_PHYADDR 改为 0x0。

```c
#if (CONFIG_FEC_ENET_DEV == 0)
#define IMX_FEC_BASE ENET_BASE_ADDR
#define CONFIG_FEC_MXC_PHYADDR 0x0
#define CONFIG_FEC_XCV_TYPE RMII
#elif (CONFIG_FEC_ENET_DEV == 1)
#define IMX_FEC_BASE ENET2_BASE_ADDR
#define CONFIG_FEC_MXC_PHYADDR 0x1
#define CONFIG_FEC_XCV_TYPE RMII
#endif
#define CONFIG_ETHPRIME			"FEC"

#define CONFIG_PHYLIB
#define CONFIG_PHY_SMSC
#endif
```

#### 删除 uboot 中 74LV595 的驱动代码

删除代码替换为

```c
#define ENET1_RESET IMX_GPIO_NR(5, 7)
#define ENET2_RESET IMX_GPIO_NR(5, 8)
```

将74LV595的初始化代码都删掉

#### 添加 I.MX6U-ALPHA 开发板网络复位引脚驱动

结构体数组 fec1_pads 和 fec2_pads 是 ENET1 和 ENET2 这两个网口的 IO 配置参数

```c
MX6_PAD_SNVS_TAMPER7__GPIO5_IO07 | MUX_PAD_CTRL(NO_PAD_CTRL),
MX6_PAD_SNVS_TAMPER8__GPIO5_IO08 | MUX_PAD_CTRL(NO_PAD_CTRL),
```

setup_iomux_fec函数添加

```c
static void setup_iomux_fec(int fec_id)
{
	if (fec_id == 0)
	{
		imx_iomux_v3_setup_multiple_pads(fec1_pads,
		ARRAY_SIZE(fec1_pads));
		gpio_direction_output(ENET1_RESET, 1);
		gpio_set_value(ENET1_RESET, 0);
		mdelay(20);
		gpio_set_value(ENET1_RESET, 1);
	}
	else
	{
		imx_iomux_v3_setup_multiple_pads(fec2_pads,
		ARRAY_SIZE(fec2_pads));
		gpio_direction_output(ENET2_RESET, 1);
		gpio_set_value(ENET2_RESET, 0);
		mdelay(20);
		gpio_set_value(ENET2_RESET, 1);
	}
}
```

#### 修改 drivers/net/phy/phy.c 文件中的函数 genphy_update_link

打开文件 drivers/net/phy/phy.c，找到函数 genphy_update_link，这是个通用 PHY 驱动函数，此函数用于更新 PHY 的连接状态和速度。使用 LAN8720A 的时候需要在此函数中添加一些代码，修改后的函数 genphy_update_link 函数如下所示：

```c
int genphy_update_link(struct phy_device *phydev)
{
	unsigned int mii_reg;

#ifdef CONFIG_PHY_SMSC
	static int lan8720_flag = 0;
	int bmcr_reg = 0;
	if (lan8720_flag == 0) {
		bmcr_reg = phy_read(phydev, MDIO_DEVAD_NONE, MII_BMCR); 
		phy_write(phydev, MDIO_DEVAD_NONE, MII_BMCR, BMCR_RESET); 
		while(phy_read(phydev, MDIO_DEVAD_NONE, MII_BMCR) & 0X8000) {
			udelay(100); 
	}
	phy_write(phydev, MDIO_DEVAD_NONE, MII_BMCR, bmcr_reg); 
	lan8720_flag = 1;
}
#endif

	/*
	* Wait if the link is up, and autonegotiation is in progress
	* (ie - we're capable and it's not done)
	*/
	mii_reg = phy_read(phydev, MDIO_DEVAD_NONE, MII_BMSR);
......

	return 0;
}

```

### bootcmd 和 bootargs 环境变量

bootcmd用于配置镜像和设备树，将镜像和设备树放在对应的地址中，然后启动

CONFIG_BOOTCOMMAND设置默认bootcmd 

bootargs是命令行参数，用于传递给Linux kernel

CONFIG_BOOTARGS设置bootargs 

```shell
#bootcmd 从网络tftp启动
setenv bootcmd 'tftp 80800000 zImage;tftp 83000000 imx6ull-alientek-emmc.dtb;bootz 80800000 - 83000000;'
saveenv
#bootargs 设置根文件系统放在EMMC的分区2下
setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'
saveenv
```

### Uboot DDR初始化

#### 裸机

imxdownload软件下载，会在bin文件头添加IVT DCD数据

#### Uboot

编译生成u-boot.imx，已经包含了IVT DCD数据

头部信息怎么添加的？

./tools/mkimage -n board/freescale/mx6ull_alientek_emmc/imximage.cfg.cfgtmp -T imximage -e 0x87800000 -d u-boot.bin u-boot.imx

在imximage.cfg中保存的就是DCD数据

如何修改DCD数据？

修改imximage.cfg，默认是给512MB写的。需要先对DDR进行校准，在裸机教程中的DDR校准、超频中。

### Uboot 图形化配置

uboot 或 Linux 内核可以通过输入“make menuconfig”来打开图形化配置界面，需要先安装ncurses 库。

```shell
sudo apt-get install build-essential 
sudo apt-get install libncurses5-dev
```

在打开图形化配置界面之前，要先使用“make xxx_defconfig”对 uboot 进行一次默认配置

```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_alientek_emmc_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
```

图形化配置操作方式：

后按下“Y”键就会将相应的代码编译进 Uboot 中，菜单前面变为“< * >”。

按下 “N”键不编译相应的代码。

按下“M”键就会将相应的代码编译为模块，菜单前面变为“< M >”。常用于Linux内核中。

按下“?”键查看此菜单的帮助信息，按下“/”键打开搜索框，可以在搜索框输入要搜索的内容。



因为 mx6ull_alientek_emmc.sh 在编译之前会清理工程，会删除掉.config 文件！

所以使用make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16编译Uboot

## Linux内核相关

### Linux内核编译

先安装lzop 库，sudo apt-get install lzop

选择一个合适的内核压缩包，例如linux-imx-rel_imx_4.1.15_2.1.1_ga_alientek_v2.2.tar.bz2

解压到某个目录下，新建名为 “mx6ull_alientek_emmc.sh”的 shell 脚本，然后在这个 shell 脚本里面输入如下所示内容：

```shell
#!/bin/sh
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- imx_v7_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- all -j16
```

执行该脚本完成编译，可能需要先赋予权限。

编译好的dtb和zImage在./linux-imx-rel_imx_4.1.15_2.1.1_ga_alientek_v2.2/arch/arm/boot下

### Linux源码目录分析

arch/arm/boot下有zImage

arch/arm/boot/dts下有dts和dtb

Documentation/devicetree/bindings目录下的文档很重要，设备树绑定信息

### NXP官方Linux内核移植

#### 官方开发板Linux内核编译

1. 将NXP的Linux内核压缩包拷贝到Ubuntu下
2. 解压
3. 编译NXP官方的EVK开发板内核，得到镜像和设备树

defconfig文件默认放在arch/arm/configs下，教程使用imx6ull-14x14-evk-emmc.dtb和zImage

将imx6ull-14x14-evk-emmc.dtb和zImage拷贝到tftpboot下，使用网络启动

显示Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)说明没有根文件系统。

#### 在Linux中添加自己的开发板

#### 添加开发板默认配置文件

1. 将 arch/arm/configs 目录下的imx_v7_mfg_defconfig 重新复制一份imx_alientek_emmc_defconfig

2. 屏蔽“CONFIG_ARCH_MULTI_V6=y“这一行，因为 I.MX6ULL 是 ARMV7 架构的，因此要屏蔽掉 V6 相关选项，否则后续驱动模块可能无法加载。

#### 添加开发板对应的设备树文件

1. 进入目录 arch/arm/boot/dts 中，复制一 份 imx6ull-14x14-evk.dts，然后将其重命名为 imx6ull-alientek-emmc.dts

2. 修改arch/arm/boot/dts/Makefile，找到”dtb- $(CONFIG_SOC_IMX6ULL)”配置项，加入imx6ull-alientek-emmc.dtb

#### CPU主频修改

1. 修改驱动之前，要保证板子能够正常启动

2. 根文件系统处理好，使用现成的根文件系统，保证emmc烧写了系统。设置bootcmd和bootargs

    ```shell
    #bootcmd 从网络tftp启动
    setenv bootcmd 'tftp 80800000 zImage;tftp 83000000 imx6ull-alientek-emmc.dtb;bootz 80800000 - 83000000;'
    saveenv
    #bootargs 设置根文件系统放在EMMC的分区2下
    setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'
    saveenv
    ```

3. EMMC驱动修补，在imx6ull-alientek-emmc.dts中找到usdhc2节点，改为

    ```shell
    &usdhc2 {
    	pinctrl-names = "default", "state_100mhz", "state_200mhz";
    	pinctrl-0 = <&pinctrl_usdhc2_8bit>;
    	pinctrl-1 = <&pinctrl_usdhc2_8bit_100mhz>;
    	pinctrl-2 = <&pinctrl_usdhc2_8bit_200mhz>;
    	bus-width = <8>;
    	non-removable;
    	no-1-8-v; #关闭 EMMC 1.8V 供电选项
    	status = "okay";
    };
    ```

输入cat /proc/cpuinfo查看cpu信息

BogoMIPS 是 Linux 系统中 衡量处理器运行速度的一个“尺子”，处理器性能越强，主频越高，BogoMIPS 值就越大。

/sys/bus/cpu/devices/cpu0/cpufreq查看详细信息



在menuconfig图形化配置中选择Ondemand模式来动态调节cpu频率，需要重新编译内核。

```shell
#图形化配置路径
CPU Power Management 
 -> CPU Frequency scaling 
 -> Default CPUFreq governor
```

#### 网络驱动修改

都是修改设备树

##### 使能 8 线 EMMC 驱动

```
&usdhc2 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc2_8bit>;
	pinctrl-1 = <&pinctrl_usdhc2_8bit_100mhz>;
	pinctrl-2 = <&pinctrl_usdhc2_8bit_200mhz>;
	bus-width = <8>;
	non-removable;
	no-1-8-v;
	status = "okay";
};
```

##### 修改 LAN8720 的复位以及网络时钟引脚驱动

删除pinctrl_spi4: spi4grp下关于SNVS_TAMPER7和SNVS_TAMPER8相关的代码

删除spi4中pinctrl-assert-gpios = <&gpio5 8 GPIO_ACTIVE_LOW>;和cs-gpios = <&gpio5 7 0>;

修改iomuxc_snvs

```
&iomuxc_snvs {
	pinctrl-names = "default_snvs";
        pinctrl-0 = <&pinctrl_hog_2>;
        imx6ul-evk {
        
        	/*省略掉其他，在该节点最下面添加*/
        	
			/*enet1 reset zuozhongkai*/
			pinctrl_enet1_reset: enet1resetgrp {
				fsl,pins = <
					/* used for enet1 reset */
					MX6ULL_PAD_SNVS_TAMPER7__GPIO5_IO07 0x10B0 
				>;
			};
		
			/*enet2 reset zuozhongkai*/
			pinctrl_enet2_reset: enet2resetgrp {
				fsl,pins = <
					/* used for enet2 reset */
					MX6ULL_PAD_SNVS_TAMPER8__GPIO5_IO08 0x10B0 
				>;
			};
        };
};
```

修改pinctrl_enet1: enet1grp和pinctrl_enet2: enet2grp，修改为

MX6UL_PAD_ENET1_TX_CLK__ENET1_REF_CLK1 0x4001b009

MX6UL_PAD_ENET2_TX_CLK__ENET2_REF_CLK2 0x4001b009

原来默认值为 0x4001b031

##### 修改 fec1 和 fec2 节点的 pinctrl-0 属性

修改fec1和fec2两个节点

```
&fec1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_enet1
				 &pinctrl_enet1_reset>;
	phy-mode = "rmii";
	phy-handle = <&ethphy0>;
	phy-reset-gpios = <&gpio5 7 GPIO_ACTIVE_LOW>;
	phy-reset-duration = <200>;
	status = "okay";
};

&fec2 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_enet2
				 &pinctrl_enet2_reset>;
	phy-mode = "rmii";
	phy-handle = <&ethphy1>;
	phy-reset-gpios = <&gpio5 8 GPIO_ACTIVE_LOW>;
	phy-reset-duration = <200>;
	status = "okay";

	mdio {
		#address-cells = <1>;
		#size-cells = <0>;

		ethphy0: ethernet-phy@0 {
			compatible = "ethernet-phy-ieee802.3-c22";
			smsc,disable-energy-detect;
			reg = <0>;
		};

		ethphy1: ethernet-phy@1 {
			compatible = "ethernet-phy-ieee802.3-c22";
			smsc,disable-energy-detect;
			reg = <1>;
		};
	};
};
```

至此Linux通用PHY驱动已经完成，剩下就是驱动LAN8720A

```shell
Configuring network interfaces... fec 20b4000.ethernet eth0: Freescale FEC PHY driver [Generic PHY] (mii_bus:phy_addr=20b4000.ethernet:01, irq=-1)
```

启动会显示Generic PHY，代表通用PHY驱动成功使用

##### 修改 fec_main.c 文件

drivers/net/ethernet/freescale/fec_main.c

在3452行加入以下内容

```c
/* 设置 MX6UL_PAD_ENET1_TX_CLK 和 MX6UL_PAD_ENET2_TX_CLK
 * 这两个 IO 的复用寄存器的 SION 位为 1。
 */
void __iomem *IMX6U_ENET1_TX_CLK;
void __iomem *IMX6U_ENET2_TX_CLK;

IMX6U_ENET1_TX_CLK = ioremap(0X020E00DC, 4);
writel(0X14, IMX6U_ENET1_TX_CLK);

IMX6U_ENET2_TX_CLK = ioremap(0X020E00FC, 4);
writel(0X14, IMX6U_ENET2_TX_CLK);
```

修改完最好先重新编译一次Linux内核

##### 配置 Linux 内核，使能 LAN8720 驱动

make menuconfig打开图形化配置界面，按照下述步骤使能LA

```
-> Device Drivers 
	-> Network device support 
 		-> PHY Device support and infrastructure 
			-> Drivers for SMSC PHYs
```

##### 修改 smsc.c 文件

drivers/net/phy/smsc.c

需要添加\#include <linux/of_gpio.h> #include <linux/io.h>

```c
static int smsc_phy_reset(struct phy_device *phydev)
{
	int err, phy_reset;
	int msec = 1;
	struct device_node *np;
	int timeout = 50000;

	if(phydev->addr == 0) /* FEC1 */ {
		np = of_find_node_by_path("/soc/aips-bus@02100000/ethernet@
							02188000");
		if(np == NULL) {
			return -EINVAL;
		}
 	}

 	if(phydev->addr == 1) /* FEC2 */ {
 		np = of_find_node_by_path("/soc/aips-bus@02000000/ethernet@
							020b4000");
		if(np == NULL) {
			return -EINVAL;
		}
	}

	err = of_property_read_u32(np, "phy-reset-duration", &msec);
	/* A sane reset duration should not be longer than 1s */
	if (!err && msec > 1000)
		msec = 1;
	phy_reset = of_get_named_gpio(np, "phy-reset-gpios", 0);
	if (!gpio_is_valid(phy_reset))
		return;

	gpio_direction_output(phy_reset, 0);
	gpio_set_value(phy_reset, 0);
	msleep(msec);
	gpio_set_value(phy_reset, 1);

	int rc = phy_read(phydev, MII_LAN83C185_SPECIAL_MODES);
	if (rc < 0)
		return rc;

	/* If the SMSC PHY is in power down mode, then set it
	 * in all capable mode before using it.
	 */
	if ((rc & MII_LAN83C185_MODE_MASK) == MII_LAN83C185_MODE_POWERDOWN) {
		int timeout = 50000;

		/* set "all capable" mode and reset the phy */
		rc |= MII_LAN83C185_MODE_ALL;
		phy_write(phydev, MII_LAN83C185_SPECIAL_MODES, rc);
	}
		phy_write(phydev, MII_BMCR, BMCR_RESET);
		/* wait end of reset (max 500 ms) */
	do {
		udelay(10);
		if (timeout-- == 0)
			return -1;
		rc = phy_read(phydev, MII_BMCR);
	} while (rc & BMCR_RESET);
	
	return 0;
}
```

启动后显示如下代表修改完成

```shell
fec 20b4000.ethernet eth0: Freescale FEC PHY driver [SMSC LAN8710/LAN8720] (mii_bus:phy_addr=20b4000.ethernet:01, irq=-1)
```

## 根文件系统构建

根文件系统首先是内核启动时所mount的第一个文件系统，内核代码映像文件保存在根文件系统中，而系统引导启动程序会在根文件系统挂载之后从中把一些基本的初始化脚本和服务等加载到内存中去运行。

/etc/：存储重要的配置文件。 

/bin/：存储常用且开机时必须用到的执行文件。 

/sbin/：存储着开机过程中所需的系统执行文件。 

/lib/：存储/bin/及/sbin/的执行文件所需的链接库，以及Linux的内核模块。 

/dev/：存储设备文件。

五大目录必须存储在根文件系统上，缺一不可。

### BusyBox 构建根文件系统

一般我们在 Linux 驱动开发的时候都是通过 nfs 挂载根文件系统的，当产品最终上市开卖的时候才会将根文件系统烧写到 EMMC 或者 NAND 中。

#### 编译 BusyBox 构建根文件系统

1. 在nfs服务器目录下创建一个rootfs的文件夹

2. 将busybox-1.29.0.tar.bz2的源码放入并解压

3. 修改makefile，添加交叉编译器，设置ARCH 和 CROSS_COMPILE

    ```
    CROSS_COMPILE ?= /usr/local/arm/gcc-linaro-4.9.4-2017.01- x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-
    
    ARCH ?= arm
    ```

4. busybox支持中文字符

    修改busybox-1.29.0/libbb/printable_string.c

    ```c
    const char* FAST_FUNC printable_string(uni_stat_t *stats, const char *str)
    {
    	char *dst;
    	const char *s;
    
    	s = str;
    	while (1) {
    		unsigned char c = *s;
    		if (c == '\0') {
    			/* 99+% of inputs do not need conversion */
    			if (stats) {
    				stats->byte_count = (s - str);
    				stats->unicode_count = (s - str);
    				stats->unicode_width = (s - str);
    			}
    			return str;
    		}
    		if (c < ' ')
    			break;
    		/* 注释掉下面这两行代码 */
    		/*if (c >= 0x7f)
    			break;*/
    		s++;
    	}
    
    #if ENABLE_UNICODE_SUPPORT
    	dst = unicode_conv_to_printable(stats, str);
    #else
    	{
    		char *d = dst = xstrdup(str);
    		while (1) {
    			unsigned char c = *d;
    			if (c == '\0')
    				break;
    			/* 修改下面代码 */
    			/* if (c < ' ' || c >= 0x7f) */
    			if (c < ' ')
    				*d = '?';
    			d++;
    		}
    		if (stats) {
    			stats->byte_count = (d - dst);
    			stats->unicode_count = (d - dst);
    			stats->unicode_width = (d - dst);
    		}
    	}
    #endif
    	return auto_string(dst);
    }
    ```

    修改busybox-1.29.0/libbb/unicode.c

    ```c
    static char* FAST_FUNC unicode_conv_to_printable2(uni_stat_t *stats, const char *src, unsigned width, int flags)
    {
    	char *dst;
    	unsigned dst_len;
    	unsigned uni_count;
    	unsigned uni_width;
    
    	if (unicode_status != UNICODE_ON) {
    		char *d;
    		if (flags & UNI_FLAG_PAD) {
    			d = dst = xmalloc(width + 1);
    			while ((int)--width >= 0) {
    				unsigned char c = *src;
    				if (c == '\0') {
    					do
    						*d++ = ' ';
    					while ((int)--width >= 0);
    					break;
    				}
    				/* 修改下面一行代码 */
    				/* *d++ = (c >= ' ' && c < 0x7f) ? c : '?'; */
    				*d++ = (c >= ' ') ? c : '?';
    				src++;
    			}
    			*d = '\0';
    		} else {
    			d = dst = xstrndup(src, width);
    			while (*d) {
    				unsigned char c = *d;
    				/* 修改下面一行代码 */
    				/* if (c < ' ' || c >= 0x7f) */
    				if(c < ' ')
    					*d = '?';
    				d++;
    			}
    		}
    ```

5. 配置busybox

    有下面三种配置选择，defconfig，缺省配置；allyesconfig，全选配置；allnoconfig，最小配置

    make defconfig

    make menuconfig

```
Location: 
   -> Settings 
    	-> Build static binary (no shared libs) #不要选中，要使用动态编译
```

```
Location: 
	-> Settings 
		-> vi-style line editing commands	#选中
```

```
Location: 
	-> Linux Module Utilities
		-> Simplified modutils	#取消勾选

```

```
Location: 
	-> Linux System Utilities 
 		-> mdev (16 kb) #确保下面的全部选中，默认都是选中的
```

```
Location: 
 -> Settings
 -> Support Unicode #选中
-> Check $LC_ALL, $LC_CTYPE and $LANG environment variables #选中
```

6. 编译busybox

```shell
make install CONFIG_PREFIX=/home/bcl/nfs/rootfs
```

### 向根文件系统添加 lib 库

#### 向 rootfs 的“/lib”目录添加库文件

1. 在rootfs下创建lib文件夹 mkdir lib

2. 拷贝交叉编译器目录下的动态库拷贝到lib下

```shell
cd /usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/lib
cp *so* *.a /home/bcl/nfs/rootfs/lib/ -d #拷贝符号连接
```

3. ld-linux-armhf.so.3本身就是一个软连接文件，所以需要拷贝原始文件

```shell
rm /home/bcl/nfs/rootfs/lib/ld-linux-armhf.so.3
cd /usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/lib
cp ld-linux-armhf.so.3 /home/bcl/nfs/rootfs/lib/ 
```

4. 拷贝另一个文件夹中的动态库

```shell
cd /usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/lib
cp *so* *.a /home/bcl/nfs/rootfs/lib/ -d
```

#### 向 rootfs 的“usr/lib”目录添加库文件

```shell
cd /usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/usr/lib
cp *so* *.a /home/bcl/nfs/rootfs/usr/lib/ -d
```

最后使用du命令查看rootfs的大小

```shell
cd /home/bcl/nfs/rootfs //进入根文件系统目录
du ./lib ./usr/lib/ -sh //查看 lib 和 usr/lib 这两个目录的大小
```

```shell
bcl@bcl-virtual-machine:~/nfs/rootfs$ du ./lib ./usr/lib/ -sh
57M	./lib
67M	./usr/lib/
```

#### 创建其他文件夹

创建 dev、proc、mnt、sys、tmp 和 root文件夹

```shell
bcl@bcl-virtual-machine:~/nfs/rootfs$ mkdir dev
bcl@bcl-virtual-machine:~/nfs/rootfs$ mkdir proc
bcl@bcl-virtual-machine:~/nfs/rootfs$ mkdir mnt
bcl@bcl-virtual-machine:~/nfs/rootfs$ mkdir sys
bcl@bcl-virtual-machine:~/nfs/rootfs$ mkdir tmp
bcl@bcl-virtual-machine:~/nfs/rootfs$ mkdir root
```

#### 设置Uboot的root值关联nfs下的rootfs

```shell
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.5.11:
/home/bcl/nfs/rootfs,proto=tcp rw ip=192.168.5.9:192.168.5.11:192.168.5.1:
255.255.255.0::eth0:off' //设置 bootargs
saveenv //保存环境变量
```

使用boot命令启动

会显示can't run '/etc/init.d/rcS': No such file or directory，需要继续完善。

### 完善根文件系统

#### 创建/etc/init.d/rcS文件

rcS 是个 shell 脚本，Linux 内核启动以后需要启动一些服务，而 rcS 就是规定启动哪些文件的脚本文件。

```shell
#!/bin/sh

PATH=/sbin:/bin:/usr/sbin:/usr/bin:$PATH
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/lib:/usr/lib
export PATH LD_LIBRARY_PATH

mount -a
mkdir /dev/pts
mount -t devpts devpts /dev/pts

echo /sbin/mdev > /proc/sys/kernel/hotplug	#热插拔
mdev -s
```

要给可执行权限！

#### 创建/etc/fstab 文件

在 rootfs 中创建/etc/fstab 文件，fstab 在 Linux 开机以后自动配置哪些需要自动挂载的分区。

```shell
#<file system>  <mount point> 	<type> 	<options> 	<dump> 	<pass>
proc		/proc		proc	defaults	0	0
tmpfs		/tmp		tmpfs	defaults	0	0
sysfs		/sys		sysfs	defaults	0	0
```

#### 创建/etc/inittab 文件

inittab 的详细内容可以参考 busybox 下的文件 examples/inittab。init 程序会读取/etc/inittab

```shell
#etc/inittab
::sysinit:/etc/init.d/rcS #系统启动以后运行/etc/init.d/rcS 这个脚本文件
console::askfirst:-/bin/sh #将 console 作为控制台终端，也就是 ttymxc0
::restart:/sbin/init	#重启的话运行/sbin/init
::ctrlaltdel:/sbin/reboot#按下 ctrl+alt+del 组合键的话就运行/sbin/reboot
::shutdown:/bin/umount -a -r#关机的时候执行/bin/umount，也就是卸载各个文件系统
::shutdown:/sbin/swapoff -a#关机的时候执行/sbin/swapoff，也就是关闭交换分区
```

至此！根文件系统要创建的文件就已经全部完成了。

### 根文件系统其他功能测试

#### 开机自启动测试

/etc/init.d/rcS这个脚本中添加命令，例如需要开始直接开始hello world！可以下面这样写

```shell
.....
#开机自启动
cd /
./hello &
```

#### 外网连接测试

ping www.baidu.com会显示bad address，原因是地址解析失败了，并没有解析出其对应的 IP 地址。

在 rootfs 中新建文件/etc/resolv.conf，然后在里面输入如下内容：

```shell
nameserver 114.114.114.114
nameserver 192.168.5.1 #网关
```

重启开发板

## MfgTool烧写工具

MfgTool 工具是 NXP 提供的专门用于给 I.MX 系列 CPU 烧写系统的软件，此软件在 Windows 下使用。

解压压缩包，选择with-rootfs的压缩包。

EMMC版本只会用到 mfgtool2-yocto-mx-evk-emmc.vbs 这个烧写脚本

### 基本原理

先开发板烧系统分两部分：

1. 先向开发板的DDR下载一个Linux系统
2. 通过前面下载到DDR中的Linux系统完成最终的烧写工作

L4.1.15_2.0.0-ga_mfg-tools\mfgtools-with-rootfs\mfgtools\Profiles\Linux\OS Firmware下有：

files和firmware两个文件夹

files里面保存着最终烧写到开发板里面的uboot.imx、zImage、dtb和rootfs

firmware里面保存着第一步的uboot.imx、zImage、dtb。

### 烧写脚本就是各种.vbs文件

L4.1.15_2.0.0-ga_mfg-tools\mfgtools-with-rootfs\mfgtools下有很多.vbs文件

imx6ull的EMMC版本使用mfgtool2-yocto-mx-evk-emmc.vbs进行烧写官方系统

mfgtool2-yocto-mx-evk-emmc.vbs的内容

```
Set wshShell = CreateObject("WScript.shell")
wshShell.run "mfgtool2.exe -c ""linux"" -l ""eMMC"" -s ""board=sabresd"" -s ""mmc=1"" -s ""6uluboot=14x14evk"" -s ""6uldtb=14x14-evk"""
Set wshShell = Nothing
```

### ucl2.xml文件

负责在files和firmware里面选择合适的文件

### 烧写自己的系统

#### 系统烧写

可以改代码或者更改自己uboot和dtb的文件名

Uboot   u-boot-imx6ull14x14evk_emmc.imx

kernel   zImage

dtb        zImage-imx6ull-14x14-evk-emmc.dtb

rootfs    rootfs_nogpu.tar.bz2

对rootfs进行打包

```shell
cd rootfs/
tar -vcjf rootfs.tar.bz2 *
```

将4个文件都拷贝到files目录下，将除了根文件系统以外的三个拷贝到firmware目录下。（覆盖）

使用mfgtool2-yocto-mx-evk-emmc.vbs烧写

#### 网络开机自启动设置

不使用NFS挂载根文件系统，Linux就不会自动打开eth0网卡，所以需要ifconfig -a查看eth0和eth1是否存在。

ifconfig eth0 up 使能eth0网卡



如果你的开发板连接的路由器，那么可以通过路由器自动分配 IP 地址

udhcpc -i eth0 //通过路由器分配 IP 地址



如果你的开发板连接着电脑，那么就可以手动设置 IP 地址

```shell
ifconfig eth0 192.168.5.9 netmask 255.255.255.0 //设置 IP 地址和子网掩码 
route add default gw 192.168.5.1 //添加默认网关
```



可以对rcS文件进行修改，使得网络开机自启动。

```shell
#网络开机自启动设置
ifconfig eth0 up
#udhcpc -i eth0 #通过路由器分配 IP 地址
ifconfig eth0 192.168.5.9 netmask 255.255.255.0
route add default gw 192.168.5.1
```

### 改造自己的烧写工具

这个看正点原子视频

# 正点原子IMX6ULL驱动编程

## 调试时Uboot配置

调试时使用tftp下载镜像和设备树，使用nfs挂载根文件系统，uboot设置如下

```shell
#设置网络参数
setenv serverip 192.168.5.11
setenv ipaddr 192.168.5.9
setenv ethaddr b8:ae:1d:01:00:00
setenv gatewayip 192.168.5.1
setenv netmask 255.255.255.0

#bootcmd 从网络tftp启动
setenv bootcmd 'tftp 80800000 zImage;tftp 83000000 imx6ull-alientek-emmc.dtb;bootz 80800000 - 83000000;'
saveenv
#bootargs 设置根文件系统放在EMMC的分区2下
setenv bootargs 'console=ttymxc0,115200 rw nfsroot=192.168.5.11:/home/bcl/nfs/rootfs ip=192.168.5.9:192.168.5.11:192.168.5.1:255.255.255.0::eth0:off'
saveenv
```

## 字符设备驱动开发步骤

### 驱动模块的加载和卸载

Linux 驱动有两种运行方式，第一种就是将驱动编译进 Linux 内核中，这样当 Linux 内核启 动的时候就会自动运行驱动程序。第二种就是将驱动编译成模块(Linux 下模块扩展名为.ko)，在 Linux 内核启动以后使用“insmod”命令加载驱动模块。

```c
/* 驱动入口函数 */
static int __init chrdevbase_init(void)
{
    /* 入口函数具体内容 */
    return 0;
}

/* 驱动出口函数 */
static void __exit chrdevbase_exit(void)
{
    /* 出口函数具体内容 */
}

/**
 * 模块入口与出口
*/
module_init(chrdevbase_init);
module_exit(chrdevbase_exit);
```

#### 加载

insmod和modprobe都可以加载模块，区别在于insmod 命令不能解决模块的依赖关系，modprobe可以。推荐使用modprobe加载，默认路径为/lib/modules/\<kernel-version>。

自己制作的根文件系统不会有这个目录，需要手动创建。

注意一个新模块首次加载想使用modprobe，需要先输入depmod命令生成modules.dep文件。

modprobe xxx（不需要.ko后缀）

insmod xxx（需要.ko后缀）

#### 卸载

rmmod和modprobe -r都可以卸载模块，一般推荐使用rmmod。

### 字符设备注册与注销

对于字符设备驱动而言，当驱动模块加载成功以后需要注册字符设备，同样，卸载驱动模块的时候也需要注销掉字符设备。

位于linux/fs.h下

```c
static inline int register_chrdev(unsigned int major, const char *name,
const struct file_operations *fops)
static inline void unregister_chrdev(unsigned int major, const char *name)
```

“cat /proc/devices”可以查看当前已经被使用掉的设备号

major填0会自动分配主设备号

### 实现设备的具体操作函数

能够对 chrtest 进行打开和关闭操作

chrtest_open

chrtest_release

对 chrtest 进行读写操作

chrtest_read

chrtest_write

### 添加 LICENSE 和作者信息

最后我们需要在驱动中加入 LICENSE 信息和作者信息，其中 LICENSE 是必须添加的，否则的话编译的时候会报错，作者信息可以添加也可以不添加。LICENSE 和作者信息的添加使用

```c
MODULE_LICENSE("GPL");//添加模块 LICENSE 信息
MODULE_AUTHOR("bcl");//添加模块作者信息
```

[各种开源协议介绍 | 菜鸟教程 (runoob.com)](https://www.runoob.com/w3cnote/open-source-license.html)

## Linux 设备号

### 设备号的组成

设备号是一个无符号32位整型，32位数据分为主设备号和次设备号，其中高 12 位为主设备号，低 20 位为次设备号。所以主设备号为0~4095。使用cat /proc/devices查看主设备号或者使用ls -l /dev查看主次设备号。

### 设备号的分配

#### 静态分配设备号

有一些常用的设备号已经被 Linux 内核开发者给分配掉了，具体分配的内容可以查看文档 Documentation/devices.txt。具体能不能用还得看我们的硬件平台运行过程中有没有使用这个主设备号，选择一个没使用的即可。

#### 动态分配设备号

设备号的申请函数

int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)

设备号释放函数

void unregister_chrdev_region(dev_t from, unsigned count)

## chrdevbase 字符设备驱动开发实验

[linux驱动开发--copy_to_user 、copy_from_user函数实现内核空间数据与用户空间数据的相互访问-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/30152)

### 代码框架

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>

#define CHRDEVBASE_MAJOR    0               //主设备号
#define CHRDEVBASE_NAME     "chrdevbase"    //设备名

static int chrdevbase_open(struct inode *inode,struct file *filp)
{
    printk("chrdevbase_open\n");
    return 0;
}

static int chrdevbase_release(struct inode *inode,struct file *file)
{
    printk("chrdevbase_release\n");
    return 0;
}

static ssize_t chrdevbase_read(struct file *file, char __user *buf, size_t count, loff_t *ppos)
{
    printk("chrdevbase_read\n");
    return 0;
}

static ssize_t chrdevbase_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos)
{
    printk("chrdevbase_write\n");
    return 0;
}

struct file_operations chrdevbase_fops = {
    .owner = THIS_MODULE,
    .open = chrdevbase_open,
    .release = chrdevbase_release,
    .read = chrdevbase_read,
    .write = chrdevbase_write,
};

static int __init chrdevbase_init(void)
{
    int ret = 0;
    printk("chrdevbase_init\n");
    ret = register_chrdev(CHRDEVBASE_MAJOR,CHRDEVBASE_NAME,
                  &chrdevbase_fops);

    if(ret < 0){
        printk("chrdevbase init failed\n");
    }
    return 0;
}

static void __exit chrdevbase_exit(void)
{
    printk("chrdevbase_exit\n");
    unregister_chrdev(CHRDEVBASE_MAJOR,CHRDEVBASE_NAME);
}

/**
 * 模块入口与出口
*/
module_init(chrdevbase_init);
module_exit(chrdevbase_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("bcl");

```

### 编写和编译驱动程序

##### 编写驱动程序

驱动给应用传递数据的时候需要用到copy_to_user函数（read中），内核空间数据到用户空间的复制。

应用给驱动传递数据的时候需要用到copy_from_user函数（write中），用户空间数据到内核空间的复制。

**unsigned long copy_from_user(void \*to, const void \*from, unsigned long n);**
to:目标地址（内核空间）
from:源地址（用户空间）
n:将要拷贝数据的字节数
返回：成功返回0，失败返回没有拷贝成功的数据字节数

**unsigned long copy_to_user(void \*to, const void \*from, unsigned long n)**
to:目标地址（用户空间）
from:源地址（内核空间）
n:将要拷贝数据的字节数
返回：成功返回0，失败返回没有拷贝成功的数据字节数

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define CHRDEVBASE_MAJOR    200               //主设备号
#define CHRDEVBASE_NAME     "chrdevbase"    //设备名

static char readBuf[100];   /* 读缓冲 */
static char writeBuf[100];  /* 写缓冲 */
static char kernelData[] = {"kernel data!\n"};

static int chrdevbase_open(struct inode *inode,struct file *filp)
{
    printk("chrdevbase_open\n");
    return 0;
}

static int chrdevbase_release(struct inode *inode,struct file *file)
{
    printk("chrdevbase_release\n");
    return 0;
}

static ssize_t chrdevbase_read(struct file *file, char __user *buf, size_t count, loff_t *ppos)
{
    int ret = 0;

    printk("chrdevbase_read\n");
    memcpy(readBuf,kernelData,sizeof(kernelData));
    ret = copy_to_user(buf,readBuf,count);  //内核空间数据到用户空间的复制
    if(ret != 0){
        printk("copy_to_user failed\n");
    }
    return ret;
}

static ssize_t chrdevbase_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos)
{
    int ret = 0;

    printk("chrdevbase_write\n");
    ret = copy_from_user(writeBuf,buf,count);   //用户空间数据到内核空间的复制
    if(!ret){
        printk("kernel recevdata:%s\n",writeBuf);
    }else{
        printk("copy_from_user failed\n");
    }
    return ret;
}

struct file_operations chrdevbase_fops = {
    .owner = THIS_MODULE,
    .open = chrdevbase_open,
    .release = chrdevbase_release,
    .read = chrdevbase_read,
    .write = chrdevbase_write,
};

static int __init chrdevbase_init(void)
{
    int ret = 0;
    printk("chrdevbase_init\n");
    ret = register_chrdev(CHRDEVBASE_MAJOR,CHRDEVBASE_NAME,
                  &chrdevbase_fops);

    if(ret < 0){
        printk("chrdevbase init failed\n");
    }
    return 0;
}

static void __exit chrdevbase_exit(void)
{
    printk("chrdevbase_exit\n");
    unregister_chrdev(CHRDEVBASE_MAJOR,CHRDEVBASE_NAME);
}

/**
 * 模块入口与出口
*/
module_init(chrdevbase_init);
module_exit(chrdevbase_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("bcl");
```

##### 编译驱动程序

```makefile
KERNELDIR := /home/bcl/Linux_src/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)
obj-m := chrdevbase.o

build: kernel_modules

kernel_modules:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules
clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

KERNELDIR 表示开发板所使用的 Linux 内核源码目录

CURRENT_PATH 表示当前路径

obj-m 表示将 chrdevbase.c 这个文件编译为 chrdevbase.ko 模块

具体的编译命令，后面的 modules 表示编译模块，-C 表示将当前的工作目录切换到指定目录中，也就是 KERNERLDIR 目录。M 表示模块源码目录，“make modules”命令 中加入 M=dir 以后程序会自动到指定的 dir 目录中读取模块的源码并将其编译为.ko 文件。

使用make命令进行编译

### 编写和编译测试APP

##### 编写APP程序

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

/**
 * ./chrdevbaseAPP <filename> <1:2> 1表示读，2表示写
 * ./chrdevbaseAPP /dev/chrdevbase 1 表示从驱动里面读数据
 * ./chrdevbaseAPP /dev/chrdevbase 2 表示从驱动里面写数据
*/
int main(int argc,char *argv[])
{
    int fd;
    int ret;
    char *filename;
    char readBuf[100],writeBuf[100];
    static char usrdata[] = {"usr data!\n"};

    filename = argv[1];

    if(argc != 3){
        printf("Error usage!\n");
        return -1;
    }

    /* open */
    fd = open(filename,O_RDWR);
    if(fd < 0){
        perror("open");
        return -1;
    }

    /* read */
    if(atoi(argv[2]) == 1)
    {
        ret = read(fd,readBuf,100);
        if(ret < 0){
            perror("read");
            return -1;
        }
        else{
            printf("APP read data:%s",readBuf);
        }
    }

    /* write */
    if(atoi(argv[2]) == 2)
    {
        memcpy(writeBuf,usrdata,sizeof(usrdata));   //拷贝usrdata数据
        ret = write(fd,writeBuf,100);
        if(ret < 0){
            perror("write");
            return -1;
        }
    }

    /* close */
    ret = close(fd);
    if(ret < 0){
        perror("close");
        return -1;
    }

    return 0;
}
```

##### 编译APP程序

交叉编译 `arm-linux-gnueabihf-gcc chrdevbaseApp.c -o chrdevbaseApp`

### 测试

1. 装载驱动.ko `modprobe chrdevbase.ko`
2. 进入/dev查看设备文件，但是实际没有/dev/chrdevbase。我们需要创建设备节点 `mknod /dev/chrdevbase c 200 0`。“c”表示这是个 字符设备，“200”是设备的主设备号，“0”是设备的次设备号。
3. 测试 `./chrdevbaseAPP /dev/chrdevbaseApp`

实现效果就是

```shell
/lib/modules/4.1.15 # ./chrdevbaseAPP /dev/chrdevbase
chrdevbase_open
chrdevbase_read
chrdevbase_write
chrdevbase_release
```

```shell
/lib/modules/4.1.15 # ./chrdevbaseAPP /dev/chrdevbase 1
chrdevbase_open
chrdevbase_read
APP read data:kernel data!chrdevbase_release

/lib/modules/4.1.15 # ./chrdevbaseAPP /dev/chrdevbase 2
chrdevbase_open
chrdevbase_write
kernel recevdata:usr data!

chrdevbase_release
```

## LED驱动实验（直接操作寄存器）

### 地址映射

MMU 全称叫做 Memory  Manage Unit，也就是内存管理单元。

1. 完成虚拟空间到物理空间的映射。

2. 内存保护，设置存储器的访问权限，设置虚拟存储空间的缓冲特性。

开发板物理内存只有 512MB，映射到整个 4GB 的虚拟空间

**ioremap 函数**

定义 在 arch/arm/include/asm/io.h 文件中

```c
#define ioremap(cookie,size)		__arm_ioremap((cookie), (size), MT_DEVICE)
void __iomem *__arm_ioremap(phys_addr_t phys_addr, size_t size,
			    unsigned int mtype)
{
	return (void __iomem *)phys_addr;
}
```

**iounmap 函数**

```c
#define iounmap				__arm_iounmap
void __arm_iounmap(volatile void __iomem *addr)
{
}
```

### I/O内存访问函数

```c
//读操作函数，分别对应8bit、16bit 和 32bit
u8 readb(const volatile void __iomem *addr)
u16 readw(const volatile void __iomem *addr)
u32 readl(const volatile void __iomem *addr)
    
//写操作函数，分别对应8bit、16bit 和 32bit
void writeb(u8 value, volatile void __iomem *addr)
void writew(u16 value, volatile void __iomem *addr)
void writel(u32 value, volatile void __iomem *addr)
```

### 编写和编译驱动程序

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/io.h>

#define LED_MAJOR   200
#define LED_NAME    "led"

/* 寄存器物理地址 */
#define CCM_CCGR1_BASE              (0X020C406C)
#define SW_MUX_GPIO1_IO03_BASE      (0X020E0068)
#define SW_PAD_GPIO1_IO03_BASE      (0X020E02F4)
#define GPIO1_DR_BASE               (0X0209C000)
#define GPIO1_GDIR_BASE             (0X0209C004)

/* 虚拟地址 */
static void __iomem *IMX6ULL_CCM_CCGR1;
static void __iomem *SW_MUX_GPIO1_IO03;
static void __iomem *SW_PAD_GPIO1_IO03;
static void __iomem *GPIO1_DR;
static void __iomem *GPIO1_GDIR;

#define LEDOFF 0 /* 关闭 */
#define LEDON  1 /* 开启 */

static void led_switch(u8 sta)
{
    u32 val = 0;

    if(sta == LEDON){
        val = readl(GPIO1_DR);
        val &= ~(1 << 3);  //bit3置0，点亮LED
        writel(val,GPIO1_DR);  
    }else{
        val = readl(GPIO1_DR);
        val |= (1 << 3);  //bit3置1，熄灭LED
        writel(val,GPIO1_DR);  
    }
}

static int led_open(struct inode *inode, struct file *filp)
{
    return 0;
}

static int led_close(struct inode *inode, struct file *filp)
{
    return 0;
}

static ssize_t led_write(struct file *filp, const char __user *buf,
			 size_t count, loff_t *ppos)
{
    u32 ret;
    u8 dataBuf[1];

    ret = copy_from_user(dataBuf,buf,count);
    if(ret != 0){
        printk("kernel write failed\n");
        return -EFAULT;
    }

    //根据dataBuf判断开灯还是关灯
    led_switch(dataBuf[0]);
    
    return 0;
}

static const struct file_operations led_fops = {
    .owner   = THIS_MODULE,
    .open    = led_open,
    .release = led_close,
    .write   = led_write,
};

/* 入口 */
static int __init led_init(void)
{
    u32 ret = 0;
    u32 val = 0;

    /* 初始化LED灯、地址映射、32位是4个字节 */
    IMX6ULL_CCM_CCGR1 = ioremap(CCM_CCGR1_BASE,4);
    SW_MUX_GPIO1_IO03 = ioremap(SW_MUX_GPIO1_IO03_BASE,4);
    SW_PAD_GPIO1_IO03 = ioremap(SW_PAD_GPIO1_IO03_BASE,4);
    GPIO1_DR = ioremap(GPIO1_DR_BASE,4);
    GPIO1_GDIR = ioremap(GPIO1_GDIR_BASE,4);

    /* 初始化时钟 */
    val = readl(IMX6ULL_CCM_CCGR1);
    val &= ~(3 << 26);  //先清除bit26、27位
    val |= (3 << 27);   //bit26、27位置1
    writel(val,IMX6ULL_CCM_CCGR1);

    writel(0x5,SW_MUX_GPIO1_IO03);  //设置复用
    writel(0x10b0,SW_PAD_GPIO1_IO03); //设置电气属性

    val = readl(GPIO1_GDIR);
    val |= 1 << 3;  //bit3置1，设置为输出
    writel(val,GPIO1_GDIR);

    val = readl(GPIO1_DR);
    val &= ~(1 << 3);  //bit3置0，点亮LED
    writel(val,GPIO1_DR);

    /* 注册字符设备 */
    ret = register_chrdev(LED_MAJOR,LED_NAME,&led_fops);
    if(ret < 0)
    {
        printk("register_chrdev failed!\n");
        return -EIO;
    }

    printk("led_init\n");
    return 0;
}

/* 出口 */
static void __exit led_exit(void)
{
    /* 取消地址映射 */
    iounmap(IMX6ULL_CCM_CCGR1);
    iounmap(SW_MUX_GPIO1_IO03);
    iounmap(SW_PAD_GPIO1_IO03);
    iounmap(GPIO1_DR);
    iounmap(GPIO1_GDIR);

    /* 注销字符设备 */
    unregister_chrdev(LED_MAJOR,LED_NAME);
    printk("led_exit\n");
}

/* 注册驱动加载和卸载 */
module_init(led_init);
module_exit(led_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("bcl");
```

### 编写和编译应用程序

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

/**
 * ./ledAPP <filename> <0:1> 0表示关灯，1表示开灯
 * ./ledAPP /dev/ledAPP 0 关灯
 * ./ledAPP /dev/ledAPP 1 开灯
*/

#define LEDOFF 0 /* 关闭 */
#define LEDON  1 /* 开启 */

int main(int argc,char *argv[])
{
    int fd;
    int ret;
    int *ledStatus;
    char *filename;

    /* 参数数量检测 */
    if (argc != 3)
    {
        printf("Error usage\n");
        return -1;
    }

    filename = argv[1];

    /* 打开文件 */
    fd = open(filename,O_RDWR);
    if(fd < 0){
        perror("open");
        return -1;
    }

    /* 写入led状态 */
    *ledStatus = atoi(argv[2]);
    ret = write(fd,ledStatus,1);
    if(ret < 0){
        perror("write");
        close(fd);
        return -1;
    }

    /* 关闭文件 */
    ret = close(fd);
    if(ret){
        perror("read");
        return -1;
    }
    
    return 0;
}
```

## 新字符设备驱动实验

老的API为register_chrdev，驱动模块加载成功以后还需要手动使用 mknod 命令创建设备节点。

现在字符设备使用新的驱动API函数。

### 分配和释放设备号

```c
//没有指定设备号的话就使用如下函数来申请设备号
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)
    
//给定了设备的主设备号和次设备号就使用如下所示函数来注册设备号即可
//参数 from 是要申请的起始设备号，也就是给定的设备号；参数 count 是要申请的数量，一般都是一个；参数 name 是设备名字。
int register_chrdev_region(dev_t from, unsigned count, const char *name)

//统一使用如下释放函数
void unregister_chrdev_region(dev_t from, unsigned count)
```

```c
//示例代码
int major; /* 主设备号 */
int minor; /* 次设备号 */
dev_t devid; /* 设备号 */ 

if (major) { /* 定义了主设备号 */
	devid = MKDEV(major, 0); /* 大部分驱动次设备号都选择 0*/
	register_chrdev_region(devid, 1, "test");
} else { /* 没有定义设备号 */
	alloc_chrdev_region(&devid, 0, 1, "test"); /* 申请设备号 */
	major = MAJOR(devid); /* 获取分配号的主设备号 */
	minor = MINOR(devid); /* 获取分配号的次设备号 */
}
```

### **新的字符设备注册方法**

#### 字符设备结构

在 Linux 中使用 cdev 结构体表示一个字符设备，cdev 结构体在 include/linux/cdev.h 文件中

```c
struct cdev {
	struct kobject kobj;
	struct module *owner;
	const struct file_operations *ops;
	struct list_head list;
	dev_t dev;
	unsigned int count;
};
```

#### cdev_init 函数

void cdev_init(struct cdev *cdev, const struct file_operations *fops)

参数 cdev 就是要初始化的 cdev 结构体变量，参数 fops 就是字符设备文件操作函数集合。

#### cdev_add 函数

int cdev_add(struct cdev *p, dev_t dev, unsigned count)

参数 p 指向要添加的字符设备(cdev 结构体变量)，参数 dev 就是设备所使用的设备号，参数 count 是要添加的设备数量。

#### cdev_del 函数

void cdev_del(struct cdev *p)

参数 p 就是要删除的字符设备。

cdev_del 和 unregister_chrdev_region 这两个函数合起来的功能相当于 unregister_chrdev 函数。

### 自动创建设备节点

udev 是一个用户程序，在 Linux 下通过 udev 来实现设备文件的创建与删除，busybox 会创建一个 udev 的简化版本—mdev，所以在嵌入式 Linux 中我们使用mdev 来实现设备节点文件的自动创建与删除，Linux 系统中的热插拔事件也由 mdev 管理。

#### 创建和删除类

```c
//创建
#define class_create(owner, name)		\
({						\
	static struct lock_class_key __key;	\
	__class_create(owner, name, &__key);	\
})
struct class *__class_create(struct module *owner, const char *name,
			     struct lock_class_key *key)

//删除
void class_destroy(struct class *cls);
```

#### 创建设备

```c
struct device *device_create(struct class *cls, struct device *parent,
			     dev_t devt, void *drvdata,
			     const char *fmt, ...);

void device_destroy(struct class *class, dev_t devt)
```

device_create 是个可变参数函数，参数 class 就是设备要创建哪个类下面；参数 parent 是父设备，一般为 NULL，也就是没有父设备；参数 devt 是设备号；参数 drvdata 是设备可能会使用 的一些数据，一般为 NULL；参数 fmt 是设备名字，如果设置 fmt=xxx 的话，就会生成/dev/xxx 这个设备文件。返回值就是创建好的设备。

参数 class 是要删除的设备所处的类，参数 devt 是要删除的设备号。

### 设置文件私有数据

首先设备所有属性都设置为结构体，编写驱动 open 函数的时候将设备结构体作为私有数据添加到设备文件中。在 open 函数里面设置好私有数据以后，在 write、read、close 等函数中直接读取 private_data 即可得到设备结构体。

```c
struct test_dev testdev;

/* open 函数 */
static int test_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &testdev; /* 设置私有数据 */
	return 0;
}

static ssize_t newchrled_write(struct file *filp, const char __user *buf,
			 size_t count, loff_t *ppos)
{
    struct test_dev *dev = (struct test_dev)filp->private_data;
}
```

### goto的对错误的用法

goto在内核和驱动中一般用于对错误的处理。最先出错的地方，goto指向的处理语句放在最下面。以保证后续错误可以将前面所做的操作进行反向还原。

### 编写和编译驱动程序

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/cdev.h>
#include <linux/device.h>

#define NEWCHRLED_NAME "newchrled"

/* 寄存器物理地址 */
#define CCM_CCGR1_BASE              (0X020C406C)
#define SW_MUX_GPIO1_IO03_BASE      (0X020E0068)
#define SW_PAD_GPIO1_IO03_BASE      (0X020E02F4)
#define GPIO1_DR_BASE               (0X0209C000)
#define GPIO1_GDIR_BASE             (0X0209C004)

/* 虚拟地址 */
static void __iomem *IMX6ULL_CCM_CCGR1;
static void __iomem *SW_MUX_GPIO1_IO03;
static void __iomem *SW_PAD_GPIO1_IO03;
static void __iomem *GPIO1_DR;
static void __iomem *GPIO1_GDIR;

#define LEDOFF 0 /* 关闭 */
#define LEDON  1 /* 开启 */

/* LED设备结构体 */
struct newchrled_dev{
    struct cdev cdev;       /* 字符设备 */
    dev_t devid;            /* 设备号 */
    struct class *class;    /* 类 */
    struct device *device;  /* 设备 */
    int major;              /* 主设备号 */
    int minor;              /* 次设备号 */
};

struct newchrled_dev newchrled; /* led设备 */

static void led_switch(u8 sta)
{
    u32 val = 0;

    if(sta == LEDON){
        val = readl(GPIO1_DR);
        val &= ~(1 << 3);  //bit3置0，点亮LED
        writel(val,GPIO1_DR);  
    }else{
        val = readl(GPIO1_DR);
        val |= (1 << 3);  //bit3置1，熄灭LED
        writel(val,GPIO1_DR);  
    }
}

static int newchrled_open(struct inode *inode, struct file *filp)
{
    return 0;
}

static int newchrled_close(struct inode *inode, struct file *filp)
{
    return 0;
}

static ssize_t newchrled_write(struct file *filp, const char __user *buf,
			 size_t count, loff_t *ppos)
{
    u32 ret;
    u8 dataBuf[1];

    ret = copy_from_user(dataBuf,buf,count);
    if(ret != 0){
        printk("kernel write failed\n");
        return -EFAULT;
    }

    //根据dataBuf判断开灯还是关灯
    led_switch(dataBuf[0]);
    
    return 0;
}

/* 设备操作函数 */
static const struct file_operations newchrled_fops = {
    .owner   = THIS_MODULE,
    .open    = newchrled_open,
    .release = newchrled_close,
    .write   = newchrled_write,
};

/* 入口函数 */
static int __init newchrled_init(void)
{
    int ret = 0;
    u32 val = 0;

    printk("newchrled_init\n");

    /* 初始化LED灯、地址映射、32位是4个字节 */
    IMX6ULL_CCM_CCGR1 = ioremap(CCM_CCGR1_BASE,4);
    SW_MUX_GPIO1_IO03 = ioremap(SW_MUX_GPIO1_IO03_BASE,4);
    SW_PAD_GPIO1_IO03 = ioremap(SW_PAD_GPIO1_IO03_BASE,4);
    GPIO1_DR = ioremap(GPIO1_DR_BASE,4);
    GPIO1_GDIR = ioremap(GPIO1_GDIR_BASE,4);

    /* 初始化时钟 */
    val = readl(IMX6ULL_CCM_CCGR1);
    val &= ~(3 << 26);  //先清除bit26、27位
    val |= (3 << 27);   //bit26、27位置1
    writel(val,IMX6ULL_CCM_CCGR1);

    writel(0x5,SW_MUX_GPIO1_IO03);  //设置复用
    writel(0x10b0,SW_PAD_GPIO1_IO03); //设置电气属性

    val = readl(GPIO1_GDIR);
    val |= 1 << 3;  //bit3置1，设置为输出
    writel(val,GPIO1_GDIR);

    val = readl(GPIO1_DR);
    val &= ~(1 << 3);  //bit3置0，点亮LED
    writel(val,GPIO1_DR);

    /* 申请设备号 */
    if(newchrled.major)
    {
        newchrled.devid = MKDEV(newchrled.major,0);
        ret = register_chrdev_region(newchrled.devid,1,NEWCHRLED_NAME);
    }else{
        ret = alloc_chrdev_region(&newchrled.devid,0,1,NEWCHRLED_NAME);
        newchrled.major = MAJOR(newchrled.devid);
        newchrled.minor = MINOR(newchrled.devid);
    }

    if(ret < 0){
        printk("newchrled chrdev_region err!\n");
        goto fail_devid;
    }

    printk("newchrled major = %d,minor = %d\n",newchrled.major,newchrled.minor);
    
    /* 注册字符设备 */
    newchrled.cdev.owner = THIS_MODULE;
    cdev_init(&newchrled.cdev,&newchrled_fops);
    ret = cdev_add(&newchrled.cdev, newchrled.devid, 1);
    if(ret < 0)
        goto fail_cdev;

    /* 自动添加设备节点 */
    /* 添加类 */
    newchrled.class = class_create(THIS_MODULE,NEWCHRLED_NAME);
    if(IS_ERR(newchrled.class)){
        ret = PTR_ERR(newchrled.class);
        goto fail_class;
    }

    /* 添加设备 */
    newchrled.device =  device_create(newchrled.class, NULL,
			     newchrled.devid, NULL,
			     NEWCHRLED_NAME);
    if(IS_ERR(newchrled.device)){
        ret = PTR_ERR(newchrled.device);
        goto fail_device;
    }

    return 0;

fail_device:
    device_destroy(newchrled.class,newchrled.devid);
fail_class:
    class_destroy(newchrled.class);
fail_cdev:
    unregister_chrdev_region(newchrled.devid,1);
fail_devid:
    return ret;
}

/* 出口函数 */
static void __exit newchrled_exit(void)
{
    printk("newchrled_exit\n");

    /* 取消地址映射 */
    iounmap(IMX6ULL_CCM_CCGR1);
    iounmap(SW_MUX_GPIO1_IO03);
    iounmap(SW_PAD_GPIO1_IO03);
    iounmap(GPIO1_DR);
    iounmap(GPIO1_GDIR);

    /* 删除字符设备 */
    cdev_del(&newchrled.cdev);

    /* 注销设备号 */
    unregister_chrdev_region(newchrled.devid,1);

    /* 摧毁设备 */
    device_destroy(newchrled.class,newchrled.devid);

    /* 摧毁类 */
    class_destroy(newchrled.class);
}

/* 注册和卸载驱动 */
module_init(newchrled_init);
module_exit(newchrled_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("bcl");
```

## Linux设备树

### 什么是设备树？

[一文搞定 Linux 设备树 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/425420889)

设备树(Device Tree)，将这个词分开就是“设备”和“树”，描述设备树的文件叫做 DTS(Device  Tree Source)，这个 DTS 文件采用树形结构描述板级设备，也就是开发板上的设备信息。

### DTS、DTB 和 DTC

设备树源文件扩展名为.dts。DTS 是设备树源码文件，DTB 是将 DTS 编译以后得到的二进制文件。

DTC 工具将.dts 编译为.dtb。DTC 工具源码在 Linux 内核的 scripts/dtc 目录下

如果只是编译设备树的话建议使用“make dtbs”命令。

编译指定的dtb使用“make xxxx.dtb”命令。

### DTS 语法

[[DTS\]设备树语法_dts设备树语法_曼巴精神传承人的博客-CSDN博客](https://blog.csdn.net/u012041204/article/details/88382686)

[42.设备树---DTS的语法_dts语法_凌琳天上的博客-CSDN博客](https://blog.csdn.net/LingLinTianShang/article/details/122759842)

设备树都是小写字符。

**label: node-name@unit-address**

unit-address一般都是外设寄存器的起始地址，或者是类似I2C设备地址

label为节点标签，引入 label 的目的就是为了方便访问节点，可以直接通过&label 来访问这个节点

#### .dtsi **头文件**

和 C 语言一样，设备树也支持头文件，设备树的头文件扩展名为.dtsi。

```
#include <dt-bindings/input/input.h>
#include "imx6ull.dtsi"
```

“#include”来引用.h、.dtsi 和.dts 文件。.dtsi一般都是芯片共有的信息。

#### 设备节点

1. “/”是根节点，每个设备树文件只有一个根节点。

2. 从/根节点开始描述设备信息

3. 在/根节点外有一些&cpu0这样的语句是“追加”

### 设备树在系统中的体现

系统启动后在根文件系统中可以看到设备树的节点信息。在/proc/device-tree目录下。节点类似于文件夹。

### 标准属性

#### compatible 属性

"manufacturer,model"其中 manufacturer 表示厂商，model 一般是模块对应的驱动名字。

根节点下的compatible属性用于检查内核是否支持该芯片启动。

#### model 属性

一般 model 属性描述设备模块信息，比如名字什么的。

#### status 属性

“okay” 表明设备是可操作的。

“disabled”表明设备当前是不可操作的，但是在未来可以变为可操作的，比如热插拔设备插入以后。至于 disabled 的具体含义还要看设备的绑定文档。

#### \#address-cells 和#size-cells 属性

\#address-cells 和#size-cells 表明了子节点应该如何编写 reg 属性值

reg = <address1 length1 address2 length2 address3 length3……>

#### ranges 属性

ranges属性值可以为空或者按照(child-bus-address,parent-bus-address,length)格式编写的数字矩阵，ranges 是一个地址映射/转换表。

### 特殊节点

在根节点“/”中有两个特殊的子节点：aliases 和 chosen。

单词 aliases 的意思是“别名”，因此 aliases 节点的主要功能就是定义别名，定义别名的目 的就是为了方便访问节点。

chosen 并不是一个真实的设备，chosen 节点主要是为了 uboot 向 Linux 内核传递数据，重点是 bootargs 参数。/proc/device-tree/chosen 目录里面会有bootargs属性。

fdt_support.c 文件中有个 fdt_chosen 函数，uboot 中的 fdt_chosen 函数在设备树的 chosen 节点中加入了 bootargs 属性，并且还设置了 bootargs 属性值。

### 绑定信息文档

在 Linux 内核源码中有详细的.txt 文档描述了如何添加节点，这些.txt 文档叫做绑定文档，路径为：Linux 源码目录/Documentation/devicetree/bindings

### Linux内核的OF函数

Linux 内核给我们提供了一系列的函数来获取设备树中的节点或者属性信息，这一系列的函数都有一个统一的前缀“of_”，所以在很多资料里面也被叫做 OF 函数。这些 OF 函数原型都定义在 include/linux/of.h 文件中。

kmalloc( ) 和 kfree( ) 都在linux/slab.h中。

OF函数的使用

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/slab.h>

/*
	backlight {
		compatible = "pwm-backlight";
		pwms = <&pwm1 0 5000000>;
		brightness-levels = <0 4 8 16 32 64 128 255>;
		default-brightness-level = <6>;
		status = "okay";
	};
*/

/* 模块入口 */
static int __init disof_init(void)
{
    int ret = 0;
    struct device_node *bl_nd = NULL; /* 节点 */
    struct property *comppro = NULL;
    const char *str;
    u32 def_value = 0;
    u32 elemsize;
    u32 *brival;
    u8 i = 0;

    printk("disof_init\n");

    /* 找到backlight节点 路径是:/backlight */
    bl_nd = of_find_node_by_path("/backlight");
    if(bl_nd == NULL){
        ret = -EINVAL;
        goto fail_findnd;
    }

    /* 获取字符串属性 */
    comppro = of_find_property(bl_nd,"compatible",NULL);
    if(comppro == NULL){
        ret = -EINVAL;
        goto fail_findpro;
    }else{
        printk("compatible = %s\n",(char *)comppro->value);
    }

    ret = of_property_read_string(bl_nd,"status",&str);
    if(ret < 0){
        goto fail_rs;
    }else{
        printk("status = %s\n",str);
    }

    /* 获取数字属性 */
    ret = of_property_read_u32(bl_nd,"default-brightness-level",&def_value);
    if(ret < 0){
        goto fail_read32;
    }else{
        printk("default-brightness-level = %d\n",def_value);
    }

    /* 获取数组属性 */
    elemsize = of_property_count_elems_of_size(bl_nd,"brightness-levels",sizeof(u32));
    if(elemsize < 0){
        ret = -EINVAL;
        goto fail_readele;
    }else{
        printk("brightness-levels elems size = %d\n",elemsize);
    }

    brival = kmalloc(elemsize*sizeof(u32),GFP_KERNEL);
    if(!brival){
        ret = -EINVAL;
        goto fail_mem;
    }

    ret = of_property_read_u32_array(bl_nd,"brightness-levels",brival,elemsize);
    if(ret < 0){
        goto fail_read32array;
    }else{
        for(i = 0;i < elemsize;i++)
            printk("brightness-levels[%d] = %d\n",i,*(brival+i));
    }
    kfree(brival);

    return 0;

fail_read32array:
    kfree(brival);
fail_mem:
fail_readele:
fail_read32:
fail_rs:
fail_findpro:
fail_findnd:
    return ret;
}

/* 模块出口 */
static void __exit disof_exit(void)
{
    printk("disof_exit\n");
    return;
}

/* 模块入口和出口 */
module_init(disof_init);
module_exit(disof_exit);

/* 注册模块入口和出口 */
MODULE_LICENSE("GPL");
MODULE_AUTHOR("bcl");
```

## 设备树下的 LED 驱动实验

①、在 imx6ull-alientek-emmc.dts 文件中创建相应的设备节点。

②、编写驱动程序(在第四十二章实验基础上完成)，获取设备树中的相关属性值。

③、使用获取到的有关属性值来初始化 LED 所使用的 GPIO。

### 修改设备树文件

```
/* alphaled */
alphaled {
	#address-cells = <1>;
	#size-cells = <1>;
	compatible = "atkalpha-led";
	status = "okay";
	reg = < 0X020C406C 0X04	/* CCM_CCGR1_BASE */
			0X020E0068 0X04	/* SW_MUX_GPIO1_IO03_BASE */
			0X020E02F4 0X04	/* SW_PAD_GPIO1_IO03_BASE */
			0X0209C000 0X04	/* GPIO1_DR_BASE */
			0X0209C004 0X04	/* GPIO1_GDIR_BASE */
	>;
};
```

修改设备树后，如果make时出现FATAL: section header offset=11259024840327220 in file 'vmlinux' is bigger than filesize=14007747，则需要重新编译内核./imx6ull_alientek_emmc.sh。

### 编写驱动程序

of_iomap函数位于linux/of_address.h中。

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/of_address.h>

#define DTSLED_CNT  1           /* 设备号个数 */
#define DTSLED_NAME "dtsled"    /* 设备名字 */

/* 虚拟地址 */
static void __iomem *IMX6ULL_CCM_CCGR1;
static void __iomem *SW_MUX_GPIO1_IO03;
static void __iomem *SW_PAD_GPIO1_IO03;
static void __iomem *GPIO1_DR;
static void __iomem *GPIO1_GDIR;

#define LEDOFF 0 /* 关闭 */
#define LEDON  1 /* 开启 */

/* led设备结构体 */
struct dtsled_dev{
    dev_t devid;            /* 设备号 */
    int major;              /* 主设备号 */
    int minor;              /* 次设备号 */
    struct cdev cdev;       /* 设备结构体 */
    struct class *class;    /* 类 */
    struct device *device;  /* 设备 */
    struct device_node *nd; /* 设备节点 */
};

struct dtsled_dev dtsled;   /* led设备 */

static void led_switch(u8 sta)
{
    u32 val = 0;

    if(sta == LEDON){
        val = readl(GPIO1_DR);
        val &= ~(1 << 3);  //bit3置0，点亮LED
        writel(val,GPIO1_DR);  
    }else{
        val = readl(GPIO1_DR);
        val |= (1 << 3);  //bit3置1，熄灭LED
        writel(val,GPIO1_DR);  
    }
}

static int dtsled_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &dtsled;   //设置私有属性
    return 0;
}

static int dtsled_close(struct inode *inode, struct file *filp)
{
    return 0;
}

static ssize_t dtsled_write(struct file *filp, const char __user *buf,
			 size_t count, loff_t *ppos)
{
    u32 ret;
    u8 dataBuf[1];

    //struct dtsled_dev *dev = (struct dtsled_dev*)filp->private_data;

    ret = copy_from_user(dataBuf,buf,count);
    if(ret != 0){
        printk("kernel write failed\n");
        return -EFAULT;
    }

    //根据dataBuf判断开灯还是关灯
    led_switch(dataBuf[0]);
    
    return 0;
}

/* 设备操作函数 */
static const struct file_operations dtsled_fops = {
    .owner   = THIS_MODULE,
    .open    = dtsled_open,
    .release = dtsled_close,
    .write   = dtsled_write,
};

/* 入口 */
static int __init dtsled_init(void)
{
    int ret = 0;
    struct property *proper;
    const char *str;
    u32 regdata[14];
    u32 val;

    /* 注册字符设备 */
    /* 1.申请设备号 */
    dtsled.major = 0;   //由内核分配
    if(dtsled.major){   //给定设备号
        dtsled.devid = MKDEV(dtsled.major,0);
        ret = register_chrdev_region(dtsled.devid,DTSLED_CNT,DTSLED_NAME);
    }else{  //没有给定设备号
        ret = alloc_chrdev_region(&dtsled.devid,0,DTSLED_CNT,DTSLED_NAME);
        dtsled.major = MAJOR(dtsled.devid);
        dtsled.minor = MINOR(dtsled.devid);
    }
    printk("dtsled major = %d,minor = %d\n",dtsled.major,dtsled.minor);
    if(ret < 0){
        goto fail_devid;
    }

    /* 2.添加字符设备 */
    dtsled.cdev.owner = THIS_MODULE;
    cdev_init(&dtsled.cdev,&dtsled_fops);
    ret = cdev_add(&dtsled.cdev,dtsled.devid,DTSLED_CNT);
    if(ret < 0){
        goto fail_cdev;
    }

    /* 自动添加设备节点 */
    /* 1.添加类 */
    dtsled.class = class_create(THIS_MODULE,DTSLED_NAME);
    if(IS_ERR(dtsled.class)){
        ret = PTR_ERR(dtsled.class);
        goto fail_class;
    }
    /* 2.添加设备 */
    dtsled.device = device_create(dtsled.class,NULL,dtsled.devid,NULL,DTSLED_NAME);
    if(IS_ERR(dtsled.device)){
        ret = PTR_ERR(dtsled.device);
        goto fail_device;
    }

    /* 获取设备树内容 */
    /* 1.获取设备节点 */
    dtsled.nd = of_find_node_by_path("/alphaled");
    if(dtsled.nd == NULL){
        ret = -EINVAL;
        goto fail_findnd;
    }

    /* 2.获取compatible属性 */
    proper = of_find_property(dtsled.nd,"compatible",NULL);
    if(proper == NULL){
        printk("compatible property find failed\r\n");
        goto fail_rs;
    }else{
        printk("compatible = %s\n",(char *)proper->value);
    }

    /* 3.获取status属性 */
    ret = of_property_read_string(dtsled.nd,"status",&str);
    if(ret < 0){
        printk("status read failed\n");
        goto fail_rs;
    }else{
        printk("status = %s\n",str);
    }

    /* 4.获取reg属性 */
    ret = of_property_read_u32_array(dtsled.nd,"reg",regdata,10);
    if(ret < 0){
        printk("reg property read failed\n");
        goto fail_rs;
    }else{
        u8 i = 0;
        printk("reg data:\n");
        for(i = 0;i < 10;i++)
            printk("%#X ",regdata[i]);
        printk("\n");
    }

    /* 初始化LED灯、地址映射、32位是4个字节 */
#if 0
    IMX6ULL_CCM_CCGR1 = ioremap(regdata[0],regdata[1]);
    SW_MUX_GPIO1_IO03 = ioremap(regdata[2],regdata[3]);
    SW_PAD_GPIO1_IO03 = ioremap(regdata[4],regdata[5]);
    GPIO1_DR = ioremap(regdata[6],regdata[7]);
    GPIO1_GDIR = ioremap(regdata[8],regdata[9]);
#else
    IMX6ULL_CCM_CCGR1 = of_iomap(dtsled.nd,0);
    SW_MUX_GPIO1_IO03 = of_iomap(dtsled.nd,1);
    SW_PAD_GPIO1_IO03 = of_iomap(dtsled.nd,2);
    GPIO1_DR = of_iomap(dtsled.nd,3);
    GPIO1_GDIR = of_iomap(dtsled.nd,4);
#endif

    /* 使能 GPIO1 时钟 */
    val = readl(IMX6ULL_CCM_CCGR1);
    val &= ~(3 << 26);  //先清除bit26、27位
    val |= (3 << 27);   //bit26、27位置1
    writel(val,IMX6ULL_CCM_CCGR1);

    /* 设置 GPIO1_IO03 的复用功能，最后设置 IO 属性 */
    writel(0x5,SW_MUX_GPIO1_IO03);  //设置复用
    writel(0x10b0,SW_PAD_GPIO1_IO03); //设置电气属性

    /* 设置 GPIO1_IO03 为输出功能 */
    val = readl(GPIO1_GDIR);
    val |= 1 << 3;  //bit3置1，设置为输出
    writel(val,GPIO1_GDIR);

    /* 默认关闭 LED */
    val = readl(GPIO1_DR);
    val |= (1 << 3);  //bit3置1，默认熄灭LED
    writel(val,GPIO1_DR);

    return 0;

fail_rs:

fail_findnd:
    device_destroy(dtsled.class,dtsled.devid);  //摧毁设备
fail_device:
    class_destroy(dtsled.class);    //摧毁类
fail_class:
    cdev_del(&dtsled.cdev); //删除cdev
fail_cdev:
    unregister_chrdev_region(dtsled.devid,DTSLED_CNT);
fail_devid:
    return ret;
}

/* 出口 */
static void __exit dtsled_exit(void)
{
    /* 关闭LED */
    led_switch(LEDOFF);

    /* 取消地址映射 */
    iounmap(IMX6ULL_CCM_CCGR1);
    iounmap(SW_MUX_GPIO1_IO03);
    iounmap(SW_PAD_GPIO1_IO03);
    iounmap(GPIO1_DR);
    iounmap(GPIO1_GDIR);

    /* 注销字符设备驱动 */
    cdev_del(&dtsled.cdev); //删除cdev
    unregister_chrdev_region(dtsled.devid,DTSLED_CNT);  //注销设备号

    device_destroy(dtsled.class,dtsled.devid);  //摧毁设备
    class_destroy(dtsled.class);    //摧毁类

    return;
}

/* 注册驱动和卸载驱动 */
module_init(dtsled_init);
module_exit(dtsled_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("bcl");
```

## pinctrl 和 gpio 子系统实验

Linux 内核针对 PIN 的配置推出了 pinctrl 子系统，对于 GPIO 的配置推出了 gpio 子系统。借助 pinctrl 和 gpio 子系统来简化 GPIO 驱动开发。

### pinctrl 子系统

①、获取设备树中 pin 信息。

②、根据获取到的 pin 信息来设置 pin 的复用功能。

③、根据获取到的 pin 信息来设置 pin 的电气特性，比如上/下拉、速度、驱动能力等。

#### PIN 配置信息详解

在imx6ull.dtsi中，有下面这个节点，用于pin的复用。

```
iomuxc: iomuxc@020e0000 {
				compatible = "fsl,imx6ul-iomuxc";
				reg = <0x020e0000 0x4000>;
			};
iomuxc_snvs: iomuxc-snvs@02290000 {
				compatible = "fsl,imx6ull-iomuxc-snvs";
				reg = <0x02290000 0x10000>;
			};
gpr: iomuxc-gpr@020e4000 {
				compatible = "fsl,imx6ul-iomuxc-gpr",
					"fsl,imx6q-iomuxc-gpr", "syscon";
				reg = <0x020e4000 0x4000>;
			};
```

根据设备的类型，创建对应的子节点，然后设备所使用的PIN都放在此节点。

举例：

```
pinctrl_hog_1: hoggrp-1 {
			fsl,pins = <
				MX6UL_PAD_UART1_RTS_B__GPIO1_IO19	0x17059 /* SD1 CD */
				MX6UL_PAD_GPIO1_IO05__USDHC1_VSELECT	0x17059 /* SD1 VSELECT */
				MX6UL_PAD_GPIO1_IO09__GPIO1_IO09        0x17059 /* SD1 RESET */
			>;
		};
```

fsl名字的宏定义位于arch/arm/boot/dts/imx6ul-pinfunc.h中，imx6ull.dtsi会引用imx6ull-pinfunc.h，然后又会引用imx6ul-pinfunc.h

**MX6UL_PAD_UART1_RTS_B__GPIO1_IO19**表示将 UART1_RTS_B 这个 IO 复用为 GPIO1_IO19。

此宏定义后面跟着 5 个数字，mux_reg 寄存器偏移地址、conf_reg 寄存器偏移地址、input_reg 寄存器偏移地址、mux_reg 寄存器值、input_reg 寄存器值

0x0090         0x031C     0x0000             0x5              0x0

<mux_reg   conf_reg   input_reg    mux_mode   input_val>

**mux_reg：**

iomuxc 节点首地址0x020e0000，因此UART1_RTS_B这个PIN的mux寄存器地址就是：0x020e0000+0x0090

**conf_reg：**

0x020e0000+0x031c=0x020e031c，这个寄存器就是UART_RTS_B的电气属性配置寄存器

**input_reg：**

UART1_RTS_B 这个 PIN 在做 GPIO1_IO19 的时候是没有 input_reg 寄存器

**mux_mode：**

设置 UART1_RTS_B 这 个 PIN 复用为 GPIO1_IO19。

**input_val：**

写给input_reg的值，没有 input_reg 寄存器，无效

**0x17059**为PIN的电气属性配置寄存器的值。

#### PIN 驱动程序讲解

Linux根据compatible 属性的值“fsl,imx6ul-iomuxc”来匹配驱动，在drivers/pinctrl/freescale/pinctrl-imx6ul.c中有以下内容

```c
static struct of_device_id imx6ul_pinctrl_of_match[] = {
	{ .compatible = "fsl,imx6ul-iomuxc", .data = &imx6ul_pinctrl_info, },
	{ .compatible = "fsl,imx6ull-iomuxc-snvs", .data = &imx6ull_snvs_pinctrl_info, },
	{ /* sentinel */ }
};

static int imx6ul_pinctrl_probe(struct platform_device *pdev)
{
	const struct of_device_id *match;
	struct imx_pinctrl_soc_info *pinctrl_info;

	match = of_match_device(imx6ul_pinctrl_of_match, &pdev->dev);

	if (!match)
		return -ENODEV;

	pinctrl_info = (struct imx_pinctrl_soc_info *) match->data;

	return imx_pinctrl_probe(pdev, pinctrl_info);
}

static struct platform_driver imx6ul_pinctrl_driver = {
	.driver = {
		.name = "imx6ul-pinctrl",
		.owner = THIS_MODULE,
		.of_match_table = of_match_ptr(imx6ul_pinctrl_of_match),
	},
	.probe = imx6ul_pinctrl_probe,
	.remove = imx_pinctrl_remove,
};
```

####  设备树中添加 pinctrl 节点模板

1、创建对应的节点

```
pinctrl_test: testgrp {
}
```

2、添加“fsl,pins”属性 

```
pinctrl_test: testgrp {
	fsl,pins = <
	/* 设备所使用的 PIN 配置信息 */
	>;
};
```

3、在“fsl,pins”属性中添加 PIN 配置信息

```
pinctrl_test: testgrp {
	fsl,pins = <
	MX6UL_PAD_GPIO1_IO00__GPIO1_IO00 config /*config 是具体设置值*/
	>;
};
```

###  gpio 子系统

gpio 子系统顾名思义，就是用于初始化 GPIO 并且提供相应的 API 函数，比如设置 GPIO 为输入输出，读取 GPIO 的值等。gpio 子系统的主要目的就是方便驱动开发者使用 gpio，驱动开发者在设备树中添加 gpio 相关信息，然后就可以在驱动程序中使用 gpio 子系统提供的 API 函数来操作 GPIO

#### 设备树中的 gpio 信息

```
&usdhc1 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc1>;
	pinctrl-1 = <&pinctrl_usdhc1_100mhz>;
	pinctrl-2 = <&pinctrl_usdhc1_200mhz>;
	cd-gpios = <&gpio1 19 GPIO_ACTIVE_LOW>;
	keep-power-in-suspend;
	enable-sdio-wakeup;
	vmmc-supply = <&reg_sd1_vmmc>;
	status = "okay";
};
```

cd-gpios = <&gpio1 19 GPIO_ACTIVE_LOW>;

属性“cd-gpios”描述了 SD 卡的 CD 引脚使用的哪个 IO。属性值一共有三个， 我们来看一下这三个属性值的含义，“&gpio1”表示 CD 引脚所使用的 IO 属于 GPIO1 组，“19” 表示 GPIO1 组的第 19 号 IO，通过这两个值 SD 卡驱动程序就知道 CD 引脚使用了 GPIO1_IO19 这 GPIO。如果为 0(GPIO_ACTIVE_HIGH) 的话表示高电平有效 ， 如果为 1(GPIO_ACTIVE_LOW)的话表示低电平有效。

关 于 I.MX 系 列 SOC 的 GPIO 控 制 器 绑 定 信 息 请 查 看 文 档 Documentation/devicetree/bindings/gpio/ fsl-imx-gpio.txt。

#### GPIO驱动程序讲解

Linux根据compatible 属性的值“fsl,imx6ul-gpio”，“fsl,imx35-gpio”来匹配驱动，查找 GPIO 驱动文件，drivers/gpio/gpio-mxc.c 就是 I.MX6ULL 的 GPIO 驱动文件

#### 设备树中添加 gpio 节点模板

1、创建 test 设备节点

```
test {
	/* 节点内容 */
};
```

2、添加 pinctrl 信息

```
test {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_test>;
	/* 其他节点内容 */
};
```

3、添加 GPIO 属性信息

```
test {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_test>;
	gpio = <&gpio1 0 GPIO_ACTIVE_LOW>;
}
```

#### GPIO操作步骤

驱动中对GPIO的操作函数

1. 获取GPIO所处的设备节点，比如of_find_node_by_path
2. 获取GPIO编号，of_get_named_gpio
3. 申请 GPIO 管脚，gpio_request
4. 设置GPIO为输入还是输出，gpio_direction_input或gpio_direction_output
5. 读取GPIO值或者设置GPIO值，gpio_get_value或gpio_set_value
6. 释放GPIO，gpio_free

### 驱动代码编写

#### 修改设备树文件

1、添加 pinctrl 节点

在 iomuxc 节点的 imx6ul-evk 子节点下创建一个名为“pinctrl_led”的子节点

```
pinctrl_led: ledgrp {
	fsl,pins = <
		MX6UL_PAD_GPIO1_IO03__GPIO1_IO03	0x10b0	/* LED0 */
	>;
};
```

2、添加 LED 设备节点

在根节点“/”下创建 LED 灯节点，节点名为“gpioled”

```
/* gpioled */
gpioled {	
	#address-cells = <1>;
	#size-cells = <1>;
	compatible = "atkalpha-gpioled";
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_led>;
	led-gpio = <&gpio1 3 GPIO_ACTIVE_LOW>;
	status = "okay";
};
```

编写驱动程序的时候会获取 led-gpio 属性的内容来得到 GPIO 编号

3、检查 PIN 是否被其他外设使用

半导体厂商提供的设备树是根据自己官方开发板编写，所以存在已经将IO使用掉的情况，需要检查是否被使用。

比如本次实验中GPIO_IO03被TSC电阻触摸屏接口使用，需要屏蔽相关代码。

在imx6ull-alientek-emmc.dts 中搜索“gpio1 3”。

#### LED驱动编写

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_gpio.h>

#define GPIOLED_CNT     1           /* 设备号个数 */
#define GPIOLED_NAME    "gpioled"   /* 名字 */
#define LEDOFF          0           /* 关灯 */
#define LEDON           1           /* 开灯 */

/* led设备结构体 */
struct gpioled_dev{
    dev_t devid;                /* 设备号 */
    int major;                  /* 主设备号 */
    int minor;                  /* 次设备号 */
    struct cdev cdev;           /* cdev */
    struct class *class;        /* 类 */
    struct device *device;      /* 设备 */
    struct device_node *nd;     /* 设备节点 */
    int led_gpio;               /* led使用的GPIO编号 */
};

struct gpioled_dev gpioled; /* led设备 */

static int gpioled_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &gpioled;   //设置私有属性
    return 0;
}

static int gpioled_close(struct inode *inode, struct file *filp)
{
    return 0;
}

static ssize_t gpioled_write(struct file *filp, const char __user *buf,
			 size_t count, loff_t *ppos)
{
    u32 ret;
    u8 dataBuf[1];
    struct gpioled_dev *dev = (struct gpioled_dev *)filp->private_data;

    //struct gpioled_dev *dev = (struct gpioled_dev*)filp->private_data;

    ret = copy_from_user(dataBuf,buf,count);
    if(ret != 0){
        printk("kernel write failed\n");
        return -EFAULT;
    }

    //根据dataBuf判断开灯还是关灯
    if(dataBuf[0] == LEDON){
        gpio_set_value(dev->led_gpio,0);
    }else{
        gpio_set_value(dev->led_gpio,1);
    }

    return 0;
}

/* 设备操作函数 */
static const struct file_operations gpioled_fops = {
    .owner   = THIS_MODULE,
    .open    = gpioled_open,
    .release = gpioled_close,
    .write   = gpioled_write,
};

/* 驱动入口函数 */
static int __init led_init(void)
{
    int ret;

    /* 申请设备号 */
    gpioled.major = 0;  //由内核分配
    if(gpioled.major){  //给定设备号
        gpioled.devid = MKDEV(gpioled.major,0);
        ret = register_chrdev_region(gpioled.devid,GPIOLED_CNT,GPIOLED_NAME);
    }else{
        ret = alloc_chrdev_region(&gpioled.devid,0,GPIOLED_CNT,GPIOLED_NAME);
        gpioled.major = MAJOR(gpioled.devid);
        gpioled.minor = MINOR(gpioled.devid);
    }
    printk("gpioled major = %d,minor = %d\n",gpioled.major,gpioled.minor);
    if(ret < 0){
        goto fail_devid;
    }

    /* 2.添加字符设备 */
    gpioled.cdev.owner = THIS_MODULE;
    cdev_init(&gpioled.cdev,&gpioled_fops);
    ret = cdev_add(&gpioled.cdev,gpioled.devid,GPIOLED_CNT);
    if(ret < 0){
        goto fail_cdev;
    }

    /* 自动添加设备节点 */
    /* 1.添加类 */
    gpioled.class = class_create(THIS_MODULE,GPIOLED_NAME);
    if(IS_ERR(gpioled.class)){
        ret = PTR_ERR(gpioled.class);
        goto fail_class;
    }

    /* 2.添加设备 */
    gpioled.device = device_create(gpioled.class,NULL,gpioled.devid,NULL,GPIOLED_NAME);
    if(IS_ERR(gpioled.device)){
        ret = PTR_ERR(gpioled.device);
        goto fail_device;
    }

    /* 设置LED所使用的GPIO */
    /* 1.获取设备节点：gpioled */
    gpioled.nd = of_find_node_by_path("/gpioled");  //设备节点名
    if(gpioled.nd == NULL){
        printk("gpioled node can't not found\n");
        ret = -EINVAL;
        goto fail_node;
    }else{
        printk("gpioled node has been found\n");
    }

    /* 2.获取设备树中的gpio属性，得到LED的编号 */
    gpioled.led_gpio = of_get_named_gpio(gpioled.nd,"led-gpio",0);//根据设备树实际命名更改
    if(gpioled.led_gpio < 0){
        printk("can't get led-gpio\n");
        ret = -EINVAL;
        goto fail_node;
    }
    printk("led-gpio num = %d\n",gpioled.led_gpio);

    /* 3.请求gpio */
    ret = gpio_request(gpioled.led_gpio,"led-gpio");
    if(ret){
        printk("can't request led-gpio\n");
        goto fail_node;
    }

    /* 4.设置 GPIO1_IO03 为输出，并且输出高电平，默认关闭 LED 灯 */
    ret = gpio_direction_output(gpioled.led_gpio,1);
    if(ret < 0){
        printk("can't set gpio\n");
        goto fail_setoutput;
    }

    return 0;

fail_setoutput:
    gpio_free(gpioled.led_gpio);
fail_node:
    device_destroy(gpioled.class,gpioled.devid);
fail_device:
    class_destroy(gpioled.class);
fail_class:
    cdev_del(&gpioled.cdev);
fail_cdev:
    unregister_chrdev_region(gpioled.devid,GPIOLED_CNT);
fail_devid:
    return ret;
}

/* 驱动出口函数 */
static void __exit led_exit(void)
{
    /* 关闭LED */
    gpio_set_value(gpioled.led_gpio,LEDOFF);
    gpio_free(gpioled.led_gpio);

    /* 注销字符设备驱动 */
    cdev_del(&gpioled.cdev);    //删除cdev
    unregister_chrdev_region(gpioled.devid,GPIOLED_CNT);    //注销设备号


    device_destroy(gpioled.class,gpioled.devid);    //摧毁设备
    class_destroy(gpioled.class);   //摧毁类

    return;
}

/* 注册驱动和卸载驱动 */
module_init(led_init);
module_exit(led_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("bcl");
```

## 蜂鸣器实验

蜂鸣器的驱动与LED类似，所以只需按照上面使用pinctrl和GPIO子系统写的代码稍作修改即可。

正点原子ALPHA开发板Beep的IO为SNVS_TAMPER1，作为GPIO5_IO01。

注意！在设备树中，6UL和6ULL的SNVS组的IO地址是不同的！注意区分！重点关注imx6ull-pinfunc-snvs.h。

### 修改设备树文件

1. 添加pinctrl节点

    ```
     pinctrl_beep: beepgrp {
    	fsl,pins = <
    		MX6ULL_PAD_SNVS_TAMPER1__GPIO5_IO01 0x10B0 /* Beep */ 
    	>;
    };
    ```

2. 添加 BEEP 设备节点

    ```
    beep {
    	#address-cells = <1>;
    	#size-cells = <1>;
    	compatible = "atkalpha-beep";
    	pinctrl-names = "default";
    	pinctrl-0 = <&pinctrl_beep>;
    	beep-gpio = <&gpio5 1 GPIO_ACTIVE_HIGH>;
    	status = "okay";
    };
    ```

3. 检查 PIN 是否被其他外设使用

在设备树中全局搜索SNVS_TAMPER1查看有无其他地方使用了这个IO，如果有就需要进行屏蔽。

### Beep驱动代码编写

主要是从led驱动改过来的，稍作修改。

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_gpio.h>

#define BEEP_CNT     1              /* 设备号个数 */
#define BEEP_NAME    "beep"         /* 名字 */
#define BEEPOFF      0              /* 关蜂鸣器 */
#define BEEPON       1              /* 开蜂鸣器 */

/* beep设备结构体 */
struct beep_dev{
    dev_t devid;                /* 设备号 */
    int major;                  /* 主设备号 */
    int minor;                  /* 次设备号 */
    struct cdev cdev;           /* cdev */
    struct class *class;        /* 类 */
    struct device *device;      /* 设备 */
    struct device_node *nd;     /* 设备节点 */
    int beep_gpio;               /* beep使用的GPIO编号 */
};

struct beep_dev beep; /* beep设备 */

static int beep_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &beep;   //设置私有属性
    return 0;
}

static int beep_close(struct inode *inode, struct file *filp)
{
    return 0;
}

static ssize_t beep_write(struct file *filp, const char __user *buf,
			 size_t count, loff_t *ppos)
{
    u32 ret;
    u8 dataBuf[1];
    struct beep_dev *dev = (struct beep_dev *)filp->private_data;

    //struct beep_dev *dev = (struct beep_dev*)filp->private_data;

    ret = copy_from_user(dataBuf,buf,count);
    if(ret != 0){
        printk("kernel write faibeep\n");
        return -EFAULT;
    }

    //根据dataBuf判断开蜂鸣器还是关蜂鸣器
    if(dataBuf[0] == BEEPON){
        gpio_set_value(dev->beep_gpio,0);
    }else{
        gpio_set_value(dev->beep_gpio,1);
    }

    return 0;
}

/* 设备操作函数 */
static const struct file_operations beep_fops = {
    .owner   = THIS_MODULE,
    .open    = beep_open,
    .release = beep_close,
    .write   = beep_write,
};

/* 驱动入口函数 */
static int __init beep_init(void)
{
    int ret;

    /* 申请设备号 */
    beep.major = 0;  //由内核分配
    if(beep.major){  //给定设备号
        beep.devid = MKDEV(beep.major,0);
        ret = register_chrdev_region(beep.devid,BEEP_CNT,BEEP_NAME);
    }else{
        ret = alloc_chrdev_region(&beep.devid,0,BEEP_CNT,BEEP_NAME);
        beep.major = MAJOR(beep.devid);
        beep.minor = MINOR(beep.devid);
    }
    printk("beep major = %d,minor = %d\n",beep.major,beep.minor);
    if(ret < 0){
        goto fail_devid;
    }

    /* 2.添加字符设备 */
    beep.cdev.owner = THIS_MODULE;
    cdev_init(&beep.cdev,&beep_fops);
    ret = cdev_add(&beep.cdev,beep.devid,BEEP_CNT);
    if(ret < 0){
        goto fail_cdev;
    }

    /* 自动添加设备节点 */
    /* 1.添加类 */
    beep.class = class_create(THIS_MODULE,BEEP_NAME);
    if(IS_ERR(beep.class)){
        ret = PTR_ERR(beep.class);
        goto fail_class;
    }

    /* 2.添加设备 */
    beep.device = device_create(beep.class,NULL,beep.devid,NULL,BEEP_NAME);
    if(IS_ERR(beep.device)){
        ret = PTR_ERR(beep.device);
        goto fail_device;
    }

    /* 设置Beep所使用的GPIO */
    /* 1.获取设备节点：beep */
    beep.nd = of_find_node_by_path("/beep");  //设备节点名
    if(beep.nd == NULL){
        printk("beep node can't not found\n");
        ret = -EINVAL;
        goto fail_node;
    }else{
        printk("beep node has been found\n");
    }

    /* 2.获取设备树中的gpio属性，得到Beep的编号 */
    beep.beep_gpio = of_get_named_gpio(beep.nd,"beep-gpio",0);//根据设备树实际命名更改
    if(beep.beep_gpio < 0){
        printk("can't get beep-gpio\n");
        ret = -EINVAL;
        goto fail_node;
    }
    printk("beep-gpio num = %d\n",beep.beep_gpio);

    /* 3.请求gpio */
    ret = gpio_request(beep.beep_gpio,"beep-gpio");
    if(ret){
        printk("can't request beep-gpio\n");
        goto fail_node;
    }

    /* 4.设置 GPIO5_IO01 为输出，并且输出高电平，默认关闭 BEEP */
    ret = gpio_direction_output(beep.beep_gpio,1);
    if(ret < 0){
        printk("can't set gpio\n");
        goto fail_setoutput;
    }

    return 0;

fail_setoutput:
    gpio_free(beep.beep_gpio);
fail_node:
    device_destroy(beep.class,beep.devid);
fail_device:
    class_destroy(beep.class);
fail_class:
    cdev_del(&beep.cdev);
fail_cdev:
    unregister_chrdev_region(beep.devid,BEEP_CNT);
fail_devid:
    return ret;
}

/* 驱动出口函数 */
static void __exit beep_exit(void)
{
    /* 关闭BEEP */
    gpio_set_value(beep.beep_gpio,BEEPOFF);
    gpio_free(beep.beep_gpio);

    /* 注销字符设备驱动 */
    cdev_del(&beep.cdev);    //删除cdev
    unregister_chrdev_region(beep.devid,BEEP_CNT);    //注销设备号


    device_destroy(beep.class,beep.devid);    //摧毁设备
    class_destroy(beep.class);   //摧毁类

    return;
}

/* 注册驱动和卸载驱动 */
module_init(beep_init);
module_exit(beep_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("bcl");
```

## Linux并发与竞争

Linux是一个多任务操作系统，肯定会存在多个任务共同操作同一段内存或者设备的情况，多个任务甚至中断都能访问的资源叫做共享资源。在驱动开发中要注意对共享资源的保护，也就是要处理对共享资源的并发访问。

现在的 Linux 系统并发产生的原因很复杂，总结一下有下面几个主要原因：

①、多线程并发访问，Linux 是多任务(线程)的系统，所以多线程访问是最基本的原因。

②、抢占式并发访问，从 2.6 版本内核开始，Linux 内核支持抢占，也就是说调度程序可以 在任意时刻抢占正在运行的线程，从而运行其他的线程。

③、中断程序并发访问，这个无需多说，STM32 中硬件中断的权利可是很大的。

④、SMP(多核)核间并发访问，现在 ARM 架构的多核 SOC 很常见，多核 CPU 存在核间并发访问。

### 原子操作

原子操作就是指不能再进一步分割的操作，一般原子操作用于变量或者位操作。

Linux 内核 提供了一组原子操作 API 函数来完成此功能，Linux 内核提供了两组原子操作 API 函数，一组是对整形变量进行操作的，一组是对位进行操作的，我们接下来看一下这些 API 函数。

#### 原子操作API

```c
typedef struct {
	int counter;
} atomic_t;	//64位SOC使用atomic64_t
```

此结构体定义在 include/linux/types.h 文件中

atomic_t b = ATOMIC_INIT(0); //定义原子变量 b 并赋初值为 0

**原子整型操作API**

|                    函数                     |                     描述                     |
| :-----------------------------------------: | :------------------------------------------: |
|             ATOMIC_INIT(int i)              |        定义原子变量的时候对其初始化。        |
|        int atomic_read(atomic_t *v)         |           读取 v 的值，并且返回。            |
|     void atomic_set(atomic_t *v, int i)     |               向 v 写入 i 值。               |
|     void atomic_add(int i, atomic_t *v)     |               给 v 加上 i 值。               |
|     void atomic_sub(int i, atomic_t *v)     |               从 v 减去 i 值。               |
|        void atomic_inc(atomic_t *v)         |           给 v 加 1，也就是自增。            |
|        void atomic_dec(atomic_t *v)         |           从 v 减 1，也就是自减。            |
|     int atomic_dec_return(atomic_t *v)      |         从 v 减 1，并且返回 v 的值。         |
|     int atomic_inc_return(atomic_t *v)      |         给 v 加 1，并且返回 v 的值。         |
| int atomic_sub_and_test(int i, atomic_t *v) | 从 v 减 i，如果结果为 0 就返回真，否则返回假 |
|    int atomic_dec_and_test(atomic_t *v)     | 从 v 减 1，如果结果为 0 就返回真，否则返回假 |
|    int atomic_inc_and_test(atomic_t *v)     | 给 v 加 1，如果结果为 0 就返回真，否则返回假 |
| int atomic_add_negative(int i, atomic_t *v) | 给 v 加 i，如果结果为负就返回真，否则返回假  |

**原子位操作API**

|                   函数                   |                       描述                        |
| :--------------------------------------: | :-----------------------------------------------: |
|      void set_bit(int nr, void *p)       |             将 p 地址的第 nr 位置 1。             |
|      void clear_bit(int nr,void *p)      |             将 p 地址的第 nr 位清零。             |
|     void change_bit(int nr, void *p)     |           将 p 地址的第 nr 位进行翻转。           |
|      int test_bit(int nr, void *p)       |            获取 p 地址的第 nr 位的值。            |
|  int test_and_set_bit(int nr, void *p)   | 将 p 地址的第 nr 位置 1，并且返回 nr 位原来的值。 |
| int test_and_clear_bit(int nr, void *p)  | 将 p 地址的第 nr 位清零，并且返回 nr 位原来的值。 |
| int test_and_change_bit(int nr, void *p) | 将 p 地址的第 nr 位翻转，并且返回 nr 位原来的值。 |

### 自旋锁

原子操作只能对整形变量或者位进行保护，但是，在实际的使用环境中怎么可能只有整形变量或位这么简单的临界区。

对于自旋锁而言，如果自旋锁正在被线程 A 持有，线程 B 想要获取自旋锁，那么线程 B 就会处于忙循环-旋转-等待状态，线程 B 不会进入休眠状态或者说去做其他的处理，而是会一直傻傻的在那里“转圈圈”的等待锁可用。

**自旋锁的缺点**：等待自旋锁的线程会一直处于自旋状态，这样会浪费处理器时间，降低系统性能，所以自旋锁的持有时间不能太长。

```c
typedef struct spinlock {
	union {
		struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
		struct {
			u8 __padding[LOCK_PADSIZE];
			struct lockdep_map dep_map;
		};
#endif
	};
} spinlock_t;
```

#### 自旋锁API

**自旋锁基本 API 函数表**

|                 函数                 |                             描述                             |
| :----------------------------------: | :----------------------------------------------------------: |
|   DEFINE_SPINLOCK(spinlock_t lock)   |                  定义并初始化一个自选变量。                  |
| int spin_lock_init(spinlock_t *lock) |                        初始化自旋锁。                        |
|   void spin_lock(spinlock_t *lock)   |                获取指定的自旋锁，也叫做加锁。                |
|  void spin_unlock(spinlock_t *lock)  |                      释放指定的自旋锁。                      |
|  int spin_trylock(spinlock_t *lock)  |         尝试获取指定的自旋锁，如果没有获取到就返回 0         |
| int spin_is_locked(spinlock_t *lock) | 检查指定的自旋锁是否被获取，如果没有被获取就返回非 0，否则返回 0。 |

上述这些API适用于线程之间的并发访问，被自旋锁保护的临界区一定不能调用任何能够引起睡眠和阻塞的 API 函数，否则的话会可能会导致死锁现象的发生。

线程之间的并发访问，中断有时候也会插手，在中断里也可以使用自旋锁，在获取锁之前一定要先禁止本地中断（也就是本 CPU 中断，对于多核 SOC 来说会有多个 CPU 核）。

**线程与中断并发访问处理 API 函数**

|                             函数                             |                            描述                             |
| :----------------------------------------------------------: | :---------------------------------------------------------: |
|             void spin_lock_irq(spinlock_t *lock)             |                禁止本地中断，并获取自旋锁。                 |
|            void spin_unlock_irq(spinlock_t *lock)            |                激活本地中断，并释放自旋锁。                 |
| void spin_lock_irqsave(spinlock_t *lock,  unsigned long flags) |         保存中断状态，禁止本地中断，并获取自旋锁。          |
| void spin_unlock_irqrestore(spinlock_t  *lock, unsigned long flags) | 将中断状态恢复到以前的状态，并且激活本地中断， 释放自旋锁。 |

建议使用 spin_lock_irqsave/spin_unlock_irqrestore，因为这一组函数会保存中断状态，在释放锁的时候会恢复中断状态。一般在线程中使用 spin_lock_irqsave/ spin_unlock_irqrestore，在中断中使用 spin_lock/spin_unlock。

```c
DEFINE_SPINLOCK(lock) /* 定义并初始化一个锁 */

/* 线程 A */
void functionA (){
	unsigned long flags; /* 中断状态 */
	spin_lock_irqsave(&lock, flags) /* 获取锁 */
	/* 临界区 */
	spin_unlock_irqrestore(&lock, flags) /* 释放锁 */
}

/* 中断服务函数 */
void irq() {
	spin_lock(&lock) /* 获取锁 */
	/* 临界区 */
	spin_unlock(&lock) /* 释放锁 */
}
```

**下半部竞争处理函数**

|                 函数                  |            描述            |
| :-----------------------------------: | :------------------------: |
|  void spin_lock_bh(spinlock_t *lock)  | 关闭下半部，并获取自旋锁。 |
| void spin_unlock_bh(spinlock_t *lock) | 打开下半部，并释放自旋锁。 |

下半部(BH)也会竞争共享资源，有些资料也会将下半部叫做底半部。

#### 其他类型的锁

读写自旋锁

顺序锁

#### 自旋锁使用注意事项

①、因为在等待自旋锁的时候处于“自旋”状态，因此锁的持有时间不能太长，一定要短，否则的话会降低系统性能。如果临界区比较大，运行时间比较长的话要选择其他的并发处理方式，比如稍后要讲的信号量和互斥体。

②、自旋锁保护的临界区内不能调用任何可能导致线程休眠的 API 函数，否则的话可能导致死锁。 

③、不能递归申请自旋锁，因为一旦通过递归的方式申请一个你正在持有的锁，那么你就必须“自旋”，等待锁被释放，然而你正处于“自旋”状态，根本没法释放锁。结果就是自己把自己锁死了！ 

④、在编写驱动程序的时候我们必须考虑到驱动的可移植性，因此不管你用的是单核的还是多核的 SOC，都将其当做多核 SOC 来编写驱动程序。

### 信号量

信号量的特点：
①、因为信号量可以使等待资源线程进入休眠状态，因此适用于那些占用资源比较久的场合。

②、因此**信号量不能用于中断**中，因为信号量会引起休眠，中断不能休眠。

 ③、如果共享资源的持有时间比较短，那就不适合使用信号量了，因为频繁的休眠、切换线程引起的开销要远大于信号量带来的那点优势。

#### 信号量API

```c
struct semaphore {
 	raw_spinlock_t lock;
 	unsigned int count;
 	struct list_head wait_list;
};
```

|                      函数                      |                             描述                             |
| :--------------------------------------------: | :----------------------------------------------------------: |
|             DEFINE_SEAMPHORE(name)             |           定义一个信号量，并且设置信号量的值为 1。           |
| void sema_init(struct semaphore *sem, int val) |            初始化信号量 sem，设置信号量值为 val。            |
|        void down(struct semaphore *sem)        |      获取信号量，因为会导致休眠，因此不能在中断中使用。      |
|    int down_trylock(struct semaphore *sem);    | 尝试获取信号量，如果能获取到信号量就获取，并且返回 0。如果不能就返回非 0，并且 不会进入休眠。 |
| int down_interruptible(struct semaphore *sem)  | 获取信号量，和 down 类似，只是使用 down 进入休眠状态的线程不能被信号打断。而使用此函数进入休眠以后是可以被信号打断的。 |
|         void up(struct semaphore *sem)         |                          释放信号量                          |

```c
/* 信号量使用示例 */
struct semaphore sem; /* 定义信号量 */
sema_init(&sem, 1); /* 初始化信号量 */
down(&sem); /* 申请信号量 */
/* 临界区 */
up(&sem); /* 释放信号量 */
```

### 互斥体

```c
struct mutex {
 	/* 1: unlocked, 0: locked, negative: locked, possible waiters */
 	atomic_t count;
 	spinlock_t wait_lock;
};
```

使用 mutex 的时候要注意如下几点：

①、mutex 可以导致休眠，因此不能在中断中使用 mutex，中断中只能使用自旋锁。 

②、和信号量一样，mutex 保护的临界区可以调用引起阻塞的 API 函数。 

③、因为一次只有一个线程可以持有 mutex，因此，必须由 mutex 的持有者释放 mutex。并且 mutex 不能递归上锁和解锁。

#### 互斥体API

|                       函数                       |                          描述                           |
| :----------------------------------------------: | :-----------------------------------------------------: |
|                DEFINE_MUTEX(name)                |              定义并初始化一个 mutex 变量。              |
|           void mutex_init(mutex *lock)           |                     初始化 mutex。                      |
|       void mutex_lock(struct mutex *lock)        | 获取 mutex，也就是给 mutex 上锁。如果获取不到就进休眠。 |
|      void mutex_unlock(struct mutex *lock)       |             释放 mutex，也就给 mutex 解锁。             |
|      int mutex_trylock(struct mutex *lock)       |  尝试获取 mutex，如果成功就返回 1，如果失败就返回 0。   |
|     int mutex_is_locked(struct mutex *lock)      | 判断 mutex 是否被获取，如果是的话就返回 1，否则返回 0。 |
| int mutex_lock_interruptible(struct mutex *lock) |  使用此函数获取信号量失败进入休眠以后可以被信号打断。   |

```c
/* 互斥体使用示例 */
struct mutex lock; /* 定义一个互斥体 */
mutex_init(&lock); /* 初始化互斥体 */

mutex_lock(&lock); /* 上锁 */
/* 临界区 */
mutex_unlock(&lock); /* 解锁 */
```

## Linux并发与竞争实验

### 原子操作实验

拷贝gpioled文件夹，更改内部的文件名，稍作修改。详细见代码。

### 自旋锁实验

拷贝gpioled文件夹，更改内部的文件名，稍作修改。详细见代码。

### 信号量实验

拷贝gpioled文件夹，更改内部的文件名，稍作修改。详细见代码。

### 互斥体实验

拷贝gpioled文件夹，更改内部的文件名，稍作修改。详细见代码。

## Linux输入按键实验

只是演示LinuxGPIO输入驱动的编写，实际按键驱动会使用input子系统。

### 修改设备树

#### 添加pinctrl节点

```
		pinctrl_key: keygrp {
			fsl,pins = <
				MX6UL_PAD_UART1_CTS_B__GPIO1_IO18	0xf080	/* KEY0 */
			>;
		};
```

#### 添加key设备节点

```
	/* key */
	key {
		#address-cells = <1>;
		#size-cells = <1>;
		compatible = "atkalpha-key";
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_key>;
		key-gpio = <&gpio1 18 GPIO_ACTIVE_LOW>;
		status = "okay";
	};
```

### 编写驱动代码

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_gpio.h>

#define KEY_CNT     1           /* 设备号个数 */
#define KEY_NAME    "key"       /* 名字 */

/* 定义按键值 */
#define KEY0VALUE         0xf0           /* 按键值 */
#define INVAKEY           0x00           /* 无效按键值 */

/* key设备结构体 */
struct key_dev{
    dev_t devid;                /* 设备号 */
    int major;                  /* 主设备号 */
    int minor;                  /* 次设备号 */
    struct cdev cdev;           /* cdev */
    struct class *class;        /* 类 */
    struct device *device;      /* 设备 */
    struct device_node *nd;     /* 设备节点 */
    int key_gpio;               /* key使用的GPIO编号 */
    atomic_t key_value;         /* 按键值 */
};

struct key_dev keydev; /* key设备 */

static int key_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &keydev;   //设置私有属性
    return 0;
}

static int key_close(struct inode *inode, struct file *filp)
{
    return 0;
}

static ssize_t key_read(struct file *filp, char __user *buf, size_t count, loff_t *ppos)
{
    int ret = 0;
    unsigned char value;
    struct key_dev *dev = (struct key_dev *)filp->private_data;

    if(gpio_get_value(dev->key_gpio) == 0){     /* key0按下 */
        while(!gpio_get_value(dev->key_gpio));  /* 等待按键释放 */
        atomic_set(&dev->key_value,KEY0VALUE);
    }else{
        atomic_set(&dev->key_value,INVAKEY);
    }

    value = atomic_read(&dev->key_value);       /* 保存按键值 */
    ret = copy_to_user(buf,&value,sizeof(value));
    
    return ret;
}

/* 设备操作函数 */
static const struct file_operations key_fops = {
    .owner   = THIS_MODULE,
    .open    = key_open,
    .release = key_close,
    .read    = key_read,
};

/* 驱动入口函数 */
static int __init key_init(void)
{
    int ret;

    /* 初始化原子变量 */
    atomic_set(&keydev.key_value,INVAKEY);

    /* 申请设备号 */
    keydev.major = 0;  //由内核分配
    if(keydev.major){  //给定设备号
        keydev.devid = MKDEV(keydev.major,0);
        ret = register_chrdev_region(keydev.devid,KEY_CNT,KEY_NAME);
    }else{
        ret = alloc_chrdev_region(&keydev.devid,0,KEY_CNT,KEY_NAME);
        keydev.major = MAJOR(keydev.devid);
        keydev.minor = MINOR(keydev.devid);
    }
    printk("key major = %d,minor = %d\n",keydev.major,keydev.minor);
    if(ret < 0){
        goto fail_devid;
    }

    /* 2.添加字符设备 */
    keydev.cdev.owner = THIS_MODULE;
    cdev_init(&keydev.cdev,&key_fops);
    ret = cdev_add(&keydev.cdev,keydev.devid,KEY_CNT);
    if(ret < 0){
        goto fail_cdev;
    }

    /* 自动添加设备节点 */
    /* 1.添加类 */
    keydev.class = class_create(THIS_MODULE,KEY_NAME);
    if(IS_ERR(keydev.class)){
        ret = PTR_ERR(keydev.class);
        goto fail_class;
    }

    /* 2.添加设备 */
    keydev.device = device_create(keydev.class,NULL,keydev.devid,NULL,KEY_NAME);
    if(IS_ERR(keydev.device)){
        ret = PTR_ERR(keydev.device);
        goto fail_device;
    }

    /* 设置KEY所使用的GPIO */
    /* 1.获取设备节点：key */
    keydev.nd = of_find_node_by_path("/key");  //设备节点名
    if(keydev.nd == NULL){
        printk("key node can't not found\n");
        ret = -EINVAL;
        goto fail_node;
    }else{
        printk("key node has been found\n");
    }

    /* 2.获取设备树中的gpio属性，得到KEY的编号 */
    keydev.key_gpio = of_get_named_gpio(keydev.nd,"key-gpio",0);//根据设备树实际命名更改
    if(keydev.key_gpio < 0){
        printk("can't get key-gpio\n");
        ret = -EINVAL;
        goto fail_node;
    }
    printk("key-gpio num = %d\n",keydev.key_gpio);

    /* 3.请求gpio */
    ret = gpio_request(keydev.key_gpio,"key-gpio");
    if(ret){
        printk("can't request key-gpio\n");
        goto fail_node;
    }

    /* 4.设置 GPIO1_IO018 为输入 */
    ret = gpio_direction_input(keydev.key_gpio);
    if(ret < 0){
        printk("can't set gpio\n");
        goto fail_setoutput;
    }

    return 0;

fail_setoutput:
    gpio_free(keydev.key_gpio);
fail_node:
    device_destroy(keydev.class,keydev.devid);
fail_device:
    class_destroy(keydev.class);
fail_class:
    cdev_del(&keydev.cdev);
fail_cdev:
    unregister_chrdev_region(keydev.devid,KEY_CNT);
fail_devid:
    return ret;
}

/* 驱动出口函数 */
static void __exit key_exit(void)
{
    /* 释放KEY的gpio */
    gpio_free(keydev.key_gpio);

    /* 注销字符设备驱动 */
    cdev_del(&keydev.cdev);    //删除cdev
    unregister_chrdev_region(keydev.devid,KEY_CNT);    //注销设备号


    device_destroy(keydev.class,keydev.devid);    //摧毁设备
    class_destroy(keydev.class);   //摧毁类

    return;
}

/* 注册驱动和卸载驱动 */
module_init(key_init);
module_exit(key_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("bcl");
```

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

/** 应用代码
 * ./keyAPP <filename> 
 * ./keyAPP /dev/key 
*/

/* 定义按键值 */
#define KEY0VALUE         0xf0           /* 按键值 */
#define INVAKEY           0x00           /* 无效按键值 */

int main(int argc,char *argv[])
{
    int fd;
    int ret;
    char *filename;
    unsigned char keyvalue;

    /* 参数数量检测 */
    if (argc != 2)
    {
        printf("Error usage\n");
        return -1;
    }

    filename = argv[1];

    /* 打开文件 */
    fd = open(filename,O_RDWR);
    if(fd < 0){
        perror("open");
        return -1;
    }

    /* 循环读取按键值数据！ */
    while(1)
    {
        read(fd,&keyvalue,sizeof(keyvalue));
        if(keyvalue == KEY0VALUE)
        {
            printf("KEY0 Press, value = %#X\r\n",keyvalue);
        }
        
    }

    /* 关闭文件 */
    ret = close(fd);
    if(ret){
        perror("close");
        return -1;
    }
    
    return 0;
}
```

## Linux内核定时器实验

### Linux 时间管理和内核定时器简介

Linux 内核中有大量的函数需要时间管理，中断周期性产生的频率就是系统频率， 也叫做节拍率(tick rate)(有的资料也叫系统频率)，比如 1000Hz，100Hz 等等说的就是系统节拍率。系统节拍率是可以设置的，单位是 Hz，我们在编译 Linux 内核的时候可以通过图形化界面设置系统节拍率，按照如下路径打开配置界面：

```
-> Kernel Features 
 -> Timer frequency (<choice> [=y]) 
```

高节拍率和低节拍率的优缺点：

①、高节拍率会提高系统时间精度。高精度时钟的好处有很多，对于那些对时间要求严格的函数来说，能够以更高的精度运行，时间测量也更加准确。

②、高节拍率会导致中断的产生更加频繁，频繁的中断会加剧系统的负担。但是现在的处理器性能都很强大，所以采用 1000Hz 的系统节拍率并不会增加太大的负载压力。

Linux 内核使用全局变量 jiffies 来记录系统从启动以来的系统节拍数，系统启动的时候会 将 jiffies 初始化为 0，jiffies 定义在文件 include/linux/jiffies.h

jiffies 分为32位和64位，32位需要考虑绕回问题，64位则不需要。

所以有以下几个API来比较：

|             函数              |
| :---------------------------: |
|   time_after(unkown, known)   |
|  time_before(unkown, known)   |
| time_after_eq(unkown, known)  |
| time_before_eq(unkown, known) |

unkown 通常为 jiffies，known 通常是需要对比的值。 

为了方便开发，Linux 内核提供了几个 jiffies 和 ms、us、ns 之间的转换函数

|                    函数                     |                      描述                      |
| :-----------------------------------------: | :--------------------------------------------: |
| int jiffies_to_msecs(const unsigned long j) |  将 jiffies 类型的参数 j 分别转换为对应的毫秒  |
| int jiffies_to_usecs(const unsigned long j) | 将 jiffies 类型的参数 j 分别转换为对应的微秒。 |
| u64 jiffies_to_nsecs(const unsigned long j) | 将 jiffies 类型的参数 j 分别转换为对应的纳秒。 |
| long msecs_to_jiffies(const unsigned int m) |          将毫秒转换为 jiffies 类型。           |
| long usecs_to_jiffies(const unsigned int u) |          将微秒转换为 jiffies 类型。           |
|    unsigned long nsecs_to_jiffies(u64 n)    |          将纳秒转换为 jiffies 类型。           |



Linux 内核使用 timer_list 结构体表示内核定时器，timer_list 定义在文件 include/linux/timer.h 中

```c
struct timer_list {
 	struct list_head entry;
 	unsigned long expires; /* 定时器超时时间，单位是节拍数 */
 	struct tvec_base *base;
 	void (*function)(unsigned long); /* 定时处理函数 */
 	unsigned long data; /* 要传递给 function 函数的参数 */
 	int slack;
};
```

内核定时器不是周期性的，一次定时时间到了之后就会关闭，需要重新打开。

比如我们现在需要定义一个周期为 2 秒的定时器，那么这个定时器的超时时间就是 jiffies+(2*HZ)，因此 expires=jiffies+(2*HZ)。

function 就是定时器超时以后的定时处理函数，我们要做的工作就放到这个函数里面，需要我们编写这个定时处理函数。

定义好定时器以后还需要通过一系列的 API 函数来初始化此定时器

1. **init_timer 函数**

    init_timer 函数负责初始化 timer_list 类型变量，当我们定义了一个 timer_list 变量以后一定要先用 init_timer 初始化一下。

2. **add_timer 函数**

    dd_timer 函数用于向 Linux 内核注册定时器，使用 add_timer 函数向内核注册定时器以后，定时器就会开始运行。

3. **del_timer 函数**

    del_timer 函数用于删除一个定时器，不管定时器有没有被激活，都可以使用此函数删除。 在多处理器系统上，定时器可能会在其他的处理器上运行，因此在调用 del_timer 函数删除定时 器之前要先等待其他处理器的定时处理器函数退出。

4. **del_timer_sync 函数**

    del_timer_sync 函数是 del_timer 函数的同步版，会等待其他处理器使用完定时器再删除， del_timer_sync 不能使用在中断上下文中。

5. **mod_timer 函数**

    mod_timer 函数用于修改定时值，如果定时器还没有激活的话，mod_timer 函数会激活定时器！

有时候我们需要在内核中实现短延时，尤其是在 Linux 驱动中。Linux 内核提供了毫秒、微秒和纳秒延时函数

|               函数                |      描述      |
| :-------------------------------: | :------------: |
| void ndelay(unsigned long nsecs)  | 纳秒延时函数。 |
| void udelay(unsigned long usecs)  | 微秒延时函数。 |
| void mdelay(unsigned long mseces) | 毫秒延时函数。 |

### 使用内核定时器周期性点灯

程序中使用了ioctl函数，在驱动中需要在设备操作函数file_operations中添加unlocked_ioctl

这里会涉及到下面两个函数的使用。

```c
long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
```

unlocked_ioctl在无大内核锁（BKL）的情况下调用。64位用户程序运行在64位的kernel，或32位的用户程序运行在32位的kernel上，都是调用unlocked_ioctl函数。

compat_ioctl是64位系统提供32位ioctl的兼容方法，也在无大内核锁的情况下调用。即如果是32位的用户程序调用64位的kernel，则会调用compat_ioctl。如果驱动程序没有实现compat_ioctl，则用户程序在执行ioctl时会返回错误Not a typewriter。

[linux驱动开发(四)：ioctl()函数_ioctl函数_精致的螺旋线的博客-CSDN博客](https://blog.csdn.net/baidu_38797690/article/details/123690825)
