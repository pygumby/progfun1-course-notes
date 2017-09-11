# Notes on "Functional Programming Principles in Scala"

## Overview
These are the notes I took while participating Martin Odersky's Coursera course "Functional Programming Principles in Scala". This course can be found at https://www.coursera.org/learn/progfun1/.

## Credit
All the credit goes to the author of the course, Prof. Dr. Martin Odersky ([@odersky](https://github.com/odersky)).

## Week 1 -- Functions & Evaluation

### Lecture 1.1 -- Programming Paradigms

- Scala is a great tool to learn functional programming and later integrate it with more traditional object-oriented programming.
- Main programming paradigms are imperative, functional and logic programming. Object-oriented programming can be integrated well with all of those.
- Imperative programming is about mutable variables, assignments, and control structures (e.g., `if-then-else`).
- These strongly correspond to the abstractions of the Von Neumann model: Mutable variables correspond to memory cells, control structures to jumps, etc.
- Problem: How can we avoid conceptualizing programs word by word? How to scale up?
- A mathematical theory consists of data types, operations on them, and laws describing the relationship between values and operations. There is no notion of mutability whatsoever.
- Theories lead to a programming style in which operators are expressed as functions, mutations are avoided, and in which there are powerful ways for the abstraction and composition of functions.
- In a restricted sense, functional programming means programming without mutable variables and imperative control structures. In a wider sense, it means focusing on functions.
- In a functional programming language, functions are first-class citizens.
  - They can be defined anywhere.
  - They can be passed as parameters and returned as results.
  - There are operators to compose them.
- Why functional programming?
  - It's simpler to reason about programs.
  - There is better modularity.
  - It's good for exploiting parallelism.

### Lecture 1.2 -- Elements of Programming

+ Every non-trivial languages provides primitive expressions, ways to combine those into larger expressions, and ways to abstract expressions, i.e., naming them.

+ Evaluation of a non-primitive expression:

  1. Consider the leftmost operator. (This is subject to precedence rules.)
  2. Evaluate the operator's operands. (This is usually done left before right.)
  3. Apply the operator to the operands.

  The evaluation of a name is simply replacing it with the right-hand side of its definition. The process of evaluation comes to an end once a value is yielded, e.g., a number.

+ Evaluation of a function application (call-by-value):

  1. Evaluate the arguments, from left to right.
  2. Replace the application with the functions right-hand side (it's body), and, at the same time, replace the formal parameters with the evaluated arguments.

+ This scheme is known as the substitution model. The idea is that evaluation simply reduces an expression to a value. This can be done as long as the expression does not contain side effects.

+ The substitution model is formalized in the lambda calculus.

+ Not every expression reduces to a value in a finite number of steps:

  ````scala
  def loop: Int = loop
  loop
  ````

+ As an alternative, one could change the evaluation of function application by replacing the formal parameters in the function's body with un-reduced arguments (call-by-name).

+ Call-by-value and call-by-name reduce to the same values as long as the reduced expression consists of pure functions and both expressions terminate.

+ Call-by-value's advantage is that each argument is only evaluated once, call-by-name's advantage is that unused arguments are not evaluated.

### Lecture 1.3 -- Evaluation Strategies and Termination

+ If call-by-value evaluation of an expression e terminates, then call-by-name evaluation of e terminates, too. This is not true in the other direction.

+ The following function application only terminates with call-by-name:

  ````scala
  def first(x: Int, y: Int) = x
  first(1, loop) // Using the definition of `loop` from above
  ````

+ Scala uses call-by-value by default, as it is often exponentially more efficient. However, when a parameter type is prefixed with `=>`, call-by-name is used.

  ````scala
  def constOne(x: Int, y: => Int) = 1
  constOne(1 + 2, loop) // Works
  ````

### Lecture 1.4 -- Conditionals and Value Definitions

+ Unlike Java, Scala's `if-then-else` is not a statement but an expression.

  ````scala
  def abs(x: Int) = if (x >= 0) x else -x
  ````

+ Consider Boolean expressions such as `true && e` . `&&` and `||` don't always need their right operand to be evaluated. They use "short-circuit evaluation".

+ Definitions can be "by-name" and "by-value", too. In the former case, the right-hand side is evaluated on each invocation, in the latter case, the right-hand side is evaluated once, at the point of definition.

  ````scala
  def x = 2 // "by-name"
  val x = 2 // "by-value"
  ````

+ Exercise: Write functions `and` and `or` such that for all argument expressions `x` and `y` 

  ````scala
  and(x, y) == x && y
  or(x, y)  == x || y
  ````

  holds true. Do not use `&&` and `||` in your implementation.

  ````scala
  def and(x: Boolean, y: => Boolean) =
    if (x) y else false
  ````

  `or` is omitted for brevity.

### Lecture 1.5 -- Example: Square Roots with Newton's Method

+ In the following, note that `sqrtIter` is recursive, its right-hand side calls itself. Recursive functions need an explicit return type in Scala.

+ Here is the source code of the example:

  ````scala
  def abs(x: Double) = if (x > 0) x else -x

  def sqrtIter(guess: Double, x: Double): Double =
    if (isGoodEnough(guess, x)) guess
    else sqrtIter(improve(guess, x), x)

  def isGoodEnough(guess: Double, x: Double) =
    abs(guess * guess - x) / x < 0.001

  def improve(guess: Double, x: Double) =
    (guess + x / guess) / 2

  def sqrt(x: Double) =
   sqrtIter(1.0, x)
  ````

### Lecture 1.6 -- Blocks and Lexical Scope

+ We want to split up a task into many small functions while avoiding "namespace pollution". With regards to the previous example, we can do so by putting the auxiliary function inside of `sqrt`.

  ````scala
  def sqrt(x: Double) = {
    def sqrtIter(guess: Double, x: Double): Double =
      if (isGoodEnough(guess, x)) guess
      else sqrtIter(improve(guess, x), x)

    def isGoodEnough(guess: Double, x: Double) =
      abs(guess * guess - x) / x < 0.001

    def improve(guess: Double, x: Double) =
      (guess + x / guess) / 2

    sqrtIter(1.0, x)
  }
  ````

+ A block is delimited by braces. It contains a sequence of definitions and/or expressions. The last element in it is an expression defining the block's value. Blocks are themselves expressions.

+ Definitions inside a block are only visible from inside it. They shadow definitions from outside it.

+ The value of `result` is `16`.

  ````scala
  val x = 0
  def f(y: Int) = y + 1
  val result = {
    val x = f(3)
    x * x
  } + x
  ````

+ Since definitions of outer blocks are visible as long as they are not shadowed, we can further simplify `sqrt`:

  ````scala
  def sqrt(x: Double) = {
    def sqrtIter(guess: Double): Double =
      if (isGoodEnough(guess)) guess
      else sqrtIter(improve(guess))

    def isGoodEnough(guess: Double) =
      abs(guess * guess - x) / x < 0.001

    def improve(guess: Double) =
      (guess + x / guess) / 2

    sqrtIter(1.0)
  }
  ````

+ In Scala, semicolons are optional in most cases. However, multiple statements on one line must be separated by semicolons. Multi-line expressions containing infix operators should either be put into parentheses or intermediate lines should end with the operator, signaling the compiler that the expression is not yet finished.

### Lecture 1.7 -- Tail Recursion

+ Consider the following definition of Euclid's algorithm `gcd` to compute the greatest common divisor of two numbers as well as the evaluation of `gcd(14, 21)`:

  ````scala
  def gcd(a: Int, b: Int): Int = 
    if (b == 0) a else gcd(b, a % b)

  gcd(14, 21) // is evaluated as follows:
  // if (21 == 0) 14 else gcd(21, 14 % 21)
  // if (false) 14 else gcd(21, 14 % 21)
  // gcd(21, 14 % 21)
  // gcd(21, 14)
  // if (14 == 0) 21 else gcd(14, 21 % 14)
  // ...
  // gcd(14, 7)
  // ...
  // gcd(7, 0)
  // if (0 == 0) 7 else gcd(0, 7 % 0)
  // 7
  ````

+ To the contrary, consider the following definition of `factorial` as well as the evaluation of `factorial(4)`:

  `````scala
  def factorial(n: Int): Int =
    if (n == 0) 1 else n * factorial(n - 1)

  factorial(4) // is evaluated as follows:
  // if (4 == 0) 1 else 4 * factorial(4 - 1)
  // ...
  // 4 * factorial(3)
  // ...
  // 4 * (3 * factorial(2))
  // ...
  // 4 * (3 * (2 * factorial(1)))
  // ...
  // 4 * (3 * (2 * (1 * factorial(0)))
  // ...
  // 4 * (3 * (2 * (1 * 1)))
  // ...
  // 120
  `````

+ If a function's last action is to call itself, its stack frame can be used again. This is called tail recursion. Unlike `factorial`, `gcd` is tail recursive.

+ If a function's last action is calling a function (maybe itself), one stack frame would be sufficient for both. Those calls are called tail calls.

+ In Scala, only tail calls to the current function are optimized. This requires the `@tailrec` annotation.

  ````scala
  @tailrec
  def gcd(a: Int, b: Int): Int =
    // ...
  ````

+ Here is a tail-recursive version of `factorial`:

  ````scala
  def factorial(n: Int): Int = {
    def loop(acc: Int, n: Int): Int = 
      if (n == 0) acc
      else loop(acc * n, n - 1)
    loop(1, n)
  }
  ````

## Week 2 -- Higher-Order Functions 

### Lecture 2.1 -- Higher-Order Functions

+ Functional programming languages treat functions as "first-class values". This means that they can be received as arguments and returned as results. "Higher-order functions" are functions that do just that.

+ `sum` is an example of a function that takes another function as an argument.

  ````scala
  def sum(f: Int => Int, a: Int, b: Int): Int =
    if (a > b) 0
    else f(a) + sum(f, a + 1, b)

  def sumInts(a: Int, b: Int) =       sum(id, a, b)
  def sumCubes(a: Int, b: Int) =      sum(cube, a, b)
  def sumFactorials(a: Int, b: Int) = sum(fact, a, b)

  def id(x: Int): Int = x
  def cube(x: Int): Int = x * x * x
  def fact(x: Int): Int = if (x == 0) 1 else fact(x - 1)
  ````

+ `A => B` denotes the type of a function that has a parameter of type `A` and returns a `B`. Therefore, the type of `sum` is `Int => Int`.

+ As it is sometimes tedious to define and name functions using `def`, we would like to write them like literals -- without giving them a name. Anonymous functions enable this. Here is an example:

  ````scala
  (x: Int, y: Int) => x + y
  ````

  The type of a parameter can be omitted if it is inferred by the compiler.

+ An anonymous function `(x1: T1, ..., xn: Tn) => E` can always be expressed as `def f(x1: T1, ..., xn ... Tn) = E; f`. Therefore, anonymous functions are said to be syntactic sugar.

+ `sumInts` and `sumCubes` can be shortened using anonymous functions:

  ````scala
  def sumInts(a: Int, b: Int)  = sum(x => x, a, b)
  def sumCubes(a: Int, b: Int) = sum(x => x * x * x, a, b)
  ````

### Lecture 2.2 -- Currying

+ Note that above, `sumInts` and `sumCubes` simply pass `a` and `b` through. We can do better:

  ````scala
  def sum(f: Int => Int): (a: Int, b: Int) => Int = {
    def sumF(a: Int, b: Int): Int = 
      if (a > b) 0
      else f(a) + sumF(a + 1, b)
    sumF
  }

  def sumInts  = sum(x => x)         // Exemplary application: sumInts(1, 10)
  def sumCubes = sum(x => x * x * x)
  ````

  `sum` is now a function that returns another function.

+ The "middlemen" `sumInts` and `sumCubes` can be avoided. We can simply call `sum(cube)(1, 10)`, where `sum(cube)` returns the sum of cubes function and is therefore equivalent to `sumCubes`. Then the resulting function is applied to `(1, 10)`.

+ Function application associates to the left.

  ````
  sum(cube)(1, 10) == (sum(cube))(1, 10)
  ````

+ This way of defining functions is so useful, it got its own syntax in Scala. The following definitions of `sum` is equivalent to the previous one using `sumF`:

  ````scala
  def sum(f: Int => Int)(a: Int, b: Int): Int = 
    if (a > b) 0 else f(a) + sum(f)(a + 1, b)
  ````

+ The expansion of multiple parameter lists is formalized like this: A function `def f(args1)...(argsn) = E` is equivalent to a function `def f(args1)...(argsn-1) = { def g(argsn) = E; g }`, or, for short, `def f(args1)...(argsn-1) = (argsn) => E`.

+ By repeating the process n times, `def f(args1)...(argsn-1)(argsn) = E` is shown to be equivalent to `def f = (args1 => (args2 => ... (argsn => E) ... ))`.

+ This style of function definition and applications is called "currying", named after the logician Haskell Brooks Curry.

+ The type of `def sum(f: Int => Int)(a: Int, b: Int): Int = ...` is `(Int => Int) => (Int, Int) => Int`.

+ Function types associate to the right.

  ````scala
  // Int => Int => Int == Int => (Int => Int)
  ````

### Lecture 2.3 -- Example: Finding Fixed Points

+ A number is called a fixed point of a function `f` if `f(x) = x`.  For example, `2` is a fixed point of `x => 1 + (x / 2)^2` as `1 + (2 / 2)^2` is `2`.

+ For some functions `f`, we can locate fixed points by starting with an initial estimate `x` (e.g., `1`) and then repeatedly applying `f` to it until the results do not vary anymore. This works for the previously mentioned function.

  ````
  x, f(x), f(f(x)), ...
  ````

+ This leads to this function for finding a function's fixed point:

  ````scala
  val tolerance = 0.0001
  def isCloseEnough(x: Double, y: Double) =
    abs((x - y) / x) / x < tolerance
  def fixedPoint(f: Double => Double)(firstGuess: Double) = {
    def iterate(guess: Double): Double = {
      val next = f(guess)
      if (isCloseEnough(guess, next)) next
      else iterate(next)
    }
    iterate(firstGuess)
  }
  ````

+ Here is a specification of the `sqrt` function: 

  + `sqrt(x)` = the number `y` so that `y * y = x`
  + By dividing both sides of `y * y = x` with `y`, we get `y = x / y`.
  + Therefore, for a given `x`, `sqrt(x)` is a fixed point of a function `y => x / y`.
    + Another way to look at it: Dividing a number by its square root results in its square root again. Example: Let `x` be `9`. Applying `sqrt(9)`, i.e., `3`, to `y => 9 / y` results in `sqrt(9)` again.



+ Can we calculate `sqrt(x)` by iterating towards a fixed point of `y => x / y`?

  ````scala
  def sqrt(x: Double) =
    fixedPoint(y => x / y)(1.0)
  ````

  Unfortunately, no. When adding `println(next)` after the line `val next = // ... ` in the body of the `iterate` function, `sqrt(2)` prints :

  ````
  2.0
  1.0
  2.0
  ...
  ````

  + This can easily be reproduced using pen and paper.

+ These oscillation can be avoided by averaging successive values:

  ````scala
  def sqrt(x: Double) =
    fixedPoint(y => (y + x / y) / 2)(1.0)
  ````

  `sqrt(2)` now prints:

  `````
  1.5
  1.4166666666666665
  1.4142156862745097
  `````

  + The function `y => (y + x / y) / 2` still has the property that for a given `x`, `sqrt(x)` is a fixed point of it. Example: Let `x` be `9`. Applying `sqrt(9)` , i.e., `3`, to `y => (y + 9 / y) / 2` results in `sqrt(9)` again.

+ This technique of stabilizing by averaging can be factored out into its own function:

  ````scala
  def averageDamp(f: Double => Double)(x: Double) = 
    (x + f(x)) / 2
  ````

  `sqrt` can then be defined as follows:

  ````scala
  def sqrt(x: Double) =
    fixedPoint(averageDamp(y => x / y))(1.0)
  ````

### Lecture 2.4 -- Scala Syntax Summary

+ In the following, the context-free syntax of the language elements seen so far are given in Extended Backus-Naur form (EBNF), where

  + `|` denotes an alternative,
  + `[...]` and option (0 or 1),
  + `{...}` a repetition (0 or more).

+ Types:

  ````
  Type         = SimpleType | FunctionType
  FunctionType = SimpleType '=>' Type
               | '(' [Types] ')' '=>' Type
  SimpleType   = Ident
  Types        = Type {',' Type}
  ````

  A type can be:

  + A numeric type, e.g., `Int`, `Double`
  + The `Boolean` type
  + The `String` type
  + A function type, e.g., `Int => Int`, `(Int, Int) => Int`

+ Expressions:

  ````
  Expr         = InfixExpr | FunctionExpr
               | if '(' Expr ')' Expr else
  InfixExpr    = PrefixExpr | InfixExpr Operator InfixExpr
  Operator     = ident
  PrefixExpr   = ['+', '-', '!', '~'] SimpleExpr
  SimpleExpr   = ident | literal | SimpleExpr '.' ident | Block
  FunctionExpr = Bindings '=>' Expr
  Bindings     = ident [':' SimpleType]
               | '(' [Binding {',' Binding}] ')'
  Binding      = ident [':' Type]
  Block        = '{' { Def ';' } Expr '}'
  ````

  An expression can be:

  + An identifier, e.g., `x`,  `isGoodEnough`
  + A literal, e.g., `0`, `"abc"`
  + A function application, e.g., `sqrt(x)`
  + An operator application, e.g., `-x`, `y + x`
  + A selection, e.g., `math.abs`
  + A conditional expression, e.g., `if (x < 0) -x else x`
  + A block, e.g., `{ val x = math.abs(y); x * 2 }`
  + An anonymous function, e.g., `x => x + 1`

+ Definitions:

  ````
  Def        = FunDef | ValDef
  FunDef     = def ident { '(' [ Parameters ] ')' } [':' Type] '=' Expr
  ValDef     = val ident [ ':' Type ] '=' Expr
  Parameter  = ident ':' [ '=>' ] Type
  Parameters = Parameter { ',' Parameter }
  ````

  A definition can be:

  + A function definition, e.g., `def square(x: Int) = x * x`
  + A value definition, e.g., `val y = square(2)`

  A parameter can be:

  + A call-by-value parameter, e.g., `(x: Int)`
  + A call-by-name parameter, e.g., `(y: => Double)`
