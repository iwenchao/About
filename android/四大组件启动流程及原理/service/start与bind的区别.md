### 关于Service
Android执行Service有两种方法，一种是startService，一种是bindService。下面让我们一起来聊一聊这两种执行Service方法的区别。


1. startService生命周期 

    - 执行startService时，Service会经历onCreate->onStartCommand。当执行stopService时，直接调用onDestroy方法。

    - 一旦利用startService启动service，那么不管调用者context的生命周期是否结束(例如finish)，service依然在后台运行，直至调用stopService，这是会回调service的onDestroy()，service的生命周期结束。

    - startService 在多个context里多次调用，只有第一次会调用service的onCreate()，所有的context调用startService都会重复调用service的onStartCommand(Intent intent, int flags, int startId)，并且每次startId数字逐次递增，并且的调用的service是同一个对象实例(即第一次创建的那个)。不同的activity之间利用startService启动的service是同一个对象实例。

    - stopService(intent)可以多次调用且不报错，但是只有第一次能够起作用，调用到service的onDestroy

2. bindService生命周期 

    - 执行bindService时，Service会经历onCreate->onBind。这个时候调用者和Service绑定在一起。调用者调用unbindService方法或者调用者Context不存在了（如Activity被finish了），Service就会调用onUnbind->onDestroy。这里所谓的绑定在一起就是说两者共存亡了。

    - 第一次执行bindService时，onCreate和onBind方法会被调用，但是多次执行bindService时，onCreate和onBind方法并不会被多次调用，即并不会多次创建服务和绑定服务。onServiceConnected会执行多次。

    - 并且我们注意到onServiceConnected方法的第二个参数也是IBinder类型的，不难猜测onBind()方法返回的对象被传递到了这里。也就是说我们可以在onServiceConnected方法里拿到了MyService服务的内部类MyBinder的对象，通过这个内部类对象，只要强转一下，我们可以调用这个内部类的非私有成员对象和方法。 onBind回调方法将返回给客户端一个IBinder接口实例，IBinder允许客户端回调服务的方法，比如得到Service运行的状态或其他操作。我们需要IBinder对象返回具体的Service对象才能操作，所以说具体的Service对象必须首先实现Binder对象。看一下android对于onBind方法的返回类型IBinder的介绍，字面上理解是IBinder是android提供的进程间和跨进程调用机制的接口。而且返回的对象不要直接实现这个接口，应该继承Binder这个类。

    - 调用unbindService结束服务，生命周期执行onDestroy方法，并且unbindService方法只能调用一次，多次调用应用会抛出异常。使用时也要注意调用unbindService一定要确保服务已经开启，否则应用会抛出异常。

    - 一般说来，bindService使用的场景比较多。另外，混合使用的场景下，只有当所有的调用者释放掉一个service的bind引用(即unbindService)，这个时候再用stopService(或者先stopService再释放所有的bind引用)，这个service才会结束生命周期。

3. 与activity之间的关系

    - startService开启服务以后，与activity就没有关联，不受影响，独立运行。

    - bindService开启服务以后，与activity存在关联，退出activity时必须调用unbindService方法，否则会报ServiceConnection泄漏的错误。


4. 既使用startService又使用bindService的情况

    - 如果一个Service又被启动又被绑定，则该Service会一直在后台运行。首先不管如何调用，onCreate始终只会调用一次。对应startService调用多少次，Service的onStart方法便会调用多少次。Service的终止，需要unbindService和stopService同时调用才行。不管startService与bindService的调用顺序，如果先调用unbindService，此时服务不会自动终止，再调用stopService之后，服务才会终止；如果先调用stopService，此时服务也不会终止，而再调用unbindService或者之前调用bindService的Context不存在了（如Activity被finish的时候）之后，服务才会自动停止。

    - 那么，什么情况下既使用startService，又使用bindService呢？
    - 如果你只是想要启动一个后台服务长期进行某项任务，那么使用startService便可以了。如果你还想要与正在运行的Service取得联系，那么有两种方法：**一种是使用broadcast**，**另一种是使用bindService**。前者的缺点是如果交流较为频繁，容易造成性能上的问题，而后者则没有这些问题。因此，这种情况就需要startService和bindService一起使用了。

    - 另外，如果你的服务只是公开一个远程接口，供连接上的客户端（Android的Service是C/S架构）远程调用执行方法，这个时候你可以不让服务一开始就运行，而只是bindService，这样在第一次bindService的时候才会创建服务的实例运行它，这会节约很多系统资源，特别是如果你的服务是远程服务，那么效果会越明显（当然在Servcie创建的是偶会花去一定时间，这点需要注意）。    

7. 本地服务与远程服务
    - 本地服务依附在主进程上，在一定程度上节约了资源。本地服务因为是在同一进程，因此不需要IPC，也不需要AIDL。相应bindService会方便很多。缺点是主进程被kill后，服务变会终止。

    - 远程服务是独立的进程，对应进程名格式为所在包名加上你指定的android:process字符串。由于是独立的进程，因此在Activity所在进程被kill的是偶，该服务依然在运行。缺点是该服务是独立的进程，会占用一定资源，并且使用AIDL进行IPC稍微麻烦一点。

    - Activity和Service（通过start方式启动）运行在两个不同的进程当中时，如果再次bindService的话，程序崩溃。 此时需要通过AIDL来进行跨进程通信IPC
    
8. 如何使用AIDL
    - 新建一个AIDL文件，该文件需要定义好Activity与Service通信的接口
    - 在自定义Service中，实现定义好的AIDL.Stub接口，然后通过onBind方法返回aidl接口实例对象
    - 至此已经实现了Activity与Service跨进程通信了，而与远程服务通信的更关注的是为了让一个应用程序去访问另一个应用程序中的Service，以实现共享Service的功能。
    - android对于跨进程通信的传递数据格式支持有限，基本上只支持java的基本类型，字符串，list和map等。如果要传递自定义类的话，需要这个类去实现Parcelable接口，并且这个类也要定义一个同名的AIDL文件。