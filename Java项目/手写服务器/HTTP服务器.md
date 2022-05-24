##### websocket和http的区别

相同：websocket和http一样都是应用层的，基于TCP协议

不同：

	1. WebSocket是双向通信协议，模拟Socket协议，可以双向发送或接受信息。HTTP是单向的。
	1. WebSocket是需要HTTP请求握手进行建立连接的，建立之后，在真正传输时候是不需要HTTP协议的
	1. WebSocket通常运用在即时通讯，替换轮询，而HTTP一般用在网页文本传输。





