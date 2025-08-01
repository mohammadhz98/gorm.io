---
title: Create
layout: страница
---

## Создание записи

### Generics API

```go
user := User{Name: "Jinzhu", Age: 18, Birthday: time.Now()}

// Create a single record
ctx := context.Background()
err := gorm.G[User](db).Create(ctx, &user) // pass pointer of data to Create

// Create with result
result := gorm.WithResult()
err := gorm.G[User](db, result).Create(ctx, &user)
user.ID             // returns inserted data's primary key
result.Error        // returns error
result.RowsAffected // returns inserted records count
```

### Traditional API

```go
user := User{Name: "Jinzhu", Age: 18, Birthday: time.Now()}

result := db.Create(&user) // передаем указатель на данные в Create

user.ID             // возвращает первичный ключ добавленной записи
result.Error        // возвращает ошибку
result.RowsAffected // возвращает количество вставленных записей
```

Мы также можем создать несколько записей с помощью `Create()`:
```go
users := []*User{
    {Name: "Jinzhu", Age: 18, Birthday: time.Now()},
    {Name: "Jackson", Age: 19, Birthday: time.Now()},
}

result := db.Create(users) // pass a slice to insert multiple row

result.Error        // returns error
result.RowsAffected // returns inserted records count
```

{% note warn %}
**ПРИМЕЧАНИЕ** В `create` необходимо передавать указатель на структуру, а не значение.
{% endnote %}

## Создание записи с указанными полями

При создании записи, используя Select, мы можем указать в какие именно поля необходимо занести значения.

```go
db.Select("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`name`,`age`,`created_at`) VALUES ("jinzhu", 18, "2020-07-04 11:05:21.775")
```

Также при создании записи, используя Omit, мы можем указать какие поля необходимо игнорировать.

```go
db.Omit("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`birthday`,`updated_at`) VALUES ("2020-01-01 00:00:00.000", "2020-07-04 11:05:21.775")
```

## <span id="batch_insert">Пакетная вставка</span>

Чтобы эффективно вставлять большое количество записей, в метод `Create` следует передавать слайс. GORM сгенерирует одну инструкцию SQL для вставки всех данных и обратного заполнения значений первичного ключа, также будут вызваны методы перехвата (*hook*). **Транзакция** начнется, когда записи можно будет разделить на несколько пакетов.

```go
var users = []User{{Name: "jinzhu1"}, {Name: "jinzhu2"}, {Name: "jinzhu3"}}
db.Create(&users)

for _, user := range users {
  user.ID // 1,2,3
}
```

Вы можете указать batch size при создании с помощью `CreateInBatches`, например:

```go
var users = []User{{Name: "jinzhu_1"}, ...., {Name: "jinzhu_10000"}}

// размер пакета 100
db.CreateInBatches(users, 100)
```

Batch Insert также поддерживается при использовании [Upsert](#upsert) и [Create With Associations](#create_with_associations)

{% note warn %}
**ПРИМЕЧАНИЕ** При инициализации GORM, в конфигурации, с помощью параметра `CreateBatchSize` можно указать размер пакета и все `INSERT` будут учитывать этот параметр при создании записей и ассоциаций
{% endnote %}

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  CreateBatchSize: 1000,
})

db := db.Session(&gorm.Session{CreateBatchSize: 1000})

users = [5000]User{{Name: "jinzhu", Pets: []Pet{pet1, pet2, pet3}}...}

db.Create(&users)
// INSERT INTO users xxx (5 пакетов)
// INSERT INTO pets xxx (15 пакетов)
```

## Создание хуков

GORM позволяет реализовать пользовательские перехватчики для `beforeSave`, `beforeCreate`, `afterSave`, `afterCreate`.  Эти методы перехвата будут вызываться при создании записи, для получения подробной информации о жизненном цикле обратитесь к [Hooks](hooks.html)

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  u.UUID = uuid.New()

    if u.Role == "admin" {
        return errors.New("неправильная роль")
    }
    return
}
```

Если вы хотите пропустить методы `Hooks`, вы можете использовать режим сессии, передав в качестве параметра `SkipHooks`, например:

```go
DB.Session(&gorm.Session{SkipHooks: true}).Create(&user)

DB.Session(&gorm.Session{SkipHooks: true}).Create(&users)

DB.Session(&gorm.Session{SkipHooks: true}).CreateInBatches(users, 100)
```

## Create с помощью Map(карты)

GORM поддерживает создание из `map[string]interface{}` и`[]map[string]interface{}{}`, например:

```go
db.Model(&User{}).Create(map[string]interface{}{
  "Name": "jinzhu", "Age": 18,
})

// пакетная вставка из `[]map[string]interface{}{}`
db.Model(&User{}).Create([]map[string]interface{}{
  {"Name": "jinzhu_1", "Age": 18},
  {"Name": "jinzhu_2", "Age": 20},
})
```

{% note warn %}
**ПРИМЕЧАНИЕ** При создании на основе map перехватчики не будут вызваны, ассоциации не будут сохранены, а значения первичных ключей не будут заполнены
{% endnote %}

## <span id="create_from_sql_expr">Метод Create с помощью SQL выражения/значения контекста</span>

GORM allows insert data with SQL expression, there are two ways to achieve this goal, create from `map[string]interface{}` or [Customized Data Types](data_types.html#gorm_valuer_interface), for example:

```go
// Создать из map
db.Model(User{}).Create(map[string]interface{}{
  "Name": "jinzhu",
  "Location": clause.Expr{SQL: "ST_PointFromText(?)", Vars: []interface{}{"POINT(100 100)"}},
})
// INSERT INTO `users` (`name`,`location`) VALUES ("jinzhu",ST_PointFromText("POINT(100 100)"));

// Создать из настраиваемого типа
type Location struct {
    X, Y int
}

// Scan реализует интерфейс sql.Scanner
func (loc *Location) Scan(v interface{}) error {
  // Scan a value into struct from database driver
}

func (loc Location) GormDataType() string {
  return "geometry"
}

func (loc Location) GormValue(ctx context.Context, db *gorm.DB) clause.Expr {
  return clause.Expr{
    SQL:  "ST_PointFromText(?)",
    Vars: []interface{}{fmt.Sprintf("POINT(%d %d)", loc.X, loc.Y)},
  }
}

type User struct {
  Name     string
  Location Location
}

db.Create(&User{
  Name:     "jinzhu",
  Location: Location{X: 100, Y: 100},
})
// INSERT INTO `users` (`name`,`location`) VALUES ("jinzhu",ST_PointFromText("POINT(100 100)"))
```

## Дополнительно

### <span id="create_with_associations">Create со связями</span>

When creating some data with associations, if its associations value is not zero-value, those associations will be upserted, and its `Hooks` methods will be invoked.

```go
type CreditCard struct {
  gorm.Model
  Number   string
  UserID   uint
}

type User struct {
  gorm.Model
  Name       string
  CreditCard CreditCard
}

db.Create(&User{
  Name: "jinzhu",
  CreditCard: CreditCard{Number: "411111111111"}
})
// INSERT INTO `users` ...
// INSERT INTO `credit_cards` ...
```

You can skip saving associations with `Select`, `Omit`, for example:

```go
db.Omit("CreditCard").Create(&user)

// пропустить все ассоциации
db.Omit(clause.Associations).Create(&user)
```

### <span id="default_values">Значения по умолчанию</span>

You can define default values for fields with tag `default`, for example:

```go
type User struct {
  ID   int64
  Name string `gorm:"default:galeone"`
  Age  int64  `gorm:"default:18"`
}
```

Then the default value *will be used* when inserting into the database for [zero-value](https://tour.golang.org/basics/12) fields

{% note warn %}
**NOTE** Any zero value like `0`, `''`, `false` won't be saved into the database for those fields defined default value, you might want to use pointer type or Scanner/Valuer to avoid this, for example:
{% endnote %}

```go
type User struct {
  gorm.Model
  Name string
  Age  *int           `gorm:"default:18"`
  Active sql.NullBool `gorm:"default:true"`
}
```

{% note warn %}
**NOTE** You have to setup the `default` tag for fields having default or virtual/generated value in database, if you want to skip a default value definition when migrating, you could use `default:(-)`, for example:
{% endnote %}

```go
type User struct {
  ID        string `gorm:"default:uuid_generate_v3()"` // функция базы данных
  FirstName string
  LastName  string
  Age       uint8
  FullName  string `gorm:"->;type:GENERATED ALWAYS AS (concat(firstname,' ',lastname));default:(-);"`
}
```

{% note warn %}
**NOTE** **SQLite** doesn't support some records are default values when batch insert. See [SQLite Insert stmt](https://www.sqlite.org/lang_insert.html). For example:

```go
type Pet struct {
    Name string `gorm:"default:cat"`
}

// In SQLite, this is not supported, so GORM will build a wrong SQL to raise error:
// INSERT INTO `pets` (`name`) VALUES ("dog"),(DEFAULT) RETURNING `name`
db.Create(&[]Pet{{Name: "dog"}, {}})
```
A viable alternative is to assign default value to fields in the hook, e.g.

```go
func (p *Pet) BeforeCreate(tx *gorm.DB) (err error) {
    if p.Name == "" {
        p.Name = "cat"
    }
}
```

You can see more info in [issues#6335](https://github.com/go-gorm/gorm/issues/6335)
{% endnote %}

When using virtual/generated value, you might need to disable its creating/updating permission, check out [Field-Level Permission](models.html#field_permission)

### <span id="upsert">Upsert (Создать или обновить) / При конфликте</span>

GORM provides compatible Upsert support for different databases

```go
import "gorm.io/gorm/clause"

// Ничего не предпринимать в связи с конфликтом
db.Clauses(clause.OnConflict{DoNothing: true}).Create(&user)

// Обновить столбцы до значения по умолчанию при конфликте `id`
db.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.Assignments(map[string]interface{}{"role": "user"}),
}).Create(&users)
// MERGE INTO "users" USING *** WHEN NOT MATCHED THEN INSERT *** WHEN MATCHED THEN UPDATE SET ***; SQL Server
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE ***; MySQL

// Использовать SQL-выражение
db.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.Assignments(map[string]interface{}{"count": gorm.Expr("GREATEST(count, VALUES(count))")}),
}).Create(&users)
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `count`=GREATEST(count, VALUES(count));

// Обновить столбцы до нового значения при конфликте `id`
db.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.AssignmentColumns([]string{"name", "age"}),
}).Create(&users)
// MERGE INTO "users" USING *** WHEN NOT MATCHED THEN INSERT *** WHEN MATCHED THEN UPDATE SET "name"="excluded"."name"; SQL Server
// INSERT INTO "users" *** ON CONFLICT ("id") DO UPDATE SET "name"="excluded"."name", "age"="excluded"."age"; PostgreSQL
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `name`=VALUES(name),`age`=VALUES(age); MySQL

// Обновить все столбцы до нового значения при конфликте, за исключением первичных ключей и тех столбцов, которые имеют значения по умолчанию из sql func
db.Clauses(clause.OnConflict{
  UpdateAll: true,
}).Create(&users)
// INSERT INTO "users" *** ON CONFLICT ("id") DO UPDATE SET "name"="excluded"."name", "age"="excluded"."age", ...;
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `name`=VALUES(name),`age`=VALUES(age), ...; MySQL
```

Also checkout `FirstOrInit`, `FirstOrCreate` on [Advanced Query](advanced_query.html)

Checkout [Raw SQL and SQL Builder](sql_builder.html) for more details
