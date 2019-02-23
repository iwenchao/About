#### apk动态加载机制分析

首先常理来讲，apk必须安装才能运行。那可不可以不安装apk，就直接运行该apk呢？理论上可行，但是实现起来会有难度。基本思路是这样的：借助一个宿主进程，如parentApk（已安装好的）。然后在宿主内将apk去加载进来运行。难度在于：首先在上下文Context。如果宿主的context无法访问apk的资源。还有一个就是apk的组件生命周期无法保证。


- 生命周期实现的原理：将宿主内的proxyActivity设置到apk内的Activity中，由proxyActivity进行操作。
- Plugin中资源文件的获取：可以使用AssetManager去得到Plugin包中的资源文件


#### 实现细节：

1. 要提供一套标准 PluginInterface.
    - 这套标准用来规范宿主与Plugin之间的上下文以及生命周期关系的标准
2. PluginManager
    - 需要一套工具，这个工具用来管理加载Plugin，以及获取Plugin中资源文件等
    1. 获取Plugin的字节码文件对象. 
       - 要拿到Plugin中的字节码文件对象，需要拿到Plugin对应的DexClassLoader。可以使用DexClassLoader的DexClassLoader(String dexPath, String optimizedDirectory, String libraryPath, ClassLoader parent)方法。
        1. dexPath是Plugin apk的路径
        2. optimizedDirectory是Plugin的缓存路径
        3. libraryPath可以为null
        4. parent为父类加载器
    2. 然后就可以使用DexClassLoader.loadClass(PluginActivityName);加载到PluginActivity的字节码文件对象了，进而创建PluginActivity的实例。
    3. 获取Plugin的Resources
        - 
                ```
            public Resources(AssetManager assets, DisplayMetrics metrics, Configuration config) {
                    this(assets, metrics, config, CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO);
            }
                ```
        - 由于要获取Plugin中的资源，所以这个assets对象应当是Plugin中的资源对象；而对于一款手机的DisplayMetrics和Configuration来说，无论是宿主还是Plugin获取的值都是一样的，所以可以使用宿主的值。
        - 获取AssetManager对象:
                ```
            public final int addAssetPath(String path) {
                    synchronized (this) {
                        int res = addAssetPathNative(path);
                        makeStringBlocks(mStringBlocks);
                        return res;
                    }
                }
                ```
            - 这个path也就是Plugin包在手机中的位置，由于这个方法被hide了，我们需要使用反射。
    4. ProxyActivity: 宿主代理Activity
        - 提供一套生命周期和上下文，给我们自己创建的PluginActivity的的实例用的。
        - 我们自己加载的PluginActivity实例只是一个对象，没有任何意义的，要给它套上生命周期，给他的上下文赋值。
        - 具体实现思路：
            - 先去启动ProxyActivity，然后再ProxyAcitivity中的oCreate方法中去创建PluginActivity的实例，然后去调用PluginActivity的onCreate方法。在ProxyActivity的onResume方法中调用PluginActivity的onResume方法等等。
    5. Plugin的BaseActivity的构建
        - 构建Plugin的BaseActivity的原因是
            1. 统一上下文为ProxyActivity的实例，关于上下文的各种操作均是调用宿主ProxyActivity的实例去进行操作。