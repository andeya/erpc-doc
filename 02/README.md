# Go语言微服务框架简洁设计与实践



**项目源码**

teleport：https://github.com/henrylee2cn/teleport

tp-micro：https://github.com/xiaoenai/tp-micro



## 背景

大家在进行业务开发时，是否是否遇到过下列问题，并且无法在Go语言开源生态中找到一套完整的解决方案？

- 高性能、可靠地通信？
- 开发效率不高？
- 无法自定义应用层协议？
- 想要动态协商Body编码类型（如JSON、protobuf等）？
- 不能以简洁的RPC方式进行业务开发？
- 没有灵活的插件扩展机制？
- 不支持服务端向客户端主动推送消息？
- 特殊场景时需要连接管理，如多种连接类型、会话管理？
- 使用了非HTTP协议框架，但不能很好的兼容HTTP协议，无法方便地与第三方对接？



我对于常见的一些相关开源项目做了一次粗略调查，发现迄今为止，除今天我向分享这款 [teleport](https://github.com/henrylee2cn/teleport) 框架外（确切讲应该是由teleport扩展出来的微服务框架 [tp-micro](https://github.com/xiaoenai/tp-micro)），貌似并没有另外一款Go语言的开源框架能够同时解决上述问题：

| 框架         | 描述                | 高性能 | 高效开发 | DIY应用层协议 | Body编码协商 | RPC范式 | 插件  | 推送  | 连接管理 | 兼容HTTP协议 |
| ------------ | ------------------- | ------ | -------- | ------------- | ------------ | ------- | ----- | ----- | -------- | ------------ |
| **teleport** | **TCP socket 框架** | ★★★★   | **√**    | **√**         | **√**        | **√**   | **√** | **√** | **√**    | **√**        |
| net          | 标准包网络工具      | ★★★★★  | x        | √             | x            | x       | x     | √     | √        | x            |
| net/rpc      | 标准包RPC           | ★★★★☆  | x        | x             | x            | √       | x     | x     | x        | x            |
| net/http(2)  | 标准包HTTP2         | ★★★☆   | x        | x             | √            | x       | x     | √     | x        | √            |
| gRPC         | 谷歌出品的RPC框架   | ★★★    | √        | x             | √            | √       | x     | √     | x        | √            |
| rpcx         | net/rpc的扩展框架   | ★★★★   | √        | x             | x            | √       | √     | √     | x        | √            |
| go-micro     | 插件化微服务框架    | ★★★☆   | √        | x             | x            | √       | √     | √     | x        | √            |



## 概述

[teleport](https://github.com/henrylee2cn/teleport) 就是在上述需求背景下被创造出来，成为一个通用、高效、灵活的Socket框架。

它可以用于Peer-Peer对等通信、RPC、长连接网关、微服务、推送服务，游戏服务等领域。

其主要特性如下：

|       *       |    *     |       *       |      *       |    *     |    *    |    *     |                           *                            |      *       |
| :-----------: | :------: | :-----------: | :----------: | :------: | :-----: | :------: | :----------------------------------------------------: | :----------: |
|    高性能     | 高效开发 | DIY应用层协议 | Body编码协商 | RPC范式  |  插件   |   推送   | 连接管理<br>（Socket文件描述符/<br>会话管理/上下文等） | 兼容HTTP协议 |
| 平滑关闭/升级 | Log接口  | 非阻塞异步IO  |   断线重连   | 对等通信 | 对等API | 反向代理 |                       慢响应报警                       |    ......    |

TODO：尚未提供多语言客户端版本



 [tp-micro](https://github.com/xiaoenai/tp-micro) 是以 teleport + plugin 的方式扩展而来的微服务。虽然目前还有一些功能未开发，但已有两家公司使用。它在完整继承 teleport 特性的同时，增加如下主要特性：

|      *       |        *         |    *     |    *     |         *          |   *    |       *        |   *    |  *   |
| :----------: | :--------------: | :------: | :------: | :----------------: | :----: | :------------: | :----: | :--: |
| 服务自动发现 | 自定义微服务路由 | 负载均衡 | 心跳机制 | 过载保护（断路器） | 热编译 | 由模板生成项目 | ...... |      |



## 架构



### 设计原则

- 面向接口设计，保证代码稳定，提供灵活定制
- 抽象核心模型，保持最简化
- 分层设计，自下而上逐层封装，利于稳定和维护
- 充分利用协程，且保证可控、可复用



### 架构示意图

![Teleport-Framework](https://github.com/henrylee2cn/teleport/raw/v4/doc/teleport_module_diagram.png)

![tp-micro flow chart](https://github.com/xiaoenai/tp-micro/raw/v3/doc/tp-micro_flow_chart.png)

注：“tp” 是 teleport 的包名，因此它代指 “teleport”。



## 简单的性能对比图

<table>
<tr><th>Environment</th><th>Throughputs</th><th>Mean Latency</th><th>P99 Latency</th></tr>
<tr>
<td width="10%"><img src="https://github.com/henrylee2cn/rpc-benchmark/raw/master/result/env.png"></td>
<td width="30%"><img src="https://github.com/henrylee2cn/rpc-benchmark/raw/master/result/throughput.png"></td>
<td width="30%"><img src="https://github.com/henrylee2cn/rpc-benchmark/raw/master/result/mean_latency.png"></td>
<td width="30%"><img src="https://github.com/henrylee2cn/rpc-benchmark/raw/master/result/p99_latency.png"></td>
</tr>
</table>



## 为兼容HTTP做准备

兼容 HTTP 最好的办法就是在设计应用层协议之初就考虑到进去。因此，teleport 对应用层协议报文的属性做了如下抽象：

- Size 整个报文的长度
- Transfer-Filter-Pipeline 报文数据过滤处理管道
- Header
  - Seq 消息序号（因为是异步通信）
  - Mtype 消息类型（如PULL、REPLY、PUSH）
  - URI 资源标识符（对照常见RPC框架中的method，但可以更好地兼容HTTP）
  - Meta 元信息（如错误信息、内容协商信息等，对照HTTP Header）
- Body
  - BodyCodec 消息正文的编码类型（如JSON、Protobuf）
  - Body 消息正文



从下图 teleport 报文属性与 HTTP 报文对比中，不难发现它们有共通之处。

![tp_data_message](https://github.com/henrylee2cn/teleport/raw/v4/doc/tp_data_message.png)



## 如何实现DIY应用层协议？

应用层协议是指建立在 TCP 协议之上的报文协议。我们希望开发者自己定制该协议，这样更具备灵活性，比如protobuf、thrift等。



首先要做的第一件事是：

抽象出一个 Message 对象，为应用层协议接口提供字节流序列化与反序列化模板。



### Step1: 抽象 Message 对象

在  teleport/socket 包中抽象出 Message 结构体（上面已经介绍过了）



### Step2: 抽象 Proto 协议接口

提供 Proto 协议接口，对 Message 对象进行序列化与反序列化，从而支持开发者的自定义实现自己的协议格式，其接口声明如下：

```go
type Proto interface {
    Version() (byte, string)
    Pack(*Message) error
    Unpack(*Message) error
}
```

解释：

- `Version` ：实现该协议接口的版本号
- `Pack` ：按照接口实现的规则，将 Message 的属性序列化为字节流
- `Unpack` ：按照接口实现的规则，将字节流反序列化进一个 Message 对象

目前框架已经提供三种协议：Raw、JSON、Protobuf。

其中以Raw为示例，展示如下：

```
# raw protocol format(Big Endian):

{4 bytes message length}
{1 byte protocol version}
{1 byte transfer pipe length}
{transfer pipe IDs}
# The following is handled data by transfer pipe
{4 bytes sequence length}
{sequence}
{1 byte message type} // e.g. CALL:1; REPLY:2; PUSH:3
{4 bytes URI length}
{URI}
{4 bytes metadata length}
{metadata(urlencoded)}
{1 byte body codec id}
{body}
```



## 如何实现 Body 编码协商？

在实际业务场景中，报文的类型是多种多样的，所以 teleport 使用 `Codec` 接口对消息正文（Message Body）进行编解码。

```go
type Codec interface {
	Id() byte
	Name() string
	Marshal(v interface{}) ([]byte, error)
	Unmarshal(data []byte, v interface{}) error
}
```

解释：

- `Id` ：编解码器的唯一识别码
- `Name` ：编解码器的名称，同样要求全局唯一，主要是便于开发者记忆和可视化
- `Marshal` ：编码
- `Unmarshal` ：解码

开发者可以将自定义的新编解码器注入 `teleport/codec` 包，从而在整个项目中使用。

框架已经提供的编解码器实现：JSON、Protobuf、Form（urlencoded）、Plain（raw text）



在自由支持各种编解码类型后，我就可以模仿 HTTP 协议头的 Content-Type 实现一下协商功能了。



在 Request/Response 的通信场景下，按以下步骤进行 Body 编码类型协商：

- Step1：请求端将当前 Body 的编码类型设置到 Message 的 `BodyCodec` 属性
- Step2：在请求端希望收到请求Body不同的编码类型时（在web开发中很常见），就可以在 Message 对象的 Meta 元信息中设置 `X-Accept-Body-Codec` 来指定响应的编码类型
- Step3：响应端根据请求的 `BodyCodec` 属性解码 Body，执行业务逻辑
- Step4：响应端在发现有 `X-Accept-Body-Codec` 元信息时，使用该元信息指定类型编码响应 Body，否则默认使用与请求相同的编码类型。当然，响应端的开发者也可以明确指定编码类型，这样就会忽略前面的规则，强制使用该指定的编码类型。



## 如何管理连接？



一般常见的 Go 语言 RPC 框架都没有重视对连接的管理，甚至是没有连接管理功能。那么，是不是就说明连接管理功能不重要？可有可无？其实不然，这只是与 RPC 框架的定位有关：

- 实现远程过程调用，并不强调连接，甚至是刻意屏蔽掉底层连接



那么，什么场景下，我们需要使用连接管理？

- 服务端主动推送消息给**指定（一批）连接**的客户端
- 服务端主动请求客户端，并获得客户端的响应
- 增加会话管理，将每条连接命名为用户ID，并绑定用户信息
- 获取文件描述符，对连接性能进行调优
- 异步主动断开**指定（一批）连接**
- 与第三方框架/组件对接



下面我们来了解一下 teleport 是如何实现连接管理的。



### Step1：封装 Socket 模块

首先，我们以分层的原则对来自net标准包的 `net.Conn` 进行封装得到 Socket 接口。它作为整个框架的底层通信接口，向上层提供应用层消息通信和连接管理的基础功能。

该接口涉及五个组件：

- 来自标准包的 `net.Conn` 接口
- 抽象的应用层协议接口 `Proto`
- 对字节流处理的接口管道 `XferPipe`
- 抽象出来的 `Message` 结构体
- 用于编解码 `Message` 中 `Body` 数据的接口 `Codec`



常用接口方法如下：

- `WriteMessage(message *Message) error` ：写入应用层消息
- `ReadMessage(message *Message) error` ：读取应用层消息
- `SetId(string)` 、`Id() string` ：设置或读取当前连接ID
- `Swap() goutil.Map` ：存储与当前连接相关的临时数据
- `ControlFD(f func(fd uintptr)) error`  ：操作当前连接的文件描述符



### Step2：封装 Session 模块



Session 对象封装了 Socket 接口 ，并负责整个会话相关的事务（相当于引擎）。如：

- 读消息协程
- 创建消息处理的上下文
- 执行路由与操作
- 写入消息
- 打印运行日志
- 连接的ID命名
- 绑定连接相关状态信息（用户资料等）
- 连接生命周期（连接超时）
- 一次请求的生命周期（请求超时）
- 主动断开连接
- 拨号端的断线重连
- 连接断开事件通知



### Step3：并发 Map 集中管理 Session

Peer 是 teleport 对通信两端的对等抽象，除了 Listener 与 Dialer 固有的角色差异外，两种角色拥有完全一致的API。Peer 就包含有一个并发 Map 用于保存全部 Session。因此，开发者可以通过 Peer 实现：

- 监听地址端口
- 拨号建立连接
- 获取指定 ID 的 Session 实例
- 向所有 Session 广播消息
- 查看当前连接数
- 平滑关闭全部连接



## 如何设计灵活的插件



插件会给框架带来灵活性和扩展性，是一个非常重要的模块。那么，如何设计好它？teleport 从三方面考虑：

- 合适且丰富的插件位置

- 按插件位置量身设计入参和出参

- 一个插件允许包含一个或多个插件位置

  以下是 teleport 的一些插件位置定义：

| 插件位置（函数）                                | 插件位置（函数）                     |
| ----------------------------------------------- | ------------------------------------ |
| PreNewPeer(*PeerConfig, *PluginContainer) error | PostNewPeer(EarlyPeer) error         |
| PostReg(*Handler) error                         | PostListen(net.Addr) error           |
| PostDial(PreSession) *Rerror                    | PostAccept(PreSession) *Rerror       |
| PreWriteCall(WriteCtx) *Rerror                  | PostWriteCall(WriteCtx) *Rerror      |
| PreWriteReply(WriteCtx) *Rerror                 | PostWriteReply(WriteCtx) *Rerror     |
| PreWritePush(WriteCtx) *Rerror                  | PostWritePush(WriteCtx) *Rerror      |
| PreReadHeader(PreCtx) error                     | PostReadCallHeader(ReadCtx) *Rerror  |
| PreReadCallBody(ReadCtx) *Rerror                | PostReadCallBody(ReadCtx) *Rerror    |
| PostReadPushHeader(ReadCtx) *Rerror             | PreReadPushBody(ReadCtx) *Rerror     |
| PostReadPushBody(ReadCtx) *Rerror               | PostReadReplyHeader(ReadCtx) *Rerror |
| PreReadReplyBody(ReadCtx) *Rerror               | PostReadReplyBody(ReadCtx) *Rerror   |
| PostDisconnect(BaseSession) *Rerror             |                                      |



上面这些函数的入参中，不带有 `*` 前缀的都是接口。

其中以 `Peer`、`Session`、 `Ctx` 为后缀的入参（接口类型），涉及到一种非常有趣、有用的 interface 用法——限制方法集。

以 `Ctx` 为例：

![tp_ctx](https://github.com/henrylee2cn/tpdoc/raw/master/02/src/ctx.png)

## 高效开发的哪些事儿

###  实现 RPC 开发范式

实现 RPC 范式的好处是代码书写简单、代码结构清晰明了、对开发者友好。

在此只贴出一个简单代码示例，不展开讨论封装细节。

- server.go

```go
package main

import (
	"fmt"
	"time"

	tp "github.com/henrylee2cn/teleport"
)

func main() {
	// graceful
	go tp.GraceSignal()

	// server peer
	srv := tp.NewPeer(tp.PeerConfig{
		CountTime:   true,
		ListenPort:  9090,
		PrintDetail: true,
	})

	// router
	srv.RouteCall(new(Math))

	// broadcast per 5s
	go func() {
		for {
			time.Sleep(time.Second * 5)
			srv.RangeSession(func(sess tp.Session) bool {
				sess.Push(
					"/push/status",
					fmt.Sprintf("this is a broadcast, server time: %v", time.Now()),
				)
				return true
			})
		}
	}()

	// listen and serve
	srv.ListenAndServe()
}

// Math handler
type Math struct {
	tp.CallCtx
}

// Add handles addition request
func (m *Math) Add(arg *[]int) (int, *tp.Rerror) {
	// test query parameter
	tp.Infof("author: %s", m.Query().Get("author"))
	// add
	var r int
	for _, a := range *arg {
		r += a
	}
	// response
	return r, nil
}
```



- client.go

```go
package main

import (
	"time"

	tp "github.com/henrylee2cn/teleport"
)

func main() {
	// log level
	tp.SetLoggerLevel("ERROR")

	cli := tp.NewPeer(tp.PeerConfig{})
	defer cli.Close()

	cli.RoutePush(new(Push))

	sess, err := cli.Dial(":9090")
	if err != nil {
		tp.Fatalf("%v", err)
	}

	var result int
	rerr := sess.Call("/math/add?author=henrylee2cn",
		[]int{1, 2, 3, 4, 5},
		&result,
	).Rerror()
	if rerr != nil {
		tp.Fatalf("%v", rerr)
	}
	tp.Printf("result: %d", result)

	tp.Printf("wait for 10s...")
	time.Sleep(time.Second * 10)
}

// Push push handler
type Push struct {
	tp.PushCtx
}

// Push handles '/push/status' message
func (p *Push) Status(arg *string) *tp.Rerror {
	tp.Printf("%s", *arg)
	return nil
}
```

