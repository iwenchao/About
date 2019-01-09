### Service与IntentService的区别
#### 简介

因为最大部分的service不需要同时处理多个请求(处理多个请求是一个比较危险的多线程的场景),这样在在这种情况下呢，最好使用IntentService类如果你实现你的服务。

#### Service【略】

#### IntentService
IntentService与Service应该有两处不同，一是，IntentService是序列化执行Intent的，也就是，如果向IntentService发送多个Intent，IntentService是不会阻塞的；二是，IntentService执行完Intent后，是会自动关闭的。而Service没有这两个特点。

IntentService是结合了HandlerThread+Looper+Msg来增加Service的异步能力

1. 直接创建一个默认的工作线程HandlerThread,该线程执行所有的intent传递给onStartCommand()，然后由ServiceHandler转发消息handleMessage,执行onHandleIntent
2. 直接创建一个工作队列,将一个意图传递给你onHandleIntent()的实现,所以我们就永远不必担心多线程。
3. 当请求完成后自己会调用stopSelf()，所以你就不用调用该方法了。
4. 提供的默认实现onBind()返回null，所以也不需要重写这个方法。
5. 提供了一个默认实现onStartCommand(),将意图工作队列,然后发送到你onHandleIntent()实现。
6. 我们需要做的就是实现onHandlerIntent()方法，还有一点就是经常被遗忘的，构造函数是必需的,而且必须调用超IntentService(字符串) ，因为工作线程的构造函数必须使用一个名称



#### 补充一点：关于android中多线程的处理
1. 继承Thread
2. 实现Runnable接口
3. AsyncTask
4. Handler
5. HandlerThread
6. IntentService
