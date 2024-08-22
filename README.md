# Go Concurrency Conundrums

Goroutines and channels are the flagship features of the Go language. Providing an alternative to threads, these features simplify concurrent programming while making best use of available CPU cores.

_Simplified_ is not quite the same as _simple_, however.

The following exercises are a mix of ideas that work, and _conundrums_ - puzzles that don't work, for you to figure out why.

## Things you need to know

Before you start, let's list a few facts about goroutines:

### goroutine 1 is privileged

- There is always at least one goroutine called _goroutine 1_.
- goroutine 1 is used to run the main() function
- Once the code inside goroutine 1 completes, goroutine 1 is terminated

### Concurrent sequential processes (CSP)

- Within any single goroutine, execution of code is _sequential_
- Between goroutines, no guarantees exist _whatsoever_ about relative ordering of the sequential code.

### Termination of goroutines

- Once goroutine 1 is terminated _all_ other goroutines are immediately terminated
- When a goroutine terminates, no further code is executed in that goroutine
- There is no concept of "waiting" for code to finish before termination. It just stops.

### Go Scheduler overview

- Is like a kind of operating system withing the Go language
- The scheduler decides when, or even if, it should execute code in a goroutine
- The scheduler may choose to run another goroutine _at any time_
- The scheduler maps goroutines onto native operating system threads
- The scheduler maps goroutines onto CPU cores, making full use of multicore processors
- The scheduler follows rules for what runs, when, and for how long
- You might not agree with the scheduler's decisions. Nobody cares. You cannot affect the rules. _This hugely affects your ability and intuition about Go concurrency_.
- Each goroutine tracks the state of any local variables inside it
- Each goroutine tracks which instruction was running last. Execution resumes from where it left off
- The instruction last running does not necessarily align with any Go source code. Source lines give rise to multiple processor instructions. The scheduler may switch to another goroutine _at any time_

### Go Scheduler rules

- Code within a goroutine may run for a long, uninterrupted piece of time
- Code within a goroutine may run for a short time with frequent interruptions
- No code may be running _at all_ within the scheduler
- When code is waiting for something outside of its own goroutine to happen, we call this _blocking_
- When a goroutine blocks, the scheduler will decide if some other goroutine can be given a turn to run
- There is no guarantee any section of code will be run completely inside an allocation of time by the scheduler

If all that sounds like it makes code hard to think about, and little bits of your brain start to dribble from your ears - _congratulations!_ You probably understand the depths of what you're about to get into.

Welcome to Go Concurrency, featuring Concurrent Sequential Processes!

## Hello world

Let's start at the beginning with Hello World:

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello, World!")
}
```

:question: How many goroutines does Hello World have?

We can find the answer by causing a [panic](https://go.dev/blog/defer-panic-and-recover):

```go
package main

import (
	"fmt"
)

func main() {
	panic("terminate goroutine")
	fmt.Println("Hello, World!")
}
```

which gives this output:

```bash
panic: terminate goroutine

goroutine 1 [running]:
main.main()
        /tmp/sandbox2796098968/prog.go:8 +0x25
```

We see there that the `main()` function runs inside _goroutine 1_. This has been created for us by the Go Scheduler, which is part of the Go runtime environment.

> All go programs have at least _one_ goroutine

The `panic()` function is intended to terminate a goroutine with immediate effect. Like a 'controlled program crash'.

We use `panic()` rarely, when our code detects that contnuing execution would do more harm than good. An example might be to `panic` ones storage is full on the computer. There is no point generating more data we cannot store.

## Creating a second goroutine

We use the 'go' keyword to create another goroutine. To do that, we need supply an executable statement to run inside the goroutine. This must follow the `go` keyword.

Let's try adding a simple `fmt.Println("Hello")` and see what happens:

```go
package main

import (
	"fmt"
)

func main() {
	go fmt.Println("Hello")
}
```

Run the code [here](https://goplay.tools/snippet/4EpvTq-Vbsj)

:question: What do you expect to see as output?
:question: What did you actually see as output?
:question: Why did that happen?

> Hint 1: What happens when goroutine 1 terminates?
> Hint 2: After which line in `main()` will goroutine 1 terminate?
> Hint 3: Is there any guarantee that the new goroutine will run before goroutine 1 terminates?

### The go keyword requests a gorutine is created, nothing more

The answer to the above behaviour is that our `main()` function completed very quickly. All it had to do was execute a `go` command. This completes almost instantly. It requests that a new goroutine is created to the scheduler. Nothing further is done. The goroutine does not start execution until later.

This means `main()` completes after that. Control is returned to the scheduler. goroutine 1 is terminated and _all other goroutines are terminated_. The `Println()` code never got the chance to run.

### Delaying goroutine 1 termination

For the next few exercises, we will kludge our way around the problem by using `time.Sleep(time.Second)` at the end of `main()`. This causes a one second pause to happen before `main()` exits and goroutine 1 terminates. This gives time for other goroutines to do some work.

> Using time.Sleep() is a terrible way to do this in production code!

## Functions and goroutines

Often, we supply a regular function call after the `go` keyword. Once the goroutine is scheduled to begin, this function will be called. Once the function exits, the goroutine will be garbage collected (cleaned up; no longer exists).

### Sequential processes

Within a single goroutine, execution is _sequential_:

```go
package main

import (
	"fmt"
	"time"
)

func showSequentialOperation() {
	fmt.Println("First")
	fmt.Println("Second")
	fmt.Println("Third")
}

func main() {
	go showSequentialOperation()

	time.Sleep(time.Second)
}
```

Run the code several times, noting the output.

:question: Is the output consistent?
:question: Do the lines run in the order we expect?

Run this code [here](https://goplay.tools/snippet/AznVcIV7op4)

This is what we mean by a sequential process. The lines of code run in sequence, one after the other. This is how we "normally" think about programming.

Concurrency changes this, because not everything is sequential anymore. And we are not in control of what happens when any more.

Tru running this code a few times, noting output:

```go
package main

import (
	"fmt"
	"time"
)

func showSequentialOperation(name string) {
	fmt.Println(name, "First")
	fmt.Println(name, "Second")
	fmt.Println(name, "Third")
}

func main() {
	go showSequentialOperation("A")
	go showSequentialOperation("B")
	go showSequentialOperation("C")

	time.Sleep(time.Second)
}
```

Run the code [here](https://goplay.tools/snippet/3F5MJGgo3wF)

We've added a name (A, B, C) to distinguish which goroutine is currently executing.

Here was the output from one run:

```
OUTPUT
A First
A Second
A Third
C First
C Second
C Third
B First
B Second
B Third
```

We can see that in each case, the output is strictly sequential: First, Second, Third.

In this run, the Go scheduler had called the function in goroutine "A" and allowed it to run all three sequential statements. Then it decided to hand control to goroutine "C" and allow that to complete all sequential statements. Then the same happens for "B".

> _This is true for this run, but it is not the only possibility_

Here was output from a different run, same code:

```
OUTPUT
A First
A Second
B First
B Second
B Third
**A Third**
C First
C Second
C Third
```

Can you see the difference?

The Go scheduler started off by running goroutine "A". That executed our `showSequentialOperations()` function, which worked in sequence. _But_ the Go Scheduler then decided enough was enough. Goroutine "A" was _blocked_ by the scheduler. It decided to give some execution time to goroutine "B".

Why? _ Because it can_. Those are the rules of the scheduler. _You may not agree with them, remember? Nobody cares, rememeber?_

In this run "B" completes all three lines of code. The function returns and hands back scontrol to the Go scheduler. This now decides to _unblock_ (i.e. _resume execution_ of) gorutine "A".

Because execution in any goroutine is sequential, the function resumes from where it left off. It had printed First anmd Second. So now it will resume to print Third.

> This is hugely important! Execution is sequential and resumes where it left off (within a goroutine)

### Goroutines preserve internal state of functions

## Hate your life: recursive concurrent functions

We've had too much fun already. Let's regret our life choices and look into recursive functions, running concurrently.

It goes without saying, these are hard to understand.

Let's start off with a recursive function `sigma(n)` that adds up all the numbers below it. So `sigma(3)` will work out 3 + 2 + 1, which is 6. `sigma(10)` works out to be 55.

We will call it _without_ concurrency first, three times:

```go
package main

import (
	"fmt"
)

// Sum of all numbers below 'number' down to 1
// example sigma(3) = 3 + 2 + 1 = 6
func sigma(number int) int {
	fmt.Println(number)
	if number == 1 {
		return 1
	}

	return number + sigma(number-1)
}

func main() {
	fmt.Println("A", sigma(3))
	fmt.Println("B", sigma(5))
	fmt.Println("C", sigma(10))
}
```

Run this code [here](https://goplay.tools/snippet/nRA2r7avJbl)

We get this output. You can see how the recursion is working from the Println values:

```
OUTPUT
3
2
1
A 6
5
4
3
2
1
B 15
10
9
8
7
6
5
4
3
2
1
C 55
```

Reasonably easy, given a grasp of recursion (which isn't easy at first!).

Now let's make each call to sigma work concurrently, by adding the `go` keyword. We will change how the printing works, so we can see the concurrency in action more clearly:

```go
package main

import (
	"fmt"
	"time"
)

// Sum of all numbers below 'number' down to 1
// example sigma(3) = 3 + 2 + 1 = 6
func sigma(number int) int {
	if number == 1 {
		return 1
	}

	res := number + sigma(number-1)
	fmt.Println(number, res)
	return res
}

func main() {

	go sigma(3)
	go sigma(5)
	go sigma(10)

	time.Sleep(time.Second)
}
```

Run this code [here](https://goplay.tools/snippet/B7Y0Yb7oRKp)

One example output run is this:

```
OUTPUT
2 3
3 6
4 10
5 15
6 21
7 28
8 36
9 45
10 55
2 3
2 3
3 6
4 10
5 15
3 6
```

If you run this a few times, you can see the Go Scheduler deciding to slice up the calculation differently each time.

We can see that the three function calls are running concurrently. Each recursive function gets allocated a slice of time to run, then the Go scheduler blocks it and unblocks another one.

Notice that the local variables are preserved. Each goroutine keeps track of the local varibale values in functions.

This is confusing at first. One function - many goroutines - many different values for your local variables in that one function!

It is best to think of the Go Scheduler as having made a "copy" of our function inside each goroutine.

It's actually just a stack; each time a goroutine is unblocked, the relevant gorutine information stack is consulted for where the code left off last. This includes execution point and all local variables, in all of the call stack.

> One function - called from ultiple goroutines - Go scheduler tracks and switches out local variable values for us

Have a nice, hot cup of tea! This is challenging to think about.

> Concurrency is hard. Recursion just makes it harder.

## Embarrasingly parallel - multiple unconnected goroutines

TODO example

## Channels: Communicating data between goroutines

TODO example

## Channel conundrum: deadlock

Channels are the preferred way for a Go program to communicate between different goroutines.

Here's an easy mistake to make:

```go
package main

func main() {
	ch := make(chan string)

	ch <- "Hello world"
}
```

> make(chan string) creates a channel, that can transport string values between goroutines. This syntax creates a maximum capacity of one value in the channel.

:question: What will go wrong when we run this code?

Try it [here](https://goplay.tools/snippet/TsgIeXzdOvY)

The output was not encouraging:

```bash
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /tmp/sandbox2174361683/prog.go:6 +0x28
```

:question: Why did that happen?

_Deadlock_ means that all the goroutines in a system are _blocked_. Each goroutine is waiting for another goroutine to do something - so none of them are able to move forward.

In this example, the main goroutine 1 tried to write a value to a channel. Normally, a different goroutine would read the value off that channel. But here, we did not create any other goroutines - so that will never be possible.

We are stuck. The act of writing to the channel has _blocked_ goroutine 1 from proceeding - and no help is in sight.

Blocking is very important in concurrent systems. But before we explain it, let's solve a few more conundrums.

## Increasing channel size

We can sort-of solve the deadlock by a simple change:

```go
package main

func main() {
	ch := make(chan string, 2)

	ch <- "Hello world"
}
```

We've added a 2 to the make() line.

> make(chan string, 2) creates a channel-of-string again, but gives a maximum capacity of two strings to the channel.

:question: What do we expect to happen when we run the code?

Try it [here](https://goplay.tools/snippet/9_HFN4Q5Kak)

Hmm. That's odd. The deadlock has gone away. But we still only have the main goroutine, goroutine 1. And in the last conundrum, we said we needed _another_ goroutine to fix the problem. But we don't have any.

Let's try another experiment. We will make a channel of capacity 2, but attempt two writes into the channel:

```go
package main

func main() {
	ch := make(chan string, 2)

	ch <- "Hello world"
	ch <- "Goodbye worries"
}
```

Try it [here](https://goplay.tools/snippet/VroODDGOn-2)

Still no deadlock.

What if we push our luck and attempt _three_ writes?

```go
package main

func main() {
	ch := make(chan string, 2)

	ch <- "Hello world"
	ch <- "Goodbye worries"
	ch <- "Hello deadlock, my old friend"
}
```

:question: What happens when we run the code now?

Try it [here](https://goplay.tools/snippet/rM7nweLx7dW)

Our old friend _deadlock_ returns:

```bash
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /tmp/sandbox2310932254/prog.go:8 +0x56
```

:question: Why does deadlock happen in this case?

We've uncovered a fundamental conept behind channels: _channel capacity_.

### Buffered and Unbuffered channels

We've experimented with _unbuffered_ and _buffered_ channels.

> _Unbuffered channel_: Maximum of zero things waiting to be read.

An _unbuffered channel_ is created by using `make()` without a size parameter:

```go
make(chan string)
```

If we have an unbuffered channel, we can only write to it _if something is already waiting to read from it_. If nothing is waiting to read _at that instant in time_, the write will fail.

There's no room at the inn, so to speak.

If the write fails, then we get a deadlock situation - the program cannot continue, because nothing will ever pick up the value we want to write.

> _Buffered channel_: Can hold a fixed number of things in a FIFO queue

A _buffered channel_ adds an internal _queue_ of values. This relaxes the rules a bit. We don't need anything waiting to read a value before we write. We only need space in the queue of values. If we have free space, we can write.

We create a buffered channel by adding a capcity parameter to make:

```go
make(chan string, 2)
```

Here, we set a capacity of two. The channel can hold two things, to be read later.

With buffered channels, the expectation is that some other goroutine will read values. This in turn frees up space for more writes.

This approach is excellent for handling bursts of traffic and work queues.

But what happens when we try to write once the queue is full?

#### Blocking

To understand what happens, we need to understand the concept of _blocking_.

> _Blocking_: Where a goroutine has to wait for something to happen elsewhere. It is blocked. the Go scheduler hands control to something that is not blocked.

- explain sequential order in goroutine
- explain some work has to wait (example)
- explain go scheduler does not waste time
- explain hands control to un blocked thing
- if everything is blocked: dead lock, as we saw

#### Blocking and buffered channels

- explain writes do not block if space
- writes do block if full
- if write blocked and nothing reading: deadlock

#### Why the unbuffered channel caused deadlock

TODO below apply knowledge to example

In our first example above, we created an _unbuffered_ channel.

An unbuffered channel is a channel that has a maximum capcity of zero things.

#### Why the buffered channel caused deadlock

TODO below apply knowledge to example

## Communicating data between two goroutines

TODO

## Unbuffered write before read conundrum

Here's a fault you'll doubtless stumble into. Attempting to write before read on an unbuffered channel.

TODO demo how read goroutine after write fails

## Fan out: worker pools

TODO

## Fan in: Reducers
