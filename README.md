# Go Concurrency Conundrums

Goroutines and channels are the flagship features of the Go language. Providing an alternative to threads, these features simplify concurrent programming while making best use of available CPU resources.

_Simplified_ is not quite the same as _simple_, however.

The following exercises are a mix of ideas that work, and _conundrums_ - puzzles that don't work, for you to figure out why.

## Things you need to know before you start

Before you start, let's list a few facts about goroutines:

### goroutine 1 is privileged

- There is always at least one goroutine called _goroutine 1_.
- goroutine 1 is used to run the `main()` function
- Once the code inside goroutine 1 completes, goroutine 1 is terminated

### Termination of goroutines

- When a goroutine terminates, no further code is executed in that goroutine
- Once goroutine 1 is terminated _all_ other goroutines are immediately terminated
- There is no concept of "waiting" for code to finish before termination. It just stops.

### Communicating sequential processes (CSP)

- A goroutine is the Go version of a [Communicating Sequential Process](https://en.wikipedia.org/wiki/Communicating_sequential_processes#:~:text=CSP%20was%20first%20described%20in,a%20secure%20e%2Dcommerce%20system.)
- Within any single goroutine, execution of code is _sequential_
- Between goroutines, no guarantees exist _whatsoever_ about relative ordering of the sequential code.
- Goroutines may communicate with each other safely by using _channels_

### Go Scheduler overview

- Is like a kind of operating system withing the Go language
- Many (1000s-100,000s) goroutines may exist
- goroutines are _lightweight_ in terms of memory/processor overhead

### Go Scheduler rules

- The scheduler decides when, or even if, it should execute code in a goroutine
- The scheduler may choose to run another goroutine _at any time_
- The scheduler maps manygoroutines onto native operating system threads
- The scheduler maps goroutines onto CPU cores, making full use of multicore processors
- The scheduler follows rules for what runs, when, and for how long
- You might not agree with the scheduler's decisions. Nobody cares. You cannot affect the rules. _This hugely affects your ability and intuition about Go concurrency_.
- Each goroutine tracks the state of any local variables inside it
- Each goroutine tracks which instruction was running last. Execution resumes from where it left off
- The instruction last running does not necessarily align with any Go source code. Source lines give rise to multiple processor instructions. The scheduler may switch to another goroutine _at any time_

### Time slicing

- The Go Scheduler will allocate a time slice for a goroutine to run
- Do not assume the code you want to complete as a single block will complete inside a time slice
- Code within a goroutine may run for a long, uninterrupted piece of time
- Code within a goroutine may run for a short time with frequent interruptions
- No code may be running _at all_ within the scheduler (waiting, or deadlock)
- There is no guarantee any section of code will be run completely inside an allocation of time by the scheduler

### Blocking

- When code is waiting for something outside of its own goroutine to happen, we call this _blocking_
- When a goroutine blocks, the scheduler will decide if some other goroutine can be given a turn to run
- _deadlock_ occurs when no goroutine is able to run. A `panic' occurs inside the Go scheduler

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

- :question: How many goroutines does Hello World have?

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

- :question: What do you expect to see as output?
- :question: What did you actually see as output?
- :question: Why did that happen?

> Hint 1: What happens when goroutine 1 terminates?
>
> Hint 2: After which line in `main()` will goroutine 1 terminate?
>
> Hint 3: Is there any guarantee that the new goroutine will run before goroutine 1 terminates?

### The go keyword requests a gorutine is created, nothing more

The answer to the above behaviour is that our `main()` function completed very quickly. All it had to do was execute a `go` command. This completes almost instantly. It requests that a new goroutine is created to the scheduler. Nothing further is done. The goroutine does not start execution until later.

This means `main()` completes after that. Control is returned to the scheduler. goroutine 1 is terminated and _all other goroutines are terminated_. The `Println()` code never got the chance to run.

> The 'go' keyword _does not execute_ the code after it. That happens later.

### Delaying goroutine 1 termination

For the next few exercises, we will kludge our way around the problem by using `time.Sleep(time.Second)` at the end of `main()`. This causes a one second pause to happen before `main()` exits and goroutine 1 terminates. This gives time for other goroutines to do some work.

> Using time.Sleep() is a terrible way to do this in production code!

## Running functions inside goroutines

The code block after the `go` keyword is often a function call:

```go
func sayHello(name string) {
    fmt.Println("Hello", name)
}

go sayHello("Alan")
```

The function

- should not return any value
- may accept zero or more local parameters.
- (usually) )should not reference module-wide variables or global variables

The `go` statement here has the following effect

- The Go Scheduler will create a new goroutine
- The function will be linked to that goroutine
- The function as 'ready to run'
- _The function will not start executing yet!_
- At _some point later_, the Go Scheduler will begin execution of that function
- The function will execute _sequentially_ ie lines run in the 'normal' order
- The function will be paused and resumed whenever the Go Scheduler feels like it
- Once the function exits, the goroutine will be terminated and removed automatically

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

- :question: Is the output consistent?
- :question: Do the lines run in the order we expect?

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

We can see that in each case, the output within each goroutine is strictly sequential: First, Second, Third.

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
** A Third **
C First
C Second
C Third
```

Can you see the difference?

The Go scheduler started off by running goroutine "A". That executed our `showSequentialOperations()` function, which worked in sequence. _But_ the Go Scheduler then decided enough was enough. Goroutine "A" was _paused_ by the scheduler. It decided to give some execution time to goroutine "B".

Why? _Because it can_. Those are the rules of the scheduler. _You may not agree with them, remember? Nobody cares, remember?_

In this run, "B" completes all three lines of code. The function returns and hands back scontrol to the Go scheduler. This now decides to _resume execution_ of goroutine "A".

Because execution in any goroutine is sequential, the function resumes from where it left off. It had printed First anmd Second. So now it will resume to print Third.

> This is hugely important! Execution is sequential and resumes where it left off (within a goroutine)

### Conundrum: One function, many goroutines

Here is one function being run inside multiple goroutines. Pay attention to the local variable 'i', as we are going to think more deeply about what it does:

```go
package main

import (
	"fmt"
	"time"
)

func count(name string) {
	for i := 1; i <= 10; i++ {
		fmt.Println(name, i)
	}
}

func main() {
	go count("B")
	go count("C")
	go count("A")

	time.Sleep(time.Second)
}
```

Run this code [here](https://goplay.tools/snippet/G5yylNL6oNC)

During one run, the following output happened:

```
B 1
B 2
B 3
B 4
B 5
B 6
B 7
B 8
B 9
B 10
** A 1 **
** C 1 **
** A 2 **
A 3
A 4
A 5
A 6
A 7
A 8
A 9
A 10
** C 2 **
C 3
C 4
C 5
C 6
C 7
C 8
C 9
C 10
```

You'll notice that the same counting loop code is run inside three different goroutines.

- The value of `i` is being reused by each one of the goroutines
- Each goroutine may be interrupted by the Go Scheduler at any time
- The value of `i` is not corrupted when goroutine C interruts goroutine A

- :question: How does that work? Why is `i` not corrupted?

### The scheduler takes snapshots of local variables and current instruction

Goroutines can be paused and resumed by the Go Scheduler. What the above code shows is that a _snapshot_ of local variables and the currently executing program instructions are preserved for every goroutine. When the Go Scheduler resumes execution, it restores that snashot.

> There is not one single copy of local variable `i` in this code: _there is one per goroutine_

This messes with our mental model of how code executes when there is no concurrency. But it makes using concurrency easier, once you get used to it.

## Hate your life: recursive concurrent functions

We've had too much fun already. Let's slowly regret our life choices and look into recursive functions, running concurrently.

It goes without saying, these are hard to understand.

Let's start off with a recursive function `sigma(n)` that adds up all the numbers (positive non-zero integers) below it. So `sigma(3)` will work out 3 + 2 + 1, which is 6. `sigma(10)` works out to be 55.

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
		return 1 // recursive exit
	}

	res := number + sigma(number-1) // recursive call
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

We can see that the three function calls are running concurrently. Each recursive function gets allocated a slice of time to run, then the Go scheduler blocks it and unblocks another one. In this example, we see that sigma(3) processing has been interrupted by sigma(6) processing.

Notice that the local variables are preserved. Each goroutine keeps track of the local varibale values in functions.

This is confusing at first. One function - many goroutines - many different values for your local variables in that one function!

It is best to think of the Go Scheduler as having made a "copy" of our function inside each goroutine.

> There is not one single copy of local variable `i` in this code: _there is one per recursive call per goroutine_

It's actually just a [stack](https://github.com/bjssacademy/go-stacks-queues-sort-filter); each time a goroutine is unblocked, the relevant gorutine information stack is consulted for where the code left off last. This includes execution point and all local variables, in all of the call stack.

> One function - called from multiple goroutines - Go scheduler tracks and switches out local variable values for us

Have a nice, hot cup of tea! This is challenging to think about.

> Concurrency is hard. Recursion just makes it harder.

## Embarrasingly parallel - multiple unconnected goroutines

The above exaples have all been [embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel) problems.

These are where the concurrent pieces are completely independent. They do not need to work together. No need to wait for one another, no need to share results with one another.

It's like having three coffee shop baristas making three separate coffees, with a separate coffee maker each. Or perhaps Dave does the weekly shop while Sue watches TV - two activities that can be done at the same time, without being linked in any way.

To move on to our next major topic, we need to consider problems that are not so easy to parallelise.

## Channels: Communicating data between goroutines

For problems that are not _embarrasingly parallel_, by definition, we need to pass synchronise activity goroutines. We may need to synchronise execution between goroutines. Or we may need to share data. A consumer of some data will need to wait for a producer to supply it. Until then, the consumer is blocked and can do no useful work.

Channels are the preferred way for a Go program to communicate between different goroutines.

## Simple channel example

Let's start easy:

```golang
package main

import (
	"fmt"
	"time"
)

func produce(ch chan string) {
	ch <- "Hello"
}

func consume(ch chan string) {
	message := <-ch
	fmt.Println(message)
}

func main() {
	ch := make(chan string, 1)
	go consume(ch)
	go produce(ch)

	time.Sleep(time.Second)
}
```

Run this code [here](https://goplay.tools/snippet/Xo7aUYGfjq_9)

You should see the rather underwhelming output of

```
OUTPUT
Hello
```

What is perhaps more useful is _how_ this happened:

- `main()` was running inside goroutine 1, as always
- two new goroutines were created
- function `produce` was scheduled to run inside goroutine 3
- function `consume` was scheduled to run inside goroutine 2
- The two new goroutines were connected by a _channel_, variable `ch`
- That channel could hold 1 value, of type string
- `produce` wrote the string `"Hello"` into the channel
- `consume` read the string from the channel and printed it out

This code demonstrates how convenient channels are to pass data from one goroutine to another.

> Channels are guaranteed to be a safe way of passing data bewteen goroutines. The data will not be corrupted during transit.

Seems almost too easy, right?

## Channel conundrum: deadlock

Here's an easy mistake to make with channels:

```go
package main

func main() {
	ch := make(chan string)

	ch <- "Hello world"
}
```

> make(chan string) creates a channel, that can transport string values between goroutines. This syntax creates a maximum capacity of zero values in the channel. That means that the only time a value can be placed into this channel is when another goroutine is waiting and ready to receive it.

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

## Blocking and channels

To understand what happens, we need to understand the concept of _blocking_.

> _Blocking_: Where a goroutine has to wait for something to happen in a _different goroutine_. It is blocked. The Go scheduler hands control to something that is not blocked.

### Channel blocking basics

Here's code to create an unbuffered channel, write to it in one goroutine and read from it in the main goroutine 1. A 3 second delay happens before the write, so we can watch the action:

```go
package main

import (
	"fmt"
	"time"
)

func waitThenSend(ch chan string) {
	fmt.Println("Started: goroutine 2")

	// Wait for 3 seconds
	time.Sleep(time.Second * 3)

	// Send the cheery message. This will block if no reader is waiting
	ch <- "Thank you for waiting! Sorry to block you so long"

	// When goroutine2 becomes unblocked ...
	fmt.Println("Resumed: goroutine 2 after the write succeeded")
}

func main() {
	fmt.Println("Started: goroutine 1")

	unbufferedChannel := make(chan string)

	fmt.Println("Requesting new goroutine 2")
	go waitThenSend(unbufferedChannel)

	fmt.Println("<- operator is *blocked* until data written to channel...")
	message := <-unbufferedChannel

	fmt.Println(message)

	fmt.Println("Program ends")

	time.Sleep(time.Second)
}
```

Run this code [here](https://goplay.tools/snippet/RG4tg1fPtCW)

Initially, we see this output:

```
Started: goroutine 1
Requesting new goroutine 2
<- operator is *blocked* until data written to channel...
Started: goroutine 2
```

After a 3 second pause, we see this output:

```
Started: goroutine 1
Requesting new goroutine 2
<- operator is *blocked* until data written to channel...
Started: goroutine 2
Resumed: goroutine 2 after the write succeeded
Thank you for waiting! Sorry to block you so long
Program ends
```

This is normal operation. But it's quite a good conundrum when we're learning:

- :question: We saw "Started: goroutine 1" when we ran this, but no "Program ends". Why?
- :question: We saw the line "<- operator is blocked" _before_ "Started: goroutine 2". Why? What does this mean about when finctions are run inside goroutines?
- :question: The `main()` function was made to wait for 3 seconds, even though it had no time.Sleep() in it. What line caused main() to be paused? Why?

The answers reveal much about goroutines.

The key to it all is that the `main()` function requests a new goroutine is created, and then attempts to read a message from the channel. But there isn't any message there yet.

What can we do if we want to read a value that isn;t there? Logically, we can either fail with a panic, or simply wait until a value is available. Wait is exactly what Go does.

One option would be to wait using what's known as a _spin lock_. We could effectively have a loop inside main like this pseudo-code:

```
while (no message available) {
    check for message available
}
```

That would work, but it means the CPU is tied up doing nothing at all.

Go does not do this. It's wasteful. Instead, the Go Scheduler _blocks_ that main goroutine, as it cannot proceed yet. It hands control over to any other available goroutines that can run.

in this example, we requested a goroutine to be created, and told it to run the function `waitThenSend` in that goroutine. The Go Scheduler picks up on this, and starts running that function in that goroutine.

That explains the order of the output "Starting: goroutine 2".

The function code then writes the message to the unbuffered channel.

As the channel has a capacity of zero, the Go Scheduler _must_ block the writing goroutine. It must find another goroutine that is actively waiting to read from this channel. Of course, it does - the main goroutine 1 already has a blocked read operation.

The Go Scheduler resumes execution of goroutine 1. The read operation succeeds and pulls the message that was written. It then prints out the message.

Meanwhile, the read has unblocked the write in goroutine 2. You can see that some time later, goroutine 2 resumes running code.

The final `time.Sleep()` in `main()` keeps goroutine 1 open long enough that the Go Scheduler has time to resume execution of goroutine 2.

- :question: If you delete the time.Sleep(time.Second) (line 36) from `main()` - what changes?
- :quesion: Why did that change happen?

### A further channel blocking experiment

```go
package main

import (
	"fmt"
)

func produce(ch chan string) {
	ch <- "First"
	ch <- "Second"
	ch <- "Third"
	ch <- "Fourth"
	ch <- "Fifth"
	ch <- "STOP"
}

func consume(ch chan string) {
	for {
		message := <-ch

		if message == "STOP" {
			return
		}

		fmt.Println(message)
	}
}

func main() {
	ch := make(chan string, 3)
	go produce(ch)
	consume(ch)

	fmt.Println("Finished")
}
```

Run this code [here](https://goplay.tools/snippet/HydDHyeaA8e)

- :question: How many goroutines are in this code?
- :question: Which goroutine does `consume` run inside?
- :question: How many string values can channel `ch` store?
- :question: Why do we not see `STOP` in the output?
- :questions: Why do we use STOP in this example?
- :questions: Why do we not need `time.Sleep(time.Second)` in this code?

There are a few things to unpack in this code.

The first thing is that we are running `consume()` in goroutine 1, the same goroutine that runs `main()`. The `consume()` code is an infinite loop that we exit on receipt of the message `STOP`.

The code is otherwise similar to the earlier example. Function`produce()` writes string values to the channel passed in as a parameter. This channel is shared with function 'consume()' which also accepts the channel as a parameter. `consume()` will read values off the channel one at a time and print them out - unless they are the _sentinel value_ of `STOP`. This value causes `consume()` to return. That returns control to `main()`, which prints `Finished` and exits. That ends the program.

### Blocking and the Go Scheduler on channel full

One thing we cannot see in this code is very important: how the scheduler divides up work.

- `produce()` will run at some point, possibly in short time-slices
- `produce()` will write values onto the channel
- Values are written in sequence First, Second, Third and so on
- Once the `main()` function reaches the call to `consume()` it will immediatley call that function (there is no new goroutine created; no `go` keyword was used)
- The for loop in consume will read messages off the queue, one at a time
- The Go scheduler _may_ decide to use a short time-slice for consume, giving it time to read only one or two values. Or it may give a long slice

What happens next is interesting:

- `produce()` attempts to write values to the channel, while `consume()` _may_ be reading them (we cannot know the exact order)
- At some point, the channel will be _full_: it will contain three values awaiting reading
- If `produce()` attempts to write to the full channel, it will _block_

Once a channel is full, no more writes can happen. Instead, the gorutine repsonsible for running that code is blocked. This returns control to the Go Scheduler, which must decide what to do:

- If other goroutines are paused but able to run, it will select one at random and resume its execution
- If no goroutine is able to run, we get a panic: _deadlock_

If `produce()` is blocked, waiting for space in the channel, the Go Scheduler will resume execution of `consume()` in this code example. This will cause reads of the values on the channel. This in turn creates space for more values on the channel.

The Go Scheduler will then be able to resume execution of `produce()`. This will resume execution _from the point of the write to the channel which was blocked_. The write will succeed this time.

We can see that this behaviour maximises throughput in the CPU core. If some code needs to wait - like `produce()` waiting for space in the channel - then other code can be executed. The whole program is not waiting around; something can always be executing. The job of the Go Scheduler is to decide which code should be running.

#### Blocking and buffered channels

A buffered channel has capacity for one or more elements, held in a [FIFO Queue](https://github.com/bjssacademy/go-stacks-queues-sort-filter)

- A buffered channel can be written to without blocking when it has space
- A write to a buffered channel will block if the channel is full
- A read from a buffered channel will block if the channel is empty

#### Blocking and unbuffered channels

An unbuffered channel is a channel that has a capacity of zero things.

- An unbuffered channel can only be written to if another goroutine is ready to read from it
- A write to a buffered channel will always block until a read occurs
- A read from a buffered channel will always block until a write occurs

This forms a strong synchronisation point between the read and write lines of code in two different goroutines.

In other lanaguages, such as Ada, this is called a _rendezvous_, as both reader and writer must expressly operate at the same time.

## Fan out: worker pools

TODO

## Fan in: Reducers

TODO

# Further Reading

[Further reading >>](/further-reading.md)
