# Arrows, Monads and Kleisli - part II

In [part I](http://virtuslab.com/blog/arrows-monads-and-kleisli-part-i) I showed how Kleisli arrows could be used to implement domain modeling. Arrows serve as a foundation for a 'DSL' in which one can implement typical scenarios that arise in business-logic code: decoupling flow control from domain code, dealing with errors etc. Much to the spirit of [Railway Oriented Programming](http://fsharpforfunandprofit.com/posts/recipe-part2), but implemented in more generic terms. 
In this part I'll fill in the missing parts of this 'framework' - taking care of side-effects, conditional execution and mixing different monads in a single 'pipeline'.
There is one more part planned where I'll introduce kind-projector and arrow transformers.
Let's pick up where we left:

```scala
  private def productionLotArrow[Env](verify: (ProductionLot, Env) => Either[Error, ProductionLot],
                                      copy: (ProductionLot, Env) => ProductionLot): Env => Long => Either[Error, Long] = {
    type Track[T] = Either[Error, T]
    def track[A, B](f: A => Track[B]) = Kleisli(f)

    val getFromDb = track { productionLotsRepository.findExistingById }
    val validate = (env: Env) => track { verify(_: ProductionLot, env) } >>> track { verifyProductionLotNotDone }
    val save = track { productionLotsRepository.save }

    (env: Env) => (
      getFromDb
      >>> validate(env)
    ).map(copy(_, env)) >>> save
  }
```
Having glued together actions needed to carry out domain operations, we'd certainly like to be able to inspect what's going on during execution. Typically that means plugging in some kind of logging framework. Let's add another **requirement: add logging**. While logging itself is a no-brainer, it is problematic from functional point of view because of `String => Unit` signature. It returns no useful value, thus it is meant to be executed only for the side-effect occurring when it is being called. In the language of Railway Oriented Programming this type of function is called dead-end function. Dead-end means that it cannot be composed any further. Yet we certainly want to be able to use them in a computation, mainly because many tasks are naturally side-effecting _actions_ (like it or not).

###Deal with dead-end functions

Clearly we're gonna need to reroute main flow around dead-end. _Action_ does not produce any useful value (but probably needs access to the current input), so composition with dead-end will just return its input, capturing the side-effect within what looks like an identity process from the outside. Let's make a small building block called _tee_ for this purpose

![](http://virtuslab.com/wp-content/uploads/2015/11/kleisli_tee.png)

If you read the first part, it'd probably ring a bell - _tee_ looks very much alike arrow _split_ operator.
That's right - in arrow language: `tee(f) = (f &&& action) >>> _._1`.
Let's add this little function to `Arrow` trait:

[Code](https://github.com/VirtusLab/kleisli-examples/blob/46d28bb924e193936e5cf306551b24943cfa957c/src/main/scala/org/virtuslab/blog/kleisli/Arrow.scala)
```scala
def tee[A, B](f: A =>: B, action: A => Unit): A =>: B = compose(arr((x: (B, Unit)) => x._1), split(f, arr(action)))
```

Using tee we can devise a _compose-with-dead-end_ operator for `ArrowOps` (second diagram above). I thought that `-|` would be a pretty good symbol for this :-) (looks like turned dead-end sign, doesn't it?)

```scala
def -|(action: B => Unit): A =>: B = >>>(arr.tee(arr.id, action))
```

This is sufficient to add logging to the `ProductionLotService` but looking at the flow we discover one quirk we'd yet like to address. How are we going to log the result? 
Result is of `Either[Error, Long]` type. When computation signals error we might want to log with higher than usual level or even perform additional steps. This obviously could be achieved through pattern-matching but weren't arrows supposed to free code from the unnecessary boilerplate? 
It turns out that choice of execution path can be easily expressed using arrows. Enter `ArrowChoice`

###Implement ArrowChoice

`ArrowChoice` is a specialization of arrow that can, as the name points out, choose which computation to perform next based on its input. It is the equivalent of `if` construct in for-comprehensions and is designed to work well with `Either` data type. The minimal definition of `ArrowChoice` calls for two primitive operations: `left` and `right` from which two compound operations are built: `multiplex(+++)` and `fan-in(|||)`. A picture is worth thousand words, so let's start with diagrams.

![](http://virtuslab.com/wp-content/uploads/2015/11/kleisli_choice.png) 

Here we have two possible paths of execution (marked red and blue) based on data held in an `Either` instance passed as input. 
`left(f)` feeds `Left` input through its argument arrow, leaving the other part unchanged. `right(f)` mirrors `left`. Thinking in terms of Scala library, these operators are maps on, respectively, `Left` and `Right` projections. 
`Multiplex(+++)` simply combines both `left` and `right` into a single operation. It applies `f` if input is a `Left` or `g` if it is a `Right`
`Fanin(|||)` is `+++` which merges the output. Think Scala's `Either.fold`.

Implementation is straightforward:

[Code](https://github.com/VirtusLab/kleisli-examples/blob/2041b8db7d3557895cfa1fb7d0f2e40cd9e821bd/src/main/scala/org/virtuslab/blog/kleisli/Choice.scala)
```scala
trait Choice[=>:[_, _]] { self: Arrow[=>:] =>
  def left[A, B, C](f: A =>: B): Either[A, C] =>: Either[B, C]
  def right[A, B, C](f: A =>: B): Either[C, A] =>: Either[C, B]

  def multiplex[A, B, C, D](f: A =>: B, g: C =>: D): Either[A, C] =>: Either[B, D] = compose(left(f), right(g))

  def fanin[A, B, C](f: A =>: C, g: B =>: C): Either[A, B] =>: C = {
    val untag: Either[C, C] => C = {
      case Left(x)  => x
      case Right(y) => y
    }
    compose(arr(untag), multiplex(f, g))
  }
}
```

... and additional symbolic ops:

```scala
final class ChoiceOps[=>:[_, _], A, B](val self: A =>: B)(implicit val choice: Choice[=>:]) {
  def +++[C, D](g: C =>: D): Either[A, C] =>: Either[B, D] = choice.multiplex(self, g)
  def |||[C](g: C =>: B): Either[A, C] =>: B = choice.fanin(self, g)
}

object Choice {
  implicit def ToChoiceOps[F[_, _]: Choice, A, B](v: F[A, B]): ChoiceOps[F, A, B] = new ChoiceOps(v)
}
```

What's left to be done is to implement `Choice` for arrow instances that support it. Both function and Kleisli do, but to keep it simple we'll do that only for functions (after all, we're using it for logging only):

[Code](https://github.com/VirtusLab/kleisli-examples/blob/2041b8db7d3557895cfa1fb7d0f2e40cd9e821bd/src/main/scala/org/virtuslab/blog/kleisli/ArrowInstances.scala)
```scala
  implicit object FunctionArrow extends Arrow[Function1] with Choice[Function1] {

// ...

    override def left[A, B, C](f: (A) => B): (Either[A, C]) => Either[B, C] = _.left.map(f)

    override def right[A, B, C](f: (A) => B): (Either[C, A]) => Either[C, B] = _.right.map(f)

    override def multiplex[A, B, C, D](f: (A) => B, g: (C) => D): (Either[A, C]) => Either[B, D] = {
      case Left(a)  => Left(f(a))
      case Right(c) => Right(g(c))
    }

    override def fanin[A, B, C](f: (A) => C, g: (B) => C): (Either[A, B]) => C = _.fold(f, g)
  }
```

Of course, it would have sufficed to implement just `left` and `right` but, as we observed, for functions it is perfectly possible to write simpler code, based on Scala standard library. Only `multiplex` is not equivalent to any method. It comes as no surprise because, as was said before, `Either` lacks _bias_, forcing us to choose explicitly which side we want to operate on. If it had been biased, `+++` would have been `map`.

The stage is set to add logging to `ProductionLotService`. Let's extend `productionLotArrow` to accept custom logging function of type `(ProductionLot, Env) => Unit` (after all, each caller wants a different message to be logged). Additionally, arrow implementation will log the outcome of flow (either success or error) by itself. Putting it together we get:

[Code](https://github.com/VirtusLab/kleisli-examples/blob/960d232b86336c57a9b72eb9ef368a57d4027581/src/main/scala/org/virtuslab/blog/kleisli/ProductionLotsService.scala)
```scala
  private val logger = Logger.getLogger(this.getClass.getName)

  private def productionLotArrow[Env](verify: (ProductionLot, Env) => Either[Error, ProductionLot],
                                      copy: (ProductionLot, Env) => ProductionLot,
                                      log: (ProductionLot, Env) => Unit): Env => Long => Either[Error, Long] = {
    type Track[T] = Either[Error, T]
    def track[A, B](f: A => Track[B]) = Kleisli[Track, A, B](f)

    val getFromDb = track { productionLotsRepository.findExistingById }
    val validate = (env: Env) => track { verify(_: ProductionLot, env) } >>> track { verifyProductionLotNotDone }
    val save = track { productionLotsRepository.save }

    val logError: Error => Unit = error => logger.warning(s"Cannot perform operation on production lot: $error")
    val logSuccess: Long => Unit = id => logger.fine(s"Production lot $id updated")

    (env: Env) =>
      ((
        getFromDb -| (log(_, env))
        >>> validate(env)).map(copy(_, env))
        >>> save)
        .run -| (logError ||| logSuccess)
  }
```

With a little help of `-|` operator, logging is painlessly stitched to where it belongs.
Curious reader might be asking, why is there a `run` called? (`run` extracts a function from `Kleisli` and there is an implicit conversion for that). Well, the short answer is that it's because implicit conversions in Scala aren't chained. Now, the long answer: to deduce that the correct call is actually `-|` coming from function arrow (not the one from Kleisli arrow), compiler would have had to construct chain of conversions leading from `Kleisli` through `Function1` (via `run` method) to `ArrowOps(FunctionArrow)`. Instead it sees that `-|` can be obtained by one step conversion `Kleisli` -> `ArrowOps(KleisliArrow)`, and sticks to that.  But `KleisliArrow` requires `Long => Unit` action, while what we've got is a conditional `Either[Error, Long] => Unit`. So we have to manually unpack function from `Kleisli`.

After all this we can freely enhance different use cases with logging:

```scala
  private case class StartProduction(productionStartDate: Date, workerId: Long)
  private val startProductionA = productionLotArrow[StartProduction](
    (pl, env) => verifyWorkerCanBeAssignedToProductionLot(pl, env.workerId),
    (pl, env) => pl.copy(
      productionStartDate = Some(env.productionStartDate),
      workerId = Some(env.workerId),
      status = ProductionLotStatus.InProduction
    ),
    (pl, env) => logger.fine(s"Starting production of $pl on ${env.productionStartDate} by ${env.workerId}")
  )
```

###Mixing monads

I promised to show how to face the situation when functions that you want to build pipeline from can't agree on what monad they use. Let's imagine that `save` method wants to return `Try` instead of `Either`:
```scala
def save(productionLot: ProductionLot): Try[Long] = ???
```
Obviously we also need to state that `Try` is actually a monad (there is some controversy about that, but for the sake of this post let's slide over that):

[Code](https://github.com/VirtusLab/kleisli-examples/blob/f2f02fc45b2dace17e40b0bb2bbafdd1086310d3/src/main/scala/org/virtuslab/blog/kleisli/KleisliInstances.scala)
```scala
  implicit object TryMonad extends Monad[Try] {
    override def point[A](a: => A): Try[A] = Success(a)

    override def bind[A, B](ma: Try[A])(f: (A) => Try[B]): Try[B] = ma flatMap f

    override def fmap[A, B](ma: Try[A])(f: (A) => B): Try[B] = ma map f
  }
```

There is no one-size-fits-all answer I believe, but I'll show you two possible ways of solving this awkwardness.

##### Lift

The crucial observation here is that if you have a function `A => B` and a functor `F[A]`, you can easily transform the function into a function of `F[A] => F[B]` type. We've already seen that in action when we integrated non-monadic functions with `Kleisli`. It's exactly what `map` does. If you recall, `map`'s signature on `Kleisli[M, A, B]` (a function `A => M[B]`) is:
```scala
def map[C](f: B ⇒ C)(implicit m: Monad[M]): Kleisli[M, A, C] = Kleisli((a: A) => m.fmap(this(a))(f))
```
`map` composes `B => C` with `A => M[B]` to get `A => M[C]`. In other words it turns a function of type `B => C`  into a `M[B] => M[C]` function.
This is possible with all functors, and, as you probably remember from part one, every monad is a functor.
Now, we can pretend that function `A => Try[B]` is kind of _non-monadic_ from the perspective of `Either` monad and write this as follows:

```scala
//...
    val save = Kleisli { productionLotsRepository.save }
// ...
    (env: Env) =>
      (
        getFromDb -| (log(_, env))
        >>> validate(env))
        .map(copy(_, env))
        .map(save)
        .run -| (logError ||| logSuccess)
```

Or, we can name this operation explicitly by adding `liftM` method to `Kleisli` class:

[Code](https://github.com/VirtusLab/kleisli-examples/blob/f2f02fc45b2dace17e40b0bb2bbafdd1086310d3/src/main/scala/org/virtuslab/blog/kleisli/Kleisli.scala)
```scala
final case class Kleisli[M[_], A, B](run: A => M[B]) {
//...
  def liftM[N[_]](implicit n: Monad[N]): Kleisli[N, A, M[B]] = Kleisli((a: A) => n.point(this(a)))
}
```
which turns arrow definition into:

```scala
    (env: Env) =>
      ((
        getFromDb -| (log(_, env))
        >>> validate(env)).map(copy(_, env))
        >>> save.liftM[Track])
        .run -| (logError ||| logSuccess)
```

This approach works fine, but it adds a lot of burden when you want to do anything with the result. The whole arrow definition after these changes looks like this:

[Code](https://github.com/VirtusLab/kleisli-examples/blob/f2f02fc45b2dace17e40b0bb2bbafdd1086310d3/src/main/scala/org/virtuslab/blog/kleisli/ProductionLotsService.scala)
```scala
  private def productionLotArrow[Env](verify: (ProductionLot, Env) => Either[Error, ProductionLot],
                                      copy: (ProductionLot, Env) => ProductionLot,
                                      log: (ProductionLot, Env) => Unit): Env => Long => Either[Error, Try[Long]] = {
    type Track[T] = Either[Error, T]
    def track[A, B](f: A => Track[B]) = Kleisli[Track, A, B](f)

    val getFromDb = track { productionLotsRepository.findExistingById }
    val validate = (env: Env) => track { verify(_: ProductionLot, env) } >>> track { verifyProductionLotNotDone }
    val save = Kleisli { productionLotsRepository.save }

    val logError: Error => Unit = error => logger.warning(s"Cannot perform operation on production lot: $error")
    val logSuccess: Try[Long] => Unit = {
      case Success(id) => logger.fine(s"Production lot $id updated")
      case Failure(ex) => logger.warning(s"Save error: $ex")
    }

    (env: Env) =>
      ((
        getFromDb -| (log(_, env))
        >>> validate(env)).map(copy(_, env))
        >>> save.liftM[Track])
        .run -| (logError ||| logSuccess)
  }
```
The bad thing is that type of the constructed function changes from nice `Either[Error, Long]` into less-than-perfect `Either[Error, Try[Long]]`. You can see how it affected logging. `logSuccess` needs to accomodate the additional layer of types and be rewritten from `Long => Unit` into `Try[Long] => Unit` and that, in turn, forces us to pattern-match on input. It's not the end of the world though. What are the other options?

#####Natural transformation
There are tons of material treating the subtleties of natural transformations, ranging from [deep theoretical explanations](http://milessabin.com/blog/2012/05/10/shapeless-polymorphic-function-values-2/) to [more practical ones](http://eed3si9n.com/learning-scalaz/Natural-Transformation.html).
The definition from [Wikipedia](https://en.wikipedia.org/wiki/Natural_transformation) states simply that:
>a natural transformation provides a way of transforming one functor into another while respecting the internal structure (i.e. the composition of morphisms)

We'll not be doing anything fancy though. For our purposes a natural transformation is just a generalization of map (`A => B`) that works on higher-order types (`M[_] => N[_]`).
[Code](https://github.com/VirtusLab/kleisli-examples/blob/c53c66cfb091e11611dcf46e0b775f395264b537/src/main/scala/org/virtuslab/blog/kleisli/Kleisli.scala)
```scala
final case class Kleisli[M[_], A, B](run: A => M[B]) {
 //...
  def transform[N[_]](f: M ~> N): Kleisli[N, A, B] = Kleisli(a => f(run(a)))
}

object Kleisli extends KleisliInstances {
//...
  trait NaturalTransformation[-F[_], +G[_]] {
    def apply[A](fa: F[A]): G[A]
  }
  type ~>[-F[_], +G[_]] = NaturalTransformation[F, G]
}
```

The fancy `~>` type used in `transform` behaves exactly like a generalized map that can transform one monad into another, irrespective of type of a value the monad carries inside. BTW what a fine example of higher-order polymorphism.
And `transform` is very simple to use:

[Code](https://github.com/VirtusLab/kleisli-examples/blob/c53c66cfb091e11611dcf46e0b775f395264b537/src/main/scala/org/virtuslab/blog/kleisli/ProductionLotsService.scala)
```scala
    val save = Kleisli { productionLotsRepository.save } transform new (Try ~> Track) {
      override def apply[A](fa: Try[A]): Track[A] = fa match {
        case Success(a)  => Right(a)
        case Failure(ex) => Left(ProductionLotUpdateError(ex))
      }
    }

    val logError: Error => Unit = error => logger.warning(s"Cannot perform operation on production lot: $error")
    val logSuccess: Long => Unit = id => logger.fine(s"Production lot $id updated")

    (env: Env) =>
      ((
        getFromDb -| (log(_, env))
        >>> validate(env)).map(copy(_, env))
        >>> save)
        .run -| (logError ||| logSuccess)
```
One can say that `save`'s just got back on track :-)

###The end

All right, that's all folks. Stay tuned for the last part of series. Hope you enjoyed it so far.