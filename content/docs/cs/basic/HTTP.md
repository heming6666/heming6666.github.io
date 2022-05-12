---
title: HTTP/HTTPS
---
# HTTP/HTTPS

## 一、状态码
### 1、有哪些常见的状态码？
* 1xx - 正在处理
    1. 100 - 正常
	2. 101 - 切换请求协议
* 2- 成功
	1. 200 - 请求成功
	2. 201 - 已创建
	3. 202 - 已接受
	4. 204 - 无内容
* 3- 重定向
	1. 301 - 永久性重定向
	2. 302 - 临时重定向
* 4- 服务端无法处理请求（客户端错误
	1. 400 - Bad Request 语法错误
	2. 401 - Unauthorized
	3. 403 - Forbidden 拒绝
	4. 404 - not found
	5. 405 - method not allowd
	6. 406 - Not Acceptable
	7. 408 - Request Time-out
* 5- 服务端处理请求出错（服务端错误
	1. 500 - Internal Server Error 内部错误
	2. 501 - Not Implemented
	3. 502 - Bad Gateway 服务器无效响无效响应
	4. 503 - 服务器过载，稍后重试
	5. 504 - Gateway Timeout 请求超时，nginx 配置不对

### 2、301和302的区别是什么？
* 301是页面或资源永久性地移到了另一个位置。应用场景：网站移到了新的地址 / 或多个域名跳转到同一个域名，有利于URL权重的集中。
* 302是暂时性转移，常被用作网址劫持，搜索引擎会抓新的内容，但保存旧的网址。

## 二、HTTP 报文

1. http请求报文
	1. 请求行：包括请求方法（GET\POST等）、请求地址URL、协议版本（比如HTTP1.1）
	2. 请求头(Request Header) ：connection(比如keep-alive)、content-length长度、origin、user-agent、content-type(POST就有三种)、encoding、language、cookie
	3. 请求正文

2. http响应报文
	1. 状态行 : 包括状态码、协议版本
	2. 响应头：
	3. 响应正文

### 0、POST 请求体都有哪些形式？
1. application/x-www-form-urlencoded： 默认的，key1=val&key2=val2这种，和GET常见的，数据量不大。
2. multipart/form-data：上传文件
3. application/json：数据结构比较复杂时

### 1、HTTPS1、HTTP 和 HTTPS 的区别？
1. 加密：http 请求是明文传输，容易被窃听或截取，https在http的基础上加了SSL加密层
2. 证书：https需要ca证书
3. 端口：80 VS 443

### 2、HTTPS的加密流程
![HTTPS的加密流程](/images/basic-http.png)
### 3、HTTP 1.0、HTTP 1.1及HTTP 2.0的主要区别是什么
1. 1.0 默认是短连接。
2. 1.1 默认长连接，TCP默认不关闭，复用TCP连接
3. 2.0 多路复用

1.1 和 2 都是基于 TCP ，而 HTTP/3 基于 UDP 。

### 4、Cookie与Session

1. 概念
	* cookie: 服务端发送给客户端的一串标识，浏览器下次发送请求时会一起带上发送给服务端。可以用于用户登录、设置等场景。
	* session：表示服务端和客户端一次会话的过程。

2. 区别
3. 实际用法
可以同时配合使用，通过 SessionID 。
4. token
Token 也是服务端生成的一串标识字符串，比如当用户第一次登录后，服务器根据提交的用户信息生成一个 Token，然后返回给客户端，客户端下次请求时带上这个 Token ，就可以验证身份。
5. 分布式session
	1. 用Nginx做代理，确保同一个IP固定访问某台机器
	2. Session 复制
	3. Session 共享，缓存/中间件

### 7、在浏览器中输入url到显示主页的过程
DNS、TCP、HTTP、TCP

1. DNS解析：
	1. 查浏览器的DNS缓存
	2. 查系统的缓存
	3. 读本地的Host文件
2. TCP三次握手
3. 发起HTTP请求
4. 响应HTTP报文
5. 浏览器解析、渲染页面
6. 四次挥手