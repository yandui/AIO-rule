# AIO-rule
How many relations are they between IO and AIO 

# Linux中AIO设计框架
## 概述

> aio异步读写是在Linux内核2.6之后才正式纳入其标准。之所以会增加此模块，是因为众所周知我们计算机CPU的执行速度远大于I/O读写的执行速度，如果我们用传统的阻塞式或非阻塞式来操作I/O的话，那么我们在同一个程序中(不用多线程或多进程)就不能同时操作俩个以上的文件I/O，每次只能对一个文件进行I/O操作，很明显这样效率很低下(因为CPU速度远大于I/O操作的速度，所以当执行I/O时，CPU其实还可以做更多的事)。因此就诞生了相对高效的异步I/O
>  --------------------------------------

##生活中的例子

---------
* 1.<font color=red size=2 face=“黑体”>同步买奶茶：
<font color=black size=2 face=“黑体”>小明点单交钱，然后等着拿奶茶。
* 2. <font color=red size=2 face=“黑体”>异步买奶茶：
<font color=black size=2 face=“黑体”> 小明点单交钱，店员给小明一个小票，等小明奶茶做好了，再来取

 <font color=red size=2 face=“黑体”>异步判断奶茶是否做好，有两种方式：
> <font color=black size=2 face=“黑体”>
> ----------------------
 1.小明主动去问店员，一会就问一下:"奶茶做好了吗?"...直到奶茶做好
 ，这叫轮训.

>2.等奶茶做好了,店员喊一声:"小明，奶茶好了!",然后小明去取奶茶,这叫回调

----------------------------

## 类比阻塞IO机制
##同步IO

### 同步阻塞IO模型
* 同步阻塞 I/O 模型。在这个模型中，用户空间的应用程序执行一个系统调用，这会导致应用程序阻塞。这意味着应用程序会一直阻塞，直到系统调用完成为止（数据传输完成或发生错误）。调用应用程序处于一种不再消费 CPU 而只是简单等待响应的状态，因此从处理的角度来看，这是非常有效的。
![同步阻塞 I/O 模型的典型流程](https://www.ibm.com/developerworks/cn/linux/l-async/figure2.gif)

-----------------------------------------
### 同步非阻塞 I/O
* 同步阻塞 I/O 的一种效率稍低的变种是同步非阻塞 I/O。在这种模型中，设备是以非阻塞的形式打开的。这意味着 I/O 操作不会立即完成，read操作可能会返回一个错误代码，说明这个命令不能立即满足（EAGAIN 或 EWOULDBLOCK）(没有从内核读取数据时，是阻塞的)，如图 3 所示。
![同步非阻塞 I/O 模型的典型流程](https://www.ibm.com/developerworks/cn/linux/l-async/figure3.gif)

---------------------------------------
##异步IO

### 异步阻塞 I/O
* IO多路复用(复用的select线程)。I/O复用模型会用到select、poll、epoll函数，这几个函数也会使进程阻塞，但是和阻塞I/O所不同的的，这两个函数可以同时阻塞多个I/O操作。而且可以同时对多个读操作，多个写操作的I/O函数进行检测，直到有数据可读或可写时，才真正调用I/O操作函数。对于每个提示符来说，我们可以获取这个描述符可以写数据、有读数据可用以及是否发生错误的通知。

![异步阻塞 I/O 模型的典型流程](https://www.ibm.com/developerworks/cn/linux/l-async/figure4.gif)

* epoll支持水平触发和边缘触发，最大的特点在于边缘触发，它只告诉进程哪些fd刚刚变为就需态，并且只会通知一次。
* epoll使用“事件”的就绪通知方式，通过 epollctl 注册fd，一旦该fd就绪，内核就会采用类似callback的回调机制
来激活该fd，epoll_wait便可以收到通知.

#### epoll的优点
*  没有最大并发连接的限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）。
*  效率提升，不是轮询的方式,只管你“活跃”的连接，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数。
*  内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递；即epoll使用mmap减少复制开销。


##### Tips

> * 表面上看epoll的性能最好，但是在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。

-------------------------------------
### 异步非阻塞 I/O
* 异步非阻塞 I/O 模型是一种CPU处理与 I/O 重叠进行的模型。读请求会立即返回，说明 read 请求已经成功发起了。在后台完成读操作时，应用程序然后会执行其他处理操作。当 read 的响应到达时，就会产生一个信号或执行一个基于线程的回调函数来完成这次 I/O 处理过程。


![异步非阻塞 I/O 模型流程](https://www.ibm.com/developerworks/cn/linux/l-async/figure5.gif)


---------------------------

## AIO编程方法
	
> AIO 接口的 API 非常简单，但是它为数据传输提供了必需的功能，并给出了两个不同的通知模型。表 1 给出了 AIO 的接口函数，本节稍后会更详细进行介绍。

![AIO基本API介绍](https://i.imgur.com/irAcWoN.png)


### 对象 
* struct aiocb

```     
    	 	 
    struct aiocb
    {
      int aio_fildes;   // File Descriptor
      int aio_lio_opcode;   // Valid only for lio_listio(r/w/nop) AIO_READ AIO_WRITE 
      volatile void *aio_buf;   // Data Buffer
      size_t aio_nbytes;// Number of Bytes in Data Buffer
      struct sigevent aio_sigevent; // Notification Structure
      /* Internal fields */
      ...
    };   
```
              




------------------------
