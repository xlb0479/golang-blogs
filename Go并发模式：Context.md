# Go并发模式：Context

## 介绍

在Go服务中，每个进来的请求有单独的Go协程来处理。请求处理时通常还会启动其他的Go协程来访问数据库或者RPC服务。一个请求对应的这一组Go协程，它们在运转时通常会访问一些请求特定的数据，比如请求对应的终端用户、授权token，以及请求的处理截止时间。当请求被取消或者发生超时，对应的Go协程应当快速退出，这样系统就可以回收它们占用的资源。

在谷歌，我们搞了一个context包，可以用来放一些请求范围内的值，以及撤销信号，还有整个API界限范围的截止时间，影响着该请求对应的所有Go协程。这个包可以通过[context](#https://golang.org/pkg/context)来访问。本文就是要介绍如何使用这个包并实现一个完整的栗子。

## Context

context包的核心就是Context类型：

```go
// 一个Context包含了一个截止时间、撤销信号以及整个API界限范围内可用的请求域的值。这里面的方法都是可以被多个Go协程同时访问的。
type Context interface {
    // Done返回一个香奈儿，当Context被取消或发生超时的时候，这个香奈儿就会被关闭。
    Done() <-chan struct{}

    // Done的香奈儿被关闭后，Err指明context为啥被取消。
    Err() error

    // 如果有的话，Deadline返回这个Context什么时候会被取消。
    Deadline() (deadline time.Time, ok bool)

    // Value返回key对应的值，没有的话就是nil。
    Value(key interface{}) interface{}
}
```

（这里的描述简写了一下；标准的还是要看[godoc](#https://golang.org/pkg/context)。）

Done方法返回了一个香奈儿，为当前Context下的这些函数提供撤销信号：当香奈儿被关闭时，这些函数应该放弃它们手头的活儿并立刻返回。Err方法返回了一个错误值，表明Context因何被取消。[管道和撤销](#Go并发模式：管道和撤销.md)一文中详细介绍了Done的思想。

Context*没有*Cancel方法，正如Done香奈儿是只读香奈儿一样：收到撤销信号的函数一般来说不会是发出该信号的函数。尤其是什么呢，当一个操作为内部操作启动了Go协程，那么这些嵌套操作就不能取消它们的父级操作。后面介绍的WithCancel函数提供了一种能够取消某个新的Contenxt的值的方法。

一个Context可以安全的被多个Go协程同时访问。可以把一个Context传给任意多个Go协程，取消该Context的时候就会给这些协程们统一发出信号。

Deadline方法可以让函数们用来决定它们是否根本就不需要开始它们的工作；如果所剩时间无几，可能就不值得了。代码还可以用截止时间来设置IO操作的超时。

Value可以允许Context携带请求域中的数据。这些数据必须都要支持同时被多个Go协程访问。

## 继承context

context包中提供了一些函数能够让新的Context值*继承*已有的Context。这些值组成了一棵树：当Context被取消时，继承它的所有Context也都会被取消。

Background时所有Context树的根；它永远不会被取消：

```go
// Background返回一个空的Context。它永远不会被取消，没有截止时间，也没有值。Background一般用在main、init、以及测试中，作为所有请求的顶层Context。
func Background() Context
```

WithCancel和WithTimeout返回了有继承的Context，它们可以比父级Context更早取消。请求所关联的Context通常在请求处理返回后就会被取消。WithCancel可以在多副本请求中取消冗余的请求。WithTimeout可以为请求设置下游服务的截止时间：

```go
// WithCancel返回parent的一个副本，当parent.Done被关闭或调用cancel时它的Done也会被关闭
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// CancelFunc可以取消一个Context。
type CancelFunc func()

// WithTimeout返回parent的一个副本，当parent.Done被关闭或调用cancel或发生超时，它的Done也会被关闭。如果设置了timeout，新Context的截止时间从当前时间加timeout，和父级截止时间二者中选择最近的那个时间。如果计时器仍在工作，cancel函数会释放它的资源。
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

WithValue可以将请求域的值根Context关联起来：

```go
// WithValue也是返回一个副本，Value方法会返回key对应的val值。
func WithValue(parent Context, key interface{}, val interface{}) Context
```

说了半天不如来点儿实际的。

## 栗子：谷歌搜索

我们的栗子就是要搞一个HTTP服务，处理类似/search?q=golang&timeout=1s这种请求，然后将“golang”的查询透传给[谷歌搜索API](#https://developers.google.com/web-search/docs/)，然后再渲染返回结果。timeout参数用来告诉服务端指定时间过后就要取消本次请求。

代码分成了三个包：

- [server][#https://blog.golang.org/context/server/server.go]提供main函数以及/search的handler。
- [userip](#https://blog.golang.org/context/userip/userip.go)提供了可以获取用户IP的方法，并将它关联到Context。
- [google](#https://blog.golang.org/context/google/google.go)提供了Search函数，用来向Google发出请求。

## server

[server](#https://blog.golang.org/context/server/server.go)程序提供/search?q=golang这种接口，然后返回谷歌搜索golang的前几条结果。它注册了handleSearch来处理/search接口。处理器创建了一个初始的Context，名为ctx，并让它在处理器返回的时候关闭。如果请求参数包含timeout，当超时时间到达时Context自动取消：

```go
func handleSearch(w http.ResponseWriter, req *http.Request) {
    // ctx是该处理器的Context。调用cancel可以关闭ctx.Done，也就是给所有由该处理器发起的请求发出了取消信号。
    var (
        ctx    context.Context
        cancel context.CancelFunc
    )
    timeout, err := time.ParseDuration(req.FormValue("timeout"))
    if err == nil {
        // 请求包含超时参数，创建自带超时光环的Context。
        ctx, cancel = context.WithTimeout(context.Background(), timeout)
    } else {
        ctx, cancel = context.WithCancel(context.Background())
    }
    defer cancel() // handleSearch返回时取消ctx。
```

通过调用userip包，处理器可以从请求中提取出客户端的IP地址。后续请求会用到这个客户端IP，所以handleSearch把它放到了ctx中：

```go
    // 校验搜索参数。
    query := req.FormValue("q")
    if query == "" {
        http.Error(w, "no query", http.StatusBadRequest)
        return
    }

    // 将用户IP保存到ctx，其它包里的代码就能继续用了。
    userIP, err := userip.FromRequest(req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    ctx = userip.NewContext(ctx, userIP)
```

处理器调用google.Search，传入ctx和query：

```go
    // 执行谷歌搜索并打印结果。
    start := time.Now()
    results, err := google.Search(ctx, query)
    elapsed := time.Since(start)
```

如果搜索成功，处理器要渲染结果：

```go
    if err := resultsTemplate.Execute(w, struct {
        Results          google.Results
        Timeout, Elapsed time.Duration
    }{
        Results: results,
        Timeout: timeout,
        Elapsed: elapsed,
    }); err != nil {
        log.Print(err)
        return
    }
```

## userip

[userip](#https://blog.golang.org/context/userip/userip.go)包可以从请求中提取出客户端IP，然后还能关联到Context上。Context提供了key-value映射，key和value的类型都是interface{}。key的类型必须要支持相等判断，value则必须要支持可以同时被多个Go协程使用。像userip这种包，它会把这些映射关系隐藏起来，并且要求使用强类型来访问指定的Context的value。

为了避免key冲突，userip中定义了一个未导出的key类型，然后用这种类型的值作为Context中的key。

```go
// key类型未导出，避免和其它包中的key冲突。
type key int

// userIPkey就是用户IP在Context中的key。随便给它赋了个零。如果此包中还要定义其他key，那就要给不同的整数值了。
const userIPKey key = 0
```

FromRequest从http.Request中提取出userIP：

```go
func FromRequest(req *http.Request) (net.IP, error) {
    ip, _, err := net.SplitHostPort(req.RemoteAddr)
    if err != nil {
        return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
    }
```

FromContext可以从Context中提取出userIP：

```go
func FromContext(ctx context.Context) (net.IP, bool) {
    // 如果没有key对应的值，ctx.Value返回nil；如果返回nil，则net.IP的类型断言结果为ok=false。
    userIP, ok := ctx.Value(userIPKey).(net.IP)
    return userIP, ok
}
```

## google

