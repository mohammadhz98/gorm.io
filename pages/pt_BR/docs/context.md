---
title: Contexto
layout: page
---

GORM's context support is a powerful feature that enhances the flexibility and control of database operations in Go applications. Isto possibilita o gerenciamento de contexto entre diferentes modos operacionais como configuração de timeout, integração entre hooks/callbacks e middlewares. Vamos nos aprofundar nestes diferentes aspectos:

## Single Session Mode

Modo de sessão única é apropriado para executar operações simples. Isto garante que a operação específica seja executada no escopo do contexto, permitindo melhor controle e monitoramento.

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

## Modo sessão continuada

O modo sessão continuada é ideal para execução de operações em série. O contexto é mantido entre as operações e é particularmente útil em cenários como transações.

```go
tx := db.WithContext(ctx)
tx.First(&user, 1)
tx.Model(&user).Update("role", "admin")
```

## Timeout de contexto

Setting a timeout on the context can control the duration of long-running queries. Isto é crucial para manter a desempenho e evitar o bloqueio de recursos em iterações com o banco de dados.

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

## Contexto em Hooks/Callbacks

O contexto também pode ser acessado por hooks/callbacks no GORM. Isto habilita informações contextuais para serem utilizadas durante o ciclo de vida dos eventos. The context is accessible through the `Statement.Context` field:

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  ctx := tx.Statement.Context
  // ... uso do contexto
  return
}
```

## Integração com Chi Middleware

O contexto do GORM suporta extensão para middleware de servidores web, como este no roteador do Chi. Assim permitindo a configuração de contexto com timeout para todas as operações de banco de dados no escopo do request.

```go
func SetDBMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    timeoutContext, _ := context.WithTimeout(context.Background(), time.Second)
    ctx := context.WithValue(r.Context(), "DB", db.WithContext(timeoutContext))
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}

// Configuração do roteador
r := chi.NewRouter()
r.Use(SetDBMiddleware)

// Configurações de rotas
r.Get("/", func(w http.ResponseWriter, r *http.Request) {
  db, ok := r.Context().Value("DB").(*gorm.DB)
  // ... Operações de BD
})

r.Get("/user", func(w http.ResponseWriter, r *http.Request) {
  db, ok := r.Context().Value("DB").(*gorm.DB)
  // ... Operações de BD
})
```

**Nota**: A configuração do `Context` com `WithContext` é goroutine-safe. Isto garante que as operações de bancos de dados são gerenciadas entre múltiplas goroutines. Para mais detalhes, consulte a [documentação de Session](session.html) em GORM.

## Integração com Logger

O GORM logger também aceita `Context`, que pode ser utilizado para log tracking e integração com a infraestrutura de log existente.

Consulte a [documentação do Logger](logger.html) para mais detalhes.
