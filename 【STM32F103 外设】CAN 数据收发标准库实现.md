@[toc]
# CAN  初始化

```c
#include "stm32f10x.h"

#define CAN_Rx 					GPIO_Pin_11
#define CAN_Tx 					GPIO_Pin_12

/** @brief CAN 初始化
  * @params None
  * @return None
  */
void MyCAN_Init(void)
{
	/* 开启时钟 */
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_CAN1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	/* GPIO 配置 */
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = CAN_Rx;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;						//复用推挽输出，将 IO 控制权交由 CAN 外设
	GPIO_InitStructure.GPIO_Pin = CAN_Tx;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	/* CAN 配置 */
	CAN_InitTypeDef CAN_InitStructure;
	CAN_InitStructure.CAN_Mode = CAN_Mode_LoopBack;						//回环模式，单个设备自收发
	CAN_InitStructure.CAN_ABOM = ENABLE;								//使能离线自动管理
	CAN_InitStructure.CAN_AWUM = ENABLE;								//使能自动唤醒
	CAN_InitStructure.CAN_NART = DISABLE;								//开启自动重传（一直传输，直至成功）
	CAN_InitStructure.CAN_RFLM = DISABLE;								//接收 FIFO 不锁定（溢出后覆盖旧数据）
	CAN_InitStructure.CAN_TTCM = DISABLE;								//不启用时间戳功能
	CAN_InitStructure.CAN_TXFP = DISABLE;								//按照报文 ID 发送（使能时，先请求先发送）
	/* 设置波特率 1e6 bit per second = 1 /（tq * （1 + BS1 + BS2））*/
	CAN_InitStructure.CAN_SJW = CAN_SJW_4tq;	
	CAN_InitStructure.CAN_BS1 = CAN_BS1_5tq;
	CAN_InitStructure.CAN_BS2 = CAN_BS2_3tq;							//（1+5+3）tq
	CAN_InitStructure.CAN_Prescaler = 8;								// tq = 8 / 71M =（1 / 9M）s
																		//发送一位数据时间为（1 / 1M）
	CAN_Init(CAN1, &CAN_InitStructure);
}
```
![位时序图](https://i-blog.csdnimg.cn/direct/ea28edda74604d83af80fc5d46c96822.png)
波特率 = (1 / 正常的位时间)
		    = 1 / (t~q~ + t~BS1~ + t~BS2~)
		    = 1 / (t~q~ + 5t~q~ + 3t~q~)
		    = 1 / 9t~q~
		    = 1 / 72t~PCLK~（对于STM32F103C8T6芯片，t~PCLK~ = (1 / 72 000 000)s）
		    = 1 000 000 bps
# CAN  标识符过滤器配置
- STM32F103C8T6的CAN控制器为应用程序提供了14个位宽可变的、可配置的过滤器组(13~0)。
- 每个过滤器组的位宽都可以独立配置，以满足应用程序的不同需求。根据位宽的不同，每个过滤器组可提供：
 	- 1个32位过滤器，包括：STDID[10:0]、EXTID[17:0]、IDE和RTR位
 	- 2个16位过滤器，包括：STDID[10:0]、IDE、RTR和EXTID[17:15]位
 	- 过滤器可配置为，屏蔽位模式和标识符列表模式
 ```
	 在标识符列表模式下，屏蔽寄存器也被当作标识符寄存器用
```
![过滤器组位宽设置](https://i-blog.csdnimg.cn/direct/ca5d94075eb74be0aa1920be07616d1b.png)
```c
#define CAN_STD_ID  			0x123
#define CAN_EXT_ID 				(uint32_t)0x1800f001
/** @brief CAN 标识符过滤器配置
  * @params None
  * @return None
  */
void CAN_Filter_Init(void)
{
	/* CAN 过滤器配置 */
	CAN_FilterInitTypeDef CAN_FilterInitStructure;
	CAN_FilterInitStructure.CAN_FilterActivation = ENABLE; 				//启动过滤器
	CAN_FilterInitStructure.CAN_FilterNumber = 0;						//指定过滤器组编号（STM32F103 有 0~13 共 14 个过滤器组）
	CAN_FilterInitStructure.CAN_FilterFIFOAssignment = CAN_Filter_FIFO0;//存放在 FIFO0 中
	/* 配置 CAN 过滤器为 32 位列表模式 */
	/* 16 位列表模式 */
	/* 16 位掩码模式 */
	/* 32 位列表模式 */
	/* 32 位掩码模式 */
	CAN_FilterInitStructure.CAN_FilterScale = CAN_FilterScale_32bit;						//过滤器位宽为 32 位
	CAN_FilterInitStructure.CAN_FilterMode = CAN_FilterMode_IdList;							//ID 列表模式
																							//收到报文的标识符必须与过滤器的值完全相等才能通过
																							//屏蔽位模式，可以指定标识符的某些位为指定值时能通过
	/* 过滤标准数据帧 ID：0x123 */
	CAN_FilterInitStructure.CAN_FilterIdHigh = (CAN_STD_ID << 5); 							//设置 STDID 
	CAN_FilterInitStructure.CAN_FilterIdLow = 0 | (CAN_ID_STD & 0xff);						//设置 IDE 位为 0 ，不使用扩展 ID 	 	
	CAN_FilterInitStructure.CAN_FilterMaskIdHigh = ((CAN_EXT_ID<<3)&0xffff)>>16;			
	CAN_FilterInitStructure.CAN_FilterMaskIdLow = ((CAN_EXT_ID<<3)&0xffff) | CAN_ID_EXT;	//设置 IDE 位为 1 ，使用扩展 ID
	CAN_FilterInit(&CAN_FilterInitStructure);
}
```
> 标识符配置说明：本例中采用32位标识符列表模式。
> 0x123 = 0b0000 0000 0000 0000 0000 0[001 0010 0011] 为标准标识符（未超过11位），方括号中表示STDID
> 0x1800 f001 = 0b000[1 1000 0000 00][00 1111 0000 0000 0001] 为扩展标识符（超过11位），第一个方括号中的11位为STDID，第一个方括号中的18位为EXTID
> 过滤器组位宽设置图可知，STDID存放在32位寄存器的21-31位，EXTID存放在32位寄存器的3-20位
> 从标识符过滤器配置结构体中可得，需要将整个32位的过滤器拆分为两个16位数据传入，而两个标识符均为32位，所以直接传入会默认强转为uint16_t，丢弃高16位。
![标识符过滤器配置结构体](https://i-blog.csdnimg.cn/direct/b77ce35b3e7f49999cfe6c2d91016de2.png)
> >对于0x123，将变为0000 0[001 0010 0011]，所以，只要左移5位即可将STDID存入高16位寄存器中，低16位寄存器中只需要将IDE置0，表示非扩展标识符即可。
> >对于0x1800 f001，若直接强转会导致高16位丢失，所以先左移3位取出扩展标识符的有效部分[1 1000 0000 0000 1111 0000 0000 0001](STDID+EXTID)，随后右移16位使有效标识符的最高位对应强转后的最高位，从而保证强转数据不失真，而后存入高16位寄存器中，有效标识符的剩余部分直接存入低16位寄存器并将IDE置1，表示扩展标识符即可。
# CAN  发送数据

```c
/** @brief CAN 发送数据（实现一）
  * @params 
  * 	@arg ID 数据标识 ID
  * 	@arg Length 数据长度（取值范围为1~8）
  * 	@arg TxData 存放数据的数组名称
  * @return None
  */
void CAN_Send_Data(uint32_t ID, uint8_t Length, uint8_t *TxData)
{
	CanTxMsg TxMessage;
	TxMessage.StdId = ID;													//标准标识符
	TxMessage.ExtId = ID;													//扩展标识符
	TxMessage.IDE = CAN_Id_Standard;										//标准 ID 有效
	TxMessage.RTR = CAN_RTR_Data;											//数据帧标志位
	TxMessage.DLC = Length;													//数据字节数目
	for (uint8_t i = 0; i < Length; i++) {TxMessage.Data[i] = TxData[i];}	//获取发送内容
	CAN_Transmit(CAN1, &TxMessage);
}

/** @brief CAN 发送数据（实现二）
  * @params TxMessage CAN 发送数据结构体指针
  * @return None
  */
void CAN_Send_Data(CanTxMsg *TxMessage)
{
	CAN_Transmit(CAN1, TxMessage);
}
```
# CAN  接收数据
```c
/** @brief CAN 接收数据（实现一）
  * @params 
  * 	@arg ID 数据标识指针
  * 	@arg Length 数据长度指针
  * 	@arg TxData 存放接收数据的数组
  * @return None
  * @note C 语言中不支持函数多返回值，采用指针传参获取函数返回值
  */
void CAN_Receive_Data(uint32_t *ID, uint8_t *Length, uint8_t *RxData)
{
	CanRxMsg RxMessage;
	CAN_Receive(CAN1, CAN_FIFO0, &RxMessage);
	if (RxMessage.IDE == CAN_Id_Standard)
	{
		*ID = RxMessage.StdId;
	}
	else
	{
		*ID = RxMessage.ExtId;
	}
	if (RxMessage.RTR == CAN_RTR_Data)
	{
		*Length = RxMessage.DLC;
		for (uint8_t i = 0; i < *Length; i++) {RxData[i] = RxMessage.Data[i];}	//获取发送内容
	}
	else
	{
		/* ... ... */
	}
}
/** @brief CAN 接收数据（实现二）
  * @params RxMessage CAN 接收数据结构体指针
  * @return None
  */
void CAN_Receive_Data(CanRxMsg *RxMessage)
{
	CAN_Receive(CAN1, CAN_FIFO0, &RxMessage);
}
```
TODO：CAN中断发送
> 欢迎批评指正！！！

> 文中图片来源：STM32F10xxx参考手册
> 参考教程：[CAN总线入门教程-全面细致 面包板教学 多机通信](https://www.bilibili.com/video/BV1vu4m1F7Gt/?spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click&vd_source=6a6c56e434d9639407c80ef57b3873ae)
