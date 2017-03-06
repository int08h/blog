+++
date = "2017-03-01T13:19:09-06:00"
title = "Ode to Dmitry Vyukov's MPSC Queue"
hidefromhome = "true"

+++

# Introduction

> In theory, there is no difference between practice and theory. In practice, there is.[^0]

[^0]: http://www.snopes.com/quotes/berra/practicetheory.asp

This post examines why [Dmitry Vyukov](https://twitter.com/dvyukov)'s[^1] Multi-Producer Single-Consumer (DV-MPSC) queue design is so practically useful despite apparent shortcomings.

[^1]: Dmitry's site [1024cores.net](http://www.1024cores.net) accumulates years of his insight into synchronization algorithms, multicore/multiprocessor development, and systems programming. If nothing else, visit his [queue taxonomy](http://www.1024cores.net/home/lock-free-algorithms/queues) and stand humbled knowing Dmitry has probably **forgotten** more about each dimension in that design space that most people presently know. As of this writing Dmitry is at Google where he's working on all manner of [wicked](https://github.com/google/syzkaller)-[cool](https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3471/original/Sanitizers.pdf) [dynamic](https://github.com/dvyukov/go-fuzz) testing tools. 

## Theoretical Shortcomings

From an academic computer-science standpoint, DV-MPSC lacks properties concurrent algorithms "must have" to be sound:

1. Overall it's a **blocking** algorithm. No *{wait,lock,obstruction}-freedom* here.
2. The algorithm is not *sequentially consistent*, not *linearizable*, not *weakly-consistent*, only **serializable**[^2]. 

[^2]: Fantastic discussions of this can be [found here](http://www.bailis.org/blog/linearizability-versus-serializability/) and on [Stackoverflow](http://stackoverflow.com/questions/4179587/difference-between-linearizability-and-serializability).

To say it another way, on paper DV-MPSC:

1. Makes no guarantee that any work is going to get done, and
2. Work that is lucky enough to get done has no overall deterministic ordering.

As concurrent queues go this sounds like a winner, no? 

## Practical Benefits

In practice DV-MPSC is a simple solution to a difficult problem space. Its simplicity makes it *easy to understand* and similarly *easy to debug*; invaluable properties for those working on concurrent systems. DV-MPSC "must be doing something right" as it's been adopted by mainstream concurrent applications including [Netty (via JCTools)](https://github.com/JCTools/JCTools/blob/master/jctools-core/src/main/java/org/jctools/queues/MpscLinkedQueue.java), [Akka](https://github.com/akka/akka/blob/master/akka-actor/src/main/java/akka/dispatch/AbstractNodeQueue.java) and [Rust](https://github.com/rust-lang/rust/blob/master/src/libstd/sync/mpsc/mpsc_queue.rs)[^3]. 

[^3]: Akka and JCTools change the algorithm slightly to make it linearizable at the expense of a weaker progress condition in `pop`. This is to necessary to conform to the `java.lang.Queue` interface. Consult [this post](http://psy-lob-saw.blogspot.com/2015/04/porting-dvyukov-mpsc.html) to start down *that* rabbit-hole (it's a deep one).

# Implementation

Our discussion will focus on the below C++ implementation of the
[non-intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/non-intrusive-mpsc-node-based-queue) 
variant[^4]. "Real world" performance sensitive applications would likely use the slightly more complicated 
[intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/intrusive-mpsc-node-based-queue)
version.

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

# Quick Terminology Review

[Progress condition](http://doc.akka.io/docs/akka/current/general/terminology.html#Non-blocking_Guarantees__Progress_Conditions_)
terminology is prone to misuse. "*Lock-free*" is the typical victim, usually employed to mean "not using locks/mutexes/critical sections" 
and not as a guarantee of overall system progress. To be clear, deadlock awaits misusers of `cmpxchg` and `pthread_mutex_lock` alike. 

| Progress Condition |Definition   | Think of it as |
|---|---|---|
| *wait-free* |  All threads complete their call in a finite number of steps. The strongest guarantee of progress.| Everybody is working |
| *lock-free* |  At least *one* thread is always able to complete its call in a finite number of steps. Some threads may starve but overall system progress is guaranteed. | Somebody is working |
| *blocking*  |  Delay or error in one thread can prevent other threads from completing their call. Potentially all threads may starve and the system makes no progress.| Somebody is probably working unless bad things happen |

"*Thread*" is an algorithm instance on a multiprocessor/multicore shared memory system. The shared memory system is able to execute multiple threads simultaneously. "*Completes its call*" is an invocation of `push` or `pop` that returns control to the caller thread. 

# Simplicity

A brief examination of the implementation will make it apparent that DV-MPSC is a linked list. 
This fact alone makes DV-MPSC more approachable than the majority of other concurrent queue algorithms. 
Furthermore the implementation is straight-forward and easy to follow:

* `push` and `pop` use a minumum of atomic operations and do so without looping
* There are no complications due to contented or partially-applied operations

# Progress

DV-MPSC provides only a **blocking** progress guarantee, but that pretty much doesn't matter.

# Linearizability (Lack Of)

FIFO per-producer is good enough. Bit Java implementations. More than once. 

# Closing Thoughts

I'm sure I have them.


