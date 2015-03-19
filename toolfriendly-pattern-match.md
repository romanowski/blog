#1. Cases in pattermaching

Standard Scala code:

```scala
foo match {
  case bar@Bar if bar.isOk() => bar.doIt()
  case baz@Bar if baz.isOk() => baz.doIt()
  case _ => ???
} 
```

Nice one - that is why we love Scala.

How tools would like to see the code?

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

Only difference is line breaking. It is not python so why it matters?
Tools like code coverage or debuggers usually use JDI. And JDI is line-based - every line of bytecode has it's mapping to a line in source code.

Let's take a look at case statement in byte code:

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

Pretty much to do in one line assuming that we want to check if our case is hit. So we put a breakpoint in our case line. And what - it is hit everytime condition is being checked!
Why? Checking the condition is also in our line - so tooling get information "I was there".

## Conclusion:

Let's write cases in this way:
```scala
foo match {
  case bar: Bar if bar.isOk() =>
    bar.doIt()
  case baz: Baz if baz.isOk() =>
    baz.doIt()
  case _ => ???
}  
```

Then when we place breakpoint inside case body - it will be hit only when the body is about to be invoked.

