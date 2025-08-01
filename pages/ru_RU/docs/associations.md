---
title: Связи
layout: страница
---

## Автоматические Create/Update

GORM автоматизирует сохранение связей и их ссылок при создании или обновлении записей, с использованием метода upsert, который в первую очередь обновляет внешние ссылки на существующие связи.

### Авто сохранение связей при создании

При создании новой записи GORM будет автоматически сохранять связанные с ней данные. Это включает в себя вставку данных в связанные таблицы и управление ссылками на внешние ключи.

```go
user := User{
  Name:            "jinzhu",
  BillingAddress:  Address{Address1: "Billing Address - Address 1"},
  ShippingAddress: Address{Address1: "Shipping Address - Address 1"},
  Emails:          []Email{
    {Email: "jinzhu@example.com"},
    {Email: "jinzhu-2@example.com"},
  },
  Languages:       []Language{
    {Name: "ZH"},
    {Name: "EN"},
  },
}

// Создание пользователя с указанием его адресов, электронной почты и языка
db.Create(&user)
// BEGIN TRANSACTION;
// INSERT INTO "addresses" (address1) VALUES ("Billing Address - Address 1"), ("Shipping Address - Address 1") ON DUPLICATE KEY DO NOTHING;
// INSERT INTO "users" (name,billing_address_id,shipping_address_id) VALUES ("jinzhu", 1, 2);
// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu@example.com"), (111, "jinzhu-2@example.com") ON DUPLICATE KEY DO NOTHING;
// INSERT INTO "languages" ("name") VALUES ('ZH'), ('EN') ON DUPLICATE KEY DO NOTHING;
// INSERT INTO "user_languages" ("user_id","language_id") VALUES (111, 1), (111, 2) ON DUPLICATE KEY DO NOTHING;
// COMMIT;

db.Save(&user)
```

### Обновление связей с `FullSaveAssociations`

Для сценариев, в которых требуется полное обновление связанных данных (а не только ссылок на внешние ключи), следует использовать режим `FullSaveAssociations`.

```go
// Обновить данные пользователя и полностью обновить все его связи
db.Session(&gorm.Session{FullSaveAssociations: true}).Updates(&user)
// SQL: Полностью обновляет таблицы адресов, пользователей, электронной почты, включая существующие связанные записи
```

Использование `FullSaveAssociations` гарантирует, что все состояние модели, включая все её связи, будет отражено в базе данных, что обеспечит целостность и согласованность данных во всём приложении.

## Пропуск автоматических Create/Update

GORM обеспечивает гибкость, позволяющую пропустить автоматическое сохранение связей во время операций создания или обновления. Этого можно добиться с помощью методов `Select` или `Omit`, которые позволяют точно указать, какие поля или связи следует включить или исключить из операции.

### Использование `Select` для включения конкретных полей

The `Select` method lets you specify which fields of the model should be saved. This means that only the selected fields will be included in the SQL operation.

```go
user := User{
  // User and associated data
}

// Only include the 'Name' field when creating the user
db.Select("Name").Create(&user)
// SQL: INSERT INTO "users" (name) VALUES ("jinzhu");
```

### Using `Omit` to Exclude Fields or Associations

Conversely, `Omit` allows you to exclude certain fields or associations when saving a model.

```go
// Skip creating the 'BillingAddress' when creating the user
db.Omit("BillingAddress").Create(&user)

// Skip all associations when creating the user
db.Omit(clause.Associations).Create(&user)
```

{% note warn %}
**NOTE:** For many-to-many associations, GORM upserts the associations before creating join table references. To skip this upserting, use `Omit` with the association name followed by `.*`:

```go
// Skip upserting 'Languages' associations
db.Omit("Languages.*").Create(&user)
```

To skip creating both the association and its references:

```go
// Skip creating 'Languages' associations and their references
db.Omit("Languages").Create(&user)
```
{% endnote %}

Using `Select` and `Omit`, you can fine-tune how GORM handles the creation or updating of your models, giving you control over the auto-save behavior of associations.

## Выбрать/пропускать поля ассоциации

In GORM, when creating or updating records, you can use the `Select` and `Omit` methods to specifically include or exclude certain fields of an associated model.

With `Select`, you can specify which fields of an associated model should be included when saving the primary model. This is particularly useful for selectively saving parts of an association.

Conversely, `Omit` lets you exclude certain fields of an associated model from being saved. This can be useful when you want to prevent specific parts of an association from being persisted.


```go
user := User{
  Name:            "jinzhu",
  BillingAddress:  Address{Address1: "Billing Address - Address 1", Address2: "addr2"},
  ShippingAddress: Address{Address1: "Shipping Address - Address 1", Address2: "addr2"},
}

// Create user and his BillingAddress, ShippingAddress, including only specified fields of BillingAddress
db.Select("BillingAddress.Address1", "BillingAddress.Address2").Create(&user)
// SQL: Creates user and BillingAddress with only 'Address1' and 'Address2' fields

// Create user and his BillingAddress, ShippingAddress, excluding specific fields of BillingAddress
db.Omit("BillingAddress.Address2", "BillingAddress.CreatedAt").Create(&user)
// SQL: Creates user and BillingAddress, omitting 'Address2' and 'CreatedAt' fields
```

## Удаление связей

GORM allows for the deletion of specific associated relationships (has one, has many, many2many) using the `Select` method when deleting a primary record. This feature is particularly useful for maintaining database integrity and ensuring related data is appropriately managed upon deletion.

You can specify which associations should be deleted along with the primary record by using `Select`.

```go
// Delete a user's account when deleting the user
db.Select("Account").Delete(&user)

// Delete a user's Orders and CreditCards associations when deleting the user
db.Select("Orders", "CreditCards").Delete(&user)

// Delete all of a user's has one, has many, and many2many associations
db.Select(clause.Associations).Delete(&user)

// Delete each user's account when deleting multiple users
db.Select("Account").Delete(&users)
```

{% note warn %}
**NOTE:** It's important to note that associations will be deleted only if the primary key of the deleting record is not zero. GORM uses these primary keys as conditions to delete the selected associations.

```go
// This will not work as intended
db.Select("Account").Where("name = ?", "jinzhu").Delete(&User{})
// SQL: Deletes all users with name 'jinzhu', but their accounts won't be deleted

// Correct way to delete a user and their account
db.Select("Account").Where("name = ?", "jinzhu").Delete(&User{ID: 1})
// SQL: Deletes the user with name 'jinzhu' and ID '1', and the user's account

// Deleting a user with a specific ID and their account
db.Select("Account").Delete(&User{ID: 1})
// SQL: Deletes the user with ID '1', and the user's account
```
{% endnote %}

## Типы ассоциаций

Association Mode in GORM offers various helper methods to handle relationships between models, providing an efficient way to manage associated data.

To start Association Mode, specify the source model and the relationship's field name. The source model must contain a primary key, and the relationship's field name should match an existing association.

```go
var user User
db.Model(&user).Association("Languages")
// Check for errors
error := db.Model(&user).Association("Languages").Error
```

### Finding Associations

Retrieve associated records with or without additional conditions.

```go
// Simple find
db.Model(&user).Association("Languages").Find(&languages)

// Find with conditions
codes := []string{"zh-CN", "en-US", "ja-JP"}
db.Model(&user).Where("code IN ?", codes).Association("Languages").Find(&languages)
```

### Appending Associations

Add new associations for `many to many`, `has many`, or replace the current association for `has one`, `belongs to`.

```go
// Append new languages
db.Model(&user).Association("Languages").Append([]Language{languageZH, languageEN})

db.Model(&user).Association("Languages").Append(&Language{Name: "DE"})

db.Model(&user).Association("CreditCard").Append(&CreditCard{Number: "411111111111"})
```

### Replacing Associations

Replace current associations with new ones.

```go
// Replace existing languages
db.Model(&user).Association("Languages").Replace([]Language{languageZH, languageEN})

db.Model(&user).Association("Languages").Replace(Language{Name: "DE"}, languageEN)
```

### Deleting Associations

Remove the relationship between the source and arguments, only deleting the reference.

```go
// Delete specific languages
db.Model(&user).Association("Languages").Delete([]Language{languageZH, languageEN})

db.Model(&user).Association("Languages").Delete(languageZH, languageEN)
```

### Clearing Associations

Remove all references between the source and association.

```go
// Clear all languages
db.Model(&user).Association("Languages").Clear()
```

### Counting Associations

Get the count of current associations, with or without conditions.

```go
// Count all languages
db.Model(&user).Association("Languages").Count()

// Count with conditions
codes := []string{"zh-CN", "en-US", "ja-JP"}
db.Model(&user).Where("code IN ?", codes).Association("Languages").Count()
```

### Batch Data Handling

Association Mode allows you to handle relationships for multiple records in a batch. This includes finding, appending, replacing, deleting, and counting operations for associated data.

- **Finding Associations**: Retrieve associated data for a collection of records.

```go
db.Model(&users).Association("Role").Find(&roles)
```

- **Deleting Associations**: Remove specific associations across multiple records.

```go
db.Model(&users).Association("Team").Delete(&userA)
```

- **Counting Associations**: Get the count of associations for a batch of records.

```go
db.Model(&users).Association("Team").Count()
```

- **Appending/Replacing Associations**: Manage associations for multiple records. Note the need for matching argument lengths with the data.

```go
var users = []User{user1, user2, user3}

// Append different teams to different users in a batch
// Append userA to user1's team, userB to user2's team, and userA, userB, userC to user3's team
db.Model(&users).Association("Team").Append(&userA, &userB, &[]User{userA, userB, userC})

// Replace teams for multiple users in a batch
// Reset user1's team to userA, user2's team to userB, and user3's team to userA, userB, and userC
db.Model(&users).Association("Team").Replace(&userA, &userB, &[]User{userA, userB, userC})
```

## <span id="delete_association_record">Удаление связей</span>

In GORM, the `Replace`, `Delete`, and `Clear` methods in Association Mode primarily affect the foreign key references, not the associated records themselves. Understanding and managing this behavior is crucial for data integrity.

- **Reference Update**: These methods update the association's foreign key to null, effectively removing the link between the source and associated models.
- **No Physical Record Deletion**: The actual associated records remain untouched in the database.

### Modifying Deletion Behavior with `Unscoped`

For scenarios requiring actual deletion of associated records, the `Unscoped` method alters this behavior.

- **Soft Delete**: Marks associated records as deleted (sets `deleted_at` field) without removing them from the database.

```go
db.Model(&user).Association("Languages").Unscoped().Clear()
```

- **Permanent Delete**: Physically deletes the association records from the database.

```go
// db.Unscoped().Model(&user)
db.Unscoped().Model(&user).Association("Languages").Unscoped().Clear()
```

## <span id="tags">Теги связей</span>

Association tags in GORM are used to specify how associations between models are handled. These tags define the relationship's details, such as foreign keys, references, and constraints. Understanding these tags is essential for setting up and managing relationships effectively.

| Тег                | Описание                                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------------ |
| `foreignKey`       | Specifies the column name of the current model used as a foreign key in the join table.          |
| `references`       | Indicates the column name in the reference table that the foreign key of the join table maps to. |
| `polymorphic`      | Defines the polymorphic type, typically the model name.                                          |
| `polymorphicValue` | Sets the polymorphic value, usually the table name, if not specified otherwise.                  |
| `many2many`        | Names the join table used in a many-to-many relationship.                                        |
| `joinForeignKey`   | Identifies the foreign key column in the join table that maps back to the current model's table. |
| `joinReferences`   | Points to the foreign key column in the join table that links to the reference model's table.    |
| `constraint`       | Specifies relational constraints like `OnUpdate`, `OnDelete` for the association.                |
