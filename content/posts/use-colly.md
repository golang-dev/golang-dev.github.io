+++
date = "2019-07-12"
title = "用Golang写爬虫(六) - 使用colly"
slug = "use-colly"
tags = ["crawler", "colly"]
categories = ["crawler"]
+++

Colly是Golang世界最知名的Web爬虫框架了，它的API清晰明了，高度可配置和可扩展，支持分布式抓取，还支持多种存储后端（如内存、Redis、MongoDB等）。这篇文章记录我学习使用它的的一些感受和理解。

首先安装它：

{{< highlight bash >}}
❯ go get -u github.com/gocolly/colly/...
{{< /highlight >}}

这个`go get`和之前安装包不太一样，最后有`...`这样的省略号，它的意思是也获取这个包的子包和依赖。

### 从最简单的例子开始

Colly的文档写的算是很详细很完整的了，而且项目下的_examples目录里面也有很多爬虫例子，上手非常容易。先看我的一个例子：

{{< highlight golang >}}
package main

import (
	"fmt"

	"github.com/gocolly/colly"
)

func main() {
	c := colly.NewCollector(
		colly.UserAgent("Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"),
	)

	c.OnRequest(func(r *colly.Request) {
		fmt.Println("Visiting", r.URL)
	})

	c.OnError(func(_ *colly.Response, err error) {
		fmt.Println("Something went wrong:", err)
	})

	c.OnResponse(func(r *colly.Response) {
		fmt.Println("Visited", r.Request.URL)
	})

	c.OnHTML(".paginator a", func(e *colly.HTMLElement) {
		e.Request.Visit(e.Attr("href"))
	})

    c.OnScraped(func(r *colly.Response) {
        fmt.Println("Finished", r.Request.URL)
    })

	c.Visit("https://movie.douban.com/top250?start=0&filter=")
}
{{< /highlight >}}

这个程序就是去找豆瓣电影Top250的全部链接，如OnHTML方法的第一个函数所描述，找类名是paginator的标签下的a标签的href属性值。

运行一下：

{{< highlight bash >}}
❯ go run colly/doubanCrawler1.go
Visiting https://movie.douban.com/top250?start=0&filter=
Visited https://movie.douban.com/top250?start=0&filter=
Visiting https://movie.douban.com/top250?start=25&filter=
Visited https://movie.douban.com/top250?start=25&filter=
...
Finished https://movie.douban.com/top250?start=25&filter=
Finished https://movie.douban.com/top250?start=0&filter=
{{< /highlight >}}

在Colly中主要实体就是一个Collector对象(用colly.NewCollector创建)，Collector管理网络通信和对于响应的回调执行。Collector在初始化时可以接受多种设置项，例如这个例子里面我就设置了UserAgent的值。其他的设置项可以去看官方网站。

Collector对象接受多种回调方法，有不同的作用，按调用顺序我列出来：

1. OnRequest。请求前
2. OnError。请求过程中发生错误
3. OnResponse。收到响应后
4. OnHTML。如果收到的响应内容是HTML调用它。
5. OnXML。如果收到的响应内容是XML 调用它。写爬虫基本用不到，所以上面我没有使用它。
6. OnScraped。在OnXML/OnHTML回调完成后调用。不过官网写的是`Called after OnXML callbacks`，实际上对于OnHTML也有效，大家可以注意一下。

### 抓取条目ID和标题

还是之前的需求，先看看豆瓣Top250页面每个条目的部分HTML代码：

{{< highlight html >}}
<ol class="grid_view">
  <li>
    <div class="item">
      <div class="info">
        <div class="hd">
          <a href="https://movie.douban.com/subject/1292052/" class="">
            <span class="title">肖申克的救赎</span>
            <span class="title">&nbsp;/&nbsp;The Shawshank Redemption</span>
            <span class="other">&nbsp;/&nbsp;月黑高飞(港)  /  刺激 1995(台)</span>
          </a>
          <span class="playable">[可播放]</span>
        </div>
      </div>
    </div>
  </li>
  ....
</ol>
{{< /highlight >}}

看看这个程序怎么写的：

{{< highlight go >}}
package main

import (
	"log"
	"strings"

	"github.com/gocolly/colly"
)

func main() {
	c := colly.NewCollector(
		colly.Async(true),
		colly.UserAgent("Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"),
	)

	c.Limit(&colly.LimitRule{DomainGlob:  "*.douban.*", Parallelism: 5})

	c.OnRequest(func(r *colly.Request) {
		log.Println("Visiting", r.URL)
	})

	c.OnError(func(_ *colly.Response, err error) {
		log.Println("Something went wrong:", err)
	})

	c.OnHTML(".hd", func(e *colly.HTMLElement) {
		log.Println(strings.Split(e.ChildAttr("a", "href"), "/")[4],
			strings.TrimSpace(e.DOM.Find("span.title").Eq(0).Text()))
    })

	c.OnHTML(".paginator a", func(e *colly.HTMLElement) {
		e.Request.Visit(e.Attr("href"))
	})

	c.Visit("https://movie.douban.com/top250?start=0&filter=")
	c.Wait()
}
{{< /highlight >}}

如果你有心运行上面的那个例子，可以感受到抓取时同步的，比较慢。而这次在`colly.NewCollector`里面加了一项`colly.Async(true)`，表示抓取时异步的。在Colly里面非常方便控制并发度，只抓取符合某个(些)规则的URLS，有一句`c.Limit(&colly.LimitRule{DomainGlob:  "*.douban.*", Parallelism: 5}) `，表示限制只抓取域名是douban(域名后缀和二级域名不限制)的地址，当然还支持正则匹配某些符合的 URLS，具体的可以看官方文档。

另外Limit方法中也限制了并发是5。为什么要控制并发度呢？因为抓取的瓶颈往往来自对方网站的抓取频率的限制，如果在一段时间内达到某个抓取频率很容易被封，所以我们要控制抓取的频率。另外为了不给对方网站带来额外的压力和资源消耗，也应该控制你的抓取机制。

这个例子里面没有OnResponse方法，主要是里面没有实际的逻辑。但是多用了Wait方法，这是因为在Async为true时需要等待协程都完成再结束。但是呢，有2个OnHTML方法，一个用来确认都访问那些页面，另外一个里面就是抓取条目信息的逻辑了。也就是这部分：

{{< highlight go >}}
c.OnHTML(".hd", func(e *colly.HTMLElement) {
    log.Println(strings.Split(e.ChildAttr("a", "href"), "/")[4],
        strings.TrimSpace(e.DOM.Find("span.title").Eq(0).Text()))
})
{{< /highlight >}}

Colly的HTML解析库用的是goquery，所以写起来遵循goquery的语法就可以了。ChildAttr方法可以获得元素对应属性的值，另外一个没有列出来的ChildText，用于获得元素的文本内容。但是我们这个例子中类名为title的span标签有2个，用ChildText回直接返回2个标签的全部的值，但是Colly又没有提供ChildTexts方法（有ChildAttrs），所以只能看源码看ChildText实现改成了`strings.TrimSpace(e.DOM.Find("span.title").Eq(0).Text())`，这样就可以拿到第一个符合的文本了。

### 在Colly中使用XPath

如果你不喜欢goquery这种形式，当然也可以切换HTML解析方案，看我这个例子：

{{< highlight go >}}
import "github.com/antchfx/htmlquery"

c.OnResponse(func(r *colly.Response) {
    doc, err := htmlquery.Parse(strings.NewReader(string(r.Body)))
    if err != nil {
        log.Fatal(err)
    }
    nodes := htmlquery.Find(doc, `//ol[@class="grid_view"]/li//div[@class="hd"]`)
    for _, node := range nodes {
        url := htmlquery.FindOne(node, "./a/@href")
        title := htmlquery.FindOne(node, `.//span[@class="title"]/text()`)
        log.Println(strings.Split(htmlquery.InnerText(url), "/")[4],
            htmlquery.InnerText(title))
    }
})
{{< /highlight >}}

这次我改在OnResponse方法里面获得条目ID和标题。`htmlquery.Parse`需要接受一个实现io.Reader接口的对象，所以用了`strings.NewReader(string(r.Body))`。其他的代码是之前 [用Golang写爬虫(五) - 使用XPath](https://strconv.com/posts/web-crawler-exercise-5/)里面写过的，直接拷贝过来就可以了。

### 后记

试用Colly后就喜欢上了它，你呢？

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/colly)找到。
