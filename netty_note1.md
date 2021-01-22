# 一、入门



## 1、线程模型



### 1)、传统阻塞I/O服务模型

---

![image-20201026202934255](https://github.com/Woodyiiiiiii/Netty_learning/blob/master/img/image-20201026202934255.png)

黄色的框表示对象， 蓝色的框表示线程，白色的框表示方法(API)。



**特点：**

1. 采用阻塞IO模式获取输入的数据
2. 每个连接都需要独立的线程完成数据的输入，业务处理，数据返回

**问题：**

1. 当并发数很大，就会创建大量的线程，占用很大系统资源
2. 连接创建后，如果当前线程暂时没有数据可读，该线程会阻塞在read 操作，造成线程资源浪费



### 2)、Reactor模式

---

* 单Reactor单线程

* 单Reactor多线程

* 主从Reactor多线程 



基于 I/O 复用模型：多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象等待，无需阻塞等待所有连接。当某个连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理

Reactor 对应的叫法: 1. 反应器模式 2. 分发者模式(Dispatcher) 3. 通知者模式(notifier)。

基于线程池复用线程资源：不必再为每个连接创建线程，将连接完成后的业务处理任务分配给线程进行处理，一个线程可以处理多个连接的业务。

**单Reactor单线程模式下：**

![image-20201026203408683](https://github.com/Woodyiiiiiii/Netty_learning/blob/master/img/image-20201026203408683.png)



**单Reactor多线程模式下：**

![image-20201026203851265](https://github.com/Woodyiiiiiii/Netty_learning/blob/master/img/image-20201026203851265.png)

1. Reactor 对象通过select 监控客户端请求事件, 收到事件后，通过dispatch进行分发
2. 如果建立连接请求, 则有Acceptor 通过accept 处理连接请求, 然后创建一个Handler对象处理完成连接后的各种事件
3. 如果不是连接请求，则由reactor分发调用连接对应的handler 来处理
4. handler 只负责响应事件，不做具体的业务处理, 通过read 读取数据后，会分发给后面的worker线程池的某个线程处理业务
5. worker 线程池会分配独立线程完成真正的业务，并将结果返回给handler
6. handler收到响应后，通过send 将结果返回给client



**主从Reactor多线程：**

![image-20201026204141970](https://github.com/Woodyiiiiiii/Netty_learning/blob/master/img/image-20201026204141970.png)

1. Reactor主线程 MainReactor 对象通过select 监听连接事件, 收到事件后，通过Acceptor 处理连接事件
2. 当 Acceptor 处理连接事件后，MainReactor 将连接分配给SubReactor 
3. subreactor 将连接加入到连接队列进行监听,并创建handler进行各种事件处理
4. 当有新事件发生时， subreactor 就会调用对应的handler处理
5. handler 通过read 读取数据，分发给后面的worker 线程处理
6. worker 线程池分配独立的worker 线程进行业务处理，并返回结果
7. handler 收到响应的结果后，再通过send 将结果返回给client
8. Reactor 主线程可以对应多个Reactor 子线程, 即MainRecator 可以关联多个SubReactor



**比喻：**

1. 单 Reactor 单线程，前台接待员和服务员是同一个人，全程为顾客服
2. 单 Reactor 多线程，1 个前台接待员，多个服务员，接待员只负责接待
3. 主从 Reactor 多线程，多个前台接待员，多个服务生



Netty 主要基于**主从** **Reactors** **多线程模型**（如图）做了一定的改进，其中主从 Reactor 多线程模型有多个 Reactor。



**原理图如下：**

![image-20201026205516878](https://github.com/Woodyiiiiiii/Netty_learning/blob/master/img/image-20201026205516878.png)

1. Netty抽象出两组线程池 BossGroup 专门负责接收客户端的连接, WorkerGroup 专门负责网络的读写
2. BossGroup 和 WorkerGroup 类型都是 NioEventLoopGroup
3. NioEventLoopGroup 相当于一个事件循环组, 这个组中含有多个事件循环 ，每一个事件循环是 NioEventLoop
4. NioEventLoop 表示一个不断循环的执行处理任务的线程， 每个NioEventLoop 都有一个selector , 用于监听绑定在其上的socket的网络通讯
5. NioEventLoopGroup 可以有多个线程, 即可以含有多个NioEventLoop
6. 每个Boss NioEventLoop 循环执行的步骤有3步
   1. 轮询accept 事件
   2. 处理accept 事件 , 与client建立连接 , 生成NioScocketChannel , 并将其注册到某个worker NIOEventLoop 上的 selector
   3. 处理任务队列的任务 ， 即 runAllTasks
7. 每个 Worker NIOEventLoop 循环执行的步骤
   1. 轮询read, write 事件
   2. 处理i/o事件， 即read , write 事件，在对应NioScocketChannel 处理
   3. 处理任务队列的任务 ， 即 runAllTasks
8. 每个Worker NIOEventLoop  处理业务时，会使用pipeline(管道), pipeline 中包含了 channel , 即通过pipeline 可以获取到对应通道, 管道中维护了很多的 处理器
