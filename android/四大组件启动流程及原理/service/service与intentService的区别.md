### Service与IntentService的区别
#### 简介

因为最大部分的service不需要同时处理多个请求(处理多个请求是一个比较危险的多线程的场景),这样在在这种情况下呢，最好使用IntentService类如果你实现你的服务。

#### Service【略】



#### 补充一点：关于android中多线程的处理
1. 继承Thread
2. 实现Runnable接口
3. AsyncTask
4. Handler
5. HandlerThread
6. IntentService
