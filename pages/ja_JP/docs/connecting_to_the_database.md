---
title: データベースに接続する
layout: page
---

GORM officially supports the databases MySQL, PostgreSQL, GaussDB, SQLite, SQL Server, and TiDB

## MySQL

```go
import (
  "gorm.io/driver/mysql"
  "gorm.io/gorm"
)

func main() {
  // 詳細は https://github.com/go-sql-driver/mysql#dsn-data-source-name を参照
  dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
  db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
}
```

{% note warn %}
**注** `time.Time` を正しく処理するには、パラメータとして `parseTime` を含める必要があります。 ([その他のパラメータ](https://github.com/go-sql-driver/mysql#parameters)) UTF-8 エンコーディングを完全にサポートするには、 `charset=utf8` を `charset=utf8mb4` に変更する必要があります。 詳細な説明は [この記事](https://mathiasbynens.be/notes/mysql-utf8mb4) を参照してください。
{% endnote %}

MySQLドライバは、 初期化時に指定できる[詳細な設定](https://github.com/go-gorm/mysql) を提供しています。

```go
db, err := gorm.Open(mysql.New(mysql.Config{
  DSN: "gorm:gorm@tcp(127.0.0.1:3306)/gorm?charset=utf8&parseTime=True&loc=Local", // data source name
  DefaultStringSize: 256, // default size for string fields
  DisableDatetimePrecision: true, // disable datetime precision, which not supported before MySQL 5.6
  DontSupportRenameIndex: true, // drop & create when rename index, rename index not supported before MySQL 5.7, MariaDB
  DontSupportRenameColumn: true, // `change` when rename column, rename column not supported before MySQL 8, MariaDB
  SkipInitializeWithVersion: false, // auto configure based on currently MySQL version
}), &gorm.Config{})
```

### ドライバをカスタマイズする

GORMでは、 `DriverName` オプションを使用してMySQLドライバをカスタマイズできます。

```go
import (
  _ "example.com/my_mysql_driver"
  "gorm.io/driver/mysql"
  "gorm.io/gorm"
)

db, err := gorm.Open(mysql.New(mysql.Config{
  DriverName: "my_mysql_driver",
  DSN: "gorm:gorm@tcp(localhost:9910)/gorm?charset=utf8&parseTime=True&loc=Local", // data source name, refer https://github.com/go-sql-driver/mysql#dsn-data-source-name
}), &gorm.Config{})
```

### 既存のデータベース接続

GORMでは、すでに確立されているデータベース接続を使って `*gorm.DB` を初期化することができます。

```go
import (
  "database/sql"
  "gorm.io/driver/mysql"
  "gorm.io/gorm"
)

sqlDB, err := sql.Open("mysql", "mydb_dsn")
gormDB, err := gorm.Open(mysql.New(mysql.Config{
  Conn: sqlDB,
}), &gorm.Config{})
```

## PostgreSQL

```go
import (
  "gorm.io/driver/postgres"
  "gorm.io/gorm"
)

dsn := "host=localhost user=gorm password=gorm dbname=gorm port=9920 sslmode=disable TimeZone=Asia/Shanghai"
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
```

Postgresのdatabase/sqlドライバとして [pgx](https://github.com/jackc/pgx) を使用しています。これはデフォルトでprepared statement cacheを有効にしています。無効にするには:

```go
// https://github.com/go-gorm/postgres
db, err := gorm.Open(postgres.New(postgres.Config{
  DSN: "user=gorm password=gorm dbname=gorm port=9920 sslmode=disable TimeZone=Asia/Shanghai",
  PreferSimpleProtocol: true, // disables implicit prepared statement usage
}), &gorm.Config{})
```

### ドライバーのカスタマイズ

GORMでは、 `DriverName` オプションを使用してPostgreSQLドライバをカスタマイズできます。例:

```go
import (
  _ "github.com/GoogleCloudPlatform/cloudsql-proxy/proxy/dialers/postgres"
  "gorm.io/gorm"
)

db, err := gorm.Open(postgres.New(postgres.Config{
  DriverName: "cloudsqlpostgres",
  DSN: "host=project:region:instance user=postgres dbname=postgres password=password sslmode=disable",
})
```

### 既存のデータベース接続

GORMでは、すでに確立されているデータベース接続を使って `*gorm.DB` を初期化することができます。

```go
import (
  "database/sql"
  "gorm.io/driver/postgres"
  "gorm.io/gorm"
)

sqlDB, err := sql.Open("pgx", "mydb_dsn")
gormDB, err := gorm.Open(postgres.New(postgres.Config{
  Conn: sqlDB,
}), &gorm.Config{})
```

## GaussDB

```go
import (
  "gorm.io/driver/gaussdb"
  "gorm.io/gorm"
)

dsn := "host=localhost user=gorm password=gorm dbname=gorm port=8000 sslmode=disable TimeZone=Asia/Shanghai"
db, err := gorm.Open(gaussdb.Open(dsn), &gorm.Config{})
```

We are using [gaussdb-go](https://github.com/HuaweiCloudDeveloper/gaussdb-go) as gaussdb's database/sql driver, it enables prepared statement cache by default, to disable it:

```go
// https://github.com/go-gorm/gaussdb
db, err := gorm.Open(gaussdb.New(gaussdb.Config{
  DSN: "user=gorm password=gorm dbname=gorm port=8000 sslmode=disable TimeZone=Asia/Shanghai",
  PreferSimpleProtocol: true, // disables implicit prepared statement usage
}), &gorm.Config{})
```

### Customize Driver

GORM allows to customize the GaussDB driver with the `DriverName` option, for example:

```go
import (
  _ "github.com/GoogleCloudPlatform/cloudsql-proxy/proxy/dialers/gaussdb"
  "gorm.io/gorm"
)

db, err := gorm.Open(gaussdb.New(gaussdb.Config{
  DriverName: "cloudsqlgaussdb",
  DSN: "host=project:region:instance user=gaussdb dbname=gaussdb password=password sslmode=disable",
})
```

### Existing database connection

GORM allows to initialize `*gorm.DB` with an existing database connection

```go
import (
  "database/sql"
  "gorm.io/driver/gaussdb"
  "gorm.io/gorm"
)

sqlDB, err := sql.Open("gaussdbgo", "mydb_dsn")
gormDB, err := gorm.Open(gaussdb.New(gaussdb.Config{
  Conn: sqlDB,
}), &gorm.Config{})
```

## SQLite

```go
import (
  "gorm.io/driver/sqlite" // Sqlite driver based on CGO
  // "github.com/glebarez/sqlite" // Pure go SQLite driver, checkout https://github.com/glebarez/sqlite for details
  "gorm.io/gorm"
)

// github.com/mattn/go-sqlite3
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{})
```

{% note warn %}
**注意:** ファイルへのパスを指定する代わりに、 `file::memory:?cache=shared` を使用することもできます。 これを指定することで、システムメモリで一時的なデータベースを使用するようSQLiteに指示します。 (詳細については [SQLite docs](https://www.sqlite.org/inmemorydb.html) を参照してください。)
{% endnote %}

## SQL Server

```go
import (
  "gorm.io/driver/sqlserver"
  "gorm.io/gorm"
)

// github.com/denisenkom/go-mssqldb
dsn := "sqlserver://gorm:LoremIpsum86@localhost:9930?database=gorm"
db, err := gorm.Open(sqlserver.Open(dsn), &gorm.Config{})
```

## TiDB

TiDBはMySQLプロトコルと互換性があります。 [MySQL](#mysql) の部分に従って、TiDB への接続を作成できます。

TiDBには注目すべきポイントがいくつかあります:

- `gorm:"primaryKey;default:auto_random()"` タグを使用して、TiDB の [`AUTO_RANDOM`](https://docs.pingcap.com/tidb/stable/auto-random) 機能を使用できます。
- TiDBは`v6.2.0`から [`SAVEPOINT`](https://docs.pingcap.com/tidb/stable/sql-statement-savepoint) をサポートしています, この機能を使用する場合は、TiDBのバージョンに注意してください。
- TiDBは`v6.6.0`から [`SAVEPOINT`](https://docs.pingcap.com/tidb/dev/foreign-key) をサポートしています, この機能を使用する場合は、TiDBのバージョンに注意してください。

```go
import (
  "fmt"
  "gorm.io/driver/mysql"
  "gorm.io/gorm"
)

type Product struct {
  ID    uint `gorm:"primaryKey;default:auto_random()"`
  Code  string
  Price uint
}

func main() {
  db, err := gorm.Open(mysql.Open("root:@tcp(127.0.0.1:4000)/test"), &gorm.Config{})
  if err != nil {
    panic("failed to connect database")
  }

  db.AutoMigrate(&Product{})

  insertProduct := &Product{Code: "D42", Price: 100}

  db.Create(insertProduct)
  fmt.Printf("insert ID: %d, Code: %s, Price: %d\n",
    insertProduct.ID, insertProduct.Code, insertProduct.Price)

  readProduct := &Product{}
  db.First(&readProduct, "code = ?", "D42") // find product with code D42

  fmt.Printf("read ID: %d, Code: %s, Price: %d\n",
    readProduct.ID, readProduct.Code, readProduct.Price)
}
```

## Clickhouse

https://github.com/go-gorm/clickhouse

```go
import (
  "gorm.io/driver/clickhouse"
  "gorm.io/gorm"
)

func main() {
  dsn := "tcp://localhost:9000?database=gorm&username=gorm&password=gorm&read_timeout=10&write_timeout=20"
  db, err := gorm.Open(clickhouse.Open(dsn), &gorm.Config{})

  // Auto Migrate
  db.AutoMigrate(&User{})
  // Set table options
  db.Set("gorm:table_options", "ENGINE=Distributed(cluster, default, hits)").AutoMigrate(&User{})

  // Insert
  db.Create(&user)

  // Select
  db.Find(&user, "id = ?", 10)

  // Batch Insert
  var users = []User{user1, user2, user3}
  db.Create(&users)
  // ...
}
```

## コネクションプール

GORMは [database/sql](https://pkg.go.dev/database/sql) を使用してコネクションプールを維持しています。

```go
sqlDB, err := db.DB()

// SetMaxIdleConns sets the maximum number of connections in the idle connection pool.
sqlDB.SetMaxIdleConns(10)

// SetMaxOpenConns sets the maximum number of open connections to the database.
sqlDB.SetMaxOpenConns(100)

// SetConnMaxLifetime sets the maximum amount of time a connection may be reused.
sqlDB.SetConnMaxLifetime(time.Hour)
```

Refer [Generic Interface](generic_interface.html) for details

## サポートされていないデータベース

Some databases may be compatible with the `mysql` or `postgres` dialect, in which case you could just use the dialect for those databases.

それ以外の場合、 [ドライバーを作ることをお勧めします。プルリクエストを歓迎しています！](write_driver.html)
