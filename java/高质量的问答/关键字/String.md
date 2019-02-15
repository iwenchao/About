---
title: String 类
---


1. 类继承
```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
}

String 类由 final 修饰，是一个不可变类。
```
2. 类成员变量
```
// JDK 10
private final byte[] value;
private final byte coder;	//采取的编码方式
static final boolean COMPACT_STRINGS;

static {
    COMPACT_STRINGS = true;		//压缩 String 存储空间
}
@Native static final byte LATIN1 = 0;
@Native static final byte UTF16  = 1;

```
在 JDK 9 之前，用的是 char 数组来存储 String 的值，之后就用 byte 数组来存储，char 是两个 byte，比如在存储 ‘A’ 这个字符串时只需一个 byte，就会造成空间浪费。

String 支持多种编码，但是如果不指定编码的话，它可能使用两种编码方式，分别是 LATIN1 和 UTF16，LATIN1 其实就是 ISO 编码，属于单字节编码，而 UTF16 为双字节编码。

String 在表示因为字符或者数字时，会可能存在浪费空间的情况，比如在存储 what 字符串时：

在 java 9 之后就变成了：

可以看到，压缩之后存储更加紧凑了。默认是开启压缩的，即 COMPACT_STRINGS 默认为 true。

3. public native String intern();
    - 对象内存分配
        - String str = "Omooo":这样创建字符串对象，首先会去常量池中找有没有这个字符串，如果有就直接指向，没有就先往常量池中添加再指向。即 栈内变量指针直接指向常量池地址
        - String str = new String("Omooo");首先在堆上创建该字符串对象，然后去看常量池中是否有该字符串，如果有就算了，没有就往常量池中添加一个。即 栈内变量指针指向堆内存的对象地址
    - ```
        String str1 = new String("str")+new String("01");
        str1.intern();
        String str2 = "str01";
        System.out.println(str2==str1);

        输出 true。

        分析：首先new String("str")会在堆中创建str，同时添加到常量池；new String("01")也是一样的，在堆中创建01，同时添加到常量池；然后两者拼接，底层用的append方法，在堆中生成一个str01；然后str1.intern()，就把str01拷贝到常量池了；此时运行到String str2 = "str01"，发现常量池中有了，所以直接指向常量池中的str01。最终str1指向堆中的str01对象，str2指向常量池的str01对象，所以结果是false。


        在 JDK 1.7 之后，intern 方法做了些改变，进行拷贝的时候不是拷贝对象，而是拷贝地址值。

        //字符串的拼接
        String str = "hello" + "world";//对于这种加号两边都是常量的，在编译阶段就会自动拼接，所以就会去常量池找"helloworld"，有就直接指向它，没有就在常量池创建再指向
        //有final的拼接：
            final String str1 = "hello";
            final String str2 = "world";
            String str3 = str1 + str2;
            //因为final修饰的变量就是常量，所以在编译期直接会变成
            String str3 = "hello" + "world";
            //再根据常量拼接规则可知最终就变成
            String str3 = "helloworld";
        //变量和常量拼接：
            变量和常量拼接的时候，底层会调用StringBuilder的append方法生成新对象。
            //情况一：
                String str1 = "hello";
                String str2 = str1 + "world";
                str1显然是在常量池中的，world也是在常量池中的，然后调用append方法在堆中生成新对象"helloworld"，str2就指向堆中的"helloworld"对象。所以这两条语句总共生成了3个对象，常量池中有"hello"和"world"，堆中有"helloword"。
            //情况二：
                String str1 = new String("hello");
                String str2 = str1 + "world";
                首先会在堆中创建一个"hello"，再把"hello"添加到常量池；然后会把"world"添加到常量池，拼接的时候，会在堆中创建一个"helloworld"。所以这两条语句总共创建了4个对象，堆中的"hello"、"helloworld"和常量池中的"hello"、"world"。
        //变量和变量拼接：
            变量和变量拼接，底层也会调用StringBuilder的append方法生成新对象。
            //情况一：
            String str1 = "hello";
            String str2 = "world";
            String str3 = str1 + str2;
            这段代码，首先会有一个"hello"在常量池中，然后有个"world"在常量池，第三行代码会调用append方法，在堆中生成一个"helloworld"。所以总共有3个对象。
            //情况二：
            String str1 = "hello";
            String str2 = new String("world");
            String str3 = str1 + str2;
            这段代码，首先在常量池中搞一个"hello"，然后在堆中new一个"world"，同时把"world"也搞到常量池中去，第三步拼接就会在堆中生成一个"helloworld"。所以总共有4个对象。
            //情况三：
            String str1 = new String("hello");
            String str2 = new String("world");
            String str3 = str1 + str2;
            第一行代码创建了两个对象，堆中一个常量池一个，第二行代码也是一样，第三行代码就在堆中创建了一个"helloworld"。所以总共创建了5个对象。



    ```


#### String、StringBuilder和StringBuffer：
