---
title: Y-combinator Python实现
excerpt: 用Python实现Y-combinator
---

[Y-combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator#Fixed_point_combinators_in_lambda_calculus)的lambda演算表示：`Y := λf.(λx.(f (x x)) λx.(f (x x)))`如何用Python实现？

首先看最外面，lambda表达式接收一个参数f，返回`λx.(f (x x))`调用（准确的术语是应用）`λx.(f (x x))`的结果。

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
fac = lambda f: lambda n: 1 if n == 1 else n * f(n - 1)
print(Y(fac)(5))
```

得到一个错误`RuntimeError: maximum recursion depth exceeded`，原因是Python的函数传参数的方式是call-by-value的（又叫eager evaluation）,导致了无限递归。

> eager evaluation是对像这样的函数调用`(lambda y:(lambda x:x)(y))(1)`，先算内部的表达式变成`(lambda y:y)(1)`。 lazy evaluation相反（和call-by-value相对，叫call-by-name），先算外部的表达式，得到`(lambda x:x)(1)`

如果要让Python的某个lambda表达式不立即求值，我们可以用函数包裹起来，例如：

```python
another_x = lambda :x
```

这样，只有在another_x调用的时候才会求出值，这个过程在lambda演算里面叫做eta变换。

回到上面的Y-combinator，我们需要对里面的表达式延缓求值，也就是第二个表达式里面的f(x(x)):

```python
Y = lambda f: (lambda x: x(x))(lambda x: f(lambda *args: x(x)(*args)))
print(Y(fac)(5))  #120
```

在lambda演算里面，上面的Y和fac都是不必要（严格来说，是不允许出现）的，我们可以得到最终版：

```python
print(
  (lambda f: (lambda x: x(x))
    (lambda x: f(lambda *args: x(x)(*args))))
  (lambda f: lambda n: 1 if n == 1 else n * f(n - 1))(5)) #120
```
