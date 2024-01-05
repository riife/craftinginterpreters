> Caring too much for objects can destroy you. Only -- if you care for a thing
> enough, it takes on a life of its own, doesn't it? And isn’t the whole point
> of things -- beautiful things -- that they connect you to some larger beauty?
> 对物品过于关心会毁了你。只是，如果你对一件事物足够关心，它就有了自己的生命，不是吗？而事物 -- 美丽的事物 -- 的全部意义不就是把你和一些更大的美联系起来吗？
>
> <cite>Donna Tartt, <em>The Goldfinch</em></cite>

The last area left to implement in clox is object-oriented programming. <span
name="oop">OOP</span> is a bundle of intertwined features: classes, instances,
fields, methods, initializers, and inheritance. Using relatively high-level
Java, we packed all that into two chapters. Now that we're coding in C, which
feels like building a model of the Eiffel tower out of toothpicks, we'll devote
three chapters to covering the same territory. This makes for a leisurely stroll
through the implementation. After strenuous chapters like [closures][] and the
[garbage collector][], you have earned a rest. In fact, the book should be easy
from here on out.
clox中需要实现的最后一个领域是面向对象编程。OOP是一堆交织在一起的特性：类、实例、字段、方法、初始化式和继承。使用相对高级的Java，我们可以把这些内容都装进两章中。现在我们用C语言编写代码，感觉就像用牙签搭建埃菲尔铁塔的模型，我们将用三章的篇幅来涵盖这些内容。这使得我们可以悠闲地漫步在实现中。在经历了闭包和垃圾回收器这样艰苦的章节之后，你赢得了休息的机会。事实上，从这里开始，这本书都是很容易的。

<aside name="oop">

People who have strong opinions about object-oriented programming -- read
"everyone" -- tend to assume OOP means some very specific list of language
features, but really there's a whole space to explore, and each language has its
own ingredients and recipes.
那些对面向对象编程有强烈看法的人 -- 读作“每个人” -- 往往认为OOP意味着一些非常具体的语言特性清单，但实际上有一个完整的空间可以探索，而每种语言都有自己的成分和配方。

Self has objects but no classes. CLOS has methods but doesn't attach them to
specific classes. C++ initially had no runtime polymorphism -- no virtual
methods. Python has multiple inheritance, but Java does not. Ruby attaches
methods to classes, but you can also define methods on a single object.
Self有对象但没有类。CLOS有方法，当没有把它们附加到特定的类中。C++最初没有运行时多态 -- 没有虚方法。Python有多重继承，但Java没有。Ruby把方法附加在类上，但你也可以在单个对象上定义方法。

</aside>

In this chapter, we cover the first three features: classes, instances, and
fields. This is the stateful side of object orientation. Then in the next two
chapters, we will hang behavior and code reuse off of those objects.
在本章中，我们会介绍前三个特性：类、实例和字段。这就是面向对象中表现出状态的一面。然后在接下来的两章中，我们会对这些对象挂上行为和代码重用能力。

[closures]: closures.html
[garbage collector]: garbage-collection.html

## Class对象

In a class-based object-oriented language, everything begins with classes. They
define what sorts of objects exist in the program and are the factories used to
produce new instances. Going bottom-up, we'll start with their runtime
representation and then hook that into the language.
在一门基于类的面向对象的语言中，一切都从类开始。它们定义了程序中存在什么类型的对象，并且它们也是用来生产新实例的工厂。自下向上，我们将从它们的运行时表示形式开始，然后将其挂接到语言中。

By this point, we're well-acquainted with the process of adding a new object
type to the VM. We start with a struct.
至此，我们已经非常熟悉向VM添加新对象类型的过程了。我们从一个结构体开始。

^code obj-class (1 before, 2 after)

After the Obj header, we store the class's name. This isn't strictly needed for
the user's program, but it lets us show the name at runtime for things like
stack traces.
在Obj头文件之后，我们存储了类的名称。对于用户的程序来说，这一信息并不是严格需要的，但是它让我们可以在运行时显示名称，例如堆栈跟踪。

The new type needs a corresponding case in the ObjType enum.
新类型需要在ObjType枚举中有一个对应的项。

^code obj-type-class (1 before, 1 after)

And that type gets a corresponding pair of macros. First, for testing an
object's type:
而该类型会有一组对应的宏。首先，用于测试对象的类型：

^code is-class (2 before, 1 after)

And then for casting a Value to an ObjClass pointer:
然后是用于将一个Value转换为一个ObjClass指针：

^code as-class (2 before, 1 after)

The VM creates new class objects using this function:
VM使用这个函数创建新的类对象：

^code new-class-h (2 before, 1 after)

The implementation lives over here:
实现在这里：

^code new-class

Pretty much all boilerplate. It takes in the class's name as a string and stores
it. Every time the user declares a new class, the VM will create a new one of
these ObjClass structs to represent it.
几乎都是模板代码。它接受并保存字符串形式的类名。每当用户声明一个新类时，VM会创建一个新的ObjClass结构体来表示它。

<aside name="klass">

<img src="image/classes-and-instances/klass.png" alt="'Klass' in a zany kidz font."/>

I named the variable "klass" not just to give the VM a zany preschool "Kidz
Korner" feel. It makes it easier to get clox compiling as C++ where "class" is
a reserved word.
我将变量命名为“klass”，不仅仅是为了给虚拟机一种古怪的幼儿园的 "Kidz Korner" 感觉。它使得clox更容易被编译为C++，而C++中“class”是一个保留字。

</aside>

When the VM no longer needs a class, it frees it like so:
当VM不再需要某个类时，这样释放它：

^code free-class (1 before, 1 after)

<aside name="braces">

The braces here are pointless now, but will be useful in the next chapter when
we add some more code to the switch case.
这里的大括号现在没有意义，但在下一章我们为 switch case 添加更多代码时将会派上用场。

</aside>

We have a memory manager now, so we also need to support tracing through class
objects.
我们现在有一个内存管理器，所以我们也需要支持通过类对象进行跟踪。

^code blacken-class (1 before, 1 after)

When the GC reaches a class object, it marks the class's name to keep that
string alive too.
当GC到达一个类对象时，它会标记该类的名称，以保持该字符串也能存活。

The last operation the VM can perform on a class is printing it.
VM可以对类执行的最后一个操作是打印它。

^code print-class (1 before, 1 after)

A class simply says its own name.
类只是简单地说出它的名称。

## 类声明

Runtime representation in hand, we are ready to add support for classes to the
language. Next, we move into the parser.
有了运行时表示形式，我们就可以向语言中添加对类的支持了。接下来，我们进入解释器。

^code match-class (1 before, 1 after)

Class declarations are statements, and the parser recognizes one by the leading
`class` keyword. The rest of the compilation happens over here:
类声明是语句，解释器通过前面的`class`关键字识别声明语句。剩下部分的编译工作在这里进行：

^code class-declaration

Immediately after the `class` keyword is the class's name. We take that
identifier and add it to the surrounding function's constant table as a string.
As you just saw, printing a class shows its name, so the compiler needs to stuff
the name string somewhere that the runtime can find. The constant table is the
way to do that.
紧跟在`class`关键字之后的是类名。我们将这个标识符作为字符串添加到外围函数的常量表中。正如你刚才看到的，打印一个类会显示它的名称，所以编译器需要把这个名称字符串放在运行时可以找到的地方。常量表就是实现这一目的的方法。

The class's <span name="variable">name</span> is also used to bind the class
object to a variable of the same name. So we declare a variable with that
identifier right after consuming its token.
类名也被用来将类对象与一个同名变量绑定。因此，我们在使用完它的词法标识后，马上用这个标识符声明一个变量。

<aside name="variable">

We could have made class declarations be *expressions* instead of statements --
they are essentially a literal that produces a value after all. Then users would
have to explicitly bind the class to a variable themselves like:
我们可以让类声明成为表达式而不是语句 -- 比较它们本质上是一个产生值的字面量。然后用户必须自己显式地将类绑定到一个变量，比如：

```lox
var Pie = class {}
```

Sort of like lambda functions but for classes. But since we generally want
classes to be named anyway, it makes sense to treat them as declarations.
这有点像lambda函数，但只是针对类的。但由于我们通常希望类被命名，所以将其视为声明是有意义的。

</aside>

Next, we emit a new instruction to actually create the class object at runtime.
That instruction takes the constant table index of the class's name as an
operand.
接下来我们发出一条新指令，在运行时实际创建类对象。该指令以类名的常量表索引作为操作数。

After that, but before compiling the body of the class, we define the variable
for the class's name. *Declaring* the variable adds it to the scope, but recall
from [a previous chapter][scope] that we can't *use* the variable until it's
*defined*. For classes, we define the variable before the body. That way, users
can refer to the containing class inside the bodies of its own methods. That's
useful for things like factory methods that produce new instances of the class.
在此之后，但是在编译类主体之前，我们使用类名定义变量。*声明*变量会将其添加到作用域中，但请回想一下前一章的内容，在定义变量之前我们不能使用它。对于类，我们在解析主体之前定义变量。这样，用户就可以在类自己的方法主体中引用类本身。这对于产生类的新实例的工厂方法等场景来说是很有用的。

[scope]: local-variables.html#another-scope-edge-case

Finally, we compile the body. We don't have methods yet, so right now it's
simply an empty pair of braces. Lox doesn't require fields to be declared in the
class, so we're done with the body -- and the parser -- for now.
最后，我们编译主体。我们现在还没有方法，所以现在它只是一对空的大括号。Lox不要求在类中声明字段，因此我们目前已经完成了主体（和解析器）的工作。

The compiler is emitting a new instruction, so let's define that.
编译器会发出一条新指令，所以我们来定义它。

^code class-op (1 before, 1 after)

And add it to the disassembler:
然后将其添加到反汇编程序中：

^code disassemble-class (2 before, 1 after)

For such a large-seeming feature, the interpreter support is minimal.
对于这样一个看起来很大的特性，解释器支持是最小的。

^code interpret-class (2 before, 1 after)

We load the string for the class's name from the constant table and pass that to
`newClass()`. That creates a new class object with the given name. We push that
onto the stack and we're good. If the class is bound to a global variable, then
the compiler's call to `defineVariable()` will emit code to store that object
from the stack into the global variable table. Otherwise, it's right where it
needs to be on the stack for a new <span name="local">local</span> variable.
我们从常量表中加载类名的字符串，并将其传递给`newClass()`。这将创建一个具有给定名称的新类对象。我们把它推入栈中就可以了。如果该类被绑定到一个全局变量上，那么编译器对`defineVariable()`的调用就会生成字节码，将该对象从栈中存储到全局变量表。否则，它就正好位于栈中新的局部变量所在的位置。

<aside name="local">

"Local" classes -- classes declared inside the body of a function or block, are
an unusual concept. Many languages don't allow them at all. But since Lox is a
dynamically typed scripting language, it treats the top level of a program and
the bodies of functions and blocks uniformly. Classes are just another kind of
declaration, and since you can declare variables and functions inside blocks,
you can declare classes in there too.
“局部（Local）”类 -- 在函数或块主体中声明的类，是一个不寻常的概念。许多语言根本不允许这一特性。但由于Lox是一种动态类型脚本语言，它会对程序的顶层代码和函数以及块的主体进行统一处理。类只是另一种声明，既然你可以在块中声明变量和函数，那你也可以在块中声明类。

</aside>

There you have it, our VM supports classes now. You can run this:
好了，我们的虚拟机现在支持类了。你可以运行这段代码：

```lox
class Brioche {}
print Brioche;
```

Unfortunately, printing is about *all* you can do with classes, so next is
making them more useful.
不幸的是，打印是你对类所能做的全部事情，所以接下来是让它们更有用。

## 类的实例

Classes serve two main purposes in a language:
类在一门语言中主要有两个作用：

*   **They are how you create new instances.** Sometimes this involves a `new`
    keyword, other times it's a method call on the class object, but you usually
    mention the class by name *somehow* to get a new instance.
*   **They contain methods.** These define how all instances of the class
    behave.


*   **它们是你创建新实例的方式**。有时这会涉及到`new`关键字，有时则是对类对象的方法调用，但是你通常会以某种方式通过类的名称来获得一个新的实例。
*   **它们包含方法**。这些方法定义了类的所有实例的行为方式。

We won't get to methods until the next chapter, so for now we will only worry
about the first part. Before classes can create instances, we need a
representation for them.
我们要到下一章才会讲到方法，所以我们现在只关心第一部分。在类能够创建实例之前，我们需要为它们提供一个表示形式。

^code obj-instance (1 before, 2 after)

Instances know their class -- each instance has a pointer to the class that it
is an instance of.  We won't use this much in this chapter, but it will become
critical when we add methods.
实例知道它们的类 -- 每个实例都有一个指向它所属类的指针。在本章中我们不会过多地使用它，但是等我们添加方法时，它将会变得非常重要。

More important to this chapter is how instances store their state. Lox lets
users freely add fields to an instance at runtime. This means we need a storage
mechanism that can grow. We could use a dynamic array, but we also want to look
up fields by name as quickly as possible. There's a data structure that's just
perfect for quickly accessing a set of values by name and
-- even more conveniently -- we've already implemented it. Each instance stores
its fields using a hash table.
对本章来说，更重要的是实例如何存储它们的状态。Lox允许用户在运行时自由地向实例中添加字段。这意味着我们需要一种可以增长的存储机制。我们可以使用动态数组，但我们也希望尽可能快地按名称查找字段。有一种数据结构非常适合于按名称快速访问一组值 -- 甚至更方便的是 -- 我们已经实现了它。每个实例都使用哈希表来存储其字段。

<aside name="fields">

Being able to freely add fields to an object at runtime is a big practical
difference between most dynamic and static languages. Statically typed languages
usually require fields to be explicitly declared. This way, the compiler knows
exactly what fields each instance has. It can use that to determine the precise
amount of memory needed for each instance and the offsets in that memory where
each field can be found.
能够在运行时自由地向对象添加字段，是大多数动态语言和静态语言之间的一个很大的实际区别。静态类型语言通常要求显式声明字段。这样，编译器就确切知道每个实例有哪些字段。它可以利用这一点来确定每个实例所需的精确内存量，以及每个字段在内存中的偏移量。

In Lox and other dynamic languages, accessing a field is usually a hash table
lookup. Constant time, but still pretty heavyweight. In a language like C++,
accessing a field is as fast as offsetting a pointer by an integer constant.
在Lox和其它动态语言中，访问字段通常是一次哈希表查询。常量时间复杂度，但仍然是相当重的。在C++这样的语言中，访问一个字段就像对指针偏移一个整数常量一样快。

</aside>

We only need to add an include, and we've got it.
我们只需要添加一个头文件引入，就可以了。

^code object-include-table (1 before, 1 after)

This new struct gets a new object type.
新结构体有新的对象类型。

^code obj-type-instance (1 before, 1 after)

I want to slow down a bit here because the Lox *language's* notion of "type" and
the VM *implementation's* notion of "type" brush against each other in ways that
can be confusing. Inside the C code that makes clox, there are a number of
different types of Obj -- ObjString, ObjClosure, etc. Each has its own internal
representation and semantics.
这里我想放慢一点速度，因为Lox*语言*中的“type”概念和*虚拟机实现*中的“type”概念是相互抵触的，可能会造成混淆。在生成clox 的C语言代码中，有许多不同类型的Obj -- ObjString、ObjClosure等等。每个都有自己的内部表示和语义。

In the Lox *language*, users can define their own classes -- say Cake and Pie --
and then create instances of those classes. From the user's perspective, an
instance of Cake is a different type of object than an instance of Pie. But,
from the VM's perspective, every class the user defines is simply another value
of type ObjClass. Likewise, each instance in the user's program, no matter what
class it is an instance of, is an ObjInstance. That one VM object type covers
instances of all classes. The two worlds map to each other something like this:
在Lox*语言*中，用户可以定义自己的类 -- 比如Cake和Pie -- 然后创建这些类的实例。从用户的角度来看，Cake实例与Pie实例是不同类型的对象。但是，从虚拟机的角度来看，用户定义的每个类都只是另一个ObjClass类型的值。同样，用户程序中的每个实例，无论它是什么类的实例，都是一个ObjInstance。这一虚拟机对象类型涵盖了所有类的实例。这两个世界之间的映射是这样的：

<img src="image/classes-and-instances/lox-clox.png" alt="A set of class declarations and instances, and the runtime representations each maps to."/>

Got it? OK, back to the implementation. We also get our usual macros.
明白了吗？好了，回到实现中。我们新增了一些熟悉的宏。

^code is-instance (1 before, 1 after)

And:

^code as-instance (1 before, 1 after)

Since fields are added after the instance is created, the "constructor" function
only needs to know the class.
因为字段是在实例创建之后添加的，所以“构造器”函数只需要知道类。

^code new-instance-h (1 before, 1 after)

We implement that function here:
我们在这里实现该函数：

^code new-instance

We store a reference to the instance's class. Then we initialize the field
table to an empty hash table. A new baby object is born!
我们存储了对实例的类的引用。然后我们将字段表初始化为一个空的哈希表。一个全新的对象诞生了！

At the sadder end of the instance's lifespan, it gets freed.
在实例生命周期的最后阶段，它被释放了。

^code free-instance (3 before, 1 after)

The instance owns its field table so when freeing the instance, we also free the
table. We don't explicitly free the entries *in* the table, because there may
be other references to those objects. The garbage collector will take care of
those for us. Here we free only the entry array of the table itself.
实例拥有自己的字段表，所以当释放实例时，我们也会释放该表。我们没有显式地释放表中的条目，因为可能存在对这些对象的其它引用。垃圾回收器会帮我们处理这些问题。这里我们只释放表本身的条目数组。

Speaking of the garbage collector, it needs support for tracing through
instances.
说到垃圾回收，它需要支持通过实例进行跟踪。

^code blacken-instance (3 before, 1 after)

If the instance is alive, we need to keep its class around. Also, we need to
keep every object referenced by the instance's fields. Most live objects that
are not roots are reachable because some instance refers to the object in a
field. Fortunately, we already have a nice `markTable()` function to make
tracing them easy.
如果这个实例是活动的，我们需要保留它的类。此外，我们还需要保留每个被实例字段引用的对象。大多数不是根的活动对象都是可达的，因为某些实例会在某个字段中引用该对象。幸运的是，我们已经有了一个很好的`markTable()`函数，可以轻松地跟踪它们。

Less critical but still important is printing.
不太关键但仍然重要的是打印。

^code print-instance (1 before, 1 after)

<span name="print">An</span> instance prints its name followed by "instance".
(The "instance" part is mainly so that classes and instances don't print the
same.)
实例会打印它的名称，并在后面加上“instance”。（“instance”部分主要是为了使类和实例不会打印出相同的内容）

<aside name="print">

Most object-oriented languages let a class define some sort of `toString()`
method that lets the class specify how its instances are converted to a string
and printed. If Lox was less of a toy language, I would want to support that
too.
大多数面向对象的语言允许类定义某种形式的`toString()`方法，让该类指定如何将其实例转换为字符串并打印出来。如果Lox部署一门玩具语言，我也想要支持它。

</aside>

The real fun happens over in the interpreter. Lox has no special `new` keyword.
The way to create an instance of a class is to invoke the class itself as if it
were a function. The runtime already supports function calls, and it checks the
type of object being called to make sure the user doesn't try to invoke a number
or other invalid type.
真正有趣的部分在解释器中，Lox没有特殊的`new`关键字。创建类实例的方法是调用类本身，就像调用函数一样。运行时已经支持函数调用，它会检查被调用对象的类型，以确保用户不会试图调用数字或其它无效类型。

We extend that runtime checking with a new case.
我们用一个新的case分支来扩展运行时的检查。

^code call-class (1 before, 1 after)

If the value being called -- the object that results when evaluating the
expression to the left of the opening parenthesis -- is a class, then we treat
it as a constructor call. We <span name="args">create</span> a new instance of
the called class and store the result on the stack.
如果被调用的值（在左括号左边的表达式求值得到的对象）是一个类，则将其视为一个构造函数调用。我们创建一个被调用类的新实例，并将结果存储在栈中。

<aside name="args">

We ignore any arguments passed to the call for now. We'll revisit this code in
the [next chapter][next] when we add support for initializers.
我们暂时忽略传递给调用的所有参数。在下一章添加对初始化器的支持时，我们会重新审视这一段代码。

[next]: methods-and-initializers.html

</aside>

We're one step farther. Now we can define classes and create instances of them.
我们又前进了一步。现在我们可以定义类并创建它们的实例了。

```lox
class Brioche {}
print Brioche();
```

Note the parentheses after `Brioche` on the second line now. This prints
"Brioche instance".
注意第二行`Brioche`后面的括号。这里会打印“Brioche instance”。

## Get和SET表达式

Our object representation for instances can already store state, so all that
remains is exposing that functionality to the user. Fields are accessed and
modified using get and set expressions. Not one to break with tradition, Lox
uses the classic "dot" syntax:
实例的对象表示形式已经可以存储状态了，所以剩下的就是把这个功能暴露给用户。字段是使用get和set表达式进行访问和修改的。Lox并不喜欢打破传统，这里也沿用了经典的“点”语法：

```lox
eclair.filling = "pastry creme";
print eclair.filling;
```

The period -- full stop for my English friends -- works <span
name="sort">sort</span> of like an infix operator. There is an expression to the
left that is evaluated first and produces an instance. After that is the `.`
followed by a field name. Since there is a preceding operand, we hook this into
the parse table as an infix expression.
句号 -- 对英国朋友来说是句号 -- 其作用有点像一个中缀运算符。左边有一个表达式，首先被求值并产生一个实例。之后是`.`后跟一个字段名称。由于前面有一个操作数，我们将其作为中缀表达式放到解析表中。

<aside name="sort">

I say "sort of" because the right-hand side after the `.` is not an expression,
but a single identifier whose semantics are handled by the get or set expression
itself. It's really closer to a postfix expression.
我说“有点”是因为`.`右边的不是表达式，而是一个标识符，其语义由get或set表达式本身来处理。它实际上更接近于一个后缀表达式。

</aside>

^code table-dot (1 before, 1 after)

As in other languages, the `.` operator binds tightly, with precedence as high
as the parentheses in a function call. After the parser consumes the dot token,
it dispatches to a new parse function.
和其它语言一样，`.`操作符绑定紧密，其优先级和函数调用中的括号一样高。解析器消费了点标识之后，会分发给一个新的解析函数。

^code compile-dot

The parser expects to find a <span name="prop">property</span> name immediately
after the dot. We load that token's lexeme into the constant table as a string
so that the name is available at runtime.
解析器希望在点运算符后面立即找到一个属性名称。我们将该词法标识的词素作为字符串加载到常量表中，这样该名称在运行时就是可用的。

<aside name="prop">

The compiler uses "property" instead of "field" here because, remember, Lox also
lets you use dot syntax to access a method without calling it. "Property" is the
general term we use to refer to any named entity you can access on an instance.
Fields are the subset of properties that are backed by the instance's state.
编译器在这里使用“属性（property）”而不是“字段（field）”，因为，请记住，Lox还允许你使用点语法来访问一个方法而不调用它。“属性”是一个通用术语，我们用来指代可以在实例上访问的任何命名实体。字段是基于实例状态的属性子集。

</aside>

We have two new expression forms -- getters and setters -- that this one
function handles. If we see an equals sign after the field name, it must be a
set expression that is assigning to a field. But we don't *always* allow an
equals sign after the field to be compiled. Consider:
我们将两种新的表达式形式 -- getter和setter -- 都交由这一个函数处理。如果我们看到字段名称后有一个等号，那么它一定是一个赋值给字段的set表达式。但我们并不总是允许编译字段后面的等号。考虑一下：

```lox
a + b.c = 3
```

This is syntactically invalid according to Lox's grammar, which means our Lox
implementation is obligated to detect and report the error. If `dot()` silently
parsed the `= 3` part, we would incorrectly interpret the code as if the user
had written:
根据Lox的文法，这在语法上是无效的，这意味着我们的Lox实现有义务检测和报告这个错误。如果`dot()`默默地解析`=3`的部分，我们就会错误地解释代码，就像用户写的是：

```lox
a + (b.c = 3)
```

The problem is that the `=` side of a set expression has much lower precedence
than the `.` part. The parser may call `dot()` in a context that is too high
precedence to permit a setter to appear. To avoid incorrectly allowing that, we
parse and compile the equals part only when `canAssign` is true. If an equals
token appears when `canAssign` is false, `dot()` leaves it alone and returns. In
that case, the compiler will eventually unwind up to `parsePrecedence()`, which
stops at the unexpected `=` still sitting as the next token and reports an
error.
问题是，set表达式中的`=`侧优先级远低于`.`部分。解析器有可能会在一个优先级高到不允许出现setter的上下文中调用`dot()`。为了避免错误地允许这种情况，我们只有在`canAssign`为true时才去解析和编译等号部分。如果在`canAssign`为false时出现等号标识，`dot()`会保留它并返回。在这种情况下，编译器最终会进入`parsePrecedence()`，而该方法会在非预期的`=`（仍然作为下一个标识）处停止，并报告一个错误。

If we find an `=` in a context where it *is* allowed, then we compile the
expression that follows. After that, we emit a new <span
name="set">`OP_SET_PROPERTY`</span> instruction. That takes a single operand for
the index of the property name in the constant table. If we didn't compile a set
expression, we assume it's a getter and emit an `OP_GET_PROPERTY` instruction,
which also takes an operand for the property name.
如果我们在允许使用等号的上下文中找到`=`，则编译后面的表达式。之后，我们发出一条新的`OP_SET_PROPERTY`指令。这条指令接受一个操作数，作为属性名称在常量表中的索引。如果我们没有编译set表达式，就假定它是getter，并发出一条`OP_GET_PROPERTY`指令，它也接受一个操作数作为属性名。

<aside name="set">

You can't *set* a non-field property, so I suppose that instruction could have
been `OP_SET_FIELD`, but I thought it looked nicer to be consistent with the get
instruction.
你不能设置非字段属性，所以我认为这个指令本该是`OP_SET_FIELD`，但是我认为它与get指令一致看起来更漂亮。

</aside>

Now is a good time to define these two new instructions.
现在是定义这两条新指令的好时机。

^code property-ops (1 before, 1 after)

And add support for disassembling them:
并在反汇编程序中为它们添加支持：

^code disassemble-property-ops (1 before, 1 after)

### 解释getter和setter表达式

Sliding over to the runtime, we'll start with get expressions since those are a
little simpler.
进入运行时，我们从获取表达式开始，因为它们更简单一些。

^code interpret-get-property (1 before, 1 after)

When the interpreter reaches this instruction, the expression to the left of the
dot has already been executed and the resulting instance is on top of the stack.
We read the field name from the constant pool and look it up in the instance's
field table. If the hash table contains an entry with that name, we pop the
instance and push the entry's value as the result.
当解释器到达这条指令时，点左边的表达式已经被执行，得到的实例就在栈顶。我们从常量池中读取字段名，并在实例的字段表中查找该名称。如果哈希表中包含具有该名称的条目，我们就弹出实例，并将该条目的值作为结果压入栈。

Of course, the field might not exist. In Lox, we've defined that to be a runtime
error. So we add a check for that and abort if it happens.
当然，这个字段可能不存在。在Lox中，我们将其定义为运行时错误。所以我们添加了一个检查，如果发生这种情况就中止。

^code get-undefined (3 before, 2 after)

<span name="field">There</span> is another failure mode to handle which you've
probably noticed. The above code assumes the expression to the left of the dot
did evaluate to an ObjInstance. But there's nothing preventing a user from
writing this:
你可能已经注意到了，还有另一种需要处理的失败模式。上面的代码中假定了点左边的表达式计算结果确实是一个ObjInstance。但是没有什么可以阻止用户这样写：

```lox
var obj = "not an instance";
print obj.field;
```

The user's program is wrong, but the VM still has to handle it with some grace.
Right now, it will misinterpret the bits of the ObjString as an ObjInstance and,
I don't know, catch on fire or something definitely not graceful.
用户的程序是错误的，但是虚拟机仍然需要以某种优雅的方式来处理它。现在，它会把ObjString 数据误认为是一个ObjInstance ，并且，我不确定，代码起火或发生其它事情绝对是不优雅的。

In Lox, only instances are allowed to have fields. You can't stuff a field onto
a string or number. So we need to check that the value is an instance before
accessing any fields on it.
在Lox中，只有实例才允许有字段。你不能把字段塞到字符串或数字中。因此，在访问某个值上的任何字段之前，检查该值是否是一个实例。

<aside name="field">

Lox *could* support adding fields to values of other types. It's our language
and we can do what we want. But it's likely a bad idea. It significantly
complicates the implementation in ways that hurt performance -- for example,
string interning gets a lot harder.
Lox*可以*支持向其它类型的值中添加字段。这是我们的语言，我们可以做我们想做的。但这可能是个坏主意。它大大增加了实现的复杂性，从而损害了性能 -- 例如，字符串驻留变得更加困难。<BR>

Also, it raises gnarly semantic questions around the equality and identity of
values. If I attach a field to the number `3`, does the result of `1 + 2` have
that field as well? If so, how does the implementation track that? If not, are
those two resulting "threes" still considered equal?
此外，它还引起了关于数值的相等和同一性的复杂语义问题。如果我给数字`3`附加一个字段，那么`1+2`的结果也有这个字段吗？如果是的话，实现上如何跟踪它？如果不是，这两个结果中的“3”仍然被认为是相等的吗？

</aside>

^code get-not-instance (1 before, 1 after)

If the value on the stack isn't an instance, we report a runtime error and
safely exit.
如果栈中的值不是实例，则报告一个运行时错误并安全退出。

Of course, get expressions are not very useful when no instances have any
fields. For that we need setters.
当然，如果实例没有任何字段，get表达式就不太有用了。因此，我们需要setter。

^code interpret-set-property (2 before, 1 after)

This is a little more complex than `OP_GET_PROPERTY`. When this executes, the
top of the stack has the instance whose field is being set and above that, the
value to be stored. Like before, we read the instruction's operand and find the
field name string. Using that, we store the value on top of the stack into the
instance's field table.
这比`OP_GET_PROPERTY`要复杂一些。当执行此指令时，栈顶有待设置字段的实例，在该实例之上有要存储的值。与前面一样，我们读取指令的操作数，并查找字段名称字符串。使用该方法，我们将栈顶的值存储到实例的字段表中。

After that is a little <span name="stack">stack</span> juggling. We pop the
stored value off, then pop the instance, and finally push the value back on. In
other words, we remove the *second* element from the stack while leaving the top
alone. A setter is itself an expression whose result is the assigned value, so
we need to leave that value on the stack. Here's what I mean:
在那之后是一些栈技巧。我们将存储的值弹出，然后弹出实例，最后再把值压回栈中。换句话说，我们从栈中删除第二个元素，而保留最上面的元素。setter本身是一个表达式，其结果就是所赋的值，所以我们需要将值保留在栈上。我的意思是：

<aside name="stack">

The stack operations go like this:
栈的操作是这样的：

<img src="image/classes-and-instances/stack.png" alt="Popping two values and then pushing the first value back on the stack."/>

</aside>

```lox
class Toast {}
var toast = Toast();
print toast.jam = "grape"; // Prints "grape".
```

Unlike when reading a field, we don't need to worry about the hash table not
containing the field. A setter implicitly creates the field if needed. We do
need to handle the user incorrectly trying to store a field on a value that
isn't an instance.
与读取字段不同，我们不需要担心哈希表中不包含该字段。如果需要的话，setter会隐式地创建这个字段。我们确实需要处理用户不正确地试图在非实例的值上存储字段的情况。

^code set-not-instance (1 before, 1 after)

Exactly like with get expressions, we check the value's type and report a
runtime error if it's invalid. And, with that, the stateful side of Lox's
support for object-oriented programming is in place. Give it a try:
就像get表达式一样，我们检查值的类型，如果无效就报告一个运行时错误。这样一来，Lox对面向对象编程中有状态部分的支持就到位了。试一试：

```lox
class Pair {}

var pair = Pair();
pair.first = 1;
pair.second = 2;
print pair.first + pair.second; // 3.
```

This doesn't really feel very *object*-oriented. It's more like a strange,
dynamically typed variant of C where objects are loose struct-like bags of data.
Sort of a dynamic procedural language. But this is a big step in expressiveness.
Our Lox implementation now lets users freely aggregate data into bigger units.
In the next chapter, we will breathe life into those inert blobs.
这感觉不太面向对象。它更像是一种奇怪的、动态类型的C语言变体，其中的对象是松散的类似结构体的数据包。有点像动态过程化语言。但这是表达能力的一大进步。我们的Lox实现现在允许用户自由地将数据聚合成更大的单元。在下一章中，我们将为这些迟缓的数据注入活力。

<div class="challenges">

## Challenges

1.  Trying to access a non-existent field on an object immediately aborts the
    entire VM. The user has no way to recover from this runtime error, nor is
    there any way to see if a field exists *before* trying to access it. It's up
    to the user to ensure on their own that only valid fields are read.
    试图访问一个对象上不存在的字段会立即中止整个虚拟机。用户没有办法从这个运行时错误中恢复过来，也没有办法在试图访问一个字段之前看它是否存在。需要由用户自己来确保只读取有效字段。

    How do other dynamically typed languages handle missing fields? What do you
    think Lox should do? Implement your solution.
    其它动态类型语言是如何处理缺少字段的？你认为Lox应该怎么做？实现你的解决方案。

2.  Fields are accessed at runtime by their *string* name. But that name must
    always appear directly in the source code as an *identifier token*. A user
    program cannot imperatively build a string value and then use that as the
    name of a field. Do you think they should be able to? Devise a language
    feature that enables that and implement it.
    字段在运行时是通过它们的*字符串*名称来访问的。但是该名称必须总是作为标识符直接出现在源代码中。用户程序不能命令式地构建字符串值，然后将其用作字段名。你认为应该这样做吗？那就设计一种语言特性来实现它。

3.  Conversely, Lox offers no way to *remove* a field from an instance. You can
    set a field's value to `nil`, but the entry in the hash table is still
    there. How do other languages handle this? Choose and implement a strategy
    for Lox.
    反过来说，Lox没有提供从实例中*删除*字段的方法。你可以将一个字段的值设置为`nil`，但哈希表中的条目仍然存在。其它语言如何处理这个问题？为Lox选择一个策略并实现。

4.  Because fields are accessed by name at runtime, working with instance state
    is slow. It's technically a constant-time operation -- thanks, hash tables
    -- but the constant factors are relatively large. This is a major component
    of why dynamic languages are slower than statically typed ones.
    因为字段在运行时是按照名称访问的，所以对实例状态的操作是很慢的。从技术上讲，这是一个常量时间的操作（感谢哈希表），但是常量因子比较大。这就是动态语言比静态语言慢的一个主要原因。

    How do sophisticated implementations of dynamically typed languages cope
    with and optimize this?
    动态类型语言的复杂实现是如何应对和优化这一问题的？

</div>


