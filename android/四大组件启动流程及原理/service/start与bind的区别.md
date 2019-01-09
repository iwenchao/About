### 关于Service
Android执行Service有两种方法，一种是startService，一种是bindService。下面让我们一起来聊一聊这两种执行Service方法的区别。



1、生命周期上的区别

    执行startService时，Service会经历onCreate->onStartCommand。当执行stopService时，直接调用onDestroy方法。调用者如果没有stopService，Service会一直在后台运行，下次调用者再起来仍然可以stopService。

    执行bindService时，Service会经历onCreate->onBind。这个时候调用者和Service绑定在一起。调用者调用unbindService方法或者调用者Context不存在了（如Activity被finish了），Service就会调用onUnbind->onDestroy。这里所谓的绑定在一起就是说两者共存亡了。

    多次调用startService，该Service只能被创建一次，即该Service的onCreate方法只会被调用一次。但是每次调用startService，onStartCommand方法都会被调用。Service的onStart方法在API 5时被废弃，替代它的是onStartCommand方法。

    第一次执行bindService时，onCreate和onBind方法会被调用，但是多次执行bindService时，onCreate和onBind方法并不会被多次调用，即并不会多次创建服务和绑定服务。onServiceConnected会执行多次。并且我们注意到onServiceConnected方法的第二个参数也是IBinder类型的，不难猜测onBind()方法返回的对象被传递到了这里。也就是说我们可以在onServiceConnected方法里拿到了MyService服务的内部类MyBinder的对象，通过这个内部类对象，只要强转一下，我们可以调用这个内部类的非私有成员对象和方法。调用unbindService结束服务，生命周期执行onDestroy方法，并且unbindService方法只能调用一次，多次调用应用会抛出异常。使用时也要注意调用unbindService一定要确保服务已经开启，否则应用会抛出异常。

    接下来我们说一下startService和bindService开启服务时，他们与activity之间的关系。
    1、startService开启服务以后，与activity就没有关联，不受影响，独立运行。
    2、bindService开启服务以后，与activity存在关联，退出activity时必须调用unbindService方法，否则会报ServiceConnection泄漏的错误。

    (一) startService生命周期 
    1. onCreate() –> onStartCommand –> onDestroy()。 
    2. startService 在多个context里多次调用，只有第一次会调用service的onCreate()，所有的context调用startService都会重复调用service的onStartCommand(Intent intent, int flags, int startId)，并且每次startId数字逐次递增，并且的调用的service是同一个对象实例(即第一次创建的那个)。 
    3. 一旦利用startService启动service，那么不管调用者context的生命周期是否结束(例如finish)，service依然在后台运行，直至调用stopService，这是会回调service的onDestroy()，service的生命周期结束。 
    4. stopService(intent)可以多次调用且不报错，但是只有第一次能够起作用，调用到service的onDestroy()。

    (二) bindService生命周期 
    1. onCreate() –> onBind() [ Activity–> onServiceConnected]–> onUnbind()。bindService方法在onCreate之前会调用service的构造函数，但是startService方法不会。 
    2. bindService在多个context里多次调用，只有第一次调用会调用service里的onCreate()和onBind()。后续的都不会触发这2个方法。 
    3. 利用bindService启动service之后，调用者如果生命周期结束(比如finish)，那么会自动调用service的onDestroy()，但是service必须要所有bind/start过它的全部unbind/stop了才会结束生命周期。要注意service生命周期结束之前一定要记得调用unbindService(conn)，否则会出现 leaked ServiceConnection错误。 
    4. unbindService(conn)只能调用一次，否则程序会崩溃。 
    5. onServiceDisconnected只会在service意外退出时调用，记得在这里回收一些资源。

    二. 控制service 
    另外，我认为利用startService完全可以在任何Activity里来控制被启动过的service来做不同的事情，而不是网上说的完全失去了控制，因为我们完全可以利用intent带过去不同的值，在service里的onStartCommand里来判断intent里带来的值，从而达到控制的效果，经过测试时完全可以的，因为不同的activity之间利用startService启动的service是同一个对象实例。

    三. 使用场景 
    一般说来，bindService使用的场景比较多。另外，混合使用的场景下，只有当所有的调用者释放掉一个service的bind引用(即unbindService)，这个时候再用stopService(或者先stopService再释放所有的bind引用)，这个service才会结束生命周期。

2、调用者如何获取绑定后的Service的方法

    onBind回调方法将返回给客户端一个IBinder接口实例，IBinder允许客户端回调服务的方法，比如得到Service运行的状态或其他操作。我们需要IBinder对象返回具体的Service对象才能操作，所以说具体的Service对象必须首先实现Binder对象。看一下android对于onBind方法的返回类型IBinder的介绍，字面上理解是IBinder是android提供的进程间和跨进程调用机制的接口。而且返回的对象不要直接实现这个接口，应该继承Binder这个类。

3、既使用startService又使用bindService的情况

    如果一个Service又被启动又被绑定，则该Service会一直在后台运行。首先不管如何调用，onCreate始终只会调用一次。对应startService调用多少次，Service的onStart方法便会调用多少次。Service的终止，需要unbindService和stopService同时调用才行。不管startService与bindService的调用顺序，如果先调用unbindService，此时服务不会自动终止，再调用stopService之后，服务才会终止；如果先调用stopService，此时服务也不会终止，而再调用unbindService或者之前调用bindService的Context不存在了（如Activity被finish的时候）之后，服务才会自动停止。

    那么，什么情况下既使用startService，又使用bindService呢？

    如果你只是想要启动一个后台服务长期进行某项任务，那么使用startService便可以了。如果你还想要与正在运行的Service取得联系，那么有两种方法：一种是使用broadcast，另一种是使用bindService。前者的缺点是如果交流较为频繁，容易造成性能上的问题，而后者则没有这些问题。因此，这种情况就需要startService和bindService一起使用了。

    另外，如果你的服务只是公开一个远程接口，供连接上的客户端（Android的Service是C/S架构）远程调用执行方法，这个时候你可以不让服务一开始就运行，而只是bindService，这样在第一次bindService的时候才会创建服务的实例运行它，这会节约很多系统资源，特别是如果你的服务是远程服务，那么效果会越明显（当然在Servcie创建的是偶会花去一定时间，这点需要注意）。    

4、本地服务与远程服务

    本地服务依附在主进程上，在一定程度上节约了资源。本地服务因为是在同一进程，因此不需要IPC，也不需要AIDL。相应bindService会方便很多。缺点是主进程被kill后，服务变会终止。

    远程服务是独立的进程，对应进程名格式为所在包名加上你指定的android:process字符串。由于是独立的进程，因此在Activity所在进程被kill的是偶，该服务依然在运行。缺点是该服务是独立的进程，会占用一定资源，并且使用AIDL进行IPC稍微麻烦一点。