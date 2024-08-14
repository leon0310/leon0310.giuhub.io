## 1.硬件设计
F103-指南者和F103-霸道开发板中STM32芯片与LED灯的连接见下图。
> F103-指南者和F103-霸道开发板LED硬件原理图
![image113](https://github.com/user-attachments/assets/41725631-3962-4de4-8663-2cef18b9acaf)

这是一个RGB灯，里面由红绿蓝（Red, Green, Blue）三个小灯构成，使用PWM控制时可以混合成256*256*256种不同的颜色。

F103-MINI开发板接的是单色的LED灯， 其中STM32芯片与LED灯的连接分别如下图所示
> F103-MINI开发板的LED硬件原理图
![image22](https://github.com/user-attachments/assets/15501def-4394-47eb-965b-c63adb9b4763)

以上所有的这些LED灯的阴极都是连接到STM32的GPIO引脚，阳极接到3.3V电源，同时每个灯都分别接了一个限流电阻。 只要我们控制GPIO引脚的电平输出状态，即可控制LED灯的亮灭。使用的实验板LED灯的连接方式或引脚不一样的话，只需根据我们的工程修改引脚即可，程序的控制原理相同。

## 2.软件设计
为了使工程更加有条理，我们把LED灯控制相关的代码独立分开存储，方便以后移植。在“工程模板”之上新建“bsp_led.c”及“bsp_led.h”文件，其中的“bsp”即Board Support Packet的缩写(板级支持包)，这些文件也可根据您的喜好命名，这些文件**不属于STM32HAL库的内容**，是由我们自己根据应用需要编写的。

### 2.1代码分析
#### 2.1.1 led灯引脚宏定义
在编写应用程序的过程中，要考虑更改硬件环境的情况，例如LED灯的控制引脚与当前的不一样， 我们希望程序只需要做最小的修改即可在新的环境正常运行。 这个时候一般把硬件相关的部分使用宏来封装，若更改了硬件环境，只修改这些硬件相关的宏即可， 这些定义一般存储在头文件，即本例子中的“bsp_led.h”文件中，见以下代码清单：
>代码清单1：F103-指南者和 F103-霸道开发板 LED控制引脚相关的宏
```c
//引脚定义
 /*******************************************************/
 //R 红色灯
 #define LED1_PIN                  GPIO_PIN_5
 #define LED1_GPIO_PORT            GPIOB
 #define LED1_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOB_CLK_ENABLE()

 //G 绿色灯
 #define LED2_PIN                  GPIO_PIN_0
 #define LED2_GPIO_PORT            GPIOB
 #define LED2_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOB_CLK_ENABLE()

 //B 蓝色灯
 #define LED3_PIN                  GPIO_PIN_1
 #define LED3_GPIO_PORT            GPIOB
 #define LED3_GPIO_CLK_ENABLE()    __HAL_RCC_GPIOB_CLK_ENABLE()
```
>代码清单2：F103-MINI开发板LED控制引脚相关的宏
```c
//引脚定义
/*******************************************************/
//R 红色灯
#define LED1_PIN                  GPIO_PIN_2
#define LED1_GPIO_PORT            GPIOC
#define LED1_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOC_CLK_ENABLE()

//G 绿色灯
#define LED2_PIN                  GPIO_PIN_3
#define LED2_GPIO_PORT            GPIOC
#define LED2_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOC_CLK_ENABLE()
```
以上代码分别把控制LED灯的GPIO端口、GPIO引脚号以及GPIO端口时钟封装起来了。在实际控制的时候我们就直接用这些宏，以达到应用代码跟硬件无关的效果。

其中的GPIO时钟宏“__GPIOF_CLK_ENABLE()”是STM32HAL库定义的GPIO端口时钟相关的宏，它的作用与“GPIO_PIN_x”这类宏类似，是用于指示寄存器位的，方便库函数使用。它们指示GPIOF时钟，下面初始化GPIO时钟的时候可以看到它的用法。

#### 2.1.2 控制LED灯亮灭状态的宏定义
为了方便控制LED灯，我们把LED灯常用的亮、灭及状态反转的控制也直接定义成宏，见以下清单：

>代码清单3：F103-指南者和F103-霸道开发板控制LED亮灭的宏
```c
/** 控制LED灯亮灭的宏，
 * LED低电平亮，设置ON=0，OFF=1
 * 若LED高电平亮，把宏设置成ON=1 ，OFF=0 即可
 */
 #define ON  GPIO_PIN_RESET
 #define OFF GPIO_PIN_SET

 /* 带参宏，可以像内联函数一样使用 */
 #define LED1(a) HAL_GPIO_WritePin(LED1_GPIO_PORT,LED1_PIN,a)
 #define LED2(a) HAL_GPIO_WritePin(LED2_GPIO_PORT,LED2_PIN,a)
 #define LED3(a) HAL_GPIO_WritePin(LED2_GPIO_PORT,LED3_PIN,a)

 /* 直接操作寄存器的方法控制IO */
 #define digitalHi(p,i)      {p->BSRR=i;}        //设置为高电平
 #define digitalLo(p,i)      {p->BSRR=(uint32_t)i << 16;}
                 //输出低电平
 #define digitalToggle(p,i)    {p->ODR ^=i;}     //输出反转状态

 /* 定义控制IO的宏 */
 #define LED1_TOGGLE   digitalToggle(LED1_GPIO_PORT,LED1_PIN)
 #define LED1_OFF    digitalHi(LED1_GPIO_PORT,LED1_PIN)
 #define LED1_ON     digitalLo(LED1_GPIO_PORT,LED1_PIN)

 #define LED2_TOGGLE   digitalToggle(LED2_GPIO_PORT,LED2_PIN)
 #define LED2_OFF    digitalHi(LED2_GPIO_PORT,LED2_PIN)
 #define LED2_ON     digitalLo(LED2_GPIO_PORT,LED2_PIN)

 #define LED3_TOGGLE   digitalToggle(LED3_GPIO_PORT,LED3_PIN)
 #define LED3_OFF    digitalHi(LED3_GPIO_PORT,LED3_PIN)
 #define LED3_ON     digitalLo(LED3_GPIO_PORT,LED3_PIN)

 /* 基本混色，后面高级用法使用PWM可混出全彩颜色,且效果更好 */

 //红
 #define LED_RED  \
         LED1_ON;\
         LED2_OFF\
         LED3_OFF
 //绿
 #define LED_GREEN   \
         LED1_OFF;\
         LED2_ON\
         LED3_OFF
 //蓝
 #define LED_BLUE  \
         LED1_OFF;\
         LED2_OFF\
         LED3_ON
 //黄(红+绿)
 #define LED_YELLOW  \
         LED1_ON;\
         LED2_ON\
         LED3_OFF
 //紫(红+蓝)
 #define LED_PURPLE  \
         LED1_ON;\
         LED2_OFF\
         LED3_ON
 //青(绿+蓝)
 #define LED_CYAN \
         LED1_OFF;\
         LED2_ON\
         LED3_ON
 //白(红+绿+蓝)
 #define LED_WHITE \
         LED1_ON;\
         LED2_ON\
         LED3_ON
 //黑(全部关闭)
 #define LED_RGBOFF  \
         LED1_OFF;\
         LED2_OFF\
         LED3_OFF
```
> 代码清单4：F103-MINI开发板控制LED亮灭的宏

```c
/** 控制LED灯亮灭的宏，
* LED低电平亮，设置ON=0，OFF=1
* 若LED高电平亮，把宏设置成ON=1 ，OFF=0 即可
*/
#define ON  GPIO_PIN_RESET
#define OFF GPIO_PIN_SET

/* 带参宏，可以像内联函数一样使用 */
#define LED1(a) HAL_GPIO_WritePin(LED1_GPIO_PORT,LED1_PIN,a)
#define LED2(a) HAL_GPIO_WritePin(LED2_GPIO_PORT,LED2_PIN,a)


/* 直接操作寄存器的方法控制IO */
#define digitalHi(p,i)      {p->BSRR=i;}        //设置为高电平
#define digitalLo(p,i)      {p->BSRR=(uint32_t)i << 16;}        //输出低电平
#define digitalToggle(p,i)  {p->ODR ^=i;}     //输出反转状态

/* 定义控制IO的宏 */
#define LED1_TOGGLE   digitalToggle(LED1_GPIO_PORT,LED1_PIN)
#define LED1_OFF      digitalHi(LED1_GPIO_PORT,LED1_PIN)
#define LED1_ON       digitalLo(LED1_GPIO_PORT,LED1_PIN)

#define LED2_TOGGLE   digitalToggle(LED2_GPIO_PORT,LED2_PIN)
#define LED2_OFF      digitalHi(LED2_GPIO_PORT,LED2_PIN)
#define LED2_ON       digitalLo(LED2_GPIO_PORT,LED2_PIN)
```
这部分宏控制LED亮灭的操作是直接向BSRR寄存器写入控制指令来实现的，对BSRR低16位写1输出高电平，对BSRR高16位写1输出低电平，对ODR寄存器某位进行异或操作可反转位的状态。

对于F103-指南者和F103-霸道开发板上的RGB彩灯可以实现混色，如最后一段代码我们控制红灯和绿灯亮而蓝灯灭，可混出黄色效果。

代码中的“”是C语言中的续行符语法，表示续行符的下一行与续行符所在的代码是同一行。代码中因为宏定义关键字“#define”只是对当前行有效，所以我们使用续行符“\”来连接起来，以下的代码是等效的：
```c
#define LED_YELLOW LED1_ON; LED2_ON; LED3_OFF
```
应用续行符的时候要注意，在“\”后面不能有任何字符(包括注释、空格)，只能直接回车。

#### 2.1.3 LED GPIO初始化函数
利用上面的宏，编写LED灯的初始化函数，见以下代码清单：
> 代码清单5 F103-指南者和F103-霸道开发板的LED GPIO初始化函数
```c
void LED_GPIO_Config(void)
 {
     /*定义一个GPIO_InitTypeDef类型的结构体*/
     GPIO_InitTypeDef  GPIO_InitStruct;
     /*开启LED相关的GPIO外设时钟*/
     LED1_GPIO_CLK_ENABLE();
     LED2_GPIO_CLK_ENABLE();
     LED3_GPIO_CLK_ENABLE();
     /*选择要控制的GPIO引脚*/
     GPIO_InitStruct.Pin = LED1_PIN;
     /*设置引脚的输出类型为推挽输出*/
     GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;
     /*设置引脚为上拉模式*/
     GPIO_InitStruct.Pull  = GPIO_PULLUP;
     /*设置引脚速率为高速 */
     GPIO_InitStruct.Speed = GPIO_SPEED_HIGH;
     /*调用库函数，使用上面配置的GPIO_InitStructure初始化GPIO*/
     HAL_GPIO_Init(LED1_GPIO_PORT, &GPIO_InitStruct);
     /*选择要控制的GPIO引脚*/
     GPIO_InitStruct.Pin = LED2_PIN;
     HAL_GPIO_Init(LED2_GPIO_PORT, &GPIO_InitStruct);
     /*选择要控制的GPIO引脚*/
     GPIO_InitStruct.Pin = LED3_PIN;
     HAL_GPIO_Init(LED3_GPIO_PORT, &GPIO_InitStruct);
     /*关闭RGB灯*/
     LED_RGBOFF;
 }
```

> 代码清单6：F103-MINI开发板的LED GPIO初始化函数
```c
void LED_GPIO_Config(void)
{
    /*定义一个GPIO_InitTypeDef类型的结构体*/
    GPIO_InitTypeDef  GPIO_InitStruct;
    /*开启LED相关的GPIO外设时钟*/
    LED1_GPIO_CLK_ENABLE();
    LED2_GPIO_CLK_ENABLE();

    /*选择要控制的GPIO引脚*/
    GPIO_InitStruct.Pin = LED1_PIN;
    /*设置引脚的输出类型为推挽输出*/
    GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;
    /*设置引脚为上拉模式*/
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    /*设置引脚速率为高速 */
    GPIO_InitStruct.Speed = GPIO_SPEED_HIGH;
    /*调用库函数，使用上面配置的GPIO_InitStructure初始化GPIO*/
    HAL_GPIO_Init(LED1_GPIO_PORT, &GPIO_InitStruct);
    /*选择要控制的GPIO引脚*/
    GPIO_InitStruct.Pin = LED2_PIN;
    HAL_GPIO_Init(LED2_GPIO_PORT, &GPIO_InitStruct);
    /*关闭RGB灯*/
    LED_RGBOFF;
}
```
整个函数与“构建库函数雏形”章节中的类似，主要区别是硬件相关的部分使用宏来代替， 初始化GPIO端口时钟时也采用了STM32库函数，函数执行流程如下：

(1)使用GPIO_InitTypeDef定义GPIO初始化结构体变量，以便下面用于存储GPIO配置。

(2) 调用宏定义函数LED1_GPIO_CLK_ENABLE()来使能LED灯的GPIO端口时钟，该函数在HAL库里边将操作寄存器部分封装起来，直接调用宏即可。

(3)向GPIO初始化结构体赋值，把引脚初始化成推挽输出模式，其中的GPIO_PIN使用宏“LEDx_PIN”来赋值，使函数的实现方便移植。

(4) 使用以上初始化结构体的配置，调用HAL_GPIO_Init函数向寄存器写入参数，完成GPIO的初始化， 这里的GPIO端口使用“LEDx_GPIO_PORT”宏来赋值，也是为了程序移植方便。

(5)初始化结构体，只修改控制的引脚和端口，初始化其它LED灯使用的GPIO引脚。

(6)使用宏控制RGB灯默认关闭。

#### 2.1.4 主函数
编写完LED灯的控制函数后，就可以在main函数中测试了，见以下代码清单：

> 代码清单7: F103-指南者和F103-霸道开发板控制LED灯 ，main文件
```c
int main(void)
 {
     /* 系统时钟初始化成72 MHz */
     SystemClock_Config();

     /* LED 端口初始化 */
     LED_GPIO_Config();

     /* 控制LED灯 */
     while (1) {
         LED1( ON );      // 亮
         HAL_Delay(1000);
         LED1( OFF );      // 灭
         HAL_Delay(1000);

         LED2( ON );     // 亮
         HAL_Delay(1000);
         LED2( OFF );      // 灭

         LED3( ON );      // 亮
         HAL_Delay(1000);
         LED3( OFF );      // 灭

         /*轮流显示 红绿蓝黄紫青白 颜色*/
         LED_RED;
         HAL_Delay(1000);

         LED_GREEN;
         HAL_Delay(1000);

         LED_BLUE;
         HAL_Delay(1000);

         LED_YELLOW;
         HAL_Delay(1000);

         LED_PURPLE;
         HAL_Delay(1000);

         LED_CYAN;
         HAL_Delay(1000);

         LED_WHITE;
         HAL_Delay(1000);

         LED_RGBOFF;
         HAL_Delay(1000);
     }
 }
```
> 代码清单8：F103-MINI开发板控制LED灯 ，main文件
```c
int main(void)
{
    /* 系统时钟初始化成72 MHz */
    SystemClock_Config();

    /* LED 端口初始化 */
    LED_GPIO_Config();

    /* 控制LED灯 */
    while (1) {
        LED1( ON );      // 亮
        HAL_Delay(1000);
        LED1( OFF );      // 灭
        HAL_Delay(1000);

        LED2( ON );     // 亮
        HAL_Delay(1000);
        LED2( OFF );      // 灭
    }
}
```
在main函数中，调用SystemClock_Config函数初始化系统的时钟为72MHz，所有程序都必须设置好系统的时钟再进行其他操作，具体设置将在RCC时钟章节详细讲解， 接着调用我们前面定义的LED_GPIO_Config初始化好LED的控制引脚， 然后直接调用各种控制LED灯亮灭的宏来实现LED灯的控制，延时采用库自带基于滴答时钟延时HAL_Delay单位为ms，直接调用即可，HAL_Delay(1000)表示延时1s。

以上，就是一个使用STM32 HAL软件库开发应用的流程。
[GPIO输出—使用固件库点亮LED](https://doc.embedfire.com/mcu/stm32/f103/hal_generalzh/latest/doc/chapter12/chapter12.html)