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

## 卡了

我们的管道函数都有着某种神秘的模式：

- 负责发送的任务都会在所有发送任务完成后关闭输出香奈儿。
- 从输入香奈儿读取数据的任务都是一直读取，直到香奈儿被关掉。

这样，接收端就可以用range循环来写，保证所有数据发送到下游后所有Go协程都可以退出了。

但在真实的管道中，接收端并不一定能收全数据。有的时候一开始就设计成这样了：接收端可能只需要一部分数据就可以干活儿了。更常见的是，接收端因为上游传过来一个错误值而提前退出。不管哪种情况吧，接收端都不需要等待剩下的值了，而且既然不需要了，那么上游也没必要继续生产了。

在我们的栗子中，如果某个接收端无法消费输入数据，那么发送端就会被无限阻塞：

```go
    // 读取第一个值。
    out := merge(c1, c2)
    fmt.Println(<-out) // 4 or 9
    return
    // 因为没有继续从out中读取第二个值，
    // 上游负责输出的某个Go协程就夯住了。
}
```

这就出现资源泄漏了：Go协程要占用内存以及其他运行时资源，Go协程栈中的堆引用导致数据无法被垃圾回收。Go协程是无法被垃圾回收的；它们必须要自食其力自己退出。

所以当下游无法消费时，上游要及时退出才行。一种方法就是给香奈儿加缓冲。有了缓冲就可以保留一定数据量的数据；如果缓冲区中还有地儿，发送操作就会立即完成：

```go
c := make(chan int, 2) // 缓冲区大小为2
c <- 1  // 立即完成
c <- 2  // 立即完成
c <- 3  // 一直阻塞，直到另一个Go协程执行<- c，并且拿到了1
```

如果创建香奈儿的时候就只要一共要发多少数据，加上缓冲就能简化代码了。比如我们可以改一下gen，直接把整数列表复制到缓冲中，这样还少了一个Go协程：

```go
func gen(nums ...int) <-chan int {
    out := make(chan int, len(nums))
    for _, n := range nums {
        out <- n
    }
    close(out)
    return out
}
```

继续解救一下上面被阻塞的管道，merge返回的香奈儿，我们可以给加个缓冲：

```go
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int, 1) // 给未读消息留下地方
    // ... 剩下的地方不改 ...
```

这么改的话，用在这儿还行，实际上不是什么好路子。缓冲大小之所以弄成1，是因为我们已经直到merge会收到多少数据，下游会处理多少数据。健壮性就很差：如果给gen再多加一个值，或者下游少读几个值，又会出现Go协程阻塞了。

所以，我们需要让下游能够通知上游，告诉它们我们已经吃饱了，不想再吃了，再吃就吐了，吴亦凡的瓜。

## 显式撤销

如果还没从out收完数据main函数就想退出，那就必须要通知上游的Go协程放弃抵抗，停止继续发送数据。具体方法就是给done发数据。它发了两遍，因为目前是有两个嵌在的Go协程会被阻塞：

```go
func main() {
    in := gen(2, 3)

    // 从in读取数据，然后将工作量分配给两个sq。
    c1 := sq(in)
    c2 := sq(in)

    // 读取第一个值。
    done := make(chan struct{}, 2)
    out := merge(done, c1, c2)
    fmt.Println(<-out) // 4 or 9

    // 告诉发送端我们该撤了。
    done <- struct{}{}
    done <- struct{}{}
}
```

发送端的Go协程要改一下发送逻辑，改成select语句，当out或者done有数据的时候才会继续执行。done中的值是空结构体类型，因为值本身没什么价值：它只是用来代表一个事件，告诉发送端不要再给out发东西了。执行output的Go协程在输入香奈儿c上循环消费，因此上游不会被阻塞。（后面我们会讲这里该如何提前结束。）

```go
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // 为每个输入香奈儿启动一个输出Go协程。输出函数将c中的值复制到out中，直到c被关闭或者从done中收到了消息，然后就会调用wg.Done。
    output := func(c <-chan int) {
        for n := range c {
            select {
            case out <- n:
            case <-done:
            }
        }
        wg.Done()
    }
    // ... 剩下的不改 ...
```

这种方式有个问题：*每个*下游接收端需要知道上游可能会有多少个生产者被阻塞，然后搞一个小编排，通知上游提前结束。老是惦记着这种事儿就很烦，吴亦凡的弟弟容易烦，而且还容易出错。

我们需要能够让未知数量的Go协程收到消息。在Go中，我们可以把香奈儿关了，因为[如果在一个已经关闭的香奈儿上继续消费，操作会立即完成，并且得到元素类型对应的零值。](#https://golang.org/ref/spec#Receive_operator)

这样的话，main函数就可以直接关闭done香奈儿，解除所有发送端的阻塞状态。这种关闭操作就相当于给所有发送端做了个广播。现在我们把管道中的*每个*函数都扩展一下，让它们把done作为一个参数传进去，通过defer语句来组织香奈儿的关闭操作。这样所有的返回路径都能够保证管道的每个阶段都能正常退出。

```go
func main() {
    // 建立一个done香奈儿，整个管道共享，
    // 管道退出时关闭，通知所有Go协程准备退出。
    done := make(chan struct{})
    defer close(done)

    in := gen(done, 2, 3)

    // 从in读取数据，然后将工作量分配给两个sq。
    c1 := sq(done, in)
    c2 := sq(done, in)

    // 读取第一个值。
    out := merge(done, c1, c2)
    fmt.Println(<-out) // 4 或 9

    // defer调用会关闭done。
}
```

现在，当done关闭时，管道的每个阶段都可以进行退出操作。merge中的output函数可以直接返回，无需等待抽干输入香奈儿中的数据，因为它知道上游的sq在done关闭时就不再发送数据了。output通过defer保证在所有的返回路径上都调用了wg.Done。

```go
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // 为每个输入香奈儿启动一个输出Go协程。输出函数将c中的值复制到out中，直到c被关闭或者从done中收到了消息，然后就会调用wg.Done。
    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out <- n:
            case <-done:
                return
            }
        }
    }
    // ... 剩下的不改 ...
```

同样，sq也可以在done关闭时做出回应。sq也是通过defer语句保证在所有的返回路径上都能顺利关闭out香奈儿：

```go
func sq(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-done:
                return
            }
        }
    }()
    return out
}
```

这里给出一些构建管道的指导方针：

- 当前阶段完成所有发送操作后要关闭输出香奈儿。
- 当前阶段从输入香奈儿中读取数据，直到香奈儿被关闭，或者发送端非阻塞。

如果管道中的发送端非阻塞，要么就确保有足够的缓冲来接收数据，要么就是在消费端丢弃香奈儿时主动通知发送端。

## 树摘要

来搞一个更现实的管道。

MD5是一个消息摘要算法，可以用于文件校验码。系统自带的md5sum命令可以输出一组文件的摘要值。

```shell
% md5sum *.go
d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```

现在我们要自己搞一个md5sum程序，我们要的参数是一个目录，然后输出目录中每个文件的摘要，按文件名排序。

```shell
% go run serial.go .
d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```

我们的主函数调用一个辅助函数MD5All，它会返回一个从路径名到摘要值的map，然后我们做排序并输出结果：

```go
func main() {
    // 计算指定文件夹内所有文件的MD5值，
    // 然后输出按路径名排序后的结果。
    m, err := MD5All(os.Args[1])
    if err != nil {
        fmt.Println(err)
        return
    }
    var paths []string
    for path := range m {
        paths = append(paths, path)
    }
    sort.Strings(paths)
    for _, path := range paths {
        fmt.Printf("%x  %s\n", m[path], path)
    }
}
```

我们主要就是研究下MD5All这个函数。在[serial.go](#https://blog.golang.org/pipelines/serial.go)中，没有用并发实现，只不过就是一个遍历、读取然后计算的过程。

```go
// MD5All以root为根目录读取整棵文件树，返沪从文件路径到MD5值的map。如果遍历出现问题，或者读取文件异常，MD5All就返回一个error。
func MD5All(root string) (map[string][md5.Size]byte, error) {
    m := make(map[string][md5.Size]byte)
    err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        if !info.Mode().IsRegular() {
            return nil
        }
        data, err := ioutil.ReadFile(path)
        if err != nil {
            return err
        }
        m[path] = md5.Sum(data)
        return nil
    })
    if err != nil {
        return nil, err
    }
    return m, nil
}
```

## 并行摘要计算

