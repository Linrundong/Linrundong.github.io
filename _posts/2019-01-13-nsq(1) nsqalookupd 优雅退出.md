[TOC]

> 利用SVC框架启动，当收到退出信号时，先等待任务处理完毕调用Stop方法等待业务处理完毕再退出

### 启动流程
- ==在程序生命周期里会有三个线程：Main线程，HTTP协程，TCP协程==<br>

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
// Start()内部会在创建两个监听端口的协程
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
#### 使用sync.WaitGroup加锁开启协程
- 加锁可保证wait()时能等待所有线程完成
<br>

```
type WaitGroupWrapper struct {
	sync.WaitGroup
}

func (w *WaitGroupWrapper) Wrap(cb func()) {
	w.Add(1)
	go func() {
		cb()
		w.Done()
	}()
}
```

### Main线程
- 流程图：
```
graph TD
A[Init] --> B[读取配置]
subgraph Start
B --> C[创建TCP监听goroutine]
C --> D[创建HTTP监听goroutine]
end
D -- 阻塞读取signalChan --> Stop
```


### 优雅退出
- 使用SVC启动 service(deamon)
    - 默认监控退出信号：syscall.SIGINT, syscall.SIGTERM

- 退出流程:退出信号触发signalChan
```
func (p *program) Stop() error {
	if l.tcpListener != nil {
		l.tcpListener.Close()
	}

	if l.httpListener != nil {
		l.httpListener.Close()
	}
	l.waitGroup.Wait()
}

```
- TCP/HTTP监听goroutine采用循环accept方式处理，当退出信号触发Stop()时关闭连接，使得Handle处理完毕进入下一次循环时accept返回error从而退出循环
- sync.waitGroup.Wait()等待两个协程处理完毕

