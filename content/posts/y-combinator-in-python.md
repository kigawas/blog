+++
title = "Y-combinator in Python"
date = "2018-11-22"
cover = ""
tags = ["Python", "functional programming"]
description = "A brief introduction to Y-combinator in Python"
showFullContent = false
+++

## A brief introduction to lambda calculus
[Lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus) is a language (formal system) to abstract and express calculation itself under only three concise rules.

| Syntax  | Name        | Description                                                     |
| ------- | ----------- | --------------------------------------------------------------- |
| x       | Variable    | Representing some value                                         |
| (λx. M) | Abstraction | Defining a function, M is also an expression of lambda calculus |
| (M N)   | Application | Calling a function, M and N are expressions of lambda calculus  |

> In lambda calculus, we can say "applying/calling/invoking a function" interchangeably.

## What is Y-combinator

In terms of the functional programming field, the famous [Y-combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator#Fixed_point_combinators_in_lambda_calculus) is expressed on the lambda calculus format: `λf. (λx. (f (x x))) (λx. (f (x x)))`.

With Y-combinator, we can implement recursion **without defining functions explicitly**. As you may know, recursion is equivalent to iteration, thus we can assure that lambda calculus is as powerful as Python (lambda calculus is Turing-complete per se), even it only has three rules compared to some sophisticated languages.

In this article, we'll discuss how to do it in Python.

## How to implement Y-combinator

Let's break the daunting lambda expression into smaller pieces. For simplicity, it can be split into two parts: inner and outer.

First, let's have a look at the outer side. It's actually accepting an argument `f` and returning the result of  `λx. (f (x x))` invoked with the argument `λx. (f (x x))`.

### Inner lambda

If you have some experience of Lisp-family languages, you may be familiar with something like `(f (x x))`, which is also called "[S-expression](https://en.wikipedia.org/wiki/S-expression)". For `(f a)`, it means that we are calling a function `f`, and it takes an argument `a`, and the `a` can also be another S-expression like `(x x)`.

So in Python, it's quite straight:

```python
lambda x: f(x(x))
```

The latter `λx. (f (x x))` is definitely the same:

```python
lambda x: f(x(x))
```

### Outer lambda

So, how do we put them together? Actually, `lambda x: f(x(x))` is a function, so we'll invoke this function whose argument is itself. I know it sounds strange, but for now we just try it:

```python
( lambda x: f(x(x)) )  ( lambda x: f(x(x)) )
#     function                argument
```

Back to outside, we'll get

```python
Y = lambda f: (lambda x: f(x(x)))(lambda x: f(x(x)))
```

## How to use Y-combinator

You may want to ask, how should we use this weird thing to implement recursions __without__ defining any function? Well, let's start from the factorial calculation.

### First try

Normally, you might write something like:

```python
def fac(n):
    return 1 if n <= 1 else n * fac(n - 1)

print(fac(5))  # 120
```

However, what if you cannot name a function? In lambda calculus, we cannot name a function as usual. Then here comes Y-combinator.

First, let's convert the factorial function into lambda by adding an outside argument:

```python
fac = lambda f: lambda n: 1 if n <= 1 else n * f(n - 1)
```

> Just need to substitute the function name with an argument.

Then, we apply Y-combinator to it:

```python
print(Y(fac)(5))
```

Hmm...You should see:

```python
RecursionError: maximum recursion depth exceeded
```

### The evaluation problem we saw

Wait, what is this? I'm sorry I forgot to mention this. It has something to do with the way how a programming language calculates arguments passed into a function.

Let's say you have functions like:

```python
def add1(x):
    return x + 1

def mul(x, y):
    return x * y

print(mul(add1(0), add1(1))) # 2
```

Let's ponder: When Python is calling `mul(add1(0), add1(1))`, how does Python calculate its arguments? Some of you may answer quickly: it'll calculate `add1(0)` and `add1(1)` first, and then `mul(1, 2)`!

Yes, that's __almost__ right. Let's dig into something else. What if we call a function like:

```python
(lambda y: (lambda x: x)(y))(1)
```
Obviously you'll see `1`, but in what order Python evaluates its arguments?

As mentioned above, Python will evaluate arguments eagerly, so it will become  `(lambda y: y)(1)`, then `1`.

```python
In [13]: (lambda y: (lambda x: x)(y))
Out[13]: <function __main__.<lambda>(y)>  # here the argument is y
```

> This is called applicative order.

This is why we got `RecursionError`. Let's review the `lambda f: (lambda x: f(x(x)))(lambda x: f(x(x)))` and call it with a simple `i = lambda f: lambda x: x`:

```python
(lambda f: (lambda x: f(x(x)))(lambda x: f(x(x))))(i)
# will be evaluated to:
(lambda x: i(x(x)))(lambda y: i(y(y))) # let's rename
# endless recursion:
i((lambda y: i(y(y)))(lambda y: i(y(i(y(i(y(...))))))))
```

Here when Python evaluates the argument `lambda y: i(y(y)`, it'll recursively call `y` as a function over and over again. Yet, how can we stop the recursion? Or how can we only recursively get it called just once or twice or thrice?

There is an answer. But before explain that, let's get back to the simple function call `(lambda y: (lambda x: x)(y))(1)`. How about to delay the inner function's evaluation:  `(lambda x: x)(1)`?

By this way, we evaluated the function before any arguments get substituted. In fact, some typical functional languages behave in this way, however unfortunately, Python doesn't follow this style.

### The way to delay evaluation

So if we want to delay the evaluation of an argument, what should we do? Say we don't want it to evaluate `3+3` right now, we can wrap them into a function:

```python
another_f = lambda: 3 + 3
```

Thus, the `3+3` will only be evaluated when the function gets invoked as `another_f()`.

> It is called "[eta conversion](https://wiki.haskell.org/Eta_conversion)" in lambda calculus.
> Eta-converted Y-combinator is called Z-combinator。

### Z-Combinator

Back to the Y-combinator we wrote, the problem comes when Python evaluates our arguments too early, so we need to delay the calculation of `x(x)` in the argument lambda:

```python
Y = lambda f: (lambda x: f(x(x)))(lambda x: f(lambda *args: x(x)(*args)))
print(Y(fac)(5))  # 120
```

Let's do it step by step.

```python
fy = lambda y: fac(lambda *args: y(y)(*args))
fy1 = lambda y1: fac(lambda *args: y1(y1)(*args))
# (lambda x: i(x(x)))(lambda y: i(lambda *args: y(y)(*args)))
# will be evaluated to
fac(fy(fy1))
# one more step
fac(fac(lambda *args: fy1(fy1)(*args)))
# here the fy1 is captured into the closure
# fac = lambda f: lambda n: 1 if n <= 1 else n * f(n - 1)
fac(lambda n: 1 if n <= 1 else n * (lambda *args: fy1(fy1)(*args))(n - 1))
```

And if you call `fy(fy)` or `fy(fy1)`, it'll not get trapped into endless recursion since inside `fy`, the `y` will not be invoked immediately. Instead, it'll just return the closure `lambda *args: y(y)(*args)`. Let's continue:

```python
# eliminate *args
ffac = lambda n: 1 if n <= 1 else n * fy1(fy1)(n - 1)
# simplify fac(lambda n: 1 if n <= 1 else n * fy1(fy1)(n - 1))
fac(ffac)
# substitute once again
(lambda n: 1 if n <= 1 else n * ffac(n - 1))
# what abount n == 3?
(lambda n: 1 if n <= 1 else n * ffac(n - 1))(3)
# in else branch
3 * ffac(2)
# expand ffac
3 * 2 * fy1(fy1)(1)
# substitute
3 * 2 * fac(lambda *args: fy1(fy1)(*args))(1)
# just for simplicity
zetafy1 = lambda *args: fy1(fy1)(*args)
3 * 2 * fac(zetafy1)(1)
# expand fac
3 * 2 * (lambda n: 1 if <= 1 else n * zetafy1(n - 1))(1)
# in if branch
3 * 2 * 1  # == 6
```

The complicated part disappeared like magic!

### The final version without variables

Remember we cannot define variables in lambda calculus? So we just remove the variable name and get the final version:

```python
print(
  (lambda f: (lambda x: f(x(x)))
    (lambda x: f(lambda *args: x(x)(*args))))
  (lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5))  # 120
```

How neat!

### Other variants

#### Recursive
However, if we are not that strict, we can define Y-combinator in a simpler way with recursion like:

```python
# By fixed-point definition
# Y = lambda f: f(Y(f))
Z = lambda f: lambda *args: f(Z(f))(*args)  # eta converted to dodge recursion bug
print(Z(lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5))  # 120
```

#### First `f` eliminated

You might have noticed that `fac(ffac)` is an overkill. What if we remove an `f`?

```python
print(
  (lambda f: (lambda x: x(x))
    (lambda x: f(lambda *args: x(x)(*args))))
  (lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5))  # 120
```

## Conclusion

Lambda calculus is concise, but it's profoundly connected with the essence of what calculation is. With Y-combinator, we are able to implement recursion as in Python without defining any variable and even [emulate a script language virtual machine](https://github.com/kigawas/computation-py/blob/master/computation/tests/interpreter/test_interpreter.py#L39).
