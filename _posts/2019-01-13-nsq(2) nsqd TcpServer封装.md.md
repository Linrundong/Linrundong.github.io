[TOC]

### 启动流程
- 启动方式与nsqlookupd相同使用svg
- 创建两个协程分别监听HTTP和TCP端口

```
// nsqd.main()
func main() {
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
}
```

#### Main线程与nsqlookupd差异
- nsqlookupd的HTTP线程与消费者通讯, TCP端口作为Server与nsq 通讯(长连接)
- nsqd的TCP端口接收客户端(生产者)消息,HTTP端口提供HTTP API接收客户端(生产者)消息
- ==TODO 交互方式未知==
    
```
func (n *NSQD) Main() {
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
    
    // 超时消息检索和处理任务
    n.waitGroup.Wrap(n.queueScanLoop)
    n.waitGroup.Wrap(n.queueScanLoop)
    n.waitGroup.Wrap(n.lookupLoop)
    if n.getOpts().StatsdAddress != "" {
    	n.waitGroup.Wrap(n.statsdLoop)
    }
}
```

#### 可复用的Tcp Server模式
- TcpServer封装
- 循环阻塞accept()
- 当退出信号触发svg设置的channel，将关闭listener，从而将在Handle处理完毕之后才会退出
- 每来一个client都开启一个Handler协程
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

#### nsqd 对Client的处理
##### 通过对应client的Handle协程的IOLoop()、
```
func (p *protocolV2) IOLoop(conn net.Conn) error {
    // 创建新client处理类
	clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
	client := newClientV2(clientID, conn, p.ctx)
	p.ctx.nsqd.AddClient(client.ID, client)

	// messagePump协程完成初始化向chan发送消息
	// TODO messagePump()作用未知
	messagePumpStartedChan := make(chan bool)
	go p.messagePump(client, messagePumpStartedChan)
	<-messagePumpStartedChan

	for {
	    // 心跳如果有设置，则设置读超时为两倍时间
		if client.HeartbeatInterval > 0 {
			client.SetReadDeadline(time.Now().Add(client.HeartbeatInterval * 2))
		} else {
			client.SetReadDeadline(zeroTime)
		}

        // 分割命令
		line, err = client.Reader.ReadSlice('\n')
        // 处理win换行
		line = line[:len(line)-1]
		if len(line) > 0 && line[len(line)-1] == '\r' {
			line = line[:len(line)-1]
		}
		params := bytes.Split(line, separatorBytes)
        p.ctx.nsqd.logf(LOG_DEBUG, "PROTOCOL(V2): [%s] %s", client, params)

		var response []byte
		// Exec()处理实际参数
		response, err = p.Exec(client, params)

		if response != nil {
		    // 发送响应
			err = p.Send(client, frameTypeResponse, response)
		}
	}

	p.ctx.nsqd.logf(LOG_INFO, "PROTOCOL(V2): [%s] exiting ioloop", client)
	conn.Close()
	close(client.ExitChan)

    // 移除Client
	p.ctx.nsqd.RemoveClient(client.ID)
	return err
}
```
#### Send()发送响应
```
func (p *protocolV2) Send(client *clientV2, frameType int32, data []byte) error {
	client.writeLock.Lock()

	var zeroTime time.Time
	// 如果设置了心跳则设置写超时
	if client.HeartbeatInterval > 0 {
		client.SetWriteDeadline(time.Now().Add(client.HeartbeatInterval))
	} else {
		client.SetWriteDeadline(zeroTime)
	}

    // protocol.SendFramedResponse()
	SendFramedResponse := func(client.Writer, frameType, data){
    	beBuf := make([]byte, 4)
    	size := uint32(len(data)) + 4
        // 转换写入数据大小
    	binary.BigEndian.PutUint32(beBuf, size)
    	n, err := w.Write(beBuf)
    	if err != nil {
    		return n, err
    	}
        // 转换写入数据
    	binary.BigEndian.PutUint32(beBuf, uint32(frameType))
    	n, err = w.Write(beBuf)
    	if err != nil {
    		return n + 4, err
    	}
    
    	n, err = w.Write(data)
    	return n + 8, err
	}

	if frameType != frameTypeMessage {
		err = client.Flush()
	}

	client.writeLock.Unlock()

	return err
}
```
