# Part 1 - Querying our data

Okay, so we can connect to our DB, but that's not a lot of use if we can't get the data out. Let's change that to execute a query against the `users` table.

First off, we need to add the `GetUsers()` function to our `postgres.go` file:

```go
func GetUsers() ([]model.User, error) {

}
```

Now we need to update our `GetUsers()` function in to execute a SQL query:

```go
func GetUsers() ([]model.User, error) {
    rows, err := DB.Query("SELECT id, name FROM users")
    
	return []model.User{}, nil
}
```

Excellent. We now have a way to execute a SQL query against our data. But at the moment we've got *rows* of data, and we're not dealing with the error.

Okay, let's handle the error first:

```go
func GetUsers() ([]model.User, error) {
    rows, err := DB.Query("SELECT id, name FROM users")

    if err != nil {
        fmt.Println("Error querying the database:", err)
        return []model.User{}, errors.New("Database could not be queried")
    }

    //rest of the code

}
```

Now we need to iterate over our rows and add the user details to a slice. There's a lot of boilerplate code you have to write to do this:

```go
func GetUsers() ([]model.User, error) {
    rows, err := DB.Query("SELECT id, name FROM users")

    if err != nil {
        fmt.Println("Error querying the database:", err)
        return []model.User{}, errors.New("Database could not be queried")
    }

    defer rows.Close()

    // Create a slice to hold the users
    users := []model.User{}

    for rows.Next() {
        var user model.User
        // Scan the values from the current row into the User struct
        if err := rows.Scan(&user.ID, &user.Name); err != nil {
            fmt.Println("Error scanning row values:", err)
            return []model.User{}, errors.New("Database could not be queried")
        }

        users = append(users, user)
    }

	return users, nil
}
```

That's a lot of code just to read the users from our DB, and we don't even know if it works yet!

Let's fix that by updating our `user-service.go` to use the postgres package:


```go
package service

import (
	"acme/db"
	"acme/model"
	"acme/postgres"
	"errors"
	"fmt"
)

func GetUsers() ([]model.User, error) {

	users, err := postgres.GetUsers() //used to be db.GetUsers()

	if err != nil {
		fmt.Println("Error getting users from DB:", err)
		return nil, errors.New("There was an error getting the users from the database.")
	}

	return users, nil

}

// all other code untouched
```

All we have done here is add in our postgres package, and point the call to the database instance rather than the in-memory instance.

Go ahead and save your code. When you perform a GET you should get back an empty array as we haven't added any users in yet.

-----

## Lots of boilerplate

And we could say that you should write this out and understand what it is about, but to be honest only fools use no tools. And it's a *lot* of copying and pasting

---

# Part 2 - SQLX package

To make our lives easier, let's install another package `sqlx` which is another package that provides extensions to `database/sql` package. From the terminal run:

`go get github.com/jmoiron/sqlx`

We'll need to add this import in our `postgres.go` package:

```go
package postgres

import (
	"acme/model"
	//"database/sql"
	"errors"
	"fmt"
    "github.com/jmoiron/sqlx"

	_ "github.com/lib/pq"
)
```

We also need to make some minor changes to use `sqlx` instead of `sql` in our `postgres.go` package where you see `//<< UPDATED`:

```go
//imports above

var DB *sqlx.DB //<< UPDATED

func InitDB(connectionString string) error {

    db, err := sqlx.Open("postgres", connectionString) //<< UPDATED

    //everything else exactly the same

}
```

How this package works is that we provide some `db` field options on our struct, that map to the names of the column in the DB table. We're using our `Users` struct for this, so we need to update that:

```go
type User struct {
    ID   int    `json:"id" db:"id"`
    Name string `json:"name" db:"name"`
}
```

This is the "magic glue" that allows `sqlx` to map columns in each row to our struct, without us having to write all that boilerplate code. Excellent.

Now our `GetUsers()` function in our `postgres.go` file is much simpler:

```go
func GetUsers() ([]model.User, error) {

    users := []model.User{}

    err := sqlx.Select(DB, &users, "SELECT * FROM users")
    if err != nil {
        fmt.Println("Error querying the database:", err)
        return []model.User{}, errors.New("Database could not be queried")
    }

	return users, nil
}
```

Whereas previously we had `rows, err := DB.Query("SELECT id, name FROM users")` we now wrap that in the sqlx function: 

`err := sqlx.Select(DB, &users, "SELECT * FROM users")`

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

[Part 8.2 - Creating Users with a Post Request >>](/Part8/create_and_return_id.md)