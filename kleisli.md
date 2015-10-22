# Arrows, Monads and Kleisli - part I

During Scala Days Amsterdam I came across a concept of [arrow](https://www.parleys.com/tutorial/functional-programming-arrows). It was called a *general interface to computation* and it looked like a technique that could bridge code that is imperative in nature with proper functional programming. Later on I read about [Railway Oriented Programming](http://fsharpforfunandprofit.com/rop/) and found it really neat. I was compelled to reinvent the wheel and check all this in practice. I dug up some old business logic Scala code, that could really be Java code with slightly different keywords, and tried to refactor it using all I'd learned about arrows. BTW, you can find code used in this blog post on Github TODO: link here. In this post I'll work with this code from the ground up and try to demonstrate curious things you can do with arrows. You can go through the examples by following commits in the GitHub repo. 
Because no one in their sane mind likes to read "War and Peace"-sized posts, I decided to break it up into parts. In the first part I'll explain arrows and Kleisli arrows, trying to show how they can help to express business logic in terms of data flow. In the next parts I'll get into more complicated examples, try to overcome various Scala type-system deficiencies and, finally, introduce arrow transformers.

##### What's an arrow?
In simple words arrow is a generalization of function i.e. it is an abstraction of things that behave like functions. [Wikipedia](https://en.wikipedia.org/wiki/Arrow_%28computer_science%29) says:
>Arrows are a type class used in programming to describe computations in a pure and declarative fashion. First proposed by computer scientist John Hughes as a generalization of monads, arrows provide a referentially transparent way of expressing relationships between logical steps in a computation. Unlike monads, arrows don't limit steps to having one and only one input.

[Haskell wiki](https://wiki.haskell.org/Arrow_tutorial):
>Arrow a b c represents a process that takes as input something of type b and outputs something of type c. 

You can treat arrow as a function type that gives you more combinators to work with than function's `compose` or `andThen`. Composing arrows is not only more expressive but can encapsulate rich semantics e.g. you can compose monadic processes.

### Let's get the party started
Let's imagine we have legacy code pulled out of production tracking system. In this system things that are being tracked are modeled by a domain object called `ProductionLot` (simplified):

TODO: link to github

```scala
case class ProductionLot(id: Option[Long],
                         status: ProductionLotStatus.Value,
                         productionStartDate: Option[Date] = None,
                         productionEndDate: Option[Date] = None,
                         workerId: Option[Long] = None)

```

You also have a repository for db access:

```scala
class ProductionLotsRepository {
  def findExistingById(productionLotId: Long): ProductionLot =
    findById(productionLotId).getOrElse(sys.error(s"ProductionLot $productionLotId not found"))

  def findById(productionLotId: Long): Option[ProductionLot] = ???

  def save(productionLot: ProductionLot): Long = ???
}
```

Business logic governing operations on production lots lies in, as you can imagine, `ProductionLotsService`. Let's say you can perform three operations in this system. You can start production of a production lot, revoke production lot (change its current status and clear other properties) and reassign production lot to a different worker. Obviously these operations need to perform some kind of validations (eg. you cannot reassign production lot to the same worker it already has), interact with db etc.

TODO: link to GitHub
```scala
class ProductionLotsService(productionLotsRepository: ProductionLotsRepository) {
  def startProductionOf(productionLotId: Long, productionStartDate: Date, workerId: Long): Unit = {
    val productionLot = productionLotsRepository.findExistingById(productionLotId)

    verifyWorkerCanBeAssignedToProductionLot(productionLot, workerId)

    val productionLotWithProductionStartData = productionLot.copy(
      productionStartDate = Some(productionStartDate),
      workerId = Some(workerId),
      status = ProductionLotStatus.InProduction
    )

    productionLotsRepository.save(productionLotWithProductionStartData)
  }

  def changeAssignedWorker(productionLotId: Long, newWorkerId: Long): Unit = {
    val productionLot = productionLotsRepository.findExistingById(productionLotId)

    verifyWorkerChange(productionLot, newWorkerId)

    val updatedProductionLot = productionLot.copy(workerId = Some(newWorkerId))

    productionLotsRepository.save(updatedProductionLot)
  }

  def revokeProductionLotTo(productionLotId: Long,
                            productionLotStatus: ProductionLotStatus.Value): Unit = {
    val productionLot = productionLotsRepository.findExistingById(productionLotId)

    val revokedProductionLot = productionLot.copy(
      status = productionLotStatus,
      workerId = None,
      productionStartDate = None,
      productionEndDate = None
    )

    productionLotsRepository.save(revokedProductionLot)
  }

  private def verifyWorkerChange(productionLot: ProductionLot, newWorkerId: Long): Unit = {
    require(productionLot.workerId.isDefined && productionLot.workerId.get != newWorkerId,
            s"Production lot worker expected to be defined and different than $newWorkerId")
  }

  private def verifyWorkerCanBeAssignedToProductionLot(productionLot: ProductionLot, workerId: Long): Unit = {
    val productionLotId = productionLot.id.get
    val productionLotHasNoWorkerAssigned = productionLot.workerId.isEmpty

    require(productionLotHasNoWorkerAssigned, s"Production lot: $productionLotId has worker already assigned")
  }
}
```

As you can see this code is littered with problems. First of all, it communicates with the outside world via exceptions. Clients of the service need to be better aware of that. Additionally, they can't react to what is wrong as there is no distinction between various types of errors. Returning Unit does not make you a good neighbor either.  All in all, API of this service is not very useful in the functional paradigm. Another problem is that it is quite hard to add new functionality. Imagine that there is **a new requirement: production lot cannot be modified if its status has been set to done**. You can't help but grit your teeth and copy-paste this piece of logic through every method that updates production lots. Let's try to improve this. 
**Our goal will be to eliminate problems we have observed and at the same time make this code functional.** We'll first attempt to refactor it so that adding our new requirement is a one liner and then try to eliminate exceptions, turning methods into pure functions. 

###Try function composition

As you probably noticed, all the methods follow the same pattern. Get something from db, validate the change, modify the entity and update db. This observation makes it appealing to try to rewrite it in function composition style. Of course, methods returning unit are not very composition-friendly, so a few adjustments have to be made, but you can write it arguably cleaner in the following way:

TODO link to GitHub
```scala
  def startProductionOf(productionLotId: Long, productionStartDate: Date, workerId: Long): Unit = {
    val verify: ProductionLot => ProductionLot = { productionLot =>
      verifyWorkerCanBeAssignedToProductionLot(productionLot, workerId)
      productionLot
    }

    val copy: ProductionLot => ProductionLot = _.copy(
      productionStartDate = Some(productionStartDate),
      workerId = Some(workerId),
      status = ProductionLotStatus.InProduction
    )

    val startProductionOfF = productionLotsRepository.findExistingById _ andThen
      verify andThen
      copy andThen
      productionLotsRepository.save

    startProductionOfF(productionLotId)
  }

  def changeAssignedWorker(productionLotId: Long, newWorkerId: Long): Unit = {
    val verify: ProductionLot => ProductionLot = { productionLot =>
      verifyWorkerChange(productionLot, newWorkerId)
      productionLot
    }

    val copy: ProductionLot => ProductionLot = _.copy(workerId = Some(newWorkerId))

    val changedAssignedWorkerF = productionLotsRepository.findExistingById _ andThen
                              verify andThen
                              copy andThen
                              productionLotsRepository.save

    changedAssignedWorkerF(productionLotId)
  }
```

This does not get us very far but you can observe an emerging pattern: every method can be rewritten as a composition over `verify` function and `copy` function plus a prologue and epilogue (find and save). `verify` and `copy` will be functions of `ProductionLot` and some environment (formed of arguments). This is a powerful insight yielding the idea on how to abstract out common behavior. I could be gluing functions together to get it done for now but, being aware of the ultimate goal of getting rid of exceptions, I'd rather embrace a more powerful abstraction of arrow.

### Implement Arrow

What's the interface of arrow? From [Wikipedia](https://en.wikipedia.org/wiki/Arrow_%28computer_science%29#Definition) 
>  A type constructor arr that takes functions from any type s to another t, and lifts those functions into an arrow A between the two types

arr (s -> t)        ->   A s t

> A piping method first that takes an arrow between two types and converts it into an arrow between tuples. The first elements in the tuples represent the portion of the input and output that is altered, while the second elements are a third type u describing an unaltered portion that bypasses the computation.

first (A s t)       ->   A (s,u) (t,u)

(Scalaz also defines `second` as a complementary function to first. 
second (A s t) -> A (u, s) (u, t)). Please note that there is no need whatsoever to restrict type of "unaltered portion".  This is to make `first` and `second` operators fit into any computation, regardless of its result type.  Hence the introduction of the third, independent, type parameter. 

> A composition operator >>> that can attach a second arrow to a first as long as the first function's output and the second's input have matching types

A s t  >>>  A t u   ->   A s u

>A merging operator *** that can take two arrows, possibly with different input and output types, and fuse them into one arrow between two compound types. 

A s t  ***  A u v   ->   A (s,u) (t,v)

Wikipedia does not mention it but there is another commonly used combinator called split (&&&). It sends the input to both argument arrows and combines their output.

A s t &&& A s u -> A s (t, u).
 
I find it really helpful to visualize arrow combinators: 
![](http://virtuslab.com/wp-content/uploads/2015/10/kleisli_combinators.png)

Looking closely at the definitions we are able to coin arrow interface in form of a trait. 

TODO: link to GitHub
```scala
trait Arrow[=>:[_, _]] {
  def id[A]: A =>: A
  def arr[A, B](f: A => B): A =>: B

  def compose[A, B, C](fbc: B =>: C, fab: A =>: B): A =>: C

  def first[A, B, C](f: A =>: B): (A, C) =>: (B, C)
  def second[A, B, C](f: A =>: B): (C, A) =>: (C, B)

  def merge[A, B, C, D](f: A =>: B, g: C =>: D): (A, C) =>: (B, D) = compose(first(f), second(g))
  def split[A, B, C](fab: A =>: B, fac: A =>: C): A =>: (B, C) = compose(merge(fab, fac), arr((x: A) => (x, x)))
}
```

I followed Scalaz convention closely here: an `id` method is added that returns identity arrow (it will come handy later on) and `compose` is reversed to resemble function composition (we'll be calling it `<<<`, instead of `>>>`). 
As you probably noticed from the diagram `merge` can be implemented as a composition of `first` and `second`. Further conclusion that can be drawn from this is that `f &&& g = arr(x -> (x, x)) >>> (f *** g)`. 
You might be curious about the naming of `Arrow` type parameter. It is called `=>:` to mimic function type's `=>`. It reflects that arrow is a generalization of function and enhances function-like types, that is, anything that takes input and produces output. It builds on a less-known Scala feature that let's you write higher-order type constructor in infix form eg. `=>[A, B]` is  `A => B`.
Next, we'd like to enhance arrow-like classes with fancy symbolic arrow combinators we all love :-) Following Scalaz convention, we'll wrap these in the `ArrowOps` class and provide appropriate implicit conversions.

```scala
final class ArrowOps[=>:[_, _], A, B](val self: A =>: B)(implicit val arr: Arrow[=>:]) {
  def >>>[C](fbc: B =>: C): A =>: C = arr.compose(fbc, self)

  def <<<[C](fca: C =>: A): C =>: B = arr.compose(self, fca)

  def ***[C, D](g: C =>: D): (A, C) =>: (B, D) = arr.merge(self, g)

  def &&&[C](fac: A =>: C): A =>: (B, C) = arr.split(self, fac)
}

object Arrow extends ArrowInstances {
  implicit def ToArrowOps[F[_, _], A, B](v: F[A, B])(implicit arr: Arrow[F]): ArrowOps[F, A, B] = new ArrowOps(v)
}

```

All this machinery gives us ability to define arrow-like types. One obvious choice is an ordinary `Function1`.

##### Function is an arrow

Mapping between functions and arrows is straightforward:
- identity arrow is identity function
- `compose` is function composition
- `first` is a function that operates on a 2-tuple argument, mapping first part and leaving 2nd unaltered

TODO: link to GitHub
```scala
trait ArrowInstances {
  // function is arrow
  implicit object FunctionArrow extends Arrow[Function1] {
    override def id[A]: A => A = identity[A]

    override def arr[A, B](f: (A) => B): A => B = f

    override def compose[A, B, C](fbc: B => C, fab: A => B): A => C = fbc compose fab

    override def first[A, B, C](f: A => B): ((A, C)) => (B, C) = prod => (f(prod._1), prod._2)

    override def second[A, B, C](f: A => B): ((C, A)) => (C, B) = prod => (prod._1, f(prod._2))

    override def merge[A, B, C, D](f: (A) => B, g: (C) => D): ((A, C)) => (B, D) = { case (x, y) => (f(x), g(y)) }
  }
}
```
I chose to replace `merge` with something more tailored to functions (and also slightly faster) but we could live with the default one.

### Rewrite code using arrows
Now that we have extended functions with arrow combinators, we can use them in our service to make it composable. Let's sketch business logic flow to analyze how it can be done:

![](http://virtuslab.com/wp-content/uploads/2015/10/kleisli_flow.png) 

Looks good. Translating this flow back to code gives us very generic function that can be used to implement all the buisness logic we have:

TODO: link to GitHub

```scala
  private def productionLotArrow[Env](verify: (ProductionLot, Env) => Unit, copy: (ProductionLot, Env) => ProductionLot): (Long, Env) => Long =
    Function.untupled(
      (productionLotsRepository.findExistingById _ *** identity[Env])
        >>> (verify.tupled
          &&& (copy.tupled >>> productionLotsRepository.save))
          >>> (_._2)
    )
```

We can customize its behavior by plugging in various functions. For example:

```scala
  private val changeWorkerA = productionLotArrow[Long] (verifyWorkerChange, (pl, id) => pl.copy(workerId = Some(id)))

  def changeAssignedWorker(productionLotId: Long, newWorkerId: Long): Unit =
    changeWorkerA(productionLotId, newWorkerId)

  private val revokeToA =
    productionLotArrow[ProductionLotStatus.Value] (
      (_, _) => (),
      (pl, status) => pl.copy(
        status = status,
        workerId = None,
        productionStartDate = None,
        productionEndDate = None
      )
    )

  def revokeProductionLotTo(productionLotId: Long,
                            productionLotStatus: ProductionLotStatus.Value): Unit =
    revokeToA(productionLotId, productionLotStatus)
```

Starting production seems to be a little harder because of multiple arguments involved. But when you think about it, each function with multiple arguments can be changed to a `Function1` using case class encapsulation:

```scala
  private case class StartProduction(productionStartDate: Date, workerId: Long)
  private val startProductionA = productionLotArrow[StartProduction] (
    (pl, env) => verifyWorkerCanBeAssignedToProductionLot(pl, env.workerId),
    (pl, env) => pl.copy(
      productionStartDate = Some(env.productionStartDate),
      workerId = Some(env.workerId),
      status = ProductionLotStatus.InProduction
    )
  )

  def startProductionOf(productionLotId: Long, productionStartDate: Date, workerId: Long): Unit =
    startProductionA(productionLotId, StartProduction(productionStartDate, workerId))
```

It is much easier to add new functionality now. Because the business logic is abstracted out to a flow, the only place that needs to be changed is the flow itself. Let's review our **goal: production lot cannot be modified if its status has been set to done**. It's trivial to introduce it now, we have already seen it in the previous diagram.

```scala
  private def productionLotArrow[Env](verify: (ProductionLot, Env) => Unit,
                                      copy: (ProductionLot, Env) => ProductionLot): (Long, Env) => Long = {
    val verifyProductionLotNotDoneF: ((ProductionLot, Env)) => Unit = { case (pl, _) => verifyProductionLotNotDone(pl) }

    Function.untupled(
      (productionLotsRepository.findExistingById _ *** identity[Env])
        >>> ((verify.tupled &&& verifyProductionLotNotDoneF)
          &&& (copy.tupled >>> productionLotsRepository.save))
          >>> (_._2)
    )
  }

  private def verifyProductionLotNotDone(productionLot: ProductionLot): Unit =
    require(productionLot.status != ProductionLotStatus.Done, "Attempt to operate on finished production lot")
```

Although we have improved internals of existing code a bit there is still much bigger problem to tackle on the outside: **to eliminate exceptions**. 
> Even Yoda recommends not to use exception handling for control flow. "Do or do not, there is no try"

### Replacing exceptions with Either

Representing various outcomes of a computation with Either forms a powerful pattern. It makes error handling the part of normal control flow and allows for a fully type-safe and functional code in the presence of errors. 
Let's revisit all the places in our code base where exception may be thrown and rewrite them using Either. We'll represent error cases as subtypes of the new Error class which encode what went wrong in the type and carry some additional information.

TODO link to GitHub
```scala
sealed abstract class Error(val message: String)

case class ProductionLotNotFoundError(id: Long) extends Error(s"ProductionLot $id does not exist")
case class ProductionLotClosedError(pl: ProductionLot) extends Error(s"Attempt to operate on finished ProductionLot $pl")
//...
```

```scala
class ProductionLotsService(productionLotsRepository: ProductionLotsRepository) {
//...
  private def verifyProductionLotNotDone(productionLot: ProductionLot): Either[Error, ProductionLot] =
    Either.cond(productionLot.status != ProductionLotStatus.Done, productionLot, ProductionLotClosedError(productionLot))

  private def verifyWorkerChange(productionLot: ProductionLot, newWorkerId: Long): Either[Error, ProductionLot] =
    productionLot.workerId.fold[Either[Error, ProductionLot]](
      Left(NoWorkerError(productionLot)))(
        workerId => Either.cond(workerId != newWorkerId, productionLot, SameWorkerError(productionLot)))

  private def verifyWorkerCanBeAssignedToProductionLot(productionLot: ProductionLot, workerId: Long): Either[Error, ProductionLot] =
    Either.cond(productionLot.workerId.isEmpty, productionLot, WorkerAlreadyAssignedError(productionLot))
}

class ProductionLotsRepository {
  def findExistingById(productionLotId: Long): Either[Error, ProductionLot] =
    findById(productionLotId).toRight(ProductionLotNotFoundError(productionLotId))

  def findById(productionLotId: Long): Option[ProductionLot] = ???

  def save(productionLot: ProductionLot): Either[Error, Long] = ???
}
```

But we immediately notice a problem - our shiny arrow stops compiling! If you tried to fix it, it'd become clear that there is no easy way to compose these functions! Every function would have to unpack a value from `Either` and pack result back into a new instance of `Either`. Resulting code would be horrible mess with a lot of boilerplate. Things look bleak. Seems we're in worse shape than before...
Fear not. As we said arrows are a general interface to computation. This implies that many different computation types can be expressed using arrows. Do not forget that arrow does not impose restrictions on underlying type as long you can come up with implementations of abstract operations that arrow typeclass needs ie. `first`/`second` and `compose`. While ordinary `A => B` functions are clearly not sufficient to express our new idea, there exists another abstraction designed to do that. It is called `Kleisli` to honor [Swiss mathematician](https://en.wikipedia.org/wiki/Heinrich_Kleisli) who studied properties of categories associated to monads. Going back to our Scala domain `Kleisli` represents _monadic_ computations i.e. functions of type `A => M[B]`. We'll be able to implement arrow on `Kleisli`, creating so called Kleisli arrows ie. arrows that can compose functions of type `A => M[B]` as long as `M` is a monad.

##### What's a monad?
Monad is bread and butter of functional programming. I'm not going to explain the concept in this blog post. It'd probably take quite a few (posts - pun intended) to just skim through theory and practice of monads. For our purposes it suffices to know that monad is a container with two operations: `bind` and `return` (sometimes called `unit` but we'll be calling it `point` to keep things faimilar for Scalaz people). These operations have the following semantics (from [wiki](https://en.wikipedia.org/wiki/Monad_%28functional_programming%29)):

>    The return operation takes a value from a plain type and puts it into a monadic container using the constructor, creating a monadic value.
>    The bind operation takes as its arguments a monadic value and a function from a plain type to a monadic value, and returns a new monadic value.

Many types you've probably used thousands of times can be treated as monads. For instance, a famous example of a `Monad` is `List` with `return` creating one element `List` and `bind` being Scala's `flatMap`. More involved examples include Slick's `Query` where `return` is a `Query#apply` and `bind` chains queries together (again, `flatMap`).
We'll also put to use the fact that monad is a kind of functor i.e. it can be mapped over (like a list or a map). So we'll add another operation to monad interface, called `fmap` that will give us ability to `map` over monadic values. 
Putting this together, we'll be representing monads as members of the following typeclass:

TODO link to GitHub

```scala
trait Monad[M[_]] {
  def point[A](a: => A): M[A]
  def bind[A, B](ma: M[A])(f: A => M[B]): M[B]
  def fmap[A, B](ma: M[A])(f: A => B): M[B]
}
```

Because we'll be using `bind` quite a lot let's make it a symbolic operator. I've chosen `>>=` to adhere to a Haskell tradition.

```scala
final class MonadOps[M[_], A](val self: M[A])(implicit val monad: Monad[M]) {
  def >>=[B](f: A => M[B]) = monad.bind(self)(f)
}

object Monad extends MonadInstances {
  implicit def ToMonadOps[M[_], A](v: M[A])(implicit m: Monad[M]): MonadOps[M, A] = new MonadOps(v)
}
```

##### Is Either a monad?

Well, no it is not. But we won't let that stop us :-)
First of all, monad is of kind `* -> *` while `Either`'s kind is `* -> * -> *` (that's fancy way of saying that `Monad` type constructor takes one type parameter while Either takes two). Then, you cannot compose Either without knowing whether it is `Right` or `Left`. Sometimes people say that `Either` lacks _bias_ e.g. had it been _right biased_ it would have been composable just like `Option` is by sticking to the `RightProjection` and bailing out on `LeftProjection`. 
But we can easily form it into `Monad` ourselves. We just have to use one of the most loathed Scala features called _type lambdas_ to curry `Either` type constructor into `* -> *`. Later on I'm going to show how to make syntax easier on the eyes but for now let's stick to standard ways:

TODO link to github
```scala
trait MonadInstances {
  implicit def eitherMonad[L] = new Monad[({ type λ[β] = Either[L, β] })#λ] {
    override def point[A](a: => A): Either[L, A] = Right(a)

    override def bind[A, B](ma: Either[L, A])(f: (A) => Either[L, B]): Either[L, B] = ma.right.flatMap(f)

    override def fmap[A, B](ma: Either[L, A])(f: (A) => B): Either[L, B] = ma.right.map(f)
  }
}
```

In this code we fixed _bias_ and type kind in one go. (Sorry for the Greek letters, I have some weakness for them :-) ). In case you didn't come across _type lambdas_ syntax before,  ```{ type λ[β] = Either[L, β] })#λ``` means: produce a new (anonymous) type with one hole but equivalent to `Either[L, β]` where L has been fixed. We'll be using type lambdas quite a lot throughout this series, so please make sure that you understand this strange syntax. Implementation-wise we bind and map only with right projection thus giving `Either` right bias we so much needed.

### Implement Kleisli and Kleisli arrow

`Kleisli` type as such is just a wrapper over functions of type `A => M[B]`. Still, we can put a few operations into it, provided that `M[_]` is an instance of monad eg. `andThen` and `compose` variants that we'll reuse for arrow composition and `map` function that we'll use later.

TODO: link to GitHub

```scala
final case class Kleisli[M[_], A, B](run: A => M[B]) {
  import Monad._
  import Kleisli._

  def apply(a: A) = run(a)

  def >=>[C](k: Kleisli[M, B, C])(implicit m: Monad[M]): Kleisli[M, A, C] = Kleisli((a: A) => this(a) >>= k.run)
  def andThen[C](k: Kleisli[M, B, C])(implicit m: Monad[M]): Kleisli[M, A, C] = this >=> k

  def >==>[C](k: B => M[C])(implicit m: Monad[M]): Kleisli[M, A, C] = this >=> Kleisli(k)
  def andThenK[C](k: B => M[C])(implicit m: Monad[M]): Kleisli[M, A, C] = this >==> k

  def <=<[C](k: Kleisli[M, C, A])(implicit m: Monad[M]): Kleisli[M, C, B] = k >=> this
  def compose[C](k: Kleisli[M, C, A])(implicit m: Monad[M]): Kleisli[M, C, B] = k >=> this

  def <==<[C](k: C => M[A])(implicit m: Monad[M]): Kleisli[M, C, B] = Kleisli(k) >=> this
  def composeK[C](k: C => M[A])(implicit m: Monad[M]): Kleisli[M, C, B] = this <==< k

  def map[C](f: B ⇒ C)(implicit m: Monad[M]): Kleisli[M, A, C] = Kleisli((a: A) => m.fmap(this(a))(f))
}

object Kleisli extends KleisliInstances {
  implicit def kleisliFn[M[_], A, B](k: Kleisli[M, A, B]): (A) ⇒ M[B] = k.run
}
```

To get back an ordinary function from `Kleisli` without much hassle I have added an implicit conversion from `Kleisli` to function type.
The most important building block of Kleisli composition is this piece of code 
```Kleisli((a: A) => this(a) >>= k.run)```. It gives a recipe of monadic composition: get a monad instance and use this monad's `bind` with a function that takes value out of monad and produces new monad. `Monad` takes care of plumbing. If you look closer that's exactly what we missed when we tried to use `Either` with arrows - now instead of implementing it by hand and cluttering our code we extracted this boilerplate into monads and arrows. What is left to be done is to implement `Arrow` made with `Kleisli`s. That's easy enough:

TODO: link to GitHub
```scala
trait KleisliInstances {
  //kleisli (a => m b) is arrow
  abstract class KleisliArrow[M[_]] extends Arrow[({ type λ[α, β] = Kleisli[M, α, β] })#λ] {
    import Kleisli._
    import Monad._

    implicit def M: Monad[M]

    override def id[A]: Kleisli[M, A, A] = Kleisli(a => M.point(a))

    override def arr[A, B](f: (A) => B): Kleisli[M, A, B] = Kleisli(a => M.point(f(a)))

    override def first[A, B, C](f: Kleisli[M, A, B]): Kleisli[M, (A, C), (B, C)] = Kleisli {
      case (a, c) => f(a) >>= ((b: B) => M.point((b, c)))
    }

    override def second[A, B, C](f: Kleisli[M, A, B]): Kleisli[M, (C, A), (C, B)] = Kleisli {
      case (c, a) => f(a) >>= ((b: B) => M.point((c, b)))
    }

    override def compose[A, B, C](fbc: Kleisli[M, B, C], fab: Kleisli[M, A, B]): Kleisli[M, A, C] = fab >=> fbc
  }

  implicit def kleisliArrow[M[_]](implicit m: Monad[M]) = new KleisliArrow[M] {
    override implicit def M: Monad[M] = m
  }
}

```

Let's explain a few details. Once again we needed to use _type lambdas_ to trick Scala into thinking that `Kleisli` is a right type parameter to `Arrow` trait. We abstracted out monad leaving only arrow's input and output. All the methods rely on `point` to create a new monad instance and `bind` to push a value out of a monad into another one. Thus, you can't have Kleisli arrow without Monad, these two need to go together. To implement `compose` we just used `andThen` syntax from `Kleisli`. We also have to tell the compiler how to convert our new arrow to `ArrowOps` to use convenient syntax:

```scala
  implicit def ToArrowOpsFromKleisliLike[G[_], F[G[_], _, _], A, B](v: F[G, A, B])(implicit arr: Arrow[({ type λ[α, β] = F[G, α, β] })#λ]) =
    new ArrowOps[({ type λ[α, β] = F[G, α, β] })#λ, A, B](v)
```

This looks terrible, but well, what can we do. Previous definition was not adequate because it insisted on type of F being of `* -> * ->*` while `Kleisli` has got `(* -> *) -> * -> * -> *` kind. 
Now we are finally able to incorporate error handling into our flow

###Implement error handling using Kleisli arrows

Let's go back to the drawing board. The flow can be much simplified because we won't be relying on side effects to propagate errors. What's more, handling of errors will be responsibility of an `Either` monad, so we do not have to write any code or alter the logic to take care of this. Thus, we can enjoy smooth linear flow. A railway, if you will. Hence the term railway oriented programming, coined by Scott Wlashin.

![](http://virtuslab.com/wp-content/uploads/2015/10/kleisli_railway.png) 

Red lines represent 'error handling track'. Their presence is caused by how we implemented `bind` in the `Either` monad i.e. binding is _right biased_. When `Left` value is encountered, it breaks the flow. Bold blue lines represent _Kleisli composition_, they combine _monadic values_ together by unpacking them and feeding the next computation. One problem you might have noticed is the copying function represented by the dotted rectangle. It is not monadic and as such cannot participate in the normal flow. 

#####Integrating non-monadic functions

When composing monadic functions we obtain a monad instance `M[B]` from `M[A]` by means of `A => M[B]`, a binding function. Non-monadic functions, on the other hand, are of form `A => B`. That's precisely the difference between `flatMap` and `map`, and that's the very reason we implemented `map` function on `Kleisli`. Mapping lets us obtain a new monad instance by means of a function that does not have to do anything with monads. To recap - if you have a need to integrate non-monaidc functions into a pipeline, use `map`.

Back to the code. Now, our business logic can handle errors and normal flow with ease:

```scala
  type E[R] = Either[Error, R]

  private def productionLotArrow[Env](verify: (ProductionLot, Env) => E[ProductionLot],
                                      copy: (ProductionLot, Env) => ProductionLot): Env => Long => E[Long] = {
    val verifyProductionLotNotDoneF: (ProductionLot) => E[ProductionLot] = verifyProductionLotNotDone

    (env: Env) => (
      Kleisli[E, Long, ProductionLot]{ productionLotsRepository.findExistingById }
      >>> Kleisli { verify(_, env) }
      >>> Kleisli { verifyProductionLotNotDoneF }
    ).map(copy(_, env)) >==> productionLotsRepository.save _
  }
```

We can cross out another requirement: **exceptions are gone**.
On the other hand we came across some difficulties. First of all, passing the environment had been quite a hassle. Environment represented additional data that logic needed to operate on, and it didn't adhere to our monadic model. That forced us to rewrite the code a bit, leveraging the fact that what we've got after all is functions, and functions can be partially applied and curried. That's how we were able to completely eliminate `env` parameter from the equation. 
Second, Scala will not understand the equivalence between `Either[A, B]` and `Either[Error, B]` unless it is explicitly stated as a type alias and asserted *everywhere* to the compiler that we can mix these two. 

>no type parameters for method ☆: (f: A => M[B])org.virtuslab.blog.kleisli.Kleisli[M,A,B] exist so that it can be applied to arguments (org.virtuslab.blog.kleisli.ProductionLot => Either[org.virtuslab.blog.kleisli.Error,org.virtuslab.blog.kleisli.ProductionLot])
>[error]  --- because ---
>[error] argument expression's type is not compatible with formal parameter type;
>[error]  found   : org.virtuslab.blog.kleisli.ProductionLot => Either[org.virtuslab.blog.kleisli.Error,org.virtuslab.blog.kleisli.ProductionLot]
>[error]  required: ?A => ?M[?B]
>[error]         ☆(verify(_, env)) >>>


I will show you how to get rid of this useless type alias along with line-noise like 
```scala 
val verifyProductionLotNotDoneF: (ProductionLot) => E[ProductionLot] = verifyProductionLotNotDone
``` 
in part two.

###Summary

In this part we've been able to achieve quite a lot. Starting from uncomposable, imperative code with side effects, we've slowly paved our way to a proper functional
code with error handling. It's as if we've built ourselves a little language to help us express various aspects of control flow normally implemented in imperative manner, without compromising on composability. In the next part we'll extend this example by implementing more requirements that you'd often find in coding business logic eg. logging (side-effects). I'll also explain various ways to make the code more readable by eliminating ugly looking _type lambdas_ and superfluous type tricks. Further down the road we'll look at the _env param_ once again and devise a way to handle it more cleanly, thus giving birth to arrow transformers. Stay tuned.