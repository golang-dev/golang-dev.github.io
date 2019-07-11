+++
date = "2019-07-09"
title = "用Golang写爬虫(三) - 使用goquery"
slug = "web-crawler-exercise-3"
tags = ["crawler", "goquery"]
categories = ["crawler"]
+++

在写爬虫的时候，想要对HTML内容进行选择和查找匹配时通常是不直接写正则表达式的：因为正则表达式可读性和可维护性比较差。用Python写爬虫这方面可选择的方案非常多了，其中有一个被开发者常用的库pyquery，而Golang也有对应的goquery，可以说goquery是jQuery的Golang版本实现。借用jQueryCSS选择器的语法可以非常方面的实现内容匹配和查找。

### 安装goquery

goquery是第三方库，需要手动安装：

{{< highlight bash >}}
❯ go get github.com/PuerkitoBio/goquery
{{< /highlight >}}

### 创建文档

goquery向外暴露的结构主要是goquery.Document，一般是由2种方法创建的：

{{< highlight go >}}
doc, error := goquery.NewDocumentFromReader(reader io.Reader)
doc, error := goquery.NewDocument(url string)
{{< /highlight >}}

第二种直接传入了url，但是往往我们会对请求做很多定制（如添加头信息、设置Cookie等），所以常用的是第一种方法，我们的代码也要做对应的改动：

{{< highlight go >}}
import (
    "fmt"
    "log"
    "net/http"
    "strconv"
    "time"

    "github.com/PuerkitoBio/goquery"
)

func fetch(url string) *goquery.Document {
    ...
    defer resp.Body.Close()
    doc, err := goquery.NewDocumentFromReader(res.Body)
    if err != nil {
        log.Fatal(err)
    }
    return doc
{{< /highlight >}}

原来是把res.Body转成字符返回，现在直接返回goquery.Document类型的doc了

### CSS选择器

这篇文章不会具体介绍选择器的语法，如果你还不了解可以直接看文末的延伸阅读链接一。

我们先看看豆瓣电影Top250单个条目的部分相关的HTML代码：

{{< highlight html >}}
<ol class="grid_view">
  <li>
    <div class="item">
      <div class="info">
        <div class="hd">
          <a href="https://movie.douban.com/subject/1292052/" class="">
            <span class="title">肖申克的救赎</span>
            <span class="title">&nbsp;/&nbsp;The Shawshank Redemption</span>
            <span class="other">&nbsp;/&nbsp;月黑高飞(港)  /  刺激1995(台)</span>
          </a>
          <span class="playable">[可播放]</span>
        </div>
      </div>
    </div>
  </li>
  ....
</ol>
{{< /highlight >}}

还是原来的需求：获得条目ID和标题。这次需要把parseUrls的逻辑改成使用goquery的版本：

{{< highlight go >}}
func parseUrls(url string, ch chan bool) {
    doc := fetch(url)
    doc.Find("ol.grid_view li").Find(".hd").Each(func(index int, ele *goquery.Selection) {
        movieUrl, _ := ele.Find("a").Attr("href")
        fmt.Println(strings.Split(movieUrl, "/")[4], ele.Find(".title").Eq(0).Text())
    })
    time.Sleep(2 * time.Second)
    ch <- true
}
{{< /highlight >}}

doc.Find的参数就是css选择器，而且Find支持链式调用。这里的意思就是先找雷鸣是"grid_view"的所有ol下的li元素，然后再找li元素里面以hd为名字的元素（看上面的HTML可以知道是div）。Find找到的结果是列表，需要使用Each方法循环获得，可以传递一个包含索引index和子元素ele参数的函数，获得具体内容的逻辑就在这个函数中。

在上面的例子中，类名叫做title的span一共有2个，所以需要取第一个（用Eq(0)），Text方法可以获得元素的内容。而获得条目ID的方法是先拿到条目页面链接（用Attr获得href属性，注意，它返回2个参数，第一个是属性值，第二是是否存在这个属性）。这样就拿到了ID和标题啦，是不是可读性和可维护性高了很多呢？

PS：其实爬虫练习的目的已经达到了，获得更多内容就是多写些逻辑罢了。

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/goquery/doubanCrawler.go)找到。

### 延伸阅读

1. http://www.w3school.com.cn/cssref/css_selectors.asp
2. https://www.itlipeng.cn/2017/04/25/goquery-%E6%96%87%E6%A1%A3/
