---
title: Y-combinator Python实现
excerpt: 用Python实现Y-combinator
---

函数式编程里面，著名的[Y-combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator#Fixed_point_combinators_in_lambda_calculus)的lambda calculus表示是`Y := λf.(λx.(f (x x)) λx.(f (x x)))`。通过使用Y-combinator，我们可以在**不定义函数**的情况下实现递归。本文将介绍如何以Python实现这个看上去有点复杂的lambda表达式。

首先看最外面，lambda表达式接收一个参数f，返回`λx.(f (x x))`调用[^1]`λx.(f (x x))`的结果。

对于第一部分，lambda表达式接受一个参数x，返回f调用x(x)的结果：

```python
lambda x: f(x(x))
```

第二部分和第一部分相同，也是`lambda x: f((x)x)`，组合起来得到：

```python
(lambda x: f(x(x)))(lambda x:f(x(x)))
```

回到最外层，有

```python
Y = lambda f: (lambda x: f(x(x)))(lambda x: f(x(x)))
```

然后我们写一个计算阶乘的lambda表达式并尝试运行：

```python
fac = lambda f: lambda n: 1 if n <= 1 else n * f(n - 1)
print(Y(fac)(5))
```

得到一个错误`RuntimeError: maximum recursion depth exceeded`，原因是Python的函数传参数的方式是call-by-value[^2]的，导致了无限递归。

> call-by-value是对像这样的函数调用`(lambda y:(lambda x:x)(y))(1)`，先计算参数变成`(lambda x:x)(1)`。call-by-name[^3]则是先展开内部的表达式，得到`(lambda y:y)(1)`。

如果要让Python的某个lambda表达式**不立即求值**，我们可以用函数包裹起来，例如：

```python
another_x = lambda :x
```

这样，只有在another_x调用的时候才会求出值。

> 这个过程在lambda演算里面叫做eta变换。

回到上面的Y-combinator，我们需要对里面的表达式延缓求值，也就是对第二个表达式里面的f(x(x))套一层函数:

```python
Y = lambda f: (lambda x: x(x))(lambda x: f(lambda *args: x(x)(*args)))
print(Y(fac)(5))  # 120
```

注意，这里我们通过把x(x)藏到lambda表达式内部，使得参数传进来之前，x(x)不会被求值。这样，我们就避免了无限递归。

> 经过eta变换的Y-combinator叫做Z-combinator。

在lambda演算里面，上面的Y和fac都是不必要（严格来说，是不允许出现）的，我们可以得到最终版：

```python
print(
  (lambda f: (lambda x: x(x))
    (lambda x: f(lambda *args: x(x)(*args))))
  (lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5)) # 120
```

不过，如果允许递归，我们有一种更简单的方法定义Y-combinator：

```python
Y = lambda f: f(Y(f)) # 根据定义
Y = lambda f: lambda *args: f(Y(f))(*args) # 进行eta变换，以避免无限递归
print(Y(lambda f: lambda n: 1 if n <= 1 else n * f(n - 1))(5)) # 120
```

[^1]: 准确的术语是应用（apply）

[^2]: 又叫eager evaluation或者应用序

[^3]: 又叫lazy evaluation或者正则序

