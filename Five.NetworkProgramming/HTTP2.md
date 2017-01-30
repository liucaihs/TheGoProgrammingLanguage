### 5.2 HTTP编程
>HTTP(HyperText Transfer Protocol,超文本传输协议)是互联网上应用最为广泛的一种网络协议，定义了客户端和服务端之间请求与响应的传输标准

Go语言标准库内建提供了`net/http`包，涵盖了HTTP客户端和服务端的具体实现。使用`net/http`包，可以很方便的编写HTTP客户端或服务端的程序。

阅读本节内容，读者需要具备如下知识点：
* [了解HTTP基础知识](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)
* [了解Go语言中接口的用法](https://github.com/Lynn--/TheGoProgrammingLanguage/blob/master/Three.ObjectOrientedProgramming/Interface5.md)

####5.2.1 HTTP客户端

Go内置的`net/http`包提供了最简洁的HTTP客户端实现，无需借助第三方网络通信库(比如`libcurl`)就可以直接使用HTTP中用得最多的`GET`和`POST`方式请求数据。

**1.基本方法**

`net/http`包的Client类型提供了如下几个方法，可以用最简洁的方式实现HTTP请求：
```go
func (c *Client) Get(url string) (resp *Response, err error)
func (c *Client) Post(url string, bodyType string, body io.Reader) (resp *Response, err error) 
func (c *Client) PostForm(url string, data url.Values) (resp *Response, err error) 
func (c *Client) Head(url string) (resp *Response, err error)
func (c *Client) Do(req *Request) (resp *Response, err error) 
```

下面该要介绍这几个方法。
* `http.Get()`
 要请求一个资格资源，只需调用`http.Get()`方法(等价于`http.DefaultClient.Get()`)即可。
 
 ``` go
 resp, err := http.Get("http://example.com/")
	if err != nil {
		//处理错误
		return
	}
	defer resp.Body.Close()
	io.Copy(os.Stdout,resp.Body)
 ```
 上面这段代码请求一个网站首页，并将其网页内容打印到标准输出流中。

* `http.Post()`
要以POST的方式发送数据，也很简单，只需调用http.Post()方法并依次传递下面3个参数即可
	* 请求的目标URL
	* 将要POST数据的资源类型(`MIMEType`)
	* 数据的比特刘(`[]byte`形式)
	
```go 
		//上传图片
	resp, err := http.Post("http://example.com/upload", "image/jpeg", &imageDataBuf)
	if err != nil {
		//处理错误
		return
	}
	if resp.StatusCode != http.StatusOK {
		//处理错误
		return
	}
```	

* `http.PostForm()`
此方法实现了标准编码格式为`application/x-www-form-urlencoded`的表单提交。

```go
	//模拟HTML表单提交一篇新文章
	resp, err := http.PostForm("http://example.com/posts", url.Values{"title":{"article title"},"content":{"articel body"}})
	if err != nil {
		//处理错误
		return
	}
```

* `http.Head()`
此表明只请求目标URL的头部信息，即`HTTPHeader`而不返回`HTTPBody`。
Go内置的`net/http`包同样也提供了`http.Head()`方法，该方法同`http.Get()`方法一样，只需传入目标URL一个参数即可。

```go
//请求一个网站首页的HTTPHeader信息
resp, err := http.Head("http://example.com/")
```

* (*http.Client).Do()
在多数情况下，`http.Get()`和`http.PostForm()`就可以满足需求，但是如果我们发起的HTTP请求需要更多的定制信息，设定一些自定义的`Http Header`字段，比如：
	* 设定自定义的"User-Agent"，而不是默认的"Go http package"
	* 传递Cookie

此时可以使用`net/http`包`http.Client`对象的`Do()`方法来实现：

```go
req, err := http.NewRequest("GET", "http://example.com", nil)
//...
req.Header.Add("User-Agent", "GoBook Custom User-Agent")
//...
client := &http.Client{}
resp, err := client.Do(req)
//...
```

**2.高级封装**

除了之前介绍的基本HTTP操作，Go语言标准库也暴露了比较底层的HTTP相关库，让开发者可以基于这些库灵活定制HTTP服务器和使用HTTP服务。

* 自定义`http.Client`

前面使用的`http.Get()`、`http.Post()`、`http.PostForm()`和`http.Head()`方法其实都是在`http.DefaultClient`的基础上进行调用的，比如`http.Get()`等价于`http.DefaultClient.Get()`，以此类推。

`http.DefaultClient`在字面上就传达了一个信息，既然存在默认的Client，那么HTTP Client大概是可以自定义的。实际上确实如何，在`net/http`包中，的确提供了Client类型。`http.Client`支持的类型：

```go
type Client struct {
	// Transport specifies the mechanism by which individual
	// HTTP requests are made.
	// If nil, DefaultTransport is used.
	//Transport用于确定HTTP请求的创建机制。
	//如果为空，将会使用DefaultTransport
	Transport RoundTripper

	// CheckRedirect specifies the policy for handling redirects.
	// If CheckRedirect is not nil, the client calls it before
	// following an HTTP redirect. The arguments req and via are
	// the upcoming request and the requests made already, oldest
	// first. If CheckRedirect returns an error, the Client's Get
	// method returns both the previous Response and
	// CheckRedirect's error (wrapped in a url.Error) instead of
	// issuing the Request req.
	//
	// If CheckRedirect is nil, the Client uses its default policy,
	// which is to stop after 10 consecutive requests.
	//CheckRedirect定义重定向策略
	//如果CheckRedirect不为空，客户端将在跟踪HTTP重定向前调用该函数
	//两个参数req和via分别为即将发起的请求和已经发起的所有请求，最早的已发起请求在最前面
	//如果CheckRedirect返回错误，客户端将直接返回错误，不会再发起该请求
	//如果CheckRedirect为空，Client将采用一种默认策略，将在10个连续请求后终止
	CheckRedirect func(req *Request, via []*Request) error

	// Jar specifies the cookie jar.
	// If Jar is nil, cookies are not sent in requests and ignored
	// in responses.
	//如果Jar为空，Cookie将不会在请求中发送，并会在响应中bei
	Jar CookieJar

	// Timeout specifies a time limit for requests made by this
	// Client. The timeout includes connection time, any
	// redirects, and reading the response body. The timer remains
	// running after Get, Head, Post, or Do return and will
	// interrupt reading of the Response.Body.
	//
	// A Timeout of zero means no timeout.
	//
	// The Client cancels requests to the underlying Transport
	// using the Request.Cancel mechanism. Requests passed
	// to Client.Do may still set Request.Cancel; both will
	// cancel the request.
	//
	// For compatibility, the Client will also use the deprecated
	// CancelRequest method on Transport if found. New
	// RoundTripper implementations should use Request.Cancel
	// instead of implementing CancelRequest.
	Timeout time.Duration
}
```
在Go语言标准库中，`http.Client`类型包含了3个公开数据成员：
* `Transport RoundTripper`
* `CheckRedirect func(req *Request, via []*Request) err`
* `Jar CookieJar`

其中**`Transport`**类型必须实现`http.RoundTripper`接口。`Transport`指定了执行一个HTTP请求的运行机制，倘若不指定具体的`Transport`，默认会使用`http.DefaultTransport`,这意味着`http.Transport`也是可以自定义的。`net/http`包中的`http.Transport`类型实现了`http.RoundTripper`接口。

**`CheckRedirect`**函数指定处理重定向的策略。当使用HTTP Client的`Get()`或者是`Head()`方法发送HTTP请求时，若响应返回的状态码为30x(比如301/302/303/307)，HTTP Client会在遵循跳转规则前先调用这个`CheckRedirect`函数。

**`Jar`**可用于在HTTP Client中设定Cookie,Jar的类型必须实现了`http.CookieJar`接口，该接口预定义的`SetCookies()` 和`Cookies()`两个方法。如果HTTP Client中没有设定Jar，Cookie将被忽略而不会发送到客户端。实际上，一般都用`http.SetCookie()`方法来设定Cookie

使用自定义的`http.Client`及其`Do()`方法，可以非常灵活地控制HTTP请求，比如发送自定义HTTP Header或是改写重定向策略等。创建自定义的HTTP Client非常简单，具体代码如下

```go
client := &http.Client{
	CheckRedirect: redirectPolicyFunc,
}
resp, err := client.Get("http.example.com")
//...
req, err := http.NewRequest("GET", "http://example.com", nil)
//...
req.Header.Add("User-Agent", "Our Custom User-Agent")
req.Header.Add("If-None-Match", `W/"TheFileEtag"`)
resp, err = client.Do(req)
```