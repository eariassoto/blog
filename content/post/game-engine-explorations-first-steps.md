---
title: "Game Engine Explorations: First steps"
date: 2021-12-02T01:25:52+01:00
draft: false
tags: [ "C++"]
categories: ["Game Engine Explorations"]
---

This post is the first one of a series of posts where I will document my learnings on Game Engine development.
<!--more-->

From all the roles within sofware engineering, I am most attracted to Software Architecture. To prepare myself for this path, I like to study software systems, understand how they work and implement my own version. I find satisfaction in designing the small pieces, testing them and putting them all together. It is a long process that requires patience and dedication, but hey, here goes my attempt at creating my own engine.

Game engines fall under this category of complex software systems. They help developers create videogames faster by putting the heaviest workload off their shoulders.

These engines come in all sizes. Commercial engines such as Unity, Godot or Unreal Engine are developed by hundreds of people and they offer the users rich UX interfaces. Videogames companies develop their own large scale in-house engines for their own games.

Given that I will take on this project by myself, I will not attempt to implement a commercial engine, nor a large scale engine. My first goal will be to create a simple 3D rendering engine, choose a particular game genre (FPS, RTS, platformer, for example), to finally create a game with the engine.

## Setting initial goals

As I mentioned before, I will start designing enough engine code so that I can render a 3D mesh with textures and some basic camera and lighting. For better performance and portability, I will be using C++ as main programming language. I will try to target Windows and Linux as platforms.

The next diagram shows a rough design of the project:

```
  +--------------+
  |   external   |
  | dependencies |
  +-------+------+
          |
+---------v--------+    +---------------------+   +------------+
|    Engine Code   +---->      Game Code      |   |    Game    |
| (static library) |    |(application project)+--->(executable)|
+------------------+    +----------^----------+   +------------+
                                   |
                           +-------+------+
                           |   external   |
                           | dependencies |
                           +--------------+
  +------------------+
  |    Asset files   |      +--------------+     +-------------+
  |(meshes, textures,|      | Data parsing |     |  Game Data  |
  | shaders, sounds) +------>   scripts    +----->(binary file)|
  +------------------+      +--------------+     +-------------+

```

The engine code compiles to a static library that the different application projects can share. The reason of using a static library is performance. It is normal to have a single videogame running in a machine so we would gain little to nothing from having the code in a dynamic library. Moreover, calls to functions inside a dynamic linked library are more expensive.

The game logic is the pieces of code that relate to the game itself. This code will compile as an executable that links to the engine code library. As a first approach, this application will be in charge of registering the assets and rendering meshes. In the future, I expect to have a scene manager and a resources manager to deal with assets in a better way.

In the previous diagram I added the game data as a binary file. This will be part of the second iteration of the project. Once I have a proper resources manager, I will be able to add additional code to load assets from an optimized file with my own specifications. As a starting point, loading individual files is okay.

## Finding resources

This is a list of books and resources I am using to start the project:

+ **Game Engine Architecture - Jason Gregory**: This one is my main guide. I have read first parts so far but I have enough learnings to go and jump to practice. This book covers the topics at a broader scale. The next books I selected will go deeper into a specific topic.
+ **3D Math Primer for Graphics and Game Development - Fletcher Dunn**: I got this one to refresh my math skills for 3D rendering.
+ **Introduction to 3D Game Programming with DirectX 11 - Frank Luna**: I am new to graphics programming so I will use DirectX11 as one of the APIs to start this engine. This API is still used, and it is easier to grasp for a rookie programmer than DirectX12 and Vulkan.
+ **[learnopengl.com](https://learnopengl.com/)**: The other graphical I will use API for the engine is OpenGL. This API abstracts configuration details, but modern version provide more control to the programmer. It will also be useful to provide support to GNU/Linux platforms.
+ **[The Cherno - Youtube channel](https://www.youtube.com/channel/UCQ-W1KE9EYfdxhL6S4twUNw)**: This channel has an awesome series on game engine development. The first videos on their series have helped me greatly to jumpstart my project.

## So what is left?

Now comes the fun part, code! In the [next post](/post/game-engine-explorations-hello-world/), I will be setting up the build system and the project configuration. I promise to end up with a classic Hello World! :)




