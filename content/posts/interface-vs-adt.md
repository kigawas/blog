+++
title = "Redefining shared behaviors algebraically"
date = "2020-12-28"
cover = ""
tags = ["Rust", "OOP", "Algebraic Data Type", "Type system"]
description = "Comparison of defining shared behaviors by OOP and algebraic data types"
showFullContent = false
+++

## Overview

Object-oriented programming is a ubiquitous paradigm all over the software development field as it could be found in the most of prevalent languages: C++, C#, Java, JavaScript, Python, etc. [Polymorphism](<https://en.wikipedia.org/wiki/Polymorphism_(computer_science)>), an important characteristic in OOP, is often implemented by sub-typing in a typical OOP language.

```java
interface Animal {
    public void talk();
}

class Cat implements Animal {
    public void talk() {
        System.out.println("meow");
    }
}

class Dog implements Animal {
    public void talk() {
        System.out.println("bowwow");
    }
}
```

The `interface`, or `abstract class` in some languages, here indicates **a protocol of shared behaviors**, normally there's no way to get access to data in the interface, thus, only a few of default behaviors can be implemented in the interface rather than in its subclasses.

In Java, interface methods cannot have body:

```java
interface Shape {
    public void printCoordinates() {
        // error: interface abstract methods cannot have body
        System.out.println(String.format("(%s, %s)", this.x, this.y));
    };
}
```

In C++, interface methods can have body, but they won't work if there're no member variables.

> C++ has no pure "interface", which doesn't allow variables. You can define variables in the abstract class willy-nilly, albeit not recommended.

```cpp
class Shape {
    public:
        virtual void printCoordinates() {
            // error: 'class Shape' has no member named 'x'
            // error: 'class Shape' has no member named 'y'
            printf("(%d, %d)", this -> x, this -> y);
        }
};
```

> It's still possible to use `getters/setters` to interact with data indirectly in the interface, but it's kind of wordy.

## OOP in Rust

Rust, which also [supports OOP](https://doc.rust-lang.org/book/ch17-00-oop.html) to some extent, allows you to define the shared behaviors with `trait` as what you would do in other languages with `interface`.

> Strictly speaking, `trait` in Rust [is different from](https://blog.rust-lang.org/2015/05/11/traits.html) `interface` or `abstract class` in other languages, that's why it has a different name `trait`.
>
> It doesn't matter if you don't know the difference while reading this article.

```rust
struct Dog;
struct Cat;

trait IAnimal {
    fn talk(&self) {}
}

impl IAnimal for Dog {
    fn talk(&self) {
        println!("bowwow");
    }
}

impl IAnimal for Cat {
    fn talk(&self) {
        println!("meow");
    }
}
```

So far so good. But is there a different way doing this? Before we step into the next level, several concepts need to be addressed.

## A prime of algebraic data types

An algebraic data type, or ADT (not to be confused with [abstract data type](https://en.wikipedia.org/wiki/Abstract_data_type)), is a composite type. There are two common algebraic data types: sum types and product types.

Their names look mathematical, but in fact they are just very simple notions. If you know some Python, a sum type in Python is [`Union`](https://docs.python.org/3/library/typing.html#typing.Union), and a product type is [`Tuple`](https://docs.python.org/3/library/typing.html#typing.Tuple).

So why call it "sum/product" type? Say you have a union `A` for `int` or `bool`, `int` can only be `1`, `2` or `3` and `bool` can only be `True` or `False`, how many different objects of `A` can you get? The answer is easy: if `A` is `int`, then you have three choices; if `A` is `bool`, then two; if you add them up, you get five.

This also applies to a product type, e.g. a tuple `(int, bool)`. For the first element, you have three choices and for the second element you have two, so you have six (3 times 2): `(1, True)`, `(2, True)`, `(3, True)`, `(1, False)`, `(2, False)`, `(3, False)`.

Now everything gets clear, for a sum type `S = A | B`, an object of `S` is either `A` or `B` which means the number of different objects is the sum of `A` and `B`; for a product type `P = (A, B)`, the number of different objects is the product of `A` and `B`.

> "The number of different objects" is called "cardinality" mathematically.
>
> If you want to know more about algebraic data types, [this article](https://fsharpforfunandprofit.com/posts/type-size-and-design/) could be a good start.

## ADT in Rust

In Rust, the sum type is `enum` and the product type is `struct`. Unlike the [`enum.Enum`](https://docs.python.org/3/library/enum.html) in Python, Rust's `enum` is powerful enough to be a composite of arbitrary types: `enum E = A | B | C | D ...` and `A`, `B`, `C`, `D` can also be sum types or product types.

```rust
// struct is a product type
// here we have 2^8 * 2^8 different dogs
struct Dog {
    id: u8,
    age: u8,
    // some extra fields can be added
}

struct Cat {
    id: u8,
    age: u8,
}

// sum type: Dog | Cat
enum Animal {
    Dog(Dog),
    Cat(Cat),
}
```

Instead of having a common interface, we implement methods for the sum type `Animal`:

```rust
impl Animal {
    fn talk(&self) {
        match self {
            &Self::Cat(_) => println!("meow"),
            &Self::Dog(_) => println!("bowwow"),
        }
    }

    fn info(&self) {
        match self {
            &Self::Cat(Cat { id, age }) => println!("Cat: id: {}, age: {}", id, age),
            &Self::Dog(Dog { id, age }) => println!("Dog: id: {}, age: {}", id, age),
        }
    }
```

The [`match`](https://doc.rust-lang.org/rust-by-example/flow_control/match.html) here stands for [pattern matching](https://en.wikipedia.org/wiki/Pattern_matching), which is a critical paradigm in functional programming. I'll omit details here, but let me just recap, you can regard it as an advanced version of `switch` in `C` or destructuring assignment in [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) or [Python](https://blog.tecladocode.com/destructuring-in-python).

So, what about the OOP equivalent in Rust?

```rust
trait IAnimal {
    fn talk(&self) {}
    fn info(&self) {}
}

impl IAnimal for Dog {
    fn talk(&self) {
        println!("bowwow");
    }
    fn info(&self) {
        println!("Dog: id: {}, age: {}", self.id, self.age);
    }
}

impl IAnimal for Cat {
    fn talk(&self) {
        println!("meow");
    }
    fn info(&self) {
        println!("Cat: id: {}, age: {}", self.id, self.age);
    }
}
```

Obviously it's more verbose, specifically, we have three `talk`s and `info`s and the implementation scatters here and there. Ever worse, the `interface` implementation is not exhaustive: If we add another animal, say `Sheep`, it won't notice us if we haven't implemented `IAnimal`. But if we choose the sum type, Rust compiler will complain like:

```rust
error[E0004]: non-exhaustive patterns: `&Sheep(_)` not covered
  --> src/main.rs:52:15
   |
43 | / enum Animal {
44 | |     Dog(Dog),
45 | |     Cat(Cat),
46 | |     Sheep(Sheep),
   | |     ----- not covered
47 | | }
   | |_- `Animal` defined here
...
52 |           match self {
   |                 ^^^^ pattern `&Sheep(_)` not covered
   |
   = help: ensure that all possible cases are being handled, possibly by adding wildcards or more match arms
   = note: the matched value is of type `&Animal`

error[E0004]: non-exhaustive patterns: `&Sheep(_)` not covered
  --> src/main.rs:58:15
   |
43 | / enum Animal {
44 | |     Dog(Dog),
45 | |     Cat(Cat),
46 | |     Sheep(Sheep),
   | |     ----- not covered
47 | | }
   | |_- `Animal` defined here
...
58 |           match self {
   |                 ^^^^ pattern `&Sheep(_)` not covered
   |
   = help: ensure that all possible cases are being handled, possibly by adding wildcards or more match arms
   = note: the matched value is of type `&Animal`

error: aborting due to 2 previous errors
```

## Conclusion

In this article, we revisited a common OOP polymorphism paradigm, which is defining shared behaviors by sub-typing interfaces, and came up with another strategy to achieve the same goal via ADTs. By leveraging the type system and pattern matching of Rust, we are capable of defining shared behaviors with less code and more confidence.

> ADT is not a silver bullet, but it's a functional way of implementing polymorphism and can be useful in some scenarios.

The Rust code above can be found in the [Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=c8640c555fce2ec520d23d446b686d3e).
