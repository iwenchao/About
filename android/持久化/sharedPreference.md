---
title: Android 数据持久化之 SharedPreferences
date: 
--- 

#### 目录
1. 常见问题
    1. SharedPreferences 是如何初始化的，它会阻塞线程嘛？如果会，是什么原因。而且每次获取 SP 对象真的会很慢吗？
        - 答：SharedPreferences 本身是一个接口，程序无法直接创建 SharedPreferences 实例，只能通过 Context 提供的 getSharedPreferences(String name, int mode) 方法来获取 SharedPreferences 实例，name 表示要存储的 xml 文件名，第二个参数直接写 Context.MODE_PRIVAT，表示该 SharedPreferences 数据只能被本应用读写。当然还有 MODE_WORLD_READABLE 等，但是已经被废弃了，因为 SharedPreference 在多进程下表现并不稳定。
2. 基本使用以及适用范围
    - 适合：保存少量的数据，且这些数据的格式简单，适用保存应用的配置参数，但不建议使用 SP 来存储大规模数据，可能会降低性能
3. 核心原理以及源码分析
    - 基于 XML 文件存储的 key-value 键值对数据，在 ``` /data/data/<package name>/shared_prefs ``` 目录下。
    - SharedPreferences 本身只能获取数据而不支持存储和修改，存储修改是通过 SharedPreferences.Editor 来实现的，它们两个都只是接口，真正的实现在 SharedPreferencesImpl 和 EditorImpl 。
    - ContentImpl.class对象，一个进程只存在一个该对象实例，里面的静态变量ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache保存了同一个进程内的所有 SharedPreferences 都保存在这个静态列表中
    - 可以稍微总结一下，sSharedPrefsCache 会保存加载到内存中的 SharedPreferences 对象，当用户需要获取 SP 对象的时候，首先会在 sSharedPrefsCache 中查找，如果没找到，就创建一个新的 SP 对象添加到 sSharedPrefsCache 中，并且以当前应用的包名为 key。
    - 需要注意的是，ContextImpl 类中并没有定义将 SharedPreferences 对象移除 sSharedPrefsCache 的方法，所以一旦加载到内存中，就会存在直至进程销毁。相对的，也就是说，SP 对象一旦加载到内存，后面任何时间使用，都是从内存中获取，不会再出现读取磁盘的情况。
    - 原来是根据传入的 file 从 ArrayMap<File, SharedPreferencesImpl> 拿到 SharedPreferences（SharedPreferencesImpl） 实例。关键代码其实并不多，但是我还是把所有代码都贴上了，因为这里我们能看到一个兼容性问题以及多进程问题，兼容性问题是指如果在 Android O 及更高版本中，通过传入的 file 拿到的 SharedPreferences 实例为空，说明该文件目录是用户无权限访问的，会直接抛出一个异常。多进程问题是指在 Context.MODE_MULTI_PROCESS 下，可能存在记录丢失的情况。
    - 果然，它是在子线程读取的磁盘文件，所以说 SP 对象初始化过程本身的确不会造成主线程的阻塞。但是真的不会阻塞嘛？这里需要注意，在读取完磁盘文件后，把 mLoaded 置为 true，继续往下看。
    - 从上面代码可知，只有子线程从磁盘加载完数据之后，mLoaded 才会被置为 true，所以说虽然从磁盘读取数据是在子线程中进行并不会阻塞主线程，但是如果文件在读取之前获取某个 SharedPreferences 的值，那么主线程就可能被阻塞住，直到子线程加载完文件为止，所以说保存的 SP 文件不宜太大。
    - EditorImpl 就是 Editor 真正的实现类，在这里面我们能看到我们经常使用的 putXxx 方法
    - 然后就是执行提交操作了，分两种，一种是 commit，一种是 apply，这里我把两个方法放在一块展示，便于查看区别：EditorImpl#commit / apply：
        - commit: 提交修改到内存，然后通过enqueueDiskWrite 将要写入磁盘的任务进行排队，commit 的写磁盘任务就直接在当前线程即 UI 线程里执行
        - apply：提交修改到内存，然后通过enqueueDiskWrite 将要写入磁盘的任务进行排队，然后QueuedWork.queue（runnable）将开启异步线程写入磁盘。这里可能会出现异步产生的文件不一致的问题。如何保证数据的一致性呢？
        - commit/apply数据一致性保证：在这两个方法里都有 mcr.writtenToDiskLatch.await()，它其实是一个 CountDownLatch。CountDownLatch 是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完成后在执行。
    - 总结：
        1. sSharedPrefsCache 是一个 ArrayMap<String,ArrayMap<File,SharedPreferencesImpl>>，它会保存加载到内存中的 SharedPreferences 对象，ContextImpl 类中并没有定义将 SharedPreferences 对象移除 sSharedPrefsCache 的方法，所以一旦加载到内存中，就会存在直至进程销毁。相对的，也就是说，SP 对象一旦加载到内存，后面任何时间使用，都是从内存中获取，不会再出现读取磁盘的情况
        2. SharedPreferences 和 Editor 都只是接口，真正的实现在 SharedPreferencesImpl 和 EditorImpl ，SharedPreferences 只能读数据，它是在内存中进行的，Editor 则负责存数据和修改数据，分为内存操作和磁盘操作
        3. 获取 SP 只能通过 ContextImpl#getSharedPerferences 来获取，它里面首先通过 mSharedPrefsPaths 根据传入的 name 拿到 File ，然后根据 File 从 ArrayMap<File, SharedPreferencesImpl> cache 里取出对应的 SharedPrederenceImpl 实例
        4. SharedPreferencesImpl 实例化的时候会启动子线程来读取磁盘文件。但是在此之前如果通过 SharedPreferencesImpl#getXxx 或者 SharedPreferences.Editor 会阻塞 UI 线程，因为在从 SP 文件中读取数据或者往 SP 文件中写入数据的时候必须等待 SP 文件加载完
        5. 在 EditorImpl 中 putXxx 的时候，是通过 HashMap 来存储数据，提交的时候分为 commit 和 apply，它们都会把修改先提交到内存中，然后在写入磁盘中。
        6. 只不过 apply 是异步写磁盘，而 commit 可能是同步写磁盘也可能是异步写磁盘，在于前面是否还有写磁盘任务。对于 apply 和 commit 的同步，是通过 CountDownLatch 来实现的，它是一个同步工具类，它允许一个线程或多个线程一致等待，直到其他线程的操作执行完之后才执行
        7. SP 的读写操作是线程安全的，它对 mMap 的读写操作用的是同一把锁，考虑到 SP 对象的生命周期与进程一致，一旦加载到内存中就不会再去读取磁盘文件，所以只要保证内存中的状态是一致的，就可以保证读写的一致性
4. 注意事项以及优化建议
    1. 强烈建议不要在 SP 里面存储特别大的 key/value ，有助于减少卡顿 / ANR
    2. 请不要高频的使用 apply，尽可能的批量提交；commit 直接在主线程操作时，更要注意了
    3. 不要使用 MODE_MULTI_PROCESS
    4. 高频写操作的 key 与高频读操作的 key 可以适当的拆分文件，以减少同步锁竞争
    5. 不要连续多次 edit，每次 edit 就是打开一次文件，应该获取一次 edit，然后多次执行 putXxx，减少内存波动，所以在封装方法的时候要注意了

