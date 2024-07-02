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