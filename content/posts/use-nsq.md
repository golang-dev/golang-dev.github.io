+++
date = "2019-07-18"
title = "使用NSQ(附Golang代码)"
slug = "use-nsq"
tags = ["mq", "nsq"]
categories = ["mq"]
+++

上篇文章已经了解了[消息中间件相关的知识](https://strconv.com/posts/message-queue/)，这篇文章学习一下Golang语言编写的知名消息中间件[NSQ](https://github.com/nsqio/nsq)。

nsq最初是由bitly公司开源出来的一款简单易用的消息中间件，它可用于大规模系统中的实时消息服务，并且每天能够处理数亿级别的消息。它有以下特性：

1. 分布式。它提供了分布式的、去中心化且没有单点故障的拓扑结构，稳定的消息传输发布保障，能够具有高容错和高可用特性。
2. 易于扩展。它支持水平扩展，没有中心化的消息代理(Broker)，内置的发现服务让集群中增加节点非常容易。
3. 运维方便。它非常容易配置和部署，灵活性高。
4. 高度集成。现在已经有官方的Golang、Python和JavaScript客户端，社区也有了其他各个语言的客户端库方便接入，自定义客户端也非常容易。

现在开始体验它~

### 安装

首先安装它，我在Mac上用Homebrew安装：

{{< highlight bash >}}
❯ brew install nsq
{{< /highlight >}}

### 组件

nsq一共有四种组件

#### nsqlookupd

nsqlookupd是负责管理拓扑信息并提供最终一致性的发现服务的守护进程(daemon)。在终端1启动它：

{{< highlight bash >}}
❯ nsqlookupd
[nsqlookupd] 2019/07/18 11:42:16.876296 INFO: nsqlookupd v1.1.0 (built w/go1.11)
[nsqlookupd] 2019/07/18 11:42:16.876864 INFO: HTTP: listening on [::]:4161
[nsqlookupd] 2019/07/18 11:42:16.876868 INFO: TCP: listening on [::]:4160
{{< /highlight >}}

默认HTTP接口监听4161，TCP接口监听4160。

#### nsqd

nsqd是一个负责接收、排队、投递消息给客户端的守护进程。客户端通过查询 nsqlookupd 来发现指定话题（topic）的nsqd生产者，nsqd节点会广播话题（topic）和通道（channel）信息。数据流模型如下：

![](https://f.cloud.github.com/assets/187441/1700696/f1434dc8-6029-11e3-8a66-18ca4ea10aca.gif)

单个nsqd可以有多个topic，每个topic可以有多个channel。channel接收这个topic所有消息的副本，从而实现多播分发，而channel上的每个消息被分发给它的订阅者，从而实现负载均衡。

在终端2启动nsqd：

{{< highlight bash >}}
❯ nsqd --lookupd-tcp-address=127.0.0.1:4160
...
[nsqd] 2019/07/18 11:47:46.427184 INFO: HTTP: listening on [::]:4151
[nsqd] 2019/07/18 11:47:46.427195 INFO: TCP: listening on [::]:4150
[nsqd] 2019/07/18 11:47:46.427203 INFO: LOOKUP(127.0.0.1:4160): adding peer
[nsqd] 2019/07/18 11:47:46.427355 INFO: LOOKUP connecting to 127.0.0.1:4160
...
{{< /highlight >}}

nsqd通过tcp端口连接到了nsqlookupd，它自己在4151接受HTTP请求，在4150接受TCP请求。

#### nsqadmin

nsqadmin 是一套WEB管理UI，用来汇集集群的实时统计，并执行不同的管理任务。在终端3启动它：

{{< highlight bash >}}
❯ nsqadmin --lookupd-http-address=127.0.0.1:4161
[nsqadmin] 2019/07/18 11:54:23.125392 INFO: nsqadmin v1.1.0 (built w/go1.11)
[nsqadmin] 2019/07/18 11:54:23.128755 INFO: HTTP: listening on [::]:4171
{{< /highlight >}}

浏览器打开`http://localhost:4171`就能访问了，需要注意，管理UI可以按需启动。

#### 功能工具

安装nsq后会增加nsq\_stat/nsq\_tail/nsq\_to\_file等功能工具，这些实用程序以数据流的形式提供了通用功能和内部检查，稍后能体验到。

### 命令行体验

{{< highlight bash >}}
❯ curl -d 'hello world 1' 'http://127.0.0.1:4151/pub?topic=test'  # 在终端4执行
OK  # 发布消息到nsqd，用Rest API完成，看参数表示话题是test。由于还没有test这个话题，会先创建话题再接收消息。
❯ nsq_tail --lookupd-http-address=127.0.0.1:4161 --topic=test  # 在终端5执行，一会再来看
# 回到终端4
❯ curl -d 'hello world 2' 'http://127.0.0.1:4151/pub?topic=test'
OK
~
❯ curl -d 'hello world 3' 'http://127.0.0.1:4151/pub?topic=test'
OK
~
# 回到终端5
❯ nsq_tail --lookupd-http-address=127.0.0.1:4161 --topic=test
2019/07/18 12:06:24 Adding consumer for topic: test
...
hello world 2
hello world 3
{{< /highlight >}}

回到终端5，可以看到接受(消费)了启动nsq\_tail后发布的2个消息，这就是功能工具的作用，另外如`nsq_to_file`是把消息发到文件。

### nsq的Go客户端使用

首先安装go-nsq:

{{< highlight bash >}}
❯ go get github.com/nsqio/go-nsq
{{< /highlight >}}

先看生产者:

{{< highlight go >}}
package main

import (
	"github.com/nsqio/go-nsq"
	"log"
	"math/rand"
	"time"
)

func main() {
	config := nsq.NewConfig()
	w, err := nsq.NewProducer("127.0.0.1:4150", config)

	if err != nil {
		log.Panic(err)
	}

	chars := []byte("ABCDEFGHIJKLMNOPQRSTUVWXYZ")

	for {
		buf := make([]byte, 4)
		for i := 0; i < 4; i++ {
			buf[i] = chars[rand.Intn(len(chars))]
		}
		log.Printf("Pub: %s", buf)
		err = w.Publish("test", buf)
		if err != nil {
			log.Panic(err)
		}
		time.Sleep(time.Second * 1)
	}

	w.Stop()
}
{{< /highlight >}}

NewProducer的第一个参数就是nsqd的地址，在这里做了个无限for循环，每次随机4个byte发布到test话题里面。

接着看消费者代码：

{{< highlight go >}}
package main

import (
	"log"
	"sync"

	"github.com/nsqio/go-nsq"
)

func main() {

	wg := &sync.WaitGroup{}
	wg.Add(1000)

	config := nsq.NewConfig()
	q, _ := nsq.NewConsumer("test", "ch", config)
	q.AddHandler(nsq.HandlerFunc(func(message *nsq.Message) error {
		log.Printf("Got a message: %s", message.Body)
		wg.Done()
		return nil
	}))
	err := q.ConnectToNSQD("127.0.0.1:4150")
	if err != nil {
		log.Panic(err)
	}
	wg.Wait()

}
{{< /highlight >}}

一开始通过sync.WaitGroup安排了1000个待执行的等待组，NewConsumer的第一个参数是话题test，第二是通道名字，然后用AddHandler添加一个消费处理函数，在处理函数中会打印这个消息。

现在就可以体验了，首先启动消费者，再启动发布者。

{{< highlight bash >}}
❯ go run consumer.go
2019/07/18 15:29:29 INF    1 [test/ch] (127.0.0.1:4150) connecting to nsqd
2019/07/18 15:29:37 Got a message: ZGBA
2019/07/18 15:29:38 Got a message: ICMR
2019/07/18 15:29:39 Got a message: AJWW
2019/07/18 15:29:40 Got a message: HTHC
2019/07/18 15:29:41 Got a message: TCUA
...
❯ go run producer.go
2019/07/18 15:29:36 INF    1 (127.0.0.1:4150) connecting to nsqd
2019/07/18 15:29:37 Pub: ZGBA
2019/07/18 15:29:38 Pub: ICMR
2019/07/18 15:29:39 Pub: AJWW
2019/07/18 15:29:40 Pub: HTHC
2019/07/18 15:29:41 Pub: TCUA
...
{{< /highlight >}}

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/nsq)找到。

### 延伸阅读

1. https://nsq.io/overview/quick_start.html
