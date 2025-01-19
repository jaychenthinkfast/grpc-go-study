# 4.4 gRPC 入门

    文档来源：https://github.com/chai2010/advanced-go-programming-book，  根据文档形成代码示例，有部分修改

gRPC 是 Google 公司基于 Protobuf 开发的跨语言的开源 RPC 框架。gRPC 基于 HTTP/2 协议设计，可以基于一个 HTTP/2 连接提供多个服务，对于移动设备更加友好。本节将讲述 gRPC 的简单用法。

## 4.4.1 gRPC 技术栈

Go 语言的 gRPC 技术栈如图 4-1 所示：

![](image/ch4-1-grpc-go-stack.png)

*图 4-1 gRPC 技术栈*

最底层为 TCP 或 Unix Socket 协议，在此之上是 HTTP/2 协议的实现，然后在 HTTP/2 协议之上又构建了针对 Go 语言的 gRPC 核心库。应用程序通过 gRPC 插件生产的 Stub 代码和 gRPC 核心库通信，也可以直接和 gRPC 核心库通信。

## 4.4.2 gRPC 入门

如果从 Protobuf 的角度看，gRPC 只不过是一个针对 service 接口生成代码的生成器。我们在本章的第二节中手工实现了一个简单的 Protobuf 代码生成器插件，只不过当时生成的代码是适配标准库的 RPC 框架的。现在我们将学习 gRPC 的用法。

创建 hello/hello.proto 文件，定义 HelloService 接口：

```proto
syntax = "proto3";

package hello;

message String {
	string value = 1;
}

service HelloService {
	rpc Hello (String) returns (String);
}
```

使用 protoc-gen-go 内置的 gRPC 插件生成 gRPC 代码：

```
//README.md 同级目录下执行
protoc --go_out=plugins=grpc:. hello.proto
```

gRPC 插件会为服务端和客户端生成不同的接口：

```go
type HelloServiceServer interface {
	Hello(context.Context, *String) (*String, error)
}

type HelloServiceClient interface {
	Hello(context.Context, *String, ...grpc.CallOption) (*String, error)
}
```

gRPC 通过 context.Context 参数，为每个方法调用提供了上下文支持。客户端在调用方法的时候，可以通过可选的 grpc.CallOption 类型的参数提供额外的上下文信息。

基于服务端的 HelloServiceServer 接口可以重新实现 HelloService 服务：

server/main.go
```go
type HelloServiceImpl struct{}

func (p *HelloServiceImpl) Hello(
	ctx context.Context, args *String,
) (*String, error) {
	reply := &String{Value: "hello:" + args.GetValue()}
	return reply, nil
}
```

gRPC 服务的启动流程和标准库的 RPC 服务启动流程类似：

server/main.go
```go
func main() {
	grpcServer := grpc.NewServer()
	RegisterHelloServiceServer(grpcServer, new(HelloServiceImpl))

	lis, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal(err)
	}
	grpcServer.Serve(lis)
}
```

首先是通过 `grpc.NewServer()` 构造一个 gRPC 服务对象，然后通过 gRPC 插件生成的 RegisterHelloServiceServer 函数注册我们实现的 HelloServiceImpl 服务。然后通过 `grpcServer.Serve(lis)` 在一个监听端口上提供 gRPC 服务。

然后就可以通过客户端连接 gRPC 服务了：

client/main.go
```go
func main() {
	conn, err := grpc.Dial("localhost:1234", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	client := NewHelloServiceClient(conn)
	reply, err := client.Hello(context.Background(), &String{Value: "hello"})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(reply.GetValue())
}
```

其中 grpc.Dial 负责和 gRPC 服务建立连接，然后 NewHelloServiceClient 函数基于已经建立的连接构造 HelloServiceClient 对象。返回的 client 其实是一个 HelloServiceClient 接口对象，通过接口定义的方法就可以调用服务端对应的 gRPC 服务提供的方法。

gRPC 和标准库的 RPC 框架有一个区别，gRPC 生成的接口并不支持异步调用。不过我们可以在多个 Goroutine 之间安全地共享 gRPC 底层的 HTTP/2 连接，因此可以通过在另一个 Goroutine 阻塞调用的方式模拟异步调用。

## 4.4.3 gRPC 流

RPC 是远程函数调用，因此每次调用的函数参数和返回值不能太大，否则将严重影响每次调用的响应时间。因此传统的 RPC 方法调用对于上传和下载较大数据量场景并不适合。同时传统 RPC 模式也不适用于对时间不确定的订阅和发布模式。为此，gRPC 框架针对服务器端和客户端分别提供了流特性。

服务端或客户端的单向流是双向流的特例，我们在 HelloService 增加一个支持双向流的 Channel 方法：

```proto
service HelloService {
	rpc Hello (String) returns (String);

	rpc Channel (stream String) returns (stream String);
}
```

关键字 stream 指定启用流特性，参数部分是接收客户端参数的流，返回值是返回给客户端的流。

重新生成代码可以看到接口中新增加的 Channel 方法的定义：

```go
type HelloServiceServer interface {
	Hello(context.Context, *String) (*String, error)
	Channel(HelloService_ChannelServer) error
}
type HelloServiceClient interface {
	Hello(ctx context.Context, in *String, opts ...grpc.CallOption) (
		*String, error,
	)
	Channel(ctx context.Context, opts ...grpc.CallOption) (
		HelloService_ChannelClient, error,
	)
}
```

在服务端的 Channel 方法参数是一个新的 HelloService_ChannelServer 类型的参数，可以用于和客户端双向通信。客户端的 Channel 方法返回一个 HelloService_ChannelClient 类型的返回值，可以用于和服务端进行双向通信。

HelloService_ChannelServer 和 HelloService_ChannelClient 均为接口类型：

```go
type HelloService_ChannelServer interface {
	Send(*String) error
	Recv() (*String, error)
	grpc.ServerStream
}

type HelloService_ChannelClient interface {
	Send(*String) error
	Recv() (*String, error)
	grpc.ClientStream
}
```

可以发现服务端和客户端的流辅助接口均定义了 Send 和 Recv 方法用于流数据的双向通信。

现在我们可以实现流服务：

server/main.go
```go
func (p *HelloServiceImpl) Channel(stream HelloService_ChannelServer) error {
	for {
		args, err := stream.Recv()
		if err != nil {
			if err == io.EOF {
				return nil
			}
			return err
		}

		reply := &String{Value: "hello:" + args.GetValue()}

		err = stream.Send(reply)
		if err != nil {
			return err
		}
	}
}
```

服务端在循环中接收客户端发来的数据，如果遇到 io.EOF 表示客户端流被关闭，如果函数退出表示服务端流关闭。生成返回的数据通过流发送给客户端，双向流数据的发送和接收都是完全独立的行为。需要注意的是，发送和接收的操作并不需要一一对应，用户可以根据真实场景进行组织代码。

客户端需要先调用 Channel 方法获取返回的流对象：

client/main.go
```go
stream, err := client.Channel(context.Background())
if err != nil {
	log.Fatal(err)
}
```

在客户端我们将发送和接收操作放到两个独立的 Goroutine。首先是向服务端发送数据：

client/main.go
```go
go func() {
	for {
		if err := stream.Send(&String{Value: "hi"}); err != nil {
			log.Fatal(err)
		}
		time.Sleep(time.Second)
	}
}()
```

然后在循环中接收服务端返回的数据：

client/main.go
```go
for {
	reply, err := stream.Recv()
	if err != nil {
		if err == io.EOF {
			break
		}
		log.Fatal(err)
	}
	fmt.Println(reply.GetValue())
}
```

这样就完成了完整的流接收和发送支持。