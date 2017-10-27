---
title: "Building a basic C/C++ project template"
date: 2017-04-17T10:41:16-06:00
comments: true
tags: [ "C++", "cmake", "templates"]
categories: ["Development", "Tutorials"]
---
Starting a C/C++ project can be as easy or as difficult as you want. Personally, I don't like to fire up an IDE just for a small program. So, I end up using a text editor and compiling by terminal. However, the compilation process can get tedious. In this post, we will build a simple project template for a C/C++ program. This project will use the CMake tool to handle all the compilation process.
<!--more-->

<a class=button href="https://github.com/eariassoto/cpp-project-template/tree/basic-template" target="%5fblank">Get the code on GitHub</a>

# Requirements
## C/C++ Code Compiler
The first thing we need is a C/C++ compiler. Most Linux distribution will have [GCC](https://gcc.gnu.org/) installed by default. To check if you have GCC install run:
{{<codeblock lang="bash">}}
$ gcc --version
gcc (Ubuntu 6.3.0-12ubuntu2) 6.3.0 20170406
Copyright (C) 2016 Free Software Foundation, Inc.
This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
{{< /codeblock >}}
If you don't get a similar message you need to install GCC. For Debian based distributions (Ubuntu, Linux Mint) you can install the **build-essential** package:
{{<codeblock lang="bash">}}
$ sudo apt-get update
$ sudo apt-get install build-essential
{{< /codeblock >}}
**build-essential** package includes GCC and other required packages. In case you are using Windows, you can install [MinGW](http://www.mingw.org/) to install and setup GCC.

## CMake
CMake is an open-source cross-platform project that provides a set of tools to build, test and install software. It uses configuration files to control the configuration process. The installation process should be easy, and for Debian bases distributions, you just need to run the command:
{{<codeblock lang="bash">}}
$ sudo apt-get install cmake
{{< /codeblock >}}
For other Linux distributions or other OS check out [CMake official page](https://cmake.org/).

# Project structure

This is how our project folder will look:
{{<codeblock lang="bash">}}
cpp-project-template/
├── CMakeLists.txt
├── include
├── src
│   └── main.cpp
{{< /codeblock >}}
`CMakeLists.txt` is the configuration file for CMake. We will put our header files on `/include` and our source files on `/src`.

# CMake configuration file
Let's see the first configuration file in more details:
{{<codeblock "CMakeLists.txt" "cmake">}}
cmake_minimum_required (VERSION 2.8)

set (PROY cpp-project-template)
project (${PROY} C CXX)
{{< /codeblock >}}
First, we need to set the minimum CMake version, in our case version 2.8. Then, we need to define a name for our project. Change the project name for one of your preference, the executable will be name after this.
{{<codeblock "CMakeLists.txt" "cmake">}}
# Source files folder
set (SRC_DIR src)

# Header files folder
set (INCL_DIR include)
{{< /codeblock >}}
Here we set the folders that contain our code. If you are going to create additional folders, make sure to create proper variables for those folders too. We will need these variables to include our files.
{{<codeblock "CMakeLists.txt" "cmake">}}
# Compilation flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Werror")
{{< /codeblock >}}
We can set our compilation flags in the `CMAKE_CXX_FLAGS` variable. For our project we will be using C++11 version. The `-Wall` flag enables all warning, and the `-Werror` flag treat warnings as errors.

{{<codeblock "CMakeLists.txt" "cmake">}}
include_directories (${PROJECT_SOURCE_DIR}/${INCL_DIR})

# Important: Include all source files on this list
set (SOURCE
${SRC_DIR}/main.cpp
)
{{< /codeblock >}}
`include_directories` command will include our headers to the build. Also, we need to fill in `SOURCE` variable with all of our source files. Otherwise, CMake will not compile our source files.

{{<codeblock "CMakeLists.txt" "cmake">}}
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/bin")

add_executable (${PROY} ${SOURCE})

# Unit tests
add_subdirectory(tests)
{{< /codeblock >}}

`CMAKE_RUNTIME_OUTPUT_DIRECTORY` specifies where CMake will save our output files. In our case, executables will be on `/bin` folder. Finally, the `add_executable` command will create a executable file with our source files.

# Using the template

Let's define a `Vector` class. For simplicity, we are going to represent 2D vectors. Also, our class will implement some basic operations between vectors.

{{<codeblock "include/Vector.h" "c++">}}
#ifndef __VECTOR_H__
#define __VECTOR_H__

// System includes
#include <iostream>

class Vector {
public:
  Vector(double a1, double a2) : a1(a1), a2(a2){};

  Vector sum(const Vector &vec);
  Vector substract(const Vector &vec);
  Vector scale(const double k);
  double dot(const Vector &vec);

  inline int getA1() const { return a1; };

  inline int getA2() const { return a2; };

  friend std::ostream &operator<<(std::ostream &os, const Vector &vec) {
    os << '(' << vec.a1 << ", " << vec.a2 << ')';
    return os;
  }

private:
  double a1;
  double a2;
};

#endif /* __VECTOR_H__ */
{{< /codeblock >}}

{{<codeblock "src/Vector.cpp" "c++">}}
#include "Vector.h"

Vector Vector::sum(const Vector &vec) {
  return Vector{a1 + vec.getA1(), a2 + vec.getA2()};
}

Vector Vector::substract(const Vector &vec) {
  return Vector{a1 - vec.getA1(), a2 - vec.getA2()};
}
Vector Vector::scale(const double k) { return Vector{k * a1, k * a2}; }
double Vector::dot(const Vector &vec) {
  return a1 * vec.getA1() + a2 * vec.getA2();
}
{{< /codeblock >}}

Let's create a couple of vectors and output the results on our main function:

{{<codeblock "src/main.cpp" "c++">}}
// System includes
#include <iostream>

// Program includes
#include "Vector.h"

int main(int argc, char *argv[]) {
  Vector vector1(3, 2);
  Vector vector2(10, 5);
  double k = (double)3 / 10;
  std::cout << "Vector 1: " << vector1 << '\n';
  std::cout << "Vector 2: " << vector2 << '\n';
  std::cout << "Vector 1 + Vector 2: " << vector1.sum(vector2) << '\n';
  std::cout << "Vector 1 - Vector 2: " << vector1.substract(vector2) << '\n';
  std::cout << k << " * Vector 2: " << vector1.scale(k) << '\n';
  std::cout << "Vector 1 dot product Vector 2: " << vector1.dot(vector2)
            << '\n';
  return 0;
}
{{< /codeblock >}}

Finally, we have to change the project name and include our new source files. Here are the `CMakeLists.txt` file with the necessary changes:
{{<codeblock "CMakeLists.txt" "cmake">}}
cmake_minimum_required (VERSION 2.8)

set (PROY math-vector-example)
project (${PROY} C CXX)

# Source files folder
set (SRC_DIR src)

# Header files folder
set (INCL_DIR include)

# Compiler flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Werror")

include_directories (${PROJECT_SOURCE_DIR}/${INCL_DIR})

set (SOURCE
${SRC_DIR}/main.cpp
${SRC_DIR}/Vector.cpp
)

# Output folder
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/bin")

add_executable (${PROY} ${SOURCE})
{{< /codeblock >}}

To compile our program, we will need to run these commands:
{{<codeblock lang="bash">}}
$ mkdir build
$ cd build
$ cmake ..
$ make
{{< /codeblock >}}
You only need to create the `build/` folder once. The `cmake ..` command will generate the makefiles and `make` will build the program. If everything went all right, the executable will be saved in the `bin/` folder. Let's run our example:

{{<codeblock lang="bash">}}
$ ./../bin/math-vector-example
Vector 1: (3, 2)
Vector 2: (10, 5)
Vector 1 + Vector 2: (13, 7)
Vector 1 - Vector 2: (-7, -3)
0.3 * Vector 2: (0.9, 0.6)
Vector 1 dot product Vector 2: 40
{{< /codeblock >}}

That's pretty much it. Now go and write your own programs without much hassle.

<a class=button href="https://github.com/eariassoto/cpp-project-template/tree/basic-template" target="%5fblank">Get the code on GitHub</a>
