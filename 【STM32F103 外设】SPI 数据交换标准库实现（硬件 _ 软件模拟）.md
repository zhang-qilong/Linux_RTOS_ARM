@[toc]
# 硬件实现
## SPI 初始化
```c
#include "stm32f10x.h"
#define GPIO_Pin_NSS  	GPIO_Pin_4	// NSS
#define GPIO_Pin_SCK  	GPIO_Pin_5	// SCK
#define GPIO_Pin_MISO  	GPIO_Pin_6	// MISO
#define GPIO_Pin_MOSI  	GPIO_Pin_7	// MOSI

/**
  * @brief SPI 初始化
  * @params None
  * @returns None
  */
void SPI_Init_(void)
{
	/* 开启外设时钟 */
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_SPI1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	
	/* SPI 对应的 GPIO 输入输出配置 */
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;  				// SCK 和 MOSI 交由外设控制
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_SCK | GPIO_Pin_MOSI;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;					//MISO 接收，设为输入模式
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_MISO;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;				// NSS 普通 IO 控制输出
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_NSS;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	/* SPI 配置 */
	SPI_InitTypeDef SPI_InitStructure;
	/* SPI 时钟极性和相位组合产生 4 种模式 */
	/*模式0： CPHA = 0，CPOL = 0 */
	/*模式1： CPHA = 1，CPOL = 0 */
	/*模式2： CPHA = 0，CPOL = 1 */
	/*模式3： CPHA = 1，CPOL = 1 */
	SPI_InitStructure.SPI_CPHA = SPI_CPHA_1Edge;							//SCK 第一个边沿将数据移入移位寄存器
																			//SCK 第二个边沿将数据从移位寄存器移出
	SPI_InitStructure.SPI_CPOL = SPI_CPOL_Low;								//空闲状态下 SCK 为低电平
	SPI_InitStructure.SPI_Mode = SPI_Mode_Master;							//主机模式
	SPI_InitStructure.SPI_DataSize = SPI_DataSize_8b;						//8 位数据宽度
	SPI_InitStructure.SPI_Direction = SPI_Direction_2Lines_FullDuplex;		//选择 2 线全双工方向
	SPI_InitStructure.SPI_FirstBit = SPI_FirstBit_MSB;						//高位先行
	SPI_InitStructure.SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_128;	//波特率分频，选择 128 分频 PSCK = PLCX / 128
	SPI_InitStructure.SPI_NSS = SPI_NSS_Soft;								//软件控制片选
	SPI_InitStructure.SPI_CRCPolynomial = 7;								//CRC 校验多项式，暂时用不到
	SPI_Init(SPI1, &SPI_InitStructure);
	SPI_Cmd(SPI1, ENABLE);
}
```
> 模式 0：CPOL = 0，CPHA = 0
> 空闲状态 SCK 为低电平，SCK 第一个边沿向移位寄存器移入数据（读取），第二个边沿从移位寄存器移出数据（发送）
![模式0](https://i-blog.csdnimg.cn/direct/7abe3d9a75d9450ba88b8dbeecb82f6a.png)

> 模式 1：CPOL = 0，CPHA = 1
>  空闲状态 SCK 为低电平，SCK 第一个边沿从移位寄存器移出数据（发送），第二个边沿向移位寄存器移入数据（读取）
![SPI 模式1](https://i-blog.csdnimg.cn/direct/e5b30442cb2d4ba682bd420ec57919c8.png)

> 模式 2：CPOL = 1，CPHA = 0
> 空闲状态 SCK 为高电平，SCK 第一个边沿向移位寄存器移入数据（读取），第二个边沿从移位寄存器移出数据（发送）
![SPI 模式2](https://i-blog.csdnimg.cn/direct/3406f783638244a0a3486fa55d8da924.png)

> 模式 3：CPOL = 1，CPHA = 1
>  空闲状态 SCK 为高电平，SCK 第一个边沿从移位寄存器移出数据（发送），第二个边沿向移位寄存器移入数据（读取）
![SPI 模式3](https://i-blog.csdnimg.cn/direct/b5014c16b0e04c0c966a440434b6feca.png)

>**总结**：读取数据时（移入寄存器），两条数据线上的数据是不能改变的（类似于 I2C 中 SCL 高电平 SDA 线不能改变，否则会发生停止或重复启动）
## SPI 交换一个字节数据
![SPI 数据交换示意图](https://i-blog.csdnimg.cn/direct/89aa7d5d1d8145d6bfb34f510e200d91.png)

```c
/**
  * @brief SPI 作为主设备与从设备交换一字节数据
  * @params ByteSend 待发送字节（主机->从机）
  * @returns 接收到的字节数据（从机->主机）
  */
uint8_t SPI_Swap_Byte(uint8_t ByteSend)
{
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_TXE) != SET);	//等待传输寄存器为空
	SPI_I2S_SendData(SPI1, ByteSend);								//发送数据
	while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_RXNE) != SET);	//等待接收寄存器为空
	return SPI_I2S_ReceiveData(SPI1);								//接收数据
}
```
# 软件实现
## GPIO 初始化

```c
#include "stm32f10x.h"
#define GPIO_Pin_NSS  	GPIO_Pin_4	// NSS
#define GPIO_Pin_SCK  	GPIO_Pin_5	// SCK
#define GPIO_Pin_MISO 	GPIO_Pin_6	// MISO
#define GPIO_Pin_MOSI  	GPIO_Pin_7	// MOSI

/**
  * @brief GPIO 初始化
  * @params None
  * @returns None
  */
void SPI_Init_(void)
{
	/* 开启外设时钟 */
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	
	/* SPI 对应的 GPIO 输入输出配置 */
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;  				
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_SCK | GPIO_Pin_MOSI | GPIO_Pin_NSS;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;					//MISO 接收，设为输入模式
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_MISO;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	/* 空闲状态不选中设备，SCK 为低电平 */
	GPIO_SetBits(GPIOA, GPIO_Pin_NSS);
	GPIO_ResetBits(GPIOA, GPIO_Pin_SCK);
}
```
## 引脚配置

```c
/**
  * @brief SPI 片选软件控制
  * @params Value 写入 NSS 线的电平（0 表示选中设备，1 表示释放设备）
  * @returns None
  */
void MySPI_W_SS(uint8_t Value)
{
	GPIO_WriteBit(GPIOA, GPIO_Pin_NSS, (BitAction)Value);		//根据Value，设置 NSS 引脚的电平
	/* SPI 通信过程中不需要像 I2C 那样复杂的时序控制。
	   数据的发送和接收可以在任意时间同步进行，
	   所以不需要在不同操作之间增加延时。*/
}

/**
  * @brief SPI SCK 电平控制
  * @params Value 写入 SCK 线的电平（0 表示低电平，1 表示高电平）
  * @returns None
  */
void MySPI_W_SCK(uint8_t Value)
{
	GPIO_WriteBit(GPIOA, GPIO_Pin_SCK, (BitAction)Value);		//根据Value，设置 SCK 引脚的电平
}

/**
  * @brief SPI MOSI 电平控制
  * @params Value 写入 MOSI 线的电平（0 表示低电平，1 表示高电平）
  * @returns None
  */
void MySPI_W_MOSI(uint8_t Value)
{
	GPIO_WriteBit(GPIOA, GPIO_Pin_MOSI, (BitAction)Value);		//根据Value，设置 MOSI 引脚的电平
}

/**
  * @brief SPI MISO 电平读取
  * @params None
  * @returns 读取到的 MISO 电平 （0 表示低电平，1 表示高电平）
  */
uint8_t MySPI_R_MISO(void)
{
	return GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_MISO);			//读取 MISO 电平
}
```
## 协议层
```c
/**
  * @brief SPI 作为主设备与从设备交换一字节数据
  * @params ByteSend 待发送字节（主机->从机）
  * @returns 接收到的字节数据（从机->主机）
  * @note 产生边沿时，从机的移入移出操作在从机代码中体现，主机移入通过 MISO 电平完成，移出通过写 MOSI 电平完成
  */
uint8_t MySPI_SwapByte(uint8_t ByteSend)
{
	uint8_t ByteReceive = 0x00;									//定义接收的数据，并赋初值 0x00
	MySPI_W_SS(RESET);											//选中设备，
	/* 此处演示为模式 0 */
	for (uint8_t i = 0; i < 8; i++)								//循环 8 次，依次交换每一位数据
	{
		MySPI_W_SCK(RESET);										//拉低 SCK，从机将数据放到 MOSI 上
		MySPI_W_MOSI(ByteSend & (0x80 >> i));					//取出 ByteSend 的指定位数据，放到 MISO 上
		MySPI_W_SCK(SET);										//拉高 SCK，从机读取 MOSI 数据
		if (MySPI_R_MISO() == 1){ByteReceive |= (0x80 >> i);}	//主机读取 MISO 数据，并存储到 Byte 变量（高位先行）
																//当 MISO 为 1 时，置变量指定位为 1
	}	
	
	return ByteReceive;
}
```
> 欢迎批评指正！！！
> 图片来源：[STM32入门教程-2023版 细致讲解 中文字幕](https://www.bilibili.com/video/BV1th411z7sn/?spm_id_from=333.337.search-card.all.click)
