# Connecting to a database

We're going to be using PostgreSQL as our database. If you don't have it installed already, check out [this guide](https://www.postgresqltutorial.com/postgresql-getting-started/install-postgresql/) on how to install it. MMake sure you install `pgAdmin`, `PostgreSQL Server` and `Command Line Tools`.

Please make a note of the password you set in the setup!

## Create the DB

Now we have that done, we need to create a new DB to connect to. Open a new terminal and type

`psql -U postgres`

Enter your password when you setup PostgreSQL. You should now be in the `psql` shell. You can tell as your terminal prompt should look like:

`postgres-#`

Next we need to create the acme database. Type in the following command:

`create database acme;`

> NOTE: It's vital you include the `;` at the end of the line.

Now if you type `\l` you should see the acme database listed.

Now type `\q` to exist the psql command line.

## Install 

We're going to use an external package to make connecting to PostgreSQL easier. From the terminal run:

`go get github.com/lib/pq`

## Our own package

Now we're going to create our own package to use. This will make it easier to handle our connections.

1. Create a new folder called `postgres` if it doesn't exist.
2. In that folder create a file called `postgres.go`.
3. Import the required packages:

```go
package postgres

import (
    "database/sql"
    "fmt"
    _ "github.com/lib/pq"
)

var DB *sql.DB
```

> When we import a package with an underscore, it tells the Go compiler that you want to import the package *solely* for its side effects, such as registering a driver or initializing global variables, but you don't intend to use any of its exported identifiers directly in your code.
>
> 1. When we import _ "github.com/lib/pq", the Go compiler ensures that the `init` function of the pq package is executed. The `init` function contains code that registers the PostgreSQL driver with the `database/sql` package.
> 
> 2. After importing, we can use the `database/sql` package to interact with the PostgreSQL database without explicitly referencing the pq package.

`var db *sql.DB` is a package level variable we will use to hold our connection to the DB.

So far, we haven't connected to it. We'll need to provide a way to do that, and we want to control that from our `main` function by providing the DB name, username, and password.

Let's create an `InitDb` function in our `postgres.go` file that we can pass the connection string to that handles our connection:

```go
func InitDB(connectionString string) error {

    db, err := sql.Open("postgres", connectionString)
    if err != nil {
        return fmt.Errorf("error connecting to the database: %w", err)
    }
    DB = db

    // Ping DB to check connection is successful
    if err := DB.Ping(); err != nil {
        return fmt.Errorf("error pinging the database: %w", err)
    }

    fmt.Println("Successfully connected to the database!")
    return nil
}
```

Okay, all that should be good, but there's only one way to find out, and that's to test our connection!

## Test the connection

Back in our `main.go` file, we now need to import our package so we can use it. Remember that importing packages is in the format `{yourmodulename}/{folderpath}`, so we need to add `acme/db`:

```go
import (
    "fmt"
    "net/http"
    "encoding/json"
    "acme/db"
)
```

Now let's check we can connect to our db:

```go
func main() {
    // Initialize the database connection
    connectionString := "user=postgres dbname=acme password=yourpassword host=localhost sslmode=disable"
    if err := postgres.InitDB(connectionString); err != nil {
        fmt.Println("Error initializing the database:", err)
        return
    }
    defer postgres.DB.Close()

    //rest of the code unchanged

}
```

`defer db.DB.Close()` if new to us. The `defer` keyword here means it will delay closing the database connection until the main function exist (when the server is closed down). This is NOT the best way to manage connections to your database!

NOTE: You need to update `password`! 

If your username was `ted`, your password was `supersecretpassword`, and your database name was `acme`, the connectionString would be:

`connectionString := "user=ted dbname=acme password=supersecretpassword host=localhost sslmode=disable"`

Okay, now save that and see if you get a successful ping on your DB connection!

## Create the users table

Okay, so we have a database, but at the moment we don't have a table. Let's fix that by adding some code into our `InitDB` function, before our ping:

```go

    //code above as before

	_, err = DB.Exec(`CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(50) NOT NULL
    );`)
    if err != nil {
        return err
    }

    // Ping DB to check connection is successful
    //code after as before

```

Excellent. Let's run our code once more and check that everything still works!!

#### Output
```
Successfully connected to the database!
Server listening on port 8080...
```

---

[Part 8.1 - Querying our Data >>](/Part8/querying_our_db.md)