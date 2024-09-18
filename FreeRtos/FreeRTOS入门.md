# FreeRTOS入门

>简介：FreeRTOS 是一个嵌入式实时操作系统，其特点（免费开源、可裁剪、简单、优先级不限、任务不限、支持抢占/协程/时间片流转任务调度）

---

## 一、FreeRtos与裸机区别

### 1、裸机程序设计模式

> **裸机程序设计模式：轮询、前后台、定时器驱动、基于状态机**
>
>> 1. 轮询：在 main函数中是一个 while 循环，里面依次调用 2 个函数以上
>> 2. 前后台：main 函数里 while 循环里的代码是后台程序，通过中断来处理紧急任务（问题就是，若是中断程序时间太长，两个函数之间就会相互影响）---**使用FreeRtos程序设计---中断的时间不能太长、所以中断的任务就是置标志位**
>> 3. 定时器驱动：是前后台模式的一种，可以按照不用的频率执行各种函数，那么就可以启动一个定时器，让 它每 1 分钟产生一次中断，让中断函数在合适的时间调用对应函数，这种模式适合调用周期性的函数，但程序依旧不能很长，若是很长，依旧会进入轮询模式的问题---**通常将函数分解断**
>> 4. 基于状态机：函数内部拆分，每一小段时间执行一段代码，但是很多场景里，函数 A、B 并不容易拆分为多个状态，并且这 些状态执行的时间并不好控制

### 2、裸机和FreeRtos对比

>**裸机：**裸机又称为前后台系统，前台系统指的中断服务函数，后台系统指的大循环，即应用程序。 （特点）实时性差（应用程序、轮流执行）、delay（空等待，CPU不执行其他代码、浪费资源）、结构臃肿（实现功能都放在无限循环）

> **Freertos（就是实时操作系统，强调的是：实时性）：**（特点）分而治之（实现功能划分为多个任务）、延时函数（不会空等待，会让出CPU的使用权给其他任务）、抢占式（高优先级任务抢占低优先级任务）、任务堆栈（每个任务都有自己的栈空间，用于保存局部变量以及任务的上下文信息）

> 注意1：中断可以打断任意任务。
>
> 注意2：任务可以同等优先级（若高优先级不主动放弃调度，则会一直运行，使得低优先级任务无法运行）

---

## 二、任务调度

>简介：调度器就是使用相关的调度算法来决定当前需要执行的任务

>FreeRTOS 一共支持三种任务调度方式:
>
>>1. 抢占式调度：主要是针**对优先级不同的任务**，每个任务都有一个优先级，优先级高的任务可以抢占优先级低的任务。
>>
>>2. 时间片调度：主要针对**优先级相同的任务**，当多个任务的优先级相同时， 任务调度器会在每一次系统时钟节拍到的时候切换任务。
>>
>>3. 协程式调度：其实就是**轮询**，FreeRTOS现在虽然还支持，但是官方已经表示不再开发协程式调度
>
>抢占式调度：1、优先级任务，优先执行。2、高优先级任务不停止，低优先级任务无法执行 。3、被抢占CPU的任务将会进入就绪态
>
>  
>
>时间片调度：1、同等优先级任务，轮流执行；时间片流转。 2、**一个时间片大小，取决为滴答定时器中断频率**。 3、注意任务中途被打断或阻塞，没有用完的时间片不会再使用，下次该任务得到执行还是按照一个时间片的时钟节拍运行。

---

## 三、任务状态

>FreeRTOS中任务共存在4种状态：
>
>1. 运行态：正在执行的任务，该任务就处于运行态，注意在STM32中，同一时间仅一个任务处于运行态 
>
>2. 就绪态：如果该任务已经能够被执行，但当前还未被执行，那么该任务处于就绪态（比如：任务被抢占，被抢占的任务就是就绪态）
>
>3. 阻塞态：如果一个任务因延时或等待外部事件发生，那么这个任务就处于阻塞态 
>
>4. 挂起态：类似暂停，调用函数 vTaskSuspend() 进入挂起态，需要调用解挂函数vTaskResume() 才可以进入就绪态
>
>
>
>注意：仅就绪态可转变成运行态、其他状态的任务想运行，必须先转变成就绪态、这四种状态中，除了运行态**，其他三种任务状态的任务都有其对应的任务状态列表**（即任务列表，就绪列表pxReadyTasksLists[x]，其中x代表任务优先级数值，阻塞列表，挂起列表）
>
>**调度器总是在所 有处于就绪列表的任务中，选择具有最高优先级的任务来执行**

---

## 四、内存管理

>简介：ARM芯片属于**精简指令计算机**，它所用的**指令比较简单**，有如下特点：1.对内存只读、写指令 2.对于数据的运算是在CPU内部实现 3.使用**RISC指令的CPU复杂度小一点，易于设计**

>无论是cortex-M3/M4、还有cortex-A7，CPU的内部都有R0、R1、……….、R15寄存器。(对于R13、R14、R15**特殊寄存器**。)
>
> R13：别名SP，栈指针
>
> R14：别名LR，用来保证返回地址
>
> R15：别名PC，程序计数器，表示当前指令地址，写入新值即可跳转
>
> 
>
>如：**加法计算**，就是CPU通过**flash读取指令和运行指令**，读RAM内存的数据，在CPU里进行运算，写指令（数据）到RAM内存里。

---

>堆：就是一块空闲的内存，**需要提供管理函数**，malloc：从堆里划出一块空间给程序使用，free：用完后，再把它标记为"空闲"的，可以再次使用。如：定义一个数组就是空闲的内存，**提供管理的函数，就是堆**
>
>堆管理函数 :(用数组开辟空闲空间，**再提供管理的函数**)
>
>1. 若不加头部，直接开辟空间，释放空间（就不知道有多大，并且空闲的**空间不好管理**）
>2. 加头部，即再开辟空间+头部内容（开辟多大空间的信息），对空闲的空间进行管理，通过链表对不连续的空间进行关联,，空闲链表的结构体（空闲空间的大小+下一个空闲空间的地址）

---

>栈：**函数调用时局部变量**保存在栈中，当前程序的环境也是保存在栈中（可以从堆中分配一块空间用作栈）

---

>BL指令：返回指令（**嵌套函数，调用函数，保存下一跳返回指令**）LR指令：**跳转下一跳指令**（调用函数的第一条指令）
>
>> 1. 在函数调用关系中，LR会被覆盖，怎么解决?
>>
>> >程序本质就**函数嵌套**，在main里进行嵌套，而栈就是函数调用时局部变量保存在栈中，当前程序的环境也是保存在栈中，在使用完函数，就会释放内存。所以LR的信息会保存在栈里，通过**指令进行读取**
>>
>> 2. 为什么每个RTOS任务都有自己的栈？
>>
>> > **不同的任务有自己的调用关系，自己的局部变量，要保证现场，所以要拥有属于自己的栈**
>>
>> 3. 局部变量在栈中分配

---

>FreeRTOS中内存管理函数：：pvPortMalloc 、vPortFree，对应于 C 库malloc、 free
>
>>管理方法1:（Heap_1）它只实现了pvPortMalloc，没有实现vPortFree，它的实现原理很简单，首先定义一个大数组
>
>>管理方法2: （Heap_2）它支持 vPortFree，找出最小的、能满足 pvPortMalloc 的内存，虽然不再推荐使用heap_2，但是它的效率还是远高于malloc、free。
>
>>管理方法3: （Heap_3）Heap_3 使用标准 C 库里的 malloc、free 函数，所以堆大小由链接器的配置决定，配置 项 configTOTAL_HEAP_SIZE 不再起作用，C库里的malloc、free函数并非线程安全的
>
>>管理方法4: 跟 Heap_1、Heap_2 一样，Heap_4 也是使用大数组来分配内存，Heap_4使用首次适应算法(first fit)来分配内存。它还会把相邻的空闲内存合并为一个更 大的空闲内存，这有**助于较少内存的碎片问题**

---

## 五、Systick和HAL时基

> 使用了FreeRTOS会强制使用systick作为自己的心跳，这个os_tick的优先级是最低的，它主要的作用就是OS任务调度，时间片查询等工作。

>timer（譬如timer1）产生HAL心跳，systick产生freertos的心跳，HAL是不依赖于OS API的一套硬件操作接口。
>
>os_tick是如何工作的：
>
>当配置了两套时基时，systick作为os的心跳，它的优先级应该配置为最低，这是因为：
>
>systick的中断服务，主要处理一些os层面的东西，譬如查看是否有blocked的任务已经就绪，然后执行任务切换等工作，作为一个实时操作系统为了保持实时性不打搅其他的中断服务，stm32cubemx里会强制systick的优先级为最低。
>
>hal_tick是如何工作的：
>
>为了保证HAL时基的可靠，防止出现上面讲到的一直忙等待问题，强制设定它的优先级为最高，我们使用基本定时器TIM6或者TIM7作为HAL时基，一般是1ms，HAL_Delay()函数就是使用的这个时基。
>

## 六、任务创建

>**创建任务包括**什么：1.做什么事：函数 2.栈和TCB结构体（链表） 3.优先级
>
>>**怎么分配空间给栈和TCB结构体：**有动态分配（栈空间由函数直接给你分配好了）和静态分配（需要自己去初始数组，然后将地址传给函数）
>
>**如何估算栈大小：**栈内容包括：1、返回地址LR，其他Reg ，函数调用深度 2、局部变量宽度  3、保存现场

>**使用同一函数创建不同任务**：虽然它们使用的是同一函数，但是**它们运行的栈不一样**，那些**局部变量是不同的版本**，不同的实体，所以运行的**结果可能不同**

### 1.动态创建任务

> 动态创建任务(xTaskCreate() )：任务的任务控制块以及任务的栈空间所需的内存，均由 FreeRTOS 从 FreeRTOS 管理的堆中分配

**动态创建流程：**

>a.使用动态创建任务，需将宏 configSUPPORT_DYNAMIC_ALLOCATION配置为 1
>
>b.定义函数入口参数，任务控制块句柄
>
>C.编写任务函数

xTaskCreate()创建的任务会立即进入就绪态，由任务调度器调度运行：内容
>a. 申请堆栈内存，任务控制块内存
>
>b. TCB结构体成员赋值
>
>c. 添加新任务到就绪列表中

**任务函数具体实现：**

>动态创建任务其内部实现：
>
>1、申请堆栈内存（返回首地址）
>
>2、 申请任务控制块内存（返回首地址）
>
>3、 把前面申请的堆栈地址，赋值给控制块的堆栈成员
>
>4、 调用prvInitialiseNewTask 初始化任务控制块中的成员
>
>5、调用prvAddNewTaskToReadyList 添加新创建任务到就绪列表中
>
>

```text
//有条件编译的 #if( configSUPPORT_DYNAMIC_ALLOCATION == 1 )，需定义对应的宏
//动态创建任务----需提供的参数---以及返回值

问1：函数指针是如何定义的
问2：const含义
问3：为什么xTaskCreate()定义了指针，也就说程序给它分配空间，为什么还要用malloc函数给他再次分配空间 ---全局变量、局部变量、malloc开辟空间，有什么不同
问4：断言是什么

注1.
//TaskFunction_t   pxTaskCode --- 任务函数指针
//typedef void (*TaskFunction_t)( nvoid * ); --- 无返回类型无参数的任务函数指针

注2.
//#define configSTACK_DEPTH_TYPE uint16_t

注3.
//TaskHandle_t * const  pxCreatedTask --- 任务控制块---创建任务函数的部分任务就是对TCB结构体执行赋值
//
   
typedef struct tskTaskControlBlock       
{
    volatile StackType_t 		* pxTopOfStack; 					/* 任务栈栈顶，必须为TCB的第一个成员 */
   	ListItem_t 					xStateListItem;           			/* 任务状态列表项 */      
	ListItem_t 					xEventListItem;						/* 任务事件列表项 */     
    UBaseType_t 				uxPriority;                			/* 任务优先级，数值越大，优先级越大 */
    StackType_t 				* pxStack;							/* 任务栈起始地址 */
    char 			pcTaskName[ configMAX_TASK_NAME_LEN ]; 			/* 任务名字 */		
	…
	省略很多条件编译的成员
} tskTCB;
    
    
typedef struct xTASK_STATUS
{
	TaskHandle_t xHandle;			
	const char *pcTaskName;			
	UBaseType_t xTaskNumber;		
	eTaskState eCurrentState;		
	UBaseType_t uxCurrentPriority;	
	UBaseType_t uxBasePriority;		
	uint32_t ulRunTimeCounter;		
	StackType_t *pxStackBase;		
	uint16_t usStackHighWaterMark;	
} TaskStatus_t;    
    

BaseType_t xTaskCreate
(	
TaskFunction_t   pxTaskCode,       			 /* 指向任务函数的指针*/
const char * const  pcName,	     			 /*任务名字，最大长度有定义*/				
const configSTACK_DEPTH_TYPE  usStackDepth,	 /*任务堆栈大小，注意字为单位*/
void * const  pvParameters,		 			 /*传递给任务函数的参数，通常为结构体*/
UBaseType_t  uxPriority,		  		     /*任务优先级*/					
TaskHandle_t * const  pxCreatedTask          /*任务句柄，任务控制块*/
)
	
{
	TCB_t *pxNewTCB;                         /*任务控制块*/
	BaseType_t xReturn;						 /*创建任务是否成功，返回值*/
	//首先判断一下，栈生长的方向----stm32的栈是往下生长，堆是往上生长  #define portSTACK_GROWTH			( -1 )
	#if( portSTACK_GROWTH > 0 )
	{  
       .....
    }
    #else /* portSTACK_GROWTH */
    {
        StackType_t *pxStack;  --- 任务堆栈首地址，每个任务都有自己的任务堆栈的
        //申请任务堆栈---usStackDepth--就是传参时，定义的参数，申请的大小为128*4字节（StackType_t == uint_32）
        pxStack = ( StackType_t * ) pvPortMalloc( ( ( ( size_t ) usStackDepth ) * sizeof( StackType_t ) ) ); 
        if( pxStack != NULL )  //判断是否申请成功
        {
            pxNewTCB = ( TCB_t * ) pvPortMalloc( sizeof( TCB_t ) ); /*lint !e961 MISRA exception as the
            if( pxNewTCB != NULL )
            {
              //任务堆栈地址赋值给任务控制块
              pxNewTCB->pxStack = pxStack;
            }
            else
            {
			  //没申请成功，即释放空间
                vPortFree( pxStack );
            }
            else
            {
                pxNewTCB = NULL;
            }
    }
    #endif /* portSTACK_GROWTH */
    if( pxNewTCB != NULL )
	{   
    	
    	//调用prvInitialiseNewTask 初始化任务控制块中的成员
   	 	prvInitialiseNewTask( pxTaskCode, pcName, ( uint32_t ) usStackDepth, pvParameters, uxPriority, 				pxCreatedTask, pxNewTCB, NULL );
   	 	
   	 	//调用prvAddNewTaskToReadyList 添加新创建任务到就绪列表中
		prvAddNewTaskToReadyList( pxNewTCB );
		xReturn = pdPASS;
    }
    else
	{
		xReturn = errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY;
	}
	return xReturn;
	}		
			
```

---

 **初始化任务控制块(prvInitialiseNewTask)---初始化任务状态信息**

>4、 调用prvInitialiseNewTask 初始化任务控制块中的成员
>
>4.1：初始化堆栈为0xa5（可选） 4.2：记录栈顶，保存在pxTopOfStack
>
>4.3：保存任务名字到pxNewTCB->pcTaskName[ x ]中
>
>4.4：保存任务优先级到pxNewTCB->uxPriority
>
>4.5：设置状态列表项的所属控制块，设置事件列表项的值
>
>4.6列表项的插入是从小到大插入，所以这里将越高优先级的任务他的事件列表项值设 
>
>置越小，这样就可以拍到前面
>
>4.7：调用pxPortInitialiseStack初始化任务堆栈，用于保存当前任务上下文寄存器信息， 
>
>已备后续任务切换使用
>
>4.8：将任务句柄等于任务控制块

```test
static void prvInitialiseNewTask
( 	
    TaskFunction_t pxTaskCode,	        	/*任务函数指针*/								
    const char * const pcName,				/*任务名字*/
    const uint32_t ulStackDepth,			/*任务堆栈大小，注意字为单位*/
    void * const pvParameters,				/*任务函数的参数*/
    UBaseType_t uxPriority,					/*任务优先级*/
    TaskHandle_t * const pxCreatedTask,		/*任务句柄 == 任务控制块*/
    TCB_t *pxNewTCB,						/*任务控制块*/
    const MemoryRegion_t * const xRegions
)

{
	StackType_t *pxTopOfStack;     			 /*任务栈顶地址*/
	UBaseType_t x;

	/*初始堆栈内容---可以根据其值，判断使用堆栈情况*/
	#if( tskSET_NEW_STACKS_TO_KNOWN_VALUE == 1 )
	{
		/*
		 *#define tskSTACK_FILL_BYTE  ( 0xa5U ) ----0XA5输出的内容就是一个方波
		 *将堆栈空间值，设置成0XA5
		 */
		( void ) memset( pxNewTCB->pxStack, ( int ) tskSTACK_FILL_BYTE, ( size_t ) ulStackDepth * sizeof( 			StackType_t ) );
	}
	/*判断栈空间是否往下生长*/
	#if( portSTACK_GROWTH < 0 )
	{
		/*栈空间是往下生长，堆是往上生长*/
		/*pxTopOfStack是栈顶，pxStack是重堆申请的空间，地址是空间最下端，所以进行指针运算 */
		pxTopOfStack = pxNewTCB->pxStack + ( ulStackDepth - ( uint32_t ) 1 );
		
		/*8字节对齐*/
		pxTopOfStack = ( StackType_t * ) ( ( ( portPOINTER_SIZE_TYPE ) pxTopOfStack ) & ( ~( ( 						portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) ) ); 
		
		/*断言*/
		configASSERT( ( ( ( portPOINTER_SIZE_TYPE ) pxTopOfStack & ( portPOINTER_SIZE_TYPE ) 						portBYTE_ALIGNMENT_MASK ) == 0UL ) );
	 }	
	 
	 /*将传参“任务名字”，赋值给任务控制块*/ 
	 for( x = ( UBaseType_t ) 0; x < ( UBaseType_t ) configMAX_TASK_NAME_LEN; x++ )
	{
		pxNewTCB->pcTaskName[ x ] = pcName[ x ];
		if( pcName[ x ] == 0x00 )
		{
			break;
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}
	 }
	pxNewTCB->pcTaskName[ configMAX_TASK_NAME_LEN - 1 ] = '\0';
	
	/*对超过优先级32的任务，设置成最高优先级*/
	if( uxPriority >= ( UBaseType_t ) configMAX_PRIORITIES )
	{
		uxPriority = ( UBaseType_t ) configMAX_PRIORITIES - ( UBaseType_t ) 1U;
	}
	else
	{
		mtCOVERAGE_TEST_MARKER();
	}

	/*对任务控制块优先级进行赋值*/
	pxNewTCB->uxPriority = uxPriority;
	/*进行优先级继承*/	
	#if ( configUSE_MUTEXES == 1 )
	{
		pxNewTCB->uxBasePriority = uxPriority;
		pxNewTCB->uxMutexesHeld = 0;
	}
	
	/*初始化状态列表项，事件项----将任务的列表项-插入就绪、阻塞等列表中*/
	vListInitialiseItem( &( pxNewTCB->xStateListItem ) );
	/*?*/
	vListInitialiseItem( &( pxNewTCB->xEventListItem ) );
	
	/*事件列表总是按照优先级顺序排列*/
	listSET_LIST_ITEM_VALUE( &( pxNewTCB->xEventListItem ), ( TickType_t ) configMAX_PRIORITIES - ( 			TickType_t ) uxPriority );
	/*?*/
	listSET_LIST_ITEM_OWNER( &( pxNewTCB->xEventListItem ), pxNewTCB 
	
	-----------------------------------------------
	-----------------------------------------------
	/*调用pxPortInitialiseStack初始化任务堆栈，用于保存当前任务上下文寄存器信息， 已备后续任务切换使用*/
	/*出栈时，将内存的内容赋值给内核寄存器*/
	
	pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxTaskCode, pvParameters );
	
	/*将任务句柄等于任务控制块*/
	if( ( void * ) pxCreatedTask != NULL )
	{
		*pxCreatedTask = ( TaskHandle_t ) pxNewTCB;
	}
}
```

---

**初始化任务堆栈，用于保存当前任务上下文寄存器信息**

![a2c310fd971162c7c801ddcf1871c59](C:\Users\15924\Documents\WeChat Files\wxid_mg132t21a1pq22\FileStorage\Temp\a2c310fd971162c7c801ddcf1871c59.jpg)

>任务入栈，将对应现场的内容保存在内存中，这叫"保存现场"
>
>任务出栈，将这些保存的内容重新赋值给内核寄存器中

```test
StackType_t *pxPortInitialiseStack( StackType_t *pxTopOfStack, TaskFunction_t pxCode, void *pvParameters )
{
	
	pxTopOfStack--; 
	*pxTopOfStack = portINITIAL_XPSR;	/* xPSR */
	pxTopOfStack--;
	*pxTopOfStack = ( ( StackType_t ) pxCode ) & portSTART_ADDRESS_MASK;	/* PC */
	pxTopOfStack--;
	*pxTopOfStack = ( StackType_t ) prvTaskExitError;	/* LR */

	pxTopOfStack -= 5;	/* R12, R3, R2 and R1. */
	*pxTopOfStack = ( StackType_t ) pvParameters;	/* R0 */
	pxTopOfStack -= 8;	/* R11, R10, R9, R8, R7, R6, R5 and R4. */

	return pxTopOfStack;
}
```

---

**调用prvAddNewTaskToReadyList 添加新创建任务到就绪列表中**

>5.1：记录任务数量uxCurrentNumberOfTasks++
>
>5.2:判断新创建的任务是否为第一个任务-->如果创建的是第一个任务，初始化任务列表prvInitialiseTaskLists---如果创建的不是第一个任务，并且调度器还未开始启动，比较新任务与正在执行的任务优先级大小，新任务优先级大的话，将当前控制块重新指向新的控制块
>
>5.3：将新的任务控制块添加到就绪列表中，使用函数prvAddTaskToReadyList
>
>5.4：将uxTopReadyPriority相应bit置一，表示相应优先级有就绪任务，比如任务优先级为5，就将该变量的位5置一，方便后续任务切换判断，对应的就绪列表是否有任务存在。将新创建的任务插入对应的就绪列表末尾。
>
>5.5：如果调度器已经开始运行，并且新任务的优先级更大的话，进行一次任务切换

```test
/*添加新创建任务到就绪列表中*/
static void prvAddNewTaskToReadyList( TCB_t *pxNewTCB )
{
	taskENTER_CRITICAL();
	{
		/*就绪列表任务数量+1*/
		uxCurrentNumberOfTasks++;
		/*判断当前任务是否是创建的第一个任务*/
		if( pxCurrentTCB == NULL )
         {
            /*将当前任务控制块，设置成该任务-----当前任务控制块就是该任务*/
            pxCurrentTCB = pxNewTCB;
            if( uxCurrentNumberOfTasks == ( UBaseType_t ) 1 )
            {
            	/*初始化列表--列表里若定义了，相关的宏，初始化就绪、延迟、阻塞等列表*/
                prvInitialiseTaskLists();
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
	     }
	    //不是第一个任务
        else
		{
			//判读断是否启动了调度器---未启动
			if( xSchedulerRunning == pdFALSE )
			{
				//该任务与之前的任务优先级比较，pxCurrentTCB始终指向优先级最高的任务
				if( pxCurrentTCB->uxPriority <= pxNewTCB->uxPriority )
				{
					pxCurrentTCB = pxNewTCB;
				}
				else
				{
					mtCOVERAGE_TEST_MARKER();
				}
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}
		//任务标号，若configUSE_TRACE_FACILITY被置1，该任务pxNewTCB控制块，保留该任务创建序号信息
		uxTaskNumber++;
		#if ( configUSE_TRACE_FACILITY == 1 )
		{
			pxNewTCB->uxTCBNumber = uxTaskNumber;
		}
        //添加进入就绪列表
		prvAddTaskToReadyList( pxNewTCB );	
		
		//已经启动了任务调度器
		if( xSchedulerRunning != pdFALSE )
		{
			//判断正在运行的任务优先级，是否低于创建的任务，若是则执行一次任务切换，运行高优先级任务
            if( pxCurrentTCB->uxPriority < pxNewTCB->uxPriority )
            {
                taskYIELD_IF_USING_PREEMPTION();
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
		}
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
}		

//添加进入就绪列表的内容
#define prvAddTaskToReadyList( pxTCB )																\
	traceMOVED_TASK_TO_READY_STATE( pxTCB );														\
	taskRECORD_READY_PRIORITY( ( pxTCB )->uxPriority );												\
	vListInsertEnd( &( pxReadyTasksLists[ ( pxTCB )->uxPriority ] ), &( ( pxTCB )->xStateListItem ) ); \
	tracePOST_MOVED_TASK_TO_READY_STATE( pxTCB )
```





### 2.清除任务

**清除任务流程**

>a.使用删除任务函数，需将宏INCLUDE_vTaskDelete 配置为 1
>
>b.当传入的参数为NULL，则代表删除任务自身（当前正在运行的任务）
>
>C.空闲任务会负责释放被删除任务中由系统分配的内存，但是由用户在任务删除前申请的内存， 则需要由用户在任务被删除前提前释放,否则将导致内存泄露

**删除任务内部实现：**

>1、获取所要删除任务的控制块，通过传入的任务句柄，判断所需要删除哪个任务，NULL代表删除自身
>
>2、将被删除任务，移除所在列表，将该任务在所在列表中移除，包括：就绪、阻塞、挂起、事件等列表
>
>3、判断所需要删除的任务--->删除任务自身，需先添加到等待删除列表，内存释放将在空闲任务执行--->删除其他任务，当前任务数量--更新下一个任务的阻塞超时时间，以防被删除的任务就是下一个阻塞超时的任务
>
>4、删除的任务为其他任务则直接释放内存prvDeleteTCB( )
>
>5、调度器正在运行且删除任务自身，则需要进行一次任务切换

**清除任务具体实现**

>1、获取所要删除任务的控制块----通过传入的任务句柄，判断所需要删除哪个任务，NULL代表删除自身
>
>2、将被删除任务，移除所在列表-----将该任务在所在列表中移除，包括：就绪、阻塞、挂起、事件等列表
>
>3、判断所需要删除的任务----删除任务自身，需先添加到等待删除列表，内存释放将在空闲任务执行
>
>----删除其他任务，当前任务数量-更新下一个任务的阻塞超时时间，以防被删除的任务就是下一个阻塞超时的任务
>
>4、删除的任务为其他任务则直接释放内存prvDeleteTCB( )
>
>5、调度器正在运行且删除任务自身，则需要进行一次任务切换

```test
void vTaskDelete( TaskHandle_t xTaskToDelete )
{
	TCB_t *pxTCB;
    taskENTER_CRITICAL();
	{
        //获取任务控制块 ---- 任务句柄就是任务控制块 ---- 如果NULL,即返回当前任务句柄
        //#define prvGetTCBFromHandle( pxHandle )
       	//( ( ( pxHandle ) == NULL ) ? ( TCB_t * ) pxCurrentTCB : ( TCB_t * ) ( pxHandle ) )
        pxTCB = prvGetTCBFromHandle( xTaskToDelete );
        
        //将被删除任务，移除所在列表-----将该任务在所在列表中移除，包括：就绪、阻塞、挂起、事件等列表
        if( uxListRemove( &( pxTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
        {
            taskRESET_READY_PRIORITY( pxTCB->uxPriority );
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
        //是否还有相应任务优先的任务
        if( uxListRemove( &( pxTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
        {
        	//若没有了，则将任务列表的标志位置0
            taskRESET_READY_PRIORITY( pxTCB->uxPriority );
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
        //删除事件列表项
        if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) != NULL )
		{
			( void ) uxListRemove( &( pxTCB->xEventListItem ) );
		}
		 else
        {
            mtCOVERAGE_TEST_MARKER();
        }
        //判断所需要删除的任务----删除任务自身，需先添加到等待删除列表，内存释放将在空闲任务执行
        //不能直接释放任务内存，是因为它是正在运行的任务
        if( pxTCB == pxCurrentTCB )
		{
			vListInsertEnd( &xTasksWaitingTermination, &( pxTCB->xStateListItem ) );
			++uxDeletedTasksWaitingCleanUp;
		}
		//删除的任务是其他任务，自己要释放内存空间prvDeleteTCB()函数--且要更新下一个任务的阻塞超时时间
		//因为删除的任务是被阻塞了，但删除了任务，就回复不了，所以要更新一下任务阻塞时间
		else
		{
			prvDeleteTCB( pxTCB );	
			prvResetNextTaskUnblockTime();
		}
		//调度器正在运行且删除任务自身，则需要进行一次任务切换
		if( xSchedulerRunning != pdFALSE )
		{
			if( pxTCB == pxCurrentTCB )
			{
				configASSERT( uxSchedulerSuspended == 0 );
				portYIELD_WITHIN_API();
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}
	}
```



---

## FreeRTOS遇到的问题

1. 不用互斥信号量，**使用变量作为标志位，实现资源管理，存在的问题**？

>OLED的I2C在数据传输的过程中是不能够被打断的，所以采样一个变量进行限制，在执行的过程中其他任务不能够使用OLED
>
>---即任务在使用I2C的过程，必须完整。任务通信被打断，必须要从头执行，所以必须完整
>
>不使用互斥信号的情况，变量作为标志位，会到该任务一直执行。
>
>---变量互斥与互斥信号量的区别就是互斥信号量不死等，将任务挂起。而变量互斥死等一个时间片，而且变量置标志位的时间太短，导致任务符合，该任务又会被执行，其他任务抢不到通信资源，导致该任务一直被执行

2. mdelay函数与Task的Delay函数的区别

>mdelay()与HAL_Delay()函数----------通过定时器中断进行计数，while（延迟计数）{ } 进行延迟跳出循环。-------在任务调度时，中断优先于任务，所以CPU不会被释放。
>
>
>
>vTaskDelay()函数----是任务函数主动释放CPU资源，CPU调用其他任务（优先级高的任务，如果不用该函数，那么CPU一直调用优先级高的任务

3. 默认任务的优先级是多少，它和创建任务的区别是什么

>
>
>

4. 优先级高的任务必须退出？

>使用vTaskDelete()---1.自杀（在自己函数中调用---内存释放，由空闲任务进行处理-vTaskDelete(NULL)），2.他杀（其他函数，调用任务删除，由该函数释放内存）-----若最高优先级任务不释放CPU的资源，那么空闲任务不会被调用，任务空间不会被释放，会导致内存泄漏-----采用vTaskDelete()--就会释放CPU资源

5. 空闲任务(Idle 任务)的作用之一：释放被删除的任务的内存。**为什么必须要有空闲任务？**

>一个良好的程序，它的任务都是事件驱 动的：平时大部分时间处于阻塞状态。有可能我们自己创建的所有任务都无法执行，但是调 度器必须能找到一个可以运行的任务：所以，我们要提供空闲任务。在使用 vTaskStartScheduler()函数来创建、启动调度器时，这个函数内部会创建空闲任务：1.空闲任务优先级为 0：它不能阻碍用户任务运行 2.空闲任务要么处于就绪态，要么处于运行态，永远不会阻塞