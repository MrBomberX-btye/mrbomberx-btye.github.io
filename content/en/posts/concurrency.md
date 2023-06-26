---
title:  "Concurrency in Go - A deeper look into Go's runtime scheduler."
date:  2023-06-18
draft:  false
enableToc: true
enableTocContent: true
description: "I talk about Concurrecny in Go and How the Go's runtime scheduler handles it."
tags:
- misc
image: "images/go/concurrency.png"
---


## Table of contents

1. Intro go CSP Concurrency model in Go (routines, channels, mutex, callback)
2. CSP concurrency patterns
3. Mutex
4. Deadlocks
5. Concurrency Patterns

## Introduction

> you can head over to this <a href="https://github.com/MrBomber0x001/hashnode-material/tree/main/Go">Repo</a>, in which I've maintained all of the code examples below.

Go's built-in support for concurrency is one of its most significant features, making it a popular choice for building high-performance and scalable systems. In this article, we're going to explore the concepts of concurrency in Golang, including its core primitives, such as **goroutines** and **channels**, and how they enable developers to build concurrent programs with ease.

First things first, let's begin with some concepts to ensure we're all in the same context!

## **Concurrency vs Parallelism**

> Concurrency in Go is the ability for functions to run independently of each other.

You see, it's that simple 😀, but we can talk a lot more about that simple fact just enough to see it's not that simple 😂

As Rob pike once said:

> Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once

That's the key difference between concurrency and parallelism, if you've got a good mental model of each one's characteristics, it will be an easy mission to tell the difference!

Before talking about the difference in depth, you should have a clear understanding of processes and threads.

You can think of a process like a container that holds all the resources an application uses and maintains as it runs.

A thread is a path of execution that’s scheduled by the operating system to run the code that you write in your functions.

<img src="https://cdn.hashnode.com/res/hashnode/image/upload/v1679430325098/7404d099-4842-47b9-a7e8-e04bac02b5d9.png" />

The operating system schedules threads to run against processors regardless of the process they belong to.

The operating system schedules threads to run against physical processors and the Go runtime schedules goroutines to run against logical processors. Each logical processor is individually bound to a single operating system thread

These logical processors are used to execute all the goroutines that are created. Even with a single logical processor, hundreds of thousands of goroutines can be scheduled to run concurrently with amazing efficiency and performance.

Concurrency is not parallelism. **Parallelism can only be achieved when multiple pieces of code are executing simultaneously against <mark>different physical </mark> processors**

> Parallelism is about doing a lot of things at once. Concurrency is about managing a lot of things at once

Concurrency is when different (unrelated/related) tasks can be executed on the same resource (CPU, machine, cluster) with **overlapping** time frames, While Parallelism is when related tasks or a task with multiple sub-tasks are executed in parallel (same start - probably same finish - time)

In many cases, concurrency can outperform parallelism, because the strain on the operating system and hardware is much less, which allows the system to do more. This less-is-more philosophy is a mantra of the language.

> If you want to run goroutines in parallel, you must use more than one logical processor.

When there are multiple logical processors, the scheduler will evenly distribute goroutines between the logical processors

But to have true parallelism, you still need to run your program on a machine with multiple physical processors

If not, then the goroutines will be running concurrently against a single physical processor, even though the Go runtime is using multiple threads.

<img src="https://cdn.hashnode.com/res/hashnode/image/upload/v1679430025472/4c7a1958-4e3f-4ca9-8725-072918295d69.png" alt="Concurrency vs parallelism" />

Go's Concurrency allows us to:

- Construct `Streaming data pipelines`
- Make efficient use of I/O and multiple CPUs
- Allows complex systems with multiple components
- `Routines` can start, run and complete simultaneously
- Raises the efficiency of the applications

## Goroutines

When a function is created as a goroutine, it’s treated as an **independent unit** of work that gets scheduled and then executed on an available logical processor.

Goroutines are extremely **lightweight** and efficient, with minimal overhead, which means that you can create thousands of them without any significant impact on the performance of your program.

A goroutine is a simple function prefixed with `go` keyword

```go
func main(){
    go SayHi("Hi")
    time.Sleep(time.Millisecond * 1000)
}
func SayHi(message string) {
    fmt.Println(mesage)
}
```

The term `goroutine` comes originally from `coroutines`

A coroutine is a unit of a program that can be paused and resumed multiple times in the same program, keeping the state of the coroutine between invocations, generator functions in python is an implementation of coroutines

While a subroutine is a unit of a program that's executed sequentially, invocations are independent so the internal state of a subroutine is not shared between invocations, a subroutine finishes when all instructions are executed, cannot be resumed afterward

<img src="https://cdn.discordapp.com/attachments/1084610105397485578/1087858713097945209/image.png" />

## **Go's runtime scheduler in depth**

Having a good understanding of Go's runtime scheduler gives you a clear and consider understanding of how concurrency gets handled in Golang, so let's uncover some of the runtime behavior

The **Go runtime scheduler** is a sophisticated piece of software that manages all the goroutines that are created and need processor time. The scheduler sits on

top of the operating system, binding the operating system’s threads to logical processors which, in turn, execute goroutines. The scheduler controls everything related to which goroutines are running on which logical processors at any given time.

**<mark>Execution steps of goroutines</mark>**

1. As goroutines are created and ready to run, they’re placed in the scheduler’s global run queue.

2. Soon after, they’re assigned to a logical processor and placed into a local run queue for that logical processor.

3. From there, a goroutine waits its turn to be given the logical processor for execution.

<img src="https://cdn.discordapp.com/attachments/1087493279777562734/1087776364180013146/Screenshot_2023-03-20_231224.png" alt="Go's runtime scheduler behaviour" />

Sometimes a running goroutine may need to perform a blocking syscall, such as opening a file.

When this happens, the thread and goroutine are detached from the logical processor and the thread continues to block waiting for the syscall to return. In the meantime, there’s a logical processor without a thread. So the scheduler creates a new thread and attaches it to the logical processor.

Then the scheduler will choose another goroutine from the local run queue for execution. Once the syscall returns, the goroutine is placed back into a local run queue, and the thread is put aside for future use.

🟢 If a goroutine needs to make a network call, the process is a bit different. In I/O this case, the goroutine is detached from the logical processor and moved to the run time integrated network poller

🟢 Once the poller indicates a read or write operation is ready, the goroutine is assigned back to a logical processor to handle the operation. There’s no restriction built into the scheduler for the number of logical processors that can be created. But the runtime limits each program to a maximum of 10,000 threads by default

Now let the fun begin,

In this section, we are going to play around with the number of logical processors assigned to handle each situation below and see how this affects the overall behavior.

```go
package main

import (
 "fmt"
 "runtime"
 "sync"
)

func main(){
 runtime.GOMAXPROCS(1) // allocate only one logical processor

 var wg sync.WaitGroup
 wg.Add(2) // add 2 to wait for 2 goroutines

 fmt.Println("Starting go routines")
 // creating an anoymous function that print the alphabets 
 go func() {
  defer wg.Done()

  for count := 0; count < 3; count++ {
   for char := 'a'; char < 'a'+26; char++ {
    fmt.Printf("%c ", char)
   }
  }
 }()

 go func() {
  defer wg.Done()

  for count := 0; count < 3; count++ {
   for char := 'A'; char < 'A'+26; char++ {
    fmt.Printf("%c ", char)
   }
  }
 }()

 fmt.Println("Waiting to finish")
 wg.Wait()

 fmt.Println("\n Terminating program...")
}
```

If you tried to run the code above, you will see the following output:

```bash
Create Goroutines
Waiting To Finish
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z A B C D E F G H I J K L M
N O P Q R S T U V W X Y Z A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
a b c d e f g h i j k l m n o p q r s t u v w x y z a b c d e f g h i j k l m
n o p q r s t u v w x y z a b c d e f g h i j k l m n o p q r s t u v w x y z
Terminating Program
```

A `waitGroup` is a counting semaphore that can be used to maintain a record of running goroutines

For each time `wg.Done` is invoked, the counter is decreased by `1` till it reaches `0` and then the `main` can terminate safely.

The keyword `defer` is used to schedule other functions from inside the executing defer function to be called when the function returns, based on the internal algorithms of the scheduler, a running goroutine can be stopped and rescheduled to run again before it finishes its work

The scheduler does this to prevent any single goroutine from holding the logical processor hostage. It will stop the currently running goroutine and give another runnable goroutine a chance to run

Here's a **diagram** of what's already happened

![](<https://cdn.hashnode.com/res/hashnode/image/upload/v1679432549952/371d8e7c-5237-4682-bda4-2d792784488a.png> align="center")

🤔 What if we tried to change the number of logical processors from 1 to 2, what do you think goin' to happen❓

```go
package main


func main() {
 runtime.GOMAXPROCS(2)
 var wg sync.WaitGroup
 wg.Add(2)
 fmt.Println("Starting go routines")
 go func() {
  defer wg.Done()

  for count := 0; count < 3; count++ {
   for char := 'a'; char < 'a'+26; char++ {
    fmt.Printf("%c ", char)
   }
  }
 }()

 go func() {
  defer wg.Done()

  for count := 0; count < 3; count++ {
   for char := 'A'; char < 'A'+26; char++ {
    fmt.Printf("%c ", char)
   }
  }
 }()

 fmt.Println("Waiting to finish")
 wg.Wait()

 fmt.Println("\n Terminating program...")
}
```

probably you will see output looks like this

```bash
Create Goroutines
Waiting To Finish
A B C a D E b F c G d H e I f J g K h L i M j N k O l P m Q n R o S p T
q U r V s W t X u Y v Z w A x B y C z D a E b F c G d H e I f J g K h L
i M j N k O l P m Q n R o S p T q U r V s W t X u Y v Z w A x B y C z D
a E b F c G d H e I f J g K h L i M j N k O l P m Q n R o S p T q U r V
s W t X u Y v Z w x y z
Terminating Program
```

> **It’s important to note** that using <mark>more than one logical processor doesn’t necessarily mean better performance</mark>, Benchmarking is required to understand how your program performs when changing any configuration parameters runtime

Another good example to demonstrate the idea of using a single logical processor more clearly is by doing some heavy work

```go
package main

import (
 "fmt"
 "runtime"
 "sync"
)

var wg sync.WaitGroup

func main() {
 runtime.GOMAXPROCS(1)
 wg.Add(2)

 fmt.Println("\ncreating goroutines")
 go printPrime("A")
 go printPrime("B")
 fmt.Println("\nwaiting to finish")
 wg.Wait()

 fmt.Println("Terminating")
}

func printPrime(prefix string) {
 // printPrime displays prime numbers for the first 5000 numbers
 defer wg.Done()
next:
 for outer := 2; outer < 5000; outer++ {
  for inner := 2; inner < outer; inner++ {
   if outer%inner == 0 {
    continue next
   }
  }
  fmt.Printf("%s:%d\n", prefix, outer)
 }

 fmt.Printf("%s completed", prefix)

}
```

notice this output

```bash
Create Goroutines
Waiting To Finish
B:2
B:3
...
B:4583
B:4591
A:3 ** Goroutines Swapped
A:5
...
A:4561
A:4567
B:4603 ** Goroutines Swapped
B:4621
...
Completed B
A:4457 ** Goroutines Swapped
A:4463
...
A:4993
A:4999
Completed A
Terminating Program
```

Goroutine B begins to display prime numbers first. Once goroutine B prints prime number 4591, the scheduler swaps out the goroutine for goroutine A. Goroutine A is then given some time on the thread and swapped out for the B goroutine once again. The B goroutine is allowed to finish all its work. Once goroutine B returns, you see that goroutine A is given back the thread to finish its work.

> Remember that goroutines can only run in parallel if there’s more than one logical processor and there’s a physical processor available to run each goroutine simultaneously.

## **CSP**

Concurrency synchronization comes from a paradigm called communicating sequential processes or CSP. CSP is a message-passing model that works by communicating data between goroutines instead of locking data to synchronize access. The key data type for synchronizing and passing messages between goroutines is called a channel

> CSP provides a model for thinking about concurrency that makes it less hard
>
> Simply take the program apart and make the pieces talk to each other

**Head over to this presentation by the creator of Erlang which discusses this concept in an in-depth details**

%[https://www.infoq.com/presentations/erlang-software-for-a-concurrent-world/]

<img src="https://www.karanpratapsingh.com/_next/image?url=%2Fstatic%2Fcourses%2Fgo%2Fchapter-IV%2Fconcurrency%2Fcsp.png&w=2048&q=75" align="left" alt="Process communication through channel - Credits: Slides"/>

### **Basic Concepts**

- Data race: two processes or more trying to access the same resources concurrently maybe one was reading while the other was writing

- Race conditions: the timing or order of events affects the correctness of a piece of code

- Deadlock: all processes are blocked while waiting for each other and the program cannot proceed further

- Livelocks: processes that perform concurrent operations, but do nothing to move the state of the program forward

- Starvation: when a process is deprived of necessary resources and unable to complete its functions, might happen because of `deadlock` or `insufficient scheduling algorithms`

#### **Deadlocks**

There are four 4 conditions, known as `Coffman conditions` which when are satisfied, then a deadlock occurs

1. mutual execution

2. Hold and wait

3. No Preemption

4. Circular wait

1. Mutual Execution A concurrent process holds at least one resource at any one time making it non-sharable.

<img src="https://www.karanpratapsingh.com/_next/image?url=%2Fstatic%2Fcourses%2Fgo%2Fchapter-IV%2Fconcurrency%2Fmutual-exclusion.png&w=1920&q=75" align="left" />

1. Hold and wait A concurrent process holds a resource and is waiting for an additional resource.

Process 2 is allocating (holding) resources 2 and 3 and waiting for resource 1 which is locked by process 1.

<img src="https://www.karanpratapsingh.com/_next/image?url=%2Fstatic%2Fcourses%2Fgo%2Fchapter-IV%2Fconcurrency%2Fhold-and-wait.png&w=1920&q=75" align="left" />

1. No preemption A resource held by a concurrent process cannot be taken away by the system. It can be freed by the process of holding it.

In the diagram below, Process 2 cannot preempt Resource 1 from Process 1. It will only be released when Process 1 relinquishes it voluntarily after its execution is complete.

<img src="https://www.karanpratapsingh.com/_next/image?url=%2Fstatic%2Fcourses%2Fgo%2Fchapter-IV%2Fconcurrency%2Fno-preemption.png&w=1920&q=75" />

1. Circular wait A process is waiting for the resource held by the second process, which is waiting for the resource held by the third process, and so on, till the last process is waiting for a resource held by the first process. Hence, forming a circular chain. (All if waiting)

<img src="<https://www.karanpratapsingh.com/_next/image?url=%2Fstatic%2Fcourses%2Fgo%2Fchapter-IV%2Fconcurrency%2Fcircular-wait.png&w=1920&q=75" align="left" />

## Conclusion

Go's runtime scheduler is very intelligent at handling concurrency, with algorithms designed to take care of the workload of your software and distribute it among available threads is something very efficient for your system resources and overall performance!

In the next write-up, we'll continue our journey talking about **conventional synchronization** such as mutexes, channels, and Data races in further detail, stay tuned!

Lastly, thanks for reading this write-up, if it helped you learn something new, please consider buying me a cup of coffee ♥.

## Resources

- [Go class \[the best\]](https://www.youtube.com/playlist?list=PLoILbKo9rG3skRCj37Kn5Zj803hhiuRK6)
- [Deep Dive into Go Concurrency](https://betterprogramming.pub/deep-dive-into-concurrency-of-go-93002344d37b)
- [wtf is goroutines](https://pstree.cc/wtf-is-goroutines/)
- [Concurrency is not Parallelism Rob Pike](https://www.youtube.com/watch?v=oV9rvDllKEg)
- [Effective Go - concurrency](https://go.dev/doc/effective_go#concurrency)
- [Go Working Pool Mode](https://www.sobyte.net/post/2022-03/go-working-pool-mode/)
- [Go Concurrency Patterns](https://xie.infoq.cn/article/a0880b7d215f7b82bc3a0380a)
- [More on Patterns](https://www.karanpratapsingh.com/courses/go/advanced-concurrency-patterns)
- [Go in action \[book\]](https://file+.vscode-resource.vscode-cdn.net/c%3A/Users/ncm/Desktop/studying/Go-library/Concurrency%20Patterns/concurrency/README.md)
- [The Go Programming Language \[book\]](https://file+.vscode-resource.vscode-cdn.net/c%3A/Users/ncm/Desktop/studying/Go-library/Concurrency%20Patterns/concurrency/README.md)
