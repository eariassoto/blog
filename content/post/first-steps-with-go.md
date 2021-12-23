---
title: "First Steps With Go"
date: 2017-09-10T18:03:18-06:00
draft: false
tags: ["go"]
categories: ["Small Projects"]
---
Go is a simple and powerful programming language. Its syntax is familiar to C/C++ but it definately has improvements in comparison. It has a bunch of great features such as static types, memory safety, garbage collection, and it is targeted to concurrent programming. I discovered it very recently, so I want to start simple and learn the basics of this language. In this post, we will write our first Go program and library. Also, we will learn how to unit tests our Go programs.
<!--more-->

# Installing Go

I will show you how to install Go in Linux. For Windows and Mac, the process should be very similar. First, you need to visit [this page](https://golang.org/dl/) and copy the link for the Linux version. In my case, I need the `go1.9.linux-amd64` version. We need to extract the file somewhere in our computer.

```bash
cd /opt
curl https://storage.googleapis.com/golang/go1.9.linux-amd64.tar.gz | tar zxf -
```



Now, we need to create our Go home directory. In that folder, we will keep our source files, third-party libraries, and install the executables for our programs. Go ahead and create this folder in your home directory:

```bash
go/
├── bin
└── src
```

Finally, we need to set two environment variables: `GOROOT` and `GOPATH`. Go binaries will assume they have been installed in `/usr/local/go` (or `c:\Go` in Windows). In case you installed them somewhere else, set `GOROOT` to your Go binaries folder.

```bash
# Point to the local installation of golang.
export GOROOT=/opt/go
```

The `GOPATH` variable, as discussed [here](https://golang.org/cmd/go/#hdr-GOPATH_environment_variable):

> [...] lists places to look for Go code. On Unix, the value is a colon-separated string. On Windows, the value is a semicolon-separated string.

```bash
# Point to the location beneath which source and binaries are installed.
export GOPATH=$HOME/go
```

Finally, let's add Go binaries and our /bin folders to the $PATH variable. This way, we can use the Go tools and our programs from the console.

```bash
# Ensure that the binary-release is on your PATH.
export PATH=${PATH}:${GOROOT}/bin

# Ensure that compiled binaries are also on your PATH.
export PATH=${PATH}:$HOME/go/bin
```

Save the `~/.bashrc` file, and check that Go is correctly installed:

```bash
$ source .bashrc
$ go version
go version go1.9 linux/amd64
```

# Writing our first "Hello World!" program

Let's create our first Go program. For this, we need to create a `hello_world` folder inside `$HOME/go/src`. Go files end with the `.go` extension. Let's add a `main.go` file with this code:

```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("hello world!")
}
```

The first statement in any Go source file will be the package name. Packages are a way to separate files in your projects. For executable projects, there should always be a `main` package.

The `import` keyword allow us to import packages in our source file. The `fmt` package is part of the Go's standard library. We will use "fmt" to print to `stdout`. Later in this post, we will see how to import our own libraries.

Finally, we need to define the `main` function. This will be the starting point of our program and it is always required in executable projects.

# Building and installing our program

To simple compile and run our program, we can use the `run` command.

```bash
$ cd $HOME/go/src/hello_world
$ go run main.go
Hello world!
```

This will not create an executable file. However, the `build` command will do it:

```bash
$ cd $HOME/go/src/hello_world
$ go build .
$ ./hello_world
Hello world!
```

Additionally, the `install` command will build and copy the executable in the `go/bin` folder. We added this folder to our `$PATH`, so the executable will get installed in our computer.

```bash
$ cd $HOME/go/src/hello_world
$ go install .
$ ./hello_world
Hello world!
```

So far, this is how your Go home folder should look like:

```bash
go
├── bin
│   └── hello_world
└── src
    └── hello_world
        ├── hello_world
        └── main.go
```


# Writing our own library

A library in Go is a project that does not define a `main` package and will not bild an executable file. For our first library, let's create a `strcaseconv` folder in `$HOME/go/src`. We will use this library in our `hello_world` folder. Inside `strcaseconv`, we will add a `strcase.go` file with this code:

```go
package strcaseconv

func ToMixCase(s string) string {
        b := []byte(s)
        for i := 0; i < len(b); i += 1 {
                if b[i] >= 'a' && b[i] <= 'z' {
                        b[i] += 32
                } else if b[i] >= 'A' && b[i] <= 'Z' {
                        b[i] -= 32
                }
        }
        return string(b)
}
```



Compile the library to check for any errors:

```bash
$ cd $HOME/go/src/strcaseconv
$ go build .
```

Now, go back to your `hello_world` projectd and modify `main.go` to use our new library:

```go
package main

import (
    "fmt"
    "strcaseconv"
)


func main() {
    fmt.Println(strcaseconv.ToMixCase("hello world!"))
}
```

# Unit testing our projects

In Go, the unit test framework comes for free. This framework consists in the `test` command and the `testing` package. All test should be written in files ending with `_test.go`. The unit test functions should be named following the `TestXXX` convention. The functions' signature ought to be `func (t *testing.T)`.



The `test` command will run all unit test functions. Those function who calls `t.Error` or `t.Fail` are considered as failed tests. Let's unit test our `strcaseconv` library in `strcase_test.go`:

```go
package strcaseconv

import "testing"

func TestTryUpper(t *testing.T) {
    cases := []struct {
        in, out byte
    }{
        {'a', 'A'},
        {'S', 'S'},
        {'!', '!'},
    }

    for _, c := range cases {
        got := tryUpper(c.in)
        if got != c.out {
            t.Errorf("tryUpper(%q) == %q, want %q", c.in, got, c.out)
        }
    }
}

func TestTryLower(t *testing.T) {
    cases := []struct {
        in, out byte
    }{
        {'a', 'a'},
        {'S', 's'},
        {'#', '#'},
    }

    for _, c := range cases {
        got := tryLower(c.in)
        if got != c.out {
            t.Errorf("tryLower(%q) == %q, want %q", c.in, got, c.out)
        }
    }
}

func TestToMixCase(t *testing.T) {
    cases := []struct {
        in, out string
    }{
        {"hello, world", "HeLlO, wOrLd"},
        {"It is a nice day", "It iS A NiCe dAy"},
        {"", ""},
    }

    for _, c := range cases {
        got := ToMixCase(c.in)
        if got != c.out {
            t.Errorf("ToMixCase(%q) == %q, want %q", c.in, got, c.out)
        }
    }
}
```

To run them, use the `test` command:

```bash
$ cd go/src/strcaseconv
$ go test
PASS
ok      strcaseconv    0.001s
```

You can find more information about the `testing` package [here](https://godoc.org/testing)

# More useful commands

To install someone else's library or program, Go offers the `get` command. To try it, let's download this hello world program I made a little while ago:

```bash
$ go get github.com/eariassoto/hello_go
$ cd go/src/github.com/eariassoto/hello_go/
$ go run main.go 
What is your name?: Emmanuel
Hello Emmanuel!
```

Go will install new libraries in the first folder pointed by `GOPATH`. In our setup, it will be in the `$HOME/go/src` folder.

Another useful feature is the `fmt` command. This command will take as parameters Go source files and it will format them using Go conventions.

If you are a Vim user, there is a nice [plugin](https://github.com/fatih/vim-go) for Go development.

# Where to go next?

There are a lot of resources and documentation about Go. If you want to learn more about Go visit these websites:

+ [Go Playground](https://play.golang.org/) Online Go editor and compiler
+ [A Tour of Go](https://tour.golang.org/welcome/1) Official Go tutorial
+ [Go by Example](https://gobyexample.com/) Excellent hand-on introduction on Go
+ [Go Documentation](https://golang.org/doc/) Official Go documentation

