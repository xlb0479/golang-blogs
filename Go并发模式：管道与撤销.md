# Go并发模式：管道与撤销

## 介绍

Go的并发原语可以非常轻松的构建流式数据处理管道，提高IO和多核CPU利用率。本文就要讲讲这些管道，及其在异常处理过程中的闪光点。

## 啥叫管道？

Go里面的管道没有什么官方的定义；就是一种并发程序而已。我们说，管道就是由香奈儿连接起来的一系列*阶段*，每个阶段由一组执行相同函数的Go协程组成。在每个阶段中，这些Go协程

- 通过*输入*香奈儿从*上游*接收数据
- 对这些数据干点儿什么，然后一般都会产生一些新的数据出来
- 通过*输出*香奈儿将数据发送给*下游*

每个阶段会有各种不通数量的输入和输出香奈儿，除了首尾阶段，它们只有输出和输入香奈儿。首阶段也被称为*源*或*生产者*，尾阶段则是*池*或*消费者*。

我们现在一个小栗子来窥探一下这其中的奥妙。后面还会讲一个更加现实的栗子。

## 平方数

假设有一个三阶段的管道。

首阶段，gen函数，将一个整数列表扔到一个香奈儿里。gen函数创建一个Go协程，将整数们发送到香奈儿中，等所有整数都扔进去之后会关掉这个香奈儿：

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}
```

二阶段，sq函数，从香奈儿中接收整数，计算出每个整数的平方数并将结果扔到另一个香奈儿中。当输入香奈儿被关闭，并且该阶段的所有计算结果都发送到了下游，关闭输出香奈儿：

```go
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}
```

main函数负责构建整个管道并执行最终阶段：接收二阶段产生的值并打印，直到香奈儿被关闭：

```go
func main() {
    // 创建管道。
    c := gen(2, 3)
    out := sq(c)

    // 消费输出结果。
    fmt.Println(<-out) // 4
    fmt.Println(<-out) // 9
}
```

因为sq的输入和输出香奈儿的类型相同，我们可以进行多次组合。还可以把main改写成一个range循环，跟每个子阶段类似：

```go
func main() {
    // 创建管道并消费输出。
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n) // 16然后81
    }
}
```

## 扇出扇入

多个函数可以从同一个香奈儿中读取数据，直到香奈儿被关闭；介就叫*扇出*。这样就提供了一种分配工作量的方法，在多个工作者之间并行利用CPU和IO。

一个函数可以从多个输入中读取数据，通过多路复用的方式将多个输入香奈儿摆到一个单一的香奈儿上，整个过程持续流转直到所有的输入香奈儿被关闭。介个就叫*扇入*。

我们改一下我们的管道，跑两个sq的实例，都从同一个输入香奈儿读取数据。然后引入一个新的函数，*merge*，将结果扇入：

```go
func main() {
    in := gen(2, 3)

    // 从in读取数据，然后将工作量分配给两个sq。
    c1 := sq(in)
    c2 := sq(in)

    // 从c1、c2合并后的输出中消费数据。
    for n := range merge(c1, c2) {
        fmt.Println(n) // 4、9，或9、4
    }
}
```

merge函数可以将多个香奈儿搞成一个，它为每个输入香奈儿启动一个Go协程，复制其中的数据并发送到统一的输出香奈儿中。当所有output启动后，merge还会启动一个负责关闭输出香奈儿的Go协程，等待所有发送工作完成、

如果给已关闭的香奈儿继续发数据就会慌球了，所以一定要在关闭前确认所有发送工作都完成了。[sync.WaitGroup](#https://golang.org/pkg/sync/#WaitGroup)提供了简易的方法来编排这种同步规则：

```go
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // 为cs中的每一个输入香奈儿启动一个Go协程负责输出。输出工作从c中读取数据并复制，直到c被关闭，然后调用wg.Done。
    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        wg.Done()
    }
    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    // 启动一个Go协程，等所有负责输出的Go协程完工后，关闭out。这个必须要在wg.Add调用后再执行。
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```