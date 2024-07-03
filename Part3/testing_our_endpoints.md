# Testing Our Endpoints

:exclamation: This section assumes you have covered the [Go testing basics](https://github.com/bjssacademy/go-testing-basics)

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

Finally we need to assert the response. In Go, there is no built in assertion library, you have to do it through basic Go code. The creators of Go make many good points as to why - I don't agree with all of them, but I understand.

Since we are doing as much as possible with minimal external libraries, we'll do it the Go way:

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

> **Side note**: This is a bit odd isn't it? ```if status := rr.Code; status != http.StatusOK {```? We have a single line that assigns the code to the variable `status` and then checks that status is the HTTP OK status.
>
> Why don't we assign outside the if block? Or why are we using it at all? Well, the answer is "idiomatic Go" as it always is. Using this style is preferred in idiomatic Go for single-use variables.

Okay, save your file and run `go test -v` to see the test run!

----

## My First Integration Test

Okay, now we have the test for the handler itself, but that test would still pass even if we never called it in our routing. To check our ful stack is wired together we need to check the HTTP call to the API endpoint itself.

First, we need to create a new test:

```go
func TestRootHandlerWithServer(t *testing.T) {
    
}
```

Now let's arrange our data:

```go
func TestRootHandlerWithServer(t *testing.T) {
    //ARRANGE
	// Create a new server with the handler
    server := httptest.NewServer(http.HandlerFunc(rootHandler))
    defer server.Close()
}
```

This creates an in-memory HTTP server using `httptest.NewServer`, and `rootHandler` is the HTTP handler function being tested.

`server.Close()` ensures that the server is closed once the test finishes, even if the test fails or panics.

Now let's act upon our instance:

```go
func TestRootHandlerWithServer(t *testing.T) {
    //ARRANGE
	// Create a new server with the handler
    server := httptest.NewServer(http.HandlerFunc(rootHandler))
    defer server.Close()

	//ACT
    // Send a GET request to the server
    resp, err := http.Get(server.URL + "/")
    if err != nil {
        t.Fatalf("Failed to send GET request: %v", err)
    }
    defer resp.Body.Close()

}
```

This ends an HTTP GET request to the server's root endpoint (/).If the request fails, the test is terminated with `t.Fatalf`, which logs the error and stops the test.

Now we have our response, we need to assert the response we got. We'll start with the status code:

```go
func TestRootHandlerWithServer(t *testing.T) {
    //ARRANGE
	// Create a new server with the handler
    server := httptest.NewServer(http.HandlerFunc(rootHandler))
    defer server.Close()

	//ACT
    // Send a GET request to the server
    resp, err := http.Get(server.URL + "/")
    if err != nil {
        t.Fatalf("Failed to send GET request: %v", err)
    }
    defer resp.Body.Close()

    // Check the status code
    if status := resp.StatusCode; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v", status, http.StatusOK)
    }

```

We've seen this before so I won't go into it. Now let's check the response body:

```go
func TestRootHandlerWithServer(t *testing.T) {
    //ARRANGE
	// Create a new server with the handler
    server := httptest.NewServer(http.HandlerFunc(rootHandler))
    defer server.Close()

	//ACT
    // Send a GET request to the server
    resp, err := http.Get(server.URL + "/")
    if err != nil {
        t.Fatalf("Failed to send GET request: %v", err)
    }
    defer resp.Body.Close()

    // Check the status code
    if status := resp.StatusCode; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v", status, http.StatusOK)
    }

    // Check the response body
    expected := "Hello, World!"
    bodyBytes, err := io.ReadAll(resp.Body)
    if err != nil {
        t.Fatalf("Failed to read response body: %v", err)
    }
    if string(bodyBytes) != expected {
        t.Errorf("handler returned unexpected body: got %v want %v", string(bodyBytes), expected)
    }
}
```

Here we read the response body into `bodyBytes` variable. If reading the response body fails, the test is terminated with `t.Fatalf`.

If it doesn't fail, we convert `bodyBytes` to a string and compares it to the expected string `"Hello, World!"`.

If the body content does not match the expected value, it logs an error with `t.Errorf`.

Go ahead and run your tests with `go test -v`.

---

## More Complex Tests

Those are very simple tests that just check for the status code and a string, however our API is mainly going to accept and return JSON.

We only have one endpoint at the moment that does this and that's the one dealing with users.

Our handler test is going to look very similar to start with:

```go
func TestGetUsersHandler(t *testing.T) {
    // ARRANGE
	// Create a new HTTP request
    req, err := http.NewRequest("GET", "/api/users", nil)
    if err != nil {
        t.Fatal(err)
    }

    // Create a new response recorder
    rr := httptest.NewRecorder()

    // Create a handler
    handler := http.HandlerFunc(getUsers)

}
```

So far, so very similar!

Let's do something more interesting. We know that the users endpoint will return a list of 3 users as defined in our `inmemory.go` file:

```go
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

So let's arrange our expected result:

```go
func TestGetUsersHandler(t *testing.T) {
    // ARRANGE
	// code as before
    //....

	//Arrange our expected response
	expected := []db.User{
		{ID: 1, Name: "User 1"},
		{ID: 2, Name: "User 2"},
		{ID: 3, Name: "User 3"},
	}

	expectedJSON, err := json.Marshal(expected)
    if err != nil {
        t.Fatalf("Failed to marshal expected JSON: %v", err)
    }

}
```

So we have created our own slice of `User`, and serialized (Marshaled in Go-speak) it to JSON in the variable `expectedJSON`.

Next up, let's act:

```go
func TestGetUsersHandler(t *testing.T) {
    // ARRANGE
	// code as before
    //...
    
	//Arrange our expected response
	// code as before
    //...

	//ACT
    // Serve the request
    handler.ServeHTTP(rr, req)

}
```

Okay, so we've sent a GET request to our endpoint and get a response back; let's assert that what we get back is what we expected. Add the following code after the `handler.ServeHTTP(rr, req)` line:

```go
    //ASSERT
    // Check the status code
    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v", status, http.StatusOK)
    }

    // Check the response body
    if rr.Body.String() != string(expectedJSON) {
        t.Errorf("handler returned unexpected body: got %v want %v", rr.Body.String(), expected)
    }
```

The only thing here is checking the response body. This example compares the JSON of both as *strings* by converting the `expectedJSON` to a string.

Go ahead and save you file and run your tests.

---

## Improving our test

Comparing JSON responses by directly comparing their string representations is not always the best approach, as differences in formatting (e.g., whitespace, ordering of keys) can cause such comparisons to fail even when the JSON structures are logically equivalent. A better approach is to deserialize (unmarshal) the JSON responses into Go data structures and then compare those structures.

Let's add that to the end of our test:

```go
    var actual []db.User
    if err := json.Unmarshal(rr.Body.Bytes(), &actual); err != nil {
        t.Fatalf("Failed to unmarshal response body: %v", err)
    }

    if !reflect.DeepEqual(actual, expected) {
        t.Errorf("handler returned unexpected body: got %v want %v", actual, expected)
    }
```

> :exclamation: You will need to add an import for `"reflect"`

Here we read and unmarshal the response body from `rr.Body.Bytes()` into a slice of `db.User` (`actual`).

We then use `reflect.DeepEqual` to compare the unmarshaled response (`actual`) with the expected data (`expected`).

Doing this ensures that the JSON responses are compared based on their *logical content* rather than their string representation, making the test more robust and less prone to false negatives due to minor formatting differences.

The bonus of this in more complex testing end-to-end scenarios is that you can take the instance and alter it (for instance if you got a single user back, then updated their name to use in a PUT request).

---

Here we have covered the basics of unit and integration testing. Now, why not write the *integration test* for the users endpoint? It should only take you a couple of minutes.

---

[Part 4 - Creating a new user >>](/Part4/posting_and_creating.md)