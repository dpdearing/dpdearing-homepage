---
layout: post
title: Delimited Continuations in Scala
tags:
- Scala
redirect_from:
  - /2011/12/delimited-continuations-in-scala/
---
First, the teaser:
> [Swarm](http://swarmframework.org) is a framework allowing the creation of web applications which can scale transparently through a novel portable continuation-based approach. Like Map-Reduce, Swarm follows the maxim "move the computation, not the data". However Swarm takes the concept much further, allowing it to be applied to almost any computation, not just those that can be broken down into map and reduce operations.

This is pretty cool.  It is accomplished using Scala delimited continuations, which are confusing.

[Delimited Continuations Explained (in Scala)](http://dcsobral.blogspot.com/2009/07/delimited-continuations-explained-in.html)

I actually found that link to be pretty helpful, but you need to be stone cold sober to get through it.  I wanted to jot down my notes here because otherwise I'll have to start from the beginning when I try to understand it again.  So the rest of this blog post is essentially just a journal entry.  Read on if you dare!

_**Update:**_ Well, that didn't work. I had to read it again anyway!

---

In Scala, continuations will be used through two keywords: <code>reset</code> and <code>shift</code>. Like <code>catch</code> and <code>throw</code>, a <code>shift</code> always happen inside a <code>reset</code>, the latter being responsible for delimiting the scope of the continuation.

Wait.  What?

<strong><code>reset</code> is responsible for delimiting the scope of the continuation</strong>

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

The Clif Notes version of how this works:

1. The code before `shift` is executed once
1. The code after `shift` as many times as the function (`k`) inside `shift` is repeated
1. the code inside the `shift` itself is only executed once, but is interrupted--going back and forth to the code after it (until the end of `reset`)

---

The first thing to recognize is that shifts are functions taking a parameter, and that parameter is another function.  In this example, that parameter is the function <code>{ k => ... }</code>.  Not only that, but this example defines the type of `k` as being `Int => Int`, which is itself a function.

When the continuation function <code>k</code> is executed (from _within_ the <code>shift</code> block):

1. Execution skips over the rest of the <code>shift</code> block and begins again at the end of it
1. Execution continues until the end of the <code>reset</code> block
1. Execution then resumes within the <code>shift</code> block after the already-completed execution of <code>k</code> until reaching the end of the <code>shift</code> block again
1. Execution skips to the end of the <code>reset</code> block
1. Repeat the previous two steps for each call to <code>k</code> within the <code>shift</code> block

So what happens in the above example is:
1. <code>baz()</code> is called, which calls <code>bar()</code>, and then <code>foo()</code>
1. <code>foo()</code> prints <code>Once here.</code>
1. The <code>shift</code> block is entered, which recursively calls <code>k</code>.  The innermost call to <code>k(7)</code> occurs first, and execution skips to the end of the <code>shift</code> block.  The result is 7.
1. Now <code>foo()</code> is complete and we're in <code>bar()</code>, adding 1 to 7 (the result) = 8
1. Back in <code>baz()</code>, 8 (the result) * 2 = 16
1. We reach the end of the <code>reset</code> block and resume execution in the <code>shift</code> block, after the previous execution of <code>k</code>.  The second call to <code>k</code> occurs with the current value of 16.  Execution again skips to the end of the <code>shift</code> block and returns to <code>bar()</code> with this new result of 16
1. 1 + 16 = 17
1. 17 * 2 = 34
1. Reach the end of the <code>reset</code> block again and resume at the end of the second call to <code>k</code>, leading to the third and final call to <code>k</code> with the current value of 34
1. 1 + 34 = 35
1. 35 * 2= 70

And the result is:
```
Once here.
70
```
