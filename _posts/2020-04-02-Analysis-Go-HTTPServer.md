---
title: 浅析 Go HTTPServer
tags: Go 协议 源码分析
key: Analysis-Go-HTTPServer
modify_date: 2020-04-07 16:22:53 
---

# 浅析 Go HTTPServer

`net/http`标准库中提供了开箱即用的HTTPServer，并且支持HTTPS协议下`HTTP2.0`的实现，用户只需几行代码即可启动一个简单的且并发性能优秀的HTTPServer：

```go
func handler(w http.ResponseWriter, r *http.Request)  {
    fmt.Fprint(w, "hello world!")
}

func main() {
    http.HandleFunc("/", handler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

本文将会对[Go1.14标准库中的HTTPServer实现](https://github.com/golang/go/blob/release-branch.go1.14/src/net/http/server.go)做一个简单的探究。读者阅读前需要对HTTP1.x协议有一个比较清晰的认识，可以从这篇文章[《阮一峰--HTTP 协议入门》](https://www.ruanyifeng.com/blog/2016/08/http.html)学习。

<!--more-->

## 数据结构
在实际分析HTTPServer的实现之前，需要先对HTTPServer中使用的数据结构有一个简单了解。

### 连接相关
```go
type conn struct {
    // server is the server on which the connection arrived.
    server *Server

    // cancelCtx cancels the connection-level context.
    cancelCtx context.CancelFunc

    // rwc is the underlying network connection.
    rwc net.Conn

    remoteAddr string

    // r is bufr's read source. It's a wrapper around rwc that provides
    // io.LimitedReader-style limiting (while reading request headers)
    // and functionality to support CloseNotifier. See *connReader docs.
    r *connReader

    // bufr reads from r.
    bufr *bufio.Reader

    // bufw writes to checkConnErrorWriter{c}, which populates werr on error.
    bufw *bufio.Writer

    curReq atomic.Value // of *response (which has a Request in it)
    curState struct{ atomic uint64 } // packed (unixtime<<8|uint8(ConnState))
}
```

首先是`conn`结构体，用于包装一个可读写的连接接口`rwc`。这个接口与底层的fd相连，通过[`net.conn`](https://github.com/golang/go/blob/release-branch.go1.14/src/net/net.go#L171-L201)中的Read、Write方法来对底层fd直接读写。连接fd会与对应TCP缓冲区相连，用户可以对TCP缓冲区进行读写，内核会对TCP的缓冲区进行自动管理与报文收发。

在读写TCP字节流时，如果每次都直接对fd进行系统调用的读写，那么会由于频繁的系统调用而降低读写效率，因此Go使用`bufr *bufio.Reader`与`bufw *bufio.Writer`添加用户缓存来对TCP缓冲区进行读写以及额外的校验。

<div style="text-align: center">
<img title="Bufio Wrapper" src="/assets/Analysis-Go-HTTPServer/Analysis-Go-HTTPServer-2020-04-03T14-13-16.png">
</div>

### 请求与响应
接下来的结构体对应HTTP协议中的Request与Response：
```go
type Header map[string][]string

type Request struct {
    Method string
    URL *url.URL
    Header Header
    Body io.ReadCloser
    GetBody func() (io.ReadCloser, error)
    ContentLength int64
    ...
}

type response struct {
    w  *bufio.Writer // buffers output in chunks to chunkWriter
    cw chunkWriter
    handlerHeader Header // handlerHeader is the Header that Handlers get access to
    trailers []string  // trailers are the headers to be sent after the handler finishes writing the body.
    ...
}
```

从文档中提到的[Response Write流程](https://github.com/golang/go/blob/release-branch.go1.14/src/net/http/server.go#L1512-L1535)可知`response`中添加了一层 2KB 的`bufio.Writer`用于缓冲用户预先输出的HTTP Content并自动添加一些头部信息，最后才会通过`chunkWriter`来对 4KB 的`conn.bufw`进行实际的写入。

<div style="text-align: center">
<img title="Response Write" src="/assets/Analysis-Go-HTTPServer/Analysis-Go-HTTPServer-2020-04-03T14-34-07.png">
</div>

每一层Writer会在调用`Write()`方法写入时，如果缓冲区已满或者手动调用`flush()`、`close()`方法后调用下一层Writer的`Write()`方法，由此形成一条**嵌套的可缓冲写入路径**。但是过长的写入路径在一定程度上会影响性能，因此文档中也提到未来将会对一些情况进行优化以减少写入路径长度。

在具体实现一节，我们将会具体介绍这样的嵌套写入路径是如何实现的。

### 路由相关
在处理连接时，需要根据HTTP Request中的URL请求路径来路由分发到对应的Handler。

路由通过结构体`ServeMux`、`muxEntry`实现

```go
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry   // m[pattern] = muxEntry
    es    []muxEntry // slice of entries sorted from longest to shortest.
    hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
    h       Handler
    pattern string
}

// DefaultServeMux is the default ServeMux used by Serve.
var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux
```

内置的`ServeMux`路由比较简单，主要结构为以pattern为key的map以及有序的`[]muxEntry`。默认会生成全局的路由变量`DefaultServeMux`，用户可以通过`http.Handle`方法来将自定义的路由处理器注册到`DefaultServeMux`中。

## 实现
下面，我们将会从一个HTTP请求的处理流程来看看HTTPServer是如何实现的。

### 路由注册
在服务启动之前，通过`http.Handle`可将自定义的路由处理器进行注册。由于路由中有添加读写锁，自然也可以在服务启动时对路由进行修改。

```go
func Handle(pattern string, handler Handler) { 
    DefaultServeMux.Handle(pattern, handler) 
}

func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()

    // Check Args ...
    
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
```
路由注册的流程比较简单，主要就是将`muxEntry`添加到map中，并插入到有序的`[]muxEntry`

### 服务启动
通过`http.ListenAndServe`可以启动HTTP服务器，对应地，`http.ListenAndServeTLS`可以启动HTTPS服务器。HTTPS服务器中主要就是添加了TLS层的加密解密工作，其他方法与HTTP服务器相同，因此下文会主要分析HTTP服务器的实现。

```go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```

`http.ListenAndServe`会根据传入的信息新生成一个`Server`结构，并调用类方法进行实际的调用。在需要自定义配置时，我们也可以通过自定义`Server`结构体来调用`Server.ListenAndServe()`。

```go
func (srv *Server) ListenAndServe() error {
    ...
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    return srv.Serve(ln)
}

func Listen(network, address string) (Listener, error) {
    var lc ListenConfig
    return lc.Listen(context.Background(), network, address)
}
```

`Server.ListenAndServe()`会根据监听地址和监听协议生成`Listener`，这个`Listener`将会用于监听连接，并返回连接socket。需要注意的是Listener中传入了`context.Background`后台上下文，后续的流程可以通过这个后台上下文来对`Listener`进行控制。

### 监听连接
在生成`Listener`之后，就可以正式使用`Server.Serve()`来运行服务。`Server.Serve()`分为两部分，一部分是监听请求前的准备工作，另一部分则是一个用于监听连接的无限循环。

```go
func (srv *Server) Serve(l net.Listener) error {
    origListener := l
    l = &onceCloseListener{Listener: l}
    defer l.Close()
    
    if !srv.trackListener(&l, true) {
        return ErrServerClosed
    }
    defer srv.trackListener(&l, false)

    baseCtx := context.Background()
    if srv.BaseContext != nil {     // Custom baseContext
        baseCtx = srv.BaseContext(origListener)
        if baseCtx == nil {
            panic("BaseContext returned a nil context")
        }
    }
    var tempDelay time.Duration // how long to sleep on accept failure

    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    ...
}

```

首先，会通过`onceCloseListener`包装类借助标准库中的`sync.Once`将原始的`Listener.Close()`方法进行包装，保证`Listener`只会被关闭一次。随后通过普通后台上下文`BaseCtx`来生成用于值传递的子上下文`ctx`，子上下文会传递到每个连接中使用。 

> :memo: `sync.Once`借助`sync.Mutex`通过双重检测的方法来保证某个方法只被调用一次。

```go
    ...
    for {
        rw, err := l.Accept()
        if err != nil {
            select {
            case <-srv.getDoneChan():
                return ErrServerClosed
            default:
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
        if cc := srv.ConnContext; cc != nil {   // Custom connContext
            connCtx = cc(connCtx, rw)
            if connCtx == nil {
                panic("ConnContext returned nil")
            }
        }
        tempDelay = 0
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(connCtx)
    }
```

准备工作完成之后，进入到常见的Server监听循环中。一般的监听循环是每一个连接会使用新的子进程或子线程来处理，Go中则是借助Goroutine以及运行时后台的`NetPoller`来并发处理连接。循环中会根据`Accept`的情况来进行不同的处理：

- 如果`Accept`返回了err，err会有两种情况：
    - `Server.doneChan`已被关闭，说明Server已被关闭，直接返回错误，不再监听请求，用于优雅关闭。
    - `Accept`中出现了临时错误，那么会通过睡眠一段时间后再进行重试。
- `Accept`返回了正常的连接，启动新的Goroutine进行连接处理

### 连接处理
连接处理时首先会使用连接地址作为值来注册值上下文，并添加panic处理。

> :information_source: Go标准库中的HTTPServer提供了[Hijacker](https://github.com/golang/go/blob/release-branch.go1.14/src/net/http/server.go#L173-L201)接口，用户可通过此接口在部分场景下自行管理连接。本文不会对如何让用户管理连接做详细的探究，因此下文中涉及到`hijack`部分的代码会被移除


```go
func (c *conn) serve(ctx context.Context) {
    c.remoteAddr = c.rwc.RemoteAddr().String()
    ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
    defer func() {
        if err := recover(); err != nil && err != ErrAbortHandler {
            const size = 64 << 10
            buf := make([]byte, size)
            buf = buf[:runtime.Stack(buf, false)]
            c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
        }
        if !c.hijacked() {
            c.close()
            c.setState(c.rwc, StateClosed)
        }
    }()
    
    // Handle TLS Connection
    ...
    // Handle HTTP 1.x
    ctx, cancelCtx := context.WithCancel(ctx)
    c.cancelCtx = cancelCtx
    defer cancelCtx()

    c.r = &connReader{conn: c}
    c.bufr = newBufioReader(c.r)
    c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)
    ...
    // Handle Request
}
```

在处理请求之前，预先使用原先的连接值上下文来派生取消上下文以便在解析请求时及时取消，同时生成用于解析和发送的`bufioReader`以及`bufioWriter`。

```go
var (
    bufioReaderPool   sync.Pool
    bufioWriter2kPool sync.Pool
    bufioWriter4kPool sync.Pool
)

// func newBufioWriterSize 与此函数逻辑类似
func newBufioReader(r io.Reader) *bufio.Reader {
    if v := bufioReaderPool.Get(); v != nil {
        br := v.(*bufio.Reader)
        br.Reset(r)
        return br
    }

    return bufio.NewReader(r)
}
```
`bufioReader`和`bufioWriter`会借助`sync.Pool`来实现临时对象缓存，以减少高并发时的GC与对象分配压力。

> :memo: `sync.Pool`可以提供并发安全的临时对象池，会定时清除池中的对象。

#### 解析请求
预处理工作结束后，正式进入一个大的无限循环，用于解析请求与发送响应。

首先是请求的解析，请求解析通过`conn.readRequest`来完成。如果在解析请求时发生错误，那么会直接返回错误响应并结束连接。
```go
    for {
        w, err := c.readRequest(ctx)
        if c.r.remain != c.server.initialReadLimitSize() {
            // If we read any bytes off the wire, we're active.
            c.setState(c.rwc, StateActive)
        }
        if err != nil {
            const errorHeaders = "\r\nContent-Type: text/plain; charset=utf-8\r\nConnection: close\r\n\r\n"

            switch {
            // err case ...
            default:
                publicErr := "400 Bad Request"
                ...
                return
            }
        }
        // Expect 100 Continue support ...
        // Register Background Read ...
        // Handle Response ...
    }
```
`conn.readRequest`首先是设置超时时间，并通过`connReader.setReadLimit()`设置可读大小为1MB + 4KB
```go
func (c *conn) readRequest(ctx context.Context) (w *response, err error) {
    var (
        wholeReqDeadline time.Time // or zero if none
        hdrDeadline      time.Time // or zero if none
    )
    t0 := time.Now()
    if d := c.server.readHeaderTimeout(); d != 0 {
        hdrDeadline = t0.Add(d)
    }
    if d := c.server.ReadTimeout; d != 0 {
        wholeReqDeadline = t0.Add(d)
    }
    c.rwc.SetReadDeadline(hdrDeadline)
    if d := c.server.WriteTimeout; d != 0 {
        defer func() {
            c.rwc.SetWriteDeadline(time.Now().Add(d))
        }()
    }

    c.r.setReadLimit(c.server.initialReadLimitSize())   // 1MB + 4KB
    ...
}
```
随后通过`readRequest()`进行正式的读取工作，函数返回后，HTTP Header解析完成。如果读取请求过程中请求头过大，则会返回`errTooLarge`。如果HTTP Header正常，那么将会构造`response`并返回。
```go
    req, err := readRequest(c.bufr, keepHostHeader)
    if err != nil {
        if c.r.hitReadLimit() {
            return nil, errTooLarge
        }
        return nil, err
    }

    c.r.setInfiniteReadLimit() 
    
    // Valid HTTP Header ...

    w = &response{
        conn:          c,
        cancelCtx:     cancelCtx,
        req:           req,
        reqBody:       req.Body,
        handlerHeader: make(Header),
        contentLength: -1,
        closeNotifyCh: make(chan bool, 1),
        wants10KeepAlive: req.wantsHttp10KeepAlive(),
        wantsClose:       req.wantsClose(),
    }
    
    w.cw.res = w
    w.w = newBufioWriterSize(&w.cw, bufferBeforeChunkingSize)
    return w, nil
```

##### 读取Header
HTTP包以字节流的形式交付至传输层TCP，TCP会提供传输的可靠性与字节流到达的有序性。如下图所示

<div style="text-align: center">
<img title="TCP Buffer" src="/assets/Analysis-Go-HTTPServer/Analysis-Go-HTTPServer-2020-04-03T23-14-13.png">
</div>

因此当我们从TCP缓冲区中接收HTTP包时，只需要进行顺序读取即可，但需要特别注意HTTP包的边界防止影响到下一个HTTP包的解析。一般来说，HTTP1.x中根据HTTP Header中的标识来区分包的边界 [^1]，即：
1. Header中明确标识`Content-Length`，指明HTTP Body的整体长度。
2. Header中明确标识`Transfer-Encoding: trunked`，表明当前为分块传输编码。
3. 

[^1]: [RFC7230 - Message Body](https://tools.ietf.org/html/rfc7230#section-3.3)

了解了HTTP的解析关键后再来看看`readRequest()`的实现。

`readRequest()`首先会通过`textproto.Reader`（同样使用了临时对象池来减少开销）来对`conn.bufr`进行包装以支持一些快捷的文本读取操作，在这之后，则是对HTTP Header的第一行文本进行解析与校验。
```go
func readRequest(b *bufio.Reader, deleteHostHeader bool) (req *Request, err error) {
    tp := newTextprotoReader(b)
    req = new(Request)

    // First line: GET /index.html HTTP/1.0
    var s string
    if s, err = tp.ReadLine(); err != nil {
        return nil, err
    }
    defer func() {
        putTextprotoReader(tp)
        if err == io.EOF {
            err = io.ErrUnexpectedEOF
        }
    }()

    var ok bool
    req.Method, req.RequestURI, req.Proto, ok = parseRequestLine(s)
    
    // Parse And Valid ...
}
```
随后，通过`tp.ReadMIMEHeader()`逐行读取Header中的键值对，并转换为`Header`类型。解析之后，还会通过Header来判断当前连接是否需要复用。

```go
    // Subsequent lines: Key: value.
    mimeHeader, err := tp.ReadMIMEHeader()
    if err != nil {
        return nil, err
    }
    req.Header = Header(mimeHeader)
    
    req.Close = shouldClose(req.ProtoMajor, req.ProtoMinor, req.Header, false)
```

读取完Header之后，则是通过[`readTransfer()`](https://github.com/golang/go/blob/release-branch.go1.14/src/net/http/transfer.go#L470)将Body的读取接口挂载等待用户Handler的读取。
```go
    err = readTransfer(req, b)
    if err != nil {
        return nil, err
    }
    
    return req, nil
```

`readTranser()`的主要工作是提供HTTP Body的读取接口，因此会在Request和Response中进行复用。读取HTTP Header中的`Content-Length`字段以及`Transfer-Encoding`字段来判断之后应提供哪类读取接口。

```go
func readTransfer(msg interface{}, r *bufio.Reader) (err error) {
    t := &transferReader{RequestMethod: "GET"}

    // Unify input
    isResponse := false
    switch rr := msg.(type) {
    case *Response:
        ...
    case *Request:
        t.Header = rr.Header
        t.RequestMethod = rr.Method
        t.ProtoMajor = rr.ProtoMajor
        t.ProtoMinor = rr.ProtoMinor
        t.StatusCode = 200
        t.Close = rr.Close
    default:
        panic("unexpected type")
    }

    // Transfer encoding, content length
    err = t.fixTransferEncoding()
    
    realLength, err := fixLength(isResponse, t.StatusCode, t.RequestMethod, t.Header, t.TransferEncoding)

```

判断的关键在于下方这个switch结构：
1. 分块传输下使用`internal.NewChunkedReader`，为用户提供分块读取接口。
2. 指明长度的情况下使用`io.LimitReader`，提供长度为`realLength`的读取接口，防止用户读取到下一个HTTP包的内容。
3. 如果没有长度则是提供`Nobody`读取接口，用户尝试读取时会直接返回EOF。

```go
    switch {
    case chunked(t.TransferEncoding):
        if noResponseBodyExpected(t.RequestMethod) || !bodyAllowedForStatus(t.StatusCode) {
            t.Body = NoBody
        } else {
            t.Body = &body{src: internal.NewChunkedReader(r), hdr: msg, r: r, closing: t.Close}
        }
    case realLength == 0:
        t.Body = NoBody
    case realLength > 0:
        t.Body = &body{src: io.LimitReader(r, realLength), closing: t.Close}
    default:
        // realLength < 0, i.e. "Content-Length" not mentioned in header
        if t.Close {
            // Close semantics (i.e. HTTP/1.0)
            t.Body = &body{src: r, closing: t.Close}
        } else {
            // Persistent connection (i.e. HTTP/1.1)
            t.Body = NoBody
        }
    }
```

Body读取接口挂载完成之后，此时相当于HTTP包已经解析完毕，返回解析后的`*Request`结构体，之后则是将控制权交由用户注册的Handler。下一节，我们会看看用户Handler是如何读取Request Body的。

##### 读取Body
Body读取时，提供给用户Handler为实现了`ReadCloser`接口的[`body`](https://github.com/golang/go/blob/release-branch.go1.14/src/net/http/transfer.go#L813)结构体，其中`body.src`挂载着辅助的读取接口，主要对应`internal.NewChunkedReader`、`io.LimitReader`以及`Nobody`，这些辅助接口的实现相对简单，下面会重点看看`body`是如何进行读取的。

首先是`body.Read()`，读取时会从`body.src`中挂载的读取接口进行读取，并判断Body的长度是否正常以及EOF的位置是否正常。

```go
func (b *body) Read(p []byte) (n int, err error) {
    b.mu.Lock()
    defer b.mu.Unlock()
    if b.closed {
        return 0, ErrBodyReadAfterClose
    }
    return b.readLocked(p)
}

func (b *body) readLocked(p []byte) (n int, err error) {
    if b.sawEOF {
        return 0, io.EOF
    }
    n, err = b.src.Read(p)

    if err == io.EOF {
        // check 
        b.sawEOF = true
        // Chunked case. Read the trailer.
        if b.hdr != nil {
            if e := b.readTrailer(); e != nil {
                err = e
                // Something went wrong in the trailer, disable we must not allow any
                // further reads of any kind to succeed from body
                b.sawEOF = false
                b.closed = true
            }
            b.hdr = nil
        } else {
            // If the server declared the Content-Length, our body is a LimitedReader
            // and we need to check whether this EOF arrived early.
            if lr, ok := b.src.(*io.LimitedReader); ok && lr.N > 0 {
                err = io.ErrUnexpectedEOF
            }
        }
    }
    
    // more check EOF ...
}
```

之后是`body.Close()`，Server会在响应包发送之前调用一次（见[发送响应](#%e5%8f%91%e9%80%81%e5%93%8d%e5%ba%94)一节，关键是将未消费的Body消费掉防止影响下一个HTTP包。

```go
func (b *body) Close() error {
    b.mu.Lock()
    defer b.mu.Unlock()
    if b.closed {
        return nil
    }
    var err error
    switch {
    case b.sawEOF:
    case b.hdr == nil && b.closing:
    case b.doEarlyClose:
        if lr, ok := b.src.(*io.LimitedReader); ok && lr.N > maxPostHandlerReadBytes {
            b.earlyClose = true
        } else {
            var n int64
            // Consume the body, or, which will also lead to us reading
            // the trailer headers after the body, if present.
            n, err = io.CopyN(ioutil.Discard, bodyLocked{b}, maxPostHandlerReadBytes)
            if err == io.EOF {
                err = nil
            }
            if n == maxPostHandlerReadBytes {
                b.earlyClose = true
            }
        }
    default:
        // Fully consume the body, which will also lead to us reading
        // the trailer headers after the body, if present.
        _, err = io.Copy(ioutil.Discard, bodyLocked{b})
    }
    b.closed = true
    return err
}
```

##### 小结
Server在解析HTTP包时，顺序读取解析HTTP Header，通过Header中的长度标识为用户提供Body的读取接口。在读取过程中，如果出现错误，则会返回错误响应，根据情况判断是否关闭连接。

#### 处理响应
##### 路由处理
在请求接收完成之后，会调用用户预先注册的路由处理器生成响应内容
```go
    for {
        // Handle Request
        ...
        // Handle Response

        // User Handler
        serverHandler{c.server}.ServeHTTP(w, w.req)
        
        // Send Response
        ...
    }
```
`serverHandler`会获取预配置的Handler，并调用`handler.ServeHTTP(rw, req)`。

```go
type serverHandler struct {
    srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}
```

如果用户没有自定义路由，那么一般的请求会经过`DefaultServeMux`中的`ServeMux.ServeHTTP`进行路由分发。

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        // Return Bad Request 
    }

    h, _ := mux.Handler(r)  // Get Handler h
    h.ServeHTTP(w, r)   // User Handle
}

func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {

    if r.Method == "CONNECT" {
        // return 
    }

    host := stripHostPort(r.Host)
    path := cleanPath(r.URL.Path)

    // If the given path is /tree and its handler is not registered,
    // redirect for /tree/.
    if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
        return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
    }

    if path != r.URL.Path {
        _, pattern = mux.handler(host, path)
        url := *r.URL
        url.Path = path
        return RedirectHandler(url.String(), StatusMovedPermanently), pattern
    }

    return mux.handler(host, r.URL.Path)
}
```

`ServeMux.ServeHTTP`首先会对请求的URL PATH进行规整化处理，随后判断是否存在需要重定向的情况：
- 需要重定向的情况下会返回`RedirectHandler`
  - 缺少末尾斜杆的情况 /tree -> /tree/
  - 规整化后的路径与原请求路径不同，返回规整化后的路径重定向响应
- 不需重定向的情况下则会通过`ServeMux.handler`查找用户注册的路由处理器

```go
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    mux.mu.RLock()
    defer mux.mu.RUnlock()

    // Host-specific pattern takes precedence over generic ones
    if mux.hosts {
        h, pattern = mux.match(host + path)
    }
    if h == nil {
        h, pattern = mux.match(path)
    }
    if h == nil {
        h, pattern = NotFoundHandler(), ""
    }
    return
}
```

`ServeMux.handler`查找过程中会对`host + path`以及`path`的模式进行匹配

```go
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
    // Check for exact match first.
    v, ok := mux.m[path]
    if ok {
        return v.h, v.pattern
    }

    // Check for longest valid match.  mux.es contains all patterns
    // that end in / sorted from longest to shortest.
    for _, e := range mux.es {
        if strings.HasPrefix(path, e.pattern) {
            return e.h, e.pattern
        }
    }
    return nil, ""
}
```

匹配首先是通过map进行完整匹配，如果无法完整匹配，则尝试从最长的路径进行前缀匹配。找到了对应的路由处理器之后就会执行用户函数，写入HTTP Content。

假如用户预先注册了如下Handler，那么用户可以通过`http.ResponseWriter`接口写入Header和Content，并且可以通过`http.Request`来获取已解析的请求内容。

```go
// User Handler
func handler(w http.ResponseWriter, r *http.Request)  {
    fmt.Fprint(w, "hello world!")
}
```

##### 发送响应
当用户的路由处理器将内容写入后，就到了发送阶段。

发送阶段主要分为响应发送与发送结束后判断是否需要复用连接。判断是否需要复用连接的处理逻辑相对简单，重点看看响应发送的处理。

```go
    for {
        // Handle Request 
        ...
        // Handle Response
        serverHandler{c.server}.ServeHTTP(w, w.req)
        w.finishRequest()
        // Reuse Connection ?
        ...
    }
```

响应发送由`response.finishRequest()`完成，主要由下列步骤完成
1. 如果Handler处理过程中没有写入Header Code，那么会写入默认的响应码200
2. 调用`w.w.Flush()`，将用户写入的内容刷入下一层Writer，并将此`bufio.Writer`放回全局的临时对象池
3. 调用`w.cw.close()`写入HTTP Content的结束符
4. 调用`w.conn.bufw.Flush()`保证`conn.bufw`将所有内容写入内核的TCP缓冲区，至此发送流程已结束。
5. 调用`w.reqBody.Close()`关闭请求中的读缓冲，使得`conn.bufr`可放入全局临时对象池以复用
6. 如果请求中附带上传文件，那么会移除这些临时上传文件。

```go
func (w *response) finishRequest() {
    w.handlerDone.setTrue()

    if !w.wroteHeader {
        w.WriteHeader(StatusOK) // 200 OK
    }

    w.w.Flush()
    putBufioWriter(w.w)
    w.cw.close()
    w.conn.bufw.Flush()

    // Close the body (regardless of w.closeAfterReply) so we can
    // re-use its bufio.Reader later safely.
    w.reqBody.Close()

    if w.req.MultipartForm != nil {
        w.req.MultipartForm.RemoveAll()
    }
}
```

##### Writer嵌套写入
在数据结构一节中介绍了Writer的可缓冲嵌套写入，本节将会分析整体写入流程的实现。

首先是User Handler中对`http.ResponseWriter`接口的写入，写入的过程中，`fmt.Fprint`实际会调用`response.Write()`对响应内容进行写入

```go
// User Handler
func handler(w http.ResponseWriter, r *http.Request)  {
    fmt.Fprint(w, "hello world!")
}

func (w *response) Write(data []byte) (n int, err error) {
    return w.write(len(data), data, "")
}

// either dataB or dataS is non-zero.
func (w *response) write(lenData int, dataB []byte, dataS string) (n int, err error) {
    if !w.wroteHeader {
        w.WriteHeader(StatusOK)
    }
    if lenData == 0 {
        return 0, nil
    }
    if !w.bodyAllowed() {
        return 0, ErrBodyNotAllowed
    }

    w.written += int64(lenData) // ignoring errors, for errorKludge
    if w.contentLength != -1 && w.written > w.contentLength {
        return 0, ErrContentLength
    }
    if dataB != nil {
        return w.w.Write(dataB)
    } else {
        return w.w.WriteString(dataS)
    }
}
```

开始写入HTTP Content时，如果用户没有预先调用`w.WriteHeader()`写入响应状态码，那么默认写入200响应成功。之后会计算用户已写入的字节数，并调用`w.w`写入用户的`bufio.Writer`写缓冲区中。

`w.w bufio.Writer`写满2KB后或者手动`Flush()`、`Close()`则会调用`chunkWriter.Write()`进行HTTP内容写入。如果用户在HTTP Header中设置了`Transfer Encoding: chunked`，那么`chunkWriter.Write()`会自动按照chunked分块传输的模式自动写入块大小以及内容块，否则按照正常模式直接写入HTTP Content。

```go
func (cw *chunkWriter) Write(p []byte) (n int, err error) {
    if !cw.wroteHeader {
        cw.writeHeader(p)
    }
    if cw.res.req.Method == "HEAD" {
        return len(p), nil
    }
    if cw.chunking {
        _, err = fmt.Fprintf(cw.res.conn.bufw, "%x\r\n", len(p))
        if err != nil {
            cw.res.conn.rwc.Close()
            return
        }
    }
    n, err = cw.res.conn.bufw.Write(p)
    if cw.chunking && err == nil {
        _, err = cw.res.conn.bufw.Write(crlf)
    }
    if err != nil {
        cw.res.conn.rwc.Close()
    }
    return
}
```

`chunkWriter.Write()`写入时，按照HTTP Header、HTTP Content的顺序手动调用`conn.bufw.Write()`将字节写入`conn`的写缓冲之中。

如果`conn.bufw bufio.Writer`写满4KB或者手动`Flush()`、`Close()`则会进入`checkConnErrorWriter.Write()`将字节流写入`net.conn`之中，`net.conn`会负责向底层fd做进一步的写入。写入的过程中如果出错，会通过连接的取消上下文取消连接的写入。

```go
func (w checkConnErrorWriter) Write(p []byte) (n int, err error) {
    n, err = w.c.rwc.Write(p)
    if err != nil && w.c.werr == nil {
        w.c.werr = err
        w.c.cancelCtx()
    }
    return
}
```

嵌套写入路径总体比较复杂，用下面一图总结一下：

<div style="text-align: center">
<img title="Response Write" src="/assets/Analysis-Go-HTTPServer/Analysis-Go-HTTPServer-2020-04-03T17-52-16.png">
</div>

##### 小结
Server处理响应时，会提供可缓冲的嵌套写入Writer，按照Header-Body的顺序以字节流的顺序写入其中。

### 优雅关闭
HTTPServer提供了优雅关闭服务的能力，用户可以通过`Server.Shutdown()`来关闭服务。同时用户也可以通过`Server.RegisterOnShutdown()`来注册关闭时需要执行的函数，通常可以借助这个HOOK来保存一些服务器状态。

我们会从`Server.Shutdown()`来看看HTTPServer是如何完成优雅关闭的。

```go
func (srv *Server) Shutdown(ctx context.Context) error {
    atomic.StoreInt32(&srv.inShutdown, 1)

    srv.mu.Lock()
    lnerr := srv.closeListenersLocked()
    srv.closeDoneChanLocked()
    for _, f := range srv.onShutdown {
        go f()
    }
    srv.mu.Unlock()

    ticker := time.NewTicker(shutdownPollInterval)
    defer ticker.Stop()
    for {
        if srv.closeIdleConns() {
            return lnerr
        }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
        }
    }
}
```

Shutdown的主要流程对应：
1. 首先关闭服务器中的所有的`Listener`并且关闭`Server.doneChan`，此时[`Server.Serve()`](https://github.com/golang/go/blob/release-branch.go1.14/src/net/http/server.go#L2901-L2907)中的监听循环会因为Listener关闭，从Accept返回的err而发现Server关闭，因此退出监听。
2. 并发启动Goroutine调用用户预先注册的退出执行函数。
3. 启动定时器，周期性地轮询以关闭空闲的连接，用户也可以通过传入的上下文来对退出轮询。当一次连接请求处理完毕之后，[`conn.serve()`](https://github.com/golang/go/blob/release-branch.go1.14/src/net/http/server.go#L1910-L1917)也会对服务是否退出进行检查


## 总结
标准库中的HTTPServer通过可缓冲的嵌套读写实现了层次清晰的响应发送方式，通过缓冲来避免频繁的底层读写，是一个比较好的io模式。同时Server实现时使用的并发模式也是十分值得学习的。













