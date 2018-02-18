# remote-debug
调试是开发过程很重要的过程，而随着移动端的普及，移动开发也越来越多，并且由于移动端的诸多限制，使得调试相对PC复杂很多。因此远程调试就显得非常重要了。本文主要介绍远程调试是什么，基本原理是什么，有哪些情况，以及如何根据不同的情况选择恰当优雅的调试方式。

将spy-debugger加入到对比列表

调试：
1. 调试本地PC（很简单）
2. 调试远程PC（本质上是一个debug server 和 一个debug target，其实下面两种也是这种模型，ios中间会多一个协议转化而已）
3. 调试android webview（很多方式，但是本质都是Chrome DevTools Protocol的扩展）
4. 调试ios webview（iOS WebKit Debug Proxy）

参考：
https://www.jianshu.com/p/19c18c924f91
https://aotu.io/notes/2017/02/24/Mobile-debug/index.html
