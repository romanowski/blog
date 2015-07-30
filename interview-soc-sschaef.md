﻿﻿
#Interview with Simon Ochsenreither and Simon Schäfer. The second appendix to Dick Wall's 'Scala needs you!'.

In the previous [interview](http://virtuslab.com/blog/interview-with-rex-kerr/) I was talking with Rex Kerr about his work and contribution to Scala. This time I had a pleasure of asking several questions to two well-known (in the community) Scala ecosystem contributors from Karlsruhe.

>**Simon Ochsenreither** ([soc](https://github.com/soc)) studies computer science and works on various Scala-related things in his free time, including the compiler, the standard library, the documentation and the language specification.

>**Simon Schäfer** ([sschaef](https://github.com/sschaef)) is a computer science student and one of the main developers behind Scala IDE. He is enthusiastic about improving tools and optimizing developers productivity – especially, but not exclusively, his own.

**Michał Pociecha: You're organizers of [The Karlsruhe Functional Programmers Meetup Group](http://www.meetup.com/The-Karlsruhe-Functional-Programmers-Meetup-Group/). Do you see a "pattern" – who is mostly interested in functional programming these days in Germany? Do you use mostly Scala during these meetups? For instance in Kraków we have two separate groups [Kraków Scala User Group](http://www.meetup.com/Krakow-Scala-User-Group/) and [Lambda Lounge Kraków](http://www.meetup.com/Lambda-Lounge-Krakow/). Lambda is focused mostly on more theoretical aspects, and Haskell still seems to be the common language for people using various languages on a daily basis.**

**Simon Schäfer**: To me it seems that functional programming is completely underrepresented in Germany. For most Meetups there are only 10-15 people attending. Interests are not limited only to Scala though. People are interested in all functional programming languages. Therefore, we have a talk exclusively about Scala every few months. This is not limited to just our region, I think, as I don't know any Scala conferences in Germany – the choice of the technologies used by professional developers here seems to be quite conservative. Most attendees to our Meetups do functional programming as their hobby or only for academia.

**Simon Ochsenreither**: I absolutely agree with Simon! German companies seem to be extremely conservative in terms of technology used. Many things are done in Java, followed by a lot of tailor-made software using Microsoft technology. Where I live, there is actually a large company using Delphi.

A lot of this is related to the issue that Germany isn't really a country in which software engineering is standing on its own feet. In many cases code plays only a supporting role for a business.

Companies write code to engineer better cars, automate chemical plants or build "machines that build machines" more efficiently.
Using inferior languages doesn't give them a substantial competitive disadvantage, because software isn't what gives them an advantage in the first place.

This sadly affects the quantity of people at our Meetups, but thankfully not the quality. The attendees as well as our speakers come from a wide variety of backgrounds, and they work on things I will never be smart enough to work on. Most of our attendees are either students, or software developers who use imperative languages at work.

I remember one of our first Meetups where our speaker didn't use slides but live-coded for more than an hour – in Agda, an interactive theorem prover!
In another one I heard the best explanation of continuations and related constructs.
Usually we have a lot of talks about languages, like Haskell, Perl 6, Q with kdb+, ClojureScript with Om, F#, ...
Sometimes we even have the honour of language designers/implementers presenting their own language, like Frege or Nim.
Apart from that, we also had talks about "How to Build a Simple Garbage Collector" or "Monads for reverse engineering".
There is also quite a bit of Scala involved (both Simon and me give talks), but we are a pretty diverse group which is interested in a lot of different topics.

**Michał Pociecha: Do you remember the first time when you heard about Scala? What made you become interested in this language?**

**Simon Ochsenreither**: Since I learned programming many years ago, I had this nagging feeling of "I can't believe that the approach required by language X/tool Y is the best way we solve issue Z in the 21st century".

After three months of Java I finally had enough, and started evaluating other options.
I quickly discarded most other JVM languages, but Scala survived because I not only agreed with many of the design decisions made, but also the way the language was developed – "good enough" was finally not "good enough" any longer. If a better approach to some problem was discovered, the old approach would be replaced. The language was really open to improvements!

This really resonated with me back then, and even today we have plenty of "worse is better" languages to choose from, but only few good ones.

**Simon Schäfer**: Of course I do! My first contact was when I read an article about Scala in a German magazine. When I finished reading it I realized that something happened (others would describe this as love on the first sight). I immediately bought "Programming in Scala", a book by Martin Odersky, and read it within a few days. At that time I didn't understand much – lambdas for example were a complete mystery to me. However, that only increased my interest in the language – an interest that has not waned to this day. On the other hand, my interest has moved a little bit from the language itself to its tooling environment. Which, as you know, still requires many improvements. What I like about Scala community is that these people really want to improve their technical abilities and to move forward on the way we develop software. When you think you can't do it better, someone else shows up and blows your mind with something you have never even heard of before.

**Michał Pociecha: soc, you're one of the most active Scala compiler contributors, and sschaef, as Typesafe's [Iulian Dragoș](https://github.com/dragos) emphasizes, is the most prolific and active Scala IDE contributor. He had the opportunity to contribute to scalac as well. Compilers and IDEs are definitely not the simplest pieces of code. You're both students and have contributed to the Scala ecosystem for a few years so it's really impressive. What was your motivation? What did you get in return?**

**Simon Schäfer**: Improving tools is something I seem to be naturally interested in. I spend a lot of time optimizing my own workflow and toolchain – even if the time that I have to invest is relatively long when you compare it with the outcome. Therefore it didn't take me long to start contributing to Scala IDE – there were just too many things that had to be and still need to be improved.

Originally I wanted to contribute to the compiler but at the beginning it was a big black box to me. I didn't know a lot about how the compiler works internally. Since Scala as a programming language was so different to what I was familiar with, I had no clue which things could be improved. The IDE was a lot more accessible because I could easily compare its functionality with the available Java IDEs. Even today they are much more powerful than the Scala IDE. Luckily, I got a lot better over time. Hence, nowadays there is not a huge knowledge-barrier that would prevent me from implementing whatever I want. The main factor which prevents me from working on a lot of things is that I don't have enough time.

Over the years I have solidified my base knowledge, but I still see things that could be improved. Actually, from the perspective of IDE development in general, I think that Scala's tooling is slowly reaching the state, where it can be even better than the tooling of more mainstream programming languages – where much more money has been invested. I'm thinking here about interactive tooling environments or live programming environments, which make developing code more fun and less painful. This movement has already been started and drew some attention with editors like [LightTable](http://lighttable.com/), but the available ones are still far from being used with general purpose applications that we use in the Scala world. There's big interest of the Scala community to move technology forward. It helps me to move Scala IDE in a similar direction. Also I believe that, thanks to this interest, we won't need to do all this work alone.

**Simon Ochsenreither**: Well, compared to many other contributors, my contributions are really really tiny. I think I just get an unfair advantage when you count the lines I have changed, because I work a lot in the deprecation/removal department. :-)

I prioritize my work by preferring issues which impact a large number of users and are unlikely to be tackled by others.

The problems I choose are usually small, somewhat easy to understand, and not of great urgency. Often it's about making the language or the standard library a tiny bit more consistent or getting rid of accumulated cruft.

I'm usually very busy and have a lot of other important priorities (family, friends, university, etc...), so I have to focus on getting things done; I don't really experiment much in the Scala space.

Sadly, the only thing I get in return are more ideas on what to work on. :-)
Like when I was travelling back from ScalaDays Amsterdam with Simon and Mathias Doenitz, and realized that there seemed to be no regular Scala conference in Germany – unlike many other European countries with one to three conferences each year!
A few days later I'm now looking into organizing a Scala conference in Germany next year.

**Michał Pociecha: Are such things what you'd like to do as a regular job?**

**Simon Ochsenreither**: What I like about open-source development is the community, the chance to have a positive impact on a lot of people, the ability to decide on what to work on, the lack of office politics and the focus on using the best tools for the job (Scala, Git & GitHub, Continuous Integration, Testing, etc...) – and then automating it.

It might be challenging to find companies which are at the very least sympathetic to these values, but I'm not too concerned about it right now. Being self-employed or starting my own company is certainly an option.

What motivates me to stay in this field is the excitement that we are only at the beginning of discovering the secrets of computer science, and the hope that – with enough shared effort and sincere work – software development can evolve from a handicraft to a real engineering discipline.

**Simon Schäfer**: I fear that working on tools is the only thing I would enjoy on a longer time period. I'm more or less only interested in working on software that either gives me technical knowledge or improves my own life. Working on tools has the advantage that you immediately can use your own improvements – this means that nearly everything I did for Scala IDE has its origin in my own desire to use it for myself. I'd also like to stay in the open source world because it makes it easy to contribute the changes I desire.

**Michał Pociecha: Do you have some advices to newcomers, maybe even students who would like to contribute to scalac and Scala IDE?**

**Simon Schäfer**: The best way to start is to tell people on the mailing list that one wants to contribute (and of course also on which feature one wants to work). Task trackers of [Scala IDE](https://scala-ide-portfolio.assembla.com) and [scalac](https://issues.scala-lang.org) are full of unresolved tickets. However, if someone is not yet familiar with how compilers or IDEs are built in general, they should expect a steep learning curve. There is a lot of technical knowledge needed. Luckily there are a lot of people on the mailing lists that are happy to help people getting up to speed, and nobody should be afraid of asking questions.

**Simon Ochsenreither**: I think that one can learn a lot by just reading pull requests and commits on GitHub.
If you have a really specific question about a change and ask on the PR "Can you explain how this change at line X fixes the issue? I'm a newcomer and trying to understand how scalac works.", I imagine that every Scala developer will be happy to help you understand it.

For less specific and all general questions a newcomer should ask on the [scala-internals mailing list](https://groups.google.com/forum/#!forum/scala-internals) or on the [scala/scala gitter channel](https://gitter.im/scala/scala).

Reading through the source of the compiler can be quite overwhelming at the beginning, but the good thing is that you don't need to know everything to fix an issue. There are thousands of tests which will tell you immediately whether you have accidentally broken something.

**Michał Pociecha: sschaef, you participated in Google Summer of Code. Could you tell something about your experiences? E.g. how did you work with the Scala IDE team during GSoC? By the way, at VirtusLab we've also started a program for students contributing to open source.**

**Simon Schäfer**: My experience with GSoC was great. On the one hand I learned a lot of new things, on the other I had the time to work on things which I couldn't have done previously due to time constraints. There are a lot of areas related to the IDE that I simply couldn't touch as a part time contributor. Another problem before GSoC was the steep learning curve, which forced me to invest too much time in learning instead of actually improving anything. Thanks to GSoC I could now touch the more difficult and more time consuming areas and afterwards I felt a lot more familiar with the codebase and therefore I could keep up when working on more complex tasks. In the end it was a big win for both me and for Scala IDE.

Over the years and also during the time when I participated in GSoC, I only worked remotely on Scala IDE. Therefore, all communication was organized through e-mail and Google Hangouts meetings. That always worked quite well even though it would sometimes have been nice if I could have gone to the next door and talked with a colleague about something.

**Michał Pociecha: If someone else would like to participate in GSoC and spend their holidays working on Scala IDE, what should they do?**

**Simon Schäfer**: GSoC actually is not for the holidays, it is for nearly an entire semester – similar to an internship. There are a few formal rules, which can be found on their [homepage](http://www.google-melange.com/gsoc/homepage/google/gsoc2015). I believe that the Scala IDE never officially announced anything to work on, which is mainly because our resources are too limited to handle a mentorship with open arms. Therefore if anyone wants to apply to GSoC together with Scala IDE, they need to come to us in due time by leaving a message on our mailing list. If one wants to increase their chances to get accepted for GSoC, it is helpful to not be a first time contributor. Sending a few even minor patches before GSoC officially starts is a big plus because it shows us that the candidate is interested in the project. It also gives us more hope that we don't loose them as a contributor after GSoC has ended. Contributing during holidays is easy of course, people just need to sit down and send some patches.

**Michał Pociecha: soc, what is your Scala editor of choice? I won't ask sschaef as we all know the answer.**

**Simon Ochsenreither**: I use the Scala IDE. I think Eclipse itself is quite terrible, but the Scala plugin is a lot more reliable than what IntelliJ offered the last time I checked.
The excellent work of the Scala IDE developers makes me tolerate Eclipse.

I'm also a heavy user of SBT's Eclipse plugin which generates the right Eclipse configuration files so that I don't have to interact with anything in Eclipse except the editor.

**Michał Pociecha: soc, during your second [talk at ScalaDays](https://www.parleys.com/tutorial/project-valhalla-part-2-value-types-jvm) you spoke about Project Valhalla (by the way, it was really interesting talk!). Since you monitor the situation, could you point out the most important changes and how they may affect Scala and its backend?**

**Simon Ochsenreither**: Project Valhalla is an experimental project led by Oracle which investigates adding support for value types and specialized Generics to the JVM.
Value types can be described as user-definable classes which act like primitives. Specialized Generics means that primitive types and value types will be supported in Generics and benefit from code which is optimized for these types, instead of boxing them to reference types.

Value types as proposed in Project Valhalla are actually pretty close to Scala's value types. There are a few remaining things which need to be figured out, like "which methods from java.lang.Object will exist on value types, and which implementation, if any, will be provided by default?" or "how will synchronization and locking primitives work, considering that value types don't have anything to 'lock' on?".

Specialized Generics will have a larger impact from my point of view.
From a Java-the-language perspective, the biggest change will be that Java will gain new syntax for these "new" kinds of Generics, because they have to preserve their "old" Generics as-is due to backward compatibility.

This is something which I'd like to avoid. I think if we have to add another syntax for generics to the language, we have failed. I'm slightly optimistic that support for these new Generics can be added without exposing all the differences of the different Generics in the syntax of the language.

Another issue that they don't plan to address in Project Valhalla is that their Generics-like Arrays will lack a common useful super-type of different instantiations. Just like int[] and String[] don't have a common parent except Object, it looks like it will be the same with ArrayList<int> and ArrayList<String>.

At ScalaDays San Francisco Vlad Ureche has [already suggested](https://www.parleys.com/tutorial/project-valhalla-the-good-bad-ugly) some pretty great ideas how this issue can be sorted out in Scala which largely preserve the language's semantics.

Project Valhalla is tentatively targeted for inclusion in Java 10, so it's still a few years away. But getting involved early is important, because only then there is enough time to debate, experiment, implement, complain, and – hopefully – get changes included which will make it easier for non-Java languages to run on the JVM.

From a Scala compiler perspective supporting value types might be easier, because it mainly involves emitting the right bytecode, not any large semantic changes to the language.
Specialized Generics will be harder to implement, because it seems like we will need to preserve some type information which is currently discarded in the erasure phase. Emitting the right bytecode will be a lot harder too. I'm not sure how much work ASM will do for us, and how much work has to be done on the Scala side. Additionally, there might be small changes to the language.

If everything works out, I think users will not have to worry about it. Just like Scala 2.12 targets Java 8 by default, a future version of Scala will target Java 10, and that's it.

**Michał Pociecha: What do you do when you don't contribute to the Scala ecosystem? What are you interested in?**

**Simon Ochsenreither**: I'm an avid runner, I love cooking and meeting friends. Family is really important to me, they always have my back and support me.
It's hard to balance everything when you have a lot to do, but I work on it. :-)

**Simon Schäfer**: I'm a martial artist. I used to train a lot when I was younger but nowadays, I'm afraid, I spend more time training my brain than my muscles. Unfortunately it seems to be less healthy (well, at least for the most of the time). Given that contributing to the Scala ecosystem is what I do in my free time, there is not so much time left to spend it on other things. And if I have some more spare time, well, I think about how to program. ;)

**Michał Pociecha: Thanks a lot, guys!**
