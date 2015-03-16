# Lambdas

Lambdas are one of the most loved Scala feature. Until someone have to debug them. Can we blame tooling for this completely?


Standard lambda, I've seen hundres of them:
```scala
list.map(x => s"Element[%x]"))
```

So let's put a breakpoint in our lambda and debug it.
Breakpoint hit. Great. But wait we are outside of our lambda. Ok, let's start again.
And again. And again.
And yeah, we are finally inside our lambda. It is happy case. In some situations our program just end.

Why?  JDI is line based. And our line holds code for:
-create new instance of our lambda
-create CanBuildFrom for our return type and collection type
-call of map function
-invoking our lambda for each element

Pretty much in one line.

## Solution

So for sake of future debugging rewrite this code to:

```scala
list.map(x => 
    println(x + "_ala"))
```

Now breakpoint is hit only once per element.

Lambdas are not always good pratice. For more:
https://www.youtube.com/watch?v=-7iquITRJVo
