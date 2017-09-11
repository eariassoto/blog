---
title: "Creating a Simple Telegram Bot"
date: 2017-09-10T18:29:05-06:00
draft: true
---
Telegram bots are a great way to add a communication layer to your IoT projects. These bots are special Telegram accounts that are run by software, not by people. Telegram bots communicate with the servers using a HTTP API. Messages sent to your bot will be passed from the Telegram servers to your code.

<!--more-->
You can create you bot using just the HTTP calls to the API. The API documentation can be found [here](https://core.telegram.org/bots/api). However, it is more simple if you find a wrapper API written in the your project’s language. In this post, I will show you how to create a simple bot running on a Raspberry Pi using a C++ Telegram Bot API wrapper.

# Installing dependencies

These are the packages you need to build the C++ library for the Telegram Bot API:

{{<codeblock lang="bash">}}
$ sudo apt-get install g++ make binutils cmake libssl-dev libboost-system-dev libboost-iostreams-dev
{{</codeblock>}}

Note: Make sure the Boost version installed after this step is 1.59 or latest. Otherwise, the C++ library will not compile. As of Set/17, Raspbian Jessie is installing Boost 1.55. If this is your case, you need to build Boost from sources or update to Raspbian Stretch.

Now we can build and install the library:
{{<codeblock lang="bash">}}
$ git clone https://github.com/reo7sp/tgbot-cpp
$ cd tgbot-cpp
$ cmake .
$ make -j4
$ sudo make install
{{</codeblock>}}

# Registering a Telegram bot

The process is fairly simple: open a Telegram conversation with BotFather and type /newbot. After that, enter a name and a username for your bot.

image

In the last confirmation message, the bot will give you a HTTP API token. Copy that token because we will need it when we write the code for our bot.

# Writing the code
We will write a bot that will tell us if a sentence is a palindrome or not. Go ahead and make a `palindrome_bot` folder. The folder needs to have these files:
{{<codeblock "CMakeLists.txt" "cmake">}}
{{</codeblock>}}

We will compile the bot using `cmake`. Add this to the `CMakeLists.txt` file:

{{<codeblock "CMakeLists.txt" "cmake">}}
cmake_minimum_required(VERSION 2.8)

set (PROY palindrome_bot)
project (${PROY} C CXX)

set(Boost_USE_MULTITHREADED ON)
find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(Boost COMPONENTS system iostreams REQUIRED)

# Source files folder
set (SRC_DIR src)

# Compiler flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall")

# Bot source files
set (SOURCE
    ${SRC_DIR}/main.cpp
)

include_directories(/usr/local/include
    ${OPENSSL_INCLUDE_DIR}
    ${Boost_INCLUDE_DIR}
)

target_link_libraries(${PROY}
    /usr/local/lib/libTgBot.a ${CMAKE_THREAD_LIBS_INIT}
    ${OPENSSL_LIBRARIES} ${Boost_LIBRARIES}
)

# Output folder
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/bin")

# Add executable
add_executable (${PROY} ${SOURCE})
{{</codeblock>}}

{{<codeblock "src/main.cpp" "c++">}}
#include <exception>
#include <signal.h>
#include <stdio.h>

#include <tgbot/tgbot.h>

using namespace std;
using namespace TgBot;

bool sigintGot = false;

int main() {
  Bot bot("YOU API KEY");
  bot.getEvents().onCommand("start", [&bot](Message::Ptr message) {
    bot.getApi().sendMessage(message->chat->id, "Hi!");
  });
  bot.getEvents().onAnyMessage([&bot](Message::Ptr message) {
    printf("User wrote %s\n", message->text.c_str());
    if (StringTools::startsWith(message->text, "/start")) {
      return;
    }
    bot.getApi().sendMessage(message->chat->id,
                             "Your message is: " + message->text);
  });

  signal(SIGINT, [](int s) {
    printf("SIGINT got");
    sigintGot = true;
  });
  try {
    printf("Bot username: %s\n", bot.getApi().getMe()->username.c_str());

    TgLongPoll longPoll(bot);
    while (!sigintGot) {
      printf("Long poll started\n");
      longPoll.start();
    }
  } catch (exception &e) {
    printf("error: %s\n", e.what());
  }

  return 0;
}
{{</codeblock>}}
