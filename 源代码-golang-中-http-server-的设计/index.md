# 源代码 —— Golang 中 HTTP Server 的设计

## 写在前面
本文将通过 Golang 源码研究其 HTTP Server 的核心实现。

研究对象主要是 Golang SDK `net/http` 包下的 `server.go`。

## HTTP 应用工作模式

一个经典的基于 HTTP 协议的应用系统，其工作模式可以简单表示为以下流程：  

![C-S 架构应用流程](/images/C-S经典架构.jpg "C-S 架构应用流程")

具体来说，请求 `Request` 由客户端发出，服务端接收请求后，通过路由组件匹配到相应的处理器，处理器处理 `Request` 后构建 `Response` 返回给客户端。

## Golang 中 HTTP Server 的核心实现

> Golang HTTP Server 的核心实现也遵循上述基本流程

### 从一个 demo 入手

> 研究源码实现, 个人认为比较好的入手方式是写一个 demo，先跑起来~

```go
package main

import (
    "fmt"
    "net/http"
)

// 一种 http handler 的实现方式
func indexHandler(w http.ResponseWriter, r *http.Request) {
    fmt.printf(w, "Hello World!")
}

// 另一种实现方式
type otherIndexHandler struct {
    data string
}

func (ih *otherIndexHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.printf(w, ih.data)
}

func main() {
    // 第一种调用方式
    http.HandleFunc("/", indexHandler)
    http.ListenAndServe(":8080", nil)

    // 第二种调用方式
    http.Handle("/", &otherIndexHandler{data: "Hello World!"})
    http.ListenAndServe(":8081", nil);
}
```

是的，这就实现了一个极简的 HTTP Server！
<br></br>
简单分析一下这段代码：通过不同的方式定义 `handler`，并使用 Go 的标准库 `http` 注册 `handler` 到指定的路由，最后再启动对特定端口的监听。

- 第一种 `handler` 实现，`indexHandler` 是自定义的 `handler`，拥有 `http.ResponseWriter` 接口和 `http.Request` 结构体作为入参，函数实现非常简单，输出 "Hello World!"。

- 第二种 `handler` 实现，我定义了一个 `otherIndexHandler` 结构体，并实现了 Go `http` 标准库中 `Handler` 接口的 `ServeHTTP(ResponseWriter, *Request)` 函数，因此，`otherIndexHandler` 就实现了 `Handler` 接口。


### 顺藤摸瓜

> 带着问题看源码，往往更有针对性，且不容易迷失在源码的各种分支细节里。

<i class="fa-regular fa-circle-question"></i> **上述 demo 采用了两种方式启动一个 HTTP Server，有什么区别？**
<br></br>
- 第一种使用函数作为 `handler`，优点是简洁，适合简单逻辑；局限性是如果 `handler` 要维护状态或者访问其他字段，就必须使用全局变量或者闭包了。

- 第二种使用结构体实现 `handler`，可以很方便地为 `handler` 添加字段以存储状态或者其他数据、也可以定义很多方法，按照不同的路由或者请求类型进行处理，且有利于代码的组织和复用，局限性自然就是实现起来复杂些。


<br></br>
<i class="fa-regular fa-circle-question"></i> **从软件工程复用性的角度考虑，底层标准库不太可能为上层应用做不同的实现，因此猜想 `http.HandleFunc` 和 `http.Handle` 最终的实现可能是一样的，甚至是同一个函数？**
<br></br>
(1) `http.HandleFunc` 源码实现：
```go
var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux

func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
    // 调用 ServeMux 的 Handle 函数
	mux.Handle(pattern, HandlerFunc(handler))
}
```
<br></br>
(2) `http.Handle` 源码实现：
```go
func Handle(pattern string, handler Handler) { 
    // DefaultServeMux 是 ServeMux 类型的指针
    // 因此这也相当于调用 ServeMux 的 Handle 函数
    DefaultServeMux.Handle(pattern, handler)
}
```

可以看到，两种方式最终都是依赖了 `ServeMux` 的 `Handle` 函数，所以下一步重点是研究它。
<br></br>
需要注意的是，`mux.Handle(pattern, HandlerFunc(handler))` 这行，因为 `ServeMux.Handle` 函数需要一个 `Handler` 接口类型的入参，因此这里使用了 `HandlerFunc` 将入参的 `handler` 转换为了 `Handler` 接口类型，具体原理可以在以下源码中看到：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```  

这里其实运用了适配器设计模式，具体来说，其实是接口和类型的适配器模式，`HandlerFunc` 函数通过实现 `ServeHTTP` 方法，实现了 `Handler` 接口，再通过 `HandlerFunc(handler)` 将普通函数 `handler` 转为 `Handler` 这个接口类型。这是 Go 语言中接口的一种常见用法：通过定义函数类型并为该类型实现接口，我们可以将其他函数通过类型转换特性，转作接口类型的值来使用，从而实现了函数和接口类型之间的适配。
<br></br>
<i class="fa-regular fa-circle-question"></i> **`ServeMux` 是什么？**

`ServeMux` 是一个很重要的结构体 —— HTTP 请求的多路复用器（`Multiplexer`，也常被称为 Mux），作用是将传入的请求 URL 与预定义的模式列表做匹配，并将匹配成功的请求分发给相应的处理函数 (handler)  

```go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry
	hosts bool
}

type muxEntry struct {
	h       Handler
	pattern string
}
```

- `mu` 是一个读写器互斥锁，该锁可以由任意数量的读取器或单个写入器持有，这里非本文主线，不展开其实现细节，知道特性就行。  

- `m` 是一个 `map`，其元素是结构体类型 `muxEntry`，负责保存 `pattern` 和对应 `handler` 的映射关系。

- `es` 是一个切片，它存储了所有的路由规则并按照一定规则进行排序。
<br></br>

<i class="fa-regular fa-circle-question"></i> **`ServeMux.Handle` 函数的工作流程是什么？**

源码如下：

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	e := muxEntry{h: handler, pattern: pattern}
	mux.m[pattern] = e
	if pattern[len(pattern)-1] == '/' {
		mux.es = appendSorted(mux.es, e)
	}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}

func appendSorted(es []muxEntry, e muxEntry) []muxEntry {
	n := len(es)
    // 使用 sort.Search 函数查找 muxEntry 应该插入的位置。
    // sort.Search 函数接受一个长度 n 和一个函数 f，并返回满足 f(i) 为真的最小的 i。
    // 在这里，f(i) 函数检查 es[i].pattern 的长度是否小于 e.pattern 的长度。
    // 这个搜索过程是二分搜索，所以时间复杂度为O(log n)。
	i := sort.Search(n, func(i int) bool {
		return len(es[i].pattern) < len(e.pattern)
	})

    // 如果应插入的位置 i 等于切片的长度 n，说明 e 应该被插入到切片的尾部
    // 那就直接使用 append 函数将 e 添加到切片的尾部并返回结果
	if i == n {
		return append(es, e)
	}

    // 如果应插入的位置 i 小于切片的长度 n，说明 e 应该被插入到切片的中间位置。
    // 首先，使用 append 函数将一个空的 muxEntry 添加到切片的尾部，让切片的长度增加 1。
    // 然后，使用 copy 函数将 i 位置及之后的元素向后移动一位，为插入 e 腾出空间。
    // 最后，将 e 复制到 i 位置。
	es = append(es, muxEntry{})
	copy(es[i+1:], es[i:])
	es[i] = e
	return es
}
```  

源码逻辑比较简单：

- 先加读写互斥锁，保证对 `ServeMux.mu` 和 `ServeMux.m` 的并发读写操作是线程安全的，避免数据不一致或者数据竞态等问题。

- 接下来是一些校验，这里有一个逻辑，`mux.m` 不支持对同一个模式（pattern）重复注册 `Handler`，这里有一个编码技巧：对 `exist` 的判断和对 `mux.m == nil` 的判断逻辑的先后顺序，从结果上来说，这两块逻辑可以调换，不会有问题，标准库这里实现的优化是：如果 `pattern` 已经存在于 `mux.m` 中，就没必要再创建一个新的 `map` 了。

- 接下来创建一个新的 `muxEntry`，将传入的 `handler` 和 `pattern` 映射写入；接着检查传入的模式字符串 `pattern` 是否以 `/` 字符结束，如果是，则通过 `appendSorted` 函数将这个新的 `muxEntry` 插入到 `mux.es` 切片的合适位置，以保持切片的排序顺序（appendSorted 的实现逻辑见上方注释）。从 `appendSorted` 函数的实现也可以确认，`es` 中 `pattern` 长度是逐渐递减的，即 `mux` 对 `pattern` 的匹配遵循最长匹配原则，而不是精准匹配，这样好处是比较灵活，尤其是对 `RESTful` 风格的 API 设计比较友好，可以更灵活地处理复杂的 URL 结构。

<br></br>
注册路由 Handler 的流程到此告一段落。
接下来看看启动服务的流程。
<br></br>

<i class="fa-regular fa-circle-question"></i> **`http.ListenAndServe` 函数做了些什么？**

依然从源码入手
```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```  
可以看到，核心就是根据传入的路由地址和处理器，生成 `Server` 结构体实例，然后调用 `server.ListenAndServe()` 函数。

`server.ListenAndServe()` 实现如下：
```go
func (srv *Server) ListenAndServe() error {
	if srv.shuttingDown() {
		return ErrServerClosed
	}
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(ln)
}
```

逻辑相对简单，核心是 `ln, err := net.Listen("tcp", addr)` 开启一个 TCP Listener，然后调用 `Server` 的 `Serve` 函数

```go
func (srv *Server) Serve(l net.Listener) error {
	if fn := testHookServerServe; fn != nil {
		fn(srv, l) // call hook with unwrapped listener
	}

    // 保存原始 Listener, 将传入的 Listener 使用 onceCloseListener 
    // 包装起来，确保在函数返回时只关闭一次监听器
	origListener := l
	l = &onceCloseListener{Listener: l}
	defer l.Close()

	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}

    // 将 Listener 添加到其内部跟踪列表中，以便可以优雅地关闭
	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
	defer srv.trackListener(&l, false)

    // 为连接创建基础上下文(context.Context), 实际上是一个空的 struct
	baseCtx := context.Background()
	if srv.BaseContext != nil {
		baseCtx = srv.BaseContext(origListener)
		if baseCtx == nil {
			panic("BaseContext returned a nil context")
		}
	}

	var tempDelay time.Duration // how long to sleep on accept failure

	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, err := l.Accept()
		if err != nil {
			if srv.shuttingDown() {
				return ErrServerClosed
			}
			if ne, ok := err.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return err
		}
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
		go c.serve(connCtx)
	}
}
```  

这部分实现较长，我提取几个关建点出来（其余流程见代码注释）：

- `setupHTTP2_Serve` 遵循 HTTP/2 协议。

- 为连接准备好上下文后，服务器会进入一个无限循环，等待新的连接。`tempDelay` 用于在网络 `Accept` 调用失败时实施指数退避。

- `go c.serve(connCtx)` 对于每一个新的连接上下文，会开启一个新的 `go` 协程来执行任务。这体现了 Go 对 HTTP Server 的并发支持。

- `Server.Serve` 封装了接受连接和处理连接的逻辑， `Serve` 方法仅处理监听和分派连接，并不关心具体的业务逻辑 —— 单一职责原则。

- 上下文管理：通过 `context.Context` 在协程间安全地传递数据和取消信号。
<br></br>

<i class="fa-regular fa-circle-question"></i> **`c.serve()` 函数如何处理客户端请求并返回响应？**

这部分源码较长，以下是主要流程

```go
func (c *conn) serve(ctx context.Context) {
    // 前置准备，对连接和上下文的一些处理
    
    for {
        w, err := c.readRequest(ctx)
        
        // ... 一些错误处理

        serverHandler{c.server}.ServeHTTP(w, w.req)
    }
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if !sh.srv.DisableGeneralOptionsHandler && req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}

	handler.ServeHTTP(rw, req)
}
```  

可以看到，在读取请求后，通过 `serverHandler{c.server}.ServeHTTP(w, w.req)` 进行了请求分发和处理，在我的 demo 中，实际是使用了 `DefaultServeMux` 中存储的 handler 映射来处理请求的。

<br></br>

至此，Golang server 启动一个 HTTP 服务并对客户端请求进行处理的主要流程就分析完毕！

<!-- 引入 fontawesome 支持 -->
<head> 
    <script defer src="https://use.fontawesome.com/releases/v5.0.13/js/all.js"></script> 
    <script defer src="https://use.fontawesome.com/releases/v5.0.13/js/v4-shims.js"></script> 
</head> 
<link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.0.13/css/all.css">
