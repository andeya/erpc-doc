## 《Go TCP Socket编程之teleport框架是怎样炼成的？》

本文通过回顾[teleport](https://github.com/henrylee2cn/teleport)框架的开发过程，讲述Go Socket的开发实战经验。

*注：下文以`TP`作为`Teleport`的简称。*

简单的性能对比：

- teleport/socket

![tp_socket_benchmark](https://github.com/henrylee2cn/teleport/raw/master/doc/tp_socket_benchmark.png)

**[test code](https://github.com/henrylee2cn/rpc-benchmark/tree/master/teleport)**

- 与rpcx的直接对比

![rpcx_benchmark](https://github.com/henrylee2cn/teleport/raw/master/doc/rpcx_benchmark.jpg)

**[test code](https://github.com/henrylee2cn/rpc-benchmark/tree/master/rpcx)**

- rpcx与其他框架的对比参考（图片来源于rpcx）

![rpc_compare](src/rpc_compare.png)

- teleport/socket火焰图

![tp_socket_torch](https://github.com/henrylee2cn/teleport/raw/master/doc/tp_socket_torch.png)

**[svg file](https://github.com/henrylee2cn/teleport/raw/master/doc/tp_socket_torch.svg)**

<!-- - 案例
TP目前应用于小恩爱app长连接网关，支撑聊天、推送、微服务代理等功能。
1条连接平均占用 25.6KB，即16GB内存承载655,360连接 -->

|目 录
|--------------------------------
|[1. TP的架构设计](#1-tp的架构设计)
|[2. Socket包](#2-socket包)

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

### 2. Socket包

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

- **编译期校验`Packet`是否已实现`Header`与`Body`接口的技巧**

```go
var (
	_ Header = new(Packet)
	_ Body   = new(Packet)
)
```

- **一种常见的自由赋值的函数用法，用于自由设置`Packet`的字段**

```go
// PacketSetting sets Header field.
type PacketSetting func(*Packet)

// WithSeq sets the packet sequence
func WithSeq(seq uint64) PacketSetting {
	return func(p *Packet) {
		p.seq = seq
	}
}

// Ptype sets the packet type
func WithPtype(ptype byte) PacketSetting {
	return func(p *Packet) {
		p.ptype = ptype
	}
}

...
```


- ***Go技巧思考：实际上`Header`和`Body`两个接口都是由`Packet`结构体实现，那么为什么不直接定义两个子结构体？***
	1. `Packet`只是用于定义统一的数据包内容元素，并给予任何关于数据结构方面（协议）的暗示、误导。因此不应该使用子结构体
	2. `Packet`全部字段均不可导出，可以增强框架对其用法的掌控力，避免开发者作出不恰当操作


#### 2.2 Socket源码片段分析

```go
type (
	// Socket is a generic stream-oriented network connection.
	//
	// Multiple goroutines may invoke methods on a Socket simultaneously.
	Socket interface {
		net.Conn

		// WritePacket writes header and body to the connection.
		// Note: must be safe for concurrent use by multiple goroutines.
		WritePacket(packet *Packet) error

		// ReadPacket reads header and body from the connection.
		// Note: must be safe for concurrent use by multiple goroutines.
		ReadPacket(packet *Packet) error

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

- ***Go技巧思考：为什么要对外提供接口，而不直接公开结构体？***

	`socket`结构体通过匿名字段`net.Conn`的方式“继承”了底层的连接操作方法，并基于该匿名字段创建了协议对象。<br>所以不能允许外部直接通过`socket.Conn=newConn`的方式改变连接句柄。<br>使用`Socket`接口封装包外不可见的`socket`结构体可达到避免外部直接修改字段的目的。


#### 2.3 Proto定义

数据包封包、解包接口。即封包时以`Packet`的字段为内容元素进行数据序列化，解包时以`Packet`为内容模板进行数据的反序列化。

```go
type (
	// Proto pack/unpack protocol scheme of socket packet.
	Proto interface {
		// Version returns the protocol's id and name.
		Version() (byte, string)
		// Pack pack socket data packet.
		// Note: Make sure to write only once or there will be package contamination!
		Pack(*Packet) error
		// Unpack unpack socket data packet.
		// Note: Concurrent unsafe!
		Unpack(*Packet) error
	}
	// ProtoFunc function used to create a custom Proto interface.
	ProtoFunc func(io.ReadWriter) Proto
)
```

### 3 Codec编解码器

`teleport/codec`包用于`socket.Packet.body`的编解码器。如TP已经自带注册了JSON、Protobuf、String三种编解码器。

#### 3.1 Codec接口定义

```go
type (
	// Codec makes Encoder and Decoder
	Codec interface {
		// Id returns codec id.
		Id() byte
		// Name returns codec name.
		Name() string
		// Marshal returns the encoding of v.
		Marshal(v interface{}) ([]byte, error)
		// Unmarshal parses the encoded data and stores the result
		// in the value pointed to by v.
		Unmarshal(data []byte, v interface{}) error
	}
)
```

#### 3.2 Codec注册中心

- 常用的依赖注入实现方式，实现编解码器的自由定制

```go
var codecMap = struct {
	nameMap map[string]Codec
	idMap   map[byte]Codec
}{
	nameMap: make(map[string]Codec),
	idMap:   make(map[byte]Codec),
}

const (
	NilCodecId   byte   = 0
	NilCodecName string = ""
)

// Reg registers Codec
func Reg(codec Codec) {
	if codec.Id() == NilCodecId {
		panic(fmt.Sprintf("codec id can not be %d", NilCodecId))
	}
	if _, ok := codecMap.nameMap[codec.Name()]; ok {
		panic("multi-register codec name: " + codec.Name())
	}
	if _, ok := codecMap.idMap[codec.Id()]; ok {
		panic(fmt.Sprintf("multi-register codec id: %d", codec.Id()))
	}
	codecMap.nameMap[codec.Name()] = codec
	codecMap.idMap[codec.Id()] = codec
}

// Get returns Codec
func Get(id byte) (Codec, error) {
	codec, ok := codecMap.idMap[id]
	if !ok {
		return nil, fmt.Errorf("unsupported codec id: %d", id)
	}
	return codec, nil
}

// GetByName returns Codec
func GetByName(name string) (Codec, error) {
	codec, ok := codecMap.nameMap[name]
	if !ok {
		return nil, fmt.Errorf("unsupported codec name: %s", name)
	}
	return codec, nil
}
```

- ***Go技巧思考：变量`codecMap`的类型为什么不用关键字`type`定义？***

	Go语法允许我们在声明变量时临时定义类型并赋值。因为`codecMap`所属类型只会有一个全局唯一的实例，且不会用于其他变量类型声明上，所以直接在声明变量时声明类型可以令代码更简洁。

### 4 Xfer数据传输处理管道

`teleport/xfer`包用于对数据包进行一系列自定义处理加工，如gzip压缩、加密、校验等。

#### 4.1 类型定义

```go
type (
	// XferPipe transfer filter pipe, handlers from outer-most to inner-most.
	// Note: the length can not be bigger than 255!
	XferPipe struct {
		filters []XferFilter
	}
	// XferFilter handles byte stream of packet when transfer.
	XferFilter interface {
		Id() byte
		OnPack([]byte) ([]byte, error)
		OnUnpack([]byte) ([]byte, error)
	}
)

var xferFilterMap = struct {
	idMap map[byte]XferFilter
}{
	idMap: make(map[byte]XferFilter),
}
```

该包设计与`teleport/codec`包类似，`xferFilterMap`为注册中心，提供注册、查询、执行等功能。


### 5 Peer通信端点

Peer结构体是TP的一个通信端点，它可以是服务端也可以是客户端，甚至可以同时是服务端与客户端。因此，TP是端对端对等通信的。

#### 5.1 定义

```go
type Peer struct {
	PullRouter *Router
	PushRouter *Router

	// Has unexported fields.
}
func NewPeer(cfg *PeerConfig, plugin ...Plugin) *Peer
func (p *Peer) Close() (err error)
func (p *Peer) CountSession() int
func (p *Peer) Dial(addr string, protoFunc ...socket.ProtoFunc) (Session, *Rerror)
func (p *Peer) DialContext(ctx context.Context, addr string, protoFunc ...socket.ProtoFunc) (Session, *Rerror)
func (p *Peer) GetSession(sessionId string) (Session, bool)
func (p *Peer) Listen(protoFunc ...socket.ProtoFunc) error
func (p *Peer) RangeSession(fn func(sess Session) bool)
func (p *Peer) ServeConn(conn net.Conn, protoFunc ...socket.ProtoFunc) (Session, error)
```
*以上代码是在teleport目录下执行`go doc Peer`获得*

#### 5.2 Peer配置信息

```go
type PeerConfig struct {
	TlsCertFile         string
	TlsKeyFile          string
	DefaultReadTimeout  time.Duration
	DefaultWriteTimeout time.Duration
	SlowCometDuration   time.Duration
	DefaultBodyCodec    string
	PrintBody           bool
	CountTime           bool
	DefaultDialTimeout  time.Duration
	ListenAddrs         []string
}
```

#### 5.3 Peer的功能列表

- 提供路由功能
- 作为服务端可同时支持监听多个地址端口
- 作为客户端可与任意服务端建立连接
- 提供会话查询功能
- 支持TLS证书安全加密
- 设置默认的建立连接和读、写超时
- 慢响应阀值（超出后运行日志由INFO提升为WARN）
- 支持打印body
- 支持在运行日志中增加耗时统计

### 6 路由、控制器与操作

TP是对等通信，路由不再是服务端的专利，只要是Peer端点就支持注册`PULL`和`PUSH`这两类消息处理路由。

#### 6.1 Router路由

```go
type Router struct {
	handlers       map[string]*Handler
	unknownApiType **Handler
	// only for register router
	pathPrefix      string
	pluginContainer PluginContainer
	typ             string
	maker           HandlersMaker
}

func (r *Router) Group(pathPrefix string, plugin ...Plugin) *Router
func (r *Router) Reg(ctrlStruct interface{}, plugin ...Plugin)
func (r *Router) SetUnknown(unknownHandler interface{}, plugin ...Plugin)

```
1.Router结构体根据HandlersMaker（Handler的构造函数）的不同，分别实现了`PullRouter`和`PushRouter`两类路由。

```go
// HandlersMaker makes []*Handler
type HandlersMaker func(pathPrefix string, ctrlStruct interface{}, pluginContainer PluginContainer) ([]*Handler, error)
```

2.路由分组的实现：

```go
// Group add handler group.
func (r *Router) Group(pathPrefix string, plugin ...Plugin) *Router {
	pluginContainer, err := r.pluginContainer.cloneAdd(plugin...)
	if err != nil {
		Fatalf("%v", err)
	}
	warnInvaildRouterHooks(plugin)
	return &Router{
		handlers:        r.handlers,
		unknownApiType:  r.unknownApiType,
		pathPrefix:      path.Join(r.pathPrefix, pathPrefix),
		pluginContainer: pluginContainer,
		maker:           r.maker,
	}
}
```

思路很简单，用法很也简单，核心点只有两个：

- 继承各级路由的共享字段：`handlers`、`unknownApiType`、`maker`
- 在上级路由节点的`pathPrefix`、`pluginContainer`字段基础上追加当前节点信息


#### 6.2 控制器

控制器是指用于提供Handler操作的结构体。

PullController Model:

```go
type Aaa struct {
	tp.PullCtx
}
// XxZz register the route: /aaa/xx_zz
func (x *Aaa) XxZz(args *<T>) (<T>, *tp.Rerror) {
	...
	return r, nil
}
// YyZz register the route: /aaa/yy_zz
func (x *Aaa) YyZz(args *<T>) (<T>, *tp.Rerror) {
	...
	return r, nil
}
```

PushController Model:

```go
type Bbb struct {
	tp.PushCtx
}
// XxZz register the route: /bbb/yy_zz
func (b *Bbb) XxZz(args *<T>) {
	...
	return r, nil
}
// YyZz register the route: /bbb/yy_zz
func (b *Bbb) YyZz(args *<T>) {
	...
	return r, nil
}
```

TP的路由路径采用类似URL的格式，并且支持query参数（如`/a/b?n=1&m=e`）

#### 6.3 Unknown操作函数

TP可通过`func (r *Router) SetUnknown(unknownHandler interface{}, plugin ...Plugin)`方法设置默认Handler，用于处理未找到路由的`PULL`或`PUSH`消息。

UnknownPullHandler Type:

```go
func(ctx UnknownPullCtx) (interface{}, *Rerror) {
	...
	return r, nil
}
```

UnknownPushHandler Type:

```go
func(ctx UnknownPushCtx)
```

#### 6.4 Handler的构造

```go
// Handler pull or push handler type info
Handler struct {
	name              string
	isUnknown         bool
	argElem           reflect.Type
	reply             reflect.Type // only for pull handler doc
	handleFunc        func(*readHandleCtx, reflect.Value)
	unknownHandleFunc func(*readHandleCtx)
	pluginContainer   PluginContainer
}
```

通过`HandlersMaker`对Controller的各个方法的解析，构造出相应数量的Handler。以`pullHandlersMaker`函数为例：


```go
func pullHandlersMaker(pathPrefix string, ctrlStruct interface{}, pluginContainer PluginContainer) ([]*Handler, error) {
	var (
		ctype    = reflect.TypeOf(ctrlStruct)
		handlers = make([]*Handler, 0, 1)
	)
	...
	var ctypeElem = ctype.Elem()
	...
	iType, ok := ctypeElem.FieldByName("PullCtx")
	...
	var pullCtxOffset = iType.Offset

	if pluginContainer == nil {
		pluginContainer = newPluginContainer()
	}

	type PullCtrlValue struct {
		ctrl   reflect.Value
		ctxPtr *PullCtx
	}
	var pool = &sync.Pool{
		New: func() interface{} {
			ctrl := reflect.New(ctypeElem)
			pullCtxPtr := ctrl.Pointer() + pullCtxOffset
			ctxPtr := (*PullCtx)(unsafe.Pointer(pullCtxPtr))
			return &PullCtrlValue{
				ctrl:   ctrl,
				ctxPtr: ctxPtr,
			}
		},
	}

	for m := 0; m < ctype.NumMethod(); m++ {
		method := ctype.Method(m)
		mtype := method.Type
		mname := method.Name
		...
		var methodFunc = method.Func
		var handleFunc = func(ctx *readHandleCtx, argValue reflect.Value) {
			obj := pool.Get().(*PullCtrlValue)
			*obj.ctxPtr = ctx
			rets := methodFunc.Call([]reflect.Value{obj.ctrl, argValue})
			ctx.output.SetBody(rets[0].Interface())
			rerr, _ := rets[1].Interface().(*Rerror)
			if rerr != nil {
				rerr.SetToMeta(ctx.output.Meta())

			} else if ctx.output.Body() != nil && ctx.output.BodyCodec() == codec.NilCodecId {
				ctx.output.SetBodyCodec(ctx.input.BodyCodec())
			}
			pool.Put(obj)
		}

		handlers = append(handlers, &Handler{
			name:            path.Join(pathPrefix, ctrlStructSnakeName(ctype), goutil.SnakeString(mname)),
			handleFunc:      handleFunc,
			argElem:         argType.Elem(),
			reply:           replyType,
			pluginContainer: pluginContainer,
		})
	}
	return handlers, nil
}
```

**该函数中用到的Go技巧：**

- 对不可变的部分进行预计算获得闭包变量，抽离可变部分的逻辑构造多个`handleFunc`子函数。在路由处理过程中直接执行这些子函数可达到显著提升性能的目的
- 使用反射来创建任意类型的实例并调用其方法，适用于类型或方法不固定的情况
- 使用对象池来复用`PullCtrlValue`，可以降低GC开销与内存占用
- 通过unsafe获取`ctrlStruct.PullCtx`字段的指针偏移量，进而可以快速获取该字段的值











