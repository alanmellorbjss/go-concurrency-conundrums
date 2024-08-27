# Rules of Goroutines and the Go Scheduler

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

## Goroutines and Communicating Sequential Processes

Let's start building with concurrency.
[Goroutines and Communicating Sequential Processes](/goroutines-csp.md)
