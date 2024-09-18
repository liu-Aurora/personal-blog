# FreeRTOS消息队列

>简介：队列是任务到任务、任务到中断、中断到任务数据交流的一种机制（消息传递）
>
>类似全局变量？
>
>全局变量的弊端：数据无保护，导致数据不安全，当多个任务同时对该变量操作时，数据易受损

>**全局变量与消息队列区别**?
>
>全局变量只是没有什么数据保护而已，而消息队列的读写都进行临界区保护，阻止任务切换，那么其他任务就不能进行读写。
>
>所以，项目中的所有任务都是独立了，不需要进行信息传递，也就不需要定义全局变量。
>
>若只是一对任务进行数据传递，全局变量也没有数据易受损的问题。

> **队列的出队、入队阻塞的特点**

>FreeRTOS基于队列， 实现了多种功能，其中包括队列集、互斥信号量、计数型信号量、二值信号量、 递归互斥信号量，因此很有必要深入了解 FreeRTOS 的队列 。

## 一、FreeRTOS队列特点

1. **数据入队出队方式**

>队列通常采用“先进先出”(FIFO)的数据存储缓冲机制，即先入队的数据会先从队列中被读取，FreeRTOS中也可以配置为“后进先出”LIFO方式；

2. **数据传递方式**

>FreeRTOS中队列采用实际值传递，即将数据拷贝到队列中进行传递， FreeRTOS采用拷贝数据传递，也可以传递指针，所以在传递较大的数据的时候采用指针传递

3. **多任务访问**

>队列不属于某个任务，任何任务和中断都可以向队列发送/读取消息

4. **出队、入队阻塞**

>当任务向一个队列发送消息时，可以指定一个阻塞时间，假设此时当队列已满无法入队
>
>①若阻塞时间为0 ：直接返回不会等待；
>
>②若阻塞时间为0~port_MAX_DELAY ：等待设定的阻塞时间，若在该时间内还无法入队，超时后直接返回不再等待；
>
>③若阻塞时间为port_MAX_DELAY ：死等，一直等到可以入队为止。出队阻塞与入队阻塞类似；
>
>**入队阻塞：**
>
> 队列满了，此时写不进去数据；
>
> ①将该任务的状态列表项挂载在pxDelayedTaskList；
>
> ②将该任务的事件列表项挂载在xTasksWaitingToSend；
>
>**出队阻塞：**
>
> 队列为空，此时读取不了数据；
>
> ①将该任务的状态列表项挂载在pxDelayedTaskList；
>
> ②将该任务的事件列表项挂载在xTasksWaitingToReceive；

>问题：当多个任务写入消息给一个“满队列”时，这些任务都会进入阻塞状态，也就是说有多个任务在等待同一 个队列的空间。那当队列中有空间时，哪个任务会进入就绪态？
>
>1. 优先级最高的任务
>
>2. 如果大家的优先级相同，那等待时间最久的任务会进入就绪态

---

## 二、**队列操作基本过程**

![image-20240808174733116](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240808174733116.png)

---

## 三、队列结构体

**队列结构体**

```c
typedef struct QueueDefinition 
{
    int8_t * pcHead								 /* 存储区域的起始地址 */
    int8_t * pcWriteTo;        					 /* 下一个写入的位置 */
    union
    {
        	QueuePointers_t     xQueue; 
			SemaphoreData_t  xSemaphore; 
    } u ;
    List_t xTasksWaitingToSend; 				 /* 等待发送列表 */
    List_t xTasksWaitingToReceive;				 /* 等待接收列表 */
    volatile UBaseType_t uxMessagesWaiting; 	 /* 非空闲队列项目的数量 */
    UBaseType_t uxLength；					    /* 队列长度 */
    UBaseType_t uxItemSize;                 	 /* 队列项目的大小 */
    volatile int8_t cRxLock; 					 /* 读取上锁计数器 */
    volatile int8_t cTxLock；				    /* 写入上锁计数器 */
   /* 其他的一些条件编译 */
} xQUEUE;

```

**当用于队列使用时**

```c
typedef struct QueuePointers
{
     int8_t * pcTail; 							/* 存储区的结束地址 */
     int8_t * pcReadFrom;						/* 最后一个读取队列的地址 */
} QueuePointers_t;

```

**当用于互斥信号量和递归互斥信号量时**

```c
typedef struct SemaphoreData
{
    TaskHandle_t xMutexHolder;					/* 互斥信号量持有者 */
    UBaseType_t uxRecursiveCallCount;			/* 递归互斥信号量的获取计数器 */
} SemaphoreData_t;

```

**队列结构体整体示意图：**

![image-20240808220309832](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240808220309832.png)	

## 四、队列相关API函数介

>使用队列的主要流程：创建队列>写队列> 读队列。

### 1、创建队列

**动态方式创建队列：**

`xQueueCreate()`

```c
#define xQueueCreate (  uxQueueLength,   uxItemSize  )   						 					
	   xQueueGenericCreate( ( uxQueueLength ), ( uxItemSize ), (queueQUEUE_TYPE_BASE )) 
```

> 动态和静态创建队列之间的区别：队列所需的内存空间由 FreeRTOS 从 FreeRTOS 管理的堆中分配，而静态创建需要用户自行分配内存。

```text
/*前面说 FreeRTOS 基于队列实现了多种功能，每一种功能对应一种队列类型，队列类型的 queue.h 文件中有定义：*/
#define queueQUEUE_TYPE_BASE                  			( ( uint8_t ) 0U )	/* 队列 */
#define queueQUEUE_TYPE_SET                  			( ( uint8_t ) 0U )	/* 队列集 */
#define queueQUEUE_TYPE_MUTEX                 			( ( uint8_t ) 1U )	/* 互斥信号量 */
#define queueQUEUE_TYPE_COUNTING_SEMAPHORE    			( ( uint8_t ) 2U )	/* 计数型信号量 */
#define queueQUEUE_TYPE_BINARY_SEMAPHORE     			( ( uint8_t ) 3U )	/* 二值信号量 */
#define queueQUEUE_TYPE_RECURSIVE_MUTEX       			( ( uint8_t ) 4U )	/* 递归互斥信号量 */
```

```text
//创建队列、信号量、队列集使用的同一个函数，运用宏定义----解决函数的重载问题
//因此创建队列只需2个参数：队列长度、队列项的大小

#if( configSUPPORT_DYNAMIC_ALLOCATION == 1 )

	QueueHandle_t xQueueGenericCreate( const UBaseType_t uxQueueLength, 
									   const UBaseType_t uxItemSize, const 																		   uint8_t ucQueueType )
	{
		Queue_t *pxNewQueue;
		size_t xQueueSizeInBytes;
		uint8_t *pucQueueStorage;
		
		//判断队列长度是否大于0，保证数据的合法性
		configASSERT( uxQueueLength > ( UBaseType_t ) 0 );
		//更加队列项的大小，来分配内存空间
		if( uxItemSize == ( UBaseType_t ) 0 )
		{
			xQueueSizeInBytes = ( size_t ) 0;
		}
		else
		{
			xQueueSizeInBytes = ( size_t ) ( uxQueueLength * uxItemSize ); 
		}
		//从堆中分配内存空间---整个队列的结构分为：队列结构体存储区、队列项的存储区
		pxNewQueue = ( Queue_t * ) pvPortMalloc( sizeof( Queue_t ) + xQueueSizeInBytes );
		//判读是否申请成功
		if( pxNewQueue != NULL )
		{
			//将申请的内存的首地址，再进行偏移。
			//---堆申请的地址是往上生长的，首地址pchead，进行偏移一个结构体的大小，就是队列项的首地址
			pucQueueStorage = ( ( uint8_t * ) pxNewQueue ) + sizeof( Queue_t );

			#if( configSUPPORT_STATIC_ALLOCATION == 1 )
			{
				
				pxNewQueue->ucStaticallyAllocated = pdFALSE;
			}
			#endif 
			//初始化新队列的函数
			prvInitialiseNewQueue( uxQueueLength, uxItemSize, pucQueueStorage, ucQueueType, pxNewQueue );
		}
		else
		{
			traceQUEUE_CREATE_FAILED( ucQueueType );
		}

		return pxNewQueue;
	}

#endif 
		
//初始化新队列的函数 ---- 参数：队列长度、队列项大小、队列项的首地址、队列的类型、队列的首地址
static void prvInitialiseNewQueue
								( const UBaseType_t uxQueueLength, const UBaseType_t uxItemSize, 
								  uint8_t *pucQueueStorage, const uint8_t ucQueueType,
                                  Queue_t *pxNewQueue )
{
	( void ) ucQueueType;
	
	//队列项大小为0，则是二值信号量
	if( uxItemSize == ( UBaseType_t ) 0 )
	{
		//将队列结构体的pchead—>初始的队列结构体的初始地址
		pxNewQueue->pcHead = ( int8_t * ) pxNewQueue;
	}
	else
	{	
		//将队列结构体的pchead->队列项的首地址
		pxNewQueue->pcHead = ( int8_t * ) pucQueueStorage;
	}
	
	//初始结构体的内容
	pxNewQueue->uxLength = uxQueueLength;
	pxNewQueue->uxItemSize = uxItemSize;
	
	( void ) xQueueGenericReset( pxNewQueue, pdTRUE );

	#if ( configUSE_TRACE_FACILITY == 1 )
	{
		pxNewQueue->ucQueueType = ucQueueType;
	}
	#endif 

	#if( configUSE_QUEUE_SETS == 1 )
	{
		pxNewQueue->pxQueueSetContainer = NULL;
	}
	#endif 

	traceQUEUE_CREATE( pxNewQueue );
}

//复位队列
BaseType_t xQueueGenericReset( QueueHandle_t xQueue, BaseType_t xNewQueue )
{
	Queue_t * const pxQueue = ( Queue_t * ) xQueue;

	configASSERT( pxQueue );

	taskENTER_CRITICAL();
	{	
		//初始化结构体的内容：队列结构体整体示意图（其内指针的指向）
		pxQueue->pcTail = pxQueue->pcHead + ( pxQueue->uxLength * pxQueue->uxItemSize );
		pxQueue->uxMessagesWaiting = ( UBaseType_t ) 0U;
		pxQueue->pcWriteTo = pxQueue->pcHead;
		pxQueue->u.pcReadFrom = pxQueue->pcHead + ( ( pxQueue->uxLength - ( UBaseType_t ) 1U ) * pxQueue-			>uxItemSize );
		pxQueue->cRxLock = queueUNLOCKED;
		pxQueue->cTxLock = queueUNLOCKED;
		
		//不是新创建队列，那就复位它，将列表xTasksWaitingToSend移除
		if( xNewQueue == pdFALSE )
		{
			
			if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToSend ) ) == pdFALSE )
			{
				if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToSend ) ) != pdFALSE )
				{
					queueYIELD_IF_USING_PREEMPTION();
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
		//是新创建的队列，那就初始化这两个列表xTasksWaitingToSend和xTasksWaitingToReceive
		else
		{
			vListInitialise( &( pxQueue->xTasksWaitingToSend ) );
			vListInitialise( &( pxQueue->xTasksWaitingToReceive ) );
		}
	}
	taskEXIT_CRITICAL();
	return pdPASS;
}
```

### 2、写队列API函数

**往队列的尾部写入消息**

`xQueueSend() `

```c
#define  xQueueSend(  xQueue,   pvItemToQueue,   xTicksToWait  )	 					    
	    xQueueGenericSend( ( xQueue ), ( pvItemToQueue ), ( xTicksToWait ), queueSEND_TO_BACK )

```

**往队列的尾部写入消息**

`xQueueSendToBack() `

```c
define  xQueueSendToBack(  xQueue,   pvItemToQueue,   xTicksToWait  )					     
	    xQueueGenericSend( ( xQueue ), ( pvItemToQueue ), ( xTicksToWait ), queueSEND_TO_BACK )
    
```

**往队列的头部写入消息**

`xQueueSendToFront() `

```c
#define  xQueueSendToFront(  xQueue,   pvItemToQueue,   xTicksToWait  ) 					   
	    xQueueGenericSend( ( xQueue ), ( pvItemToQueue ), ( xTicksToWait ), queueSEND_TO_FRONT )
```

---

**主函数----实现函数重载**

```text
BaseType_t     xQueueGenericSend	
                ( 	
                    QueueHandle_t 	xQueue,		  
                    const void * const 	pvItemToQueue,				      
                    TickType_t 	xTicksToWait,					        
                    const BaseType_t 	xCopyPosition  
                )
{
    BaseType_t xEntryTimeSet = pdFALSE, xYieldRequired;
	TimeOut_t xTimeOut;
	Queue_t * const pxQueue = ( Queue_t * ) xQueue;
    
    --------------------------------------------------------
    判断数据内容是否正确
    --------------------------------------------------------
    for( ;; )
	{
		taskENTER_CRITICAL();
		//判断队列是否有空闲队列项，或是是否是复写
		if( ( pxQueue->uxMessagesWaiting < pxQueue->uxLength ) || ( xCopyPosition == queueOVERWRITE ) )
			{
				traceQUEUE_SEND( pxQueue );
				//复制数据到队列里面
				xYieldRequired = prvCopyDataToQueue( pxQueue, pvItemToQueue, xCopyPosition );
				//判断是否使用队列集
				#if ( configUSE_QUEUE_SETS == 1 )
				{
					
				}
				#else 
				{
					//判断是否有因为读不到消息而阻塞的任务，有的话，将解除阻塞态---阻塞任务
					//解除阻塞任务，就是将它放入就绪列表，可能任务优先级大于该任务优先级的，所以要进行任务切换
					//通过这个函数xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToReceive )
					//    ----判断调度器是否被挂起
					//    ----没有挂起：将移除相应的事件列表项和状态列表项，并且将任务添加到就绪列表中
					//    ----挂起：将移除事件列表项，将事件列表项添加到等待就绪列表：xPendingReadyList，
					//	  ----当调用恢复调度器时xTaskResumeAll( )，xPendingReadyList中多任务就会被处理
					if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToReceive ) ) == pdFALSE )
					{
						if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToReceive ) ) != pdFALSE )
						{
							
							queueYIELD_IF_USING_PREEMPTION();
						}
						else
						{
							mtCOVERAGE_TEST_MARKER();
						}
					}
					else if( xYieldRequired != pdFALSE )
					{
						
						queueYIELD_IF_USING_PREEMPTION();
					}
					else
					{
						mtCOVERAGE_TEST_MARKER();
					}
				}
				#endif 
				taskEXIT_CRITICAL();
				return pdPASS;
	//此时不能写入消息，因此要将任务阻塞
	else
    {	
    	//如果阻塞时间为0 ，代表不阻塞，直接返回队列满错误
        if( xTicksToWait == ( TickType_t ) 0 )
        {
          
            taskEXIT_CRITICAL();

            traceQUEUE_SEND_FAILED( pxQueue );
            return errQUEUE_FULL;
        }
        //如果阻塞时间不为0，任务需要阻塞，记录下此时系统节拍计数器的值和溢出次数，用于下面对阻塞时间进行补偿
        else if( xEntryTimeSet == pdFALSE )
        {
            vTaskInternalSetTimeOutState( &xTimeOut );
            xEntryTimeSet = pdTRUE;
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
    }
    taskEXIT_CRITICAL();
    vTaskSuspendAll();
	prvLockQueue( pxQueue );
	//判断阻塞时间补偿后，是否还需要阻塞
	if( xTaskCheckForTimeOut( &xTimeOut, &xTicksToWait ) == pdFALSE )
	{
			//需要：将任务的事件列表项添加到等待发送列表中，将任务状态列表项添加到阻塞列表中进行阻塞，队列解锁，恢复调度器	
			if( prvIsQueueFull( pxQueue ) != pdFALSE )
			{
				traceBLOCKING_ON_QUEUE_SEND( pxQueue );
				vTaskPlaceOnEventList( &( pxQueue->xTasksWaitingToSend ), xTicksToWait );
				
				prvUnlockQueue( pxQueue );

				if( xTaskResumeAll() == pdFALSE )
				{
					portYIELD_WITHIN_API();
				}
			}
			else
			{
				prvUnlockQueue( pxQueue );
				( void ) xTaskResumeAll();
			}
	}
	else
	{
		//不需要：队列解锁，恢复调度器，返回队列满错误	
        prvUnlockQueue( pxQueue );
        ( void ) xTaskResumeAll();

        traceQUEUE_SEND_FAILED( pxQueue );
        return errQUEUE_FULL;
	}
  }
}


static BaseType_t prvCopyDataToQueue
										( Queue_t * const pxQueue, 
						 		         const void *pvItemToQueue, 
						  	            const BaseType_t xPosition )
{
	BaseType_t xReturn = pdFALSE;
	UBaseType_t uxMessagesWaiting;
	uxMessagesWaiting = pxQueue->uxMessagesWaiting;
	
	//队列项的大小为0.二值信号的写入
	if( pxQueue->uxItemSize == ( UBaseType_t ) 0 )
	{
		#if ( configUSE_MUTEXES == 1 )
		{
			if( pxQueue->uxQueueType == queueQUEUE_IS_MUTEX )
			{
				xReturn = xTaskPriorityDisinherit( ( void * ) pxQueue->pxMutexHolder );
				pxQueue->pxMutexHolder = NULL;
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}
		#endif 
	}
	//队列尾部写入
	else if( xPosition == queueSEND_TO_BACK )
	{	
		//pxQueue->pcWriteTo 指向第一个队列项，pvItemToQueue写入的内容，uxItemSize大小
		( void ) memcpy( ( void * ) pxQueue->pcWriteTo, pvItemToQueue, ( size_t ) pxQueue->uxItemSize );
        //确定下一个写入的位置
		pxQueue->pcWriteTo += pxQueue->uxItemSize;
		if( pxQueue->pcWriteTo >= pxQueue->pcTail ) 
		{	
			//将写入位置，重新指向第一个，形成环形
			pxQueue->pcWriteTo = pxQueue->pcHead;
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}
	}
	//队列的头部和复写
	else
	{
		( void ) memcpy( ( void * ) pxQueue->u.pcReadFrom, pvItemToQueue, ( size_t ) pxQueue->uxItemSize ); 
		pxQueue->u.pcReadFrom -= pxQueue->uxItemSize;
		if( pxQueue->u.pcReadFrom < pxQueue->pcHead ) 
		{
			pxQueue->u.pcReadFrom = ( pxQueue->pcTail - pxQueue->uxItemSize );
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}
		//判断是否是复写，将空闲队列项先减，再加
		if( xPosition == queueOVERWRITE )
		{
			if( uxMessagesWaiting > ( UBaseType_t ) 0 )
			{
				
				--uxMessagesWaiting;
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

	pxQueue->uxMessagesWaiting = uxMessagesWaiting + ( UBaseType_t ) 1;
	//返回值，判断是否需要进行任务切换，解决任务优先级继承的问题
	return xReturn;
}
```

>- xQueue                        ---待写入的队列
>- pvItemToQueue         ---待写入消息
>- xTicksToWait               ---阻塞超时时间
>- xCopyPosition             ---写入的位置

**队列一共有 3 种写入位置 ：**

```c
#define queueSEND_TO_BACK                     		( ( BaseType_t ) 0 )		/* 写入队列尾部 */
#define queueSEND_TO_FRONT                    		( ( BaseType_t ) 1 )		/* 写入队列头部 */
#define queueOVERWRITE                        		( ( BaseType_t ) 2 )		/* 覆写队列*/
```

**覆写队列消息**

`xQueueOverwrite() `

**在中断向队列写消息**

>在中断中往队列的尾部写入消息：xQueueSendFromISR() 
>在中断中往队列的尾部写入消息：xQueueSendToBackFromISR() 
>在中断中往队列的头部写入消息：xQueueSendToFrontFromISR() 
>在中断中覆写队列消息：xQueueOverwriteFromISR()

### 3、读队列API函数

**从队列中读取消息**

```c
BaseType_t    xQueueReceive( QueueHandle_t   xQueue,  void *   const pvBuffer,  TickType_t   xTicksToWait )
```

>此函数用于在任务中，从队列中读取消息，并且消息读取成功后，会将消息从队列中移除。
>
>- xQueue               ---待读取的队列
>- pvBuffer              ---信息读取缓冲区
>- xTicksToWait       ---阻塞超时时间

```text
BaseType_t xQueueReceive( QueueHandle_t xQueue, void * const pvBuffer, TickType_t xTicksToWait )
{
	BaseType_t xEntryTimeSet = pdFALSE;
	TimeOut_t xTimeOut;
	Queue_t * const pxQueue = ( Queue_t * ) xQueue;
	
	for( ;; )
	{
		taskENTER_CRITICAL();
		{	
			/*判断队列是否有消息，大于0，则有消息*/
			const UBaseType_t uxMessagesWaiting = pxQueue->uxMessagesWaiting;

			if( uxMessagesWaiting > ( UBaseType_t ) 0 )  /*队列有数据*/
			{
				//将头部数据拷贝到了buffer里
				prvCopyDataFromQueue( pxQueue, pvBuffer );
				traceQUEUE_RECEIVE( pxQueue );
				//队列消息-1
				pxQueue->uxMessagesWaiting = uxMessagesWaiting - ( UBaseType_t ) 1;
				/*因为前面已经减了一个队列项，所以队列已经有空位了，如果xTasksWaitingToSend等待发送列表
				中，有任务，则解除阻塞态，通过这个函数xTaskRemoveFromEventList( )
				判断调度器是否被挂起:没有挂起：将移除相应的事件列表项和状态列表项，并且将任务添加到就绪列表中
				挂起：将移除事件列表项，将事件列表项添加到等待就绪列表：xPendingReadyList，
				当调用恢复调度器时xTaskResumeAll( )，xPendingReadyList中多任务就会被处理
				*/
				if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToSend ) ) == pdFALSE )
				{
					if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToSend ) ) != pdFALSE )
					{
						queueYIELD_IF_USING_PREEMPTION();
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

				taskEXIT_CRITICAL();
				return pdPASS;
			}
			else
			{	//此时读取不到消息，因此要将任务阻塞,如果阻塞时间为0 ，代表不阻塞，直接返回队列空错误
				if( xTicksToWait == ( TickType_t ) 0 )
				{
					taskEXIT_CRITICAL();
					traceQUEUE_RECEIVE_FAILED( pxQueue );
					return errQUEUE_EMPTY;
				}
				//如果阻塞时间不为0，任务需要阻塞，记录下此时系统节拍计数器的值和溢出次数，用
				于下面对阻塞时间进行补偿
				else if( xEntryTimeSet == pdFALSE )
				{
					vTaskInternalSetTimeOutState( &xTimeOut );
					xEntryTimeSet = pdTRUE;
				}
				else
				{
					/* Entry time was already set. */
					mtCOVERAGE_TEST_MARKER();
				}
			}
		}
		taskEXIT_CRITICAL();

		vTaskSuspendAll();
		prvLockQueue( pxQueue );

		//判断阻塞时间补偿后，是否还需要阻塞
		if( xTaskCheckForTimeOut( &xTimeOut, &xTicksToWait ) == pdFALSE )
		{
			//需要：将任务的事件列表项添加到等待接收列表中
			//将任务状态列表项添加到阻塞列表中进行阻塞，队列解锁，恢复调度器
			if( prvIsQueueEmpty( pxQueue ) != pdFALSE )
			{
				traceBLOCKING_ON_QUEUE_RECEIVE( pxQueue );
				vTaskPlaceOnEventList( &( pxQueue->xTasksWaitingToReceive ), xTicksToWait );
				prvUnlockQueue( pxQueue );
				if( xTaskResumeAll() == pdFALSE )
				{
					portYIELD_WITHIN_API();
				}
				else
				{
					mtCOVERAGE_TEST_MARKER();
				}
			}
			else
			{
				//不需要：队列解锁，恢复调度器，返回队列空错误
				prvUnlockQueue( pxQueue );
				( void ) xTaskResumeAll();
			}
		}
		else
		{
			prvUnlockQueue( pxQueue );
			( void ) xTaskResumeAll();

			if( prvIsQueueEmpty( pxQueue ) != pdFALSE )
			{
				traceQUEUE_RECEIVE_FAILED( pxQueue );
				return errQUEUE_EMPTY;
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}
	}	
}


static void prvCopyDataFromQueue( Queue_t * const pxQueue, void * const pvBuffer )
{
	if( pxQueue->uxItemSize != ( UBaseType_t ) 0 )
	{	
		/*pxQueue指针指向读队列的指针，加32字节，就指向队列消息的头，也就是从头部对数据*/
		pxQueue->u.pcReadFrom += pxQueue->uxItemSize;
		if( pxQueue->u.pcReadFrom >= pxQueue->pcTail ) 
		{
			pxQueue->u.pcReadFrom = pxQueue->pcHead;
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}
		//将数据拷贝到buffer里面
		( void ) memcpy( ( void * ) pvBuffer, ( void * ) pxQueue->u.pcReadFrom, ( size_t ) pxQueue->uxItemSize ); 
	}
}
```

```test
从队列头部读取消息，并删除消息：xQueueReceive() 
从队列头部读取消息：xQueuePeek() 
在中断中从队列头部读取消息，并删除消息：xQueueReceiveFromISR() 
在中断中从队列头部读取消息：xQueuePeekFromISR()
```

```c
BaseType_t   xQueuePeek( QueueHandle_t   xQueue,   void * const   pvBuffer,   TickType_t   xTicksToWait )
```

>此函数用于在任务中，从队列中读取消息， 但与函数 xQueueReceive()不同，此函数在成功读取消息后，并不会移除已读取的消息！ 

**在中断向队列读消息**

>在中断中从队列头部读取消息，并删除消息

`xQueueReceiveFromISR() `

>在中断中从队列头部读取消息

`xQueuePeekFromISR()`





