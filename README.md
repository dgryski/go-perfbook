# go-perfbook

[![Buy Me A Coffee](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://www.buymeacoffee.com/dgryski)

This document outlines best practices for writing high-performance Go code.

The first sections cover writing optimized code in any language.
The later sections cover Go-specific techniques.

### Multiple Language Versions

* [English](performance.md)
* [中文](performance-zh.md)

### Table of Contents

1. [Writing and Optimizing Go code](performance.md#writing-and-optimizing-go-code)
1. [How to Optimize](performance.md#how-to-optimize)
   1. [Optimization Workflow](performance.md#optimization-workflow)
   1. [Concrete Optimization Tips](performance.md#concrete-optimization-tips)
1. [Data Changes](performance.md#data-changes)
1. [Algorithmic Changes](performance.md#algorithmic-changes)
1. [Benchmark Inputs](performance.md#benchmark-inputs)
1. [Program Tuning](performance.md#program-tuning)
1. [Optimization Workflow Summary](performance.md#optimization-workflow-summary)
1. [Tooling](performance.md#tooling)
   1. [Profiling](performance.md#introductory-profiling)
   1. [Tracer](performance.md#tracer)
1. [Garbage Collection](performance.md#garbage-collection)
1. [Runtime and Compiler](performance.md#runtime-and-compiler)
1. [Unsafe](performance.md#unsafe)
1. [Common gotchas with the standard library](performance.md#common-gotchas-with-the-standard-library)
1. [Alternate Implementations](performance.md#alternate-implementations)
1. [CGO](performance.md#cgo)
1. [Advanced Techniques](performance.md#advanced-techniques)
1. [Assembly](performance.md#assembly)
1. [Optimizing an Entire Service](performance.md#optimizing-an-entire-service)
1. Appendix
   1. [Implementing Research Papers](performance.md#appendix-implementing-research-papers)

### Contributing

This is a work-in-progress book in Go performance.

There are different ways to contribute:

   1) add to or summarizes the resources in TODO
   2) add bullet points or new topics to be covered
   3) write prose and flesh  out the sections in the book

Eventually sample programs to optimize and exercises will be needed (maybe).

Coordination will be done in the #performance channel on the Gophers slack.

