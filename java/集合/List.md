#### 1. ArrayList源码解读

- ArrayList是基于数组实现的，是一个动态数组，其容量能自动增长，类似于C语言中的动态申请内存，动态增长内存。
- ArrayList不是线程安全的，只能用在单线程环境下，多线程环境下可以考虑用Collections.synchronizedList(List l)函数返回一个线程安全的ArrayList类，也可以使用concurrent并发包下的CopyOnWriteArrayList类。
- ArrayList容量自动增长会带来数据向新数组的重新拷贝。注意，此实现不是同步的。如果多个线程同时访问一个ArrayList实例，而其中至少一个线程从结构上修改了列表，那么它必须保持外部同步。

- ensureCapacity(int minCapacity)方法中：Object oldData[] = elementData;//为什么要用到oldData[]
乍一看来后面并没有用到关于oldData， 这句话显得多此一举！但是这是一个牵涉到内存管理的类， 所以要了解内部的问题。 而且为什么这一句还在if的内部，这跟elementData = Arrays.copyOf(elementData, newCapacity); 这句是有关系的，下面这句Arrays.copyOf的实现时新创建了newCapacity大小的内存，然后把老的elementData放入。好像也没有用到oldData，有什么问题呢。问题就在于旧的内存的引用是elementData， elementData指向了新的内存块，如果有一个局部变量oldData变量引用旧的内存块的话，在copy的过程中就会比较安全，因为这样证明这块老的内存依然有引用，分配内存的时候就不会被侵占掉，然后copy完成后这个局部变量的生命期也过去了，然后释放才是安全的。不然在copy的的时候万一新的内存或其他线程的分配内存侵占了这块老的内存，而copy还没有结束，这将是个严重的事情。


#### 2. LinkedList源码分析

- LinkedList 是一个继承于AbstractSequentialList的双向链表。它也可以被当作堆栈、队列或双端队列进行操作。
- LinkedList 实现 List 接口，能对它进行队列操作。
- LinkedList 实现 Deque 接口，即能将LinkedList当作双端队列使用。

- 双向链表，那么必定存在一种数据结构——我们可以称之为节点 E element;
 3     Entry<E> next;
 4     Entry<E> previous;


#### CopyOnWriteArrayList的优与缺点
- ArrayList的线程安全变体，其中增删改都是通过对底层数组的新拷贝实现的。这通常开销太大，但是当遍历操作远远超过修改、插入时，可能比其他方法更有效，当您不能或不想同步遍历时，这是很有用的，但是需要排除并发线程之间的干扰。
- “快照”风格的迭代器方法在创建迭代器的时候使用一个指向数组状态的引用。这个数组迭代器的生命周期期间从未改变,所以干涉是不可能的,迭代器是可以保证不抛出ConcurrentModificationException异常。迭代器不会在创建迭代器时反映添加、删除或更改。不支持对迭代器本身进行元素更改操作(删除、设置和添加)。这些方法抛出UnsupportedOperationException。
- 内存一致性影响 : 与其他并发集合一样，会发生对象被放置到CopyOnWriteArrayList之前的线程中，然后在另一个线程中从CopyOnWriteArrayList中访问或删除该元素之前的操作（线程修改时，其他线程并不能访问到最新的数据）
- 总结以上说明：要对原容器A做增删改，就先拷贝一份为B，在B中做增、删、改。此时其他线程读的是A的数据。修改完成之后把指向A的引用改变指向到B，这样完成操作。

- transient volatile Object[] elements;// 保证对象的可见性
- add():可知各个操作之间通过并发锁来保证 【写】的一致性，先复制一份，然后添加到最后，然后修改指向。


- 应用场景：遍历操作远多于修改、内存开销可忽略
- 优点：
    1. 保证多线程的并发读写的线程安全
- 缺点：
    1. 内存:有数组拷贝自然有内存问题。如果实际应用数据比较多，而且比较大的情况下，占用内存会比较大，这个可以用ConcurrentHashMap来代替。
    2. 数据一致性:CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器
        - 不能保证实时一致性的原因：针对iterator使用了一个叫 COWIterator的阉割版迭代器，因为不支持写操作，当获取CopyOnWriteArrayList的迭代器时，是将迭代器里的数据引用指向当前 引用指向的数据对象，无论未来发生什么写操作，都不会再更改迭代器里的数据对象引用，所以迭代器也很安全。

