---
title: "First Steps With Go"
date: 2017-09-10T18:03:18-06:00
draft: true
---
Go is a simple and powerful programming language. It has a bunch of great features such as static types, memory safety, garbage collection, and it is targets concurrent programming. I discovered very recently, so I want to start simple and learn the basics of this language. In this post, we will write our first Go program and library. Also, we will learn how to unit tests our Go programs.
<!--more-->

# Installing Go

I will show you how to install Go in Linux. For Windows and Mac the process should be very similar. First, you need to visit [this page](https://golang.org/dl/) and copy the link for the Linux version. In my case, I will go for the `go1.9.linux-amd64` version. We need to extract the file somewhere in our computer.



{{<codeblock lang="bash">}} 
# cd /opt
# curl https://storage.googleapis.com/golang/go1.9.linux-amd64.tar.gz | tar zxf -
{{</codeblock>}}



Now, we need to create our Go home directory. In that folder we will keep our source files, install external libraries and save  the executables for our programs. Go ahead and create this folder in your home directory:

{{<codeblock lang="bash">}}
go/
├── bin
└── src
{{</codeblock>}}



Finally, add this variables to your `~/.bashrc` file:

{{<codeblock lang="bash">}}
# Point to the local installation of golang.
export GOROOT=/opt/go

# Point to the location beneath which source and binaries are installed.
export GOPATH=$HOME/go

# Ensure that the binary-release is on your PATH.
export PATH=${PATH}:${GOROOT}/bin

# Ensure that compiled binaries are also on your PATH.
export PATH=${PATH}:${GOPATH}/bin
{{</codeblock>}}

Check that Go is correctly installed:

{{<codeblock lang="bash">}}
$ source .bashrc
$ go version
go version go1.9 linux/amd64
{{</codeblock>}}

# Writing our first Hello World!

We will create our first Go program. For this, we need to create a `hello_world` folder inside `go/src`. Go files end with the `.go` extension. Let's add a `main.go` file with this code:

{{<codeblock "main.go" "go">}}
package main

import (
    "fmt"
)

func main() {
    fmt.Println("hello world!")
}
{{</codeblock>}}

The first statement in any Go source file will be the package name. Packages are a way to separate files in your projects. For executable projects, there should always be a `main` package.


The `import` keyword allow us to import packages in our source file. The `fmt` package is a part of the Go's standard library and we will use it to print to `stdout`. Later in this post we will see how to import our own libraries.



Finally, we need to define the `main` function. This will be the starting point of our program and is always required in executable projects.



# Building and installing our program

To simple compile and run our program we can use the `run` command.

{{<codeblock lang="bash">}}
$ cd go/src/hello_world
$ go run main.go
Hello world!
{{</codeblock>}}

This will not create an executable file. However, the `build` command will do it:

{{<codeblock lang="bash">}}
$ cd go/src/hello_world
$ go build .
$ ./hello_world
Hello world!
{{</codeblock>}}

Additionally, the `install` command will build and copy the executable in the `go/bin` folder. We added this folder to our `$PATH`, so the executable will get installed in our computer.

{{<codeblock lang="bash">}}
$ cd go/src/hello_world
$ go install .
$ ./hello_world
Hello world!
{{</codeblock>}}

So far, this is how your Go home folder should look like:

{{<codeblock lang="bash">}}
go
├── bin
│   └── hello_world
└── src
    └── hello_world
        ├── hello_world
        └── main.go
{{</codeblock>}}


# Writing our own library

A library in Go is a project that does not define a `main` package and will not build an executable file. Let's create a `strcaseconv` folder in `go/src`. We will use this library in our `hello_world` folder. Inside the `strcaseconv` we will a `strcase.go` file with this code:



{{<codeblock "strcase.go" "go">}}
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
{{</codeblock>}}



Compile the library to see if compilation was successful:

{{<codeblock lang="bash">}}
$ cd go/src/strcaseconv

$ go build .
{{</codeblock>}}


Now, go back to `hello_world/main.go` and modify it:

{{<codeblock "main.go" "go">}}
package main

import (
    "fmt"
    "strcaseconv"
)


func main() {
    fmt.Println(strcaseconv.ToMixCase("hello world!"))
}
{{</codeblock>}}



# Unit testing our projects

In Go, the unit test framework comes for free. This framework consists in the `test` command and the `testing` package. All test should be written in files ending with `_test.go`. The unit test function should be named following the `TestXXX` convention. The functions' signature ought to be `func (t *testing.T)`.



The test command will run all functions, those function who calls `t.Error` or `t.Fail` are considered as failed tests. Let's unit test our `strcaseconv` library creating file `strcase_test.go`:

{{<codeblock "strcase_test" "go">}}
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
{{</codeblock>}}


To run them, use the `test` command:

{{<codeblock lang="bash">}}
$ cd go/src/strcaseconv
$ go test
PASS
ok      strcaseconv    0.001s
{{</codeblock>}}



You can find more information about the `testing` package [here](https://godoc.org/testing)



# More useful commands

To install someone else's library or program, Go offers the `get` command. To try it, let's download this hello world program I made a little while ago:

{{<codeblock lang="bash">}}
$ go get github.com/eariassoto/hello_go
$ cd go/src/github.com/eariassoto/hello_go/
$ go run main.go 
What is your name?: Emmanuel
Hello Emmanuel!
{{</codeblock>}}



Go will install the packages in the `go/src` folder. You can import them as any other project in your home folder.



Another nice feature is the `fmt` command. This command will take as parameters Go source files and will format them using Go conventions.



If you are a Vim user, there is a nice [plugin](https://github.com/fatih/vim-go) for Go development.

# Where to go next?

There are a lot of resources and documentation about Go. If you want to learn more about Go visit these websites:



+ [Go Playground](https://play.golang.org/) Online Go editor and compiler
+ [A Tour of Go](https://tour.golang.org/welcome/1) Official Go tutorial
+ [Go by Example](https://gobyexample.com/) Excellent hand-on introduction on Go
+ [Go Documentation](https://golang.org/doc/) Official Go documentation

