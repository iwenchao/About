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


### Looper 死循环为什么不会导致应用卡死，会消耗大量资源吗？
    首先线程默认没有Looper的，如果需要使用Handler就必须为线程创建Looper。我们经常提到的主线程，也叫UI线程，它就是ActivityThread，ActivityThread被创建时就会初始化Looper，这也是在主线程中默认可以使用Handler的原因。

    其次那么在所有的事情完成以后应该调用quit方法来终止消息循环，否则这个线程就会一直处于等待（阻塞）状态，而如果退出Looper以后，这个线程就会立刻（执行所有方法并）终止，因此建议不需要的时候终止Looper。

    那么回到我们的问题上，这个死循环会不会导致应用卡死，即使不会的话，它会慢慢的消耗越来越多的资源吗？对于线程即是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，例如，binder线程也是采用死循环的方法，通过循环方式不同与Binder驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？通过创建新线程的方式。真正会卡死主线程的操作是在回调方法onCreate/onStart/onResume等操作时间过长，会导致掉帧，甚至发生ANR，looper.loop本身不会导致应用卡死。 

    主线程的死循环一直运行是不是特别消耗CPU资源呢？ 其实不然，这里就涉及到Linux pipe/epoll机制，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。








### 主线程的消息循环机制是什么（死循环如何处理其它事务）？
    事实上，会在进入死循环之前便创建了新binder线程，在代码ActivityThread.main()中：
    ```
        public static void main(String[] args) {
                ....

                //创建Looper和MessageQueue对象，用于处理主线程的消息
                Looper.prepareMainLooper();

                //创建ActivityThread对象
                ActivityThread thread = new ActivityThread(); 

                //建立Binder通道 (创建新线程)
                thread.attach(false);

                Looper.loop(); //消息循环运行
                throw new RuntimeException("Main thread loop unexpectedly exited");
            }
    ```
    Activity的生命周期都是依靠主线程的Looper.loop，当收到不同Message时则采用相应措施：一旦退出消息循环，那么你的程序也就可以退出了。 
    从消息队列中取消息可能会阻塞，取到消息会做出相应的处理。如果某个消息处理时间过长，就可能会影响UI线程的刷新速率，造成卡顿的现象。

    thread.attach(false)方法函数中便会创建一个Binder线程（具体是指ApplicationThread，Binder的服务端，用于接收系统服务AMS发送来的事件），该Binder线程通过Handler将Message发送给主线程。「Activity 启动过程」
    比如收到msg=H.LAUNCH_ACTIVITY，则调用ActivityThread.handleLaunchActivity()方法，最终会通过反射机制，创建Activity实例，然后再执行Activity.onCreate()等方法；
    再比如收到msg=H.PAUSE_ACTIVITY，则调用ActivityThread.handlePauseActivity()方法，最终会执行Activity.onPause()等方法。
    主线程的消息又是哪来的呢？当然是App进程中的其他线程通过Handler发送给主线程


    system_server进程是系统进程，java framework框架的核心载体，里面运行了大量的系统服务，比如这里提供ApplicationThreadProxy（简称ATP），ActivityManagerService（简称AMS），这个两个服务都运行在system_server进程的不同线程中，由于ATP和AMS都是基于IBinder接口，都是binder线程，binder线程的创建与销毁都是由binder驱动来决定的。

    App进程则是我们常说的应用程序，主线程主要负责Activity/Service等组件的生命周期以及UI相关操作都运行在这个线程； 另外，每个App进程中至少会有两个binder线程 ApplicationThread(简称AT)和ActivityManagerProxy（简称AMP），除了图中画的线程，其中还有很多线程.

    Binder用于不同进程之间通信，由一个进程的Binder客户端向另一个进程的服务端发送事务，；而handler用于同一个进程中不同线程的通信。

    总结：ActivityThread通过ApplicationThread和AMS进行进程间通讯，AMS以进程间通信的方式完成ActivityThread的请求后会回调ApplicationThread中的Binder方法，然后ApplicationThread会向H发送消息，H收到消息后会将ApplicationThread中的逻辑切换到ActivityThread中去执行，即切换到主线程中去执行，这个过程就是。主线程的消息循环模型


    另外，ActivityThread并非是一个线程，它并没有继承Thread。那么mainLooper绑定的是哪个Thread的呢，也可以说它承载环境是什么呢？ 
    答：1.  进程     每个app运行时前首先创建一个进程，该进程是由Zygote fork出来的，用于承载App上运行的各种Activity/Service等组件。进程对于上层应用来说是完全透明的，这也是google有意为之，让App程序都是运行在Android Runtime。大多数情况一个App就运行在一个进程中，除非在AndroidManifest.xml中配置Android:process属性，或通过native代码fork进程。 
    2. 线程    线程对应用来说非常常见，比如每次new Thread().start都会创建一个新的线程。该线程与App所在进程之间资源共享，从Linux角度来说进程与线程除了是否共享资源外，并没有本质的区别，都是一个task_struct结构体，在CPU看来进程或线程无非就是一段可执行的代码，CPU采用CFS调度算法，保证每个task都尽可能公平的享有CPU时间片。

    **因此 其实承载ActivityThread的主线程就是由Zygote fork而创建的进程。**




### Handler 是如何能够线程切换，发送Message的？（线程间通讯）
    线程间是共享资源的。所以Handler处理不同线程问题就只要注意异步情况即可。这里再引申出Handler的一些小知识点。 
    Handler创建的时候会采用当前线程的Looper来构造消息循环系统，Looper在哪个线程创建，就跟哪个线程绑定，并且Handler是在他关联的Looper对应的线程中处理消息的。（敲黑板）

    那么Handler内部如何获取到当前线程的Looper呢—–ThreadLocal。ThreadLocal可以在不同的线程中互不干扰的存储并提供数据，通过ThreadLocal可以轻松获取每个线程的Looper。当然需要注意的是①线程是默认没有Looper的，如果需要使用Handler，就必须为线程创建Looper。我们经常提到的主线程，也叫UI线程，它就是ActivityThread，②ActivityThread被创建时就会初始化Looper，这也是在主线程中默认可以使用Handler的原因。






### android系统为什么不允许子线程访问UI？ 子线程在真的不能直接访问ui吗？子线程有哪些更新UI的方法。
1. 首先android的ui控件不是线程安全的，所以如果允许多线程访问ui控件，会增加ui控件的不可预期性。 如果为ui控件增加了锁机制，则有会以下两个缺点：
    - 加锁机制后，会使得ui访问逻辑变得复杂；
    - 锁机制会降低ui访问的效率，因为锁机制会阻塞某些线程的执行。增加系统风险

    因此最简单且高效的方式就是采用单线程模型来处理ui操作。
2. 当然可以在子线程直接访问，只是在checkThread方法属于ViewRootImpl的成员方法，那么会不会是此时我们的ViewRootImpl根本就没被创建呢？怀着这个出发点，我们再度审视ActivtyThread调度Activity生命周期的各个环节，首先看performLaunchActivity方法中的处理：

    一个Activity对应着一个window，一个decorView和一个ViewRootImpl。 window的创建时机是在performLaunchActivity中，调用了Activity.attach（）创建PhoneWindow实例，在phoneWindow中实例化decorView。而ViewRootImpl是在handleResumeActivity的时候，由makeVisible进入Activity中通过WindowManager addView方法将mDecor放入到ViewRootImpl中。View的绘制流程都是由ViewRootImpl发起的
3. - 主线程中定义Handler，子线程通过mHandler发送消息，主线程Handler的handleMessage更新UI。
    - 用Activity对象的runOnUiThread方法。
    - View.post(Runnable r) 。//实际上也是Handler从中作祟，根据Handler的注释，也可以清楚该Handler可以处理UI事件，也就是说它的Looper也是主线程的sMainLooper。这就是说我们常用的更新UI都是通过Handler实现的。

### 子线程中Toast，showDialog，的方法。（和子线程不能更新UI有关吗）
1. 子线程中Toast报错的原因，因为在TN中使用Handler，所以需要创建Looper对象。Toast本质是通过window显示和绘制的（操作的是window），而主线程不能更新UI 是因为ViewRootImpl的checkThread方法在Activity维护的View树的行为。 
Toast中TN类使用Handler是为了用队列和时间控制排队显示Toast，所以为了防止在创建TN时抛出异常，需要在子线程中使用Looper.prepare();和Looper.loop();（但是不建议这么做，因为它会使线程无法执行结束，导致内存泄露）


