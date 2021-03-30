---
title: 一个用 golang 开发的轻量restfull框架
date: 2021-3-30 08:40:11
tags: 
    - blog
    - 框架
    - go
---

## 先把项目地址给大家
* https://github.com/tonny-zhang/cotton
* https://gitee.com/tonnyzhang/cotton
功能还在不断增加和完善中，希望大家多多支持
---
## 初衷
在`golang`的学习及工作使用中，会经常遇到提供http服务的场景，这时有两个选择：自己使用`http`原生包去做（适合简单的api)；使用第三方框架；我本身喜欢“重复造车轮”，这样使用自己开发的框架时遇到问题也可以很快的解决，而且也可以根据自身的业务特点进行快速适配。

当前有好多web框架，性能和功能也都各有不同。抱着学习的心态去从头做了一个轻量级的restfull框架，`cotton`就在这样的背景下诞生了。

`cotton`意指“棉花”，我也希望这个框架是轻量好用的。 

## 要支持的特性
* 速度快
* 支持 `restfull` 格式参数
* 支持中间件
* 支持分组
* 自定义日志
* 自定义 `panic`
* 自定义 `NotFound`
* 分组自定义 `NotFound`
* 静态文件
* 模板
* `post`参数相关，及文件上传相关

## 可用的http框架
* [httprouter](https://github.com/julienschmidt/httprouter)
* [Gin](https://github.com/gin-gonic/gin) 在 httprouter基础上构建
* gin
* echo
* beego
## 开发中遇到的问题
主要对标的是 `httprouter`
### 路由结构存储
最开始时使用全路径正则实现，虽然功能都已经实现，但性能和 `httprouter` 相差太多，做 `Benchmark` 时不是一个数量级
#### 全路径正则实现
```
    /user/:id/:name      =>   regexp.MustCompile("/user/(:\w+)/(:\w+)")
```
#### Benchmark 结果
```
cotton-bench tonny$ go test -bench=.
GithubAPI Routes: 203
           cottonRouter:     93080 bytes
             HttpRouter:     35768 bytes
goos: darwin
goarch: amd64
pkg: cottonbench
cpu: Intel(R) Core(TM) i5-8279U CPU @ 2.40GHz
BenchmarkCottonRouterWithGithubAPI-8                1543            792932 ns/op  598146 B/op        8100 allocs/op
BenchmarkHttpRouterWithGithubAPI-8                 25250             46397 ns/op   20320 B/op         334 allocs/op

PASS
ok      cottonbench     10.259s
```
#### 思考
从结果可以`ns/op`和`allocs/op`看出每次单次内存分配和运算所用的时间比较多，细分析每次请求后此方案只能按顺序从所有的已经路由正则里去匹配，在大量路由面前性能会直线下降。

知道了问题出在哪里就知道怎么去优化了

## 性能优化之路
### 1. 内存逃逸
内存逃逸相关的概念这里就不多说了，就说下我使用的方案
####1.1 路由Handle时使用的`Context` 使用了 `sync.Pool`
```go
var ctxPool sync.Pool

func init() {
	ctxPool.New = func() interface{} {
		return &Context{}
	}
}

func newContext(w http.ResponseWriter, r *http.Request, router *Router) *Context {
	// use sync.Pool
	ctx := ctxPool.Get().(*Context)

	// reset all property
	ctx.Request = r
	ctx.Response = &resWriter{
		ResponseWriter: w,
		statusCode:     http.StatusOK,
	}
	ctx.router = router
	ctx.indexAbort = -1
	ctx.index = -1
	ctx.handlers = ctx.handlers[0:0]
	ctx.paramCache = nil
	ctx.queryCache = nil

	ctxPool.Put(ctx)
	return ctx
}
```
####1.2 路由匹配到的`restfull`参数使用`sync.Pool`
```go
var paramsPool sync.Pool
func init() {
	paramsPool.New = func() interface{} {
		return make(map[string]string)
	}
}

result.params = paramsPool.Get().(map[string]string)

paramsPool.Put(result.params)
```
### 2. 数据结构
仔细研读了`httprouter`的源码，发现它使用的是[前缀树或字典树](https://baike.baidu.com/item/%E5%AD%97%E5%85%B8%E6%A0%91/9825209?fromtitle=%E5%89%8D%E7%BC%80%E6%A0%91&fromid=2501595&fr=aladdin)，一种节省存储但查询效率很高的数据结构，但其实现的算法有些深奥，自己决定使用自己的数据结构，我使用的是按路径分割存储，很直观的树存储。
```
/search/
/support/
/blog/:post/
/about-us/
/about-us/team/
/contact/
```
#### 2.1 前缀树
```
Priority   Path             Handle
9          \                *<1>
3          ├s               nil
2          |├earch\         *<2>
1          |└upport\        *<3>
2          ├blog\           *<4>
1          |    └:post      nil
1          |         └\     *<5>
2          ├about-us\       *<6>
1          |        └team\  *<7>
1          └contact\        *<8>
```

#### 2.2 直观树
```
Deep        Path                       Handle
0           /                           nil
1             |--search                 nil
2                 |--/                  *1
1             |--support                nil
2                 |--/                  *2
1             |--blog                   nil
2                 |--/                  *3
2                 |--:post              *4
1             |--about-us               nil
2                 |--/                  *5
2                 |--team               nil
3                     |--/              *6
1             |--contact                nil
2                 |--/                  *7
```
### 3. 最终效果
```
GithubAPI Routes: 205
   cottonRouter:     95352 bytes
     HttpRouter:     36016 bytes
goos: darwin
goarch: amd64
pkg: cottonbench
cpu: Intel(R) Core(TM) i5-8279U CPU @ 2.40GHz
BenchmarkHttpRouterWithGithubAPI-8                 39334             31510 ns/op           13856 B/op        169 allocs/op
BenchmarkCottonRouterWithGithubAPI-8               34222             35289 ns/op               0 B/op          0 allocs/op

PASS
ok      cottonbench     11.384s
```
可以看出和 httprouter的性能差不多，性能优化算是很成功的。Benchmark 的代码参考 [Github](https://github.com/tonny-zhang/cotton-bench) 或 [Gitee](https://gitee.com/tonnyzhang/cotton-bench)

## 如何使用
```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"path/filepath"

	"net/http"

	"github.com/tonny-zhang/cotton"
)

func main() {

	r := cotton.NewRouter()

	// writer logger to file
	// f, e := os.OpenFile("1.log", os.O_WRONLY|os.O_CREATE|os.O_APPEND, 0600)
	// fmt.Println(f, e)
	// r.Use(cotton.Logger(), cotton.LoggerWidthConf(cotton.LoggerConf{
	// 	Writer: f,
	// }))

	// r.Use(cotton.Recover())
	r.Use(cotton.Logger())
	// r.Use(func(ctx *cotton.Context) {
	// 	fmt.Println("first")
	// 	ctx.Abort()
	// })
	r.Use(cotton.RecoverWithWriter(nil, func(ctx *cotton.Context, err interface{}) {
		strErr := ""
		switch err.(type) {
		case string:
			strErr = err.(string)
		case error:
			strErr = err.(error).Error()
		default:
			if b, err := json.Marshal(err); err == nil {
				strErr = string(b)
			}
		}
		ctx.String(http.StatusInternalServerError, "[500 error]"+strErr)
	}))

	dir, _ := os.Getwd()
	r.Group("/static/", cotton.LoggerWidthConf(cotton.LoggerConf{
		Writer: os.Stdout,
		Formatter: func(param cotton.LoggerFormatterParam, ctx *cotton.Context) string {
			return fmt.Sprintf("[INFO-STATIC] %v\t %d %s\n",
				param.TimeStamp.Format("2006/01/02 15:04:05"),
				param.StatusCode,
				filepath.Join(dir, ctx.Param("file")),
			)
		},
	})).Get("/*file", func(ctx *cotton.Context) {
		// file := filepath.Join(dir, ctx.Param("file"))

		// http.ServeFile(ctx.Response, ctx.Request, file)
		// // ctx.Response.GetStatusCode() for log
		// fmt.Println(ctx.Response.GetStatusCode(), file)

		http.StripPrefix("/static/", http.FileServer(http.Dir(dir))).ServeHTTP(ctx.Response, ctx.Request)
		// http.StripPrefix("", http.FileServer(nil)).ServeHTTP()
	})
	gs := r.Group("/s/")
	gs.StaticFile("/", dir, false)
	r.StaticFile("/m/", dir, true)
	r.Get("/panic", func(ctx *cotton.Context) {
		// i := 0
		// fmt.Println(1 / i)
		panic([]int{1, 2})
	})
	r.Get("/hello/", func(ctx *cotton.Context) {
		ctx.String(http.StatusOK, "hello get2")
	})
	// r.Use(cotton.LoggerWidthConf(cotton.LoggerConf{
	// 	Formatter: func(param cotton.LoggerFormatterParam) string {
	// 		return fmt.Sprintf("[info] %s %s %s\t%d %s\n",
	// 			utils.TimeFormat(param.TimeStamp),
	// 			param.ClientIP, param.Method, param.StatusCode,
	// 			param.Path,
	// 		)

	// 	},
	// }))
	r.Get("/user/", func(ctx *cotton.Context) {
		ctx.String(http.StatusOK, "/user")
	})
	r.Get("/user/:name", func(ctx *cotton.Context) {
		ctx.String(http.StatusOK, "user name = "+ctx.Param("name"))
	})
	r.Get("/user/:name/:id", func(ctx *cotton.Context) {
		ctx.String(http.StatusOK, "user id = "+ctx.Param("id")+" name = "+ctx.Param("name"))
	})
	r.Get("/user/:name/:id/one", func(ctx *cotton.Context) {
		ctx.String(http.StatusOK, "one user id = "+ctx.Param("id")+" name = "+ctx.Param("name"))
	})
	r.Get("/user/:name/:id/two", func(ctx *cotton.Context) {
		ctx.String(http.StatusOK, "two user id = "+ctx.Param("id")+" name = "+ctx.Param("name"))
	})
	r.Get("/info/*file", func(ctx *cotton.Context) {
		ctx.String(http.StatusOK, "info file = "+ctx.Param("file"))
	})
	r.Post("/user/:id", func(ctx *cotton.Context) {
		ctx.String(http.StatusOK, "hello post "+ctx.Param("id"))
	})

	g1 := r.Group("/v1/", func(ctx *cotton.Context) {
		fmt.Println("g1 middleware")
	})
	g1.NotFound(func(ctx *cotton.Context) {
		ctx.String(http.StatusNotFound, "page ["+ctx.Request.RequestURI+"] not found")
	})
	{
		g1.Get("/a", func(ctx *cotton.Context) {
			ctx.String(http.StatusOK, "g1 a")
		})
		g1.Get("/info", func(ctx *cotton.Context) {
			ctx.JSON(http.StatusOK, cotton.M{
				"message": "from g1 info",
			})
		})
	}
	g2 := r.Group("/v2/")
	{
		g2.Get("/a", func(ctx *cotton.Context) {
			ctx.String(http.StatusOK, "g2 a "+ctx.Param("method"))
		})
		g2.Get("/b", func(ctx *cotton.Context) {
			ctx.String(http.StatusOK, "g2 b "+ctx.Param("method"))
		})
		g2.Get("/c/:id", func(ctx *cotton.Context) {
			ctx.String(http.StatusOK, "g2 c "+ctx.Param("method")+" id = "+ctx.Param("id"))
		})
	}

	g3 := r.Group("/v3/:method/")
	g3.Use(func(ctx *cotton.Context) {
		if ctx.Param("method") != "test" {
			ctx.Abort()
			ctx.String(http.StatusBadRequest, "no method test")
		}
	})
	{
		g3.Get("/a", func(ctx *cotton.Context) {
			ctx.String(http.StatusOK, "g3 a "+ctx.Param("method"))
		})
		g3.Get("/b", func(ctx *cotton.Context) {
			ctx.String(http.StatusOK, "g3 b "+ctx.Param("method"))
		})
		g3.Get("/c/:id", func(ctx *cotton.Context) {
			ctx.String(http.StatusOK, "g3 c "+ctx.Param("method")+" id = "+ctx.Param("id"))
		})
	}

	r.Group("/nohandle")
	r.Get("/redirect", func(ctx *cotton.Context) {
		urlto := ctx.GetDefaultQuery("url", "https://www.baidu.com")
		ctx.Redirect(302, urlto)
	})

	// r.PrintTree(http.MethodGet)
	r.Run(":5000")
}

```
