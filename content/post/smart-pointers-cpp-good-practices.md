---
title: "Make your pointers smart - C++ good practices"
date: 2018-02-28T21:05:41-06:00
comments: true
tags: [ "C++", "pointers"]
categories: ["Development", "Tutorials"]
---

Ever since the C++ language was first standardized, `new` and `delete` were defined as the methods to create/delete objects dynamically. The `new` operator allocates a memory block to construct an object and then calls the proper class' constructor to initialize it. If successful, this operator will return a pointer to the location of the memory block. Otherwise, it will return `nullptr` or it will throw an exception. The `delete` operator executes the inverse operation, it deallocates object's memory block. Before freeing the memory, `delete` calls the object's destructor. The destructor is used to free memory that the object must delete.

Let's create a small example that created objects dynamically with `new` and `delete`:

```c++
#include <iostream>
#include <string>
#include <sstream>

class Contact {
 public:
  Contact(const std::string& name, const std::string& email)
      : name{name}, email{email} {
    std::cout << "Constructed: [" << this << "]\n";
  }
  ~Contact() { std::cout << "Destructing: [" << this << "]\n"; };

  const std::string print() {
    std::stringstream printStream;
    printStream << "{Name:" << name << ", Email:" << email << "}";
    return printStream.str();
  }

 protected:
  std::string name;
  std::string email;
};

int main() {
  Contact* contact = new Contact("John Doe", "jdoe@mail.com");
  std::cout << "Contact: " << contact->print() << '\n';
  delete contact;
  return 0;
}
```

If we run this program we will get:

```bash
Constructed: [0x162bc80]
Contact: {Name:John Doe, Email:jdoe@mail.com}
Destructing: [0x162bc80]
```

Using regular pointers, like this example can give you problems. If you miss a `delete` call on an object, the object's memory blocks will not be reassigned to another process. This problem is called memory leak. Leaking memory can exhaust all of the system's memory!

Another problem related to `new` and `delete` is pointer ownership. A regular pointer does not have the context to indicate who is its owner. By ownership, I mean whose object is in charge of deleting the pointer's reference. If no one deletes the memory, we will get our memory leaked. If `delete` gets called twice for the same object, we will get a segmentation fault. Further, if someone deletes an object, and another object tries to reference it, we will also hit a segmentation fault. With regular pointers, you must handle memory deallocation carefully. The larger your program grows, the harder will be to not make mistakes.

# Introducing smart pointers

Both ownership and memory deallocation are addressed by the new pointer types, introduced by the C++11 specification. Let's will talk about `unique_ptr` first. The `unique_ptr` type wraps dynamic memory and disposes of it when the pointer goes out of scope. A `unique_ptr` can only have one owner. Ownership can be passed between objects using the `std::move()` function. This type implements both the `*` and `->` operators to dereference memory, just like regular pointers. You can also get a raw pointer with the `get()` function.

Let's change our main function so it uses `unique_ptr` instead:

```c++
#include <memory>

int main() {
  std::cout << "Foo\n";
  std::unique_ptr<Contact> contact =
      std::unique_ptr<Contact>(new Contact("John Doe", "jdoe@mail.com"));
  std::cout << "Contact: " << contact->print() << '\n';
  std::cout << "Bar\n";
  return 0;
}
```


```bash
Foo
Constructed: [0x1e68c80]
Contact: {Name:John Doe, Email:jdoe@mail.com}
Bar
Destructing: [0x1e68c80]
```

I added two logs so you can verify that `contact` deletes the object after "Bar" is logged. This happens because `contact` is owned by `main`, and `main` goes out of scope after this log. Let's surround `contact` with an anonymous scope:

```c++
int main() {
  std::cout << "Foo\n";
  {
    std::unique_ptr<Contact> contact =
        std::unique_ptr<Contact>(new Contact("John Doe", "jdoe@mail.com"));
    std::cout << "Contact: " << contact->print() << '\n';
  }
  std::cout << "Bar\n";
  return 0;
}
```


```bash
Foo
Constructed: [0x1e68c80]
Contact: {Name:John Doe, Email:jdoe@mail.com}
Destructing: [0x1e68c80]
Bar
```

Now the object is deleted when the anonymous scope goes out of scope, right before the "Bar" log. Notice that in both examples, there is no `delete` call. We can even get rid of the `new` operator with C++14's `make_unique` function:

```c++
  std::unique_ptr<Contact> contact =
      std::make_unique<Contact>("John Doe", "jdoe@mail.com");
}
```

There are cases where we need to share the ownership of an object. For this, C++11 introduced the `shared_ptr` type. Several `shared_ptr`s can refer to the same object, thus sharing ownership. Internally, these pointers keep a use counter. This counter indicates how many `shared_ptr` are sharing ownership of an object. The last `shared_ptr` owning an object deletes the object once it goes out of scope. This pointer type allows you to distribute ownership with the safety that memory will be properly freed. Let's add to our example a `compare` function that receives a copy `shared_ptr` as a parameter. I added logs so we can check the pointers' use counter:

```c++
class Contact {
 public:
  Contact(const std::string& name, const std::string& email)
      : name{name}, email{email} {
    std::cout << "Constructed: [" << this << "]\n";
  }
  ~Contact() { std::cout << "Destructing: [" << this << "]\n"; };

  const std::string print() {
    std::stringstream printStream;
    printStream << "{Name:" << name << ", Email:" << email << "}";
    return printStream.str();
  }

  bool compare(std::shared_ptr<Contact> otherContact) {
    if (otherContact == nullptr) {
      std::cout << "Invalid parameter(s)\n";
      return false;
    }
    std::cout << __FUNCTION__ << " otherContact: " << otherContact
              << " reference count: " << otherContact.use_count() << '\n';
    return name == otherContact->name && email == otherContact->email;
  }

 protected:
  std::string name;
  std::string email;
};

int main() {
  std::cout << "Foo\n";
  std::shared_ptr<Contact> contactJohn =
      std::make_shared<Contact>("John Doe", "jdoe@mail.com");
  std::shared_ptr<Contact> contactJane =
      std::make_shared<Contact>("Jane Smith", "jsmith@mail.com");
  std::cout << "ContactJane pointer: " << contactJane
            << " reference count: " << contactJane.use_count() << '\n';
  std::cout << "Are they equals?: " << contactJohn->compare(contactJane) << '\n';
  std::cout << "ContactJane pointer: " << contactJane
            << " use count: " << contactJane.use_count() << '\n';
  std::cout << "Bar\n";
  return 0;
}
```

```bash
Constructed: [0x1bf4c30]
Constructed: [0x1bf4cc0]                                             
ContactJane pointer: 0x1bf4cc0 use count: 1
compare otherContact: 0x1bf4cc0 use count: 2
Are they equals?: 0
ContactJane pointer: 0x1bf4cc0 use count: 1
Bar
Destructing: [0x1bf4cc0]
Destructing: [0x1bf4c30]
```

Notice how calling `compare` increases the use counter for `contactJane` to two. Then, `compare` returns and the use counter goes back to one. When `main` finishes, the counter goes to zero, and memory is properly deleted. We do not have to worry about whether `compare` will invalidate our memory or not. Another problem solved by the smart pointers!

In conclusion, you should avoid the use of regular pointers and start using `unique_ptr` and `shared_ptr` pointers. They will save you a lot of headaches and you will get the most out of the modern C++ language.


# More resources

+ [C++ Dynamic memory management reference documentation](http://en.cppreference.com/w/cpp/memory)
