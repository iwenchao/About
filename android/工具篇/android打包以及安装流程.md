[https://juejin.im/entry/58b78d1b61ff4b006cd47e5b](https://juejin.im/entry/58b78d1b61ff4b006cd47e5b)

#### 打包流程
1. 通过aapt打包res资源文件，生成R.java、resources.arsc和res文件（二进制 & 非二进制如res/raw和pic保持原样）
2. 处理.aidl文件，生成对应的Java接口文件
3. 通过Java Compiler编译R.java、Java接口文件、Java源文件，生成.class文件
4. 通过dex命令，将.class文件和第三方库中的.class文件处理生成classes.dex
5. 通过apkbuilder工具，将第一步生成的文件与class.dex文件合并，打包成apk，未签名的
6. 通过Jarsigner工具，对上面的apk进行debug或release签名
7. 通过zipalign工具，将签名后的apk进行对齐处理。


[http://blog.zhaiyifan.cn/2016/02/13/android-reverse-2/](http://blog.zhaiyifan.cn/2016/02/13/android-reverse-2/)
##### 更详细的打包流程
1. 通过aapt工具将各个模块的manifest文件，resources资源（layout,drawable等），assets资源进行加载以及解析最后进行merge合并，生成R.java,以及生成最终资源表。
    - res中图片和raw文件下内容保持原样，res中其他xml文件内容均转化为二进制形式；assets文件内容保持原样resources.arsc，res文件（二进制 & 非二进制如res/raw和pic保持原样）。res中的文件会被映射到R.java文件中，访问的时候直接使用资源ID即R.id.filename；
    - assets文件夹下的文件不会被映射到R.java中，访问的时候需要AssetManager类
2. 处理aidl文件生成java接口文件，以及编译RenderScript，生成BuildConfig。
3. 由javac java编译器 编译R.java,工程源码java文件，以及生成的中间java文件。最后生成.class文件
4. 根据项目中的混淆配置，将class文件进行混淆（class文件包括项目文件，以及第三方库，以及jar包等）最后生成混淆过的jar包
5. 然后根据dx工具将jar包进一步的编译以及优化成classes.dex文件
6. 将第一步已编译好的resources.arsc，res文件以及classes.dex文件一起合并，通过apkbuilder工具打包，生成未签名的apk文件
    1. 以包含resources.arcs的.ap_文件为基础，new一个ApkBuilder，设置debugMode
    2. apkBuilder.addZipFile(f);
    3. apkBuilder.addSourceFolder(f);
    4. apkBuilder.addResourcesFromJar(f);
    5. apkBuilder.addNativeLibraries(nativeFileList);
    6. apkBuilder.sealApk(); // 关闭apk文件
    7. generateDependencyFile(depFile, inputPaths,outputFile.getAbsolutePath());
7. 通过jarsinger对apk签名
8. 有zipalign工具对最后的签名apk文件进行最后的对齐处理。
    - 使apk中所有资源文件距离文件起始偏移为4字节的整数倍，从而在通过内存映射访问apk文件时会更快。


#### dex方法数64k限制

