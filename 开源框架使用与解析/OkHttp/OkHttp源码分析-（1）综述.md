[TOC]


#### 1.OkHttp的使用

- [OkHttp使用教程](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0106/2275.html)
- [OkHttp使用进阶 译自OkHttp Github官方教程](http://www.cnblogs.com/ct2011/p/3997368.html)
- [ Android OkHttp完全解析 是时候来了解OkHttp了](http://blog.csdn.net/lmj623565791/article/details/47911083)

----

- [OkHttp Github](https://github.com/square/okhttp)
- [OkHttp Wiki](https://github.com/square/okhttp/wiki/Calls)


#### 2.OkHttp优点

根据[http://square.github.io/okhttp/](http://square.github.io/okhttp/)所述，OkHttp是一个高效率的HTTP客户端。

- 支持HTTP/2，通过HTTP/2协议允许一个Host的所有请求共享一个Socket
- 复用连接池，减少请求延迟
- 透明的GZIP压缩，减少下载大小
- 响应缓存
- 当网络请求出现问题，OkHttp会尽力去解决，比如服务器拥有多个ip，OkHttp会在第一次连接失败后尝试下一次请求。
- 支持SPDY

#### 2.OkHttp源码解析

- [OKHttp源码解析](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0326/2643.html)
- [OkHttp3源码分析[综述]](https://www.jianshu.com/p/aad5aacd79bf)
- [Android网络编程（七）源码解析OkHttp前篇[请求网络]](http://liuwangshu.cn/application/network/7-okhttp3-sourcecode.html)