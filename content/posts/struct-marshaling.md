+++
date = "2020-07-05"
title = "Golang中的结构体编码"
slug = "struct-marshaling"
tags = ["struct"]
categories = ["stdlib", "struct"]
+++

Go不支持类，而是提供了结构体支持面向对象的编程风格。可以给结构体中添加方法，将数据和操作数据的方法绑定在一起，实现与类相似的效果。

结构体是由一系列具有相同类型或不同类型的数据构成的数据集合:

```go
type Person struct {
    FirstName string
    LastName  string
}
```

在这个结构体中有2个字段`FirstName`和`LastName`。Go的标准库默认就可以对结构体进行JSON编码:

```go
func main() {
    bytes, _ := json.Marshal(&Person{"Dany", "Boon"})
    fmt.Printf("%s", bytes)
}

// 输出: {"FirstName":"Dany","LastName":"Boon"}
```

这是所见即所得的编码。当然我们可以通过结构体标签(Struct Tags)对字段进行自定义处理:

```go
type Person struct {
  FirstName string `json:"first_name"`
  LastName  string `json:"last_name"`
}
```

Struct Tag是声明的每个类型后面的注释，这样在编码时就改变了键的值，现在的输出是:

```bash
{"first_name":"Dany","last_name":"Boon"}
```

支持结构体标签的标准库列表可以看延伸阅读链接1

当然我这篇文章主要时间记录在实际使用JSON编码是遇到的2个问题

### 自定义JSON编码

前面的例子都是按字段值输出，但有时候编码需要改变这个原始值，最常见的就是Time类型，比如输出CreatedAt的值，默认是这样的:

```go
type Subject struct {
    ID        int `json:id`
    CreatedAt time.Time
}

func main() {
    bytes, _ := json.Marshal(&Subject{111, time.Now()})
    fmt.Printf("%s", bytes)
}
// 输出: {"ID":111,"CreatedAt":"2020-07-05T18:03:19.753651+08:00"}
```

Go语言时间类型默认输出的是RFC 3339格式的，但实际上我们大部分API仅需要时间部分。这时我早期的写法是不用`time.Time`，自定义一个结构体，并定义它的MarshalJSON方法:

```go
type JSONTime struct {
  time.Time
}

func (t *JSONTime) MarshalJSON() ([]byte, error) {
  return []byte(fmt.Sprintf(`"%s"`, t.Format("2006-01-02 15:04:05"))), nil
}

type Subject struct {
  ID        int `json:id`
  CreatedAt JSONTime
}

func main() {
  bytes, _ := json.Marshal(&Subject{111, JSONTime{time.Now()}})
  fmt.Printf("%s", bytes)
}
// 输出: {"ID":111,"CreatedAt":"2020-07-05 18:10:09"}
```

这样做的好处是通用, 不需要在用到的类型中反复实现。如果只在Subject中实现也可以:

```go
func (s *Subject) MarshalJSON() ([]byte, error) {

    return json.Marshal(struct {
        ID        int `json:id`
        CreatedAt string
    }{
        ID:        s.ID,
        CreatedAt: s.CreatedAt.Format("2006-01-02 15:04:05"),
    })
}
// 输出: {"ID":111,"CreatedAt":"2020-07-05 18:13:56"}
```

在MarshalJSON方法中，引入一个附加的、匿名的结构体，重新声明了2个字段和类型(这次CreatedAt不再是Time而是string了)。这样就实现了结构体某个(些)字段输出值的自定义

### 扩展结构体字段

想了很久没找到更好的子标题。先说一个Python例子吧:

```python
class Person:
    def __init__(self, first_name, last_name):
        self.first_name = first_name
        self.last_name = last_name

    @property
    def full_name(self):
        return self.first_name + self.last_name
```

可以很容易的在里面扩展出其他的属性，例如上面的实例会有一个叫做`full_name`的属性。这件事在Go里面就很麻烦，因为JSON编码的那些内容可能并不是摆在那里设置好的，而是需要计算得出的。还是同样的例子，怎么返回`first_name`，`last_name`和`full_name`呢？

可以这样写MarshalJSON方法:

```go
func (p *Person) MarshalJSON() ([]byte, error) {
    type Alias Person
    return json.Marshal(&struct {
        *Alias
        FullName string `json:"full_name"`
    }{
        Alias:    (*Alias)(p),
        FullName: fmt.Sprintf("%s %s", p.FirstName, p.LastName),
    })
}
// 输出: {"first_name":"Dany","last_name":"Boon","full_name":"Dany Boon"}
```

首先里面`type Alias Person`这句为原始类型Person起一个别名Alias，Alias会有原始struct所有的字段，但是不会继承它的方法，如果不这么用就会在编码时循环调用Person的MarshalJSON方法而造成堆栈溢出。

其次是`json.Marshal`里面用的那个匿名的结构体除了组合了Alias，还扩展除了新的、我们需要的字段FullName。这样写是不是很优美也很灵活呢？

### 后记

这里只讨论了编码方法，对应的解码可以通过自定义UnmarshalJSON来实现。

### 延伸阅读

1. https://github.com/golang/go/wiki/Well-known-struct-tags#list-of-well-known-struct-tags
2. http://choly.ca/post/go-json-marshalling
3. https://go-academy.gitbook.io/go-academy/idioms/custom-json-marshaling
