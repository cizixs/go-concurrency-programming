# 带缓存的 channel

前面介绍的 channel 作用更多是同步，在乒乓球游戏中，channel 是为了保证两个队员按照顺序执行。
但是并发最重要的功能是让多个运行实体（goroutine）能够**同时**做事情，而不是像打乒乓球那样，
一方接球的时候，另外一方只能在那等着什么都不做。

提到并发模型，最经典的就是[生产者和消费者问题](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem)。
这个问题的描述是这样的：有一个生产数据的实体，称为生产者；
另外有一个消费数据的实体，称为消费者，它们之间通过固定大小的队列作为缓存来通信。生产者把产生数据，
并把数据放到队列中；同时，消费者从队列中取出数据，执行任务。而且，生产者在队列满的时候不会继续
往里面放数据，消费者在队列空的时候不能从里面读数据。

和乒乓球游戏不同，生产者和消费者不需要互相等待，因为在实现中，它们根本不知道对方的存在。
这个问题更多关注如何保证数据传输的正确性，生产者和消费者是解耦的，而且因为缓存队列的出现，
可能有不同的处理速率。

这一节我们看看如何用 go 语言来解决生产者和消费者的问题。

## buffered channel

生产者和消费者之间通过缓存队列通信，不仅使它们功能解耦，不需要知道互相的存在。
在并发上还有一个重要的功能：不需要等待（准确的说，避免了大部分情况的等待，因为在缓存队列
满或者为空的时候还是会等待）。生产者制造出东西，没必要一定要等到有人来消费才能继续下去。

go 语言支持有缓存和无缓存的 channel, 前面几节讲到 channel 是通过 make 关键词创建的：

```
ch := make(chan int)
```

但其实，在创建 channel 的时候可以带上第二个参数，表示 channel 可以缓存多少个数据：

```
// 创建一个能缓存 10 个整数的 channel
ch := make(chan int, 10)
```

缓存 channel 的特点是，当往里面写入数据时，如果缓存还没有满，则会立即写入成功，不会阻塞等待；
当从里面读取数据的时候，如果缓存不空，则读取会立即成功，也不会阻塞等待。

无缓存的 channel 只是有缓存 channel 的特例，可以看做缓存长度为 0。也就是说，下面两种定义是等价的：


```
// 创建一个无缓存的 channel
ch := make(chan int)

// 等价于，缓存长度为 0 的 channel
ch := make(chan int, 0)
```

`cap` 函数可以获取有缓存 channel 缓存区的长度：

```
capacity := cap(ch)
```

`len` 函数可以获取有缓存 channel 中当前有多少数据，但是在一个并发的程序中，这个数据是
一直处于快速变化的，因此只适合作为参考（比如日志、测试或者调优）。

举个我们取快递的例子，无缓存 channel 是快递员打电话给你，一定要等你亲手签收；有缓存 channel
更像是快递员直接把包裹放到前台或者快递柜，等你有空的时候自己取就行。只有当快递非常重要时（
goroutine 之间同步非常重要），才会采取前者；更一般的情况（goroutine 之间只是为了交换数据，对
同步不感兴趣），我们倾向于后者。

## channel 的方向性

乒乓球游戏中，双方队员都需要发球和接球，在代码中表现为要对 `table channel` 做读取数据和写入数据两种操作。
但是在生产者和消费者中，生产者只会往 channel 中写入数据，消费者只会从 channel 中读取数据，这种情况
在实际中很常见，channel 作为参数传递给函数时基本上都能确定它是只读还是只写的。

从安全性角度考虑，应该秉承最小权限原则，让生产者不能从 channel 中读取数据，让消费者不能从 channel 中写入数据。
为此，go 语言定义了单向 channel 类型，只暴露读写操作中的一个。比如 `chan<- int` 是 int 类型的写 channel，
只允许往里面写入数据，不允许从里面读取数据；相反的，`<-chan int` 是 int 类型的读 channel，只允许从里面
读取数据，不允许往里面写入数据。

**NOTE：**在编译的时候，go 就会检测单向 channel 是否被正常使用，避免了运行时可能产生的错误。

另外，因为 close 的作用是保证不会再往 channel 中写入数据（还记得吧，我们可以从已经关闭的 channel 中读取数据），
所以只有写 channel 才能调用 close 函数，关闭只读 channel 会导致编译错误。

细心的读者可能发现，一个完全只读或者完全只写的 channel 是没有实际意义的。在实际应用中，
更多的是创建一个双向的 channel，然后根据使用情况把它转换成只读或者只写的 channel。

## 生产者-消费者 go 语言解决方案

跳过一个生产者 VS 一个消费者，以及一个生产者 VS 多个消费者，和一个消费者 VS 多个生产者的情况，
我们直接来看多个生产者和多个消费者的情况，完整的代码如下：

```
package main 

import (
    "fmt"
    "sync"
    "time"
)

const (
    // NumOfProducers 表示要运行多少个生产者
    NumOfProducers = 3

    // NumOfConsumers  表示要运行多少消费者
    NumOfConsumers = 5
)

// Producer 生产者结构体，只有一个字段，用来标识生产者的 ID
type Producer struct {
    producerID int
}

// 构建函数，返回一个生产者指针对象
func newProducer(ID int) *Producer {
    return &Producer{
        producerID: ID,
    }
}

// Run 就是生产者的核心逻辑，产生数据，并放到 channel 中
func (p* Producer) Run(ch chan<- string){
    for i:=0; i<5; i++{
        fmt.Printf("producer %d put data %d\n", p.producerID, i)
        data := fmt.Sprintf("data %d from producer %d", i, p.producerID)
        ch <- data

		// 休息一段时间模拟生产者工作时间花费
        time.Sleep(time.Microsecond * 10)
    }
}

// Consumer  消费者结构体，也有一个标识消费者身份的 ID
type Consumer struct {
    consumerID int
}

func newConsumer(ID int) *Consumer {
    return &Consumer{
        consumerID: ID,
    }
}

// Run 消费者的核心逻辑：不断从 channel 中读取数据进行处理
func (c *Consumer) Run(ch <-chan string){
    for {
        data, ok := <- ch
        if !ok {
            fmt.Printf("consumer %d: detect channel close\n", c.consumerID)
            return
        }

        fmt.Printf("consumer %d got: %s\n", c.consumerID, data)
        time.Sleep(time.Microsecond * 10)
    }
}

func main(){
    buffer := make(chan string, 10)

    // 以 goroutine 运行多个生产者
    prodWg := sync.WaitGroup{}
    prodWg.Add(NumOfProducers)
    for i:=0; i<NumOfProducers;i++ {
        go func(ID int){
            p := newProducer(ID)
            p.Run(buffer)
            prodWg.Done()
        }(i+1)
    }

	// 以 goroutine 运行多个消费者
    consumerWg := sync.WaitGroup{}
    consumerWg.Add(NumOfConsumers)
    //run consumers
    for i:=0; i<NumOfConsumers; i++{
        go func(ID int){
            c := newConsumer(ID)
            c.Run(buffer)
            consumerWg.Done()
        }(i+1)
    }

    // 等生产者都运行完成，则关闭 channel
    prodWg.Wait()
    close(buffer)

    // 消费者任务完成后，则退出程序
    consumerWg.Wait()
    fmt.Printf("exit...\n")
}
```

这段程序中，我们分别把生产者和消费者都封装为一个结构体，每个结构体有一个用来标识身份的 ID。
然后各自定义了 `Run` 方法接受 channel 作为参数，运行核心的逻辑。

在实际的程序中，生产者和消费者很可能会一直运行，不会自动退出，除非程序被手动结束。而在我们的例子中，
为了让运行结束，我们让每个生产者只生产特定数量的数据；然后关闭 channel，当消费者读取数据发现 channel 
关闭了，就知道已经没有数据要处理，因此也会退出。

可以看到，我们只创建一个 channel，然后传递给不同的 goroutine 使用，而且这个 channel 是有缓存的，可以
存放 10 个数据：

```
buffer := make(chan string, 10)
```

其次，为了等待生产者和消费者 goroutine 运行完成，我们封装了匿名函数。需要注意的是，
匿名函数接受 ID 作为参数，每次调用会把 `i+1` 传递过去，这是函数闭包的特性；如果直接使用下面的代码：

```
for i:=0; i<NumOfProducers;i++ {
    go func(){
        p := newProducer(i+1)
        p.Run(buffer)
        prodWg.Done()
    }()
}
```

不使用参数传递，而是直接使用 `i+1`，那么所有的生产者内部变量 `producerID` 其实都指向同一个值，也就是说它们的
ID 将会一样。

最后，我们在 main 函数中创建的 channel 是双向的，但是生产者接收的 channel 是只写的，而消费者接受的 channel
是只读的。这是因为在做参数传递时，go 会自动做类型转换。

运行以上程序，可能会得到类似下面的结果：

```
producer 2 put data 0
producer 1 put data 0
producer 3 put data 0
consumer 2 got: data 0 from producer 3
consumer 1 got: data 0 from producer 2
consumer 5 got: data 0 from producer 1
producer 1 put data 1
consumer 3 got: data 1 from producer 1
producer 3 put data 1
consumer 4 got: data 1 from producer 3
producer 2 put data 1
consumer 2 got: data 1 from producer 2
producer 3 put data 2
producer 1 put data 2
consumer 1 got: data 2 from producer 3
producer 2 put data 2
consumer 3 got: data 2 from producer 2
consumer 5 got: data 2 from producer 1
producer 3 put data 3
consumer 4 got: data 3 from producer 3
producer 1 put data 3
consumer 2 got: data 3 from producer 1
producer 2 put data 3
consumer 1 got: data 3 from producer 2
producer 3 put data 4
producer 1 put data 4
consumer 5 got: data 4 from producer 1
consumer 3 got: data 4 from producer 3
producer 2 put data 4
consumer 4 got: data 4 from producer 2
consumer 2: detect channel close
consumer 3: detect channel close
consumer 5: detect channel close
consumer 1: detect channel close
consumer 4: detect channel close
exit...
```
因为并发程序的特定，每次运行的结果不一定完全相同。

充当缓存队列的 channel 有效解决了生产者和消费者之间数据同步的问题，而且也能缓解速率不平衡的问题。
但是后面这个问题并没有那么简单，如果生产者生产数据的速率明显大于消费者，缓存队列会经常是满的，
那么生产者就需要等待；反之如果消费者消费数据的速率明显大于生产者，缓存队列会经常为空，消费者需要等待。
一般情况下，应该事先预估两者的速率，然后配置合适的缓存大小以及生产者和消费者的数量。
