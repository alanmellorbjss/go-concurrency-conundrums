# Go Concurrency Conundrums

Goroutines and channels are the flagship features of the Go language. Providing an alternative to threads, these features simplify concurrent programming while making best use of available CPU cores.

_Simplified_ is not quite the same as _simple_, however.

The following exercsies are a mix of ideas that work, and _conundrums_ - puzzles that don't work, for you to figure out why.

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

We can make this clear by causing a `panic`:

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

We see there that the `main()` function runs inside goroutine 1. This has been created for us by the Go runtime environment.

> All go programs have at least _one_ goroutine

The `panic()` function is intended to terminate a goroutine with immediate effect. Like a 'controlled program crash'.

We use `panic()` rarely, when our code detects that contnuing execution would do more harm than good. An example might be to `panic` ones storage is full on the computer. There is no point generating more data we cannot store.

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
