+++
date = "2017-03-01T13:19:09-06:00"
title = "Ode to Dmitry Vyukov's MPSC Queue"
hidefromhome = "true"

+++

Computer science practitioners are no doubt familiar with this saying:

> In theory, there is no difference between practice and theory. In practice, there is.

[Dmitry Vyukov](https://twitter.com/dvyukov)'s Multi-Producer Single-Consumer (MPSC) queue 
design is an embodiment of the above witticism. It lacks many qualities that concurrent 
queue algorithms "must have" to be theoretically sound. While in practice it's a simple
and elegant solution to 99% of the problem space. This post pays homage to that connundrum.

## Queue Implementation

This article will reference the C++ implementation of the
[non-intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/non-intrusive-mpsc-node-based-queue) 
variant below. "Real world" perfomance sensitive applications will use the slightly more complicated 
[intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/intrusive-mpsc-node-based-queue)
version, but for explanatory purposes the non-intrusive version is better.

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

### Dmitry

Dmitry is a prolific developer. His site [1024cores.net](http://www.1024cores.net) accumulates years of 
his insight into synchronization algorithms, multicore/multiprocessor development, and systems 
programming. If nothing else, visit his [queue taxonomy](http://www.1024cores.net/home/lock-free-algorithms/queues)
and be humbled knowing Dmitry has probably explored the very corners of each dimension in that design space.

As of this writting Dmitry is at Google where he's working on all manner of 
[wicked](https://github.com/google/syzkaller)-[cool](https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3471/original/Sanitizers.pdf)
[dynamic](https://github.com/dvyukov/go-fuzz) 
testing tools. 


Strictly speaking, DV-MPSC looks pretty bad:

1. It provides only a **blocking** progress guarantee 
2. The `push` operation is not linearizable.




## Progress

DV-MPSC provides only a **blocking** progress guarantee, but that pretty much doesn't matter.

## Linearizability (Lack Of)

Bit Java implementations. More than once. 

> **Parsimonious** (adjective)
> ...[u]sing a minimal number of assumptions, steps, or conjectures.

