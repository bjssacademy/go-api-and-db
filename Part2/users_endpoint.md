# Creating our /api/users endpoint

Okay, now we have our basic server up and running, let's add a new handler for the route `api/users`:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
	
    fmt.Printf("got /api/users request\n")
	io.WriteString(writer, "a list of all users from the db")

}
```

NOTE: We have changed to use `io.WriteString` here as it's more efficient and clearer that using the `fmt` package to return strings.

And add a new handler in the `main` function:

```go
func main(){

    http.HandleFunc("/", rootHandler)
	http.HandleFunc("/api/users", getUsers)

    ///rest of the code is unchanged

}
```
If you now save the file and run the server again, you can go to `http://127.0.0.1:8080/api/users` and should see the string printed out!

Obviously this doesn't help us very much at the moment. We've not actually returned anything from the database, or even structured JSON. Let's first check what we want to return and make sure we can get JSON back.

## Returning JSON

Let's bodge our way to returning a JSON structure by using a string representation of a JSON object:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
    
    fmt.Printf("got /api/users request\n")

    //1. We define a usersJSON string containing the JSON representation of the list 
    //of users. This JSON string directly represents the users' data without needing 
    //to define a struct or marshal it.
    usersJSON := `[{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]` 
    
    //2. We set the Content-Type header of the response to application/json to 
    //indicate that the response contains JSON data.
    writer.Header().Set("Content-Type", "application/json") 

    //3. Finally, we write the JSON response string directly to the 
    //http.ResponseWriter using writer.Write([]byte(usersJSON)).
    _, err := writer.Write([]byte(usersJSON))
    if err != nil {
        // Handle error if writing response fails
        http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
        return
    }
}
```

1. We define a usersJSON string containing the JSON representation of the list of users. This JSON string directly represents the users' data without needing to define a struct or marshal it.

2. We set the Content-Type header of the response to application/json to indicate that the response contains JSON data.

3. Finally, we write the JSON response string directly to the http.ResponseWriter using `writer.Write([]byte(usersJSON))`.

Okay, let's save our code and give it a go! Now if you go to the `api/users` endpoint you get back a JSON object.

## Serializing - "Marshalling" in Go 

Okay, so if we had static data that never changed we could use a string as we have done above. However, that seems very unlikely considering we want to get the data from a database.

We'll probably be working on data that has a structure, and - as in many other languages - to convert an object to JSON you need to *serialize* it. In go this is called *marshalling* (and deserializing from JSON to an object is called *unmarshalling*).

To return JSON from an object we're going to need to import the `encoding/json` package:

```go
package main

import (
    "fmt"
    "net/http"
    "encoding/json"
)
```

Next, because we want to return *all* users, we'll need a struct to represent the data we want to return. For now, we'll just assume our users have an ID and a Name rather than any other data.

```go
type User struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}
```

Now we have a structure to represent a single user, we are going to want to update our `getUsers` function, Remove the line `usersJSON := '[{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]' ` and replace it with:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
	fmt.Printf("got /api/users request\n")

	// Hardcoded list of users
	users := []User{
		{ID: 1, Name: "Alice"},
		{ID: 2, Name: "Bob"},
	}

    //rest of our code

}
```

Okay, so now we have a slice of our hardcoded users, Alice and Bob. Next we need to marshall that to be JSON, using the `json.Marshal()` method:

```go
func getUsers(writer http.ResponseWriter, request *http.Request) {
	fmt.Printf("got /api/users request\n")

	// Hardcoded list of users
	users := []User{
		{ID: 1, Name: "Alice"},
		{ID: 2, Name: "Bob"},
	}

    // Marshal users slice into JSON
	usersJSON, err := json.Marshal(users)
	if err != nil {
		// Handle error if marshalling fails
		http.Error(writer, "Internal Server Error", http.StatusInternalServerError)
		return
	}

    //rest of our code

}

```

The magic is essentially `usersJSON, err := json.Marshal(users)`, where you get a serialized JSON object back from your slice of objects. If you want, this has done the job or turning it into a byte array for us!

Finally, we just need to change one line where we write the response to the `writer`, changing `[]byte(usersJSON)` to just `usersJSON` and the marshalling does the conversion to a byte array for us:

```go
    // Write JSON response to the ResponseWriter
	_, err = writer.Write(usersJSON)
```

Okay, let's check everything still works and we still get JSON back in the browser!

---

[>> Part 3 - Connecting to our Database](/Part3/connecting_to_a_database.md)