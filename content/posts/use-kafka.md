+++
date = "2019-07-19"
title = "使用Kafka(附Golang代码)"
slug = "use-kafka"
tags = ["mq", "kafka"]
categories = ["mq"]
+++

Kafka是由LinkedIn开发的一个分布式的消息中间件。

### 安装

首先到[官网下载页面](https://kafka.apache.org/downloads)下载最新的发布版本，目前最新版是2.3.0(发布于2019年6月25日)。
{{< highlight bash >}}
❯ wget http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.3.0/kafka_2.12-2.3.0.tgz
❯ tar -xzf kafka_2.12-2.3.0.tgz
❯ cd kafka_2.12-2.3.0
{{< /highlight >}}

Kafka需要配置Zookeeper使用，Zookeeper是Hadoop和Hbase的重要组件，可以为分布式应用程序协调服务。所以需要先启动一个ZooKeeper服务器，kafka源码中自带了便捷脚本来快速简单地创建一个单节点ZooKeeper实例：

{{< highlight bash >}}
❯ bin/zookeeper-server-start.sh config/zookeeper.properties  # 放在终端1或者tmux里面
{{< /highlight >}}

然后启动Kafka服务器（启动前应该已经安装Openjdk了）：

{{< highlight bash >}}
❯ bin/kafka-server-start.sh config/server.properties
{{< /highlight >}}

这样Kafka服务器就启动了，首先体验下在终端用源码自带的脚本创建Topic，发布和消费消息等：

{{< highlight bash >}}
# 创建叫做“strconv” 的topic， 它有一个分区和一个副本
❯ bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic strconv
# 列出全部topic
❯ bin/kafka-topics.sh --list --zookeeper localhost:2181
__consumer_offsets
strconv
test
# 启动生产者，在交互模式下输入2条消息
❯ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic strconv
>Message 1
>Message 2
# 启动消费者，从开始部分消费，会将消息转储到标准输出
❯ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic strconv --from-beginning
Message 1
Message 2
{{< /highlight >}}

我就不继续演示多代理集群等用法了，可以看官方文档了解。

接着用Golang编写生产者和消费者，目前有2个主流的Golang客户端，我们挨个体验下。

顺便解释下，虽然 LinkedIn 开源了 Kafka，但是这个公司的核心语言用的是 Java，而且鲜少 Golang 应用，所以自己并没有 Golang 客户端。

### confluent-kafka-go

[confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go)是Confluent公司的开源的Golang客户端。它其
实是C/C++客户端librdkafka的Golang封装。先安装它：

{{< highlight bash >}}
❯ brew install librdkafka pkg-config
❯ go get -u gopkg.in/confluentinc/confluent-kafka-go.v1/kafka
{{< /highlight >}}

项目下的examples 目录下有很多例子供参考。我就仿着写一个例子，首先看生产者：

{{< highlight go >}}
package main

import (
	"fmt"
	"gopkg.in/confluentinc/confluent-kafka-go.v1/kafka"
	"os"
)

func main() {

	if len(os.Args) != 3 {
		fmt.Fprintf(os.Stderr, "Usage: %s <broker> <topic>\n",
			os.Args[0])
		os.Exit(1)
	}

	broker := os.Args[1]
	topic := os.Args[2]

	p, err := kafka.NewProducer(&kafka.ConfigMap{"bootstrap.servers": broker})

	if err != nil {
		fmt.Printf("Failed to create producer: %s\n", err)
		os.Exit(1)
	}

	fmt.Printf("Created Producer %v\n", p)
	deliveryChan := make(chan kafka.Event)

	value := "Hello Go!"
	err = p.Produce(&kafka.Message{
		TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
		Value:          []byte(value),
		Headers:        []kafka.Header{{Key: "myTestHeader", Value: []byte("header values are binary")}},
	}, deliveryChan)

	e := <-deliveryChan
	m := e.(*kafka.Message)

	if m.TopicPartition.Error != nil {
		fmt.Printf("Delivery failed: %v\n", m.TopicPartition.Error)
	} else {
		fmt.Printf("Delivered message to topic %s [%d] at offset %v\n",
			*m.TopicPartition.Topic, m.TopicPartition.Partition, m.TopicPartition.Offset)
	}

	close(deliveryChan)
}
{{< /highlight >}}

例子里面用了os.Args，可以获得终端位置参数的结果，`confluent_producer.go`需要传递2个参数：broker和topic。用Produce方法就发布一条消息，内容是"Hello Go!"，另外消息中有个键为myTestHeader值为"header values are binary"的头信息" 。最后看Message的结果判断是不是交付成功了，成功后会打印分区和消费进度offset。然后是消费者：

{{< highlight go >}}
package main

import (
	"fmt"
	"gopkg.in/confluentinc/confluent-kafka-go.v1/kafka"
	"os"
	"os/signal"
	"syscall"
)

func main() {

	if len(os.Args) < 4 {
		fmt.Fprintf(os.Stderr, "Usage: %s <broker> <group> <topics..>\n",
			os.Args[0])
		os.Exit(1)
	}

	broker := os.Args[1]
	group := os.Args[2]
	topics := os.Args[3:]
	sigchan := make(chan os.Signal, 1)
	signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)

	c, err := kafka.NewConsumer(&kafka.ConfigMap{
		"bootstrap.servers": broker,
		"broker.address.family": "v4",
		"group.id":              group,
		"session.timeout.ms":    6000,
		"auto.offset.reset":     "earliest"})

	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create consumer: %s\n", err)
		os.Exit(1)
	}

	fmt.Printf("Created Consumer %v\n", c)

	err = c.SubscribeTopics(topics, nil)

	run := true

	for run == true {
		select {
		case sig := <-sigchan:
			fmt.Printf("Caught signal %v: terminating\n", sig)
			run = false
		default:
			ev := c.Poll(100)
			if ev == nil {
				continue
			}

			switch e := ev.(type) {
			case *kafka.Message:
				fmt.Printf("%% Message on %s:\n%s\n",
					e.TopicPartition, string(e.Value))
				if e.Headers != nil {
					fmt.Printf("%% Headers: %v\n", e.Headers)
				}
			case kafka.Error:
				fmt.Fprintf(os.Stderr, "%% Error: %v: %v\n", e.Code(), e)
				if e.Code() == kafka.ErrAllBrokersDown {
					run = false
				}
			default:
				fmt.Printf("Ignored %v\n", e)
			}
		}
	}

	fmt.Printf("Closing consumer\n")
	c.Close()
}
{{< /highlight >}}

消费者接受三个参数：broker地址、GroupID和Topic名字。GroupID作用于消费者组，相同的GroupID标识消费者在一个组内，这些消费者协调在一起来消费订阅主题的所有分区，所以用一个新的GroupID可以再订阅主题的所有分区的消息一次。先生产2个消息：

{{< highlight bash >}}
❯ go run confluent_producer.go localhost:9092 strconv
Created Producer rdkafka#producer-1
Delivered message to topic strconv [0] at offset 0
❯ go run confluent_producer.go localhost:9092 strconv
Created Producer rdkafka#producer-1
Delivered message to topic strconv [0] at offset 1
{{< /highlight >}}

可以看到执行一次就会发布一个消息，由于strconv只有一个分区，所以输出都是`[0]`，而offset会从0开始递增。接着启动消费者：

{{< highlight bash >}}
# 终端1
❯ go run confluent_consumer.go localhost:9092 1 strconv
Created Consumer rdkafka#consumer-1
% Message on strconv[0]@0:
Hello Go!
% Headers: [myTestHeader="header values are binary"]
% Message on strconv[0]@1:
Hello Go!
% Headers: [myTestHeader="header values are binary"]
Ignored OffsetsCommitted (<nil>, [strconv[0]@2])
# 终端2
❯ go run confluent_consumer.go localhost:9092 2 strconv
Created Consumer rdkafka#consumer-1
% Message on strconv[0]@0:
Hello Go!
% Headers: [myTestHeader="header values are binary"]
% Message on strconv[0]@1:
Hello Go!
% Headers: [myTestHeader="header values are binary"]
Ignored OffsetsCommitted (<nil>, [strconv[0]@2])
{{< /highlight >}}

2个终端下，GroupID不同，所以他们各自消费了全部消息(2个)。

另外说明一下，这篇文章里所有代码部分，生产者都接收2个参数：消息代理服务器地址、Topic名字，消费者都接收三个参数：消息代理服务器地址、GroupID、Topic名字。

### Sarama

[sarama](https://github.com/Shopify/sarama)是Shopify开源的Golang客户端。第一步还是先安装它：

{{< highlight bash >}}
❯ go get -u github.com/Shopify/sarama
{{< /highlight >}}

为了演示多分区消息，这次重新创建一个Topic(叫做sarama)，建2个分区：

{{< highlight bash >}}
❯ bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 2 --topic sarama
{{< /highlight >}}

然后看生产者：

{{< highlight go >}}
package main

import (
	"fmt"
	"log"
	"math/rand"
	"os"
	"os/signal"
	"strconv"
	"time"

	"github.com/Shopify/sarama"
)

func main() {
	if len(os.Args) != 3 {
		fmt.Fprintf(os.Stderr, "Usage: %s <broker> <topic>\n",
			os.Args[0])
		os.Exit(1)
	}

	broker := os.Args[1]
	topic := os.Args[2]

	config := sarama.NewConfig()
	config.Producer.Retry.Max = 5
	config.Producer.RequiredAcks = sarama.WaitForAll
	producer, err := sarama.NewAsyncProducer([]string{broker}, config)
	if err != nil {
		panic(err)
	}

	defer func() {
		if err := producer.Close(); err != nil {
			panic(err)
		}
	}()

	signals := make(chan os.Signal, 1)
	signal.Notify(signals, os.Interrupt)

	chars := []byte("ABCDEFGHIJKLMNOPQRSTUVWXYZ")

	var enqueued, errors int
	doneCh := make(chan struct{})
	go func() {
		for {

			time.Sleep(1 * time.Second)

			buf := make([]byte, 4)
			for i := 0; i < 4; i++ {
				buf[i] = chars[rand.Intn(len(chars))]
			}

			strTime := strconv.Itoa(int(time.Now().Unix()))
			msg := &sarama.ProducerMessage{
				Topic: topic,
				Key:   sarama.StringEncoder(strTime),
				Value: sarama.StringEncoder(buf),
			}
			select {
			case producer.Input() <- msg:
				enqueued++
				fmt.Printf("Produce message: %s\n", buf)
			case err := <-producer.Errors():
				errors++
				fmt.Println("Failed to produce message:", err)
			case <-signals:
				doneCh <- struct{}{}
			}
		}
	}()

	<-doneCh
	log.Printf("Enqueued: %d; errors: %d\n", enqueued, errors)
}
{{< /highlight >}}

sarama有AsyncProducer和SyncProducer2中，这里是异步的。每次把键位当时时间戳，值为随机4个字符串的消息通过`producer.Input()`通道发布。

消费程序也是用消费组：

{{< highlight go >}}
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"sync"
	"syscall"

	"github.com/Shopify/sarama"
)

type Consumer struct {
	ready chan bool
}

func (consumer *Consumer) Setup(sarama.ConsumerGroupSession) error {
	close(consumer.ready)
	return nil
}

func (consumer *Consumer) Cleanup(sarama.ConsumerGroupSession) error {
	return nil
}

func (consumer *Consumer) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for message := range claim.Messages() {
		log.Printf("Message claimed: key = %s, value = %v, topic = %s, partition = %v, offset = %v", string(message.Key), string(message.Value), message.Topic, message.Partition, message.Offset)
		session.MarkMessage(message, "")
	}

	return nil
}

func main() {
	if len(os.Args) < 4 {
		fmt.Fprintf(os.Stderr, "Usage: %s <broker> <group> <topics..>\n",
			os.Args[0])
		os.Exit(1)
	}

	broker := os.Args[1]
	group := os.Args[2]
	topics := os.Args[3:]

	version, err := sarama.ParseKafkaVersion("2.3.0")
	if err != nil {
		log.Panicf("Error parsing Kafka version: %v", err)
	}

	config := sarama.NewConfig()
	config.Version = version
	consumer := Consumer{
		ready: make(chan bool, 0),
	}

	ctx, cancel := context.WithCancel(context.Background())
	client, err := sarama.NewConsumerGroup([]string{broker}, group, config)
	if err != nil {
		log.Panicf("Error creating consumer group client: %v", err)
	}

	wg := &sync.WaitGroup{}
	go func() {
		wg.Add(1)
		defer wg.Done()
		for {
			if err := client.Consume(ctx, topics, &consumer); err != nil {
				log.Panicf("Error from consumer: %v", err)
			}
			if ctx.Err() != nil {
				return
			}
			consumer.ready = make(chan bool, 0)
		}
	}()

	<-consumer.ready

	sigterm := make(chan os.Signal, 1)
	signal.Notify(sigterm, syscall.SIGINT, syscall.SIGTERM)
	select {
	case <-ctx.Done():
		log.Println("terminating: context cancelled")
	case <-sigterm:
		log.Println("terminating: via signal")
	}
	cancel()
	wg.Wait()
	if err = client.Close(); err != nil {
		log.Panicf("Error closing client: %v", err)
	}
}
{{< /highlight >}}

消费者要这个要复杂一些，一开始声明了Consumer结构体，包含Setup/Cleanup/ConsumeClaim方法，这个都是处理时需要的方法。另外在消费组用法下需要`sarama.ParseKafkaVersion("2.3.0")`指定Kafka版本,另外这里面也添加了信号Signal，当终端程序时会判断终止原因，如果是Ctrl+c之类的信号引起的会打印`terminating: via signal`。

另外用了消费逻辑是放在协程中运行的，用了sync.WaitGroup保证协程运行结束再关闭。还要要注意用`context.WithCancel`创建的上下文也也返回了取消函数，需要在最后取消上下文，要不然信号无法终止程序。

运行一下：

{{< highlight bash >}}
❯ go run sarama_producer.go localhost:9092 sarama
Produce message: BZGB
Produce message: CTCU
Produce message: SJFB
Produce message: DNJO
Produce message: EZQL
Produce message: JZPF
Produce message: SBZR
Produce message: FDZD

❯ go run sarama_consumer.go localhost:9092 1 sarama
2019/07/19 13:01:07 Message claimed: key = 1563512466, value = BZGB, topic = sarama, partition = 0, offset = 0
2019/07/19 13:01:07 Message claimed: key = 1563512467, value = CTCU, topic = sarama, partition = 1, offset = 0
2019/07/19 13:01:08 Message claimed: key = 1563512468, value = SJFB, topic = sarama, partition = 0, offset = 1
2019/07/19 13:01:09 Message claimed: key = 1563512469, value = DNJO, topic = sarama, partition = 1, offset = 1
2019/07/19 13:01:10 Message claimed: key = 1563512470, value = EZQL, topic = sarama, partition = 1, offset = 2
2019/07/19 13:01:11 Message claimed: key = 1563512471, value = JZPF, topic = sarama, partition = 0, offset = 2
2019/07/19 13:01:12 Message claimed: key = 1563512472, value = SBZR, topic = sarama, partition = 1, offset = 3
2019/07/19 13:01:13 Message claimed: key = 1563512473, value = FDZD, topic = sarama, partition = 0, offset = 3
{{< /highlight >}}

可以看到这些消息被相对均匀的分布到0和1这2个分区里面。另外可以在其他终端上用别的GroupID订阅消息如`go run sarama_consumer.go localhost:9092 2 sarama`(GroupID为2)。

如果多个终端的GroupID一样，不同的进程会消费绑定的对应分区里面的消息，不会重复消费。举个例子：有2个终端执行`go run sarama_consumer.go localhost:9092 1 sarama`，那么终端A会消费0分区的，终端B会消费1分区的消息。但有3个终端执行的话，由于目前只有2个分区，终端C由于没有绑定到分区，什么都不会消费

### 后记

Sarama无论是文档还是API都差很多，至于Star会比confluent-kafka-go高很多我不太理解，可能是Sarama创建时间比较早。

我决定在以后的项目中使用confluent-kafka-go，如果你有对应的在生产环境中使用的经验，欢迎留言告诉我~

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/kafka)找到。

### 延伸阅读

1. https://kafka.apache.org/quickstart
