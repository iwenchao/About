[关于HashMap的问答](https://juejin.im/post/5c1da988f265da6143130ccc)
[为什么String会采用31作为哈希因子](https://zhanjia.iteye.com/blog/2426892)
[为什么String采用16作为初始容量](https://blog.csdn.net/qq_35583089/article/details/80048285)

#### 1. HashMap源码分析
- hash表，
    - hash函数：
    - hash冲突：解决冲突就是通过链表
- 称为哈希数组，数组的每个元素都是一个单链表的头节点，链表是用来解决冲突的，如果不同的key映射到了数组的同一位置处，就将其放入单链表中。

- 首先HashMap里面实现一个静态内部类Entry，其重要的属性有 key , value, next，从属性key,value我们就能很明显的看出来Entry就是HashMap键值对实现的一个基础bean，我们上面说到HashMap的基础就是一个线性数组，这个数组就是Entry[]，Map里面的内容都保存在Entry[]里面。其中next也是一个Entry对象，它就是用来处理hash冲突的，形成一个链表。

- 加载因子：loadFactor加载因子是表示Hsah表中元素的填满的程度.
    - 若:加载因子越大,填满的元素越多,好处是,空间利用率高了,但:冲突的机会加大了.链表长度会越来越长,查找效率降低。
- 默认初始容量为16，默认加载因子为0.75。
- 确保容量为2的n次幂，使capacity为大于initialCapacity的最小的2的n次幂，
    - **至于为什么要把容量设置为2的n次幂：首先，length为2的整数次幂的话，h&(length-1)就相当于对length取模，这样便保证了散列的均匀，同时也提升了效率；其次，length为2的整数次幂的话，为偶数，这样length-1为奇数，奇数的最后一位是1，这样便保证了h&(length-1)的最后一位可能为0，也可能为1（这取决于h的值），即与后的结果可能为偶数，也可能为奇数，这样便可以保证散列的均匀性，而如果length为奇数的话，很明显length-1为偶数，它的最后一位是0，这样h&(length-1)的最后一位肯定为0，即只能为偶数，这样任何hash值都只会被散列到数组的偶数下标位置上，这便浪费了近一半的空间，因此，length取2的整数次幂，是为了使不同hash值发生碰撞的概率较小，这样就能使元素在哈希表中均匀地散列。**
- 计算hash值的方法 通过键的hashCode来计算，位运算
- 根据hash值和数组长度length算出索引值
    - h & (length-1);//这里不能随便算取，用hash&(length-1)是有原因的，这样可以确保算出来的索引是在数组大小范围内，不会超出
    - 这个我们要重点说下，我们一般对哈希表的散列很自然地会想到用hash值对length取模（即除法散列法），Hashtable中也是这样实现的，这种方法基本能保证元素在哈希表中散列的比较均匀，但取模会用到除法运算，效率很低，HashMap中则通过h&(length-1)的方法来代替取模，同样实现了均匀的散列，但效率要高很多，这也是HashMap对Hashtable的一个改进。

- 当HashMap中的元素越来越多的时候，hash冲突的几率也就越来越高，因为数组的长度是固定的。所以为了提高查询的效率，就要对HashMap的数组进行扩容，
    - 而在HashMap数组扩容之后，最消耗性能的点就出现了：原数组中的数据必须重新计算其在新数组中的位置，并放进去，这就是resize。
    - 扩容是需要进行数组复制的，复制数组是非常消耗性能的操作，



#### 2. ConcurrentHashMap源码分析
- Collections.synchronizedXxx()同步容器等相比，util.concurrent中引入的并发容器主要解决了两个问题： 
1）根据具体场景进行设计，尽量避免synchronized，提供并发性。 
2）定义了一些并发安全的复合操作，并且保证并发环境下的迭代操作不会出错。

- util.concurrent中容器在迭代时，可以不封装在synchronized中，可以保证不抛异常，但是未必每次看到的都是"最新的、当前的"数据。

- ConcurrentHashMap代替同步的Map（Collections.synchronized（new HashMap()）），众所周知，HashMap是根据散列值分段存储的，同步Map在同步的时候锁住了所有的段，而ConcurrentHashMap加锁的时候根据散列值锁住了散列值锁对应的那段，因此提高了并发性能。ConcurrentHashMap也增加了对常用复合操作的支持，

- 　CopyOnWriteArrayList和CopyOnWriteArraySet分别代替List和Set，主要是在遍历操作为主的情况下来代替同步的List和同步的Set，这也就是上面所述的思路：迭代过程要保证不出错，除了加锁，另外一种方法就是"克隆"容器对象。


- ConcurrentHashMap为了提高本身的并发能力，在内部采用了一个叫做**Segment的结构**，一个Segment其实就是一个**类Hash Table的结构**，Segment内部维护了一个**链表数组**
    - 因此我们可以了解到，ConcurrentHashMap定位一个元素的过程需要进行两次Hash操作，第一次Hash定位到Segment，第二次Hash定位到元素所在的链表的头部，
    - 但是带来的好处是写操作的时候可以只对元素所在的Segment进行加锁即可，不会影响到其他的Segment，这样，在最理想的情况下，ConcurrentHashMap可以最高同时支持Segment数量大小的写操作，并发能力可以大大的提高。
    - Segment的数量永远是2的指数个，这样的好处是方便采用移位操作来进行hash，加快hash的过程

- 可以看到HashEntry的一个特点，除了value以外，其他的几个变量key，hash，next都是final的，这样做是为了防止链表结构被破坏，出现ConcurrentModification的情况。

- 后续如果ConcurrentHashMap的元素数量增加导致ConrruentHashMap需要扩容，ConcurrentHashMap不会增加Segment的数量，而只会增加Segment中链表数组的容量大小，这样的好处是扩容过程不需要对整个ConcurrentHashMap做rehash，而只需要对Segment里面的元素做一次rehash就可以了


#### SpareArray原理。与HashMap的区别
- SparseArray从名字上看就能猜到跟数组有关系，事实上他底层是两条数组，一组存放key，一组存放value
    - 使用int[]数组存放key，避免了HashMap中基本数据类型需要装箱的步骤，其次不使用额外的结构体（Entry)，单个元素的存储成本下降。
    - 另外，SparseArray为了提升性能，在删除操作时做了一些优化： 
    当删除一个元素时，并不是立即从value数组中删除它，并压缩数组， 
    而是将其在value数组中标记为已删除。这样当存储相同的key的value时，可以重用这个空间。
    如果该空间没有被重用，随后将在合适的时机里执行gc（垃圾收集）操作，将数组压缩，以免浪费空间。
    - put(int key, E value) :
        1. 利用二分查找，找到 待插入key 的 下标index
        2.  //如果返回的index是正数，说明之前这个key存在，直接覆盖value即可
        3. //若返回的index是负数，说明 key不存在.
            1. //先对返回的i取反，得到应该插入的位置i
            2. //如果i没有越界，且对应位置是已删除的标记，则复用这个空间.//赋值后，返回
            3. //如果需要GC，且需要扩容
                1. //先触发GC
                2. //gc后，下标i可能发生变化，所以再次用二分查找找到应该插入的位置i
                    1. //插入key（可能需要扩容）
                    2. //插入value（可能需要扩容）
        4. 扩容操作依然是用数组的复制、覆盖完成。



- 优点：
    1. 避免了基本数据类型的装箱操作
    2. 不需要额外的结构体，单个元素的存储成本更低
    3. 数据量小的情况下，随机访问的效率更高
    4. 扩容时只需要数组拷贝工作，不需要重建哈希表。
- 缺点：
    1. 插入操作需要复制数组，增删效率降低
    2. 数据量巨大时，复制数组成本巨大，gc()成本也巨大
    3. 数据量巨大时，查询效率也会明显下降
    4. 不适合大容量的数据存储。存储大量数据时，它的性能将退化至少50%。