---
title: "Introduction to benchmarking and profiling"
date: 2020-05-15
tags: [benchmarking,profiling,introduction]
---

# Benchmarking and profiling

When it comes to writing code, performance is always a concern. To measure the
performance of code there are two major ways of measurement:

- Profiling: measuring the time spent on each code section/line
- Benchmarking: measuring the overall performance of code compared to a
  baseline

## Profiling

Profiling is helpful when you want to identify the slower parts of your code
and refactor them to make them faster. According to Amdahl's law, the most
amount of performance improvement can come from the slowest part of your code.

Profiling is also useful to show the nature of your application. For example,
if they are IO-bound, CPU, or memory bound, etc. This information is
important to understand which aspect of your app can be improved.

### Profiling an application that forks

If you are profiling a server that is designed to handle multiple clients at a
time, chances are the server makes forks of itself to handle the requests, a
profiler then needs to know how to attach itself to all those processes/threads
and merge the result to get a unified view of the whole application.

### Profiler overhead

Profilers need to do their best to avoid affecting the application execution
that is being profiled. One way is to delay analysis of the result until after the
application is finished, this usually means saving the raw metrics in a file,
but profilers need to be careful not to include the time required to write
that info to the disk in their time measurements.

### Sampling vs. non-sampling

Sampling profilers look at the call stack at a point of time, however,
non-sampling profilers use provided callbacks to mark the entry and exit points
of the code and calculate the time based on that.  Sampling profilers have less
performance overhead than non-sampling profilers and are more suitable if you
are measuring the performance of code, however, because they sample the call
stack with an interval, it's likely that they miss functions that take less
time (about 5ms or less) to execute, and therefore are less accurate.

### Statement vs. Subroutine

Statement profiling works by measuring the time spent between entering one
statement and entering the next statement. A subroutine profiler, however,
measures the time between entering a function execution and exiting the
function. Most of the time subroutine profiling should be preferred over
statement profiling as it leaves less profile footprint on the time
measurements. Statement profiling may be useful in case the subroutine
profiling result is not granular enough.

### Memory profiling

Memory profiling works by taking a snapshot of the memory allocated for the
running application. Those snapshots then can be used to detect unwanted growth
in the allocated memory and potential memory leaks.

## Benchmarking

Benchmarking can be done to compare the performance of two applications, maybe
two versions of the same application, (software benchmark), or performance of
the same application on two different machines (hardware benchmark).

### Hardware benchmark

There are standard benchmarks with a known result on different hardware that
can be used to measure the performance of your machine.

### Software benchmark

Examples are comparing the performance of different browsers rendering a page,
or different databases querying data.

## Micro-benchmarking

Micro-benchmarking is used when we want to compare a small piece of software
(a.k.a. a function) against a similar function or a different version of the
same function. Micro-benchmarking is mostly useful after you identified
functions that are suitable candidates for optimization and you're working on
optimizing them.

In that case, you might want to use one or more implementations of the same
algorithm to pick the most suitable solution.
