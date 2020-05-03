---
layout: post
title: "Newton's square root method in Scheme"
date: 2020-05-03
---

A scheme implementation of Newton's square root method from SICP 1.1.7.

{% highlight scheme %}
(define (sqrt x)
  (define (sqrt-iter guess)
    (if (good-enough? guess)
        guess
        (sqrt-iter (improve guess)))
    )

  (define (improve guess)
    (average guess (/ x guess)))

  (define (average x y)
    (/ (+ x y) 2))

  (define (good-enough? guess)
    (< (abs (- x (square guess))) .001))

  (define (square x)
    (* x x))

  (define (abs x)
    (if (< x 0)
        (- x)
        x))
  
  (sqrt-iter 1)
 )
{% endhighlight %}
