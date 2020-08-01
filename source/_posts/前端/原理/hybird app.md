---
title: hybird app
category:
  - 前端
  - 原理
tags:
  - 前端
  - 原理
abbrlink: 4d1f8c28
date: 2019-10-29 06:00:00
updated: 2019-10-29 06:00:00
---

hybird app 介于 native app、web app 之间，性能比 web app 好，比 native app 差，但是适应了更多的移动开发场景。hybird app 具有如下特点：因为受到移动设备CPU、内存、网卡、网络连接等限制，hybird app 可用的系统网络资源有限；hybird app 支持更新的浏览器特性；hybird app 可实现离线应用，即借助浏览器最新的特性或 Native 的文件读取机制进行文件级的文件缓存或离线更新；hybird app 需要考虑不同设备机型的兼容性问题；hybird app 支持唤起 Native 的能力，如摄像头、定位、传感器、本地文件访问等。理想情况下，hybird app 须唤起 native 能力绘制通用导航菜单、系统 UI、核心动效等，这样能充分利用 native 的性能优势（WebView 的执行性能只有移动端浏览器的 1/3 ~ 1/4）。

### web 到 native 通信协议

#### 通过 url

Native 应用在移动端系统中注册一个 Schema 协议的 URI，这个 URI 可在系统的任意地方授权访问来掉漆一段原生方法或一个原生的界面。在此基础上，Native 应用的 WebView 控件中的 js 脚本也可以远程请求匹配 Schema 协议的 URI，如通过 iframe 的 src 属性，这个请求就能被 Native 应用的系统捕获并调起 Native 应用注册匹配的 Schema 协议内容，以唤起原生能力。

```javascript
let iframe = document.createElement('iframe');
iframe.setAttribute('style', 'display: none');
document.appendChild(iframe);
iframe.setAttribute('src', 'myApp://className/method?args');
```

#### 通过 addJavascriptInterface

Native 应用也可以通过 addJavascriptInterface 方法将 Native 的一个对象方法注入到页面 window 对象中，供 js 调用。

```java
Websettings websettings = webView.getSettings();
websettings.setJavascriptEnabled(true);// 设置 webView 允许执行 js 脚本
webView.loadUrl('file:///android_asset/index.html');// 加载页面到 webView 中
webView.addJavascriptInterface(new JsInterface(), 'native');// window 加入 native 对象

public class JsInterface {
  @JavascriptInterface
  public void showToast(String toast) {
    Toast.makeText(MainActivity.this, toast, Toast.LENGTH_SHORT).show();
  }
}
```

addJavascriptInterface 在 Android 4.2 以下版本有安全漏洞。另一种可行的方法时，当 js 调用 alert 或 prompt 方法时，会执行 Native 中的 onJsAlert 或 onJsPrompt 方法，因此就可以用这两个方法加以监听 html5 传递的消息。

```java
webView.setWebChromeClient(new WebChromeClient() {
  @Override
  public boolean onJsPrompt(WebView view, String url, String message, 
    String defaultValue, JsPromptResult result) {
    result.confirm(JSBridge.callJsPrompt(MainActivity.this, view, message));
    return true;
  }
})
```

### native 到 web 通信协议

首先在 html5 页面的全局作用域下声明一个方法；其次在 Android 中使用 loadUrl 唤起这个方法，或者在 iOS 中使用 stringByEvaluatingJavaScriptFromString 唤起这个方法，这样就能使 Native 应用主动调用 html5 中的 js 方法。

```java
Websettings websettings = webView.getSettings();
websettings.setJavascriptEnabled(true);// 设置 webView 允许执行 js 脚本
webView.loadUrl('file:///android_asset/index.html');// 加载页面到 webView 中
JsInterface jsInterface = new JsInterface();
jsInterface.log('hello world');

public class JsInterface {
  public void log(final String msg) {
    webView.post(new Runnable() {// 创建一个新线程执行 js 中的 log 方法
      @Override
      public void run() {
        // 调用 html5 中 window.log 方法
        webView.loadUrl('javascript: log(' + '"' + msg + '"' + ')')
      }
    })
  }
}
```

### 通信协议

目前较多使用的通信协议方案为 jsbridge://className:callbackMethod/methodName?jsonObj。jsbridge 为 Native 注册的协议头；className 为调用 Native 的类；methodName 为类中的方法名；jsonObj 为传递的参数；callbackMethod 为 Native 回调 js 的方法名，即 Native 调用 webView.loadUrl(‘javascript: callbackMethod()’)。以 html5 中请求 jsbridge://Util:success/toString?{“msg”: “hello world”} 为例，Native 使用 Util 类中的 toString 方法处理参数 {“msg”: “hello world”}，完成后再回调 html5 中的 success 方法。

通常 jsBridge 中会使用公共方法发送请求。如下：

```javascript
JsBridge.call = function(className, methodName, params, callback) {
  let bridgeStr;
  let paramsStr = JSON.stingify(params || {});

  if (className && methodName){
    bridgeStr = `jsbridge://${className}:${callback}/${methodName}?${paramsStr}`;
    try {
      sendToNative(bridgeStr);
    } catch(e) {
      console.log(e);
    }
  } else {
    console.log('Invalid className or methodName');
  }
}

function setToNative(url, data){
  window.prompt(url, JSON.stringify(data || {}));
}
```

### 离线缓存与更新

#### ServiceWorker

使用 ServiceWorker 作离线缓存可参考 Service Worker —— 这应该是一个挺全面的整理，ServiceWorker 有兼容性问题。

#### localeStorage

使用 localeStorage 作离线缓存就是带版本号形式缓存静态资源、页面内容、响应。典型如下：

```javascript
const newVersion = document.getElementById('versionStore').getAttribute('data-version');
const oldVersion = localeStorage.getItem('version');

if (newVersion > (oldVersion || 0)){
  loadScript(`scriptpath/${newVersion}.js`).then(content => {
    localeStorage.setItem('scriptpath', content);
  })
} else {
  const script = document.createElement('script');
  script.innerHtml = localeStorage.getItem('scriptpath');
  document.appendChild(script);
}
```

使用 localeStorage 有以下缺点：大小有限制（同域一般认为 5M 以内）；用户手动清空会使 localeStorage 失效；读取 localeStorage 较慢。

#### 文件增量更新

文件增量更新指的是：客户端通过 localeStorage 获取本地缓存资源的版本号，与 html 页面中最新版本号对比，如果本地版本号较小，则加载增量的静态资源，如本地为 1.1.js，远程最新版本为 1.4.js，则增量获取 1.1-1.4.js。有两种机制可以实现文件的增量更新：基于代码分块、基于编辑距离。基于代码分块的思路是，以增量描述说明代码块的变动内容，新增、删除、修改、保持原样；然后根据原文件和增量描述产生新文件。基于编辑距离（Levenshtein Distance）是计算从一个字符串变更到另一个字符串的最少操作步骤，适合于少量字符串变更的文件内容。

使用文件增量更新机制，有必要埋点统计不同版本号的用户覆盖率。

#### native 离线资源

当用户访问页面，native 首先会检查离线资源包中是否存在本地目录中的文件，然后将其与线上资源对比：如果线上有最新资源，则拉取线上资源并缓存，没有，就使用本地资源。这样当再次访问页面时，WebView 就可以读取本地资源了。

### 支付宝 jssdk

[alipayjsapi](https://gw.alipayobjects.com/as/g/h5-lib/alipayjsapi/3.1.1/alipayjsapi.js) 首先添加 es6-promise 垫片。

#### 核心流程

alipayjsapi 通过 addJavascriptInterface 为 window 对象注入 AlipayJSBridge 对象，html5 页面中即可以用如 AlipayJSBridge.call(‘alert’,{message: 12345}) 形式唤起 native 能力。alipayjsapi 另外构造了一个 _JSAPI 对象。_JSAPI 约定了各方法是怎样处理入参、唤起 native 能力、处理出参等的。下面以 redirectTo 说明 _JSAPI 对象的构造：

```javascript
var _JSAPI = {
  compressImage: {
    b: function b(opt) {
      opt.level = __isUndefined(opt.level) ? 4 : opt.level;
      // _mapping 将 opt._ 更名为 opt.apFilePaths
      return _mapping(opt, {
        _: 'apFilePaths',
        level: 'compressLevel%d'
      });
    },
    d: function d(_opt, cb) {
      if (__isAndroid()) {
        _JS_BRIDGE.call('compressImage', _opt, cb);
      } else {
        // _fakeCallBack 使用定时器直接调用 cb
        _fakeCallBack(cb, {
          apFilePaths: _opt.apFilePaths || []
        });
      }
    }
  }
}
```

在 _JSAPI 对象中，每个 api 可能包含以下方法或属性：m，即 mapping 的缩写，指定 native 端实际的接口名；b，即 before 的缩写，前置处理函数，用于转化入参；d，即 doing 的缩写，指定实际的执行流程（在不指定的情况下，将使用 AlipayJSBridge.call 的形式唤起 native 能力）；a，即 after 的缩写，后置处理函数，用于转化出参；e 或者 extra，指定 handleEventData 等扩展字段。在上面一段代码中，compressImage.b 就在转换选项，compressImage.d 就在执行图片压缩操作或不作任何处理。

在 _JSAPI 的基础上，alipayjsapi 的核心流程即使用 _JSAPI 处理出入参数，再使用 AlipayJSBridge.call 唤起 native 能力；偶然绑定事件。

```javascript
var AP = {
  call: function call() {
    var args = __argumentsToArg(arguments);
    if (__isSupportPromise()) {
      return AP.ready().then(function () {
        return new Promise(realCall);
      });
    } else {
      //如果直接加到 ready 事件里会有不触发调用的情况
      //AP.ready(realCall);

      if (_isBridgeReady()) {
        realCall();
      } else {
        //保存在待执行队列
        _WAITING_QUEUE.push(args);
      }
    }

    function realCall(resolve, reject) {
      var apiName;
      var opt; //原始 option
      var cb; //原始 callback
      var _opt; //处理过的 option
      var _cbSFC; //不同状态回调
      var _cb; //处理过的 callback
      var onEvt;
      var offEvt;
      var doingFn;
      var logOpt;
      //强制转为 name + object + function 形式的入参
      apiName = args[0] + '';
      opt = args[1];
      cb = args[2];
      //处理 cb 和 opt 的顺序
      if (__isUndefined(cb) && __isFunction(opt)) {
        cb = opt;
        opt = {};
      }
      //接口有非对象入参，设为快捷入参
      if (!__isObject(opt) && args.length >= 2) {
        //before、doing、after 方法中直接取 opt._ 作为参数
        opt = {
          _: opt
        };
      }
      //兜底
      if (__isUndefined(opt)) {
        opt = {};
      }

      // 处理入参，使用 _JSAPI[apiName].b 处理 opt
      _opt = _getApiOption(apiName, opt, cb);

      // 获取回调，_opt 以 success、fail、complete 属性约定回调
      _cbSFC = _getApiCallBacks(apiName, _opt);

      if (__isUndefined(_opt)) {
        console.error('please confirm ' + apiName + '.before() returns the options.');
      }

      // 获取 _JSAPI[apiName].d 方法
      doingFn = _getApiDoing(apiName);

      // 输出入参
      logOpt = __hasOwnProperty(opt, '_') ? opt._ : opt;

      // AP.debug 置为真值时，在控制台打印信息
      _apiLog(apiName, logOpt, _opt);

      // 是否是事件监听，apiName 以 on 起始
      onEvt = _getApiOnEvent(apiName);
      // 是否是事件移除，apiName 以 off 起始
      offEvt = _getApiOffEvent(apiName);

      // 处理回调
      _cb = function _cb(res) {
        var _res = void 0;
        res = res || {};

        if (onEvt && _getApiExtra(apiName, 'handleEventData') !== false) {
          _res = _handleEventData(res);
        }

        // 使用 _JSAPI[apiName].a 处理结果
        _res = _getApiResult(apiName, _res || res, _opt, opt, cb);

        if (__isUndefined(_res)) {
          console.error('please confirm ' + apiName + '.after() returns the result.');
        }

        // 处理错误码及相应，即处理 _res.error 及 res
        _res = _handleApiError(apiName, _res);

        // 打印 debug 日志
        _apiLog(apiName, logOpt, _opt, res, _res);

        if (__hasOwnProperty(_res, 'error') || __hasOwnProperty(_res, 'errorMessage')) {
          if (__isFunction(reject)) {
            reject(_res);
          }
          if (__isFunction(_cbSFC.fail)) {
            _cbSFC.fail(_res);
          }
        } else {
          if (__isFunction(resolve)) {
            resolve(_res);
          }
          if (__isFunction(_cbSFC.success)) {
            _cbSFC.success(_res);
          }
        }
        if (__isFunction(_cbSFC.complete)) {
          _cbSFC.complete(_res);
        }

        // 执行用户的回调
        if (__isFunction(cb)) {
          cb(_res);
        }
      };

      // 如果存在 d 直接执行，否则执行 AlipayJSBridge.call
      if (__isFunction(doingFn)) {
        doingFn(_opt, _cb, opt, cb);
      } else if (onEvt) {
        // 将事件、处理函数等存入 _CACHE.EVENTS
        _cacheEventHandler(onEvt, cb, _cb, _cbSFC);
        // 使用 document.addEventListener 绑定事件
        AP.on(onEvt, _cb);
      } else if (offEvt) {
        _removeEventHandler(offEvt, cb);
      } else {
        // 执行 AlipayJSBridge.call
        _JS_BRIDGE.call(_getApiName(apiName), _opt, _cb);
      }

      // 埋点，发送日志到远程服务器
      _apiRemoteLog(apiName);
    }
  },
}
```

#### 埋点

alipayjsapi 先收集调用的 api，当收集的 api 到 6 个时，再使用定时器上报。alipayjsapi 兜底使用返回按钮的 back 事件上报日志。

```javascript
var _apiRemoteLog = function () {
  var apiInvokeQueue = [];
  var timerId = void 0;
  var isTimerActived = false;
  //发送日志
  function triggerSendLog() {
    setTimeout(function () {
      if (apiInvokeQueue.length > 0) {
        var param1 = apiInvokeQueue.join('|');
        AP.ready(function () {
          _JS_BRIDGE.call('remoteLog', {
            type: 'monitor',
            bizType: 'ALIPAYJSAPI',
            logLevel: 1, // 1 - high, 2 - medium, 3 - low
            actionId: 'MonitorReport',
            seedId: 'ALIPAYJSAPI_INVOKE_COUNTER',
            param1: param1
          });
        });
        AP.debug && console.info('REMOTE_LOG_QUEUE>', apiInvokeQueue);
        apiInvokeQueue = [];
      }
      // 停止计时器
      clearTimer();
    }, 0);
  }

  // 计时器
  function timer() {
    // 计时激活标致
    isTimerActived = true;
    // 启动计时器
    timerId = setTimeout(function () {
      // 日志发送
      triggerSendLog();
    }, 5000); // 5 秒上报
  }

  // 清除计时器
  function clearTimer() {
    !__isUndefined(timerId) && clearTimeout(timerId);
    isTimerActived = false;
  }

  // back 事件上报日志，作为兜底
  AP.on('back', function () {
    triggerSendLog();
  });

  return function (apiName) {
    apiInvokeQueue.push(apiName);
    // 6 个上报
    if (apiInvokeQueue.length >= 6) {
      triggerSendLog();
    } else if (!isTimerActived) {
      timer();
    }
  };
}();
```