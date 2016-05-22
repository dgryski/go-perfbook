
This document outlines best practices for writing high-performance Go code.

At the moment, it's a collection of links to videos, slides, and blog posts
("awesome-go-performance"), but I would like this to evolve into a longer book
format where the content is here instead of external.  The links should be sorted into categories.

* All optimizations should follow these steps:
	0) determine your performance goals and confirm you are not meeting them
	1) profile to identify the areas to improve.  This can be CPU, heap allocations, or goroutine blocking.
	2) benchmark to determine the speed up your solution will provide using
	   the built-in benchmarking framework (http://golang.org/pkg/testing/ and benchcmp).
	3) profile again afterwards to verify the issue is gone
	4) use https://godoc.org/rsc.io/benchstat or
	   https://github.com/codahale/tinystat to verify if a set of timings
	   are 'sufficiently' different for an optimization to be worth the
	   added code complexity.
	5) use https://github.com/tsenart/vegeta for load testing http services
	6) make sure your latency numbers make sense: https://youtu.be/lJ8ydIuPFeU

Step 0 is important.  It tells you when and where to start optimizing.  More
importantly, it also tells you when to stop.

The basic rules of the game are:

    1) minimize CPU usage
        - do less work
        - this generally means "a faster algorithm"
        - but CPU caches and the hidden constants in O() can play tricks on you
    2) minimize allocations (which leads to less CPU stolen by the GC)
    3) make your data quick to access

Introductory Profiling:
    Techniques applicable to source code in general

    introduction to pprof
        -cpuprofile
        net/http/pprof
        go tool pprof (and github.com/google/pprof)
    How to read it
    What are the different pieces of the runtime that show up

Advanced Techniques:

    Techniques specific to the architecture running the code
        introduction to CPU caches
        (also branch prediction)

    Comment about Jeff Dean's 2002 numbers (plus updates)
        cpus have gotten faster, but memory hasn't kept up

Runtime:
    cost of calls via interfaces (indirect calls on the CPU level)
    runtime.convT2E / runtime.convT2I
    type assertions vs. type switches
    defer

Common gotchas with the standard library:
    time.After() leaks until it fires
    Reusing HTTP connections...
    ....

Unsafe:
    And all the dangers that go with it
    Common uses for unsafe
    mmap'ing data files
    serialization

Assembly:

Popular replacements for standard library packages:
    encoding/json -> ffjson
    net/http -> fasthttp
    regexp -> ragel (or other regular expression package)
    encoding/gob -> https://github.com/alecthomas/go_serialization_benchmarks
	serialization is all about tradeoffs
   protobuf -> gogo/protobuf

Tooling:
    Look at some more interesting/advanced tooling
        perf  (perf2pprof)
        go-torch (+flamegraphs)
