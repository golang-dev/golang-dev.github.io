+++
date = "2019-07-13T12:10:10"
title = "用Golang写爬虫(八) - 使用GORM存入MySQL数据库"
slug = "use-gorm"
tags = ["crawler", "gorm"]
categories = ["crawler"]
+++

上篇文章把数据存进了文件，这篇文章将把数据存入MySQL数据库。数据库的使用是每个开发者必备的技能，所以本篇文章我们使用Golang操作数据库。Golang没有内置的驱动支持任何的数据库，但是Go定义了database/sql接口，开发者可以基于驱动接口开发相应数据库的驱动。当然现在各种数据库驱动生态已经很稳定了，可以直接使用。

我在实际开发工作中一般不直接用数据库驱动（如github.com/go-sql-driver/mysql），而是用ORM。ORM有三大优点：

1. 安全性。使用底层的数据库驱动是在「拼」SQL，这样很容易出现SQL注入等风险。
2. 易于理解。拼 SQL 这种形式放在代码中是很乱的，只有开发者在当时编写时可以理解，但是后来的维护者就不那么好理解了。
3. 易于维护。如果数据库操作代码写多了往往会有很多重复、可重用的SQL其实是可以避免的，如果用ORM，他把CURD这些操作都费装起来，这种代码看起来会非常直观，很容易找到一些错误的，不好的，重复的用法。
4. 可移植。ORM往往支持多种数据库，如MySQL、PostgreSQL、SQLite等，这样在本地开发你可选择SQLite，而在生产环境可以切换MySQL，完全不需要改动代码逻辑，只是改一下配置。

所以使用ORM能提高开发效率，降低开发成本，但是缺点是ORM的封装是有成本的，会稍微影响到性能，不过我认为这点性能损耗微乎其微。

目前Golang世界里面最受欢迎的ORM叫做[gorm](https://github.com/jinzhu/gorm)，我们就使用它，先安装它：

{{< highlight bash >}}
❯ go get -u github.com/jinzhu/gorm
❯ go get -u github.com/go-sql-driver/mysql  # 也要安装对应数据库驱动
{{< /highlight >}}

gorm用的MySQL驱动其实就是`go-sql-driver/mysql`，但通常这么import：

{{< highlight go >}}
import (
  "github.com/jinzhu/gorm"
  _ "github.com/jinzhu/gorm/dialects/mysql"
)
{{< /highlight >}}

有时候对于某个包，不需要把整个包都导入进来，只是执行他的init函数。可以使用`import _`这样的写法，不过要注意，这种方式无法通过包名来调用包中的其他函数。

### 定义模型

接着我们声明Model:

{{< highlight go >}}
type Movie struct {
    ID int
    Title string `gorm:"type:varchar(100);unique_index"`
}
{{< /highlight >}}

这个结构体叫做Movie，模型有2个字段ID和Title。Title后面有个反引号，这个部分叫做成员变量标签(Tag)，当需要用到Tag中的内容时可以使用反射包（reflect）中的方法来获取，这是gorm支持的用法。它表示定义title资格字段的长度是100，并且有唯一索引。其他支持的标签可以看官方文档。

如果看文档或者很多文章，经常能看到继承了` gorm.Model`的模型，如下面这个：

{{< highlight go >}}
type Movie struct {
    gorm.Model
    Title string `gorm:"type:varchar(100);unique_index"`
}
{{< /highlight >}}

gorm.Model 自带了ID, CreatedAt, UpdatedAt, DeletedAt这么4个字段，在我们这里完全必要。

### 初始化

在main函数里面连接数据库，并创建表：

{{< highlight go >}}
func main() {
    start := time.Now()
    ch := make(chan bool)
    db, err := gorm.Open("mysql", "root:@/test?charset=utf8")
    defer db.Close()
    checkError(err)

    db.DropTableIfExists(&Movie{})
    db.Set("gorm:table_options", "ENGINE=InnoDB").AutoMigrate(&Movie{})

    for i := 0; i < 11; i++ {
        go parseUrls("https://movie.douban.com/top250?start="+strconv.Itoa(25*i), ch, db)
    }

    for i := 0; i < 11; i++ {
        <-ch
    }

    elapsed := time.Since(start)
    log.Printf("Took %s", elapsed)
}
{{< /highlight >}}

gorm.Open的第一个参数是数据库驱动名字，第二个就是具体的db uri，我这里用了默认的test库，用户名root，密码为空，字符集为utf-8。

为了每次都初始化表，所以先用`DropTableIfExists`删除表，再用`AutoMigrate`创建表。Migrate是迁移的意思，自动迁移仅仅会创建表，缺少列和索引，并且不会改变现有列的类型或删除未使用的列以保护数据。这样可以更新表结构，一般不会直接用CreateTable创建表了。db.Set是为了创建表时添加表后缀。

parseUrls的最后一个参数db是*gorm.DB类型的。

好了，运行一下：

{{< highlight bash >}}
❯ go run doubanCrawler.go
{{< /highlight >}}

执行结束可以用MySQL CLI看到数据了：

{{< highlight sql >}}
mysql> select * from movies order by id desc limit 3;
+----------|-----------------------------+
| id       | title                       |
+----------|-----------------------------+
| 30170448 | 何以为家                    |
| 27191492 | 四个春天                    |
| 26799731 | 请以你的名字呼唤我          |
+----------|-----------------------------+
3 rows in set (0.00 sec)
{{< /highlight >}}

### 用gorm查询

上面用到的只是db.Create创建记录，接着看看怎么查询(按行注释)：

{{< highlight bash >}}
func main() {
    db, err := gorm.Open("mysql", "root:@/test?charset=utf8")
    defer db.Close()
    checkError(err)

    var movie Movie
    var movies []Movie
    db.First(&movie, 30170448)  // 查询ID为30170448的记录，只支持主键
    log.Println(movie)
    log.Println(movie.ID, movie.Title) // 可以获得对应属性

    db.Order("id").Limit(3).Find(&movies) // 按ID升序排，取前三个记录赋值给movies
    log.Println(movies)
    log.Println(movies[0].ID)

    db.Order("id desc").Limit(3).Offset(1).Find(&movies)  // Order也支持desc选择降序, offset表示对结果集从第2个记录开始
    log.Println(movies)
    db.Select("title").First(&movies, "title = ?", "四个春天") // 用Select可以限定只返回那些字段，First也支持条件
    log.Println(movie)

    var count int64
    db.Where("id = ?", 30170448).Or("title = ?", "四个春天").Find(&movies).Count(&count)  // Where后加条件，在这里是id为30170448或者title为"四个春天"2个条件符合之一即可，最后用Count算一下符合的记录数
    log.Println(count)
}
{{< /highlight >}}

执行一下：

{{< highlight bash >}}
❯ go run doubanCrawler2.go
2019/07/13 21:16:24 {30170448 何以为家}
2019/07/13 21:16:24 30170448 何以为家
2019/07/13 21:16:24 [{1291543 功夫} {1291545 大鱼} {1291546 霸王别姬}]
2019/07/13 21:16:24 1291543
2019/07/13 21:16:24 [{27191492 四个春天} {26799731 请以你的名字呼唤我} {26787574 奇迹男孩}]
2019/07/13 21:16:24 [{0 何以为家}]
2019/07/13 21:16:24 {30170448 何以为家}
2019/07/13 21:16:24 2
{{< /highlight >}}
更新和删除用法就不展示了，看文档即可。

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/gorm)找到。

### 延伸阅读

1. https://gorm.io/docs/
