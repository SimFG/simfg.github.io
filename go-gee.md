# 自己动手写一个Web框架
## 框架基本要求
1. 根据请求url映射到不同的函数上
2. 封装Context对象，如请求参数获取、常见结果形式返回等，使得请求处理逻辑更加专注于业务
3. 动态路由，相同规则形式的url，映射到同一个函数上，如获取用户信息url为/:id/info
4. 路由分组，可以对不同组api进行不同的控制，如鉴权、流量控制、是否可匿名访问，同时也支持中间件
5. 静态文件分发，由url获取服务器上目录文件；支持html模版渲染
6. 内部错误处理，当业务函数出现panic，框架需保证正常运行

## Web服务的基本组成
Web服务最基本的两个组成：端口监听和请求映射，前者接收请求，后者当获得请求后对请求进行相应处理。
go语言内置http网络编程库为`net.http`，使用它实现上诉两个要求：

```go
func main() {
	// 请求映射
	http.HandleFunc("/", indexHandler)
	// 端口监听
	http.ListenAndServe(":9999", nil)
}

func indexHandler(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
}
```

其实可以发现，`ListenAndServe`函数第二个参数为接口`Handler`，形式为：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

这个接口处理请求映射的，如果传nil，其请求映射会由`net.http`包中的`DefaultServeMux`进行管理，为了更好实现请求映射，则可以实现`Handler`接口。

## Engine封装

对外，暴露一个Engine的接口，提供框架的基本功能，如上述的运行和请求监听，分别为：`Run`和`Get`/`Post`，还有添加分组、添加路由、静态资源映射、添加中间件等，后续会一一介绍。

```go
func (e *engine) Run(address string) error {
	r := e.router.(*router)
	for k, v := range r.nodes {
		fmt.Printf("-------%s-------\n", k)
		v.Print(0)
	}
	return http.ListenAndServe(address, e)
}

func (e *engine) ServeHTTP(writer http.ResponseWriter, request *http.Request) {
    e.router.Handle(NewCtx(writer, request, e))
}
```

`Run`接口的实现，首先会打印出路由中的前缀树，后续会有详细介绍，然后具体请求如何分发交给`router`，因为`engine`实现了`Handler`接口中的`ServerHTTP`方法，该方法将请求交给`router`处理。

```go
type Engine interface {
	Group
	AddGroup(g Group)
	AddRoute(method string, pattern string, handler HandlerFunc)
	Run(address string) error

	SetFuncMap(t template.FuncMap)
	LoadHtmlGlob(pattern string)
	ExecuteTemplate(w http.ResponseWriter, pattern string, data interface{}) error
}

type Group interface {
    NewGroup(prefix string) Group
    Get(pattern string, handler HandlerFunc)
    Post(pattern string, handler HandlerFunc)
    Static(relative string, real string)
    AddWrapper(w Wrapper)
}
```

请求处理的方法，放在`Group`接口中，对应：`Get`和`Post`，当然也可以添加其他的，如put、delete等，放在分组接口是便于后续对于在一组的api进行管理，如添加相同的中间件、避免过多的相同前缀等。在group中，做了两个处理，为拼接前缀和组装中间件，最终调用`Router`接口的`AddRoute`方法。

```go
func (g *group) Get(pattern string, handler HandlerFunc) {
	g.engine.AddRoute("GET", g.prefix+pattern, g.chainHandler(handler))
}

func (e *engine) AddRoute(method string, pattern string, handler HandlerFunc) {
    e.router.AddRoute(method, pattern, handler)
}
```

## Context 封装
在编写请求处理逻辑的时候，会从请求中获取参数、url或者header信息，在逻辑处理完成后，将结果以不同格式返回到response流中，这部分也是框架所必须的，没有这部分，在框架的使用过程，将会有大量的重复代码，导致逻辑臃肿，也不易后续扩展。
```go
type Ctx interface {
	PostForm(key string) string
	Query(key string) string
	SetHeader(key string, val string)
	String(code int, format string, values ...interface{})
	Json(code int, obj interface{})
	Data(code int, data []byte)
	Html(code int, name string, data interface{})
	File(h http.Handler)
	Path() string
	Method() string
	Header() http.Header
	Params(p map[string]string)
}
```

`Ctx`接口，提供了业务处理时获取请求常见的参数，同时也封装了不同格式的结果输出方法，如果要形成一个成熟的框架，或许应该还需要将请求中原生的参数进行暴露，即`http.Request`和`http.ResponseWriter`，因为可能在业务处理过程中需要其他不常见的参数。

```go
func (e *engine) ServeHTTP(writer http.ResponseWriter, request *http.Request) {
	e.router.Handle(NewCtx(writer, request, e))
}

func (e *engine) AddRoute(method string, pattern string, handler HandlerFunc) {
    e.router.AddRoute(method, pattern, handler)
}
```

在接收到请求时，则会调用`ServeHTTP`函数，可以看到一个请求会使用`NewCtx`生成一个context对象，真正的分发逻辑则是在`router`对象的`Handle`方法中。在添加请求处理逻辑时，最终会调用`AddRoute`方法，在`router`对象中注册请求处理，其中最后一个参数是`HandlerFunc`函数，其入参便是`Ctx`对象。
## 请求路由
主要是通过请求路径映射到已经注册的函数上，路由的工作也就包含了：添加路由和分发请求。

```go
type Router interface {
	AddRoute(method string, pattern string, handler HandlerFunc)
	Handle(c Ctx)
}
```

在不考虑动态路由的情况下，其接口的实现可以通过一个map进行实现，其key为请求标识，由请求方法和路径组成，因为相同路径可请求方法不一致，如get、post等。添加路由则是往map中放入一个元素，value为`HandlerFunc`，接收到请求后则根据请求方法和路径组成key，然后从map中获取元素。

```go
type router struct {
	nodes    map[string]Node
	handlers map[string]HandlerFunc
}
```

在最终的实现中可以看到，除了存储handlers的map，还有一个存储Node的map，这个便是为了实现动态路由所需要的。动态路由，也就是在注册时url并不明确，只有在收到请求才知道，有点类似于接口，例如url为：/:id/name，其中`:id`则表示一个参数，最终的url可能是`/123/name`。

实现动态路由，其问题可以抽象为一个字符串匹配问题，这里采用了前缀树的方案，核心就是将需要字符串进行分段匹配。前缀树的节点功能抽象，主要包括了：插入和查找，这里为了便于调试，新增了一个Print的函数。

```go
type Node interface {
	Insert(pattern string, parts []string, height int)
	Search(parts []string, height int) (string, map[string]string)
	Print(i int)
}
```

在前缀树中，一个节点需要记录的信息包括了：
- 当前节点信息，pattern
- 如果当前节点是可匹配节点，则需记录匹配信息，part
- 孩子节点列表，children
- 是否为模糊匹配，isWild
一个node结构组成：

```go
type node struct {
	pattern  string
	part     string
	children []*node
	isWild   bool
}
```

考虑一个基本方法，给出url一个节点信息，然后与当前节点进行匹配，这个匹配结果可以用来插入，也有可能用来查询。首先，遍历当前节点的孩子节点，如果孩子节点存储的信息与当前节点信息一样，自然需要放入返回结果列表中；另外还需要考虑一种状况，就是假如是在查找，这个时候如果当前节点是模糊匹配的节点，它也要被包含，但是插入的时候不需要，因为插入的时候只需要确定每个节点的信息是否一直，无需考虑当前节点是模糊匹配。具体实现如下：

```go
func (n *node) matchChildren(part string, matchOnlyOne bool) []*node {
	var res []*node
	for _, child := range n.children {
		if child.part == part || (!matchOnlyOne && child.isWild) {
			res = append(res, child)
			if matchOnlyOne {
				break
			}
		}
	}
	return res
}
```

有了上述方法，再来分析如何实现插入和查询。先介绍下，url是如何被划分的。假如一个url是`/index/hello`，需要两个节点进行记录，节点分别记录信息`index`和`hello`，如果在插入一个url，值为`/index/foo`，这个时候只会在之前`index`的节点下新添一个子节点，那就是`foo`节点，也就是当前树中只有三个节点。

插入和搜索，其实都是一个递归问题。先考虑插入实现，其结束条件就是当前树的高度与需插入信息的长度一直，然后在返回之前将当前url信息记录在`pattern`字段中。递归过程，首先获取当前树高度需要记录的信息，然后匹配当前节点的子节点，如果未获得节点，则新生成一个节点，并将该节点添加到当前节点的子节点列表中，如果找到，则保存该节点信息。然后在调用匹配子节点的Insert的函数，开启递归。

```go
func (n *node) Insert(pattern string, parts []string, height int) {
	if len(parts) == height {
		n.pattern = pattern
		return
	}

	part := parts[height]
	children := n.matchChildren(part, true)
	var child *node
	if len(children) == 0 {
		child = &node{part: part, isWild: isWild(part)}
		n.children = append(n.children, child)
	} else {
		child = children[0]
	}
	child.Insert(pattern, parts, height+1)
}
```

搜索的核心思想也是类似，这里做了一些调整，也就是并不返回当前节点，而是返回当前节点的`pattern`，根据上面程序设计可知，如果当前节点是某个url的最后节点信息，则`pattern`会被赋值，而这个`pattern`也是存储在`router`中的key，所以返回值就直接返回这个字符串。

这个前缀树存在两个模糊匹配的标识，分别为`:`和`*`，前者只能匹配一个节点，后者可以匹配多个节点。举个例子，`/:id/hello`可以匹配`/123/hello`或者`/456/hello`，但是无法匹配`/123/456/hello`，则`*`可以匹配多个节点信息，但是只能放在最后，常见的使用则是静态文件匹配，如`/file/*filepath`，可以匹配`/file/index.html`，也可以匹配`/file/asset/style.css`。

另外，还有一个常见的需求，如果是模糊匹配，在处理请求时，经常会需要获取其对应模糊标识的对应值，如`/:id/foo`，如果真是url为`/123/foo`，可以通过`id`获取到`123`这个值。

搜索是入参数包括了当前url的所有节点信息和当前节点高度，递归的结束条件为：当前树高度与节点信息长度一直，或者当前节点是通配符节点，达到结束条件，则判断当前节点是不是模糊节点，是的话记录当前节点信息与真实节点信息的对应关系在返回。然后同样是获取当前需要匹配的节点信息，搜索子节点，遍历子节点进行再一次递归搜索，如果搜索存在结果，说明存在一条完整的匹配路径，这时候在搜集当前节点信息与真实节点信息之间的关系，然后返回。其实现如下：

```go
func (n *node) Search(parts []string, height int) (string, map[string]string) {
	params := make(map[string]string)
	if len(parts) == height || strings.HasPrefix(n.part, "*") {
		if n.isWild {
			params[n.part[1:]] = parts[height-1]
		}
		return n.pattern, params
	}

	part := parts[height]
	children := n.matchChildren(part, false)
	for _, child := range children {
		result, p := child.Search(parts, height+1)
		if result != "" {
			if child.isWild {
				params[child.part[1:]] = parts[height]
			}
			for s, s2 := range p {
				params[s] = s2
			}
			return result, params
		}
	}
	return "", params
}
```

完成上述实现，打印整颗前缀树很快就能有答案。

```go
func (n *node) Print(i int) {
	fmt.Printf("height: %d, %+v\n", i, n)
	for _, child := range n.children {
		child.Print(i + 1)
	}
}
```

## 路由分租
一个路由分组，也就是多个请求组合，接口自然包含了添加请求、创建子分租及其后续涉及的中间件。

```go
type Group interface {
	NewGroup(prefix string) Group
	Get(pattern string, handler HandlerFunc)
	Post(pattern string, handler HandlerFunc)
	Static(relative string, real string)
	AddWrapper(w Wrapper)
}
```

`Static`是处理文件映射，后续会进行介绍。分组的信息最后都是由`engine`进行记录，添加子分租则是创建一个`group`对象，然后通知`engine`，具体实现如下：

```go
func (g *group) NewGroup(prefix string) Group {
	res := &group{
		prefix: g.prefix + prefix,
		engine: g.engine,
		parent: g,
	}
	g.engine.AddGroup(res)
	return res
}
```

中间件的装配，则是通过`chainHandler`方法进行实现的。最终返回值是一个匿名函数，可以看到有一个`defer`，这个就是为了防止编写业务逻辑的时候存在panic，这样即使panic服务也可以正常运行。

```go
defer func() {
    if err := recover(); err != nil {
        message := fmt.Sprintf("%s", err)
        log.Printf("%s\n\n", trace(message))
        c.String(http.StatusInternalServerError, "Internal Server Error")
    }
}()
```

`chainFunc`，接收了两个参数，一个是真实的请求处理接口，另外一个则是装饰器，实现逻辑就是在`origin`方法调用前后调用装饰器的相应方法，然后最后就是使用`chainFunc`组装所有的装饰器。假如由装饰器A和B，真实的业务逻辑为x，每次chain包含的执行过程是：
- 未进入循环，则是：x
- 读取最后一个装饰器，即B，这个时候chain通过`chainFunc`执行逻辑变成了：B.Pre() -> x -> B.Post()
- 读取下一个装饰器，即A，则是：A.Pre() -> B.Pre() -> x -> B.Post() -> A.Post
最终实现则是：

````go
func (g *group) chainHandler(handler HandlerFunc) HandlerFunc {
	middlewares := g.middlewares
	n := len(middlewares)
	return func(c Ctx) {
		// defer ....

		chainFunc := func(origin HandlerFunc, wrapper Wrapper) HandlerFunc {
			return func(c Ctx) {
				wrapper.Pre(c)
				origin(c)
				wrapper.Post(c)
			}
		}
		chain := handler
		for i := n - 1; i >= 0; i-- {
			chain = chainFunc(chain, middlewares[i])
		}
		chain(c)
	}
}
````

## 静态文件分发
处理文件，之前已经由前缀树支持了尾缀的模糊匹配，对于文件的映射，则go中已经做好了封装，具体实现如下：

```go
func (g *group) Static(relative string, real string) {
	absoultPath := path.Join(g.prefix, relative)
	fs := http.Dir(real)
	fileServer := http.StripPrefix(absoultPath, http.FileServer(fs))
	g.Get(path.Join(relative, "/*filepath"), func(c Ctx) {
		file := c.Query("filepath")
		if _, err := fs.Open(file); err != nil {
			c.String(http.StatusNotFound, "File Not Found, path: %s", c.Path())
			return
		}
		c.File(fileServer)
	})
}
func (c *ctx) File(h http.Handler) {
    h.ServeHTTP(c.writer, c.req)
}
```

其中`relative`表示url文件前缀，而`real`表示文件系统中的真实路径。
## html文件渲染
这部分go也提供了核心实现，组装一下即可。`Engine`中涉及的三个函数：

```go
type Engine interface {
	SetFuncMap(t template.FuncMap)
	LoadHtmlGlob(pattern string)
	ExecuteTemplate(w http.ResponseWriter, pattern string, data interface{}) error
}
```

依次作用是：供模版文件调用的函数列表、加载的模版目录、执行模版，最后一个执行模版其实就是在html结果返回中被调用。

```go
func (e *engine) SetFuncMap(t template.FuncMap) {
	e.funcMap = t
}
func (e *engine) LoadHtmlGlob(pattern string) {
	e.htmlTemplate = template.Must(template.New("").Funcs(e.funcMap).ParseGlob(pattern))
}
func (c *ctx) Html(code int, name string, data interface{}) {
    c.SetHeader("Content-Type", "text/html")
    c.status(code)
    if err := c.engine.ExecuteTemplate(c.writer, name, data); err != nil {
    c.fail(err)
    }
}
```

在main函数中的使用：

```go
func main() {
    e := gee.NewEngine()
    e.AddWrapper(&gee.ApiTimer{})
    e.SetFuncMap(template.FuncMap{
        "FormatAsDate": FormatAsDate,
    })
    e.LoadHtmlGlob("templates/*")
    e.Static("/assets", "./static")
    e.Get("/", func (c gee.Ctx) {
    c.Html(http.StatusOK, "index.tmpl", map[string]interface{}{
            "title": "hello go-gee",
            "now":   time.Now(),
        })
    })
}
```

## 参考

原文链接：https://geektutu.com/post/gee.html

### 与原文不一致的内容

1. 将对象都抽象成接口，例如Engine接口

```go
type Engine interface {
	Group
	AddGroup(g Group)
	AddRoute(method string, pattern string, handler HandlerFunc)
	Run(address string) error

	SetFuncMap(t template.FuncMap)
	LoadHtmlGlob(pattern string)
	ExecuteTemplate(w http.ResponseWriter, pattern string, data interface{}) error
}
```

2. 读取动态路由的参数，是在匹配前缀树过程中完成记录的

```go
func (n *node) Search(parts []string, height int) (string, map[string]string) {
	params := make(map[string]string)
	if len(parts) == height || strings.HasPrefix(n.part, "*") {
		if n.isWild {
			params[n.part[1:]] = parts[height-1]
		}
		return n.pattern, params
	}

	part := parts[height]
	children := n.matchChildren(part, false)
	for _, child := range children {
		result, p := child.Search(parts, height+1)
		if result != "" {
			if child.isWild {
				params[child.part[1:]] = parts[height]
			}
			for s, s2 := range p {
				params[s] = s2
			}
			return result, params
		}
	}
	return "", params
}
```

3. 中间件放在路由分组中，与grpc拦截器类似

```
func (g *group) Get(pattern string, handler HandlerFunc) {
    g.engine.AddRoute("GET", g.prefix+pattern, g.chainHandler(handler))
}

func (g *group) chainHandler(handler HandlerFunc) HandlerFunc {
	middlewares := g.middlewares
	n := len(middlewares)
	return func(c Ctx) {
		defer func() {
			if err := recover(); err != nil {
				message := fmt.Sprintf("%s", err)
				log.Printf("%s\n\n", trace(message))
				c.String(http.StatusInternalServerError, "Internal Server Error")
			}
		}()

		chainFunc := func(origin HandlerFunc, wrapper Wrapper) HandlerFunc {
			return func(c Ctx) {
				wrapper.Pre(c)
				origin(c)
				wrapper.Post(c)
			}
		}
		chain := handler
		for i := n - 1; i >= 0; i-- {
			chain = chainFunc(chain, middlewares[i])
		}
		chain(c)
	}
}
```