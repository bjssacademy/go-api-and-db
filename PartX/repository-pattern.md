# Repository Pattern

> Work In Progress

What you may have noticed in our little tour is that we have to explicitly change the code when we want to call a different database. This is a bit of a shame really, as it means we are *tightly-coupled* to our database.

We can't easily change out our database for another one without updating a load of code. Sad times indeed.

Wouldn't it be good if we could change minimal code, maybe a line in a config file, and then it would automatically switch from in-memory to postgres, or mysql?

Fortunately larger brains than mine have thought about this and have come up with the *repository pattern*.

The repository pattern is a design pattern commonly used in software development, particularly in applications that interact with a data storage system such as a database. It provides an *abstraction layer* between the application's business logic and the data access code, encapsulating the logic required to access and manipulate data.

It does this by using an *interface*. First, you define an interface that represents the contract for data access operations. This interface typically includes methods for common CRUD (Create, Read, Update, Delete) operations and any other data access methods required.

Then you create *concrete* implementations of the repository *interface* for specific data storage systems. Each implementation contains the logic to interact with the corresponding data storage system.

You can now use the interface definition directly, by utilising *dependency injection*. That is, rather than the service layer being in charge of what dat layer it is using, we *tell* it which one to use, or *inject* it, normally via a constructor.

This is amazing news for us, as we don't have to touch the business logic or api layer at all to make this change.

## The Config

We're going to want to switch between configurations. I'm going to do this in code rather than reading from an actual config file for now.

```go
package config

type DatabaseConfig struct {
	Type     string // e.g., "postgres", "inmemory"
	Host     string
	User     string
	Password string
	SSLMode  string
	DBName   string
}

var InMemory DatabaseConfig = DatabaseConfig{Type: "inmemory", Host: "N/A", User: "", Password: ""}

var Postgres DatabaseConfig = DatabaseConfig{Type: "postgres", Host: "localhost", User: "postgres", Password: "yourpassword", SSLMode: "disable"}
```

The package config has a struct `DatabaseConfig`, which contains our properties. Then we have two public variables with an instance of `databaseConfig` - one for InMemory and one for Postgres - which contain the connection info.

Now, in our `main.go` file we can "load" a config:

```go
func main() {
    // Load configuration
    config := config.InMemory

    //...
}
```

## The Interface

We need a contract - an *interface* - for all concrete implementations to adhere to. The interface has functions it says every concrete implementation will have. We've named the struct `Repository` so it's easier to see what it is.

```go
package db

import (
	"acme/model"
)

type Repository interface {
    GetUsers() ([]model.User, error)
    GetUser(id int) (model.User, error)
    AddUser(user model.User) (id int, err error)
    UpdateUser(id int, user *model.User) (model.User, error)
    DeleteUser(id int) error
	Close()
}

```