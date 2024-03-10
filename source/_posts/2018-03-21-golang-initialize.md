---
title: golang 初始化顺序初探
update: 2018-03-21 10:56:01
tags: 
- golang
categories: 
- coding
---

写了一个简单的程序, 想通过外部环境变量传递给程序的全局变量, 这个全局变量又被另一个全局变量使用, 发现另一个变量无法改变, 然后发现一些有意思的golang程序初始化顺序, 需要稍微注意一下. 

```go
package main

import (
    "fmt"
    "flag"
)

var (
    // 1
    size  = flag.String("size", "3", "size")
    a = "1"
    b = a + "2"
    c = b + *size
)

func init() {
    // 2
    fmt.Printf("%s,  %s, %s, %s \n", a, b, c, *size)
    a = "2"
    b = a + "2"
}

func main() {
    fmt.Printf("%s,  %s, %s, %s \n", a, b, c, *size)
    // 3
    flag.Parse()
    fmt.Printf("%s,  %s, %s, %s \n", a, b, c, *size)
}
```
输出如下:
```shell
/home/shentao/.IdeaIC2016.1/config/plugins/Go/lib/dlv/linux/dlv --listen=localhost:34206 --headless=true exec "/tmp/Build initializetest.go and rungo" -- -size=10
1,  12, 123, 3 
2,  22, 123, 3 
2,  22, 123, 10 
```
注释标记的是实际运行顺序, 先是变量的初始化, 此时size还没传入解析, 所以使用默认值"3"; 其次是调用init函数; 最后是main函数, main函数再进行flag.Parse(), 此时size改变, 但是其他变量不会再变.