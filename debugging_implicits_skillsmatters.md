Debugging implicits
======

I was recently asked what are my favorite and least favorite Scala features. Probably few options would be matching here, but I decided to give one answer for both questions: implicits. Why? Implicits are very simple and yet very powerful concept but with great power comes great responsibility. They can be used for implicit parameters, implicit conversions, view bonds and context bounds (type classes) letting us solve couple of different problems with one language feature. But at the same time we must be aware in what context we are using them and - which is much harder - which implicits are available in the current context.

As long as you are application programmer (see: [Scala levels: beginner to expert, application programmer to library designer](http://www.scala-lang.org/old/node/8610)) this is not a big concern. Most of the Scala libraries are designed cleverly enough to hide complexity from the user.
At library designer level it is a completely different story though.

At VirtusLab we constantly have to debug internals of third party libraries and the Scala itself. We often find ourselves profiling scalac, debugging Sbt, or looking into bizarre Slick issues. If you ever had to do that you know that it can be painful experience. Huge amount of esoteric language features used together is not making it any easier. And heavy usage of implicits is almost constant pattern.
If you are not really experienced Scala programmer you can be lost.


Luckily we are not left alone with that problem. Main Scala IDEs like [ScalaIDE](http://scala-ide.org/docs/current-user-doc/features/typingviewing/implicit-highlighting/index.html) or [IntelliJ](http://confluence.jetbrains.com/display/IntelliJIDEA/Working+with+Scala+Implicit+Conversions) are able to list currently used implicits and we can do the same in REPL as well.

Let's go through a basic example with Ordering type class to check how things work together and what to do in case you are stuck. First, let's create simple class wrapper around Int value and make array of its elements:

```scala
case class IntWrapper(n: Int)

val list = Array(IntWrapper(2), IntWrapper(7), IntWrapper(3))
```

Now we want to sort our list using one of the methods from the standard library. We can do this using _sortBy_, _sortWith_, or _sorted_. We will use _sorted_:

```scala
scala> list.sorted
<console>:11: error: No implicit Ordering defined for IntWrapper.
              list.sorted
                   ^
```

Ouh, looks like we forgot about implicit value of type _Ordering[IntWrapper]_. Checking method signature would tell us the same thing:

```scala
def sorted[B >: A](implicit ord: math.Ordering[B]): List[A]
```

We can fix that by bringing implicit value of type _Ordering[IntWrapper]_ in scope, or by extending _IntWrapper_ with class _Ordered[IntWrapper]_ or _Ordering[IntWrapper]_:

```scala
object IntWrapper {
  implicit val orderingByN: Ordering[IntWrapper] = Ordering.by(_.n)
}
```

That fixed our problem:

```scala
scala> list.sorted
res0: Array[IntWrapper] = Array(IntWrapper(2), IntWrapper(3), IntWrapper(7))
```

There is nothing difficult here. Required and provided implicit values have exactly the same type. Now let's try how it works with inheritance version. To make things a bit more difficult we choose to extend our class with _Ordered[IntWrapper]_ rather than _Ordering[IntWrapper]_:

```scala
case class IntWrapper(n:Int) extends Ordered[IntWrapper] {
  def compare(that: IntWrapper) =  this.n - that.n
}
```

And everything works again. But it raises one interesting question: how? Our method _sorted_ requires one implicit parameter of type _Ordering[IntWrapper]_. We provided none, and we didn't even mention class _Ordering_ anywhere. It would be reasonable to guess that some implicit conversions or implicits chaining happens behind the scene. Possibly one word of explanation what I understand by the term 'implicits chaining' is required. Martin Odersky in his book "Programming in Scala" says:

_"One-at-a-time Rule: Only one implicit is tried. The compiler will never rewrite x + y to convert1(convert2(x)) + y. Doing so would cause compile times to increase dramatically on erroneous code, and it would increase the difference between what the programmer writes and what the program actually does. For sanity’s sake, the compiler does not insert further implicit conversions when it is already in the middle of trying another implicit. **However, it’s possible to circumvent this restriction by having implicits take implicit parameters, which will be described later in this chapter.**"_

Bold emphasis is mine. We will see example of behavior described by Odersky in a moment but first we need to introduce some methods to check what really happens under the hood. Except IDEs support mentioned at the beggining of this article we have at least a few other options.

Compiler flag _-Xprint:typer_
------
To use this method you need to start Scala REPL with -Xprint:typer parametr. It will produce a lot of bloat which you can ignore. If we use _sorted_ method on the list we will see:

```scala
scala> list.sorted
[[syntax trees at end of                     typer]] // <console>
package $line5 {
  object $read extends scala.AnyRef {
    def <init>(): $line5.$read.type = {
      $read.super.<init>();
      ()
    };
    object $iw extends scala.AnyRef {
      def <init>(): type = {
        $iw.super.<init>();
        ()
      };
      import $line3.$read.$iw.$iw.IntWrapper;
      import $line3.$read.$iw.$iw.IntWrapper;
      import $line4.$read.$iw.$iw.list;
      object $iw extends scala.AnyRef {
        def <init>(): type = {
          $iw.super.<init>();
          ()
        };
        private[this] val res0: Array[IntWrapper] = scala.this.Predef.refArrayOps[IntWrapper]($line4.$read.$iw.$iw.list).sorted[IntWrapper](math.this.Ordering.ordered[IntWrapper](scala.this.Predef.$conforms[IntWrapper]));
        <stable> <accessor> def res0: Array[IntWrapper] = $iw.this.res0
      }
    }
  }
}

[[syntax trees at end of                     typer]] // <console>
package $line5 {
  object $eval extends scala.AnyRef {
    def <init>(): $line5.$eval.type = {
      $eval.super.<init>();
      ()
    };
    lazy private[this] var $result: Array[IntWrapper] = _;
    <stable> <accessor> lazy def $result: Array[IntWrapper] = {
      $eval.this.$result = $line5.$read.$iw.$iw.res0;
      $eval.this.$result
    };
    lazy private[this] var $print: String = _;
    <stable> <accessor> lazy def $print: String = {
      $eval.this.$print = {
        $line5.$read.$iw.$iw;
        "res0: Array[IntWrapper] = ".+(scala.runtime.ScalaRunTime.replStringOf($line5.$read.$iw.$iw.res0, 1000))
      };
      $eval.this.$print
    }
  }
}

res0: Array[IntWrapper] = Array(IntWrapper(2), IntWrapper(3), IntWrapper(7))
```
It actually contains information about implicits, but it's hidden in the background noise. Can you find it?

Additional downside of this method is that it will produce that kind of noisy output for every expression you enter.

REPL command _:javap_
------
_:javap_ command gives us possibility to print various details of the bytecode generated by Scala compiler. Let's try to use it to check what happens inside _sorted_:

```scala
scala> def sortedList = list.sorted
sortedList: Array[IntWrapper]

scala> :javap sortedList
  Size 1435 bytes
  MD5 checksum f966840f5603604143f443eb193f93cc
  Compiled from "<console>"
public class
  SourceFile: "<console>"
  InnerClasses:
       public static #63= #60 of #62; //=class  of class $line5/$read
       public static #63= #65 of #67; //=class  of class $line3/$read
       public static #63= #69 of #71; //=class  of class $line4/$read
       public static #63= #2 of #60; //=class  of class
       public static #63= #73 of #65; //=class  of class
       public static #63= #21 of #69; //=class  of class
       public static abstract #78= #75 of #77; //$less$colon$less=class scala/Predef$$less$colon$less of class scala/Predef
       public static #81= #80 of #73; //IntWrapper=class IntWrapper of class
    Scala: length = 0x0

  minor version: 0
  major version: 50
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Utf8
   #2 = Class              #1             //
   #3 = Utf8               java/lang/Object
   #4 = Class              #3             //  java/lang/Object
   #5 = Utf8               <console>
   #6 = Utf8               MODULE$
   #7 = Utf8               L;
   #8 = Utf8               <clinit>
   #9 = Utf8               ()V
  #10 = Utf8               <init>
  #11 = NameAndType        #10:#9         //  "<init>":()V
  #12 = Methodref          #2.#11         //  ."<init>":()V
  #13 = Utf8               sortedList
  #14 = Utf8               ()[LIntWrapper;
  #15 = Utf8               scala/Predef$
  #16 = Class              #15            //  scala/Predef$
  #17 = Utf8               Lscala/Predef$;
  #18 = NameAndType        #6:#17         //  MODULE$:Lscala/Predef$;
  #19 = Fieldref           #16.#18        //  scala/Predef$.MODULE$:Lscala/Predef$;
  #20 = Utf8
  #21 = Class              #20            //
  #22 = Utf8               L;
  #23 = NameAndType        #6:#22         //  MODULE$:L;
  #24 = Fieldref           #21.#23        //  .MODULE$:L;
  #25 = Utf8               list
  #26 = NameAndType        #25:#14        //  list:()[LIntWrapper;
  #27 = Methodref          #21.#26        //  .list:()[LIntWrapper;
  #28 = Utf8               [Ljava/lang/Object;
  #29 = Class              #28            //  "[Ljava/lang/Object;"
  #30 = Utf8               refArrayOps
  #31 = Utf8               ([Ljava/lang/Object;)Lscala/collection/mutable/ArrayOps;
  #32 = NameAndType        #30:#31        //  refArrayOps:([Ljava/lang/Object;)Lscala/collection/mutable/ArrayOps;
  #33 = Methodref          #16.#32        //  scala/Predef$.refArrayOps:([Ljava/lang/Object;)Lscala/collection/mutable/ArrayOps;
  #34 = Utf8               scala/math/Ordering$
  #35 = Class              #34            //  scala/math/Ordering$
  #36 = Utf8               Lscala/math/Ordering$;
  #37 = NameAndType        #6:#36         //  MODULE$:Lscala/math/Ordering$;
  #38 = Fieldref           #35.#37        //  scala/math/Ordering$.MODULE$:Lscala/math/Ordering$;
  #39 = Utf8               $conforms
  #40 = Utf8               ()Lscala/Predef$$less$colon$less;
  #41 = NameAndType        #39:#40        //  $conforms:()Lscala/Predef$$less$colon$less;
  #42 = Methodref          #16.#41        //  scala/Predef$.$conforms:()Lscala/Predef$$less$colon$less;
  #43 = Utf8               ordered
  #44 = Utf8               (Lscala/Function1;)Lscala/math/Ordering;
  #45 = NameAndType        #43:#44        //  ordered:(Lscala/Function1;)Lscala/math/Ordering;
  #46 = Methodref          #35.#45        //  scala/math/Ordering$.ordered:(Lscala/Function1;)Lscala/math/Ordering;
  #47 = Utf8               scala/collection/mutable/ArrayOps
  #48 = Class              #47            //  scala/collection/mutable/ArrayOps
  #49 = Utf8               sorted
  #50 = Utf8               (Lscala/math/Ordering;)Ljava/lang/Object;
  #51 = NameAndType        #49:#50        //  sorted:(Lscala/math/Ordering;)Ljava/lang/Object;
  #52 = InterfaceMethodref #48.#51        //  scala/collection/mutable/ArrayOps.sorted:(Lscala/math/Ordering;)Ljava/lang/Object;
  #53 = Utf8               [LIntWrapper;
  #54 = Class              #53            //  "[LIntWrapper;"
  #55 = Utf8               this
  #56 = Methodref          #4.#11         //  java/lang/Object."<init>":()V
  #57 = NameAndType        #6:#7          //  MODULE$:L;
  #58 = Fieldref           #2.#57         //  .MODULE$:L;
  #59 = Utf8
  #60 = Class              #59            //
  #61 = Utf8               $line5/$read
  #62 = Class              #61            //  $line5/$read
  #63 = Utf8
  #64 = Utf8
  #65 = Class              #64            //
  #66 = Utf8               $line3/$read
  #67 = Class              #66            //  $line3/$read
  #68 = Utf8
  #69 = Class              #68            //
  #70 = Utf8               $line4/$read
  #71 = Class              #70            //  $line4/$read
  #72 = Utf8
  #73 = Class              #72            //
  #74 = Utf8               scala/Predef$$less$colon$less
  #75 = Class              #74            //  scala/Predef$$less$colon$less
  #76 = Utf8               scala/Predef
  #77 = Class              #76            //  scala/Predef
  #78 = Utf8               $less$colon$less
  #79 = Utf8               IntWrapper
  #80 = Class              #79            //  IntWrapper
  #81 = Utf8               IntWrapper
  #82 = Utf8               Code
  #83 = Utf8               LocalVariableTable
  #84 = Utf8               LineNumberTable
  #85 = Utf8               SourceFile
  #86 = Utf8               InnerClasses
  #87 = Utf8               Scala
{
  public static final  MODULE$;
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL

  public static {};
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: new           #2                  // class
         3: invokespecial #12                 // Method "<init>":()V
         6: return

  public IntWrapper[] sortedList();
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: getstatic     #19                 // Field scala/Predef$.MODULE$:Lscala/Predef$;
         3: getstatic     #24                 // Field .MODULE$:L;
         6: invokevirtual #27                 // Method .list:()[LIntWrapper;
         9: checkcast     #29                 // class "[Ljava/lang/Object;"
        12: invokevirtual #33                 // Method scala/Predef$.refArrayOps:([Ljava/lang/Object;)Lscala/collection/mutable/ArrayOps;
        15: getstatic     #38                 // Field scala/math/Ordering$.MODULE$:Lscala/math/Ordering$;
        18: getstatic     #19                 // Field scala/Predef$.MODULE$:Lscala/Predef$;
        21: invokevirtual #42                 // Method scala/Predef$.$conforms:()Lscala/Predef$$less$colon$less;
        24: invokevirtual #46                 // Method scala/math/Ordering$.ordered:(Lscala/Function1;)Lscala/math/Ordering;
        27: invokeinterface #52,  2           // InterfaceMethod scala/collection/mutable/ArrayOps.sorted:(Lscala/math/Ordering;)Ljava/lang/Object;
        32: checkcast     #54                 // class "[LIntWrapper;"
        35: areturn
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
               0      36     0  this   L;
      LineNumberTable:
        line 10: 0

  public ();
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #56                 // Method java/lang/Object."<init>":()V
         4: aload_0
         5: putstatic     #58                 // Field MODULE$:L;
         8: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
               0       9     0  this   L;
      LineNumberTable:
        line 15: 0
}
```

Oh no, it's even bigger than previous output. But I would argue that it's easier to read. Output is a bit more structured and since we know we are looking for _sortedList_ we can just search for it. Section which starts with _public IntWrapper[] sortedList()_ contains what we need and has around 20 lines of code. Not so bad, but still far from perfect.

If you have problem with finding exact information about implicits, here it is:

```scala
12: invokevirtual #33                 // Method scala/Predef$.refArrayOps:([Ljava/lang/Object;)Lscala/collection/mutable/ArrayOps;
15: getstatic     #38                 // Field scala/math/Ordering$.MODULE$:Lscala/math/Ordering$;
18: getstatic     #19                 // Field scala/Predef$.MODULE$:Lscala/Predef$;
21: invokevirtual #42                 // Method scala/Predef$.$conforms:()Lscala/Predef$$less$colon$less;
24: invokevirtual #46                 // Method scala/math/Ordering$.ordered:(Lscala/Function1;)Lscala/math/Ordering;
```

We will analyze that in a moment, but first let's look at our third - and most effective - debugging method.

Scala reflection API
------
We are almost there. As you can read in the [scala reflection overview](http://docs.scala-lang.org/overviews/reflection/overview.html):
_"In Scala 2.10, a new reflection library was introduced not only to address the shortcomings of Java’s runtime reflection on Scala-specific and generic types, but to also add a more powerful toolkit of general reflective capabilities to Scala. Along with full-featured runtime reflection for Scala types and generics, Scala 2.10 also ships with compile-time reflection capabilities, in the form of macros, as well as the ability to reify Scala expressions into abstract syntax trees."_

Let's try if we can use these new features to help us with debugging implicits:

```scala
scala> import scala.reflect.runtime.universe._
import scala.reflect.runtime.universe._

scala> def tree = reify{ list.sorted }.tree
tree: reflect.runtime.universe.Tree

scala> show(tree)
res0: String = Predef.refArrayOps($read.list).sorted(Ordering.ordered(Predef.$conforms))
```
Woha! That is exactly what we need without any bloat or syntactic noise. We can clearly see what happened and in what order.

First _Predef.refArrayOps_ implicit conversion is applied to convert _Array_ to _ArrayOps_ (which contains method _sorted_). Since _sorted_ requires _Ordering[IntWrapper]_ as implicit parameter _Ordering.ordered_ implicit conversion is applied:

```scala
implicit def ordered[A](implicit arg0: (A) ⇒ Comparable[A]): Ordering[A]
```

As you can see it requires another implicit argument (that's what I called 'implicits chaining' before) which in our case should have type _IntWrapper => Comparable[IntWrapper]_. We haven't defined it ourselves, but it already exists in Predef in form of $conforms[A] function:

```scala
@implicitNotFound(msg = "Cannot prove that ${From} <:< ${To}.")
sealed abstract class <:<[-From, +To] extends (From => To) with Serializable
private[this] final val singleton_<:< = new <:<[Any,Any] { def apply(x: Any): Any = x }
// The dollar prefix is to dodge accidental shadowing of this method
// by a user-defined method of the same name (SI-7788).
// The collections rely on this method.
implicit def $conforms[A]: A <:< A = singleton_<:<.asInstanceOf[A <:< A]
```

_IntWrapper_ inherits from _Ordered[IntWrapper]_ (which in turn extends _Any with java.lang.Comparable[IntWrapper]_) and _Function1_ is covariant in it's return type so _IntWrapper => IntWrapper_ is actually subtype of _IntWrapper => Comparable[IntWrapper]_ and types match. In fact we are passing to _Ordering.ordered_ identity function.

Last thing worth to notice here is implementation of _Ordering.ordered_. It always creates new object which is nice to be aware of:

```scala
implicit def ordered[A <% Comparable[A]]: Ordering[A] = new Ordering[A] {
  def compare(x: A, y: A): Int = x compareTo y
}
```

Conclusion
------
As we saw, mix of implicit conversions and implicits chaining could become hard to figure out without reaching help from IDE or compiler. Does it mean that implicits are inherently bad idea?  
Not necessarily. Do we normally care what happens under the hood when we are using Ordering or other type classes? Or any other implicit conversions or views for that matter? Usually not. Implicits allow us to significantly reduce boilerplate code which would normally only obscure our view.  
I guess it's a matter of taste. Some people prefer to have all that boilerplate generated, and then possibly hidden by IDE. Others prefer to have it generated by compiler which is IDE-agnostic, but that at first glance could be a bit harder to grasp and could look too 'magically'.  
Both approaches guarantee full control over the code, because we can always pass implicits explicitly or apply conversions by hand. We just don't do that, because usually it's so much easier to let the compiler do the dirty work.  
Yes, in cases, you are lost and really need to debug implicits, you need to know your tools.  
But is that really a bad thing?
