# Channels: Communicating data between goroutines

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

# Fan in, fan out

[Fan in, fan out >>](/fan-in-fan-out.md)
