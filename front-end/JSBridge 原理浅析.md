## JSBridge原理浅析与实践

​		在字节跳动实习了一段时间，JSBridge使用的比较频繁，之前只是看了些简单的JSBridge概念，一直没有时间去了解从客户端到JavaScript的一个通信原理（JSBridge）。最近花了点时间学习了从Android端到JavaScript的通信-**JSBridge**（主要是太闲了），对JSBridge有了更深入的理解，特地写下了这篇文章分享一下。

### JSBridge是什么

​		顾名思义：**就是JavaScript(H5)与Native通信的桥梁**，在H5开发中经常有操作客户端的需求，比如获取App信息，打开/关闭一个WebView，吊起支付面板等等，但这些功能只能在Native中实现，因此诞生JSBridge，通过JSBridge与Native通信，给JavaScript操作Native的能力同时也给了Native调用JavaScript的能力。

​									 <img src="/Users/dengpengfei/Library/Application Support/typora-user-images/image-20191221140303289.png" alt="image-20191221140303289" style="zoom:50%;" />

### JSBridge与Native间通信原理

在H5中JavaScript调用Native的方式主要用两种

1. **注入API**，注入Native对象或方法到JavaScript的window对象中（可以类比于RPC调用）。
2. **拦截URL Schema**，客户端拦截WebView的请求并做相应的操作（可以类比于JSONP）。

下面将以Android端的JSBridge通信为例，讲解这两种方式的实现原理（本人比较菜，只会Java不会Swift和OC😭）。

#### 注入API

通过WebView提供的接口，向JavaScript的window中注入对象或方法(Android使用```addJavascriptInterface()```方法)，让JavaScript调用时相当于执行相应的Native逻辑，达到JavaScript调用Native的效果。

对于Android实现方式如下，核心代码在于

```java
webView.addJavascriptInterface(new InjectNativeObject(this), "NativeBridge");
```

示例如下

在Android的main页面放一个Webview

![image-20191214204302240](/Users/dengpengfei/Library/Application Support/typora-user-images/image-20191214204302240.png)

然后Android端对应的代码如下

```java
public class MainActivity extends AppCompatActivity {
    private WebView webView;
    // 不要用localhost或127.0.0.1
    private final String host = "192.168.199.231";

    public class InjectNativeObject { // 注入到JavaScript的对象
        private Context context;
        public InjectNativeObject(Context context) {
            this.context = context;
        }
        @JavascriptInterface
        public void openNewPage(String msg) { // 打开新页面，接受前端传来的参数
            if (msg.equals("")) {
                Toast.makeText(context, "please type!", Toast.LENGTH_LONG).show();
                return;
            }
            startActivity(new Intent(context, SecondActivity.class));
            Toast.makeText(context, msg, Toast.LENGTH_SHORT).show();
        }

        @JavascriptInterface
        public void quit() { // 退出app
            finish();
        }
    }

    @SuppressLint("SetJavaScriptEnabled")
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        webView = findViewById(R.id.loginWebView);
        webView.getSettings().setJavaScriptEnabled(true);
        // JS注入
        webView.addJavascriptInterface(new InjectNativeObject(this), "NativeBridge");
        webView.loadUrl(String.format("http://%s:3000/login_webview", host)); // 加载Webview
    }

}

```



对于JavaScript侧，则可以直接调用Native端注入的InjectNativeObjec对象的方法

```javascript
window.NativeBridge = window.NativeBridge || {}; // 注入的对象
// 登录按钮点击，调用注入的openNewPage方法，并传入相应的值
loginButton.addEventListener("click", function (e) {
    window.NativeBridge.openNewPage(accountInput.value + passwordInput.value);
}, false);
// 退出按钮点击，调用quit方法
quitButton.addEventListener("click", function (e) {
    window.NativeBridge.quit();
}, false)
```

实际效果如下，其中login按钮点击调用Native端openNewPage方法并传相应的参数给客户端。

<img src="/Users/dengpengfei/Desktop/app.gif" alt="app" style="zoom:33%;" />

**缺陷：**Android4.2及以下的版本使用addJavascriptInterface方法有漏洞

> 该漏洞源于程序没有正确限制使用WebView.addJavascriptInterface方法，远程攻击者可通过使用Java Reflection API利用该漏洞执行任意Java对象的方法，简单的说就是通过addJavascriptInterface给WebView加入一个JavaScript桥接接口，JavaScript通过调用这个接口可以直接操作本地的Java接口。

在Android4.2以上提供```@JavascriptInterface```注解来规避该漏洞，但对于4.2以下版本则没有任何方法。所以使用该方法有一定的风险和兼容性问题。

#### 拦截URL Schema

H5端通过**iframe.src**或**localtion.href**发送Url Schema请求，之后Native（Android端通过```shouldOverrideUrlLoading()```方法）拦截到请求的Url Schema（包括参数等）进行相应的操作。

通俗点讲就是，H5发一个普通的https请求可能是: https://daydream.com/?a=1&b=1，而与客户端约定的JSBridge Url Schema可能是: Daydream://jsBridgeTest/?data={a:1,b:2}，客户端可以通过schema来区分是JSBridge调用还是普通的https请求从而做不同的处理。

其实现过程原理类似于JSONP

1. 首先在H5中注入一个callback方法，放在window对象中，如

```javascript
function callback_1(data) { console.log(data); delete window.callback_1 };
window.callback_1 = callback_1;
```

然后把callback的名字通过Url Schema传到Native

2. Native通过```shouldOverrideUrlLoading()```，拦截到WebView的请求，并通过与前端约定好的Url Schema判断是否是JSBridge调用。
3. Native解析出前端带上的callback，并使用下面方式调用callback

```java
webView.loadUrl(String.format("javascript:callback_1(%s)", isChecked)); // 可以带上相应的参数
```

或者

```java
webView.evaluateJavascript(String.format("callback_1(%s)", isChecked), value -> {
       // value callback_1执行是返回值
       Toast.makeText(this, value, Toast.LENGTH_LONG).show();
});
```

通过上面几步就可以实现JavaScript到Native的通信。下面可以看看处理Url Schema的拦截的**shouldOverrideUrlLoading**方法的相关例子

```java
public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
        String schema = request.getUrl().getScheme(); // https or Daydream
        if (this.schema.equals(schema)) { // 如果和约定好的Schema一致，则处理JSBridge调用
            String callback = request.getUrl().getQueryParameter("callback");
            String comment = request.getUrl().getQueryParameter("comment");
            assert comment != null;
            if (comment.equals("")) {
                Toast.makeText(context, "please type some comment!", Toast.LENGTH_LONG).show();
                return false;
            }
            // 使用loadUrl的方式来调用window上的方法
            view.loadUrl(String.format("javascript:%s('%s')", callback, comment));
        }
        return super.shouldOverrideUrlLoading(view, request);
}
```

示例

Android页面布局

![image-20191214205819584](/Users/dengpengfei/Library/Application Support/typora-user-images/image-20191214205819584.png)

Android端代码如下

``` java
class MyWebViewClient extends WebViewClient {
    private final String schema = "sundial-dreams";
    private Context context;
    public MyWebViewClient(Context context) {
        this.context = context;
    }
    // 拦截Schema
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
        String schema = request.getUrl().getScheme(); // 获取Schema https or Daydream
        if (this.schema.equals(schema)) {
            String callback = request.getUrl().getQueryParameter("callback");
            String comment = request.getUrl().getQueryParameter("comment");
            assert comment != null;
            if (comment.equals("")) {
                Toast.makeText(context, "please type some comment!", Toast.LENGTH_LONG).show();
                return false;
            }
            // 使用loadUrl的方式来调用window上的方法
            view.loadUrl(String.format("javascript:%s('%s')", callback, comment));
        }
        return super.shouldOverrideUrlLoading(view, request);
    }

    @Override
    public void onPageStarted(WebView view, String url, Bitmap favicon) {
        super.onPageStarted(view, url, favicon);
    }

    @Override
    public void onPageFinished(WebView view, String url) {
        super.onPageFinished(view, url);
    }
}

public class SecondActivity extends AppCompatActivity {
    private WebView webView;
    private SearchView searchView;
    private Switch aSwitch;
    private VideoView videoView;

    private final String host = "192.168.199.231";

    @SuppressLint({"SetJavaScriptEnabled"})
    @Override
    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        setContentView(R.layout.activity_second);

        webView = findViewById(R.id.commentWebView);
        searchView = findViewById(R.id.searchView);
        aSwitch = findViewById(R.id.comment_switch);
        videoView = findViewById(R.id.videoView);

        videoView.setMediaController(new MediaController(this));
//        videoView.setVideoPath(Uri.parse("url").toString());
//        videoView.start();
//        videoView.requestFocus();

        aSwitch.setChecked(true);
        aSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            // Native调用JS，loadUrl 或者 evaluateJavascript
            webView.evaluateJavascript(String.format("DisplayCommentCard(%s)", isChecked), value -> {
                Toast.makeText(this, value, Toast.LENGTH_LONG).show();
            });
//            webView.loadUrl(String.format("javascript:DisplayCommentCard(%s)", isChecked));
        });

        webView.setWebViewClient(new MyWebViewClient(this));
        webView.getSettings().setJavaScriptEnabled(true);
        webView.loadUrl(String.format("http://%s:3000/page_webview", host));
    }
}

```

JavaScript侧

window上挂载的方法可以使用```webView.loadUrl() / webView.evaluateJavascript()```调用

```javascript
 /**
   * @return {string}
   */
function DisplayCommentCard (display) {
    commentList.style.opacity = +display;
    return "JavaScriptFunction";
}
window.DisplayCommentCard = DisplayCommentCard;
```

基于JSONP原理可以定义JSBridge类

```javascript
class JSBridge {

  constructor () {
    this.schema = 'sundial-dreams'; // 与客户端约定的schema
    this.iframe = this.createIFrameElement();
    this.id = 0; // callback id
  }

  createIFrameElement () { // 基于iframe.src发请求
    const iframe = document.createElement('iframe');
    iframe.style.display = 'none';
    document.body.appendChild(iframe);
    return iframe;
  }

  call (params = {}, callback) {
    params = Object.keys(params).reduce((acc, curKey) => acc + `${ curKey }=${ params[curKey] }&`, '');
    const name = `__callback__${ this.id++ }`;
    const src = `${ this.schema }://JSBridge?${ params }callback=${ name }`;
    window[name] = function (value) {
      delete window[name];
      typeof callback === 'function' && callback(value);
    };
    this.iframe.src = src;
  }
}
```

JavaScript使用例子

```javascript
const jsBridge = new JSBridge();
// 评论按钮点击，调用JSBridge
commentButton.addEventListener("click", function (e) {
  jsBridge.call({ comment: commentInput.value }, value => {
    commentInput.value = "";
    AddComment(value);
  });
}, false);
```

效果如下

<img src="/Users/dengpengfei/Desktop/app2.gif" alt="app2" style="zoom:33%;" />

**缺陷：** 1. 使用URL Schema有一定的长度问题，url过长可能会导致丢失

 		   2. 一次JSBridge调用耗时可能比较长，创建请求需要一定的时间

###总结

本文简单介绍了JSBridge以及在Android端的通信，通过相应的分析与代码实现可以发现JSBridge原理其实并没有那么难，因此希望本文能对读者对JSBridge有一定的理解，最后生活不止有前端，还有客户端在等着咱o(T^T)o（前端可太难了。。。）

### 完整代码

GitHub: [JSBridgeTest](https://github.com/sundial-dreams/JSBridgeTest)