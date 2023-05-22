---
layout: post
title: Turning type-level magic into money
categories: [Scala 3, Type-level magic, Functional programming]
description: Saves you some mistakes, for sure.
---
### Money, money, money
Every single one of us handles money-related code at some point in time &mdash; be it because of the particular domain we work on, our own convenience, or just as a mental exercise. Regardless of the motivation, we all learn sooner or later that this is a surprisingly non-trivial topic, especially that it's critically important to do everything right.

### We've all been there
Let's say we have some cash in different currencies, and wish to know how much dollars these amount to. A naive model could look like this:
```Scala
@main def run =
  val pounds: BigDecimal = 2.0
  val exchange: BigDecimal = 1.2 //Bad naming!
  val dollars = pounds * exchange
  val savingsDollars: BigDecimal = 3.0
```
We all know how it ends:
```Scala
  val dollarsInCash = dollars + savingsDollars
  val dollarsInCashAndFromExchange = dollars + savingsDollars + exchange //Horribly wrong!
  println(s"Dollars from selling pounds: $dollars, from savings: $savingsDollars. Total: $dollarsInCash")
```
A mistake that could be fixed with better naming, one could argue &mdash; the problem however is that this bug is especially hard to spot, and nothing helps us to do so!

### Enter the Model
Let's address that issue, and give our values a proper representation:
```Scala
case class Money(value: BigDecimal)
case class Rate(rate: BigDecimal)
```
Now confusing an exchange rate with money should not be possible, at least on the surface. Let's look at the usage:
```Scala
@main def run =
  val pounds: Money = Money(2.0)
  val poundsToDollars = Rate(1.2)
  val dollars = pounds.value + poundsToDollars.rate
```
Turns out wrapping values in domain classes enforces nothing if the first thing we have to do in order to perform any sensible operation is to unwrap the values manually. The next step is to enrich the model with combinators:
```Scala
case class Money private(value: BigDecimal):
  def +(other: Money): Money = Money(value + other.value)
  def exchange(rate: Rate): Money = Money(value * rate.rate)
  override def toString = value.toString
case class Rate(rate: BigDecimal)
```
Now we are safe.

Or are we?
### Foiled again
```Scala
@main def run =
  val pounds: Money = Money(2.0)
  val poundsToDollars = Rate(1.2)
  val dollars = pounds.exchange(poundsToDollars)
  val dollarsInCash = pounds + dollars
```
Our model misses some crucial information &mdash; there are several different currencies that cannot be added, at least not without converting them first. Modeling this information is straightforward:
```Scala
case class Currency(name: String):
  override def toString = name
case class Money(value: BigDecimal, currency: Currency):
  def +(other: Money): Money =
    if currency == other.currency
    then Money(value + other.value, currency)
    else throw IllegalArgumentException(s"Mismatched currencies: $currency and ${other.currency}")
  def exchange(rate: Rate): Money =
    if currency == rate.from
    then Money(value * rate.rate, rate.to)
    else throw IllegalArgumentException(s"Mismatched currencies: $currency and ${rate.from}")
case class Rate(rate: BigDecimal, from: Currency, to: Currency)
```

Now, should anyone try to add mismatching currencies or convert money using a wrong rate, a nasty exception would be thrown in their faces.
```Scala
@main def run =
  val gbp = Currency("GBP")
  val usd = Currency("USD")
  val pounds = Money(2.0, gbp)
  val poundsToDollars = Rate(1.2, gbp, usd)
  val dollars = pounds.exchange(poundsToDollars)

  //These two blow up!
  val totalCash = pounds + dollars
  val dollarsToPounds = dollars.exchange(poundsToDollars)
```

### All is good
As we can see, a relatively simple model of the domain can make our lives so much simpler, saving us from wasting hours (if not days) of debugging to find a simplest mistake possible.

As a final touch, let's get rid of the exceptions in favor of a more sensible error handling mechanism:
```Scala
case class Currency(name: String):
  override def toString = name
case class Money(value: BigDecimal, currency: Currency):
  def +(other: Money): Either[MismatchedCurrencies, Money] =
    if currency == other.currency
    then Right(Money(value + other.value, currency))
    else Left(MismatchedCurrencies(currency, other.currency))
  def exchange(rate: Rate): Either[MismatchedCurrencies, Money] =
    if currency == rate.from
    then Right(Money(value * rate.rate, rate.to))
    else Left(MismatchedCurrencies(currency, rate.from))
case class Rate(rate: BigDecimal, from: Currency, to: Currency)
case class MismatchedCurrencies(c1: Currency, c2: Currency)
```

### ...or is it?
```Scala
def calculateUSIncomeTax(dollars: Money): Money =
  //Skipping the implementation, as taxes are ridiculously complicated
  ???

@main def run =
  val gbp = Currency("GBP")
  val usd = Currency("USD")
  val pounds = Money(2.0, gbp)
  val tax = calculateUSIncomeTax(pounds)
```
There is nothing we can do to prevent passing a wrong currency into a wrong place. Even worse, if the `calculateUSIncomeTax` implementor did not place a check inside, we would have no clue what went wrong. The nightly debugging session we worked so hard to prevent seems inevitable.

In addition, switching from exceptions to `Either` made our API a bit less convenient &mdash; now we have to use pattern matching or for-comprehensions everywhere, and cannot chain our operations as nicely as previously:
```Scala
@main def run =
  val gbp = Currency("GBP")
  for
    pounds <- Money(10, gpb) + Money(11, gbp)
    morePounds <- pounds + Money(0.37, gbp) //We cannot do `Money(10, gbp) + Money(11, gbp) + Money(0.37, gbp)` anymore!
  yield morePounds
```

Can we somehow make the implementation safer? Or at least get the nice API back?

### Phantom menace
The answer is yes &mdash; if you wish to sell your soul to the type-level devil and let the compiler check the validity of your operations. First let's lift the currency information into the type level:
```Scala
sealed trait Currency
sealed trait GBP extends Currency
sealed trait USD extends Currency
sealed trait CHF extends Currency
```
Then let's add some type parameters to our `Money` and `Rate` classes:
```Scala
case class Money[A](value: BigDecimal):
  def +(other: Money[A]): Money[A]
  def exchange[B](rate: Rate[A, B]): Money[B] =
    Money(value * rate.rate)
case class Rate[A, B](rate: BigDecimal)
```
These type parameters don't serve any purpose except distinguishing between different currencies, and don't interact in any way with the "value" world &mdash; - therefore we call them "phantom type parameters".
Now we can observe that invalid additions and conversions are no longer possible:
```Scala
@main def run =
  val pounds = Money[GBP](2.0)
  val dollars = Money[USD](3.0)
  //pounds + dollars //Does not compile!
  val poundsToDollars = Rate[GBP, USD](1.2)
  val total = pounds.exchange(poundsToDollars) + dollars
  //dollars.exchange(poundsToDollars) //Does not compile!
  //val money: Money[GBP] = Money[CHF](2.0) //Does not compile!
```
We both have prevented invalid behavior at compile time **and** regained our nice API!

### The price
We have, however, lost a couple of things. First, we cannot display our `Money` objects nicely anymore, since the currency info is no longer just a value, but instead a type parameter that ceases to exist at runtime; second, the implementation became less elegant, as now we have a dedicated trait hierarchy just for marking our `Money` while most likely still needing the value-level representation for different use cases. The former is an easily fixable detail, the latter is a sacrifice we might be willing to make when safety is critical.

### Singleton of Scalatown
There is one more thing that we can do, a trick that relies on an obscure, borderline black magic Scala feature. If you are satisfied with the solution above, feel free to skip the rest &mdash; otherwise fasten your seatbelts and abandon all hope.

Enter the singleton types.

When we think about types, our intuition tends to revolve around sets of values that share some common traits, constraints or behaviors. The `Int`s are plain integers from a certain range, `String`s are sequences of characters, various object types are whatever we want them to be. Some of these sets of values are subsets of other, broader sets &mdash; establishing a relationship we call "subtyping". This intuition does not prevent us from constructing types inhabited by a single value (or none, for what it's worth); so when we reverse our thinking, every single value gives rise to a distinct type - a type that we might interpret as a restriction "values of this type are only this single, specific thing". Scala 2.13 introduced this notion to its type system[^SIP-23], allowing for constructs like:
```Scala
val thisIs213: "213" = "213"
def doSomethingWith213(argument: "213"): String =
  s"$argument is 213, as it cannot be anything else"
```
Thanks to this we can impose restrictions of power previously unknown, and we can refine our `Money` model even further:
```Scala
case class Currency(name: String):
  override def toString = name
case class Money[A](value: BigDecimal, currency: A):
  def +(other: Money[A]) = Money(value + other.value, currency)
  def exchange[B](rate: Rate[A, B]): Money[B] =
    Money(value * rate.rate, rate.to)
  override def toString = s"$value $currency"
case class Rate[A, B](rate: BigDecimal, from: A, to: B)

@main def run =
  val gbp = Currency("GBP")
  val usd = Currency("USD")
  val pounds = Money[gbp.type](2.0, gbp)
  val poundsToDollars = Rate[gbp.type, usd.type](1.2, gbp, usd)
  val dollars: Money[usd.type] = pounds.exchange(poundsToDollars)
  //println(pounds + dollars) //Does not compile!
  //println(dollars.exchange(poundsToDollars)) //Does not compile!
  //val dollarsAgain: Money[usd.type] = pounds //Does not compile!
```

We can now freely add new currencies as we go, we have retained our nice API[^tradeoff], and we are as safe as it gets!

### What's the value of a singleton type?
The example above has the downside of unnecessary constructor parameters. We'd like to get rid of them, but that would mean our overloaded `toString` wouldn't work again. Fortunately, retrieving a value of a singleton type should be quite simple &mdash; and indeed, Scala provides us with a nice way of doing exactly that. With the `ValueOf` context bound we can rewrite our code into its final form (taking the liberty to organize stuff a bit):

```Scala
//Here's our model:
case class Currency(name: String):
  override def toString = name

case class Money[A: ValueOf](value: BigDecimal):
  def +(other: Money[A]) = Money(value + other.value)
  def exchange[B: ValueOf](rate: Rate[A, B]): Money[B] =
    Money(value * rate.rate)
  def exchangeFor[B: ValueOf](using rate: Rate[A, B]): Money[B] =
    exchange(rate)
  //The `ValueOf` context bound effectively marks our type as a singleton type.
  //By its virtue we get the `valueOf` method, which allows summoning the value of our singleton type.
  override def toString = s"$value ${valueOf[A]}"

case class Rate[A, B](rate: BigDecimal):
  def andThen[C](other: Rate[B, C]): Rate[A, C] =
    Rate(rate * other.rate)

//We can define our custom apply for convenience:
object Money:
  def apply(value: BigDecimal, currency: Currency): Money[currency.type] =
    Money(value)

//And some example values:
object Currencies:
  val gbp = Currency("GBP")
  type GBP = gbp.type
  val usd = Currency("USD")
  type USD = usd.type
  val chf = Currency("CHF")
  type CHF = chf.type

object Rates:
  import Currencies.*
  given Rate[GBP, USD] = Rate(1.2)
  given Rate[USD, CHF] = Rate(0.9)
  given transitiveRate[A: ValueOf, B, C: ValueOf](using r1: Rate[A, B], r2: Rate[B,C]): Rate[A, C] =
    r1.andThen(r2)

@main def run =
  import Currencies.*
  import Rates.given
  val pounds: Money[GBP] = Money(2.0)
  val alsoPounds = Money(2.0, gbp) //using the convenience apply method
  val dollars = pounds.exchangeFor[USD]
  val francs = pounds.exchangeFor[CHF]
```
With this technique we have forced the compiler to check the correctness of our domain logic &mdash; the benefits are hard to overstress. And all that thanks to a simple concept!

[^SIP-23]: [A detailed description of the feature](https://docs.scala-lang.org/sips/42.type.html)
[^tradeoff]: There is a limitation to this &mdash; singleton types rely on the notion of a stable path, so `Currency("EUR").type` unfortunately is not a valid singleton. It also means that `val eur1 = Currency("EUR")` and `val eur2 = Currency("EUR")` give rise to two different singleton types.

