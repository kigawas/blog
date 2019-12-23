---
title: Y-combinator in Python
excerpt: A simple introduction to Y-combinator in Python
---

## What is Y-combinator

In the functional programming field, the famous [Y-combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator#Fixed_point_combinators_in_lambda_calculus) is expressed in the lambda calculus format: `Y := lambda f.(lambda x.(f (x x)) lambda x.(f (x x)))`. With Y-combinator, we can implement recursion **without defining functions explicitly**. In this article, we'll discuss how to do it in Python.

For simplicity, the calculus format can be split into two parts: inner/outer lambda.

## How to implement Y-combinator

Let's break it into small pieces.

First, we take a look on the outer side. It's actually accepting an argument `f` and return the function `lambda x.(f (x x))` invoking (or "applying" as a FP jargon) with the argument `lambda x.(f (x x))`'s result.

### Inner lambda

If you have Lisp experience, you may be familiar with something like `(f (x x))`, which is also called "[S-expression](https://en.wikipedia.org/wiki/S-expression)". For `(f a)`, it means that we are calling a function `f`, and it takes an argument `a`, and the `a` can also be another S-expression like `(x x)`.

So in Python, it's quite straight:

```python
lambda x: f(x(x))
```

And the latter `lambda x.(f (x x))` is definitely the same:

```python
lambda x: f(x(x))
```

### Outer lambda

So, how do we put them together? Actually, `lambda x: f(x(x))` returns a function, so we'll call this function whose argument is itself. I know it sounds strange, but for now we just do it:

```python
( lambda x: f(x(x)) )  ( lambda x: f(x(x)) )
#     function                argument
```

And back to outside, we'll get

```python
Y = lambda f: (lambda x: f(x(x)))(lambda x: f(x(x)))
```

## How to use Y-combinator

You may want to ask, how should we use this weird thing to implement recursions __without__ defining any function? Well, let's start from the factorial calculation.

### First trying

In normal way, you might write something like:

```python
def fac(n):
    return 1 if n <= 1 else n * fac(n - 1)

print(fac(5))  # 120
```

But what if you cannot define or name a function? In lambda calculus, we cannot define variable or name a function as usual. And then here comes Y-combinator.

First, let's convert the factorial function into lambda by adding an outside argument:

```python
fac = lambda f: lambda n: 1 if n <= 1 else n * f(n - 1)
```

And then, we apply Y-combinator:

```python
print(Y(fac)(5))
```

And we get:

```python
RecursionError: maximum recursion depth exceeded
```

### The evaluation problem we get

Wait, what is this? I'm sorry I forgot to mention this. It has something to do with the way how we calculate arguments passed into a function.

Let's say you have functions like:

```python
def add1(x):
    return x + 1

def mul(x, y):
    return x * y

print(mul(add1(0), add1(1))) # 2
```

Let's ponder: When Python is calling `mul(add1(0), add1(1))`, how does python calculate it's arguments? Some of you may raise your hand: it'll calculate `add1(0)` and `add1(1)` first, and then `mul(1, 2)`.

Yes, that's almost right. Let's dig something else. What if we call a function like:

```python
(lambda y: (lambda x: x)(y))(1)
```

As mentioned above, Python will calculate arguments first, so it will become `(lambda 1: (lambda x: x)(1)`, then `(lambda x: x)(1)`.

But there is another way. So how about we call the inner function first? Let's keep the 1 outside: `(lambda y: (lambda y: y)())(1)` then `(lambda y: y)(1)`.

By this way, we simplify the function before we actually do any calculation. In fact, some functional languages behave this way.

### The way to delay evaluation

So if we want to delay the evaluation of an argument, what should we do? Say we don't want it to calculate `3+3` right now, we can wrap them into a function:

```python
another_f = lambda : 3 + 3
```

And the `3+3` will only be evaluated when we call it like `another_f()`.

> It is called "[eta conversion](https://wiki.haskell.org/Eta_conversion)" in lambda calculus.

### Z-Combinator

Back to the Y-combinator we wrote, the problem comes when Python is evaluating our arguments too early, so we need to delay the calculation of `x(x)` in the argument lambda:

```python
Y = lambda f: (lambda x: x(x))(lambda x: f(lambda *args: x(x)(*args)))
print(Y(fac)(5))  # 120
```

By this way, `x(x)` will not be evaluated util `5` passed into it.

> Eta-converted Y-combinator is called Z-combinator。

### Final version without variables

Remember we cannot define variables in lambda calculus? So we just remove the variable name and get the final version:

```python
print(
  (lambda f: (lambda x: x(x))
    (lambda x: f(lambda *args: x(x)(*args))))
  (lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5))  # 120
```

This is very neat!

### A simpler version

However, if we are not that strict, we can define Y-combinator in a simpler way with recursion like:

```python
Y = lambda f: f(Y(f))  # By definition
Z = lambda f: lambda *args: f(Z(f))(*args)  # eta converted
print(Z(lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5))  # 120
```
