### ThreadLocal 原理
1. ThreadLocal是怎么实现了多个线程之间每个线程一个变量副本的？它是如何实现共享变量的。
    - ThreadLocal提供了set和get访问器用来访问与当前线程相关联的线程局部变量。
    - ThreadLocal.get：根据当前线程t，来获取t.threadLocalMap对象，里面存放了当前线程的私有对象；如果t.threadLocalMap为null，则进行初始化并赋值
- ThreadLocalMap：类HashMap，数组，entry非链表，并且继承弱引用。ThreadLocalMap 是使用 ThreadLocal 的弱引用作为 Key 的，弱引用的对象在 GC 时会被回收。
- 变量是保存在线程中的，而不是保存在ThreadLocal变量中；
- ThreadLocal整体上给我的感觉就是，一个包装类。声明了这个类的对象之后，每个线程的数据其实还是在自己线程内部通过threadLocals引用到的自己的数据。只是通过ThreadLocal访问这个数据而已


### ThreadLocal为什么会内存泄漏
ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用来引用它，那么系统 GC 的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value永远无法回收，造成内存泄漏。

其实，ThreadLocalMap的设计中已经考虑到这种情况，也加上了一些防护措施：在ThreadLocal的get(),set(),remove()的时候都会清除线程ThreadLocalMap里所有key为null的value。

```
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
```

###### 为什么使用弱引用而不是强引用？
官方：为了应对非常大和长时间的用途，哈希表使用弱引用的 key。
