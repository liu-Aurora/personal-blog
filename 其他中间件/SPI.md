# SPI通信协议

## 一、概述

>SPI（Serial Peripheral Interface，串行外设接口）是一种高速、全双工、同步的通信总线，常用于微控制器与外设之间的数据交换，例如传感器、存储器、显示屏等。
>
> 
>
>**SPI 的特点:**
>
>- **高速:** SPI 的通信速率可以达到几十 MHz，比 I²C 总线快得多。
>- **全双工:** SPI 可以同时进行数据的发送和接收。
>- **同步:** SPI 使用一个时钟信号来同步数据的传输，保证数据传输的可靠性。
>- **简单:** SPI 的硬件连接和软件驱动都比较简单，易于实现。
>- **灵活:** SPI 支持多种工作模式，可以根据不同的应用场景进行配置。

## 二、SPI协议

### 1、**SPI 的四种信号线:**

- **SCLK (Serial Clock):** 串行时钟，由主设备产生，用于同步数据传输。
- **MOSI (Master Out Slave In):** 主设备输出，从设备输入，用于主设备向从设备发送数据。
- **MISO (Master In Slave Out):** 主设备输入，从设备输出，用于从设备向主设备发送数据。
- **SS/CS (Slave Select/Chip Select):** 从设备选择/片选，由主设备控制，用于选择与哪个从设备进行通信。

### 2、SPI数据发送接收流程

**SPI主机和从机都有一个串行移位寄存器，主机通过向它的SPI串行寄存器写入一个字节来发起一次传输。**

1. **首先拉低片选信号线，表示与该设备进行通信**

2. **主机通过发送SCLK时钟信号，来告诉从机写数据或者读数据的时序**

> **SCLK时钟信号可能是低电平有效，也可能是高电平有效，因为SPI有四种模式，所以SPI 支持多种工作模式，可以根据不同的应用场景进行配置。**

3. **主机(Master)将要发送的数据写到发送数据缓存区(Menory)，缓存区经过移位寄存器(0~7)，串行移位寄存器通过MOSI信号线将字节一位一位的移出去传送给从机，，同时MISO接口接收到的数据经过移位寄存器一位一位的移到接收缓存区**
4. **从机(Slave)也将自己的串行移位寄存器(0~7)中的内容通过MISO信号线返回给主机。同时通过MOSI信号线接收主机发送的数据，这样，两个移位寄存器中的内容就被交换。**

![image-20240909162657697](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909162657697.png)

### 3、SPI工作模式

**时钟极性(CPOL)：**没有数据传输时时钟线的空闲状态电平

- 0：SCK在空闲状态保持低电平
- 1：SCK在空闲状态保持高电平

**时钟相位(CPHA)：**时钟线在第几个时钟边沿采样数据

- 0：SCK的第一(奇数)边沿进行数据位采样，数据在第一个时钟边沿被锁存
- 1：SCK的第二(偶数)边沿进行数据位采样，数据在第二个时钟边沿被锁存

![image-20240909163049130](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909163049130.png)

### 4、软件模拟SPI通信时序

<img src="C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909163521867.png" alt="image-20240909163521867" style="zoom:80%;" />	

## 三、硬件SPI通信时序

### 1、SPI结构框图介绍

​	![image-20240909163814069](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909163814069.png)

- ① SPI相关引脚：MOSI（输出数据线）、MISO（输入数据线）、SCK（时钟）、NSS（片选）
- ② 数据发送和接收：与缓冲区、移位寄存器以及引脚相关
- ③ 时钟信号：SPI时钟信号是通过SPI_CR1寄存器配置
- ④ 主控制逻辑：涉及两个控制寄存器SPI_CR1/2用于配置SPI工作，SPI_SR用于查看工作状态

### 2、stm32相关寄存器内容

![image-20240909164717604](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909164717604.png)

**SPI控制寄存器（配置相关工作参数）**

![image-20240909164735726](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909164735726.png)

![image-20240909164803703](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909164803703.png)

**SPI状态寄存器（SPI_SR）**

![image-20240909165001153](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909165001153.png)

**SPI数据寄存器（SPI_DR）**

![image-20240909165033973](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909165033973.png)

### 3、stm32相关HAL库驱动

![image-20240909165200842](C:\Users\15924\AppData\Roaming\Typora\typora-user-images\image-20240909165200842.png)