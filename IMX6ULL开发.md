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
