[TOC]

### nsqd 对Client的处理
#### 通过对应protocol类的IOLoop()
- ==TODO messagePump()协程作用未知==
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
