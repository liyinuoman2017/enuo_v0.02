# enuo_2
从零开始构建嵌入式实时操作系统2——重构

![在这里插入图片描述](https://img-blog.csdnimg.cn/9dd36edc3569476899707ab048ba917b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)



## 1.前言

本人是一个普通的中年程序员，并不是圈内的大牛，写嵌入式操作系统这一系列的文章并不是要显示自己的技术，而是出于对嵌入式的热爱。非常幸运，本人毕业后的十几年一直从事嵌入式行业，遇到过各种坑，也收获过各种喜悦。希望嵌入式操作系统系列文章能对其它的嵌入式爱好者能有所帮助，帮助热爱嵌入式行业的朋友快速了解嵌入式操作系统的运行原理。

我将一步一步地完善我们的**嵌入式实时操作系统enuo**，每完成一步软件的构建，我将输出一篇总结性的文件，来分享软件构建过程，并开源软件工程和源码。

操作系统enuo的名字来源于我5岁儿子的伊诺，希望在我的守护下enuo和伊诺都能健康快乐，茁壮成长！

![在这里插入图片描述](https://img-blog.csdnimg.cn/212f0902aba94e4eafc15d2d2c75fe22.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_12,color_FFFFFF,t_70,g_se,x_16)

## 2.设计背景

书接上文我们完成了一个可以实现任务切换的软件工程V0.01版本。V0.01版本的软件工程中包含：main.c ，startup_stm32f401xc.s 和 readme三个文件。startup_stm32f401xc.s 文件为STM32F401的启动文件，main.c文件实现任务初始化，任务切换和任务轮询调度功能，readme文件用于记录版本修改日志。
V0.01版只能算一个功能验证型软件，**接下来需要使用正规的软件设计方法来改造和重构整个工程，使软件系统具有较高的扩展性，移植性，复用性和可读性**。


## 3.设计目标

首先运用软件设计五大原则中的**单一原则**，建立一个独立的**文件夹enuo**用于存放于操作系统相关的源文件，**每个源文件完成一个单一的功能**。使用“**分而治之**”的设计思维，并将操作系统分为多个功能单一的模块，每个模块以一个C文件的形式承载，这样就提高了软件的**可读性和移植性**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/747a2940bbf741ca915fb5948a6376e1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

其次同时使用**面向对象的设计思维**，将任务设计为一个抽象的对象，任务对象将任务的信息封装起来，这样就提高了软件的**扩展性和复用性**。虽然C语言不是面向对象编程语言，但是通过一些设计技巧可以实现面向对象设计。

最后构建一个**任务链表用于加入和删除任务**。链表数据结构可以在保留原有物理顺序的情况下，高效的实现插入和删除。


## 4.设计环境

硬件环境是使用STM32F401RE为核心的自制开发板。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7d53576eb79e4a4691c42f5afa9853a3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

软件环境是使用的KEIL V5.2 开发工具。

![在这里插入图片描述](https://img-blog.csdnimg.cn/d6aca02319a34e3c9a2c5cb95f8c942b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_15,color_FFFFFF,t_70,g_se,x_16)

## 5.设计过程

**5.1构建任务对象**
任务对象包含一个栈指针和一个任务链表，其定义如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/41e8ad51d190413c86c4e2f1b60285be.png)

**任务栈指针为第一个元素，这样栈指针就和任务对象为同一个地址（结构体的第一个元素就是结构体的首地址），这样就可以极大的简化任务切换过程中对栈指针的操作。**
任务栈指针指向用户为任务定义的静态数据块，通常情况下任务栈是一个全局静态数组，数组的大小就是任务栈的大小。

![在这里插入图片描述](https://img-blog.csdnimg.cn/e02eecb740f34fe195ebf3dffdcac29b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

任务链表的作用是将多个任务串联起来，**方便依次检索和操作**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/da12dd8d6603472b85e57d8b3b578386.png)

**5.2构建链表结构**
我们使用链表结构来存放任务对象，使用**链表结构有如下优势**：
1、在保留原有物理顺序的情况下，插入和删除速度快，效率高。插入和删除只需要改变几个指针变量。
2、链表中的表项数量没有上限。存储的表项上限只与内存空间大小有关，理论上如果内存无限大，链表中的表项可以动态增加到无限个。
3、动态分配内存，需要用多少个表项，就分配几个表项，不需要预先分配内存，不存在内存浪费的情况。
链表结构定义如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20cae1977e2741bb8f092040555bf8a7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

链表采用的是数据中包含链表的数据结构，采用这种方式的**优点**是：**当用户数据结构改变时，整个链表结构可以保持不变，不同的用户数据可以通用这一个的链表结构**。示意图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/dca8440d282943c4b2ca0a7ab9e17363.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

数据中包含链表的数据结构，在操作链表后无法得到整个数据对象的地址。由下图可知图可知我们通过next指针可以得到下一个list元素的地址，但是**我们无法获取整个数据对象的地址**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/4e33955a71c9404a8800b47ad1b69b3b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

在链表结构种增加一个void * owner指针，void * 类型指针可以指向任意类型的对象，用这个指针指向整个数据对象的地址,这样就可以**定位到整个数据对象的地址**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/5fddd76bd94f4244a1e143252864f690.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

```c
struct list_node_def
{
	struct list_node_def *  next;		/* 指向下一个列表节点 */
	struct list_node_def *  previous;	/* 指向上一个列表节点 */
	void 	*owner;						/* 指向链表节点数据结构 */
};
```

**哨兵机制（表头机制）**
哨兵是一个哑对象，作用是简化边界条件处理。**使用哨兵机制（表头机制）后链表在空状态和非空状态插入和删除对象的操作是相同的，这样可以使得代码紧凑，清晰。**

```c
typedef struct list_def
	{
		list_node_t   	*index;			/* 索引指针 */
		list_node_t 	head;			/* 列表头 */
	} list_t;
```

**5.3任务初始化**
任务初始化代码如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/212d73c1a71843b4ac3115ea40643013.png)

在创建任务之前，需要定义任务栈空间和用户任务函数：

```c
/*********************************************************************************************************
* @名称	: enuo
* @作者	: 李巍
**********************************************************************************************************/

#define STACK_NUM 	(64)
/* 任务0-任务2栈空间 */
uint32_t task0_stack[STACK_NUM];		
uint32_t task1_stack[STACK_NUM];		
uint32_t task2_stack[STACK_NUM];

/* 任务0-任务2对象*/
task_tcb_t  my_task0;
task_tcb_t  my_task1;
task_tcb_t  my_task2;

void task0(void)
{
	static uint16_t clk = 0;

	while(1)
	{
		if( ( ( clk++ )%9999 ) == 0 ) 
		{
			task_debug_num0++;	/* 测试跟踪 */
			test_function();
		}
	}
}
```

**task_create函数代码如下：**

```c
void task_create(task_tcb_t *task , task_function_t function ,uint32_t *stack_space ,uint32_t stack_number)
{
	list_node_t * node_tail = &task_list.head;
	/* 寻求列表尾端 */
	while( node_tail->next != NULL )
		node_tail = node_tail->next;
	/* 任务列表加入下一个任务 */
	node_tail->next = &task->task_list;
	/* 列表所有者指针指向任务  */
	task->task_list.owner = task;
	/* 当前任务下个列表指针设置为空  */
	task->task_list.next = NULL;	
	/* 初始化任务栈  */	
	task_stack_init( (uint32_t *)task, function , stack_space , stack_number );	
}
```
task_create函数完成任务**链表操作**，**初始化任务栈指针**和**任务栈空间**。
task_stack_init代码如下：

```c
void task_stack_init(uint32_t 		*stack_pointer , 
					task_function_t task ,
					uint32_t 		*stack_space ,
					uint32_t 		stack_number)
{
	/* 将任务psp栈指针指向任务栈底部*/	
	*stack_pointer =  ( (uint32_t)stack_space + ( (stack_number - 16)<<2 ) );
	/* 初始化任务栈中的程序寄存器 */
	*( (uint32_t *)( (uint32_t )( *stack_pointer) + (14<<2) ) ) = ( uint32_t )task;
	/* 初始化任务栈中的XPSR*/
	*( (uint32_t *)( (uint32_t )( *stack_pointer) + (15<<2) ) ) = 0x01000000;
}
```
**5.4开始调度任务**
**完成任务创建之后，就可以开始调度任务**，开始调度任务代码如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/7e2f5f4bbd2d4111affd45834a89e4da.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

enuo_schedule函数代码如下：

```c
/*********************************************************************************************************
* @名称	: enuo
* @作者	: 李巍
**********************************************************************************************************/
__asm   void  start_schedule(void)
{
	/* 设置CONTROL寄存器 配置PSP栈指针模式 */
	MOV R0 ,#0X02
	MSR CONTROL,R0
	
	/*读取current_task 地址 */
	LDR R3, =__cpp(&current_task)         
	/* 读取curr_task中的PSP指针数值 */
	LDR R1,[R3]                      
	LDR R0,[R1]
	
	/* 出栈R4-R11八个寄存器 */
	LDMIA R0!,{R4-R11}                 
	/* 设置PSP指针 */
	MSR PSP,R0	
	/* POP 寄存器  POP  PC实现跳转 */
	POP { R0-R3 , R12 ,R11 ,PC }
	/* 对齐 */
	ALIGN 4
}
```
enuo_schedule函数主要完成3个功能：
**1、设置处理器栈指针模式。
2、读取current_task 地址。
3、恢复任务寄存器，POP  PC实现跳转用户任务代码。**

**5.5调度任务方法**
任务调度采用**时间片轮转调度**，使用SysTick定时器产生定时中断，在定时器中断函数中依次从task_list任务链表中读取任务对象，并用next_task指向新读出的任务对象，最后挂起PendSV系统中断标志位，当SysTick定时器中断函数退出时执行PendSV_Handler进行任务切换。
SysTick定时器中断函数代码如下：

```c
/*********************************************************************************************************
* @名称	: enuo
* @作者	: 李巍
**********************************************************************************************************/
void SysTick_Handler(void)
{
	
	static list_node_t * node_tail = &task_list.head;
	/* 轮流切换任务 */
	if(node_tail->next != NULL  )
	{
		/* 当下一个列表项不为NULL时  next_task为下一个列表项指向的任务*/
		next_task = node_tail->next->owner;
		/* 更新任务指针  */
		node_tail = node_tail->next;
	}
	else
	{
		/* 当下一个列表项为NULL时  ，node_tail指向任务列表的表头*/
		node_tail = &task_list.head;
		/*  next_task为表头指向的下一个的任务*/
		next_task = node_tail->next->owner;
		node_tail = node_tail->next;
	}
	
	/* PendSV系统中断置位 */
	SCB->ICSR |=SCB_ICSR_PENDSVSET_Msk;	
	return;
}
```
**5.6任务切换**
任务切换在PendSV_Handler异常（中断）中任务切换。PendSV_Handler函数代码如下：

```c
/*********************************************************************************************************
* @名称	: enuo
* @作者	: 李巍
**********************************************************************************************************/
__asm   void  PendSV_Handler(void)
{
	/* 读取当前进程栈指针数值 */
	MRS R0,PSP                        	
	/* 保存R4-R11八个寄存器的值到当前任务栈中  同时将回写的地址写入R0 */
	STMDB R0!,{R4-R11} 	

	/* 读取current_task 地址 */
	LDR R3, =__cpp(&current_task)         
	/* 将当前进程PSP指针值 写入 相应的 current_task   */
	STR  R0,[R3] 

	
	/* 获取next_task 地址 */
	LDR R4,=__cpp(&next_task)          
	LDR R4,[R4]
	/* 读取next_task中的PSP指针 */
	LDR R0,[R4] 

	/* 出栈 R4-R11八个寄存器 */
	LDMIA R0!,{R4-R11}                 
	/* 设置PSP指针 */
	MSR PSP,R0	
	/* 中断返回 */
	BX LR 
	/* 对齐 */	
	ALIGN 4
}
```
PendSV_Handler函数完成以下3个功能：
**1、进入PendSV_Handler中断前处理器自动保存了R0,R1,R2,R3, R12,LR,PC,XPSR，进入中断后完成R4~R11入栈保存工作，从而实现任务保存工作。
2、读取current_task 地址将当前进程PSP指针值保存到current_task指向的任务对象中。读取next_task指向的任务对象，并加载任务对象的PSP指针值。
3、出栈 R4-R11八个寄存器，设置PSP指针。中断返回时处理器自动保存了R0,R1,R2,R3 ,R12, LR,PC,XPSR，从而实现任务恢复工作。**


## 6.运行结果

代码仿真运行后的结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/5e78f1b6fb9546418d95c6dfbec9a5a7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

**运行结果反应创建的3个任务都得到了调度，说明enuo系统正常工作。**
enuo系统目前包括3个文件：
**1、task.h文件**
本文件中包含**任务对象**的定义，链表数据结构的定义和任务列表数据结构的定义。

**2、task.c文件**
本文件中包含task_create和enuo_schedule两个函数。task.c文件只用于存放与任务操作相关的函数(符合单一原则)。

**3、interface.c文件**
本文件中包含SysTick_Handler,PendSV_Handler，start_schedule和task_stack_init三个函数。interface.c文件只用与存放跟处理器相关的操作。后期更换处理器时只用修改这个文件即可，增强了系统的移植性。

![在这里插入图片描述](https://img-blog.csdnimg.cn/94980adf50e34aaa9ca2a7d7b2599266.png)

<font color=red>**希望获取源码的朋友们在评论区里留言。**

**未完待续…**
	
**实时操作系统系列将持续更新**
	
**创作不易希望朋友们点赞，转发，评论，关注。**
	
**您的点赞，转发，评论，关注将是我持续更新的动力**
	
**作者：李巍**
	
**Github：liyinuoman2017**
	
**CSDN：liyinuo2017**
	
**今日头条：程序猿李巍**
