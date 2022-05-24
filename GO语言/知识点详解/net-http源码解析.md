https://zhuanlan.zhihu.com/p/101995755

Go语言中的HTTP协议
------------------

实现的版本有HTTP1.1和HTTP2.0,HTTP/3 在 UDP 协议上实现了新的传输层协议 QUIC 并使用 QUIC 传输数据，这也意味着 HTTP 既可以跑在 TCP 上，也可以跑在 UDP 上。

### 层级结构

`net/http.RoundTripper`：表示执行 HTTP 请求的接口，调用方将请求作为参数可以获取请求对应的响应

`net/http.Handler` 主要用于 HTTP 服务器响应客户端的请求，客户端可以实现Handle接口，定义处理 HTTP 请求的逻辑。

`net/http.ResponseWriter`主要提供的三个接口 `Header`、`Write` 和 `WriteHeade`，用来构造 HTTP 响应。

### 客户端

客户端可以直接通过 `net/http.Get` 使用默认的客户端 `net/http.DefaultClient` 发起 HTTP 请求，也可以自己构建新的 `net/http.Client`实现自定义的 HTTP 事务.

1.  调用 `net/http.NewRequest` 根据方法名、URL 和请求体构建请求；
2.  调用 `net/http.Transport.RoundTrip` 开启 HTTP 事务、获取连接并发送请求；
3.  在 HTTP 持久连接的 `net/http.persistConn.readLoop` 方法中等待响应；

`net/http.Client`是 HTTP 客户端,默认`DefaultClient`

`net/http.Transport`是 [`net/http.RoundTripper`](https://draveness.me/golang/tree/net/http.RoundTripper) 接口的实现，它的主要作用就是支持 HTTP/HTTPS 请求和 HTTP 代理

`net/http.persistConn` 封装了一个 TCP 的持久连接，是我们与远程交换消息的句柄（Handle）
