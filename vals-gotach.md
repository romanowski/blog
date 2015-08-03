# Vals gotcha!
Immutability in Scala is great. A lot of concerns just go away when you use it, but those that remain are not so clear to reason about. Look at the code below:
```scala
class Foo {
  val a = 1
  val b = a
}

class Bar extends Foo {
  override val a = 2
}
```
My question is: what is the value of expression ```(new Bar).b```? In this post I will explain why it is neither *1* nor *2*.
There is a [general explanation](http://docs.scala-lang.org/tutorials/FAQ/initialization-order.html) of why similar problems spring up in Scala but this post contains more in-depth one.

## Actual problem
In every book about Scala you will read something more or less like that: vals are implemented as private fields with a public getter. But this is not the full truth about vals. Unfortunately, this public getter is the source of all evil in this example. Let's look again at the previous code snippet:
```scala
class Foo {
  val a = 1
  val b = a
}

class Bar extends Foo {
  override val a = 2
}
```

Consider how it is that field is marked private and final and you can still inherit it from superclass? How is it made to be polimorphic? The thing that makes it possible is an implicit polimorphic getter in class body for each declared val. What is more, fields and methods share a single namespace in Scala. This implies that when you call ```fooInstance.a```, what you are actually calling in bytecode is ```Foo::a()```. When you override ```vals``` inherited from parent class, compiler actually overrides these public getters.

## What is the actual value and why?
Look at another code snippet showing constructors translated to bytecode
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
Two things are noticeable. The first is that ```Foo```'s constructor initializes ```a``` as expected - by assigning 1 to it (```iconst_1``` and ```putfield #14```) ( *reminder!* JVM uses stack to manipulate data - it does not have any form of registers). ```a``` initialization in ```Bar``` class looks very similar. But ```b``` initialization looks odd. ```b``` is initialized by taking result of ```public int a()``` called by ```invokevirtual #24```([invokevirtual](http://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.invokevirtual)). If there was no inheritance or overriding in this example this wouldn't look suspicious at all. But there is one. Using polimorphic methods in constructors is not a good idea. It can lead to wrong initialization like in the example above. ```Foo::a()``` is redefined in ```Bar``` class. It means that the one from ```Bar``` will be used instead of ```Foo```'s during ```Foo``` initialization. Because there is no such thing as unspecified value in JVM, ```Bar::a()``` returns 0. And that answers the question what is the actual value. It is 0.

## Summary
Could it be done better than that? Java does not provide polimorphic fields at all. Such feature should have been done only by workaround. Primitives are particularly good example here because [default primitves values](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.3) are inserted instead of ```null```.  This leads to non obvious program behaviour at runtime.  If it had been a reference type, there would have been an immediate NullPointerException drawing attention to this piece of code.