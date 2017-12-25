# 互斥锁

锁机制是传统的编程语言（比如 Java、C 等）对于并发程序的解决方案，线程读写数据区之前先进行加锁操作，只能加锁成功才能执行读写逻辑，执行完成后需要释放锁，
以供其他线程能够使用。当某个线程执行加锁动作，其他想要执行相同加锁操作的线程只能等待，等到锁解开后才能继续运行。

需要注意的是，加锁其实并没有锁住任何东西，更像是在门上贴上“此门已锁”的标语，解锁就是把这个标语撕掉。
如果大家都主动去读标语，并遵守标语的规定，那么并发读写就是安全的。但是如果一个线程对关键区加锁，但是另外一个线程完全不关心
锁的事情，直接去读写关键区的数据，也是能读写成功的，并没有任何机制会阻止它这么做，当然这么做会导致并发处理的数据有问题。
所以，**一定要对所有可能并发读写的地方进行加锁和解锁的操作**，漏掉任何一个地方都会让程序出问题。

加锁和解锁的动作必须是成对出现的，如果某个线程只执行加锁操作，但是忘记执行解锁操作，那么所有要读写关键区变量的进程都会
一直处于阻塞的状态，这被称为**死锁**，在后面的章节我们会介绍死锁的检测。

go 语言也提供了锁机制，只不过不是在语言本身的语法中，而是出现在 `sync` 标准函数库。我们在之前介绍过 `sync.WaitGroup`，这一节介绍另外一个类型。
`sync.Mutex` 是 go 语言提供的互斥锁类型，它不包含任何对外公开的字段，因此声明一个该类型的变量（zero value）就能直接使用，代表着一个未被锁住的 mutex。

mutex 提供两个方法：`Lock()` 和 `Unlock()`，都不接受任何的参数，从名字中可以看出它们的作用分别是锁住当前 mutex，以及解锁当前 mutex。
多次加锁只有其中一个会成功，其他加锁的 goroutine 会处于阻塞的状态，直到锁被解开；如果对一个未被加锁的 mutex 执行解锁操作，会触发程序的
运行时错误。另外，加锁和解锁操作是独立的，完全可以由不同的 goroutine 来执行，也就是说一个 goroutine 对 mutex 加锁，
另外一个 goroutine 对同一个 mutex 解锁的行为完全是 ok 的。尽管可以这么做，我们还是不要这么做，而是尽量让 mutex 加锁和解锁操作在同一个 goroutine 
执行，并且让 mutex 只出现在同一个结构体里，不要对 mutex 进行复制和参数传递。

因为加锁和解锁的行为是成对出现的，而且不成对出现会导致错误，所以推荐在加锁之后使用 `defer` 跟着解锁的动作，这样可以减少因为人工失误导致的错误。
这种方法的锁定区域是加锁的地方一直到函数结束，如果需要对加锁的区域进行精细的控制，只能抛弃 `defer`，使用手动在需要的地方解锁的方式。

了解了这些，使用 mutex 来改进银行账户的例子就非常简单了。在 `Account` 结构体中新加一个 `sync.Mutex` 类型的字段 `mu`，它用来保护账户余额的读写操作：

```
type Account struct {
    name   string
    amount uint32
    mu     sync.Mutex
}
```

然后在存钱和查询余额的时候，分别执行加锁和解锁的操作：

```
func (a *Account) Deposit(amount uint32) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.amount = a.amount + amount
}

func (a *Account) Balance() uint32 {
    a.mu.Lock()
    defer a.mu.Unlock()
    return a.amount
}
```

其他不需要改动，对外的接口还是一样的，改动完之后运行程序，每次执行的结果都是一样的：

```
➜  exclusive-lock git:(master) ✗ go run main.go
200000
➜  exclusive-lock git:(master) ✗ go run main.go
200000
➜  exclusive-lock git:(master) ✗ go run main.go
200000
```

加锁保证了程序的正确性，但是却影响了程序的性能，因为加锁之后，其他所有需要读写数据的 goroutine 只能等待，什么事情都做不了。
为了证明 mutex 锁机制导致性能降低，我们对程序进行 benchmark 测试。因为第一个程序的写操作是有问题的，所以和加锁之后的版本进行性能测试没有什么可比性，
所以我们只测试了读取账户余额的方法，对应的 benchmark 测试用例的代码如下：

```
package main

import (
    "testing"
)

func BenchmarkAccountRead(b *testing.B) {
    a := Account{name: "cizixs", amount: 0}
    for i := 0; i < b.N; i++ {
        a.Balance()
    }
}
```

代码很短，就是基本的 go 语言 Benchmark 的例子，在没使用锁的版本运行结果如下：

```
➜  race git:(master) ✗ go test -test.bench=".*" .
goos: darwin
goarch: amd64
pkg: github.com/cizixs/playground/lock/race
BenchmarkAccountRead-4          2000000000               0.37 ns/op
PASS
ok      github.com/cizixs/playground/lock/race  0.780s
```

在使用锁机制的代码运行结果如下：

```
➜  exclusive-lock git:(master) ✗ go test -test.bench=".*" .
goos: darwin
goarch: amd64
pkg: github.com/cizixs/playground/lock/exclusive-lock
BenchmarkAccountRead-4          20000000                75.9 ns/op
PASS
ok      github.com/cizixs/playground/lock/exclusive-lock        1.604s
```

具体结果会因为机器的配置以及每次运行时系统的负载不太相同，但是可以从上面两个简单的结果看出，
加锁之后每次操作的的时间从 `0.37ns` 变成了 `75.9 ns`，如此简单的例子就能带来这么大的性能差距，在实际上更负责的代码中，锁机制带来的性能损失可能会更严重，
是我们必须要考虑中的事情。在下一节中，我们将介绍如何使用读写锁来减少某些情况下锁机制带来的性能损耗。
