For your edification, here is the code produced by [the little script
we built][generator] to automate generating the syntax tree classes for jlox.
为了方便你们学习，下面是我们为自动生成jlox语法树类而[构建的小脚本](http://www.craftinginterpreters.com/representing-code.html#metaprogramming-the-trees)所产生的代码。

[generator]: representing-code.html#metaprogramming-the-trees

## Expressions  表达式

Expressions are the first syntax tree nodes we see, introduced in "[Representing
Code](representing-code.html)". The main Expr class defines the visitor
interface used to dispatch against the specific expression types, and contains
the other expression subclasses as nested classes.
表达式是我们看到的第一个语法树节点，在“表示代码”中介绍过。主要的Expr类定义了用于针对特定表达式类型进行调度的访问者接口，并将其它表达式子类作为嵌套类包含其中。

^code expr

### Assign expression

Variable assignment is introduced in "[Statements and
State](statements-and-state.html#assignment)".
变量赋值在“表达式与状态”中介绍过。

^code expr-assign

### Binary expression

Binary operators are introduced in "[Representing
Code](representing-code.html)".
二元运算符在“表示代码”中介绍过。

^code expr-binary

### Call expression

Function call expressions are introduced in
"[Functions](functions.html#function-calls)".
函数调用语句在“函数”中介绍过。

^code expr-call

### Get expression

Property access, or "get" expressions are introduced in
"[Classes](classes.html#properties-on-instances)".
属性访问，或者说“get”表达式，在“类”中介绍过。

^code expr-get

### Grouping expression

Using parentheses to group expressions is introduced in "[Representing
Code](representing-code.html)".
使用括号进行分组的表达式在“表示代码”中介绍过。

^code expr-grouping

### Literal expression

Literal value expressions are introduced in "[Representing
Code](representing-code.html)".
字面量值表达式在“表示代码”中介绍过。

^code expr-literal

### Logical expression

The logical `and` and `or` operators are introduced in "[Control
Flow](control-flow.html#logical-operators)".
逻辑运算符`and`和`or`在“控制流”中介绍过。

^code expr-logical

### Set expression

Property assignment, or "set" expressions are introduced in
"[Classes](classes.html#properties-on-instances)".
属性赋值，或者叫“set”表达式，在“类”中介绍过。

^code expr-set

### Super expression

The `super` expression is introduced in
"[Inheritance](inheritance.html#calling-superclass-methods)".
`super`表达式在“继承”中介绍过。

^code expr-super

### This expression

The `this` expression is introduced in "[Classes](classes.html#this)".
`this`表达式在“类”中介绍过。

^code expr-this

### Unary expression

Unary operators are introduced in "[Representing Code](representing-code.html)".
一元运算符在“表示代码”中介绍过。

^code expr-unary

### Variable expression

Variable access expressions are introduced in "[Statements and
State](statements-and-state.html#variable-syntax)".
变量访问表达式在“语句和状态”中介绍过。

^code expr-variable

## Statements  语句

Statements form a second hierarchy of syntax tree nodes independent of
expressions. We add the first couple of them in "[Statements and
State](statements-and-state.html)".
语句形成了独立于表达式的第二个语法树节点层次。我们在“声明和状态”中添加了前几个。

^code stmt

### Block statement

The curly-braced block statement that defines a local scope is introduced in
"[Statements and State](statements-and-state.html#block-syntax-and-semantics)".
在“语句和状态”中介绍过的花括号块语句，可以定义一个局部作用域。

^code stmt-block

### Class statement

Class declarations are introduced in, unsurprisingly,
"[Classes](classes.html#class-declarations)".
类声明是在“类”中介绍的，毫不意外。

^code stmt-class

### Expression statement

The expression statement is introduced in "[Statements and
State](statements-and-state.html#statements)".
表达式语句在“语句和状态”中介绍过。

^code stmt-expression

### Function statement

Function declarations are introduced in, you guessed it,
"[Functions](functions.html#function-declarations)".
函数声明是在“函数”中介绍的。

^code stmt-function

### If statement

The `if` statement is introduced in "[Control
Flow](control-flow.html#conditional-execution)".
`if`语句在“控制流”中介绍过。

^code stmt-if

### Print statement

The `print` statement is introduced in "[Statements and
State](statements-and-state.html#statements)".
`print`语句在“语句和状态”中介绍过。

^code stmt-print

### Return statement

You need a function to return from, so `return` statements are introduced in
"[Functions](functions.html#return-statements)".
你需要一个函数才能返回，所以`return`语句是在“函数”中介绍的。

^code stmt-return

### Variable statement

Variable declarations are introduced in "[Statements and
State](statements-and-state.html#variable-syntax)".
变量声明在“语句和状态”中介绍过。

^code stmt-var

### While statement

The `while` statement is introduced in "[Control
Flow](control-flow.html#while-loops)".
`while`语句在“控制流”中介绍过。

^code stmt-while


