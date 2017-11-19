---
title: "Test your C++ code with the Google Test framework"
date: 2017-10-31T01:53:18Z
draft: false
tags: [ "C++", "cmake", "templates", "unit test"]
categories: ["Development", "Tutorials"]
---
[In a previous post](https://eariassoto.github.io/2017/04/building-a-basic-c/c---project-template/), I showed you a C/C++ template that you can use for a project. I felt that it needed a basic testing framework. Therefore, we are going to learn how to install and use the Google Test framework to write tests. When we have finished this tutorial, we will have an executable that will run tests for our code.
<!--more-->

<a class=button href="https://github.com/eariassoto/cpp-project-template/tree/template-with-ut" target="%5fblank">Get the code on GitHub</a>

I will assume that you are familiarized with unit testing methods. In case you do not know about the Google Test framework, [here](https://github.com/google/googletest/blob/master/googletest/docs/Primer.md) is a basic introduction to this library. This post will be more of a tutorial rather than a technical post. I plan to write about test driven development (TDD) in the future.

## Requirements

We need to have installed a C/C++ compiler and CMake on our machine. Next, we need to download the C/C++ template project. [Here](https://github.com/eariassoto/cpp-project-template/tree/basic-template) is the link to the repo where you can get it. As I mentioned before, [in this post](https://eariassoto.github.io/2017/04/building-a-basic-c/c---project-template/) I got further into details about this template. Feel free to read it, and then come back here to continue. If you have your own CMake project, you can follow the steps and install Google Test on your own project.

## Folder structure

Our project should look like this now:
{{<codeblock lang="bash">}}
cpp-project-template
├── CMakeLists.txt
├── include
│   ├── Greeter.h
│   ├── IPerson.h
│   └── Person.h
├── src
│   ├── greeter.cpp
│   └── main.cpp
└── tests
    ├── CMakeLists.txt
    ├── CMakeLists.txt.in
    ├── include
    │   └── mocks
    │       └── MockPerson.h
    └── src
        ├── greeterTests.cpp
        └── main.cpp
{{</codeblock>}}

The `tests/` folder will contain our unit testing project. It is also a CMake project, so it has its own `CMakeLists.txt` file. The `tests/CMakeLists.txt.in` file will help us include the Google Test library.

## Setting up CMake

To include our unit testing project, we only need to add this line at the end of our `CMakeLists.txt` file:
{{<codeblock "CMakeLists.txt" "cmake">}}
# Unit tests
add_subdirectory(tests)
{{</codeblock>}}

Now, let's go over the `tests/CMakeLists.txt` file:

{{<codeblock "tests/CMakeLists.txt" "cmake">}}
cmake_minimum_required(VERSION 2.8)
set (PROJECT_TEST_NAME ${PROJECT_NAME}-ut)

# Download and unpack googletest at configure time
configure_file(CMakeLists.txt.in googletest-download/CMakeLists.txt)
execute_process(COMMAND "${CMAKE_COMMAND}" -G "${CMAKE_GENERATOR}" .
      WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/tests/googletest-download" )
execute_process(COMMAND "${CMAKE_COMMAND}" --build .
      WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/tests/googletest-download" )
# Add googletest directly to our build. This adds
# the following targets: gtest, gtest_main, gmock
# and gmock_main
add_subdirectory("${CMAKE_BINARY_DIR}/googletest-src"
                 "${CMAKE_BINARY_DIR}/googletest-build")

# Test folder
set (TESTS_DIR tests)
# Header files folder
set (INCL_DIR include)
set (MOCKS_DIR mocks)
include_directories (${PROJECT_SOURCE_DIR}/${TESTS_DIR}/${INCL_DIR})
include_directories (${PROJECT_SOURCE_DIR}/${TESTS_DIR}/${INCL_DIR}/${MOCKS_DIR})

set (SRC_DIR src)
set(PROY_UT_SRC
    ${SRC_DIR}/main.cpp
    ${SRC_DIR}/greeterTests.cpp
)

set (MAIN_SRC_DIR ../src)
include_directories (${PROJECT_SOURCE_DIR}/${INCL_DIR})
# Include all project source files here
set (MAIN_SOURCE 
    ${MAIN_SRC_DIR}/greeter.cpp
)

add_executable (${PROJECT_TEST_NAME} ${PROY_UT_SRC} ${MAIN_SOURCE})

set_target_properties(${PROJECT_TEST_NAME}
                      PROPERTIES
                      COMPILE_FLAGS "-std=c++11 -Wall -Werror"
)
# Link libraries
target_link_libraries(${PROJECT_TEST_NAME}
                      gtest
                      gmock_main
)
{{</codeblock>}}

There are several ways I could have included the Google Test library in the project. The `find_package` command would have done it just fine, but the library needs to be installed in the computer. Another option would have been to manually copy the library in my project and compile it there. Luckily, I found a cleaner and more portable way to install the library. We will tell CMake to automatically clone and compile the library when it builds the project.

The line 5 of `tests/CMakeLists.txt` is copying `tests/CMakeLists.txt.in` to a subdirectory under `build/`. This file has the information to download Google Tests as an external project:

{{<codeblock "tests/CMakeLists.txt.in" "cmake">}}
cmake_minimum_required(VERSION 2.8.2)
project(googletest-download NONE)
 
include(ExternalProject)
ExternalProject_Add(googletest
  GIT_REPOSITORY    https://github.com/google/googletest.git
  GIT_TAG           master
  SOURCE_DIR        "${CMAKE_BINARY_DIR}/googletest-src"
  BINARY_DIR        "${CMAKE_BINARY_DIR}/googletest-build"
  CONFIGURE_COMMAND ""
  BUILD_COMMAND     ""
  INSTALL_COMMAND   ""
  TEST_COMMAND      ""
)
{{</codeblock>}}

Lines 6-9 of `tests/CMakeLists.txt` execute the commands that download and build the library. Lines 13-14 add the library as a submodule. Finally, the linking of the library is made in lines 44-47. Notice that this library will only be available to our unit testing project.

The rest of the `tests/CMakeLists.txt` file should be familiar to you. Similarly to our main project, your can keep your headers in `tests/include` and your source code in `tests/src`. It is important that you also include the headers and source files of your main project. Otherwise, your tests will fail to compile.

We need a main function for out unit testing project's executable file. This function will search for tests and run them all:
{{<codeblock "tests/main.cpp" "cpp">}}
#include <gtest/gtest.h>

int main(int argc, char **argv) {
  testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
{{</codeblock>}}

And that is pretty much it. We do not need additional commands to compile our unit testing project. After we build the main project, the executable for the unit tests will be under the `bin/` folder. The executable is named like the main project, but it will have the `-ut` suffix.

## A simple example

To give you a small example of unit testing with Google Test, let's make a simple program to greet people. We want to encapsulate the person's data, and then pass it to our greeter.

As you can imagine, we will implement classes `Person` and `Greeter`. Additionally, we will abstract `Person`'s implementation in the interface `IPerson`. With this, `Greeter` can take any `IPerson` implementation. Using interfaces to handle dependencies between classes makes our classes independent of each other. Furthermore, it makes our code more easy to maintain and test.

Without more, let's see the code for out program:

{{<codeblock "include/IPerson.h" "c++">}}
#ifndef __IPERSON__H__
#define __IPERSON__H__
#include <string>

class IPerson {
public:
  virtual ~IPerson(){};
  virtual std::string getName() const = 0;
};
#endif /** __IPERSON__H__ */
{{</codeblock>}}

{{<codeblock "include/Person.h" "c++">}}
#ifndef __PERSON__H__
#define __PERSON__H__
#include <IPerson.h>

class Person : public IPerson {
public:
  Person(std::string name) : name(name){};
  ~Person(){};

  std::string getName() const { return name; };

protected:
  std::string name;
};
#endif /** __PERSON__H__ */
{{</codeblock>}}

{{<codeblock "include/Greeter.h" "c++">}}
#ifndef __GREETER__
#define __GREETER__
#include "IPerson.h"
#include <string>

class Greeter {
public:
  std::string greet();
  std::string greetTo(IPerson &person);
};
#endif /* __GREETER__ */
{{</codeblock>}}

{{<codeblock "src/Greeter.cpp" "c++">}}
#include "Greeter.h"

std::string Greeter::greet() { return "Hello world!"; }

std::string Greeter::greetTo(IPerson &person) {
  return "Hi " + person.getName() + "!";
}
{{</codeblock>}}

{{<codeblock "src/main.cpp" "c++">}}
#include <iostream>
#include "Greeter.h"
#include "Person.h"

int main(int argc, char *argv[]) {
  Person p{"Emmanuel"};
  Greeter g;
  std::cout << g.greet() << '\n';
  std::cout << g.greetTo(p) << '\n';
  return 0;
}
{{</codeblock>}}

Let's compile and run both our program and our unit testing executable:
{{<codeblock lang="bash">}}
~/cpp-project-template $ mkdir build
~/cpp-project-template $ cd build
~/cpp-project-template/build $ cmake ..
~/cpp-project-template/build $ make
~/cpp-project-template/build $ ./../bin/cpp-project-template
Hello world!
Hi Emmanuel!
~/cpp-project-template/build $ ./../bin/cpp-project-template-ut
[==========] Running 0 tests from 0 test cases.
[==========] 0 tests from 0 test cases ran. (0 ms total)
[  PASSED  ] 0 tests.
{{</codeblock>}}

Everything seems fine, our program is showing the correct output, and the unit testing executable ran zero tests. We will start writing the tests for the `Person` class. We only need one test for the `std::string getName()` function. 

{{<codeblock "tests/src/personTests.cpp" "c++">}}
#include "Person.h"
#include <gtest/gtest.h>

TEST(GreeterTests, PersonGetNameTest) {
  Person p{"Woz"};
  ASSERT_STREQ(p.getName().c_str(), "Woz");
}
{{</codeblock>}}

For the `Greeter` class, we need to deal with the `IPerson`. interface. Passing a `Person` instance might bring us problems. If `Person` has bugs, our tests will probably be affected. To isolate the testing of `Greeter` from whatever `IPerson` implementation we have, we need to implement mock objects. A mock is a simulated object that mimics the behavior of a real object.

To create mocks in our unit testing project, Google Test provides the [Google Mock](https://github.com/google/googletest/blob/master/googlemock/docs/ForDummies.md) library. We will define a `MockIPerson` class that will simulate the behavior for interface `IPerson`. We will make this mock behave correctly in our `Greeter` unit tests.

{{<codeblock "tests/include/mockIPerson.h" "c++">}}
#ifndef __MOCKIPERSON__
#define __MOCKIPERSON__
#include "IPerson.h"
#include "gmock/gmock.h"
#include <string>

class MockIPerson : public IPerson {
public:
  MockIPerson(){}
  ~MockIPerson(){}
  MOCK_CONST_METHOD0(getName, std::string());
};
#endif /** __MOCKIPERSON__ */
{{</codeblock>}}

{{<codeblock "tests/src/greeterTests.cpp" "c++">}}
#include "Greeter.h"
#include "MockPerson.h"
#include <gtest/gtest.h>
using ::testing::Return;

TEST(GreeterTests, GreetTest) {
  Greeter g;
  ASSERT_STREQ(g.greet().c_str(), "Hello world!");
}
 
TEST(GreeterTests, GreetToTest) {
  MockPerson mockPerson;
  EXPECT_CALL(mockPerson, getName()).WillOnce(Return("Greg"));
  Greeter g;
  ASSERT_STREQ(g.greetTo(mockPerson).c_str(), "Hi Greg!");
}
{{</codeblock>}}

Notice that, in line 13, we catch the `getName()` call from our mock. `greetTo` will call this function, and the mock will return the string we provided. Finally, let's recompile and run our tests.

{{<codeblock lang="bash">}}
~/cpp-project-template/build $ cmake ..
~/cpp-project-template/build $ make
~/cpp-project-template/build $ ./../bin/cpp-project-template-ut
[[==========] Running 3 tests from 1 test case.
[----------] Global test environment set-up.
[----------] 3 tests from GreeterTests
[ RUN      ] GreeterTests.GreetTest
[       OK ] GreeterTests.GreetTest (0 ms)
[ RUN      ] GreeterTests.GreetToTest
[       OK ] GreeterTests.GreetToTest (0 ms)
[ RUN      ] GreeterTests.PersonGetNameTest
[       OK ] GreeterTests.PersonGetNameTest (0 ms)
[----------] 3 tests from GreeterTests (0 ms total)

[----------] Global test environment tear-down
[==========] 3 tests from 1 test case ran. (1 ms total)
[  PASSED  ] 3 tests.
{{</codeblock>}}

## Where to go next

The documentation for Google Test is quite extensive. Here is a list of resources you can check to learn how to use this framework:

+ [Google Test documentation](https://github.com/google/googletest/blob/master/googletest/docs/Documentation.md)
+ [Google Mock documentation](https://github.com/google/googletest/tree/master/googlemock/docs)

<a class=button href="https://github.com/eariassoto/cpp-project-template/tree/template-with-ut" target="%5fblank">Get the code on GitHub</a>
