# 你的第一个 go 并发程序

很多介绍 go 语言的文章和书籍都会提到它的并发模型，夸赞用 go 语言编写并发应用是多简单。
并发编程已经成为 go 语言最有标志性的特性，也是很多程序员和技术公司选择它的主要原因。

go 语言提出了 goroutine 的概念，作为程序运行和调度的基本单位。可以把 goroutine 类比于操作系统的
进程和线程，只不过非常轻量（或许你曾看到过蝇量级这个说法），所以可以创建大量的 goroutine，并且它们的调度
也更高效。

创建一个 goroutine 在 go 中非常简单，之前在正常的函数前面加上 `go` 关键词，那么这个函数就会作为一个 goroutine
在后台运行。来看一个并发版本的 hello world 程序：


```go
package main

import (
    "fmt"
	"time"
)

func worker(name string) {
    for i:=0; i<10; i++ {
        fmt.Println(name)
		time.Sleep(10 * time.Millisecond)
    }
}

func main() {
    go worker("hello")
    go worker("world")
}
```

我们编写了一个 `worker` 函数，简单地把某个字符创打印 10 遍。然后在 main 函数中启动两个 goroutine，分别打印
 `hello` 和 `world` 字符串。运行程序我们期望在终端交替看到两个单词的出现，直到打印完毕。可以看到，和普通的程序相比，
 我们不需要依赖额外的库，只是添加了两个 `go` 关键字就把顺序执行的程序变成了并发执行，这也许是很多人说 go 并发编程简单的原因吧。
 
如果上面的程序保存为 `main.go`，执行 `go run main.go` 来运行，你会发现终端上什么都没有打印。这是为什么？不是说很简单吗，为什么上来就遇到问题？

虽然 go 语言提供了语言级别特性让并发编程变得简单，但是想要编写正确高效的并发代码并不是件容易的事情，我们还是
要了解 goroutine 的各种特性，并认真设计代码逻辑。对于这里遇到的第一个问题，答案在于 main 函数本身也是一个 goroutine，
创建完两个 goroutine 之后，main 函数就运行结束并退出，而不会等待它创建的 goroutine 运行完成。因此 **两个 woker goroutine 还没有运行，main 函数就退出了，**所以我们自然看不到输出。

知道了问题，解决的思路就是让 main 函数等待两个 goroutine 运行结束再退出。提到等待，最容易的想法是在程序中 sleep 一段时间，所以把上面的代码修改一下：

```go
package main

import (
    "fmt"
    "time"
)

func worker(name string) {
    for i:=0; i<10; i++ {
        fmt.Println(name)
        time.Sleep(time.Millisecond * 10)
    }
}

func main() {
    go worker("hello")
    go worker("world")

    time.Sleep(time.Second * 2)
}
```

再次执行，你会在终端看到期望的输出结果：hello 和 world 交替出现，而且运行多次会发现输出的顺序也不是不断变化的。

虽然上面的程序得到的期望结果，但还是有一个致命的问题：sleep 的时间应该设置为多少？对于这个简单的程序两秒钟就足够了，但是对于复杂的程序
要评估一个合理的运行时间非常困难。如果设置的时间太短，会导致上面 goroutine 没有运行完成的问题；如果设置的时间太长，在实际中又会导致时间浪费。

理想情况下，我们希望 goroutine 一旦运行完，main 函数得到通知，并立即退出。go 当然提供了对应的解决方案，这正是我们下一节要讲的内容。
