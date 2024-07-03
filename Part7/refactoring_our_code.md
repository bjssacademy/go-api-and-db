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



# OLD

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {

    err = sqlx.Select(db.DB, &users, "SELECT * FROM users")

     if err != nil {
        fmt.Println("Error querying the database:", err)
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }

    json.NewEncoder(writer).Encode(users)
}

```
---

## Our tests

You now need to update your tests if you haven't already. Because we now inject our services, we need to recreate that.

You need to make sure you are importing everything:

```go
import (
	"acme/api"
	"acme/db/inmemory"
	"acme/model"
	"acme/service"
	"encoding/json"
	"io"
	"net/http"
	"net/http/httptest"
	"reflect"
	"testing"
)
```

### In the `TestGetUsersHandler` test

Change:

```go
    // Create a handler
    dbRepo := inmemory.NewInMemoryRepository();
    userService := service.NewUserService(dbRepo)
    userAPI := api.NewUserAPI(userService)

    handler := http.HandlerFunc(userAPI.GetUsers)
```

Remove: 

```go
    expectedJSON, err := json.Marshal(expected)
    if err != nil {
        t.Fatalf("Failed to marshal expected JSON: %v", err)
    }
```

and

```go
    // Check the response body
    if rr.Body.String() != string(expectedJSON) {
        t.Errorf("handler returned unexpected body: got %v want %v", rr.Body.String(), expected)
    }
```

### In the `TestGetUsersHandlerWithServer` test

Change:

```go
    //ARRANGE
	// Create a new server with the handler
    dbRepo := inmemory.NewInMemoryRepository();
    userService := service.NewUserService(dbRepo)
    userAPI := api.NewUserAPI(userService)

    server := httptest.NewServer(http.HandlerFunc(userAPI.GetUsers))
```

Save and run your tests to ensure they work using `go test -v`.

---

## Testing with Mocks

Now that we inject our db instance into the service, we can actually create our *own* instance that conforms the the repository interface if we wanted.

This allows us to separate our database implementation and create our own test version. What's the benefit of this?

Well we can set up our data any way we want it, we don't rely on the one supplied by the code. This is actually brilliant for us in terms of testing as we can try with different data, inspect our uwn instance, add extra methods just for us and so on.

Whilst there are packages that will do this for us, let's create our own mock. First, in the `db` folder create a new folder named `mock` and a file in it called `mock.go`.

We need to ensure this conforms to the `repository.go` interface. Add the following code:

```go
package mock

import (
	"acme/model"
)

type MockRepository struct {
	MockGetUsers    func() ([]model.User, error)
	MockGetUser     func(id int) (model.User, error)
	MockAddUser     func(user model.User) (int, error)
	MockUpdateUser  func(id int, user *model.User) (model.User, error)
	MockDeleteUser  func(id int) error
	MockClose       func()
}

func (m *MockRepository) GetUsers() ([]model.User, error) {
	return m.MockGetUsers()
}

func (m *MockRepository) GetUser(id int) (model.User, error) {
	return m.MockGetUser(id)
}

func (m *MockRepository) AddUser(user model.User) (int, error) {
	return m.MockAddUser(user)
}

func (m *MockRepository) UpdateUser(id int, user *model.User) (model.User, error) {
	return m.MockUpdateUser(id, user)
}

func (m *MockRepository) DeleteUser(id int) error {
	return m.MockDeleteUser(id)
}

func (m *MockRepository) Close() {
	m.MockClose()
}
```

### Using our mock

Add the following to you `main_test.go`:

```go
func TestGetUsersHandlerWithMock(t *testing.T) {

    req, err := http.NewRequest("GET", "/api/users", nil)
    if err != nil {
        t.Fatal(err)
    }

    rr := httptest.NewRecorder()

    expected := []model.User{
        {ID: 1, Name: "Alice"},
        {ID: 2, Name: "Bob"},
        {ID: 3, Name: "Terry"},
    }

    mockRepo := &mock.MockRepository{
		MockGetUsers: func() ([]model.User, error) {
			return expected, nil
		},
	}

    userService := service.NewUserService(mockRepo)
    userAPI := api.NewUserAPI(userService)
    
    handler := http.HandlerFunc(userAPI.GetUsers)

    handler.ServeHTTP(rr, req)

    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v", status, http.StatusOK)
    }

	var actual []model.User
    if err := json.Unmarshal(rr.Body.Bytes(), &actual); err != nil {
        t.Fatalf("Failed to unmarshal response body: %v", err)
    }

    if !reflect.DeepEqual(actual, expected) {
        t.Errorf("handler returned unexpected body: got %v want %v", actual, expected)
    }

}
```

#### What's going on?

The new code is: 

```go
    expected := []model.User{
        {ID: 1, Name: "Alice"},
        {ID: 2, Name: "Bob"},
        {ID: 3, Name: "Terry"},
    }

    mockRepo := &mock.MockRepository{
		MockGetUsers: func() ([]model.User, error) {
			return expected, nil
		},
	}
```

We set up our slice of what we expect back from the mock, as we have done before, *however* a new key piece is that these users are *not* hard-coded into our instance like they are in `inmemory.go`.

```go
mockRepo := &db.MockRepository{
```

This line creates a new instance of `MockRepository`, a struct designed to mock the `Repository` interface.

```go
MockGetUsers: func() ([]model.User, error) {
```

`MockGetUsers` is a field in the `MockRepository` struct. It is of type `func() ([]model.User, error)`, which matches the signature of the `GetUsers` method in the `Repository` interface.

The function defined here returns a slice of `model.User` and an `error`, *simulating* the behavior of the actual `GetUsers` method.

```go
return []model.User{
    {ID: 1, Name: "Alice"},
    {ID: 2, Name: "Bob"},
    {ID: 3, Name: "Terry"},
}, nil
```

Inside the `MockGetUsers` function, a slice of `model.User` containing three users (Alice, Bob, and Terry) is returned, along with a `nil` error.

This *simulates* a successful database query that retrieves a list of users.

This mocked version of the `Repository` interface can be injected into services or handlers that depend on it, allowing us to control and predict the behaviour of the `GetUsers` method without needing an actual database connection.

In short - we can *tell* the mock what to return in any instance without having to hard code anything into the mock struct!

### Mocks are powerful

Mocks are powerful because they allow us to test our code in isolation. 

Imagine we have a service that interacts with a database, an external API, or any other dependency. Testing this service can become really complicated and unreliable if we always have to rely on these external systems being available and in a specific state.

Now image we can mock that dependency, because we use an interface. We can separate our implementation from that of the third-party.

Mocks give you *complete control* over the dependencies. We can define exactly what data the mocks should return and how they should behave. As an example, we can simulate a database returning a specific set of users, or an API responding with an error. This allows us to test how the service handles different scenarios, including edge cases.

So now you know the basics of mocks!

---

[Part 8 - Connecting To A Real Database >>](/Part8/connecting_to_a_database.md)





