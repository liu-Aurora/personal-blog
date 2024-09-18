# cJSON使用详细介绍

## 一、JSON与cJSON

>**JSON —— 轻量级的数据格式**
>
>> [JSON](https://www.json.org/) 全称 JavaScript Object Notation，即 JS对象简谱，是一种**轻量级的数据格式**。
>>
>> 它采用完全独立于编程语言的文本格式来存储和表示数据，语法简洁、层次结构清晰，易于人阅读和编写，同时也易于机器解析和生成，有效的提升了网络传输效率
>
>**JSON语法规则**
>
>>JSON对象是一个无序的"名称/值"键值对的集合：
>>
>>- 以"`{`“开始，以”`}`"结束，允许嵌套使用；
>>- 每个名称和值成对出现，名称和值之间使用"`:`"分隔；
>>- 键值对之间用"`,`"分隔
>>- 在这些字符前后允许存在无意义的空白符
>>
>>对于键值，可以有如下值：
>>
>>- 一个新的json对象
>>- 数组：使用"`[`“和”`]`"表示
>>- 数字：直接表示，可以是整数，也可以是浮点数
>>- 字符串：使用引号`"`表示
>>- 字面值：false、null、true中的一个(必须是小写)
>
>**cJSON**：cJSON是一个使用C语言编写的JSON数据解析器，具有超轻便，可移植，单文件的特点，使用MIT开源协议。
>
>>从Github拉取cJSON源码后，文件非常多，但是其中cJSON的源码文件只有两个：
>>
>>- `cJSON.h`
>>- `cJSON.c`

## 二、 cJSON数据结构和设计思想

**cJSON的设计思想从其数据结构上就能反映出来。**

cJSON使用cJSON结构体来表示**一个JSON数据**，定义在`cJSON.h`中，源码如下

```test
typedef struct cJSON
{
    struct cJSON *next;				//指向下一个键值对
    struct cJSON *prev;				//指针指向上一个键值对
       
    int type;						//用于表示该键值对中值的类型；
    struct cJSON *child;			//child指针指向该条新链表

    char *valuestring;				//如果键值类型(type)是字符串，则将该指针指向键值；
    int valueint;					//如果键值类型(type)是整数，则将该指针指向键值
    double valuedouble;				//如果键值类型(type)是浮点数，则将该指针指向键值；
    char *string;				    //用于表示该键值对的名称；
} cJSON;

```

首先，**它不是将一整段JSON数据抽象出来，而是将其中的一条JSON数据抽象出来，也就是一个键值对**

其次，一段完整的JSON数据中由很多键值对组成，并且涉及到键值对的查找、删除、添加，所以**使用链表来存储整段JSON数据**

最后，因为JSON数据支持嵌套，所以一个**键值对的值会是一个新的JSON数据对象（一条新的链表）**

## 三、JSON数据封装

封装JSON数据的过程，其实就是创建链表和向链表中添加节点的过程。

首先来讲述一下链表中的一些术语：

- 头指针：指向链表头结点的指针；
- 头结点：不存放有效数据，方便链表操作；
- 首节点：第一个存放有效数据的节点；
- 尾节点：最后一个存放有效数据的节点；

**① 创建头指针**

 ```c
  cJSON* cjson_test = NULL;
 ```

**② 创建头结点，并将头指针指向头结点**

```c
cjson_test = cJSON_CreateObject();
```

**③ 尽情的向链表中添加节点**

```c
cJSON_AddNullToObject(cJSON * const object, const char * const name);

cJSON_AddTrueToObject(cJSON * const object, const char * const name);

 /* 添加一个值为 False 的布尔类型的JSON数据(添加一个链表节点) */
cJSON_AddFalseToObject(cJSON * const object, const char * const name);

cJSON_AddBoolToObject(cJSON * const object, const char * const name, const cJSON_bool boolean);

 /* 添加一条整数、浮点类型的JSON数据(添加一个链表节点) */
cJSON_AddNumberToObject(cJSON * const object, const char * const name, const double number);

  /* 添加一条字符串类型的JSON数据(添加一个链表节点) */
cJSON_AddStringToObject(cJSON * const object, const char * const name, const char * const string);

cJSON_AddRawToObject(cJSON * const object, const char * const name, const char * const raw);

cJSON_AddObjectToObject(cJSON * const object, const char * const name);

cJSON_AddArrayToObject(cJSON * const object, const char * const name);
```

**输出JSON数据**

cJSON提供了一个API，可以将整条链表中存放的JSON信息输出到一个字符串中

```c
(char *) cJSON_Print(const cJSON *item
```

![image-20240804140326092](D:\markdown\其他中间件\assets\image-20240804140326092.png)

## 四、cJSON数据解析

**解析方法**：解析JSON数据的过程，其实就是剥离一个一个链表节点(键值对)的过程。

**① 创建链表头指针**

```C
cJSON* cjson_test = NULL;
```

**② 解析整段JSON数据，并将链表头结点地址返回，赋值给头指针**

```c
(cJSON *) cJSON_Parse(const char *value);
```

**③ 根据键值对的名称从链表中取出对应的值，返回该键值对(链表节点)的地址**

```c
(cJSON *) cJSON_GetObjectItem(const cJSON * const object, const char * const string);
```

**④ 如果JSON数据的值是数组，使用下面的两个API提取数据：**

```c
(int) cJSON_GetArraySize(const cJSON *array);
(cJSON *) cJSON_GetArrayItem(const cJSON *array, int index);
```

 **解析示例**

```c
#include <stdio.h>
#include "cJSON.h"

char *message = 
"{                              \
    \"name\":\"mculover666\",   \
    \"age\": 22,                \
    \"weight\": 55.5,           \
    \"address\":                \
        {                       \
            \"country\": \"China\",\
            \"zip-code\": 111111\
        },                      \
    \"skill\": [\"c\", \"Java\", \"Python\"],\
    \"student\": false          \
}";

int main(void)
{
    cJSON* cjson_test = NULL;
    cJSON* cjson_name = NULL;
    cJSON* cjson_age = NULL;
    cJSON* cjson_weight = NULL;
    cJSON* cjson_address = NULL;
    cJSON* cjson_address_country = NULL;
    cJSON* cjson_address_zipcode = NULL;
    cJSON* cjson_skill = NULL;
    cJSON* cjson_student = NULL;
    int    skill_array_size = 0, i = 0;
    cJSON* cjson_skill_item = NULL;

    /* 解析整段JSO数据 */
    cjson_test = cJSON_Parse(message);
    if(cjson_test == NULL)
    {
        printf("parse fail.\n");
        return -1;
    }

    /* 依次根据名称提取JSON数据（键值对） */
    cjson_name = cJSON_GetObjectItem(cjson_test, "name");
    cjson_age = cJSON_GetObjectItem(cjson_test, "age");
    cjson_weight = cJSON_GetObjectItem(cjson_test, "weight");

    printf("name: %s\n", cjson_name->valuestring);
    printf("age:%d\n", cjson_age->valueint);
    printf("weight:%.1f\n", cjson_weight->valuedouble);

    /* 解析嵌套json数据 */
    cjson_address = cJSON_GetObjectItem(cjson_test, "address");
    cjson_address_country = cJSON_GetObjectItem(cjson_address, "country");
    cjson_address_zipcode = cJSON_GetObjectItem(cjson_address, "zip-code");
    printf("address-country:%s\naddress-zipcode:%d\n", cjson_address_country->valuestring, cjson_address_zipcode->valueint);

    /* 解析数组 */
    cjson_skill = cJSON_GetObjectItem(cjson_test, "skill");
    skill_array_size = cJSON_GetArraySize(cjson_skill);
    printf("skill:[");
    for(i = 0; i < skill_array_size; i++)
    {
        cjson_skill_item = cJSON_GetArrayItem(cjson_skill, i);
        printf("%s,", cjson_skill_item->valuestring);
    }
    printf("\b]\n");

    /* 解析布尔型数据 */
    cjson_student = cJSON_GetObjectItem(cjson_test, "student");
    if(cjson_student->valueint == 0)
    {
        printf("student: false\n");
    }
    else
    {
        printf("student:error\n");
    }
    
    return 0;
}
```

在本示例中，因为我提前知道数据的类型，比如字符型或者浮点型，所以我直接使用指针指向对应的数据域提取，**在实际使用时，如果提前不确定数据类型，应该先判断type的值，确定数据类型，再从对应的数据域中提取数据**。

## 五、STM32堆栈空间大小设置

**STM32堆栈空间**

> 在使用STM32编程时，一般情况下我们不会关注堆栈空间的大小，因为在STM32的启动文件中，已经帮我们预先设置好了堆栈空间的大小。如下图所示的启动代码中，**Stack栈的大小为：0x400（1024Byte），Heap堆的大小为：0x200（512Byte）。**
>
> > **这也是为什么一个基础的工程编译后，RAM的空间也占用了1.6K左右的原因，因为堆栈的空间均分配在RAM中，可在编译的map文件中查看RAM资源占用的情况。**
>
> 若工程中使用的局部变量较多，定义的数据长度较大时，若不调整栈的空间大小，则会导致程序出现栈溢出，程序运行结果与预期的不符或程序跑飞。这时我们就需要手动的调整栈的大小。
>
> 当工程中使用了malloc动态分配内存空间时，这时分配的空间就为堆的空间。所以若默认的堆空间大小不满足工程需求时，就需要手动调整堆空间的大小。

**理解堆和栈的区别**

> - 栈区（stack）：由编译器自动分配和释放，存放函数的参数值、局部变量的值等，其操作方式类似于数据结构中的栈。
> - 堆区（heap）：一般由程序员分配和释放，若程序员不释放，程序结束时可能由操作系统回收。分配方式类似于数据结构中的链表。 
> - 全局区（静态区）（static）：全局变量和静态变量的存储是放在一块的，初始化的全局变量和静态变量在一块区域，未初始化的全局变量和未初始化的静态变量在相邻的另一块区域。程序结束后由系统自动释放。 
> - 文字常量区：常量字符串就是存放在这里的。 
> - 程序代码区：存放函数体的二进制代码。
>
> >注意：堆和栈，一般堆是由低地址往上（高地址）增长，栈是由高地址向下（低地址）增长。都是连续的，C语言不提供内存保护机制类似的功能，如果一直堆一直增长，栈一直申请，然后就会导致栈溢出，程序崩溃。

## 六、cJSON使用过程中的内存问题

cJSON的所有操作都是基于链表的，所以cJSON在使用过程中**大量的使用`malloc`从堆中分配动态内存的，所以在使用完之后，应当及时调用下面的函数，清空cJSON指针所指向的内存**，该函数也可用于删除某一条数据：

```C
(void) cJSON_Delete(cJSON *item);
```

>注意：该函数删除一条JSON数据时，如果有嵌套，会连带删除

 **在STM32里移植cJSON遇到的问题和解决方法**

1. cJSON_Parse();函数的返回值一直是0

>原因，可能是因为接收到的文本不是标准的JSON格式。建议可以在后面加一个判断，判断返回值是否为NULL，如果是，调用cJSON_GetErrorPtr();函数来获知在哪里运行出错。如果不是，表明对JSON的解析完成

2. cJSON_Print();函数的返回值一直是0

>原因，可能是因为malloc不成功的原因。也就是给cJSON分配的内存较小。在STM32里，如果不是用自己编写的malloc函数，而是用的通用的malloc函数，那么需要在keil的Option->Target界面把“Use MiceoLIB”勾选。即使用微函数。
>
>并且加入Print函数对于个数较小的JSON能够打印，但是对于个数较多的JSON不能打印，则还需要在文件中寻找Heap_Size变量，然后将后面的数字改大一些，以给cJSON分配更多的内存。

3. cJSON_GetObjectItem();函数的返回值为0

>原因：返回值为0表明没有找到该键名对应的cJSON结构体。因为cJSON_GetObjectItem()只能对get到object的子系，不能get到object的子系的子系。也就是上文我们所说的，剥洋葱只能剥一层这个概念。所以注意是不是所写的键名是更内层的cJSON。