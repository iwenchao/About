### 统计task执行的耗时
根据gradle的提供的TaskExecutionListener，BuildListener 在task执行前后进行时间统计，结果发现transformClassesWithDexForDebug最为耗时。

### 先列出关键task以及其作用
1. mergeDebugResources：解压所有的aar包输出到app/build/intermediates/exploded-aar，并且把所有的资源文件合并到app/build/intermediates/res/merged/debug目录里
    1. 调用aapt生成项目和所有aar依赖的R.java,输出到app/build/generated/source/r/debug目录
    2. 生成资源索引文件app/build/intermediates/res/resources-debug.ap_
    3. 把符号表输出到app/build/intermediates/symbols/debug/R.txt
2. processDebugManifest: 把所有aar包里的AndroidManifest.xml中的节点，合并到项目的AndroidManifest.xml中，并根据app/build.gradle中当前buildType的manifestPlaceholders配置内容替换manifest文件中的占位符，最后输出到app/build/intermediates/manifests/full/debug/AndroidManifest.xml

3. compileDebugJavaWithJavac: 将java文件编译成class文件。
    1. 项目源码，
    2. aidl相关的java代码
    3. buildConfig
    4. apt编译工具生成的代码。

4. transformClassesWithJarMergingForDebug： 将上一步生成的class文件，以及其他第三方的库的arr包的class.jar包进行合并。

5. transformClassesWithMultidexlistForDebug：
    1. 扫描项目的AndroidManifest.xml文件和分析类之间的依赖关系，计算出那些类必须放在第一个dex里面,最后把分析的结果写到app/build/intermediates/multi-dex/debug/maindexlist.txt文件里面
    2. 生成混淆配置项输出到app/build/intermediates/multi-dex/debug/manifest_keep.txt文件里

6. transformClassesWithDexForDebug： 将所有class jar包转换成dex文件。class文件越多转换的越慢


#### 需要优化的是
第一次全量打包dex，并缓存下来。然后对当前所有java文件做快照，方便下次打包的时候根据快照进行对比。然后将combined.jar中没有变化的class移除，仅仅把变化class进行dx处理，得到dex。然后在打包apk的时候，优先后来的dex文件。


[https://juejin.im/post/59cc90a35188257e8f03b8d4](https://juejin.im/post/59cc90a35188257e8f03b8d4)