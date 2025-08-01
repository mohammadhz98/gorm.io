---
title: Context
layout: page
---

GORM's context support is a powerful feature that enhances the flexibility and control of database operations in Go applications. It allows for context management across different operational modes, timeout settings, and even integration into hooks/callbacks and middlewares. Let's delve into these various aspects:

## Single Session Mode

Single session mode is appropriate for executing individual operations. It ensures that the specific operation is executed within the context's scope, allowing for better control and monitoring.

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

## Continuous Session Mode

Continuous session mode is ideal for performing a series of related operations. It maintains the context across these operations, which is particularly useful in scenarios like transactions.

```go
tx := db.WithContext(ctx)
tx.First(&user, 1)
tx.Model(&user).Update("role", "admin")
```

## Context Timeout

Setting a timeout on the context can control the duration of long-running queries. This is crucial for maintaining performance and avoiding resource lock-ups in database interactions.

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

## Contexto en Hooks/Callbacks

The context can also be accessed within GORM's hooks/callbacks. This enables contextual information to be used during these lifecycle events. The context is accessible through the `Statement.Context` field:

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  ctx := tx.Statement.Context
  // ... use context
  return
}
```

## Integration with Chi Middleware

GORM's context support extends to web server middlewares, such as those in the Chi router. This allows setting a context with a timeout for all database operations within the scope of a web request.

```go
func SetDBMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    timeoutContext, _ := context.WithTimeout(context.Background(), time.Second)
    ctx := context.WithValue(r.Context(), "DB", db.WithContext(timeoutContext))
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}

// Router setup
r := chi.NewRouter()
r.Use(SetDBMiddleware)

// Route handlers
r.Get("/", func(w http.ResponseWriter, r *http.Request) {
  db, ok := r.Context().Value("DB").(*gorm.DB)
  // ... db operations
})

r.Get("/user", func(w http.ResponseWriter, r *http.Request) {
  db, ok := r.Context().Value("DB").(*gorm.DB)
  // ... db operations
})
```

**Note**: Setting the `Context` with `WithContext` is goroutine-safe. This ensures that database operations are safely managed across multiple goroutines. For more details, refer to the [Session documentation](session.html) in GORM.

## Logger Integration

GORM's logger also accepts `Context`, which can be used for log tracking and integrating with existing logging infrastructures.

Refer to [Logger documentation](logger.html) for more details.
