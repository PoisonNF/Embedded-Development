# Linux-UNIX系统编程手册

[在 Ubuntu 上安装 .NET - .NET | Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/core/install/linux-ubuntu#2004)

## 常见问题

1. 编译出现undefined reference to xxx

    - 链接时缺失了相关目标文件（.o）

        ```makefile
        # test.c中有main.c的函数
        # 生成.o文件
        gcc -c test.c
        gcc -c main.c	
        
        # 编译main.c
        gcc -o main main.o test.o
        
        # 直接生成
        gcc -o main main.c test.c 
        ```
    - 链接时缺少相关的库文件（.a/.so）
    
        ```makefile
        # test.c中有main.c的函数
        # 先将test.c编译成静态库.a
        gcc -c test.c
        ar -rc test.a test.o
        
        # 编译main.c
        gcc -c main.c
        
        # 链接
        gcc -o main mian.o ./test.a
        
        # 一次性生成
        gcc -o main main.c ./test.a
        ```
    - 多个库文件链接顺序问题
    
        如果存在a引用b，b引用c，c引用d，需要注意库之间的依赖顺序，依赖其他库的库一定要放到被依赖库的前面，此时顺序变成abcd，
    
    > [“undefined reference to“ 解决方法_daijingxin的博客-CSDN博客](https://blog.csdn.net/daijingxin/article/details/117292473)

## Chapter1 历史和标准

## Chapter2 基本概念

## Chapter3 常用的函数及头文件

- tlpi_hdr.h
- error_functions.c
- error_functions.h
- ename.c.inc
- get_num.c
- get_num.h

上述文件为该书中普遍采用的定义和头文件，具体代码可以在Linux_Code中的lib文件夹中查看。

一般警告使用printf语句。

> strtol() [C 库函数 – strtol() | 菜鸟教程 (runoob.com)](https://www.runoob.com/cprogramming/c-function-strtol.html)
>
> ssize_t和size_t的区别：
>
> - ssize_t是有符号整型，在32位机器上等同与int，在64位机器上等同与long int
> - size_t 就是无符号型的ssize_t，也就是unsigned long/ unsigned int (在32位下）

## Chapter4 文件I/O

### 打开一个文件：open()

```c
#include <unistd.h>
int open(const char *pathname,int flags,mode_t mode);
```

open()调用既能打开一个业已存在的文件，也能创建并打开一个新文件。

参数 flags 为位掩码，用于指定文件的访问模式

当调用 open()创建新文件时，位掩码参数 mode 指定了文件的访问权限

> 使用 I/O 系统调用 fileio/copy.c 
>
> open 函数使用的例子 fileio/open.c 

![image-20230312210717507](./Linux开发.assets/image-20230312210717507.png)

![image-20230312210732673](./Linux开发.assets/image-20230312210732673.png)

### 读取文件内容：read()

```c
#include <unistd.h>
ssize_t read(int fd,void *buffer,size_t count);
```

read()系统调用从文件描述符 fd 所指代的打开文件中读取数据。

count 参数指定最多能读取的字节数。（size_t 数据类型属于无符号整数类型。）

buffer 参数 提供用来存放输入数据的内存缓冲区地址。缓冲区至少应有 count 个字节。

### 数据写入文件：write()

```c
#include <unistd.h>
ssize_t write(int fd,void *buffer,size_t count);
```

write()系统调用将数据写入一个已打开的文件中。

buffer 参数为要写入文件中数据的内存地址，

count 参数为欲从 buffer 写入文件的数据字节数，

fd 参数为一文件描述符，指代数据要写入的文件

### 关闭文件：close()

```c
#include <unistd.h>
int close(int fd);
```

close()系统调用关闭一个打开的文件描述符，并将其释放回调用进程，供该进程继续使用。

### 改变文件偏移量：lseek()

```c
#include <unistd.h>
off_t lseek(int fd,off_t offset,int whence);
```

lseek()系统调用依照 offset 和 whence 参数值调整该文件的偏移量。

offset 参数指定了一个以字节为单位的数值。

whence 参数则表明应参照哪个基点来解释 offset 参数，应为下列其中之一：

-  SEEK_SET  将文件偏移量设置为从文件头部起始点开始的 offset 个字节。 
- SEEK_CUR  相对于当前文件偏移量，将文件偏移量调整 offset 个字节。 
- SEEK_END  将文件偏移量设置为起始于文件尾部的 offset 个字节。也就是说，offset 参数应该从文件 最后一个字节之后的下一个字节算起。

### 文件空洞

从文件结尾后到新写入数据间的这段空间被称为文件空洞。从编程角度看，文件空洞中是存在字节的，读取空洞将返回以 0（空字节）填充的缓冲区。

> read()、write()和 lseek()的使用示范 fileio/seek_io.c

### 非通用I/O模型：ioctl()

```c
#Include <sys/ioctl.h>
int ioctl(int fd,int request,...);
```

## Chapter5 深入探究文件I/O

### 原子操作和竞争

原子操作：内核保证了某系统调用中的所有步骤会作为独立操作而一次性加以执行，其间不会为其他进程或线程所中断。

在open()的时候可以使用O_CREAT和O_EXCL标志一次性调用，确保检查文件和创建文件的步骤属于一个原子操作。

在write()的时候，打开文件时加入O_APPEND标志，保证文件偏移量的移动与数据写操作纳入同一原子操作。

竞争状态：：操作共享资源的两个进程（或线程），其 结果取决于一个无法预期的顺序，即这些进程1 获得 CPU 使用权的先后相对顺序。

> 试图以独占方式打开文件的错误代码 fileio/bad_exclusive_open.c

### 文件控制操作：fcntl()

```c
#include <fcntl.h>
int fcntl(int fd,int cmd,...);
```

fcntl()系统调用对一个打开的文件描述符执行一系列控制操作。cmd在后续章节中。

### 打开文件的状态标志

fcntl()用途

- 就是针对一个打开的文件，获取或修改其访问模式和状态标志。此时cmd参数为F_GETFL。

- 复制文件描述符，F_DUPFD操作

    `newfd = fcntl(oldfd,F_DUPFD,startfd);`

    该调用为 oldfd 创建一个副本，且将使用大于等于 startfd 的最小未用值作为描述符编号。 该调用还能保证新描述符（newfd）编号落在特定的区间范围内。

O_ACCMODE为0003，用于与flag相与获取访问模式。O_RDONLY(0)、O_WRONLY(1)和 O_RDWR(2)

```c
/* 修改一个文件的状态标志，添加O_APPEND标志 */

int flags;
/* 使用F_GETFL获取标志的副本 */
flags = fcntl(fd,F_GETFL);
if(flags == -1)
    errExit("fcntl");
/* 修改变更的比特位 */
flags |= O_APPEND;
/* 使用F_SETFL更新状态标志 */
if(fcntl(fd,F_SETFL,flags) == -1)
    errExit("fcntl");
```

### 文件描述符和打开文件之间的关系

内核维护的3个数据结构

1. 进程级的文件描述符表
    - 控制文件描述符操作的一组标志
    - 对打开文件句柄的引用
2. 系统级的打开文件表
    - 当前文件偏移量（调用 read()和 write()时更新，或使用 lseek()直接修改）
    - 打开文件时所使用的状态标志（即，open()的 flags 参数）
    - 文件访问模式（如调用 open()时所设置的只读模式、只写模式或读写模式）
    - 与信号驱动 I/O 相关的设置
    - 对该文件 i-node 对象的引用
3. 文件系统的 i-node 表
    - 文件类型（例如，常规文件、套接字或 FIFO）和访问权限
    - 一个指针，指向该文件所持有的锁的列表
    - 文件的各种属性，包括文件大小以及与不同类型操作相关的时间戳

![image-20230312205821907](./Linux开发.assets/image-20230312205821907.png)

### 复制文件描述符dup/dup2

#### dup

```c
#include <unistd.h>
int dup(int oldfd);
```

dup()调用复制一个打开的文件描述符 oldfd，并返回一个新描述符，二者都指向同一打开 的文件句柄。系统会保证新描述符一定是编号值最低的未用文件描述符。

如果复制文件描述符1，希望返回文件描述符2，可以使用以下操作

```c
close(2);
newfd = dup(1);
```

#### dup2

```c
#include <unistd.h>
int dup2(int oldfd,int newfd);
```

dup2()系统调用会为 oldfd 参数所指定的文件描述符创建副本，其编号由 newfd 参数指定。 如果由 newfd 参数所指定编号的文件描述符之前已经打开，那么 dup2()会首先将其关闭。（dup2() 调用会默然忽略 newfd 关闭期间出现的任何错误。故此，编码时更为安全的做法是：在调用 dup2()之前，若 newfd已经打开，则应显式调用 close()将其关闭。

#### dup3（Linux特有）

```c
#define _GUN_SOURCE
#include <unistd.h>
int dup3(int oldfd,int newfd,int flags);
```

dup3()系统调用完成的工作与 dup2()相同，只是新增了一个附加参数 flag，这是一个可以 修改系统调用行为的位掩码。目前，dup3()只支持一个标志 O_CLOEXEC

### 在文件特定偏移量处的I/O：pread()和pwrite()

```c
#include <unistd.h>
ssize_t pread(int fd,void *buf,size_t count,off_t offset);
ssize_t pwrite(int fd,const void *buf,size_t count,off_t offset);
```

这两个系统调用只会在offset指定的位置进行I/O操作，不会改变当前文件的偏移量。常用于多线程编程。

### 分散输入和集中输出：readv()和writev()

```c
#include <sys/uio.h>
ssize_t readv(int fd,const struct iovec *iov,int iovcnt);
ssize_t writev(int fd,const struct iovec *iov,int iovcnt);
```

这些系统调用并非只对单个缓冲区进行读写操作，而是一次即可传输多个缓冲区的数据。 数组 iov 定义了一组用来传输数据的缓冲区。整型数 iovcnt 则指定了 iov 的成员个数。iov 中的每个成员都是如下形式的数据结构。

```c
struct iovec{
	void *iov_base;	/* Start address of buffer */
	size_t iov_len; /* Number of bytes to transfer to/from buffer */
}
```

> 使用 readv()执行分散输入 fileio/t_readv.c

#### 在指定位置分散输入/集中输出

```c
#define _BSD_SOURCE
#include <sys/uio.h>
ssize_t preadv(int fd,const struct iovec *iov,int iovcnt,off_t offset);
ssize_t pwritev(int fd,const struct iovec *iov,int iovcnt,off_t offset);
```

上述两种系统调用的结合

### 截断文件：truncate()和ftruncate()

```c
#include <unistd.h>
int truncate(const cahr *pathname,off_t length);
int ftruncate(int fd,off_t length);
```

truncate()和 ftruncate()系统调用将文件大小设置为 length 参数指定的值。

truncate()指定路径，ftruncate()指定文件描述符。

> truncate()无需先以 open()（或是一些其他方法）来获取文件描述符，却可修改文件内容， 在系统调用中可谓独树一帜。

### 非阻塞I/O

打开文件时指定O_NONBLOCK标志

- 若 open()调用未能立即打开文件，则返回错误，而非陷入阻塞。
- 调用 open()成功后，后续的 I/O 操作也是非阻塞的。

管道、FIFO、套接字、设备（比如终端、伪终端）都支持非阻塞模式。（因为无法通过 open() 来获取管道和套字的文件描述符，所以要启用非阻塞标志，就必须使用fcntl()的 F_SETFL 命令。）

### 大文件I/O

文件偏移量的数据类型off_t是一个有符号的长整型。在32位体系架构中，该长度不超过2^31^-1个字节（2GB）。

所以需要使用LFS拓展，应用程序可以使用如下两种方式之一获得LFS功能：

- 使用支持大文件操作的备选 API。（过渡型拓展）

- 在编译应用程序时，将宏_FILE_OFFSET_BITS 的值定义为 64。（推荐使用）

#### 过渡型LFS API

在编译时定义_LARGEFILE64_SOURCE测试宏，可以通过命令行指定。

函数命名发生变化，包括：fopen64()、open64()、lseek64()、truncate64()、stat64()、mmap64()、setrlimit64()

新数据类型：

- struct stat64：类似于stat，支持大文件尺寸
- off64_t：64位类型，用于表示文件偏移量

> 访问大文件 fileio/large_file.c

#### _FILE_OFFSET_BITS宏

1. 命令行编译时增加选项，例如`$ cc -D_FILE_OFFSET_BITS=64 prog.c`

2. 在所有头文件之前添加宏定义，#define _FILE_OFFSET_BITS 64

#### printf()传递off_t值

将off_t强制转换到long long类型，printf()使用%lld限定符。

### /dev/fd 目录

每个进程都会有个一个虚拟目录，其中包含“/dev/fd/n”形式的文件名，n就是文件描述符的编号。

符号链接： /dev/stdin /dev/stdout /dev/stderr 对应标准输入、标准输出、标准错误

### 创建临时文件

#### mkstemp()

```c
#include <stdlib.h>
int mkstemp(char *template);
```

模板参数采用路径名形式，其中最后 6 个字符必须为 XXXXXX。这 6 个字符将被替换，
以保证文件名的唯一性，且修改后的字符串将通过 template 参数传回。因为会对传入的 template
参数进行修改，所以必须将其指定为字符数组，而非字符串常量。

```c
char template[] = "/tmp/somestringXXXXXX";
fd = mkstemp(template);
if(fd == -1)
    prinf("mkstemp\n");
printf("Generated filename was: %s\n",template);
unlink(template); //删除临时文件
```

#### tmpfile()

```c
#include <stdio.h>
FILE *tmpfile(void);
```

tmpfile()函数会创建一个名称唯一的临时文件，并以读写方式将其打开。（打开该文件时 使用了 O_EXCL 标志，以防一个可能性极小的冲突，即另一个进程已经创建了一个同名文件。

tmpfile()函数执行成功，将返回一个文件流供 stdio 库函数使用。文件流关闭后将自动删除临时文件。为达到这一目的，tmpfile()函数会在打开文件后，从内部立即调用 unlink()来删除该文件名 。

## Chapter6 进程

### 进程和程序

进程（process）是一个可执行程序（program）的实例。程序是包含了一系列信息的文件，这些信息描述了如何在运行时创建一个进程。

### 进程号与父进程号

```c
#include <unistd.h>
pid_t getpid(void);
```

返回调用进程的进程号。进程号超过32767，会重置为300，低数值的进程号为系统进程和守护进程。

```c
#include <unistd.h>
pid_t getppid(void);
```

返回父进程的进程号。1 号进程—init 进程，即所有进程的始祖。

### 进程内存布局

1. 文本段，机器语言指令，只读

2. 数据段，显式初始化的全局变量和静态变量

3. BSS段，未初始化的全局变量和静态变量

4. 栈，系统分配

5. 堆，手动申请释放。堆顶端称作program break

> 程序变量在进程内存各段中的位置 proc/mem_segments.c

![image-20230312205929300](./Linux开发.assets/image-20230312205929300.png)

### 虚拟内存管理

虚拟内存技术：

- 空间局部性：是指程序倾向于访问在最近访问过的内存地址附近的内存
- 时间局部性：是指程序倾向于在不久的将来再次访问最近刚访问过的内存地址

虚拟内存的规划之一是将每个程序使用的内存切割成小型的、固定大小的“页”（page） 单元。相应地，将 RAM 划分成一系列与虚存页尺寸相同的页帧。任一时刻，每个程序仅有部分页需要驻留在物理内存页帧中。这些页构成了所谓驻留集（resident set）。程序未使用的页 拷贝保存在交换区（swap area）内—这是磁盘空间中的保留区域，作为计算机 RAM 的补充—  仅在需要时才会载入物理内存。若进程欲访问的页面目前并未驻留在物理内存中，将会发生页面错误（page fault），内核即刻挂起进程的执行，同时从磁盘中将该页面载入内存。

![image-20230312210134869](./Linux开发.assets/image-20230312210134869.png)

x86-32中，页面大小为4096个字节。

进程的有效虚拟地址范围在其生命周期中可以发生变化。

虚拟内存的优点；

- 进程与进程、进程与内核相互隔离，所以一个进程不能读取或修改另一进程或内核的内存。
- 适当情况下，两个或者更多进程能够共享内存。
- 便于实现内存保护机制。
- 程序员和编译器、链接器之类的工具无需关注程序在 RAM 中的物理布局。
- 因为需要驻留在内存中的仅是程序的一部分，所以程序的加载和运行都很快。
- 每个进程使用的 RAM 减少了，RAM 中同时可以容纳的进程数量就增多了。

### 栈和栈帧

栈驻留在内存的高端并向下增长，栈指针（stack pointer），用于跟踪当前栈顶。每次调用函数时，会在栈上新分配一帧，每当函数返回时，再从栈上将此帧移去。

每个（用户）栈帧包括以下信息：

- 函数实参和局部变量
- （函数）调用的链接信息

### 命令行参数（argc，argv）

int argc 命令行参数的个数

char *argv[ ] 指向命令行参数的指针数组，argv[argc]为 NULL

> 回显命令行参数 proc/necho.c

### 环境列表

环境列表中每个字符串都以名称=值（name=value）形式定义。

新进程在创建之时，会继承其父进程的环境副本。这是一种原始的进程间通信方式，却颇为常用。（单向，一次性）

setenv设置环境变量，unset撤销环境变量，printenv显示当前环境变量

#### 从程序中访问环境

在 C 语言程序中，可以使用全局变量 char **environ 访问环境列表。

> 显示进程环境 proc/display_env.c

另外，还可以通过声明 main()函数中的第三个参数来访问环境列表：

`int main(int argc,char *argv[],char *envp[])`

要避免使用，因为除了局限于作用域限制外， 该特性也不在 SUSv3 的规范之列。

```c
#include <stdlib.h>
char *getenv(const char *name);
```

向 getenv()函数提供环境变量名称，该函数将返回相应字符串指针。如果不存在指 定名称的环境变量，那么 getenv()函数将返回 NULL。

#### 修改环境

```c
#include <stdlib.h>
int putenv(char *string);
```

putenv()函数向调用进程的环境中添加一个新变量，或者修改一个已经存在的变量值。

putenv()调用失败返回非0值，而不是-1

```c
#include <stdlib.h>
int setenv(const char *name,const char *value,int overwrite);
```

name和value对应环境变量左右，setenv()会添加等号。

overwrite 为非0值，强制改变环境。如果环境中存在name标识的变量，且overwrite值为0，则不改变环境。

```c
#include <stdlib.h>
int unsetenv(const char *name);
```

unsetenv()函数从环境中移除由 name 参数标识的变量。

```c
#define _BSD_SOURCE		/* Or: #define _SVID_SOURCE */
#include <stdlib.h>
int clearenv(void);
```

清除整个环境。

> 修改进程环境  proc/modify_env.c

### 非局部跳转：setjmp()和longjmp()

这两个函数可以实现跨函数的跳转。C 语言的 goto 语句存在一个限制，即不能从当前函数跳转到另一函数。

```c
#include <setjmp.h>
int setjmp(jmp_buf env);
void longjmp(jmp_buf env,int val);
```

初始调用返回值为 0，后续“伪”返回的返回值为 longjmp()调用中 val 参数所指定的任意值。

> 展示函数 setjmp()和 longjmp()的用法 proc/longjmp.c

#### 优化编译器的问题

一般包含指针变量和 char、int、float、long 等任何简单类型的变量会受到longjmp()的干扰，将变量声明为 volatile，是告诉优化器不要对其进行优化，从而避免了代码重组。最好将所有局部变量都声明为volatile

> 编译器的优化和 longjmp()函数相互作用的示例  proc/setjmp_vars.c

#### 尽可能避免使用 setjmp()函数和 longjmp()函数

## Chapter7 内存分配

### 在堆上分配内存

堆是一段长度可变的连续虚拟内存，通常将堆的当前内存边界称为“program break”堆顶

### 调整堆顶：brk()和sbrk()

```c
#include <unistd.h>
int brk(void *end_data_segment);
void *sbrk(intptr_t increment);		//sbrk是在brk基础上实现的一个库函数
```

系统调用 brk()会将 program break 设置为参数 end_data_segment 所指定的位置。不足一页会到下一个内存页的边界处。

调用 sbrk()将 program break 在原有地址上增加从参数 increment 传入的大小。sbrk()返回前一个 program break 的地址，指向这块新分配内存起始位置的指针。调用 sbrk(0)将返回 program break 的当前位置，追踪堆的大小。

### 在堆上分配内存：malloc()和free()

```c
#include <stdlib.h>
void *malloc(size_t size);
void free(void *ptr);
```

malloc( )函数在堆上分配参数 size 字节大小的内存（字节对齐），并返回指向新分配内存起始位置处的指针，其所分配的内存未经初始化。由于 malloc()的返回类型为 void*，因而可以将其赋给任意类型的 C 指针。

free()函数释放 ptr 参数所指向的内存块，并不降低 program break 的位置，而是将这块内存填加到空闲内存列表 中，供后续的 malloc()函数循环使用。

> 示范释放内存时 program break 的行为 memalloc/free_and_sbrk.c

### 在堆上分配内存的其他方法

#### 使用calloc()和realloc()分配

```c
#include <stdlib.h>
void *calloc(size_t numitems,size_t size);
```

参数 mumitems 指定分配对象的数量，size 指定每个对象的大小。calloc()会将已分配的内存初始化为 0。

```c
#include <stdlib.h>
void *realloc(void *ptr,size_t size);
```

调整一块内存的大小，返回的位置可能不同。

若原内存块位于堆的顶部，将对堆空间进行拓展；若原内存在堆的中部，且紧邻其后的空闲内存空间大小不足，realloc()会分配一块新内存，并将原有数据复制到新内存块中。最后这种情况最为常见，还会占用大量 CPU 资源。一般情况下，应尽量避免调用 realloc()。

#### 分配对齐内存

```c
#include <malloc.h>
void *memalign(size_t boundary,size_t size);
```

函数 memalign()分配 size 个字节的内存，起始地址是参数 boundary 的整数倍，而 boundary 必须是 2 的整数次幂。函数返回已分配内存的地址。

```c
#include <stdlib.h>
int posix_memalign(void **memptr,size_t alignment,size_t size);
```

已分配的内存地址通过参数 memptr 返回。内存与 alignment 参数的整数倍对齐 ，alignment 必须是 sizeof（void*）（在大多数硬件架构上是 4 或 8 个字节）与 2 的整数次幂两者间的乘积。

### 在堆栈上分配内存：alloca()

```c
#include <alloca.h>
void *alloca(size_t size);
```

通过增加栈帧的大小从堆栈上分配。不需要使用free()来释放内存。也不可能调用 realloc()来调整由 alloca()分配的内存大小。

## Chapter8 用户与组

### 密码文件：/etc/passwd

bcl：x:1000:1000:bcl,,,:/home/bcl:/bin/bash

登录名 加密的密码：x 用户ID 组ID 注释 主目录 登录shell

### shadow密码文件：/etc/shadow

加密处理的密码则由 shadow 密码文件单独维护，仅供具有特权的程序读取。

### 组文件： /etc/group

users :x：100:

组名 加密过的密码 组ID 用户列表

### 获取用户和组的信息

#### 从密码文件获取记录

```c
#include <pwd.h>
struct passwd *getpwnam(const char *name);
struct passwd *getwuid(uid_t uid);
```

提供name，返回指针。

两者都不可重入。

> [C语言函数重入_c语言可重入函数有哪些_hNicholas的博客-CSDN博客](https://blog.csdn.net/zhongbeida_xue/article/details/81355641)

#### 从组文件获取记录

```c
#include <grp.h>
struct group *getgrnam(const char *name);
struct group *getgrgid(gid_t gid);
```

函数 getgrnam()和 getgrgid()分别通过组名和组 ID 来查找属组信息。

> 在用户名/组名和用户 ID/组 ID 之间互相转换的函数 users_groups/ugid_functions.c

#### 扫描密码文件和组文件中的所有记录

```c
#include <pwd.h>
struct passwd *getwent(void);
void setpwent(void);
void endpwent(void);
```

函数 getpwent()能够从密码文件中逐条返回记录。

当密码文件处理完毕后，可调用 endpwent()将其关闭。

调用 setpwent()函数重返文件起始处。

#### 从 shadow 密码文件中获取记录

```c
#include <shadow.h>
struct spwd *getspnam(const char *name);
struct spwd *getspent(void);

void setspent(void);
void endspent(void);
```

下列函数的作用包括从 shadow 密码文件中获取个别记录，以及扫描该文件中的所有记录。

### 密码加密和用户认证

```c
#define _XOPEN_SOURCE
#include <unistd.h>
char *crypt(const char *key,const char *salt);
```

crypt()算法会接受一个最长可达 8 字符的密钥（即密码），并施之以数据加密算法（DES） 的一种变体。salt 参数指向一个两字符的字符串，用来扰动（改变）DES 算法，设计该技术， 意在使得经过加密的密码更加难以破解。该函数会返回一个指针，指向长度为 13 个字符的字符串，该字符串为静态分配而成，内容即为经过加密处理的密码。

==要想在 Linux 中使用 crypt()==，在编译程序时需在末尾添加–lcrypt 选项，以便程序链接 crypt 库。

```c
#define _BSD_SOURCE
#include <unistd.h>
char *getpass(const char *prompt);
```

读取用户密码。该函数会打印出 prompt 所指向的字符串，读取一行输入，返回以 NULL 结尾的输入字符串（剥离尾部的换行符）作为函数结果。

> 根据 shadow 密码文件验证用户 users_groups/check_password.c

```c
char *fgets(char *s,int size,FILE *stream);
```

s 代表要保存到的内存空间的首地址，可以是字符数组名，也可以是指向字符数组的字符指针变量名。size 代表的是读取字符串的长度。stream 表示从何种流中读取，可以是标准输入流 stdin，也可以是文件流。

获取Linux的root权限：

1. 在终端输入sudo passwd root，输入登录密码
2. 接着输入su，在输入一次密码获取root权限

## Chapter9 进程凭证

### 实际用户ID和实际组ID

实际用户ID为/etc/passwd文件中对应记录的第三字段

实际组ID为/etc/passwd文件中对应记录的第四字段

### 有效用户ID和有效组ID

用于确定授予进程的权限，有效用户ID为0（root的用户ID）的进程拥有超级用户的所有权限。

通常，有效用户ID及组ID与其对应的实际ID相等。有办法使两者不相同。

### Set-User-ID 和Set-Group-ID程序

设置了 set-user-ID 权限位，那么当运行该程序时，进程会取得超级用户权限

访问受保护的文件：创建一个具有对该文件访问权限的专用用户（组）ID，然后再创建一个 set-user-ID （set-group-ID）程序，将进程有效用户（组）ID 变更为这个专用 ID。这样，无需拥有超级用户的所有权限，程序就能访问该文件。

> 将8-2的程序设置为任意用户可以执行，设置为set-user-ID-root程序
>
> chown root check_password
>
> chmod u+s check_password
>
> ls -l check_password

### 保存Set-User-ID 和Set-Group-ID程序

举例说明上述操作的效果，假设某进程的实际用户 ID、有效用户 ID 和保存 set-user-ID 均 为 1000，当其执行了 root 用户（用户 ID 为 0）拥有的 set-user-ID 程序后，进程的用户 ID 将 发生如下变化：

real =1000,effective= 0,saved =0

### 文件系统用户ID和组ID

历史遗留问题，文件系统用户 ID 和组 ID 的值等同于相应的有效用户 ID 和组 ID。

### 辅助组ID

辅助组 ID 用于标识进程所属的若干附加的组。

### 获取和修改实际、有效和保存设置标识

#### 获取实际和有效ID

```c
#include <unistd.h>
uid_t getuid(void);
uid_t geteuid(void);
gid_t getgid(void);
gid_t getegid(void);
```

获取进程的有效ID。

#### 修改有效ID

```c
#include <unistd.h>
int setuid(uid_t uid);
int setgid(gid_t gid);
```

规则：

1. 当非特权进程调用 setuid()时，仅能修改进程的有效用户 ID。
2. 当特权进程以一个非 0 参数调用 setuid()时，其实际用户 ID、有效用户 ID 和保存 set-user-ID 均被置为 uid 参数所指定的值。（会失去特权，无法恢复）

```c
#include <unistd.h>
int seteuid(uid_t uid);
int setegid(gid_t gid);
```

跟上面类似，但是可以对特权收放自如

#### 修改实际ID和有效ID

```c
#include <unistd.h>
int setreuid(uid_t ruid,uid_t euid);
int setregid(gid_t rgid,gid_t egid);
```

这两个系统调用的第一个参数都是新的实际 ID，第二个参数都是新的有效 ID。若只想修 改其中的一个 ID，可以将另外一个参数指定为−1。

永久放弃特权的方法：`setreuid(getuid(),getuid())`

#### 获取实际、有效和保存设置ID

```c
#define _GNU_SOURCE
#include <unistd.h>
int getresuid(uid_t *ruid,uid_t *euid,uid_t *suid);
int getresgid(gid_t *rgid,git_t *egid,gid_t *sgid);
```

getresuid()系统调用将调用进程的当前实际用户 ID、有效用户 ID 和保存 set-user-ID 值返
回至给定 3 个参数所指定的位置。

#### 修改实际、有效和保存设置ID

```c
#define _GNU_SOURCE
#include <unistd.h>
int setresuid(uid_t ruid,uid_t euid,uid_t suid);
int setresgid(gid_t rgid,git_t egid,gid_t sgid);
```

无意修改的参数指定为-1

### 获取和修改文件系统ID

Linux 特有

```c
#include <sys/fsuid.h>
int setfsuid(uid_t fsuid);
int setfsgid(gid_t fsguid);
```

setfsuid()系统调用将进程文件系统用户 ID 修改为参数 fsuid 所指定的值。setfsgid()系统调用将文件系统组 ID 修改为参数 fsgid 所指定的值。

### 获取和修改辅助组ID

```c
#include <unistd.h>
int getgroups(int gidsetsize,gid_t grouplist[]);
```

getgroups()系统调用会将当前进程所属组的集合返回至由参数 grouplist 指向的数组中。

```c
#define _BSD_SOURCE
#include <grp.h>
int setgroups(size_t gidsetsize,const gid_t *grouplist);
int initgroups(const char *user,gid_t group);
```

特权进程能够使用这两个函数修改辅助组ID集合。

setgroups()系统调用用 grouplist 数组所指定的集合来替换调用进程的辅助组 ID。参数 gidsetsize 指定了置于参数 grouplist 数组中的组 ID 数量。

initgroups()函数将扫描/etc/groups 文件，为 user 创建属组列表，以此来初始化调用进程的辅助组 ID。另外，也会将参数 group 指定的组 ID 追加到进程辅助组 ID 的集合中。

### 修改进程凭证的系统调用总结

![image-20230315164344297](./Linux开发.assets/image-20230315164344297.png)

> 显示进程的所有用户 ID 和组 ID proccred/idshow.c

## Chapter10 时间

### 日历时间

内部对时间的表达方式为从Epoch以来的秒数来度量。从格林威治标准时间1970 年 1 月 1 日早晨零点开始。存储在类型为time_t的变量中。

```c
#include <sys/time.h>
int gettimeofday(struct timeval *tv,struct timezone *tz);
```

tv为一个指针

```c
struct timeval{
	time_t 		tv_Sec;		/* Seconds since 00:00:00 ,1 Jan 1970 UTC */
	suseconds_t tv_usec;	/* Addtional microseconds (long int) */
}
```

tz已遭废弃，应始终置为NULL

```c
#include <time.h>
time_t time(time_t *timep);
```

time()系统调用返回自 Epoch 以来的秒数。如果 timep 参数不为 NULL，那么还会将自 Epoch 以来的秒数置于 timep 所指向的位置。

往往采用下面的调用，`t = time(NULL);`

### 时间转换函数

![image-20230315193201199](./Linux开发.assets/image-20230315193201199.png)

#### 将time_t转换为可打印格式

```c
#include <time.h>
char *ctime(const time_T *timep);
```

把一个指向 time_t 的指针作为 timep 参数传入函数 ctime()，将返回一个长达 26 字节的字符串，内含标准格式的日期和时间。

> ctime_r()是ctime()的可重入版本，需要额外指定一个指针参数，用于返回时间字符串。

#### time_t和分解时间之间的转换

```c
#include <time.h>
struct tm *gmtime(const time_t *timep);
struct tm *localtime(const time_t *timep);
```

函数 gmtime()和 localtime()可将一 time_t 值转换为一个所谓的分解时间。

gmtime()转换为对应于UTC的分解时间；localtime(）转换为系统本地时间的分解时间。

```c
#include <time.h>
time_t mktime(struct tm *timeptr);
```

函数 mktime() 将一个本地时区的分解时间翻译为 time_t 值，并将其作为函数结果返回。

mktime()在进行转换时会对时区进行设置。此外，DST 设置的使用与否取决于输入字段 tm_isdst 的值。 

- 若 tm_isdst 为 0，则将这一时间视为标准间（即，忽略夏令时，即使实际上每年的这 一时刻处于夏令时阶段）。 
- 若 tm_isdst 大于 0，则将这一时间视为夏令时（即，夏令时生效，即使每年的此时不 处于夏令时阶段）。 
- 若 tm_isdst 小于 0，则试图判定 DTS 在每年的这一时间是否生效。这往往是众望所归的设置。

#### 分解时间和打印时间的转换

```c
#include <time.h>
char *asctime(const struct tm *limeptr);
```

在参数 tm 中提供一个指向分解时间结构的指针，asctime()则会返回一指针，指向经由静态分配的字符串，内含时间，格式则与 ctime ()相同。

> 获取和转换日历时间 time/calendar_time.c

```c
#include <time.h>
size_t strftime(char *outstr,size_t maxsize,const char *format,const struct tm *timeptr);
```

outstr 中返回的字符串按照 format 参数定义的格式做了格式化。Maxsize 参数指定 outstr 的最大长度。strftime()的 format 参数是一字符串，与赋予 printf()的参数相类似。strftime()返回 outstr 所指缓冲区的字节长度，且不包括终止空字节。

> 返回当前时间的字符串的函数 curr_time.c

```c
#define _XOPEN_SOURCE
#include <time.h>
char *strptime(const char *str,const char *format,struct tm *timeptr);
```

函数strptime()按照参数format内的格式要求，对由日期和时间组成的字符串str加以解析， 并将转换后的分解时间置于指针 timeptr 所指向的结构体中。如果成功，strptime()返回一指针，指向 str 中下一个未经处理的字符。出错为NULL

> 获取和转换日历时间 time/strtime.c

### 时区

#### 时区定义

在目录/usr/share/zoneinfo中，每个文件都包含一个特定国家或地区的相关信息

系统的本地时间由时区文件/etc/localtime 定义

#### 为程序指定时区

TZ环境变量设置为由一冒号(:)和时区名称组成的字符串。设置时区会自动影响到函数 ctime()、 localtime()、mktime()和 strftime()。上述函数都会调用 tzset(3)，对3个全局变量进行初始化。

> 演示时区和地区的效果 time/show_time.c

### 地区

#### 地区定义

地区信息维护于/usr/share/local之下的目录

命名约定为 language[_territory[.codeeset]] [@modifier]，例如de_DE.utf-8@euro

命令locale -a将列出系统上定义的整套地区

#### 为程序设置地区

```c
#include <locale.h>
char *setlocale(int category,const char *locale);
```

函数 setlocale()既可设置也可查询程序的当前地区。

category 参数选择设置或查询地区的哪一部分，更多使用LC_ALL来指定设置地区的所有部分的值

locale 参数可能是一个字符串，指定系统上已定义的一个地区如 de_DE 或 en_US，空字符串从环境变量获取设置。

如果只需要查看地区设置不改变它，可以指定locale参数为NULL。

例如setlocale(LC_ALL," ");

### 更新系统时钟

这些接口很少被应用程序使用。

```c
#define _BSD_SOURCE
#include <sys/time.h>
int settimeofday(const struct timeval *tv,const struct timezone *tz);
```

tz以及被废弃，指定为NULL

```c
#define _BSD_SOURCE
#include <sys/time.h>
int adjtime(struct timeval *delta,struct timeval *olddelta);
```

当对时间做微小调整时（几秒钟误差），通常是推荐使用库函数 adjtime()， 它将系统时钟逐步调整到正确的时间。

delta 参数指向一个 timeval 结构体，指定需要改变时间的秒和微秒数。如果这个值是正数，那么每秒系统时间都会额外拨快一点点，直到增加完所需的时间。如果 delta 值为负时，时钟以类似的方式减慢。

### 软件时钟（jiffies）

系统软件时钟，根据时钟频率不同，对应的jiffy也不同。

### 进程时间

- 用户CPU时间：在用户模式下执行所花费的时间数量，又称作虚拟时间。
- 系统CPU时间：是在内核模式中执行所花费的时间数量。内核执行系统调用的时间。

进程时间是指处理过程中所消耗的总 CPU 时间

```c
#include <sys/time.h>
clock_t times(struct tms *buf);
```

系统调用 times()，检索进程时间信息，并把结果通过 buf 指向的结构体返回。

buf指向的结构体

```c
struct tms{
	clock_t tms_utime;	/* User CPU time used by caller */
	clock_t tms_stime;	/* System CPU time used by caller */
	clock_t tms_cutime;	/* User CPu time of all (waited for) children */
	clock_T tms_cstime;	/* System CPU time of all (wait for) children */
}
```

tms 结构体的前两个字段返回调用进程到目前为止使用的用户和系统组件的 CPU 时间。 最后两个字段返回的信息是：父进程（比如，times()的调用者）执行了系统调用 wait()的所有已经终止的子进程使用的 CPU 时间。

```c
#include <time.h>
clock_t clock(void);
```

它返回一个值描述了调用进程使 用的总的 CPU 时间（包括用户和系统）。

> 获取进程 CPU 时间 time/process_time.c

## Chapter11 系统限制和选项

### 系统限制

最小值定义为<limits.h>文件中的常量，其命名则冠以字符串\_POSIX\_，而且 （通常）还包含字符串\_MAX，因此，常量命名形如_POSIX_XXX_MAX

每个限制都有一个名称，与上述最小值的名称相对应，但缺少了_POSIX_前缀。某个实现可以在文件中以该名称定义一个常量，用以表示该实现的相应限制。若已然定义， 则该限制值总是至少等同于前述最大值（即 XXX_MAX >= _POSIX_XXX_MAX）。

#### 运行时恒定值

所谓运行时恒定值是指某一限制，若已然在文件中定义，则对于实现而言固定不变。然而该值可能是不确定的（因为该值可能依赖于可用的内存空间），因而在文 件中会忽略对其定义。在这种情况下（即使在文件中已然定义了该限制），应用程序 可以使用 sysconf()来获取运行时的值。

获取消息队列优先级限制：`lim = sysconf(_SC_MQ_PRIO_MAX);`

#### 路径变量值

在限制可能因路径名而发生变化的情况下，应用程序可以使用 pathconf()或 fpathconf()来获取该值。

NAME_MAX 限制是路径名变量值的例子之一。此限制定义了在一个特定文件系统中文件名的最大长度。

获取指定路径最大文件名长度：`lim = pathconf(directory_path,_PC_NAME_MAX);`

#### 运行时可增加值

运行时可增加值是指某一限制，相对于特定实现其值固定，且运行此实现的所有系统至少都应支持这一最小值。然而，特定系统在运行时可能会增加该值，应用程序可以使用 sysconf() 来获得系统所支持的实际值。

#### 限制总结

![image-20230317095431864](./Linux开发.assets/image-20230317095431864.png)

#### 在shell中获取限制：getconf

getconf variable-name [pathname]

### 在运行时获取系统限制

```c
#include <unistd.h>
long sysconf(int name);
```

参数 name 应为定义于文件中的_SC_系列常量之一，返回值为限制值。

当函数无法确定限制或者发生错误时都会返回-1，所以区分上述两种情况需要在调用函数前将errno置为0。若调用后errno不为0，则发生了错误。

> 使用sysconf()函数 syslim/t_sysconf.c

### 运行时获取与文件相关的限制

```c
#include <unistd.h>
long pathconf(const char *pathname,int name);
long fpathconf(int fd,int name);
```

pathconf()和 fpathconf()之间唯一的区别在于对文件或目录的指定方式。pathconf()采用路径名方式来指定，而 fpathconf()则使用（之前已经打开的）文件描述符。

> 使用 fpathconf()函数 syslim/t_fpathconf.c

## Chapter12 系统和进程信息

### /proc文件系统

许多现代 UNIX 实现提供了一个/proc 虚拟文 件系统。该文件系统驻留于/proc 目录中，包含了各种用于展示内核信息的文件，并且允许进 程通过常规文件 I/O 系统调用来方便地读取，有时还可以修改这些信息。

#### 获取与进程有关的信息：/proc/PID

在此目录中的各种文件和子目录包含了进程的相关信息。

通过查看/proc/1 目录下的文件，可以获取 init 进程的信息，该进程的 ID 总是为 1。

#### 线程：/proc/PID/task 目录

因为线程组中的一些属性对于线程而言是唯一的，所以 Linux 2.4 在/proc/PID 目录下增加了一个 task 子目录。针对进程中的每个线程，内核提供了以/proc/PID/task/TID 命名的子目录，其中 TID 是该线程的线程 ID。

#### /proc目录下的系统信息

![image-20230318142504906](./Linux开发.assets/image-20230318142504906.png)

#### 访问/proc文件

shell命令

```shell
echo 100000 > /proc/sys/kernel/pid.max
cat /proc/sys/kernel/pid.max
100000
```

#### 访问/proc/PID目录中的文件

> 访问/proc/sys/kernel/pid_max 文件 sysinfo/procfs_pidmax.c

**%.*s 用途**

在printf中使用,\*表示用后面的形参替代\*的位置，实现动态格式输出。

例如：
printf("%*s", 10, s);
//意思是输出字符串s，但至少占10个位置，不足的在字符串s左边补空格，这里等同于printf("%10s", s);

### 系统标识：uname()

```c
#include <sys/utsname.h>
int uname(struct utsname *utsbuf);
```

uname()系统调用返回了一系列关于主机系统的标识信息，存储于 utsbuf 所指向的结构中。

utsbuf 参数是一个指向 utsname 结构的指针，其定义如下:

```c
struct utsname
{
    /* Name of the implementation of the operating system.  */
    char sysname[_UTSNAME_SYSNAME_LENGTH];

    /* Name of this node on the network.  */
    char nodename[_UTSNAME_NODENAME_LENGTH];

    /* Current release level of this implementation.  */
    char release[_UTSNAME_RELEASE_LENGTH];
    /* Current version level of this release.  */
    char version[_UTSNAME_VERSION_LENGTH];

    /* Name of the hardware type the system is running on.  */
    char machine[_UTSNAME_MACHINE_LENGTH];
    
    /* Name of the domain of this node on the network.  */
# ifdef __GNU_SOURCE
    char domainname[_UTSNAME_DOMAIN_LENGTH];
};
```

> 使用 uname() sysinfo/t_uname.c

## Chapter13 文件I/O缓冲

### 文件I/O的内核缓冲

read()和write()系统调用在操作磁盘文件时不会直接发起磁盘访问，而是仅仅在用户空间 缓冲区与内核缓冲区高速缓存（kernel buffer cache）之间复制数据。

写入时，先写入缓冲区，在后续的某个时刻写入磁盘。读取时，内核会读取到内核缓冲区，直到缓冲区读取完，再将文件下一段读入缓冲区。

#### 缓冲区大小对 I/O 系统调用性能的影响

如果与文件发生大量的数据传输，通过采用大块空间缓冲数据，以及执行更少的系统调用，可以极大地提高 I/O 性能。

采用大缓冲区时的耗时绝大部 分花在了对磁盘的读取上。

若强制在数据传输到磁盘前阻塞输出操作，则调用 write()所需的时间会显著上升。

### stdio库的缓冲

当操作磁盘文件时，缓冲大块数据以减少系统调用，C 语言函数库的 I/O 函数（比如， fprintf()、fscanf()、fgets()、fputs()、fputc()、fgetc()）正是这么做的。

#### 设置一个 stdio 流的缓冲模式

```c
#include <stdio.h>
int setvbuf(FILE *stream,char *buf,int mode,size_t size);
```

setvbuf()出错返回非0值。

**参数 stream** 标识将要修改哪个文件流的缓冲。打开流后，必须在调用任何其他 stdio 函数之前先调用 setvbuf()。setvbuf()调用将影响后续在指定流上进行的所有 stdio 操作。

**参数buf**：

- 如果参数 buf 不为 NULL，那么其指向 size 大小的内存块以作为 stream 的缓冲区。因 为 stdio 库将要使用 buf 指向的缓冲区，所以应该以动态或静态在堆中为该缓冲区分配 一块空间(使用 malloc()或类似函数)，而不应是分配在栈上的函数本地变量。否则，函 数返回时将销毁其栈帧，从而导致混乱。
- 若 buf 为 NULL，那么 stdio 库会为 stream 自动分配一个缓冲区（除非选择非缓冲的 I/O，如下所述）。SUSv3 允许，但不强制要求库实现使用 size 来确定其缓冲区的大小。 glibc 实现会在该场景下忽略 size 参数。

**参数mode**：

- _IONBF 不缓冲，stdio库函数直接调用write()或者 read()。
- _IOLBF 行缓冲，对于输出流，在输出一个换行符 （除非缓冲区已经填满）前将缓冲数据。对于输入流，每次读取一行数据。
- _IOFBF 全缓冲，单次读、写数据（通过 read()或 write()系统调用）的大小与缓冲区相同。 指代磁盘的流默认采用此模式。

```c
#include <stdio.h>
void setbuf(FILE *stream,char *buf);
```

setbuf(fp,buf)调用除了不返回函数结果外，就相当于：

`setvbuf(fp,buf,(buf != NULL)?_IOFBF:_IONBF,BUFSIZ);`

要么将参数 buf 指定为 NULL 以表示无缓冲，要么指向由调用者分配的 BUFSIZ 个字节大小的缓冲区。

(BUFSIZ位于<stdio.h>中，通常数值为8192)

```c
#define _BSD_SOURCE
#include <stdio.h>
void setbuffer(FILE *stream,char *buf,size_t size);
```

对 setbuffer(fp,buf,size)的调用相当于如下调用：

`setvbuf(fp,buf,(buf != NULL)?_IOFBF:_IONBF,size);`

#### 刷新stdio缓冲区

```c
#include <stdio.h>
int fflush(FILE *stream);
```

无论当前采用何种缓冲区模式，在任何时候，都可以使用 fflush()库函数强制将 stdio 输出流中的数据（即通过 write()）刷新到内核缓冲区中。此函数会刷新指定 stream 的输出缓冲区。

若参数 stream 为 NULL，则 fflush()将刷新所有的 stdio 缓冲区。

### 控制文件I/O的内核缓冲

SUSv3 定义的第一种同步 I/O 完成类型是 synchronized I/O data integrity completion，旨在确保针对文件的一次更新传递了足够的信息（到磁盘），以便于之后对数据的获取。

Synchronized I/O file integrity completion 是 SUSv3 定义的另一种同步 I/O 完成，也是上述 synchronized I/O data integrity completion 的超集。该 I/O 完成模式的区别在于在对文件的一次 更新过程中，要将所有发生更新的文件元数据都传递到磁盘上，即使有些在后续对文件数据 的读操作中并不需要。

#### 用于控制文件 I/O 内核缓冲的系统调用

```c
#include <unistd.h>
int fsync(int fd);
```

fsync()系统调用将使缓冲数据和与打开文件描述符 fd 相关的所有元数据都刷新到磁盘 上。调用 fsync()会强制使文件处于 Synchronized I/O file integrity completion 状态。

```c
#include <unistd.h>
int fdatasync(int fd);
```

fdatasync()系统调用的运作类似于 fsync()，只是强制文件处于 synchronized I/O data  integrity completion 的状态。

fdatasync()可能会减少对磁盘操作的次数，由 fsync()调用请求的两次变为一次。例如，若修改了文件数据，而文件大小不变，那么调用 fdatasync()只强制进行了数据更新。（前面已然述及，针对 synchronized I/O data completion 状态，如果是诸如最近修改时间戳之类的元数据属性发生了变化，那么是无需传递到磁盘的。）相比之下，fsync()调用会强制将元数据传递到磁盘上。

```c
#include <unistd.h>
void sync(void);
```

sync()系统调用会使包含更新文件信息的所有内核缓冲区（即数据块、指针块、元数据等）刷新到磁盘上

#### 使所有写入同步：O_SYNC

为open函数中的标志，则会使所有后续输出同步。每个 write()调用会自动将文件数据和元数据刷新到磁盘上（即，按照 Synchronized I/O file integrity completion 的要求执行写操作）

#### O_SYNC 对性能的影响

采用 O_SYNC 标志（或者频繁调用 fsync()、fdatasync()或 sync()）对性能的影响极大。

### 就 I/O 模式向内核提出建议

```c
#define _XOPEN_SOURCE 600
#include <fcntl.h>
int posix_fadvise(int fd,off_t offset,off_t len,int advice);
```

posix_fadvise()系统调用允许进程就自身访问文件数据时可能采取的模式通知内核。

参数 fd 所指为一文件描述符，调用期望通知内核进程对 fd 指代文件的访问模式。

参数 offset 和 len 确定了建议所适用的文件区域。offset 指定了区域起始的偏移量，len 指定了区域 的大小（以字节数为单位）。len 为 0 表示从 offset 开始，直至文件结尾。

参数advice：

- POSIX_FADV_NORMAL 默认行为
- POSIX_FADV_SEQUENTIAL 进程预计会从低偏移量到高偏移量顺序读取数据。在 Linux 中，该操作将文件预读窗口大 小置为默认值的两倍。
- POSIX_FADV_RANDOM 进程预计以随机顺序访问数据。在 Linux 中，该选项会禁用文件预读。
- POSIX_FADV_WILLNEED 进程预计会在不久的将来访问指定的文件区域。内核将由 offset 和 len 指定区域的文件数 据预先填充到缓冲区高速缓存中。
- POSIX_FADV_DONTNEED 进程预计在不久的将来将不会访问指定的文件区域。这一操作给内核的建议是释放相关的 高速缓存页面
- POSIX_FADV_NOREUSE 进程预计会一次性地访问指定文件区域，不再复用。在 Linux 中，该操作目前不起作用。

### 绕过缓冲区高速缓存：直接 I/O 

可针对一个单独文件或块设备（比如，一块磁盘）执行直接 I/O。要做到这点，需要在调用 open()打开文件或设备时指定 O_DIRECT 标志。

#### 直接 I/O 的对齐限制

遵守限制：

- 用于传递数据的缓冲区，其内存边界必须对齐为块大小的整数倍。
- 数据传输的开始点，亦即文件和设备的偏移量，必须是块大小的整数倍。
- 待传递数据的长度必须是块大小的整数倍。

> 使用 O_DIRECT 跳过缓冲区高速缓存 filebuff/direct_read.c

### 混合使用库函数和系统调用进行文件 I/O

```c
#include <stdio.h>
int fileno(FILE *stream);
FILE *fdopen(int fd,const char *mode);
```

给定一个（文件）流，fileno()函数将返回相应的文件描述符（即 stdio 库在该流上已经打开的文件描述符）。随即可以在诸如 read()、write()、dup()和 fcntl()之类的 I/O 系统调用中正常使用该文件描述符。

fdopen()函数与 fileno()函数的功能相反。给定一个文件描述符，该函数将创建了一个使用该描述符进行文件 I/O 的相应流。mode 参数与 fopen()函数中 mode 参数含义相同。例如，r 为读，w 为写，a 为追加。若该参数与文件描述符 fd 的访问模式不一致，则对 fdopen()的调用将失败。
