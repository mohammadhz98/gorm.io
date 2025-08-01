---
title: Tratamento de Erro
layout: page
---

Manipular os erros corretamente é um pilar no desenvolvimento de aplicações robustas em GO, particularmente quando estamos interagindo com banco de dados usando GORM. GORM's approach to error handling requires a nuanced understanding based on the API style you're using.

## Manipulação básica de erros

### Generics API

With the Generics API, errors are returned directly from the operation methods, following Go's standard error handling pattern:

```go
ctx := context.Background()

// Error handling with direct return values
user, err := gorm.G[User](db).Where("name = ?", "jinzhu").First(ctx)
if err != nil {
  // Handle error...
}

// For operations that don't return a result
err := gorm.G[User](db).Where("id = ?", 1).Delete(ctx)
if err != nil {
  // Handle error...
}
```

### Traditional API

With the Traditional API, GORM integrates error handling into its chainable method syntax. A instância `*gorm.DB` contém o campo  `Error`, que é definido quando um erro acontece. Uma prática comum é verificar esse campo após executar uma operação no banco de dados, especialmente após [Finisher Methods](method_chaining.html#finisher_method).

Depois de uma cadeia de métodos, é crucial verificar o campo `Error`:

```go
if err := db.Where("name = ?", "jinzhu").First(&user).Error; err != nil {
  // Manipular erro...
}
```

Ou alternativamente:

```go
if result := db.Where("name = ?", "jinzhu").First(&user); result.Error != nil {
  // manipular erro...
}
```

## `ErrRecordNotFound`

GORM retorna `ErrRecordNotFound` quando nenhum registro é encontrado quando métodos como `First`, `Last`, `Take` são utilizados.

### Generics API

```go
ctx := context.Background()

user, err := gorm.G[User](db).First(ctx)
if errors.Is(err, gorm.ErrRecordNotFound) {
  // Handle record not found error...
}
```

### Traditional API

```go
err := db.First(&user, 100).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
  // Manipular o erro de registro não encontrado...
}
```

## Manipulando Códigos de Erro

Muitos bancos de dados retornam erros com códigos específicos, que podem indicar várias categorias de problemas como violação de constraints, problemas de conexão, ou erros de sintaxe. Handling these error codes in GORM requires parsing the error returned by the database and extracting the relevant code.


```go
import (
    "github.com/go-sql-driver/mysql"
    "gorm.io/gorm"
)

// ...

result := db.Create(&newRecord)
if result.Error != nil {
    if mysqlErr, ok := result.Error.(*mysql.MySQLError); ok {
        switch mysqlErr.Number {
        case 1062: // código MySQL para entrada duplicada
            // Manipular entrada duplicada
        // Adicionar casos para outros erros específicos
        default:
            // Manipular outros erros
        }
    } else {
        // Manipular erros que não são do MySQL, ou erros desconhecidos
    }
}
```

## Erros de dialetos traduzidos

GORM pode retornar erros específicos relacionados ao dialeto do banco de dados, quando `TranslateError` está ativo, GORM converto os erros específicos do banco de dados para seu próprio tipo generalizado de erro.

```go
db, err := gorm.Open(postgres.Open(postgresDSN), &gorm.Config{TranslateError: true})
```

- **ErrDuplicatedKey**

Esse erro acontece quando uma operação de insert viola uma unique constraint:

```go
result := db.Create(&newRecord)
if errors.Is(result.Error, gorm.ErrDuplicatedKey) {
    // Manipular erro de chave duplicada...
}
```

- **ErrForeignKeyViolated**

Esse erro é encontrado quando uma chave estrangeira é violada:

```go
result := db.Create(&newRecord)
if errors.Is(result.Error, gorm.ErrForeignKeyViolated) {
    // Manipular erro de violação de chave estrangeira...
}
```

Habilitando `TranslateError`, GORM fornece uma maneira mais unificada de manipular os erros em diversos tipos de bancos de dados, traduzindo erros específicos de banco de dados para erros que pertencem ao GORM.

## Erros

Para uma lista completa de erros que o GORM pode retornar, verifique a documentação GORM em [Errors List](https://github.com/go-gorm/gorm/blob/master/errors.go).
