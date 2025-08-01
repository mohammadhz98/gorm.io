---
title: 错误处理
layout: 页面
---

在Go语言的应用开发中，特别是在使用GORM与数据库交互时，有效的错误处理是构建稳健应用的基石。 GORM's approach to error handling requires a nuanced understanding based on the API style you're using.

## 基本错误处理

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

With the Traditional API, GORM integrates error handling into its chainable method syntax. `*gorm.DB`实例包含一个`Error`字段，当发生错误时会被设置。 通常的做法是在执行数据库操作后，特别是在[完成方法（Finisher Methods）](method_chaining.html#finisher_method)后，检查这个字段。

在一系列方法之后，检查`Error`字段是至关重要的：

```go
if err := db.Where("name = ?", "jinzhu").First(&user).Error; err != nil {
  // 处理错误...
}
```

或者

```go
if result := db.Where("name = ?", "jinzhu").First(&user); result.Error != nil {
  // 处理错误...
}
```

## `ErrRecordNotFound`

当使用`First`、`Last`、`Take`等方法未找到记录时，GORM会返回`ErrRecordNotFound`。

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
  // 处理未找到记录的错误...
}
```

## 处理错误代码

许多数据库返回带有特定代码的错误，这些代码可能表明各种问题，如约束违规、连接问题或语法错误。 Handling these error codes in GORM requires parsing the error returned by the database and extracting the relevant code.


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
        case 1062: // MySQL中表示重复条目的代码
            // 处理重复条目
        // 为其他特定错误代码添加案例
        default:
            // 处理其他错误
        }
    } else {
        // 处理非MySQL错误或未知错误
    }
}
```

## 方言转换错误

当启用`TranslateError`时，GORM可以返回与所使用的数据库方言相关的特定错误，GORM将数据库特有的错误转换为其自己的通用错误。

```go
db, err := gorm.Open(postgres.Open(postgresDSN), &gorm.Config{TranslateError: true})
```

- **ErrDuplicatedKey**

当插入操作违反唯一约束时，会发生此错误：

```go
result := db.Create(&newRecord)
if errors.Is(result.Error, gorm.ErrDuplicatedKey) {
    // 处理重复键错误...
}
```

- **ErrForeignKeyViolated**

当违反外键约束时，会遇到此错误：

```go
result := db.Create(&newRecord)
if errors.Is(result.Error, gorm.ErrForeignKeyViolated) {
    // 处理外键违规错误...
}
```

通过启用`TranslateError`，GORM提供了一种更统一的错误处理方式，将不同数据库的特定错误转换为常见的GORM错误类型。

## Errors

要获取GORM可能返回的完整错误列表，请参考[GORM](https://github.com/go-gorm/gorm/blob/master/errors.go)文档中的错误列表。
