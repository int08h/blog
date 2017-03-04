+++
date = "2017-03-01T13:19:09-06:00"
title = "Ode to Dmitry Vyukov's MPSC Queue"
hidefromhome = "true"

+++

> In theory, there is no difference between practice and theory. In practice, there is.

This post pay homage to an embodiment of the above witticism: [Dmitry Vyukov](https://twitter.com/dvyukov)'s[^1] Multi-Producer Single-Consumer (MPSC) queue design.

[^1]: Dmitry's site [1024cores.net](http://www.1024cores.net) accumulates years of his insight into synchronization algorithms, multicore/multiprocessor development, and systems programming. If nothing else, visit his [queue taxonomy](http://www.1024cores.net/home/lock-free-algorithms/queues) and stand humbled knowing Dmitry has probably **forgotten** more about each dimension in that design space that most people presently know. As of this writing Dmitry is at Google where he's working on all manner of [wicked](https://github.com/google/syzkaller)-[cool](https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3471/original/Sanitizers.pdf) [dynamic](https://github.com/dvyukov/go-fuzz) testing tools. 

*In theory* the design lacks many of the properties that modern concurrent algorithms "must have" to be sound:

1. Overall it's a **blocking** algorithm. No *{wait,lock,obstruction}-freedom* here.
2. The algorithm is not *sequentially consistent*, not *linearizable*, not *weakly-consistent*, only **serializable**[^2]. 

[^2]: Fantastic discussions of this can be [found here](http://www.bailis.org/blog/linearizability-versus-serializability/) and on [Stackoverflow](http://stackoverflow.com/questions/4179587/difference-between-linearizability-and-serializability).

To say it another way, on paper this algorithm:

1. Makes no guarantee that any work is going to get done. 
2. Any work that is accomplished has no deterministic ordering.

As concurrent queues go, the above sounds like a winner, no?

*In practice* the design is a simple and elegant solution to 99.9% of a difficult problem space. It works very well. The design has seen widespread adoption in mainstream concurrent applications including [Netty (via JCTools)](https://github.com/JCTools/JCTools/blob/master/jctools-core/src/main/java/org/jctools/queues/MpscLinkedQueue.java), [Akka](https://github.com/akka/akka/blob/master/akka-actor/src/main/java/akka/dispatch/AbstractNodeQueue.java)[^3] and [Rust](https://github.com/rust-lang/rust/blob/master/src/libstd/sync/mpsc/mpsc_queue.rs). Below we'll examine the reasons why.

[^3]: Akka and JCTools change the algorithm slightly to make it linearizable at the expense of a weaker progress condition in `pop`. This is to necessary to conform to the `java.lang.Queue` interface. Consult [this post](http://psy-lob-saw.blogspot.com/2015/04/porting-dvyukov-mpsc.html) to start down *that* rabbit-hole (it's a deep one).

## Implementation

Below I'll reference the C++ implementation of the
[non-intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/non-intrusive-mpsc-node-based-queue) 
variant below[^4]. "Real world" performance sensitive applications would likely use the slightly more complicated 
[intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/intrusive-mpsc-node-based-queue)
version, but for explanatory purposes I prefer the non-intrusive version.

[^4]: The version here differs from Dmitry's in that elements are enqueued at the *tail* and removed from the *head*. The 1024cores implementation uses 'head' and 'tail' in the reverse manner (as does most of Dmitry's code).

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

## Progress Conditions

The Internet is replete with concurrent algorithm discussions where sloppy use of 
[progress condition](http://doc.akka.io/docs/akka/current/general/terminology.html#Non-blocking_Guarantees__Progress_Conditions_)
terminology causes confusion. *Lock-free* is particularly prone to misuse, typically 
in the context of "this code is implemented without locks/mutexes/critical sections". 
An implementation void of locks is quite different from an algorithm that 
guarantees overall system progress: deadlock awaits misusers of `cmpxchg` 
and `pthread_mutex_lock` alike. 

This article uses technical progress condition terms as follows: 

| Progress Condition |Definition   | Think of it as |
|---|---|---|
| *wait-free* |  All threads complete their call in a finite number of steps. The strongest guarantee of progress.| Everybody is working |
| *lock-free* |  At least *one* thread is always able to complete its call in a finite number of steps. Some threads may starve but overall system progress is guaranteed. | Somebody is working |
| *blocking*  |  Delay or error in one thread can prevent other threads from completing their call. Potentially all threads may starve and the system makes no progress.| Somebody is probably working unless bad things happen |

Here "*thread*" is an algorithm instance on a multiprocessor/multicore shared memory system able to execute multiple threads simultaneously. "*Completes its call*" is an invocation of `push` or `pop` that returns control to the caller. 

## Progress

DV-MPSC provides only a **blocking** progress guarantee, but that pretty much doesn't matter.

## Linearizability (Lack Of)

Bit Java implementations. More than once. 

> **Parsimonious** (adjective)
> ...[u]sing a minimal number of assumptions, steps, or conjectures.

