# Database Migrations

Currently our code has hard-coded SQL in it. Whilst this is okay for now, very soon it will make our `postgres.go` file very large and unwieldy.

We're going to make use of [database migrations](https://github.com/bjssacademy/go-database-migrations) using [goose](https://github.com/pressly/goose).

## Install goose

### Windows

`go install github.com/pressly/goose/cmd/goose@latest`

### Unix

https://pressly.github.io/goose/installation/#linux

## Set Environment Variables

> :exclamation: Replace YOURPASSWORD in the below with your actual postgres password!

### Windows (Command Prompt)

```cmd
set GOOSE_DRIVER=postgres
set GOOSE_DBSTRING=user=postgres password=YOURPASSWORD dbname=acme sslmode=disable
```

### Windows (Powershell)

```powershell
$env:GOOSE_DRIVER="postgres"
$env:GOOSE_DBSTRING="user=postgres password=YOURPASSWORD dbname=acme sslmode=disable"
```

### Unix

```bash
export GOOSE_DRIVER=postgres
export GOOSE_DBSTRING=user=postgres password=YOURPASSWORD dbname=acme sslmode=disable
```

## Migrations

Make a new folder named `migrations` under the root in the terminal and move into that folder:

```
mkdir migrations
cd migrations
```

Create our first migration:

```
goose create users_table sql
```

Open the created file and update it:

```sql
-- +goose Up
-- +goose StatementBegin
CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(50) NOT NULL
    );
-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin
DROP TABLE users;
-- +goose StatementEnd
```

## Execute Migrations On Start

We don't want to have to manually run our migrations, so let's get goose to do the dirty work for us. In the `postgres.go` file, add an import to `"github.com/pressly/goose"`.

Then we are going to remove our code that executed the CREATE statement and replace it with `goose.up()`:

```go
    /*_, err = DB.Exec(`CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(50) NOT NULL
    );`)
    if err != nil {
        return err
    }*/

     // Run migrations
     err = goose.Up(db.DB, "./migrations")
     //err = goose.Run("/migrations", db.DB, "postgres")
     if err != nil {
         panic(err)
     }
```

Now when you start your webserver, it will check for migrations and apply any new ones - really useful when you are pulling changes from others into your code!

[Chapter 10 - The Repository Pattern >>](/Part10/repository-pattern.md)
