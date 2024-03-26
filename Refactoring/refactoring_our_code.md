# Refactoring our Code

Well, we've managed to create an HTTP server, take web requests, perfomr CRUD operations on a database. Good for us!

But our code is *ugly*. Yeah it works. But that's the simple part. Now we want it to be clean and usable.

By this point our main.go file is probably about 200 lines long, and we've mixed up our API code with our DB layer, making them hard to separate and therefore test separately.

We've also got a long-running single connection. Oh dear.

## Refactor getUsers code

First off, let's tidy up our `getUsers` function. This code we used to marshall a struct and return it:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
    
    //all the code before
    
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

We can replace the entire function with:

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

## Extract DB Functionality

## 