### android 统计启动时长，标准
- 冷启动，application没有被创建，需要先创建进程，然后启动MainActivity。由于这个过程需要fork一个新进程，所以耗时。
- 热启动，同上面对照，已经启动过application，并驻留在系统内存内，只是需要唤醒该进程，并启动MainActivity。


### 统计方式
1. adb shell am start -w pageage/activityname  通过这条命令启动，可以获得启动时间。
2. 线上版本统计
    - startTime：applicaiton 创建，可以从attachBaseContext()开始，得到startTime。
    - endTime: MainActivity的第一个可视画面，onResume其实还没有看到画面，最合适的回调是onWindowFocusChanged，也就是获得焦点。

- 这里有2个关键点，activity的启动流程 & applicaiton到activity的生命周期。

### activity启动流程：
1. 通过Launcher点击应用图标，startActivity
2. 通过binder机制，将调用到Ams，AMS所在的SystemServer进程由binder向Zygote进程发送消息 fork进程。然后AMS通过ActivityStackSuperviosor启动对应package的home activity。
3. 进程创建后，进行初始化ActivityThread，然后attach(system=true)，创建Application上下文，并调用oncreate；应用程序启动
4. 应用程序启动后，创建Looper，然后attach(system=false)，将ApplicationThread对象在AMS注册。启动第一个Activity；开始实例化Activity进行attach
5. AMS保存进程的代理对象，然后AMS通过该进程，创建activity的实例& 执行各生命周期。
