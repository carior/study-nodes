---
title: Gin 框架入门
date: 2026-04-16
tags:
  - Go
  - Gin
  - Web框架
aliases:
  - Gin框架
  - Gin教程
status: in-progress
---

# Gin 框架入门

> [!info] 为什么要学 Gin？
> Go 标准库的 `net/http` 写接口够用，但项目大了以后：
> - 路由用字符串匹配，手写 Path 参数解析
> - 中间件写法繁琐
> - 没有参数校验、没有分组、没有日志
>
> **Gin** 是 Go 最流行的 Web 框架，用它写接口更简洁、更工程化。

---

## 一、安装 Gin

```bash
# 进入项目目录
cd ~/go/src/hello

# 安装 Gin
go get github.com/gin-gonic/gin

# 安装验证
go mod tidy
```

> [!tip] Gin 安装问题
> 如果 `go get` 失败，设置 GOPROXY：
> ```bash
> go env -w GOPROXY=https://goproxy.cn,direct
> ```

---

## 二、第一个 Gin 项目

### 目录结构

```
hello/
├── main.go
├── go.mod
└── go.sum
```

### main.go

```go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    // 创建 Gin 引擎
    r := gin.Default()

    // 注册路由
    r.GET("/hello", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "message": "Hello from Gin!",
        })
    })

    // 启动服务
    r.Run(":8080")
}
```

```bash
go run main.go
curl http://localhost:8080/hello
# {"message":"Hello from Gin!"}
```

---

## 三、路由详解

### 基本路由

```go
r.GET("/path", handler)           // GET
r.POST("/path", handler)          // POST
r.PUT("/path", handler)          // PUT
r.DELETE("/path", handler)        // DELETE
r.PATCH("/path", handler)         // PATCH
r.OPTIONS("/path", handler)       // OPTIONS
r.HEAD("/path", handler)          // HEAD
```

### Path 参数（路径变量）

```go
// GET /user/123
r.GET("/user/:id", func(c *gin.Context) {
    id := c.Param("id")  // "123"
    c.JSON(http.StatusOK, gin.H{"id": id})
})

// GET /user/123/profile
r.GET("/user/:id/profile", func(c *gin.Context) {
    id := c.Param("id")  // "123"
    c.JSON(http.StatusOK, gin.H{"id": id})
})
```

> [!tip] `:id` vs `*id`
> - `:id` — 必填路径参数
> - `*id` — 可选路径参数（匹配 `/user` 和 `/user/123`）

### Query 参数

```go
// GET /search?name=张三&age=25
r.GET("/search", func(c *gin.Context) {
    name := c.Query("name")         // "张三"，不存在返回 ""
    age := c.DefaultQuery("age", "0") // 有默认值
    c.JSON(http.StatusOK, gin.H{"name": name, "age": age})
})
```

### 请求体解析（JSON）

```go
// 请求结构体
type CreateUserRequest struct {
    Name string `json:"name" binding:"required"`  // binding:required 必填校验
    Age  int    `json:"age" binding:"gte=0"`      // gte=0 最小值校验
}

r.POST("/user", func(c *gin.Context) {
    var req CreateUserRequest
    // 自动解析 JSON、自动校验参数
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, req)
})
```

> [!success] binding 标签自动校验
> 不用手写 `if req.Name == ""`，Gin 自动根据 `binding` 标签校验，参数不对直接返回错误。

---

## 四、中间件

### 全局中间件

```go
r := gin.Default()  // 默认包含 Logger + Recovery 中间件

// 或自定义全局中间件
r.Use(func(c *gin.Context) {
    fmt.Println("请求来了:", c.Request.URL.Path)
    c.Next()  // 继续执行后续 Handler
    fmt.Println("请求结束")
})
```

### 路由组中间件

```go
// 需要登录的路由组
auth := r.Group("/api")
auth.Use(middleware.AuthRequired()) {
    auth.GET("/profile", getProfile)
    auth.PUT("/profile", updateProfile)
}
```

### 常见内置中间件

| 中间件 | 作用 |
|--------|------|
| `gin.Logger()` | 打印请求日志 |
| `gin.Recovery()` | panic 时返回 500，避免服务崩溃 |
| `gin.CORS()` | 处理跨域（需要 `github.com/gin-contrib/cors`）|

---

## 五、分组路由

```go
// API v1 版本分组
v1 := r.Group("/api/v1")
{
    v1.GET("/user/:id", getUser)
    v1.POST("/user", createUser)
    v1.PUT("/user/:id", updateUser)
    v1.DELETE("/user/:id", deleteUser)
}

// 生成的路由：
// GET    /api/v1/user/:id
// POST   /api/v1/user
// PUT    /api/v1/user/:id
// DELETE /api/v1/user/:id
```

> [!tip] 路由分组的好处
> - URL 统一前缀，便于 Nginx 代理和版本管理
> - 可以对整个分组统一加中间件（如认证、限流）

---

## 六、统一响应格式

约定一个统一的响应结构体：

```go
// 统一响应结构
type Response struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`  // omitempty: 空则不输出
}

// 辅助函数
func Success(c *gin.Context, data interface{}) {
    c.JSON(http.StatusOK, Response{Code: 0, Message: "success", Data: data})
}

func Fail(c *gin.Context, code int, message string) {
    c.JSON(http.StatusOK, Response{Code: code, Message: message})
}

func FailError(c *gin.Context, err error) {
    Fail(c, 1, err.Error())
}

// 使用
r.GET("/user/:id", func(c *gin.Context) {
    user, err := getUserByID(c.Param("id"))
    if err != nil {
        FailError(c, err)
        return
    }
    Success(c, user)
})
```

---

## 七、CORS 跨域处理

前端调接口经常遇到跨域，用 Gin 中间件解决：

```go
import "github.com/gin-contrib/cors"

func main() {
    r := gin.Default()
    r.Use(cors.New(cors.Config{
        AllowOrigins:     []string{"http://localhost:3000"},  // 前端地址
        AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowHeaders:     []string{"Content-Type", "Authorization"},
        AllowCredentials: true,
    }))
    // ...
}
```

---

## 八、用 Gin 重构 Todo 服务

用 Gin 把 [[06_HTTP与接口基础]] 的 Todo 服务重写一遍：

```go
package main

import (
    "net/http"
    "strconv"
    "sync"

    "github.com/gin-gonic/gin"
)

type Todo struct {
    ID    int    `json:"id"`
    Title string `json:"title" binding:"required"`
    Done  bool   `json:"done"`
}

var (
    todos = make(map[int]Todo)
    mu    sync.RWMutex
    nextID = 1
)

func main() {
    r := gin.Default()
    r.Use(corsMiddleware())

    // 路由
    r.GET("/todos", listTodos)
    r.POST("/todos", createTodo)
    r.PUT("/todos/:id", updateTodo)
    r.DELETE("/todos/:id", deleteTodo)

    r.Run(":8080")
}

func corsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
        c.Next()
    }
}

func listTodos(c *gin.Context) {
    mu.RLock()
    defer mu.RUnlock()
    list := []Todo{}
    for _, t := range todos {
        list = append(list, t)
    }
    c.JSON(http.StatusOK, list)
}

func createTodo(c *gin.Context) {
    var todo Todo
    if err := c.ShouldBindJSON(&todo); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    mu.Lock()
    todo.ID = nextID
    todos[nextID] = todo
    nextID++
    mu.Unlock()
    c.JSON(http.StatusCreated, todo)
}

func updateTodo(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))
    var todo Todo
    if err := c.ShouldBindJSON(&todo); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    mu.Lock()
    todo.ID = id
    todos[id] = todo
    mu.Unlock()
    c.JSON(http.StatusOK, todo)
}

func deleteTodo(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))
    mu.Lock()
    delete(todos, id)
    mu.Unlock()
    c.Status(http.StatusNoContent)
}
```

---

## ✅ 验收清单

- [ ] 能用 Gin 写一个带 Path 参数的 GET 接口
- [ ] 能用 `ShouldBindJSON` 解析 POST 请求
- [ ] 能用路由分组组织 API
- [ ] 能用 `corsMiddleware` 处理跨域

---

## 📝 更新记录

- 2026-04-16：初始化

## 📁 关联笔记

- [[00_学习路线图]] — 返回学习路线总览
- [[06_HTTP与接口基础]] — 标准库 HTTP 基础
- [[08_项目实战_TodoAPI]] — 下一章：完整 Todo API 项目
