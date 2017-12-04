## 《Go TCP Socket编程之teleport框架是怎样炼成的？》

本文通过回顾[teleport](https://github.com/henrylee2cn/teleport)框架的开发过程，讲述Go Socket的开发实战经验。

*注：下文以`TP`作为`Teleport`的简称。*

### 目录

- [1. TP的架构设计](#1-tp的架构设计)
- [2. TP的实现理念](#TP的实现理念)


### 1. TP的架构设计

#### 1.1 设计理念

TP定位于提供socket通信解决方案，遵循以下三点设计理念。

- 通用：不作定向深入，专注长连接通信
- 高效：高性能，低消耗
- 灵活：用法灵活简单，易于深入定制


#### 1.2 架构图

![Teleport-Architecture](https://github.com/henrylee2cn/teleport/raw/master/doc/teleport_architecture.png)

- `Peer`: 通信端点，可以是服务端或客户端
- `Plugin`: 贯穿于通信各个环节的插件
- `Handler`: 用于处理推、拉请求的函数
- `Router`: 通过请求信息（如URI）索引响应函数（Handler）的路由器
- `Socket`: 对net.Conn的封装，增加自定义包协议、传输管道等功能
- `Session`: 基于Socket封装的连接会话，提供的推、拉、回复、关闭等会话操作
- `Context`: 连接会话中一次通信（如PULL-REPLY, PUSH）的上下文对象
- `Packet`: 约定数据报文包含的内容元素（注意：它不是协议格式）
- `Protocol`: 数据报文封包解包操作，即通信协议的实现接口
- `Codec`: 数据包body部分（请求参数或响应结果）的序列化接口
- `XferPipe`: 数据包字节流的编码处理管道，如压缩、加密、校验等


#### 1.3 重要特性

- 支持自定义通信协议和包数据处理管道
- TCP长连接使用I/O缓冲区与多路复用技术，提升数据吞吐量
- 支持设置读取包的大小限制（如果超出则断开连接）
- 支持插件机制，可以自定义认证、心跳、微服务注册中心、统计信息插件等
- 服务端和客户端之间对等通信，统一为peer端点，具有基本一致的用法：
	- 推、拉、回复等通信方法
	- 丰富的插件挂载点，可以自定义认证、心跳、微服务注册中心、统计信息等等
	- 平滑重启与关闭
	- 日志信息详尽，支持打印输入、输出消息的详细信息（状态码、消息头、消息体）
	- 支持设置慢操作报警阈值
	- 提供Hander的上下文（pull、push的handler）

### 2. TP-Socket

`telepot/socket`包在net.Conn的基础上增加自定义包协议、传输管道等功能，是TP的基础通信包。

#### 2.1 Packet源码片段分析

```go
type (
	// Packet a socket data packet.
	Packet struct {
		// packet sequence
		seq uint64
		// packet type, such as PULL, PUSH, REPLY
		ptype byte
		// URL string
		uri string
		// metadata
		meta *utils.Args
		// body codec type
		bodyCodec byte
		// body object
		body interface{}
		// newBodyFunc creates a new body by packet type and URI.
		// Note:
		//  only for writing packet;
		//  should be nil when reading packet.
		newBodyFunc NewBodyFunc
		// XferPipe transfer filter pipe, handlers from outer-most to inner-most.
		// Note: the length can not be bigger than 255!
		xferPipe *xfer.XferPipe
		// packet size
		size uint32
		next *Packet
	}
	// packet header interface
	Header interface {
		// Ptype returns the packet sequence
		Seq() uint64
		// SetSeq sets the packet sequence
		SetSeq(uint64)
		// Ptype returns the packet type, such as PULL, PUSH, REPLY
		Ptype() byte
		// Ptype sets the packet type
		SetPtype(byte)
		// Uri returns the URL string string
		Uri() string
		// SetUri sets the packet URL string
		SetUri(string)
		// Meta returns the metadata
		Meta() *utils.Args
		// SetMeta sets the metadata
		SetMeta(*utils.Args)
	}
	// packet body interface
	Body interface {
		// BodyCodec returns the body codec type id
		BodyCodec() byte
		// SetBodyCodec sets the body codec type id
		SetBodyCodec(bodyCodec byte)
		// Body returns the body object
		Body() interface{}
		// SetBody sets the body object
		SetBody(body interface{})
		// SetNewBody resets the function of geting body.
		SetNewBody(newBodyFunc NewBodyFunc)
		// NewBody creates a new body by packet type and URI.
		// Note:
		//  only for writing packet;
		//  should be nil when reading packet.
		// NewBody(seq uint64, ptype byte, uri string) interface{}

		// MarshalBody returns the encoding of body.
		MarshalBody() ([]byte, error)
		// UnmarshalNewBody unmarshal the encoded data to a new body.
		// Note: seq, ptype, uri must be setted already.
		UnmarshalNewBody(bodyBytes []byte) error
		// UnmarshalBody unmarshal the encoded data to the existed body.
		UnmarshalBody(bodyBytes []byte) error
	}

	// NewBodyFunc creates a new body by header.
	NewBodyFunc func(Header) interface{}
)
```

- **Go技巧思考：**

*实际上`Header`和`Body`两个接口都是由`Packet`结构体实现，那么为什么不直接定义两个子结构体？*

1. `Packet`只是用于定义统一的数据包内容元素，并给予任何关于数据结构方面（协议）的暗示、误导。因此不应该使用子结构体
2. `Packet`全部字段均不可导出，可以增强框架对其用法的掌控力，避免开发者作出不恰当操作

#### 2.2 Socket源码片段分析

```go
type (
	// Socket is a generic stream-oriented network connection.
	//
	// Multiple goroutines may invoke methods on a Socket simultaneously.
	Socket interface {
		// LocalAddr returns the local network address.
		LocalAddr() net.Addr

		// RemoteAddr returns the remote network address.
		RemoteAddr() net.Addr

		// SetDeadline sets the read and write deadlines associated
		// with the connection. It is equivalent to calling both
		// SetReadDeadline and SetWriteDeadline.
		//
		// A deadline is an absolute time after which I/O operations
		// fail with a timeout (see type Error) instead of
		// blocking. The deadline applies to all future and pending
		// I/O, not just the immediately following call to Read or
		// Write. After a deadline has been exceeded, the connection
		// can be refreshed by setting a deadline in the future.
		//
		// An idle timeout can be implemented by repeatedly extending
		// the deadline after successful Read or Write calls.
		//
		// A zero value for t means I/O operations will not time out.
		SetDeadline(t time.Time) error

		// SetReadDeadline sets the deadline for future Read calls
		// and any currently-blocked Read call.
		// A zero value for t means Read will not time out.
		SetReadDeadline(t time.Time) error

		// SetWriteDeadline sets the deadline for future Write calls
		// and any currently-blocked Write call.
		// Even if write times out, it may return n > 0, indicating that
		// some of the data was successfully written.
		// A zero value for t means Write will not time out.
		SetWriteDeadline(t time.Time) error

		// WritePacket writes header and body to the connection.
		// Note: must be safe for concurrent use by multiple goroutines.
		WritePacket(packet *Packet) error

		// ReadPacket reads header and body from the connection.
		// Note: must be safe for concurrent use by multiple goroutines.
		ReadPacket(packet *Packet) error

		// Read reads data from the connection.
		// Read can be made to time out and return an Error with Timeout() == true
		// after a fixed time limit; see SetDeadline and SetReadDeadline.
		Read(b []byte) (n int, err error)

		// Write writes data to the connection.
		// Write can be made to time out and return an Error with Timeout() == true
		// after a fixed time limit; see SetDeadline and SetWriteDeadline.
		Write(b []byte) (n int, err error)

		// Reset reset net.Conn
		// Reset(net.Conn)

		// Close closes the connection socket.
		// Any blocked Read or Write operations will be unblocked and return errors.
		Close() error

		// Public returns temporary public data of Socket.
		Public() goutil.Map
		// PublicLen returns the length of public data of Socket.
		PublicLen() int
		// Id returns the socket id.
		Id() string
		// SetId sets the socket id.
		SetId(string)
	}
	socket struct {
		net.Conn
		protocol  Proto
		id        string
		idMutex   sync.RWMutex
		ctxPublic goutil.Map
		mu        sync.RWMutex
		curState  int32
		fromPool  bool
	}
)
```

- 关键函数与方法


```go
// NewSocket wraps a net.Conn as a Socket.
func NewSocket(c net.Conn, protoFunc ...ProtoFunc) Socket {
	return newSocket(c, protoFunc)
}

func newSocket(c net.Conn, protoFuncs []ProtoFunc) *socket {
	var s = &socket{
		protocol: getProto(protoFuncs, c),
		Conn:     c,
	}
	s.optimize()
	return s
}

func getProto(protoFuncs []ProtoFunc, rw io.ReadWriter) Proto {
	if len(protoFuncs) > 0 {
		return protoFuncs[0](rw)
	} else {
		return defaultProtoFunc(rw)
	}
}
```

```go
func (s *socket) WritePacket(packet *Packet) error {
	s.mu.RLock()
	protocol := s.protocol
	s.mu.RUnlock()
	if packet.BodyCodec() == codec.NilCodecId {
		packet.SetBodyCodec(defaultBodyCodec.Id())
	}
	err := protocol.Pack(packet)
	if err != nil && s.isActiveClosed() {
		err = ErrProactivelyCloseSocket
	}
	return err
}
```

```go
func (s *socket) ReadPacket(packet *Packet) error {
	s.mu.RLock()
	protocol := s.protocol
	s.mu.RUnlock()
	return protocol.Unpack(packet)
}
```

- **Go技巧思考：**

*为什么要对外提供接口，而不直接公开结构体？*

`socket`结构体通过匿名字段`net.Conn`的方式“继承”了底层的连接操作方法，并基于该匿名字段创建了协议对象。
所以不能允许外部直接通过`socket.Conn=newConn`的方式改变连接句柄。

使用`Socket`接口封装包外不可见的`socket`结构体可达到避免外部直接修改字段的目的。


#### 2.3 Proto定义




