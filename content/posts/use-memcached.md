+++
date = "2019-07-30"
title = "操作Memcached(附Golang代码)"
slug = "use-memcached"
tags = ["memcached"]
categories = ["memcached"]
+++

Memcached是一个高性能、分布式内存对象缓存系统。可以把数据库查询结果缓存到Memcached里，达到减少数据库访问次数的目的，从而提高动态Web应用的速度、提高可扩展性。Memcached虽然诞生的非常早，但是由于使用运维简单且作用巨大直到现在依然被很多互联网公司使用。

在Golang语言中主流的客户端是[gomemcache](https://github.com/bradfitz/gomemcache)，它是Memcached作者实现的，所以在代码质量和执行效率上肯定会有保障的。

### 安装

首先安装它：

{{< highlight bash >}}
❯ brew install memcached  # 安装Memcached
❯ go get github.com/bradfitz/gomemcache/memcache  # 下载gomemcache
{{< /highlight >}}

### 使用gomemcache

接着我们体验一下Memcached的基本操作：

{{< highlight go >}}
package main

import (
	"fmt"
	"log"

	"github.com/bradfitz/gomemcache/memcache"
)

var (
	server = "127.0.0.1:11211"
)

func main() {
	mc := memcache.New(server)
	if mc == nil {
		log.Fatal("New failed")
	}
	// Create
	err := mc.Set(&memcache.Item{Key: "a", Value: []byte("abcd")})
	if err != nil {
		log.Fatal("Set failed", err)
	}
	mc.Set(&memcache.Item{Key: "b", Value: []byte("123")})
	// Retrieve
	item, err := mc.Get("a")
	if err != nil { // err 为 ErrCacheMiss 表示没有这个键的缓存
		log.Printf("Key a 's value is %s", item.Value)
	}
	items, err := mc.GetMulti([]string{"a", "b", "c"})
	for k, v := range items {
		log.Printf("k=%s, v=%s", k, v.Value)
	}
	if err != nil {
		log.Print("GetMulti failed: %v", err)
	}
	// Delete
	err = mc.Delete("a")

	item, err = mc.Get("a")
	log.Printf("Get failed: %v, item=%v", err, item)

	err = mc.Delete("a")
	if err != nil {
		fmt.Println("Delete failed:", err)
	}
	// Incrby
	mc.Set(&memcache.Item{Key: "num", Value: []byte("1")})

	num, err := mc.Increment("num", 7) // 结果为1+7=8
	if err != nil {
		fmt.Println("Increment failed", err)
	} else {
		fmt.Println("The current value is :", num)
	}
	// Decrby
	num, err = mc.Decrement("num", 3) // 结果为8-3=5
	if err != nil {
		fmt.Println("Decrement failed", err)
	} else {
		fmt.Println("The current value is :", num)
	}
}
{{< /highlight >}}

在这个程序中体验了Set/Get/GetMulti/Delete/Increment/Decrement等操作，运行一下：

{{< highlight bash >}}
❯ go run main.go
2019/07/30 12:40:36 k=a, v=abcd
2019/07/30 12:40:36 k=b, v=123
2019/07/30 12:40:36 Get failed: memcache: cache miss, item=<nil>
Delete failed: memcache: cache miss
The current value is : 8
The current value is : 5
{{< /highlight >}}

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/memcached)找到。

### 延伸阅读

1. https://michaelheap.com/golang-using-memcached/
2. https://godoc.org/github.com/bradfitz/gomemcache/memcache
