# goroutine 通过 channel 通信

通过前面几节的学习，我们已经知道如何在 go 语言中启动 goroutine 来实现并发地运行程序逻辑。
但是这些并发的函数都是相互独立的，不需要互相之间有任何的交互，但现实生活中以及实际项目中，并发的 goroutine 之间
往往需要知道对方的存在，同步事情的进展，发送数据给对方。

go 提供了另外一个概念: channel。channel 一般翻译成管道，或者通道，能够比较形象地阐释数据的发送和接收。
对于程序员开发，也可以把 channel 简单类比成队列。和队列一样，goroutine 可以往 channel 中发送数据和读取数据，
而且数据能保证先进先出的顺序，但是和队列不同的是，这个队列是 go 运行时维护的，能够保证并发运行的正确性，还提供了
一些更复杂的功能。

## channel 基本知识

要定义一个 channel 变量，需要使用 `make` 关键词:

```
ch := make(chan int)
```

`chan` 关键词说明我们是在创建一个 channel，而后面的 `int` 代表 channel 中可以传输的数据类型。这个类型可以是 go 语言自带的类型、命名类型、struct 类型等，
也可以是这些类型的指针类型。


既然类似于队列，那么就一定会支持两种基本操作：从 channel 中读取数据和往 channel 中写入数据。go 语言定义了个特殊的符号: `<-`，而没有提供内嵌的函数。

往 channel 中写入数据是这样的：

```
ch <- 42
```

`<-` 右边是要写入的数据，左边是写到的 channel 变量。

而从 channel 中读取数据变量符号两边的内容正好相反，左右是存放数据的变量，右边是 channel，因为是赋值操作，中间还需要 `=` 或者 `:=` 赋值符号：

```
answer := <- ch
```

## 使用 channel 模拟乒乓游戏

为了说明 channel 的使用，我们写一个模拟打乒乓球的 go 程序。和之前打印字符串的程序不同，打乒乓球需要两个选手参与（goroutine），
而且他们要等待同一个乒乓球（同步数据），只有乒乓球被打到自己这边的时候才能接球，否则就要一直处于准备状态（等待）。


```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// palyer 函数模拟每个乒乓球队员的行为
// 接球 -> 把球打过去 -> 等待球回来
func player(name string, table chan int){
    for {
        ball := <- table
        fmt.Printf("%d %s\n", ball, name)
        time.Sleep(time.Second)
        ball++
        table <- ball
    }
}

func main() {
    var wg sync.WaitGroup

	// 定义 table 模拟球台，传递乒乓球
    table := make(chan int)
    ball := 1

    wg.Add(2)

	// 第一个乒乓球员上线
    go func(){
        player("ping", table)
        wg.Done()
    }()

	// 第二个乒乓球员上线
	// 使用 Sleep 是为了让第一个球员先接球
    go func(){
        time.Sleep(time.Millisecond)
        player("\tpong", table)
        wg.Done()
    }()

	// 发球
    table <- ball

    wg.Wait()
	
	// 实际上，这句话并不会执行到
    fmt.Println("Game Over")
}
```


这个简单的程序继续使用 `WaitGroup` 来等待 goroutine 执行完成（但是实际上两个 goroutine 不会自动结束），每个队员在后台作为 goroutine 运行。
player 函数模拟每个乒乓球队员的行为，不断循环以下逻辑：

- 等待球发到自己这里
- 在终端打印自己收到球的信息，并修改击球的次数
- 把球发出去

乒乓球在这里定义为一个整数，它只是记录了游戏中球被击中了几次。这里需要重点说明的是，我们定义了 `table` 各个变量来模拟球台。
因为每次击球都是通过球台把球发送给对方的，因此球台定义为 channel 类型，其中传递的球是 int 类型。

运行程序，可以看到终端依次打印 `ping` 和 `pong`，以及每次击球的次数，输入 `Ctrl + c` 来结束程序的运行： 

```
➜  ping-pong go run main.go
1 ping
2     pong
3 ping
4     pong
5 ping
6     pong
^Csignal: interrupt
```

我们已经让两个 goroutine 进行同步，等待对方完成一个事件后自己才继续执行，这个逻辑非常简单，但是很使用，在实际的代码中会经常遇到。

虽然我们可以通过在顺序执行的程序里依次调用两个队员来实现相同的逻辑，
但是我们的程序明显更符合现实世界的情况：每个队员都是独立地完成自己的动作，并没有统一的控制中心。

这个游戏还有一个明显的缺陷：我们假设两个乒乓球选手永远都不会失误，每次都能准确地把球击中回去。这在实际生活中显然是不可能的，在下一节，我们继续
改进程序，让乒乓游戏能够正常运行结束。
