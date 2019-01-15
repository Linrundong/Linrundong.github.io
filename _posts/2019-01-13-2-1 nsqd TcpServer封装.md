[TOC]

### 启动流程
- 启动方式与nsqlookupd相同使用svg
- 创建两个协程分别监听HTTP和TCP端口

```
prg := &program{}
if err := svc.Run(prg, syscall.SIGINT, syscall.SIGTERM); err != nil {
	log.Fatal(err)
}

// 初始化
if err = prg.Init(service); err != nil {
	return err
}

// 启动业务进程
err := prg.Start()
if err != nil {
	return err
}

// 阻塞等待退出信号
signalChan := make(chan os.Signal, 1)
svg.signalNotify(signalChan, ws.signals...)
<-signalChan

// 退出
err = prg.Stop()
```

#### Main线程与nsqlookupd差异
    - nsqlookupd的HTTP线程与消费者通讯, TCP端口作为Server与nsq 通讯(长连接)
    - nsqd的TCP端口接收客户端(生产者)消息,HTTP端口提供HTTP API接收客户端(生产者)消息
    - ==TODO 交互方式未知==
    
```
tcpServer := &tcpServer{ctx: ctx}
n.waitGroup.Wrap(func() {
	// 协程连接客户端（生产者）
	protocol.TCPServer(n.tcpListener, tcpServer, n.logf)
})

// 监听端口提供HTTP API
httpServer := newHTTPServer(ctx, false, n.getOpts().TLSRequired == TLSRequired)
n.waitGroup.Wrap(func() {
	http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf)
})
if n.tlsConfig != nil && n.getOpts().HTTPSAddress != "" {
	httpsServer := newHTTPServer(ctx, true, true)
	n.waitGroup.Wrap(func() {
		http_api.Serve(n.httpsListener, httpsServer, "HTTPS", n.logf)
	})
}

// 退出等待
n.waitGroup.Wrap(n.queueScanLoop)
n.waitGroup.Wrap(n.lookupLoop)
if n.getOpts().StatsdAddress != "" {
	n.waitGroup.Wrap(n.statsdLoop)
}
```

#### 可复用的Tcp Server模式
- TcpServer封装
- 循环阻塞accept()
- 当退出信号触发svg设置的channel，将关闭listener，从而将在Handle处理完毕之后才会退出
- 
```
func TCPServer(listener net.Listener, handler TCPHandler, logf lg.AppLogFunc) {
	logf(lg.INFO, "TCP: listening on %s", listener.Addr())

	for {
		clientConn, err := listener.Accept()
		if err != nil {
			if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
				logf(lg.WARN, "temporary Accept() failure - %s", err)
				runtime.Gosched()
				continue
			}
			// theres no direct way to detect this error because it is not exposed
			if !strings.Contains(err.Error(), "use of closed network connection") {
				logf(lg.ERROR, "listener.Accept() - %s", err)
			}
			break
		}
		go handler.Handle(clientConn)
	}

	logf(lg.INFO, "TCP: closing %s", listener.Addr())
}
```
##### 数据读取，使用protocol拓展的模式
```
func (p *tcpServer) Handle(clientConn net.Conn) {
	p.ctx.nsqd.logf(LOG_INFO, "TCP: new client(%s)", clientConn.RemoteAddr())

	// 读取4字节的协议版本
	buf := make([]byte, 4)
	_, err := io.ReadFull(clientConn, buf)
    ...
	protocolMagic := string(buf)
    ...

	var prot protocol.Protocol
	switch protocolMagic {
	case "  V2":
	    // 用报文与nsqd本身信息初始化一个protocolV2
		prot = &protocolV2{ctx: p.ctx}
	default:
		clientConn.Close()
		...
		return
	}

    // protocolV2.IOLoop()来处理客户端的业务内容
	err = prot.IOLoop(clientConn)
    ...
}
```
