# Path dependent types - modeling algebraic structures has never been easier

Let's start with small refreshment of what actually the dependent typing is. Using [Wikipedia](http://en.wikipedia.org/wiki/Dependent_type) as a source:
>In computer science and logic, a dependent type is a type that depends on a value. 

Neat, huh? But what does it mean in practice? If you take a look at the Wikipedia's [list](http://en.wikipedia.org/wiki/Dependent_type#Comparison_of_languages_with_dependent_types)  of languages that implement dependent typing you might get strange sensation that it is something really obscure and esoteric. Languages mentioned there range from "never heard of" (Guru/Matita anyone?) to "yeah, I recall reading about it on Reddit once or twice" (Coq/Idris). Oh wait, there is `F#`. Ooops, sorry, it is not `F#` but [`F*`](http://research.microsoft.com/en-us/projects/fstar/). Thanks goodness it **is** something from Microsoft at least. And look ma', no Haskell.  If it is not implemented in Haskell then truly, what problems can it solve?

Well, actually dependent typing is helpful in solving very down-to-earth problems. I bet that everyone programming in a language with sufficiently expressive type system  has wondered at least once - why my type system does not prevent me from, say, accessing head of empty list? This question is in fact the canonical showcase for dependent typing. Just imagine - if you could somehow encode the length of a list (a value) into its type then it'd be trivial to state that list of length 0 (type that depends on a value, mind you) has no head to be accessed. Standard library would have just been changed to (please bear with my completely made up syntax):

```scala
sealed abstract class List[+A, N is Int]  {
  // no head here
}

case object Nil extends List[Nothing, 0] {
  // no head here either
}

sealed abstract class NonEmptyList extends List[+A, n where n > 0] {
  def head: A
}
```

Oh, by the way, just observe what you could have achieved when it comes to concatenation:

```scala
final case class ::[B](override val head: B, var tl: List[B, n]) extends List[B, (n + 1)] {
  override def tail : List[B, n] = tl
}
```

See how you would have been able to track the actual length of the list throughout the program by saying that e.g. appending an element to a list of length `n` produces list with length `n + 1`.

Curious reader will notice that a concept of separation empty and non-empty lists is already present in Scalaz [`NonEmptyList`](http://docs.typelevel.org/api/scalaz/nightly/index.html#scalaz.NonEmptyList). That's true, but observe how calling tail of `NonEmptyList` produces an ordinary `List` with no extra guarantees. That's because at this point the type system cannot tell apart lists of length 1 (which tails are obviously empty) and longer lists. Of course you could try to remedy this problem by introducing `OneElementList` type with appropriate tail/head signatures but this remedy quickly falls short when you discover that you really want `List` to have `drop(n: Int)` method and you just can't express that, because there is no way to tell what is the type of the result when there is no "link" between values' world and types' world.

Of course there have been attempts to have this neat "link" in Scala.  Make sure you are familiar with ideas behind [Church encoding](http://en.wikipedia.org/wiki/Church_encoding) and then (and **only** then) take a deep dive into renowned [Shapeless](https://github.com/milessabin/shapeless) lib, and its implementation of [type-level naturals](https://github.com/milessabin/shapeless/blob/master/core/src/main/scala/shapeless/nat.scala). It looks like this is what we are looking for until you realize that out-of-the-box representation takes you only up to 1.000 or so before compiler gives up (1.000 encoded as a type is represented as `Succ[Succ[Succ[... _0]]]]`) and there is no way to encode a number that is not compile-time constant. But, all in all you are able to do type-level computations and all this is pretty close to the real thing.

Would it be possible to overcome these limitations and have a simple value <-> type reasoning in Scala without making your compiler burn tires? Sure there is ...

##### Path dependent types

You need to know that Scala has a notion of a type dependent on a value. Only that the notion of dependency is not expressed in type signature but rather in type *placement*. Martin Odersky outlined this idea in a co-authored paper [Foundations of Path-Dependent Types](http://lampwww.epfl.ch/~amin/dot/fpdt.pdf) where we read:

> (...) types must be able to refer to objects, i.e. contain terms that serve as static approximation of a set of dynamic objects. In other words, some level of dependent types is required; the usual notion is that of *path-dependent* types.

I found pretty nice and suggestive introduction to path dependent types [here](http://danielwestheide.com/blog/2013/02/13/the-neophytes-guide-to-scala-part-13-path-dependent-types.html). You can find a good read on this topic in ktoso's [vademecum of types](http://ktoso.github.io/scala-types-of-types/#path-dependent-type) as well. To recap, let's contemplate for a minute the following snippet:

```scala
class A {
  class B
  var b: Option[B] = None
}
val a1 = new A
val a2 = new A
val b1 = new a1.B
val b2 = new a2.B
a1.b = Some(b1)
a2.b = Some(b1) //does not compile
```

When you have become enlightened we will be proceeding to the main part of this post. That is, how on earth I used that when solving one of the [Codility](https://codility.com/) [challenges](https://codility.com/programmers/challenges/)?

##### Problem to solve

When I feel my algorithmic skills are getting rusty, I usually end up checking Codility's web page and finding myself a challenging problem to work on (BTW, I recommend it to everyone - it simply makes you a better programmer). I always try not only to solve a problem, but also to write the most idiomatic code in a language I am using. This way you can enjoy two benefits simultaneously: your problem solving fu is going up and the mastery of your language of choice is  skyrocketing. 

The particular problem that pushed me toward path dependent types is called [Omicron](https://codility.com/programmers/challenges/omicron2012). It has already been "officially" [solved](http://blog.codility.com/2012/03/omicron-2012-codility-programming.html) so you can spare yourself some heavy thinking. The  problem is about efficiently computing `Fib(A ** B) % 10000103`. It's too slow to use BigInteger arithmetic, so you need to employ [modular arithmetic](http://en.wikipedia.org/wiki/Modular_arithmetic). The official solution does that, but, alas, the code is littered with modulo operations

```scala
exp := (N * exp(N, M - 1)) mod Q
```

which is bad-looking and error prone. Nah, no self-respecting Scala programmer would do that to her code. In fact, we are easily able to do better in Scala, it just takes a second to notice that implicit conversions are in the language for a reason... But wait - convert to **what** exactly? How can you represent a type with semantics of an integer modulo N where N is inevitably a large prime number?

Obviously, you could just make N an implicit and put it in a scope of code doing the operation. But in the challenge you will actually need two moduli - one for the result, and the other that is period of Fibonacci sequence (I recommend to try solving it for yourself, I am not going to analyze the problem itself in this post). What's more, we'd ideally want to have as much type-safety as possible. We want compiler to prevent us from accidentally operating on integers from one finite field (that is integers mod N) in a different field (integers mod M). You could try to encode modulus as Shapeless' type-level natural, but numbers used in our solution are pretty large (10000103 and 20000208) so that would very probably blow up your compiler.

Path-dependent typing to the rescue!

##### Solution

To summarize, we need to devise a system that lets us:

1. perform modulo computations transparently,
2. be sure that compiler won't let us use int mod N as int mod M.

Let's stick with the 2nd part for a while. This requirement seems to fit path-dependent typing nicely. We need to make sure that whenever we operate on a modulus it comes from an unique "path", so the compiler can track it for us. This path's got to be represented by some type. Let's call this type Z to mimic mathematical notation (we can say that it represents a finite field for some N).

```scala
case class Z(modulus: Int) {
  sealed class Modulus {
    val value = modulus
  }
}
```
	
Let's check if we can mix moduli from different fields:

```scala
scala> val z1 = Z(10000103)
z1: Z = Z(10000103)

scala> val z2 = Z(20000208)
z2: Z = Z(20000208)

scala> val m = new z1.Modulus
m: z1.Modulus = Z$Modulus@6c6cb480

scala> val n: z2.Modulus = m
error: type mismatch;
 found   : z1.Modulus
 required: z2.Modulus
```

Good, no way to do harm. Having done this, let's try to define a type representing an integer mod some modulus.

```scala
class IntMod[N <: Z#Modulus] private(val value: Long) extends AnyVal 
```

In other words, our "integer mod N" is parametrized by the path-dependent Modulus type (notice `Z#` notation) and by virtue of this the `IntMod[z1.Modulus]` is an entirely different type from `IntMod[z2.Modulus]`. This is exactly what we wanted in order to achieve 2 from our little list. By the way, we have made `IntMod`'s constructor private to take a stab at the 1st point. We don't wanna let anyone instantiate our integers because we want them to be indistinguishable from "real" numerics. We have also made it value type for a good measure, we'd certainly like to squeeze a little bit of performance where possible.

So how do we mix `IntMods` with Scala native types? We need to be able to lift any numeric to our `IntMod` implicitly. Let's make a companion object to help with this.

```scala
object IntMod {
  implicit def intAsIntModN[N <: Z#Modulus](i: Int) = ???
  implicit def longAsIntModN[N <: Z#Modulus](i: Long) = ???

  implicit def intModN2Long(a: IntMod[_]): Long = a.value
	
  implicit def intModN2Int(a: IntMod[_]): Int = a.value.toInt
}
```

So far so good, but how do we implement the `longAsIntModN` method? We'll need a value of (correctly-typed) modulus for this. Where can we get one from? Well, let's make it an implicit parameter and take advantage of the fact that type parameters are searched in the process of implicit resolution. In other words, if we introduced a `Modulus` companion object producing the implicit, it'd be resolved from there.  After these changes our code looks like this:

```scala
case class Z(modulus: Int) {
  sealed class Modulus {
    val value = modulus
  }

  object Modulus {
    implicit val mod = new Modulus()
  }
}

class IntMod[N <: Z#Modulus] private(val value: Long) extends AnyVal 
	
object IntMod {
  implicit def intAsIntModN[N <: Z#Modulus](i: Int)(implicit modulus: N): IntMod[N] =
    new IntMod[N](i % modulus.value)
  implicit def longAsIntModN[N <: Z#Modulus](i: Long)(implicit modulus: N): IntMod[N] =
    new IntMod[N](i % modulus.value)
	
  implicit def intModN2Long(a: IntMod[_]): Long = a.value
	
  implicit def intModN2Int(a: IntMod[_]): Int = a.value.toInt
}
```

Let's check how we are doing:

```scala
scala> val i1 = 10: IntMod[z1.Modulus]
i1: IntMod[z1.Modulus] = IntMod@a

scala> val i2 = 100000000: IntMod[z1.Modulus]
i2: IntMod[z1.Modulus] = IntMod@9892e1

scala> i2.value
res1: Long = 9999073

scala> val i3 = 10: IntMod[z2.Modulus]
i3: IntMod[z2.Modulus] = IntMod@a

scala> val i4: IntMod[z2.Modulus] = i1
error: type mismatch;
 found   : IntMod[z1.Modulus]
 required: IntMod[z2.Modulus]
```

Lookin' good, huh? Actually, we have almost completed our task. What remains is just to implement missing algebraic operations. And our implicit conversions render it a trivial task (mostly, [division](http://en.wikipedia.org/wiki/Modular_multiplicative_inverse) is tricky).

```scala
class IntMod[N <: Z#Modulus] private(val value: Long) extends AnyVal {
  def +(x: Long)(implicit m: N): IntMod[N] = value + x

  def -(x: Long)(implicit m: N): IntMod[N] = value - x

  def *(x: Long)(implicit m: N): IntMod[N] = value * x
	
  def /(x: IntMod[N])(implicit m: N): IntMod[N] = ??? // left as an exercise to the reader
}
```

```scala
scala> val ring = Z(20000208)
ring: Z = Z(20000208)

scala> type Q = ring.Modulus
defined type alias Q

scala> val k = (10000: IntMod[Q]) * 100000
k: IntMod[Q] = IntMod@1310530

scala> val l = k + 10000000
l: IntMod[Q] = IntMod@986de0

scala> val m = l * k
m: IntMod[Q] = IntMod@8cfff0

scala> m.value
res0: Long = 9240560
```

#### Epilogue

I hope I showed you how various flavors of dependent typing can be useful in everyday programming. While we can't [yet](https://github.com/lampepfl/dotty)  enjoy expressiveness of type systems with [full dependent types](http://www.idris-lang.org/), Scala has enough to control type dependencies in amazing ways you'd think are only possible at the run time. If you're wondering what is the precise relation between path-dependent types and dependent types, let me point you to Miles Sabin's thoughtful and detailed [reply](http://stackoverflow.com/questions/12935731/any-reason-why-scala-does-not-explicitly-support-dependent-types/12937819#12937819) to similar question on Stack Overflow:
>Syntactic convenience aside, the combination of singleton types, path-dependent types and implicit values means that Scala has surprisingly good support for dependent typing. 
In any case, I don't think this (...) can be used as an objection to Scala's status as a dependently typed language. If it is, then the same could be said for Dependent ML (which only allows dependencies on natural number values) which would be a bizarre conclusion.
