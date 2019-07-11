+++
date = "2019-07-11"
title = "用Golang写爬虫(五) - 使用XPath"
slug = "web-crawler-exercise-5"
tags = ["crawler", "Xpath"]
categories = ["crawler"]
+++

在这个系列文章里面已经介绍了BeautifulSoup的替代库soup和Pyquery的替代库goquery，但其实我写Python爬虫最愿意用的页面解析组合是lxml+XPath。为什么呢？先分别说一下lxml和XPath的优势吧

### lxml

lxml是HTML/XML的解析器，它用 C 语言实现的 libxml2 和l ibxslt 的P ython 绑定。除了效率高，还有一个特点是文档容错能力强。

### XPath

XPath全称`XML Path Language`，也就是`XML路径语言`，是一门在XML文档中查找信息的语言，最初是用来搜寻XML文档的，但是它同样适用于HTML文档的搜索。通过编写对应的路径表达式或者使用内置的标准函数，可以方便的直接获取到想要的任何内容，不用像soup和goquery那样要用Find方法链式的找节点再用Text之类的方法或者对应的值（**也就是一句代码就拿到结果了**），这就是它的特点和优势，而lxml正好支持XPath，所以lxml+XPath一直是我写爬虫的首选。

XPath与BeautifulSoup(soup)、Pyquery(goquery)相比，学习曲线要高一些，但是学会它是非常有价值的，你会爱上它。你看我现在，原来用Python写爬虫学会了XPath，现在可以直接找支持XPath的库直接用了。

另外说一点，如果你非常喜欢BeautifulSoup，一定要选择BeautifulSoup+lxml这个组合，因为BeautifulSoup默认的HTML解析器用的是Python标准库中的html.parser，虽然文档容错能力也很强，但是效率会差很多。

我学习XPath是通过w3school，可以从延伸阅读找到链接

### Golang中的Xpath库

用Golang写的Xpath库是很多的，由于我还没有什么实际开发经验，所以能搜到的几个库都试用一下，然后再出结论吧。

首先把豆瓣Top250的部分HTML代码贴出来

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

还是原来的需求：获得条目 ID 和标题

### github.com/lestrrat-go/libxml2

lestrrat-go/libxml2是一个libxml2的Golang绑定库，

首先安装它：

{{< highlight bash >}}
❯ go get github.com/lestrrat-go/libxml2
{{< /highlight >}}

接着改代码

{{< highlight golang >}}
import (
        "log"
        "time"
        "strings"
        "strconv"
        "net/http"

        "github.com/lestrrat-go/libxml2"
        "github.com/lestrrat-go/libxml2/types"
        "github.com/lestrrat-go/libxml2/xpath"
)

func fetch(url string) types.Document {
        log.Println("Fetch Url", url)
        client := &http.Client{}
        req, _ := http.NewRequest("GET", url, nil)
        req.Header.Set("User-Agent", "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)")
        resp, err := client.Do(req)
        if err != nil {
                log.Fatal("Http get err:", err)
        }
        if resp.StatusCode != 200 {
                log.Fatal("Http status code:", resp.StatusCode)
        }
        defer resp.Body.Close()
        doc, err := libxml2.ParseHTMLReader(resp.Body)
        if err != nil {
                log.Fatal(err)
        }
        return doc
}
{{< /highlight >}}

fetch 函数和之前的整体一致，doc 是用` libxml2.ParseHTMLReader(resp.Body)`获得的。parseUrls的改动比较大：

{{< highlight golang >}}
func parseUrls(url string, ch chan bool) {
        doc := fetch(url)
        defer doc.Free()
        nodes := xpath.NodeList(doc.Find(`//ol[@class="grid_view"]/li//div[@class="hd"]`))
        for _, node := range nodes {
                urls, _ := node.Find("./a/@href")
                titles, _ := node.Find(`.//span[@class="title"]/text()`)
                log.Println(strings.Split(urls.NodeList()[0].TextContent(), "/")[4],
                        titles.NodeList()[0].TextContent())
        }
        time.Sleep(2 * time.Second)
        ch <- true
}
{{< /highlight >}}

个人觉得libxml2设计的接口用起来体验不好，每次都要用`NodeList()[index].TextContent`这么麻烦的写法获得匹配值。

另外文档写的非常简陋，看项目源码，还有能用`xpath.NewContext`创建一个上下文，然后用`xpath.String(ctx.Find("/foo/bar"))`的方式获得对应XPath语句的结果，但依然很麻烦！

### github.com/antchfx/htmlquery

htmlquery如其名，是一个对HTML文档做XPath查询的包。它的核心是[antchfx/xpath](https://github.com/antchfx/xpath)，项目更新频繁，文档也比较完整。

首先安装它：

{{< highlight bash >}}
❯ go get github.com/antchfx/htmlquery
{{< /highlight >}}

接着按需求修改：

{{< highlight golang >}}
import (
    "log"
    "time"
    "strings"
    "strconv"
    "net/http"

    "golang.org/x/net/html"
    "github.com/antchfx/htmlquery"
)

func fetch(url string) *html.Node {
    log.Println("Fetch Url", url)
    client := &http.Client{}
    req, _ := http.NewRequest("GET", url, nil)
    req.Header.Set("User-Agent", "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)")
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal("Http get err:", err)
    }
    if resp.StatusCode != 200 {
        log.Fatal("Http status code:", resp.StatusCode)
    }
    defer resp.Body.Close()
    doc, err := htmlquery.Parse(resp.Body)
    if err != nil {
        log.Fatal(err)
    }
    return doc
}
{{< /highlight >}}

fetch函数主要就是修改`htmlquery.Parse(resp.Body)`和函数返回值类型`*html.Node`。再看看`parseUrls`：

{{< highlight golang >}}
func parseUrls(url string, ch chan bool) {
    doc := fetch(url)
    nodes := htmlquery.Find(doc, `//ol[@class="grid_view"]/li//div[@class="hd"]`)
    for _, node := range nodes {
        url := htmlquery.FindOne(node, "./a/@href")
        title := htmlquery.FindOne(node, `.//span[@class="title"]/text()`)
        log.Println(strings.Split(htmlquery.InnerText(url), "/")[4],
            htmlquery.InnerText(title))
    }
    time.Sleep(2 * time.Second)
    ch <- true
}
{{< /highlight >}}

`antchfx/htmlquery`的体验比`lestrrat-go/libxml2`要好，Find是选符合的节点列表，FindOne是找符合的第一个节点。

### 后记

通过这个爬虫算是基本了解了这2个库，还有个`gopkg.in/xmlpath.v2`我没有写，主要是它很久不更新了(最近一次是2015年更新的)。

随便说一下gopkg.in，gopkg是一种包管理方式，其实是用约定好的方式「代理」Github上对应项目的对应分支的包。具体的请看延伸阅读链接2。

xmlpath.v2这个包就是Github上的[go-xmlpath/xmlpath](https://github.com/go-xmlpath/xmlpath), 分支是v2。

这些库里面我推荐使用`antchfx/htmlquery`，接口更好用一些。性能、功能这方面还有更多经验，如果之后发现其他问题我再写文章吧！

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/xpath)找到。

### 延伸阅读

1. http://www.w3school.com.cn/xpath/xpath_intro.asp
2. http://labix.org/gopkg.in
