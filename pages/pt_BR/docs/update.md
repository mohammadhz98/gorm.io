---
title: Atualização
layout: page
---

## Save All Fields

### Traditional API

`Save` will save all fields when performing the Updating SQL

```go
db.First(&user)

user.Name = "jinzhu 2"
user.Age = 100
db.Save(&user)
// UPDATE users SET name='jinzhu 2', age=100, birthday='2016-01-01', updated_at = '2013-11-17 21:34:10' WHERE id=111;
```

`Save` is an upsert function:
- If the value contains no primary key, it performs `Create`
- If the value has a primary key, it first executes **Update** (all fields, by `Select(*)`).
- If `rows affected = 0` after **Update**, it automatically falls back to `Create`.

> 💡 **Note**: `Save` guarantees either an update or insert will occur.  
> To prevent unintended creation when no rows match, use [ `Select(*).Updates()` ](update.html#Update-Selected-Fields).

```go
db.Save(&User{Name: "jinzhu", Age: 100})
// INSERT INTO `users` (`name`,`age`,`birthday`,`update_at`) VALUES ("jinzhu",100,"0000-00-00 00:00:00","0000-00-00 00:00:00")

db.Save(&User{ID: 1, Name: "jinzhu", Age: 100})
// UPDATE `users` SET `name`="jinzhu",`age`=100,`birthday`="0000-00-00 00:00:00",`update_at`="0000-00-00 00:00:00" WHERE `id` = 1
```

{% note warn %}
**NOTE** Don't use `Save` with `Model`, it's an **Undefined Behavior**.
{% endnote %}

{% note warn %}
**NOTE** The `Save` method is intentionally removed from the Generics API to prevent ambiguity and concurrency issues. Please use `Create` or `Updates` methods instead.
{% endnote %}

## Update single column

When updating a single column with `Update`, it needs to have any conditions or it will raise error `ErrMissingWhereClause`, checkout [Block Global Updates](#block_global_updates) for details.

### Generics API

```go
ctx := context.Background()

// Update with conditions
err := gorm.G[User](db).Where("active = ?", true).Update(ctx, "name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE active=true;

// Update with ID condition
err := gorm.G[User](db).Where("id = ?", 111).Update(ctx, "name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;

// Update with multiple conditions
err := gorm.G[User](db).Where("id = ? AND active = ?", 111, true).Update(ctx, "name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;
```

### Traditional API

When using the `Model` method and its value has a primary value, the primary key will be used to build the condition, for example:

```go
// Update with conditions
db.Model(&User{}).Where("active = ?", true).Update("name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE active=true;

// User's ID is `111`:
db.Model(&user).Update("name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;

// Update with conditions and model value
db.Model(&user).Where("active = ?", true).Update("name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;
```

## Updates multiple columns

`Updates` supports updating with `struct` or `map[string]interface{}`, when updating with `struct` it will only update non-zero fields by default

### Generics API

```go
ctx := context.Background()

// Update attributes with `struct`, will only update non-zero fields
err := gorm.G[User](db).Where("id = ?", 111).Updates(ctx, User{Name: "hello", Age: 18, Active: false})
// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;

// Update attributes with `map`
err := gorm.G[User](db).Where("id = ?", 111).Updates(ctx, map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello', age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
```

### Traditional API

```go
// Update attributes with `struct`, will only update non-zero fields
db.Model(&user).Updates(User{Name: "hello", Age: 18, Active: false})
// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;

// Update attributes with `map`
db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello', age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
```

{% note warn %}
**NOTE** When updating with struct, GORM will only update non-zero fields. You might want to use `map` to update attributes or use `Select` to specify fields to update
{% endnote %}

## Update Selected Fields

If you want to update selected fields or ignore some fields when updating, you can use `Select`, `Omit`

### Generics API

```go
ctx := context.Background()

// Select with Map
err := gorm.G[User](db).Where("id = ?", 111).Select("name").Updates(ctx, map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello' WHERE id=111;

err := gorm.G[User](db).Where("id = ?", 111).Omit("name").Updates(ctx, map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;

// Select with Struct (select zero value fields)
err := gorm.G[User](db).Where("id = ?", 111).Select("Name", "Age").Updates(ctx, User{Name: "new_name", Age: 0})
// UPDATE users SET name='new_name', age=0 WHERE id=111;

// Select all fields (select all fields include zero value fields)
err := gorm.G[User](db).Where("id = ?", 111).Select("*").Updates(ctx, User{Name: "jinzhu", Role: "admin", Age: 0})

// Select all fields but omit Role (select all fields include zero value fields)
err := gorm.G[User](db).Where("id = ?", 111).Select("*").Omit("Role").Updates(ctx, User{Name: "jinzhu", Role: "admin", Age: 0})
```

### Traditional API

```go
// Select with Map
// User's ID is `111`:
db.Model(&user).Select("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello' WHERE id=111;

db.Model(&user).Omit("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;

// Select with Struct (select zero value fields)
db.Model(&user).Select("Name", "Age").Updates(User{Name: "new_name", Age: 0})
// UPDATE users SET name='new_name', age=0 WHERE id=111;

// Select all fields (select all fields include zero value fields)
db.Model(&user).Select("*").Updates(User{Name: "jinzhu", Role: "admin", Age: 0})

// Select all fields but omit Role (select all fields include zero value fields)
db.Model(&user).Select("*").Omit("Role").Updates(User{Name: "jinzhu", Role: "admin", Age: 0})
```

## Update Hooks

GORM allows the hooks `BeforeSave`, `BeforeUpdate`, `AfterSave`, `AfterUpdate`. Those methods will be called when updating a record, refer [Hooks](hooks.html) for details

```go
func (u *User) BeforeUpdate(tx *gorm.DB) (err error) {
    if u.Role == "admin" {
        return errors.New("admin user not allowed to update")
    }
    return
}
```

## Batch Updates

If we haven't specified a record having a primary key value with `Model`, GORM will perform a batch update

```go
// Update with struct
db.Model(User{}).Where("role = ?", "admin").Updates(User{Name: "hello", Age: 18})
// UPDATE users SET name='hello', age=18 WHERE role = 'admin';

// Update with map
db.Table("users").Where("id IN ?", []int{10, 11}).Updates(map[string]interface{}{"name": "hello", "age": 18})
// UPDATE users SET name='hello', age=18 WHERE id IN (10, 11);
```

### <span id="block_global_updates">Block Global Updates</span>

If you perform a batch update without any conditions, GORM WON'T run it and will return `ErrMissingWhereClause` error by default

You have to use some conditions or use raw SQL or enable the `AllowGlobalUpdate` mode, for example:

```go
db.Model(&User{}).Update("name", "jinzhu").Error // gorm.ErrMissingWhereClause

db.Model(&User{}).Where("1 = 1").Update("name", "jinzhu")
// UPDATE users SET `name` = "jinzhu" WHERE 1=1

db.Exec("UPDATE users SET name = ?", "jinzhu")
// UPDATE users SET name = "jinzhu"

db.Session(&gorm.Session{AllowGlobalUpdate: true}).Model(&User{}).Update("name", "jinzhu")
// UPDATE users SET `name` = "jinzhu"
```

### Updated Records Count

Get the number of rows affected by a update

```go
// Get updated records count with `RowsAffected`
result := db.Model(User{}).Where("role = ?", "admin").Updates(User{Name: "hello", Age: 18})
// UPDATE users SET name='hello', age=18 WHERE role = 'admin';

result.RowsAffected // returns updated records count
result.Error        // returns updating error
```

## Advanced

### <span id="update_from_sql_expr">Update with SQL Expression</span>

GORM allows updating a column with a SQL expression, e.g:

```go
// product's ID is `3`
db.Model(&product).Update("price", gorm.Expr("price * ? + ?", 2, 100))
// UPDATE "products" SET "price" = price * 2 + 100, "updated_at" = '2013-11-17 21:34:10' WHERE "id" = 3;

db.Model(&product).Updates(map[string]interface{}{"price": gorm.Expr("price * ? + ?", 2, 100)})
// UPDATE "products" SET "price" = price * 2 + 100, "updated_at" = '2013-11-17 21:34:10' WHERE "id" = 3;

db.Model(&product).UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
// UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = 3;

db.Model(&product).Where("quantity > 1").UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
// UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = 3 AND quantity > 1;
```

And GORM also allows updating with SQL Expression/Context Valuer with [Customized Data Types](data_types.html#gorm_valuer_interface), e.g:

```go
// Create from customized data type
type Location struct {
    X, Y int
}

func (loc Location) GormValue(ctx context.Context, db *gorm.DB) clause.Expr {
  return clause.Expr{
    SQL:  "ST_PointFromText(?)",
    Vars: []interface{}{fmt.Sprintf("POINT(%d %d)", loc.X, loc.Y)},
  }
}

db.Model(&User{ID: 1}).Updates(User{
  Name:  "jinzhu",
  Location: Location{X: 100, Y: 100},
})
// UPDATE `user_with_points` SET `name`="jinzhu",`location`=ST_PointFromText("POINT(100 100)") WHERE `id` = 1
```

### Update from SubQuery

Update a table by using SubQuery

```go
db.Model(&user).Update("company_name", db.Model(&Company{}).Select("name").Where("companies.id = users.company_id"))
// UPDATE "users" SET "company_name" = (SELECT name FROM companies WHERE companies.id = users.company_id);

db.Table("users as u").Where("name = ?", "jinzhu").Update("company_name", db.Table("companies as c").Select("name").Where("c.id = u.company_id"))

db.Table("users as u").Where("name = ?", "jinzhu").Updates(map[string]interface{}{"company_name": db.Table("companies as c").Select("name").Where("c.id = u.company_id")})
```

### Without Hooks/Time Tracking

If you want to skip `Hooks` methods and don't track the update time when updating, you can use `UpdateColumn`, `UpdateColumns`, it works like `Update`, `Updates`

```go
// Update single column
db.Model(&user).UpdateColumn("name", "hello")
// UPDATE users SET name='hello' WHERE id = 111;

// Update multiple columns
db.Model(&user).UpdateColumns(User{Name: "hello", Age: 18})
// UPDATE users SET name='hello', age=18 WHERE id = 111;

// Update selected columns
db.Model(&user).Select("name", "age").UpdateColumns(User{Name: "hello", Age: 0})
// UPDATE users SET name='hello', age=0 WHERE id = 111;
```

### Returning Data From Modified Rows

Returning changed data only works for databases which support Returning, for example:

```go
// return all columns
var users []User
db.Model(&users).Clauses(clause.Returning{}).Where("role = ?", "admin").Update("salary", gorm.Expr("salary * ?", 2))
// UPDATE `users` SET `salary`=salary * 2,`updated_at`="2021-10-28 17:37:23.19" WHERE role = "admin" RETURNING *
// users => []User{{ID: 1, Name: "jinzhu", Role: "admin", Salary: 100}, {ID: 2, Name: "jinzhu.2", Role: "admin", Salary: 1000}}

// return specified columns
db.Model(&users).Clauses(clause.Returning{Columns: []clause.Column{{Name: "name"}, {Name: "salary"}}}).Where("role = ?", "admin").Update("salary", gorm.Expr("salary * ?", 2))
// UPDATE `users` SET `salary`=salary * 2,`updated_at`="2021-10-28 17:37:23.19" WHERE role = "admin" RETURNING `name`, `salary`
// users => []User{{ID: 0, Name: "jinzhu", Role: "", Salary: 100}, {ID: 0, Name: "jinzhu.2", Role: "", Salary: 1000}}
```

### Check Field has changed?

GORM provides the `Changed` method which could be used in **Before Update Hooks**, it will return whether the field has changed or not.

The `Changed` method only works with methods `Update`, `Updates`, and it only checks if the updating value from `Update` / `Updates` equals the model value. It will return true if it is changed and not omitted

```go
func (u *User) BeforeUpdate(tx *gorm.DB) (err error) {
  // if Role changed
    if tx.Statement.Changed("Role") {
    return errors.New("role not allowed to change")
    }

  if tx.Statement.Changed("Name", "Admin") { // if Name or Role changed
    tx.Statement.SetColumn("Age", 18)
  }

  // if any fields changed
    if tx.Statement.Changed() {
        tx.Statement.SetColumn("RefreshedAt", time.Now())
    }
    return nil
}

db.Model(&User{ID: 1, Name: "jinzhu"}).Updates(map[string]interface{"name": "jinzhu2"})
// Changed("Name") => true
db.Model(&User{ID: 1, Name: "jinzhu"}).Updates(map[string]interface{"name": "jinzhu"})
// Changed("Name") => false, `Name` not changed
db.Model(&User{ID: 1, Name: "jinzhu"}).Select("Admin").Updates(map[string]interface{
  "name": "jinzhu2", "admin": false,
})
// Changed("Name") => false, `Name` not selected to update

db.Model(&User{ID: 1, Name: "jinzhu"}).Updates(User{Name: "jinzhu2"})
// Changed("Name") => true
db.Model(&User{ID: 1, Name: "jinzhu"}).Updates(User{Name: "jinzhu"})
// Changed("Name") => false, `Name` not changed
db.Model(&User{ID: 1, Name: "jinzhu"}).Select("Admin").Updates(User{Name: "jinzhu2"})
// Changed("Name") => false, `Name` not selected to update
```

### Change Updating Values

To change updating values in Before Hooks, you should use `SetColumn` unless it is a full update with `Save`, for example:

```go
func (user *User) BeforeSave(tx *gorm.DB) (err error) {
  if pw, err := bcrypt.GenerateFromPassword(user.Password, 0); err == nil {
    tx.Statement.SetColumn("EncryptedPassword", pw)
  }

  if tx.Statement.Changed("Code") {
    user.Age += 20
    tx.Statement.SetColumn("Age", user.Age)
  }
}

db.Model(&user).Update("Name", "jinzhu")
```
