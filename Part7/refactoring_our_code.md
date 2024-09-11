# Refactoring our Code

Well, we've managed to create an HTTP server, take web requests, perform CRUD operations on a database. Good for us!

But our code is *ugly*. Yeah it works. But that's the simple part. Now we want it to be clean and usable.

By this point our `main.go` file is probably about 200 lines long, and we've mixed up our API code with our DB layer, making them hard to separate and therefore test separately.

We've also got a long-running single connection. Oh dear.

> :exclamation: This is going to break your tests. You can either refactor as we go, or see the solution at the end if you're not sure what's going on - in which case you are better off commenting them all out before you start.

## Refactor getUsers code

First off, let's tidy up our `getUsers` function. This code we used to marshall a struct and return it:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
    
    users := db.GetUsers()

	// Marshal users slice into JSON
	usersJSON, errMarshal := json.Marshal(users)
	if errMarshal != nil {
		// Handle error if marshalling fails
		http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
		return
	}

    writer.Header().Set("Content-Type", "application/json") 

    _, err := writer.Write([]byte(usersJSON))
    if err != nil {
        // Handle error if writing response fails
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }
}
```

We can replace the entire function with:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
    
    users := db.GetUsers()
    json.NewEncoder(writer).Encode(users)

}
```

## Multi-layered approach

We want to separate our business logic from our API logic. At the moment, our API routes directly handle the business logic of calling the database, as well as the unmarshalling and response.

We're going to have three layers - `api`, `service` and `db`.

> :exclamation: You'll need to update your tests to import the right packages and update some calls.

### 1. Create new api layer

Firstly create a new `api` folder, and a new file named `users-api.go`.

We want to copy across all the user-based functions - `updateSingleUser`, `deleteSingleUser`, `getSingleUser` and `createUser` and remove them from our `main.go` file.

---

You'll need to update your imports in both files:

#### `users-api.go`

```go
package api

import (
	"acme/db"
	"encoding/json"
	"fmt"
	"net/http"
	"strconv"
)

//all the functions below
```

#### `main.go`

```go
package main

import (
	"fmt"
	"net/http"
)

//all the other code

```

---

Now we have that, there should be a lot of red on your screen!

### 2. Update private to public 

First, we need to make our function in `users-api.go` *public". We go this by uppercasing the first letter of each method e.g `updateSingleUser` becomes `UpdateSingleUser`.

### 3. Import api package


Next we need to import our new `api` package into `main.go`:

```go
import (
	"fmt"
	"net/http"
    "acme/api"
)
```

### 4. User api package

We've still got some red. We now need to update our route handlers to use the api package:

```go
    router.HandleFunc("GET /", rootHandler)
    router.HandleFunc("GET /api/users", api.GetUsers)
    router.HandleFunc("POST /api/users", api.CreateUser)
    router.HandleFunc("GET /api/users/{id}", api.GetSingleUser)
    router.HandleFunc("DELETE /api/users/{id}", api.DeleteSingleUser)
    router.HandleFunc("PUT /api/users/{id}", api.UpdateSingleUser)
```

Save your files and check using ThunderClient everything works!

---

## Separate our logic from our API

Now we want to separate our API from our logic. We don't want our APi to have direct access to our DB layer.

> There are many reasons for this, relating to Separation of Concern, Security, Flexibility and re-use which we won't cover right now. Trust us, when we connect to a database for real, you'll understand!

### Service layer

Create a new folder named `service` and within it a file named `users-service.go`.

The first thing we are going to do is simply move the call to the DB in `users-api.go` function `GetUsers` to a function in the `users-service.go`:

#### `users-service.go`

```go
package service

import (
	"acme/db"
)

func GetUsersService() []db.User {
	return db.GetUsers()
}
```

#### `main.go`

```go
import (
	//others
    "acme/service"
)

// other code

func GetUsers(writer http.ResponseWriter, request *http.Request) {
    
    users := service.GetUsers()
    json.NewEncoder(writer).Encode(users)
    
}
```
---

Okay, we've now got a service layer the api calls, and a db layer the service layer calls!

Save all your files and run the server, using ThunderClient to check it all still works.

## Error Handling

Whilst we are here, we don't currently handle errors. We're assuming everything is just always going to be great.

Because we have separated our layers, we need to handle errors from the DB up.

1. Update `inmemory.go` to return two types

```go
func GetUsers() ([]User, error) {
	return users, nil
}
```

2. Update `users-service.go` to handle errors and throw a new, non-technical error

> NOTE: Don't forget to import the **errors** package

```go
func GetUsers() ([]db.User, error) {
	users, err := db.GetUsers()

	if err != nil {
		fmt.Println("Error getting users from DB:", err)
		return nil, errors.New("There was an error getting the users from the database.")
	}

	return users, nil

}
```

3. Update `users-api.go` to handle errors returned from the service layer

```go
func GetUsers(writer http.ResponseWriter, request *http.Request) {
    
    users, err := service.GetUsers()

    if err != nil {
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }

    json.NewEncoder(writer).Encode(users)
    
}
```

Save your code and let's check it all works!

---

## Removing (some) Duplication

We've got some duplication (welcome to Go, they quite like it). Let's move some common things out.

We parse the `id` twice separately, and parse teh body three times. So we can extract them into their own functions:

```go
func parseId(idStr string) (id int, err error){
    
    id, err = strconv.Atoi(idStr)
    if err != nil {
        fmt.Println("Error parsing ID:", err)
        return 0, err
    }

    return id, nil

}

func decodeUser(body io.ReadCloser) (user db.User, err error) {
    
    err = json.NewDecoder(body).Decode(&user)
    if err != nil {
        fmt.Println("Error decoding request body:", err)
        return db.User{}, err
    }

    return user, nil
}
```

And utilise it in our functions at our api layer:

```go
func UpdateSingleUser(writer http.ResponseWriter, request *http.Request) {

    id, err := parseId(request.PathValue("id"))

    if err != nil {
        http.Error(writer, "Bad Request ID", http.StatusBadRequest)
        return
    }

    user, err := decodeUser(request.Body)

    if err != nil {
        http.Error(writer, "Bad Request Body", http.StatusBadRequest)
        return
    }

    updated := db.UpdateUser(id, user)

    json.NewEncoder(writer).Encode(updated)

}

func DeleteSingleUser(writer http.ResponseWriter, request *http.Request) {

    id, err := parseId(request.PathValue("id"))

    if err != nil {
        http.Error(writer, "Bad Request ID", http.StatusBadRequest)
        return
    }

    deleted := db.DeleteUser(id)

    json.NewEncoder(writer).Encode(deleted)

}

func GetSingleUser(writer http.ResponseWriter, request *http.Request) {
    
    id, err := parseId(request.PathValue("id"))
    
    if err != nil {
        http.Error(writer, "Bad Request ID", http.StatusBadRequest)
        return
    }

    user := db.GetUser(id)

    json.NewEncoder(writer).Encode(user)

}

func CreateUser(writer http.ResponseWriter, request *http.Request) {
    
    user, err := decodeUser(request.Body)

    if err != nil {
        http.Error(writer, "Bad Request ID", http.StatusBadRequest)
        return
    }

    id := db.AddUser(user)
    writer.WriteHeader(http.StatusCreated)
	fmt.Fprintf(writer, "User created successfully: %d", id)

}
```

---

## Updating in-memory.go to return errors

I'm going to provide all the code here for `inmemory.db` as it's boring to update:

```go
package db

import (
	"errors"
	"slices"
)

type User struct {
	ID   int    `json:"id" db:"id"`
	Name string `json:"name" db:"name"`
}

var count int = 3
var users []User

func init() {
	// Initialize the in-memory database with some sample data
	users = []User{
		{ID: 1, Name: "User 1"},
		{ID: 2, Name: "User 2"},
		{ID: 3, Name: "User 3"},
	}
}

func GetUsers() ([]User, error) {
	return users, nil
}

func AddUser(user User) (id int, err error) {
	count++
	user.ID = count

	users = append(users, user)

	return count, nil
}

func GetUser(id int) (User, error) {
	var user User

	for _, user := range users {
		if user.ID == id {
			return user, nil
		}
	}

	return user, errors.New("User id not found.")

}

func DeleteUser(id int) error {

	for index, user := range users {
		if user.ID == id {
			users = slices.Delete(users, index, index +1)
			return nil
		}
	}

	return errors.New("USer id not found to delete.")

}

/*
WITHOUT using pointers
*/

func UpdateUser(id int, updatedUser User) (User, error) {

	for index, user := range users {
		if user.ID == id {
			users[index].Name = updatedUser.Name
			return user, nil
		}
	}

	return User{}, errors.New("User id not found to update.")

}
```
---

## Task

Fix all the broken code! By updating the `inmemory.db` to return errors, your api service layer needs to be updated!

---

## Refactor the DeleteSingleUser function

Let's take the next simplest thing, the `DeleteSingleUser()` function.

1. Create the `Deleteuser` function

In `users-service.go` add the new function:

```go
func DeleteUser(id int) error {
	err := db.DeleteUser(id)

	if err != nil{
		fmt.Println("Error deleting user from DB:", err)
		return errors.New("Could not delete user")
	}

	return nil
}
```

And update our `users-api.go` function:

```go
func DeleteSingleUser(writer http.ResponseWriter, request *http.Request) {

    id, err := parseId(request.PathValue("id"))

    if err != nil {
        http.Error(writer, "Bad Request ID", http.StatusBadRequest)
        return
    }

    err = service.DeleteUser(id)
    
    if err != nil {
        http.Error(writer, "Could not delete user", http.StatusBadRequest)
        return
    }

    writer.WriteHeader(http.StatusOK)

}
```

---

## Tasks

1. Refactor so the service layer is the only layer that calls the database layer, and the api layer only calls the service layer.

2. Create a new `model` folder, and move the `User` struct into it from the `inmemory` file. Fix all the references!

------

[Part 8 - Connecting To A Real Database >>](/Part8/connecting_to_a_database.md)





