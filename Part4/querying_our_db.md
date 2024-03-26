# Querying our data

Okay, so we can connect to our DB, but that's not a lot of use if we can't get the data out. Let's change that to execute a query against the `users` table.

First off, we need to update our `getUsers()` function to execute a SQL query:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {

    rows, err := db.DB.Query("SELECT id, name FROM users")

    //rest of the code

}
```

Excellent. We now have a way to execute a SQL query against our data. But at the moment it's passing back *rows* of data, and we're not dealing with the error.

Okay, let's handle the error first:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {

    rows, err := db.DB.Query("SELECT id, name FROM users")
    if err != nil {
        fmt.Println("Error querying the database:", err)
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }

    //rest of the code

}
```

Now we need to iterate over our rows and add the user details to a slice. There's a lot of boilerplate code you have to write to do this:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
    // Use the db package to interact with the database
    rows, err := db.DB.Query("SELECT id, name FROM users")
    if err != nil {
        fmt.Println("Error querying the database:", err)
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }
    defer rows.Close()

    // Create a slice to hold the users
    users := []User{}

    for rows.Next() {
        var user User
        // Scan the values from the current row into the User struct
        if err := rows.Scan(&user.ID, &user.Name); err != nil {
            fmt.Println("Error scanning row values:", err)
            http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
            return
        }

        users = append(users, user)
    }

    if err := rows.Err(); err != nil {
        fmt.Println("Error iterating over rows:", err)
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }

    usersJSON, err := json.Marshal(users)
    if err != nil {
        fmt.Println("Error marshaling users into JSON:", err)
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }

    writer.Header().Set("Content-Type", "application/json")

    // Write JSON response to the ResponseWriter
    _, err = writer.Write(usersJSON)
    if err != nil {
        fmt.Println("Error writing JSON response:", err)
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }

}
```

That's a lot of code just to read the users from our DB!

And we could say that you should write this out and understand what it is about, but to be honest only fools use no tools. However, feel free to copy the code and check out what you get back from your DB, which should be an empty array!

## SQLX package

To make our lives easier, let's install another package `sqlx` which is another package that provides extensions to `database/sql` package. From the terminal run:

`go get github.com/jmoiron/sqlx`

We'll need to add this import in our `main.go` package:

```go
package main

import (
    "fmt"
    "net/http"
    "encoding/json"
    "acme/db"
    "github.com/jmoiron/sqlx"
)
```

and our `db.go` package:

```go
package db

import (
	"fmt"

	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"
)
```

We also need to make some minor changes to use `sqlx` instead of `sql` in our `db.go` package where you see `//<< UPDATED`:

```go
package db

import (
	"fmt"

	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"
)

var DB *sqlx.DB //<< UPDATED

func InitDB(connectionString string) error {

    db, err := sqlx.Open("postgres", connectionString) //<< UPDATED

    //everything else exactly the same

}
```

How this package works is that we provide some `db` field options on our struct, that map to the names of the column in the DB table. We're using our Users struct for this, so we need to update that:

```go
type User struct {
    ID   int    `json:"id" db:"id"`
    Name string `json:"name" db:"name"`
}
```

This is the "magic glue" that allows `sqlx` to map columns in each row to our struct, without us having to write all that boilerplate code. Excellent.

Now our `getUsers()` function is much simpler:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
    
    fmt.Printf("got /api/users request\n")
    users := []User{}

    err := sqlx.Select(db.DB, &users, "SELECT * FROM users")
    if err != nil {
        fmt.Println("Error querying the database:", err)
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }

    // same code as before from the next line

    usersJSON, err := json.Marshal(users)
    // ...
    
}
```

Whereas previously we had `rows, err := db.DB.Query("SELECT id, name FROM users")` we now wrap that in the sqlx function: 

`err := sqlx.Select(db.DB, &users, "SELECT * FROM users")`

So we provide it with the three parameters it wants - the DB connection, a pointer to the slice, and the basic query to execute.

Right, let's give it a whirl. Save you files and run your `main.go` file and check the `api/users` endpoint - you should get back an empty array.

## Add some users

Okay, let's make it return something more interesting than an empty array!

Back on the terminal you [logged into and created the DB](/Part3/connecting_to_a_database.md#create-the-db), you now need to connect to the db using the `\c acme` command:

```
postgres=# \c acme
You are now connected to database "acme" as user "postgres".
acme=#
```

Now let's add a single user named 'Alan' to our DB by running `INSERT INTO users (name) VALUES ('Alan');`, which will look like this.

```
acme=# INSERT INTO users (name) VALUES ('Alan');
INSERT 0 1
acme=#
```

Now when you go to your /api/users endpoint you should see this JSON:

```json
[
  {
    "id": 1,
    "name": "Alan"
  }
]
```

---

[>> Part 5 - Creating Users with a Post Request](/Part5/posting_and_creating.md)