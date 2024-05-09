# Create A User In The DB

We've manually inserted data into our users table, but we've got a perfectly good post request that currently only updated our in-memory database, rather than our postgres database. Let's fix that.

In your `postgres.go` file, add the following function:

```go
func AddUser(user model.User) (id int, err error) {

}
```

Now we need to call the `INSERT` statement like we did on the command line, but with the name we have been given:

```go
func AddUser(user model.User) (id int, err error) {

    err = DB.QueryRow("INSERT INTO users (name) VALUES ($1) RETURNING id", user.Name).Scan(&id)
    if err != nil {
        fmt.Println("Error inserting user into the database:", err)
        return 0, errors.New("Could not insert user")
    }

    return id, nil   
}
```

The query also includes a RETURNING clause, which instructs the database to return the id of the newly inserted row.

After executing the query, the `Scan` method is called on the `QueryRow` result. This method scans the result row and assigns the value of the id column to the variable `id`.

Now we need to update our `users-service.go` to call the postgres function rather than the in-memory one:

```go
func CreateUser(user model.User) (id int, err error) {
	
    id, err = postgres.AddUser(user)  // << was db.AddUser(user)

	if err != nil {
		fmt.Println("Error creating user in DB:", err)
		return 0, errors.New("Could not create user")
	}

	return id, nil
}
```

Now, save all your changes, and run your webserver. Now, when you POST using ThunderClient to the endpoint, you should get a success message, and then if you run the GET request, you will see them listed!

---

Add the following functions to `postgres.go` and implement the SQL:

1. GetUser() - gets a single user based on id
2. DeleteUser() - deletes a user based on id
3. UpdateUser() - updates the user name based on id

Make sure you update your `users-service.go` to call the postgres instance!

---

