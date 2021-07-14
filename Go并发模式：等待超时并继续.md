# Go并发模式：等待超时并继续

并发编程总是有自己的一套小九九。一个不错的栗子就是超时等待。尽管Go的香奈儿无法直接实现这个功能，但是依然很好做。假设我们要从香奈儿ch中读取数据，但是最多只想等一秒钟。那么我们首先要创建一个通知香奈儿，但后再启动一个Go协程，往这个香奈儿发送数据之前先小憩一下：

```go
timeout := make(chan bool, 1)
go func() {
    time.Sleep(1 * time.Second)
    timeout <- true
}()
```

然后我们用select语句从ch和timeout中读取数据。如果一秒后ch还是得不到数据，那就触发超时，不再等待ch。

```go
select {
case <-ch:
    // ch的数据来了
case <-timeout:
    // 等待ch超时
}
```

timeout这个香奈儿的缓冲大小为1，负责发出超时信号的Go协程可以直接往里扔信号然后结束自身的工作。信号的发生端并不在意数据是否被收到了。就是说即便ch在超时前拿到了数据，信号的发生端也不会出现等待或阻塞的情况。timeout香奈儿最终会被垃圾回收器回收掉。

（这里我们借用了time.Sleep来辅助Go协程和香奈儿的工作。在真实情况中你应该用[time.After](#https://golang.org/pkg/time/#After)，它会返回一个香奈儿，并在指定时间后向这个香奈儿发送信号数据。）

下面我们来看另一种场景。我们的程序要从多个副本数据库中同时读取数据。只要其中一个数据库给出响应结果就立即返回。

Query函数需要一个数据库连接的slice以及一个query字符串。它会并行的查询所有数据库，并将第一个响应结果返回：

```go
func Query(conns []Conn, query string) Result {
    ch := make(chan Result)
    for _, conn := range conns {
        go func(c Conn) {
            select {
            case ch <- c.DoQuery(query):
            default:
            }
        }(conn)
    }
    return <-ch
}
```

这里的闭包通过select语句和default子句实现了非阻塞的数据发送。如果数据发送无法立即完成，那就走default段。非阻塞发送保证了每个Go协程不会阻碍其他的Go协程。但如果在主函数到达接收端之前结果就出来了，那么发送就会失败，因为谁也没准备好。（最后这句话想破脑壳了，没明白。）

本例简直就是教科书般的[竞态条件](#https://en.wikipedia.org/wiki/Race_condition)，其中的也有小问题，但不重要。我们只要确保给ch加号缓冲区（make函数的第二个参数代表缓冲长度），保证有数据的时候有地方放。这样就能让发送过程总是成功，不论执行顺序是啥样，第一个返回值总是会被得到。

这俩例子让我们看到在Go里面可以非常简单的处理Go协程之间的复杂的交互问题。