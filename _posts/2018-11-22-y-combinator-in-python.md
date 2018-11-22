---
title: Y-combinator in Python
excerpt: A simple introdution to Y-combinator in Python
---

## What is Y-combinator

In the functional programming field, the famous [Y-combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator#Fixed_point_combinators_in_lambda_calculus) is expressed in lambda calculus like `Y := lambda f.(lambda x.(f (x x)) lambda x.(f (x x)))`. Y-combinator can enable us to implement recursion **without defining functions** explicitly. In this article, we'll discuss how to implement it in Python.

## How to implement Y-combinator

Let's break it into small pieces.

First, we take a look on the outer side. It's actullay accepting an argument `f` and return the function `lambda x.(f (x x))` invoking (or "applying" as a FP jargon) with the argument `lambda x.(f (x x))`'s result.

### Inside lambda

If you have Lisp experience, you may be familiar with something like `(f (x x))`. It is also called "[S-expression](https://www.wikiwand.com/en/S-expression)". For something like `(f a)`, it means that we are calling a function `f`, and it takes an argument `a`, ane the `a` can be another S-expression like `(x x)`.

So in python, it's simple:

```python
lambda x: f(x(x))
```

And the latter `lambda x.(f (x x))` is definitely the same:

```python
lambda x: f(x(x))
```

### Outside lambda

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

You may want to ask, how should we use this weird thing to implement recursions without defining any function? Let's start from the factorial calculation.

### First trying

In normal way, you will write it like:

```python
def fac(n):
    return 1 if n <= 1 else n * fac(n - 1)

print(fac(5))  # 120
```

But what if you cannot define or name a function? In lambda calculus, we cannot define variable or name a function as usual. And then here comes Y-combinator.

First, let's convert the factorial function into lambda:

```python
fac = lambda f: lambda n: 1 if n <= 1 else n * f(n - 1)
```

And then, we use Y-combinator:

```python
print(Y(fac)(5))
```

And we get:

```python
RecursionError: maximum recursion depth exceeded
```

### The evaluation problem we get

Wait, what is this? I'm sorry I forgot to mention this. This has something to do with the way how we calculate arguments when passing them into the function.

Let's say you call function like:

```python
def add1(x):
    return x + 1

def mul(x, y):
    return x * y

print(mul(add1(0), add1(1))) # 2
```

Let's think about: When Python is calling `mul()`, how does python calculate it's arguments? Some of you may raise your hand: it'll calculate `add1(0)` and `add1(1)` first, and then `mul(1, 2)`.

Yes, that's almost right. Let's think a little deeper. What if we call function like:

```python
(lambda y: (lambda x: x)(y))(1)
```

As mentioned above, Python will calculate argument first, so it will become `(lambda 1: (lambda x: x)(1)`, then `(lambda x: x)(1)`.

But there is another way. So how about we call the inner function first? Let's keep the 1 outside: `(lambda y: (lambda y: y)())(1)` then `(lambda y: y)(1)`.

By this way, we simplify the function before we actually do any calculation. In fact, some functional languages do like this.

### The way to delay evaluation

So if we want to delay the evaluation of an argument, what should we do? Say we don't want it to calculate `3+3` right now, we can put them into a function:

```python
another_f = lambda : 3 + 3
```

And the `3+3` will only be evaluated when we call it like `another_f()`.

> It is called "[eta conversion](https://www.wikiwand.com/en/Lambda_calculus#/%CE%B7-conversion)" in lambda calculus.

### Z-Combinator

Back to the Y-combinator we wrote, the problem happened when Python evaluate our argument too early, so we need to delay the argument to the latter lambda inside:

```python
Y = lambda f: (lambda x: x(x))(lambda x: f(lambda *args: x(x)(*args)))
print(Y(fac)(5))  # 120
```

By this way, until the argument is passed into the latter lambda, `x(x)` will not be evaluated.

> Eta-converted Y-combinator is called Z-combinator。

### Final version without variables

Remember we cannot define variables in lambda calculus? So we just remove the variable name and get the final version:

```python
print(
  (lambda f: (lambda x: x(x))
    (lambda x: f(lambda *args: x(x)(*args))))
  (lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5))  # 120
```

### A simpler version

However, if we are not that strict, we can define Y-combinator in a simpler way like:

```python
Y = lambda f: f(Y(f))  # By definition
Z = lambda f: lambda *args: f(Z(f))(*args)  # eta converted
print(Z(lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5))  # 120
```
