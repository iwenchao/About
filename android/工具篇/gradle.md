### Gradle的配置
#### Gradle是什么
1. Gradle是一个自动化构建工具
2. 兼容Maven等仓库
3. 基于Groovy的特定领域语言来声明名目设置

#### GradleWraper
1. Gradle Wrapper是一个脚本文件
2. 它会在没有安装Gradle的情况下为我们下载Gradle，之后我们就可以使用gradlew命令，像使用gradle一样来使用Gradle了
3. GradleWraper简化了gradle的安装部署

#### Gradle文件结构
1. settings.gradle:整个Project的配置文件，可以设置包含哪些module
2. build.gradle （Project的gradle文件）:整个Project的配置文件
3. build.gradle（Module）：Module的配置文件
4. gradle.properties：可以在 gradle.properties 文件中配置一些变量

#### Gradle命令
1. gradlew clean: 清除app目录下的build文件夹
2. gradlew check: 执行lint检查
3. gradlew assemble：打release和debug包
4. gradlew build ： 执行check和assemble
5. gradlew assembleRelease/gradlew assembleDebug：打全部渠道的Release或者debug包


####  Gradle常见配置
1. 指定仓库
```
repositories {
    jcenter()
}
```
2. 指定依赖
```
dependencies {
    classpath 'com.android.tools.build:gradle:1.3.1'
    //依赖最新的1.x版本
    compile "org.codehaus.cargo:cargo-ant:1.+"
}
```
3. 设置脚本的运行环境
```
buildscript{}
```
4. 声明引用的插件
```
apply plugin: 'com.android.application'

```
5. 设置编译android项目的参数
```
android {
    // 编译SDK的版本
    compileSdkVersion 22
    // build tools的版本
    buildToolsVersion "23.0.1"

    //aapt配置
    aaptOptions {
        //不用压缩的文件
        noCompress 'pak', 'dat', 'bin', 'notice'
        //打包时候要忽略的文件
        ignoreAssetsPattern "!.svn:!.git"
        //分包
        multiDexEnabled true
        //--extra-packages是为资源文件设置别名：意思是通过该应用包名+R，com.android.test1.R和com.android.test2.R都可以访问到资源
        additionalParameters '--extra-packages', 'com.android.test1','--extra-packages','com.android.test2'
    }

    //默认配置
    defaultConfig {
        //应用的包名
        applicationId "com.example.heqiang.androiddemo"
        minSdkVersion 21
        targetSdkVersion 22
        versionCode 1
        versionName "1.0"
    }

    //编译配置
    compileOptions {
        // java版本
        sourceCompatibility JavaVersion.VERSION_1_7
        targetCompatibility JavaVersion.VERSION_1_7
    }

    //源文件目录设置
    sourceSets {
        main {
             //jni lib的位置
             jniLibs.srcDirs = jniLibs.srcDirs << 'src/jniLibs'
             //定义多个资源文件夹,这种情况下，两个资源文件夹具有相同优先级，即如果一个资源在两个文件夹都声明了，合并会报错。
             res.srcDirs = ['src/main/res', 'src/main/res2']
             //指定多个源文件目录
             java.srcDirs = ['src/main/java', 'src/main/aidl']
        }
    }

    //签名配置
    signingConfigs {
        debug {
            keyAlias 'androiddebugkey'
            keyPassword 'android'
            storeFile file('keystore/debug.keystore')
            storePassword 'android'
        }
    }

    buildTypes {
        //release版本配置
        release {
            debuggable false
            // 是否进行混淆
            minifyEnabled true
            //去除没有用到的资源文件，要求minifyEnabled为true才生效
            shrinkResources true
            // 混淆文件的位置
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.txt'
            signingConfig signingConfigs.debug
            //ndk的一些相关配置，也可以放到defaultConfig里面。
            //指定要ndk需要兼容的架构(这样其他依赖包里mips,x86,arm-v8之类的so会被过滤掉)
            ndk {
                abiFilter "armeabi"
            }
        }
        //debug版本配置
        debug {
            debuggable true
            // 是否进行混淆
            minifyEnabled false
            //去除没有用到的资源文件，要求minifyEnabled为true才生效
            shrinkResources true
            // 混淆文件的位置
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.txt'
            signingConfig signingConfigs.debug
            //ndk的一些相关配置，也可以放到defaultConfig里面。
            //指定要ndk需要兼容的架构(这样其他依赖包里mips,x86,arm-v8之类的so会被过滤掉)
            ndk {
                abiFilter "armeabi"
            }
        }
    }
    // lint配置 
    lintOptions {
      //移除lint检查的error
      abortOnError false
      //禁止掉某些lint检查
      disable 'NewApi'
    }
}
```
android中还可以有以下配置：
productFlavors{ } 产品风格配置，ProductFlavor类型；testOptions{ } 测试配置，TestOptions类型； dexOptions{ } dex配置，DexOptions类型；packagingOptions{ } PackagingOptions类型；jacoco{ } JacocoExtension类型。 用于设定 jacoco版本；splits{ } Splits类型。

#### 几种依赖的区别
1. compile: 最常用的依赖方式，编译时提供并打包进apk
2. provide：编译时提供，但是不打进apk包中
3. 在gradlew3.0中compile废弃，使用implement，api替代
4. api == compile
5. implement：将该依赖隐藏在内部，而不对外部模块公开

#### 为什么会有两套repositories和dependencies

1. buildscript里面的那个是插件初始化环境用的，用于设定插件的下载仓库
2. android 中是工程依赖的一些模块和远程library的下载仓库的

#### 排除依赖传递,解决依赖冲突
1. exclude：设置不编译指定的模块，排除指定模块的依赖
2. transitive：用于自动处理子依赖项，默认为true，gradle自动添加子依赖项。设置为false，排除所有的传递依赖
3. force：强制设置某个模块的版本


#### Gradle打包时的Proguard
1. 通过buildTypes中配置minifyEnable来开启和关闭proguard
2. 通过proguardFiles来配置混淆参数与keep的内容
3. 混淆的作用：
    - 压缩（shrink）：检测并移除代码中的无用类，字段，方法和特性Attribute
    - 优化（Optimize）：对字节码进行优化，移除无用的指令
    - 混淆（Obfuscate）：使用a，b，c简短无意义的名称对类，字段，方法进行重命名
    - 预检（Preverify）：在java平台上对处理后的代码进行预检，确保加载的class文件是可执行的。

#### 多渠道打包
1. 先在AndroidManifest.xml配置mete-data
```
<meta-data
    android:name="UMENG_CHANNEL"
    android:value="Channel_ID" />
```
2. 配置Flavors：
```

android {  
    productFlavors {
        xiaomi {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "xiaomi"]
        }
        _360 {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "_360"]
        }
        baidu {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "baidu"]
        }
        wandoujia {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "wandoujia"]
        }
    }  
}

    
或者批量修改

android {  
    productFlavors {
        xiaomi {}
        _360 {}
        baidu {}
        wandoujia {}
    }  

    productFlavors.all { 
        flavor -> flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name] 
    }
}


```
3. 在APP内读取 mete-data 配置确定渠道
4. 然后用 ./gradlew assembleRelease 这条命令会把Product Flavor下的所有渠道的Release版本都打出来。


#### 多渠道打包2
1. 因为以上方法需要多次编译，速度较慢，当渠道变多之后不适合多渠道打包
2. 改进的方法1 ： apk反编译后重写AndroidManifest文件，再重新编译签名
3. 改进的方法2 : 如果在META-INF目录内添加空文件，可以不用重新签名应用。因此，通过为不同渠道的应用添加不同的空文件，可以唯一标识一个渠道

#### 多渠道打包3
1. 在采用V2签名后，以上方法不再适用。因为v1签名可以在不改变签名情况下二次打包，我们可以在gradle中对dex文件进行自己的签名
2. 考虑到V2签名的特点（对APK Signing Block是不进行验证的），我们向V2签名后的APK签名区块写入渠道号，实现多渠道打包

#### 哪些不做混淆
1. Android系统组件
1. JNI
1. 反射
1. WebView的JS调用
1. 内部类
1. Annottation
1. enum
1. 范型
1. 序列化
1. 第三方

#### gradle声明周期
1. 初始化阶段：会去读取根工程的setting.gradle中的include信息，决定有哪几个模块加入构建，创建project实例，
2. 配置阶段：执行所有模块的build.gradle脚本，配置project对象，一个对象由多个task组成，此阶段也会创建，配置task以及相关信息
3. 运行阶段：根据gradle的命令，传递过来的task名称，执行相关呢依赖任务


#### 如何通过gradle配置差异较大的多渠道包
通过配置productFlavors，将区别代码放置在对应的问价下，gradle会自动打出相应包