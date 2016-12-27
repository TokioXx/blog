---
title: 从你输入URL到看到页面发生了什么
tag:
    - http
    - web
    - html
---
最近刚好在面试，被问到这个问题，因此在此记录下来。

### 前提

如果深入细致的考虑所有细节，那估计可以写好几本书了。所以在问题开始之前，我们先对问题做一些简单的约束，包括以下几个方面：

1. 不考虑硬件层面，不深入讨论底层协议
2. URL是一个合法的[统一资源定位符][]，且协议为http或https
3. 网络连接是正常的



### 分析

##### 1. 获取IP和端口

端口一般情况下为80或者443，也可以在URL里现实指定。IP获取一般通过以下几种方式：

- URL中的服务器就是IP
- 在操作系统的host中指定了服务器对应的IP
- 通过DNS获取

##### 2. 建立TCP连接

TCP连接需要三次握手，如果协议是HTTPS，则还需要TLS层的四次握手

##### 3. 发起HTTP请求

目前HTTP主流有三种版本：1.0，1.1和2.0。如果浏览器支持高版本协议，会在请求头部包含 Upgrade 头，进行[HTTP协议协商][]。根据协商结构，响应处理会有所不同。

##### 4. 处理HTTP响应

浏览器根据协议(如HTTP2.0的二进制帧处理)、响应头(如GZIP压缩，编码)、响应状态码(如302重定向，404资源未找到)进行处理。

##### 5. 加载页面

1. 解析页面头部的CSS，下载外部CSS，不阻塞dom生成，最后生成css rule tree
2. 解析html生成dom
3. 解析页面底部的JS，下载外部JS，阻塞渲染树生成(使用async、defer属性的外部js略有不同)。
4. 通过dom和css rule tree生成渲染树，并计算每个节点的位置(layout和reflow)
5. 绘制到屏幕
6. 触发domcontentloaded事件

##### 6. 加载剩余资源

所有资源加载完毕后触发load事件

 

### 参考

- [统一资源定位符]: https://zh.wikipedia.org/wiki/%E7%BB%9F%E4%B8%80%E8%B5%84%E6%BA%90%E5%AE%9A%E4%BD%8D%E7%AC%A6

- [HTTP协议协商]: https://imququ.com/post/protocol-negotiation-in-http2.html

- [浏览器的渲染原理简介]: http://coolshell.cn/articles/9666.html
