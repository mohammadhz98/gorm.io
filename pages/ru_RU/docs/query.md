---
title: Запрос
layout: страница
---

## Получение одного объекта

GORM предоставляет методы `First`, `Take`, `Last` для получения одного объекта из базы данных. Эти методы добавляют условие `LIMIT 1` при запросе к базе данных, но если запись не найдена, вернется ошибка `ErrRecordNotFound`.

### Generics API

```go
ctx := context.Background()

// Get the first record ordered by primary key
user, err := gorm.G[User](db).First(ctx)
// SELECT * FROM users ORDER BY id LIMIT 1;

// Get one record, no specified order
user, err := gorm.G[User](db).Take(ctx)
// SELECT * FROM users LIMIT 1;

// Get last record, ordered by primary key desc
user, err := gorm.G[User](db).Last(ctx)
// SELECT * FROM users ORDER BY id DESC LIMIT 1;

// check error ErrRecordNotFound
errors.Is(err, gorm.ErrRecordNotFound)
```

### Traditional API

```go
// Получение первой записи при сортировке по первичному ключу "id"
db.First(&user)
// SELECT * FROM users ORDER BY id LIMIT 1;

// Получение одной записи без указания порядка сортировки
db.Take(&user)
// SELECT * FROM users LIMIT 1;

// Получение последней записи, отсортированные по первичному ключу "id" по убыванию
db.Last(&user)
// SELECT * FROM users ORDER BY id DESC LIMIT 1;

result := db.First(&user)
result.RowsAffected // возвращает количество найденных записей
result.Error        // возвращает error или nil

// проверка ошибки на ErrRecordNotFound
errors.Is(result.Error, gorm.ErrRecordNotFound)
```

{% note warn %}
Если вы хотите избежать ошибки `ErrRecordNotFound`, используйте метод `Find`, например `db.Limit(1).Find(&user)`, метод `Find` принимает как структурные, так и срезанные данные
{% endnote %}

{% note warn %}
Using `Find` without a limit for single object `db.Find(&user)` will query the full table and return only the first object which is non-deterministic and not performant
{% endnote %}

Методы `First` и `Last` найдут первую и последнюю запись (соответственно) в порядке, установленном первичным ключом. Они работают только тогда, когда указатель на целевую структуру передается методам в качестве аргумента или когда модель указана с помощью `db.Model()`. Кроме того, если для соответствующей модели не определен первичный ключ, то модель будет упорядочена по первому полю. Например:

```go
var user User
var users []User

// работает, потому что передается целевая структура
db.First(&user)
// SELECT * FROM `users` ORDER BY `users`.`id` LIMIT 1

// работает, потому что модель указана с помощью `db.Model()`
result := map[string]interface{}{}
db.Model(&User{}).First(&result)
// SELECT * FROM `users` ORDER BY `users`.`id` LIMIT 1

// не работает
result := map[string]interface{}{}
db.Table("users").First(&result)

// работает с методом `Take`
result := map[string]interface{}{}
db.Table("users").Take(&result)

// первичный ключ не определен, результаты будут упорядочены по первому полю (т.е. "Коду").
type Language struct {
  Code string
  Name string
}
db.First(&Language{})
// SELECT * FROM `languages` ORDER BY `languages`.`code` LIMIT 1
```

### Получение объектов по первичному ключу

Объекты могут быть извлечены с использованием первичного ключа с помощью [Встроенных условий](#inline_conditions), если первичным ключом является число. При работе со строками необходимо соблюдать особую осторожность, чтобы избежать SQL-инъекции; ознакомьтесь с [Раздел "Безопасность"](security.html) для получения подробной информации.

#### Generics API

```go
ctx := context.Background()

// Using numeric primary key
user, err := gorm.G[User](db).Where("id = ?", 10).First(ctx)
// SELECT * FROM users WHERE id = 10;

// Using string primary key
user, err := gorm.G[User](db).Where("id = ?", "10").First(ctx)
// SELECT * FROM users WHERE id = 10;

// Using multiple primary keys
users, err := gorm.G[User](db).Where("id IN ?", []int{1,2,3}).Find(ctx)
// SELECT * FROM users WHERE id IN (1,2,3);

// If the primary key is a string (for example, like a uuid)
user, err := gorm.G[User](db).Where("id = ?", "1b74413f-f3b8-409f-ac47-e8c062e3472a").First(ctx)
// SELECT * FROM users WHERE id = "1b74413f-f3b8-409f-ac47-e8c062e3472a";
```

#### Traditional API

```go
db.First(&user, 10)
// SELECT * FROM users WHERE id = 10;

db.First(&user, "10")
// SELECT * FROM users WHERE id = 10;

db.Find(&users, []int{1,2,3})
// SELECT * FROM users WHERE id IN (1,2,3);
```

Если первичным ключом является строка (например, как uuid), запрос будет записан следующим образом:

```go
db.First(&user, "id = ?", "1b74413f-f3b8-409f-ac47-e8c062e3472a")
// SELECT * FROM users WHERE id = "1b74413f-f3b8-409f-ac47-e8c062e3472a";
```

Когда целевой объект имеет первичное значение, первичный ключ будет использоваться для построения условия, например:

```go
var user = User{ID: 10}
db.First(&user)
// SELECT * FROM users WHERE id = 10;

var result User
db.Model(User{ID: 10}).First(&result)
// SELECT * FROM users WHERE id = 10;
```

{% note warn %}
**NOTE:** If you use gorm's specific field types like `gorm.DeletedAt`, it will run a different query for retrieving object/s.
{% endnote %}
```go
type User struct {
  ID           string `gorm:"primarykey;size:16"`
  Name         string `gorm:"size:24"`
  DeletedAt    gorm.DeletedAt `gorm:"index"`
}

var user = User{ID: 15}
db.First(&user)
//  SELECT * FROM `users` WHERE `users`.`id` = '15' AND `users`.`deleted_at` IS NULL ORDER BY `users`.`id` LIMIT 1
```
## Получение всех объектов

```go
// Get all records
result := db.Find(&users)
// SELECT * FROM users;

result.RowsAffected // returns found records count, equals `len(users)`
result.Error        // returns error
```

## Условия

### Строковые условия

```go
// Get first matched record
db.Where("name = ?", "jinzhu").First(&user)
// SELECT * FROM users WHERE name = 'jinzhu' ORDER BY id LIMIT 1;

// Get all matched records
db.Where("name <> ?", "jinzhu").Find(&users)
// SELECT * FROM users WHERE name <> 'jinzhu';

// IN
db.Where("name IN ?", []string{"jinzhu", "jinzhu 2"}).Find(&users)
// SELECT * FROM users WHERE name IN ('jinzhu','jinzhu 2');

// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)
// SELECT * FROM users WHERE name LIKE '%jin%';

// AND
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;

// Time
db.Where("updated_at > ?", lastWeek).Find(&users)
// SELECT * FROM users WHERE updated_at > '2000-01-01 00:00:00';

// BETWEEN
db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)
// SELECT * FROM users WHERE created_at BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';
```

{% note warn %}
If the object's primary key has been set, then condition query wouldn't cover the value of primary key but use it as a 'and' condition. For example:
```go
var user = User{ID: 10}
db.Where("id = ?", 20).First(&user)
// SELECT * FROM users WHERE id = 10 and id = 20 ORDER BY id ASC LIMIT 1
```
This query would give `record not found` Error. So set the primary key attribute such as `id` to nil before you want to use the variable such as `user` to get new value from database.
{% endnote %}


### Условия в структурах и картах

```go
// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 ORDER BY id LIMIT 1;

// Map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;

// Slice of primary keys
db.Where([]int64{20, 21, 22}).Find(&users)
// SELECT * FROM users WHERE id IN (20, 21, 22);
```

{% note warn %}
**NOTE** When querying with struct, GORM will only query with non-zero fields, that means if your field's value is `0`, `''`, `false` or other [zero values](https://tour.golang.org/basics/12), it won't be used to build query conditions, for example:
{% endnote %}

```go
db.Where(&User{Name: "jinzhu", Age: 0}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu";
```

To include zero values in the query conditions, you can use a map, which will include all key-values as query conditions, for example:

```go
db.Where(map[string]interface{}{"Name": "jinzhu", "Age": 0}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 0;
```

For more details, see [Specify Struct search fields](#specify_search_fields).

### <span id="specify_search_fields">Указание полей поиска для структуры</span>

When searching with struct, you can specify which particular values from the struct to use in the query conditions by passing in the relevant field name or the dbname to `Where()`, for example:

```go
db.Where(&User{Name: "jinzhu"}, "name", "Age").Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 0;

db.Where(&User{Name: "jinzhu"}, "Age").Find(&users)
// SELECT * FROM users WHERE age = 0;
```

### <span id="inline_conditions">Встроенное условие</span>

Query conditions can be inlined into methods like `First` and `Find` in a similar way to `Where`.

```go
// Get by primary key if it were a non-integer type
db.First(&user, "id = ?", "string_primary_key")
// SELECT * FROM users WHERE id = 'string_primary_key';

// Plain SQL
db.Find(&user, "name = ?", "jinzhu")
// SELECT * FROM users WHERE name = "jinzhu";

db.Find(&users, "name <> ? AND age > ?", "jinzhu", 20)
// SELECT * FROM users WHERE name <> "jinzhu" AND age > 20;

// Struct
db.Find(&users, User{Age: 20})
// SELECT * FROM users WHERE age = 20;

// Map
db.Find(&users, map[string]interface{}{"age": 20})
// SELECT * FROM users WHERE age = 20;
```

### Условие Not

Build NOT conditions, works similar to `Where`

```go
db.Not("name = ?", "jinzhu").First(&user)
// SELECT * FROM users WHERE NOT name = "jinzhu" ORDER BY id LIMIT 1;

// Not In
db.Not(map[string]interface{}{"name": []string{"jinzhu", "jinzhu 2"}}).Find(&users)
// SELECT * FROM users WHERE name NOT IN ("jinzhu", "jinzhu 2");

// Struct
db.Not(User{Name: "jinzhu", Age: 18}).First(&user)
// SELECT * FROM users WHERE name <> "jinzhu" AND age <> 18 ORDER BY id LIMIT 1;

// Not In slice of primary keys
db.Not([]int64{1,2,3}).First(&user)
// SELECT * FROM users WHERE id NOT IN (1,2,3) ORDER BY id LIMIT 1;
```

### Условие Or

```go
db.Where("role = ?", "admin").Or("role = ?", "super_admin").Find(&users)
// SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';

// Struct
db.Where("name = 'jinzhu'").Or(User{Name: "jinzhu 2", Age: 18}).Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' OR (name = 'jinzhu 2' AND age = 18);

// Map
db.Where("name = 'jinzhu'").Or(map[string]interface{}{"name": "jinzhu 2", "age": 18}).Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' OR (name = 'jinzhu 2' AND age = 18);
```

For more complicated SQL queries. please also refer to [Group Conditions in Advanced Query](advanced_query.html#group_conditions).

## Выбор определенных полей

`Select` allows you to specify the fields that you want to retrieve from database. Otherwise, GORM will select all fields by default.

```go
db.Select("name", "age").Find(&users)
// SELECT name, age FROM users;

db.Select([]string{"name", "age"}).Find(&users)
// SELECT name, age FROM users;

db.Table("users").Select("COALESCE(age,?)", 42).Rows()
// SELECT COALESCE(age,'42') FROM users;
```

Also check out [Smart Select Fields](advanced_query.html#smart_select)

## Порядок сортировки Order

Specify order when retrieving records from the database

```go
db.Order("age desc, name").Find(&users)
// SELECT * FROM users ORDER BY age desc, name;

// Multiple orders
db.Order("age desc").Order("name").Find(&users)
// SELECT * FROM users ORDER BY age desc, name;

db.Clauses(clause.OrderBy{
  Expression: clause.Expr{SQL: "FIELD(id,?)", Vars: []interface{}{[]int{1, 2, 3}}, WithoutParentheses: true},
}).Find(&User{})
// SELECT * FROM users ORDER BY FIELD(id,1,2,3)
```

## Limit и Offset

`Limit` specify the max number of records to retrieve `Offset` specify the number of records to skip before starting to return the records

```go
db.Limit(3).Find(&users)
// SELECT * FROM users LIMIT 3;

// Cancel limit condition with -1
db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
// SELECT * FROM users LIMIT 10; (users1)
// SELECT * FROM users; (users2)

db.Offset(3).Find(&users)
// SELECT * FROM users OFFSET 3;

db.Limit(10).Offset(5).Find(&users)
// SELECT * FROM users OFFSET 5 LIMIT 10;

// Cancel offset condition with -1
db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
// SELECT * FROM users OFFSET 10; (users1)
// SELECT * FROM users; (users2)
```

Refer to [Pagination](scopes.html#pagination) for details on how to make a paginator

## Group и Having

```go
type result struct {
  Date  time.Time
  Total int
}

db.Model(&User{}).Select("name, sum(age) as total").Where("name LIKE ?", "group%").Group("name").First(&result)
// SELECT name, sum(age) as total FROM `users` WHERE name LIKE "group%" GROUP BY `name` LIMIT 1


db.Model(&User{}).Select("name, sum(age) as total").Group("name").Having("name = ?", "group").Find(&result)
// SELECT name, sum(age) as total FROM `users` GROUP BY `name` HAVING name = "group"

rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Rows()
defer rows.Close()
for rows.Next() {
  ...
}

rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Rows()
defer rows.Close()
for rows.Next() {
  ...
}

type Result struct {
  Date  time.Time
  Total int64
}
db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Scan(&results)
```

## Distinct

Selecting distinct values from the model

```go
db.Distinct("name", "age").Order("name, age desc").Find(&results)
```

`Distinct` works with [`Pluck`](advanced_query.html#pluck) and [`Count`](advanced_query.html#count) too

## Joins

Specify Joins conditions

```go
type result struct {
  Name  string
  Email string
}

db.Model(&User{}).Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Scan(&result{})
// SELECT users.name, emails.email FROM `users` left join emails on emails.user_id = users.id

rows, err := db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Rows()
for rows.Next() {
  ...
}

db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Scan(&results)

// multiple joins with parameter
db.Joins("JOIN emails ON emails.user_id = users.id AND emails.email = ?", "jinzhu@example.org").Joins("JOIN credit_cards ON credit_cards.user_id = users.id").Where("credit_cards.number = ?", "411111111111").Find(&user)
```

### Joins с предварительной загрузкой

You can use `Joins` eager loading associations with a single SQL, for example:

```go
db.Joins("Company").Find(&users)
// SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` LEFT JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id`;

// inner join
db.InnerJoins("Company").Find(&users)
// SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` INNER JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id`;
```

Join with conditions

```go
db.Joins("Company", db.Where(&Company{Alive: true})).Find(&users)
// SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` LEFT JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id` AND `Company`.`alive` = true;
```

For more details, please refer to [Preloading (Eager Loading)](preload.html).

### Joins к производной таблице

You can also use `Joins` to join a derived table.

```go
type User struct {
    Id  int
    Age int
}

type Order struct {
    UserId     int
    FinishedAt *time.Time
}

query := db.Table("order").Select("MAX(order.finished_at) as latest").Joins("left join user user on order.user_id = user.id").Where("user.age > ?", 18).Group("order.user_id")
db.Model(&Order{}).Joins("join (?) q on order.finished_at = q.latest", query).Scan(&results)
// SELECT `order`.`user_id`,`order`.`finished_at` FROM `order` join (SELECT MAX(order.finished_at) as latest FROM `order` left join user user on order.user_id = user.id WHERE user.age > 18 GROUP BY `order`.`user_id`) q on order.finished_at = q.latest
```


## <span id="scan">Scan</span>

Scanning results into a struct works similarly to the way we use `Find`

```go
type Result struct {
  Name string
  Age  int
}

var result Result
db.Table("users").Select("name", "age").Where("name = ?", "Antonio").Scan(&result)

// Raw SQL
db.Raw("SELECT name, age FROM users WHERE name = ?", "Antonio").Scan(&result)
```
