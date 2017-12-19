# for 和 select 

## for...range

从 goroutine 中读取数据，一个常见的模式是用 for 循环不断从 channel 中读数据，直到 channel 关闭才推出循环。
在前一节生产者和消费者模型中，消费者的逻辑就是如此，对应的代码当时是这样写的：

```
for {
    data, ok := <- ch
    if !ok {
        fmt.Printf("consumer %d: detect channel close\n", c.consumerID)
        return
    }

    fmt.Printf("consumer %d got: %s\n", c.consumerID, data)
    time.Sleep(time.Microsecond * 10)
}
```

因为这种用法非常普遍，所以 go 语言提供了快捷的方法：for...range。
如果熟悉 go 的语法，会知道 for...range 一般用来遍历 slice 或者 map，但是它也可以用来读取 channel 中的内容。
当 range 后面跟着的是可读 channel 时，go 语言会每次从 channel 中读取一个数据;
如果 channel 中没有数据可读则阻塞在这里，直到能读到数据；如果 channel 关闭，则退出循环逻辑。

所以上面的代码可以用 for...range 修改成：

```
for data := range ch {
    fmt.Printf("consumer %d got: %s\n", c.consumerID, data)
    time.Sleep(time.Microsecond * 10)
}
fmt.Printf("consumer %d: detect channel close\n", c.consumerID)
```

是不是精简了很多！

## select

另外一个常见的需求是：从多个 channel 中读取数据，只要任意一个 channel 有数据就执行对应的逻辑。
我们不能一次去循环这些 channel，因为第一个执行的逻辑只有运行完成才会继续运行后面的逻辑，而不是
预期的从多个 channel 中选择。

对于这种情况，go 提供了 select...case 语句，select...case 和 switch...case 结构类似，都是从多个分支中
选择一个执行。但是区别在于，select...case 从多个 channel 操作中选择一个可执行的，switch...case 是从多个
语句中选择一个值为真或者值匹配的。

select...case 的结构大致是这样的：

```
select {
    case <- ch1:
        // do something if read from ch1
    case x := <-ch2:
        // do something if read from ch2, and assign value to x
    case ch3 <- y:
        // do something if can write to ch3
    default:
        // do something if none of the above happens
}
```

select 可以跟多个 case 语句，以及可选的 default 语句。每个 case 后面是一个通信操作（从 channel 中读数据，或者往 channel 中写数据），
下面跟着一个代码块。从 channel 中读数据可以丢弃读到的值（第一种情况），或者把读到的值赋给某个变量（第二种情况）。

如果没有 default 语句，select 会阻塞，直到某个通信操作可以执行，go 执行它的通信操作，以及下面跟着的代码逻辑块，其他的 select 语句不会执行。
如果有 default 语句，运行到这里时，select 如果发现有 case 语句可以执行，则执行相关逻辑；如果没有可以执行的 case，也不会阻塞，而是直接运行 default 
下面跟着的代码块。

如果有多个 case 可以执行，go 会**随机选择一个**，我们不应该对它的顺序有什么期望。

比如要执行一个很耗时的任务，我们希望打印出进展，那么可以使用 select 语句：


```
package main

import (
    "fmt"
    "time"
)

func worker(done chan<-struct{}){
    time.Sleep(5 * time.Second)

    done <- struct{}{}
}

func main(){
    done := make(chan struct{})
    tick := time.NewTicker(1 * time.Second)

    go worker(done)

    for {
        select {
        case <- tick.C:
            fmt.Printf(".")
        case <- done:
            fmt.Printf("\nwork done.\n")
            return 
        }
    }
}
```

上面的代码中，我们运行一个 woker，用 time.Sleep 来模拟它运行需要的时间，在后台以 goroutine 方式运行它，当运行完成后会往 `done` channel 中发送一个数据。
同时，我们创建了一个 `time.Ticker` 对象，它是一个可读 channel，会每秒钟发送一个数据。

我们在 main 函数中使用 `select` 从 `tick.C` 和 `done` 中选择一个读取数据，在 worker 执行完成之前，一定是 `tick.C` 中能读到数据，终端会打印一个点 `.`
表示程序还在运行；当 worker 执行完成时，`done` 中能够读到数据，我们就打印程序完成的消息，然后退出。

执行上面的代码，可以下面的结果：

```
$ go run main.go
.....
work done.
```
