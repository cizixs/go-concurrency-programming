## 总结

- goroutine 是 go 语言中可以并发运行的实体
    - 创建 goroutine 只需要在正常的函数前面加上 `go` 关键字就行
    - 使用 `sync.WaitGroup` 可以等待一组 goroutine 运行结束
- channel 可以用来在 goroutine 之间传输数据
    - channel 可以读数据，也可以写数据，go 语言会保证数据的顺序和安全
    - 无缓存（unbuffered）channel 需要读写数据的 goroutine 同时进行才能完成；缓存（buffered）channel 允许 goroutine 异步写入数据，不是每次都要等待
    - channel 是有方向的，分为只读或者只写。一般在创建一个双向 channel 之后，通过类型转换把不同方向的 channel 交给不同的处理函数
    - 关闭 channel 之后，往里面写数据会报错，从里面读数据会返回数据类型的空值
    - 可以使用 `for ... range` 循环地从 channel 中读取数据；`select` 可以从多个 channel 读写操作中选择一个可用的
- go 语言还提供了锁机制
