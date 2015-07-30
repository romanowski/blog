
#Interview with Rex Kerr. The first appendix to Dick Wall's 'Scala needs you!'.

##Preface
During Scala Days in SF, Dick Wall was [trying to convince people](https://www.parleys.com/tutorial/scala-needs-you) to spend their spare time helping to make Scala language, compiler and documentation even better. He focused mostly on showing the Scala's web page and available resources which can help newcomers. Also some time ago [Adriaan Moors](https://github.com/adriaanm) completely redesigned readme.md on github having beginners in mind. I wish these resources had been so informative a year ago.

Having been inspired by Dick's talk I decided to contact a few currently active collaborators and open source contributors from outside EPFL and Typesafe's Scala compiler team. As a result I will publish a short series of interviews focused primarily on Scala language, contributing to Scala ecosystem etc. Not to mention that I believe it's worthwhile to get to know these extraordinarily interesting people.

My first interlocutor was Rex Kerr.

##Interview

>**Rex Kerr** ([Ichoran](https://github.com/Ichoran)) is a research scientist at the University of California, San Francisco where he studies the neural basis of behavior and the aging of the nervous system in simple model organisms.  He is also a consultant with Typesafe and responsible for maintaining and optimizing the Scala library.

**Michał Pociecha: Do you remember the first time when you heard about Scala? What made you become interested in this language?**

**Rex Kerr**: Not precisely—I picked up a few references to it reading various sites on the internet, and eventually was intrigued enough to try it out.  Initially, I was converting unacceptably slow math-heavy Python code to Java, and finding that it was really hard to see the math underneath all the Java boilerplate.  Scala had operator overloading, and was supposedly as fast as Java, so I decided to give it a try!

**MP: You're a neurobiologist. Are you able to use Scala at work?**

**RK**: Absolutely.  Almost all data analysis I do is in Scala.  I have a lot of medium-scale data (single digit to hundreds of gigabytes), and do a lot of data exploration where fast feedback is essential.  Scala lets me process sample data sets quickly enough so that I usually don't have time to get a cup of tea, and full data sets in less time than it takes to eat lunch, while still allowing me to express what I want in a compact format.  It's made a huge difference in being able to really play with data.  (Incidentally, I think the term "big data" is massively overused.  A lot of "big data" is really "medium data but my algorithms are so bad and my code so unoptimized, I can't handle it".)

**MP: Which other programming languages do you use/like?**

**RK**: I try to use whatever is best for the job, which means C++ (real-time machine vision), LabView (glue to hold machine vision hardware together), Matlab (quick yet sophisticated analysis of tables of numbers), R (for wonky but neccessary statistics), Java (libraries, legacy code), Mathematica (symbolic math and numeric approximations).  I'm pretty fussy about languages.  I like that each of those languages work really well on *some* job, but I really don't like how badly they handle other jobs.  Pointers leaking all over the place, scrambled wires all over the screen, barely usable strings and exception handling, weird syntax and slow on anything custom, and Where's-Waldo with the important part of the code in all the boilerplate, every command using conventions from another universe.  If I needed to write a lot of red-black tree implementations, I'd use Haskell—for structure-manipulation algorithms the code is clear and lovely.  But I haven't had problems like that yet.  I'm really intrigued by Rust.  I'm hoping I can drop C++ from some of my "best at" tasks.  I haven't tried Rust out enough to be sure, but I think ownership is potentially a very powerful concept, and zero-cost abstractions are something I really miss in Scala.  (You have a pretty high cognitive load to make absolutely sure that value classes end up being zero-cost in any particular case.  So they are zero-cost at runtime, but really big cost at coding time.)  In general, I'm a huge fan of the compiler doing lots of work for me; I want computers to do the things which computers are supposed to be good at, like figuring out when two things are logically equivalent and picking the better way, or keeping track of details that have to be gotten right, but which should "just work".  Most languages make it very difficult for me to tell the compiler about what I need help with.  The type system of Scala or Haskell is a great way to encode *some* things that I need help with, but it's *awful* for other things like making sure I remember to increment at least one of three counters when recursing or looping.  I'm really excited to see if the next decade or so brings us some big improvements in that regard.

**MP: Let me make a little digression: could you tell us something about your day-to-day work? Maybe could you reveal some interesting results of a research which you got? Personally I'm quite curious about.**

**RK**: Sure!  One thing I'm interested in is decision making.  Computers are usually pretty dumb about decisions: if anything unexpected varies, they tend to behave blindly or erratically or even crash and become nonfunctional.  Living creatures, in contrast, tend to come up with sensible-seeming compromises.  How do they do that?  To approach this question I study nematode worms, Caenorhabditis elegans specifically.  They're about a millimeter long and eat bacteria, and, naturally, one of their main behaviors is to find food.  To do it, they just crawl around, and if the smell of food gets stronger, they keep going; if it gets weaker, they back up and turn around—kind of like they're playing a hotter/colder game.  Now, it turns out they also get eaten by fungi which trap them with tiny loops and then digest them.  No worries, though: they can feel the contraction and get out of the way by backing up and turning around.  So, what happens if they're after food, going the right way, and they feel something?  Now you've got just the kind of decision that computers do badly: "just go find food" is pretty dumb, because you might get eaten by a fungus, and "get away, then worry about food!" is also pretty dumb because most of the time it's a false alarm (not really something dangerous).  I built a machine vision system to watch a huge number of these worms going after food while feeling something (I vibrated the dish they were on), and found that they're actually quite clever: they respond to the vibration as often as they ever would (food or no), but if they're going towards food when they react, they don't back up as far or turn away as much.  So they're kind of hedging their bets in a way that will probably keep them alive and probably not mess up their quest for food too much.  Now the question is: how do they *do* that?  We know how all the neurons in the animal are connected, but staring at the wiring diagram, as it's called, doesn't provide much insight into precisely what and how they are computing to make that decision.  So more experiments will be needed.  But we know that they're doing this surprisingly canny decision-making, and with very, very few computational elements (in a network), and that means we can figure out how they're doing it—hopefully there will be some general principle to learn.

**MP: When did you start contributing to Scala? What was your motivation?**

**RK**: I don't actually remember; sometime in 2013 maybe?  I kind of drifted from making fixes in my personal library to get around Scala library bugs to fixing the bugs in the Scala library without quite noticing it.  (I'm sure GitHub could tell us when my first commit was.)  My motivation was to help improve a tool that's really useful—why fix it once for myself when, for a little more effort, I can fix it for everyone?  You get much more bang for your buck that way, even if the bang isn't always where I can directly benefit.

**MP: As it's described in readme.md, you're involved in improving collections library and performance. Hence, you're also the recommended person to review changes. Are such things (I mean optimisations—not the code review) something you particularly like doing?**

**RK**: I think so.  I've been doing it for a very long time.  For example, I had the fastest text-mode windowing library for Turbo Pascal (written in 8086 assembly) for a while back in 1988 or so.  I wrote an implementation of a machine learning algorithm that sped up one by a pretty competent CS grad student, who was writing in C++, by a factor of about 100.  That was fun (and allowed a lot more exploration of the uses of that ML algorithm).  But also, I guess I fundamentally get bored easily and dislike waiting, so I'd rather spend ten times longer optimizing than I'd wait just to keep myself from having to wait.  If I really hated optimizing things, I would find some other way to avoid waiting for stuff to run.

**MP: Do you improve performance of the parts of code with already known issues (maybe even with some existing tickets created) or you just rather look through the code, profile it, run benchmarks and try to find problematic parts?**

**RK**: I've done all of the above, and I also often benchmark a change or workaround for some other issue to make sure that it isn't going to kill performance.  For example, there's an issue where Stream filterNot didn't have the correct laziness properties that filter did.  You can fix the issue by altering filtering code in the appropriate superclass to special-case Stream, but you don't want to because then you have a performance hit on *every* filterNot.  Better just to write filter(!p) in your Stream code.  Incidentally, I spend a lot more time benchmarking than profiling.  The problem with profiling is that the JVM architecture is not very friendly to profilers that want to perform appropriate credit assignment on a small scale(for all sorts of reasons).  In my experience, profilers are good at picking up whopping performance problems and getting the big picture right ("we spend 20% of our time in parsing and 80% of our time in computation"), but benchmarking is more reliable (which is to say: still only kind of reliable!) for detailed optimizations.

**MP: Some time ago also e.g. [Denton Cockburn](https://github.com/kanielc) started making optimisations (he's really good, I think) so, as one can see, there are people that would like to help. Do you have some advice for such people?  Especially useful micro-benchmarks, typical mistakes to avoid, etc.**

**RK**: I've also been pleased with the quality of Denton's work.  Optimization is tricky to do well.  The meme of premature optimization being a bad thing tends to make people avoid thinking deeply about optimization, so that once they really need to optimize they don't have the skill set needed to actually do it.  But premature optimization *is* a bad thing, so it's a fine line to walk.  Anyway, optimization on the JVM is especially hard because the JIT compiler plays all sorts of tricks and will not necessarily tell you what it's doing and almost never will tell you why.  And then you still have to keep in mind all the tricks your CPU will play like branch prediction and prefetching and whatnot in order to have a good shot at predicting what code will perform well.  But there are still plenty of cases where it's just the wrong algorithm, or something really obviously unhelpful like creating a bunch of extra objects that could just be primitives.  Anyway, the best micro-benchmark is the one that is closest to a demanding real-world test of whatever functionality you're trying to speed up.  In particular, it's very important to make the output of the benchmark depend on most everything that happens because often the JVM is clever enough to skip work that it knows will just heat up your CPU but otherwise not do anything.  Use a good benchmarking framework like JMH (or try out my quicker-and-dirtier-but-wow-is-it-quicker Thyme benchmarking utility—I guess it says something about how much I like optimizing that I even optimized my benchmarking tools!).  And be sure to know https://wikis.oracle.com/display/HotSpotInternals/MicroBenchmarks backwards and forwards.

**MP: [VirtusLab](https://www.parleys.com/tutorial/scala-ide-from-0-4-0) works with Typesafe to improve ScalaIDE for Eclipse</a>. What is your Scala editor of choice? What do you think about the current state of Scala tooling in general?**

**RK**: I don't like tooling much.  I'm quite difficult to please with tooling, perhaps because I spend the majority of my time both scientifically and not making tools of various sorts, and I spend a non-negligible fraction of that time thinking about what makes a tool a good tool.  A good tool should either be as close to invisible as possible, as close to completely intuitive as possible, or provide something so awesome that it's worth an investment of time to learn it and *still* take as little time as possible.  You're not programming to enjoy the tool/tooling: you want to get stuff done, and the tool is a necessary step.  So, that said, I rarely use ScalaIDE or any IDE, mostly because they get in the way of me writing code.  Delays and glitches while typing are really distracting.  Crashes are abhorrent.  Syntax error highlighting when I know I'm not done writing the code is also really distracting, but if I have to ask for it every time it's also distracting.  Panels of buttons and text output boxes and so on waste screen space.  Step-through debugging isn't much use when you're running with multiple threads and timing is important for semi-reproducible behavior.  So I end up using Kate (debugging with println) more than anything else—it lets me write code and doesn't interfere (at least if I turn autocomplete off).  (Kate is the heavyweight text editor / lightweight code editor that's part of KDE.)  I've tried Sublime Text enough to know that I would like it if I already knew it, but I haven't quite reached the threshold where the overhead to learn it is worth it.  I use SBT, and I do like how easy it is to just get started, but I'm kind of annoyed that the state of Java tooling and Maven and whatnot is such that I have to use SBT to manage it all.  I do a lot of code generation, which recent versions of SBT have made not toooooo painful, but it's still rather arcane.  I'm kind of jealous of Rust's cargo system; from afar, it seems to have been able to jettison most of the non-essential cognitive overhead that SBT has to tackle to work properly with the Java ecosystem.  There is one bit of tooling, not Scala specific, that I think is awesome: git.  I'm old enough to remember "mkdir old2; cp *.c *.h old2", and CVS, and SVN.  Now I put practically everything in git: source code, papers, presentations.  It's made a huge difference in being able to move between different versions of things, try things out safely, and so on.  Very, very worth the time to learn.  That's what a tool should be like.

**MP: I know you're very busy person. What do you do when you have a bit spare time?**

**RK**: I have two young children, so most extra time goes to interesting activities with them—we never go to the beach enough, swim enough, hike in forests enough, etc.  If it's not that much spare time, I'll probably just answer a StackOverflow question.

**MP: Thanks for your time.**

**RK**: Thanks for the interest!