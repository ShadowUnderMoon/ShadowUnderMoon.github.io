# Grpc源码分析


## gprc流程概括



![Image.png](grpc流程概括.png)

grpc的流程可以大致分成两个阶段，分别为grpc连接阶段和grpc交互阶段，如图[^1]所示。

在RPC连接阶段，client和server之间建立起TCP连接，grpc底层依赖于HTTP2，因此client和server还需要协调frame的相关设置，例如frame的大小，滑动窗口的大小等。

在RPC交互阶段，client将数据发送给server，并等待server执行执行method之后返回结果。

### Client的流程

在RPC连接阶段，client接收到一个目标地址和一系列的DialOptions，然后

1. 配置连接参数，interceptor等，启动resolver
2. Rosovler根据目标地址获取server的地址列表（比如一个DNS name可能会指向多个server ip，dnsResolver是grpc内置的resolver之一），启动balancer
3. Balancer根据平衡策略，从诸多server地址中选择一个或者多个建立TCP连接
4. client在TCP连接建立完成之后，等待server发来的HTTP2 Setting frame，并调整自身的HTTP2相关配置，随后向server发送HTTP2 Setting frame

在RPC交互阶段，某个rpc方法被调用后

1. Client创建一个stream对象用来管理整个交互流程
2. Client将service name, method name等信息放到header frame中并发送给server
3. client将method的参数信息放到data frame中并发送给server
4. client等待server传回的header frame和data frame，一次rpc call的result status会被包含在header frame中，而method的返回值被包含在data frame中

### Server流程

在rpc连接阶段，server在完成一些初始化的配置之后，开始监听某个tcp端口，在和某个client建立了tcp连接之后完成http2 settting frame的交互。

在rpc交互阶段：

1. server等待client发来的header frame，从而创建出一个stream对象来管理真个交互流程，根据header frame中的信息，server知道client请求的是哪一个service的那一个method
2. server接受到client发来的data frame，并执行method
3. server将执行是否成功等信息放在header frame中发送给client
4. server将method执行的结果（返回值）放在data frame中发送给client

## grpc Server的rpc连接阶段

```go
func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

如上是一个简单的服务端程序，流程如下

1. 首先通过`net.Listener`监听tcp端口，毕竟grpc服务是基于tcp的
2. 创建grpc server，并注册服务，`&server{}`实际上就是服务的实现
3. 阻塞等待来自client的访问

```go
func (s *Server) Serve(lis net.Listener) error {
	s.serve = true
	for {
		rawConn, err := lis.Accept()
		s.serveWG.Add(1)
		go func() {
			s.handleRawConn(lis.Addr().String(), rawConn)
			s.serveWG.Done()
		}()

}

```

grpc在一个for循环中等待来自client的访问，每次有新的client端访问，创建一个`net.Conn`，并创建一个新的goroutine处理这个`net.Conn`，所以这个连接上的请求，无论客户端调用哪一个远程访问或者调用几次，都会由这个goroutine处理。

```go
func (s *Server) handleRawConn(lisAddr string, rawConn net.Conn) {
  // 如果grpc server已经关闭，那么同样关闭这个tcp连接
	if s.quit.HasFired() {
		rawConn.Close()
		return
	}
  // 设置tcp超时时间
	rawConn.SetDeadline(time.Now().Add(s.opts.connectionTimeout))

	// Finish handshaking (HTTP2)
	st := s.newHTTP2Transport(rawConn)
  // 清理tcp超时时间
	rawConn.SetDeadline(time.Time{})
  // rpc交互阶段，创建新的goroutine处理来自client的数据
	go func() {
		s.serveStreams(context.Background(), st, rawConn)
		s.removeConn(lisAddr, st)
	}()
}
```

在这里，首先判断gprc server是否关闭，如果关闭，则同样关闭tcp连接。然后进行HTTP2的握手，这里专门设置了tcp超时时间，避免握手迟迟不结束，导致资源占用，在握手结束后，清理tcp超时时间，避免对后面请求的影响。最后新启动一个goroutine，用来处理实际的请求。

### grpc服务端HTTP2握手

```go
func (s *Server) newHTTP2Transport(c net.Conn) transport.ServerTransport {
  // 组装 serverconfig
	config := &transport.ServerConfig{
		MaxStreams:            s.opts.maxConcurrentStreams,
		ConnectionTimeout:     s.opts.connectionTimeout,
		...
	}
  // 根据config的配置项，和client进行http2的握手
	st, err := transport.NewServerTransport(c, config)
```

根据使用者在启动grpc server时的配置项，或者默认的配置项，调用`transport.NewServerTransport`完成和client的http2握手。

```go
	writeBufSize := config.WriteBufferSize
	readBufSize := config.ReadBufferSize
	maxHeaderListSize := defaultServerMaxHeaderListSize
	if config.MaxHeaderListSize != nil {
		maxHeaderListSize = *config.MaxHeaderListSize
	}
	framer := newFramer(conn, writeBufSize, readBufSize, config.SharedWriteBuffer, maxHeaderListSize)
```

首先创建framer，用来负责接受和发送HTTP2 frame，是server和client交流的实际接口。

```go
// Send initial settings as connection preface to client.
isettings := []http2.Setting{{
  ID:  http2.SettingMaxFrameSize,
  Val: http2MaxFrameLen,
}}
if config.MaxStreams != math.MaxUint32 {
  isettings = append(isettings, http2.Setting{
    ID:  http2.SettingMaxConcurrentStreams,
    Val: config.MaxStreams,
  })
}
...

if err := framer.fr.WriteSettings(isettings...); err != nil {
  return nil, connectionErrorf(false, err, "transport: %v", err)
}
```

grpc server端首先明确自己的HTTP2的初始配置，比如MaxFrameSize等，并将这些配置信息通过`frame.fr`发送给client，`frame.fr`实际上就是golang原生的`http2.Framer`，在底层，这些配置信息会被包装成一个Setting Frame发送给client。

client在收到Setting Frame后，根据自身情况调整参数，同样发送一个Setting Frame给sever。

```go
t := &http2Server{
  done:              done,
  conn:              conn,
  peer:              peer,
  framer:            framer,
  readerDone:        make(chan struct{}),
  loopyWriterDone:   make(chan struct{}),
  maxStreams:        config.MaxStreams,
  inTapHandle:       config.InTapHandle,
  fc:                &trInFlow{limit: uint32(icwz)},
  state:             reachable,
  activeStreams:     make(map[uint32]*ServerStream),
  stats:             config.StatsHandlers,
  kp:                kp,
  idle:              time.Now(),
  kep:               kep,
  initialWindowSize: iwz,
  bufferPool:        config.BufferPool,
}
// controlbuf用来缓存Setting Frame等和设置相关的Frame的缓存
t.controlBuf = newControlBuffer(t.done)
// 自增连接id
t.connectionID = atomic.AddUint64(&serverConnectionCounter, 1)
// flush framer，确保向client发送了setting frame
t.framer.writer.Flush()
```

grpc server在发送了setting frame之后，创建好`http2Server`对象，并等待client的后续消息。

```go
// Check the validity of client preface.
preface := make([]byte, len(clientPreface))
// 读取客户端发来的client preface，并验证是否和预期一致
if _, err := io.ReadFull(t.conn, preface); err != nil {
  // In deployments where a gRPC server runs behind a cloud load balancer
  // which performs regular TCP level health checks, the connection is
  // closed immediately by the latter.  Returning io.EOF here allows the
  // grpc server implementation to recognize this scenario and suppress
  // logging to reduce spam.
  if err == io.EOF {
    return nil, io.EOF
  }
  return nil, connectionErrorf(false, err, "transport: http2Server.HandleStreams failed to receive the preface from client: %v", err)
}
if !bytes.Equal(preface, clientPreface) {
  return nil, connectionErrorf(false, nil, "transport: http2Server.HandleStreams received bogus greeting from client: %q", preface)
}

// 读取client端发来的frame
frame, err := t.framer.fr.ReadFrame()
if err == io.EOF || err == io.ErrUnexpectedEOF {
  return nil, err
}
if err != nil {
  return nil, connectionErrorf(false, err, "transport: http2Server.HandleStreams failed to read initial settings frame: %v", err)
}
atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())
// 转成SettingFrame
sf, ok := frame.(*http2.SettingsFrame)
if !ok {
  return nil, connectionErrorf(false, nil, "transport: http2Server.HandleStreams saw invalid preface type %T from client", frame)
}
// 处理SettingFrame
t.handleSettings(sf)
```

```go
func (t *http2Server) handleSettings(f *http2.SettingsFrame) {
  // 如果是ack frame，则直接返回
	if f.IsAck() {
		return
	}
	var ss []http2.Setting
	var updateFuncs []func()
	f.ForeachSetting(func(s http2.Setting) error {
		switch s.ID {
    // 更新http2Server中的配置信息
		case http2.SettingMaxHeaderListSize:
			updateFuncs = append(updateFuncs, func() {
				t.maxSendHeaderListSize = new(uint32)
				*t.maxSendHeaderListSize = s.Val
			})
		default:
			ss = append(ss, s)
		}
		return nil
	})
  // 这里又遇到了controlBuf
  // 执行updateFuncs()，然后将incommingSetting加入到controlBuf的队列中
	t.controlBuf.executeAndPut(func() bool {
		for _, f := range updateFuncs {
			f()
		}
		return true
	}, &incomingSettings{
		ss: ss,
	})
}
```

在HTTP2中，client和server都要求在建立连接前发送一个connection preface，作为对所使用协议的最终确认，并确定HTTP2连接的初始设置，client发送的preface一一个24字节的序列开始，之后紧跟着一个setting frame，用来表示client端最终决定的HTTP2配置参数。

```go
go func() {
  t.loopy = newLoopyWriter(serverSide, t.framer, t.controlBuf, t.bdpEst, t.conn, t.logger, t.outgoingGoAwayHandler, t.bufferPool)
  err := t.loopy.run()
  close(t.loopyWriterDone)
  if !isIOError(err) {
    timer := time.NewTimer(time.Second)
    defer timer.Stop()
    select {
    case <-t.readerDone:
    case <-timer.C:
    }
    t.conn.Close()
  }
}()
go t.keepalive()
```

此时server和client之间的HTTP2链接已经建立完成，RPC连接阶段完毕。在`NewServerTransport`的最后启动了`loopyWriter`，开始了rpc交互阶段，`loopyWriter`不断从`controlBuf`中读取control frames（包括setting frame)，并将缓存中的frame发送给client，可以说`loopyWriter`就是grpc server控制流量已经发送数据的地方。

这里另外启动了一个keepalive的goroutine，这个goroutine应该和grpc的keepalive机制有关。

## 参考文献

[^1] [gprc源码分析 zhengxinzx](https://juejin.cn/post/7089739785035579429) 一系列grpc源码分析，主要介绍了grpc的原理和流量控制

