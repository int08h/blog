+++
date = "2017-03-01T13:19:09-06:00"
title = "Ode to Dmitry Vyukov's MPSC Queue"

+++

This post pays homage to [Dmitry Vyukov](https://twitter.com/dvyukov)'s 
elegant, practical, and deceptively subtle Multi-Producer Single-Consumer (MPSC) queue design. 

## Queue Implementation

The remainder of this article will reference the C++ implementation below. This is the 
[non-intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/non-intrusive-mpsc-node-based-queue) 
version of the algorithm which is slightly simpler. Perfomance sensitive applications likely prefer the 
slightly more complicated 
[intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/intrusive-mpsc-node-based-queue)
version.

```c++
/*
 * Minimal C++ implementation of the non-intrusive Vyukov MPSC queue. 
 * Illustrates the spirit of the algorithm while being accessible 
 * to most readers. Not production code.
 *
 * Nodes are enqueued at the *tail*, and removed from the *head*.
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

It's easy to find discussions of concurrent algorithms where sloppy use of 
[progress condition](http://doc.akka.io/docs/akka/current/general/terminology.html#Non-blocking_Guarantees__Progress_Conditions_)
terms causes confusion. *Lock-free* in particular is prone to misuse, typically used 
incorrectly in the context of "this code is implemented without using any 
locks/mutexes/critical sections". 

These terms have specific technical meanings. In the discussion below they mean the following:

| Progress Condition |Definition   | A.K.A.|
|---|---|---|
| *wait-free* |  All threads can complete their call in a finite number of steps. This is the strongest guarantee of progress.| Everybody is getting work done |
| *lock-free* |  At least *one* thread is always able to complete its call in a finite number of steps. Other threads may starve. A weaker than wait-free but overall progress is still guaranteed. | Somebody is getting work done  |
| *blocking*  |  Delay or interruption by one thread can prevent other threads from completing their call. Potentially all threads may starve. No guarantee of progress can be made.|Can't work, hammer is broken |


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

