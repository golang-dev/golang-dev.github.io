+++
date = "2019-07-04"
title = "用Golang写爬虫(一)"
slug = "web-crawler-exercise-1"
tags = ["crawler"]
categories = ["crawler"]
+++

之前一直都是再用Python写爬虫，最近想体验下Golang写爬虫的感觉，所以就有了这个系列。我想要抓取的页面是[豆瓣Top250页面](https://movie.douban.com/top250)，选择它的理由有3个:

1. 豆瓣页面代码相对规范
2. 豆瓣对爬虫爱好者相对更宽容
3. Top250页面简洁，很适合拿来练手

我们先看第一版的代码。

按逻辑我把抓取代码分成2个部分：

1. HTTP请求
2. 解析页面中的内容

我们先看HTTP请求，Golang语言的HTTP请求库不需要使用第三方的库，标准库就内置了足够好的支持：

{{< highlight go >}}
import (
	"fmt"
	"net/http"
	"io/ioutil"
)

func fetch (url string) string {
	fmt.Println("Fetch Url", url)
	client := &http.Client{}
	req, _ := http.NewRequest("GET", url, nil)
	req.Header.Set("User-Agent", "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)")
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println("Http get err:", err)
        return ""
	}
	if resp.StatusCode != 200 {
		fmt.Println("Http status code:", resp.StatusCode)
		return ""
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println("Read error", err)
		return ""
	}
	return string(body)
}
{{< /highlight >}}

我把URL请求的逻辑都放在了fetch函数中，里面做了一些异常处理。值得说的有2点：

1. 在Header中设置了User-Agent，让访问看起来更像搜索引擎Bot。如果一个网站希望自己的内容被Google收录那么他就不会拒绝这样的UA的访问。
2. 需要通过ioutil.ReadAll 读取resp的body内容，最后用string(body)把它转化成字符串

接着就是解析页面的部分：

{{< highlight go >}}
import (
    "regexp"
	"strings"
)

func parseUrls(url string) {
	body := fetch(url)
	body = strings.Replace(body, "\n", "", -1)
	rp := regexp.MustCompile(`<div class="hd">(.*?)</div>`)
	titleRe := regexp.MustCompile(`<span class="title">(.*?)</span>`)
	idRe := regexp.MustCompile(`<a href="https://movie.douban.com/subject/(\d+)/"`)
	items := rp.FindAllStringSubmatch(body, -1)
	for _, item := range items {
		fmt.Println(idRe.FindStringSubmatch(item[1])[1],
			titleRe.FindStringSubmatch(item[1])[1])
	}
}
{{< /highlight >}}

这篇文章我们主要体验用标准库完成页面的解析，也就是用正则表达式包regexp来完成。不过要注意需要用`strings.Replace(body, "\n", "", -1)`这步把body内容中的回车符去掉，要不然下面的正则表达式`.*`就不符合了。`FindAllStringSubmatch`方法会把符合正则表达式的结果都解析出来（一个列表），而`FindStringSubmatch`是找第一个符合的结果。

Top250页面是要翻页的，最后在main函数里面实现抓取全部Top250页面。另外为了和之后的改进做对比，我们加上代码运行耗时的逻辑：

{{< highlight go >}}
import (
       "time"
       "strconv"
)
func main() {
        start := time.Now()
        for i := 0; i < 10; i++ {
                parseUrls("https://movie.douban.com/top250?start=" + strconv.Itoa(25 * i))
        }
        elapsed := time.Since(start)
        fmt.Printf("Took %s", elapsed)
}
{{< /highlight >}}

在Golang中把数字转成字符串需要使用`strconv.Itoa`（嘿嘿，本博客域名就是这个模块），这样就可以根据start的参数的不通拼出正确的页面路径。用一个for循环完成翻页。

运行起来非常快：

{{< highlight bash >}}
❯ go run crawler/doubanCrawler1.go
... # 省略输出
Took 1.454627547s
{{< /highlight >}}

通过终端输出可以看到我们拿到了对应电影条目的ID和电影标题！

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/crawler/doubanCrawler1.go)找到。
