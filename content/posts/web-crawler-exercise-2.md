+++
date = "2019-07-08"
title = "用Golang写爬虫(二) - 并发"
slug = "web-crawler-exercise-2"
tags = ["crawler", "concurrency"]
categories = ["crawler"]
+++

在上篇文章里面我用Go写了一个爬虫，但是它的执行是串行的，效率很低，这篇文章把它改成并发的。由于这个程序只抓取10个页面，大概1s多就完成了，为了对比我们先给之前的`doubanCrawler1.go`加一点Sleep的代码，让它跑的「慢」些：

{{< highlight go >}}
func parseUrls(url string) {
    ...
	time.Sleep(2 * time.Second)
}
{{< /highlight >}}

这样运行起来大体可以计算出来程序跑完约需要21s+，我们运行一下试试：

{{< highlight bash >}}
❯ go run doubanCrawler2.go
...
Took 21.315744555s
{{< /highlight >}}

已经很慢了。接着我们开始让它变得更快~

### goroutine的错误用法

先修改成用Go原生支持的并发方案goroutine来做。在Golang中使用goroutine非常方便，直接使用Go关键字就可以，我们看一个版本：

{{< highlight go >}}
func main() {
	start := time.Now()
	for i := 0; i < 10; i++ {
		go parseUrls("https://movie.douban.com/top250?start=" + strconv.Itoa(25*i))
	}
	elapsed := time.Since(start)
	fmt.Printf("Took %s", elapsed)
}
{{< /highlight >}}

就是在parseUrls函数前加了go关键字。但其实这样就是不对的，运行的话不会抓取到任何结果。因为协程刚生成，整个程序就结束了，goroutine还没抓完呢。怎么办呢？可以结束前Sleep一个时间，这个时间应该要大于所有goroutine执行最慢的那个，这样就保证了全部协程都能正常运行完再结束(doubanCrawler3.go)：

{{< highlight go >}}
func main() {
    start := time.Now()
    for i := 0; i < 10; i++ {
        go parseUrls("https://movie.douban.com/top250?start=" + strconv.Itoa(25*i))
    }
    time.Sleep(4 * time.Second)
    elapsed := time.Since(start)
    fmt.Printf("Took %s", elapsed)
}
{{< /highlight >}}

在for循环后加了Sleep 4秒。运行一下：

{{< highlight bash >}}
❯ go run doubanCrawler3.go
...
Took 4.000849896s  # 这个时间大致就是4s
{{< /highlight >}}

当然这个Sleep的时间不好控制，假设某次请求花的时间超了，总体时间超过4s程序结束了但这个协程其实还没运行结束；而假如全部goroutine都在3秒（2秒固定Sleep+1秒程序运行）结束，那么多Sleep的一秒就浪费了！

### goroutine的正确用法

那怎么用goroutine呢？有没有像Python多进程/线程的那种等待子进/线程执行完的join方法呢？当然是有的，可以让Go 协程之间信道（channel）进行通信：从一端发送数据，另一端接收数据，信道需要发送和接收配对，否则会被阻塞：

{{< highlight go >}}
func parseUrls(url string, ch chan bool) {
    ...
    ch <- true
}

func main() {
    start := time.Now()
    ch := make(chan bool)
    for i := 0; i < 10; i++ {
        go parseUrls("https://movie.douban.com/top250?start="+strconv.Itoa(25*i), ch)
    }

    for i := 0; i < 10; i++ {
        <-ch
    }

    elapsed := time.Since(start)
    fmt.Printf("Took %s", elapsed)
}
{{< /highlight >}}

在上面的改法中，parseUrls都是在goroutine中执行，但是注意函数签名改了，多接收了信道参数ch。当函数逻辑执行结束会给信道ch发送一个布尔值。

而在main函数中，在用一个for循环，<- ch 会等待接收数据（这里只是接收，相当于确认任务完成）。这样的流程就实现了一个更好的并发方案：

{{< highlight bash >}}
❯ go run doubanCrawler4.go
...
Took 2.450826901s  # 这个时间比之前的写死了4s的那个优化太多了！
{{< /highlight >}}

### sync.WaitGroup

还有一个好的方案sync.WaitGroup。我们这个程序只是打印抓到的对应内容，所以正好用WaitGroup：等待一组并发操作完成：

{{< highlight go >}}
import (
	...
	"sync"
)
...
func main() {
	start := time.Now()
	var wg sync.WaitGroup
	wg.Add(10)

	for i := 0; i < 10; i++ {
		go func(i int) {
			defer wg.Done()
			parseUrls("https://movie.douban.com/top250?start="+strconv.Itoa(25*i))
		}(i)
	}

	wg.Wait()

	elapsed := time.Since(start)
	fmt.Printf("Took %s", elapsed)
}
{{< /highlight >}}

一开始我们给调用wg.Add添加要等待的goroutine量，我们的页面总数就是10，所以这里可以直接写出来。

另外这里使用了defer关键字来调用wg.Done，以确保在退出goroutine的闭包之前，向WaitGroup表明了我们已经退出。由于要执行wg.Done和parseUrls2件事，所以不能直接用go关键字，需要把语句包一下。不过要注意，在闭包中需要把参数i作为func的参数传入，要不然i会使用最后一次循环的那个值：

{{< highlight go >}}
// 错误代码👇
for i := 0; i < 10; i++ {
    go func() {
        defer wg.Done()
        parseUrls("https://movie.douban.com/top250?start="+strconv.Itoa(25*i))
    }()
}
❯ go run crawler/doubanCrawler5.go
Fetch Url https://movie.douban.com/top250?start=75
Fetch Url https://movie.douban.com/top250?start=250
Fetch Url https://movie.douban.com/top250?start=250
Fetch Url https://movie.douban.com/top250?start=250
Fetch Url https://movie.douban.com/top250?start=250
Fetch Url https://movie.douban.com/top250?start=250
Fetch Url https://movie.douban.com/top250?start=250
Fetch Url https://movie.douban.com/top250?start=250
Fetch Url https://movie.douban.com/top250?start=250
Fetch Url https://movie.douban.com/top250?start=200
...
{{< /highlight >}}

咦，看代码，i在等于9的时候循环结束，start应该是225(9 *  25)，但为什么250呢？这是因为在最后还有个`i++`，虽然不符合条件没有进行循环，但是i的值确实发生了改变！

在这样的用法中，WaitGroup相当于是一个协程安全的并发计数器：调用Add增加计数，调用Done减少计数。调用Wait会阻塞并等待至计数器归零。这样也实现了并发和等待全部goroutine执行完成：

{{< highlight bash >}}
❯ go run doubanCrawler5.go
...
Took 2.382876529s  # 这个时间和之前的信道用法效果一致！
{{< /highlight >}}

### 后记

好啦，这篇文章先写到这里啦~

### 代码地址

完整代码都可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/crawler)找到。
