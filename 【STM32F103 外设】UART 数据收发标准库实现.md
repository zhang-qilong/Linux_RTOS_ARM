@[toc]
# UART  初始化
使能外设时钟、配置GPIO、配置NVIC
```c
#include "stm32f10x.h"
#include <math.h>
#include <stdio.h>
#include <stdarg.h>

/**
  * @brief 串口初始化
  * @params None
  * @returns None
  */
void UART_Init(void)
{
	/* 开启外设时钟 */
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	/* GPIOA_9 发送 GPIOA_10 接收 */
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;  //复用推挽输出，将IO控制权交给USART外设
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	/* 串口参数配置 */
	USART_InitTypeDef USART_InitStructure;
	USART_InitStructure.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;
	USART_InitStructure.USART_BaudRate = 115200;
	USART_InitStructure.USART_WordLength = USART_WordLength_8b;
	USART_InitStructure.USART_StopBits = USART_StopBits_1;
	USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
	USART_InitStructure.USART_Parity = USART_Parity_No;
	USART_Init(USART1, &USART_InitStructure);
	
	/* 串口中断配置 */
	USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);					//开启串口接收数据的中断
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);					//配置NVIC分组为两位抢占优先级，两位响应优先级
	
	NVIC_InitTypeDef NVIC_InitStructure;							//定义结构体变量
	NVIC_InitStructure.NVIC_IRQChannel = USART1_IRQn;				//选择配置NVIC的USART1线
	NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;					//指定NVIC线路使能
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;		//指定NVIC线路的抢占优先级为1（根据实际需要修改）
	NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;				//指定NVIC线路的响应优先级为1（根据实际需要修改）
	NVIC_Init(&NVIC_InitStructure);									//将结构体变量交给NVIC_Init，配置NVIC外设

	/* 串口使能 */
	USART_Cmd(USART1, ENABLE);
}
```
# UART 发送一byte（8 bit）数据
```c
/**
  * @brief 串口发送一个字节
  * @params Data 待发送字节
  * @returns None
  */
void UART_Send_Byte(uint8_t const Data)
{
	USART_SendData(USART1, Data);									//把要发送的数据写进发送缓冲寄存器（自动清除TXE位）
	while (RESET == USART_GetFlagStatus(USART1, USART_FLAG_TXE));	//等待发送数据缓冲区数据转移至移位寄存器
}
```
# UART 发送多个byte（n*8 bit）数据
```c
/**
  * @brief 串口发送一个数组（多个字节）
  * @params 
  *		@arg Array 待发送数组
  *     @arg Length 数组长度
  * @returns None
  */
void UART_Send_Array(uint8_t *Array, uint16_t Length)
{
	for  (uint16_t i = 0; i < Length; i++)
	{
		UART_Send_Byte(Array[i]);
	}
}
```
# UART 发送一个字符串数据
```c
/**
  * @brief 串口发送一个字符串
  * @params String 要发送字符串的首地址
  * @returns None
  */
void UART_Send_String(char *String)
{
	while (*String != '\0')
	{
		UART_Send_Byte(*String);
		String++;
	}
	while (RESET == USART_GetFlagStatus(USART1, USART_FLAG_TC));	//确认所有数据发送完成
}
```
# UART 发送一个十进制数据
单独处理每个数字，转化为字符数据后发送
```c
/**
  * @brief 串口发送十进制数字
  * @params 
  * 	@arg Number 要发送的数字，范围：0~4294967295
  * 	@arg Length 要发送数字的长度，范围：0~10
  * @returns None
  */
void UART_Send_Number(uint32_t Number, uint8_t Length)
{
	for (uint8_t i = 0; i < Length; i ++)										//根据数字长度遍历数字的每一位
	{
		UART_Send_Byte(Number / (uint8_t) pow(10, Length - i - 1) % 10 + '0'); //获得ASSCII码
	} 
	while (RESET == USART_GetFlagStatus(USART1, USART_FLAG_TC));				//确认所有数据发送完成
}
```
# 将标准输出函数（fprintf）重定向至 UART
可以通过串口助手打印信息，方便调试
```c
/**
  * @brief 将printf重定向到底层函数
  * @params 固定格式，无需变动
  * @returns 固定格式，无需变动
  * @note 包含头文件stdio.h
  */
int fputc(int ch, FILE *f)
{
	UART_Send_Byte(ch);			//将printf的底层重定向到自己的发送字节函数
	return ch;
}
```
# 自己封装的 printf 函数
```c
/**
  * @brief自己封装的prinf函数
  * @params 
  * 	@arg format 格式化字符串
  * 	@arg ... 可变的参数列表
  * @note 包含头文件stdarg.h
  * @returns None
  */
void UART_Printf(char *format, ...)
{
	char String[100];				//定义字符数组
	va_list args;					//定义可变参数列表数据类型的变量args
	va_start(args, format);			//从format开始，接收参数列表到args变量
	vsprintf(String, format, args);	//使用vsprintf打印格式化字符串和参数列表到字符数组中
	va_end(args);					//结束变量args
	UART_Send_String(String);		//串口发送字符数组（字符串）
}
```
# UART 接收字符数据包
通过中断方式接收，避免使用轮询方式占用CPU
```c
/**
  * @brief USART1中断服务函数（以数据包格式接收文本数据）
  * @params None
  * @return None
  * @note 中断函数，无需调用，中断触发后自动执行，函数名为预留的指定名称，可以从启动文件复制
  *       数据帧包组织格式为 “@数据\r\n” （包头为@，包尾为\r\n）
  */
uint8_t UART_RxFlag = 0;
uint8_t UART_RxPacket[100];
void USART1_IRQHandler(void)
{
	static uint8_t RxState = 0;								//定义表示当前状态机状态的静态变量
	static uint8_t pRxPacket = 0;							//定义表示当前接收数据位置的静态变量
	if (SET == USART_GetITStatus(USART1, USART_IT_RXNE))	//判断是否是USART1的接收事件触发的中断
	{
		uint8_t RxData = USART_ReceiveData(USART1);			//读取接收数据寄存器
		
		/* 接收数据包包头 */
		if (RxState == 0)
		{
			if (RxData == '@' && UART_RxFlag == 0)		//如果数据确实是包头，并且上一个数据包已处理完毕
			{
				RxState = 1;							//置下一个状态
				pRxPacket = 0;							//数据包的位置归零
			}
		}
		/* 接收数据包数据，同时判断是否接收到了第一个包尾 */
		else if (RxState == 1)
		{
			if (RxData == '\r')							//如果收到第一个包尾
			{
				RxState = 2;							//置下一个状态
			}
			else										//接收到了非包尾
			{
				UART_RxPacket[pRxPacket] = RxData;		//将数据存入数据包数组的指定位置
				pRxPacket++;							//数据包的位置自增
			}
		}
		/* 接收数据包第二个包尾 */
		else if (RxState == 2)
		{
			if (RxData == '\n')							//如果收到第二个包尾
			{
				RxState = 0;							//回到接收包头状态
				UART_RxPacket[pRxPacket] = '\0';		//将收到的字符数据包添加一个字符串结束标志
				UART_RxFlag = 1;						//接收数据包标志位置1，成功接收一个数据包
			}
		}
		USART_ClearITPendingBit(USART1, USART_IT_RXNE);	//清除中断标志位
	}
}
```

> TODO：UART+DMA 数据收发
欢迎批评指正... ...

