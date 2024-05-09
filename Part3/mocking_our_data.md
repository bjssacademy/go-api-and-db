# Mocking Our Database

At the moment our responses are hard-coded. In reality, we'd be connecting to a database to return that data.

## Mock Our Users

Create a new folder `db`, and in that folder create a new file, `inmemory.go`.

We'll move our code that is responsible for data into this file:

```go
package db

type User struct {
    ID   int    `json:"id" db:"id"`
    Name string `json:"name" db:"name"`
}

var users []User

func init() {
    // Initialize the in-memory database with some sample data
    users = []User{
        {ID: 1, Name: "User 1"},
        {ID: 2, Name: "User 2"},
        {ID: 3, Name: "User 3"},
    }
}

func GetUsers() []User {
	return users
}
```

We've moved our `User` struct out of our `main.go`, and added a *package-level slice variable* `users` which is private.

Next we have the `init` function, which fills our slice with users.

---

### The `init` function

> The init function in Go is a special function that is called automatically, without being explicitly invoked, when a package is initialized. Specifically, the init function is executed in two situations:
> 
> **Package Initialization** - When a Go program starts, the package-level variables and `init` functions are initialized in the order in which they are declared. If a package imports other packages, the imported packages are initialized first, and then the importing package's `init` function is called.
>
> **Multiple Initializations** - If a package is imported multiple times in a program, the `init` function is called only once, regardless of how many times the package is imported. This ensures that initialization code is executed only once per package.

---

Finally we have the `GetUsers()` function that returns the entire slice of `Users[]`.

## Using Our InMemory DB

Now, back in our `main.go` file, we add the package to our imports:

```go
import (
	"acme/db"
	"encoding/json"
	"fmt"
	"net/http"
)
```

> NOTE: The `Users` struct should have been removed from main as it now exists in out `inmemory.go` file. If not, remove it now.

Change the following line to call the function in our db package:

```go
users := []User{
		{ID: 1, Name: "Alice"},
		{ID: 2, Name: "Bob"},
	}
```

to be:

```go
users := db.GetUsers()
```

Now when we run our code and go to the `/api/users` endpoint, we see there are 3 users listed, which matches our in-memory Users slice.

---

[Part 4 - Creating a new user >>](/part4/posting_and_creating.md)

