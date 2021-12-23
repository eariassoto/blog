---
title: "Memory allocators in C++ - Part 1"
date: 2020-09-09T23:28:52+02:00
draft: false
tags: [ "C++"]
categories: ["Cpp Development"]
---
When you want to instantiate a class or a struct in the dynamic memory space, we normally use the `new` and `delete` operators.

<!--more-->

```c++
class Point {
	float m_X;
	float m_Y;
	float m_Z;
  public:
    Point(float, float, float);
};

Point* my_point = new Point(0.f, 12.5f, -0.5f);
// use point
//
delete my_point;
```

So what happens under the hood for these operators? The `new` operator asks the operative system for a block of memory of size `sizeof(Point)`. Then, it calls the class constructor and returns the allocated memory pointer. Similarly, `delete` calls the class destructor and then free the memory block. The operative system handles dynamic memory allocation. Therefore these two operators need to do a system call. A system call is expensive because it interrupts the program's execution so that a kernel routine can allocate/free the memory.

When performance is not a concern, this way of allocating/freeing memory is all right. However, if our program will continuously use dynamic-allocated objects, we will need a better strategy.

The `new-expression` we just saw has optional placement parameters. If we provide a memory address as a parameter, the `new` function will use that memory location to construct our object.

```c++
char* memPtr = malloc(sizeof(Point)); // allocate enough memory
Point* my_point_2 = new (memPtr) Point(0.f, 12.5f, -0.5f);
// use point
//
my_point_2->~Point();
free(memPtr);
```

In this scenario, we cannot use `delete`. The `delete` operator can only free memory allocated by the new operator. We have to call the destructor first, and then free the memory. However, we have the same problem because we are allocating space for only one object.

The solution to our problem would be to have one large allocation at the beginning of our program. The allocator would be responsible for distributing that memory. When we do not need to allocate anymore, we make one call to free the memory.

In this series of articles, we will look into some memory allocators designs that help to solve this problem. In these articles, you will find details about their basic operations and the pros/cons they entail. For this first part, stack allocators will be our topic.

![Stack of punch cards](/static/img/stack.jpg "A stack of punch cards")

A simple way to allocate memory is by using a stack. A stack allocator can return a marker that points to the top of it. When a new memory block that fits in the available space is requested, the allocator changes the marker pointer. Freeing memory is performed by passing a marker pointer to the allocator. The allocator will set that pointer as the new marker pointer and release the block in-between.

A more visual explanation is in the figure below:

```C++
const size_t ALLOCATOR_SIZE_IN_BYTES = 64;
StackAllocator allocator{ALLOCATOR_SIZE_IN_BYTES};
```

```
       +------------------------------------------------------------------+
       |                                                                  |
       |                              AVAILABLE                           |
       |                                                                  |
       +------------------------------------------------------------------+
       ^                                                                  ^
       |                                                                  |
       +                                                                  +
 Lowest address                                                    Highest address
 Current Marker                                                        0x0040
     0x0000
```

```c++
struct Point {
	int x;
	int y;
};
Marker base_marker = allocator.GetMarker();

void* address = allocator.Allocate(sizeof(Point));
Point* my_point_1 = new (address) Point();

Point* my_point_2 = new ( allocator.Allocate(sizeof(Point)) ) Point();
// use points ..
```

```
      +-------+-------+--------------------------------------------------+
      |       |       |                                                  |
      |point_1|point_2|                     AVAILABLE                    |
      |       |       |                                                  |
      +-------+----------------------------------------------------------+
      ^               ^                                                  ^
      |               |                                                  |
      +               +                                                  +
Lowest address Current Marker                                     Highest address
  base_marker       0x0010                                             0x0040
    0x0000
```

```C++
allocator.FreeToMarker(base_marker);
```

```
       +------------------------------------------------------------------+
       |                                                                  |
       |                              AVAILABLE                           |
       |                                                                  |
       +------------------------------------------------------------------+
       ^                                                                  ^
       |                                                                  |
       +                                                                  +
 Lowest address                                                    Highest address
 Current Marker                                                        0x0040
     0x0000base_marker
```

We will go over the main functions of my stack allocator. We will leave behind simple functions, like setters and getters. The constructor tries to allocate the requested size and saves the size of it. The initial marker pointer starts as 0. This because it is going to be an offset from the initial address.

```c++
StackAllocator::StackAllocator(size_t stackSizeInBytes)
    : m_StackSizeInBytes{stackSizeInBytes}, m_CurrentMarker{0} {
    m_AllocatedMemory = malloc(m_StackSizeInBytes);
}
```

The allocate function checks for the available memory and calculates the new address and marker pointer if the requested size fits. In my implementation, the memory grows from bottom to top because it assumes an x86 architecture, and the allocated memory comes from the heap.

```C++
void* StackAllocator::Allocate(size_t sizeInBytes) {
    assert(m_AllocatedMemory != 0);
    const size_t availableSize = m_StackSizeInBytes - m_CurrentMarker;
    if (sizeInBytes > 0 && availableSize >= sizeInBytes) {
        Marker newMarker = m_CurrentMarker + sizeInBytes;
        uintptr_t assignedAddress =
            reinterpret_cast<uintptr_t>(m_AllocatedMemory) + newMarker;
        m_CurrentMarker = newMarker;
        return reinterpret_cast<void*>(assignedAddress);
    }
    return nullptr;
}
```

The free function is as simple as it can be. It will only check that the marker to free the memory is a number between 0 and the current marker pointer. An invalid marker pointer will cause memory corruption. The allocator assumes that the user will use the provided function to store all the necessary marker pointers.

```C++
void StackAllocator::FreeToMarker(Marker marker) {
    // Important: markers are not checked. I.e. a random marker
    // can be passed, thus corrupting memory
    if (marker < m_CurrentMarker) {
        m_CurrentMarker = marker;
    
```

The stack implementation is simple and naïve. The allocator does not keep track of the valid markers. The user is in charge of giving a valid pointer to free. Otherwise, the next memory block to be allocation might be already in use.

Finally, the `clear` function is a one-line function.

```C++
void StackAllocator::Clear() { m_CurrentMarker = 0; }
```

# Stack allocator? Make it double

This kind of allocator is useful when you need to allocate blocks of data that do not persist in time. Let's suppose we have a stack allocator to handle the dynamic memory used when processing a game's frame. Functions can then save the last marker pointer, make use of a block of memory, and set the allocator back to the previous state. At the end of the frame, we clear the allocator, and it will be ready for the next frame.

Another example is to allocate linear data. Imagine a platform game where your character can move through 2D maps.  Let's say that the player enters a dungeon map, and then the boss area. We could use a stack allocator to load the maps' resources. When the player leaves the boss' area, the stack allocator frees the entire level's resources at once.

What would happen if you have both necessities? On one side, we have a memory that will be persistent for some amount of time. On the other side, we would like to have an allocator for quick, temporary memory allocations. For these cases, we can upgrade our stack allocator to a double-ended stack allocator.

A visual example of this type of allocator:

```
      +-------+-------+-------+-------------------------+------+----+----+
      |       |       |       |                         |      |    |    |
      |level_1|level_2|level_3|        AVAILABLE        |      |    |    |
      |       |       |       |                         |      |    |    |
      +-------+-------+----------------------------------------+----+----+
      ^                       ^                         ^                ^
      |                       |                         |                |
      +                       +                         +                +
Lowest address           Lower Marker              Upper Marker   Highest address
  base_marker               0x0010                   0x0090            0x0100
    0x0000
```

The implementation of this stack is very similar to the simple stack allocator. Instead of one maker pointer, we will have to. The lower marker pointer will reference the memory allocated from the bottom of the stack. Similarly, the upper marker pointer refers to the memory allocated from the top. As you may imagine, the allocator calculates the available memory from these two pointers.

```C++
void* DoubleStackAllocator::AllocateLower(size_t sizeInBytes) {
    assert(m_AllocatedMemory != 0);
    assert(m_UpperMarker > m_LowerMarker);
    const size_t availableSize = m_UpperMarker - m_LowerMarker;
    if (sizeInBytes > 0 && availableSize >= sizeInBytes) {
        Marker newMarker = m_LowerMarker + sizeInBytes;
        uintptr_t assignedAddress =
            reinterpret_cast<uintptr_t>(m_AllocatedMemory) + newMarker;
        m_LowerMarker = newMarker;
        return reinterpret_cast<void*>(assignedAddress);
    }
    return nullptr;
}
```

The "free" function also checks that the markers will not corrupt the other end of the allocator. However, it won't validate marker pointers within the stack.

```C++
void DoubleStackAllocator::FreeToUpperMarker(Marker marker) {
    // Important: markers are not checked. I.e. a random marker
    // can be passed, thus corrupting memory
    if (marker > m_UpperMarker && marker < m_StackSizeInBytes) {
        m_UpperMarker = marker;
    }
}
```

That should cover the overall implementation of this simple memory allocator. You can find the full source code in [this repository](https://github.com/eariassoto/custom-allocators-cpp). For the next article, we will go into the details of pool allocators.
