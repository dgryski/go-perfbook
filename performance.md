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
       Make sure you're benchmarking the right thing.
    1. profile again afterwards to verify the issue is gone
    1. use <https://godoc.org/rsc.io/benchstat> or
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
* Using sync.Pool effectively

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
* brief into to syntax
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
* go-torch (+flamegraphs)
