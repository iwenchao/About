1.activity启动流程：

跨进程启动；


1、如何跨app启动activity？有哪些注意事项？
    a. sharedUserId="包名"。 startActivity(Intent().setComponent（包名，类名）)
    b. exported="true" 。 startActivity(Intent().setComponent（包名，类名）)
    c. intent-filter ：action+category
    d. 自定义权限。 A 使用 uses-permission B声明permission自定义。但是B要先于A安装才可以
注意事情：
    a. 拒绝服务漏洞：try-catch


3、如何解决activity参数传递的类型安全问题？
    - 类型安全：bundle的k-v不能在编译器保证类型；
    - 接口繁琐：启动activity时参数和结果传递都依赖intent
    - 等价的问法：设计一个框架，解决上述问题；  使用apt技术，通过对参数注解，以及注解反射处理器，生成中间代码，完成参数传递。建造者builder模式
        - 读取注解信息-- 构造生成的类结构，---java-poet/kotlin-poet --- java /kotlin
        - 注意：
            - 类的继承关系。类的内部类情况。java与kotlin类型映射的问题；
            - 把握好代码生成和直接依赖的边界。



元编程：
        - apt：Arouter，Dagger
        - bytecode：Replugin
        - Generic：
        - Reflect：
        - Proxy：Retrofit



    a。是否有代码优化和代码重构的意识
    b。是否对反射，注解处理器有了解
    c。是否具备一定的框架设计能力；


### 如何在代码的任意位置为当前activity添加View？
1. 
    - 如何获取当前activity：ActivityLifecycleCallbacks，小心activity内存泄漏，使用弱引用。
    - 如何在不影响正常view展示的情况下添加View？
    - 既然能添加，就应当有移除，如何安全移除呢？
    - 这样做的目的是什么？添加全局view是否更合适？
2. 