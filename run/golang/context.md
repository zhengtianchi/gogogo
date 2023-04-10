Go 1.7 标准库引入 context，中文译作“上下文”，准确说它是 goroutine 的上下文，包含 goroutine 的运行状态、环境、现场等信息。

context 主要用来在 goroutine 之间传递上下文信息，包括：取消信号、超时时间、截止时间、k-v 等。

随着 context 包的引入，标准库中很多接口因此加上了 context 参数，例如 database/sql 包。context 几乎成为了并发控制和超时控制的标准做法。

> context.Context 类型的值可以协调多个 groutine 中的代码执行“取消”操作，并且可以存储键值对。最重要的是它是并发安全的。

> 与它协作的 API 都可以由外部控制执行“取消”操作，例如：取消一个 HTTP 请求的执行。







## 优雅的关闭goroutine

在很多场景下，在执行一个任务时，我们会将这个任务拆分成几个子任务，然后开启几个不同的goroutine去执行。当因为某些原因这个任务需要终止时，我们需要将这些goroutine也全都终止掉。

比如Go http包的Server中，每一个请求都有对应的goroutine去处理。请求处理函数通常会启动额外的 goroutine 用来访问后端服务，比如数据库和RPC服务。用来处理一个请求的 goroutine 通常需要访问一些与请求特定的数据，比如终端用户的身份认证信息、验证相关的token、请求的截止时间。 当一个请求被取消或超时时，所有用来处理该请求的 goroutine 都应该迅速退出，然后系统才能释放这些 goroutine 占用的资源。

下面我们举一个简单的例子，然后分别使用不同的方法去关闭例子中的goroutine。

### sync.WaitGroup 实现

```go
package main

import (
	"fmt"
	"strconv"
	"sync"
	"time"
)

var wg sync.WaitGroup

func run(task string) {
	fmt.Println(task, "start...")
	time.Sleep(time.Second * 2)
	// 每个groutine运行完毕后就释放WaitGroup的计时器
	wg.Done()
}

func main() {
	wg.Add(2)
	for i := 1; i < 3; i++ {
		taskName := "task" + strconv.Itoa(i)
		go run(taskName)
	}
	wg.Wait()
	fmt.Println("所有任务结束")
}

// task2 start...
// task1 start...
// 所有任务结束

复制代码
```

上面例子中，一个任务结束了必须等待另外一个任务也结束了才算全部结束了，先完成的必须等待其他未完成的，所有的goroutine都要全部完成才OK。

这种方式的优点：使用等待组的并发控制模型，尤其适用于好多个goroutine协同做一件事情的时候，因为每个goroutine做的都是这件事情的一部分，只有全部的goroutine都完成，这件事情才算完成。

这种方式的缺陷：在实际生产中，需要我们主动的通知某一个 goroutine 结束。

### channel + select 实现

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	stop := make(chan bool)
	go func() {
		for {
			select {
			case <- stop:
				fmt.Println("任务1 结束了")
				return
			default:
				fmt.Println("任务1 运行中")
				time.Sleep(time.Second)
			}
		}
	}()

	// 运行5秒后停止
	time.Sleep(time.Second * 5)
	stop <- true
	// 停止检测goroutine是否已经结束
	time.Sleep(time.Second * 3)
}

// 任务1 运行中
// 任务1 运行中
// 任务1 运行中
// 任务1 运行中
// 任务1 运行中
// 任务1 结束了


复制代码
```

使用channel + select的优点：比较优雅。 使用channel + select的缺点：如果有多个goroutine需要关闭怎么办？可以使用全局bool类型变量的方法，但是在为全局变量赋值的时候需要用到锁来保证协程安全，这样势必会对便利性和性能造成影响。更有甚者，如果每个goroutine中又嵌套了goroutine呢？

### context 实现

将以上的代码使用context重写：

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	// 开启goroutine，传入ctx
	go func(ctx context.Context) {
		for {
			select {
			case <- ctx.Done():
				fmt.Println("任务1 结束了")
				return
			default:
				fmt.Println("任务1 运行中")
				time.Sleep(time.Second)
			}
		}
	}(ctx)

	// 运行五秒以后停止
	time.Sleep(time.Second * 5)
	cancel()
	// 停止检测goroutine是否已经结束
	time.Sleep(time.Second * 3)
}

// 任务1 运行中
// 任务1 运行中
// 任务1 运行中
// 任务1 运行中
// 任务1 运行中
// 任务1 结束了

复制代码
```

使用context重写比较简单，当然上述只是启动一个goroutine的情况，如果有多个goroutine呢？

```go
package main

import (
	"context"
	"fmt"
	"time"
)

// 使用context控制多个goroutine
func watch(ctx context.Context, name string) {
	for {
		select {
		case <- ctx.Done():
			fmt.Println(name, "退出, 停止了")
			return
		default:
			fmt.Println(name, "运行中")
			time.Sleep(time.Second)
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go watch(ctx, "任务1")
	go watch(ctx, "任务2")
	go watch(ctx, "任务3")

	time.Sleep(time.Second * 3)
	// 通知任务停止
	cancel()
	time.Sleep(time.Second * 2)
	fmt.Println("确定任务全部停止")
}

//任务3 运行中
//任务1 运行中
//任务2 运行中
//任务2 运行中
//任务3 运行中
//任务1 运行中
//任务1 运行中
//任务2 运行中
//任务3 运行中
//任务2 退出, 停止了
//任务1 退出, 停止了
//任务3 退出, 停止了
//确定任务全部停止
复制代码
```

上述Context就像一个控制器一样，按下开关后，所有基于这个 Context 或者衍生的子 Context 都会收到通知，这时就可以进行清理操作了，最终释放 goroutine，这就优雅的解决了 goroutine 启动后不可控的问题。

## 优雅的关闭多个goroutine嵌套

```go
package main

import (
	"context"
	"fmt"
	"time"
)

// 定义一个包含context的新类型
type otherContext struct {
	context.Context
}

func work(ctx context.Context, name string)  {
	for {
		select {
		case <- ctx.Done():
			fmt.Println(name, " get msg to cancel")
			return
		default:
			fmt.Println(name, " is running")
			time.Sleep(time.Second)
		}
	}
}

func workWithValue(ctx context.Context, name string) {
	for {
		select {
		case <- ctx.Done():
			fmt.Println(name, " get msg to cancel")
			return
		default:
			value := ctx.Value("key").(string)
			fmt.Println(name, " is running value = ", value)
			time.Sleep(time.Second)
		}
	}
}

func main() {
	// 使用context.Background()构建一个WithCancel类型的上下文
	ctxa, cancel := context.WithCancel(context.Background())

	// work模拟运行并检测前端的退出通知
	go work(ctxa, "work1")

	// 使用WithDeadline包装前面的上下文对象ctxa
	tm := time.Now().Add(3 * time.Second)
	ctxb, _ := context.WithDeadline(ctxa, tm)

	go work(ctxb, "work2")

	// 使用WithValue包装前面的上下文对象ctxb
	oc := otherContext{ctxb}
	ctxc := context.WithValue(oc, "key", "andes, pass from main")

	go workWithValue(ctxc, "work3")

	// 故意 "sleep" 10秒, 让work2、work3超时退出
	time.Sleep(10 * time.Second)

	// 显示调用work1的cancel方法通知其退出
	cancel()

	// 等待work1打印退出信息
	time.Sleep(5 *time.Second)
	fmt.Println("main stop")
}

//work3  is running value =  andes, pass from main
//work1  is running
//work2  is running
//work2  is running
//work3  is running value =  andes, pass from main
//work1  is running
//work1  is running
//work3  is running value =  andes, pass from main
//work2  is running
//work3  get msg to cancel
//work2  get msg to cancel
//work1  is running
//work1  is running
//work1  is running
//work1  is running
//work1  is running
//work1  is running
//work1  is running
//work1  get msg to cancel
//main stop

复制代码
```

在使用Context的过程中，程序在底层实际上维护了两条关系链。

1. children key构成从根到叶子Context实例的引用关系，在调用With函数时，会调用propagateCancel(parent Context, child canceler) 函数进行维护，程序有一层这样的树状结构：

   ```lua
   ctxa.children --> ctxb
   ctxb.children --> ctxc
   复制代码
   ```

   这棵树提供一种从根节点开始遍历树的方法，context包的取消广播通知就是基于这棵树实现的，取消通知沿着这条链从根节点向下层节点逐层广播。当然也可以在任意一个子树上调用取消通知，一样会扩散到整棵树。

2. 在构造 context 的对象中不断地包裹 context 实例形成一个引用关系链，这个关系链的方向是相反的，是自底向上的。

   ```rust
   ctxc.Context --> oc
   ctxc.Context.Context --> ctxb
   ctxc.Context.Context.cancelCtx --> ctxa
   ctxc.Context.Context.Context.cancelCtx.Context -> new(emptyCtx) // context.Background()
   复制代码
   ```

   这个关系链主要用于切断当前Context实例和上层的Context实例之间的关系。，比如 ctxb调用了退出通知或定时器到期了， ctxb 后续就没有必要在通知广播树上继续存在，它需要找到自己的 parent ，然后执行 delete(parent.children,ctxb）， 把自己从广播树上清理掉。

## net/http包中的context

net/http包源码在实现http server时就用到了context, 下面简单分析一下:

1.首先Server在开启服务时会创建一个valueCtx,存储了server的相关信息，之后每建立一条连接就会开启一个协程，并携带此valueCtx。

```go
func (srv *Server) Serve(l net.Listener) error {

    ...

    var tempDelay time.Duration     // how long to sleep on accept failure
    baseCtx := context.Background() // base is always background, per Issue 16220
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        rw, e := l.Accept()

        ...

        tempDelay = 0
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx)
    }
}

复制代码
```

2.建立连接之后会基于传入的context创建一个valueCtx用于存储本地地址信息，之后在此基础上又创建了一个cancelCtx，然后开始从当前连接中读取网络请求，每当读取到一个请求则会将该cancelCtx传入，用以传递取消信号。一旦连接断开，即可发送取消信号，取消所有进行中的网络请求。

```go
func (c *conn) serve(ctx context.Context) {
    c.remoteAddr = c.rwc.RemoteAddr().String()
    ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
    ...

    ctx, cancelCtx := context.WithCancel(ctx)
    c.cancelCtx = cancelCtx
    defer cancelCtx()

    ...

    for {
        w, err := c.readRequest(ctx)

        ...

        serverHandler{c.server}.ServeHTTP(w, w.req)

        ...
    }
}

复制代码
```

3.读取到请求之后，会再次基于传入的context创建新的cancelCtx,并设置到当前请求对象req上，同时生成的response对象中cancelCtx保存了当前context取消方法。

```go
func (c *conn) readRequest(ctx context.Context) (w *response, err error) {

    ...

    req, err := readRequest(c.bufr, keepHostHeader)

    ...

    ctx, cancelCtx := context.WithCancel(ctx)
    req.ctx = ctx

    ...

    w = &response{
        conn:          c,
        cancelCtx:     cancelCtx,
        req:           req,
        reqBody:       req.Body,
        handlerHeader: make(Header),
        contentLength: -1,
        closeNotifyCh: make(chan bool, 1),

        // We populate these ahead of time so we're not
        // reading from req.Header after their Handler starts
        // and maybe mutates it (Issue 14940)
        wants10KeepAlive: req.wantsHttp10KeepAlive(),
        wantsClose:       req.wantsClose(),
    }

    ...
    return w, nil
}

复制代码
```

这样处理的目的主要有以下几点：

- 一旦请求超时，即可中断当前请求；
- 在处理构建response过程中如果发生错误，可直接调用response对象的cancelCtx方法结束当前请求；
- 在处理构建response完成之后，调用response对象的cancelCtx方法结束当前请求。

在整个server处理流程中，使用了一条context链贯穿Server、Connection、Request，不仅将上游的信息共享给下游任务，同时实现了上游可发送取消信号取消所有下游任务，而下游任务自行取消不会影响上游任务。

## 关于Context传递数据

首先要清楚使用 context 包主要是解决 goroutine 的通知退出，传递数据是其一个额外功能。 可以使用它传递一些元信息 ，总之使用 context 传递的信息不能影响正常的业务流程，程序不要期待在 context 中传递某些必需的参数等，没有这些参数，程序也应该能正常工作。

**context应该传递什么数据**

1.日志信息。 2.调试信息。 3.不影响业务主逻辑的可选数据。

## 小结

1.context主要用于父子任务之间的同步取消信号，本质上是一种协程调度的方式。另外在使用context时有两点值得注意：上游任务仅仅使用context通知下游任务不再需要，但不会直接干涉和中断下游任务的执行，由下游任务自行决定后续的处理操作，也就是说context的取消操作是无侵入的；context是线程安全的，因为context本身是不可变的（immutable），因此可以放心地在多个协程中传递使用。

2.context 包提供的核心的功能是多 goroutine 之间的退出通知机制，传递数据只是个辅助功能，应谨慎使用 context 传递数据。
