Our Java interpreter, jlox, taught us many of the fundamentals of programming
languages, but we still have much to learn. First, if you run any interesting
Lox programs in jlox, you'll discover it's achingly slow. The style of
interpretation it uses -- walking the AST directly -- is good enough for *some*
real-world uses, but leaves a lot to be desired for a general-purpose scripting
language.
我们的 Java 解释器 jlox 教会了我们许多编程语言的基础知识，但我们仍有许多东西要学。首先，如果你在 jlox 中运行任何有趣的 Lox 程序，你会发现它慢得令人心痛。它使用的解释方式 -- 直接遍历 AST -- 对于某些实际应用来说已经足够好了，但对于通用脚本语言来说，还有很多不足之处。

Also, we implicitly rely on runtime features of the JVM itself. We take for
granted that things like `instanceof` in Java work *somehow*. And we never for a
second worry about memory management because the JVM's garbage collector takes
care of it for us.
此外，我们还隐性地依赖于 JVM 本身的运行时特性。我们想当然地认为，Java 中的 `instanceof` 之类的东西会以 *某种方式* 正常运行。我们从未担心过内存管理问题，因为 JVM 的垃圾回收器会帮我们解决这个问题。

When we were focused on high-level concepts, it was fine to gloss over those.
But now that we know our way around an interpreter, it's time to dig down to
those lower layers and build our own virtual machine from scratch using nothing
more than the C standard library...
当我们专注于高层次概念时，对这些概念一带而过是没有问题的。但现在，我们对解释器已经了如指掌，是时候深入到底层，仅使用 C 标准库来从头开始构建我们自己的虚拟机了 ......

