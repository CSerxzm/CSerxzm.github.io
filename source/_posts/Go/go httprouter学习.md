---
title: go httprouter学习
comments: false
date: 2022-06-03 22:39:21
categories:
  - go
tags:
  - http/net
---

对于 go 语言而言，对于 server 的支持较好，net/http 对于 server 支持。httprouter 也对 server 支持，还有就是框架 gin。
在本文中主要对学习的 net/http 和 httprouter 进行介绍。

<!--more-->

# net/http

对于 net/http 的主要步骤有以下：

- http.Server 创建 http.Server 结构体，其中主要包括 Addr 表示监听的地址，一般为`localhost:8080`,还有就是 Handler,在 net/http 中可以不指定。

- 将 url 和 handler 进行映射，有以下两种方法实现。
  1. Handle，该方法需要传入结构体，并且该结构体需要实现接口`func ServeHTTP(w http.ResponseWriter, r *http.Request)`,如文中的结构体`WorldHandler`。
  2. HandleFunc,该方法只需要传入的定义的方法，方法的参数为固定的形式，为 http.ResponseWriter 和指针 http.Request 形式，方法名随意不重复即可。
- 设置监听和服务，即可搭建 server 服务。

```go
package main

import (
	"fmt"
	"net/http"
)

func HandlerHello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello!")
}

type WorldHandler struct{}

func (h *WorldHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "World!")
}

func main() {
	fmt.Println("打开应用.")
	server := http.Server{
		Addr: "127.0.0.1:8080",
	}
	http.HandleFunc("/hello", HandlerHello)
	worldHandler := WorldHandler{}
	http.Handle("/world", &worldHandler)
	server.ListenAndServe()
}
```

# 管道处理函数

管道处理函数即串联处理函数，相当于函数的包装，它时常用于日志的记录，或者在执行函数前进行权限的校验等。

```go
func helloFunc(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "hello!")
}

func log(h http.HandlerFunc) http.HandlerFunc {
	//返回匿名函数
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("log...")
		h(w, r)
	}
}
func main() {
	fmt.Println("应用启动.")
	server := http.Server{Addr: "localhost:8080"}
	http.HandleFunc("/", log(helloFunc))
	server.ListenAndServe()
}
```

# httprouter

httprouter 作为多路复用器，可传入 http.Server 作为 Handler 的对象。

```go
// 一定要存在参数 p httprouter.Params ，否则会报错
//对于query参数的获取，则通过 http.Request 获得。如 http://localhost:8080/hello/xzm?age=23
func helloHandler(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	values := r.URL.Query()
	age := values["age"][0]
	fmt.Fprint(w, "Hello,"+p.ByName("name")+",age="+age)
}

func main() {
	fmt.Println("应用启动.")
	mux := httprouter.New()
	mux.GET("/hello/:name", helloHandler)
	server := http.Server{
		Addr:    "localhost:8080",
		Handler: mux,
	}
	server.ListenAndServe()
}
```

对于 httprouter 的路由命名匹配，主要有以下两种。

- :name named parameter (只能根据路由命名进行捕获)
- *name catch-all parameter (*表示捕获任意内容)

```go
Path: /blog/:category/:post
router.GET("/blog/:category/:post", Hello) //（category/post可以看成是一个变量）

当请求路径为：
/blog/go/request-routers            match: category="go", post="request-routers"
/blog/go/request-routers/           no match, but the router would redirect
/blog/go/                           no match
/blog/go/request-routers/comments   no match
```

## http header

对于 http header 中包含很多字段，如 Cookie,Location,Content-Type，在 header 中可以使用函数`w.Header().Set(key, val)`进行设置，需要注意的是对于状态码的写入是最后一个步骤完成`w.WriteHeader(code int)`,否则后面对于头部的写入就不会完成。

```go
type Post struct {
	User    string
	Threads []string
}

func writeexample(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	str := "<h1>hello!</h1>"
	w.Write([]byte(str))
}

func headerexample(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	w.WriteHeader(220)
	str := "<h1>hello!</h1>"
	w.Write([]byte(str))
}

func redictexample(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	//注意，在这里设置状态码应该是最后的操作，否则将不能对首部进行写入
	w.Header().Set("Location", "www.baidu.com")
	w.WriteHeader(302)
}

func encodingexample(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	w.Header().Set("Content-Type", "application/json")
	post := Post{
		User:    "xiang",
		Threads: []string{"first", "second"},
	}
	json, _ := json.Marshal(post)
	w.Write(json)
}
```

对于上面的函数`func json.Marshal(v any) ([]byte, error)`是将结构体转换成字符串。

接下来以 Header 中 Cookie 为示例，来进行实验对其进行增加、修改和删除。
对于 Cookie 的过期设置是设置 cookie 过期的方式是设置 MaxAge 为负数或 0，为了兼容所有浏览器，可以设置 Expires 为过去的一段时间。

```go
// 设置cookie
func setcookiehandler(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	cookie := http.Cookie{
		Name:     "my_cookie",
		Value:    "this is go program!",
		HttpOnly: true,
	}
	//注意，这里的key为"Set-Cookie"为固定的值，表示设置头部的该字段
	//w.Header().Set("Set-Cookie", cookie.String())
	//w.Header().Add("Set-Cookie", cookie.String())
	http.SetCookie(w, &cookie)
	w.Write([]byte("hello world!"))
}

//获得cookie
func getcookiehandler(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	//方式0
	//cookies := r.Cookies()
	// 方式1(方式0和1的"Cookie")为固定值
	//cookies := r.Header.Get("Cookie")
	//w.Write([]byte(cookie))
	//方式2
	cookies := r.Header["Cookie"]
	fmt.Fprint(w, cookies)
}

//获得指定的cookie
func onecookiehandler(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	cookie, _ := r.Cookie("my_cookie")
	fmt.Fprint(w, cookie)
}
```

## http template

对于 server 中，有很多界面的展示会借助 template 来完成的，如 jsp 等。我们只需要在渲染模板的时候准备好相关的数据，而后就可以渲染出携带数据信息的界面。

- 服务端，服务端程序主要是准备好相关的数据。

```go
var t *template.Template

func init() {
	//t, _ = template.ParseFiles([]string{"t1.html", "t2.html"}...)
	//等同于,不过New里面只能传递单个文件
	//t = template.New("tmpl.html")
	//t, _ = t.ParseFiles("tmpl.html")
	//Must函数可以包裹一个函数，被包裹的函数会返回一个指向模板的指针和一个错误，如果这个错误不是nil，那么Must函数将产生一个 panic。
	//（在 Go 里面，panic 会导致正常的执行流程被终止：如果 panic 是在函数内部产生的，那么函数会将这个 panic 返回给它的调用者。panic 会一直向调用栈的上方传递，直至main函数为止，并且程序也会因此而崩溃。）
	t = template.Must(template.ParseFiles([]string{"t1.html", "t2.html"}...))
}

func formatDate(t time.Time) string {
	return t.Format("2020-12-23")
}

func process_t1(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	//t.Execute(w,"Hello word,t1!")
	t.ExecuteTemplate(w, "t1.html", "Hello word,t1!")
}

func process_t2(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	//返回的数据，前端采用 .Title 和 .Addr 的方式进行读取
	type Post struct {
		Title string
		Addr  string
	}
	post := Post{"题目", "湖北武汉"}
	t.ExecuteTemplate(w, "t2.html", post)
}
```

- 前端，前端主要是准备好后端需要渲染的模板文件，如上面代码的`t1.html`和`t2.html`.

```html
<!--t2.html-->
<!DOCTYPE html>
<html>
	<head>
		<title>demo</title>
	</head>
	<body>
		<h1>t2</h1>
		<!--获得所有的数据-->
		{{ . }}
		<!--获得某个字段数据-->
		{{ .Title }} {{ .Addr }}
	</body>
</html>
```

我们也可以从后端传入函数到前段(实质是渲染的时候执行这些函数)。如下面的示例中对时间进行格式化操作。在`t3.html`中可以使用两种方式调用该函数，分别是`函数名 数据`和`数据 | 函数名`，后者形似管道。

```go
func formatDate(t time.Time) string {
	return t.Format("2006-01-02 03:04")
}

func process_t1(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	funcMap := template.FuncMap{"fdate": formatDate}
	t := template.New("t3.html").Funcs(funcMap)
	t, _ = t.ParseFiles("t3.html")
	t.Execute(w, time.Now())
}
```

```html
<!--t3.html-->
<!DOCTYPE html>
<html>
  <head>
    <title>demo</title>
  </head>
  <body>
    <h1>t2</h1>
    {{. | fdate}}
  </br>
    {{fdate .}}
  </body>
</html>
```

对于模板中的动作主要有以下四种。

- 条件动作，即控制展示与否

```text
{{ if arg }}
	some content
{{ else }}
	other content
{{ end }}
```

- 迭代动作,即进行列表的展示

```text
{{ range array }}
	Dot is set to the element {{ . }}
{{ end }}
<!--另外的方式-->
{{ range $key,$value := . }}
	The key is {{ key }} and the value is {{ value }}.
{{ end }}
```

- 设置动作，在`{{ with arg }}`和`{{ end }}`之间的点将被设置为参数 arg 的值

```text
{{ with arg }}
	 Now the dot is set to {{ . }} // 此时{{ . }}为 arg
{{ end }}
```

- 包含动作,用于页面的拆分后的重组展示，如 header 和 footer

```text
{{ template "t2.html" }}
```
