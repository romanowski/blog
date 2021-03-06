People complain a lot about Scala tooling. People complain a lot about weather. But is Scala tooling like weather? Something you can’t do anything about except from moving to different climate (language)? Luckily no.

Scala tooling was created for Java (especially metadata handling that is vital for tools) and usually has only the JVM stack to work with. Tool-friendly code for me is Scala code that make tools’ lives easier.

I’m not trying to convince anyone to write java-like code. I just want to to point out some small details that can make your code much more tool-friendly. In forthcoming blog posts I will show a few codestyle ideas that will make your code easier to debug, profile and - surprisingly - understand. Let's start with pattern matching.

## Tool-friendly pattern matching
Pattern matching is one of the features we love in Scala. Is it tool-friendly?

Let's look at a standard pattern match snippet:

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

Why? Checking a condition is also in our line, so tooling gets the information: "I was there".

How would tools like to see the code?

```scala
foo match {
  case bar @ Bar 
    if bar.isOk() => 
      bar.doIt()
 
  case baz @ Bar 
    if baz.isOk() => 
      baz.doIt()
 
  case _ => ???
} 
```

The only difference is line breaking, but this isn’t python so why does it matter? Tools like code coverage or debuggers usually use JDI, which is line-based: every line of the bytecode has its mapping to a line in the source code.

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
