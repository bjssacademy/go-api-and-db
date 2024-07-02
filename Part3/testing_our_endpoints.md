# Testing Our Endpoints

Well, we have strayed from our TDD routes here a bit. We've violated the *test-first* aspect.

But you might end up on a project and have to do some POUTing (Plain Old Unit Testing). We'll show you why that is less preferable.

There are 2 main ways to test your API endpoints - by unit testing the individual HTTP handlers or by spinning up the webserver itself and testing the full-stack implementation from the HTTP call down to the database and the response.

We do this using the built-in library `httptest` and the methods on it `httptest.NewRecorder` and `httptest.NewServer`.

## httptest.NewRecorder vs. httptest.NewServer

### httptest.NewRecorder

- Purpose: Used for *unit testing* individual HTTP handlers.
- Efficiency: Does not start an actual HTTP server, making tests faster and consuming fewer resources.
- Simplicity: Directly tests the handler functions without involving the networking stack.

### httptest.NewServer
- Purpose: Used for *integration testing*, where you need to test the entire server stack, including middleware, routing, and network.
- Overhead: Starts an actual HTTP server, which can be slower and consume more resources.
- Use Case: Suitable for end-to-end tests or when you need to simulate real HTTP requests and responses.

>For most handler-level unit tests, `httptest.NewRecorder` is preferred because it directly exercises the handler logic without the overhead of starting a server.

---

## Which one do I use?

It depends what you are interested in. Normally your integration tests are created by testers who treat the system more like an end user, or consumer, of your API.

It'll make more sense later when we split our services and api out and then introduce mocks as to what you should use when.

However, for the purposes of understanding we are going to use both!

## My First API Handler Test

Okay, first off, let's add a new file in the same directory as our `main.go` file called `main_test.go` to test our root handling:

```go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestRootHandler(t *testing.T) {

}
```

Okay, we're going to want to arrange what we need first. And the first thing we need is an *httprequest* to send to our rooHandler function:

```go
func TestRootHandler(t *testing.T) {
    req, err := http.NewRequest("GET", "/", nil)
    if err != nil {
        t.Fatal(err)
    }
}
```

So that's our *request*, which is a GET request to the root (`/`).

Now we need to create a new instance of the `httptest.NewRecorder`:

```go
func TestRootHandler(t *testing.T) {
    req, err := http.NewRequest("GET", "/", nil)
    if err != nil {
        t.Fatal(err)
    }

    // Create a new response recorder
    rr := httptest.NewRecorder()
}
```

Excellent, so we have our request and our response recorder. Next we need our actual HTTP handler for the rootHandler function:

```go
// Create a handler
handler := http.HandlerFunc(rootHandler)
```

Now we have all our pieces. We have an HTTP Handler that uses the rootHandler, a response recorder, and an http request.

Now we need to serve the request:

```go
//ACT
// Serve the request
handler.ServeHTTP(rr, req)
```

This is shorthand for saying "Hey there HTTP Handler! I would like you to process this httpRequest with this responseWriter."

Because our `rootHandler` function looks like this:

```go
func rootHandler(writer http.ResponseWriter, request *http.Request) 
```

We are directly instantiating it to deal with our request, rather than having to actually *start* our web server to handle the request at the API layer and do any routing.

Of course, this means we are *not* checking our routing, just that *if* we get a request on a certain path, then we will check that it is processed correctly, not that everything is wired together!

Finally we need to assert the response:

```go
    //ASSERT
    // Check the status code
    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v", status, http.StatusOK)
    }

    // Check the response body
    expected := "Hello, World!"
    if rr.Body.String() != expected {
        t.Errorf("handler returned unexpected body: got %v want %v", rr.Body.String(), expected)
    }
```

Okay, save your file and run `go test -v` to see the test run!

----

[Part 4 - Creating a new user >>](/Part4/posting_and_creating.md)