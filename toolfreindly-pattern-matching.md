People complain a lot about Scala tooling. People complain a lot about weather. Is Scala tooling like weather then? Is it something you cannot do anything about but to accept it or move out to a place with different climate (language)? Luckily, no.

Scala tooling usually has only JVM stack to work with and it was created for Java (especially metadata handling that is vital for tools).
Tool-friendly code for me is a Scala code that make tools live easier.

I do not want to convince anyone to write Java-like code. I just would like to point out some small details that can make your code much more tool-friendly.
In the next blog posts I will show a few codestyle ideas that make your code easier to debug, profile and, surprisingly, understand. Let's start with pattern matching.

## Tool-friendly pattern matching
Pattern matching is one of the features we love in Scala. Is it tool-friendly?

Let's take a look at standard pattern match snippet:

```scala
foo match {
  case bar @ Bar if bar.isOk() => bar.doIt()
  case baz @ Bar if baz.isOk() => baz.doIt()
  case _ => ???
} 
```

Let's see how the first case statement is compiled into byte code:

```asm
      18: instanceof    #15                 // class test/Bar
      21: ifeq          51
      24: iconst_1      
      25: istore_3      
      26: aload         5
      28: checkcast     #15                 // class test/Bar
      31: astore        4
      33: aload         4
      35: invokevirtual #20                 // Method test/Bar.isOk:()Z
      38: ifeq          51
      41: aload         4
      43: invokevirtual #23                 // Method test/Bar.doIt:()Ljava/lang/String;
      46: astore        6
      48: goto          70
      51: iload_3       
      52: ifeq          176
      55: aload         4
      57: invokevirtual #20                 // Method test/Bar.isOk:()Z
      60: ifeq          176
      63: aload         4
      65: invokevirtual #23                 // Method test/Bar.doIt:()Ljava/lang/String;
      68: astore        6
      70: aload         6
      72: astore_2      
      73: aload_1       
      74: astore        8
      76: aload         8
      78: instanceof    #15                 // class test/Bar
      81: ifeq          109
      84: aload         8
      86: checkcast     #15                 // class test/Bar
      89: astore        9
      91: aload         9
      93: invokevirtual #20                 // Method test/Bar.isOk:()Z
      96: ifeq          109
      99: aload         9
     101: invokevirtual #23                 // Method test/Bar.doIt:()Ljava/lang/String;
     104: astore        10
     106: goto          139
     109: aload         8
```


There is a lot going on in the single line. Let's assume that we want to check if our case is hit and we put a breakpoint on the case line. What happens? It is hit every time the condition is being checked!
Why? Because the condition checking is also on this line - so tooling gets information "I was there".

How would tools like to see the code?

```scala
foo match {
  case bar@Bar 
    if bar.isOk() => 
      bar.doIt()
 
  case baz@Bar 
    if baz.isOk() => 
      baz.doIt()
 
  case _ => ???
} 
```

The only difference is line breaking. It is not Python so why does it matter?
Tools for code coverage or debuggers usually use JDI. JDI is line-based - every line of bytecode has its mapping to a line in the source code.

## Conclusion

Let's write cases this way:

```scala
foo match {
  case bar: Bar if bar.isOk() =>
    bar.doIt()
  case baz: Baz if baz.isOk() =>
    baz.doIt()
  case _ => ???
}  
```

Then, when we place breakpoint inside a case body, it will be hit only when the body is about to be invoked.
