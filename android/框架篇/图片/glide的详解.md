#### glide的深入分析
本文旨在对glide进行源码探索，实现原理以及涉及模式进行剖析，所以对于基础的使用可以参考[官方文档](https://muyangmin.github.io/glide-docs-cn/)

[glide3基本使用](https://blog.csdn.net/guolin_blog/article/details/53759439)
[glide3源码解读glide的执行流程](https://blog.csdn.net/guolin_blog/article/details/53939176)
[glide3缓存机制分析](https://blog.csdn.net/guolin_blog/article/details/54895665)
[glide3的回调与监听](https://blog.csdn.net/guolin_blog/article/details/70215985)
[glid3e的图片变换](https://blog.csdn.net/guolin_blog/article/details/71524668)
[glide3的自定义模块功能](https://blog.csdn.net/guolin_blog/article/details/78179422)
[带进度的glide3图片加载](https://blog.csdn.net/guolin_blog/article/details/78357251)
[glide4的用法](https://blog.csdn.net/guolin_blog/article/details/78582548)

#### glide提供的功能
1. with(context)；和applicationContext，activity，Fragment，view等上下文的声明周期绑定，可以实现在生命周期针对性的加载以及优化处理
    - 实现细节：
        - application类：ApplicationLifecycle
        - 非application类：activity中添加一个fragment，由fragment进行attach到activity，然后监听fragment的生命周期状态就可以捕捉到activity的生命周期
        - 每一种context对应着一个单例requestManager
2. load()：提供多种方式加载资源：网络url，file，assets，drawabe，bitmap，byte[]，stream等
3. into()：
#### 三步走：先with()，再load()，最后into()。
```
Glide.with(this).load(url).into(imageView);
```

###### with()
###### with()
###### with()


#### glide的缓存分析
1. 内存缓存；
    - 防止应用重复将相同图片数据读取到内存中

2. 磁盘缓存
    - 防止应用重复从网络或其他地方重复下载和读取数据。

###### 缓存key
Glide的缓存Key生成规则非常繁琐，决定缓存Key的参数竟然有10个之多。不过繁琐归繁琐，至少逻辑还是比较简单的，我们先来看一下Glide缓存Key的生成逻辑EngineKey对象。（url,signature，width，height，其他编解码，变换，过度等）




