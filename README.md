# SQLBoiler

[![License](https://img.shields.io/badge/license-BSD-blue.svg)](https://github.com/vattle/sqlboiler/blob/master/LICENSE)
[![GoDoc](https://godoc.org/github.com/vattle/sqlboiler?status.svg)](https://godoc.org/github.com/vattle/sqlboiler)
[![CircleCI](https://circleci.com/gh/vattle/sqlboiler.svg?style=shield)](https://circleci.com/gh/vattle/sqlboiler)
[![Go Report Card](https://goreportcard.com/badge/vattle/sqlboiler)](http://goreportcard.com/report/vattle/sqlboiler)

SQLBoiler is a tool to generate a Go data model tailored to your database schema.

It is a "database-first" ORM as opposed to "code-first" (like gorm/gorp).
That means you must first create your database schema. Please use something
like goose or some other migration tool to manage this part of the database's
lifecycle.


## Why?
Well...


---

## About SQL Boiler

#### Features

- Full model generation
- High performance through generation
- Extremely fast code generation
- Uses boil.Executor (simple interface, sql.DB, sqlx.DB etc. compatible)
- Easy workflow (models can always be regenerated, full auto-complete)
- Strongly typed querying (usually no converting or binding to pointers)
- Hooks (Before/After Create/Select/Update/Delete/Upsert)
- Automatic CreatedAt/UpdatedAt
- Relationships/Associations
- Eager loading (recursive)
- Transactions
- Raw SQL fallbacks
- Compatibility tests (Run against your own DB schema)
- Debug logging

#### Supported Databases

- PostgreSQL

**Note:** Seeking contributors for other database engines.

#### Small Taste

For a comprehensive list of available operations and examples please see [Features & Examples](#features-examples).

```go
import (
  // Import this so we don't have to use qm.Limit etc.
  . "github.com/vattle/sqlboiler/boil/qm"
)

// Open handle to database like normal
db, err := sql.Open("postgres", "dbname=fun user=abc")
if err != nil {
  return err
}

// Query all users
users, err := models.Users(db).All()

// Panic-able if you like to code that way
users := models.Users(db).AllP()

// More complex query
users, err := models.Users(db, Where("age > ?", 30), Limit(5), Offset(6)).All()

// Ultra complex query
users, err := models.Users(db,
  Select("id", "name"),
  InnerJoin("credit_cards c on c.user_id = users.id"),
  Where("age > ?", 30),
  AndIn("c.kind in ?", "visa", "mastercard"),
  Or("email like ?", "%aol.com%"),
  GroupBy("id", "name"),
  Having("count(c.id) > ?", 2),
  Limit(5),
  Offset(6),
).All()

// Use any "boil.Executor" implementation (*sql.DB, *sql.Tx, data-dog mock db)
// for any query.
tx, err := db.Begin()
if err != nil {
  return err
}
users, err := models.Users(tx).All()

// Relationships
user, err := models.Users(db).One()
if err != nil {
  return err
}
movies, err := user.FavoriteMovies(db).All()

// Eager loading
users, err := models.Users(db, Load("FavoriteMovies")).All()
if err != nil {
  return err
}
fmt.Println(len(users.R.FavoriteMovies))
```

---

## Requirements & Pro Tips

### Requirements

* Table names and column names should use `snake_case` format.
  * At the moment we require `snake_case` table names and column names. This
  is a recommended default in Postgres. We can reassess this for future database drivers.
* Join tables should use a *composite primary key*.
  * For join tables to be used transparently for relationships your join table must have
  a *composite primary key* that encompasses both foreign table foreign keys. For example, on a
  join table named `user_videos` you should have: `primary key(user_id, video_id)`, with both `user_id`
  and `video_id` being foreign key columns to the users and videos tables respectively.

### Pro Tips
* Foreign key column names should end with `_id`.
  * Foreign key column names in the format `x_id` will generate clearer method names.
  It is advisable to use this naming convention whenever it makes sense for your database schema.
* If you never plan on using the hooks functionality you can disable generation of this 
  feature using the `--no-hooks` flag. This will save you some binary size.

---

## Getting started

#### Download

```shell
go get -u -t github.com/vattle/sqlboiler
```

#### Configuration

Create a configuration file. Because the project uses [viper](github.com/spf13/viper), TOML, JSON and YAML
are all supported. Environment variables are also able to be used.
We will assume TOML for the rest of the documentation.

The configuration file should be named `sqlboiler.toml` and is searched for in the following directories in this
order:

- `./`
- `$XDG_CONFIG_HOME/sqlboiler/`
- `$HOME/.config/sqlboiler/`

We require you pass in the `postgres` configuration via the configuration file rather than env vars.
There is no command line argument support for database configuration. Values given under the `postgres`
block are passed directly to the [pq](github.com/lib/pq) driver. Here is a rundown of all the different
values that can go in that section:

| Name | Required | Default |
| --- | --- | --- |
| dbname  | yes       | none      |
| host    | yes       | none      |
| port    | no        | 5432      |
| user    | yes       | none      |
| pass    | no        | none      |
| sslmode | no        | "require" |

You can also pass in these top level configuration values if you would prefer
not to pass them through the command line or environment variables:

| Name | Default |
| --- | --- |
| base_dir  | none      |
| pkg_name  | "models"  |
| out_folder| "models"  |
| exclude   | [ ]       |

**Example:**

```toml
[postgres]
dbname="dbname"
host="localhost"
port=5432
user="dbusername"
pass="dbpassword"
```


#### Initial Generation

After creating a configuration file that points at the database we want to
generate models for, we can invoke the sqlboiler command line utility.

```text
SQL Boiler generates a Go ORM from template files, tailored to your database schema.
Complete documentation is available at http://github.com/vattle/sqlboiler

Usage:
  sqlboiler [flags] <driver>

Examples:
sqlboiler postgres

Flags:
  -b, --basedir string        The base directory templates and templates_test folders are
  -d, --debug                 Debug mode prints stack traces on error
  -x, --exclude stringSlice   Tables to be excluded from the generated package (default [])
      --no-auto-timestamps    Disable automatic timestamps for created_at/updated_at
      --no-hooks              Disable hooks feature for your models
  -o, --output string         The name of the folder to output to (default "models")
  -p, --pkgname string        The name you wish to assign to your generated package (default "models")
```

Follow the steps below to do some basic model generation. Once we've generated
our models, we can run the compatibility tests which will exercise the entirety
of the generated code. This way we can ensure that our database is compatible
with sqlboiler. If you find there are some failing tests, please check the
[Diagnosing Problems](#diagnosing-problems) section.

```shell
# Generate our models and exclude the migrations table
sqlboiler -x goose_migrations postgres

# Run the generated tests
go test ./models # This requires an administrator postgres user because of some
                 # voodoo we do to disable triggers for the generated test db
```

---

## Diagnosing Problems

The most common causes of problems and panics are:

- Forgetting to exclude tables you do not want included in your generation, like migration tables.
- Tables without a primary key. All tables require one.
- Forgetting foreign key constraints on your columns that reference other tables.
- The compatibility tests that run against your own DB schema require a superuser, ensure the user
  supplied in your config has adequate privileges.
- A nil or closed database handle. Ensure your passed in `boil.Executor` is not nil.
  - If you decide to use the `G` variant of functions instead, make sure you've initialized your
    global database handle using `boil.SetDB()`.

For errors with other causes, it may be simple to debug yourself by looking at the generated code.
Setting `boil.DebugMode` to `true` can help with this. You can change the output using `boil.DebugWriter` (defaults to `os.Stdout`).

If you're still stuck and/or you think you've found a bug, feel free to leave an issue and we'll do our best to help you.

---

## Features & Examples

### Function Variations

### Automatic CreatedAt/UpdatedAt

If your generated SQLBoiler models package can find columns with the
names `created_at` or `updated_at` it will automatically set them
to `time.Now()` in your database, and update your object appropriately.
To disable this feature use `--no-auto-timestamps`.

**Note:** You can set the timezone for this feature by calling `boil.SetLocation()`

#### Overriding Automatic Timestamps

* **Insert**
  * Timestamps for both `updated_at` and `created_at` that are zero values will be set automatically.
  * To set the timestamp to null, set `Valid` to false and `Time` to a non-zero value.
  This is somewhat of a work around until we can devise a better solution in a later version.
* **Update**
  * The `updated_at` column will always be set to `time.Now()`. If you need to override
  this value you will need to fall back to another method in the meantime: `boil.SQL()`,
  overriding `updated_at` in all of your objects using a hook, or create your own wrapper.
* **Upsert**
  * `created_at` will be set automatically if it is a zero value, otherwise your supplied value
  will be used. To set `created_at` to `null`, set `Valid` to false and `Time` to a non-zero value.
  * The `updated_at` column will always be set to `time.Now()`.

### Hooks

### Query Mod System

### Finishers

### Raw Query

### Binding

### Transactions

### Debug Logging

### Select

### Find

### Insert

### Update
updateall slice and query

### Delete
deleteall slice and query

### Upsert

### Reload
reloadall slice

### Relationships
relationships to one and to many
relationship set ops (to one: set, remove, tomany: add, set, remove)
eager loading (nested and flat)

---

## Benchmarks

---

## FAQ

### Won't compiling models for a huge database be very slow?

No, because Go's toolchain - unlike traditional toolchains - makes the compiler do most of the work
instead of the linker. This means that when the first `go install` is done it can take
a little bit of time because there is a lot of code that is generated. However, because of this
work balance between the compiler and linker in Go, linking to that code afterwards in the subsequent
compiles is extremely fast.