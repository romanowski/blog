# ScalaSphere 2016: few impressions from tooling developer

## Scalasphere: conference focused on technical problems and solutions

[ScalaSphere](http://scalasphere.org) was one-of-a-kind conference focused on Scala tooling. Those who expected presentations full of theory or math, might feel disappointed - conference was dedicated purly to scala tooling. "_I've never been on such deeply technical conference_" was often heard and I agree with those comments.


## Quo vadis, Scala tooling?

After Eugene's presentations at ScalaDays in [Berlin](http://scalamacros.org/paperstalks/2014-06-17-EasyMetaprogrammingForEveryone.pdf) and [Amsterdam](https://www.parleys.com/tutorial/state-meta-summer-2015) we put high hopes in scala.meta. Scala.Meta was presented as API over scala AST designed to reuse library/plugins/macros between Scala versions or platforms. I know that some tooling developers started making plans like _"if only we had Scala.Meta then we finally can make "this" and "that properly"..."_.

The only problem is that Eugene cannot make it work with current scala AST what makes usages really limited. We can operate on AST but we lack information about types. We can modify AST (and this is relatively simple) but it won't be reflected in generated bytecode.

Eugene asked for help and inspiration, so:

![alt tag](scala-meta-needs-you.jpg)

## SBT, the only choice

Everybody uses SBT, most complains about it but no one really knows it well. This is a sad true about build tools for Scala.

We probably cannot make sbt's internals simpler but we can fix DSL and dependencies resolution (they both generated most of complaints).
Alexander [presented](http://scalasphere.org/speaker/alexandre-archambault/) his [coursier project](https://github.com/alexarchambault/coursier) addressing the later problem but we are still waiting for solution for sbt's DSL...

_..or YET another build tool for scala_

## IDEless or almost IDEless as alternative for "heavy" IDEs?

Sam and Rory gave great [talk](https://www.youtube.com/watch?v=XqzDDULhPWU) about progress done in [ensime](http://ensime.github.io/) - project that transforms your favourite editor (not only Emacs!) into the light IDE. [Dave presented ideas](https://www.youtube.com/watch?v=N_bM0JRiNC0) that were even more radical about 'IDElessness' and he claimed they work.

A year back I couldn't imagine developing Scala code without IDE. Today I am starting to belive that it is possible to effectively work on big Scala projects outside IntelliJ or ScalaIDE.

## ScalaIDE, now we know why...

[Iulian](http://scalasphere.org/speaker/iulian-dragos/) opened conference with his [talk](https://www.youtube.com/watch?v=74OePFNvy0o) about beginnings and history of ScalaIDE. I finally heard explanation for many weird choices back in those days (like weaving inside Java model instead of creating new Scala one). This talk was a big warning to all tooling creators about dangers behind 'let's hack this and we'll got that for free' approach. There ain't no such thing as free lunch so you will pay a price for all _hacky_ designs.

## Licensed under...

If you ever wonder what if you are open source or which copyright should be used, Sam's [the lightning talk](https://www.youtube.com/watch?v=6pEQ4xT1LMQ) is for you. I've never expected to get almost all required knowledge about licences and all law-related stuff from lightning talk!