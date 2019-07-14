+++
date = "2019-07-14T10:10:10"
title = "在Golang中实现RPC"
slug = "rpc"
tags = ["crawler", "rpc"]
categories = ["rpc"]
+++

### 什么是RPC

远程过程调用（Remote Procedure Call，缩写为 RPC）是一个计算机通信协议。`远程调用`是因为被调用方法的具体实现不在程序运行本地，而是在远程服务器上。需要将对象名、函数名、参数等传递给远程服务器，服务器将处理结果返回给客户端。RPC 的消息可以通过  TCP、UDP 或者  HTTP等传输。

在Golang中实现RPC的方式大体有三种，分别来看。

### net/rpc

Golang官方的`net/rpc`包使用`encoding/gob`进行编解码，支持tcp或http数据传输方式。但是由于gob编码是Golang独有的所以它只支持Golang开发的服务器与客户端之间的交互。

RPC采用客户机/服务器模式。先看一下server例子：

{{< highlight golang >}}
package main

import (
	"fmt"
	"log"
	"net"
	"net/rpc"
)

type Listener int

type Reply struct {
	Data string
}

func (l *Listener) GetLine(line []byte, reply *Reply) error {
	rv := string(line)
	fmt.Printf("Receive: %v\n", rv)
	*reply = Reply{rv}
	return nil
}

func main() {
	addy, err := net.ResolveTCPAddr("tcp", "0.0.0.0:12345")
	if err != nil {
		log.Fatal(err)
	}

	inbound, err := net.ListenTCP("tcp", addy)
	if err != nil {
		log.Fatal(err)
	}

	listener := new(Listener)
	rpc.Register(listener)
	rpc.Accept(inbound)
}
{{< /highlight >}}

在这里例子中给Listener添加了GetLine方法，这个方法的返回值必须是error类型，第一个参数是客户端传来的内容，第二个参数是响应的内容：需要是一个指针，所以定义了一个叫做Reply的结构体，只有一个Data成员用于存储响应内容。

在main函数中，首先用`net.ResolveTCPAddr`和`net.ListenTCP`创建一个TCP连接，监听所有地址的12345端口。最后用`rpc.Register`注册监听对象，接受上面的tcp连接的请求。

然后是客户端：

{{< highlight golang >}}
package main

import (
	"bufio"
	"log"
	"net/rpc"
	"os"
)

type Reply struct {
    Data string
}

func main() {
	client, err := rpc.Dial("tcp", "localhost:12345")
	if err != nil {
		log.Fatal(err)
	}

	in := bufio.NewReader(os.Stdin)
	for {
		line, _, err := in.ReadLine()
		if err != nil {
			log.Fatal(err)
		}
		var reply Reply
		err = client.Call("Listener.GetLine", line, &reply)
		if err != nil {
			log.Fatal(err)
		}
		log.Printf("Reply: %v, Data: %v", reply, reply.Data)
	}
}
{{< /highlight >}}

客户端用`rpc.Dial`创建连接到服务端的主机和端口，然后是一个永久的for循环，ReadLine方法会接收终端输入，如果写了一些内容回车，就会执行`client.Call`，开始过程调用，调用成功后，reply就被写入数据，可以拿到reply.Data了（其实就是输入什么，收到什么）。体验一下：

{{< highlight bash >}}
❯ go run simple_server.go
Receive: hi
Receive: haha

❯ go run simple_client.go
hi
2019/07/14 18:19:14 Reply: {hi}, Data: hi
haha
2019/07/14 18:19:15 Reply: {haha}, Data: haha
{{< /highlight >}}

### net/rpc/jsonrpc

使用`net/rpc`实现的RPC只能使用Golang语言编写的服务端/客户端之间交互，所以Go语言标准库通过`net/rpc/jsonrpc`这个包支持跨语言的RPC。要实现上面一样的效果，代码主要是改了main的rpc.Accept部分就可以了：

{{< highlight golang >}}
import "net/rpc/jsonrpc"

func main() {
    addy, err := net.ResolveTCPAddr("tcp", "0.0.0.0:12345")
    if err != nil {
        log.Fatal(err)
    }

    inbound, err := net.ListenTCP("tcp", addy)
    if err != nil {
        log.Fatal(err)
    }

    listener := new(Listener)
    rpc.Register(listener)
    for {
        conn, err := inbound.Accept()
        if err != nil {
            continue
        }
        jsonrpc.ServeConn(conn)
    }
}
{{< /highlight >}}

客户端部分也只是改动rpc.Dial部分：

{{< highlight golang >}}
func main() {
    client, err := jsonrpc.Dial("tcp", "localhost:12345") // 只改动这一行
    if err != nil {
        log.Fatal(err)
    }

    in := bufio.NewReader(os.Stdin)
    for {
        line, _, err := in.ReadLine()
        if err != nil {
            log.Fatal(err)
        }
        var reply Reply
        err = client.Call("Listener.GetLine", line, &reply)
        if err != nil {
            log.Fatal(err)
        }
        log.Printf("Reply: %v, Data: %v", reply, reply.Data)
    }
}
{{< /highlight >}}

 json-rpc是基于TCP协议实现的，目前它还不支持HTTP方式。运行效果和上面的一样：

{{< highlight bash >}}
❯ go run simple_jsonrpc_server.go
Receive: hi
Receive: haha

❯ go run simple_jsonrpc_client.go
hi
2019/07/14 20:22:02 Reply: {hi}, Data: hi
haha
2019/07/14 20:22:03 Reply: {haha}, Data: haha
{{< /highlight >}}

请求的json数据对象在内部对应两个结构体：客户端是clientRequest，服务端是serverRequest。大抵是这样

{{< highlight golang >}}
type serverRequest struct {
	Method string           `json:"method"`
	Params *json.RawMessage `json:"params"`
	Id     *json.RawMessage `json:"id"`
}

type clientRequest struct {
	Method string         `json:"method"`
	Params [1]interface{} `json:"params"`
	Id     uint64         `json:"id"`
}
{{< /highlight >}}

所以我们可以基于这个格式用其他语言拼消息。简单一点，在命令行试试：

{{< highlight bash >}}
❯ echo -n "hihi" |base64  # 参数需要用base64编码
aGloaQ==

~/strconv.code/rpc master*
❯ echo -e '{"method": "Listener.GetLine","params": ["aGloaQ=="], "id": 0}' | nc localhost 12345
{"id":0,"result":{"Data":"hihi"},"error":null}
{{< /highlight >}}

看到了吧，可以拿到对应的结果。其中id不是必须的，但是可以基于id在并发高或者异步调用中对应某一次调用。

### gRPC

jsonrpc虽然可以支持跨语言但是不支持HTTP传输，而且性能不高，所以在实际生产环境中都不会用标准库里面的方式，而是选择Thrift、gRPC等方案。

gRPC是Google开源的高性能、通用的开源RPC框架，其主要面向移动应用开发并基于HTTP/2协议标准而设计，基于ProtoBuf序列化协议开发，支持Python、Golang、Java等众多开发语言。

#### ProtoBuf协议

Protobuf是Protocol Buffers的简称，它是Google公司开发的一种数据描述语言，类似于XML、JSON等数据描述语言，它非常轻便高效，很适合做数据存储或 RPC 数据交换格式。由于它一次定义，可生成多种语言的代码，非常适合用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。首先安装它和Go语言的生成工具：

{{< highlight bash >}}
❯ brew install protobuf
❯ protoc --version
libprotoc 3.7.1
go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
{{< /highlight >}}

然后按照最新的proto3协议写一个描述文件

{{< highlight golang >}}
syntax = "proto3";

package simple;

// 请求
message SimpleRequest {
    string data = 1;
}

// 响应
message SimpleResponse {
    string data = 1;
}

// rpc方法
service Simple {
    rpc  GetLine (SimpleRequest) returns (SimpleResponse);
}
{{< /highlight >}}

其中描述了请求SimpleRequest（只有一个字符串参数data）、响应SimpleResponse（只有一个字符串参数data）和rpc方法。Simple服务只有一个GetLine方法，请求是SimpleRequest，响应SimpleResponse。然后基于.proto文件生成数据操作代码：

{{< highlight bash >}}
❯ mkdir src/simple
❯ protoc --go_out=plugins=grpc:src/simple simple.proto
❯ ll src/simple
total 8.0K
-rw-r--r-- 1 dongwm staff 7.0K Jul 14 21:43 simple.pb.go
{{< /highlight >}}

执行命令完成，就在`src/simple`下生成了一个叫做`simple.pb.go`的文件，它支持gRPC。放在src/simple目录下是为了让它作为一个包（package）。

### 体验 gRPC

首先需要安装 gRPC

{{< highlight bash >}}
❯ go get -u google.golang.org/grpc
{{< /highlight >}}

然后就可以基于`src/simple`这个包写代码了

{{< highlight golang >}}
package main

import (
	"fmt"
	"log"
	"net"

	pb "./src/simple"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
)

type Listener int

func (l *Listener) GetLine(ctx context.Context, in *pb.SimpleRequest) (*pb.SimpleResponse, error) {
	rv := in.Data
	fmt.Printf("Receive: %v\n", rv)
	return &pb.SimpleResponse{Data: rv}, nil
}

func main() {
	addy, err := net.ResolveTCPAddr("tcp", "0.0.0.0:12345")
	if err != nil {
		log.Fatal(err)
	}

	inbound, err := net.ListenTCP("tcp", addy)
	if err != nil {
		log.Fatal(err)
	}

	s := grpc.NewServer()
	listener := new(Listener)
    pb.RegisterSimpleServer(s, listener)
    s.Serve(inbound)
}
{{< /highlight >}}

其中`pb "./src/simple"`表示把当前目录下的src/simple作为一个包，给它取了个别名pb，内容就来自前面创建的simple.pb.go。

GetLine方法要重新定义，它的第一个参数是context.Context，第二个是`*pb.SimpleRequest`（在proto文件中定义的请求），返回的结果是(*pb.SimpleResponse, error)，`pb.SimpleResponse`在proto文件中定义的响应。另外要注意，虽然在proto文件中SimpleRequest和SimpleResponse的成员data是小写开头的，但是使用时要首字母大写（Data）。

再看客户端：

{{< highlight golang >}}
package main

import (
	"bufio"
	"log"
	"os"

	pb "./src/simple"
    "golang.org/x/net/context"
    "google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial("localhost:12345", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}

	c := pb.NewSimpleClient(conn)

	in := bufio.NewReader(os.Stdin)
	for {
		line, _, err := in.ReadLine()
		if err != nil {
			log.Fatal(err)
		}
		reply, err := c.GetLine(context.Background(), &pb.SimpleRequest{Data: string(line)})
		if err != nil {
			log.Fatal(err)
		}
		log.Printf("Reply: %v, Data: %v", reply, reply.Data)
	}
}
{{< /highlight >}}

首先用`grpc.Dial("localhost:12345", grpc.WithInsecure()) `创建连接，然后用`pb.NewSimpleClient`创建`simpleClient`对象。为什么叫`SimpleClient`呢？其实格式是`XXXClient`, XXX是前面在proto文件中定义的`service Simple`中的`Simple`。

rpc调用时要这样写：`reply, err := c.GetLine(context.Background(), &pb.SimpleRequest{Data: string(line)})`，GetLine就是proto文件中定义的方法（`rpc  GetLine (SimpleRequest) returns (SimpleResponse)`），第一个参数`context.Background()`，第二个参数是请求，由于line是[]byte类型的，所以需要用string转换成字符串。响应reply是SimpleResponse对象，可以从reply.Data中获得返回的实际结果:

{{< highlight bash >}}
❯ go run grpc_server.go
Receive: hi
Receive: Haha
Receive: vvv

❯ go run grpc_client.go
hi
2019/07/15 07:57:48 Reply: data:"hi" , Data: hi
Haha
2019/07/15 07:57:51 Reply: data:"Haha" , Data: Haha
vvv
2019/07/15 07:57:53 Reply: data:"vvv" , Data: vvv
{{< /highlight >}}

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/rpc)找到。

### 延伸阅读

1. https://books.studygolang.com/NPWG_zh/Text/chapter-rpc.html
2. https://golang.org/pkg/net/rpc/
3. https://developers.google.com/protocol-buffers/docs/proto3
4. https://github.com/grpc/grpc-go
