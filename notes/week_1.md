# progfun1-course-notes

**Week 1 - Functions & Evaluation**

---

* [Lecture 1.1 - Programming Paradigms](#lecture-11---programming-paradigms)
* [Lecture 1.2 - Elements of Programming](#lecture-12---elements-of-programming)
* [Lecture 1.3 - Evaluation Strategies and Termination](#lecture-13---evaluation-strategies-and-termination)
* [Lecture 1.4 - Conditionals and Value Definitions](#lecture-14---conditionals-and-value-definitions)
* [Lecture 1.5 - Example: Square Roots with Newton's Method](#lecture-15---example-square-roots-with-newtons-method)
* [Lecture 1.6 - Blocks and Lexical Scope](#lecture-16---blocks-and-lexical-scope)
* [Lecture 1.7 - Tail Recursion](#lecture-17---tail-recursion)

---

## Lecture 1.1 - Programming Paradigms

+ Scala is a great tool to learn functional programming and later integrate it with more traditional object-oriented programming.

+ The main programming paradigms are imperative, functional and logic programming. Object-oriented programming can be integrated well with all of those.
- Imperative programming is about mutable variables, assignments, and control structures (e.g., `if-then-else`).

+ These concepts strongly correspond to the abstractions of the Von Neumann model: Mutable variables correspond to memory cells, control structures to jumps, etc.

+ Problem: How can we avoid conceptualizing programs word by word? How to scale up?

+ A mathematical theory consists of data types, operations on them, and laws describing the relationship between values and operations. There is no notion of mutability whatsoever.

+ Theories lead to a programming style in which operators are expressed as functions, mutations are avoided, and in which there are powerful ways for the abstraction and composition of functions.

+ In a restricted sense, functional programming means programming without mutable variables and imperative control structures. In a wider sense, it means focusing on functions.

+ In a functional programming language, functions are first-class citizens.
  
  + They can be defined anywhere.
  + They can be passed as parameters and returned as results.
  + There are operators to compose them.

+ Why functional programming?
  
  + It's simpler to reason about programs.
  + There is better modularity.
  + It's good for exploiting parallelism.

## Lecture 1.2 - Elements of Programming

+ Every non-trivial language provides primitive expressions, ways to combine those into larger expressions, and ways to abstract expressions, i.e., to name them.

+ Evaluation of a non-primitive expression:

  1. Consider the leftmost operator. (This is subject to precedence rules.)
  2. Evaluate the operator's operands. (This is usually done left before right.)
  3. Apply the operator to the operands.

  The evaluation of a name is simply replacing it with the right-hand side of its definition. The process of evaluation comes to an end once a value is yielded, e.g., a number.

+ Evaluation of a function application (call-by-value):

  1. Evaluate the arguments, from left to right.
  2. Replace the application with the function's right-hand side (it's body), and, at the same time, replace the formal parameters with the evaluated arguments.

+ This scheme is known as the substitution model. The idea is that evaluation simply reduces an expression to a value. This can be done as long as the expression does not contain side effects.

+ The substitution model is formalized in the lambda calculus.

+ Not every expression reduces to a value in a finite number of steps:

  ````scala
  def loop: Int = loop
  loop
  ````

+ As an alternative, one could change the evaluation of function application by replacing the formal parameters in the function's body with un-reduced arguments (call-by-name).

+ Call-by-value and call-by-name reduce to the same values as long as the reduced expression consists of pure functions and both expressions terminate.

+ Call-by-value's advantage is that each argument is only evaluated once, call-by-name's advantage is that unused arguments are not evaluated at all.

## Lecture 1.3 - Evaluation Strategies and Termination

+ If the call-by-value evaluation of an expression `e` terminates, then the call-by-name evaluation of `e` terminates, too. This is not true in the other direction.

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

## Lecture 1.4 - Conditionals and Value Definitions

+ Unlike Java, Scala's `if-then-else` is not a statement but an expression.

  ````scala
  def abs(x: Int) = if (x >= 0) x else -x
  ````

+ Consider Boolean expressions such as `true && e`. `&&` and `||` don't always need their right operand to be evaluated. They use "short-circuit evaluation".

+ Definitions can be "by-name" and "by-value", too. In the former case, the right-hand side is evaluated on each invocation, in the latter case, the right-hand side is evaluated only once, at the point of definition.

  ````scala
  def x = 2 // "by-name"
  val x = 2 // "by-value"
  ````

+ Exercise: Write functions `and` and `or` such that for all argument expressions `x` and `y` 

  ````scala
  and(x, y) == x && y
  or(x, y)  == x || y
  ````

  holds true. Do not use `&&` and `||` in your implementation. Here is one solution:

  ````scala
  def and(x: Boolean, y: => Boolean) =
    if (x) y else false
  ````

  An implementation of `or` is omitted for brevity.

## Lecture 1.5 - Example: Square Roots with Newton's Method

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

## Lecture 1.6 - Blocks and Lexical Scope

+ We want to split up a task into many small functions while avoiding "namespace pollution". With regards to the previous example, we can do so by putting the auxiliary functions inside of `sqrt`.

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

+ In the following, the value of `result` is `16`.

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

## Lecture 1.7 - Tail Recursion

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
