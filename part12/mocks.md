# Testing with Mocks
Now that we inject our db instance into the service, we can actually create our *own* instance that conforms the the repository interface if we wanted.

This allows us to separate our database implementation and create our own test version. 

What's the benefit of this?
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

## Using our mock

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

### What's going on?

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

## Mocks are powerful

Mocks are powerful because they allow us to test our code in isolation. 

Imagine we have a service that interacts with a database, an external API, or any other dependency. Testing this service can become really complicated and unreliable if we always have to rely on these external systems being available and in a specific state.
Now image we can mock that dependency, because we use an interface. We can separate our implementation from that of the third-party.

Mocks give you *complete control* over the dependencies. We can define exactly what data the mocks should return and how they should behave. 

As an example, we can simulate a database returning a specific set of users, or an API responding with an error. This allows us to test how the service handles different scenarios, including edge cases.

So now you know the basics of mocks!