---
title: "Minimal implementations in Modern C++: Producer-Consumer problem"
date: 2020-04-20T14:51:44+02:00
categories:
- Development
tags:
- C++
- Design patterns
#thumbnailImage: //example.com/image.jpg
draft: false
---

This implementation was inspired by Stackoverflow user  [Yakk - Adam Nevraumont](https://stackoverflow.com/users/1774667/yakk-adam-nevraumont)'s answer for a [question about `std::condition_variable`](https://stackoverflow.com/questions/57219650/stdcondition-variablenotify-all-only-wakes-up-one-thread-in-my-threadpool). I extended it to make it a working example, and I plan to use it as a base for a new project I have in mind.

<!--more-->

## What is the Producer-Consumer problem?

This problem is a classic example of synchronization and parallel computing. Suppose you have available multiple execution threads. In addition to that, you also have tasks that can be executed at the same time, independent from each other. The problem now is how we distribute the tasks between those threads.

![Producer - Consumer problem](/static/img/prodcons.png)

In a simple producer-consumer problem, the producer will create tasks and push them to a queue. The queue thread will handle the synchronization. The consumer threads will wake up only when they have to execute a task. When the consumer thread finishes a task, it sleeps until the queue thread sends a new signal.

## To the implementation

In C++11 a new synchronization primitive was introduced: [`std::condition_variable`](https://en.cppreference.com/w/cpp/thread/condition_variable). This primitive allows us to block one or more threads until a shared state is modified and the thread, or threads, has been modified. This primitive needs a `std::mutex` for the blocker thread to modify the stated state, and for the blocked threads to wait on the condition variable.

As an overview, our producer and queue will be on the same thread. Our consumers will run on separate threads and will get the `std::condition_variable` and `std::mutex` from the queue. They will start a loop in which they will wait on the condition variable and check if there is a new task, or if the queue has stopped pushing new tasks. Our sample program will send 10 tasks, wait for an amount of time, and stop the queue.

The task we are going to push into the queue is going to be very simple. It will require an Id number, a duration in milliseconds, and a mutex.

```c++ {linenos=inline} {linenos=inline}
#pragma once
#include <chrono>
#include <mutex>

class Task {
   public:
    Task(unsigned int id, std::chrono::milliseconds duration,
         std::mutex& coutMutex);

    void Execute();

    unsigned int GetId() const { return m_Id; }

   private:
    unsigned int m_Id = 0;
    std::chrono::milliseconds m_Duration;
    std::mutex& m_CoutMutex;
};
```

We are going to use this mutex to have exclusive use in the `std::cout` buffer. Otherwise, our messages are going to be all mixed out in the standard output.

The `Execute` method will also be very simple: it will sleep the thread for the `m_Duration` milliseconds, and print out this duration before exiting.

```c++ {linenos=inline}
void Task::Execute() {
    std::this_thread::sleep_for(m_Duration);
    {
        std::scoped_lock<std::mutex> guard(m_CoutMutex);
        std::cout << "Task {" << m_Id << "} finished in " << m_Duration.count()
                  << "ms.\n";
    }
}
```

The `std::scoped_lock` was introduced in C++17 and it works similarly as `std::lock_guard`. One difference is that `std::scoped_lock` allows us to try to acquire more than one `std::mutex`, preventing possible deadlocks.

It is very important to surround the critical section in brackets. That will make the lock to release the `std::mutex` as soon as it is no longer needed. Remember that `std::scoped_lock` behaves according to the [RAII principle](https://en.cppreference.com/w/cpp/language/raii).

We are going to implement a `TaskQueue` class to wrap our `std::queue` and our synchronization mechanism.

```c++ {linenos=inline}
#pragma once
#include <mutex>
#include <queue>
#include <tuple>

class Task;

class TaskQueue {
   public:
    ~TaskQueue();

    // Thread safe functions for producers
    void PushTask(Task* t);
    void PushTasks(std::vector<Task*>& tasks);
    void StopQueue();
    
    // Way for consumers to get the sync variables
    std::tuple<std::mutex&, std::condition_variable&> Subscribe();
    
    // Non-thread safe function. Consumers must ensure
    // lock acquisition
    bool HasPendingTask() const { return !m_Queue.empty(); }
    bool IsQueueStopped() const { return m_QueueIsStopped; }
    Task* GetNextTask();
   
   private:
    bool m_QueueIsStopped = false;
    std::queue<Task*> m_Queue;
    std::mutex m_Mutex;
    std::condition_variable m_ConditionVariable;
};
```

Producers will interact with this queue by pushing `Task` instances. In our case, we will handle the object destruction in the queue, but it can be improved with smart pointers. `StopQueue` will allow the producer to stop the queue and send a signal to the consumers that the queue is no longer pushing new tasks.

Consumers will get access to the `std::mutex`, and `std::condition_variable` to wait for new tasks. The `PushTask` and `PushTasks` functions will acquire the mutex to exclusively add new tasks. After that, the lock is no longer needed to wake up the waiting threads.

```c++ {linenos=inline}
void TaskQueue::PushTask(Task* t) {
    {
        std::scoped_lock l{m_Mutex};
        m_Queue.push(t);
    }
    m_ConditionVariable.notify_one();
}

void TaskQueue::PushTasks(std::vector<Task*>& tasks) {
    {
        std::scoped_lock l{m_Mutex};
        for (Task* t : tasks) {
            m_Queue.push(t);
        }
    }
    m_ConditionVariable.notify_all();
}
```

When the consumers awake, they will use the non-thread safe functions to verify if they need to either: consume a new task, or finish execution because the queue will no longer send new tasks.

For the consumer thread, I will break the method for a more detailed look:

```c++ {linenos=inline}
void WorkerThread(const int workerId, TaskQueue& taskQueue,
                  std::mutex& coutMutex) {
    auto& [m, cv] = taskQueue.Subscribe();
    
```
This declaration is a new feature introduced in C++17 called [Structured binding declaration](https://en.cppreference.com/w/cpp/language/structured_binding). It makes the declaration more readable, and it is the same as declaring and assigning to a `std::tuple`. Now that we have the synchronization mechanism, we can go into our main loop:

```c++ {linenos=inline,linenostart=5}
    while (true) {
        auto data = [&]() -> std::optional<Task*> {
            std::unique_lock l{m};
            cv.wait(l, [&] {
                return taskQueue.IsQueueStopped() || taskQueue.HasPendingTask();
            });
            if (taskQueue.IsQueueStopped()) {
                return {};
            } else {
                Task* taskToProcess = taskQueue.GetNextTask();
                assert(taskToProcess != nullptr);
                return taskToProcess;
            }
        }();
```

We will try to get out Task using C++17's `std::optional`. This container allows us to create an instance that may not have a value after initialization. The worker thread will wait for it to be woken up inside of the `std::optional` constructor. If the task queue has a pending task, we will get it and return it as the value inside the `std::optional`.

```c++ {linenos=inline,linenostart=17}
        if (!data) {
            break;
        }

        Task* taskPtr = *task;
        {
            std::scoped_lock<std::mutex> guard(coutMutex);
            std::cout << "Worker {" << workerId << "} is executing task {"
                      << taskPtr->GetId() << "}.\n";
        }

        // process the data
        taskPtr->Execute();
        delete taskPtr;
    }
}
```
If the `std::optional` returns without a value, the thread ends execution. Otherwise we print a debug message and execute the task.

Let's put it all together in our main function:

```c++ {linenos=inline}
int main(int argc, char* argv[]) {
    const unsigned int thread_pool_size =
        std::thread::hardware_concurrency() - 1;
    assert(thread_pool_size > 0);

    std::mutex coutMutex;

    std::vector<std::thread> thread_pool(thread_pool_size);
    TaskQueue taskQueue;

    for (size_t i = 0; i < thread_pool_size; ++i) {
        thread_pool[i] = std::thread(WorkerThread, i, std::ref(taskQueue),
                                     std::ref(coutMutex));
    }
```

Here we create an array of worker threads. At this point they will start execution and immediately sleep until we push new tasks.

```c++ {linenos=inline,linenostart=15}
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> taskDuration(0, 1000);

    for (int i = 0; i < 5; ++i) {
        taskQueue.PushTask(
            new Task(i, std::chrono::milliseconds(taskDuration(gen)),
                     std::ref(coutMutex)));
    }

    std::vector<Task*> taskBatch;
    taskBatch.resize(5);
    for (int i = 0; i < 5; ++i) {
        taskBatch[i] =
            new Task(i + 5, std::chrono::milliseconds(taskDuration(gen)),
                     std::ref(coutMutex));
    }
    taskQueue.PushTasks(taskBatch);
```

Usually, the producer thread is also a loop that creates tasks until the program finishes. For our example, we will create only 10 tasks with random durations. The duration is a random value from 0 to 1000 milliseconds.

```c++ {linenos=inline,linenostart=33}
    std::this_thread::sleep_for(std::chrono::seconds(10));
    taskQueue.StopQueue();

    for (size_t i = 0; i < thread_pool_size; ++i) {
        thread_pool[i].join();
    }
    return 0;
}
```

To make the end of our program simple, we will sleep the producer thread for 10 seconds to hopefully process all tasks, and stop the queue. In the last statement, we wait for all consumers to finish, and we exit the program.

An example output of this program may be:

```bash
Worker {0} is executing task {0}.
Worker {1} is executing task {1}.
Worker {2} is executing task {2}.
Worker {3} is executing task {3}.
Worker {4} is executing task {4}.
Task {4} finished in 418ms.
Worker {4} is executing task {5}.
Task {3} finished in 526ms.
Worker {3} is executing task {6}.
Task {5} finished in 179ms.
Worker {4} is executing task {7}.
Task {2} finished in 640ms.
Worker {2} is executing task {8}.
Task {1} finished in 781ms.
Worker {1} is executing task {9}.
Task {6} finished in 277ms.
Task {0} finished in 912ms.
Task {7} finished in 337ms.
Task {8} finished in 527ms.
Task {9} finished in 861ms.
```
