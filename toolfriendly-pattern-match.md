#1. Cases in pattermaching

Standart Scala code:

<script src="https://gist.github.com/romanowski/837213790bdef72087cd.js"></script>

Nice one - that is why we love Scala

How tools would like to see the code?

<script src="https://gist.github.com/romanowski/6691a6b308188f1048a8.js"></script>

Only difference is line breaking. It is not python so why it matters?
Tools like code coverage or debuggers usually use JDI. And JDI is line-based - every line of bytecode has its mapping to line in source code.
Lets look at case statement in byte code:
<script src="https://gist.github.com/romanowski/2954e7b15f34c5ac407f.js"></script> 

Pretty much to do in one line?
And assume that we want to check if our case is eg. hitted. So we put a breakpoint in our case line. And what - it is hit everytime condition is checked!
Why? Checking the condition is also in our line - so tooling get information "I was there".
<h4>Conclusion:</h4>
Let's write cases in this way:
<script src="https://gist.github.com/romanowski/4a950e37a1a5fda4b306.js"></script>

Then when we place breakpoint inside case body - it will be hit only when the body is about to be invoked.

