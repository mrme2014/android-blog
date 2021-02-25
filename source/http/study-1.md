---
title: Android网络编程核心技术概述
---

## 目录

## Android网络编程核心技术概述

<center><img src="/imgs/http/Http协议进化史.png" alt="http_protocal_1" style="zoom:50%;" /></center>

### HTTP 报文格式



- 请求报文

<center><img src="/imgs/http/http_protocal_1.png" alt="http_protocal_1" style="zoom:50%;" /></center>



- 响应报文

<center><img src="/imgs/http/http_protocal_2.png" alt="http_protocal_2" style="zoom:50%;" /></center>

- 连接无法复用：HTTP是建立在TCP传输层协议之上的，1.0版本TCP通道无法复用，延迟高，带宽利用率低。

  

### HTTP1.1

- HTTP1.1的优化

  1.长连接：引入`Connection:keep-alive`头支持TCP连接复用,在一定程度上提高了网络响应速度。

  2.缓存控制：引入Entity tag，If-Unmodified-Since, If-Match,cache-control。提供更多缓存数据控制策略。

  3.带宽优化：引入`range`头域，它允许只请求资源的某个部分，文件断点续传基础

- HTTP1.1现存问题

  1.数据传输时每次都需要重新建立连接

  2.所有传输的内容都是明文，无法验证对方的身份，无法保证数据的安全性。

  3.header里携带的内容过大，在一定程度上增加了传输的成本。

  <br/>

### HTTPS 

- HTTPS网络模型

<center><img src="/imgs/http/WechatIMG12.png" alt="WechatIMG12" style="zoom:50%;" /></center>

- HTTPS加解密过程

<center><img src="/imgs/http/WechatIMG4.png" alt="WechatIMG4" style="zoom:50%;" /></center>

<br/>

### SPDY 

- 多路复用TCP通道,降低HTTP的高延时
- 允许请求设置优先级
- header数据压缩
- 基于SSL的安全传输

<center><img src="/imgs/http/WechatIMG5.png" alt="WechatIMG5" style="zoom:40%;" /></center>

<br/>

### HTTP2.0

- HTTP2.0对数据报文重新定义了二进制格式

- TCP通道多路复用

- HTTP2.0支持明文传输，SPDY强制使用SSL/TLS

- HTTP2.0采用HPACK专有算法压缩消息头

  <center><img src="/imgs/http/http2.0.png" alt="http2.0" style="zoom:50%;" width="1600"/></center>



### HTTP3.0

- 减少了TCP三次握手及TLS握手时间
- 优化重传策略
- 流量控制

<center><img src="/imgs/http/http3.0.png" alt="http3.0" style="zoom:50%;" /></center>
