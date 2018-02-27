# remote-debug
调试是开发过程很重要的过程，而随着移动端的普及，移动开发也越来越多，并且由于移动端的诸多限制，使得调试相对PC复杂很多。因此远程调试就显得非常重要了。
近几年，浏览器厂商也纷纷推出自己的远程调试工具，比如Opera Mobile 推出的Opera Dragonfly，iOS Safari 可以开启Web检查器在 Mac OS X系统中实现远程调试。Android 4.0+系统的 Chrome for Android可以配合 ADB（Android Debug Bridge）实现桌面远程调试，桌面版Chrome 32+已经支持免安装ADB即可实现远程调试移动设备页面/WebView 。国内的UC浏览器开发者版也推出了自己的远程调试工具RemoteInspector。除了浏览器厂商之外，也涌现出许多第三方开发的远程调试工具，诸如支持全平台调试的Weinre等。

本文主要介绍远程调试是什么，基本原理是什么，有哪些情况，以及如何根据不同的情况选择恰当优雅的调试方式。

## 本地调试
远程调试是相对于本地调试来讲的，那么理解本地调试对于理解远程调试是很重要的。
本地调试指的是直接调试运行在本地的APP。常见的就是调试本地PC（很简单）。
比如我在本地运行了一个webpack-dev-server，端口号为8080.
那么访问8080，并且打开浏览器的开发者工具，就可以在本地进行调试了。再比如我要调试google的官网，那么我只需要
访问www.google.com， 同样打开开发者工具就可以进行本地调试了。
## 远程调试
那么远程调试就是调试运行在远程的APP。比如手机上访问google，我需要在PC上调试手机上运行的google APP。
这个就叫做远程调试。

远程调试大概有三种类型：

* 调试远程PC（本质上是一个debug server 和 一个debug target，其实下面两种也是这种模型，ios中间会多一个协议转化而已）
这种类型下的debug target就是pc, debug server 也是pc。

* 调试android webpage/webview（很多方式，但安卓4.4以后本质都是Chrome DevTools Protocol的扩展）
这种类型下的debug target就是android webview，debug server 是pc。

* 调试ios webpag/webview（可以使用iOS WebKit Debug Proxy代理，然后问题便退化成上述两种场景）
这种类型下的debug target就是ios webview， debug server 是pc。

### chrome远程调试
提到chrome的远程调试，就不得不提[chrome remote debug protocol](https://developer.chrome.com/devtools/docs/debugger-protocol)。
它采用websocket来与页面建立通信通道，由发送给页面的command和data组成。chrome的开发者工具是这个协议主要的使用者，第三方开发者也可以调用这个协议来与页面交互调试。简而言之，有了它我们就可以和chrome中的页面进行双向通信。

chrome 启动的时候，默认是关闭了调试端口的，如果要对一个chrome PC 浏览器进行调试，那么启动的时候，可以通过传递参数来开启 Chrome 的调试开关：

```
sudo /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222
```

这个时候就可以通过http://127.0.0.1:9222 查看所有远程调试目标

http://127.0.0.1:9222/json 可以查看特定的远程调试目标信息，类型为json数组。
```json
[
  {
    "description": "",
    "devtoolsFrontendUrl": "/devtools/inspector.html?ws=127.0.0.1:9222/devtools/page/fefa.....-ffa",
    "id": "fefa.....-ffa",
    "title": "test",
    "type": "page",
    "url": "http://127.0.0.1:51004/view/:id",
    "webSocketDebuggerUrl": "ws://127.0.0.1:9222/devtools/page/fefa.....-ffa"
  }
]
```

其中id是一个唯一的标示，[chrome dev protocol](https://chromedevtools.github.io/devtools-protocol/)基本都依赖这个id。

比如关闭一个页面：

```json
http://localhost:9222/json/close/477810FF-323E-44C5-997C-89B7FAC7B158
```

再比如激活一个页面：

```json
http://localhost:9222/json/activate/477810FF-323E-44C5-997C-89B7FAC7B158
```
具体可以查看官方信息。

webSocketDebuggerUrl是调试页面需要用到的WebSocket连接的地址。

比如我需要清空浏览器缓存，就用websocket连接到该页面之后调用send方法
```js
const ws = new WebSocket('ws://127.0.0.1:9222/devtools/page/fefa.....-ffa')
ws.send('{"id": 1, "method": "Network.clearBrowserCache", "params": {}}')
```

还有很多类似的api，因此就可以构造复杂的[扩展](https://developer.chrome.com/extensions/samples)
## 常见的远程调试框架对比
明白了远程调试的类型，那么对于不同的类型应该采取什么样的手段是我们最为关心的问题。
在回答这个问题之前，我们先来看下市面上的远程调试框架，他们做了什么事情，解决了什么问题。

如下是我对比较常见的远程调试框架的简单对比。

![remote-debug](https://user-gold-cdn.xitu.io/2018/2/25/161cab2a2b8cc4cf?w=878&h=569&f=png&s=93455)

后面虚线里面的是除了抓包功能之外调试框架，可以看出灰色部分是他们不支持的。
这时候就需要专门的抓包工具来代替。通常来说专门的抓包工具功能包括但不限于请求拦截和修改，https支持，重放和构造请求，(web)socket。

抓包工具的原理非常简单，本质上它就是一个正向代理，所有的请求经过它，就可以将其记录下来，甚至可以重放构造请求等。

对于抓包工具，步骤基本就是三部曲。

第一步：手机和PC保持在同一网络下（比如同时连到一个Wi-Fi下）

第二步：设置手机的HTTP代理，代理IP地址设置为PC的IP地址，端口为代理的启动端口。

Android设置代理步骤：设置 - WLAN - 长按选中网络 - 修改网络 - 高级 - 代理设置 - 手动
iOS设置代理步骤：设置 - 无线局域网 - 选中网络 - HTTP代理手动

对于https请求需要多一步：

第三步：手机安装证书。

> 对于第二步，我们可以通过连接USB的方式，然后设置端口转发和虚拟主机映射，建立tcp连接，这样就ip就可以设置为localhost，以后ip变动，代理设置也不必变动，但是却需要连接USB，可谓各有千秋。
## 调试的辅助手段
通过上面的方式我们已经建立了调试的准备环境。但是真正调试应用，发现问题，解决问题。还需要
其他信息来辅助。下面来讲解一些调试的辅助手段。

### 用户轨迹

有时候我们需要知道用户的浏览轨迹，从而方便定位问题。
浏览轨迹的粒度可以自己决定，可以是组件级，也可以是页面级。
### 应用数据

获取足够的信息对于调试是非常重要的。尤其是数据驱动（data driven）的应用，
知道了数据，基本上就可以还原现场，定位问题。

比如我们使用vuex或者redux这样的状态管理框架，一种方式是将中央的store挂载到window上。
这样我们就可以通过访问window的属性获取到全局的store。
如果使用其他的状态管理框架或者自制的状态管理，也可以采取类似的方式。

然而我们也可以使用现成的工具，比如使用react-dev-tools调试react应用。
比如使用redux-dev-tools调试使用redux的应用等等。

### 其他调试信息

比如用户的id，客户端数据，登陆的session信息等等对于调试有所帮助的，都可以将其收集起来。

## 参考
https://www.jianshu.com/p/19c18c924f91

https://aotu.io/notes/2017/02/24/Mobile-debug/index.html
