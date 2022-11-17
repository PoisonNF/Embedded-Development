# 软件相关

## Keil可能遇到的问题

**Keil修改背景色**

> [Keil MDK背景颜色配置，实现仿Sublime效果，并且调用外部编辑器vscode - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/108523090)

> [(4条消息) MDK Keil配色方案及配置方法_荻夜的博客-CSDN博客_mdk 配色](https://blog.csdn.net/u012121390/article/details/111352819?spm=1001.2101.3001.6650.8&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-8-111352819-blog-78342711.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-8-111352819-blog-78342711.pc_relevant_aa&utm_relevant_index=13)

**STM32 Keil新建工程报错Loading PDSC Debug Description Failed for STMicroelectronics STM32Lxxxxxxx**

> [(4条消息) STM32 Keil新建工程报错“Loading PDSC Debug Description Failed for STMicroelectronics STM32Lxxxxxxx”_他乡&学子的博客-CSDN博客_loading pdsc](https://blog.csdn.net/weixin_40779546/article/details/81940587)

**STM32下载程序时提示“active write protected stm32 device detected this could”**

> [(4条消息) STM32下载程序时提示“active write protected stm32 device detected this could”_qq_888192的博客-CSDN博客](https://blog.csdn.net/qq_34447192/article/details/100582441)

***

## STM32CubeIDE可能遇到的问题

**STM32CubeIDE主题设置与汉化**

1.  help->Eclipse Market->输入"Devstyle"查找主题插件->install安装
2.  窗口->首选项 ->常规->外观 选择喜欢的主题
3.  设置完成后会提示重启，重启后完成xiu

> [(4条消息) 【分享】STM32CubeIDE汉化教程 直接在线安装或下载语言包安装 稳定可靠！_像河与海ywr的博客-CSDN博客_stm32cubeide汉化](https://blog.csdn.net/m0_46681107/article/details/119478800)

**STM32CubeIDE自动补全**

> [(4条消息) STM32CubeIDE 1.9.0 代码自动补全_藤一泓的博客-CSDN博客_cubeide自动补全](https://blog.csdn.net/qq_45507036/article/details/123133648)

**STM32添加.c和.h文件**

> [(3条消息) stm32CubeIDE 在自己工程中添加.c 和.h文件_超级网吧的博客-CSDN博客_stm32cubeide添加文件](https://blog.csdn.net/qq_36300069/article/details/103226568)

**STM32CubeIDE输入输出浮点数**

在project -> properties -> C/C++ build -> Setting -> MCU Setting -> 勾选`use float with printf from newlib-nano(-u _printf float)`和`use float with scanf from newlib-nano(-u _scanf float)`

**STM32CubeIDE调整优化等级**

芯片flash如果太小，则可以project -> properties -> C/C++ build -> Setting -> MCU GCC Compiler -> Optimization 

Optimization level选择 `optimize most (-o3)`

***
## VS Code搭环境

> [(3条消息) 一个超级好用的插件—EIDE，在VSCODE下快速创建ARM工程_小麦大叔的博客-CSDN博客](https://great.blog.csdn.net/article/details/119067457?spm=1001.2014.3001.5506)

> [(3条消息) 如何利用VScode打造优雅的STM32开发环境(超详细，新手向)——Keil Assistant插件_WZH灬的博客-CSDN博客](https://blog.csdn.net/weixin_43395116/article/details/114238722?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522166609830516782412548841%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=166609830516782412548841&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~top_click~default-1-114238722-null-null.nonecase&utm_term=keil%20assistant&spm=1018.2226.3001.4450)

***
# 常见问题

- 功能配置中的结构体名要与具体操作中的结构体名保持一致，不然就会无效，详见MDK\_IIC\_EEPROM与MDK\_SPI\_FLASH。

- 若使用printf重定义时，显示FILE未定义，则需要引用`<stdio.h>`，同时打开`use microLIB`。重定义代码见UART

- `_weak` 修饰的函数是弱函数，用户可以在用户文件中重新定义一个同名函数，最终编译器编译的时候，会选择用户定义的函数

- 注意要在LED点亮之前，对所有LED进行==熄灭==操作，不然所有都是点亮状态。

- 如果STM32 调试停留在 `LDR R0, =SystemInit` ,不能运行。原因是在程序中使用了C语言的标准库，例如stdio.h。解决方法是在魔术棒中开启`use microLIB`重映射问题，需要开启AFIO和相对应外设的重映射函数。具体实例参照

> [(4条消息) rtthread 重映射引脚输出pwm 注意事项\_arno-1104的博客-CSDN博客](https://blog.csdn.net/spu20134823091/article/details/123362385)

- ==Scanf不能放入提示字符串==，不然会无效，切记，只能写格式符。

- ->和.的区别：
    -> 的左边是结构体指针   .的左边是结构体名称。
    `a -> b == (\*a).b`

- C 语言不允许返回一个完整的数组作为函数的参数。但是，可以通过指定不带索引的数组名来返回一个指向数组的指针。例如在crucis中的水平航向计算函数。使用时需设定指针变量。

```c
/**
 * @brief 水平航向计算
 * @param Pro 总推进力    P的范围为[0,560]    经测试最大占空比为8.9(1780hz)，最小占空比为6(1200hz)
 * @param θ 与X轴的夹角 θ为[0,2*pi)
 * @retval Pout 存放四个推进器的分量数组的首地址
*/
float *Task_Horizontal_Heading_Calculation(uint16_t Pro,float θ)
{
    static float Pout[4];
    if(0 < Pro <= 560)
    {
        Pout[0] = 0.5*Pro*cos(θ+PI/4);     //P1的推进力
        Pout[1] = 0.5*Pro*cos(θ-5*PI/4);    //P2的推进力
        //Pout[2] = 0.5*Pro*cos((θ-135)*pi/180);    //P3的推进力
        //Pout[3] = 0.5*Pro*cos((θ-45)*pi/180);     //P4的推进力
        Pout[2] = -Pout[0];
        Pout[3] = -Pout[1];
    }
    return Pout;
}
```

- C 不支持在函数外返回局部变量的地址，除非定义数组（局部变量）为 `static`变量。

- Code是代码占用的空间:

RO-data是 Read Only 只读常量的大小，如const型;

RW-data是（Read Write） 初始化了的可读写变量的大小;

ZI-data是（Zero Initialize） 没有初始化的可读写变量的大小。

ZI-data不会被算做代码里因为不会被初始化;

在烧写的时候是FLASH中的被占用的空间为：     Code + RO Data + RW Data

程序运行的时候，芯片内部RAM使用的空间为：               RW Data + ZI Data

***
# GPIO

- 在HAL库中所有外设的硬件初始化函数名`HAL_XXX_MspInit(xxx)`;可以在对应的XXX_Init(  )中找到。
- LED使用输出模式，按键使用输入模式。
- 可以使用`flag = （HAL_GPIO_ReadPin（GPIOx,GPIO_PIN_x）==GPIO_PIN_RESET）?0:1; //按下为0，松开为1` <font color =red>（仅针对正点原子的开发板）</font>
- 如果需要重映射I/O，需要先使用复用时钟`__HAL_RCC_AFIO_CLK_ENABLE( )`，对应的重映射函数可以在`stm32f1xx_hal_gpio_ex.h`查找。

## 一般GPIO初始化步骤

1.开启对应外设的时钟
2.构造结构体，为结构体成员赋值
3.调用库函数初始化函数，初始化I/O

> <font color=red>Tips:</font> 在GPIO模式设置为输出模式时，必须设置Speed；设置为输入模式时，必须设置Pull。

**举例说明**：
```c
__HAL_RCC_GPIOx_CLK_ENABLE( ); //使能哪个分区的I/O

GPIO_InitTypeDef GPIOINIT;
GPIOINIT.Mode = GPIO_MODE_OUTPUT_PP;//推挽输出
GPIOINIT.Pin = GPIO_PIN_x;//选择x号
GPIOINIT.Speed = GPIO_SPEED_FREQ_HIGH;//高速50mhz
HAL_GPIO_Init(GPIOx,&GPIOINIT); //调用hal库初始化函数，对GPIOx进行初始化

HAL_GPIO_WritePin(GPIOx,GPIO_PIN_x,GPIO_PIN_SET); //拉低电平
```

***
# RCC

## 时钟配置示例

```c
static void S_HAL_CLKConfig(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};
  /** Initializes the CPU, AHB and APB busses clocks
  */
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
  RCC_OscInitStruct.HSEState = RCC_HSE_ON;
  RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
  RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;
    
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    while(1);
  }

  /** Initializes the CPU, AHB and APB busses clocks
  */
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
			                   |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK)
  {
    while(1);
  }
}
```

其中`HAL_RCC_ClockConfig( )`函数中关于输入参数FLatency的选择
- 0 < SYSCLK <= 24      选择0WS(1CPU cycle)     <font style="background:green">FLASH_LATENCY_0</font>
- 24 < SYSCLK <= 48    选择1WS(2CPU cycle)     <font style="background:green">FLASH_LATENCY_1</font>
- 48 < SYSCLK <= 72    选择2WS(3CPU cycle)     <font style="background:green">FLASH_LATENCY_2</font>

***
# NVIC

- NVIC相关的库函数声明在`stm32f1xx_hal_cortex.h`
- NVIC中断向量表在启动文件`startup_stm32f103xe.h`
- 设置中断优先级可选参数`IRQn_Type`在`stm32f103xe.h`
> <font color =red>Tips:</font> 根据使用芯片库不同，文件名会有些许变化。

## 设置中断优先级相关的函数

1.`HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_x)` 
设置优先级分组，根据参数`NVIC_PRIORITYGROUP_x`确定主优先级位数，最高为4位

2.`HAL_NVIC_SetPriority(IRQn_Type, 0, 0)`
设置中断优先级，第二参数为主优先级，第三参数为次优先级，数字越小优先级越高

3.`HAL_NVIC_EnableIRQ(IRQn_Type)`
使能中断优先级

## 常用中断写法

### 外部中断

外部中断多用于按键

```c
void EXTIx_IRQHandler(void)
{
   HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_x); 
} //固定写法，交给HAL库处理

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	//写中断发生时需要完成的事件  
}
```
> <font color =red>Tips:</font> EXTIx中x的取值与GPIO_PIN_x相关，具体看中断向量表

### 其他中断

例如串口、定时器、SPI等

```c
void xxxx_IRQHandler(void)
{
   HAL_xxxx_IRQHandler(&对应句柄); 
} //固定写法，交给HAL库处理

/*回调函数可以没有*/
void HAL_xxxx_Callback(&对应句柄)
{
	//写中断发生时需要完成的事件  
}
```
***

# OLED

不同的OLED驱动方式也不相同，根据引脚数量，有8080并口、SPI、IIC。

## 8080并口模式

- 8080的驱动可以参考正点原子的代码。并且正点原子的OLED在改变工作模式时需要用烙铁更改模块背后的BS1和BS2。

- 按照正点原子的写法，只输入数字和英文时：

	1.字号12 可以显示21列	5行
	2.字号16 可以显示16列	4行
	3.字号24 可以显示10列	2行

![](STM32开发.assets/image-20221115114334077.png)



| 接口方式 | 4线SPI | IIC  | 8位6800 | 8位8080 |
| :------: | :----: | :--: | :-----: | :-----: |
|   BS1    |   0    |  1   |    0    |    1    |
|   BS2    |   0    |  0   |    1    |    1    |

> [(3条消息) OLED显示屏驱动：8080并口，IIC，SPI三种驱动方式_tutu-hu的博客-CSDN博客_8080并口屏](https://blog.csdn.net/weixin_42700740/article/details/94380147)


## SPI模式

### 硬件SPI
> [(5条消息) STM32 硬件 SPI 驱动 0.96 寸 的 OLED_霁风AI的博客-CSDN博客](https://blog.csdn.net/wwt18811707971/article/details/78681547)

### 软件SPI

> [(5条消息) 一步步学习0.96寸七针OLED显示屏SPI接口驱动(附移植代码)--基于STM32_hhh_little_hu的博客-CSDN博客_7针oled显示](https://blog.csdn.net/weixin_40134414/article/details/105308234)

## IIC模式

### 硬件IIC

> [STM32_HAL_IIC_SSD1306: 基于HAL库的SSD1306驱动 (gitee.com)](https://gitee.com/snitro/stm32_hal_iic_ssd1306)

> <font color = red>Tips:</font>一般不使用STM32的硬件IIC，一是有BUG，二是不如软件IIC方便，任意两个I/O口即可使用。

### 软件IIC

> [(4条消息) STM32之0.96寸 4针 OLED显示屏（IIC协议）驱动代码（程序稳定，清晰明了）_xiaohai@Linux的博客-CSDN博客](https://blog.csdn.net/qq_34885669/article/details/89847349)

> [(4条消息) 四针OLED驱动代码——IIC驱动_新时代城市农民工的博客-CSDN博客_oled驱动代码](https://blog.csdn.net/weixin_46199479/article/details/124454194)

## 其他问题

1.如何让屏幕显示浮点数？

> [(3条消息) STM32 OLED_I2C动态显示 float类型（浮点型）数据_Guard_Byte的博客-CSDN博客_oled显示浮点数stm32](https://blog.csdn.net/Guard_Byte/article/details/123675709)

***

# UART

## 串口初始化结构体

```c
typedef struct {
    uint32_t BaudRate;            //波特率
    uint32_t WordLength;          //字长
    uint32_t StopBits;            //停止位
    uint32_t Parity;              //校验位
    uint32_t Mode;                //UART模式
    uint32_t HwFlowCtl;           //硬件流控制
    uint32_t OverSampling;        // 过采样模式
    uint32_t CLKLastBit;          // 最尾位时钟脉冲
} USART_InitTypeDef;
```

1.BaudRate：波特率设置。一般设置为2400、9600、19200、115200。HAL库函数会根据设定值计算得到UARTDIV值，见公式 20‑1，并设置UART_BRR寄存器值。

2.WordLength：数据帧字长，可选8位或9位。它设定UART_CR1寄存器的M位的值。 如果没有使能奇偶校验控制，一般使用8数据位；如果使能了奇偶校验则一般设置为9数据位。

3.StopBits：停止位设置，可选0.5个、1个、1.5个和2个停止位， 它设定USART_CR2寄存器的STOP[1:0]位的值，一般我们选择1个停止位。

4.Parity：奇偶校验控制选择，可选USART_PARITY_NONE (无校验)、USART_PARITY_EVEN (偶校验)以及USART_PARITY_ODD (奇校验)，它设定UART_CR1寄存器的PCE位和PS位的值。

5.Mode：UART模式选择，有USART_MODE_RX和USART_MODE_TX， 允许使用逻辑或运算选择两个，它设定USART_CR1寄存器的RE位和TE位。

## 串口工作模式初始化示例

```c
__HAL_RCC_USART1_CLK_ENABLE();
UART_HandleTypeDef huart1;

huart1.Instance = USART1;
huart1.Init.BaudRate = 115200;
huart1.Init.WordLength = UART_WORDLENGTH_8B;
huart1.Init.StopBits = UART_STOPBITS_1;
huart1.Init.Parity = UART_PARITY_NONE;
huart1.Init.Mode = UART_MODE_TX_RX;
huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
huart1.Init.OverSampling = UART_OVERSAMPLING_16;
if (HAL_UART_Init(&huart1) != HAL_OK)
{
	Error_Handler();
}

//如果需要中断，添加下述代码（这里是以UART1中断举例）
HAL_NVIC_SetPriority(USART1_IRQn, 0, 0);
HAL_NVIC_EnableIRQ(USART1_IRQn);
  
//如果需要DMA，添加下述代码（这里是以UART1的发送举例）
DMA_HandleTypeDef hdma_usart1_tx;
  
hdma_usart1_tx.Instance = DMA1_Channel4;
hdma_usart1_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
hdma_usart1_tx.Init.PeriphInc = DMA_PINC_DISABLE;
hdma_usart1_tx.Init.MemInc = DMA_MINC_ENABLE;
hdma_usart1_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
hdma_usart1_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
hdma_usart1_tx.Init.Mode = DMA_NORMAL;
hdma_usart1_tx.Init.Priority = DMA_PRIORITY_LOW;
if (HAL_DMA_Init(&hdma_usart1_tx) != HAL_OK)
{
	Error_Handler();
}
__HAL_LINKDMA(uartHandle,hdmatx,hdma_usart1_tx); //链接两个设备，不能遗漏

//DMA也有中断优先级
HAL_NVIC_SetPriority(DMA1_Channel5_IRQn,0，0);
HAL_NVIC_EnableIRQ(DMA1_Channel5_IRQn);
```

> <font color=red>Tips:</font>
>
> 1.这里并没有将串口GPIO的初始化列出来，在实际中一定要初始化GPIO。
>
> 2.不要忘记开始对应时钟和链接设备。
>
> 3.UART 数据帧格式是低位先发。

## Printf重定向

串口输出函数一般使用HAL_UART_Transmit( ),但是这样调试不够方便，在标准C语言中，我们一般使用printf( )函数，因此需要些许调整，以使用该函数。

### Keil重定向方法

```c
extern UART_HandleTypeDef huart1;//根据需要修改huart,此处为UART1
//定义变量存储用户数据
uint8_t ch;
uint8_t ch_r;
//重写这个函数,重定向printf函数到串口
int fputc(int c, FILE * f)
{
   ch = c;//存储一个数据
   HAL_UART_Transmit(&huart1,&ch,1,1000);//通过串口发送这一个数据(超时时间1000毫秒)
   return c;//发送完成后返回这个数据
}

 //重定向scanf函数到串口 意思就是说接受串口发过来的数据
 int fgetc(FILE * F)
{
   HAL_UART_Receive(&huart1,&ch_r,1,0xffff);//通过串口接收数据(超时时间65535毫秒)
   return ch_r;//返回接收到的数据信息
}
```

> [keil工程当中实现printf重定向串口打印 (shuzhiduo.com)](https://www.shuzhiduo.com/A/QW5YGA73zm/)

### STM32CubeIDE重定向方法

> [在stm32使用printf的两种方法 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/265612712)
>
> [(3条消息) 在STM32中使用printf()和scanf_Monster xn的博客-CSDN博客_stm32scanf](https://blog.csdn.net/weixin_45636061/article/details/117969437)

## 串口接收函数

### 串口中断接收服务函数示例

```c
//使用串口中断时，在服务函数中可以使用中断函数完成嵌套，继续接收下一位或者x位。
Void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
	/*自写处理buffer代码*/
	HAL_UART_Receive_IT(&huart1，buffer，1);
}
```

> <font color=red>Tips:</font>不要忘记先使用HAL_UART_Receive_IT(&huart1，buffer，x)接收一次数据，很容易遗忘。

### 串口DMA中断接收处理函数示例

```c
void USART1_IRQHandler(UART_HandleTypeDef *huart)
{
	if(__HAL_UART_GET_FLAG(&huart1,UART_FLAG_IDLE) != RESET) // 空闲中断标记被置位
	{
		__HAL_UART_CLEAR_IDLEFLAG(&huart1);// 清除中断标记
		HAL_UART_DMAStop(&huart1);// 停止DMA接收
		/* 可以加入计算接收数据长度代码 */
         DMARxCplt = 1;// 标记接收结束
         HAL_UART_Receive_DMA(&huart1,buffer,10);// 重新启动DMA接收
	}
}
```

> <font color=red>Tips:</font>
>
> 1.在主函数中判断接收标志位，如果为1，进行数据分析，并且在结束之后，要将接受标志位重新置0
>
> 2.在SGA库中这段代码也被封装了起来，成为一个通用的API。

## HAL库串口DMA数据收发参考

> [HAL UART DMA 数据收发 - DW039 - 博客园 (cnblogs.com)](https://www.cnblogs.com/dw039/p/11692472.html) 

***

# DMA

## DMA初始化结构体

```c
typedef struct {

    uint32_t Direction;            //传输方向
    uint32_t PeriphInc;            //外设递增
    uint32_t MemInc;               //存储器递增
    uint32_t PeriphDataAlignment;  //外设数据宽度
    uint32_t MemDataAlignment;     //存储器数据宽度
    uint32_t Mode;                 //模式选择
    uint32_t Priority;             //优先级

} DMA_InitTypeDef;
```

1.Direction：传输方向选择，可选外设到存储器、存储器到外设以及存储器到存储器。 它设定DMA_SxCR寄存器的DIR[1:0]位的值。ADC采集显然使用外设到存储器模式。

2.PeripheralInc：如果配置为DMA_PINC_ENABLE，使能外设地址自动递增功能， 它设定DMA_CCR寄存器的PINC位的值；一般外设都是只有一个数据寄存器，所以一般不会使能该位。

3.MemoryInc：如果配置为DMA_MINC_ENABLE，使能存储器地址自动递增功能，它设定DMA_CCR寄存器的MINC位的值； 我们自定义的存储区一般都是存放多个数据的，所以要使能存储器地址自动递增功能。

4.PeriphDataAlignment：外设数据宽度，可选字节(8位)、半字(16位)和字(32位)， 它设定DMA_SxCR寄存器的PSIZE[1:0]位的值。 ADC数据寄存器只有低16位数据有效，使用半字数据宽度。

5.Mode：DMA传输模式选择，可选一次传输或者循环传输，它设定DMA_SxCR寄存器的CIRC位的值。 我们希望ADC采集是持续循环进行的，所以使用循环传输模式。

6.软件设置数据流的优先级，有4个可选优先级分别为非常高、高、中和低，它设定DMA_SxCR寄存器的PL[1:0]位的值。 DMA优先级只有在多个DMA数据流同时使用时才有意义，一般设置为非常高优先级就可以了。

## DMA使用注意事项

1.在UART和DMA的使用中，<font color=red>关联函数`__HAL_LINKDMA()`非常重要，构建数据链路，必不可少</font>

2.UART5没有DMA通道！！！

3.在使用串口DMA传输时，记得打开串口的全局中断。以下两个中断函数（可以在startup中找到相对应的名字）都需要在stm32f1xx_it.c中出现

```c
void DMA1_Channel4_IRQHandler(void)
{
	HAL_DMA_IRQHandler(&hdma_usart1_tx);
}
void USART1_IRQHandler(void)
{
	HAL_UART_IRQHandler(&huart1);
}
```

4.在使用串口DMA时，记得将DMA初始化放在串口初始化之前，DMA的时钟开启必须在USART之前，否则USART是无法使用DMA进行数据接收的。

## DMA外设通道图

