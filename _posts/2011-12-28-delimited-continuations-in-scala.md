---
layout: post
title: Delimited Continuations in Scala
tags:
- Scala
redirect_from:
  - /2011/12/delimited-continuations-in-scala/
excerpt: Delimited continuations are like a functional version of GOTO statements, but much less evil.  They are a way of changing the execution flow of a program, which makes them very powerful but also _extremely_ confusing.
---
First, the teaser:
> [Swarm](http://swarmframework.org) is a framework allowing the creation of web applications which can scale transparently through a novel portable continuation-based approach. Like Map-Reduce, Swarm follows the maxim "move the computation, not the data". However Swarm takes the concept much further, allowing it to be applied to almost any computation, not just those that can be broken down into map and reduce operations.

This is pretty cool.  It is accomplished using Delimited Continuations with the [Scala continuations plugin](https://github.com/scala/scala-continuations).  Delimited continuations are like a functional version of GOTO statements, but way less evil.  They are a way of changing the execution flow of a program, which makes them very powerful but also _extremely_ confusing.

[Delimited Continuations Explained (in Scala)](http://dcsobral.blogspot.com/2009/07/delimited-continuations-explained-in.html)

I actually found that link to be pretty helpful, but you need to be stone cold sober to get through it.  I wanted to jot down my notes here because otherwise I'll have to start from the beginning when I try to understand it again.  So the rest of this blog post is essentially just a journal entry.  Read on if you dare!

_**Update:**_ Well, that didn't work. I had to read it again anyway!

---

In Scala, continuations will be used through two keywords: `reset` and `shift`. Like `catch` and `throw`, a `shift` always happen inside a `reset`, the latter being responsible for delimiting the scope of the continuation.

Wait.  What?

<strong>`reset` is responsible for delimiting the scope of the continuation</strong>

Oh, that didn't clear things up?  How about an example:
```scala
def foo() = {
  println("Once here.")
  shift((k: Int => Int) => k(k(k(7))))
}
def bar() = {
  1 + foo()
}
def baz() = {
  reset(bar() * 2)
}
baz()
// result:
// Once here.
// 70
```

**The CliffsNotes version of how this works:**

1. The code before `shift` is executed once
1. The code after `shift` as many times as the function `k` inside `shift` is repeated
1. the code inside the `shift` itself is only executed once, but is interrupted--going back and forth to the code after it (up until the end of the `reset` block)

---

The first thing to recognize is that shifts are functions taking a parameter, and that parameter is another function.  In this example, that parameter is the function `{ k => ... }`.  Not only that, but this example defines the type of `k` as being `Int => Int`, which is itself a function.

When the continuation function `k` is executed (from _within_ the `shift` block):

1. Execution skips over the rest of the `shift` block and begins again at the end of it
1. Execution continues until the end of the `reset` block
1. Execution then resumes within the `shift` block after the already-completed execution of `k` until reaching the end of the `shift` block again
1. Execution skips to the end of the `reset` block
1. Repeat the previous two steps for each call to `k` within the `shift` block

So what happens in the above example is:
1. `baz()` is called, which calls `bar()`, and then `foo()`
1. `foo()` prints `Once here.`
1. The `shift` block is entered, which recursively calls `k`.  The innermost call to `k(7)` occurs first, and execution skips to the end of the `shift` block.  The result is 7.
1. Now `foo()` is complete and we're in `bar()`, adding 1 to 7 (the result) = 8
1. Back in `baz()`, 8 (the result) * 2 = 16
1. We reach the end of the `reset` block and resume execution in the `shift` block, after the previous execution of `k`.  The second call to `k` occurs with the current value of 16.  Execution again skips to the end of the `shift` block and returns to `bar()` with this new result of 16
1. 1 + 16 = 17
1. 17 * 2 = 34
1. Reach the end of the `reset` block again and resume at the end of the second call to `k`, leading to the third and final call to `k` with the current value of 34
1. 1 + 34 = 35
1. 35 * 2= 70

And the result is:
```
Once here.
70
```
