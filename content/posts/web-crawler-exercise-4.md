+++
date = "2019-07-10"
title = "用Golang写爬虫(四) - 使用soup"
slug = "web-crawler-exercise-4"
tags = ["crawler"]
categories = ["crawler"]
+++

Python爬虫工程师有个常用的提取数据的库BeautifulSoup，而在Golang语言也有一个对应的库soup，由于我比较喜欢Python写爬虫所以自然而然的就想到了soup，这篇文章就是就来体验一下它。

### 安装soup

soup是第三方库，需要手动安装：

{{< highlight bash >}}
❯ go get github.com/anaskhan96/soup
{{< /highlight >}}

### 使用soup

就如之前的练习，我们是要定义头信息的，但是soup这个库只开放了Get方法接收url参数。不过其他soup也是可以定义Header和Cookie的，可以改成这样：

{{< highlight go >}}
import (
    "fmt"
    "log"
    "strconv"
    "strings"
    "time"

    "github.com/anaskhan96/soup"
)

func fetch(url string) soup.Root {
    fmt.Println("Fetch Url", url)
    soup.Headers = map[string]string{
        "User-Agent": "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)",
    }

    source, err := soup.Get(url)
    if err != nil {
        log.Fatal(err)
    }
    doc := soup.HTMLParse(source)
    return doc
}
{{< /highlight >}}

这次不再用内置的`net/http`这个包了。soup支持直接设置Headers（以及Cookies）的值，也可以实现自定义头信息和Cookie，然后就可以`soup.Get(url)`了，然后用`soup.HTMLParse`就可以获得文档对象了。

然后再看看豆瓣电影Top250单个条目的部分相关的HTML代码：

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

还是原来的需求：获得条目ID和标题。这次需要把parseUrls的逻辑改成使用soup的版本：

{{< highlight go >}}
func parseUrls(url string, ch chan bool) {
	doc := fetch(url)
	for _, root := range doc.Find("ol", "class", "grid_view").FindAll("div", "class", "hd") {
		movieUrl, _ := root.Find("a").Attrs()["href"]
		title := root.Find("span", "class", "title").Text()
		fmt.Println(strings.Split(movieUrl, "/")[4], title)
	}
	time.Sleep(2 * time.Second)
	ch <- true
}
{{< /highlight >}}

可以感受到和goquery都用了Find这个方法名字，但是参数形式不一样，需要传递三个：「标签名」、「类型」、「具体值」。如果有多个可以使用FindAll(Find是找第一个)。如果想要找属性的值需要用Attrs方法，从map里面获得。

获得文本还是用Text方法。另外它内有goquery那样的Each方法，需要手动写一个`for range`格式的循环。

### 后记

通过这个爬虫算是基本了解了这个库，我觉得总体上soup是足够用的

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/soup/doubanCrawler.go)找到。
