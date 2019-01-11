### 构建handler消息机制中的几个阶段分别作了什么事？
1. 准备阶段 Looper.prepare();
    - 在当前线程实例化了 Looper对象，并放到静态变量sThreadLocal中。而构造Looper的时候，实例化了MessageQueue对象，并持有native的一个指针，方便后续的消息导致线程唤醒以及阻塞的状态
2. 循环阶段 Looper.loop();
    - 通过sThreadLocal获取到当前线程的looper对象，并启动无限循环。从MessageQueue中获得msg，若无消息，则通过native层将线程处于阻塞状态
3. 发送消息 Handler.sendMessage()
    - 使用当前线程创建的handler，在其他线程通过调用sendMessage()方法发送msg到MessageQueue；MessageQueue插入新的Msg，然后通过native层唤醒线程，然后MessageQueue的next方法就会返回msg对象，
4. 获取消息 Handler.dispatchMessage()
    - 在MessageQueue的next方法就会返回msg对象后，开始使用具体的target进行消息分发，到handler所处的线程中，处理消息


### 为什么我们需要这样的消息处理机制？

1. 不阻塞主线程

    Android应用程序启动时，系统会创建一个主线程，负责与UI组件（widget、view）进行交互，比如控制UI界面界面显示、更新等；分发事件给UI界面处理，比如按键事件、触摸事件、屏幕绘图事件等，因此，Android主线程也称为UI线程。

2. 并发程序设计的有序性

    单线程模型的UI主线程也是不安全的，会造成不可确定的结果。
    
    Android只允许主线程更新UI界面，子线程处理后的结果无法和主线程交互，即无法直接访问主线程，这就要用到Handler机制来解决此问题。基于Handler机制，在子线程先获得Handler对象，该对象将数据发送到主线程消息队列，主线程通过Loop循环获取消息交给Handler处理。


### 是如何完成跨线程通信的？

    Handler发送消息后添加消息到消息队列，然后消息在恰当时候出列，都是由Handler来执行，那么是如何完成跨线程通信的？

    这里就牵涉到了Linux系统的跨线程通信的知识，**Android中采用的是Linux中的管道通信**
    Looper是通过管道(pipe)实现的。

    关于管道，简单来说，管道就是一个文件。在管道的两端，分别是两个打开文件文件描述符，这两个打开文件描述符都是对应同一个文件，其中一个是用来读的，别一个是用来写的。

    一般的使用方式就是，一个线程通过读文件描述符中来读管道的内容，当管道没有内容时，这个线程就会进入等待状态，而另外一个线程通过写文件描述符来向管道中写入内容，写入内容的时候，如果另一端正有线程正在等待管道中的内容，那么这个线程就会被唤醒。
    
    这个等待和唤醒的操作是如何进行的呢? 这就要借助Linux系统中的epoll机制了。 
    Linux系统中的epoll机制为处理大批量句柄而作了改进的poll，是Linux下多路复用IO接口select/poll的增强版本，它能显著减少程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。
    (01) pipe(wakeFds)，该函数创建了两个管道句柄。
    (02) mWakeReadPipeFd=wakeFds[0]，是读管道的句柄。
    (03) mWakeWritePipeFd=wakeFds1，是写管道的句柄。
    (04) epoll_create(EPOLL_SIZE_HINT)是创建epoll句柄。
    (05) epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem)，它的作用是告诉mEpollFd，它要监控mWakeReadPipeFd文件描述符的EPOLLIN事件，即当管道中有内容可读时，就唤醒当前正在等待管道中的内容的线程。

    epoll是Linux中的一种IO多路复用方式，也叫做event-driver-IO。

    Linux的select 多路复用IO通过一个select()调用来监视文件描述符的数组，然后轮询这个数组。如果有IO事件，就进行处理。

    select的一个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，select()所维护的存储大量文件描述符的数据结构，随着文件描述符数量的增大，其复制的开销也线性增长。

    epoll在select的基础上（实际是在poll的基础上）做了改进，epoll同样只告知那些就绪的文件描述符，而且当我们调用epoll_wait()获得就绪文件描述符时，返回的不是实际的描述符，而是一个代表就绪描述符数量的值，你只需要去epoll指定的一个数组中依次取得相应数量的文件描述符即可。

    另一个本质的改进在于epoll采用基于事件的就绪通知方式（设置回调）。在select中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知

    关于epoll和select，可以举一个例子来表达意思。select的情况和班长告诉全班同学交作业类似，会挨个去询问作业是否完成，如果没有完成，班长会继续询问。

    而epoll的情况则是班长询问的时候只是统计了待交作业的人数，然后告诉同学作业完成的时候告诉把作业放在某处，然后喊一下他。然后班长每次都去这个地方收作业。

    这样一个线程（比如UI线程）消息队列和Looper就准备就绪了。

    消息队列创建时，会调用JNI函数，初始化NativeMessageQueue对象。NativeMessageQueue则会初始化Looper对象。Looper的作用就是，当Java层的消息队列中没有消息时，就使Android应用程序主线程进入等待状态，而当Java层的消息队列中来了新的消息后，就唤醒Android应用程序的主线程来处理这个消息。


### 可以在子线程中创建Handler吗? 为什么每个线程只会有一个Looper?
    1.可以在子线程中创建handler，不过要在创建前，先调用Looper.prepare()方法，创建一个Looper+MessageQueue，否则报错！
    2.因为在调用Looper.prepare()方法的时候，会在当前线程实例化一个looper对象，并存储到sThreadLocal中。

