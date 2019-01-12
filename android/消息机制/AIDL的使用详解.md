### AIDL的详细讲解
为了使得一个程序能够在同一时间里处理许多用户的要求。即使用户可能发出一个要求，也肯能导致一个操作系统中多个进程的运行（PS:听音乐，看地图）。而且多个进程间需要相互交换、传递信息，IPC方法提供了这种可能。IPC方法包括管道（PIPE）、消息排队、旗语、共用内存以及套接字（Socket）。

Android中的IPC方式有Bundle、文件共享、Messager、AIDL、ContentProvider和Socket。在有些地方，AIDL就是一种可以考虑的方式。那么下面主要讲解一下如何使用aidl，以及在使用过程中可能遇到的问题。

#### 预备知识
AIDL使用简单的语法来声明接口，以及描述其方法以及方法的参数和返回值。这些参数和返回值可以是任何类型，甚至是其他AIDL生成的接口。重要的是**必须手动**导入所有非内置类型，哪怕是这些类型是在与接口相同的包中。
1. 通常引入引用方式传递的其他AIDL生成的接口（也就是自定义类型），必须要import 语句声明。
2. Java编程语言的主要类型 (int, boolean等) —不需要 import 语句。
3. 在AIDL文件中，并不是所有的数据类型都是可以使用的，那么到底AIDL文件中支持哪些数据类型呢？
    - 基本数据类型（int,long,char,boolean,float,double,byte,short八种基本类型）;
    - String和CharSequence;
    - List:只支持ArrayList,里面每个元素都必须能够被AIDL支持；
    - Map:只支持HashMap，里面的每个元素都必须被AIDL支持，包括key和value;
    - 自定义类型必须实现Parcelable:所有实现了Parcelable接口的对象；
    - AIDL：所有的AIDL接口本身也可以在AIDL文件中使用；
    - 补充：自定义的Parcelable对象和AIDL对象必须要显式import进来，不管它们是否和当前的AIDL文件位于同一个包内

4. 在定义接口参数的时候，需要注意的地方
    - AIDL中除了基本数据类型，其他类型的参数**必须**标上方向：in、out或者inout；
    - in表示输入型参数（Server可以获取到Client传递过去的数据，但是不能对Client端的数据进行修改）
    - out表示输出型参数（Server获取不到Client传递过去的数据，但是能对返回给Client端的数据进行修改）
    - inout表示输入输出型参数（Server可以获取到Client传递过去的数据，也能对返回给Client端的数据进行修改）
5. 序列化
    - Serializeable
    - Parcelable

#### 开发步骤
1. 先穿件对应的aidl包，然后创建aidl文件；如Book.aidl
```
    // Book.aidl
    package com.aidldemo.aidl;

    // Declare any non-default types here with import statements
    //所有注释掉的内容都是Android Studio帮你写的，但是我们不需要。
    //我们创建的是aidl数据对象，所以我们只需写出parcelable 后面跟对象名。
    //parcelabe前的字母‘p’是小写的哦~
    parcelable Book;
    
```
2. 在aidl包下，创建自定义类型的java类，如Book.java，然后实现Parcelable接口
3. 创建aidl接口文件，即跨进程服务调用接口
```
    // IBookManager.aidl
    package com.tzx.aidldemo.aidl;
    //通常引用方式传递自定义对象，必须要import语句声明
    import com.tzx.aidldemo.aidl.Book;
    interface IBookManager {
        List<Book> getBookList();
        void addBook(in Book book);
    }

```
4. 在BookManagerService中，onBind返回 实现IBookManager.Stub接口的binder对象，作为客户端将要引用的服务端对象
5. 客户端连接后IBookManager.Stub.asInterface(service);获得服务端在本地对象实例



####  在实际开发过程中可能遇到的问题：
    - 找不到文件
        - 在Android Studio中如果先创建Java类文件，然后创建AIDL文件则会提示命名重复，但顺序反过来就可以。
    - 自定义类型找不到
        -因为在使用aidl的时候客户端和服务器端包名也要完全一样我们为了方便直接复制整个aidl文件夹到客户端会把aidl文件和aidl接口中用到的自定义类型都放到这个文件中  
        ```
            sourceSets {
                main {
                    manifest.srcFile 'src/main/AndroidManifest.xml'
                    java.srcDirs = ['src/main/java', 'src/main/aidl']
                    resources.srcDirs = ['src/main/java', 'src/main/aidl']
                    aidl.srcDirs = ['src/main/aidl']
                    res.srcDirs = ['src/main/res']
                    assets.srcDirs = ['src/main/assets']
                }
            }
        ```
    - 在客户端调用service端接口时，出现错误提示
        - These exceptions do indeed get thrown and you should write appropriate try/catch logic to handle the situation where a remote method you invoked on a service did not complete.


#### 补充1：死亡代理
我们知道，Binder运行在服务端进程，如果服务端进程由于某些原因异常终止，这个时候客户端到服务端的Binder链接中断，则导致之后的远程调用失败，甚至我们不知道什么时候中断的。那么Binder提供了 linkToDeath 和 unlinkToDeath两个方法。当Binder连接断开是，我们就会收到通知。那么如何使用呢？
```
    IBinder.DeathRecipient mDeathRecipient = new IBinder.Recipient(){
        //
        @Override
        public void binderDied(){
            //先移除旧的binder代理，再重新绑定
            mLocalService.asBinder().unlinkToDeath(mDeathRecipient,0);
            mLocalService = null;
            //重新绑定

        }
    }
```

#### 补充2：AIDL的权限验证