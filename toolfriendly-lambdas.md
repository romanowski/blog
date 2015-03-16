# Lambdas

Standard lamda usage:
```scala
list.map(x => println(x + "_ala"))
```

So again lets put a breakpoint in our lambda. Run and before we are inside actual lambda call jvm hit breakpoint 4 times. Why? Same as before – JDI is line based. And in this single line happens things like:
-create new instance of our lambda
-call of map function
-calling apply for each argument
-invoking body of lambda

Pretty much as for one line.

So for sake of future debugging rewrite this code to:

```scala
list.map(x => 
    println(x + "_ala"))
```

Now breakpoint is hit only once.

Lambdas are not always good pratice. For more:
https://www.youtube.com/watch?v=-7iquITRJVo
