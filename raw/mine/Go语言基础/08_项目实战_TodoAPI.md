---
title: 项目实战 - Todo API
date: 2026-04-16
tags:
  - Go
  - Gin
  - 项目实战
  - CRUD
aliases:
  - TodoAPI
  - Todo项目
status: in-progress
---

# 项目实战：Todo API

> [!abstract] 项目目标
> 用 Go + Gin 从零构建一个完整的 Todo CRUD 接口，并配上**前端调用示例**。
> 学完本章，你应该能独立完成一个带增删改查的接口项目。

---

## 一、项目概览

### 技术栈

| 层级 | 技术 |
|------|------|
| 后端 | Go + Gin |
| 数据存储 | 内存（Map）|
| 部署 | 本地运行（可一键改为 MySQL）|

### API 设计

| 方法 | 路径 | 功能 |
|------|------|------|
| `GET` | `/api/todos` | 列表 |
| `POST` | `/api/todos` | 新建 |
| `PUT` | `/api/todos/:id` | 更新 |
| `DELETE` | `/api/todos/:id` | 删除 |

---

## 二、完整代码

### 项目结构

```
todo-api/
├── main.go        # 入口
├── handler/       # 路由处理
│   └── todo.go
├── model/         # 数据模型
│   └── todo.go
├── go.mod
└── frontend/     # 前端调用示例
    └── index.html
```

### go.mod

```bash
go mod init todo-api
go get github.com/gin-gonic/gin
```

### model/todo.go

```go
package model

type Todo struct {
    ID        int    `json:"id"`
    Title     string `json:"title" binding:"required"`
    Completed bool   `json:"completed"`
    CreatedAt string `json:"created_at"`
}
```

### handler/todo.go

```go
package handler

import (
    "net/http"
    "strconv"
    "sync"
    "time"

    "github.com/gin-gonic/gin"
    "todo-api/model"
)

var (
    todos  = make(map[int]model.Todo)
    mu     sync.RWMutex
    nextID = 1
)

// GET /api/todos
func List(c *gin.Context) {
    mu.RLock()
    defer mu.RUnlock()
    list := []model.Todo{}
    for _, t := range todos {
        list = append(list, t)
    }
    c.JSON(http.StatusOK, list)
}

// POST /api/todos
func Create(c *gin.Context) {
    var todo model.Todo
    if err := c.ShouldBindJSON(&todo); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "标题不能为空"})
        return
    }
    mu.Lock()
    todo.ID = nextID
    todo.CreatedAt = time.Now().Format("2006-01-02 15:04:05")
    todos[nextID] = todo
    nextID++
    mu.Unlock()
    c.JSON(http.StatusCreated, todo)
}

// PUT /api/todos/:id
func Update(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "无效的 ID"})
        return
    }
    var todo model.Todo
    if err := c.ShouldBindJSON(&todo); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    mu.Lock()
    defer mu.Unlock()
    if _, exists := todos[id]; !exists {
        c.JSON(http.StatusNotFound, gin.H{"error": "Todo 不存在"})
        return
    }
    todo.ID = id
    todos[id] = todo
    c.JSON(http.StatusOK, todo)
}

// DELETE /api/todos/:id
func Delete(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "无效的 ID"})
        return
    }
    mu.Lock()
    defer mu.Unlock()
    if _, exists := todos[id]; !exists {
        c.JSON(http.StatusNotFound, gin.H{"error": "Todo 不存在"})
        return
    }
    delete(todos, id)
    c.Status(http.StatusNoContent)
}
```

### main.go

```go
package main

import (
    "github.com/gin-gonic/gin"
    "todo-api/handler"
)

func main() {
    r := gin.Default()

    // 跨域中间件
    r.Use(func(c *gin.Context) {
        c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
        c.Writer.Header().Set("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS")
        c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type")
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }
        c.Next()
    })

    // 路由
    api := r.Group("/api")
    {
        api.GET("/todos", handler.List)
        api.POST("/todos", handler.Create)
        api.PUT("/todos/:id", handler.Update)
        api.DELETE("/todos/:id", handler.Delete)
    }

    r.Run(":8080")
}
```

---

## 三、测试接口

### 用 curl

```bash
# 创建
curl -X POST http://localhost:8080/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "学习 Go", "completed": false}'

# 列表
curl http://localhost:8080/api/todos

# 更新
curl -X PUT http://localhost:8080/api/todos/1 \
  -H "Content-Type: application/json" \
  -d '{"title": "学习 Gin", "completed": true}'

# 删除
curl -X DELETE http://localhost:8080/api/todos/1
```

### 用 Postman

1. 新建 Collection，Base URL 设为 `http://localhost:8080`
2. 按上表添加 4 个请求
3. Body 选 `raw` → `JSON` 类型

---

## 四、前端调用示例

### frontend/index.html

```html
<!DOCTYPE html>
<html lang="zh">
<head>
<meta charset="UTF-8">
<title>Todo 前端</title>
<style>
  body { font-family: sans-serif; max-width: 600px; margin: 40px auto; }
  .todo { padding: 10px; border-bottom: 1px solid #eee; display: flex; align-items: center; }
  .todo.done span { text-decoration: line-through; color: #999; }
  .todo button { margin-left: auto; }
</style>
</head>
<body>
  <h1>我的 Todo</h1>

  <div>
    <input id="titleInput" placeholder="输入待办事项" style="width: 70%; padding: 8px">
    <button onclick="createTodo()" style="padding: 8px 16px">添加</button>
  </div>

  <div id="todoList"></div>

  <script>
    const API = 'http://localhost:8080/api/todos'

    // 加载列表
    async function loadTodos() {
      const res = await fetch(API)
      const todos = await res.json()
      const list = document.getElementById('todoList')
      list.innerHTML = todos.map(t => `
        <div class="todo ${t.completed ? 'done' : ''}">
          <input type="checkbox" ${t.completed ? 'checked' : ''}
            onchange="toggleTodo(${t.id}, this.checked)">
          <span>${t.title}</span>
          <button onclick="deleteTodo(${t.id})">删除</button>
        </div>
      `).join('')
    }

    // 新建
    async function createTodo() {
      const title = document.getElementById('titleInput').value
      if (!title) return
      await fetch(API, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title, completed: false })
      })
      document.getElementById('titleInput').value = ''
      loadTodos()
    }

    // 切换完成状态
    async function toggleTodo(id, completed) {
      const res = await fetch(`${API}/${id}`)
      const todo = await res.json()
      await fetch(`${API}/${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title: todo.title, completed })
      })
      loadTodos()
    }

    // 删除
    async function deleteTodo(id) {
      await fetch(`${API}/${id}`, { method: 'DELETE' })
      loadTodos()
    }

    loadTodos()
  </script>
</body>
</html>
```

> [!success] 运行方式
> 1. 启动后端：`go run main.go`（端口 8080）
> 2. 双击打开 `frontend/index.html`（或在 VS Code 用 Live Server 插件）
> 3. 开始增删改查，前后端完整打通！

---

## 五、后续扩展方向

| 方向 | 说明 |
|------|------|
| 持久化存储 | 把 Map 换成 MySQL / PostgreSQL |
| 用户认证 | 加 JWT 中间件，实现多用户 Todo |
| 分类标签 | 给 Todo 加 `tags` 字段 |
| 部署上线 | Docker 打包，部署到云服务器 |

---

## ✅ 验收清单

- [ ] 后端 `go run main.go` 启动成功
- [ ] curl / Postman 调通所有 4 个接口
- [ ] 前端 HTML 页面能增删改查
- [ ] 能说出哪些地方用了 `sync.RWMutex`（并发安全）

---

## 📝 更新记录

- 2026-04-16：初始化

## 📁 关联笔记

- [[00_学习路线图]] — 返回学习路线总览
- [[07_Gin框架入门]] — Gin 框架基础
- [[09_项目实战_短链接服务]] — 下一个项目：短链接服务
