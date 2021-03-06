# http

## 状态码

1xx：指示信息--表示请求已接收，继续处理
2xx：成功--表示请求已被成功接收、理解、接受
3xx：重定向--要完成请求必须进行更进一步的操作
4xx：客户端错误--请求有语法错误或请求无法实现
5xx：服务器端错误--服务器未能实现合法的请求

## http常见请求头

### 跨域

- Access-Control-Allow-Origin: *
  - 允许所有来源访问
- Access-Control-Allow-Method: POST,GET
  - 允许访问的方式
- Access-Control-Allow-Headers: Content-type, Authorization, Accept
  - 允许携带的头部
- Access-Control-Allow-Credentials: true
  - 该字段可选。它的值是一个布尔值，表示是否允许发送Cookie。默认情况下，Cookie不包括在CORS请求之中。设为`true`，即表示服务器明确许可，Cookie可以包含在请求中，一起发给服务器。这个值也只能设为`true`，如果服务器不要浏览器发送Cookie，删除该字段即可。
- Access-Control-Max-Age: 5
  - 该字段可选，用来指定本次预检请求的有效期，单位为秒。上面结果中，有效期是5s，即允许缓存该条回应5秒，在此期间，不用发出另一条预检请求。

> 注：以上头部都是响应头部

### 缓存

- expires
- cache-control

- If-Modified-Since
  - 这个请求头后面跟的是一个时间戳，表示在这个时间点以后，资源被修改过，则重新请求，这个时间点会在首次请求资源的响应头中携带
- If-None-Match
  - 这个请求头的值是一个Etag，在第一次请求资源时会在响应头中携带

## http缓存机制

### 强缓存

只要请求了一次，在有效时间内，不会再请求服务器（请求都不会发起），直接从浏览器本地缓存中获取资源。

主要字段有

- expires：date（过期日期）
- cache-control： max-age=time（毫秒数，多久之后过期）|no-cache|no-store

如果expires和cache-control同时存在，cache-control会覆盖expires。建议两个都写，cache-control是http1.1的头字段，expires是http1.0的头字段，都写兼容会好点。

### 协商缓存

无论是否变化，是否过期都会发起请求，如果内容没过期，直接返回304，从浏览器缓存中拉取文件，否则直接返回更新后的内容

### 优先级问题

- cache-control和expires
  - **cache-control**胜出
- If-Modified-Since和If-None-Match
  - **If-None-Match**胜出，或者说浏览器会只匹配etag值
  - 这里有点迷，有一种说法说RFC规定两者之间没有优先级，如果都有的话需要都发给服务器，没有优先级之分

## http各版本

http协议目前有三个版本

- http1.0
- http1.1
- http2

### HTTP1.0和HTTP1.1

**HTTP1.0**：默认是短连接

> 什么是短连接：简单来说就是，**每次与服务器交互，都需要新开一个连接**，试想一下这种场景：请求一张图片，新开一个连接，请求一个CSS文件，新开一个连接，请求一个JS文件，新开一个连接。HTTP协议是基于TCP的，TCP每次都要经过**三次握手，四次挥手，慢启动**...这都需要去消耗我们非常多的资源的！
>
> **HTTP 1.0规定浏览器与服务器只保持短暂的连接，浏览器的每次请求都需要与服务器建立一个TCP连接，服务器完成请求处理后立即断开TCP连接，服务器不跟踪每个客户也不记录过去的请求。**

**HTTP1.1**：默认是持久化连接

>持久化连接：**建立一次连接，多次请求均由这个连接完成**！(如果阻塞了，还是会开新的TCP连接的)，相关字段：keep-alive

以上是http1.0和1.1的最显著区别，除此之外，HTTP1.1还加入了其他特性

1. 请求的流水线(`Pipelining`)处理
2. 增加缓存处理（新的字段如cache-control）
3. 增加Host字段、支持断点传输等

### HTTP1.1的Pipelining理论

**请求的流水线（Pipelining）处理，在一个TCP连接上可以传送多个HTTP请求和响应，减少了建立和关闭连接的消耗和延迟。**例如：一个包含有许多图像的网页文件的多个请求和应答可以在一个连接中传输，但每个单独的网页文件的请求和应答仍然需要使用各自的连接。**HTTP 1.1还允许客户端不用等待上一次请求结果返回，就可以发出下一次请求，但服务器端必须按照接收到客户端请求的先后顺序依次回送响应结果，以保证客户端能够区分出每次请求的响应内容。**

### HTTP1.1和HTTP2.0

#### HTTP/1.1存在的缺陷

1、**TCP连接数限制**

对于同一个域名，浏览器最多只能同时创建 6~8 个 TCP 连接 (不同浏览器不一样)。为了解决数量限制，出现了 `域名分片` 技术，其实就是资源分域，将资源放在不同域名下 (比如二级子域名下)，这样就可以针对不同域名创建连接并请求，以一种讨巧的方式突破限制，但是滥用此技术也会造成很多问题，比如每个 TCP 连接本身需要经过 DNS 查询、三步握手、慢启动等，还占用额外的 CPU 和内存，对于服务器来说过多连接也容易造成网络拥挤、交通阻塞等

2、**线头阻塞（Head Of Line Blocking)问题**

每个 TCP 连接同时只能处理一个请求 - 响应，浏览器按 FIFO 原则处理请求，如果上一个响应没返回，后续请求 - 响应都会受阻。为了解决此问题，出现了 **管线化 - pipelining 技术**，但是管线化存在诸多问题，比如第一个响应慢还是会阻塞后续响应、服务器为了按序返回响应需要缓存多个响应占用更多资源、浏览器中途断连重试服务器可能得重新处理多个请求、还有必须客户端 - 代理 - 服务器都支持管线化

#### HTTP/2.0优势

·1、二进制分帧

帧是数据传输的最小单位，以二进制传输代替原本的明文传输，原本的报文消息被划分为更小的数据帧，帧代表着最小的数据单位，每个帧会标识出该帧属于哪个流，流也就是多个帧组成的数据流。

2、**多路复用**

在一个 TCP 连接上，我们可以向对方不断发送帧，每帧的 stream identifier 的标明这一帧属于哪个流，然后在对方接收时，根据 stream identifier 拼接每个流的所有帧组成一整块数据。
把 HTTP/1.1 每个请求都当作一个流，那么多个请求变成多个流，请求响应数据分成多个帧，不同流中的帧交错地发送给对方，这就是 HTTP/2 中的多路复用。

3、服务端推送

浏览器发送一个请求，服务器主动向浏览器推送与这个请求相关的资源，这样浏览器就不用发起后续请求。

Server-Push 主要是针对资源内联做出的优化，相较于 http/1.1 资源内联的优势:

- 客户端可以缓存推送的资源
- 客户端可以拒收推送过来的资源
- 推送资源可以由不同页面共享
- 服务器可以按照优先级推送资源

4、Header 压缩 (HPACK)

5、应用层的重置连接

对于 HTTP/1 来说，是通过设置 tcp segment 里的 reset flag 来通知对端关闭连接的。这种方式会直接断开连接，下次再发请求就必须重新建立连接。HTTP/2 引入 RST_STREAM 类型的 frame，可以在不断开连接的前提下取消某个 request 的 stream，表现更好。

6、请求优先级设置

HTTP/2.0 里的每个 stream 都可以设置依赖 (Dependency) 和权重，可以按依赖树分配优先级，解决了关键请求被阻塞的问题

7、流量控制

每个 http/2.0 流都拥有自己的公示的流量窗口，它可以限制另一端发送数据。对于每个流来说，两端都必须告诉对方自己还有足够的空间来处理新的数据，而在该窗口被扩大前，另一端只被允许发送这么多数据。

## https

概念：是以安全为目标的HTTP通道，简单讲是HTTP的安全版，即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。

### HTTP缺点

1. 通信使用明文，内容可能会被窃听
2. 不验证通信双方的身份，通信双方可能被伪装
3. 无法证明报文的完整性，所以容易被篡改

解决方案：

1. 加密防止窃听
2. 通信之前验证证书
3. MD5，SHA-1校验

### HTTP+加密+认证+完整性保护=HTTPS

HTTP+SSL(TLS)=HTTPS

- 加密方式
  - HTTP采用对称秘钥和非对称秘钥的混合加密方式
- 证书
  - 证书用来确保对称秘钥的真实性，也可确认通信方的真实身份。