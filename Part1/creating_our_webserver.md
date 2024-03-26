# Creating our server

The first thing we are going to do in creating our API is set up Go to run as an HTTP web server. This handles our requests and returns responses, normally in JSON format.

Fortunately it's super easy to create a webserver in Go, as it has built-in packages for this, namely `net/http`.

## Our First Web Server

Let's create a new folder for our code and run `go mode init acme` to generate our `go.mod` file. Now create a new `main.go` file and import the `net/http` package:


```go
package main

import (
    "fmt"
    "net/http"
)
```

Okay, so now we need to use the package. The main work of this will be done using the `http.HandleFunc` function. This has two parameters, the *route* you want to serve, and the *function* that you want to handle requests on that route.

> If you've not worked with APIs before, then we'll need to talk about *routing*. 
>
> Routing is like having signposts or maps that tell you which roads (URLs) lead to which destinations (web pages). So, when you type in different addresses or click on different links on a website, the routing system knows where to guide you, just like how you follow signs or use a map to find your way to a specific place in a city.

Let's handle our default, or root path:

```go
func main() {

    http.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
        fmt.Fprintf(writer, "Hello, World!")
    })

```

Don't panic! That looks incomprehensible at first but let's break it down.

1. We have our `http.HandleFunc` function. This essentially says "when you get a request at this URL or *route*, then execute this code".
2. The first parameter is the route, in this case `"/"` which means the default, base path, or root. For example when you go to `www.google.com` without typing anything else, it executes the route matching `"/"`.
3. Next we have a function. This is an *anonymous* function - that is, we haven't created a separate named function outside of `main` to execute. This function has two parameters `http.ResponseWriter` and `*http.Request`, and executes the code `fmt.Fprintf(writer, "Hello, World!")`.

This is because the `http.HandleFunc` function requires its parameters to be of type string, and the function to have a particular signature. You can see from the intellisense:

```go 
HandleFunc(pattern string, handler func(http.ResponseWriter, *http.Request))
```

`http.ResponseWriter` is passed in as the variable `writer`, which we use to write our *response* back. For now we are just returning a string.

 `*http.Request` is passed in as the variable `request`. We'll use this later as it allows us to access information in the request from headers, to payload (in a POST request for example).

 Okay, so now we have that done, we still have a bit more wiring to do. We've got a route to handle requests, but we haven't started up our server yet.

 Let's do that now:

```go
func main() {

    http.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
        fmt.Fprintf(writer, "Hello, World!")
    })

    // Starting the HTTP server on port 8080
    fmt.Println("Server listening on port 8080...")
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        fmt.Println("Error starting server:", err)
    }
}

```

Okay, so here we are calling to the `http.ListenAndServe` method, telling it to start a server on port 8080, and tell us if it has any errors whilst doing so.

Save the file and run it by typing `go run main.go`. Open up a new browser tab and go to `http://127.90.0.1:8080` and you should see "Hello, World!" printed in your browser window!

NOTE: Don't forget to stop your server once you are done by using Ctrl+C in the terminal!

## Anonymous functions are fine sometimes...

Okay, we've got an anonymous function there. Now whilst this is fine, we probably don't want to do that for production as it makes our code hard to read and hard to maintain.

Let's extract it into its own function:

```go
func rootHandler(writer http.ResponseWriter, request *http.Request) {
    fmt.Fprintf(writer, "Hello, World!")
}
```

And tidy up our `main` to use it:

```go
func main() {

    http.HandleFunc("/", rootHandler)
    
    //rest of the code unchanged
    ...
}
```

We can already see our code is cleaner and easier to read an maintain.

Save the file and run it again to make sure everything still works!

[>> Part 2 - Creating an Endpoint](/Part2/users_endpoint.md)
