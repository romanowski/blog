Smooth operator with Slick 3
==============
Or how to use custom / database specific operators in Slick 3.x.

![Tool image](tools-1551459_640.jpg)

Couple of days ago we had our company Xmas dinner. As we are a software company we obviously talked mostly about technology (surprise!). One of the conversation we had was about a comparison: [Slick](https://slick.lightbend.com) vs [Quill](http://getquill.io/). It was about difficulties of adding customization. Specifically, it was about a problem of adding custom operator that was required for performing certain calculations on database level. As I was recently having a talk about [Slick 3.x on BeeScala](https://www.youtube.com/watch?v=t1EbXHVJXfs) [(slides here)](http://slides.com/pdolega/slick-101) and had an opportunity to dig a little bit deeper into Slick building blocks I felt I could give this problem a try. 

Problem
-------------
So the problem is: **how do you add database specific operator to Slick**. Mentioned operator is in this case binary ^ (or XOR). It may be useful for instance if you need to calculate some flavor of [Hamming distance](https://en.wikipedia.org/wiki/Hamming_distance) on your database for some binary strings (e.g. *integers*). At least this was the case that we discussed.

Things like that (custom extensions in DSL) are often considered to be hard in Slick. This is often put in contrast to Quill, where there is a lot of flexibility on how you can extend SQL generation. Yet, as I had spent some time digging a little bit deeper into Slick recently, I instantly had a feeling that this is something that should be doable (and without too much of hustle).

Laying out the groundwork
-------------
Before we commence: 
* all the code is available on GitHub [here](https://github.com/pdolega/slick-xor) (together with a small set of unit tests to present to working code).
* I will be using [Slick 3.2.0-M2](http://slick.lightbend.com/doc/3.2.0-M2) (which changes few imports but alas not really significantly) and Scala 2.12 (which doesn’t change anything relevant, at least not in the context of our task here).

We need to start with something. To verify my solution works I created following table:

```scala
CREATE TABLE slick_xor.xor_test (
    id BIGINT AUTO_INCREMENT NOT NULL PRIMARY KEY,
    number1 INTEGER NOT NULL,
    number2 INTEGER NOT NULL,
    number3 INTEGER
);
```

and mapped it into Slick definition (or in other words — created both *mixed* and *unpacked* type — as it is called in Slick terminology).

```scala
import slick.jdbc.MySQLProfile.api._

object Model {
  case class XorTest(number1: Int,
                     number2: Int,
                     maybeNumber3: Option[Int],
                     id: Long = -1)

  class XorTestTable(tag: Tag) extends Table[XorTest](tag, _schemaName = Option("slick_xor"), "xor_test") {
    def number1 = column[Int]("number1")
    def number2 = column[Int]("number2")
    def maybeNumber3 = column[Option[Int]]("number3")
    def id = column[Long]("ID", O.PrimaryKey, O.AutoInc)

    def * = (number1, number2, maybeNumber3, id) <> (XorTest.tupled, XorTest.unapply)
  }

  lazy val XorTestTable = TableQuery[XorTestTable]
}
```

After that we are ready to experiment.

Workaround solution
-------------
I didn’t know actually how to start with overloading actual database operator (I had some thoughts though — about which later). I did know however that Slick allows you to add custom functions. You can read about it [here (Slick docs)](http://slick.lightbend.com/doc/3.2.0-M2/userdefined.html#scalar-database-functions). 

The reasoning behind is this: even if you can’t support some specific database expressions in Slick DSL you often should be able to wrap it within function that you can call later on (assuming it is a *scalar function* — that is function that returns simple value; this is in contrast to function returning e.g. database cursor which Slick does not support easily within DSL).

Creating such a wrapping function is trivial:

```sql
CREATE FUNCTION xorf (a INTEGER, b INTEGER)
RETURNS INTEGER DETERMINISTIC
RETURN a ^ b;
```

I named it *xorf* to avoid clash with build-in xor function in MySQL.

The only thing we need to do on Scala side is this definition:

```scala
val xor = SimpleFunction.binary[Int, Int, Int]("xorf")
```

Now if we put it somewhere in our scope, we can use it like this:

```scala
db.run(
    XorTestTable
      .map(_ => xor(282, 119))
      .take(1)
      .result
)
```


or like this:

```scala
db.run(
    (for {
      x <- XorTestTable
                 if x.number3 === xor(x.number1, x.number2)            
    } yield (x)).result
)
```

This Slick query above generates exactly the SQL we would expect:

```sql
select `number1`, `number2`, `number3`, `ID` 
from `slick_xor`.`xor_test` 
where `number3` = xorf(`number1`,`number2`)
```


Cool, we got a fallback solution then. Well, kind of.

We still won’t be able to compile following query:

```scala
db.run(
    XorTestTable
      .map(x => xor(x.number1, x.maybeNumber3)) // here we have Rep[Int] and Rep[Option[Int]]
      .result
)
```

It yields following compilation error:

```
Error:(66, 81) type mismatch;
 found : slick.lifted.Rep[Option[Int]]
 required: slick.lifted.Rep[Int]
 XorTestTable.map(x => xor(x.number1, x.maybeNumber3)).result
```
 
Reason? Our function takes two `Int`s as parameters and return another `Int`. However we tried to provide `Option[Int]` as one of the parameters above. But there is an easy fix. Let’s overload our function so we have a version that deals with `Option`s.

```scala
def xor = SimpleFunction.binary[Int, Int, Int]("xorf")

  def xor(rep1: Rep[Option[Int]], rep2: Rep[Option[Int]]) = {
    val func = SimpleFunction.binary[Option[Int], Option[Int], Option[Int]]("xorf")
    func(rep1, rep2)
  }
```

Now we would only need to remember to wrap our Int parameter into `Option[Int]` in case we call xor with mixed parameters (one being pure Int and the other being `Option[Int]`). 

```scala
db.run(
    XorTestTable
      .map(x => xor(x.number1.?, x.maybeNumber3)) // added ? to number1
      .result
)
```


In fact we don’t even need to remember it that much. In case we forget actual compiler error is pretty user-friendly:

```
Error:(66, 70) method columnToOptionColumn in trait API is deprecated (since 3.0): Use an explicit conversion to an Option column with `.?`
XorTestTable.map(x => xor(x.number1, x.maybeNumber3)).result
```

Neat!

Still it is only a workaround solution. Let’s try to dig deeper and hopefully introduce our custom operator into Slick DSL.

Proper solution
-------------
This one is trickier. I didn’t find much helpful information in Slick docs. However I did once answered a related question on SO [(here)](http://stackoverflow.com/a/40915510/2239369). It was about UUID but forced me to tinker with profile extensions. And this is precisely where we should turn our attention to. Also it would be reasonable to take a look how existing Slick operators are implemented. Let’s take for example logical operators:

```scala
/** Extension methods for Rep[Boolean] and Rep[Option[Boolean]] */
final class BooleanColumnExtensionMethods[P1](val c: Rep[P1]) extends AnyVal with ExtensionMethods[Boolean, P1] {
  protected[this] implicit def b1Type = implicitly[TypedType[Boolean]]

  def &&[P2, R](b: Rep[P2])(implicit om: o#arg[Boolean, P2]#to[Boolean, R]) =
    om.column(Library.And, n, b.toNode)
  def ||[P2, R](b: Rep[P2])(implicit om: o#arg[Boolean, P2]#to[Boolean, R]) =
    om.column(Library.Or, n, b.toNode)
  def unary_! = Library.Not.column[P1](n)
}
```


That is a nasty piece of Scala library code. Actually it is probably not that bad but it may look scary on the first glimpse. Let’s decompose what’s going on here. This `BooleanColumnExtensionMethods` is used for implicit conversion between `Rep[P1]` (representation of our column) and a type that implements these new operators (`&&`, `||` and `!` in this case). All this is pulled into scope when you import driver/profile API like this:

```scala
import slick.jdbc.MySQLProfile.api._
```

Here is implementation of our Custom MySQL profile:

```scala
trait ExtendedMySQLProfile extends MySQLProfile {
  trait ExtApi extends API {
    implicit def extendedIntColumn(c: Rep[Int]): BaseIntExtendedIntMethods[Int] = new BaseIntExtendedIntMethods[Int](c)
    implicit def extendedIntOptionColumn(c: Rep[Option[Int]]): BaseIntExtendedIntMethods[Option[Int]] = new BaseIntExtendedIntMethods[Option[Int]](c)
  }

  override val api: ExtApi = new ExtApi {}

  final class BaseIntExtendedIntMethods[P1](val c: Rep[P1]) extends BaseExtensionMethods[P1] {
    def ^[P2, R](e: Rep[P2])(implicit om: o#arg[P1, P2]#to[P1, R]) =
      om.column(Operators.^, n, e.toNode)
  }

  object Operators {
    val ^ = new SqlOperator("^")
  }
}

object ExtendedMySQLProfile extends ExtendedMySQLProfile
```


This may be a little too involved so let’s once again decompose what we have here. 
First this part below directly mimics what we have in MySQL driver for natively supported operations already mentioned (e.g. `&&`):

```scala
  final class BaseIntExtendedIntMethods[P1](val c: Rep[P1]) extends BaseExtensionMethods[P1] {
    def ^[P2, R](e: Rep[P2])(implicit om: o#arg[P1, P2]#to[P1, R]) =
      om.column(Operators.^, n, e.toNode)
  }

  object Operators {
    val ^ = new SqlOperator("^")
  }
```

Our column representation will be wrapped with object of `BaseIntExtendedIntMethods` class which knows how to handle `^` operation.

As always, to add new operation to our existing type, we need to perform implicit conversion. This is done simply by this part:

```scala
implicit def extendedIntColumn(c: Rep[Int]): BaseIntExtendedIntMethods[Int] = 
  new BaseIntExtendedIntMethods[Int](c)

implicit def extendedIntOptionColumn(c: Rep[Option[Int]]): BaseIntExtendedIntMethods[Option[Int]] = 
  new BaseIntExtendedIntMethods[Option[Int]](c)
```

The only question remaining is how do we bring all of this into scope automatically with our driver/profile (so we don’t need to remember about additional imports every time we use this operation). To achieve that we need to add our implicit conversions to our API:

```scala
trait ExtendedMySQLProfile extends MySQLProfile {
  trait ExtApi extends API {
    // ...
    // here goes our implicit methods
  }

  override val api: ExtApi = new ExtApi {}

  // ...
  // here goes definition of our class implementing ^ operation
}

object ExtendedMySQLProfile extends ExtendedMySQLProfile
```

Parts of code already discussed before are removed in above sample for clarity. Other than that, it is exactly the code we started with. 

Now instead of using regular MySQL profile API import we would need to use our new one in all Slick-related source files:


`import ExtendedMySQLProfile.api._`

This basically wraps up our solutions.

Let’s see now how Slick query would look like.

```scala
db.run(
    XorTestTable
      .map(row => row.number1 ^ row.number2)
      .take(1)
  )
```


And generated query:

```sql
select `number1` ^ `number2` 
from `slick_xor`.`xor_test` 
limit 1
```


Perfect! This is precisely what we wanted to achieve. We can even mix both plain `Int`s and `Option[Int]`s without any modifications:

```scala
db.run(
    XorTestTable
      .map(row => row.number1 ^ row.number2 ^ row.maybeNumer3)
      .take(1)
  )
```


Not surprisingly it would generate following SQL:

```sql
select (`number1` ^ `number2`) ^ `maybeNumber3` 
from `slick_xor`.`xor_test` 
limit 1
```


Conclusion
-------------
We went through two variants of delivering required functionality:
* wrapping our operator within simple scala SQL function
* adding ^ operator to Slick DSL

Neither of these solutions required much coding nor sophisticated solutions (it is basically couple of lines here and there). It is also worth to emphasize here that they are still pretty much type-safe: our implementation of ^ will only work with `Int` or `Option[Int]` — anything else will result in compilation error. 

Apparently, contrary perhaps to general opinion, you are able fairly easily to customize some parts of generated SQL in Slick. 

Whole code together with unit tests is available here.
