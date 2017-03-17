+++
date = "2017-03-15T13:19:09-06:00"
title = "Ode to a Vyukov Queue"
hidefromhome = "false"

+++

# Introduction

This post celebrates an instance of simplicity in action: [Dmitry Vyukov](https://twitter.com/dvyukov)'s[^1] Multi-Producer Single-Consumer queue (DV-MPSC). 

[^1]: Dmitry's site [1024cores.net](http://www.1024cores.net) accumulates years of his insight into synchronization algorithms, multicore/multiprocessor development, and systems programming. If nothing else, visit his [queue taxonomy](http://www.1024cores.net/home/lock-free-algorithms/queues) and stand humbled knowing Dmitry has probably **forgotten** more about each dimension in that design space that most people presently know. As of this writing Dmitry is at Google where he's working on all manner of [wicked](https://github.com/google/syzkaller)-[cool](https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3471/original/Sanitizers.pdf) [dynamic](https://github.com/dvyukov/go-fuzz) testing tools. 

DV-MPSC is one of those rare solutions that addresses **essential complexity** and no more. The result is something *easy to understand* and *easy to implement*. Such clarity is uncommon in the concurrent algorithm space and has no doubt contributed to adoption of DV-MPSC in mainstream concurrent systems like [Netty (via JCTools)](https://github.com/JCTools/JCTools/blob/master/jctools-core/src/main/java/org/jctools/queues/MpscLinkedQueue.java), [Akka](https://github.com/akka/akka/blob/master/akka-actor/src/main/java/akka/dispatch/AbstractNodeQueue.java) and [Rust](https://github.com/rust-lang/rust/blob/master/src/libstd/sync/mpsc/mpsc_queue.rs)[^3]. 

[^3]: Akka and JCTools change the algorithm slightly to make it linearizable at the expense of a weaker progress condition in `pop`. This is to necessary to conform to the `java.lang.Queue` interface. Consult [this post](http://psy-lob-saw.blogspot.com/2015/04/porting-dvyukov-mpsc.html) to start down *that* rabbit-hole (beware it's a deep one).

Such elegance is not without tradeoffs. We'll investigate two commonly discussed shortcomings below--a blocking progress guarantee and serializable consistency--and review their real-world impact. 

> ## Terminology Review
>
> This post dicusses the [progress condition](http://doc.akka.io/docs/akka/current/general/terminology.html#Non-blocking_Guarantees__Progress_Conditions_) properties of DV-MPSC. To get all readers on the same page, terms herein are used as follows:
>
> | Progress Condition |Definition   | Think of it as |
> |---|---|---|
> | *wait-free* |  All threads complete their call in a finite number of steps. The strongest guarantee of progress.| Everybody is working |
> | *lock-free* |  At least *one* thread is always able to complete its call in a finite number of steps. Some threads may starve but overall system progress is guaranteed. | Somebody is working |
> | *blocking*  |  Delay or error in one thread can prevent other threads from completing their call. Potentially all threads may starve and the system makes no progress.| Somebody is probably working unless bad things happen |
> 
> Additionally: 
>
>   * **Thread** - An algorithm instance on a multiprocessor/multicore shared memory system. We assume multiple threads are able to execute simultaneously.
>   * **Completes its call** - An invocation of the `push` or `pop` method that returns control to the caller thread.

# Algorithm Properties

Looking at DV-MPSC from an academic computer-science standpoint one might be a tad disappointed: DV-MPSC lacks 
properties typically associated with concurrent algorithms:

1. Overall it's a **blocking** algorithm. No *{wait,lock,obstruction}-freedom* here.
2. The algorithm is not *sequentially consistent* nor *linearizable*, only **serializable**[^2]. 

[^2]: Discussions of these finer points [here](http://www.bailis.org/blog/linearizability-versus-serializability/) and [here](http://stackoverflow.com/questions/4179587/difference-between-linearizability-and-serializability).

Put another way, on paper DV-MPSC:

1. Makes no guarantee that any work is going to get done, and
2. Work that is lucky enough to get done has no overall deterministic ordering.

But all is not lost...read on.

# Examining the Implementation 

```c++
/*
 * Minimal C++ implementation of the non-intrusive Vyukov MPSC queue. 
 * Illustrates the spirit of the algorithm while being accessible 
 * to most readers.
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
    Node* prev = tail.exchange(node, memory_order_acq_rel);  
    prev->next.store(node, memory_order_release);           
  }

  Node* pop() {
    Node* head_copy = head.load(memory_order_relaxed);
    Node* next = head_copy->next.load(memory_order_acquire);

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

(This is the
[non-intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/non-intrusive-mpsc-node-based-queue) 
variant[^4])

[^4]: The version here differs from 1024cores in that elements are enqueued at the *tail* and removed from the *head*. The 1024cores implementation uses 'head' and 'tail' in the reverse manner. Additionally, performance sensitive applications would likely use the [intrusive](http://www.1024cores.net/home/lock-free-algorithms/queues/intrusive-mpsc-node-based-queue) version, but the non-intrusive version is easier to explain.

DV-MPSC boils down to linked list + atomic operations. Additionally:

* `push` and `pop` employ a minimal number of atomic operations.
* There are no special cases to handle contended or partially-applied operations. 
* Also no compare-and-swap loops or retries.

These qualities make DV-MPSC more approachable than many other concurrent queue algorithms
(compare to the familiar [M&S queue](https://www.cs.rochester.edu/research/synchronization/pseudocode/queues.html) 
which has loops and needs at least two compare-and-swaps to enqueue a node[^6]). Developers can lean on 
prior experience with linked lists to quickly grasp basics and save cognitive 
heavy-lifting for understand how the atomic operations and corresponding memory orderings work.

[^6]: I fully acknowledge that the M&S queue addresses the more challenging MPMC case. I highlight due to it being frequently encountered as an exemplar of concurrent queues in general. 

# Block me, maybe?

Generally DV-MPSC operations are *wait-free*. Recall that both
`push` and `pop` have no loops or special cases; they execute straight-through every time.

However there is a corner-case lurking in `push` that knocks DV-MPSC into the *blocking* category.
The window of vulnerability for this is quite small, one instruction on x86_64, but it's
a concern nonetheless. Let's examine `push` in more detail and see what is going on:

## How the queue becomes blocking 
Recall how `push` is implemented:

```c++
 void push(Node* node) {
   node->next.store(nullptr, memory_order_relaxed);
 
   Node* prev = tail.exchange(node, memory_order_acq_rel);  // #1 Serialize producers
                                                            // #2 DANGER ZONE 
   prev->next.store(node, memory_order_release);            // #3 Serialize consumer
 }
```

A new queue starts in this state:

```text
Queue:
 head -> stub 
 tail -> stub

List contents:
 stub
 next -> nullptr
```

A producer thread executes `push(a)` and the queue now looks like this:

```text
Queue:
 head -> stub 
 tail -> a

List contents:
 stub           a  
 next -> a      next -> nullptr
```

Now a producer begins to execute `push(b)` completing `#1` 
but stopping **before** `#3`. The producer thread ceased execution (crashed, 
preempted, etc) exactly at `#2` leaving the queue in this state:

```text
Queue:
 head -> stub 
 tail -> b

List contents:
 stub           a                    b
 next -> a      next -> nullptr      next -> nullptr
```

Uh-oh, the queue is inconsistent: 

* `tail` points to `b`; however 
* `b` is unreachable starting from `head`

Additional calls to `push` will add more unreachable nodes to the list.
Calls to `pop` will return `a` and then subsequently `nullptr` as if the list were empty.

Revisiting the above definition of *blocking*:

> ...[p]otentially all threads may starve and the system makes no progress.

And we see why DV-MPSC taken as a whole is *blocking*: if this inconsistent condition is 
encountered the algorithm will never deliver elements to the consumer again.
Despite this broken state individual calls to `push` and `pop` will continue to execute 
in a finite number of steps however the system **overall** fails to make progress.

## How bad is it really?

Not so bad. Vulnerability is one instruction long. In user-mode code how would one get there? 

# Serializable ain't so bad

Imagine there are two threads invoking `push` simultaneously:

```text
Producer 1  push(a)
Producer 2  push(b)
Consumer            pop(?) pop(?)
---------- program order ------------>
```

Later in program order the consumer executes two sequential `pop` operations 
and retrieves `[a, b]`. Intuitively this makes sense: the `push` operations were 
simultaneous and we don't know which producer "won" the race
to enqueue an element first. We can also accept that the two sequential `pop` 
operations could have instead produced `[b, a]` without violating our intuition
(the empty result case is ignored in this example).

Now imagine three producers:

```text
Producer 1   push(a)         push(d)
Producer 2           push(c)
Producer 3   push(b)
Consumer                             pop(?)
------------ program order -------------->
```

What value will `pop` have? Ignoring the empty case, `pop` could return 
`a`, `b`, or `c` but *not* `d`. In fact, all of the lists below are valid orderings 
for sequential `pop` operations (non-exhaustive):

```text
[a, b, c, d]   
[b, c, a, d]   
[b, a, c, d]   
[c, a, d, b]   
[a, b, d, c]
```

DV-MPSC ensures that *individual producers* are FIFO--there is no valid history where
`d` is read before `a`--but provides no ordering *between* producers. This
may not seem like much, but this partial ordering is sufficient to implement 
powerful concurrent systems such as the actor model.

# Closing Thoughts

Up for learning more? The footnotes will get you started and you can also take a look at what 
[others](http://concurrencyfreaks.blogspot.com/2014/04/multi-producer-single-consumer-queue.html)
[have](http://psy-lob-saw.blogspot.com/2015/04/porting-dvyukov-mpsc.html) 
[written](https://blogs.oracle.com/dave/entry/ptlqueue_a_scalable_bounded_capacity) 
about DV-MPSC.


