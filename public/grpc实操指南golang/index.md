# gRPC实操指南（golang）


## gRPC 实操指南（golang）

### 1 RPC(Remote Procedure Call Protocol) 

#### 1.1 什么是RPC

RPC即远程调用协议，简单来说就是调用远程的函数。

正常单机开发的情况下，我们通过函数的方式实现部分功能的解耦

```go
func sum(num1,num2 int) int {
  return num1 + num2
}
```

如上是一个最简单的求和函数，我们只需要调用函数就可以实现求和的功能。

但大部分时候函数不会这么简单，尤其对于非单机的分布式系统，远程调用就尤为重要。

#### 1.2 RPC业务场景

RPC的应用场景很广泛：

* 所有的分布式机都需要进行登陆的验证，对于所有的主机都实现相同的登陆验证逻辑维护极差，同时也失去部分分布式意义，所以从解耦的角度考虑，我们需要定义一个统一的登陆验证业务来做。
* C/S架构的传输业务，如股票软件，每天需要用户登陆的时候去服务器拉取最新的数据，或者较简单的文件传输业务，登陆验证业务，证书业务都可以使用rpc的方式
* 跨语言开发的项目，比如web业务使用golang进行开发，底层使用cpp或c，部分脚本使用py，跨语言通信可以通过RPC提供的不同语言的开发机制进行实现。

因而实际上，**RPC就是一个远程的函数**，只不过RPC协议做的就是把整个过程透明化，以使得从开发角度来看，和本地函数调用没有区别。

#### 1.3 主流RPC框架

目前主流的RPC，有ali的Dubbo，还有google的gRPC（本文主题）等

一般RPC框架如下所示：

* 客户端：客户端作为整个RPC业务的发起者，如上所说的股票软件，需要客户端主动发起请求去拉取最新的股票数据。
* 服务端：服务端接受客户端的请求，并做出相应的回应。简单来说，函数实体在服务端，数据处理在服务端。

服务端和客户端是每个RPC框架，开发者可见度最高的部分，实现RPC业务的重点就在于对C/S的设计和理解。首先，客户端一定是**率先发起请求的部分**，服务端一定是**具体处理请求的部分**。比如之前我们说的求和函数，函数主体一定是在服务端，客户端有两个数字num1，num2，向服务端发起RPC远程调用，并最后拿到求和结果。

**分清C/S很重要！！！！！**

* 客户端stub，服务端stub，可以变相的理解为应用层。主要是对客户端的rpc调用和服务端的返回进行序列化和反序列化，并进行传输，即把rpc业务抽象成tcp socket的send和receive。（gRPC使用的就是tcp，http2.0协议，建立在传输层）

![img](https://tva1.sinaimg.cn/large/008eGmZEly1gn73rrahdej31440nytdu.jpg)

### 2 gRPC

#### 2.1 什么是gRPC

gRPC是google的开源RPC框架，引用官网的一句话

```go
A high-performance, open-source universal RPC framework
```

如图，展示了gRPC跨语言开发的结构图，本文将描述golang使用grpc的过程。

严格来说，grpc通过tcp进行通信，使用http2.0协议，同时使用protobuf定义接口，因而相对于传统的restful api来说，速度更快，数据更小，接口要求更严谨。（protobuf此处不做详细介绍，[Google Protobuf](https://developers.google.com/protocol-buffers)）

![img](https://tva1.sinaimg.cn/large/008eGmZEly1gn73rpia9nj30xc0lv75g.jpg)

#### 2.2 四种gRPC服务类型

准确来说不应称为四种，实际上是因为rpc入参和出参都可实现流式或非流式，进而排列组合形成四种常用的gRPC模式。

* 简单RPC

  ![ ](https://tva1.sinaimg.cn/large/008eGmZEly1gn8417szgxj30uq0bi74s.jpg)

  即客户端发起一次请求，服务端进行响应（类似restful api）。这种模式下，rpc调用和本地函数基本相同，常常用于登陆验证，握手协议，简单业务等。

* 客户端流RPC ![image-20210201162715588](https://tva1.sinaimg.cn/large/008eGmZEly1gn8448fq4tj30t80a4gm5.jpg)

  即客户端流式发送请求，有序发送很多req包（如文件流上传），server接收到所有的req包后会检测到EOF，回发一个res并关闭连接。比如云计算应用，客户端传输众多基础数据，等待服务端计算完成并返回结果。

* 服务端流RPC

  ![image-20210201163150302](https://tva1.sinaimg.cn/large/008eGmZEly1gn84901o9rj30uu0bq74w.jpg)

  即客户端发起一次请求，服务端回发很多res包（如文件流下载），server发送完成后关闭连接。常用于数据的拉取，如请求大量数据，无法及时进行反馈，进而通过流式进行反馈。

* 双端流RPC

  ![image-20210201163606911](https://tva1.sinaimg.cn/large/008eGmZEly1gn84dfxdeqj30su0amq3k.jpg)

  即双方对话，可以实现一问一答，一问多答，多问一答等，常用于聊天室等及时通讯业务。



### 3 gRPC实操

#### 3.1 环境配置

##### 3.1.1 首先使用go get获取grpc的官方软件包

```go
 go get google.golang.org/grpc
```

##### 3.1.2 下载protobuf编译器

[protobuf代码生成工具](https://github.com/protocolbuffers/protobuf/releases)，通过proto文件生成对应的代码。

（此处需要加入环境变量，各个系统操作不同，不赘述，protoc命令能够正常使用即可）

##### 3.1.3 安装golang编译插件

我们需要.proto最终生成可用的golang代码，因而需要独立安装golang grpc的插件

```go
go get -u github.com/golang/protobuf/protoc-gen-go
```



#### 3.2 编写proto文件

protobuf的详细语法见官方文档，此处主要介绍rpc相关的内容

proto中rpc业务实际上就是一个函数，由服务端重写（overwrite）的函数，一般网上的文章会把gRPC分为四种：简单RPC，服务端流RPC，客户端流RPC，双端流RPC。实际上区别就在于rpc函数的入参和出参，接下来详细介绍一下四种情况，和一般的应用场景。

##### 3.2.1 简单RPC

```protobuf
//指定使用proto3（proto2，3有很多不同，不可混写）
syntax = "proto3";
//指定生成的go_package,简单来说就是生成的go代码使用什么包，即package proto
option go_package = ".;proto";

//定义rpc服务
//此处rpc服务的定义，一定要从服务端的角度考虑，即接受请求，处理请求并返回响应的一端
//请求接受一个LoginReq（username+password)
//响应回发一条msg（"true" or "false")
service Login{
  rpc Login(LoginReq)returns(LoginRes){}
}

message LoginReq {
  string username = 1;
  string password = 2;
}

message LoginRes {
  string msg = 1;
}
```

以上就是一个简单的RPC业务，功能是进行登陆验证。

但实际上业务不会这么简单，比如请求或者响应体特别大，肯定不能封装到一个protobuf包进行传输，因而需要使用流式传输，如请求视频资源，或者上传文件等，此时就引出了两种单向流类型，即客户端流和服务端流。

##### 3.2.2 客户端流RPC

简单来说，就是客户端请求是个流，其他和简单RPC类似。

```protobuf
syntax = "proto3";
option go_package = ".;proto";

//下载服务
//请求接受一个UploadReq（username+password)
//响应回发多条数据（"true" or "false")
service Upload{
  rpc Upload(stream UploadReq)returns(UploadRes){}
}

message UploadReq {
  string path = 1;
  int64 offset = 2;
  int64 size = 3;
  bytes data = 4;
}

message UploadRes {
  string msg = 1;
}
```

这里展示的应用场景为上传文件，即客户端指定文件路径，数据偏移量和大小，以及传输的二进制数据，打包通过protobuf发送给服务端，服务端不停接受req并写文件，最终写完之后给客户端一个反馈res。

RPC的流指的是客户端流式发送数据，本质上是分块写的思想。即每个数据包指定路径，偏移和写入大小，同时包含数据内容，每次写一个固定大小的块（如2M），流式指的是流式发送很多个块，如1G为512个2M的块。

##### 3.2.3 服务端流RPC

同上～

```protobuf
syntax = "proto3";
option go_package = ".;proto";

//下载服务
//请求接受一个DownloadReq（username+password)
//响应回发多条数据（"true" or "false")
service Download{
  rpc Download(DownloadReq)returns(stream DownloadRes){}
}

message DownloadReq {
  string path = 1;
  int64 offset = 2;
  int64 size = 3;
}

message DownloadRes {
  int64 offset = 1;
  int64 size = 2;
  bytes data = 3;
}
```

理解了客户端流，服务端流也一样的道理，客户端发送一个请求，服务端不停的发送响应，直到全部发送完成。

上述代码的场景即为下载文件，发送一次请求，请求读取某个路径下的文件，比如读取6M大小，从2M的位置开始读，响应即分为三个块，分别包含2-4，4-6，6-8的数据（块大小可以定制，仅以2M举例）。

##### 3.2.4 双端流RPC

双端流RPC就是入参，出参皆为流。一般的应用场景，如聊天室，聊天室需要维持一个长链接，连接过程中双方进行通信，都是流式的信息，类似应用场景使用双端流式的RPC。

综上，其实分类的四种RPC本质上只是RPC函数在入参和出参上有一些不同，本质上没有太大区别。但go中具体每个rpc业务的复写，针对流式和非流式处理不同，下面会详细描述，golang中如何实现除双端流之外的三种RPC（双端流同理）。



#### 3.3 生成go rpc代码

编写完proto文件就可以通过proto去生成对应的go语言代码了～

```shell
 protoc --go_out=plugins=grpc:. *.proto
```

protoc为编译器的命令，指定使用插件为grpc，输出目录为.（grpc:.）当前目录，待编译文件为*.proto。此处可以指定某个文件编译，也可以指定输出目录，这条命令会编译当前目录下的所有proto文件并生成到当前目录。

以login为例子，生成的pb.go，rpc的核心就在Client和Server的两个interface中

![截屏2021-02-01 下午4.10.02](https://tva1.sinaimg.cn/large/008eGmZEly1gn83mmn6hkj31u00jctcb.jpg)

Client interface

```go
// LoginClient is the client API for Login service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type LoginClient interface {
	Login(ctx context.Context, in *LoginReq, opts ...grpc.CallOption) (*LoginRes, error)
}
```

Server interface

```go
// LoginServer is the server API for Login service.
type LoginServer interface {
	Login(context.Context, *LoginReq) (*LoginRes, error)
}
```

**客户端调用Client interface的方法，服务端重写Server interface的方法**

一定要理解上述这句话！！！！！

例如这个列出服务器目录的rpc方法，客户端只需要创建客户端实例对象，然后调用这个方法就可以，传入req，接受res。因而我们说，对于客户端来说，此次调用和**本地函数**没有区别，但实际上是gRPC实现的远程调用，对于客户端开发是不可见的。

再说服务端，服务端需要重写Server中的方法，即服务端需要实现Server接口，对req进行处理，并生成res，同时提供ctx上下文用作并发处理。

**综上！！！！客户端是这个函数的调用者，需要调用这个函数，服务端是这个函数的定义者，需要重写这个函数**



#### 3.4 服务端

下述代码皆可从我的github库中获得源码[grpc-example](https://github.com/cjq99419/grpc-example)

##### 3.4.1 重写Server interface

###### 3.4.1.1 简单RPC

```go
package main

import (
	"context"
	"grpcExample/simple_rpc/proto"
)

type LoginServer struct {}

//判断用户名，密码是否为root,123456，验证正确即返回
func (*LoginServer)Login(ctx context.Context, req *proto.LoginReq) (*proto.LoginRes, error) {
	//为降低复杂度，此处不对ctx进行处理
	if req.Username == "root" && req.Password == "123456" {
		return &proto.LoginRes{Msg: "true"},nil
	} else {
		return &proto.LoginRes{Msg: "false"},nil
	}
}
```

此处的login函数即为server端重写的server interface的login函数，目的是处理req，生成res并返回。整个rpc业务的核心就在于服务端重写的方法，此处验证用户名和密码并返回提示信息。（仅用于grpc演示，忽略网络安全相关内容）

###### 3.4.1.2 客户端流RPC

```go
package main

import (
	"grpcExample/client_stream_rpc/proto"
	"io"
	"log"
)

type UploadServer struct{}

func (*UploadServer) Upload(uploadServer proto.Upload_UploadServer) error {
	for {
		//循环接受客户端传的流数据
		recv, err := uploadServer.Recv()
		//检测到EOF（客户端调用close）
		if err == io.EOF {
			//发送res
			err := uploadServer.SendAndClose(&proto.UploadRes{Msg: "finish"})
			if err != nil {
				return err
			}
			return nil
		} else if err != nil{
			return err
		}
		log.Printf("get a upload data package~ offset:%v, size:%v\n",recv.Offset,recv.Size)
	}
}
```

客户端流式的rpc的入参是一个server对象，可以通过这个server对象调用Recv函数获取客户端发送的每一个流。此处如果客户端关闭连接，服务端会收到一个io.EOF的error，因而此处需要对err进行判断处理，如果客户端方传输完成关闭连接等待响应，服务端检测到EOF，应调用SendAndClose发送res响应信息并关闭连接，进而完成客户端流的传输。

###### 3.4.1.3 服务端流RPC

```go
package main

import (
	"grpcExample/server_stream_rpc/proto"
	"log"
)

type DownloadServer struct{}

func (*DownloadServer) Download(req *proto.DownloadReq, downloadServer proto.Download_DownloadServer) error {
	offset := req.Offset
	//循环发送数据
	for {
		err := downloadServer.Send(&proto.DownloadRes{
			Offset: offset,
			Size:   4 * 1024,
			Data:   nil,
		})
		if err != nil {
			return err
		}
		offset += 4 * 1024
		if offset >= req.Offset + req.Size {
			break
		}
	}
	return nil
}
```

##### 3.4.2 注册服务

```go
func main() {
	lis, err := net.Listen("tcp", ":6012")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	//构建一个新的服务端对象
	s := grpc.NewServer()
	//向这个服务端对象注册服务
	proto.RegisterDownloadServer(s,&DownloadServer{})
	//注册服务端反射服务
	reflection.Register(s)

	//启动服务
	s.Serve(lis)

	//可配合ctx实现服务端的动态终止
	//s.Stop()
}
```

实际使用中，可以将这部分独立为一个模块，通过ctx控制server的启动和停止，进而灵活的控制grpc服务。



#### 3.5 客户端

##### 3.5.1 调用Client func

###### 3.5.1.1 简单RPC

```go
package main

import (
	"context"
	"google.golang.org/grpc"
	"grpcExample/simple_rpc/proto"
	"log"
	"time"
)

func main() {
	//创立grpc连接
	grpcConn, err := grpc.Dial("127.0.0.1"+":6012", grpc.WithInsecure())
	if err != nil {
		log.Fatalln(err)
	}

	//通过grpc连接创建一个客户端实例对象
	client := proto.NewLoginClient(grpcConn)

	//设置ctx超时（根据情况设定）
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	//通过client客户端对象，调用Login函数
	res, err := client.Login(ctx, &proto.LoginReq{
		Username: "root",
		Password: "123456",
	})
	if err != nil {
		log.Fatalln(err)
	}

	//输出登陆结果
	log.Println("the login answer is", res.Msg)
}
```

所以，客户端只需要维持一个实例化的client对象，通过client调用方法就可以使用RPC服务，注意和服务端不同的是，每个服务都需要一个客户端，即服务端是在一个对象上注册很多个服务，而客户端调用每个RPC业务都需要一个对应函数的Client对象。

###### 3.5.1.2 客户端流RPC

```go
package main

import (
	"context"
	"google.golang.org/grpc"
	"grpcExample/client_stream_rpc/proto"
	"log"
	"time"
)

func main(){
	//创立grpc连接
	grpcConn, err := grpc.Dial("127.0.0.1"+":6012", grpc.WithInsecure())
	if err != nil {
		log.Fatalln(err)
	}

	//通过grpc连接创建一个客户端实例对象
	client := proto.NewUploadClient(grpcConn)

	//设置ctx超时（根据情况设定）
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	//和简单rpc不同，此时获得的不是res，而是一个client的对象，通过这个连接对象去发送数据
	uploadClient,err := client.Upload(ctx)
	if err != nil {
		log.Fatalln(err)
	}

	var offset int64
	var size int64
	size = 4 * 1024

	//循环处理数据，当大于64kb退出
	for {
		err := uploadClient.Send(&proto.UploadReq{
			Path:   "../test.txt",
			Offset: offset,
			Size:   size,
			Data:   nil,
		})
		if err != nil {
			log.Fatalln(err)
		}
		offset += size
		//发送超过64KB，调用CloseAndRecv方法接收response
		if offset >= 64 * 1024 {
			res, err := uploadClient.CloseAndRecv()
			if err != nil {
				log.Fatalln(err)
			}
			log.Println("upload over~, response is ",res.Msg)
			break
		}
	}
}
```

客户端流在调用函数的时候获得的不是单纯的res对象，而是一个client对象，通过这个对象控制流的发送，并且在发送完成后主动调用CloseAndRecv去关闭连接并接受服务端的返回res。

###### 3.5.1.3 服务端流RPC

```go
package main

import (
	"context"
	"google.golang.org/grpc"
	"grpcExample/server_stream_rpc/proto"
	"log"
	"time"
)

func main(){
	//创立grpc连接
	grpcConn, err := grpc.Dial("127.0.0.1"+":6012", grpc.WithInsecure())
	if err != nil {
		log.Fatalln(err)
	}

	//通过grpc连接创建一个客户端实例对象
	client := proto.NewDownloadClient(grpcConn)

	//设置ctx超时（根据情况设定）
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	//和简单rpc不同，此时获得的不是res，而是一个client的对象，通过这个连接对象去读取数据
	downloadClient,err := client.Download(ctx,&proto.DownloadReq{
		Path:   "../test.txt",
		Offset: 0,
		Size:   64 * 1024,
	})
	if err != nil {
		log.Fatalln(err)
	}

	//循环处理数据，当监测到读取完成后退出
	for {
		res, err := downloadClient.Recv()
		if err != nil {
			log.Fatalln(err)
		}
		log.Printf("get a date package~ offset:%v, size:%v\n",res.Offset,res.Size)
		if res.Size + res.Offset >= 64 * 1024 {
			break
		}
	}

	log.Println("download over~")
}
```

此处获取的也是一个读取数据需要的对象，即客户端发送请求后的到该对象，通过该对象调用Recv来读取服务端流式发送的数据。



### 4 写在最后

**建议先理解grpc的C/S架构**

建议阅读：

* [Go gRPC教程](https://studygolang.com/articles/28205)
* [gRPC-go example](https://github.com/grpc/grpc-go/tree/master/examples)

github(vx)：cjq99419 欢迎提问和批评指正!
