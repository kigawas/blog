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

In terms of the functional programming field, [Y-combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator#Fixed_point_combinators_in_lambda_calculus) is expressed on the lambda calculus format: `λf. (λx. (f (x x))) (λx. (f (x x)))`.

With Y-combinator, we can implement recursion **without defining functions explicitly**. As you may know, recursion is equivalent to iteration, thus we can assure that lambda calculus is as powerful as Python (lambda calculus is Turing-complete per se), even it only has three rules compared to some sophisticated languages.

In this article, we'll discuss how to do it in Python.

## How to implement Y-combinator

Let's break the daunting lambda expression into smaller pieces. For simplicity, it can be split into two parts: inner and outer.

First, let's have a look at the outer side. It's actually accepting an argument `f` and returning the result of `λx. (f (x x))` invoked with the argument `λx. (f (x x))`.

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

So, how do we put them together? Actually, `lambda x: f(x(x))` is a function, so we'll invoke this function whose argument is itself. I know it sounds cryptic, but for now we just try:

```python
( lambda x: f(x(x)) )  ( lambda x: f(x(x)) )
#     function                argument
```

Back to the outside, we'll get

```python
Y = lambda f: (lambda x: f(x(x)))(lambda x: f(x(x)))
```

## How to use Y-combinator

You may want to ask, how should we use this weird thing to implement recursion **without** defining any function? Well, let's take the factorial calculation as an example.

### First try

Normally, you might write something like:

```python
def fac(n):
    return 1 if n <= 1 else n * fac(n - 1)

print(fac(5))  # 120
```

However, what if you cannot name a function? Let's recap the rules of lambda calculus, we may notice there's no rule for naming or defining a function. So we have to find another way to emulate that.

One way is to substitute the recursion call with an argument representing the function:

```python
fac = lambda f: lambda n: 1 if n <= 1 else n * f(n - 1)
```

You may wonder how to call it like in Python, in fact, this is why we need Y-combinator.

Let's try applying it:

```python
print(Y(fac)(5))
```

Hmm...You should see:

```python
RecursionError: maximum recursion depth exceeded
```

### The evaluation problem we saw

Wait, what is this? I'm sorry I forgot to mention you'd encounter with the error. It has something to do with the way how a programming language evaluates arguments passed into a function.

Say you have functions like:

```python
def add1(x):
    return x + 1

def mul(x, y):
    return x * y

print(mul(add1(0), add1(1))) # 2
```

Let's ponder: When Python is calling `mul(add1(0), add1(1))`, how does Python evaluate its arguments? Some of you may answer quickly: it'll evaluate `add1(0)` and `add1(1)` first, and then `mul(1, 2)`!

Yes, that's almost right. Let's dig into something else. What if we call a function like:

```python
(lambda y: (lambda x: x)(y))(1)
```

Obviously you'll see `1`, but in what order Python evaluates its arguments?

As mentioned above, Python will evaluate arguments eagerly, so it will become `(lambda x: x)(1)`, then `1`.

> This is called applicative order.

This is why we got `RecursionError`. Let's review the `lambda f: (lambda x: f(x(x)))(lambda x: f(x(x)))` and call it with a simple `i = lambda f: lambda x: x`:

```python
(lambda f: (lambda x: f(x(x)))(lambda x: f(x(x))))(i)
# will be evaluated to:
(lambda x: i(x(x)))(lambda y: i(y(y)))  # let's rename to avoid ambiguity
# endless recursion:
i( lambda y: i(y(y)) )( lambda y: i(y(i(y(i(y(...)))))) )
```

Here when Python evaluates the argument `lambda y: i(y(y))`, it'll recursively call `y` as a function over and over again. Yet, how can we cease the endless recursion? Or how can we only recursively get it called just once or twice or thrice?

In effect, we'd like to evaluate the function itself **before** any arguments get substituted.

> This is called normal order.

Evaluation strategy is just a choice, and some typical functional languages behave like the normal order, however unfortunately, Python doesn't follow this style.

### The way to delay evaluation

In order to delay the evaluation of an expression, what should we do? Say we don't want Python to evaluate `3 + 3` right now, we can wrap them into a function:

```python
another_f = lambda: 3 + 3
```

Thus, the `3 + 3` will only be evaluated when the function gets invoked as `another_f()`.

> It is called "[eta conversion](https://wiki.haskell.org/Eta_conversion)" in lambda calculus, which can be regarded as saving some code for future execution.

### Z-Combinator

Back to the Y-combinator we wrote, the problem comes when Python evaluates our arguments too early, so we need to delay the evaluation of `x(x)` in the argument lambda:

```python
Y = lambda f: (lambda x: f(x(x)))(lambda x: f(lambda *args: x(x)(*args)))
print(Y(fac)(5))  # 120
```

> Eta-converted Y-combinator is called Z-combinator.

It works! Let's examine it step by step.

```python
# fac = lambda f: lambda n: 1 if n <= 1 else n * f(n - 1)
fy = lambda y: fac(lambda *args: y(y)(*args))
fy1 = lambda y1: fac(lambda *args: y1(y1)(*args))
# (lambda x: fac(x(x)))(lambda y: fac(lambda *args: y(y)(*args)))
# will be evaluated to
fac(fy(fy1))
# expand fy
fac(fac(lambda *args: fy1(fy1)(*args)))
# here the fy1 is captured into the closure
fac(lambda n: 1 if n <= 1 else n * (lambda *args: fy1(fy1)(*args))(n - 1))
```

So if you call `fy(fy)` or `fy(fy1)`, it'll not get trapped into endless recursion since inside `fy`, the `y` will not be invoked immediately. Instead, it'll just return the closure `lambda *args: y(y)(*args)`. Let's continue:

```python
ffac = lambda n: 1 if n <= 1 else n * (lambda *args: fy1(fy1)(*args))(n - 1)
# simplify
fac(ffac)
# expand once again
(lambda n: 1 if n <= 1 else n * ffac(n - 1))
# what abount n == 3?
(lambda n: 1 if n <= 1 else n * ffac(n - 1))(3)
# in else branch
3 * ffac(2)
# expand ffac(2)
3 * 2 * (lambda *args: fy1(fy1)(*args))(1)
# eliminate args
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

The complicated part `zetafy1(n - 1)` disappeared like magic!

> Every magic can be explained. The recursion stops when falling into the basic case `n <= 1`.
> 
> More theoretically speaking, Y-combinator is one of [fixed-point combinators](https://en.wikipedia.org/wiki/Fixed-point_combinator), defined as `Y(f) = f(Y(f))`.
> By this definition, we have `Y(fac)(n) = fac(Y(fac))(n)`. Recap `fac = lambda f: lambda n: 1 if n <= 1 else n * f(n - 1)`, to simplify notation, let `f(n) = Y(fac)(n)`.
> 
> 1. For `n = 1`, `f(1) = Y(fac)(1) = fac(Y(fac))(1) = 1`.
> 2. For `n > 1`, `f(n) = Y(fac)(n) = fac(Y(fac))(n) = n * Y(fac)(n-1)`.
> 
> Because `f(n-1) = Y(fac)(n-1)`, we have `f(n) = n * f(n-1) = n * (n-1) * ... * f(1) = n!` - this is exactly what we wanted to calculate.

### The final version without variables

Remember we cannot define variables in lambda calculus? So we just remove variables and reach the final version:

```python
print(
  (lambda f: (lambda x: f(x(x)))
    (lambda x: f(lambda *args: x(x)(*args))))
  (lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5))  # 120
```

How neat!

### Other variants

#### Recursive

However, if we are not that strict, we can define Y-combinator simply with recursion like:

```python
# By fixed-point definition
# Y = lambda f: f(Y(f))
Z = lambda f: lambda *args: f(Z(f))(*args)  # eta converted to avoid endless recursion
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

You'll get the same result. If you'd like to know why, evaluate it step by step.

## Conclusion

Lambda calculus is concise, but it's profoundly connected with the essence of calculation. With Y-combinator, we are able to implement recursion as in Python without defining any variable and even [emulate a script language virtual machine](https://github.com/kigawas/computation-py/tree/master/computation/interpreter).
