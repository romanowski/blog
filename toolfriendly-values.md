# Values, values everywhere

We have to format and pretty print polish mobile numbers. Like this:
<script src="https://gist.github.com/romanowski/a4aa16ccaa5b6b05ca26.js"></script>
Ok, it is trivial so 2 mins and we end up with:
```scala
//takes number like 123456789 and format it as +48 123 456 789 
def format(nr: String) = nr.sliding(3).mkString("+48", " ", "")  
 
val numbers = Seq("+48730409203", "890890886", "700 440 675", "700 45 34 34")
```

And lets assume that we have some problems with one of replace. So lets see how our numbers looks after first map. And we can't do it. JVM store return values direcly on stack and JDI has no direct access to stack.

Solution? lets name what we get after each transformation. Then debugging or even reading is plasure :)

```scala
val withoutSpaces = numbers.map(_.replaceAll(" ", ""))
val withoutLocal = withoutSpaces.map(_.replace("+48", ""))
withoutLocal.map(format)
```
