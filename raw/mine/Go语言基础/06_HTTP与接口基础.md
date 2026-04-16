---
title: Go HTTP 与接口基础
date: 2026-04-16
tags:
  - Go
  - HTTP
  - 接口
  - 前后端交互
aliases:
  - Go HTTP
  - Go接口
status: in-progress
---

# Go HTTP 与接口基础

> [!success] 里程碑章节
> 读完这一章，你应该能：
> - 用 Go 写一个可以被前端调用的 HTTP 接口
> - 理解 HTTP 请求的解析和响应的构造
> - 完成前后端打通的第一个最小闭环

---

## 一、前后端通信的本质

作为前端开发者，你熟悉这个流程：

```
前端（浏览器/Node）  →  HTTP 请求（JSON）  →  后端服务
前端（浏览器/Node）  ←  HTTP 响应（JSON）  ←  后端服务
```

Go 的 `net/http` 包就是用来写"后端服务"的。

---

## 二、最简单的 HTTP 接口

### 1. 启动一个 HTTP 服务

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

func main() {
    // 注册路由和处理函数
    http.HandleFunc("/hello", helloHandler)

    // 启动服务，监听 8080 端口
    fmt.Println("服务启动在 :8080")
    http.ListenAndServe(":8080", nil)
}

// GET /hello
func helloHandler(w http.ResponseWriter, r *http.Request) {
    // 1. 只处理 GET 方法
    if r.Method != http.MethodGet {
        http.Error(w, "只支持 GET", http.StatusMethodNotAllowed)
        return
    }

    // 2. 设置响应 Header
    w.Header().Set("Content-Type", "application/json")

    // 3. 返回 JSON 响应
    json.NewEncoder(w).Encode(map[string]string{
        "message": "Hello from Go!",
    })
}
```

### 2. 用 curl 测试

```bash
# 启动服务后，新开一个终端
curl http://localhost:8080/hello

# 返回：
# {"message":"Hello from Go!"}
```

### 3. 用前端 fetch 测试

```javascript
// 在前端项目中运行
const res = await fetch('http://localhost:8080/hello')
const data = await res.json()
console.log(data.message) // Hello from Go!
```

> [!success] 前后端打通！
> 这就是最小闭环——Go 服务 ↔ HTTP ↔ 前端。
> **现在你已经有了一个可以工作的后端接口。**

---

## 三、解析请求（后端读前端数据）

### GET Query 参数

```
GET /search?name=张三&age=25
```

```go
func searchHandler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    age := r.URL.Query().Get("age")
    fmt.Fprintf(w, "查询: name=%s, age=%s", name, age)
}
```

### Path 参数（URL 路径）

```
GET /user/123
```

```go
// 标准库不支持路径参数，需要用第三方的路由库（如 Gin）
// 见 [[07_Gin框架入门]]
```

### POST JSON Body

```go
// 请求体结构体
type CreateUserRequest struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
    // 1. 只允许 POST
    if r.Method != http.MethodPost {
        http.Error(w, "只支持 POST", http.StatusMethodNotAllowed)
        return
    }

    // 2. 解析 JSON Body
    var req CreateUserRequest
    err := json.NewDecoder(r.Body).Decode(&req)
    if err != nil {
        http.Error(w, `{"error": "JSON 格式错误"}`, http.StatusBadRequest)
        return
    }

    // 3. 业务逻辑（这里简单返回创建后的数据）
    json.NewEncoder(w).Encode(map[string]interface{}{
        "id":   1,
        "name": req.Name,
        "age":  req.Age,
    })
}
```

> [!tip] 用 Postman/curl 发送 POST 请求
> ```bash
> curl -X POST http://localhost:8080/user \
>   -H "Content-Type: application/json" \
>   -d '{"name": "张三", "age": 25}'
> ```

---

## 四、构造响应

### 返回 JSON（最常用）

```go
// 方式一：map（简单响应）
json.NewEncoder(w).Encode(map[string]string{
    "message": "success",
})

// 方式二：结构体（复杂响应）
type Response struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data"`
}

resp := Response{Code: 0, Message: "成功", Data: user}
json.NewEncoder(w).Encode(resp)
```

### 常见 HTTP 状态码

| 状态码 | 含义 | 使用场景 |
|--------|------|---------|
| `200 OK` | 成功 | 默认成功响应 |
| `201 Created` | 创建成功 | POST 新建资源 |
| `400 Bad Request` | 请求格式错误 | JSON 解析失败 |
| `404 Not Found` | 资源不存在 | GET 找不到 |
| `500 Internal Server Error` | 服务器内部错误 | 业务逻辑报错 |

```go
// 返回 201
w.WriteHeader(http.StatusCreated)
json.NewEncoder(w).Encode(newUser)

// 返回 400
http.Error(w, `{"error": "参数缺失"}`, http.StatusBadRequest)
```

---

## 五、中间件（日志示例）

中间件在请求到达 handler **之前/之后**执行，类似前端的请求/响应拦截器：

```go
// 日志中间件
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        fmt.Printf("[%s] %s %s\n", r.Method, r.URL.Path, r.RemoteAddr)

        // 执行下一个 Handler
        next.ServeHTTP(w, r)

        fmt.Printf("耗时: %v\n", time.Since(start))
    })
}

// 使用中间件
http.Handle("/hello", loggingMiddleware(http.HandlerFunc(helloHandler)))
```

> [!tip] Gin 框架让中间件更简单
> 标准库的中间件写法比较繁琐，用 Gin 框架几行代码搞定。
> 详见 [[07_Gin框架入门]]。

---

## 六、完整示例：Todo 服务

一个包含增删改查的完整接口集：

```go
package main

import (
    "encoding/json"
    "net/http"
    "sync"
    "time"
)

type Todo struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Completed bool      `json:"completed"`
    CreatedAt time.Time `json:"created_at"`
}

var (
    todos  = make(map[int]Todo)
    nextID = 1
    mu     sync.Mutex
)

func main() {
    http.HandleFunc("/todos", handleTodos)
    http.HandleFunc("/todos/", handleTodoByID)
    http.ListenAndServe(":8080", nil)
}

// GET /todos  | POST /todos
func handleTodos(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")

    switch r.Method {
    case http.MethodGet:
        list := []Todo{}
        for _, t := range todos {
            list = append(list, t)
        }
        json.NewEncoder(w).Encode(list)

    case http.MethodPost:
        var todo Todo
        if err := json.NewDecoder(r.Body).Decode(&todo); err != nil {
            http.Error(w, `{"error": "无效的 JSON"}`, http.StatusBadRequest)
            return
        }
        mu.Lock()
        todo.ID = nextID
        todo.CreatedAt = time.Now()
        todos[nextID] = todo
        nextID++
        mu.Unlock()
        json.NewEncoder(w).Encode(todo)

    default:
        http.Error(w, "不支持该方法", http.StatusMethodNotAllowed)
    }
}

// GET/PUT/DELETE /todos/{id}
func handleTodoByID(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    // ...
}
```

> [!success] 项目验收
> 这个 Todo 服务完整演示了：
> - HTTP 方法区分（GET/POST/PUT/DELETE）
> - JSON 请求解析与响应构造
> - 内存数据存储
> - 完整代码见 [[08_项目实战_TodoAPI]]

---

## ✅ 验收清单

- [ ] 能用 Go 写一个 GET 接口并用 curl / Postman 调通
- [ ] 能解析 POST 请求中的 JSON Body
- [ ] 能返回正确的 HTTP 状态码
- [ ] 能用前端 fetch 调通 Go 接口

---

## 📝 更新记录

- 2026-04-16：初始化

## 📁 关联笔记

- [[00_学习路线图]] — 返回学习路线总览
- [[05_结构体与接口]] — 结构体与接口基础
- [[07_Gin框架入门]] — 下一章：Gin 框架（让接口写起来更优雅）
- [[08_项目实战_TodoAPI]] — Todo API 完整项目
