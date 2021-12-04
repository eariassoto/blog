---
title: "Game Engine Explorations: Hello World"
date: 2021-12-04T03:41:43+01:00
draft: false
---

My previous post was an introduction to my first explotations in game engine development. In this post, I am laying down the foundation of the project. I will be using a tool that automatically creates Visual Studio solutions and Makefiles.
<!--more-->

This first iteration consists of an executable program that calls a function from a static library. This is how the folder structure looks like without any configuration files:

```
└───sources
    ├───engine_lib
    │   │
    │   └───engine_lib
    │           hello.cpp
    │           hello.h
    │
    └───game_app
            main.cpp
```

The main function is quite simple:

```C++
#include "engine_lib/hello.h"

int main(int /*argc*/, char* /*argv*/[])
{
    say_hello();

    return 0;
}
```

The static library is also as simple as it gets:

```C++
// .h
#pragma once

void say_hello();

// .cpp
#include "hello.h"

#include <iostream>

void say_hello() { std::cout << "Hello World!\n"; }

```

I also added a `.clang-format` file in the `sources` foldder with some personal preferences:

```
---
BasedOnStyle: Google
IndentWidth: 4
BreakBeforeBraces: Linux
SortIncludes: false
PointerAlignment: Left
```

Visual Studio will recognize the format file and apply it to source files in this project. I can also use it to configure it source files formatting in editors like Vim.

## Choosing the build configurator tool

I used to go with CMake to setup my C++ projects, but I found two alternatives that I better target large projects. At my current job at Ubisoft, we use a tool called Sharpmake. It takes a C# program as input that defines your solutions and projects. Sharpmake support multiple platforms, for our interests, it can generate Visual Studio solutions and Makefiles. This tool has an [open source version](https://github.com/ubisoft/Sharpmake) available to use. A similar and more popular tool is [Premake](https://premake.github.io/). Premake comes as a lightweight executable that reads Lua files that can generate VS solutions and Makefiles as well. I have decided to use Premake for this project.

## Setting up Premake

To make things easier, I will create a `premake` folder that will hold the binaries and the main Lua project file:

```
├───premake
│   │   premake5.lua
│   │
│   └───bin
│       └───win64
│               LICENSE.txt
│               premake5.exe
│
└───sources
```

The default filename for Premake script files is`premake5.lua`. In this file, I will configure my main workspace. Premake generator translates this workspace to VS solutions and Makefiles. Let´s look this file by pieces:

{{< highlight lua "linenos=inline,linenostart=1" >}}
-- Copyright (c) 2021 Emmanuel Arias
local ROOT = "../"

{{< / highlight >}}

Paths in Premake are relative to where the current `.lua` file is located. For consistent paths, I will define a local variable `ROOT` in every script. This variable will help me to share global path variables.

{{< highlight lua "linenos=inline,linenostart=4" >}}
-- _ACTION is set to nil when premake is run but no generation is needed
-- for example "premake5 -help"
local gen_action = "NULL"
if _ACTION ~= nill then gen_action = _ACTION end
local GEN_FOLDER = ("generated/" .. gen_action .. "/")
{{< / highlight >}}

Here I am setting up the folder where Premake will use to create the project files. Everything in this folder is generated and should be omitted by the version control system. `_ACTION` is a global variable set by Premake and it stores the action set by the user. You can find more info [here](https://premake.github.io/docs/_ACTION/). For example, if I execute these two commands:

```
> premake5 vs2019
> premake5 gmake
```

I will get two folders: `generated/vs2019` and `generated/gmake` with two different and independent configurations. This also means that I will get separate folders with the binary files by the VS solutions and the Makefiles. 

{{< highlight lua "linenos=inline,linenostart=10" >}}
-- Global variables
PROJECT_ROOT = "/sources/%{prj.name}/"

local OUTPUT_DIR = "%{cfg.buildcfg}-%{cfg.architecture}/"
TARGET_FOLDER = (GEN_FOLDER .. "target/" .. "/%{prj.name}/" .. OUTPUT_DIR)
INTERMEDIATE_FOLDER = (GEN_FOLDER .. "intermediate/" .. "/%{prj.name}/" .. OUTPUT_DIR)
{{< / highlight >}}

Here I define global variables for the projects' lua scripts. Premake projects need to know where source files are and where to save internediate and output files. The `intermediate/` folder is not relevant for me, but `target/` will have all the executables and library files. The variables you see in between  `%{ }` are Premake tokens. These tokens are replaced in runtime and you can find more info [here](https://premake.github.io/docs/Tokens).

{{< highlight lua "linenos=inline,linenostart=17" >}}
workspace "Tamarindo Engine"
   startproject "game_app"
   filename "tamarindo_engine"
   location (ROOT .. GEN_FOLDER)

   configurations { "Debug", "Release" }
   platforms { "x32", "x64" }

   filter "configurations:Debug"
      defines { "DEBUG" }
      symbols  "On"
   
   filter "configurations:Release"
      defines { "NDEBUG" }
      optimize "On"

   filter "platforms:x64"
      architecture "x86_64"

   filter "platforms:x32"
      architecture "x86"

   flags {
      "FatalWarnings",
      "MultiProcessorCompile"
   }
{{< / highlight >}}

Now I start declaring the main workspace. It has simple configurations: it supports 32 and 64 bits architetures, and defines two optimization levels. For VS solutions, I can define a start-up project. This project will be the executable application.

{{< highlight lua "linenos=inline,linenostart=44" >}}
include (ROOT .. "sources/engine_lib")
include (ROOT .. "sources/game_app")
{{< / highlight >}}

Finally, I need to include the two projects: the engine library and the application. Premake expects `premake5.lua` files in both of those folders.

The script for the engine library is quite simple:

{{< highlight lua "linenos=inline,hl_lines=5 " >}}
-- Copyright (c) 2021 Emmanuel Arias
local ROOT = "../../"

project "engine_lib"
   kind "StaticLib"
   language "C++"
   cppdialect "C++17"
   staticruntime "on"

   targetdir (ROOT .. TARGET_FOLDER)
   objdir (ROOT .. INTERMEDIATE_FOLDER)

   files {
       (ROOT .. PROJECT_ROOT .. "engine_lib/**.h" ),
       (ROOT .. PROJECT_ROOT .. "engine_lib/**.cpp" )
    }

    includedirs {
      (ROOT .. PROJECT_ROOT .. "engine_lib")
    }
{{< / highlight >}}

This script is very self-explanatory. From this snippet, I like that the `kind` command makes configuration easier. The application project script has few differences:

{{< highlight lua "linenos=inline,hl_lines=5 19 22-24" >}}
-- Copyright (c) 2021 Emmanuel Arias
local ROOT = "../../"

project "game_app"
   kind "ConsoleApp"
   language "C++"
   cppdialect "C++17"
   staticruntime "on"

   targetdir (ROOT .. TARGET_FOLDER)
   objdir (ROOT .. INTERMEDIATE_FOLDER)

   files {
       (ROOT .. PROJECT_ROOT .. "**.h" ),
       (ROOT .. PROJECT_ROOT .. "**.cpp" )
    }

    includedirs {
        "../engine_lib"
    }

    links {
        "engine_lib"
    }
{{< / highlight >}}


Last thing I need to make things easier is to create a batch script to run Premake.

```bat
:: Copyright (c) 2021 Emmanuel Arias
@echo off

call premake\bin\win64\premake5.exe --file=premake/premake5.lua vs2019
```

After running the script, my project folder will look like this:

![Base project structure](/static/img/engine_explorations/base_struct.PNG)

I can run the executable from Visual Studio, or I can go and execute it in the `target` folder. You can find this code in the project's [Github repo](https://github.com/eariassoto/tamarindo_engine/tree/a4ca92dbdb4bca9a467d26275929db3588e9f1e1).

That is all for now :). In the next post, I will lay down the basic class hierarchy and main function for the engine.
