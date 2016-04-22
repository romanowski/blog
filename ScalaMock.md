ScalaMock: macros strike back
==============

Introduction
-------------
ScalaMock is a powerful mocking library written purely in Scala. It uses macros to create mocks. Macros and compile-time reflection allow to create type safe code or manipulate programs. It is almost mythological tool, the Holy Grail of programming languages. Almost all modern languages have it.

All features of ScalaMock are listed on its [website](http://scalamock.org/user-guide/features/) and it may seem like limitless tool. This post presents pitfalls and weaknesses of this library that are very important to remember. There is no perfect tool or library.

All code snippets are written against ScalaMock 3.2.2 and Scala 2.11.8. This library is still under development and anything may change. The project's [roadmap](http://scalamock.org/roadmap/) is also available on the website.

This post is based on my personal experience with ScalaMock and my contribution to issues on GitHub. If you prefer looking at code rather than reading the wall of text you can check [examples on GitHub](https://github.com/bkowalik/mocking-things) where almost everything is expressed.


How does ScalaMock work?
-----------------------------------
As mentioned above, ScalaMock uses macros to generate mock objects. Let's focus on a trivial class as below:
```scala
class Foo {
}
```

There is nothing special about it. For given class `mock[Foo]` macro would generate this:
```scala
{
  final class $anon extends Foo {
    def <init>() = { // the constructor representation in JVM
      super.<init>(); // parent constructor call
      () // Interesting, constructor is just a regular method with special call syntax. It returns void.
    };
    val mock$special$mockName = MockTest.this._factory.generateMockDefaultName("mock").name
  };
  new $anon()
}.asInstanceOf[Foo]
```

If you are interested in JVM capabilities and features vs Java syntax (like constructor comment above) please read [this StackOverflow thread](https://stackoverflow.com/questions/6827363/bytecode-features-not-available-in-the-java-language). It is about byte code in version 6 but main concepts remain the same.

As you can see, it uses inheritance to create a mock class and then returns its instance using `asInstanceOf[T]` (in our case `T` is `Foo`). Because `Foo` does not contain any methods or fields, the mock contains only its name for debugging purposes like printing call traces. It is worth noticing that the mock object is generated at compile time but the behavior is defined in the run-time. This means that, besides using macros in favor of reflection, there is no conceptual major difference between ScalaMock and for example Mockito.

Now we can add a method to `Foo` to show simple mock in action.
```scala
class Foo {
  def bar = 5
}

val m = mock[Foo]

m.bar _ expects() returning 6

println(m.bar) // 6
```

The purpose of this simple example is to outline ScalaMock usage. List of full capabilities is available on the [website](http://scalamock.org/user-guide/features/). Let's look again on the macro-generated code but this time with an additional field and method:

```scala
{
  final class $anon extends Foo {
    def <init>() = {
      super.<init>();
      ()
    };
    val mock$special$mockName = MockTest.this._factory.generateMockDefaultName("mock").name;
    override def bar: Int = $anon.this.mock$bar$0.apply();
    val mock$bar$0: MockFunction0[Int] = new MockFunction0[Int](MockTest.this._factory, scala.Symbol.apply(scala.Predef.augmentString("<%s> %s%s.%s%s").format($anon.this.mock$special$mockName, "Foo", "", "bar", "")))
  };
  new $anon()
}.asInstanceOf[Foo]
```

What we can see from code above is the new member of `Foo` mock. Now you can start to wonder how ScalaMock inserts a call definition to such mock. Do you remember that I wrote mock behavior is run-time? It is not completely true. There is implicit macro application whenever you state your expectations. So, when writing `m.bar _ expects()`, the first thing that happens is implicit macro application. In this case this is `toMockFunction0` which inserts this kind of code:
```scala
m.getClass().getMethod("mock$bar$0").invoke(m).asInstanceOf[org.scalamock.function.MockFunction0[Int]]
```
There are macros up to `toMockFunction22` which means that functions from argumentless up to 22 can be mocked (pro-tip: if you need more than that, think over your codebase). But the part I wrote about run-time definition of call is still valid. As you can see it allows to mock behavior of `mock$bar$0` method.

Beside `mock`s there is something called `stub`s. The same macro generates code but different behavior is available so I will not go into details of `stub`s. Having this knowledge we are ready to explore ScalaMock limitations.

Cannot mock class without parameterless constructor
----------------------------------------------------------------------
This is one of the most hacked and well-known limitations of ScalaMock. It is almost as old as the library. Let's look at the class:
```scala
class Foo(a: Int) {
}
```
what you get from macro is:
```scala
{
  final class $anon extends Foo {
    def <init>() = {
      super.<init>();
      ()
    };
    val mock$special$mockName = MockTest.this._factory.generateMockDefaultName("mock").name
  };
  new $anon()
}.asInstanceOf[Foo]
```
and compiler error `not enough arguments for constructor Foo: (a: Int)Foo`. If you look at generated super call to parent constructor, it is missing parameter. Workaround for that is to inherit from class:
```scala
class Foo(a: Int)
class MockableFoo extends Foo(0)
```
which provides default empty constructor. It works also for `case class`es. In one of next posts I will describe my [pull request](https://github.com/paulbutcher/ScalaMock/pull/134) which solves that issue. For now intermediate class providing proper constructor is only way to solve that.

java.lang.Object methods
--------------------------------
You cannot mock any method that is inherited from `java.lang.Object`. It is understandable that mocking such methods as `finalize`, `wait`, `notify` or `notifyAll` could potentially break something. Most of them are final and, as you can expect, you cannot mock final methods. But exclusion of methods `hashCode`, `equals` and `toString` causes confusion. This is just a theory, let's look at some example which is inspired by one of the [issues](https://github.com/paulbutcher/ScalaMock/issues/130):

```scala
class Foo

val m = mock[Foo]

(m.hashCode _).expects() returning 6

Set(m,m,m,m)
```

Test based on code above fails and the exception is `java.lang.NoSuchMethodException: ... hashCode()`. If you look at first paragraph you will see that there is no `mock` for method inherited from `java.lang.Object`. But if you remove `hashCode` mock and rely on default implementation, test would end successfully.

Scala-Java array interoperability
---------------------------------
This is very annoying [issue](https://github.com/paulbutcher/ScalaMock/issues/136) to which I have responded recently. It cannot be overcome and shows weakness of Scala-Java interoperability. For almost everyone it is well known that using highly functional Scala code in Java is almost impossible. But take a look at following example:
```java
// MemberInterface.java
public interface MemberInterface {}
// Member.java
public class Member implements MemberInterface {}
// ContainterInterface.java
public interface ContainerInterface {
    MemberInterface[] get();
}
// Container.java
public class Container implements ContainerInterface {
    public Member[] get() {
        return new Member[1];
    } 
}
```

Code above is perfectly fine, it compiles, works and there is no catch in it. But let's make a use of it in Scala. Let's say we want to write test which contains:
```scala
val m = mock[Container]
// some other testing
```

And compiler will complain with error `overriding method get in trait ContainerInterface of type ()Array[MemberInterface];`. That is quite interesting error. We did nothing wrong and we want to mock external library.

Following error is related to how arrays are implemented in both languages. In Java all collections are invariant. Generally there is no notion of type variance in collections, but in arrays there is one. In consequence you can emulate variance in Java using arrays. In Scala mutable collections are invariant to prevent `ClassCastException` which is good design. When mock is created it overrides method which returns array but with variant type. Scala compiler cannot resolve that even when you manually override such method and change signature to one from interface. This issue is related to Scala not only to ScalaMock but it is more probable that you encounter such problem using the latter. Remember that! You can inherit from `Container` but there is no way to override method `get` because of Java arrays variance.

Case classes mocking confusion
------------------------------------------
This example also concentrates on the same issue as previous paragraph. Let's concentrate on following code (omit purpose of `case class` with empty constructor):
```scala
case class Foo(a: String)
class MockableFoo extends Foo("")

Set(
  mock[MockableFoo],
  mock[MockableFoo],
  mock[MockableFoo],
  mock[MockableFoo]
 )
```

This code not only compiles but test passes. That's fine. So next step is to add another `mock[MockableFoo]` to `Set`. Guess what? Test fails: `java.lang.NullPointerException` without reason. This example also needs additional explanation. ScalaMock does not allow to mock `class`es without empty default constructor. So there is this trick with inheritance. But back to example. This code reveals two problems. First is that `Set` breaks [Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle). This is significant accusation. Why? Because when we use `Set` as interface we expect very the same behavior of every implementation in regard to interface. So when you create `Set` of any number of elements greater than 0 (empty `Set` is something static) you would expect the same kind of error. So why it does not throw exception when size is less than 5? The very well known fact is that Scala collections are optimized. So that `Set` with less than 5 elements is implemented in functional way as a function `A => Boolean` and it does not invoke `hashCode` when inserting new element. When you exceed this magic threshold of 5, you will get `HashSet` implementation which uses `hashCode` to insert into collection. That is why I made this statement.

After previous paragraph there is still one question pending. Why was the exception thrown? It is caused by Scala `case class` magic. Scala uses `scala.runtime.ScalaRunTime` object to fill overridden methods in `case class` like `hashCode`, `equals` and `toString`. This object is full of util functions which inspect object at run-time and those utils are invoked in `case class` 'magically' implemented methods. But it is still not fully true. Look at following code:
```scala
case class Foo(a: Int)
class MockableFoo extends Foo(0)
case class Bar(a: String)
class MockableBar extends Bar("")

mock[MockableFoo].hashCode
mock[MockableBar].hashCode
```

`NullPointerException` is thrown only by calling `hashCode` on `MockableBar` mock. This is even more weird but still related to `case class` and `scala.runtime.ScalaRunTime`. Without going too much into compiler details, `case class`es overrides methods in a different manner depending on their structure and `scala.runtime.ScalaRunTime` applies different strategies to inspect objects at run-time.

Summary
------------
ScalaMock is a great library. It deals with all Scala features but there are some pitfalls for which it is worth to have idea how to overcome them. I believe that this is not complete list of issues that could happen. In next post I will describe step by step how I made my pull request that solves issue with constructors.