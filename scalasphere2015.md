## ScalaSphere 2016 - impression from tooling developer

Scalasphere was one-of-the-kind conference focused completly on Scala tooling. Who expect presentations full of theoretical problems can be disappinted - conference was deeply technical. I've heard mutiple times comments like "I've never been on such deeply technical conferences" during chat with atendees. I totally agree and I thnik this formula is really good.

_Scalasphere if confere about technical problems and solutions_

## Quo vadis, scala tooling?

After Egugene's presentations on ScalaDays (link) and ScalaExchange (link, sprawdz) all put high hopes in scala.meta. Someone can even say that most tooling developers start making plans like "when we will have scala.meta we can then ...".

The only problem is that Eugene cannot make it work with current scala AST what makes it useless. We can operate on AST but we lack information about types. We can modify AST (and this is relativly simple) but it won't be reflected in generate bytecode.

Eugene asked for help and inspiration, so:

![alt tag](scala-meta-needs-you.jpg)

## SBT, the only choice

Everybody using SBT, most compilains and no one knows it. This is a sad true about build tool for scala.

We probably can't make sbt's internal simpler but we can fix DSL and dependecy resolution (that generated most complains).
Alexander presnted his coursair project (link) addressing 2nd problem so we are still waitnig for solution for sbt's DSL

## IDEless or almost IDEless as alternative for "heavy" IDEs?

A year back I can't imagine developing Scala code without IDE. However impressed by progress done in ensime (mostly open to editors other then emacs) or David's ideas about IDEless work enviroment I think we are possible to effectively work on scala outside Intellij or ScalaIDE.
