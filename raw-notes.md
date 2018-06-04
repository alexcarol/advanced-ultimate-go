https://github.com/ardanlabs/gotraining/blob/master/topics/courses/go/README.md

We are going to learn how to read code rather than write it.

Mechanical sympathy: if perf matters, your code must be sympathetic with the machine.

Even though there's abstraction we

Performance will come from 4 places:
- Latency (I/O, networking, handle sync...)
- Garbage collection (how do you make sure the only garbage you produce is "the right garbage"?)
- How you access data
- Algorithm efficiency

Do not optimize for performance, optimize for readability (machines are very fast and they do a lot of optimizationsfor you)



Goroutines start at 2K (31/May/2018), this value might change



Values and pointer semantics
============================
Data: can be either value or pointer





When calling functions this changes the active frame (from whichever func we're executing to that func)
Value semantics: immutability and no side effects
Pointer semantics:
- we pass a copy of the address, pointers are for one thing: sharing if you don't need to share, you don't need a pointer
- with pointers the goroutine gets to manipulate data out of it's sandbox (that's how side-effects get created)


What is the cost of value vs pointer semantics? (every choice you make comes with a consequence)
Value semantics:
Pros: immutability (isolation), no side effects. We <3 value semantics.
Cons: inefficiency because you need to make copies to the data and you need to be able to pass the changes you make up the call ladder

Pointer semantics:
Pros: you make a change, everybody sees it
Cons: you have side effects, especially if you're writing multi-threaded software you have have to handle orchestration


You want maintainable go code? Focus on value and pointer semantics.




Mechanics vs semantics
Mechanics: how something works
Semantics: behaviour

**Values on the stack are not considered allocations**
Stacks are clean, they do not need to be garbage collected.
If you can do  everything with values, you need 0 allocation.



There are no constructors in go, we call them factory functions


"&" means "sharing"


Sharing a value up the callstack: 100% (actually, if the function is inlined this might not be true) it's on the heap (down the callstack not necessarily)
When a function in go returns a newly created pointer, go allocates memory in the heap (escape analysis determines where a value is allocated in memory).


Rules that  don't make Bill cry:
- If you're constructing a value to a variable: never use pointer semantics


**go build -gcflags "-m -m" => it generates escape analysis, that tells you why things are allocated** // investigar què passa quan passes punters i jugues amb goroutines

Why something might be allocated if something is shared in a way that it needs to go on the heap.

Slices go on the heap: If the compiler does not know the size of something at compile time then it has to allocate it to the heap (no way around it) 

You have to balance value and pointer semantics

What happens if you use up your whole stack?
go creates contiguous stacks: if the stack fills it copies the stack to a new stack with more memory available

If two or more stacks have to share a value they have to go on the heap.
In go stack values move, but heap values don't  (go doesn't have a *compacting* garbage collector).

Always go back to "optimizing for readability rather than performance"

Go cares about: what is the smaller heap size I have to maintain?
The collector is optimized for latency. It's allowed to use up to 25% of available CPU capacity.
To be able to "stop the world" all of the  Ps (processes) have to be taken to a safe point. The only way to take a goroutine to a safe point (up to the current go version) is to make function calls, if you don't do function calls in a long-running goroutine, you will stall garbage collector. They might fix this in go 1.11


The heap is nothing more than a graph of values connected through pointers (that starts on the stack => unless you use global variables (don't use global variables) ). 

The less objects we allocate the less the gc has to do.

Every 2-4 ms the gc does pass.




Go is lacking a lot of data structures


NUMA: Non-uniform memory access
Hardware tries to bring NUMA into L3 caches => means processors have accesses to a small part of main memory, if they want more they have to go through another process 
 
It's not a problem for most people, but if you allocate enough memory (like facebook) it becomes problematic (they only use single processor machines)
 


How to write code that creates predictable access patterns to memory?
Easiest way: arrays
maps are continously restructuring themselves for data to be contiguous


Go only gives you a slice because it pushes you in the direction of mechanical sympathy.

Don't use "object-oriented patterns" in go. Instead use data oriented design.
Philosophy: If you don't understand the data, you don't understand the problem

https://github.com/ardanlabs/gotraining/tree/master/topics/go#data-oriented-design

If your data is changing, your problem is changing.

BE PRECISE
APIs like:
func (u *User) SendEmail()
or 
func SendEmail(u *User)
Are very problematic, they tell nothing about what they use, and things can go wrong in prod easily

he says he has like 12 parameters sometimes

instead better use:
func SendEmail(name, email string)
 
It will break if you want to change the api, rather than requiring a field in user to be initialized. This will cause compiler errors rather than runtime errors.
 
 
 Hardware loves arrays
 
 
 https://github.com/ardanlabs/gotraining/blob/master/topics/go/language/arrays/README.md
 Videos to watch:
 https://www.youtube.com/watch?v=WDIkqP4JbkE => watch twice a year
 https://www.youtube.com/watch?v=rX0ItVEVjHc
 
 
 var strings [5]string

https://play.golang.org/p/wUzREuHhLY
for i, fruit := range strings { // value semantics => this version makes copies of everything
    fmt.Println(i, fruit)
}

for i := range strings { // pointer semantics (fruit is a pointer to the string memory position)
    fmt.Println(i, strings[i])
}


Range semantics:
https://play.golang.org/p/Hym5wBsEMO
TODO review this, range semantics (pointer vs value can be very tricky)

Never use pointers in general for parameters or fields (the only case is things like sql null)


Value semantic mutation: return the modified value instead of editing it (like the time package)
If you always use value semantics 
Exceptions: decode and unmarshal need pointer semantics

Value to pointer  => ok
Pointer to Value =>  no
You're never allowed to make a copy of a value a pointer points to.


Code readability reviews are important (google has code readability review, and after a technical review)

If you're not sure what to use, then use pointer semantics => then you have to be careful about mutation, side effects... 



setters and getters are not APIs
don't mix pointer and value receivers

Every interface causes an allocation => if you need high performance you should not use interfaces => it looks like this needs to be solved in the go escape analysis, but it's really hard, who knows if/when it'll be done





Concurrency and why you shouldn't use it
========================================
Concurrency is about managing a lot of things at once, parallelism is about doing a lot of things at once.


Threads states:
- Running
- Runnable
- Waiting (I/O)

Context switching is expensive: a lot of state to move around


2004: Multicore processors became mainstream

Multicore problems
  
Runqueues: any thread in runnable state goes to runqueue

When you have cpu-bound work more threads than cores are harmful
When you have I/O-bound work more threads go on waiting mode, so it's useful to have more threads


False sharing: two cores acess to contiguous data that is stored in the same cache row => perf disaster
Skylight => 24-core processor

Golang scheduler sits on top of the OS scheduler (so that it can leverage OS scheduler improvements).

Two Queues where runnable goroutines can go:
Global Run Queue
Local Run Queue 

Gs: goroutines
If you do cpu-bound work, more Gs than Ps are not useful => though the go scheduler can help you
If you do  I/O-bound work then more  Gs than Ps are useful

Synchronization: standing in line => cost of latency
Orchestration: example => channels // TODO look up meaning
go scheduler: preemptive scheduler // TODO look up meaning
 
 
Scheduling decisions are made when:

- Creating a goroutine
- Garbage collection
- Syscalls
- Block ???


For most systems you can make asynchronous syscalls => then no problem
If you can't make async syscalls: new thread needs to be created (confirm this, not sure)


If two OS threads need to exchange messages, there's a lot of latency and one of them needs to be stopped.
If two goroutines  need to exchange messages, latency is low. =>  turns I/O bound work into CPU-bound work 



Patterns:
Goroutines are for one thing: "signaling" => a goroutine signals an event to another goroutine

Unbuffered channels: guaranteed delivery
    con: unknown latency

Buffered channels: not guaranteed delivery


You don't need to close a channel for it to be gc'd, it's purely a state change
https://github.com/ardanlabs/gotraining/blob/master/topics/go/concurrency/channels/README.md

patterns:
- waitForTask()
- waitForResult()
- waitForFinished() => the example should really be implemented with waitGroup
- pooling() => hesitate to create pools of goroutines unless you need to limit concurrent access to something (but most of the time you shouldn't need to because the scheduler is pretty good about it) => avoid using buffered channels, as we won't have guarantee that someone is working on it
- fanOut()
- fanOutSem() => if one of the fanned out goroutines failed we don't want the others to continue, so if one fails we want to prevent any others from starting
- drop()
- cancellation() 
see examples in: https://github.com/ardanlabs/gotraining/blob/master/topics/go/concurrency/channels/example1/example1.go   


You can't use print statements to orchestrate code, orchestration is only in the channel, print order is arbitrary.



Problem example:
All our code does logging. What if the device stops accepting data? The program blocks





Construction of structs: just construct them all at once, don't set some variables first and some later on



Tracing
=======
https://github.com/ardanlabs/gotraining/blob/master/topics/go/profiling/README.md


go tool pprof p.out

web list find => opens the callgraph
pprof is not very useful with the syscalls

use trace instead:
go tool trace t.out => opens the browser


be wary of accessing global variables too many times in a goroutine, as multiple goroutines writing on a the same will make performance very poor  

look out for what's coming relating to tracing in go 1.11


GODEBUG=gctrace=1 ./myprogram >/dev/null


interesting tools:
https://github.com/rakyll/hey => program to test load on web apps

https://ngrok.com/




Crud based service =>  this is the production code that they use at ardanlabs
https://github.com/ardanlabs/service


net/http/pprof => every project should import it and make sure to run the default http server, but make sure to put it behind a **firewall**
=> it has 0 cost

pprof
top 40 -cum =>  top 40 functions sorted by cumulative number


Example project for profiling
https://github.com/ardanlabs/gotraining/tree/master/topics/go/profiling/project



https://github.com/ardanlabs/gotraining/tree/c469e1fdcc7dad2e77c611e5c53ae51b74a6c433/topics/go/profiling/memcpu
go tool pprof -alloc-space p.out
list algOne => algOne is the func name)

profiler doesn't take into account inline optimizations when showing allocations

go test -gcflags "-m -m" ...


we avoided using an interface


You MUST validate your results, if you don't, you'll be in trouble



go test -run none -bench NameBenchmark -benchtime 3s


Rule #1 when running a benchmark: the machine must be idle => run your benchmarks individually to validate your results


Do not guess about performance, check


gcore => do a dump of your go program =>  can be read by delve and allows you to debug

panic stack traces shows the variables passed to functions (as words in little endian), the more precise is your api the more information you have 