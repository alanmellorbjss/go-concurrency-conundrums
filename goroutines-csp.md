# Goroutines and Communicating Sequential Processes

How do we use goroutines in a Go program? What are the pitfalls to be aware of?

Let's start at the beginning with the traditional Hello World program:

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

[Channels and Blocking >>](/channels-blocking.md)
