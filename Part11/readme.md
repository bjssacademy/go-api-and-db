# Part 11 - Environment Variables

Environment variables are a key-value pair used by operating systems to store configuration settings and information that can be accessed by applications and processes. They provide a way to pass configuration data to applications without hardcoding them in the code, making applications more portable and easier to manage in different environments.

## Setting and Getting Environment Variables
### Unix/Linux
#### Setting Environment Variables

In Unix-like systems (such as Linux and macOS), you can set environment variables in the shell using the export command.

Temporarily for the current session:

```bash
export MY_VARIABLE="some_value"
```

Persistently for future sessions:

Add the export command to your shell's profile script `(~/.bashrc, ~/.bash_profile, ~/.zshrc, etc.)`, depending on the shell you are using.

```bash
echo 'export MY_VARIABLE="some_value"' >> ~/.bashrc
source ~/.bashrc
```
#### Getting Environment Variables

You can retrieve the value of an environment variable using the echo command combined with the variable name prefixed by a dollar sign $.

Example:
```bash
echo $MY_VARIABLE
```

### Windows
#### Setting Environment Variables

In Windows, you can set environment variables through the Command Prompt, PowerShell, or the System Properties interface.

Temporarily for the current Command Prompt session:

```bash
set MY_VARIABLE=some_value
```

Using PowerShell for the current session:

```bash
$env:MY_VARIABLE="some_value"
```

Persistently through System Properties:

Open the Start menu, search for "Environment Variables", and select "Edit the system environment variables".

In the System Properties window, click on the "Environment Variables" button.

In the Environment Variables window, you can add, edit, or remove variables for the user or the system.

Add a new variable by clicking "New" and entering the variable name (MY_VARIABLE) and value (some_value).

#### Getting Environment Variables

Using Command Prompt:

```bash
echo %MY_VARIABLE%
```

Using PowerShell:

```ps
echo $env:MY_VARIABLE
```

## Examples
### Unix/Linux Example
Setting the variable:

```bash
export DB_HOST="localhost"
export DB_PORT="5432"
```

Getting the variable:

```bash
echo $DB_HOST
echo $DB_PORT
```

### Windows Example
Setting the variable in Command Prompt:

```ps
set DB_HOST=localhost
set DB_PORT=5432
```


Getting the variable in Command Prompt:

```ps
echo %DB_HOST%
echo %DB_PORT%
```

Setting the variable in PowerShell:

```ps
$env:DB_HOST="localhost"
$env:DB_PORT="5432"
```

Getting the variable in PowerShell:

```ps
echo $env:DB_HOST
echo $env:DB_PORT
```

## Why Use Environment Variables?

1. They allow you to manage configuration separately from your code, making it easier to handle different environments (development, testing, production).

2. Sensitive information like API keys and database credentials can be stored as environment variables instead of hardcoding them into your source code.

3. Applications can be easily moved between environments without changing the code, simply by adjusting the environment variables.

4. Environment variables can be changed without restarting or redeploying applications, offering more flexibility in managing running applications.

## Using Environment Variables in Go

In Go, you can access environment variables using the `os` package.

:exclamation: We'll do this in a *new* Go project before coming back to our webserver.

> If you have not set temporary environment variables as above, please do so now, or it will return "..is not set" to the console.

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // Get the value of an environment variable
    dbHost := os.Getenv("DB_HOST")
    dbPort := os.Getenv("DB_PORT")

    // If the environment variable is not set, it returns an empty string
    if dbHost == "" {
        fmt.Println("DB_HOST is not set")
    } else {
        fmt.Println("DB_HOST:", dbHost)
    }

    if dbPort == "" {
        fmt.Println("DB_PORT is not set")
    } else {
        fmt.Println("DB_PORT:", dbPort)
    }
}

```

Of course, you can also use this to set environment variables:

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // Set an environment variable
    err := os.Setenv("DB_HOST", "localhost")
    if err != nil {
        fmt.Println("Error setting DB_HOST:", err)
    }

    // Verify that the variable is set
    dbHost := os.Getenv("DB_HOST")
    fmt.Println("DB_HOST:", dbHost)
}

```
---

## .env file

One of the things that is problematic about environment variables is that sometimes you don't know which ones need to be set, and you may only temporarily set them etc.

One way round this is to use a `.env` file that contains all the variables you need locally.

Switch back to your **main Go project** and in the terminal run:

```
go get github.com/joho/godotenv
```

Now we want to create a `.env` file in our root. Open the file and add the following (remembering to change "yourpassword" to your actual postgres password):

```bash
DBTYPE="postgres"
DBHOST="localhost"
DBNAME="acme"
DBUSER="postgres"
DBPASSWORD="yourpassword"
DBSSLMODE="disable"
```

> :exclamation: You don't want to commit your .env file to your repository! You can do this by adding a `.gitignore` rule. Ask your tutor!

## Using our .env file

Our config is currently hard coded - that is, it has our secret password in it and we have to tell it to load the type of database we want. Let's change that.

### Update `config.go`

We're going to add a function to our config that allows us to load the .env file (or another .env file, we'll come back to that later):

```go
package config

import (
	"log"
	"os"

	"github.com/joho/godotenv"
)

type DatabaseConfig struct {
	Type     string // e.g., "postgres", "inmemory"
	Host     string
	User     string
	Password string
	SSLMode  string
	DBName   string
}

func LoadDatabaseConfig(filename ...string) DatabaseConfig {
	
	// Default to ".env" if no filename is provided
	envFile := ".env"
	if len(filename) > 0 {
		envFile = filename[0]
	}

	// Load the specified .env file
	err := godotenv.Load(envFile)
	if err != nil {
		log.Println(".env file not found, using environment variables")
	}

	return DatabaseConfig{
		Type:     os.Getenv("DBTYPE"),
		Host:     os.Getenv("DBHOST"),
		User:     os.Getenv("DBUSER"),
		Password: os.Getenv("DBPASSWORD"),
		SSLMode:  os.Getenv("DBSSLMODE"),
		DBName:   os.Getenv("DBNAME"),
	}
}
```

### Update `main.go`

Of course, now we need to update our `main.go` file to use this function:

```go
// Load configuration
config := config.LoadDatabaseConfig()
```

Okay, assuming everything is up and running, you can now use `go run .` to make sure you connect to the database!

## More .env files!

It’s common practice to have different .env files for different environments (e.g., `.env.development`, `.env.production`). This makes it easier to manage environment-specific configurations.

Let's create an `.env.inmemory" file and see it in action:

```bash
DBTYPE="inmemory"
DBHOST=""
DBNAME=""
DBUSER=""
DBPASSWORD=""
DBSSLMODE=""
```

And update your `main.go` file:


```go
// Load configuration
config := config.LoadDatabaseConfig(".env.inmemory")
```

Run your code to check it's all working!

## Testing

We can also use this in our testing now (this is just an example, you don't have to update your tests).

```go
    //We no longer create the mock directly, instead we load an env file
    // This is nice, because we can just change the .env file and point it at an actual DB
    //or in memory, or whatever

    //dbRepo := inmemory.NewInMemoryRepository();
    config := config.LoadDatabaseConfig(".env.inmemory")
    dbRepo, err := initializeDatabase(config)
    if err != nil {
        t.Fatalf("Error initializing the database: %v", err)
        return
    }
    defer dbRepo.Close()
```

Again, this makes our tests a bit more flexible, and we can have them run in different environments in our pipeline, some with in-memory and some with actual db.

---

## Behaviour of os.Getenv

The `os.Getenv` function in Go retrieves the value of an environment variable that is currently set in the environment at runtime.

When you load variables from a file using `godotenv.Load`, those variables are set in the environment for the current process. However, if an environment variable is already set (e.g., by Azure), the `godotenv.Load` *does not overwrite it*.

This is important to know! Essentially, if you have environment variables set in Azure (or whatever cloud environment) then godotenv will not overwrite them.

### What does this mean?

Well, it's actually good news. We don't want to have our connection details and secrets in our git repo .env file, but we *do* want our cloud environment to have variables that our application will use (where's the database I should be connecting to), and these might be different in each environment.

We wouldn't want our test environment to accidentally point at our production environment because we accidentally committed our.env file and it overwrote the settings. That would make our client very sad. And possibly very angry too.

---

Now of course, if we push our app and deploy it, it won't work - because we are missing our environment variables. Oh dear.

## Azure Environment Variables

In the Azure Portal, you can set environment variables for your app. Since we're using a container, we'll need to set them slightly differently that you would do for a normal web app:

https://learn.microsoft.com/en-us/azure/container-apps/environment-variables?tabs=portal

> :exclamation: Okay, normally you'd want to set up a [key vault and have secrets](https://learn.microsoft.com/en-us/azure/container-apps/manage-secrets?tabs=azure-portal) for your environment variables. However, for now I'd recommend not doing that as it adds extra complexity. But if you fancy having a go, feel free!

We don't have to do this now. When we get to steel threading we will look at everything that needs to be done.