# 关闭 channel

在上一节，我们介绍了 channel 的基本概念，并使用 channel 模拟了打乒乓球的过程，最后留下了一个问题：怎么结束乒乓球游戏？
这一节，我们将解答这个问题。

不妨我们先思考一下，现实中一场乒乓球是怎么结束的？无非是两种情况：一个是没接住对方的球；另外是接到了球，但是没有打到对方桌面上。
我们可以把这两种情况简化为一种情况：某个选手接到球之后，直接失败了。因为不同选手的能力不同，所以失败的几率也不同，
根据选手的接球成功率，我们可以用简单的算法判断它每次接球是成功还是失败。
一方选手接球失败，另外一方就能直接看到，接收到这个消息，准备下一个回合的比赛，而不是像之前那样傻等着对方继续发球过来。

回到我们 goroutine 的代码中，我们可以让某个选手接球失败之后就直接退出 goroutine，但是另外一个 goroutine 并不会自动
知道这个 goroutine 已经退出，还是会执行 `ball := <- table` 的指令，傻傻地等待对方发球过来。

当然，我们可以创建另外一个 channel 单独传递某一方失败的消息，另外一方定时去查看这个 channel 来判断是否比赛已经结束。
但这无疑让整个问题变得很复杂，其实我们可以用 channel 自带的另外一个操作来完成相同的功能，那就是接下来要讲解的关闭
channel。

## channel close 操作

和队列不同的是，channel 还可以关闭，这个有点像 socket，双方通信完成之后，需要关闭连接。
关闭 channel 只需要调用 `close` 函数：

```
ch := make(chan int)
close(ch)
```

那关闭的 channel 会有哪些行为呢？首先，channel 只能关闭一次，如果尝试关闭一个已经关闭的 channel 会报错：

```
panic: close of closed channel
```

尝试往一个已经关闭的 channel 中写入数据也会报错：


```
panic: send on closed channel
```

但是从已经关闭的 channel 中是可以读取数据的，它会返回 channel 中传输数据类型的默认值，比如如果 channel 中传输的数据类型为 int，
那么则会一直返回 0。

**NOTE:** go 语言中各种类型的默认值请参考相关数据或者文档。

```
package main

import (
    "fmt"
)

func main() {
    ch := make(chan int)
    close(ch)

    for i := 0; i < 10; i++ {
        data := <- ch
        fmt.Printf("%d ", data)
    }
}
```

上面的程序会打印出：`0 0 0 0 0 0 0 0 0 0`，并不会报任何错误。

那么，这就有一个问题：接收方（消费者）怎么知道 channel 已经关闭了呢？还是说 channel 就是在一直发送 0 过来呢？
答案是：可以在从 channel 中读取数据的时候提供第二个接收变量，它是一个布尔值。当为真时，表明 channel 是打开状态，
当为假时，表明 channel 已经关闭。

```
data, ok := <- ch
if !ok {
    fmt.Printf("channel closed. exit...")
    return
}
fmt.Printf("%d ", data)
```

因为 channel 的这种特性，一般发送方（生产者）在没有数据发送后关闭 channel，并且保证不会再往 channel 中写数据；
接收方（消费者）判断 channel 是否关闭，根据请求执行不同的逻辑。

在使用完 channel 之后，不必一定要把它关闭，go 语言会保证未关闭的 channel 不会造成资源泄露。
只有当需要通知接收方数据已经发送完毕时，才需要关闭 channel。

## 完整的乒乓球模拟代码

根据 channel 上面的特性，我们来继续改进乒乓球游戏。

首先，定义一个每次接球的成功率，取值是 0-100 之间。
每次接到球，生成一个随机数，结合成功率来决定这次能否接球成功。

如果接球失败，则关闭 channel（通知对方），然后退出函数。每次接球的时候判断 channel 是否已经关闭（对方失败），
如果已经关闭，则退出函数；否则继续游戏。

完整的代码如下：

```go
package main

import (
    "fmt"
    "sync"
    "time"
    "math/rand"
)

func init(){
	// 每次运行用当前时间重置随机数生成器，增加随机性
    rand.Seed(time.Now().UnixNano())
}

type player struct {
    name string
    successRatio int // a number between [0, 100]
}

func play(p *player, table chan int){
    for {
        ball, ok := <- table

        // channel 已经关闭了，只有对方失败了才会关闭 channel，
		// 也就是说，当前队员赢得了游戏。
        if !ok {
            fmt.Printf("%s win!!!\n", p.name)
            return
        }
        
        // 生成一个 100 以内的随机数
		// 如果这个值大于成功率，则判定接球失败
        r := rand.Intn(100)
        if r > p.successRatio {
			// 失败之后，关闭 channel，然后退出函数
            fmt.Printf("%s lose.\n", p.name)
            close(table)
            return
        }    

        fmt.Printf("%d %s\n", ball, p.name)
        time.Sleep(time.Millisecond * 200)
        ball++
        table <- ball
    }
}

func main() {
    var wg sync.WaitGroup
    table := make(chan int)
    ball := 1

    wg.Add(2)
    go func(){
        play(&player{
            name: "Zhang",
            successRatio: 90,
        }, table)
        wg.Done()
    }()

    go func(){
        time.Sleep(time.Millisecond)
        play(&player{
            name: "cizixs",
            successRatio: 80,
        }, table)
        wg.Done()
    }()

    table <- ball

    wg.Wait()
    fmt.Println("Game Over")
}
```

运行上面的程序，会出现类似下面的结果：

```
1 Zhang
2 cizixs
3 Zhang
4 cizixs
5 Zhang
6 cizixs
Zhang lose.
cizixs win!!!
Game Over
```
