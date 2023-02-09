
# http 请求超时入门

https://studygolang.com/articles/14405



在分布式系统中，超时是基本可靠性概念之一。他可以缓和分布式系统中不可避免出现的失败所带来的影响。


## 问题： 如何条件性地模拟504 http.statusGatewayTimeout 响应


只创建客户端的超时，创建了一个标准的带有超时的 HTTP Client。
```
client:=http.Client{Timeout:5*time.Second}
```

当想要创建一个 client 用以发送 http 请求的时候，上面的代码看起来非常简单和直观。但是它下面却隐藏了大量低层次的细节，包括客户端超时，服务端超时和负载均衡超时。


## 客户端超时

客户端的 http 请求超时有多种形式，具体取决于在整个请求流程中超时帧的位置。整个请求响应流程由 `Dialer`，`TLS Handshake`，`Request Header`，`Request Body`，`Response Header` 和 `Response Body` 构成。根据请求响应流程中上述不同的部件，Go 提供了如下方式来创建带有超时的请求。

DialTLS指定可选的拨号功能，用于为非代理的HTTPS请求创建TLS连接
dial

TLS handshake 这个是请求验证

## http.client

`http.client` 超时是超时的高层实现，包含了从 `Dial` 到 `Response Body` 的整个请求流程。`http.client` 的实现提供了一个结构体类型可以接受一个额外的 `time.Duration` 类型的 `Timeout` 属性。这个参数定义了从请求开始到响应消息体被完全接收的时间限制。

## context

Go 语言的 `context` 包提供了一些有用的工具通过使用 `WithTimeout`，`WithDeadline` 和 `WithCancel` 方法来处理超时，Deadline 和可取消的请求。有了 `WithTimeout`，你就可以通过 `req.WithContext` 方法给 `http.Request` 添加一个超时时间了。

```
var url string  
ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)  
  
// 最后需要将这个context进行取消？  
defer cancel()  
  
request, err := http.NewRequest("GET", url, nil)  
if err != nil {  
  
}  
_, err = http.DefaultClient.Do(request.WithContext(ctx))
```


## http.Transport

你也可以通过使用带有 `DialContext` 的自定义 `http.Transport` 来创建 `http.client` 这种低层次的实现来指定超时时间。

```
transport := &http.Transport{  
   DialContext: (&net.Dialer{  
      Timeout: 1 * time.Minute,  
   }).DialContext,  
}  
  
_ = http.Client{Transport: transport}

```


## 服务端超时

使用 `context.WithTimeout()` 的问题是它仍然只是模拟的请求的客户端。万一请求的头部或者消息体超出了预定义的超时时间，请求会在客户端直接失败而不会从服务端返回 `504 http.StatusGatewayTimeout` 状态码。




创建一个每次都超时的 httptest 服务代码如下：

```
httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request){  
   w.WriteHeader(http.StatusGatewayTimeout)  
}))
```


但我只希望服务端根据客户端设置的值超时。为了让服务端根据客户端超时时间返回 504 状态码，你可以使用 `http.TimeoutHandler()` 函数来包装 handler 使请求在服务端失败。如下为符合场景需求的可运行的测试代码。







```
```



## 设置请求超时时间

### 1 client.Timeout

```
// 设置1s超时 
cli := http.Client{Timeout: time.Second}

// 当想要创建一个 client 用以发送 http 请求的时候，上面的代码看起来非常简单和直观。但是它下面却隐藏了大量低层次的细节，包括客户端超时，服务端超时和负载均衡超时。
```

### 2 req.WithContext

```
// 设置1s超时 
req := http.NewRequest(....) 
ctx, _ := context.WithTimeout(time.Second) 
req.WithContext(ctx)
```

先说说1，根据 Timeout 设置一个定时器 timer, 然后起一个goroutine等待 timer结束，如果等到就关闭 req.cancel。参考这里。关闭req.cancel会导致当前链接关闭从而结束本次请求。参考这里和这里

再来说说2，这里并没有像1一样起一个timer，而是根据req.Context是否结束来判断是否超时。参考这里，如果超时，同1一样，关闭当前链接。

这里总结一下，

1 和 2 效果一样，都是通过关闭当前链接结束本次请求。
1 和 2 一样，超时时间都包括 链接建立，请求发送，读取返回。如果没有及时读取resp.Body，都会引起超时错误。
