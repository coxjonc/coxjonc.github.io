---
layout: post
title: "How to slow down a fast program"
date: 2020-05-03
---

The following problem, from _Structure and Interpretation of Computer Programs_ (SICP), shows how a small change can wreck the performance of a fast program.

Take a look at the following program for doing fast modular exponentiation. It calculates \\(b^e \bmod m\\) by repeatedly squaring \\(b\\) and taking the remainder of the intermediate results mod \\(m\\), with a little extra logic to handle odd exponents.

{% highlight scheme %}
(define (expmod base exp m)
  (cond ((= exp 0) 1)
        ((even? exp)
         (remainder (square (expmod base (/ exp 2) m)) m))
         (else
         (remainder (* base (expmod base (- exp 1) m)) m))))
{% endhighlight %}

Don't get too hung up on how the algorithm works - we're mostly interested in its running time. At each step of the recurrence, we split the input in half and do a constant time operation by calling `*` or `square`. Writing this all out as a recurrence relation we get:

$$
	T(n)= T(n/2) + O(1)
$$

This relation should look familiar - it's the same recurrence you get by recursively searching a binary tree, or any other operation that repeatedly halves its input. Like binary search, we have an overall time complexity of \\(log(n)\\).

Now that we have a fast procedure, let's try slowing it way down.

{% highlight scheme %}
(define (slow-expmod base exp m)
  (cond ((= exp 0) 1)
        ((even? exp)
         (remainder (* (slow-expmod base (/ exp 2) m)
                       (slow-expmod base (/ exp 2) m)) m)) ; bad idea!
         (else
         (remainder (* base (slow-expmod base (- exp 1) m)) m))))
{% endhighlight %}

No big changes. We just removed the call to `square` and replaced it with a multiplication. Logically, this function does the exact same thing as the first function. If we run it on a small input, we may not even notice any difference. But if we pass this function a large exponent, something like `(slow-expmod 9 100000 5)` we'll realize that our once fast procedure has slowed to a crawl.

Why should `((f x) * (f x))` run more slowly than `(square (f x))`? To understand why, we first need to think about how the Scheme interpreter evaluates expressions. Lisp interpreters use _`applicative order evaluation`_. I'll let SICP define that term for us:

  > the interpreter first evaluates the operator and operands and then applies the resulting procedure to the resulting arguments.

This means that, before the interpreter calls a function, it evaluates the arguments to that function. When encounters an expression like `(square (f x))`, it evaluates `(f x)` just once, and gets a number to pass to `square`.

The problems start when the interpreter sees an expression like `(* (f x) (f x))`. It needs to evaluate the operands to figure out what numbers to multiple. First the interpreter computes `(f x)`. In the case of modular exponentiation, this could kick off a chain of recursive calls to `f`. Once all those recursive calls complete, the interpreter moves on to the second operand and computes the exact same value again.

Using racket's built-in [`(trace)`](https://docs.racket-lang.org/reference/debugging.html) we can visualize these duplicate function calls. By calling `(trace f)` we can see when `f calls itself, giving us a good way to visualize the way the stack grows and shrinks as the process runs. Here is a trace of a call to `(expmod 2 3 5)`.

```
>(expmod 2 3 5)
> (expmod 2 2 5)
> >(expmod 2 1 5)
> > (expmod 2 0 5)
< < 1
< <2
< 4
<3
```

We can see that `expmod` calls itself 3 times. Now let's see what `slow-expmod` does:

```
>(slow-expmod 2 3 5)
> (slow-expmod 2 2 5)
> >(slow-expmod 2 1 5)
> > (slow-expmod 2 0 5)
< < 1
< <2
> >(slow-expmod 2 1 5)
> > (slow-expmod 2 0 5)
< < 1
< <2
< 4
<3
```

`slow-expmod` calls itself six times, with an extra call to `(slow-expmod 2 1 5)`. With such a tiny input these extra calls don't make a practical difference, but the difference becomes clear when you run it with much larger input. Now that we've seen this slowdown in practice, let's write it out as a formal recurrence relation. We know that `slow-expmod` calls itself twice, so we get:

$$
T(x) = 2T(x/2) + O(1)
$$

Writing this recurrence out as a series makes it clear which term dominates.

$$
T(x) = O(1) * (2 + 2^2 + 2^3 + ... 2^{log_2n})
$$

It looks like this series is dominated by the last term, \\(2^{log_2n} = n\\), giving us an \\(O(n)\\) running time, much worse than our original algorithm.
