# [GORM](https://gorm.io/) Snowflake Driver

[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-2.1-4baaaa.svg)](CODE-OF-CONDUCT.md)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
![GitHub release](https://img.shields.io/github/release/go-obvious/gorm-snowflake.svg)


Simple Snowflake Driver for [GORM](https://gorm.io/) inspired by [omixen/gorm-snowflake](https://github.com/omixen/gorm-snowflake).

## Snowflake Features

Notable Snowflake (SF) features that affect decisions in this driver:

- **Case Sensitivity**: Use of quotes in SF enforces case-sensitivity, requiring string conditions to match. Currently, all quotes are removed internally to make the driver case-insensitive, and only uppercase is used when working with internal tables (INFORMATION_SCHEMA).
- **Indexing**: SF does not support INDEX. It performs micro-partitioning automatically for all tables to optimize performance. Therefore, all index-related functions return nil.
- **Transactions**: SF transactions do not support SAVEPOINT. Refer to the [Snowflake documentation](https://docs.snowflake.com/en/sql-reference/transactions.html) for more details.
- **Default Values**: GORM relies on querying back inserted rows in every transaction to retrieve default values. SF does not support this directly like SQL Server (`OUTPUT INSERTED`) or Postgres (`RETURNING`). Instead, the `CHANGE_TRACKING` feature is enabled for all tables, allowing `CHANGES` queries after DML operations. However, due to the non-deterministic nature of returns from `MERGE`, updates are not supported.
- **Merge Statements**: The `SELECT...CHANGES` feature of SF does not return unchanged rows from `MERGE` statements. Therefore, only the `APPEND_ONLY` option is supported, and only fields from inserted rows in the same order are returned.
- **Constraints**: SF enforces only NOT NULL constraints. This driver expects all tests and features related to enforcing constraints to be disabled. Refer to the [Snowflake documentation](https://docs.snowflake.com/en/user-guide/table-considerations.html#referential-integrity-constraints) for more details.

## Usage

How to use this project:

```go
package main

import (
    "encoding/json"
    "fmt"
    "time"

    "github.com/snowflakedb/gosnowflake"
    snowflake "github.com/go-obvious/gorm-snowflake"
    "gorm.io/gorm"
)

type User struct {
    ID        int64
    FirstName string
    LastName  string
    Status    string
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Optional if you specify the Schema in the Config
func (r *User) TableName() string {
    return "MY_SCHEMA.USERS"
}

func main() {
    config := gosnowflake.Config{
        Account:   "12345678.us-east-1",
        User:      "<SNOWFLAKE USER>",
        Password:  "<PASSWORD>",
        Database:  "MY_DATABASE",
        Schema:    "MY_SCHEMA",    // Optional if your models have a TableName method
        Warehouse: "MY_WAREHOUSE", // Optional
    }

    connStr, err := gosnowflake.DSN(&config)
    if err != nil {
        panic(err)
    }

    db, err := gorm.Open(snowflake.Open(connStr), &gorm.Config{
        NamingStrategy: snowflake.NewNamingStrategy(),
    })
    if err != nil {
        panic(err)
    }

    var users []User
    tx := db.Where("status = ?", "active").Find(&users)
    if tx.Error != nil {
        fmt.Println(tx.Error)
    }

    bytes, err := json.MarshalIndent(users, "", "\t")
    if err != nil {
        panic(err)
    }
    fmt.Print(string(bytes))
}
```
