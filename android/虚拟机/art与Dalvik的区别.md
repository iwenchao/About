1. android虚拟机发展史
     1. 诞生Dalvik
        - Dalvik 担任虚拟机的角色，每次运行程序的时候，Dalvik 负责加载 dex/odex 文件并解析成机器码交由系统调用
     2. android 2.2 引入 JIT（即时编译）
        - JIT 编译器可以对热点代码进行编译优化，将 dex/odex 中的 Dalvik Code ( Smali 指令集 ) 翻译成相当精简的 Native Code 去执行，JIT 的引入使得 Dalvik 的性能提升了 3~6 倍。
     3. android 4.4 引入ART和AOT 此时Dalvik与ART共存
     4. android 5.0 ART全面取代Dalvik
     5. android 7.0 JIT回归
        - 形成了 AOT / JIT 混合编译模式。该混合模式综合了 AOT 和 JIT 的各种优点，使得应用在安全速度加快的同时，运行速度、存储空间和耗电量等指标都得到了优化。
2. AOT与JIT的区别
    - AOT：
        - 优点：
            1. 安装时预编译优化，使得应用启动更快，运行更加流畅
            2. 耗电问题得以优化
        - 缺点
            1. 应用安装后的应用优化比较耗时
            2. 优化后的文件会占用额外的存储空间
    - JIT：
        - 优点：
            1. 热点代码得到编译优化
        - 缺点：
            1. 每次应用启动都需要重新编译
            2. 运行时更加耗电
3. ART与Dalvik的区别
    1. AOT与JIT的区别
    2. ART在垃圾回收上更优于Dalvik