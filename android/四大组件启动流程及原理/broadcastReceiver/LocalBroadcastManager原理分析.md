1. 概述 
当不需要通过send broadcasts来完成跨应用的通信，那么建议采用LocalBroadcastManager， 将会是更加地高效、安全地方式，并且对系统带来的影响也是更小。

    BroadcastReceiver采用的binder方式实现跨进程间的通信；
    LocalBroadcastManager使用Handler通信机制。