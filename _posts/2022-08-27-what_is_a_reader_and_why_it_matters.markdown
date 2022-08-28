---
layout: post
title: A primer on Readers
categories: [Scala 2.13, Functional programming]
description: Without mentioning monads!
---
### The dream
Imagine we have functions:

```Scala
def f: A => B = ???
def g: B => C = ???
def h: C => D = ???
```

The implementation may be whatever we want it to be, these functions might parse some data, perform calculations, format strings et cetera. Now, often we want to chain multiple operations together &mdash; ie. parse a JSON with sales data, then extract monthly sales figures and finally output an average. After all, coding is all about gluing blocks together to create a useful program.
We could, of course, write `h(g(f(x)))` each time we want the whole functionality, but if it's used frequently, giving it a proper name is a good idea.
Scala gives us many different ways of gluing functions together, but I'm going to stick to the `andThen` from `Function1` trait &mdash; because it behaves almost like the mathematical `∘` operator, just in reverse order[^compose]. With it creating a function `i: A => D` is as simple as writing:

```Scala
def i: A => D = f andThen g andThen h
```

### The reality
This kind of composition, obviously, works only for functions with one argument &mdash; with multi-argument functions we quickly run into a world of trouble. Quite unfortunate, since the vast majority of those we encounter on a daily basis need more than one input. But what if all of our functions look like this?:

```Scala
def f: E => A => B = ???
def g: E => B => C = ???
def h: E => C => D = ???
```

This is again a common thing &mdash; oftentimes we need to pass some sort of a context (or other dependency) to our methods. It would be useful to be able to compose them, even if just to save us some writing.
They share the type of the first argument &mdash; it must have made our task easier. What do we need to do to be able to compose them in a sensible way?

## The promise
First of all, let's simplify things a bit and imagine a function that magically reads some value `V` from an environment `E`. It's going to have a type `E => V`. We're going to call it `Reader` for precisely that reason, and we're going to wrap it in a case class for convenience:

```Scala
case class Reader[E, V](read: E => V)
```

We're halfway there, we just need to notice that function parameters should be swappable (with possible minor tweaks to the logic inside &mdash; but usually we don't depend on their order anyway). Then we can rewrite our `f`, `g`, and `h` as:

```Scala
def f: A => E => B = ???
def g: B => E => C = ???
def h: C => E => D = ???
```

There's a lot of `Reader`s just waiting in line. We can now rewrite our functions again:

```Scala
def f: A => Reader[E, B] = ???
def g: B => Reader[E, C] = ???
def h: C => Reader[E, D] = ???
```

### The solution
Now we just need to compose these somehow. First of all, let's give our Reader a helper method:

```Scala
case class Reader[E, V](read: E => V) {
  def pipeTo[W](other: V => Reader[E, W]): Reader[E, W] = Reader { env =>
    val result1 = read(env)
    other(result1).read(env)
  }
}
```

Then we can make ourselves a helper:

```Scala
object Reader {
  implicit class ReaderOps[E, A, B](f1: A => Reader[E, B]) {
    def andThenR[C](f2: B => Reader[E, C]): A => Reader[E, C] =
      a => f1(a).pipeTo(f2)
  }
}
```

Et voilà, we can now write this:

```Scala
def i: A => Reader[E, D] = f andThenR g andThenR h
```

And it works!

### A bit of black magic
We can do one more trick. Let's first refactor the Reader to be able to use Scala's wonderful for-comprehensions by renaming `pipeTo` to `flatMap` and adding a `map` method[^Function1]:

```Scala
case class Reader[E, V](read: E => V) {
  def map[W](f: V => W): Reader[E, W] = Reader(env => f(read(env)))
  def flatMap[W](f: V => Reader[E, W]): Reader[E, W] = Reader { env =>
    val result1 = read(env)
    f(result1).read(env)
  }
}
```

and then define:

```Scala
def ask[E]: Reader[E, E] = Reader(identity)
```

Now we can write stuff like this:

```Scala
val readIncrementAndStringify = for {
  intFromEnvironment <- ask[Int]
  incremented = intFromEnvironment + 1
} yield s"$incremented"
```

This yields a `Reader[Int, String]` value, which we can run afterwards to retrieve the final `String` when we need it &mdash; but the logic itself looks like pulling `Int`s out of thin air. Beautiful, isn't it?

## The use
One might ask: why does it matter at all? Aside of the obvious, this can serve as a dependency injection mechanism &mdash; functional and type safe. The `E` we plug into the `Reader` can be whatever we want &mdash; a default value for a given type, an input to a whole computation, a validator or a service with various utilities. We can even stuff multiple things into it!

```Scala
type Env = (UserDeserializer, UserValidator)

def createUser(input: String): Reader[Env, Option[User]] = for {
  (deserializer, validator) <- ask[Env] //assuming we use better-monadic-for plugin. Who doesn't?
  deserialized = deserializer.deserialize(input)
  isCorrect = validator.validate(deserialized)
} yield if (isCorrect) Some(deserialized) else None
```

This idea is quite similar to how ZIO handles dependency injection[^ZIO], and how it's often done in the wide functional world. Of course this implementation is quite limited &mdash; but its purpose is to give the reader (pun intended) an intuition on how `Reader`s work.

So to sum up: `Reader` is simply a functional programming pattern for composing two-argument functions that share one of its arguments, and it can be used for anything from passing common values around with less clutter to full-blown dependency injection. Go forth and read!

---

[^compose]: Scala also defines the `compose` method that behaves exactly like mathematical function composition, but `andThen` is usually easier to follow, as it reflects the exact order of function application.

[^Function1]: Now it becomes clearer why we decided to wrap our `E => V` function instead of relying on plain `Function1` &mdash; we want to have `map` and `flatMap` methods available.

[^ZIO]: ZIO defines the `ZIO[R, E, A]` type, which uses the `R` type parameter to pass dependencies around in a `Reader`-esque fashion. For detailed explanation see:
    - [ZIO documentation](https://zio.dev/)
    - [Wonderful talk by Kit Langton](https://youtu.be/yXcqjQ7Kcwk)
    - [Adam Warski's post on structuring ZIO 2 applications](https://softwaremill.com/structuring-zio-2-applications/)

