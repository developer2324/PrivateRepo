# 《Huawei LiteOS入门指导》

## 一：获取Huawei LiteOS源码

LiteOS开源代码放置在Github进行托管，路径为https://github.com/LiteOS/LiteOS

开发者在使用LiteOS前，需要将代码下载本地(选择Clone or download)

- **了解Huawei LiteOS源代码目录结构**

LiteOS源代码目录包括arch、components、doc、examples、kernle、targets等目录。

详细介绍，请访问链接进一步了解
https://github.com/LiteOS/LiteOS/blob/master/doc/LiteOS_Code_Info.md

## 二：移植Huawei LiteOS到目标芯片（开发板）

- **移植前须知**

1. 明确芯片架构，选择合适的CPU代码，LiteOS开源项目目前支持ARM Cortex-M0，Cortex-M3，Cortex-M4，Cortex-M7等芯片架构；
	
2. 选择合适的IDE（目前开源示例IDE主要有MDK、IAR、GCC）；
	
3. 对于已经支持的CPU架构，建议参考doc目录下的移植文档进行移植；
	
4. 移植完成后，参考user目录下的main.c文件下的代码对内核进行测试，全部测试用例成功后才证明移植成功。
	
5. 已移植适配的开发板可以直接使用，具体请见LiteOS/targets/目录，例如STM32F103、STM32F429、STM32F746等开发板

- **移植常见步骤**

1. 适配系统调度汇编（los_dispatch.s） ，主要修改函数LOS_StartToRun、LOS_IntLock、LOS_IntUnLock、TaskSwitch等；
	
2. 根据芯片设置系统相关参数，包括时钟频率，tick中断配置，los_config.h系统参数配置（内存池大小、信号量、队列、互斥锁、软件定时器数量等）；
	
3. 适配中断管理模块 ，LiteOS的中断向量表由m_pstHwiForm[OS_VECTOR_CNT]数组管理，需要根据芯片配置中断使能，重定向等；

4. 适配系统log函数，方便OS问题追踪，修改los_printf.h文件，将系统的PRINT_INFO等函数映射到硬件串口；

## 三：创建Huawei LiteOS任务（线程）

- **了解LiteOS启动流程**

LiteOS入口在工程对应的main.c中

1. 首先进行硬件初始化 HardWare_Init();

2. 初始化LiteOS内核 LOS_KernelInit();

3. 初始化内核例程 LOS_Inspect_Entry();

4. 最后调用LOS_Start();开始task调度，LiteOS开始正常工作;


- **LiteOS任务创建流程**
		
1. 开发者编写用户任务函数（业务代码）；
	
2. 配置任务参数（堆栈、优先级，入口函数等）；
	
3. 在调用LOS_Start函数前调用LOS_TaskCreate函数创建任务。

- **LiteOS任务模块功能**
		
		- 任务的创建和删除
		LOS_TaskCreateOnly	创建任务，并使该任务进入suspend状态，并不调度
		LOS_TaskCreate	    创建任务，并使该任务进入ready状态，并调度
		 LOS_TaskDelete	    删除指定的任务

		- 任务状态控制
		LOS_TaskResume	    恢复挂起的任务
		LOS_TaskSuspend	    挂起指定的任务
		LOS_TaskDelay	    任务延时等待
		LOS_TaskYield       显式放权，调整指定优先级的任务调度顺序

		- 任务调度的控制
		LOS_TaskLock	    锁任务调度
		LOS_TaskUnlock	    解锁任务调度

		- 任务优先级的控制
		LOS_CurTaskPriSet	设置当前任务的优先级
		LOS_TaskPriSet	    设置指定任务的优先级
		LOS_TaskPriGet	    获取指定任务的优先级


## 四：熟悉Huawei LiteOS中断管理机制

LiteOS的中断模块支持中断初始化、创建、删除、开关恢复中断。

- **LiteOS中断开发流程**

		1. 在los_config.h中修改配置项：
		   打开硬中断裁剪开关：LOSCFG_PLATFORM_HWI定义为YES. 
			配置硬中断使用最大数：LOSCFG_PLATFORM_HWI_LIMIT。
		
		2. 中断初始化：
		   osHwiInit，该操作在系统初始化（los_config.c文件中完成）中由osmain函数调用，开发者一般不需要修改。
		
		3. 编写中断服务程序
		  void IRQ_Handler(void)。
		4. 创建中断
		  LOS_HwiCreate(IRQn, 0,0,IRQ_Handler,NULL)。
	
		5. 删除中断：
		  调用LOS_HwiDelete函数。

- **LiteOS中断模块功能**

		 LOS_HwiCreate: 硬中断创建，注册硬中断处理程序
		 LOS_IntUnLock:开中断
		 LOS_IntRestoreL:恢复到关中断之前的状态
		 LOS_IntLock:关中断
		 LOS_HwiDelete:硬中断删除

*注意：LiteOS的所有中断向量由m_pstHwiForm[]数组管理，osHwiInit将所有外部中断服务程序入口初始化为osHwiDefaultHandler，调用LOS_HwiCreate函数才会改变中断服务程序入口.*


## 五：熟悉Huawei LiteOS内存管理机制
内存模块用于管理系统的内存资源，是LiteOS的核心模块之一。主要包括内存的初始化、分配以及释放。
在系统运行过程中，内存管理模块通过对内存的申请/释放操作，来管理用户和OS对内存的使用，使内存的利用率和使用效率达到最优，同时最大限度地解决系统的内存碎片问题。

Huawei LiteOS的内存管理分为静态内存管理和动态内存管理，提供内存初始化、分配、释放等功能。

	动态内存：在动态内存池中分配用户指定大小的内存块。
	−	优点：按需分配。
	−	缺点：内存池中可能出现碎片。

	静态内存：在静态内存池中分配用户初始化时预设（固定）大小的内存块。
	−	优点：分配和释放效率高，静态内存池中无碎片。
	−	缺点：只能申请到初始化预设大小的内存块，不能按需申请。


- **LiteOS内存模块开发流程**

		1.在Los_config.h文件中配置参数：
		OS_SYS_MEM_ADDR：系统动态内存池起始地址
		OS_SYS_MEM_SIZE：  系统动态内存池大小，以byte为单位
		LOSCFG_BASE_MEM_NODE_INTEGRITY_CHECK：内存越界检测开关，默认关闭
		
		2.初始化内存池：
		LOS_MemInit(VOID *pPool, UINT32  uwSize) 
		// 该操作在系统初始化（los_config.c文件中完成）中由osMemSystemInit函数调用，开发者一般不需要修改。
		
		3.申请动态内存：
		 VOID *LOS_MemAlloc (VOID *pPool, UINT32  uwSize)；
		
		4.释放动态内存：
		LOS_MemFree(VOID *pPool, VOID *pMem)

- **LiteOS内存模块功能**

详细内容请参见开发指导手册。

- **LiteOS Kernel更多功能**

LiteOS的基础内核的模块还包括队列、事件、互斥锁、信号量，时间管理，软件定时器，双向链表，详细使用教程请参考Huawei LiteOS开发指导书。


## 六：熟悉Huawei LiteOS端云开发

Agent Tiny是华为物联网解决方案中，资源受限的终端对接到IoT云的重要组件，开发者只需调用几个简单的API接口，便可实现设备快速接入到华为IoT云平台（OceanConnect）以及数据上报和命令接收等功能。

Agent Tiny提供端云协同能力，集成了LWM2M、CoAP、mbedtls、LwIP全套IoT互联互通协议栈，且在LWM2M的基础上，提供了Agent Tiny开放API，用户只需关注自身的应用，而不必关注LwM2M实现细节，直接使用Agent Tiny 封装的API，通过四个步骤就能简单快速地实现与华为OceanConnect平台的安全可靠连接。使用Agent Tiny，用户可以大大减少开发周期，聚焦自己的业务开发，快速构建自己的产品。

- **LiteOS快速入云开发步骤**

<font color='red'>这里插入一张图：快速入云开发步骤</font>

- **LiteOS Agent Tiny端云组件适配层说明**

Agent Tiny为了方便移植，提供了常用硬件驱动适配接口以及网络适配接口

atiny_adapter（硬件驱动接口）：随机数、内存操作、log、数据存储、信号量、互斥锁

atiny_socket（网络层适配接口）：atiny_net_connect,atiny_net_send,atiny_net_recv,atiny_net_recv_timeout
