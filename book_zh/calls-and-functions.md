> Any problem in computer science can be solved with another level of
> indirection. Except for the problem of too many layers of indirection.
>
> 计算机科学中的任何问题都可以通过引入一个中间层来解决。除了中间层太多这个问题。
>
> <cite>David Wheeler</cite>

This chapter is a beast. I try to break features into bite-sized pieces, but
sometimes you gotta swallow the whole <span name="eat">meal</span>. Our next
task is functions. We could start with only function declarations, but that's
not very useful when you can't call them. We could do calls, but there's nothing
to call. And all of the runtime support needed in the VM to support both of
those isn't very rewarding if it isn't hooked up to anything you can see. So
we're going to do it all. It's a lot, but we'll feel good when we're done.
这一章是一头猛兽。我试图把功能分解成小块，但有时候你不得不吞下整顿饭。我们的下一个任务是函数。我们可以只从函数声明开始，但是如果你不能调用它们，那就没什么用了。我们可以实现调用，但是也没什么可调用的。而且，为了实现这两个功能所需的所有运行时支持，如果不能与你能直观看到的东西相挂钩，就不是很有价值。所以我们都要做。虽然内容很多，但等我们完成时，我们会感觉很好。

<aside name="eat">

Eating -- consumption -- is a weird metaphor for a creative act. But most of the
biological processes that produce "output" are a little less, ahem, decorous.
吃 -- 消费 -- 是对创造性行为的一个奇怪比喻。但大多数产生 "产出" 的生物过程就没那么 "体面" 了。

</aside>

## Function Objects  函数对象

The most interesting structural change in the VM is around the stack. We already
*have* a stack for local variables and temporaries, so we're partway there. But
we have no notion of a *call* stack. Before we can make much progress, we'll
have to fix that. But first, let's write some code. I always feel better once I
start moving. We can't do much without having some kind of representation for
functions, so we'll start there. From the VM's perspective, what is a function?
虚拟机中最有趣的结构变化是围绕堆栈进行的。我们已经有了用于局部变量和临时变量的栈，所以我们已经完成了一半。但是我们还没有调用堆栈的概念。在我们取得更大进展之前，必须先解决这个问题。但首先，让我们编写一些代码。一旦开始行动，我就感觉好多了。如果没有函数的某种表示形式，我们就做不了太多事，所以我们先从这里开始。从虚拟机的角度来看，什么是函数？

A function has a body that can be executed, so that means some bytecode. We
could compile the entire program and all of its function declarations into one
big monolithic Chunk. Each function would have a pointer to the first
instruction of its code inside the Chunk.
函数有一个可以被执行的主体，也就是一些字节码。我们可以把整个程序和所有的函数声明编译成一个大的字节码块。每个函数都有一个指针指向其在字节码块中的第一条指令。

This is roughly how compilation to native code works where you end up with one
solid blob of machine code. But for our bytecode VM, we can do something a
little higher level. I think a cleaner model is to give each function its own
Chunk. We'll want some other metadata too, so let's go ahead and stuff it all in
a struct now.
这大概就是编译为本地代码的工作原理，你最终得到的是一大堆机器码。但是对于我们的字节码虚拟机，我们可以做一些更高层次的事情。我认为一个更简洁的模型是给每个函数它自己的字节码块。我们还需要一些其它的元数据，所以我们现在来把它们塞进一个结构体中。

^code obj-function (2 before, 2 after)

Functions are first class in Lox, so they need to be actual Lox objects. Thus
ObjFunction has the same Obj header that all object types share. The `arity`
field stores the number of parameters the function expects. Then, in addition to
the chunk, we store the function's <span name="name">name</span>. That will be
handy for reporting readable runtime errors.
函数是Lox中的一等公民，所以它们需要作为实际的Lox对象。因此，ObjFunction具有所有对象类型共享的Obj头。`arity`字段存储了函数所需要的参数数量。然后，除了字节码块，我们还需要存储函数名称。这有助于报告可读的运行时错误。

<aside name="name">

Humans don't seem to find numeric bytecode offsets particularly illuminating in
crash dumps.
人们似乎并不觉得数值型的字节码偏移量在崩溃转储中特别有意义。

</aside>

This is the first time the "object" module has needed to reference Chunk, so we
get an include.
这是“object”模块第一次需要引用Chunk，所以我们需要引入一下。

^code object-include-chunk (1 before, 1 after)

Like we did with strings, we define some accessories to make Lox functions
easier to work with in C. Sort of a poor man's object orientation. First, we'll
declare a C function to create a new Lox function.
就像我们处理字符串一样，我们定义一些辅助程序，使Lox函数更容易在C语言中使用。有点像穷人版的面向对象。首先，我们会声明一个C函数来创建新Lox函数。

^code new-function-h (3 before, 1 after)

The implementation is over here:
实现如下：

^code new-function

We use our friend `ALLOCATE_OBJ()` to allocate memory and initialize the
object's header so that the VM knows what type of object it is. Instead of
passing in arguments to initialize the function like we did with ObjString, we
set the function up in a sort of blank state -- zero arity, no name, and no
code. That will get filled in later after the function is created.
我们使用好朋友`ALLOCATE_OBJ()`来分配内存并初始化对象的头信息，以便虚拟机知道它是什么类型的对象。我们没有像对ObjString那样传入参数来初始化函数，而是将函数设置为一种空白状态 -- 零参数、无名称、无代码。这里会在稍后创建函数后被填入数据。

Since we have a new kind of object, we need a new object type in the enum.
因为有了一个新类型的对象，我们需要在枚举中添加一个新的对象类型。

^code obj-type-function (1 before, 2 after)

When we're done with a function object, we must return the bits it borrowed back
to the operating system.
当我们使用完一个函数对象后，必须将它借用的比特位返还给操作系统。

^code free-function (1 before, 1 after)

This switch case is <span name="free-name">responsible</span> for freeing the
ObjFunction itself as well as any other memory it owns. Functions own their
chunk, so we call Chunk's destructor-like function.
这个switch语句负责释放ObjFunction本身以及它所占用的其它内存。函数拥有自己的字节码块，所以我们调用Chunk中类似析构器的函数。

<aside name="free-name">

We don't need to explicitly free the function's name because it's an ObjString.
That means we can let the garbage collector manage its lifetime for us. Or, at
least, we'll be able to once we [implement a garbage collector][gc].
我们不需要显式地释放函数名称，因为它是一个ObjString。这意味着我们可以让垃圾回收为我们管理它的生命周期。或者说，至少在实现[垃圾回收][gc]之后，我们就可以这样做了。

[gc]: garbage-collection.html

</aside>

Lox lets you print any object, and functions are first-class objects, so we
need to handle them too.
Lox允许你打印任何对象，而函数是一等对象，所以我们也需要处理它们。

^code print-function (1 before, 1 after)

This calls out to:
这就引出了：

^code print-function-helper

Since a function knows its name, it may as well say it.
既然函数知道它的名称，那就应该说出来。

Finally, we have a couple of macros for converting values to functions. First,
make sure your value actually *is* a function.
最后，我们有几个宏用于将值转换为函数。首先，确保你的值实际上 *是* 一个函数。

^code is-function (2 before, 1 after)

Assuming that evaluates to true, you can then safely cast the Value to an
ObjFunction pointer using this:
假设计算结果为真，你就可以使用这个方法将Value安全地转换为一个ObjFunction指针：

^code as-function (2 before, 1 after)

With that, our object model knows how to represent functions. I'm feeling warmed
up now. You ready for something a little harder?
这样，我们的对象模型就知道如何表示函数了。我现在感觉已经热身了。你准备好来点更难的东西了吗？

## 编译为函数对象

Right now, our compiler assumes it is always compiling to one single chunk. With
each function's code living in separate chunks, that gets more complex. When the
compiler reaches a function declaration, it needs to emit code into the
function's chunk when compiling its body. At the end of the function body, the
compiler needs to return to the previous chunk it was working with.
现在，我们的编译器假定它总会编译到单个字节码块中。由于每个函数的代码都位于不同的字节码块，这就变得更加复杂了。当编译器碰到函数声明时，需要在编译函数主体时将代码写入函数自己的字节码块中。在函数主体的结尾，编译器需要返回到它之前正处理的前一个字节码块。

That's fine for code inside function bodies, but what about code that isn't? The
"top level" of a Lox program is also imperative code and we need a chunk to
compile that into. We can simplify the compiler and VM by placing that top-level
code inside an automatically defined function too. That way, the compiler is
always within some kind of function body, and the VM always runs code by
invoking a function. It's as if the entire program is <span
name="wrap">wrapped</span> inside an implicit `main()` function.
这对于函数主体内的代码来说很好，但是对于不在其中的代码呢？Lox程序的“顶层”也是命令式代码，而且我们需要一个字节码块来编译它。我们也可以将顶层代码放入一个自动定义的函数中，从而简化编译器和虚拟机的工作。这样一来，编译器总是在某种函数主体内，而虚拟机总是通过调用函数来运行代码。这就像整个程序被包裹在一个隐式的`main()`函数中一样。

<aside name="wrap">

One semantic corner where that analogy breaks down is global variables. They
have special scoping rules different from local variables, so in that way, the
top level of a script isn't like a function body.
这种类比在语义上有个行不通的地方就是全局变量。它们具有与局部变量不同的特殊作用域规则，因此从这个角度来说，脚本的顶层并不像一个函数体。

</aside>

Before we get to user-defined functions, then, let's do the reorganization to
support that implicit top-level function. It starts with the Compiler struct.
Instead of pointing directly to a Chunk that the compiler writes to, it instead
has a reference to the function object being built.
在我们讨论用户定义的函数之前，让我们先重新组织一下，支持隐式的顶层函数。这要从Compiler结构体开始。它不再直接指向编译器写入的Chunk，而是指向正在构建的函数对象的引用。

^code function-fields (1 before, 1 after)

We also have a little FunctionType enum. This lets the compiler tell when it's
compiling top-level code versus the body of a function. Most of the compiler
doesn't care about this -- that's why it's a useful abstraction -- but in one or
two places the distinction is meaningful. We'll get to one later.
我们也有一个小小的FunctionType枚举。这让编译器可以区分它在编译顶层代码还是函数主体。大多数编译器并不关心这一点 -- 这就是为什么它是一个有用的抽象 -- 但是在一两个地方，这种区分是有意义的。我们稍后会讲到其中一个。

^code function-type-enum

Every place in the compiler that was writing to the Chunk now needs to go
through that `function` pointer. Fortunately, many <span
name="current">chapters</span> ago, we encapsulated access to the chunk in the
`currentChunk()` function. We only need to fix that and the rest of the compiler
is happy.
编译器中所有写入Chunk的地方，现在都需要通过`function`指针。幸运的是，在很多章节之前，我们在`currentChunk()`函数中封装了对字节码块的访问。我们只需要修改它，编译器的其它部分就可以了。

<aside name="current">

It's almost like I had a crystal ball that could see into the future and knew
we'd need to change the code later. But, really, it's because I wrote all the
code for the book before any of the text.
这就像我有一个可以看到未来的水晶球，知道我们以后需要修改代码。但是，实际上，这是因为我在写文字之前已经写了本书中的所有代码。

</aside>

^code current-chunk (1 before, 2 after)

The current chunk is always the chunk owned by the function we're in the middle
of compiling. Next, we need to actually create that function. Previously, the VM
passed a Chunk to the compiler which filled it with code. Instead, the compiler
will create and return a function that contains the compiled top-level code --
which is all we support right now -- of the user's program.
当前的字节码块一定是我们正在编译的函数所拥有的块。接下来，我们需要实际创建该函数。之前，虚拟机将一个Chunk传递给编译器，编译器会将代码填入其中。现在取而代之的是，编译器创建并返回一个包含已编译顶层代码的函数 -- 这就是我们目前所支持的。

### 编译时创建函数

We start threading this through in `compile()`, which is the main entry point
into the compiler.
我们在`compile()`中开始执行此操作，该方法是进入编译器的主要入口点。

^code call-init-compiler (1 before, 2 after)

There are a bunch of changes in how the compiler is initialized. First, we
initialize the new Compiler fields.
在如何初始化编译器方面有很多改变。首先，我们初始化新的Compiler字段。

^code init-compiler (1 after)

Then we allocate a new function object to compile into.
然后我们分配一个新的函数对象用于编译。

^code init-function (1 before, 1 after)

<span name="null"></span>

<aside name="null">

I know, it looks dumb to null the `function` field only to immediately assign it
a value a few lines later. More garbage collection-related paranoia.
我知道，让`function`字段为空，但在几行之后又立即为其赋值，这看起来很蠢。更像是与垃圾回收有关的偏执。

</aside>

Creating an ObjFunction in the compiler might seem a little strange. A function
object is the *runtime* representation of a function, but here we are creating
it at compile time. The way to think of it is that a function is similar to a
string or number literal. It forms a bridge between the compile time and runtime
worlds. When we get to function *declarations*, those really *are* literals
-- they are a notation that produces values of a built-in type. So the <span
name="closure">compiler</span> creates function objects during compilation.
Then, at runtime, they are simply invoked.
在编译器中创建ObjFunction可能看起来有点奇怪。函数对象是一个函数的运行时表示，但这里我们是在编译时创建它。我们可以这样想：函数类似于一个字符串或数字字面量。它在编译时和运行时之间形成了一座桥梁。当我们碰到函数 *声明* 时，它们确实 *是* 字面量 -- 它们是一种生成内置类型值的符号。因此，编译器在编译期间创建函数对象。然后，在运行时，它们被简单地调用。

<aside name="closure">

We can create functions at compile time because they contain only data available
at compile time. The function's code, name, and arity are all fixed. When we add
closures in the [next chapter][closures], which capture variables at runtime,
the story gets more complex.
我们可以在编译时创建函数，是因为它们只包含编译时可用的数据。函数的代码、名称和元都是固定的。等我们在[下一章][closures]中添加闭包时（在运行时捕获变量），情况就变得更加复杂了。

[closures]: closures.html

</aside>

Here is another strange piece of code:
下面是另一段奇怪的代码：

^code init-function-slot (1 before, 1 after)

Remember that the compiler's `locals` array keeps track of which stack slots are
associated with which local variables or temporaries. From now on, the compiler
implicitly claims stack slot zero for the VM's own internal use. We give it an
empty name so that the user can't write an identifier that refers to it. I'll
explain what this is about when it becomes useful.
请记住，编译器的`locals`数组记录了哪些栈槽与哪些局部变量或临时变量相关联。从现在开始，编译器隐式地要求栈槽0供虚拟机自己内部使用。我们给它一个空的名称，这样用户就不能向一个指向它的标识符写值。等它起作用时，我会解释这是怎么回事。

That's the initialization side. We also need a couple of changes on the other
end when we finish compiling some code.
这就是初始化这一边的工作。当我们完成一些代码的编译时，还需要在另一边做一些改变。

^code end-compiler (1 after)

Previously, when `interpret()` called into the compiler, it passed in a Chunk to
be written to. Now that the compiler creates the function object itself, we
return that function. We grab it from the current compiler here:
以前，当调用`interpret()`方法进入编译器时，会传入一个要写入的Chunk。现在，编译器自己创建了函数对象，我们返回该函数。我们从当前编译器中这样获取它：

^code end-function (1 before, 1 after)

And then return it to `compile()` like so:
然后这样将其返回给`compile()`：

^code return-function (1 before, 1 after)

Now is a good time to make another tweak in this function. Earlier, we added
some diagnostic code to have the VM dump the disassembled bytecode so we could
debug the compiler. We should fix that to keep working now that the generated
chunk is wrapped in a function.
现在是对该函数进行另一个调整的好时机。之前，我们添加了一些诊断性代码，让虚拟机转储反汇编的字节码，以便我们可以调试编译器。现在生成的字节码块包含在一个函数中，我们要修复这些代码，使其继续工作。
*compiler.c*，在 *endCompiler* ()方法中替换1行：

^code disassemble-end (2 before, 2 after)

Notice the check in here to see if the function's name is `NULL`? User-defined
functions have names, but the implicit function we create for the top-level code
does not, and we need to handle that gracefully even in our own diagnostic code.
Speaking of which:
注意到这里检查了函数名称是否为`NULL`吗？用户定义的函数有名称，但我们为顶层代码创建的隐式函数却没有，即使在我们自己的诊断代码中，我们也需要优雅地处理这个问题。说到这一点：

^code print-script (1 before, 1 after)

There's no way for a *user* to get a reference to the top-level function and try
to print it, but our `DEBUG_TRACE_EXECUTION` <span
name="debug">diagnostic</span> code that prints the entire stack can and does.
用户没有办法获取对顶层函数的引用并试图打印它，但我们用来打印整个堆栈的诊断代码`DEBUG_TRACE_EXECUTION`可以而且确实这样做了。

<aside name="debug">

It is no fun if the diagnostic code we use to find bugs itself causes the VM to
segfault!
如果我们用来寻找bug的诊断代码本身导致虚拟机发生故障，那就不好玩了。

</aside>

Bumping up a level to `compile()`, we adjust its signature.
为了给`compile()`提升一级，我们调整其签名。
*compiler.h*，在函数*compile*()中替换1行：

^code compile-h (2 before, 2 after)

Instead of taking a chunk, now it returns a function. Over in the
implementation:
现在它不再接受字节码块，而是返回一个函数。在实现中：

^code compile-signature (1 after)

Finally we get to some actual code. We change the very end of the function to
this:
最后，我们得到了一些实际的代码。我们把方法的最后部分改成这样：

^code call-end-compiler (4 before, 1 after)

We get the function object from the compiler. If there were no compile errors,
we return it. Otherwise, we signal an error by returning `NULL`. This way, the
VM doesn't try to execute a function that may contain invalid bytecode.
我们从编译器获取函数对象。如果没有编译错误，就返回它。否则，我们通过返回`NULL`表示错误。这样，虚拟机就不会试图执行可能包含无效字节码的函数。

Eventually, we will update `interpret()` to handle the new declaration of
`compile()`, but first we have some other changes to make.
最终，我们会更新`interpret()`来处理`compile()`的新声明，但首先我们要做一些其它的改变。

## 调用帧

It's time for a big conceptual leap. Before we can implement function
declarations and calls, we need to get the VM ready to handle them. There are
two main problems we need to worry about:
是时候进行一次重大的概念性飞跃了。在我们实现函数声明和调用之前，需要让虚拟机准备好处理它们。我们需要考虑两个主要问题：

### 分配局部变量

The compiler allocates stack slots for local variables. How should that work
when the set of local variables in a program is distributed across multiple
functions?
编译器为局部变量分配了堆栈槽。当程序中的局部变量集分布在多个函数中时，应该如何操作？

One option would be to keep them totally separate. Each function would get its
own dedicated set of slots in the VM stack that it would own <span
name="static">forever</span>, even when the function isn't being called. Each
local variable in the entire program would have a bit of memory in the VM that
it keeps to itself.
一种选择是将它们完全分开。每个函数在虚拟机堆栈中都有自己的一组专用槽，即使在函数没有被调用的情况下，它也会永远拥有这些槽。整个程序中的每个局部变量在虚拟机中都有自己保留的一小块内存。

<aside name="static">

It's basically what you'd get if you declared every local variable in a C
program using `static`.
这基本就是你在C语言中使用`static`声明每个局部变量的结果。

</aside>

Believe it or not, early programming language implementations worked this way.
The first Fortran compilers statically allocated memory for each variable. The
obvious problem is that it's really inefficient. Most functions are not in the
middle of being called at any point in time, so sitting on unused memory for
them is wasteful.
信不信由你，早期的编程语言实现就是这样工作的。第一个Fortran编译器为每个变量静态地分配了内存。最显而易见的问题是效率很低。大多数函数不会随时都在被调用，所以一直占用未使用的内存是浪费的。

The more fundamental problem, though, is recursion. With recursion, you can be
"in" multiple calls to the same function at the same time. Each needs its <span
name="fortran">own</span> memory for its local variables. In jlox, we solved
this by dynamically allocating memory for an environment each time a function
was called or a block entered. In clox, we don't want that kind of performance
cost on every function call.
不过，更根本的问题是递归。通过递归，你可以在同一时刻处于对同一个函数的多次调用“中”。每个函数的局部变量都需要自己的内存。在jlox中，我们通过在每次调用函数或进入代码块时为环境动态分配内存来解决这个问题。在clox中，我们不希望在每次调用时都付出这样的性能代价。

<aside name="fortran">

Fortran avoided this problem by disallowing recursion entirely. Recursion was
considered an advanced, esoteric feature at the time.
Fortran完全不允许递归，从而避免了这个问题。递归在当时被认为是一种高级、深奥的特性。

</aside>

Instead, our solution lies somewhere between Fortran's static allocation and
jlox's dynamic approach. The value stack in the VM works on the observation that
local variables and temporaries behave in a last-in first-out fashion.
Fortunately for us, that's still true even when you add function calls into the
mix. Here's an example:
相反，我们的解决方案介于Fortran的静态分配和jlox的动态方法之间。虚拟机中的值栈的工作原理是：局部变量和临时变量的后进先出的行为模式。幸运的是，即使你把函数调用考虑在内，这仍然是正确的。这里有一个例子：

```lox
fun first() {
  var a = 1;
  second();
  var b = 2;
}

fun second() {
  var c = 3;
  var d = 4;
}

first();
```

Step through the program and look at which variables are in memory at each point
in time:
逐步执行程序，看看在每个时间点上内存中有哪些变量：

<img src="image/calls-and-functions/calls.png" alt="Tracing through the execution of the previous program, showing the stack of variables at each step." />

As execution flows through the two calls, every local variable obeys the
principle that any variable declared after it will be discarded before the first
variable needs to be. This is true even across calls. We know we'll be done with
`c` and `d` before we are done with `a`. It seems we should be able to allocate
local variables on the VM's value stack.
在这两次调用的执行过程中，每个局部变量都遵循这样的原则：当某个变量需要被丢弃时，在它之后声明的任何变量都会被丢弃。甚至在不同的调用中也是如此。我们知道，在我们用完`a`之前，已经用完了`c`和`d`。看起来我们应该能够在虚拟机的值栈上分配局部变量。

Ideally, we still determine *where* on the stack each variable will go at
compile time. That keeps the bytecode instructions for working with variables
simple and fast. In the above example, we could <span
name="imagine">imagine</span> doing so in a straightforward way, but that
doesn't always work out. Consider:
理想情况下，我们仍然在编译时确定每个变量在栈中的位置。这使得处理变量的字节码指令变得简单而快速。在上面的例子中，我们可以想象以一种直接的方式这样做，但这并不总是可行的。考虑一下：

<aside name="imagine">

I say "imagine" because the compiler can't actually figure this out. Because
functions are first class in Lox, we can't determine which functions call which
others at compile time.
我说“想象”是因为编译器实际上无法弄清这一点。因为函数在Lox中是一等公民，我们无法在编译时确定哪些函数调用了哪些函数。

</aside>

```lox
fun first() {
  var a = 1;
  second();
  var b = 2;
  second();
}

fun second() {
  var c = 3;
  var d = 4;
}

first();
```

In the first call to `second()`, `c` and `d` would go into slots 1 and 2. But in
the second call, we need to have made room for `b`, so `c` and `d` need to be in
slots 2 and 3. Thus the compiler can't pin down an exact slot for each local
variable across function calls. But *within* a given function, the *relative*
locations of each local variable are fixed. Variable `d` is always in the slot
right after `c`. This is the key insight.
在对`second()`的第一次调用中，`c`和`d`将进入槽1和2。但在第二次调用中，我们需要为`b`腾出空间，所以`c`和`d`需要放在槽2和3里。因此，编译器不能在不同的函数调用中为每个局部变量指定一个确切的槽。但是在特定的函数中，每个局部变量的相对位置是固定的。变量`d`总是在变量`c`后面的槽里。这是关键的见解。

When a function is called, we don't know where the top of the stack will be
because it can be called from different contexts. But, wherever that top happens
to be, we do know where all of the function's local variables will be relative
to that starting point. So, like many problems, we solve our allocation problem
with a level of indirection.
当函数被调用时，我们不知道栈顶在什么位置，因为它可以从不同的上下文中被调用。但是，无论栈顶在哪里，我们都知道该函数的所有局部变量相对于起始点的位置。因此，像很多问题一样，我们使用一个中间层来解决分配问题。

At the beginning of each function call, the VM records the location of the first
slot where that function's own locals begin. The instructions for working with
local variables access them by a slot index relative to that, instead of
relative to the bottom of the stack like they do today. At compile time, we
calculate those relative slots. At runtime, we convert that relative slot to an
absolute stack index by adding the function call's starting slot.
在每次函数调用开始时，虚拟机都会记录函数自身的局部变量开始的第一个槽的位置。使用局部变量的指令通过相对于该槽的索引来访问它们，而不是像现在这样使用相对于栈底的索引。在编译时，我们可以计算出这些相对槽位。在运行时，加上函数调用时的起始槽位，就能将相对位置转换为栈中的绝对索引。

It's as if the function gets a "window" or "frame" within the larger stack where
it can store its locals. The position of the **call frame** is determined at
runtime, but within and relative to that region, we know where to find things.
这就好像是函数在更大的堆栈中得到了一个“窗口”或“帧”，它可以在其中存储局部变量。**调用帧**的位置是在运行时确定的，但在该区域内部及其相对位置上，我们知道在哪里可以找到目标。

<img src="image/calls-and-functions/window.png" alt="The stack at the two points when second() is called, with a window hovering over each one showing the pair of stack slots used by the function." />

The historical name for this recorded location where the function's locals start
is a **frame pointer** because it points to the beginning of the function's call
frame. Sometimes you hear **base pointer**, because it points to the base stack
slot on top of which all of the function's variables live.
这个记录了函数局部变量开始的位置的历史名称是**帧指针**，因为它指向函数调用帧的开始处。有时你会听到**基指针**，因为它指向一个基本栈槽，函数的所有变量都在其之上。

That's the first piece of data we need to track. Every time we call a function,
the VM determines the first stack slot where that function's variables begin.
这是我们需要跟踪的第一块数据。每次我们调用函数时，虚拟机都会确定该函数变量开始的第一个栈槽。

### 返回地址

Right now, the VM works its way through the instruction stream by incrementing
the `ip` field. The only interesting behavior is around control flow
instructions which offset the `ip` by larger amounts. *Calling* a function is
pretty straightforward -- simply set `ip` to point to the first instruction in
that function's chunk. But what about when the function is done?
现在，虚拟机通过递增`ip`字段的方式在指令流中工作。唯一有趣的行为是关于控制流指令的，这些指令会以较大的数值对`ip`进行偏移。调用函数非常直接 -- 将`ip`简单地设置为指向函数块中的第一条指令。但是等函数完成后怎么办？

The VM needs to <span name="return">return</span> back to the chunk where the
function was called from and resume execution at the instruction immediately
after the call. Thus, for each function call, we need to track where we jump
back to when the call completes. This is called a **return address** because
it's the address of the instruction that the VM returns to after the call.
虚拟机需要返回到调用函数的字节码块，并在调用之后立即恢复执行指令。因此，对于每个函数调用，在调用完成后，需要记录调用完成后需要跳回什么地方。这被称为**返回地址**，因为它是虚拟机在调用后返回的指令的地址。

Again, thanks to recursion, there may be multiple return addresses for a single
function, so this is a property of each *invocation* and not the function
itself.
同样，由于递归的存在，一个函数可能会对应多个返回地址，所以这是每个调用的属性，而不是函数本身的属性。

<aside name="return">

The authors of early Fortran compilers had a clever trick for implementing
return addresses. Since they *didn't* support recursion, any given function
needed only a single return address at any point in time. So when a function was
called at runtime, the program would *modify its own code* to change a jump
instruction at the end of the function to jump back to its caller. Sometimes the
line between genius and madness is hair thin.
早期Fortran编译器的作者在实现返回地址方面有一个巧妙的技巧。由于它们 *不* 支持递归，任何给定的函数在任何时间点都只需要一个返回地址。因此，当函数在运行时被调用时，程序会 *修改自己的代码* ，更改函数末尾的跳转指针，以跳回调用方。有时候，天才和疯子之间只有一线之隔。

</aside>

### 调用栈

So for each live function invocation -- each call that hasn't returned yet -- we
need to track where on the stack that function's locals begin, and where the
caller should resume. We'll put this, along with some other stuff, in a new
struct.
因此，对于每个活动的函数执行（每个尚未返回的调用），我们需要跟踪该函数的局部变量在堆栈中的何处开始，以及调用方应该在何处恢复。我们会将这些信息以及其它一些数据放在新的结构体中。

^code call-frame (1 before, 2 after)

A CallFrame represents a single ongoing function call. The `slots` field points
into the VM's value stack at the first slot that this function can use. I gave
it a plural name because -- thanks to C's weird "pointers are sort of arrays"
thing -- we'll treat it like an array.
一个CallFrame代表一个正在进行的函数调用。`slots`字段指向虚拟机的值栈中该函数可以使用的第一个槽。我给它取了一个复数的名字是因为我们会把它当作一个数组来对待（感谢C语言中“指针是一种数组”这个奇怪的概念）。

The implementation of return addresses is a little different from what I
described above. Instead of storing the return address in the callee's frame,
the caller stores its own `ip`. When we return from a function, the VM will jump
to the `ip` of the caller's CallFrame and resume from there.
返回地址的实现与我上面的描述有所不同。调用者不是将返回地址存储在被调用者的帧中，而是将自己的`ip`存储起来。等到从函数中返回时，虚拟机会跳转到调用方的CallFrame的`ip`，并从那里继续执行。

I also stuffed a pointer to the function being called in here. We'll use that to
look up constants and for a few other things.
我还在这里塞了一个指向被调用函数的指针。我们会用它来查询常量和其它一些事情。

Each time a function is called, we create one of these structs. We could <span
name="heap">dynamically</span> allocate them on the heap, but that's slow.
Function calls are a core operation, so they need to be as fast as possible.
Fortunately, we can make the same observation we made for variables: function
calls have stack semantics. If `first()` calls `second()`, the call to
`second()` will complete before `first()` does.
每次函数被调用时，我们会创建一个这样的结构体。我们可以在堆上动态地分配它们，但那样会很慢。函数调用是核心操作，所以它们需要尽可能快。幸运的是，我们意识到它和变量很相似：函数调用具有堆栈语义。如果`first()`调用`second()`，对`second()`的调用将在`first()`之前完成。

<aside name="heap">

Many Lisp implementations dynamically allocate stack frames because it
simplifies implementing [continuations][cont]. If your language supports
continuations, then function calls do *not* always have stack semantics.
许多Lisp实现都是动态地分配堆栈帧的，因为它简化实现了[续延][cont]。如果你的语言支持续延，那么函数调用并不一定具有堆栈语义。

[cont]: https://en.wikipedia.org/wiki/Continuation

</aside>

So over in the VM, we create an array of these CallFrame structs up front and
treat it as a stack, like we do with the value array.
因此在虚拟机中，我们预先创建一个CallFrame结构体的数组，并将其作为堆栈对待，就像我们对值数组所做的那样。

^code frame-array (1 before, 1 after)

This array replaces the `chunk` and `ip` fields we used to have directly in the
VM. Now each CallFrame has its own `ip` and its own pointer to the ObjFunction
that it's executing. From there, we can get to the function's chunk.
这个数组取代了我们过去在VM中直接使用的`chunk`和`ip`字段。现在，每个CallFrame都有自己的`ip`和指向它正在执行的ObjFunction的指针。通过它们，我们可以得到函数的字节码块。

The new `frameCount` field in the VM stores the current height of the CallFrame
stack -- the number of ongoing function calls. To keep clox simple, the array's
capacity is fixed. This means, as in many language implementations, there is a
maximum call depth we can handle. For clox, it's defined here:
VM中新的`frameCount`字段存储了CallFrame栈的当前高度 -- 正在进行的函数调用的数量。为了使clox简单，数组的容量是固定的。这意味着，和许多语言的实现一样，存在一个我们可以处理的最大调用深度。对于clox，在这里定义它：

^code frame-max (2 before, 2 after)

We also redefine the value stack's <span name="plenty">size</span> in terms of
that to make sure we have plenty of stack slots even in very deep call trees.
When the VM starts up, the CallFrame stack is empty.
我们还以此重新定义了值栈的大小，以确保即使在很深的调用树中我们也有足够的栈槽。当虚拟机启动时，CallFrame栈是空的。

<aside name="plenty">

It is still possible to overflow the stack if enough function calls use enough
temporaries in addition to locals. A robust implementation would guard against
this, but I'm trying to keep things simple.
如果除了局部变量之外，还有足够多的临时变量，仍然有可能溢出堆栈。一个健壮的实现可以防止这种情况，但我想尽量保持简单。

</aside>

^code reset-frame-count (1 before, 1 after)

The "vm.h" header needs access to ObjFunction, so we add an include.
“vm.h”头文件需要访问ObjFunction，所以我们加一个引入。

^code vm-include-object (2 before, 1 after)

Now we're ready to move over to the VM's implementation file. We've got some
grunt work ahead of us. We've moved `ip` out of the VM struct and into
CallFrame. We need to fix every line of code in the VM that touches `ip` to
handle that. Also, the instructions that access local variables by stack slot
need to be updated to do so relative to the current CallFrame's `slots` field.
现在我们准备转移到VM的实现文件中。我们还有很多艰巨的工作要做。我们已经将`ip`从VM结构体移到了CallFrame中。我们需要修改VM中使用了`ip`的每一行代码来解决这个问题。此外，需要更新根据栈槽访问局部变量的指令，使其相对于当前CallFrame的`slots`字段进行访问。

We'll start at the top and plow through it.
我们从最上面开始，彻底解决这个问题。

^code run (1 before, 1 after)

First, we store the current topmost CallFrame in a <span
name="local">local</span> variable inside the main bytecode execution function.
Then we replace the bytecode access macros with versions that access `ip`
through that variable.
首先，我们将当前最顶部的CallFrame存储在主字节码执行函数中的一个局部变量中。然后我们将字节码访问宏替换为通过该变量访问`ip`的版本。

<aside name="local">

We could access the current frame by going through the CallFrame array every
time, but that's verbose. More importantly, storing the frame in a local
variable encourages the C compiler to keep that pointer in a register. That
speeds up access to the frame's `ip`. There's no *guarantee* that the compiler
will do this, but there's a good chance it will.
我们可以通过每次查看CallFrame数组来访问当前帧，但这太繁琐了。更重要的是，将帧存储在一个局部变量中，可以促使C编译器将该指针保存在一个寄存器中。这样就能加快对帧中`ip`的访问。我们不能保证编译器会这样做，但很有可能会这样做。

</aside>

Now onto each instruction that needs a little tender loving care.
现在我们来看看每条需要温柔呵护的指令。

^code push-local (2 before, 1 after)

Previously, `OP_GET_LOCAL` read the given local slot directly from the VM's
stack array, which meant it indexed the slot starting from the bottom of the
stack. Now, it accesses the current frame's `slots` array, which means it
accesses the given numbered slot relative to the beginning of that frame.
以前，`OP_GET_LOCAL`直接从虚拟机的栈数组中读取给定的局部变量槽，这意味着它是从栈底开始对槽进行索引。现在，它访问的是当前帧的`slots`数组，这意味着它是访问相对于该帧起始位置的给定编号的槽。

Setting a local variable works the same way.
设置局部变量的方法也是如此。

^code set-local (2 before, 1 after)

The jump instructions used to modify the VM's `ip` field. Now, they do the same
for the current frame's `ip`.
跳转指令之前是修改VM的`ip`字段。现在，它会对当前帧的`ip`做相同的操作。

^code jump (2 before, 1 after)

Same with the conditional jump:
条件跳转也是如此：

^code jump-if-false (2 before, 1 after)

And our backward-jumping loop instruction:
还有向后跳转的循环指令：

^code loop (2 before, 1 after)

We have some diagnostic code that prints each instruction as it executes to help
us debug our VM. That needs to work with the new structure too.
我们还有一些诊断代码，可以在每条指令执行时将其打印出来，帮助我们调试虚拟机。这也需要能处理新的结构体。

^code trace-execution (1 before, 1 after)

Instead of passing in the VM's `chunk` and `ip` fields, now we read from the
current CallFrame.
现在我们从当前的CallFrame中读取数据，而不是传入VM的`chunk` 和`ip` 字段。

You know, that wasn't too bad, actually. Most instructions just use the macros
so didn't need to be touched. Next, we jump up a level to the code that calls
`run()`.
其实，这不算太糟。大多数指令只是使用了宏，所以不需要修改。接下来，我们向上跳到调用`run()`的代码。

^code interpret-stub (1 before, 2 after)

We finally get to wire up our earlier compiler changes to the back-end changes
we just made. First, we pass the source code to the compiler. It returns us a
new ObjFunction containing the compiled top-level code. If we get `NULL` back,
it means there was some compile-time error which the compiler has already
reported. In that case, we bail out since we can't run anything.
我们终于可以将之前的编译器修改与我们刚刚做的后端更改联系起来。首先，我们将源代码传递给编译器。它返回给我们一个新的ObjFunction，其中包含编译好的顶层代码。如果我们得到的是`NULL`，这意味着存在一些编译时错误，编译器已经报告过了。在这种情况下，我们就退出，因为我们没有可以运行的代码。

Otherwise, we store the function on the stack and prepare an initial CallFrame
to execute its code. Now you can see why the compiler sets aside stack slot zero
-- that stores the function being called. In the new CallFrame, we point to the
function, initialize its `ip` to point to the beginning of the function's
bytecode, and set up its stack window to start at the very bottom of the VM's
value stack.
否则，我们将函数存储在堆栈中，并准备一个初始CallFrame来执行其代码。现在你可以看到为什么编译器将栈槽0留出来 -- 其中存储着正在被调用的函数。在新的CallFrame中，我们指向该函数，将`ip`初始化为函数字节码的起始位置，并将堆栈窗口设置为从VM值栈的最底部开始。

This gets the interpreter ready to start executing code. After finishing, the VM
used to free the hardcoded chunk. Now that the ObjFunction owns that code, we
don't need to do that anymore, so the end of `interpret()` is simply this:
这样解释器就准备好开始执行代码了。完成后，虚拟机原本会释放硬编码的字节码块。现在ObjFunction持有那段代码，我们就不需要再这样做了，所以`interpret()`的结尾是这样的：

^code end-interpret (2 before, 1 after)

The last piece of code referring to the old VM fields is `runtimeError()`. We'll
revisit that later in the chapter, but for now let's change it to this:
最后一段引用旧的VM字段的代码是`runtimeError()`。我们会在本章后面重新讨论这个问题，但现在我们先将它改成这样：

^code runtime-error-temp (2 before, 1 after)

Instead of reading the chunk and `ip` directly from the VM, it pulls those from
the topmost CallFrame on the stack. That should get the function working again
and behaving as it did before.
它不是直接从VM中读取字节码块和`ip`，而是从栈顶的CallFrame中获取这些信息。这应该能让函数重新工作，并且表现像以前一样。

Assuming we did all of that correctly, we got clox back to a runnable
state. Fire it up and it does... exactly what it did before. We haven't added
any new features yet, so this is kind of a let down. But all of the
infrastructure is there and ready for us now. Let's take advantage of it.
假如我们都正确执行了所有这些操作，就可以让clox回到可运行的状态。启动它，它就会……像以前一样。我们还没有添加任何新功能，所以这有点让人失望。但是所有的基础设施都已经就绪了。让我们好好利用它。

## 函数声明

Before we can do call expressions, we need something to call, so we'll do
function declarations first. The <span name="fun">fun</span> starts with a
keyword.
在我们确实可以调用表达式之前，首先需要一些可以用来调用的东西，所以我们首先要处理函数声明。一切从关键字开始。

<aside name="fun">

Yes, I am going to make a dumb joke about the `fun` keyword every time it
comes up.
是的，每次提到这个有趣的关键词(`fun`)，我都会开一个愚蠢的玩笑。

译者注： 作者这里使用了一个小小的双关 `function` 的开头是 `fun`
</aside>

^code match-fun (1 before, 1 after)

That passes control to here:
它将控制权传递到这里：

^code fun-declaration

Functions are first-class values, and a function declaration simply creates and
stores one in a newly declared variable. So we parse the name just like any
other variable declaration. A function declaration at the top level will bind
the function to a global variable. Inside a block or other function, a function
declaration creates a local variable.
函数是一等公民，函数声明只是在新声明的变量中创建并存储一个函数。因此，我们像其它变量声明一样解析名称。顶层的函数声明会将函数绑定到一个全局变量。在代码块或其它函数内部，函数声明会创建一个局部变量。

In an earlier chapter, I explained how variables [get defined in two
stages][stage]. This ensures you can't access a variable's value inside the
variable's own initializer. That would be bad because the variable doesn't
*have* a value yet.

[stage]: local-variables.html#another-scope-edge-case

Functions don't suffer from this problem. It's safe for a function to refer to
its own name inside its body. You can't *call* the function and execute the body
until after it's fully defined, so you'll never see the variable in an
uninitialized state. Practically speaking, it's useful to allow this in order to
support recursive local functions.
函数不会遇到这个问题。函数在其主体内引用自己的名称是安全的。在函数被完全定义之后，你才能调用函数并执行函数体，所以你永远不会看到处于未初始化状态的变量。实际上，为了支持递归局部函数，允许这样做是很有用的。

To make that work, we mark the function declaration's variable "initialized" as
soon as we compile the name, before we compile the body. That way the name can
be referenced inside the body without generating an error.
为此，在我们编译函数名称时（编译函数主体之前），就将函数声明的变量标记为“已初始化”。这样就可以在主体中引用该名称，而不会产生错误。

We do need one check, though.
不过，我们确实需要做一个检查。

^code check-depth (1 before, 1 after)

Before, we called `markInitialized()` only when we already knew we were in a
local scope. Now, a top-level function declaration will also call this function.
When that happens, there is no local variable to mark initialized -- the
function is bound to a global variable.
以前，只有在已经知道当前处于局部作用域中时，我们才会调用`markInitialized()`。现在，顶层的函数声明也会调用这个函数。当这种情况发生时，没有局部变量需要标记为已初始化 -- 函数被绑定到了一个全局变量。

Next, we compile the function itself -- its parameter list and block body. For
that, we use a separate helper function. That helper generates code that
leaves the resulting function object on top of the stack. After that, we call
`defineVariable()` to store that function back into the variable we declared for
it.
接下来，我们编译函数本身 -- 它的参数列表和代码块主体。为此，我们使用一个单独的辅助函数。该函数生成的代码会将生成的函数对象留在栈顶。之后，我们调用`defineVariable()`，将该函数存储到我们为其声明的变量中。

I split out the code to compile the parameters and body because we'll reuse it
later for parsing method declarations inside classes. Let's build it
incrementally, starting with this:
我将编译参数和主体的代码分开，因为我们稍后会重用它来解析类中的方法声明。我们来逐步构建它，从这里开始：

^code compile-function

<aside name="no-end-scope">

This `beginScope()` doesn't have a corresponding `endScope()` call. Because we
end Compiler completely when we reach the end of the function body, there's no
need to close the lingering outermost scope.
这里的`beginScope()`并没有对应的`endScope()`调用。因为当达到函数体的末尾时，我们会完全结束整个Compiler，所以没必要关闭逗留的最外层作用域。

</aside>

For now, we won't worry about parameters. We parse an empty pair of parentheses
followed by the body. The body starts with a left curly brace, which we parse
here. Then we call our existing `block()` function, which knows how to compile
the rest of a block including the closing brace.
现在，我们不需要考虑参数。我们解析一对空括号，然后是主体。主体以左大括号开始，我们在这里会解析它。然后我们调用现有的`block()`函数，该函数知道如何编译代码块的其余部分，包括结尾的右大括号。

### 编译器栈

The interesting parts are the compiler stuff at the top and bottom. The Compiler
struct stores data like which slots are owned by which local variables, how many
blocks of nesting we're currently in, etc. All of that is specific to a single
function. But now the front end needs to handle compiling multiple functions
<span name="nested">nested</span> within each other.
有趣的部分是顶部和底部的编译器。Compiler结构体存储的数据包括哪些栈槽被哪些局部变量拥有，目前处于多少层的嵌套块中，等等。所有这些都是针对单个函数的。但是现在，前端需要处理编译相互嵌套的多个函数的编译。

<aside name="nested">

Remember that the compiler treats top-level code as the body of an implicit
function, so as soon as we add *any* function declarations, we're in a world of
nested functions.
请记住，编译器将顶层代码视为隐式函数的主体，因此只要添加任何函数声明，我们就会进入一个嵌套函数的世界。

</aside>

The trick for managing that is to create a separate Compiler for each function
being compiled. When we start compiling a function declaration, we create a new
Compiler on the C stack and initialize it. `initCompiler()` sets that Compiler
to be the current one. Then, as we compile the body, all of the functions that
emit bytecode write to the chunk owned by the new Compiler's function.
管理这个问题的诀窍是为每个正在编译的函数创建一个单独的Compiler。当我们开始编译函数声明时，会在C语言栈中创建一个新的Compiler并初始化它。`initCompiler()`将该Compiler设置为当前编译器。然后，在编译主体时，所有产生字节码的函数都写入新Compiler的函数所持有的字节码块。

After we reach the end of the function's block body, we call `endCompiler()`.
That yields the newly compiled function object, which we store as a constant in
the *surrounding* function's constant table. But, wait, how do we get back to
the surrounding function? We lost it when `initCompiler()` overwrote the current
compiler pointer.
在我们到达函数主体块的末尾时，会调用`endCompiler()`。这就得到了新编译的函数对象，我们将其作为常量存储在 *外围* 函数的常量表中。但是，等等。我们怎样才能回到外围的函数中呢？在`initCompiler()`覆盖当前编译器指针时，我们把它丢了。

We fix that by treating the series of nested Compiler structs as a stack. Unlike
the Value and CallFrame stacks in the VM, we won't use an array. Instead, we use
a linked list. Each Compiler points back to the Compiler for the function that
encloses it, all the way back to the root Compiler for the top-level code.
我们通过将一系列嵌套的Compiler结构体视为一个栈来解决这个问题。与VM中的Value和CallFrame栈不同，我们不会使用数组。相反，我们使用链表。每个Compiler都指向包含它的函数的Compiler，一直到顶层代码的根Compiler。

^code enclosing-field (2 before, 1 after)

Inside the Compiler struct, we can't reference the Compiler *typedef* since that
declaration hasn't finished yet. Instead, we give a name to the struct itself
and use that for the field's type. C is weird.
在Compiler结构体内部，我们不能引用Compiler*类型定义*，因为声明还没有结束。相反，我们要为结构体本身提供一个名称，并将其用作字段的类型。C语言真奇怪。

When initializing a new Compiler, we capture the about-to-no-longer-be-current
one in that pointer.
在初始化一个新的Compiler时，我们捕获即将更换的当前编译器。

^code store-enclosing (1 before, 1 after)

Then when a Compiler finishes, it pops itself off the stack by restoring the
previous compiler to be the new current one.
然后，当编译器完成时，将之前的编译器恢复为新的当前编译器，从而将自己从栈中弹出。

^code restore-enclosing (2 before, 1 after)

Note that we don't even need to <span name="compiler">dynamically</span>
allocate the Compiler structs. Each is stored as a local variable in the C stack
-- either in `compile()` or `function()`. The linked list of Compilers threads
through the C stack. The reason we can get an unbounded number of them is
because our compiler uses recursive descent, so `function()` ends up calling
itself recursively when you have nested function declarations.
请注意，我们甚至不需要动态地分配Compiler结构体。每个结构体都作为局部变量存储在C语言栈中 -- 不是`compile()`就是`function()`。编译器链表在C语言栈中存在。我们之所以能得到无限多的编译器，是因为我们的编译器使用了递归下降，所以当有嵌套的函数声明时，`function()`最终会递归地调用自己。

<aside name="compiler">

Using the native stack for Compiler structs does mean our compiler has a
practical limit on how deeply nested function declarations can be. Go too far
and you could overflow the C stack. If we want the compiler to be more robust
against pathological or even malicious code -- a real concern for tools like
JavaScript VMs -- it would be good to have our compiler artificially limit the
amount of function nesting it permits.
使用本地堆栈存储编译器结构体确实意味着我们的编译器对函数声明的嵌套深度有一个实际限制。如果嵌套太多，可能会导致C语言堆栈溢出。如果我们想让编译器能够更健壮地抵御错误甚至恶意的代码（这是JavaScript虚拟机等工具真正关心的问题），那么最好是人为地让编译器限制所允许的函数嵌套层级。

</aside>

### 函数参数

Functions aren't very useful if you can't pass arguments to them, so let's do
parameters next.
如果你不能向函数传递参数，那函数就不是很有用，所以接下来我们实现参数。

^code parameters (1 before, 1 after)

Semantically, a parameter is simply a local variable declared in the outermost
lexical scope of the function body. We get to use the existing compiler support
for declaring named local variables to parse and compile parameters. Unlike
local variables, which have initializers, there's no code here to initialize the
parameter's value. We'll see how they are initialized later when we do argument
passing in function calls.
语义上讲，形参就是在函数体最外层的词法作用域中声明的一个局部变量。我们可以使用现有的编译器对声明命名局部变量的支持来解析和编译形参。与有初始化器的局部变量不同，这里没有代码来初始化形参的值。稍后在函数调用中传递参数时，我们会看到它们是如何初始化的。

While we're at it, we note the function's arity by counting how many parameters
we parse. The other piece of metadata we store with a function is its name. When
compiling a function declaration, we call `initCompiler()` right after we parse
the function's name. That means we can grab the name right then from the
previous token.
在此过程中，我们通过计算所解析的参数数量来确定函数的元数。函数中存储的另一个元数据是它的名称。在编译函数声明时，我们在解析完函数名称之后，会立即调用`initCompiler()`。这意味着我们可以立即从上一个标识中获取名称。

^code init-function-name (1 before, 2 after)

Note that we're careful to create a copy of the name string. Remember, the
lexeme points directly into the original source code string. That string may get
freed once the code is finished compiling. The function object we create in the
compiler outlives the compiler and persists until runtime. So it needs its own
heap-allocated name string that it can keep around.
请注意，我们谨慎地创建了名称字符串的副本。请记住，词素直接指向了源代码字符串。一旦代码编译完成，该字符串就可能被释放。我们在编译器中创建的函数对象比编译器的寿命更长，并持续到运行时。所以它需要自己的堆分配的名称字符串，以便随时可用。

Rad. Now we can compile function declarations, like this:
太棒了。现在我们可以编译函数声明了，像这样：

```lox
fun areWeHavingItYet() {
  print "Yes we are!";
}

print areWeHavingItYet;
```

We just can't do anything <span name="useful">useful</span> with them.
只是我们还不能用它们来做任何有用的事情。

<aside name="useful">

We can print them! I guess that's not very useful, though.
我们可以打印出来！不过，我想这也没什么用。

</aside>

## Function Calls  函数调用

By the end of this section, we'll start to see some interesting behavior. The
next step is calling functions. We don't usually think of it this way, but a
function call expression is kind of an infix `(` operator. You have a
high-precedence expression on the left for the thing being called -- usually
just a single identifier. Then the `(` in the middle, followed by the argument
expressions separated by commas, and a final `)` to wrap it up at the end.
在本小节结束时，我们将开始看到一些有趣的行为。下一步是调用函数。我们通常不会这样想，但是函数调用表达式有点像是一个中缀`(`操作符。在左边有一个高优先级的表达式，表示被调用的内容 -- 通常只是一个标识符。然后是中间的`(`，后跟由逗号分隔的参数表达式，最后是一个`)`把它包起来。

That odd grammatical perspective explains how to hook the syntax into our
parsing table.
这个奇怪的语法视角解释了如何将语法挂接到我们的解析表格中。

^code infix-left-paren (1 before, 1 after)

When the parser encounters a left parenthesis following an expression, it
dispatches to a new parser function.
当解析器遇到表达式后面的左括号时，会将其分派到一个新的解析器函数。

^code compile-call

We've already consumed the `(` token, so next we compile the arguments using a
separate `argumentList()` helper. That function returns the number of arguments
it compiled. Each argument expression generates code that leaves its value on
the stack in preparation for the call. After that, we emit a new `OP_CALL`
instruction to invoke the function, using the argument count as an operand.
我们已经消费了`(`标识，所以接下来我们用一个单独的`argumentList()`辅助函数来编译参数。该函数会返回它所编译的参数的数量。每个参数表达式都会生成代码，将其值留在栈中，为调用做准备。之后，我们发出一条新的`OP_CALL`指令来调用该函数，将参数数量作为操作数。

We compile the arguments using this friend:
我们使用这个助手来编译参数：

^code argument-list

That code should look familiar from jlox. We chew through arguments as long as
we find commas after each expression. Once we run out, we consume the final
closing parenthesis and we're done.
这段代码看起来跟jlox很相似。只要我们在每个表达式后面找到逗号，就会仔细分析函数。一旦运行完成，消耗最后的右括号，我们就完成了。

Well, almost. Back in jlox, we added a compile-time check that you don't pass
more than 255 arguments to a call. At the time, I said that was because clox
would need a similar limit. Now you can see why -- since we stuff the argument
count into the bytecode as a single-byte operand, we can only go up to 255. We
need to verify that in this compiler too.
嗯，大概就这样。在jlox中，我们添加了一个编译时检查，即你不能向一次调用传递的参数不超过255个。当时，我说这是因为clox需要类似的限制。现在你可以明白为什么了 -- 因为我们把参数数量作为单字节操作数填充到字节码中，所以最多只能达到255。我们也需要在这个编译器中验证。

^code arg-limit (1 before, 1 after)

That's the front end. Let's skip over to the back end, with a quick stop in the
middle to declare the new instruction.
这就是前端。让我们跳到后端继续，不过要在中间快速暂停一下，声明一个新指令。

^code op-call (1 before, 1 after)

### 绑定形参与实参

Before we get to the implementation, we should think about what the stack looks
like at the point of a call and what we need to do from there. When we reach the
call instruction, we have already executed the expression for the function being
called, followed by its arguments. Say our program looks like this:
在我们开始实现之前，应该考虑一下堆栈在调用时是什么样子的，以及我们需要从中做什么。当我们到达调用指令时，我们已经执行了被调用函数的表达式，后面是其参数。假设我们的程序是这样的：

```lox
fun sum(a, b, c) {
  return a + b + c;
}

print 4 + sum(5, 6, 7);
```

If we pause the VM right on the `OP_CALL` instruction for that call to `sum()`,
the stack looks like this:
如果我们在调用`sum()`的`OP_CALL`指令处暂停虚拟机，栈看起来是这样的：

<img src="image/calls-and-functions/argument-stack.png" alt="Stack: 4, fn sum, 5, 6, 7." />

Picture this from the perspective of `sum()` itself. When the compiler compiled
`sum()`, it automatically allocated slot zero. Then, after that, it allocated
local slots for the parameters `a`, `b`, and `c`, in order. To perform a call to
`sum()`, we need a CallFrame initialized with the function being called and a
region of stack slots that it can use. Then we need to collect the arguments
passed to the function and get them into the corresponding slots for the
parameters.
从`sum()`本身的角度来考虑这个问题。当编译器编译`sum()`时，它自动分配了槽位0。然后，它在该位置后为参数`a`、`b`、`c`依次分配了局部槽。为了执行对`sum()`的调用，我们需要一个通过被调用函数和可用栈槽区域初始化的CallFrame。然后我们需要收集传递给函数的参数，并将它们放入参数对应的槽中。

When the VM starts executing the body of `sum()`, we want its stack window to
look like this:
当VM开始执行`sum()`函数体时，我们需要栈窗口看起来像这样：

<img src="image/calls-and-functions/parameter-window.png" alt="The same stack with the sum() function's call frame window surrounding fn sum, 5, 6, and 7." />

Do you notice how the argument slots that the caller sets up and the parameter
slots the callee needs are both in exactly the right order? How convenient! This
is no coincidence. When I talked about each CallFrame having its own window into
the stack, I never said those windows must be *disjoint*. There's nothing
preventing us from overlapping them, like this:
你是否注意到，调用者设置的实参槽和被调用者需要的形参槽的顺序是完全匹配的？多么方便啊！这并非巧合。当我谈到每个CallFrame在栈中都有自己的窗口时，从未说过这些窗口一定是不相交的。没有什么能阻止我们将它们重叠起来，就像这样：

<img src="image/calls-and-functions/overlapping-windows.png" alt="The same stack with the top-level call frame covering the entire stack and the sum() function's call frame window surrounding fn sum, 5, 6, and 7." />

<span name="lua">The</span> top of the caller's stack contains the function
being called followed by the arguments in order. We know the caller doesn't have
any other slots above those in use because any temporaries needed when
evaluating argument expressions have been discarded by now. The bottom of the
callee's stack overlaps so that the parameter slots exactly line up with where
the argument values already live.
调用者栈的顶部包括被调用的函数，后面依次是参数。我们知道调用者在这些正在使用的槽位之上没有占用其它槽，因为在计算参数表达式时需要的所有临时变量都已经被丢弃了。被调用者栈的底部是重叠的，这样形参的槽位与已有的实参值的位置就完全一致。

<aside name="lua">

Different bytecode VMs and real CPU architectures have different *calling
conventions*, which is the specific mechanism they use to pass arguments, store
the return address, etc. The mechanism I use here is based on Lua's clean, fast
virtual machine.
不同的字节码虚拟机和真实的CPU架构有不同的调用约定，也就是它们传递参数、存储返回地址等的具体机制。我在这里使用的机制是基于Lua干净、快速的虚拟机。

</aside>

This means that we don't need to do *any* work to "bind an argument to a
parameter". There's no copying values between slots or across environments. The
arguments are already exactly where they need to be. It's hard to beat that for
performance.
这意味着我们不需要做 *任何* 工作来“将形参绑定到实参”。不用在槽之间或跨环境复制值。这些实参已经在它们需要在的位置了。很难有比这更好的性能了。

Time to implement the call instruction.
是时候来实现调用指令了。

^code interpret-call (1 before, 1 after)

We need to know the function being called and the number of arguments passed to
it. We get the latter from the instruction's operand. That also tells us where
to find the function on the stack by counting past the argument slots from the
top of the stack. We hand that data off to a separate `callValue()` function. If
that returns `false`, it means the call caused some sort of runtime error. When
that happens, we abort the interpreter.
我们需要知道被调用的函数以及传递给它的参数数量。我们从指令的操作数中得到后者。它还告诉我们，从栈顶向下跳过参数数量的槽位，就可以在栈中找到该函数。我们将这些数据传给一个单独的`callValue()`函数。如果函数返回`false`，意味着该调用引发了某种运行时错误。当这种情况发生时，我们中止解释器。

If `callValue()` is successful, there will be a new frame on the CallFrame stack
for the called function. The `run()` function has its own cached pointer to the
current frame, so we need to update that.
如果`callValue()`成功，将会在CallFrame栈中为被调用函数创建一个新帧。`run()`函数有它自己缓存的指向当前帧的指针，所以我们需要更新它。

^code update-frame-after-call (2 before, 1 after)

Since the bytecode dispatch loop reads from that `frame` variable, when the VM
goes to execute the next instruction, it will read the `ip` from the newly
called function's CallFrame and jump to its code. The work for executing that
call begins here:
因为字节码调度循环会从`frame`变量中读取数据，当VM执行下一条指令时，它会从新的被调用函数CallFrame中读取`ip`，并跳转到其代码处。执行该调用的工作从这里开始：

^code call-value

<aside name="switch">

Using a `switch` statement to check a single type is overkill now, but will make
sense when we add cases to handle other callable types.
使用`switch`语句来检查一个类型现在看有些多余，但当我们添加case来处理其它调用类型时，就有意义了。

</aside>

There's more going on here than just initializing a new CallFrame. Because Lox
is dynamically typed, there's nothing to prevent a user from writing bad code
like:
这里要做的不仅仅是初始化一个新的CallFrame，因为Lox是动态类型的，所以没有什么可以防止用户编写这样的糟糕代码：

```lox
var notAFunction = 123;
notAFunction();
```

If that happens, the runtime needs to safely report an error and halt. So the
first thing we do is check the type of the value that we're trying to call. If
it's not a function, we error out. Otherwise, the actual call happens here:
如果发生这种情况，运行时需要安全报告错误并停止。所以我们要做的第一件事就是检查我们要调用的值的类型。如果不是函数，我们就报错退出。否则，真正的调用就发生在这里：

^code call

This simply initializes the next CallFrame on the stack. It stores a pointer to
the function being called and points the frame's `ip` to the beginning of the
function's bytecode. Finally, it sets up the `slots` pointer to give the frame
its window into the stack. The arithmetic there ensures that the arguments
already on the stack line up with the function's parameters:
这里只是初始化了栈上的下一个CallFrame。其中存储了一个指向被调用函数的指针，并将调用帧的`ip`指向函数字节码的开始处。最后，它设置`slots`指针，告诉调用帧它在栈上的窗口位置。这里的算法可以确保栈中已存在的实参与函数的形参是对齐的。

<img src="image/calls-and-functions/arithmetic.png" alt="The arithmetic to calculate frame-&gt;slots from stackTop and argCount." />

The funny little `- 1` is to account for stack slot zero which the compiler set
aside for when we add methods later. The parameters start at slot one so we
make the window start one slot earlier to align them with the arguments.
这个有趣的`-1`是为了处理栈槽0，编译器留出了这个槽，以便稍后添加方法时使用。形参从栈槽1开始，所以我们让窗口提前一个槽开始，以使它们与实参对齐。

Before we move on, let's add the new instruction to our disassembler.
在我们更进一步之前，让我们把新指令添加到反汇编程序中。

^code disassemble-call (1 before, 1 after)

And one more quick side trip. Now that we have a handy function for initiating a
CallFrame, we may as well use it to set up the first frame for executing the
top-level code.
还有一个快速的小改动。现在我们有一个方便的函数用来初始化CallFrame，我们不妨用它来设置用于执行顶层代码的第一个帧。

^code interpret (1 before, 2 after)

OK, now back to calls...
好了，现在回到调用……

### 运行时错误检查

The overlapping stack windows work based on the assumption that a call passes
exactly one argument for each of the function's parameters. But, again, because
Lox ain't statically typed, a foolish user could pass too many or too few
arguments. In Lox, we've defined that to be a runtime error, which we report
like so:
重叠的栈窗口的工作基于这样一个假设：一次调用中正好为函数的每个形参传入一个实参。但是，同样的，由于Lox不是静态类型的，某个愚蠢的用户可以会传入太多或太少的参数。在Lox中，我们将其定义为运行时错误，并像这样报告：

^code check-arity (1 before, 1 after)

Pretty straightforward. This is why we store the arity of each function inside
the ObjFunction for it.
非常简单直接。这就是为什么我们要在ObjFunction中存储每个函数的元数。

There's another error we need to report that's less to do with the user's
foolishness than our own. Because the CallFrame array has a fixed size, we need
to ensure a deep call chain doesn't overflow it.
还有一个需要报告的错误，与其说是用户的愚蠢行为，不如说是我们自己的愚蠢行为。因为CallFrame数组具有固定的大小，我们需要确保一个深的调用链不会溢出。

^code check-overflow (2 before, 1 after)

In practice, if a program gets anywhere close to this limit, there's most likely
a bug in some runaway recursive code.
在实践中，如果一个程序接近这个极限，那么很可能在某些失控的递归代码中出现了错误。

### 打印栈跟踪记录

While we're on the subject of runtime errors, let's spend a little time making
them more useful. Stopping on a runtime error is important to prevent the VM
from crashing and burning in some ill-defined way. But simply aborting doesn't
help the user fix their code that *caused* that error.
既然我们在讨论运行时错误，那我们就花一点时间让它们变得更有用。在出现运行时错误时停止很重要，可以防止虚拟机以某种不明确的方式崩溃。但是简单的中止并不能帮助用户修复导致错误的代码。

The classic tool to aid debugging runtime failures is a **stack trace** -- a
print out of each function that was still executing when the program died, and
where the execution was at the point that it died. Now that we have a call stack
and we've conveniently stored each function's name, we can show that entire
stack when a runtime error disrupts the harmony of the user's existence. It
looks like this:
帮助调试运行时故障的经典工具是**堆栈跟踪** -- 打印出程序死亡时仍在执行的每个函数，以及程序死亡时执行的位置。现在我们有了一个调度栈，并且方便地存储了每个函数的名称。当运行时错误破坏了用户的和谐时，我们可以显示整个堆栈。它看起来像这样：

^code runtime-error-stack (2 before, 2 after)

<aside name="minus">

The `- 1` is because the IP is already sitting on the next instruction to be
executed but we want the stack trace to point to the previous failed
instruction.
这里的`-1`是因为IP已经指向了下一条待执行的指令上 ，但我们希望堆栈跟踪指向前一条失败的指令。

</aside>

After printing the error message itself, we walk the call stack from <span
name="top">top</span> (the most recently called function) to bottom (the
top-level code). For each frame, we find the line number that corresponds to the
current `ip` inside that frame's function. Then we print that line number along
with the function name.
在打印完错误信息本身之后，我们从顶部（最近调用的函数）到底部（顶层代码）遍历调用栈。对于每个调用帧，我们找到与该帧的函数内的当前`ip`相对应的行号。然后我们将该行号与函数名称一起打印出来。

<aside name="top">

There is some disagreement on which order stack frames should be shown in a
trace. Most put the innermost function as the first line and work their way
towards the bottom of the stack. Python prints them out in the opposite order.
So reading from top to bottom tells you how your program got to where it is, and
the last line is where the error actually occurred.
关于栈帧在跟踪信息中显示的顺序，存在一些不同的意见。大部分把最内部的函数放在第一行，然后向堆栈的底部。Python则以相反的顺序打印出来。因此，从上到下阅读可以告诉你程序是如何达到现在的位置的，而最后一行是错误实际发生的地方。

There's a logic to that style. It ensures you can always see the innermost
function even if the stack trace is too long to fit on one screen. On the other
hand, the "[inverted pyramid][]" from journalism tells us we should put the most
important information *first* in a block of text. In a stack trace, that's the
function where the error actually occurred. Most other language implementations
do that.
这种风格有一个逻辑。它可以确保你始终可以看到最里面的函数，即使堆栈跟踪信息太长而无法在一个屏幕上显示。另一方面，新闻业中的“[倒金字塔][inverted pyramid]”告诉我们，我们应该把最重要的信息放在一段文字的前面。在堆栈跟踪中，这就是实际发生错误的函数。大多数其它语言的实现都是如此。

[inverted pyramid]: https://en.wikipedia.org/wiki/Inverted_pyramid_(journalism)

</aside>

For example, if you run this broken program:
举例来说，如果你运行这个坏掉的程序：

```lox
fun a() { b(); }
fun b() { c(); }
fun c() {
  c("too", "many");
}

a();
```

It prints out:
它会打印：

```text
Expected 0 arguments but got 2.
[line 4] in c()
[line 2] in b()
[line 1] in a()
[line 7] in script
```

That doesn't look too bad, does it?
看起来还不错，是吧？

### 从函数中返回

We're getting close. We can call functions, and the VM will execute them. But we
can't *return* from them yet. We've had an `OP_RETURN` instruction for quite
some time, but it's always had some kind of temporary code hanging out in it
just to get us out of the bytecode loop. The time has arrived for a real
implementation.
我们快完成了。我们可以调用函数，而虚拟机会执行它们。但是我们还不能从函数中返回。我们支持`OP_RETURN`指令已经有一段时间了，但其中一直有一些临时代码，只是为了让我们脱离字节码循环。现在是真正实现它的时候了。

^code interpret-return (1 before, 1 after)

When a function returns a value, that value will be on top of the stack. We're
about to discard the called function's entire stack window, so we pop that
return value off and hang on to it. Then we discard the CallFrame for the
returning function. If that was the very last CallFrame, it means we've finished
executing the top-level code. The entire program is done, so we pop the main
script function from the stack and then exit the interpreter.
当函数返回一个值时，该值会在栈顶。我们将会丢弃被调用函数的整个堆栈窗口，因此我们将返回值弹出栈并保留它。然后我们丢弃CallFrame，从函数中返回。如果是最后一个CallFrame，这意味着我们已经完成了顶层代码的执行。整个程序已经完成，所以我们从堆栈中弹出主脚本函数，然后退出解释器。

Otherwise, we discard all of the slots the callee was using for its parameters
and local variables. That includes the same slots the caller used to pass the
arguments. Now that the call is done, the caller doesn't need them anymore. This
means the top of the stack ends up right at the beginning of the returning
function's stack window.
否则，我们会丢弃所有被调用者用于存储参数和局部变量的栈槽，其中包括调用者用来传递实参的相同的槽。现在调用已经完成，调用者不再需要它们了。这意味着栈顶的结束位置正好在返回函数的栈窗口的开头。

We push the return value back onto the stack at that new, lower location. Then
we update the `run()` function's cached pointer to the current frame. Just like
when we began a call, on the next iteration of the bytecode dispatch loop, the
VM will read `ip` from that frame, and execution will jump back to the caller,
right where it left off, immediately after the `OP_CALL` instruction.
我们把返回值压回堆栈，放在新的、较低的位置。然后我们更新`run`函数中缓存的指针，将其指向当前帧。就像我们开始调用一样，在字节码调度循环的下一次迭代中，VM会从该帧中读取`ip`，执行程序会跳回调用者，就在它离开的地方，紧挨着`OP_CALL`指令之后。

<img src="image/calls-and-functions/return.png" alt="Each step of the return process: popping the return value, discarding the call frame, pushing the return value." />

Note that we assume here that the function *did* actually return a value, but
a function can implicitly return by reaching the end of its body:
请注意，我们这里假设函数确实返回了一个值，但是函数可以在到达主体末尾时隐式返回：

```lox
fun noReturn() {
  print "Do stuff";
  // No return here.
}

print noReturn(); // ???
```

We need to handle that correctly too. The language is specified to implicitly
return `nil` in that case. To make that happen, we add this:
我们也需要正确地处理这个问题。在这种情况下，语言被指定为隐式返回`nil`。为了实现这一点，我们添加了以下内容：

^code return-nil (1 before, 2 after)

The compiler calls `emitReturn()` to write the `OP_RETURN` instruction at the
end of a function body. Now, before that, it emits an instruction to push `nil`
onto the stack. And with that, we have working function calls! They can even
take parameters! It almost looks like we know what we're doing here.
编译器调用`emitReturn()`，在函数体的末尾写入`OP_RETURN`指令。现在，在此之前，它会生成一条指令将`nil`压入栈中。这样，我们就有了可行的函数调用！它们甚至可以接受参数！看起来我们好像知道自己在做什么。

## Return语句

If you want a function that returns something other than the implicit `nil`, you
need a `return` statement. Let's get that working.
如果你想让某个函数返回一些数据，而不是隐式的`nil`，你就需要一个`return`语句。我们来完成它。

^code match-return (1 before, 1 after)

When the compiler sees a `return` keyword, it goes here:
当编译器看到`return`关键字时，会进入这里：

^code return-statement

The return value expression is optional, so the parser looks for a semicolon
token to tell if a value was provided. If there is no return value, the
statement implicitly returns `nil`. We implement that by calling `emitReturn()`,
which emits an `OP_NIL` instruction. Otherwise, we compile the return value
expression and return it with an `OP_RETURN` instruction.
返回值表达式是可选的，因此解析器会寻找分号标识来判断是否提供了返回值。如果没有返回值，语句会隐式地返回`nil`。我们通过调用`emitReturn()`来实现，该函数会生成一个`OP_NIL`指令。否则，我们编译返回值表达式，并用`OP_RETURN`指令将其返回。

This is the same `OP_RETURN` instruction we've already implemented -- we don't
need any new runtime code. This is quite a difference from jlox. There, we had
to use exceptions to unwind the stack when a `return` statement was executed.
That was because you could return from deep inside some nested blocks. Since
jlox recursively walks the AST, that meant there were a bunch of Java method
calls we needed to escape out of.
这与我们已经实现的`OP_RETURN`指令相同 -- 我们不需要任何新的运行时代码。这与jlox有很大的不同。在jlox中，当执行`return`语句时，我们必须使用异常来跳出堆栈。这是因为你可以从某些嵌套的代码块深处返回。因为jlox递归地遍历AST。这意味着我们需要从一堆Java方法调用中退出。

Our bytecode compiler flattens that all out. We do recursive descent during
parsing, but at runtime, the VM's bytecode dispatch loop is completely flat.
There is no recursion going on at the C level at all. So returning, even from
within some nested blocks, is as straightforward as returning from the end of
the function's body.
我们的字节码编译器把这些都扁平化了。我们在解析时进行递归下降，但在运行时，虚拟机的字节码调度循环是完全扁平的。在C语言级别上根本没有发生递归。因此，即使从一些嵌套代码块中返回，也和从函数体的末端返回一样简单。

We're not totally done, though. The new `return` statement gives us a new
compile error to worry about. Returns are useful for returning from functions
but the top level of a Lox program is imperative code too. You shouldn't be able
to <span name="worst">return</span> from there.
不过，我们还没有完全完成。新的`return`语句为我们带来了一个新的编译错误。return语句从函数中返回是很有用的，但是Lox程序的顶层代码也是命令式代码。你不能从那里返回。

```lox
return "What?!";
```

<aside name="worst">

Allowing `return` at the top level isn't the worst idea in the world. It would
give you a natural way to terminate a script early. You could maybe even use a
returned number to indicate the process's exit code.
允许在顶层 `return` 并不是世界上最糟糕的主意。它可以为你提供一种自然的方式来提前终止脚本。你甚至可以用返回的数字来表示进程的推出码。

</aside>

We've specified that it's a compile error to have a `return` statement outside
of any function, which we implement like so:
我们已经规定，在任何函数之外有`return`语句都是编译错误，我们这样实现：

^code return-from-script (1 before, 1 after)

This is one of the reasons we added that FunctionType enum to the compiler.
这是我们在编译器中添加FunctionType枚举的原因之一。

## 本地函数

Our VM is getting more powerful. We've got functions, calls, parameters,
returns. You can define lots of different functions that can call each other in
interesting ways. But, ultimately, they can't really *do* anything. The only
user-visible thing a Lox program can do, regardless of its complexity, is print.
To add more capabilities, we need to expose them to the user.
我们的虚拟机越来越强大。我们已经支持了函数、调用、参数、返回。你可以定义许多不同的函数，它们可以以有趣的方式相互调用。但是，最终，它们什么都做不了。不管Lox程序有多复杂，它唯一能做的用户可见的事情就是打印。为了添加更多的功能，我们需要将函数暴露给用户。

A programming language implementation reaches out and touches the material world
through **native functions**. If you want to be able to write programs that
check the time, read user input, or access the file system, we need to add
native functions -- callable from Lox but implemented in C -- that expose those
capabilities.
编程语言的实现通过**本地函数**向外延伸并接触物质世界。如果你想编写检查时间、读取用户输入或访问文件系统的程序，则需要添加本地函数 -- 可以从Lox调用，但是使用C语言实现 -- 来暴露这些能力。

At the language level, Lox is fairly complete -- it's got closures, classes,
inheritance, and other fun stuff. One reason it feels like a toy language is
because it has almost no native capabilities. We could turn it into a real
language by adding a long list of them.
在语言层面，Lox是相当完整的 -- 它支持闭包、类、继承和其它有趣的东西。它之所以给人一种玩具语言的感觉，是因为它几乎没有原生功能。我们可以通过添加一系列功能将其变成一种真正的语言。

However, grinding through a pile of OS operations isn't actually very
educational. Once you've seen how to bind one piece of C code to Lox, you get
the idea. But you do need to see *one*, and even a single native function
requires us to build out all the machinery for interfacing Lox with C. So we'll
go through that and do all the hard work. Then, when that's done, we'll add one
tiny native function just to prove that it works.
然而，辛辛苦苦地完成一堆操作系统的操作，实际上并没有什么教育意义。只要你看到如何将一段C代码与Lox绑定，你就会明白了。但你确实需要看到一个例子，即使只是一个本地函数，我们也需要构建将Lox与C语言对接的所有机制。所以我们将详细讨论这个问题并完成所有困难的工作。等这些工作完成之后，我们会添加一个小小的本地函数，以证明它是可行的。

The reason we need new machinery is because, from the implementation's
perspective, native functions are different from Lox functions. When they are
called, they don't push a CallFrame, because there's no bytecode code for that
frame to point to. They have no bytecode chunk. Instead, they somehow reference
a piece of native C code.
我们需要新机制的原因是，从实现的角度来看，本地函数与Lox函数不同。当它们被调用时，它们不会压入一个CallFrame，因为没有这个帧要指向的字节码。它们没有字节码块。相反，它们会以某种方式引用一段本地C代码。

We handle this in clox by defining native functions as an entirely different
object type.
在clox中，我们通过将本地函数定义为一个完全不同的对象类型来处理这个问题。

^code obj-native (1 before, 2 after)

The representation is simpler than ObjFunction -- merely an Obj header and a
pointer to the C function that implements the native behavior. The native
function takes the argument count and a pointer to the first argument on the
stack. It accesses the arguments through that pointer. Once it's done, it
returns the result value.
其表示形式比ObjFunction更简单 -- 仅仅是一个Obj头和一个指向实现本地行为的C函数的指针。该本地函数接受参数数量和指向栈中第一个参数的指针。它通过该指针访问参数。一旦执行完成，它就返回结果值。

As always, a new object type carries some accoutrements with it. To create an
ObjNative, we declare a constructor-like function.
一如既往，一个新的对象类型会带有一些附属品。为了创建ObjNative，我们声明一个类似构造器的函数。

^code new-native-h (1 before, 1 after)

We implement that like so:
我们这样实现它：

^code new-native

The constructor takes a C function pointer to wrap in an ObjNative. It sets up
the object header and stores the function. For the header, we need a new object
type.
该构造函数接受一个C函数指针，并将其包装在ObjNative中。它会设置对象头并保存传入的函数。至于对象头，我们需要一个新的对象类型。

^code obj-type-native (2 before, 2 after)

The VM also needs to know how to deallocate a native function object.
虚拟机也需要知道如何释放本地函数对象。

^code free-native (1 before, 1 after)

There isn't much here since ObjNative doesn't own any extra memory. The other
capability all Lox objects support is being printed.
因为ObjNative并没有占用任何额外的内存，所以这里没有太多要做的。所有Lox对象需要支持的另一个功能是能够被打印。

^code print-native (1 before, 1 after)

In order to support dynamic typing, we have a macro to see if a value is a
native function.
为了支持动态类型，我们用一个宏来检查某个值是否本地函数。

^code is-native (1 before, 1 after)

Assuming that returns true, this macro extracts the C function pointer from a
Value representing a native function:
如果返回值为真，下面这个宏可以从一个代表本地函数的Value中提取C函数指针：

^code as-native (1 before, 1 after)

All of this baggage lets the VM treat native functions like any other object.
You can store them in variables, pass them around, throw them birthday parties,
etc. Of course, the operation we actually care about is *calling* them -- using
one as the left-hand operand in a call expression.
所有这些使得虚拟机可以像对待其它对象一样对待本地函数。你可以将它们存储在变量中，传递它们，给它们举办生日派对，等等。当然，我们真正关心的是*调用*它们 -- 将一个本地函数作为调用表达式的左操作数。

Over in `callValue()` we add another type case.
在 `callValue()`中，我们添加另一个类型的case分支。

^code call-native (2 before, 1 after)

If the object being called is a native function, we invoke the C function right
then and there. There's no need to muck with CallFrames or anything. We just
hand off to C, get the result, and stuff it back in the stack. This makes native
functions as fast as we can get.
如果被调用的对象是一个本地函数，我们就会立即调用C函数。没有必要使用CallFrames或其它任何东西。我们只需要交给C语言，得到结果，然后把结果塞回栈中。这使得本地函数的运行速度能够尽可能快。

With this, users should be able to call native functions, but there aren't any
to call. Without something like a foreign function interface, users can't define
their own native functions. That's our job as VM implementers. We'll start with
a helper to define a new native function exposed to Lox programs.
有了这个，用户应该能够调用本地函数了，但是还没有任何函数可供调用。如果没有外部函数接口之类的东西，用户就不能定义自己的本地函数。这就是我们作为虚拟机实现者的工作。我们将从一个辅助函数开始，定义一个新的本地函数暴露给Lox程序。

^code define-native

It takes a pointer to a C function and the name it will be known as in Lox.
We wrap the function in an ObjNative and then store that in a global variable
with the given name.
它接受一个指向C函数的指针及其在Lox中的名称。我们将函数包装在ObjNative中，然后将其存储在一个带有指定名称的全局变量中。

You're probably wondering why we push and pop the name and function on the
stack. That looks weird, right? This is the kind of stuff you have to worry
about when <span name="worry">garbage</span> collection gets involved. Both
`copyString()` and `newNative()` dynamically allocate memory. That means once we
have a GC, they can potentially trigger a collection. If that happens, we need
to ensure the collector knows we're not done with the name and ObjFunction so
that it doesn't free them out from under us. Storing them on the value stack
accomplishes that.
你可能像知道为什么我们要在栈中压入和弹出名称与函数。看起来很奇怪，是吧？当涉及到垃圾回收时，你必须考虑这类问题。`copyString()`和`newNative()`都是动态分配内存的。这意味着一旦我们有了GC，它们就有可能触发一次收集。如果发生这种情况，我们需要确保收集器知道我们还没有用完名称和ObjFunction ，这样垃圾回收就不会将这些数据从我们手下释放出来。将它们存储在值栈中可以做到这一点。

<aside name="worry">

Don't worry if you didn't follow all that. It will make a lot more sense once we
get around to [implementing the GC][gc].
如果你没搞懂也不用担心，一旦我们开始[实现GC][gc]，它就会变得更有意义。

[gc]: garbage-collection.html

</aside>

It feels silly, but after all of that work, we're going to add only one
little native function.
这感觉很傻，但是在完成所有这些工作之后，我们只会添加一个小小的本地函数。

^code clock-native

This returns the elapsed time since the program started running, in seconds. It's
handy for benchmarking Lox programs. In Lox, we'll name it `clock()`.
该函数会返回程序开始运行以来经过的时间，单位是秒。它对Lox程序的基准测试很有帮助。在Lox中，我们将其命名为`clock()`。

^code define-native-clock (1 before, 1 after)

To get to the C standard library `clock()` function, the "vm" module needs an
include.
为了获得C语言标准库中的`clock()`函数，`vm`模块需要引入头文件。

^code vm-include-time (1 before, 2 after)

That was a lot of material to work through, but we did it! Type this in and try
it out:
这部分有很多内容要处理，但是我们做到了！输入这段代码试试：

```lox
fun fib(n) {
  if (n < 2) return n;
  return fib(n - 2) + fib(n - 1);
}

var start = clock();
print fib(35);
print clock() - start;
```

We can write a really inefficient recursive Fibonacci function. Even better, we
can measure just <span name="faster">*how*</span> inefficient it is. This is, of
course, not the smartest way to calculate a Fibonacci number. But it is a good
way to stress test a language implementation's support for function calls. On my
machine, running this in clox is about five times faster than in jlox. That's
quite an improvement.
我们已经可以编写一个非常低效的递归斐波那契函数。更妙的是，我们可以测量它有多低效。当然，这不是计算斐波那契数的最聪明的方法，但这是一个针对语言实现对函数调用的支持进行压力测试的好方法。在我的机器上，clox中运行这个程序大约比jlox快5倍。这是个相当大的提升。

<aside name="faster">

It's a little slower than a comparable Ruby program run in Ruby 2.4.3p205, and
about 3x faster than one run in Python 3.7.3. And we still have a lot of simple
optimizations we can do in our VM.
它比在Ruby 2.4.3p205中运行的同类Ruby程序稍慢，比在Python 3.7.3中运行的程序快3倍左右。而且我们仍然可以在我们的虚拟机中做很多简单的优化。

</aside>

<div class="challenges">

## Challenges

1.  Reading and writing the `ip` field is one of the most frequent operations
    inside the bytecode loop. Right now, we access it through a pointer to the
    current CallFrame. That requires a pointer indirection which may force the
    CPU to bypass the cache and hit main memory. That can be a real performance
    sink.
    读写`ip`字段是字节码循环中最频繁的操作之一。新增，我们通过一个指向当前CallFrame的指针来访问它。这里需要一次指针间接引用，可能会迫使CPU绕过缓存而进入主存。这可能是一个真正的性能损耗。

    Ideally, we'd keep the `ip` in a native CPU register. C doesn't let us
    *require* that without dropping into inline assembly, but we can structure
    the code to encourage the compiler to make that optimization. If we store
    the `ip` directly in a C local variable and mark it `register`, there's a
    good chance the C compiler will accede to our polite request.
    理想情况下，我们一个将`ip`保存在一个本地CPU寄存器中。在不引入内联汇编的情况下，C语言中不允许我们这样做，但是我们可以通过结构化的代码来鼓励编译器进行优化。如果我们将`ip`直接存储在C局部变量中，并将其标记为`register`，那么C编译器很可能会同意我们的礼貌请求。

    This does mean we need to be careful to load and store the local `ip` back
    into the correct CallFrame when starting and ending function calls.
    Implement this optimization. Write a couple of benchmarks and see how it
    affects the performance. Do you think the extra code complexity is worth it?
    这确实意味着在开始和结束函数调用时，我们需要谨慎地从正确的CallFrame中加载和保存局部变量`ip`。请实现这一优化。写几个基准测试，看看它对性能有什么影响。您认为增加的代码复杂性值得吗？

2.  Native function calls are fast in part because we don't validate that the
    call passes as many arguments as the function expects. We really should, or
    an incorrect call to a native function without enough arguments could cause
    the function to read uninitialized memory. Add arity checking.
    本地函数调用之所以快，部分原因是我们没有验证调用时传入的参数是否与期望的一样多。我们确实应该这样做，否则在没有足够参数的情况下错误地调用本地函数，会导致函数读取未初始化的内存空间。请添加参数数量检查。

3.  Right now, there's no way for a native function to signal a runtime error.
    In a real implementation, this is something we'd need to support because
    native functions live in the statically typed world of C but are called
    from dynamically typed Lox land. If a user, say, tries to pass a string to
    `sqrt()`, that native function needs to report a runtime error.
    目前，本机函数还没有办法发出运行时错误的信号。在一个真正的语言实现中，这是我们需要支持的，因为本机函数存在于静态类型的C语言世界中，却被动态类型的Lox调用。假如说，用户试图向`sqrt()`传递一个字符串，则该本地函数需要报告一个运行时错误。

    Extend the native function system to support that. How does this capability
    affect the performance of native calls?
    扩展本地函数系统，以支持该功能。这个功能会如何影响本地调用的性能？

4.  Add some more native functions to do things you find useful. Write some
    programs using those. What did you add? How do they affect the feel of the
    language and how practical it is?
    添加一些本地函数来做你认为有用的事情。用它们写一些程序。你添加了什么？它们是如何影响语言的感觉和实用性的？

</div>


