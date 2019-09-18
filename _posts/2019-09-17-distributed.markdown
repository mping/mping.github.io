---
layout: post
title: "On Distributed Stuff"
date: 2019-09-17T12:10:31+00:00
comments: true
---

--
Feel free to correct any inaccuracies
--

### sources

 * www.bailis.org/blog/linearizability-versus-serializability/
 * https://dddpaul.github.io/blog/2016/03/17/linearizability-and-serializability/
 * https://aphyr.com/posts/313-strong-consistency-models
 * http://gvsmirnov.ru/blog/tech/2014/02/10/jmm-under-the-hood.html
 * http://preshing.com/20120625/memory-ordering-at-compile-time/
 * https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html
 * https://www.logicbig.com/tutorials/core-java-tutorial/java-multi-threading/happens-before.html
 * internet


## intro

Today I want to share what I learned about distributed systems.

There are a couple of notions that are intrinsic to any kind of system where the parties have to agree (or not) on values, the so-called **consistency models**.

Why would I want to talk about this topic, if there are smarter people already explaining it? I feel there's a huge gap between people that are starting to research these concepts and people that already employ them everyday. Terms like consistency model, compare-and-swap, wait-free, processes, total ordering, histories, etc can be confusing to someone who's starting, so I'll try and fill the gap.

## Consistency Models

Consistency models define how memory operations apply within a distributed system. Right off the bat, let's dissect a bit:

* Distributed means there's more than one process (can be a thread, or a different host)
* On a distributed system, operations (read or write) may take some time (there's latency involved), and within that time the system is processing other reads and writes
* Consistency models can be enforced by your RDBMS (MySQL) but also by your CPU; it doesn't necessary apply just to multiple hosts
* If consistency models are enforced correctly (ie, if they do what they say the do), they help developers reason correctly about the program

### Linearizability

What **linearizable** means is that **when** you tell your distributed system to write a value and it ACKs the write, all subsequent reads **will** get the written value. Basically it says that all reads and writes should appear atomic from the point of view of the processes that read/write the value.

What would NOT be a linearizable execution? Here's a snippet courtesy of Stack Overflow:

```
public class Main extends Thread {

    boolean keepRunning = true; //should be volatile

    public void run() {
        while (keepRunning) {}
        System.out.println("Thread terminated.");
    }

    public static void main(String[] args) throws InterruptedException {
        Main t = new Main();
        t.start();
        Thread.sleep(1000);
        t.keepRunning = false;
        System.out.println("keepRunning set to false.");
    }
}
```


* `keepRunning` is shared between two threads (thread `t` and `main` thread)
* `main` writes to shared value `keepRunning`
* because we are not enforcing any consistency model, `keepRunning` is always `true` from `main`'s point of view; so the program won't stop.

A way to fix this is to make `keepRunning` `volatile`, thus enforcing linearizability. Linearizable means that when a write to `keepRunning` is finished (by executing `keepRunning = false`), eventually the other thread will see the new value.

The `volatile` keyword takes care of that because it makes sure reads and writes are atomic. So why wouldn't setting a variable be atomic? 

#### Short digression on CPU architectures

Modern systems can have multiple CPUs and multiple representations of the same value; typically a CPU will have registers, caches (L1, L2, L3), and main memory (the DIMMs on your motherboard). Registers are much faster to access than L1, L1 is faster than L2 and so on. Besides, a single assembly operation has some latency, so the CPU makes all sorts of tricks to make the code fast, including pipelining, reordering, speculative execution, etc.

Compilers can also reorder generated assembly as long as they deem it safe.
So how to fix this conundrum? Well, it depends on the language and the runtime (and the architecture). In case of JVM-based languages, the JVM has your back. The Java Language Specification introduced **semantics** that define what to expect, the so-called *within-thread as-if-serial semantics*, meaning that for a single `Thread`, the program order will be what that `Thread` observes:

```
new Runnable(() -> {
  a = 1;
  b = 2;
  c = 0;
  // this thread will observe value in the program order
});
```

**BUT**, what happens between threads?

### Happens-Before & synchronizes-with

Happens-before is a **guarantee** that the language & runtime gives about the behaviour of a program. It's a guarantee about the relation between reads and writes. It basically says that if instruction `A` has a *happens-before* relation with `B`, the results of `A` will surely be observed in `B`. It's not about actually happening before in wallclock time, it's about what *happens-before* (`A` and `B`) will be observed as if it happened before  - the actual assembly can happen in any order.

AFAIK, *synchronizes-with* is *happens-before* within Threads.

So, in a single-threaded program, there's a *happens-before* guarantee of program order, meaning that each source code line has a *happens-before* relation with the next line. <b>*Happens-before* is transitive too.</b>

What about multi-thread programs? The Java spec defines some ways of establishing a *happens-before* relation:

* Single thread and program order
* Monitor locks and unlocks (`synchronized` keyword)

		//T1
		synchronized(lock) {...}//monitor unlock
		
		//T2
		//monitor lock
		synchronized(lock) {...}

* Thread `start` and all of thread's actions

		Thread.start(() -> {a=1; b=2;})
		
* Thread `join` and all of the joining thread's actions

		//T1
		t = Thread.start(() -> {a=1; b=2;})
		
		//T2
		t.join()

* Volatile writes and reads

		//T1
		volatile int a = 0;
		...
		a = 1
		
		//T2
		if (a == 1) ...
		

----

Coming back, so what exactly does the JVM do?

By calling javap on the classfile `javap -h -p out.Main` you will see this: 

```
  volatile boolean keepRunning;
    descriptor: Z
    flags: ACC_VOLATILE
```

But can we see the actual assembly?

`-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -Xcomp -XX:CompileCommand=compileonly,*Main`

> I got sidestepped while trying to see the assembly, had to follow this guide to allow the JVM to print the assembly: https://github.com/liuzhengyang/hsdis

```
...
0x0000000109ca4697: lock addl $0x0,(%rsp)     ;*putstatic keepRunning
                                                ; - test.Main::main@21 (line 17)
```

