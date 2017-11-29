# 关闭 channel

在上一节，我们介绍了 channel 的基本概念，并使用 channel 模拟了打乒乓球的过程，最后留下了一个问题：怎么结束乒乓球游戏？
这一节，我们将解答这个问题。

不妨我们先思考一下，现实中一场乒乓球是怎么结束的？无非是两种情况：一个是没接住对方的球；另外是没有打到对方桌面上。
我们可以把这两种情况简化为一种情况：某个选手接到球之后，直接失败了。因为不同选手的能力不同，所以失败的几率也不同，
根据选手的接球成功率，我们可以用简单的算法判断它每次接球是成功还是失败。
一方选手接球失败，另外一方就能直接看到，接收到这个消息，准备下一个回合的比赛，而不是像之前那样傻等着对方继续发球过来。

回到我们 goroutine 的代码中，我们可以让某个选手接球失败之后就直接退出 goroutine，但是另外一个 goroutine 并不会自动
知道这个 goroutine 已经退出，还是会执行 `ball := <- table` 的指令，傻傻地等待对方发球过来。

当然，我们可以创建另外一个 channel 单独传递某一方失败的消息，另外一方定时去查看这个 channel 来判断是否比赛已经结束。
但这无疑让整个问题变得很复杂，其实我们可以用 channel 自带的另外一个操作来完成相同的功能，那就是接下来要讲解的关闭
channel。

## channel close 操作


## 完整的乒乓球模拟代码


```go
package main

import (
    "fmt"
    "sync"
    "time"
    "math/rand"
)

func init(){
    rand.Seed(time.Now().UnixNano())
}

type player struct {
    name string
    successRatio int // a number between [0, 100]
}

func play(p *player, table chan int){
    for {
        ball, ok := <- table

        // the channel is closed, which means the other side loses, so we win
        if !ok {
            fmt.Printf("%s win!!!\n", p.name)
            return
        }
        
        // calculate if we lose
        r := rand.Intn(100)
        if r > p.successRatio {
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
