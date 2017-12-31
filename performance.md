This document outlines best practices for writing high-performance Go code.

At the moment, it's a collection of links to videos, slides, and blog posts
("awesome-go-performance"), but I would like this to evolve into a longer book
format where the content is here instead of external.  The links should be sorted into categories.

All the content will be licensed under CC-BY-SA.

## Optimization Workflow

* All optimizations should follow these steps:

    1. determine your performance goals and confirm you are not meeting them
    1. profile to identify the areas to improve.  This can be CPU, heap allocations, or goroutine blocking.
    1. benchmark to determine the speed up your solution will provide using
       the built-in benchmarking framework (<http://golang.org/pkg/testing/>)
       Make sure you're benchmarking the right thing on your target operating system and architecture.
    1. profile again afterwards to verify the issue is gone
    1. use <https://godoc.org/golang.org/x/perf/benchstat> or
       <https://github.com/codahale/tinystat> to verify that a set of timings
       are 'sufficiently' different for an optimization to be worth the
       added code complexity.
    1. use <https://github.com/tsenart/vegeta> for load testing http services
    1. make sure your latency numbers make sense: <https://youtu.be/lJ8ydIuPFeU>

The first step is important. It tells you when and where to start optimizing.
More importantly, it also tells you when to stop.  Pretty much all
optimizations add code complexity in exchange for speed.  And you can *always*
make code faster.  It's a balancing act.

The basic rules of the game are:

1. minimize CPU usage
 * do less work
 * this generally means "a faster algorithm"
 * but CPU caches and the hidden constants in O() can play tricks on you
1. minimize allocations (which leads to less CPU stolen by the GC)
1. make your data quick to access

This book is split into different sections:
   1) basic tips for writing not-slow software
     * CS 101-level stuff
   2) tips for writing fast software
     * Go-specific sections on how to get the best from Go
   3) advanced tips for writing *really* fast software
     * For when your optimized code isn't fast enough

### When and Where to Optimize

I'm putting this first because it's really the most important step. Should
you even be doing this at all?

Every optimization has a cost. Generally this cost is expressed in terms of
code complexity or cognitive load -- optimized code is rarely simpler than
the unoptimized version.

But there's another side that I'll call the economics of optimization. As a
programmer, your time is valuable. There's the opportunity cost of what else
you could be working on for your project, which bugs to fix, which features
to add. Optimizing things is fun, but it's not always the right task to
choose. Performance is a feature, but so is shipping, and so is correctness.

Choosing the most important thing to work on. Sometimes this isn't an
optimization at all. Sometimes it's not an actual CPU optimization, but a
user-experience one. Making something start up faster by doing computation in
the background after drawing the main window, for example.

Think of this as optimizing *your* time.

Choosing what to optimize.  Choosing when to optimize.

Fast software or fast deployment.

http://bitfunnel.org/strangeloop . has numbers. Hypothetical search engine
needing 30k machines @ $1k USD / year. Doubling the speed of your software
can save $15M/year. Even a developer spending an entire year to shave off 1%
will pay for itself

Once you've decided you're going to do this, keep reading.

### How to Optimize

Optimization is refactoring, but with an extra constraint on performance. You
need to check after each step that you haven't broken either of these.

Make sure the benchmarks you're using to check (CPU, memory, reqs/second) are
consistent and reproducible. If the individual runs have too high variance,
it makes improvements more difficult to spot.  Then you *really* need benchstat
or equivalent statistical tests and can't just eyeball it.

Decide what it is you're optimizing for. Are you trying to reduce memory
usage? By how much? How much slower is acceptable for what change in memory
usage?

Just because something is easy to optimize doesn't mean it's worth optimizing.
Ignoring low-hanging fruit is a valid development strategy.

Anything that can be measured can be optimized. Make sure you're measuring
the right thing. Beware bad metrics. There are generally competing factors.

This book is mostly going to talk about reducing CPU uage, reducing memory
usage, and reducing latency. Although remember that you can very rarely do
all three. Maybe CPU time is faster, but now it's using more memory. Maybe
you need to reduce memory space, but now the program takes longer.

Amdals Law: focus on the bottlenecks. If double the speed of routine that
only takes 5% of the runtime, that's only a 2.5% speedup in total runtime. A
good profile will help identify where time is actually spent.

In general, optimizations should procede from top to bottom. Optimizations
at the system level will have more impact than expression-level ones.

Do we have to do this at all?  The fastest code is the code that's not there.
If yes, is this the best algorithm.
If yes, is this the best *implementation* of this algorithm.

Basic techniques:

    http://www.crowl.org/lawrence/programming/Bentley82.html

    Approached program optimization as an engineering problem. Many of the
    tips from Bentley are now done automatically by compilers (for example,
    all the "loop" and "expression" ones). It's the programmers job to use
    transformations that compilers can't do.

Trade space for time:
  - smaller data structures: pack things, compress data structures in memory
  - precompute things you need (size of a linked list)
    http://www.smallmemory.com/

Most of the time if you're optimizing for CPU, your program will use more
memory. This is the classic space-time trade-off:
https://en.wikipedia.org/wiki/Space%E2%80%93time_tradeoff

Note that modern computers and the memory hierarchy make this trade-off less
clear. It's very easy for lookup tables to be "far away" in memory (and
therefore expensive to access) making it faster to just recompute every time
it's needed. This also means that benchmarking will frequently show
improvements that are not realized in the production system due to cache
contention (e.g., lookup tables are in the processor cache during
benchmarking but always flushed by "real data" when used in a real system.
See the graphs 4 and 5 in the Jump Hash paper:  https://arxiv.org/pdf/1406.2294.pdf )

Further, while data compression increases CPU time, if there are data
transfers involved (disk or network), the CPU time spent decompressing will
be trivial compared to the saved transfer time which will be orders of
magnitude slower.

algorithmic tuning:
  keep the old implementation around for testing

program tuning:
   best done in tiny steps, a few statements at a time
   moving from floating point math to integer math
   or mandelbrot removing sqrt, or lttb removing abs

some tunings are working around runtime or compiler code generation issue:
  always flag these with the appropriate issue so you can revisit
  assembly math.Abs() vs code generation vs function call overhead
  exploit a mathematical identity: https://go-review.googlesource.com/c/go/+/85477

Keep comments. If something doesn't need to be done, explain why.  Frequently
when optimizing an algorithm you'll discover steps that don't need to be
performed under some circumstances.  Document them. Somebody else might think
it's a bug and needs to be put back.

Empty program gives the wrong answer in no time at all. It's easy to be fast
if you don't have to be correct. But it means you can use an optimization
some of the time if you're sure it's in range.

Beware high constants Look for simpler algorithms with small constants.
Debugging an optimized algorithm is harder than debugging a simple one. Look
for algorithm the paper you're implementing claims to best and do that one
instead.

Choose algorithms based on problem size: (stdlib quicksort)
Detect and specialize for common or easy cases: stdlib string

Cache common cases: Your cache doesn't even need to be huge.
  Optimized a log processing script to cache the previous time passed to time.parse() for significant speedup
  But beware cache invalidatation, thread issues, etc

## Basics

1. choose the best algorithm
 * traditional computer science analysis
 * O(n^2) vs O(n log n) vs O(log n) vs O(1)
 * this should handle the majority of your optimization cases
 * be aware of http://accidentallyquadratic.tumblr.com/
 * https://agtb.wordpress.com/2010/12/23/progress-in-algorithms-beats-moore%E2%80%99s-law/
1. pre-compute things you need
1. add a cache -> reduces work

## Introductory Profiling

Techniques applicable to source code in general

1. introduction to pprof
 * go tool pprof (and <https://github.com/google/pprof>)
1. Writing and running (micro)benchmarks
 * -cpuprofile / -memprofile / -benchmem
1. How to read it pprof output
1. What are the different pieces of the runtime that show up
1. Macro-benchmarks (Profiling in production)
 * net/http/pprof

## Tracer


## Advanced Techniques

* Techniques specific to the architecture running the code
 * introduction to CPU caches
   * building intuition around cache-lines: sizes, padding, alignment
   * false-sharing
   * OS tools to view cache-misses
 * (also branch prediction)

* Comment about Jeff Dean's 2002 numbers (plus updates)
  * cpus have gotten faster, but memory hasn't kept up

## Heap Allocations
* Stack vs. heap allocations
* What causes heap allocations?
* Understanding escape analysis

## Runtime
* cost of calls via interfaces (indirect calls on the CPU level)
* runtime.convT2E / runtime.convT2I
* type assertions vs. type switches
* defer
* special-case map implementations for ints, strings

## Common gotchas with the standard library

* time.After() leaks until it fires
* Reusing HTTP connections...
* ....

## Unsafe
* And all the dangers that go with it
* Common uses for unsafe
* mmap'ing data files
* speedy de-serialization

## cgo
* Performance characteristics of cgo calls
* Tricks to reduce the costs
* Passing pointers between Go and C

## Assembly
* Stuff about writing assembly code for Go
* brief intro to syntax
* calling convention
* using opcodes unsupported by the asm
* notes about why intrinsics are hard

## Alternate implementations
* Popular replacements for standard library packages:
  * encoding/json -> ffjson
  * net/http -> fasthttp
  * regexp -> ragel (or other regular expression package)
  * serialization
      * encoding/gob -> <https://github.com/alecthomas/go_serialization_benchmarks>
      * protobuf -> <https://github.com/gogo/protobuf>
      * all formats have trade-offs; choose one that matches what you need

## Tooling

Look at some more interesting/advanced tooling

* perf  (perf2pprof)
