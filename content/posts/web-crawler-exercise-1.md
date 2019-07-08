---
title: "用Golang写爬虫(一)"
date: 2019-07-03T21:18:56+08:00
categories: crawler
---

{{< highlight golang >}}
func main() {
	start := time.Now()
	for i := 0; i < 10; i++ {
		parseUrls("https://movie.douban.com/top250?start=" + strconv.Itoa(25 * i))
	}
	elapsed := time.Since(start)
	fmt.Printf("Took %s", elapsed)
}
{{< /highlight >}}
