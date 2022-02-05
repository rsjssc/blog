# 背景
一个线上运行的系统，使用websocket通信时，客户端偶尔会遇到`Could not decode a text frame as UTF-8`的错误

Google这个报错得到的结果很多都是说，由于协商的是Text，实际发的是Binary（WebSocket的数据帧一共有两种格式，
TextMessage和BinaryMessage），只要发送数据的时候指定正确的格式就不会报错了。但是经过下面的详细分析，发现是由于
WebSocket协议中的`压缩`引起的问题。

这个问题对应的服务端是如下版本
```text
org.apache.tomcat.embed:tomcat-embed-websocket:jar:9.0.46
org.springframework:spring-websocket:jar:5.3.7
```

## 复现尝试1：将发送的数据从TextMessage改成BinaryMessage
本地修改后浏览器中WS看到的返回值是`Binary Message`，并不是跟线上系统一样的报错。
经过后面的了解，发现tomcat中的websocket实现，会根据发送的是TextMessage还是BinaryMessage，
自动调整发送的websocket数据帧中的`opcode`字段，客户端收到数据帧后根据这个字段，便能正确识别`payload`部分是什么类型的数据。
所以只要tomcat实现的WebSocket没问题，就不会出现发送给的是TextMessage，却让客户端误认为是Binary Message的情况。

## 复现尝试2：将发送的文本数据从UTF-8编码改成UTF-16编码或者IOS-8859-1编码
WebSocket协议中，明确指出TextMessage使用UTF-8编码，尝试一下是不是用其他编码方式编码会造成跟线上系统一样的错误。
修改后，浏览器中WS看到的返回值是乱码，跟线上报错不一致，说明换了一种编码后，相应的码点依然存在字符，但是不是正确的字符

## 复现尝试3：在对应的WebSocketHandler的afterConnectionEstablished方法中直接抛出异常
浏览器中看到WS连接在建立之后立刻关闭了，没有任何的message传递

# 抓包分析
通过wireshark在用户侧抓包后，发现对于相同的内容，被浏览器正确和错误解析的websocket包，服务端返回的数据帧中的
`payload`的字节是不一样的， 经过对 [websocket](https://datatracker.ietf.org/doc/html/rfc6455) 协议的学习和
研究，发现websocket存在使用`deflate`压缩的扩展。将浏览器无法解析的数据帧的payload部分使用`inflate`解压，
发现就是能正确解析的数据，说明在浏览器无法解析的请求里，服务端使用了deflate压缩，但是浏览器没有使用`inflate`的方式解压

# 原因分析
那么客户端和服务器怎么能够知道是否应该对`payload`使用压缩或者解压呢，又是什么原因导致浏览器没有使用`inflate`解压呢
WebSocket协议中提到，在链接建立阶段（也称为HTTP协议 upgrade 到 WebSocket 协议阶段），服务端会根据自己支持的扩展
和客户端发过来请求头中的扩展Sec-WebSocket-Extensions，决定在本连接中使用使用哪些WebSocket扩展，协议中称之为 
[Negotiating Extensions](https://datatracker.ietf.org/doc/html/rfc6455#section-9.1)

经过对WebSocket建立链接过程中的断点分析，发现Tomcat的WebSocket实现的其中一个工厂类`TransformationFactory`中，
默认只支持一种扩展，就是`permessage-deflate`， 当客户端建立连接的请求header中包含
`Sec-WebSocket-Extensions: permessage-deflate; `的时候，
会在response header中返回`Sec-WebSocket-Extensions: permessage-deflate;`
这样后续客户端和服务端之间传输的数据帧的payload都会使用deflate算法进行压缩

经过对线上系统的分析，发现建立链接是，request header总会发包含`Sec-WebSocket-Extensions: permessage-deflate; `的头，
但是response header中却永远没有这个头。

由于tomcat没有进行特殊的配置，例如配置WebSocket不支持deflate压缩等，所以理论上只要到达服务端的流量包含上述扩展协商的头，
服务端就一定会在response中增加该头，实际上没有，并且无法解析payload的错误是偶现的。说明肯定是链路中某个中间节点除了问题

## 分析步骤1
到服务器上使用tcpdump抓包，确认收到的WebSocket头是否包含`Sec-WebSocket-Extensions: permessage-deflate; `，
WebSocket的返回值是否是经过deflate压缩的结果。
经过抓包发现，服务端收到的建立连接的请求，有时候有这个头，有时候没有，并且有这个扩展头的请求中，服务端都对数据
进行了deflate压缩

说明服务端的实现逻辑没有问题，根据客户端请求中的`Sec-WebSocket-Extensions`决定是否压缩，并在握手请求的返回头中明确使用了压缩

## 分析步骤2
找到负责接入层中间件节点的同事，调查中间件节点是否对包内容进行了修改。
最终发现是接入层的header过滤器有一个bug，在过滤request header时，有一定几率会将 `Sec-WebSocket-Extensions: permessage-deflate; `
这个头去掉。导致服务端收到的WebSocket链接有时候使用了这个扩展，对payload进行了压缩，有时候没有。
同时在服务端能收到这个扩展头的时候，理论上response header里也有这个扩展头，用来告诉客户端这条WebSocket链接会使用这个扩展，
但是由于存在另外一个 response header过滤器，一定会将response中的`Sec-WebSocket-Extensions: permessage-deflate; `去掉，
导致客户端在握手响应中永远不会收到服务端表明自己使用了压缩的头

# 解决方式
在中间件中将这个header过滤器修复或者禁用

# 总结

这次的问题，对WebSocket协议进行了深入的学习，除了`Sec-WebSocket-Extensions`之外，还了解到WebSocket协议中的掩码功能，
在wss没有普及的时候，可以用于 [防止缓存勿扰攻击](https://www.infoq.cn/article/deep-in-websocket-protocol/)

