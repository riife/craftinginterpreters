> If you find that you're spending almost all your time on theory, start turning
> some attention to practical things; it will improve your theories. If you find
> that you're spending almost all your time on practice, start turning some
> attention to theoretical things; it will improve your practice.
> 如果你发现你几乎把所有的时间都花在了理论上，那就开始把一些注意力转向实际的东西；这会提高你的理论水平。如果你发现你几乎把所有的时间都花在了实践上，那就开始把一些注意力转向理论上的东西；这将改善你的实践。
>
> <cite>Donald Knuth</cite>

We already have ourselves a complete implementation of Lox with jlox, so why
isn't the book over yet? Part of this is because jlox relies on the <span
name="metal">JVM</span> to do lots of things for us. If we want to understand
how an interpreter works all the way down to the metal, we need to build those
bits and pieces ourselves.
我们已经有了一个Lox 的完整实现jlox，那么为什么这本书还没有结束呢？部分原因是jlox依赖JVM为我们做很多事情。如果我们想要了解一个解释器是如何工作的，我们就需要自己构建这些零碎的东西。

<aside name="metal">

Of course, our second interpreter relies on the C standard library for basics
like memory allocation, and the C compiler frees us from details of the
underlying machine code we're running it on. Heck, that machine code is probably
implemented in terms of microcode on the chip. And the C runtime relies on the
operating system to hand out pages of memory. But we have to stop *somewhere* if
this book is going to fit on your bookshelf.
当然，我们的第二个解释器会依赖C标准库来实现内存分配等基本功能，而C编译器将我们从运行它的底层机器码的细节中解放出来。糟糕的是，该机器码可能是通过芯片上的微码来实现的。而C语言的运行时依赖于操作系统来分配内存页。但是，如果要想在你的书架放得下这本书，我们必须在 *某个地方* 停下来。

</aside>

An even more fundamental reason that jlox isn't sufficient is that it's too damn
slow. A tree-walk interpreter is fine for some kinds of high-level, declarative
languages. But for a general-purpose, imperative language -- even a "scripting"
language like Lox -- it won't fly. Take this little script:
jlox不够用的一个更根本的原因在于，它太慢了。树遍历解释器对于某些高级的声明式语言来说是不错的，但是对于通用的命令式语言——即使是Lox这样的“脚本”语言——这是行不通的。以下面的小脚本为例：

```lox
fun fib(n) {
  if (n < 2) return n;
  return fib(n - 1) + fib(n - 2); // [fib]
}

var before = clock();
print fib(40);
var after = clock();
print after - before;
```

<aside name="fib">

This is a comically inefficient way to actually calculate Fibonacci numbers.
Our goal is to see how fast the *interpreter* runs, not to see how fast of a
program we can write. A slow program that does a lot of work -- pointless or not
-- is a good test case for that.
这种计算斐波那契数列的方式效率低得可笑。我们的目的是查看 *解释器* 的运行速度，而不是看我们编写的程序有多快。一个做了大量工作的程序，无论是否有意义，都是一个很好的测试用例。

</aside>

On my laptop, that takes jlox about 72 seconds to execute. An equivalent C
program finishes in half a second. Our dynamically typed scripting language is
never going to be as fast as a statically typed language with manual memory
management, but we don't need to settle for more than *two orders of magnitude*
slower.
在我的笔记本电脑上，jlox大概需要72秒的时间来执行。一个等价的C程序在半秒内可以完成。我们的动态类型的脚本语言永远不可能像手动管理内存的静态类型语言那样快，但我们没必要满足于慢两个数量级以上的速度。

We could take jlox and run it in a profiler and start tuning and tweaking
hotspots, but that will only get us so far. The execution model -- walking the
AST -- is fundamentally the wrong design. We can't micro-optimize that to the
performance we want any more than you can polish an AMC Gremlin into an SR-71
Blackbird.
我们可以把jlox放在性能分析器中运行，并进行调优和调整热点，但这也只能到此为止了。它的执行模型（遍历AST）从根本上说就是一个错误的设计。我们无法将其微优化到我们想要的性能，就像你无法将AMC Gremlin打磨成SR-71 Blackbird一样。

We need to rethink the core model. This chapter introduces that model, bytecode,
and begins our new interpreter, clox.
我们需要重新考虑核心模型。本章将介绍这个模型——字节码，并开始我们的新解释器，clox。

## 字节码？

In engineering, few choices are without trade-offs. To best understand why we're
going with bytecode, let's stack it up against a couple of alternatives.
在工程领域，很少有选择是不需要权衡的。为了更好地理解我们为什么要使用字节码，让我们将它与几个备选方案进行比较。

### 为什么不遍历AST？

Our existing interpreter has a couple of things going for it:
我们目前的解释器有几个优点：

*   Well, first, we already wrote it. It's done. And the main reason it's done
    is because this style of interpreter is *really simple to implement*. The
    runtime representation of the code directly maps to the syntax. It's
    virtually effortless to get from the parser to the data structures we need
    at runtime.
    嗯，首先我们已经写好了，它已经完成了。它能完成的主要原因是这种风格的解释器*实现起来非常简单*。代码的运行时表示直接映射到语法。从解析器到我们在运行时需要的数据结构，几乎都毫不费力。

*   It's *portable*. Our current interpreter is written in Java and runs on any
    platform Java supports. We could write a new implementation in C using the
    same approach and compile and run our language on basically every platform
    under the sun.
    它是可移植的。我们目前的解释器是使用Java编写的，可以在Java支持的任何平台上运行。我们可以用同样的方法在C语言中编写一个新的实现，并在世界上几乎所有平台上编译并运行我们的语言。

Those are real advantages. But, on the other hand, it's *not memory-efficient*.
Each piece of syntax becomes an AST node. A tiny Lox expression like `1 + 2`
turns into a slew of objects with lots of pointers between them, something like:
这些是真正的优势。但是，另一方面，它的内存使用效率不高。每一段语法都会变成一个AST节点。像`1+2`这样的Lox表达式会变成一连串的对象，对象之间有很多指针，就像：

<span name="header"></span>

<aside name="header">

The "(header)" parts are the bookkeeping information the Java virtual machine
uses to support memory management and store the object's type. Those take up
space too!
"(header)" 部分是Java虚拟机用来支持内存管理和存储对象类型的记录信息，这些也会占用空间。

</aside>

<img src="image/chunks-of-bytecode/ast.png" alt="The tree of Java objects created to represent '1 + 2'." />

Each of those pointers adds an extra 32 or 64 bits of overhead to the object.
Worse, sprinkling our data across the heap in a loosely connected web of objects
does bad things for <span name="locality">*spatial locality*</span>.
每个指针都会给对象增加32或64比特的开销。更糟糕的是，将我们的数据散布在一个松散连接的对象网络中的堆上，会对空间局部性造成影响。

<aside name="locality">

I wrote [an entire chapter][gpp locality] about this exact problem in my first
book, *Game Programming Patterns*, if you want to really dig in.
在我的第一本书 *《游戏编程模式》* 中，我写了[一整章][gpp locality]来讨论这个问题，如果你想深入了解的话。

[gpp locality]: http://gameprogrammingpatterns.com/data-locality.html

</aside>

Modern CPUs process data way faster than they can pull it from RAM. To
compensate for that, chips have multiple layers of caching. If a piece of memory
it needs is already in the cache, it can be loaded more quickly. We're talking
upwards of 100 *times* faster.
现代CPU处理数据的速度远远超过它们从RAM中提取数据的速度。为了弥补这一点，芯片中有多层缓存。如果它需要的一块存储数据已经在缓存中，它就可以更快地被加载。我们谈论的是100倍以上的提速。

How does data get into that cache? The machine speculatively stuffs things in
there for you. Its heuristic is pretty simple. Whenever the CPU reads a bit of
data from RAM, it pulls in a whole little bundle of adjacent bytes and stuffs
them in the cache.
数据是如何进入缓存的？机器会推测性地为你把数据塞进去。它的启发式方法很简单。每当CPU从RAM中读取数据时，它就会拉取一块相邻的字节并放到缓存中。

If our program next requests some data close enough to be inside that cache
line, our CPU runs like a well-oiled conveyor belt in a factory. We *really*
want to take advantage of this. To use the cache effectively, the way we
represent code in memory should be dense and ordered like it's read.
如果我们的程序接下来请求一些在缓存行中的数据，那么我们的CPU就能像工厂里一条运转良好的传送带一样运行。我们真的很想利用这一点。为了有效的利用缓存，我们在内存中表示代码的方式应该像读取时一样紧密而有序。

Now look up at that tree. Those sub-objects could be <span
name="anywhere">*anywhere*</span>. Every step the tree-walker takes where it
follows a reference to a child node may step outside the bounds of the cache and
force the CPU to stall until a new lump of data can be slurped in from RAM. Just
the *overhead* of those tree nodes with all of their pointer fields and object
headers tends to push objects away from each other and out of the cache.
现在抬头看看那棵树。这些子对象可能在任何地方。树遍历器的每一步都会引用子节点，都可能会超出缓存的范围，并迫使CPU暂停，直到从RAM中拉取到新的数据块（才会继续执行）。仅仅是这些树形节点及其所有指针字段和对象头的开销，就会把对象彼此推离，并将其推出缓存区。

<aside name="anywhere">

Even if the objects happened to be allocated in sequential memory when the
parser first produced them, after a couple of rounds of garbage collection --
which may move objects around in memory -- there's no telling where they'll be.
即使在解析器首次生成对象时，这些对象碰巧被分配到了顺序内存中，但经过几轮垃圾回收（可能会在内存中移动对象）后，谁也不知道它们会在哪里。

</aside>

Our AST walker has other overhead too around interface dispatch and the Visitor
pattern, but the locality issues alone are enough to justify a better code
representation.
我们的AST遍历器在接口调度和Visitor模式方面还有其它开销，但仅仅是局部性问题就足以证明使用更好的代码表示是合理的。

### 为什么不编译成本地代码？

If you want to go *real* fast, you want to get all of those layers of
indirection out of the way. Right down to the metal. Machine code. It even
*sounds* fast. *Machine code.*
如果你想真正快，就要摆脱所有的中间层，一直到最底层——机器码。听起来就很快，*机器码*。

Compiling directly to the native instruction set the chip supports is what the
fastest languages do. Targeting native code has been the most efficient option
since way back in the early days when engineers actually <span
name="hand">handwrote</span> programs in machine code.
最快的语言所做的是直接把代码编译为芯片支持的本地指令集。从早期工程师真正用机器码手写程序以来，以本地代码为目标一直是最有效的选择。

<aside name="hand">

Yes, they actually wrote machine code by hand. On punched cards. Which,
presumably, they punched *with their fists*.
是的，他们实际上是用手写机器码的。写在打孔卡片上 据推测，他们是用 *拳头* 打出来的。

</aside>

If you've never written any machine code, or its slightly more human-palatable
cousin assembly code before, I'll give you the gentlest of introductions. Native
code is a dense series of operations, encoded directly in binary. Each
instruction is between one and a few bytes long, and is almost mind-numbingly
low level. "Move a value from this address to this register." "Add the integers
in these two registers." Stuff like that.
如果你以前从来没有写过任何机器码，或者是它略微讨人喜欢的近亲汇编语言，那我给你做一个简单的介绍。本地代码是一系列密集的操作，直接用二进制编码。每条指令的长度都在一到几个字节之间，而且几乎是令人头疼的底层指令。“将一个值从这个地址移动到这个寄存器”“将这两个寄存器中的整数相加”，诸如此类。

The CPU cranks through the instructions, decoding and executing each one in
order. There is no tree structure like our AST, and control flow is handled by
jumping from one point in the code directly to another. No indirection, no
overhead, no unnecessary skipping around or chasing pointers.
通过解码和按顺序执行指令来操作CPU。没有像AST那样的树状结构，控制流是通过从代码中的一个点跳到另一个点来实现的。没有中间层，没有开销，没有不必要的跳转或指针寻址。

Lightning fast, but that performance comes at a cost. First of all, compiling to
native code ain't easy. Most chips in wide use today have sprawling Byzantine
architectures with heaps of instructions that accreted over decades. They
require sophisticated register allocation, pipelining, and instruction
scheduling.
闪电般的速度，但这种性能是有代价的。首先，编译成本地代码并不容易。如今广泛使用的大多数芯片都有着庞大的拜占庭式架构，其中包含了几十年来积累的大量指令。它们需要复杂的寄存器分配、流水线和指令调度。

And, of course, you've thrown <span name="back">portability</span> out. Spend a
few years mastering some architecture and that still only gets you onto *one* of
the several popular instruction sets out there. To get your language on all of
them, you need to learn all of their instruction sets and write a separate back
end for each one.
当然，你可以把可移植性抛在一边。花费几年时间掌握一些架构，但这仍然只能让你接触到一些流行的指令集。为了让你的语言能在所有的架构上运行，你需要学习所有的指令集，并为每个指令集编写一个单独的后端。

<aside name="back">

The situation isn't entirely dire. A well-architected compiler lets you
share the front end and most of the middle layer optimization passes across the
different architectures you support. It's mainly the code generation and some of
the details around instruction selection that you'll need to write afresh each
time.
情况也没有那么可怕。一个架构良好的编译器，可以让你跨不同的架构共享前端和大部分中间层的优化通道。每次都需要重新编写的主要是代码生成和指令选择的一些细节。

The [LLVM][] project gives you some of this out of the box. If your compiler
outputs LLVM's own special intermediate language, LLVM in turn compiles that to
native code for a plethora of architectures.
[LLVM][]项目提供了一些开箱即用的功能。如果你的编译器输出LLVM自己特定的中间语言，LLVM可以反过来将其编译为各种架构的本地代码。

[llvm]: https://llvm.org/

</aside>

### 什么是字节码？

Fix those two points in your mind. On one end, a tree-walk interpreter is
simple, portable, and slow. On the other, native code is complex and
platform-specific but fast. Bytecode sits in the middle. It retains the
portability of a tree-walker -- we won't be getting our hands dirty with
assembly code in this book. It sacrifices *some* simplicity to get a performance
boost in return, though not as fast as going fully native.
记住这两点。一方面，树遍历解释器简单、可移植，而且慢。另一方面，本地代码复杂且特定与平台，但是很快。字节码位于中间。它保留了树遍历型的可移植性——在本书中我们不会编写汇编代码，同时它牺牲了一些简单性来换取性能的提升，虽然没有完全的本地代码那么快。

Structurally, bytecode resembles machine code. It's a dense, linear sequence of
binary instructions. That keeps overhead low and plays nice with the cache.
However, it's a much simpler, higher-level instruction set than any real chip
out there. (In many bytecode formats, each instruction is only a single byte
long, hence "bytecode".)
结构上讲，字节码类似于机器码。它是一个密集的、线性的二进制指令序列。这样可以保持较低的开销，并可以与高速缓存配合得很好。然而，它是一个更简单、更高级的指令集，比任何真正的芯片都要简单。（在很多字节码格式中，每条指令只有一个字节长，因此称为“字节码”）

Imagine you're writing a native compiler from some source language and you're
given carte blanche to define the easiest possible architecture to target.
Bytecode is kind of like that. It's an idealized fantasy instruction set that
makes your life as the compiler writer easier.
想象一下，你在用某种源语言编写一个本地编译器，并且你可以全权定义一个尽可能简单的目标架构。字节码就有点像这样，它是一个理想化的幻想指令集，可以让你作为编译器作者的生活更轻松。

The problem with a fantasy architecture, of course, is that it doesn't exist. We
solve that by writing an *emulator* -- a simulated chip written in software that
interprets the bytecode one instruction at a time. A *virtual machine (VM)*, if
you will.
当然，幻想架构的问题在于它并不存在。我们提供编写模拟器来解决这个问题，这个模拟器是一个用软件编写的芯片，每次会解释字节码的一条指令。如果你愿意的话，可以叫它*虚拟机（VM）*。

That emulation layer adds <span name="p-code">overhead</span>, which is a key
reason bytecode is slower than native code. But in return, it gives us
portability. Write our VM in a language like C that is already supported on all
the machines we care about, and we can run our emulator on top of any hardware
we like.
模拟层增加了开销，这是字节码比本地代码慢的一个关键原因。但作为回报，它为我们提供了可移植性。用像C这样的语言来编写我们的虚拟机，它已经被我们所关心的所有机器所支持，这样我们就可以在任何我们喜欢的硬件上运行我们的模拟器。

<aside name="p-code">

One of the first bytecode formats was [p-code][], developed for Niklaus Wirth's
Pascal language. You might think a PDP-11 running at 15MHz couldn't afford the
overhead of emulating a virtual machine. But back then, computers were in their
Cambrian explosion and new architectures appeared every day. Keeping up with the
latest chips was worth more than squeezing the maximum performance from each
one. That's why the "p" in p-code doesn't stand for "Pascal", but "portable".
最早的字节码格式之一是 [p-code]，是为 Niklaus Wirth的 Pascal 语言开发的。你可能会认为一个运行在15MHz的PDP-11 无法承担模拟虚拟机的开销。但在当时，计算机正处于寒武纪大爆发时期，每天都有新的架构出现。跟上最新的芯片要比从某个芯片中压榨出最大性能更有价值。这就是为什么p-code中的 "p" 指的不是 "Pascal" 而是 "portable"(可移植性) 。

[p-code]: https://en.wikipedia.org/wiki/P-code_machine

</aside>

This is the path we'll take with our new interpreter, clox. We'll follow in the
footsteps of the main implementations of Python, Ruby, Lua, OCaml, Erlang, and
others. In many ways, our VM's design will parallel the structure of our
previous interpreter:
这就是我们的新解释器clox要走的路。我们将追随Python、Ruby、Lua、OCaml、Erlang和其它主要语言实现的脚步。在许多方面，我们的VM设计将与之前的解释器结构并行。

<img src="image/chunks-of-bytecode/phases.png" alt="Phases of the two
implementations. jlox is Parser to Syntax Trees to Interpreter. clox is Compiler
to Bytecode to Virtual Machine." />

Of course, we won't implement the phases strictly in order. Like our previous
interpreter, we'll bounce around, building up the implementation one language
feature at a time. In this chapter, we'll get the skeleton of the application in
place and create the data structures needed to store and represent a chunk of
bytecode.
当然，我们不会严格按照顺序实现这些阶段。像我们之前的解释器一样，我们会反复地构建实现，每次只构建一种语言特性。在这一章中，我们将了解应用程序的框架，并创建用于存储和表示字节码块的数据结构。

## 开始

Where else to begin, but at `main()`? <span name="ready">Fire</span> up your
trusty text editor and start typing.
除了`main()`还能从哪里开始呢？启动你的文本编辑器，开始输入。

<aside name="ready">

Now is a good time to stretch, maybe crack your knuckles. A little montage music
wouldn't hurt either.
现在是舒展筋骨的好时机，也许还可以敲敲指关节。来点蒙太奇音乐也无妨。

</aside>

^code main-c

From this tiny seed, we will grow our entire VM. Since C provides us with so
little, we first need to spend some time amending the soil. Some of that goes
into this header:
从这颗小小的种子开始，我们将成长为整个VM。由于C提供给我们的东西太少，我们首先需要花费一些时间来培育土壤。其中一部分就在下面的header中。

^code common-h

There are a handful of types and constants we'll use throughout the interpreter,
and this is a convenient place to put them. For now, it's the venerable `NULL`,
`size_t`, the nice C99 Boolean `bool`, and explicit-sized integer types --
`uint8_t` and friends.
在整个解释器中，我们会使用一些类型和常量，这是一个方便放置它们的地方。现在，它是古老的`NULL`、`size_t`，C99中的布尔类型`bool`，以及显式声明大小的整数类型——`uint8_t`和它的朋友们。

## 指令块

Next, we need a module to define our code representation. I've been using
"chunk" to refer to sequences of bytecode, so let's make that the official name
for that module.
接下来，我们需要一个模块来定义我们的代码表示形式。我一直使用“chunk”指代字节码序列，所以我们把它作为该模块的正式名称。

^code chunk-h

In our bytecode format, each instruction has a one-byte **operation code**
(universally shortened to **opcode**). That number controls what kind of
instruction we're dealing with -- add, subtract, look up variable, etc. We
define those here:
在我们的字节码格式中，每个指令都有一个字节的**操作码**（通常简称为**opcode**）。这个数字控制我们要处理的指令类型——加、减、查找变量等。我们在这块定义这些：

^code op-enum (1 before, 2 after)

For now, we start with a single instruction, `OP_RETURN`. When we have a
full-featured VM, this instruction will mean "return from the current function".
I admit this isn't exactly useful yet, but we have to start somewhere, and this
is a particularly simple instruction, for reasons we'll get to later.
现在，我们从一条指令`OP_RETURN`开始。当我们有一个全功能的VM时，这个指令意味着“从当前函数返回”。我承认这还不是完全有用，但是我们必须从某个地方开始下手，而这是一个特别简单的指令，原因我们会在后面讲到。

### 指令动态数组

Bytecode is a series of instructions. Eventually, we'll store some other data
along with the instructions, so let's go ahead and create a struct to hold it
all.
字节码是一系列指令。最终，我们会与指令一起存储一些其它数据，所以让我们继续创建一个结构体来保存所有这些数据。

^code chunk-struct (1 before, 2 after)

At the moment, this is simply a wrapper around an array of bytes. Since we don't
know how big the array needs to be before we start compiling a chunk, it must be
dynamic. Dynamic arrays are one of my favorite data structures. That sounds like
claiming vanilla is my favorite ice cream <span name="flavor">flavor</span>, but
hear me out. Dynamic arrays provide:
目前，这只是一个字节数组的简单包装。由于我们在开始编译块之前不知道数组需要多大，所以它必须是动态的。动态数组是我最喜欢的数据结构之一。这听起来就像是在说香草是我最喜爱的冰淇淋口味，但请听我说完。动态数组提供了：

<aside name="flavor">

Butter pecan is actually my favorite.
黄油核桃其实是我的最爱。

</aside>

* Cache-friendly, dense storage
缓存友好，密集存储

* Constant-time indexed element lookup
索引元素查找为常量时间复杂度

* Constant-time appending to the end of the array
数组末尾追加元素为常量时间复杂度

Those features are exactly why we used dynamic arrays all the time in jlox under
the guise of Java's ArrayList class. Now that we're in C, we get to roll our
own. If you're rusty on dynamic arrays, the idea is pretty simple. In addition
to the array itself, we keep two numbers: the number of elements in the array we
have allocated ("capacity") and how many of those allocated entries are actually
in use ("count").
这些特性正是我们在jlox中以ArrayList类的名义一直使用动态数组的原因。现在我们在C语言中，可以推出我们自己的动态数组。如果你对动态数组不熟悉，其实这个想法非常简单。除了数组本身，我们还保留了两个数字：数组中已分配的元素数量（容量，capacity）和实际使用的已分配元数数量（计数，count）。

^code count-and-capacity (1 before, 2 after)

When we add an element, if the count is less than the capacity, then there is
already available space in the array. We store the new element right in there
and bump the count.
当添加元素时，如果计数小于容量，那么数组中已有可用空间。我们将新元素直接存入其中，并修改计数值。

<img src="image/chunks-of-bytecode/insert.png" alt="Storing an element in an
array that has enough capacity." />

If we have no spare capacity, then the process is a little more involved.
如果没有多余的容量，那么这个过程会稍微复杂一些。

<img src="image/chunks-of-bytecode/grow.png" alt="Growing the dynamic array
before storing an element." class="wide" />

1.  <span name="amortized">Allocate</span> a new array with more capacity. 分配一个容量更大的新数组。
2.  Copy the existing elements from the old array to the new one. 将旧数组中的已有元素复制到新数组中。
3.  Store the new `capacity`. 保存新的`capacity`。
4.  Delete the old array. 删除旧数组。

5.  Update `code` to point to the new array. 更新`code`指向新的数组。
6.  Store the element in the new array now that there is room. 现在有了空间，将元素存储在新数组中。
7.  Update the `count`. 更新`count`。

<aside name="amortized">

Copying the existing elements when you grow the array makes it seem like
appending an element is *O(n)*, not *O(1)* like I said above. However, you need
to do this copy step only on *some* of the appends. Most of the time, there is
already extra capacity, so you don't need to copy.
增长数组时会复制现有元素，使得追加元素的复杂度看起来像是O(n)，而不是O(1)。但是，你只需要在某些追加操作中执行这个操作步骤。大多数时候，已有多余的容量，所以不需要复制。

To understand how this works, we need [**amortized
analysis**](https://en.wikipedia.org/wiki/Amortized_analysis). That shows us
that as long as we grow the array by a multiple of its current size, when we
average out the cost of a *sequence* of appends, each append is *O(1)*.
要理解这一点，我们需要进行[**摊销分析**](https://en.wikipedia.org/wiki/Amortized_analysis)。这表明，只要我们把数组大小增加到当前大小的倍数，当我们把 *一系列* 追加操作的成本平均化时，每次追加都是 *O(1)*。

</aside>

We have our struct ready, so let's implement the functions to work with it. C
doesn't have constructors, so we declare a function to initialize a new chunk.
我们的结构体已经就绪，现在我们来实现和它相关的函数。C语言没有构造函数，所以我们声明一个函数来初始化一个新的块。

^code init-chunk-h (1 before, 2 after)

And implement it thusly:
并这样实现它：

^code chunk-c

The dynamic array starts off completely empty. We don't even allocate a raw
array yet. To append a byte to the end of the chunk, we use a new function.
动态数组一开始是完全空的。我们甚至还没有分配原始数组。要将一个字节追加到块的末尾，我们使用一个新函数。

^code write-chunk-h (1 before, 2 after)

This is where the interesting work happens.
这就是有趣的地方。

^code write-chunk

The first thing we need to do is see if the current array already has capacity
for the new byte. If it doesn't, then we first need to grow the array to make
room. (We also hit this case on the very first write when the array is `NULL`
and `capacity` is 0.)
我们需要做的第一件事是查看当前数组是否已经有容纳新字节的容量。如果没有，那么我们首先需要扩充数组以腾出空间（当我们第一个写入时，数组为`NULL`并且`capacity`为0，也会遇到这种情况）

To grow the array, first we figure out the new capacity and grow the array to
that size. Both of those lower-level memory operations are defined in a new
module.
要扩充数组，首先我们要算出新容量，然后将数组容量扩充到该大小。这两种低级别的内存操作都在一个新模块中定义。

^code chunk-c-include-memory (1 before, 2 after)

This is enough to get us started.
这就足够我们开始后面的事情了。

^code memory-h

This macro calculates a new capacity based on a given current capacity. In order
to get the performance we want, the important part is that it *scales* based on
the old size. We grow by a factor of two, which is pretty typical. 1.5&times; is
another common choice.
这个宏会根据给定的当前容量计算出新的容量。为了获得我们想要的性能，重要的部分就是基于旧容量大小进行扩展。我们以2的系数增长，这是一个典型的取值。1.5是另外一个常见的选择。

We also handle when the current capacity is zero. In that case, we jump straight
to eight elements instead of starting at one. That <span
name="profile">avoids</span> a little extra memory churn when the array is very
small, at the expense of wasting a few bytes on very small chunks.
我们还会处理当前容量为0的情况。在这种情况下，我们的容量直接跳到8，而不是从1开始。这就避免了在数组非常小的时候出现额外的内存波动，代价是在非常小的块中浪费几个字节。

<aside name="profile">

I picked the number eight somewhat arbitrarily for the book. Most dynamic array
implementations have a minimum threshold like this. The right way to pick a
value for this is to profile against real-world usage and see which constant
makes the best performance trade-off between extra grows versus wasted space.
我在这本书中选择了数字8，有些随意。大多数动态数组实现都有一个这样的最小阈值。挑选这个值的正确方法是根据实际使用情况进行分析，看看那个常数能在额外增长和浪费的空间之间做出最佳的性能权衡。

</aside>

Once we know the desired capacity, we create or grow the array to that size
using `GROW_ARRAY()`.
一旦我们知道了所需的容量，就可以使用`GROW_ARRAY()`创建或扩充数组到该大小。

^code grow-array (2 before, 2 after)

This macro pretties up a function call to `reallocate()` where the real work
happens. The macro itself takes care of getting the size of the array's element
type and casting the resulting `void*` back to a pointer of the right type.
这个宏简化了对`reallocate()`函数的调用，真正的工作就是在其中完成的。宏本身负责获取数组元素类型的大小，并将生成的`void*`转换成正确类型的指针。

This `reallocate()` function is the single function we'll use for all dynamic
memory management in clox -- allocating memory, freeing it, and changing the
size of an existing allocation. Routing all of those operations through a single
function will be important later when we add a garbage collector that needs to
keep track of how much memory is in use.
这个`reallocate()`函数是我们将在clox中用于所有动态内存管理的唯一函数——分配内存，释放内存以及改变现有分配的大小。当我们稍后添加一个需要跟踪内存使用情况的垃圾收集器时，通过单个函数路由所有这些操作是很重要的。

The two size arguments passed to `reallocate()` control which operation to
perform:
传递给`reallocate()` 函数的两个大小参数控制了要执行的操作：
| oldSize | newSize | Operation |
| | | |
| 0 | Non‑zero | Allocate new block. 分配新块 |
| Non‑zero | 0 | Free allocation. 释放已分配内存 |
| Non‑zero | Smaller than `oldSize` | Shrink existing allocation. 收缩已分配内存 |
| Non‑zero | Larger than `oldSize` | Grow existing allocation. 增加已分配内存 |

<table>
  <thead>
    <tr>
      <td>oldSize</td>
      <td>newSize</td>
      <td>Operation</td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>Non&#8209;zero</td>
      <td>Allocate new block.</td>
    </tr>
    <tr>
      <td>Non&#8209;zero</td>
      <td>0</td>
      <td>Free allocation.</td>
    </tr>
    <tr>
      <td>Non&#8209;zero</td>
      <td>Smaller&nbsp;than&nbsp;<code>oldSize</code></td>
      <td>Shrink existing allocation.</td>
    </tr>
    <tr>
      <td>Non&#8209;zero</td>
      <td>Larger&nbsp;than&nbsp;<code>oldSize</code></td>
      <td>Grow existing allocation.</td>
    </tr>
  </tbody>
</table>

That sounds like a lot of cases to handle, but here's the implementation:
看起来好像有很多情况需要处理，但下面是其实现：

^code memory-c

When `newSize` is zero, we handle the deallocation case ourselves by calling
`free()`. Otherwise, we rely on the C standard library's `realloc()` function.
That function conveniently supports the other three aspects of our policy. When
`oldSize` is zero, `realloc()` is equivalent to calling `malloc()`.
当`newSize`为0时，我们通过调用`free()`来自己处理回收的情况。其它情况下，我们依赖于C标准库的`realloc()`函数。该函数可以方便地支持我们策略中的其它三个场景。当`oldSize`为0时，`realloc()` 等同于调用`malloc()`。

The interesting cases are when both `oldSize` and `newSize` are not zero. Those
tell `realloc()` to resize the previously allocated block. If the new size is
smaller than the existing block of memory, it simply <span
name="shrink">updates</span> the size of the block and returns the same pointer
you gave it. If the new size is larger, it attempts to grow the existing block
of memory.
有趣的情况是当`oldSize`和`newSize`都不为0时。它们会告诉`realloc()`要调整之前分配的块的大小。如果新的大小小于现有的内存块，它就只是更新块的大小，并返回传入的指针。如果新块大小更大，它就会尝试增长现有的内存块。

It can do that only if the memory after that block isn't already in use. If
there isn't room to grow the block, `realloc()` instead allocates a *new* block
of memory of the desired size, copies over the old bytes, frees the old block,
and then returns a pointer to the new block. Remember, that's exactly the
behavior we want for our dynamic array.
只有在该块之后的内存未被使用的情况下，才能这样做。如果没有空间支持块的增长，`realloc()`会分配一个所需大小的*新*的内存块，复制旧的字节，释放旧内存块，然后返回一个指向新内存块的指针。记住，这正是我们的动态数组想要的行为。

Because computers are finite lumps of matter and not the perfect mathematical
abstractions computer science theory would have us believe, allocation can fail
if there isn't enough memory and `realloc()` will return `NULL`. We should
handle that.
因为计算机是有限的物质块，而不是计算机科学理论所认为的完美的数学抽象，如果没有足够的内存，分配就会失败，`reealloc()`会返回`NULL`。我们应该解决这个问题。

^code out-of-memory (1 before, 1 after)

There's not really anything *useful* that our VM can do if it can't get the
memory it needs, but we at least detect that and abort the process immediately
instead of returning a `NULL` pointer and letting it go off the rails later.
如果我们的VM不能得到它所需要的内存，那就做不了什么有用的事情，但我们至少可以检测这一点，并立即中止进程，而不是返回一个`NULL`指针，然后让程序运行偏离轨道。

<aside name="shrink">

Since all we passed in was a bare pointer to the first byte of memory, what does
it mean to "update" the block's size? Under the hood, the memory allocator
maintains additional bookkeeping information for each block of heap-allocated
memory, including its size.
既然我们传入的只是一个指向内存第一个字节的裸指针，那么“更新”块的大小意味着什么呢？在内部，内存分配器为堆分配的每个内存块都维护了额外的簿记信息，包括它的大小。

Given a pointer to some previously allocated memory, it can find this
bookkeeping information, which is necessary to be able to cleanly free it. It's
this size metadata that `realloc()` updates.
给定一个指向先前分配的内存的指针，它就可以找到这个簿记信息，为了能干净地释放内存，这是必需的。`realloc()`所更新的正是这个表示大小的元数据。

Many implementations of `malloc()` store the allocated size in memory right
*before* the returned address.
许多`malloc()`的实现将分配的大小存储在返回地址之前的内存中。

</aside>

OK, we can create new chunks and write instructions to them. Are we done? Nope!
We're in C now, remember, we have to manage memory ourselves, like in Ye Olden
Times, and that means *freeing* it too.
好了，我们可以创建新的块并向其中写入指令。我们完成了吗？不！要记住，我们现在是在C语言中，我们必须自己管理内存，就像在《Ye Olden Times》中那样，这意味着我们也要*释放*内存。
*
实现为:

^code free-chunk-h (1 before, 1 after)

The implementation is:
然后在实现中修改：

^code free-chunk

We deallocate all of the memory and then call `initChunk()` to zero out the
fields leaving the chunk in a well-defined empty state. To free the memory, we
add one more macro.
我们释放所有的内存，然后调用`initChunk()`将字段清零，使字节码块处于一个定义明确的空状态。为了释放内存，我们再添加一个宏。

^code free-array (3 before, 2 after)

Like `GROW_ARRAY()`, this is a wrapper around a call to `reallocate()`. This one
frees the memory by passing in zero for the new size. I know, this is a lot of
boring low-level stuff. Don't worry, we'll get a lot of use out of these in
later chapters and will get to program at a higher level. Before we can do that,
though, we gotta lay our own foundation.
与`GROW_ARRAY()`类似，这是对`reallocate()`调用的包装。这个函数通过传入0作为新的内存块大小，来释放内存。我知道，这是一堆无聊的低级别代码。别担心，在后面的章节中，我们会大量使用这些内容。但在此之前，我们必须先打好自己的基础。

## 反汇编字节码块

Now we have a little module for creating chunks of bytecode. Let's try it out by
hand-building a sample chunk.
现在我们有一个创建字节码块的小模块。让我们手动构建一个样例字节码块来测试一下。

^code main-chunk (1 before, 1 after)

Don't forget the include.
不要忘记include。

^code main-include-chunk (1 before, 2 after)

Run that and give it a try. Did it work? Uh... who knows? All we've done is push
some bytes around in memory. We have no human-friendly way to see what's
actually inside that chunk we made.
试着运行一下，它起作用了吗？额……谁知道呢。我们所做的只是在内存中存入一些字节。我们没有友好的方法来查看我们制作的字节码块中到底有什么。

To fix this, we're going to create a **disassembler**. An **assembler** is an
old-school program that takes a file containing human-readable mnemonic names
for CPU instructions like "ADD" and "MULT" and translates them to their binary
machine code equivalent. A *dis*assembler goes in the other direction -- given a
blob of machine code, it spits out a textual listing of the instructions.
为了解决这个问题，我们要创建一个**反汇编程序**。**汇编程序**是一个老式程序，它接收一个文件，该文件中包含CPU指令（如 "ADD "和 "MULT"）的可读助记符名称，并将它们翻译成等价的二进制机器代码。反汇编程序则相反——给定一串机器码，它会返回指令的文本列表。

We'll implement something <span name="printer">similar</span>. Given a chunk, it
will print out all of the instructions in it. A Lox *user* won't use this, but
we Lox *maintainers* will certainly benefit since it gives us a window into the
interpreter's internal representation of code.
我们将实现一个类似的模块。给定一个字节码块，它将打印出其中所有的指令。Lox用户不会使用它，但我们这些Lox的维护者肯定会从中受益，因为它给我们提供了一个了解解释器内部代码表示的窗口。

<aside name="printer">

In jlox, our analogous tool was the [AstPrinter class][].
在 jlox 中，我们的类似工具是 [AstPrinter 类][]。

[astprinter class]: representing-code.html#a-not-very-pretty-printer

</aside>

In `main()`, after we create the chunk, we pass it to the disassembler.
在`main()`中，我们创建字节码块后，将其传入反汇编器。

^code main-disassemble-chunk (2 before, 1 after)

Again, we whip up <span name="module">yet another</span> module.
我们又创建了另一个模块。

<aside name="module">

I promise you we won't be creating this many new files in later chapters.
我向你保证，在后面的章节中，我们不会再创建这么多新文件了。

</aside>

^code main-include-debug (1 before, 2 after)

Here's that header:
下面是这个头文件：

^code debug-h

In `main()`, we call `disassembleChunk()` to disassemble all of the instructions
in the entire chunk. That's implemented in terms of the other function, which
just disassembles a single instruction. It shows up here in the header because
we'll call it from the VM in later chapters.
在`main()`方法中，我们调用`disassembleChunk()`来反汇编整个字节码块中的所有指令。这是用另一个函数实现的，该函数只反汇编一条指令。因为我们将在后面的章节中从VM中调用它，所以将它添加到头文件中。

Here's a start at the implementation file:
下面是简单的实现文件：

^code debug-c

To disassemble a chunk, we print a little header (so we can tell *which* chunk
we're looking at) and then crank through the bytecode, disassembling each
instruction. The way we iterate through the code is a little odd. Instead of
incrementing `offset` in the loop, we let `disassembleInstruction()` do it for
us. When we call that function, after disassembling the instruction at the given
offset, it returns the offset of the *next* instruction. This is because, as
we'll see later, instructions can have different sizes.
要反汇编一个字节码块，我们首先打印一个小标题（这样我们就知道正在看哪个字节码块），然后通过字节码反汇编每个指令。我们遍历代码的方式有点奇怪。我们没有在循环中增加`offset`，而是让`disassembleInstruction()` 为我们做这个。当我们调用该函数时，在对给定偏移量的位置反汇编指令后，会返回*下一条*指令的偏移量。这是因为，我们后面也会看到，指令可以有不同的大小。

The core of the "debug" module is this function:
“debug”模块的核心是这个函数：

^code disassemble-instruction

First, it prints the byte offset of the given instruction -- that tells us where
in the chunk this instruction is. This will be a helpful signpost when we start
doing control flow and jumping around in the bytecode.
首先，它会打印给定指令的字节偏移量——这能告诉我们当前指令在字节码块中的位置。当我们在字节码中实现控制流和跳转时，这将是一个有用的路标。

Next, it reads a single byte from the bytecode at the given offset. That's our
opcode. We <span name="switch">switch</span> on that. For each kind of
instruction, we dispatch to a little utility function for displaying it. On the
off chance that the given byte doesn't look like an instruction at all -- a bug
in our compiler -- we print that too. For the one instruction we do have,
`OP_RETURN`, the display function is:
接下来，它从字节码中的给定偏移量处读取一个字节。这也就是我们的操作码。我们根据该值做switch操作。对于每一种指令，我们都分派给一个小的工具函数来展示它。如果给定的字节看起来根本不像一条指令——这是我们编译器的一个错误——我们也要打印出来。对于我们目前仅有的一条指令`OP_RETURN`，对应的展示函数是：

<aside name="switch">

We have only one instruction right now, but this switch will grow throughout the
rest of the book.
我们现在只有一个指令，但在本书的其余部分， switch 会越来越多。

</aside>

^code simple-instruction

There isn't much to a return instruction, so all it does is print the name of
the opcode, then return the next byte offset past this instruction. Other
instructions will have more going on.
return指令的内容不多，所以它所做的只是打印操作码的名称，然后返回该指令后的下一个字节偏移量。其它指令会有更多的内容。

If we run our nascent interpreter now, it actually prints something:
如果我们现在运行我们的新解释器，它实际上会打印出来：

```text
== test chunk ==
0000 OP_RETURN
```

It worked! This is sort of the "Hello, world!" of our code representation. We
can create a chunk, write an instruction to it, and then extract that
instruction back out. Our encoding and decoding of the binary bytecode is
working.
成功了！这有点像我们代码表示中的“Hello, world!”。我们可以创建一个字节码块，向其中写入一条指令，然后将该指令提取出来。我们对二进制字节码的编码和解码工作正常。

## 常量

Now that we have a rudimentary chunk structure working, let's start making it
more useful. We can store *code* in chunks, but what about *data*? Many values
the interpreter works with are created at runtime as the result of operations.
现在我们有了一个基本的块结构，我们来让它变得更有用。我们可以在块中存储*代码*，但是*数据*呢？解释器中使用的很多值都是在运行时作为操作的结果创建的。

```lox
1 + 2;
```

The value 3 appears nowhere in the code here. However, the literals `1` and `2`
do. To compile that statement to bytecode, we need some sort of instruction that
means "produce a constant" and those literal values need to get stored in the
chunk somewhere. In jlox, the Expr.Literal AST node held the value. We need a
different solution now that we don't have a syntax tree.
这里的代码中没有出现3这个值。但是，字面量`1`和`2`出现了。为了将该语句编译成字节码，我们需要某种指令，其含义是“生成一个常量”，而这些字母值需要存储在字节码块中的某个地方。在jlox中，Expr.Literal 这个AST节点中保存了这些值。因为我们没有语法树，现在我们需要一个不同的解决方案。

### 表示值

We won't be *running* any code in this chapter, but since constants have a foot
in both the static and dynamic worlds of our interpreter, they force us to start
thinking at least a little bit about how our VM should represent values.
在本章中我们不会运行任何代码，但是由于常量在解释器的静态和动态世界中都有涉足，这会迫使我们开始思考我们的虚拟机中应该如何表示数值。

For now, we're going to start as simple as possible -- we'll support only
double-precision, floating-point numbers. This will obviously expand over time,
so we'll set up a new module to give ourselves room to grow.
现在，我们尽可能从最简单的开始——只支持双精度浮点数。这种表示形式显然会逐渐扩大，所以我们将建立一个新的模块，给自己留出扩展的空间。

^code value-h

This typedef abstracts how Lox values are concretely represented in C. That way,
we can change that representation without needing to go back and fix existing
code that passes around values.
这个类型定义抽象了Lox值在C语言中的具体表示方式。这样，我们就可以直接改变表示方法，而不需要回去修改现有的传递值的代码。

Back to the question of where to store constants in a chunk. For small
fixed-size values like integers, many instruction sets store the value directly
in the code stream right after the opcode. These are called **immediate
instructions** because the bits for the value are immediately after the opcode.
回到在字节码块中存储常量的问题。对于像整数这种固定大小的值，许多指令集直接将值存储在操作码之后的代码流中。这些指令被称为**即时指令**，因为值的比特位紧跟在操作码之后。

That doesn't work well for large or variable-sized constants like strings. In a
native compiler to machine code, those bigger constants get stored in a separate
"constant data" region in the binary executable. Then, the instruction to load a
constant has an address or offset pointing to where the value is stored in that
section.
对于字符串这种较大的或可变大小的常量来说，这并不适用。在本地编译器的机器码中，这些较大的常量会存储在二进制可执行文件中的一个单独的“常量数据”区域。然后，加载常量的指令会有一个地址和偏移量，指向该值在区域中存储的位置。

Most virtual machines do something similar. For example, the Java Virtual
Machine [associates a **constant pool**][jvm const] with each compiled class.
That sounds good enough for clox to me. Each chunk will carry with it a list of
the values that appear as literals in the program. To keep things <span
name="immediate">simpler</span>, we'll put *all* constants in there, even simple
integers.

[jvm const]: https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.4

<aside name="immediate">

In addition to needing two kinds of constant instructions -- one for immediate
values and one for constants in the constant table -- immediates also force us
to worry about alignment, padding, and endianness. Some architectures aren't
happy if you try to say, stuff a 4-byte integer at an odd address.
除了需要两种常量指令（一种用于即时值，一种用于常量表中的常量）之外，即时指令还要求我们考虑对齐、填充和字节顺序的问题。如果你尝试在一个奇数地址填充一个4字节的整数，有些架构中会出错。

</aside>

### 值数组

The constant pool is an array of values. The instruction to load a constant
looks up the value by index in that array. As with our <span
name="generic">bytecode</span> array, the compiler doesn't know how big the
array needs to be ahead of time. So, again, we need a dynamic one. Since C
doesn't have generic data structures, we'll write another dynamic array data
structure, this time for Value.
常量池是一个值的数组。加载常量的指令根据数组中的索引查找该数组中的值。与字节码数组一样，编译器也无法提前知道这个数组需要多大。因此，我们需要一个动态数组。由于C语言没有通用数据结构，我们将编写另一个动态数组数据结构，这次存储的是Value。

<aside name="generic">

Defining a new struct and manipulation functions each time we need a dynamic
array of a different type is a chore. We could cobble together some preprocessor
macros to fake generics, but that's overkill for clox. We won't need many more
of these.
我这里对于“加载”或“产生”一个常量的含义含糊其辞，因为我们还没有学到虚拟机在运行时是如何执行的代码的。关于这一点，你必须等到（或者直接跳到）下一章。

</aside>

^code value-array (1 before, 2 after)

As with the bytecode array in Chunk, this struct wraps a pointer to an array
along with its allocated capacity and the number of elements in use. We also
need the same three functions to work with value arrays.
与Chunk中的字节码数组一样，这个结构体包装了一个指向数组的指针，以及其分配的容量和已使用元素的数量。我们也需要相同的三个函数来处理值数组。

^code array-fns-h (1 before, 2 after)

The implementations will probably give you déjà vu. First, to create a new one:
对应的实现可能会让你有似曾相识的感觉。首先，创建一个新文件：

^code value-c

Once we have an initialized array, we can start <span name="add">adding</span>
values to it.
一旦我们有了初始化的数组，我们就可以开始向其中添加值。

<aside name="add">

Fortunately, we don't need other operations like insertion and removal.
幸运的是，我们不需要插入和移除等其他操作。

</aside>

^code write-value-array

The memory-management macros we wrote earlier do let us reuse some of the logic
from the code array, so this isn't too bad. Finally, to release all memory used
by the array:
我们之前写的内存管理宏确实让我们重用了代码数组中的一些逻辑，所以这并不是太糟糕。最后，释放数组所使用的所有内存：

^code free-value-array

Now that we have growable arrays of values, we can add one to Chunk to store the
chunk's constants.
现在我们有了可增长的值数组，我们可以向Chunk中添加一个来保存字节码块中的常量值。

^code chunk-constants (1 before, 1 after)

Don't forget the include.
不要忘记include。

^code chunk-h-include-value (1 before, 2 after)

Ah, C, and its Stone Age modularity story. Where were we? Right. When we
initialize a new chunk, we initialize its constant list too.
初始化新的字节码块时，我们也要初始化其常量值列表。

^code chunk-init-constant-array (1 before, 1 after)

Likewise, we free the constants when we free the chunk.
同样地，我们在释放字节码块时，也需要释放常量值。

^code chunk-free-constants (1 before, 1 after)

Next, we define a convenience method to add a new constant to the chunk. Our
yet-to-be-written compiler could write to the constant array inside Chunk
directly -- it's not like C has private fields or anything -- but it's a little
nicer to add an explicit function.
接下来，我们定义一个便捷的方法来向字节码块中添加一个新常量。我们尚未编写的编译器可以在Chunk内部直接把常量值写入常量数组——它不像C语言那样有私有字段之类的东西——但是添加一个显式函数显然会更好一些。

^code add-constant-h (1 before, 2 after)

Then we implement it.
然后我们实现它。

^code add-constant

After we add the constant, we return the index where the constant was appended
so that we can locate that same constant later.
在添加常量之后，我们返回追加常量的索引，以便后续可以定位到相同的常量。

### 常量指令

We can *store* constants in chunks, but we also need to *execute* them. In a
piece of code like:
我们可以将常量存储在字节码块中，但是我们也需要执行它们。在如下这段代码中：

```lox
print 1;
print 2;
```

The compiled chunk needs to not only contain the values 1 and 2, but know *when*
to produce them so that they are printed in the right order. Thus, we need an
instruction that produces a particular constant.
编译后的字节码块不仅需要包含数值1和2，还需要知道何时生成它们，以便按照正确的顺序打印它们。因此，我们需要一种产生特定常数的指令。

^code op-constant (1 before, 1 after)

When the VM executes a constant instruction, it <span name="load">"loads"</span>
the constant for use. This new instruction is a little more complex than
`OP_RETURN`. In the above example, we load two different constants. A single
bare opcode isn't enough to know *which* constant to load.
当VM执行常量指令时，它会“加载”常量以供使用。这个新指令比`OP_RETURN`要更复杂一些。在上面的例子中，我们加载了两个不同的常量。一个简单的操作码不足以知道要加载哪个常量。

<aside name="load">

I'm being vague about what it means to "load" or "produce" a constant because we
haven't learned how the virtual machine actually executes code at runtime yet.
For that, you'll have to wait until you get to (or skip ahead to, I suppose) the
[next chapter][vm].
我之所以对 "加载" 或 "生成" 常量的含义含糊其辞，是因为我们还没有学习虚拟机如何在运行时实际执行代码。关于这一点，你必须等到（或者跳到）[下一章][vm]。

[vm]: a-virtual-machine.html

</aside>

To handle cases like this, our bytecode -- like most others -- allows
instructions to have <span name="operand">**operands**</span>. These are stored
as binary data immediately after the opcode in the instruction stream and let us
parameterize what the instruction does.
为了处理这样的情况，我们的字节码像大多数其它字节码一样，允许指令有**操作数**。这些操作数以二进制数据的形式存储在指令流的操作码之后，让我们对指令的操作进行参数化。

<img src="image/chunks-of-bytecode/format.png" alt="OP_CONSTANT is a byte for
the opcode followed by a byte for the constant index." />

Each opcode determines how many operand bytes it has and what they mean. For
example, a simple operation like "return" may have no operands, where an
instruction for "load local variable" needs an operand to identify which
variable to load. Each time we add a new opcode to clox, we specify what its
operands look like -- its **instruction format**.
每个操作码会定义它有多少操作数以及各自的含义。例如，一个像“return”这样简单的操作可能没有操作数，而一个“加载局部变量”的指令需要一个操作数来确定要加载哪个变量。每次我们向clox添加一个新的操作码时，我们都会指定它的操作数是什么样子的——即它的**指令格式**。

<aside name="operand">

Bytecode instruction operands are *not* the same as the operands passed to an
arithmetic operator. You'll see when we get to expressions that arithmetic
operand values are tracked separately. Instruction operands are a lower-level
notion that modify how the bytecode instruction itself behaves.
字节码指令的操作数与传递给算术操作符的操作数不同。当我们讲到表达式时，你会看到算术操作数的值是被单独跟踪的。指令操作数是一个较低层次的概念，它可以修改字节码指令本身的行为方式。

</aside>

In this case, `OP_CONSTANT` takes a single byte operand that specifies which
constant to load from the chunk's constant array. Since we don't have a compiler
yet, we "hand-compile" an instruction in our test chunk.
在这种情况下，`OP_CONSTANT`会接受一个单字节的操作数，该操作数指定从块的常量数组中加载哪个常量。由于我们还没有编译器，所以我们在测试字节码块中“手动编译”一个指令。

^code main-constant (1 before, 1 after)

We add the constant value itself to the chunk's constant pool. That returns the
index of the constant in the array. Then we write the constant instruction,
starting with its opcode. After that, we write the one-byte constant index
operand. Note that `writeChunk()` can write opcodes or operands. It's all raw
bytes as far as that function is concerned.
我们将常量值添加到字节码块的常量池中。这会返回常量在数组中的索引。然后我们写常量操作指令，从操作码开始。之后，我们写入一字节的常量索引操作数。注意， `writeChunk()` 可以写操作码或操作数。对于该函数而言，它们都是原始字节。

If we try to run this now, the disassembler is going to yell at us because it
doesn't know how to decode the new instruction. Let's fix that.
如果我们现在尝试运行上面的代码，反汇编器会遇到问题，因为它不知道如何解码新指令。让我们来修复这个问题。

^code disassemble-constant (1 before, 1 after)

This instruction has a different instruction format, so we write a new helper
function to disassemble it.
这条指令的格式有所不同，所以我们编写一个新的辅助函数来对其反汇编。

^code constant-instruction

There's more going on here. As with `OP_RETURN`, we print out the name of the
opcode. Then we pull out the constant index from the subsequent byte in the
chunk. We print that index, but that isn't super useful to us human readers. So
we also look up the actual constant value -- since constants *are* known at
compile time after all -- and display the value itself too.
这里要做的事情更多一些。与`OP_ETURN`一样，我们会打印出操作码的名称。然后，我们从该字节码块的后续字节中获取常量索引。我们打印出这个索引值，但是这对于我们人类读者来说并不十分有用。所以，我们也要查找实际的常量值——因为常量毕竟是在编译时就知道的——并将这个值也展示出来。

This requires some way to print a clox Value. That function will live in the
"value" module, so we include that.
这就需要一些方法来打印clox中的一个Value。这个函数放在“value”模块中，所以我们要将其include。

^code debug-include-value (1 before, 2 after)

Over in that header, we declare:
在这个头文件中，我们声明：

^code print-value-h (1 before, 2 after)

And here's an implementation:
下面是对应的实现：

^code print-value

Magnificent, right? As you can imagine, this is going to get more complex once
we add dynamic typing to Lox and have values of different types.
很壮观，是吧？你可以想象，一旦我们在Lox中加入动态类型，并且包含了不同类型的值，这部分将会变得更加复杂。

Back in `constantInstruction()`, the only remaining piece is the return value.
回到`constantInstruction()`中，唯一剩下的部分就是返回值。

^code return-after-operand (1 before, 1 after)

Remember that `disassembleInstruction()` also returns a number to tell the
caller the offset of the beginning of the *next* instruction. Where `OP_RETURN`
was only a single byte, `OP_CONSTANT` is two -- one for the opcode and one for
the operand.
记住，`disassembleInstruction()`也会返回一个数字，告诉调用方*下一条*指令的起始位置的偏移量。`OP_RETURN`只有一个字节，而`OP_CONSTANT`有两个字节——一个是操作码，一个是操作数。

## 行信息

Chunks contain almost all of the information that the runtime needs from the
user's source code. It's kind of crazy to think that we can reduce all of the
different AST classes that we created in jlox down to an array of bytes and an
array of constants. There's only one piece of data we're missing. We need it,
even though the user hopes to never see it.
字节码块中几乎包含了运行时需要从用户源代码中获取的所有信息。想到我们可以把jlox中不同的AST类减少到一个字节数组和一个常量数组，这实在有一点疯狂。我们只缺少一个数据。我们需要它，尽管用户希望永远不会看到它。

When a runtime error occurs, we show the user the line number of the offending
source code. In jlox, those numbers live in tokens, which we in turn store in
the AST nodes. We need a different solution for clox now that we've ditched
syntax trees in favor of bytecode. Given any bytecode instruction, we need to be
able to determine the line of the user's source program that it was compiled
from.
当运行时错误发生时，我们会向用户显示出错的源代码的行号。在jlox中，这些数字保存在词法标记中，而我们又将词法标记存储在AST节点中。既然我们已经抛弃了语法树而采用了字节码，我们就需要为clox提供不同的解决方案。对于任何字节码指令，我们需要能够确定它是从用户源代码的哪一行编译出来的。

There are a lot of clever ways we could encode this. I took the absolute <span
name="side">simplest</span> approach I could come up with, even though it's
embarrassingly inefficient with memory. In the chunk, we store a separate array
of integers that parallels the bytecode. Each number in the array is the line
number for the corresponding byte in the bytecode. When a runtime error occurs,
we look up the line number at the same index as the current instruction's offset
in the code array.
我们有很多聪明的方法可以对此进行编码。我采取了我能想到的绝对最简单的方法，尽管这种方法的内存效率低得令人发指。在字节码块中，我们存储一个单独的整数数组，该数组与字节码平级。数组中的每个数字都是字节码中对应字节所在的行号。当发生运行时错误时，我们根据当前指令在代码数组中的偏移量查找对应的行号。

<aside name="side">

This braindead encoding does do one thing right: it keeps the line information
in a *separate* array instead of interleaving it in the bytecode itself. Since
line information is only used when a runtime error occurs, we don't want it
between the instructions, taking up precious space in the CPU cache and causing
more cache misses as the interpreter skips past it to get to the opcodes and
operands it cares about.
这种脑残的编码至少做对了一件事：它将行信息保存一个单独的数组中，而不是将其编入字节码本身中。由于行信息只在运行时出现错误时才使用，我们不希望它在指令之间占用CPU缓存中的宝贵空间，而且解释器在跳过行数获取它所关心的操作码和操作数时，会造成更多的缓存丢失。

</aside>

To implement this, we add another array to Chunk.
为了实现这一点，我们向Chunk中添加另一个数组。

^code chunk-lines (1 before, 1 after)

Since it exactly parallels the bytecode array, we don't need a separate count or
capacity. Every time we touch the code array, we make a corresponding change to
the line number array, starting with initialization.
由于它与字节码数组完全平行，我们不需要单独的计数值和容量值。每次我们访问代码数组时，也会对行号数组做相应的修改，从初始化开始。

^code chunk-null-lines (1 before, 1 after)

And likewise deallocation:
回收也是类似的：

^code chunk-free-lines (1 before, 1 after)

When we write a byte of code to the chunk, we need to know what source line it
came from, so we add an extra parameter in the declaration of `writeChunk()`.
当我们向块中写入一个代码字节时，我们需要知道它来自哪个源代码行，所以我们在`writeChunk()`的声明中添加一个额外的参数。

^code write-chunk-with-line-h (1 before, 1 after)

And in the implementation:
然后在实现中修改：

^code write-chunk-with-line (1 after)

When we allocate or grow the code array, we do the same for the line info too.
当我们分配或扩展代码数组时，我们也要对行信息进行相同的处理。

^code write-chunk-line (2 before, 1 after)

Finally, we store the line number in the array.
最后，我们在数组中保存行信息。

^code chunk-write-line (1 before, 1 after)

### 反汇编行信息

Alright, let's try this out with our little, uh, artisanal chunk. First, since
we added a new parameter to `writeChunk()`, we need to fix those calls to pass
in some -- arbitrary at this point -- line number.
好吧，让我们手动编译一个小的字节码块测试一下。首先，由于我们向`writeChunk()`添加了一个新参数，我们需要修改一下该方法的调用，向其中添加一些行号（这里可以随意选择行号值）。

^code main-chunk-line (1 before, 2 after)

Once we have a real front end, of course, the compiler will track the current
line as it parses and pass that in.
当然，一旦我们有了真正的前端，编译器会在解析时跟踪当前行，并将其传入字节码中。

Now that we have line information for every instruction, let's put it to good
use. In our disassembler, it's helpful to show which source line each
instruction was compiled from. That gives us a way to map back to the original
code when we're trying to figure out what some blob of bytecode is supposed to
do. After printing the offset of the instruction -- the number of bytes from the
beginning of the chunk -- we show its source line.
现在我们有了每条指令的行信息，让我们好好利用它吧。在我们的反汇编程序中，展示每条指令是由哪一行源代码编译出来的是很有帮助的。当我们试图弄清楚某些字节码应该做什么时，这给我们提供了一种方法来映射回原始代码。在打印了指令的偏移量之后——从字节码块起点到当前指令的字节数——我们也展示它在源代码中的行号。

^code show-location (2 before, 2 after)

Bytecode instructions tend to be pretty fine-grained. A single line of source
code often compiles to a whole sequence of instructions. To make that more
visually clear, we show a `|` for any instruction that comes from the same
source line as the preceding one. The resulting output for our handwritten
chunk looks like:
字节码指令往往是非常细粒度的。一行源代码往往可以编译成一个完整的指令序列。为了更直观地说明这一点，我们在与前一条指令来自同一源码行的指令前面显示一个“|”。我们的手写字节码块的输出结果如下所示：

```text
== test chunk ==
0000  123 OP_CONSTANT         0 '1.2'
0002    | OP_RETURN
```

We have a three-byte chunk. The first two bytes are a constant instruction that
loads 1.2 from the chunk's constant pool. The first byte is the `OP_CONSTANT`
opcode and the second is the index in the constant pool. The third byte (at
offset 2) is a single-byte return instruction.
我们有一个三字节的块。前两个字节是一个常量指令，从该块的常量池中加载1.2。第一个字节是`OP_CONSTANT`字节码，第二个是在常量池中的索引。第三个字节（偏移量为2）是一个单字节的返回指令。

In the remaining chapters, we will flesh this out with lots more kinds of
instructions. But the basic structure is here, and we have everything we need
now to completely represent an executable piece of code at runtime in our
virtual machine. Remember that whole family of AST classes we defined in jlox?
In clox, we've reduced that down to three arrays: bytes of code, constant
values, and line information for debugging.
在接下来的章节中，我们将用更多种类的指令来充实这个结构。但是基本结构已经在这里了，我们现在拥有了所需要的一切，可以在虚拟机运行时完全表示一段可执行的代码。还记得我们在jlox中定义的整个AST类族吗？在clox中，我们把它减少到了三个数组：代码字节数组，常量值数组，以及用于调试的行信息。

This reduction is a key reason why our new interpreter will be faster than jlox.
You can think of bytecode as a sort of compact serialization of the AST, highly
optimized for how the interpreter will deserialize it in the order it needs as
it executes. In the [next chapter][vm], we will see how the virtual machine does
exactly that.
这种减少是我们的新解释器比jlox更快的一个关键原因。你可以把字节码看作是AST的一种紧凑的序列化，并且解释器在执行时按照需要对其反序列化的方式进行了高度优化。在下一章中，我们将会看到虚拟机是如何做到这一点的。
:当然，我们的第二个解释器会依赖C标准库来实现内存分配等基本功能，而C编译器将我们从运行它的底层机器码的细节中解放出来。糟糕的是，该机器码可能是通过芯片上的微码来实现的。而C语言的运行时依赖于操作系统来分配内存页。但是，如果要想在你的书架放得下这本书，我们必须在某个地方停下来。
:这种计算斐波那契数列的方式效率低得可笑。我们的目的是查看解释器的运行速度，而不是看我们编写的程序有多快。一个做了大量工作的程序，无论是否有意义，都是一个很好的测试用例。
:“（header）”部分是Java虚拟机用来支持内存管理和存储对象类型的记录信息，这些也会占用空间。
:情况也没有那么可怕。一个架构良好的编译器，可以让你跨不同的架构共享前端和大部分中间层的优化通道。每次都需要重新编写的主要是代码生成和指令选择的一些细节。[LLVM](https://llvm.org/)项目提供了一些开箱即用的功能。如果你的编译器输出LLVM自己特定的中间语言，LLVM可以反过来将其编译为各种架构的本地代码。
:最早的字节码格式之一是p-code，是为Niklaus Wirth的Pascal语言开发的。你可能会认为一个运行在15MHz的PDP-11无法承担模拟虚拟机的开销。但在当时，计算机正处于寒武纪大爆发时期，每天都有新的架构出现。跟上最新的芯片要比从某个芯片中压榨出最大性能更有价值。这就是为什么p-code中的“p”指的不是“Pascal”而是“可移植性Portable”。
:增长数组时会复制现有元素，使得追加元素的复杂度看起来像是O(n)，而不是O(1)。但是，你只需要在某些追加操作中执行这个操作步骤。大多数时候，已有多余的容量，所以不需要复制。要理解这一点，我们需要进行[摊销分析](https://en.wikipedia.org/wiki/Amortized_analysis)。这表明，只要我们把数组大小增加到当前大小的倍数，当我们把一系列追加操作的成本平均化时，每次追加都是O(1)。
:我在这本书中选择了数字8，有些随意。大多数动态数组实现都有一个这样的最小阈值。挑选这个值的正确方法是根据实际使用情况进行分析，看看那个常数能在额外增长和浪费的空间之间做出最佳的性能权衡。
:既然我们传入的只是一个指向内存第一个字节的裸指针，那么“更新”块的大小意味着什么呢？在内部，内存分配器为堆分配的每个内存块都维护了额外的簿记信息，包括它的大小。给定一个指向先前分配的内存的指针，它就可以找到这个簿记信息，为了能干净地释放内存，这是必需的。`realloc()`所更新的正是这个表示大小的元数据。许多`malloc()`的实现将分配的大小存储在返回地址之前的内存中。
:除了需要两种常量指令（一种用于即时值，一种用于常量表中的常量）之外，即时指令还要求我们考虑对齐、填充和字节顺序的问题。如果你尝试在一个奇数地址填充一个4字节的整数，有些架构中会出错。
:我这里对于“加载”或“产生”一个常量的含义含糊其辞，因为我们还没有学到虚拟机在运行时是如何执行的代码的。关于这一点，你必须等到（或者直接跳到）下一章。
:字节码指令的操作数与传递给算术操作符的操作数不同。当我们讲到表达式时，你会看到算术操作数的值是被单独跟踪的。指令操作数是一个较低层次的概念，它可以修改字节码指令本身的行为方式。
: 这种脑残的编码至少做对了一件事：它将行信息保存一个单独的数组中，而不是将其编入字节码本身中。由于行信息只在运行时出现错误时才使用，我们不希望它在指令之间占用CPU缓存中的宝贵空间，而且解释器在跳过行数获取它所关心的操作码和操作数时，会造成更多的缓存丢失。

<div class="challenges">

## Challenges

1.  Our encoding of line information is hilariously wasteful of memory. Given
    that a series of instructions often correspond to the same source line, a
    natural solution is something akin to [run-length encoding][rle] of the line
    numbers.

    Devise an encoding that compresses the line information for a
    series of instructions on the same line. Change `writeChunk()` to write this
    compressed form, and implement a `getLine()` function that, given the index
    of an instruction, determines the line where the instruction occurs.
    设计一个编码方式，压缩同一行上一系列指令的行信息。修改`writeChunk()` 以写入该压缩形式，并实现一个`getLine()` 函数，给定一条指令的索引，确定该指令所在的行。

    *Hint: It's not necessary for `getLine()` to be particularly efficient.
    Since it is called only when a runtime error occurs, it is well off the
    critical path where performance matters.*
    *提示：`getLine()`不一定要特别高效。因为它只在出现运行时错误时才被调用，所以在它并不是影响性能的关键因素。*


2.  Because `OP_CONSTANT` uses only a single byte for its operand, a chunk may
    only contain up to 256 different constants. That's small enough that people
    writing real-world code will hit that limit. We could use two or more bytes
    to store the operand, but that makes *every* constant instruction take up
    more space. Most chunks won't need that many unique constants, so that
    wastes space and sacrifices some locality in the common case to support the
    rare case.
    因为`OP_CONSTANT`只使用一个字节作为操作数，所以一个块最多只能包含256个不同的常数。这已经够小了，用户在编写真正的代码时很容易会遇到这个限制。我们可以使用两个或更多字节来存储操作数，但这会使*每个*常量指令占用更多的空间。大多数字节码块都不需要那么多独特的常量，所以这就浪费了空间，并牺牲了一些常规情况下的局部性来支持罕见场景。

    To balance those two competing aims, many instruction sets feature multiple
    instructions that perform the same operation but with operands of different
    sizes. Leave our existing one-byte `OP_CONSTANT` instruction alone, and
    define a second `OP_CONSTANT_LONG` instruction. It stores the operand as a
    24-bit number, which should be plenty.
    为了平衡这两个相互冲突的目标，许多指令集具有多个执行相同操作但操作数大小不同的指令。保留现有的使用一个字节的`OP_CONSTANT`指令，并定义一个新的`OP_CONSTANT_LONG`指令。它将操作数存储为24位的数字，这应该就足够了。

    Implement this function:
    实现该函数：

    ```c
    void writeConstant(Chunk* chunk, Value value, int line) {
      // Implement me...
    }
    ```

    It adds `value` to `chunk`'s constant array and then writes an appropriate
    instruction to load the constant. Also add support to the disassembler for
    `OP_CONSTANT_LONG` instructions.
    它向`chunk`的常量数组中添加`value`，然后写一条合适的指令来加载常量。同时在反汇编程序中增加对 `OP_CONSTANT_LONG`指令的支持。

    Defining two instructions seems to be the best of both worlds. What
    sacrifices, if any, does it force on us?
    定义两条指令似乎是两全其美的办法。它会迫使我们做出什么牺牲呢（如果有的话）？


3.  Our `reallocate()` function relies on the C standard library for dynamic
    memory allocation and freeing. `malloc()` and `free()` aren't magic. Find
    a couple of open source implementations of them and explain how they work.
    How do they keep track of which bytes are allocated and which are free?
    What is required to allocate a block of memory? Free it? How do they make
    that efficient? What do they do about fragmentation?
    我们的`reallocate()`函数依赖于C标准库进行动态内存分配和释放。`malloc()` 和 `free()` 并不神奇。找几个它们的开源实现，并解释它们是如何工作的。它们如何跟踪哪些字节被分配，哪些被释放？分配一个内存块需要什么？释放的时候呢？它们如何实现高效？它们如何处理碎片化内存？

    *Hardcore mode:* Implement `reallocate()` without calling `realloc()`,
    `malloc()`, or `free()`. You are allowed to call `malloc()` *once*, at the
    beginning of the interpreter's execution, to allocate a single big block of
    memory, which your `reallocate()` function has access to. It parcels out
    blobs of memory from that single region, your own personal heap. It's your
    job to define how it does that.
    *硬核模式*：在不调用`realloc()`, `malloc()`, 和 `free()`的前提下，实现`reallocate()`。你可以在解释器开始执行时调用一次`malloc()`，来分配一个大的内存块，你的`reallocate()`函数能够访问这个内存块。它可以从这个区域（你自己的私人堆内存）中分配内存块。你的工作就是定义如何做到这一点。

</div>

[rle]: https://en.wikipedia.org/wiki/Run-length_encoding

<div class="design-note">

## Design Note: 测试你的语言

We're almost halfway through the book and one thing we haven't talked about is
*testing* your language implementation. That's not because testing isn't
important. I can't possibly stress enough how vital it is to have a good,
comprehensive test suite for your language.

I wrote a [test suite for Lox][tests] (which you are welcome to use on your own
Lox implementation) before I wrote a single word of this book. Those tests found
countless bugs in my implementations.

[tests]: https://github.com/munificent/craftinginterpreters/tree/master/test

Tests are important in all software, but they're even more important for a
programming language for at least a couple of reasons:

*   **Users expect their programming languages to be rock solid.** We are so
    used to mature, stable compilers and interpreters that "It's your code, not
    the compiler" is [an ingrained part of software culture][fault]. If there
    are bugs in your language implementation, users will go through the full
    five stages of grief before they can figure out what's going on, and you
    don't want to put them through all that.

*   **A language implementation is a deeply interconnected piece of software.**
    Some codebases are broad and shallow. If the file loading code is broken in
    your text editor, it -- hopefully! -- won't cause failures in the text
    rendering on screen. Language implementations are narrower and deeper,
    especially the core of the interpreter that handles the language's actual
    semantics. That makes it easy for subtle bugs to creep in caused by weird
    interactions between various parts of the system. It takes good tests to
    flush those out.

*   **The input to a language implementation is, by design, combinatorial.**
    There are an infinite number of possible programs a user could write, and
    your implementation needs to run them all correctly. You obviously can't
    test that exhaustively, but you need to work hard to cover as much of the
    input space as you can.

*   **Language implementations are often complex, constantly changing, and full
    of optimizations.** That leads to gnarly code with lots of dark corners
    where bugs can hide.

[fault]: https://blog.codinghorror.com/the-first-rule-of-programming-its-always-your-fault/

All of that means you're gonna want a lot of tests. But *what* tests? Projects
I've seen focus mostly on end-to-end "language tests". Each test is a program
written in the language along with the output or errors it is expected to
produce. Then you have a test runner that pushes the test program through your
language implementation and validates that it does what it's supposed to.
Writing your tests in the language itself has a few nice advantages:

*   The tests aren't coupled to any particular API or internal architecture
    decisions of the implementation. This frees you to reorganize or rewrite
    parts of your interpreter or compiler without needing to update a slew of
    tests.

*   You can use the same tests for multiple implementations of the language.

*   Tests can often be terse and easy to read and maintain since they are
    simply scripts in your language.

It's not all rosy, though:

*   End-to-end tests help you determine *if* there is a bug, but not *where* the
    bug is. It can be harder to figure out where the erroneous code in the
    implementation is because all the test tells you is that the right output
    didn't appear.

*   It can be a chore to craft a valid program that tickles some obscure corner
    of the implementation. This is particularly true for highly optimized
    compilers where you may need to write convoluted code to ensure that you
    end up on just the right optimization path where a bug may be hiding.

*   The overhead can be high to fire up the interpreter, parse, compile, and
    run each test script. With a big suite of tests -- which you *do* want,
    remember -- that can mean a lot of time spent waiting for the tests to
    finish running.

I could go on, but I don't want this to turn into a sermon. Also, I don't
pretend to be an expert on *how* to test languages. I just want you to
internalize how important it is *that* you test yours. Seriously. Test your
language. You'll thank me for it.

</div>


<div class="design-note">

我们的书已经过半了，有一件事我们还没有谈及，那就是*测试*你的语言实现。这并不是因为测试不重要。语言实现有一个好的、全面的套件是多么重要，我怎么强调都不为过。

在我写本书之前，我为Lox写了一个[测试套件](https://github.com/munificent/craftinginterpreters/tree/master/test)（你也可以在自己的Lox实现中使用它）。这些测试在我的语言实现中发现了无数的bug。

测试在所有软件中都很重要，但对于编程语言来说，测试甚至更重要，至少有以下几个原因：

* **用户希望他们的编程语言能够坚如磐石**。我们已经习惯了成熟的编译器、解释器，以至于“是你的代码（出错了），而不是编译器”成为[软件文化中根深蒂固的一部分](https://blog.codinghorror.com/the-first-rule-of-programming-its-always-your-fault/)。如果你的语言实现中有错误，用户需要经历全部五个痛苦的阶段才能弄清楚发生了什么，而你并不想让他们经历这一切。
* **语言的实现是一个紧密相连的软件**。有些代码库既广泛又浮浅。如果你的文本编辑器中的文件加载代码被破坏了，它不会导致屏幕上的文本渲染失败（希望如此）。语言的实现则更狭窄和深入，特别是处理语言实际语义的解释器核心部分。这使得系统的各个部分之间奇怪的交互会造成微妙的错误。这就需要好的测试来清除这些问题。
* **从设计上来说，语言实现的输入是组合性的**。用户可以写出无限多的程序，而你的实现需要能够正确地运行这些程序。您显然不能进行详尽地测试，但需要努力覆盖尽可能多的输入空间。
* **语言的实现通常是复杂的、不断变化的，而且充满了优化**。这就导致了粗糙代码中有很多隐藏错误的黑暗角落。

所有这些都意味着你需要做大量的测试。但是什么测试呢？我见过的项目主要集中在端到端的“语言测试”上。每个测试都是一段用该语言编写的程序，以及它预期产生的输出或错误。然后，你还需要一个测试运行器，将这些测试程序输入到你的语言实现中，并验证它是否按照预期执行。用语言本身编写测试有一些很好的优势：

* 测试不与任何特定的API或语言实现的内部结构相耦合。这样你可以重新组织或重写解释器或编译器的一部分，而不需要更新大量的测试。
* 你可以对该语言的多种实现使用相同的测试。
* 测试通常是简洁的，易于阅读和维护，因为它们只是语言写就的简单脚本。

不过，这并不全是好事：

* 端到端测试可以帮助你确定是否存在错误，但不能确认错误在哪里。在语言实现中找出错误代码的位置可能更加困难，因为测试只能告诉你没有出现正确的输出。
* 要编写一个有效的程序来测试实现中一些不太明显的角落，可能是一件比较麻烦的事。对于高度优化的编译器来说尤其如此，你可能需要编写复杂的代码，以确保最终能够到达正确的优化路径，以测试其中可能隐藏的错误。
* 启动解释器、解析、编译和运行每个测试脚本的开销可能很高。对于一个大的测试套件来说，（如果你确实需要的话，请记住）这可能意味着需要花费很多时间来等待测试的完成。

我可以继续说下去，但是我不希望这变成一场说教。此外，我并不想假装自己是语言测试专家。我只是想让你在内心深处明白，测试你的语言是多么重要。我是认真的。测试你的语言。你会为此感谢我的。

</div>
