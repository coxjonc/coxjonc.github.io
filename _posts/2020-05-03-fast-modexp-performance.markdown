---
layout: post
title: "How to slow down a fast program"
date: 2020-05-03
---

The following problem, from [_Structure and Interpretation of Computer Programs_](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book.html) (SICP), shows how a small change can wreck the performance of a fast program. I like this problem because it forces you to think about the internals of the Lisp interpreter, and about the practical effects of trading a linear recursive algorithm for a tree-recursive one.

The example program we'll consider computes a modular exponent - $b^e \bmod m$. It's written in a dialect of Scheme used in SICP. If you want to run it locally I recommend the [DrRacket IDE](https://racket-lang.org/). DrRacket has an [sicp language](https://docs.racket-lang.org/sicp-manual/) that you can install using the built-in package manager, as well as some utilities for tracing function calls, which will come in handy later.

{% highlight scheme %}
(define (expmod base exp m)
  (cond ((= exp 0) 1)
        ((even? exp)
         (remainder (square (expmod base (/ exp 2) m)) m))
         (else
         (remainder (* base (expmod base (- exp 1) m)) m))))
{% endhighlight %}

I won't go into detail about how this algorithm works, but you can read SICP section 1.2.6 for a more thorough explanation, and a motivation for why we want to do fast exponentiation in the first place. What we are concerned with here is not the correctness of this program, but its running time.

To keep things simple, let's assume our input value $n$ is a power of two. At each step of the recurrence, we split the input in half and do a constant time operation. Writing this out as a recurrence relation we get:

$$
T(n)= T(n/2) + O(1)
$$

This relation should look familiar - it's the same recurrence you get by recursively searching a binary tree, or any other operation that repeatedly halves its input and does a constant-time operation on each step. Like binary search, we have an overall time complexity of $log(n)$.

Now that we've defined our fast function, let's see how a small change slows it down.

{% highlight scheme %}
(define (slow-expmod base exp m)
  (cond ((= exp 0) 1)
        ((even? exp)
         (remainder (* (slow-expmod base (/ exp 2) m)
                       (slow-expmod base (/ exp 2) m)) m)) ; bad idea!
         (else
         (remainder (* base (slow-expmod base (- exp 1) m)) m))))
{% endhighlight %}

No big changes. We just removed the call to `square` and explicitly multiplied the values. Logically, this function does the exact same thing as the first function, and it may not be immediately obvious why it should run slower. If we run it on a small input, we may not even notice any difference. But if we pass this function a large exponent we'll realize that our once fast procedure has slowed to a crawl.

Why should `((f x) * (f x))` run more slowly than `(square (f x))`? To understand why, we first need to think about how the interpreter evaluates expressions. Lisp interpreters use _`applicative order evaluation`_. I'll let SICP define that term for us:

  > _The interpreter first evaluates the operator and operands and then applies the resulting procedure to the resulting arguments._

This means that, before the interpreter calls a function, it evaluates the arguments to that function. When it encounters an expression like `(square (f x))`, it evaluates `(f x)`, and gets a number to pass to `square`. And when the interpreter sees an expression like `(* (f x) (f x))`, it evaluates `(f x)` not once, but twice.

In the case of our recursive modular exponent function, each function call could kick off a long chain of recursive calls to `f`. Once all those recursive calls complete, the interpreter moves on to the second operand, and does that work all over again. An optimizing compiler would probably get rid of these duplicate function calls, but our interpreter blindly executes the same function again, repeating its work and crippling our performance.

Using racket's built-in [`(trace)`](https://docs.racket-lang.org/reference/debugging.html) we can visualize these duplicate function calls. Calling `(trace f)` tells the interpreter to keep track of when `f` is called, and what value it eventually returns. When a traced function calls another traced function, then that invocation is _nested_, making it easy to visualize the way the stack grows and shrinks as the process runs. Here is a trace of a call to `(expmod 2 8 5)`, using our fast $O(\log n)$ algorithm.

```
>(expmod 2 8 5)
> (expmod 2 4 5)
> >(expmod 2 2 5)
> > (expmod 2 1 5)
> > >(expmod 2 0 5)
< < <1
< < 2
< <4
< 1
<1
1
```

We can see that `expmod` calls itself 4 times. Now let's see what `slow-expmod` does on the same input:

```
>(slow-expmod 2 8 5)
> (slow-expmod 2 4 5)
> >(slow-expmod 2 2 5)
> > (slow-expmod 2 1 5)
> > >(slow-expmod 2 0 5)
< < <1
< < 2
> > (slow-expmod 2 1 5)
> > >(slow-expmod 2 0 5)
< < <1
< < 2
< <4
> >(slow-expmod 2 2 5)
> > (slow-expmod 2 1 5)
> > >(slow-expmod 2 0 5)
< < <1
< < 2
> > (slow-expmod 2 1 5)
> > >(slow-expmod 2 0 5)
< < <1
< < 2
< <4
< 1
> (slow-expmod 2 4 5)
> >(slow-expmod 2 2 5)
> > (slow-expmod 2 1 5)
> > >(slow-expmod 2 0 5)
< < <1
< < 2
> > (slow-expmod 2 1 5)
> > >(slow-expmod 2 0 5)
< < <1
< < 2
< <4
> >(slow-expmod 2 2 5)
> > (slow-expmod 2 1 5)
> > >(slow-expmod 2 0 5)
< < <1
< < 2
> > (slow-expmod 2 1 5)
> > >(slow-expmod 2 0 5)
< < <1
< < 2
< <4
< 1
<1
1
```

Even on a small input, the difference is dramatic. `slow-expmod` calls itself 22 times, compared to `expmod`'s four times. With such a tiny input these extra calls don't change the running time of our procedure, but the difference becomes clear when you run it with much larger input. To test this, I wrote the following higher-order function that uses `runtime` to output the time elapsed, in microseconds, for the execution of `f`.

{% highlight scheme %}
(define (expmod start-time f base exp m)
  (f base exp m)
  (display (- (runtime) start-time)))
{% endhighlight %}

Calling this function on our slow and fast modular exponent programs, we see that the fast function takes about 5 µs to calculate $2^{10000} \bmod 5$, and the slow function takes about 2000 µs to calculate the same value. Even allowing for inaccuracies and variance in our timing function, that's a significant difference.
