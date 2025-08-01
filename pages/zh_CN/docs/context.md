---
title: 上下文
layout: page
---

GORM's context support is a powerful feature that enhances the flexibility and control of database operations in Go applications. 它允许在不同的操作模式、超时设置以及甚至集成到钩子/回调和中间件中进行上下文管理。 让我们从不同方面深入了解：

## Single Session Mode

单会话模式适用于执行单独的操作。 它确保特定操作在上下文的范围内执行，从而提供更好的控制和监控。

### Generics API

With the Generics API, context is passed directly as the first parameter to the operation methods:

```go
users, err := gorm.G[User](db).Find(ctx)
```

### Traditional API

With the Traditional API, context is passed using the `WithContext` method:

```go
db.WithContext(ctx).Find(&users)
```

## 持续会话模式

持续会话模式非常适合执行一系列相关的操作。 它在这些操作之间保持上下文，对于事务等场景特别有用。

```go
tx := db.WithContext(ctx)
tx.First(&user, 1)
tx.Model(&user).Update("role", "admin")
```

## Context超时

Setting a timeout on the context can control the duration of long-running queries. 这对于保持性能和避免数据库交互中的资源锁定至关重要。

### Generics API

With the Generics API, you pass the timeout context directly to the operation:

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

users, err := gorm.G[User](db).Find(ctx)
```

### Traditional API

With the Traditional API, you pass the timeout context to `WithContext`:

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

db.WithContext(ctx).Find(&users)
```

## Hooks/Callbacks 中的 Context

Context也可以在 GORM 的Hook/Callbacks中访问。 这使得在这些生命周期事件中可以使用上下文信息。 The context is accessible through the `Statement.Context` field:

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  ctx := tx.Statement.Context
  // ...使用ctx做一些你想做的事
  return
}
```

## 在Chi 中间件的集成

GORM 的Context支持扩展到 web 服务器中间件，例如 Chi 路由器中的中间件。 这允许为 web 请求范围内的所有数据库操作设置一个带有超时的上下文。

```go
func SetDBMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    timeoutContext, _ := context.WithTimeout(context.Background(), time.Second)
    ctx := context.WithValue(r.Context(), "DB", db.WithContext(timeoutContext))
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}

// 路由初始化
r := chi.NewRouter()
r.Use(SetDBMiddleware)

// 路由接口
r.Get("/", func(w http.ResponseWriter, r *http.Request) {
  db, ok := r.Context().Value("DB").(*gorm.DB)
  // ... db操作
})

r.Get("/user", func(w http.ResponseWriter, r *http.Request) {
  db, ok := r.Context().Value("DB").(*gorm.DB)
  // ... db操作
})
```

**注意**：使用 `WithContext` 设置`Context`是 goroutine 安全的。 这确保了数据库操作在多个 goroutine 中被安全地管理。 如需更多详细信息，请参考 GORM 的[Session文档](session.html)。

## 日志记录器集成

GORM 的日志记录器也接受`Context`，这可以用于日志跟踪，并与现有的日志基础设施集成。

有关更多详细信息，请参考 [Logger](logger.html)文档
