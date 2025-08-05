# STM32 桌面小狗

---

## 项目简介

基于stm32开发的桌面DIY智能小狗，可进行简单的与人交互功能，支持语言与蓝牙同时操控

## 效果图

![](C:\Users\admin\Desktop\智能小狗\img\image-20250805113728504-1754365478852-2.png)

![](C:\Users\admin\Desktop\智能小狗\img\image-20250805113810143-1754365493280-5.png)

- STM32单片机是跟着UP主【铁头山羊】学习的。
- OLED的使用看的是UP主【江协科技】老师的OLED教程，可进行自定义设置表情。
- 嘉立创PCB画板跟着UP主【Expert电子实验室】学习的。
- 语音模块用的是智元的语音控制，可进行自定义设置唤醒词命令词。

## 硬件

### 项目参数

- 语音模块用的是su-03t1，可进行自定义设置唤醒词命令词
- OLED模块用的是江科老师的OLED模块代码，也可进行自定义设置表情

### 硬件组成

项目由以下部分组成，电源部分、舵机部分、OLED部分、蓝牙部分，语音部分，本项目的控制采用串口控制，主要是通过麦克风接收语音信号并进行处理，提取人声进行解析比较，当声音符合指令后，进行对应的控制操作，或者用手机蓝牙控制。

### 焊接图

![v1.1电路板.png](C:\Users\admin\Desktop\智能小狗\img\74890faad95e4cab864fdefe0bfb031d.png)

---

## 功能代码

注：代码采用的是标准库，对各个模块进行了功能封装

### 主函数代码

主函数里的Action_Mode是全局变量，值是进串口中断更改的，主循环遍历这个值来切换动作模式

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "BlueTooth.h"
#include "Servo.h"
#include "PetAction.h"
#include "Face_Config.h"
#include "PWM.h"

uint16_t Time;
uint16_t HuXi;
uint16_t PanDuan=1;
uint16_t Wait=0;
 
int main(void)
{
	Servo_Init();
	OLED_Init();//OLED初始化
	BlueTooth_Init();//蓝牙初始化
	OLED_ShowImage(0,0,128,64,Face_sleep);
	OLED_Update();
	while(1)
	{	
		if(Action_Mode==0){Action_relaxed_getdowm();WServo_Angle(90);}//放松趴下
		else if(Action_Mode==1){Action_sit();}//坐下
		else if(Action_Mode==2){Action_upright();}//站立
		else if(Action_Mode==3){Action_getdowm();}//趴下
		else if(Action_Mode==4){Action_advance();}//前进
		else if(Action_Mode==5){Action_back();}//后退
		else if(Action_Mode==6){Action_Lrotation();}//左转
		else if(Action_Mode==7){Action_Rrotation();}//右转
		else if(Action_Mode==8){Action_Swing();}//摇摆
		else if(Action_Mode==9){Action_SwingTail();}//摇尾巴
		else if(Action_Mode==10){Action_JumpU();}//前跳
		else if(Action_Mode==11){Action_JumpD();}//后跳
		else if(Action_Mode==12){Action_upright2();}//站立方式2
		else if(Action_Mode==13){Action_Hello();}//打招呼
		else if(Action_Mode==14){Action_stretch();}//伸懒腰
		else if(Action_Mode==15){Action_Lstretch();}//后腿拉伸
	}
}
```

### PWM模式

由于一个计时器只能开启4个输出比较模式，对应四条腿，但还需要一个尾巴，所以只能再开启一个定时器了

```c
#include "stm32f10x.h"                  // Device header
 
void PWM_Init(void)
{
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2,ENABLE);//开启TIM2时钟
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3,ENABLE);//开启TIM3时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA,ENABLE);//开启GPIOA时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB,ENABLE);//开启GPIOB时钟
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode=GPIO_Mode_AF_PP;//复用推挽输出模式
	GPIO_InitStructure.GPIO_Pin=GPIO_Pin_0|GPIO_Pin_1|GPIO_Pin_2|GPIO_Pin_3|GPIO_Pin_6;//默认PA0是TIM2通道1的复用，PA1是TIM2通道2的复用所以开启这俩IO口...
	GPIO_InitStructure.GPIO_Speed=GPIO_Speed_50MHz;
	GPIO_Init(GPIOA,&GPIO_InitStructure);
	
	GPIO_InitStructure.GPIO_Mode=GPIO_Mode_AF_PP;//复用推挽输出模式
	GPIO_InitStructure.GPIO_Pin=GPIO_Pin_0|GPIO_Pin_1;
	GPIO_InitStructure.GPIO_Speed=GPIO_Speed_50MHz;
	GPIO_Init(GPIOB,&GPIO_InitStructure);
	
	TIM_InternalClockConfig(TIM2);//TIM2切换为内部定时器
	TIM_InternalClockConfig(TIM3);//TIM3切换为内部定时器
	
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
	TIM_TimeBaseInitStructure.TIM_ClockDivision=TIM_CKD_DIV1;//不分频
	TIM_TimeBaseInitStructure.TIM_CounterMode=TIM_CounterMode_Up;//向上计数
	TIM_TimeBaseInitStructure.TIM_Period=20000-1;
	TIM_TimeBaseInitStructure.TIM_Prescaler=72-1;
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter=0;
	TIM_TimeBaseInit(TIM2,&TIM_TimeBaseInitStructure);
	TIM_TimeBaseInit(TIM3,&TIM_TimeBaseInitStructure);
	
		//TIM3定时中断配置
	NVIC_InitTypeDef NVIC_InitStructure;
	NVIC_InitStructure.NVIC_IRQChannel=TIM3_IRQn;
	NVIC_InitStructure.NVIC_IRQChannelCmd=ENABLE;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority=0;
	NVIC_InitStructure.NVIC_IRQChannelSubPriority=0;
	NVIC_Init(&NVIC_InitStructure);
	TIM_ITConfig(TIM3,TIM_IT_Update,ENABLE);
	
	TIM_OCInitTypeDef TIM_OCInitStructure;
	TIM_OCStructInit(&TIM_OCInitStructure);
	TIM_OCInitStructure.TIM_OCMode=TIM_OCMode_PWM1;//输出比较模式采用PWM1
	TIM_OCInitStructure.TIM_OCPolarity=TIM_OCPolarity_High;
	TIM_OCInitStructure.TIM_OutputState=TIM_OutputState_Enable;
	TIM_OCInitStructure.TIM_Pulse=0;//初始化CCR的值为0
	TIM_OC1Init(TIM2,&TIM_OCInitStructure);//TIM2复用通道1开启
	TIM_OC2Init(TIM2,&TIM_OCInitStructure);//TIM2复用通道2开启
	TIM_OC3Init(TIM2,&TIM_OCInitStructure);//TIM2复用通道3开启
	TIM_OC4Init(TIM2,&TIM_OCInitStructure);//TIM2复用通道4开启
	
	TIM_OC1Init(TIM3,&TIM_OCInitStructure);//TIM3复用通道1开启
	TIM_OC3Init(TIM3,&TIM_OCInitStructure);//TIM3复用通道3开启
	TIM_OC4Init(TIM3,&TIM_OCInitStructure);//TIM3复用通道4开启
	
	TIM_Cmd(TIM2,ENABLE);//使能TIM2
	TIM_Cmd(TIM3,ENABLE);//使能TIM3
}
 
void PWM_SetCompare1(uint16_t Compare)
{
	
	TIM_SetCompare1(TIM2, Compare);//设置CCR1的值		
}
 
void PWM_SetCompare2(uint16_t Compare)
{
			
	TIM_SetCompare2(TIM2, Compare);//设置CCR2的值
}
 
void PWM_SetCompare3(uint16_t Compare)
{
			
	TIM_SetCompare3(TIM2, Compare);//设置CCR3的值
}
 
void PWM_SetCompare4(uint16_t Compare)
{
			
	TIM_SetCompare4(TIM2, Compare);//设置CCR4的值
}
 
void PWM_WSetCompare(uint16_t Compare)
{
	TIM_SetCompare1(TIM3, Compare);//设置尾巴CCR1的值
}
 
void PWM_LED1(uint16_t Compare)
{
	TIM_SetCompare3(TIM3,Compare);
}
 
void PWM_LED2(uint16_t Compare)
{
	TIM_SetCompare4(TIM3,Compare);
}
```

头文件

```c
#ifndef __PWM_H
#define __PWM_H
 
void PWM_Init(void);
void PWM_SetCompare1(uint16_t Compare);
void PWM_SetCompare2(uint16_t Compare);
void PWM_SetCompare3(uint16_t Compare);
void PWM_SetCompare4(uint16_t Compare);
void PWM_WSetCompare(uint16_t Compare);
void PWM_LED1(uint16_t Compare);
void PWM_LED2(uint16_t Compare);
 
#endif
```

### 舵机设置

为了统一角度，比如设置0度，那么所有舵机执行的方向朝向一致。所以舵机2和舵机4的Angle取补角

```c
#include "stm32f10x.h"                  // Device header
#include "PWM.h"
 
void Servo_Init()
{
	PWM_Init();	
}
 
void Servo_Angle1(float Angle)//左上
{
	PWM_SetCompare1(Angle / 180 * 2000 + 500);			
}
 
void Servo_Angle2(float Angle)//右上
{
	PWM_SetCompare2((180-Angle) / 180 * 2000 + 500);		
}
 
void Servo_Angle3(float Angle)//左下
{
	PWM_SetCompare3(Angle / 180 * 2000 + 500);			
}
 
void Servo_Angle4(float Angle)//右下
{
	PWM_SetCompare4((180-Angle) / 180 * 2000 + 500);			
}
 
void WServo_Angle(float Angle)//尾巴
{
	PWM_WSetCompare(Angle / 180 * 2000 + 500);			
}
```

头文件

```c
#ifndef __SERVO_H
#define __SERVO_H
 
void Servo_Init(void);
void Servo_Angle1(float Angle);
void Servo_Angle2(float Angle);
void Servo_Angle3(float Angle);
void Servo_Angle4(float Angle);
void WServo_Angle(float Angle);
 
#endif
```

动作代码
舵机的前进、后退、左转、右转比较难，这里是参考了bilibili上的up主石桥北关于舵机步态的视频。

```c
#include "stm32f10x.h"                  // Device header
#include "Servo.h"
#include "Delay.h"
#include "BlueTooth.h"
 
	/*舵机位置
	    1        2
	
	
	
	    3        4
	*/
	
	//舵机充当腿，括号里的值为0时表示的是腿向前进的方向甩，90度是站立
 
#define Chongfunumber 2  //动作重复次数、前进后退左转右转
#define SwingRepeatnumber 3  //摇摆重复次数
#define HelloRepeatnumber 4  //打招呼重复次数
 
uint16_t PAnumbers=Chongfunumber;//动作重复次数
 
uint16_t TiaoTurn=0;
uint16_t TiaoTurn2=0;
 
 
 
void Action_relaxed_getdowm(void)
{
	Servo_Angle1(20);
	Servo_Angle2(20);
	Delay_ms(80);
	Servo_Angle3(160);
	Servo_Angle4(160);
}
 
void Action_upright(void)//站立
{
	Servo_Angle1(90);
	Servo_Angle2(90);
	Delay_ms(80);
	Servo_Angle3(90);
	Servo_Angle4(90);
	
	if(WeiBa==1)
	{
		Action_Mode=9;
	}
}
 
void Action_upright2(void)//站立
{
	Servo_Angle3(90);
	Servo_Angle4(90);
	Delay_ms(80);
	Servo_Angle1(90);
	Servo_Angle2(90);
	
	if(WeiBa==1)
	{
		Action_Mode=9;
	}
}
 
void Action_getdowm(void)//趴下
{
	Servo_Angle1(20);
	Servo_Angle2(20);
	Delay_ms(80);
	Servo_Angle3(20);
	Servo_Angle4(20);
	
	if(WeiBa==1)
	{
		Action_Mode=9;
	}
}
 
void Action_sit(void)//坐下
{
	Servo_Angle1(90);
	Servo_Angle2(90);
	Delay_ms(80);
	Servo_Angle3(20);
	Servo_Angle4(20);
	
	if(WeiBa==1)
	{
		Action_Mode=9;
	}
}
 
 
 
void Action_advance(void)//前进
{
 
	while(Action_Mode==4)
	{
		PAnumbers=Chongfunumber;
			while((PAnumbers || Sustainedmove)&& Action_Mode==4)
			{
				Servo_Angle2(45);	
				Servo_Angle3(45);
				Delay_ms(SpeedDelay);
				if(Action_Mode!=4)break;
				Servo_Angle1(135);	
				Servo_Angle4(135);
				Delay_ms(SpeedDelay);
				if(Action_Mode!=4)break;
				Servo_Angle2(90);	
				Servo_Angle3(90);
				Delay_ms(SpeedDelay);
				if(Action_Mode!=4)break;
				Servo_Angle1(90);	
				Servo_Angle4(90);
				Delay_ms(SpeedDelay);
				if(Action_Mode!=4)break;
		
				Servo_Angle1(45);	
				Servo_Angle4(45);
				Delay_ms(SpeedDelay);
				if(Action_Mode!=4)break;
				Servo_Angle2(135);	
				Servo_Angle3(135);
				Delay_ms(SpeedDelay);
				if(Action_Mode!=4)break;
				Servo_Angle1(90);	
				Servo_Angle4(90);
				Delay_ms(SpeedDelay);
				if(Action_Mode!=4)break;
				Servo_Angle2(90);	
				Servo_Angle3(90);
				Delay_ms(SpeedDelay);
				if(Action_Mode!=4)break;
				
				PAnumbers--;
			}
			if(Sustainedmove!=1 && Action_Mode==4)
				Action_Mode=2;
	}
}
 
void Action_back(void)//后退
{
	while(Action_Mode==5)
	{
		PAnumbers=Chongfunumber;
		while((PAnumbers || Sustainedmove) && Action_Mode==5 )
		{
			Servo_Angle2(135);	
			Servo_Angle3(135);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=5)break;
			Servo_Angle1(45);	
			Servo_Angle4(45);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=5)break;
			Servo_Angle2(90);	
			Servo_Angle3(90);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=5)break;
			Servo_Angle1(90);	
			Servo_Angle4(90);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=5)break;
			
			Servo_Angle1(135);	
			Servo_Angle4(135);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=5)break;
			Servo_Angle2(45);	
			Servo_Angle3(45);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=5)break;
			Servo_Angle1(90);	
			Servo_Angle4(90);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=5)break;
			Servo_Angle2(90);	
			Servo_Angle3(90);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=5)break;
			
			PAnumbers--;
		}
		if(Sustainedmove!=1 && Action_Mode==5)
		Action_Mode=2;
 
		
	}
}
 
void Action_Lrotation(void)//向左旋转
{
	while(Action_Mode==6)
	{
		PAnumbers=Chongfunumber;
		PAnumbers=PAnumbers+Chongfunumber;
		while((PAnumbers || Sustainedmove) && Action_Mode==6)
		{
			Servo_Angle2(45);
			Servo_Angle3(135);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=6)break;
			Servo_Angle1(45);
			Servo_Angle4(135);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=6)break;
			Servo_Angle2(90);
			Servo_Angle3(90);	
			Delay_ms(SpeedDelay);
			if(Action_Mode!=6)break;
			Servo_Angle1(90);
			Servo_Angle4(90);	
			Delay_ms(SpeedDelay);
			if(Action_Mode!=6)break;
			
			PAnumbers--;
		}
		if(Sustainedmove!=1 && Action_Mode==6)
		Action_Mode=2;
	}
 
}
 
 
void Action_Rrotation(void)//向右旋转
{
	while(Action_Mode==7)
	{
		PAnumbers=Chongfunumber;
		PAnumbers=PAnumbers+Chongfunumber;
		while((PAnumbers || Sustainedmove)  && Action_Mode==7)
		{
			Servo_Angle1(45);
			Servo_Angle4(135);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=7)break;
			Servo_Angle2(45);
			Servo_Angle3(135);
			Delay_ms(SpeedDelay);
			if(Action_Mode!=7)break;
			Servo_Angle1(90);
			Servo_Angle4(90);	
			Delay_ms(SpeedDelay);
			if(Action_Mode!=7)break;
			Servo_Angle2(90);
			Servo_Angle3(90);	
			Delay_ms(SpeedDelay);
			if(Action_Mode!=7)break;
			
			PAnumbers--;
		}
		if(Sustainedmove!=1 && Action_Mode==7)
		Action_Mode=2;
	}
 
}
 
void Action_Swing(void)//摇摆
{
	uint16_t SwingNumber=SwingRepeatnumber;
	while(SwingNumber && Action_Mode==8)
	{
		for(uint8_t i=30;i<150;i++)
		{
			Servo_Angle1(i);
			Servo_Angle2(i);
			Servo_Angle3(i);
			Servo_Angle4(i);
			Delay_ms(SwingDelay);
			if(Action_Mode!=8)break;
		}
		if(Action_Mode!=8)break;
		for(uint8_t i=150;i>30;i--)
		{
			Servo_Angle1(i);
			Servo_Angle2(i);
			Servo_Angle3(i);
			Servo_Angle4(i);
			Delay_ms(SwingDelay);
			if(Action_Mode!=8)break;
		}
		if(Action_Mode!=8)break;
		
		SwingNumber--;
	}
		for(uint8_t i=30;i<90;i++)
		{
			Servo_Angle1(i);
			Servo_Angle2(i);
			Servo_Angle3(i);
			Servo_Angle4(i);
			Delay_ms(SwingDelay);
			if(Action_Mode!=8)break;
		}
		if(Action_Mode==8)
		Action_Mode=2;
}
 
 
void Action_SwingTail(void)//摇尾巴
{
	uint16_t SwingTailNumber=3;
	Delay_ms(60);
	while(SwingTailNumber && Action_Mode==9)
	{
		for(uint8_t i=30;i<150;i++)
		{
			WServo_Angle(i);
			Delay_ms(SwingDelay);
			if(Action_Mode!=9)break;
		}
		if(Action_Mode!=9)break;
		for(uint8_t i=150;i>30;i--)
		{
			WServo_Angle(i);
			Delay_ms(SwingDelay);
			if(Action_Mode!=9)break;
		}
		if(Action_Mode!=9)break;
		SwingTailNumber--;
	}
	Delay_ms(60);
}
 
void Action_JumpU(void)//向前跳
{
	if(TiaoTurn==0)
	{
		Servo_Angle1(140);
		Servo_Angle4(35);
		Delay_ms(SpeedDelay);
		
		Servo_Angle2(140);
		Servo_Angle3(35);
		Delay_ms(SpeedDelay+80);
		
		Action_Mode=2;
		TiaoTurn=1;
	}
	else
	{
		Servo_Angle2(140);
		Servo_Angle3(35);
		Delay_ms(SpeedDelay);
		
		Servo_Angle1(140);
		Servo_Angle4(35);
		Delay_ms(SpeedDelay+80);
		
		Action_Mode=2;
		TiaoTurn=0;
	}
}
 
void Action_JumpD(void)//向后跳
{
	if(TiaoTurn2==0){
		Servo_Angle4(35);
		Servo_Angle1(140);
		Delay_ms(SpeedDelay);
	
		Servo_Angle3(35);
		Servo_Angle2(140);
		Delay_ms(SpeedDelay);
	
		Action_Mode=12;
		TiaoTurn2=1;
	}
	else
	{
		Servo_Angle3(35);
		Servo_Angle2(140);
		Delay_ms(SpeedDelay);
	
		Servo_Angle4(35);
		Servo_Angle1(140);
		Delay_ms(SpeedDelay);
	
		Action_Mode=12;
		TiaoTurn2=0;
	}
}
 
void Action_Hello(void)
{
	uint16_t HelloNumber=HelloRepeatnumber;
	
	Servo_Angle3(20);
	Servo_Angle4(45);
	Delay_ms(80);
	Servo_Angle1(90);
	while(HelloNumber && Action_Mode==13)
	{
		if(Action_Mode!=13)break;
		for(int i=0;i<=45;i++)
		{
			if(Action_Mode!=13)break;
			Servo_Angle2(i);
			Delay_ms(SwingDelay);
		}
		for(int i=45;i>0;i--)
		{
			if(Action_Mode!=13)break;
			Servo_Angle2(i);
			Delay_ms(SwingDelay);
		}
		if(Action_Mode!=13)break;
		
		HelloNumber--;
	}
	if(Action_Mode==13)
	Action_Mode=2;
}
 
void Action_stretch(void)//伸懒腰
{
	Servo_Angle3(90);
	Servo_Angle4(90);
	Delay_ms(80);
	for(int i=90;i>10;i--)
	{
		Servo_Angle1(i);
		Servo_Angle2(i);
		if(Action_Mode!=14)break;
		Delay_ms(15);
	}
	for(int i=10;i<90;i++)
	{
		Servo_Angle1(i);
		Servo_Angle2(i);
		if(Action_Mode!=14)break;
		Delay_ms(15);
	}
	for(int i=90;i<170;i++)
	{
		Servo_Angle3(i);
		Servo_Angle4(i);
		if(Action_Mode!=14)break;
		Delay_ms(15);
	}
	
	for(int i=170;i>90;i--)
	{
		Servo_Angle3(i);
		Servo_Angle4(i);
		if(Action_Mode!=14)break;
		Delay_ms(15);
	}
	if(Action_Mode==14)
	Action_Mode=15;
}
 
void Action_Lstretch(void)//后腿拉伸
{
	int breakvalue=1;
	int temp=3;
	while(breakvalue)
	{
		Servo_Angle1(90);
		Servo_Angle2(20);
		Delay_ms(60);
		Servo_Angle4(110);
		for(int i=90;i<180;i++)
		{
			if(Action_Mode!=15)break;
			Servo_Angle3(i);
			Delay_ms(6);
		}
		while(temp && Action_Mode==15)
		{
			for(int i=180;i>150;i--)
			{
				if(Action_Mode!=15)break;
				Servo_Angle3(i);
				Delay_ms(15);
			}
			temp--;
		}
		if(Action_Mode!=15)break;
		Delay_ms(100);
		Servo_Angle1(90);
		Servo_Angle2(90);
		if(Action_Mode!=15)break;
		Delay_ms(80);
		Servo_Angle3(90);
		Servo_Angle4(90);
		Delay_ms(100);
		if(Action_Mode!=15)break;
		
		temp=3;
		
		Servo_Angle2(90);
		Servo_Angle1(20);
		if(Action_Mode!=15)break;
		Delay_ms(60);
		Servo_Angle3(110);
		for(int i=90;i<180;i++)
		{
			if(Action_Mode!=15)break;
			Servo_Angle4(i);
			Delay_ms(6);
		}
		while(temp && Action_Mode==15)
		{
			for(int i=180;i>150;i--)
			{
				if(Action_Mode!=15)break;
				Servo_Angle4(i);
				Delay_ms(15);
			}
			temp--;
		}
		if(Action_Mode==15)
		Action_Mode=2;
		
		
		breakvalue=0;
	}
}
```

### 蓝牙与语音

USART1配置语音模块的串口，因为用的语音模块是机芯智能商家的su-03t1模块，只能通过USART1串口通道

USART3配置蓝牙模块的串口（USART2被定时器2占用了）

中断函数中，是改变Action_Mode的值与Face_Mode的值，并进入Face_Config()函数来更改表情。主循环遍历根据Action_Mode的值来改变动作。
```c
#include "stm32f10x.h"                  // Device header
#include "PWM.h"
#include "PetAction.h"
#include "Face_Config.h"
 
uint16_t AllLed=1;  //开启灯光
uint16_t BreatheLed=0;//开启呼吸灯
uint16_t Sustainedmove=0;//持续运动
 
 
uint16_t Action_Mode=0;
uint16_t SpeedDelay=200;
uint16_t SwingDelay=6;
uint16_t Face_Mode=0;
uint8_t WeiBa=0;
 
void BlueTooth_Init(void)
{
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA,ENABLE);//开启GPIOA时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1,ENABLE);//开启串口时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB,ENABLE);//开启GPIOB时钟
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART3,ENABLE);//开启串口时钟
	//语音
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode=GPIO_Mode_AF_PP;//复用推挽输出模式
	GPIO_InitStructure.GPIO_Pin=GPIO_Pin_9;//默认PA9是USART1_TX的复用
	GPIO_InitStructure.GPIO_Speed=GPIO_Speed_50MHz;
	GPIO_Init(GPIOA,&GPIO_InitStructure);
	GPIO_InitStructure.GPIO_Mode=GPIO_Mode_IPU;//复用上拉输入模式
	GPIO_InitStructure.GPIO_Pin=GPIO_Pin_10;//默认PA9是USART1_RX的复用
	GPIO_InitStructure.GPIO_Speed=GPIO_Speed_50MHz;
	GPIO_Init(GPIOA,&GPIO_InitStructure);
	//蓝牙
	GPIO_InitStructure.GPIO_Mode=GPIO_Mode_AF_PP;//复用推挽输出模式
	GPIO_InitStructure.GPIO_Pin=GPIO_Pin_10;//默认PB10是USART3_TX的复用
	GPIO_InitStructure.GPIO_Speed=GPIO_Speed_50MHz;
	GPIO_Init(GPIOB,&GPIO_InitStructure);
	GPIO_InitStructure.GPIO_Mode=GPIO_Mode_IPU;//复用上拉输入模式
	GPIO_InitStructure.GPIO_Pin=GPIO_Pin_11;//默认PB11是USART3_RX的复用
	GPIO_InitStructure.GPIO_Speed=GPIO_Speed_50MHz;
	GPIO_Init(GPIOB,&GPIO_InitStructure);
	
	USART_InitTypeDef UASRT_InitStructure;//USART初始化
	UASRT_InitStructure.USART_BaudRate=9600;//波特率9600
	UASRT_InitStructure.USART_HardwareFlowControl=USART_HardwareFlowControl_None;//不需要硬件流控制
	UASRT_InitStructure.USART_Mode=USART_Mode_Rx|USART_Mode_Tx;//接受与发送均打开
	UASRT_InitStructure.USART_Parity=USART_Parity_No;//不需要奇偶校验
	UASRT_InitStructure.USART_StopBits=USART_StopBits_1;//停止位为1
	UASRT_InitStructure.USART_WordLength=USART_WordLength_8b;//字长8位
	USART_Init(USART1,&UASRT_InitStructure);
	USART_Init(USART3,&UASRT_InitStructure);
	
	USART_ITConfig(USART1,USART_IT_RXNE,ENABLE);//语音接收中断配置，也就是如果接送到消息就直接中断
	USART_ITConfig(USART3,USART_IT_RXNE,ENABLE);//蓝牙接收中断配置，也就是如果接送到消息就直接中断
	
	//中断
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);//分组2
	//语音中断配置
	NVIC_InitTypeDef NVIC_InitStructure;
	NVIC_InitStructure.NVIC_IRQChannel=USART1_IRQn;//特定的通道
	NVIC_InitStructure.NVIC_IRQChannelCmd=ENABLE;//通道使能
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority=1;//抢占式优先级为1
	NVIC_InitStructure.NVIC_IRQChannelSubPriority=1;//响应优先级为1
	NVIC_Init(&NVIC_InitStructure);
	//蓝牙中断配置
	NVIC_InitStructure.NVIC_IRQChannel=USART3_IRQn;//特定的通道
	NVIC_InitStructure.NVIC_IRQChannelCmd=ENABLE;//通道使能
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority=2;//抢占式优先级为2
	NVIC_InitStructure.NVIC_IRQChannelSubPriority=1;//响应优先级为1
	NVIC_Init(&NVIC_InitStructure);
	
	USART_Cmd(USART1,ENABLE);//USART1使能打开
	USART_Cmd(USART3,ENABLE);//USART3使能打开
}
 
void USART1_IRQHandler(void)
{
	if(USART_GetITStatus(USART1,USART_IT_RXNE)==SET)//如果接受到
	{
		Sustainedmove=0;//持续运动关闭
		
		if(USART_ReceiveData(USART1)==0x29)//放松的趴下
		{
			Face_Mode=0;
			Face_Config();
			Action_Mode=0;
		}
		else if(USART_ReceiveData(USART1)==0x30)//蹲下
		{
			Face_Mode=1;
			Face_Config();
			Action_Mode=1;
		}
		else if(USART_ReceiveData(USART1)==0x31)//直立
		{
			Face_Mode=5;
			Face_Config();
			Action_Mode=2;
 
		}
		else if(USART_ReceiveData(USART1)==0x32)//趴下
		{
			Face_Mode=1;
			Face_Config();
			Action_Mode=3;
		}
		else if(USART_ReceiveData(USART1)==0x33)//前进
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=4;
		}
		else if(USART_ReceiveData(USART1)==0x34)//后退
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=5;
		}
		else if(USART_ReceiveData(USART1)==0x35)//左转
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=6;
		}
		else if(USART_ReceiveData(USART1)==0x36)//右转
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=7;
		}
		else if(USART_ReceiveData(USART1)==0x37)//摇摆
		{
			Face_Mode=4;
			Face_Config();
			Action_Mode=8;
		}
		else if(USART_ReceiveData(USART1)==0x38)//减少移动延迟，增加移动速度
		{
			if(SpeedDelay==120){Face_Mode=3;Face_Config();}
			if(SpeedDelay>100)
			SpeedDelay-=20;	
			else
			{
				Face_Mode=2;
				Face_Config();
				SpeedDelay=200;
			}
 
				
		}
		else if(USART_ReceiveData(USART1)==0x39)//减少摇摆延迟，增加摇摆速度
		{
			if(SwingDelay==4){Face_Mode=3;Face_Config();}
			if(SwingDelay>3)
			SwingDelay--;	
			else
			{	
				Face_Mode=4;
				Face_Config();
				SwingDelay=9;
			}
		}
		
		else if(USART_ReceiveData(USART1)==0x40)//摇尾巴
		{
			(WeiBa==0)?(WeiBa=1):(WeiBa=0);
			Face_Mode=1;
			Face_Config();
			Action_Mode=9;
		}
		
		else if(USART_ReceiveData(USART1)==0x41)//向前跳
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=10;
		}
		
		else if(USART_ReceiveData(USART1)==0x42)//向后跳
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=11;
		}
		else if(USART_ReceiveData(USART1)==0x43)//打招呼
		{
			Face_Mode=6;
			Face_Config();
			Action_Mode=13;
		}
		else if(USART_ReceiveData(USART1)==0x44)//开启灯光
		{
			AllLed=1;
		}
		else if(USART_ReceiveData(USART1)==0x45)//关闭灯光
		{
			AllLed=0;
		}
		else if(USART_ReceiveData(USART1)==0x46)//开启呼吸灯
		{
			BreatheLed=1;
		}
		else if(USART_ReceiveData(USART1)==0x47)//关闭呼吸灯
		{
			BreatheLed=0;
		}
		else if(USART_ReceiveData(USART1)==0x48)//伸懒腰
		{
			Face_Mode=6;
			Face_Config();
			Action_Mode=14;
		}
		else if(USART_ReceiveData(USART1)==0x49)//伸懒腰
		{
			Face_Mode=6;
			Face_Config();
			Action_Mode=15;
		}
		USART_ClearITPendingBit(USART1,USART_IT_RXNE);
	}
	
}
 
void USART3_IRQHandler(void)
{
	if(USART_GetITStatus(USART3,USART_IT_RXNE)==SET)//如果接受到
	{
		Sustainedmove=1;//持续运动开启
		
		if(USART_ReceiveData(USART3)==0x29)//放松的趴下
		{
			Face_Mode=0;
			Face_Config();
			Action_Mode=0;
		}
		else if(USART_ReceiveData(USART3)==0x30)//蹲下
		{
			Face_Mode=1;
			Face_Config();
			Action_Mode=1;
		}
		else if(USART_ReceiveData(USART3)==0x31)//直立
		{
			Face_Mode=5;
			Face_Config();
			Action_Mode=2;
 
		}
		else if(USART_ReceiveData(USART3)==0x32)//趴下
		{
			Face_Mode=1;
			Face_Config();
			Action_Mode=3;
		}
		else if(USART_ReceiveData(USART3)==0x33)//前进
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=4;
		}
		else if(USART_ReceiveData(USART3)==0x34)//后退
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=5;
		}
		else if(USART_ReceiveData(USART3)==0x35)//左转
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=6;
		}
		else if(USART_ReceiveData(USART3)==0x36)//右转
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=7;
		}
		else if(USART_ReceiveData(USART3)==0x37)//摇摆
		{
			Face_Mode=4;
			Face_Config();
			Action_Mode=8;
		}
		else if(USART_ReceiveData(USART3)==0x38)//减少移动延迟，增加移动速度
		{
			if(SpeedDelay==120){Face_Mode=3;Face_Config();}
			if(SpeedDelay>100)
			SpeedDelay-=20;	
			else
			{
				Face_Mode=2;
				Face_Config();
				SpeedDelay=200;
			}
 
				
		}
		else if(USART_ReceiveData(USART3)==0x39)//减少摇摆延迟，增加摇摆速度
		{
			if(SwingDelay==4){Face_Mode=3;Face_Config();}
			if(SwingDelay>3)
			SwingDelay--;	
			else
			{	
				Face_Mode=4;
				Face_Config();
				SwingDelay=9;
			}
		}
		
		else if(USART_ReceiveData(USART3)==0x40)//摇尾巴
		{
			(WeiBa==0)?(WeiBa=1):(WeiBa=0);
			Face_Mode=1;
			Face_Config();
			Action_Mode=9;
		}
		
		else if(USART_ReceiveData(USART3)==0x41)//向前跳
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=10;
		}
		
		else if(USART_ReceiveData(USART3)==0x42)//向后跳
		{
			Face_Mode=2;
			Face_Config();
			Action_Mode=11;
		}
		else if(USART_ReceiveData(USART3)==0x43)//打招呼
		{
			Face_Mode=6;
			Face_Config();
			Action_Mode=13;
		}
		else if(USART_ReceiveData(USART3)==0x44)//开启灯光
		{
			AllLed=1;
		}
		else if(USART_ReceiveData(USART3)==0x45)//关闭灯光
		{
			AllLed=0;
		}
		else if(USART_ReceiveData(USART3)==0x46)//开启呼吸灯
		{
			BreatheLed=1;
		}
		else if(USART_ReceiveData(USART3)==0x47)//关闭呼吸灯
		{
			BreatheLed=0;
		}
		else if(USART_ReceiveData(USART3)==0x48)//伸懒腰
		{
			Face_Mode=6;
			Face_Config();
			Action_Mode=14;
		}
		else if(USART_ReceiveData(USART3)==0x49)//拉伸腿
		{
			Face_Mode=6;
			Face_Config();
			Action_Mode=15;
		}
		
		USART_ClearITPendingBit(USART3,USART_IT_RXNE);
	}
}
```

头文件

```c
#ifndef __BLUE_TOOTH_H
#define __BLUE_TOOTH_H
 
extern uint16_t Action_Mode;
extern uint16_t Face_Mode;
extern uint16_t SpeedDelay;
extern uint16_t SwingDelay;
extern uint8_t WeiBa;
extern uint16_t AllLed;  //开启灯光
extern uint16_t BreatheLed;//开启呼吸灯
extern uint16_t Sustainedmove;
 
void BlueTooth_Init(void);
 
#endif
```

### 表情代码函数

OLED模块代码是用的江协老师的，OLED显示图片教程也是跟着这个老师学的，表情的图片可以从网上寻找DIY表情。

```c
#include "stm32f10x.h"      
#include "OLED.h"
#include "BlueTooth.h"
 
//实现表情变化，调节是进中断后
 
void Face_Config(void)
{
	if(Face_Mode==0)
	{	
		OLED_Clear();
		OLED_ShowImage(0,0,128,64,Face_sleep);//睡觉
		OLED_Update();
	}
	else if(Face_Mode==1)
	{	
		OLED_Clear();
		OLED_ShowImage(0,0,128,64,Face_stare);//瞪大眼
		OLED_Update();
	}
	else if(Face_Mode==2)
	{
		OLED_Clear();
		OLED_ShowImage(0,0,128,64,Face_happy);//快乐
		OLED_Update();
	}
	else if(Face_Mode==3)
	{
		OLED_Clear();
		OLED_ShowImage(0,0,128,64,Face_mania);//狂热
		OLED_Update();
	}
	else if(Face_Mode==4)
	{
		OLED_Clear();
		OLED_ShowImage(0,0,128,64,Face_very_happy);//非常快乐
		OLED_Update();
	}
	else if(Face_Mode==5)
	{
		OLED_Clear();
		OLED_ShowImage(0,0,128,64,Face_eyes);//眼睛
		OLED_Update();
	}
	else if(Face_Mode==6)
	{
		OLED_Clear();
		OLED_ShowImage(0,0,128,64,Face_hello);//打招呼
		OLED_Update();
	}
}
```

头文件

```c
#ifndef __FACE_CONFIG_H
#define __FACE_CONFIG_H
 
void Face_Config(void);
 
#endif
```

## 小狗外壳

这里是直接买的淘宝开模好的模型把硬件组装进去

<img src="C:\Users\admin\Desktop\智能小狗\img\image-20250805101821822-1754364784831-1.png" alt="image-20250805101821822" style="zoom:50%;" />



总体模块大概是五个舵机，一个OLED显示屏，蓝牙模块，语音模块
