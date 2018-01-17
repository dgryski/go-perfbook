# Writing and Optimizing Go code

This document outlines best practices for writing high-performance Go code.

At the moment, it's a collection of links to videos, slides, and blog posts
("awesome-golang-performance"), but I would like this to evolve into a longer
book format where the content is here instead of external.  The links should be
sorted into categories.

While some discussions will be made for individual services faster (caching,
etc), designing performant distributed systems is beyond the scope of this
work.

All the content will be licensed under CC-BY-SA.

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

Some times this will be obvious: an hourly report that completes in three hours
is probably less useful that one that completes in less than one.

Just because something is easy to optimize doesn't mean it's worth
optimizing. Ignoring low-hanging fruit is a valid development strategy.

Think of this as optimizing *your* time.

Choosing what to optimize.  Choosing when to optimize.

Clarify "Premature optimization" quote.

TPOP: Should you optimize? "Yes, but only if the problem is important, the
program is genuinely too slow, and there is some expectation that it can be
made faster while maintaining correctness, robustness, and clarity."

Fast software or fast deployment.

http://bitfunnel.org/strangeloop . has numbers. Hypothetical search engine
needing 30k machines @ $1k USD / year. Doubling the speed of your software
can save $15M/year. Even a developer spending an entire year to shave off 1%
will pay for itself

Once you've decided you're going to do this, keep reading.

### How to Optimize

## Optimization Workflow

Before we get into the specifics, lets talk about the general process of
optimization.

Optimization is a form of refactoring. But each step, rather than improving
some aspect of the source code (code duplication, clarity, etc), improves
some aspect of the performance: lower CPU, memory usage, latency, etc. This
means that in addition to a comprehensive set of unit tests (to ensure your
changes haven't broken anything), you also need a good set of benchmarks to
ensure your changes are having the desired effect on performance. You must be
able to verify that your change really *is* lowering CPU. Sometimes a change
you thought would improve will actually turn out to have a zero or negative
change.  Always make sure you undo your fix in these cases.

The benchmarks you are using must be correct and provide reproducible numbers
on representative workloads. If individual runs have too high a variance, it
will make small improvements more difficult to spot. You will need to use
benchstat or equivalent statistical tests and won't be able just eyeball it.
(Note that using statistical tests is a good idea anyways.) The steps to run
the benchmarks should be documented, and any custom scripts and tooling
should be committed to the repository with instructions for how to run them.
Be mindful of large benchmark suites that take a long time to run: it will
make the development iterations slower.

(Note also that anything that can be measured can be optimized. Make sure
you're measuring the right thing.)

The next step is to decide what you are optimizing for. If the goal is to
improve CPU, what is an acceptable speed. Do you want to improve the current
performance by 2x? 10x? Can you state it as "problem of size N in less than
time T"? Are you trying to reduce memory usage? By how much? How much slower
is acceptable for what change in memory usage? What are you willing to give
up in exchange for lower space?

Optimizing for service latency is a trickier proposition. Entire books have
been written on how to performance test web servers. The primary issue is
that for single-threaded code, the performance is fairly consistent for a
given problem size. For webservices, you don't have a single number. A proper
web-service benchmark suite will provide a latency distribution for a given
reqs/second level. ...

The performance goals must be specific. You will (almost) always be able to
make something faster. Optimizing is frequently a game of diminishing
returns. You need to know when to stop.

The difference between what your target is and the current performance will
also give you an idea of where to start. If you need only a 10%-20%
performance improvement, you can probably get that with some implementation
tweaks and smaller fixes. If you need a factor of 10x or more, then just
replacing a multiplication with a left-shift isn't going to cut it. That's
probably going to call for changes up and down your stack.

Good performance work requires knowledge at many different levels, from
system design, networking, hardware (CPU, caches, storage), algorithms,
tuning, and debugging. With limited time and resources, consider which level
will give the most improvement: it won't always be algorithm or program
tuning.

In general, optimizations should proceed from top to bottom. Optimizations at
the system level will have more impact than expression-level ones. Make sure
you're solving the problem at the appropriate level.

This book is mostly going to talk about reducing CPU usage, reducing memory
usage, and reducing latency. It's good to point out that you can very rarely
do all three. Maybe CPU time is faster, but now your program uses more
memory. Maybe you need to reduce memory space, but now the program will take
longer.

Amdahl's Law tells us to focus on the bottlenecks. If you double the speed of
routine that only takes 5% of the runtime, that's only a 2.5% speedup in
total wall-clock. On the other hand, speeding up routine that takes 80% of
the time by only 10% will improve runtime by almost 8%. Profiles will help
identify where time is actually spent.

When optimizing, you want to reduce the amount of work the CPU has to do.

A profiler might show you that lots of time is spent in a particular routine.
It could be this is an expensive routine, or it could be a cheap routine that
is just called many many times. Rather than immediately trying to speed up
that one routine, see if you can reduce the number of times it's called or
eliminate it completely.

The Three Optimization Questions:

- Do we have to do this at all?  The fastest code is the code that's not there.
- If yes, is this the best algorithm.
- If yes, is this the best *implementation* of this algorithm.

### Concrete optimization tips

Jon Bentley's 1982 work "Writing Efficient Programs" approached program
optimization as an engineering problem: Benchmark. Analyze. Improve. Verify.
Iterate. A number of his tips are now done automatically by compilers. A
programmers job is to use the transformations compilers *can't* do.

There's a summary of this book:
http://www.crowl.org/lawrence/programming/Bentley82.html

When thinking changes you can make to your program, there are two basic options:
you can either change your data or you can change your code.

Changing your data means either adding to or altering the representation of
the data you're processing.

Ideas for augmenting your data structure:

- extra fields: For example, store the size of a linked lists rather than
iterating when asked for it. Or storing additional pointers to frequently
needed other nodes to multiple searches (for example, "backwards" links in a
doubly-linked list to make removal O(1) ). These sorts of changes are useful
when the data you need is cheap to store and keep up-to-date.

- extra search indexes: Most data structures are designed for a single type of query.
If you need two different query types, having an additional "view" onto your data can be large improvement.
For example, []struct, referenced by ID but sometimes string -> map[string]id (or \*struct)

- extra information about elements: for example, a bloom filter. These need to
be small and fast to not overwhelm the rest of the data structure.

- if queries are expensive, add a cache.  We're all familiar with memcache, but there are in-process caches.
* over the wire, the network + cost of serialization will hurt
* in-process caches, but now you need to worry about expiration
* even a single item can help (logfile time parse example)

TODO: "cache" might not even be key-value, just a pointer to where you were
working. This can be as simple as a "search finger"

These are all clear examples of "do less work" at the data structure level.
They all cost space. Most of the time if you're optimizing for CPU, your
program will use more memory. This is the classic space-time trade-off:
https://en.wikipedia.org/wiki/Space%E2%80%93time_tradeoff

If your program uses too much memory, it's also possible to go the other way.
Reduce space usage in exchange for increased computation. Rather than storing
things, calculate them every time. You can also compress the data in memory
and decompress it on the fly when you need it.

There's a book available on line covering techniques for reducing the space
used by your programs. While it was originally written targetting embedded
developers, the ideas are applicable for programs on modern hardware dealing
with huge amounts of data. http://www.smallmemory.com/

Rearrange your data: Eliminate padding. Remove extra fields.
Change to a slower data structure.
Skip pointer-heavy tree structure and use slice and linear search instead.
Custom compression format for your data: floating point (go-tsz), integers (delta, xor + huffman)

We will talk more about data layouts later.

Modern computers and the memory hierarchy make the space/time trade-off less
clear. It's very easy for lookup tables to be "far away" in memory (and
therefore expensive to access) making it faster to just recompute a value
every time it's needed.

This also means that benchmarking will frequently show improvements that are
not realized in the production system due to cache contention (e.g., lookup
tables are in the processor cache during benchmarking but always flushed by
"real data" when used in a real system. Google's Jump Hash paper in fact
addressed this directly, comparing performance on both a contented and
uncontended processor cache. See graphs 4 and 5 in the Jump Hash paper:
https://arxiv.org/pdf/1406.2294.pdf )

TODO: how to simulate a contented cache, show incremental cost

Another aspect to consider is data-transfer time. Generally network and disk
access is very slow, and so being able to load a compressed chunk will be
much faster than the extra CPU time required to decompress the data once it
has been fetched.  As always, benchmark.

If you're not changing the data, the other main option is to change the code.

Algorithmic Changes

Keep comments. If something doesn't need to be done, explain why.  Frequently
when optimizing an algorithm you'll discover steps that don't need to be
performed under some circumstances.  Document them. Somebody else might think
it's a bug and needs to be put back.

Empty program gives the wrong answer in no time at all. It's easy to be fast
if you don't have to be correct. But it means you can use an optimization
some of the time if you're sure it's in range.

Have an intuitive grasp of the different O() levels:
  - field access, array or map lookup, O(1)
  - simple loop, O(n)
  - nested loop, O(n*m)
  - binary-search O(log n)
  - divide-and-conquer O(n log n)
  - combinatoric - look out!!

Know how big each of these input sizes is likely to be when coding. You don't
always have to shave cycles, but also don't be dumb.

Tips for implementing papers:  (For `algorithm` read also `data structure`)
* Don't.  Start with the obvious solution and reasonable data structures.
* "Modern" algorithms tend to have lower theoretical complexities but high constants and lots of implementation complexity.
* Look for the paper their algorithm claims to beat and implement that.
* Make sure you understand the algorithm.  This sounds obvious, but it will be impossible to debug otherwise.
* The original paper for a data structure or algorithm isn't always the best.  Later papers may have better explanations.
* Make sure the assumptions the algorithm makes about your data hold. 
* Some papers release reference source code which you can compare against, but
   - 1) academic code is almost universally terrible
   - 2) beware licensing restrictions
   - 3) beware bugs
Also look out for other implementations on GitHub: they may have the same (or different!) bugs as yours.

Sometimes the best algorithm for a particular problem is not a single
algorithm, but a collection of algorithms specialized for slightly different
input classes. This "polyalgorithm" quickly detects what kind of input it
needs to deal with and then dispatches to the appropriate code path.

There are examples of this are in the standard library sorting and string
packages.

Choose algorithms based on problem size: (stdlib quicksort)
Detect and specialize for common or easy cases: stdlib string

Beware algorithms with high startup costs.  For example,
   search is O(log n), but you have to sort first.
   If you just have a single search to do, a linear scan will be faster.
   But if you're doing many sorts, the O(n log n) sort overhead will not matter as much

But you can also limit the search space by bucketing your data:
But if you just need to test membership, maybe you want a hash.
You can also bucket your data to reduce the size you need to scan.

Your benchmarks must use appropriately-sized inputs. As we've seen, different
algorithms make sense at different input sizes. If your expected input range
in <100, then your benchmarks should reflect that. Otherwise, choosing an
algorithm which is optimal for n=10^6 might not be the fastest.

Be able to generate representative test data. Different distributions of data
can provoke different behaviours in your algorithm: think of the classic
"quicksort is O(n^2) when the data is sorted" example. Similarly,
interpolation search is O(log log n) for uniform random data, but O(n) worst
case. Knowing what your inputs look like is the key to both representative
benchmarks and for choosing the best algorithm.

Cache common cases: Your cache doesn't even need to be huge.
  Optimized a log processing script to cache the previous time passed to time.parse() for significant speedup
  But beware cache invalidation, thread issues, etc
  Random cache eviction is fast and sufficiently effective.
     - only put "some" items in cache (probabilistically) to limit cache size to popular items with minimal logic
  Compare cost of cache logic to cost of refetching the data.

The standard library implementations need to be "fast enough" for most cases.
If you have higher performance needs you will probably need specialized
implementations.

This also means your benchmark data needs to be representative of the real
world. If repeated requests are sufficiently rare, it's more expensive to
keep them around than to recompute them. If your benchmark data consists of
only the same repeated request, your cache will give an inaccurate view of
the performance.

Program tuning used to be an art form, but then compilers got better. So now
it turns out that compilers can optimize straight-forward code better than
complicated code. The Go compiler still has a long way to go to match gcc and
clang, but it does mean that you need to be careful when tuning and
especially when upgrading that your code doesn't become "worse". There are
definitely cases where tweaks to work around the lack of a particular
compiler optimization became slower once the compiler was improved.

If you are working around a specific runtime or compiler code generation
issue, always document your change with a link to the upstream issue. This
will allow you to quickly revisit your optimization once the bug is fixed.

Fight the temptation to cargo cult folklore-based "performance tips".

Iterative program improvements:
  - ensure progress at each step
  - but frequently one improvement will enable others
  - which means you need to keep looking at the entire picture

program tuning:
   if possible, keep the old implementation around for testing
   if not possible, generate sufficient golden test cases to compare output
   exploit a mathematical identity: https://go-review.googlesource.com/c/go/+/85477
   just clearing the parts you used, rather than an entire array
   best done in tiny steps, a few statements at a time
   moving from floating point math to integer math
   or mandelbrot removing sqrt, or lttb removing abs
   cheap checks before more expensive checks:
    e.g., strcmp before regexp, (q.v., bloom filter before query)



Profile regularly to ensure the track the performance characteristics of your
system and be prepared to re-optimize as your traffic changes. Know the
limits of your system and have good metrics that allow you to predict when
you will hit those limits.

De-optimize when possible. I removed from mmap + reflect + unsafe when it
stopped being necessary.

## Optimization workflow summary

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

1. choose the best algorithm
 * traditional computer science analysis
 * O(n^2) vs O(n log n) vs O(log n) vs O(1)
 * this should handle the majority of your optimization cases
 * be aware of http://accidentallyquadratic.tumblr.com/
 * https://agtb.wordpress.com/2010/12/23/progress-in-algorithms-beats-moore%E2%80%99s-law/
1. add a cache -> reduces work
1. if you add a cache up front, then it becomes pre-compute things you need

# Tooling

## Introductory Profiling

Techniques applicable to source code in general

1. introduction to pprof
 * go tool pprof (and <https://github.com/google/pprof>)
1. Writing and running (micro)benchmarks
 * profile, extract hot code to benchmark, optimize benchmark, profile.
 * -cpuprofile / -memprofile / -benchmem
 * 0.5 ns/op means it was optimized away -> how to avoid
 * tips for writing good microbenchmarks (remove unnecessary work, but add baselines)
1. How to read it pprof output
1. What are the different pieces of the runtime that show up
1. Macro-benchmarks (Profiling in production)
 * net/http/pprof

## Tracer


## Advanced Techniques

* Techniques specific to the architecture running the code
 * introduction to CPU caches
   * performance cliffs
   * building intuition around cache-lines: sizes, padding, alignment
   * false-sharing
   * true sharing -> sharding
   * OS tools to view cache-misses
   * maps vs. slices
   * SOA vs AOS layouts
   * reducing pointer chasing
 * branch prediction
 * function call overhead

* Comment about Jeff Dean's 2002 numbers (plus updates)
  * cpus have gotten faster, but memory hasn't kept up

## Garbage Collection
* Stack vs. heap allocations
* What causes heap allocations?
* Understanding escape analysis (and the current limitation)
* API design to limit allocations: allow passing in buffers so caller can reuse rather than forcing an allocation
  - you can even modify a slice in place carefully while you scan over it
* reducing pointers to reduce gc scan times
* GOGC
* buffer reuse (sync.Pool vs or custom via go-slab, etc)

## Runtime
* cost of calls via interfaces (indirect calls on the CPU level)
* runtime.convT2E / runtime.convT2I
* type assertions vs. type switches
* defer
* special-case map implementations for ints, strings
* bounds check elimination

## Common gotchas with the standard library

* time.After() leaks until it fires
* Reusing HTTP connections...
* ....
* rand.Int() and friends are 1) mutex protected and 2) expensive to create
  - consider alternate random number generation

## Unsafe
* And all the dangers that go with it
* Common uses for unsafe
* mmap'ing data files
  - struct padding
* speedy de-serialization
* string <-> slice conversion, []byte <-> []uint32, ...

## cgo
* Performance characteristics of cgo calls
* Tricks to reduce the costs: batching
* Rules on passing pointers between Go and C
* syso files

## Assembly
* Stuff about writing assembly code for Go
* always have pure-Go version (noasm build tag): testing,
* brief intro to syntax
* calling convention
* using opcodes unsupported by the asm
* notes about why intrinsics are hard
* all the tooling to make this easier: asmfmt, peachpy, c2goasm, ...

## Alternate implementations
* Popular replacements for standard library packages:
  * encoding/json -> ffjson
  * net/http -> fasthttp (but incompatible API)
  * regexp -> ragel (or other regular expression package)
  * serialization
      * encoding/gob -> <https://github.com/alecthomas/go_serialization_benchmarks>
      * protobuf -> <https://github.com/gogo/protobuf>
      * all formats have trade-offs: choose one that matches what you need
        encoded space, decoding speed, language/tooling compatibility, ...
  * database/sql -> jackx/pgx, ...
  * gccgo

## Tooling

Look at some more interesting/advanced tooling

* perf  (perf2pprof)
