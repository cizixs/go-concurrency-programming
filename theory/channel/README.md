# goroutine 和 channel

go 语言提供了 goroutine 作为并发的实体，而不是传统编程语言中的进程和线程机制。
goroutine 可以看做轻量级的线程，可以轻松创建成千上万的 goroutine 共同运行，它们的调度
由 go 语言运行时负责，而不是操作系统内核负责。

goroutine 之间通信需要通过 channel，channel 一般翻译成管道，它是一种类似于队列、unix 管道和 socket 的存在，
支持读写两种操作，一个 goroutine 往里面写入数据，另外一个 goroutine 就能从里面读取数据。而且 channel 的读写是
并发安全的，使用起来不用担心数据不一致的问题。

goroutine 和 channel 并发编程中最重要的两个概念，这部分我们就讲解它们两个的使用方法和技巧。
