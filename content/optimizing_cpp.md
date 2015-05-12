## Menu
- 1. Introduction
- 2. Choosing the optimal platform
- 3. Finding the biggest time consumers
- 4. Performance and usability
- 5. Choosing the optimal algorithm
- 6. Developing process
- 7. The efficiency of different C++ constructs
- 8. Optimizations in the compiler
- 9. Optimizing memory access
- 10. Multithreading
- 11. Out of order execution
- 12. Using vector operations
- 13. Making critical code in multiple versions for different instruction sets
- 14. Specific optimization topics
- 15. Metaprogramming
- 16. Testing speed
- 17. Optimization in embedded systems
- 18. Overview of compiler options
- 19. Literature

## 1. Introduction
This manual is for advanced programmers and software developers who want to make their software faster. 
It is assumed that the reader has a good knowledge of the C++ programming language and a basic understanding 
of how compiler work. The C++ language is chosen as the basis for this manual for reasons explained below.


This is the first in a series of five manuals(available from [agner](http://www.agner.org/optimize/)):

- Optimizing software in C++: An optimization guide for Windows, Linux and Mac platforms.
- Optimizing subroutines in assembly language: An optimization guide for x86 platforms.
- The microarchitecture of Intel, AMD and VIA CPUs: An optimization guide for assembly programmers and compiler makers.
- Instruction tables: Lists of instruction latencies, throughputs and micro-operation breakdowns for Intel, AMD and VIA CPUs.
- Calling conventions for different C++ compilers and operating systems.
 
It is discussed how to identify and isolate the most critical part of a program and concentrate the optimization effort on that particular part. It is discussed how to overcome the dangers of a relatively primitive programming style that doesn't automatically check for array bounds violations, invalid pointers, etc. And it is discussed which of the advanced programming constructs are costly and which are cheap, in relation to execution time.

##2. Choosing the optimal platform
### Choice of hardware platform
Today, the choice of hardware platform for a given task is often determined by considerations such as price, compability, second source, and the availability of good development tools, rather than by the processing power.