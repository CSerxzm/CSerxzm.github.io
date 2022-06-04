---
title: gin学习
comments: false
date: 2022-06-04 11:30:52
categories:
  - go
tags:
  - gin
---

在本文将会介绍 gin 框架。gin 是一个用 go 语言编写, 基于 httprouter 开发的 web 框架。 它是一个类似于 martini 但拥有更好性能的 api 框架, 由于使用了 httprouter，速度提高了近 40 倍。如果你是性能和高效的追求者, 你会爱上 gin。

<!--more-->

# gin 的安装

```shell
go get -u github.com/gin-gonic/gin
```

# helloworld

- 创建路由，使用 `gin.Default()` 返回 `*gin.Engine`。
- 绑定路由规则，执行的函数。
- 监听端口，`r.Run()`，默认为 `8080`，可传入端口，如`:8000`。

```go
func main() {
	// 1.创建路由
	r := gin.Default()
	// 2.绑定路由规则，执行的函数
	// gin.Context，封装了request和response
	r.GET("/:name/*action", func(c *gin.Context) {
		name := c.Param("name")
		action := c.Param("action")
		// 去掉前后的斜线 /
		//action = strings.Trim(action, "/")

		//获得参数值 DefaultQuery 在没有该值的时候返回默认值，Query在没有该值的时候返回空值
		age := c.Query("age")
		// age := c.DefaultQuery("age", 23)
		c.String(http.StatusOK, fmt.Sprintf("name=%s,action=%s,age=%s\n", name, action, age))
	})
	// 3.监听端口，默认在8080
	// Run("里面不指定端口号默认为8080")
	r.Run(":8000")
}
```

对于上面的程序，需要注意参数的获取，主要是路由参数的获取和 query 参数的获取。

- 路由参数的获取。路由参数的获取使用函数`func (*gin.Context).Param(key string) string`进行获取。在 url 中可以使用占位符，即`:name`或`*name`,对于`*name`获得的参数值每次是以`/`开头。
- query 参数的获取。使用函数`func (*gin.Context).Query(key string) (value string)`获取 query 参数。（对于该函数，没有获取的值返回为 nil,也可以使用函数`func (*gin.Context).DefaultQuery(key string, defaultValue string) string`获取默认的值）

```shell
GET /?name=Manu&lastname=
c.DefaultQuery("name", "unknown") == "Manu"
c.DefaultQuery("id", "none") == "none"
c.DefaultQuery("lastname", "none") == ""
```

> 注意: 对于上面示例中的 `action := c.Param("action")` 获取的是 `action` 之后 `url` 的所有路由参数。如 `GET /localhost/xzm/url/path?age=20`,对于 action 的值则为`/url/path`

# 数据的绑定和获取

在程序中，主要数据的形式为`url`、`json` 和 `form` 的表单数据的绑定和获取，xml 数据这里就不进行介绍。
在本节主要使用的数据结构如下,其中映射的字段有 `form` 表单数据、`json` 的数据、`url` 数据和 `xml` 数据。`binding`则是进行绑定的时候验证。

```go
type Login struct {
	// binding:"required"修饰的字段，若接收为空值，则报错，是必须字段
	User string `form:"username" json:"user" uri:"user" xml:"user"
	binding:"required"`
	Password string `form:"password" json:"password" uri:"password"
	xml:"password" binding:"required"`
}
```

在使用绑定数据时，使用通用方法为`func (*gin.Context).Bind(obj any) error`，该方法能够通过请求头中`content-type`自动推断数据的类型。也可以使用对应数据类型的方法,如`func (*gin.Context).BindUri(obj any) error`。

> Bindxxx：解析错误会在 header 中添加状态码 400 的返回信息；ShouldBindxxx：解析错误直接返回，返回什么错误状态码由自己决定。

- `url`数据的获取

`url` 的方式，需要在 `url` 中进行数据占位，即使用`:name`或`*name`。

```go
	r := gin.Default()
	r.GET("/url/:user/:password", urlBind)
```

```go
func urlBind(c *gin.Context) {
	var url Login
	if err := c.BindUri(&url); err != nil {
		// 返回错误信息
		// gin.H封装了生成json数据的工具
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	// 判断用户名密码是否正确
	if url.User != "root" || url.Password != "admin" {
		c.JSON(http.StatusBadRequest, gin.H{"status": "304"})
		return
	}
	c.JSON(http.StatusOK, gin.H{"status": "200"})
}
```

- `json`数据的获取

```go
func JsonBind(c *gin.Context) {
	var json Login
	// 将request的body中的数据，自动按照json格式解析到结构体
	if err := c.ShouldBindJSON(&json); err != nil {
		// 返回错误信息
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	// 判断用户名密码是否正确
	if json.User != "root" || json.Password != "admin" {
		c.JSON(http.StatusBadRequest, gin.H{"status": "304"})
		return
	}
	c.JSON(http.StatusOK, gin.H{"status": "200"})
}
```

- `form`数据的获取

```go
func formBind(c *gin.Context) {
	var form Login
	// Bind()默认解析并绑定form格式
	// 根据请求头中content-type自动推断
	if err := c.Bind(&form); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	// 判断用户名密码是否正确
	if form.User != "root" || form.Password != "admin" {
		c.JSON(http.StatusBadRequest, gin.H{"status": "304"})
		return
	}
	c.JSON(http.StatusOK, gin.H{"status": "200"})
}
```

# 返回响应

对于返回的响应，支持`string`,`json`,结构体和`xml`。同样`xml` 的方式此处不进行描述。

```go
   //创建路由
	r := gin.Default()

	// 1.string
	r.GET("/someString", func(c *gin.Context) {
		c.String(200, "%s", "hello")
	})

	// 2.json
	r.GET("/someJSON", func(c *gin.Context) {
		c.JSON(200, gin.H{"message": "someJSON", "status": 200})
	})

	// 3. 结构体响应
	r.GET("/someStruct", func(c *gin.Context) {
		var msg struct {
			Name    string
			Message string
			Number  int
		}
		msg.Name = "root"
		msg.Message = "message"
		msg.Number = 123
		c.JSON(200, msg)
	})
```

# 表单数据 and 文件上传

对于表单的数据，除了前面的`Bind`函数可以实现数据的绑定和获取。在本部分将会介绍函数`func (*gin.Context).PostForm(key string) (value string)`用于获得表单数据，同时也可以使用函数`func (*gin.Context).DefaultPostForm(key string, defaultValue string) string`。

对于文件的上传，则使用函数`func (*gin.Context).FormFile(name string) (*multipart.FileHeader, error)`获得文件，而后使用函数`func (*gin.Context).SaveUploadedFile(file *multipart.FileHeader, dst string) error`将文件存储到磁盘。

```go
	r := gin.Default()
	//限制上传文件的大小为8M
	r.MaxMultipartMemory = 8 << 20
	r.POST("/form", func(c *gin.Context) {
		//获得表单数据，也可以使用c.DefaultPostForm()给未获得数据传入默认值
		typevalue := c.DefaultPostForm("type", "get")
		name := c.PostForm("name")
		password := c.PostForm("password")
		//获得文件
		file, err := c.FormFile("file")
		if err != nil {
			fmt.Println("file error", err.Error())
		} else {
			c.SaveUploadedFile(file, file.Filename)
		}
		c.String(http.StatusOK, fmt.Sprintf("name=%s,password=%s,type=%s\n", name, password, typevalue))
	})
	r.Run(":8000")
```

# url 相关

## url 分组

对于应用较多时，会发生 api 重叠的现象。使用 url 分组就可以进行避免。如可请求`/v1/login`和`/v2/login`进行事件或业务的区分，同时后面可以根据分组来注册中间件。

```go
func main() {
	r := gin.Default()
	v1 := r.Group("/v1")
	{
    //这里使用{}是为了代码规范
		v1.GET("/login", login)
		v1.GET("/submit", submit)
	}
	v2 := r.Group("/v2")
	{
		v2.GET("/login", login)
		v2.GET("/submit", submit)
	}
	r.Run()
}
```

## 路由拆分

当项目的规模增大后就不太适合继续在项目的 `main.go` 文件中去实现路由注册相关逻辑了，我们会倾向于把路由部分和 `app` 代码都拆分出来，形成一个单独的文件或包。
以下为示例的工程结构，其中业务包括 `shop` 和 `blog` 部分。

```txt
gin_demo
├─main.go
├─routers
|    └router.go
├─app
   ├─shop
   |  ├─handler.go
   |  └router.go
   ├─blog
      ├─handler.go
      └router.go
```

从上面的工程结构可见，`app` 目录中包含的是相对独立的应用。在文件`handler.go`主要是处理方法。而`router.go`则是完成将`url`和处理方法的映射。
文件`gin_demo/app/shop/handler.go`代码如下。

```go
func postHandler(c *gin.Context) {
	c.String(http.StatusOK, fmt.Sprintf("hello, %s\n", "blog post"))
}

func commitHandler(c *gin.Context) {
	c.String(http.StatusOK, fmt.Sprintf("hello, %s\n", "blog commit"))
}
```

文件`gin_demo/app/shop/router.go`代码如下,注意这里的`func Routers(g *gin.Engine)`函数名是首字母大写，这个是`shop`暴露的函数，用于在其他包中注册。

```go
func Routers(g *gin.Engine) {
	// g.POST("/post", postHandler)
	// g.GET("/commit", commitHandler)
	blog := g.Group("/blog")
	{
		blog.POST("/post", postHandler)
		blog.GET("/commit", commitHandler)
	}
}
```

文件`gin_demo/routers/router.go`代码如下。在这个文件(包)中含有全局数组变量`options`,该数组元素的类型是`func(*gin.Engine)`的类型的函数，也就是上面的说的函数`func Routers(g *gin.Engine)`。

在该文件(包)中的函数都是暴露的，在`main`包中通过这里面的函数进行路由的注册。

```go
// 注意这里类型的定义，是函数func的类型
type Option func(*gin.Engine)

var options []Option
//这里传入的是可变参数
func Include(opts ...Option) {
	options = append(options, opts...)
}

//创建路由，并且将映射部分加入到路由中
func Init() *gin.Engine {
	r := gin.Default()
	for _, opt := range options {
    //这里opt其实是函数
		opt(r)
	}
	return r
}
```

文件`gin_demo/main.go`代码如下，主要在这个包中完成路由的创建、注册和启动操作。

```go
func main() {
	routers.Include(blog.Routers, shop.Routers)
	r := routers.Init()
	r.Run()
}
```

# html/template

在 gin 中同样设计到模板的渲染。在 gin 框架中使用比较简单，主要有以下注意的地方。

- 使用函数`func (*gin.Engine).LoadHTMLGlob(pattern string)`加载模板文件。该函数支持正则匹配，一般会将其指向模板文件的根目录，而后扫描该根目录的所有的符合条件的模板文件。
- 使用函数` func (*gin.RouterGroup).Static(relativePath string, root string) gin.IRoutes`引入静态文件，在前端使用`/assets/**`来引用文件夹`assets`中的静态文件，如`<img src="/assets/head.jpg"/>`。
- 使用函数`func (*gin.Context).HTML(code int, name string, obj any)`来完成模板的渲染。其中`code`为 http 的状态码，`name`为模板中最开始一句`define`定义的模板名，obj 为传入的数据。

```go
func main() {
	r := gin.Default()
	//加载模板文件
	r.LoadHTMLGlob("template/**/*")
	//引入静态文件
	r.Static("/assets", "./assets")
	//主页显示
	r.GET("/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "user/index.html", gin.H{"name": "xzm",
			"address": "湖北省武汉市", "message": "这是消息"})
	})
	//重定向到www.baidu.com
	r.GET("/redirect", func(c *gin.Context) {
		c.Redirect(http.StatusMovedPermanently, "https://www.baidu.com")
	})

	r.GET("/long_async", func(c *gin.Context) {
		// 在启动新的goroutine时，不应该使用原始上下文，必须使用它的只读副本
		copyContext := c.Copy()
		// 异步处理
		go func() {
			time.Sleep(3 * time.Second)
			log.Println("异步执行：" + copyContext.Request.URL.Path)
		}()
	})

	// 2.同步
	r.GET("/long_sync", func(c *gin.Context) {
		time.Sleep(3 * time.Second)
		log.Println("同步执行：" + c.Request.URL.Path)
	})

	r.Run()
}
```

对于文件`template/user/index.html`的代码如下。其中有定义模板名和嵌入其他模板文件。

```html
{{ define "user/index.html"}}
<!DOCTYPE html>
<html>
	<head>
		<title>demo</title>
	</head>
	<body>
		{{template "public/header.html" .}}
		<div>hello!{{.message}}</div>
		{{template "public/footer.html" .}}
	</body>
</html>
{{ end }}
```

# 中间件

中间件在 go 语言中用于鉴权、日志和统计时间等功能。

## 定义中间件

中间件 MiddleWare 实际上就是一个返回值为`Handler` 的中间处理函数。在 gin 中返回的值为`gin.HandlerFunc`。

```go
func NextMiddleWare() gin.HandlerFunc {
	return func(c *gin.Context) {
		t := time.Now()
		fmt.Println("中间件开始执行了")
		// 设置变量到Context的key中，可以通过Get()取
		c.Set("request", "中间件")
		//这是关键之处，执行函数
		c.Next()
		status := c.Writer.Status()
		fmt.Println("中间件执行完毕", status)
		t2 := time.Since(t)
		//注意，打印的函数执行时间
		fmt.Println("time:", t2)
	}
}
```

## 使用中间件

中间件的使用可分为全局使用和局部使用。

- 全局使用，使用函数 use，即所有的请求都会通过中间件。

```go
  r := gin.Default()
  //定义全局middleware
  r.Use(NextMiddleWare())
  {
    r.GET("/index", func(c *gin.Context) {
      req, _ := c.Get("request")
      fmt.Println("request:", req)
      //延时3秒
      time.Sleep(3 * time.Second)
      // 页面接收
      c.JSON(200, gin.H{"request": req})
    })
  }
```

- 局部使用

```go
	group := r.Group("/v1")
	{
		group.GET("/index", NextMiddleWare(), func(c *gin.Context) {
			req, _ := c.Get("request")
			fmt.Println("request:", req)
			//延时3秒
			time.Sleep(3 * time.Second)
			// 页面接收
			c.JSON(200, gin.H{"request": req})
		})
		group.GET("/home", func(c *gin.Context) {
			req, _ := c.Get("request")
			fmt.Println("request:", req)
			//延时3秒
			time.Sleep(3 * time.Second)
			// 页面接收
			c.JSON(200, gin.H{"request": req})
		})
	}
```

```go
group := r.Group("/v1").Use(NextMiddleWare())
	{
		group.GET("/home1", func(c *gin.Context) {
			req, _ := c.Get("request")
			fmt.Println("request:", req)
			//延时3秒
			time.Sleep(3 * time.Second)
			// 页面接收
			c.JSON(200, gin.H{"request": req})
		})
	}
```

# cookie

在本节，结合中间件来实现 `Cookie` 的设置。访问`/login`设置`Cookie`,访问`/user/index`,需要通过检验`Cookie`的值是不是设置值，不存在或者不对将会调用函数`c.Abort()`舍弃该请求，返回错误信息。通过则会调用函数`c.Next()`执行函数。

```go
func AuthMiddleWare() gin.HandlerFunc {
	return func(c *gin.Context) {
		val, err := c.Cookie("cookie_key")
		if err == nil {
			fmt.Println(val)
			if val == "123" {
				c.Next()
				return
			}
		}
		// 返回错误
		c.JSON(http.StatusUnauthorized, gin.H{"error": err.Error()})
		// 若验证不通过，不再调用后续的函数处理
		c.Abort()
		return
	}
}
func main() {
	r := gin.Default()

	r.GET("/login", func(c *gin.Context) {
		// 给客户端设置cookie
		// maxAge int, 单位为秒
		// path,cookie所在目录
		// domain string,域名
		// secure 是否智能通过https访问
		// httpOnly bool 是否允许别人通过js获取自己的cookie
		c.SetCookie("cookie_key", "123", 360, "/", "localhost", false, true)
		c.JSON(http.StatusOK, gin.H{"meaasge": "login success!"})
	})
	user := r.Group("/user")
	{
		user.GET("/index", AuthMiddleWare(), func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{"data": "/user/index"})
		})
	}
	r.Run()
}
```

# 参数验证

```go
type Person struct {
	Name     string    `json:"name" form:"name" binding:"required",msg:"用户名必须传入"`
	Age      int       `json:"age" form:"age" binding:"required,gt=10"`
	Birthday time.Time `json:"birthday" form:"birthday" time_format:"2006-01-02" time_utc:"1"`
}

func main() {
	r := gin.Default()
	r.GET("/data", func(c *gin.Context) {
		var person Person
		if err := c.ShouldBind(&person); err != nil {
      //shouldBind()允许自己设置状态码
			c.String(500, err.Error())
			return
		}
		c.JSON(200, gin.H{"msg": fmt.Sprintf("%#v", person)})
	})
	r.Run()
}
```

## 自定义验证

对绑定解析到结构体上的参数，自定义验证功能如我们要对 name 字段做校验，要不能为空，并且不等于 admin ，类似这种需求，就无法使用现成的方法要我们自己验证方法才能实现 ,这里需要下载引入`gopkg.in/go-playground/validator.v8`

```go
type Person struct {
	Age int `form:"age" binding:"required,gt=10"`
	// 2、在参数 binding 上使用自定义的校验方法函数注册时候的名称
	Name    string `form:"name" binding:"NotNullAndAdmin"`
	Address string `form:"address" binding:"required"`
}

// 1、自定义的校验方法
func nameNotNullAndAdmin(v *validator.Validate, topStruct reflect.Value, currentStructOrField reflect.Value, field reflect.Value,
	fieldType reflect.Type, fieldKind reflect.Kind, param string) bool {
	if value, ok := field.Interface().(string); ok {
		// 字段不能为空，并且不等于 admin
		return value != "" && !("5lmh" == value)
	}
	return true
}

func main() {
	r := gin.Default()
	// 3、将我们自定义的校验方法注册到 validator中
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		// 这里的 key 和 fn 可以不一样最终在 struct 使用的是 key
		v.RegisterValidation("NotNullAndAdmin", nameNotNullAndAdmin)
	}
	r.GET("/data", func(c *gin.Context) {
		var person Person
		if e := c.ShouldBind(&person); e == nil {
			c.String(http.StatusOK, "%v", person)
		} else {
			c.String(http.StatusOK, "person bind err:%v", e.Error())
		}
	})
	r.Run()
}
```
