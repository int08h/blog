+++
date = "2017-03-01T13:19:09-06:00"
title = "Ode to Dmitry Vyukov's MPSC Queue"
hidefromhome = "true"

+++

# Introduction

> **parsimonious** (adjective)  
> *Using a minimal number of assumptions, steps, or conjectures.*

[^0]: http://www.snopes.com/quotes/berra/practicetheory.asp

This post examines an instance of simplicity in action: [Dmitry Vyukov](https://twitter.com/dvyukov)'s[^1] Multi-Producer Single-Consumer (DV-MPSC) queue. 

[^1]: Dmitry's site [1024cores.net](http://www.1024cores.net) accumulates years of his insight into synchronization algorithms, multicore/multiprocessor development, and systems programming. If nothing else, visit his [queue taxonomy](http://www.1024cores.net/home/lock-free-algorithms/queues) and stand humbled knowing Dmitry has probably **forgotten** more about each dimension in that design space that most people presently know. As of this writing Dmitry is at Google where he's working on all manner of [wicked](https://github.com/google/syzkaller)-[cool](https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3471/original/Sanitizers.pdf) [dynamic](https://github.com/dvyukov/go-fuzz) testing tools. 

Attacking one aspect of the more general producer-consumer problem, DV-MPSC addresses the **essential complexity** at hand and no more. Compared to similar solutions, DV-MPSC's simplicity makes it *easy to understand*, *easy to implement*, and *easy to debug*. These invaluable (and uncommon) properties have driven its adoption in mainstream concurrent systems like [Netty (via JCTools)](https://github.com/JCTools/JCTools/blob/master/jctools-core/src/main/java/org/jctools/queues/MpscLinkedQueue.java), [Akka](https://github.com/akka/akka/blob/master/akka-actor/src/main/java/akka/dispatch/AbstractNodeQueue.java) and [Rust](https://github.com/rust-lang/rust/blob/master/src/libstd/sync/mpsc/mpsc_queue.rs)[^3]. 

[^3]: Akka and JCTools change the algorithm slightly to make it linearizable at the expense of a weaker progress condition in `pop`. This is to necessary to conform to the `java.lang.Queue` interface. Consult [this post](http://psy-lob-saw.blogspot.com/2015/04/porting-dvyukov-mpsc.html) to start down *that* rabbit-hole (beware it's a deep one).

# Implementation

I'll focus on the below C++ implementation of the
[non-intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/non-intrusive-mpsc-node-based-queue) 
variant[^4]. Granted performance sensitive applications would likely use the 
[intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/intrusive-mpsc-node-based-queue)
version, but the non-intrusive version is easier to explain.

[^4]: The version here differs from 1024cores in that elements are enqueued at the *tail* and removed from the *head*. The 1024cores implementation uses 'head' and 'tail' in the reverse manner (as does most of Dmitry's code).

```c++
/*
 * Minimal C++ implementation of the non-intrusive Vyukov MPSC queue. 
 * Illustrates the spirit of the algorithm while being accessible 
 * to most readers. Not production code.
 *
 * Nodes are enqueued at the *tail* and removed from the *head*.
 */
struct Node {
  std::atomic<Node*> next;
  SomeOpaqueType value; 
};

class MpscQueue {
public:
  MpscQueue() : stub(new Node()), head(stub.get()), tail(stub.get()) {
    stub->next.store(nullptr);
  }

  void push(Node* node) {
    node->next.store(nullptr, memory_order_relaxed);

    Node* prev = tail.exchange(node, memory_order_acq_rel);  // (1) Serialize producers
                                                             // (2) DANGER ZONE 
    prev->next.store(node, memory_order_release);            // (3) Serialize consumer
  }

  Node* pop() {
    Node* head_copy = head.load(memory_order_relaxed);
    Node* next = head_copy->next.load(memory_order_acquire); // (4) rel/acq with #3

    if (next != nullptr) {
      head.store(next, memory_order_relaxed);
      head_copy->value = next->value;
      return head_copy;
    }
    return nullptr;
  }

private:
  std::unique_ptr<Node> stub;
  std::atomic<Node*> head;
  std::atomic<Node*> tail;
};
```

> ## Quick Terminology Review
>
> This post uses concurrent [progress condition](http://doc.akka.io/docs/akka/current/general/terminology.html#Non-blocking_Guarantees__Progress_Conditions_)
> terminology to describe properties of DV-MPSC. Such terminology is prone to misuse and can be confusing. To be clear terms are used as follows:
>
> | Progress Condition |Definition   | Think of it as |
> |---|---|---|
> | *wait-free* |  All threads complete their call in a finite number of steps. The strongest guarantee of progress.| Everybody is working |
> | *lock-free* |  At least *one* thread is always able to complete its call in a finite number of steps. Some threads may starve but overall system progress is guaranteed. | Somebody is working |
> | *blocking*  |  Delay or error in one thread can prevent other threads from completing their call. Potentially all threads may starve and the system makes no progress.| Somebody is probably working unless bad things happen |
> 
> Additional terms: 
>
>   * **Thread** - An algorithm instance on a multiprocessor/multicore shared memory system. We assume multiple threads are able to execute simultaneously.
>   * **Completes its call** - An invocation of the `push` or `pop` method that returns control to the caller thread.

# Shortcomings

Looking at DV-MPSC from an academic computer-science standpoint one might walk away disappointed. DV-MPSC lacks 
properties typically associated with "useful" concurrent algorithms:

1. Overall it's a **blocking** algorithm. No *{wait,lock,obstruction}-freedom* here.
2. The algorithm is not *sequentially consistent*, not *linearizable*, not *weakly-consistent*, only **serializable**[^2]. 

[^2]: Discussions of these finer points [here](http://www.bailis.org/blog/linearizability-versus-serializability/) and [here](http://stackoverflow.com/questions/4179587/difference-between-linearizability-and-serializability).

To say it another way, on paper DV-MPSC:

1. Makes no guarantee that any work is going to get done, and
2. Work that is lucky enough to get done has no overall deterministic ordering.

As concurrent queues go this sounds like a winner, no? 

# Implementation Simplicity

A cursory examination of the implementation makes it apparent that DV-MPSC is a linked list
plus some atomic operations. Additionally:

* `push` and `pop` employ a minimal number of atomic operations.
* No need to handle contented or partially-applied operations. 
* No loops (`compare-and-swap` or `load-linked/store-conditional` loops).

These qualities make DV-MPSC more approachable than the many other concurrent queue algorithms.  
Developers can lean on prior experience with linked lists to quickly grasp basics and 
save cognitive heavy-lifting for understand how the (straight-forward) atomic operations work. 

# Progress 

As an algorithm DV-MPSC provides a **blocking** progress guarantee. However, absent a rare
worst-case that I'll discuss, the algorithm will behave as if its *wait-free*. Let's examine 
`push` and `pop` in more detail and see why this is.

```c++
 void push(Node* node) {
   node->next.store(nullptr, memory_order_relaxed);
 
   Node* prev = tail.exchange(node, memory_order_acq_rel);  // (1) Serialize producers
                                                            // (2) DANGER ZONE 
   prev->next.store(node, memory_order_release);            // (3) Serialize consumer
 }
```



Producers 1 and 2 are both enqueing  they will contend
on `tail`. `exchange` is an atomic read-modify-write (RMW) 2-consensus operation[^5].

```text
stub
next = nullptr
value = nullptr
```


[^5]: http://www.cs.yale.edu/homes/aspnes/pinewiki/WaitFreeHierarchy.html


# Partial order is sufficient

Imagine there are two threads invoking `push` simultaneously:

```text
Producer 1  push(a)
Producer 2  push(b)
Consumer            pop(?) pop(?)
---------- program order ------------>
```

Later in program order the consumer executes two sequential `pop` operations 
and retreives `[a, b]`. Intuitively this makes sense: the `push` operations were 
simultaneous and we don't know which producer "won" the race
to enqueue an element first. We can also accept that the two sequential `pop` 
operations could have instead produced `[b, a]` without violating our intuition.
(The empty result case is ignored in this example.)

Now imagine three producers:

```text
Producer 1   push(a)         push(d)
Producer 2           push(c)
Producer 3   push(b)
Consumer                             pop(?)
------------ program order -------------->
```

What value will `pop` have? Ignoring the empty case, `pop` could return 
`a`, `b`, or `c` but *not* `d`. In fact, all the lists below are valid orderings of 
values returned by sequential `pop` operations (non-exhaustive):

```text
[a, b, c, d]   
[b, c, a, d]   
[b, a, c, d]   
[c, a, d, b]   
[a, b, d, c]
```

DV-MPSC ensures that individual producers are FIFO--there is no valid history where
`d` is read before `a`--but provides no ordering *between* producers. A bit unintuitive
at first, but this partial ordering is sufficient to implement the Actor Model.

# Closing Thoughts

I'm sure I have them.


