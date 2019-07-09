---
title: Android之Binder机制
date: 2019-07-09 16:51:35
tags: IPC
categories: Android艺术探索

---

## 1. 简介

Binder，中文即粘合剂，意思是粘合了两个不同的进程。从IPC角度来说，Binder是Android中的一种跨进程通信方式。

## 2. 基础概念介绍

### 2.1 进程隔离&跨进程通信

- 进程隔离：为了保证安全性和独立性，一般情况下，一个进程不能直接操作或访问另外一个进程。即Android中的进程是相互隔离，独立的
- 进程通信：即IPC，不同进程需要进行数据的交互和通信

### 2.2 内核空间&用户空间

- 一个进程空间分为内核空间和用户空间，即内核空间和用户空间相隔离
- 内核空间：即Karnel Space，是Linux内核的运行空间。可以执行任何命令， 调用系统的一切资源。与用户空间隔离，即使用户的程序崩溃了，内核也不会受到影响。内核空间可进行进程间，进程内的交互。内核空间的数据是共享的，故内核空间=可共享空间
- 用户空间：即User Space，是用户程序的运行空间。只能进行简单的操作，在用户空间的进程不能直接交互，可以通过内核空间来进行间接交互，但是又不能直接调用系统的资源，必须通过系统接口（System Call），向内核发出命令。用户空间的数据不共享，给用户空间=不可共享空间

### 2.3  内存映射

- 定义：关联一个虚拟内存区域和一个磁盘上的共享对象，使两者存在映射关系
- 实现过程：通过Linux系统调用函数：mmap（），这个函数的作用就是创建虚拟内存区域，并与共享对象建立映射关系
- 特点：减少了数据拷贝的次数，并通过映射区域实现用户空间和内核空间的交互

> 传统跨进程通信需要拷贝数据两次，Binder机制只需要1次，主要是因为Binder机制使用到了内存映射

## 3. 四大角色

- Client：使用服务的进程
- Server：提供服务的进程
- ServiceManager：管理系统的Server的注册和查询（将字符形式的Binder名字转化成Client对该Binder的引用）
- Binder驱动：虚拟设备驱动，连接Client，Server，ServiceManager的桥梁

### 3.1 Binder驱动

Binder驱动是Binder通信的核心，它工作于内核空间，提供open，mmap,ioctl等标准文件操作。驱动负责进程间Binder通信的建立。分别管理着Server端的Binder实体核Client端的引用。

## 4. Binder框架

- Binder通信采用C/S模式，包括四个组件：Client,Server,ServiceManager,Binder驱动。其中Client，Server，ServiceManager运行在用户空间，Binder驱动运行在内核空间。
- Binder在Framework层进行封装，通过JNI技术调用Native层的Binder架构
- 在Native层的Binder以ioctl，open等操作与Binder驱动通讯。

## 5. Binder原理及步骤

![](Binder通信模型原理图.png)

### 5.1 注册服务

- Server创建Binder实体，取一个可读易记的名字，将Binder实体连同名字以数据包的形式通过Binder驱动发送给ServiceManager，通知ServiceManger注册这个Binder实体。在这里Server需要通过0号引用与ServiceManager的Binder进行通讯
- Binder驱动为这个穿越边界的Binder实体在内核空间创建节点和ServiceManager的引用，并将名字和引用打包传递给ServiceManager
- ServiceManager收到数据包后，从中取出名字和引用填入查找表中

### 5.2 获取服务

- Client利用保留的0号引用向ServiceManager请求访问某个Binder

- ServiceManager根据请求的数据包获取Binder名字，在查找表中找到该名字对应的Binder引用

- 将该引用作为回复返回给发起请求的Client

  

### 5.2 使用服务

- Client将进程参数发送到Server进程
- Binder驱动为跨进程通信做准备，实现内存映射
- Server根据Client的参数，执行目标方法
- Server进程将目标方法的结果返回给Client进程

## 6. Binder机制的优点

### 6.1 高效

消息队列和管道采用的是存储-转发方式，至少需要两次数据拷贝过程，效率比较低。共享内存虽然无需拷贝，但控制复杂，难以使用。而Binder由于使用了内存映射，只需进行一次数据拷贝，效率高

### 6.2 安全性高

Binder机制为每个进程分配了UID／PID来鉴别身份的标识，并且在Binder通信时会根据UID/PID进行有效性检测

### 6.3 使用简单，可操作性高

Binder机制采用Client/Server的通信方式，对于管道，共享内存，消息队列以及Socket来说只有Socket采用了Client/Server的通信方式，但是Socket主要用于跨网络的进程通信，开销大，效率低。另外Binder机制还实现了面向对象的调用方式。

## 7. Android中的IPC方式

### 7.1 Bundle

两个使用场景：一是直接利用传递数据，在Bundle中附加信息，通过Intent发送出去。二是转移目标跨进程通信，将需要在A进程计算的任务转移到B进程的后台Service中执行。

### 7.2 文件共享

两个进程通过读/写同一个文件实现交换数据。在这里需注意的是SharePreferences,它的底层是使用Xml文件来存储键值对，当使用SharePreferences时，内存中会有一份缓存，故在多进程读写时，就会出现数据过期的状况，另外在高并发读取时，很大几率会出现数据丢失，故不建议在进程通信使用Share Preferences

### 7.4 Messenger

轻量级IPC，其底层的实现其实是AIDL。首先来看看其工作原理图

![](Messenger.png)

 根据工作原理图来分析其实现过程：

#### 服务进程

1. 创建一个Service来处理客户端的连接请求
2. Service中创建一个Handler来接受客户端发送过来的消息，并通过这个Handler来创建一个Messenger对象，在onBind中返回Messenger对象的底层binder

#### 客户进程

1. 首先绑定Service，绑定成功后利用Service返回的Binder对象创建一个Messenger对象
2. 通过这个Messenger对象向服务端发送消息，消息类型为Message

#### 服务进程回复客户进程

1. 当服务进程收到客户进程的消息后，进行回复时，则客户端需在上面的基础上创建一个Handler来接受服务端的回复信息，并利用这个Handler对象创建Messenger对象，并把这个对象通过message的reply参数在发送信息时传递给服务端。
2. 服务端在接受到客户端的信息时，利用携带的reply参数得到Messenger对象，然后利用这个对象就可以向客户端发送message消息类型的消息。

### 7.5 AIDL

#### 大致流程

首先创建一个Service和AIDL接口，接着创建一个类并继承AIDL接口中Stub，重写Stub的抽象方法，在Service中的onBind返回这个类的对象，然后客户端绑定服务端的Service，绑定成功后将返回的Binder对象转化成AIDL接口所属的类型，然后就可以调用接口中的方法。

#### 难点

1. 支持的数据类型

- 基本数据类型
- String和CharSequence
- List:只支持ArrayList，里面的元素必须能够被AIDL支持
- map：只支持HashMap，键值都必须能够被AIDL支持
- Parcelable
- AIDL

2. AIDL文件中使用到自定义的Parcelable对象都要新建一个和它同名的AIDL文件，只要在文件中声明那个类为Parcelable即可
3. 自定义的Parcelable对象和AIDL对象必须都要显示import进来
4. 除基本类型外，其他类型需标上方向，in、out、inout
5. AIDL接口只支持方法，不支持声明静态常量

### 7.6 ContentProvider

ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，故它天生就适合进程间的通讯，和Messenger一样，它的底层实现也是Binder

### 7.7  Socket

又称套接字，网络通信中的概念。主要分为流式套接字（TCP）和用户数据报套接字（UDP）

### 7.8 Android中各种IPC方式的优缺点

| 名称 | 优点 | 缺点 | 适用场景 |
| ---- | ---- | ---- | -------- |
| Bundle          | 简单易用                                             | 只能传输Bundle支持的数据类型                                 |                   四大组件的进程间的通信                   |
| 文件共享        | 简单易用                                             | 不适合高并发场景，无法做到即时通信                           |       无并发访问情况，交换简单的数据实时性不高的场景       |
| AIDL            | 功能强大，支持一对多并发通信，支持实时通信           | 操作较复杂，需处理好线程同步                                 |                   一对多通信，有RPC需求                    |
| Messenger       | 支持一对多的串行通信，支持实时通信                   | 不能很好处理高并发情形，不支持RPC，数据通过Message传递，故只能传输Bundle支持的数据类型 | 低并发的一对多及时需求，无RPC需求，或者无返回结果的RPC需求 |
| ContentProvider | 数据访问功能强大，支持一对多并发数据共享             | 受约束的AIDL                                                 |                  一对多的进程间的数据共享                  |
| Socket          | 功能强大，可以通过网络传递字节流，支持一对多实时通信 | 实现细节繁琐，不支持直接的RPC                                |                        网络数据交换                        |

## 参考资料

[图文详解 Android Binder跨进程通信的原理](https://www.jianshu.com/p/4ee3fd07da14)

[操作系统：图文详解 内存映射](https://www.jianshu.com/p/719fc4758813)

[Android Bander设计与实现](https://blog.csdn.net/universus/article/details/6211589)

[一篇文章了解相见恨晚的 Android Binder 进程间通讯机制](https://blog.csdn.net/freekiteyu/article/details/70082302)

开发艺术探索















   





