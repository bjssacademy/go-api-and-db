# Multiplexing

Ok, so now we can send Post and Get requests to our endpoint. But it we want to updated a user's name with a Get request, or send a Delete request we'll have to write a lot of "if" statements.

Fortunately in Go (from 1.22) there's an update to allow multiple handlers on a single route, or endpoint.

Before Go 1.22 there was a _multiplexer_ that offered rudimentary path matching, but not much else. This meant a proliferation of third-party libraries to provide more powerful routing options (such as gorilla/mux and chi).

Multiplexers are used for routing incoming requests depending on the path and HTTP method to the correct _handler_. They do other things too such as supporting middleware (maybe session-based auth, or logging all requests). They also make it far easier to accept parameters on the path, such as `api/users/123` to get a specific user.

## Starting with a Multiplexer

Okay, first off we are going to need to change our code. Currently we have in our main():

```go
func main(){

    //DB connection code

    http.HandleFunc("/", rootHandler)
	http.HandleFunc("/api/users", handleUsers)

    ///rest of the code

}
```

There are a couple of steps we need to refactor our code. First we need to extract our code that deals with the Post to its own function. Let's call it `createUser()`, and copy the code that deals with Posts. Don't forget to remove the corresponding code from your handleUsers() function! We have no need for the `if` statement now, and I have commented it out from below but you can safely delete it:

```go
func createUser(writer http.ResponseWriter, request *http.Request) {

    //if request.Method == http.MethodPost {

    var user db.User
    err := json.NewDecoder(request.Body).Decode(&user)
    if err != nil {
        fmt.Println("Error decoding request body:", err)
        http.Error(writer, "Bad Request", http.StatusBadRequest)
        return
    }

    id := db.AddUser(user)
    writer.WriteHeader(http.StatusCreated)
	fmt.Fprintf(writer, "User created successfully: %d", id)

        //return
    //}

}
```

Let's also update out `handleUsers()` function to be something that explains that it's not for all user requests, and rename it `getUsers` instead:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
    //remaining code as before to handle Get requests
}
```

Great. Now that's done, we have two separate functions for handling requests to the same `api/users` endpoint, so we need to set up our multiplexer.

```go
func main() {

    router := http.NewServeMux()

    router.HandleFunc("GET /", rootHandler)
    router.HandleFunc("GET /api/users", getUsers)
    router.HandleFunc("POST /api/users", createUser)

    // Starting the HTTP server on port 8080 and providing router variable to ListenAndServe
    fmt.Println("Server listening on port 8080...")
    err := http.ListenAndServe(":8080", router)
    if err != nil {
        fmt.Println("Error starting server:", err)
    }

}
```

NOTE: We are passing the `router` variable now to the `ListenAndServe` function, whereas we previously provided `nil`.

## What's new?

In the first handler, the HTTP method (GET in this case) is specified explicitly as part of the pattern. This means that this handler will only trigger for GET requests to paths beginning with `/api/users`, not for other HTTP methods. In the same way, our third handler will only work for POST requests.

Let's save our file and run our server, and test it our with Thunder Client for Get and Post requests. Everything should be working as normal!

## Pattern Matching - getting information from the URL

Okay, now let's say we want to get a single user from the DB, based on their _id_. As in most APIs you provide this as part of the URL path, eg `api/users/1` will return the single user whose id matches 1 in the database.

To do this, we have to provide the parameter we want from the path in a special format using curly braces and a variable name between them in the places we expect in the path to our multiplexer. In our example, we want `api/users/{id}`.

Let's add a new handler to our code:

```go
router.HandleFunc("GET /api/users/{id}", getSingleUser)
```

Now we need to add our new function `getSingleUser`:

```go
func getSingleUser(writer http.ResponseWriter, request *http.Request) {

    idStr := request.PathValue("id")
    fmt.Fprintf(writer, "handling task with id=%v\n", idStr)

}
```

The code we're interested in here is the one that extracts the value `id` from the _path_: `idStr := request.PathValue("id")`.

> Things passed on the URL path are always _strings_.

Go ahead and save the file, and create a new POST request in Thunder Client to test it out with the endpoint `api/users/1`.

## Retrieving a single user from the database

Now we have extracted it from the path and stored it in a variable, we can query the DB for it.

There are a couple of things we need to do here that you many not have come across.

Because path variables are all strings, we need to try and convert the provided `{id}` value to an int, as that's the type we have stored in the DB column for id.

> If we passed an invalid value, we'd get a SQL error. But even more importantly, it might leave us open to _SQL injection attacks_, so we are validating our inputs.

To do this we use the [strconv package](https://pkg.go.dev/strconv) which we add to our imports:

```go
package main

import (
    "fmt"
    "net/http"
    "encoding/json"
    "acme/db"
    "strconv"
)
```

The `strconv` package allows you to convert from string to int, in to string. It also has a `Parse` function for bool, int, floats and so on. We're going to use it to check that the provided `{id}` value is a valid integer.

```go
func getSingleUser(writer http.ResponseWriter, request *http.Request) {

    idStr := request.PathValue("id")

    id, err := strconv.Atoi(idStr)
    if err != nil {
        fmt.Println("Error parsing ID:", err)
        http.Error(writer, "Bad Request", http.StatusBadRequest)
        return
    }

}
```

As you can see, the `strconv.Atoi()` function return an error if the passed value could not be converted to a valid integer, which we catch and return a Bad request error (status code 400).

Now that we have a valid `id` that is an integer, we can query the mock database.

To do that, in your `inmemory.go` file, add a new function `GetUser()`:

```go
func GetUser(id int) User {

}
```

Now, because this is Go, the idiomatic way is to range over the slice, and pull out the correct response:

```go
func GetUser(id int) User {
	var user User

	for _, user := range users{
		if user.ID == id {
			return user
		}
	}

	return user

}
```

Now we need to call this function from our `getSingleUser` function in `main.go`:

```go
func getSingleUser(writer http.ResponseWriter, request *http.Request) {

    idStr := request.PathValue("id")

    id, err := strconv.Atoi(idStr)
    if err != nil {
        fmt.Println("Error parsing ID:", err)
        http.Error(writer, "Bad Request", http.StatusBadRequest)
        return
    }

    user := db.GetUser(id)

    json.NewEncoder(writer).Encode(user)

}
```

---

Okay, let's save our files and check that we can get a single user back from our database with Thunder Client. Create a new GET request for `http://127.0.0.1/api/users/1` and inspect the response.

## Workshop

Whilst we can Create and Read a user, at the moment we're missing 2 parts of CRUD - the Update and Delete!

1. Create a function to handle the deletion of a user when provided a valid id, using the DELETE HTTP Method

> You may need to import package `slices` and use `slices.Delete`. There are other ways using arcane reslicing.

2. Create a function to handle the update of a user name, when provided an id and a User struct as JSON using the PUT HTTP Method.

> This can be confusing as you cannot directly update the user when you range over a slice in Go, as user will be a copy:
>
> Here's an example of code that looks like it works, but when you call GET again, you will see it has NOT been updated:
>
> ```go
> for index, user := range users {
> 		if user.ID == id {
> 			user.Name = updatedUser.Name
> 			return user
> 		}
> 	}
> ```

---

[>> Part 6 - Middleware & CORS](/Part6/middleware_and_cross_origin_requests.md)
