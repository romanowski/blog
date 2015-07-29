# Vals gotcha!
Immutability in Scala is greate. A lot of conceres just go away but ones which stay are not so clear. Look at the code below:
```scala
class Foo {
  val a = 1
  val b = a
}

class Bar extends Foo {
  override val a = 2
}
```
My question is: what is the value of such expression ```(new Bar).b```? In this post I will explain why it is neither *1* nor *2*.
There is available [general overview](http://docs.scala-lang.org/tutorials/FAQ/initialization-order.html) of initialization process for similar problems in Scala with description but this post contains in depth one.

## Actual problem
In every book about Scala you will read something more or less like that: vals are implemented as private fileds of class with public getter. But this is not full truth about vals. Unfortunately this public getter is source of all evil in this example. Look at previous code snippet:
```scala
class Foo {
  val a = 1
  val b = a
}

class Bar extends Foo {
  override val a = 2
}
```

Think of it. What they did that field is private final and you can still inherit it from superclass? How have they done them polimorphic? This thing that makes is possible is polimorphic getter in class body for each val that is in. What is more, fields and methods are in one namespace in Scala. This implies that when you call ```fooInstance.a``` you are actually calling ```public int a()``` from bytecode perspective. So when you override inherited ```vals``` from parent class the compiler override those getters.

## What is the actual value and why?
Look at another code snippet. This time it is bytecode that is generated in constructors:
```asm
  public Foo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #22                 // Method java/lang/Object."<init>":()V
         4: aload_0
         5: iconst_1
         6: putfield      #14                 // Field a:I
         9: aload_0
        10: aload_0
        11: invokevirtual #24                 // Method a:()I
        14: putfield      #18                 // Field b:I
        17: return

  // Bar class constructor
  public Bar();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #19                 // Method Foo."<init>":()V
         4: aload_0
         5: iconst_2
         6: putfield      #13                 // Field a:I
         9: return
```
Two things are noticeable. First is that ```Foo```'s constructor initializes ```a``` as it would be expected - inserting 1 to it (```iconst_1``` and ```putfield #14```) - *reminder!* JVM uses stack to manipulate data, so it does not have registers or any form of that. Similar to it looks ```a``` initialization in ```Bar``` class. But ```b``` initialization looks odd. When ```b``` is beeing initialized it is done by accesing method ```public int a()``` which is done by ```invokevirtual #24```([ivokevirtual](http://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.invokevirtual)). If there was no inheritance or overriding in this example that wouldn't look so suspicious. But there is one. Using polimorphic methods in constructors is not a good idea. It can lead to wrong initialization like in the example above. Now ```public int a()``` is defined in ```Bar``` class. It means that this one form ```Bar``` will be taken insead of ```Foo```'s in ```Foo``` initialization. Because there is no such thing like unspecified vlaue in JVM ```Bar.a()``` returns 0. So to answer question what is actual value - 0.

## Summary
Could it be done better than that? Java does not provide polimorphic fields. Such feature should have been done only by workaround. Primitives are good example here because this wrong initialization leads to non obvious program behaviour at runtime. Instead of ```null``` [default primitves values](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.3) are inserted. If there were reference type there would be immediate NullPointerException and attention in this piece of code.