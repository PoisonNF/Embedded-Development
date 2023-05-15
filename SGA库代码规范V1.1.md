# SGA库代码编写规范 V1.1

为了保证SGA库（特指Driver层、OCD层、Dev层、Algo层）的代码风格统一，现指定以下规范，本规范从5个部分来讲解，分别是**文件与目录**、**排版**、**注释**、**变量|结构体|常量|宏**、**函数**。这些规范是C语言比较常见的，并不需要死记硬背，可以参考已经写好的驱动文件进行编写

# 一、文件与目录

1. 所有源文件后缀为小写字母 .c和.h

2. Driver层新建全新驱动时，两份文件都需要添加`#include "drv_hal_conf.h"`,需要在驱动文件中加入对应的条件编译，同时在`drv_hal_conf.h`中加入条件编译

    ```c
    #ifdef DRV_HAL_UART_ENABLE
    #include "drv_hal_uart.h"
    #endif
    ```

    OCD层、Dev层、Algo层新建文件，只需将自身同名的头文件加入到.c文件中即可

3. 头文件引用规则

    - 在引用头文件时，使用 <> 来引用预定义或者特定目录的头文件，一般为C语言标准库使用

    - 使用 " "来引用当前目录或者路径相对于当前目录的头文件，一般为自写的头文件使用

4. 为了防止头文件被重复引用，应当用 ifndef/define/endif 结构产生预处理块

    ```c
    #ifndef __DRV_UART_H_
    #define __DRV_UART_H_
    /* 头文件主体*/
    #endif
    ```

5. 头文件中只包含**结构体**、**宏定义**、**变量声明**、**函数声明**，不能存放函数体

    头文件顺序为 *条件编译 （预处理块）-> 头文件引用 -> 宏定义 -> 变量声明 ->结构体定义 ->函数声明*

# 二、排版

1. 函数或过程的开始、结构的定义及循环、判断等语句中的代码都要采用**缩进**风格，case 语句下的情况处理语句也要遵从语句缩进要求，**缩进空格为4个**，即使用TAB键

2. 在独立的程序块之间、局部变量说明之后==必须==加空行

    ```c
    void DemoFun(void)
    {
    	uint8_t i;
    	<------空一行----->
    	/* 功能块1 */
    	if()
    	{
    		
    	}
    	<------空一行----->
    	/* 功能块2 */
    	{
    		
    	}
    }
    ```

3. <font color=red>不允许</font>多个短语句写在一行，特别是在多个变量命名时

    ```c
    rect.length = 0; rect.width = 0; <---- 不正确的写法
    
    rect.length = 0; <---- 正确的写法
    rect.width = 0;
    ```

4. 程序块的分界符（如大括号‘{’和‘}’ ）应各独占一行并且位于同一列，同时与引用它们的语句左对齐。

    ```c
    for (...) { <---- 不规范的写法
    	 /* program code */
    }
    
    for (...)
    { <---- 规范的写法
    	/* program code1 */
        if(...)
    	{
        	/* program code2 */
    	}
    }
    ```

5. 比较操作符，赋值操作符"="、 "+="，算术操作符"+"、"%"，逻辑操作符"&&"、"&"，位域操作符"<<"、"^"等双目操作符的前后加空格。

    ```c
    if (current_time >= MAX_TIME_VALUE)
    a = b + c;
    a *= 2;
    a = b ^ 2;
    ```

# 三、注释

1. 注释的原则是有助于对程序的阅读理解，在该加的地方都加，注释不宜太多也不能太少，注释语言必须准确、易懂、简洁。源程序有效注释量必须在==20％以上==。

2. 对变量的定义和分支语句（条件分支、循环语句等）==必须==编写注释。对于维护人员来说，良好的注释帮助更好的理解程序，有时甚至优于看设计文档。

3. 注释格式尽量统一，强烈建议使用**/* …… */**，因为 C++注释**//**并不被所有 C 编译器支持。且两个星号和注释之间都需要空一格，例如： /* 注释 */

4. 在文件开始部分需要加入关于文件版权、内容简介、修改历史等项目的说明，并且作出修改后需在此处写上修改内容（<u>版本号可以空着</u>）

    ```c
    /****************************************************************************
    
    * Sigma团队
    
    * 文件名: ocd_jy901.c
    
    * 内容简述：jy901模块文件
    
    * 文件历史：
    
    * 版本号	      日期  	  作者		说明
    * 1.1.8		2022-10-22	鲍程璐		对结构体进行了整合，提供IT模式和DMA模式
    								  提供jy901数据的处理、转换、打印函数
    
    * 1.0.0a 	2020-03-14	李环宇		创建该文件
    
    * 1.1.0a 	2020-03-29	李环宇		修改部分文本
    
    ****************************************************************************/
    ```

5. 在各个函数前==必须==需要添加和函数的实现相关的足够而精练的注释信息。内容包括本函数功能介绍，调用的变量、常量说明，形参说明。

    <font color=red>Tips:</font>函数体所需要的全局变量应放在函数注释上，以免注释无法对函数生效。

    示例如下

    ```c
    const uint8_t s_ucResetBuf[] = {0xFF,0xFF,0xFF,0xFF,0xFF,0xFF,0xFF,0xFF};
    /**
     * @brief PWM初始化函数
     * @param _tPWM-PWM结构体指针
     * @param _ucNum-PWM数量
     * @note 为多个PWM初始化		<---------- note看情况书写
     * @retval Null
    */
    void Drv_PWM_Init(tagPWM_T *_tPWM, uint8_t _ucNum)
    {
    	uint8_t index;
    	
    	for(index = 0;index < _ucNum;index++)
    	{
    		S_PWM_CLKEnable(&_tPWM[index]);
    		S_PWM_GPIOConfig(&_tPWM[index]);
    		S_PWM_PramConfig(&_tPWM[index]);
    	}
    }
    ```

6. 注释应与其描述的代码靠近，对代码的注释应放在其上方或右方（对单条语句的注释）。同时注释与所描述内容进行同样的缩排。

    - **放于上方**的注释需与其上面的代码用空行隔开。

    - **放于右方**的注释需按一个tab后写入。如果有多个单行注释，最好所有右方注释在一条直线上。

          

    ```c
    /* code one comments */
    program code one
    
    /* code two comments */
    program code two
    
    /* code two comments */
    for()
    {
        /* code two comments */
        code
    }
    
    program code three	/* code two comments */
    program code four	/* code two comments */    
    ```

# 四、变量、结构体、常量、宏

## 变量

1. ==变量命名==要清晰、明了，有明确含义，同时使用完整的单词或大家基本可以理解的缩写，避免使人产生误解。

    - temp 可缩写为 tmp;
    - flag 可缩写为 flg;
    - statistic 可缩写为 stat;
    - message 可缩写为 msg;

2. <font color=red>禁止使用</font>类似i、j、k...等单字符变量命名。但 i、j、k 作局部循环变量是允许的，尽量使用index作为循环变量。

3. 命名规范必须与库中其他变量保持相似风格，同时适当大小写。驱动层<font color=red>必须</font>遵循以下命名！作为==函数形参==时需要加上前缀“__”。

    当然也有特殊变量可以使用，例如表示坐标时可以使用x，y。

    

    <font color=red>Tips:</font>遇到没在表格中的类型可以根据规律自行拼接。目前int类型可以不增加前缀。

    |   变量类型    | 前缀 |   举例    |   变量类型   | 前缀 |   举例   |
    | :-----------: | :--: | :-------: | :----------: | :--: | :------: |
    |    uint8_t    |  uc  |   ucSum   |    int8_t    |  c   |   cSum   |
    |   uint16_t    |  us  |   usSum   |   int16_t    |  s   |   sSum   |
    |   uint32_t    |  ul  |   ulSum   |   int32_t    |  l   |   lSum   |
    |   uint8_t *   | ucp  | ucpWrite  |   int8_t *   |  cp  | cpWrite  |
    |  uint16_t *   | usp  | uspWrite  |  int16_t *   |  sp  | spWrite  |
    |  uint32_t *   | ulp  | ulpWrite  |  int32_t *   |  lp  | lpWrite  |
    | uint8_t 数组  | uca  | ucaNum[5] | int8_t 数组  |  ca  | caNum[5] |
    | uint16_t 数组 | usa  | usaNum[5] | int16_t 数组 |  sa  | saNum[5] |
    | uint32_t 数组 | ula  | ulaNum[5] | int32_t 数组 |  la  | laNum[5] |
    |    double     |  d   |   dNum    |    float     |  f   |   fNum   |
    |     bool      |  b   | bTxEnable |  结构体指针  |  t   |  tGPIO   |

    <center><p>变量前缀命名定义表</p></center>

    具体举例:

    `uint8_t ucDMARxPriority;     /* DMA接收中断优先级，0-15 */`

    `uint16_t usRxLenth;      	  /* 接收数据长度 */`

    `void Drv_PMW_FreqSet(tagPWM_T *_tPWM, uint32_t _ulFreq);`

4. 除非必要，不要用数字或较奇怪的字符来定义标识符，使人产生疑惑。

    `uint8_t dat01;`

    `void Set00(uint_8 c);`

5. ==变量作用域的前缀==

    为了清晰的标识变量的作用域，减少发生命名冲突，应该在变量类型前缀之前再加上表示变量作用域的前缀，并在变量类型前缀和变量作用域前缀之间用下划线‘__’隔开。

    具体的规则如下：

    - 对于全局变量（global variable），在其名称前加“**g**”和变量类型符号前缀。

    `uint32_t g_ulParaWord;`

    `uint8_t g_ucByte;`

    - 对于静态变量（static variable），在其名称前加“**s**”和变量类型符号前缀。

    `static uint32_t s_ulParaWord;`

## 结构体

对于==结构体命名==类型，表示类型的名字，所有名字以小写字母“tag”开头，之后每个英文单词的第一个字母大写（包括第一个单词的第一个字母），其他字母小写，结尾_T 标识。

<font color=red>Tips:</font>单词之间不使用下划线分隔。

> 后续结构体句柄调用时，结构体变量尽量以 t 开头。非强制，但是命名要和用途相关。

```c
/* 结构体命名 */
typedef struct
{
...
}tagDMAUart_T;

/* 结构体句柄调用 */
tagDMAUart_T tUSART1;
```

## 常量、宏

1. 对于==枚举定义==全部采用大写，结尾**_E** 标识。

    ```c
    typedef enum
    {
        KB_F1 = 0, /* F1 键代码 */
        KB_F2, 	   /* F2 键代码 */
        KB_F3      /* F3 键代码 */
    }KEY_CODE_E;
    ```

2. ==常量、宏==的名字应该全部大写。如果这些名字由多个单词组成，则单词之间用下划线分隔。

    `#define LOG_BUF_SIZE 8000`

# 五、函数

## 函数名的命名规则

函数名应准确描述函数的功能。避免使用无意义或含义不清的动词为函数命名。

规范如下：

- 对于需要开放接口全局调用的驱动函数，功能分段使用下划线，首字母大写，各个分段首字母大写。在函数名前加上**Drv_**

    例如：

    - `void Drv_Delay_Init(void);`
    - `void Drv_GPIO_Write(tagGPIO_T *_tGPIO, GPIO_PinState _pin);`

- 对于需要开放接口全局调用的第三方外设OCD函数，功能分段使用下划线，首字母大写，各个分段首字母大写。在函数名前加上**OCD_**

    例如：

    - `void OCD_JY901_DataProcess(tagJY901_T *_tJY901);`

- 对于需要开放接口全局调用的设备Dev函数，功能分段使用下划线，首字母大写，各个分段首字母大写。在函数名前加上**Dev_**

    例如：

    - `uint8_t Dev_PS2_DataKey(tagPS2_T *_tPS2);`

- 对于需要开放接口全局调用的算法Algo函数，功能分段使用下划线，首字母大写，各个分段首字母大写。在函数名前加上**Algo_**

    例如：

    - `float Algo_PID_Calculate(tagPID_T *_tPid,float _fCurrValue,float _fExpValue);`

- 对于本地文件调用的局部函数或者静态函数，功能分段使用下划线，首字母大写，各个分段首字母大写。

    在函数名前加上**S_**，也可以加上**static**的修饰符，增加库函数的封装性。

    例如:

    - `static void S_GPIO_CLKEnable(tagGPIO_T *_tGPIO);`

## 函数形参命名规则

**形参都以下划线_开头**，与普通变量进行区分，对于没有形参为空的函数(void)括号紧跟函数后面。

形参前缀按照第四章中的==变量前缀命名定义表==来定义。

`uint32_t UartConvUartBaud(uint32_t _ulBaud);`

## 源码函数体摆放顺序

*（内部）局部函数或者静态函数 -> 外部可调用功能函数 -> 初始化函数 -> 中断调用操作集成函数*

以下是串口DMA举例

```c
/* (内部）局部函数或者静态函数 */
static void S_UART_DMA_CLKEnable(tagUART_T *_tUART)
{
	...
}

/* 外部可调用功能函数 */
void Drv_Uart_Transmit_DMA(tagUART_T *_tUART, uint8_t *_ucpTxData, uint16_t _uspSize)
{
	...
}

/* 初始化函数 */
void Drv_Uart_DMAInit(tagUART_T *_tUART)
{
	...
}

/* 中断调用操作集成函数 */
void Drv_Uart_DMA_RxHandler(tagUART_T *_tUART)
{
	...
}
```

# 六、提交前准备

请先阅读[如何在github上提交PR(Pull Request) - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1999727)（PR教程）

1. 需要在自己的IM板或者开发板上进行测试，<font color=red>测试无误</font>后写入库中。测试的工程可以私发给我作为例程使用。

2. 在Doc文件夹中的**句柄资源示例.txt**中，放入所写入驱动的句柄示例（如果是修复BUG可选）。

3. 在Doc文件夹中的**工程维护记录.txt**中，按照之前的示例，写上日期、作者、说明，同时需要在下方写上具体修改记录。版本号不写，由版主进行统一命名。

4. 检查GitHub上最新的库版本与当前使用的是否一致，不一致需要先拉取一份最新代码。

5. 进行Git操作，在commit时遵循PR规范（结尾不加句号）

    - commit规范如下：
        `<type>(<scope>): <subject>` （冒号后面有个空格）

    - subject可选：

        - feat: 新功能（feature）
        - fix: 修补bug
        - docs: ⽂档（documentation）
        - style: 格式（不影响代码运⾏的变动）
        - refactor: 重构（即不是新增功能，也不是修改 bug 的代码变动）
        - perf: 性能优化
        - test: 增加测试
        - chore: 构建过程或辅助⼯具的变动，例如 requirement 的变动可以放到这⾥

    - commit举例：

        fix: 修复串口速率过慢问题

6. 通过GitHub **Pull Request**的方式提交。
