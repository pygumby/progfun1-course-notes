# Notes on "Functional Programming Principles in Scala"

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

### Lecture 2.5 -- Functions and Data

+ In this section, we design a package for doing rational arithmetic. A rational number `x / y` is represented by its numerator `x` and its denominator `y` (both integers).

+ We define the following class `Rational` , introducing two entities, i.e., a new type `Rational` and a constructor `Rational`. Even tough they have the same name, there is no conflict, as Scala keeps the names of types and value in different namespaces.

+ Instances of a class are called objects. They are created by prefixing the application of the constructor with `new `. Their two members `numer` and `denom` are selected using an infix operator.

  ````scala
  val x = new Rational(1, 2)
  x.numer
  ````

+ In a first approach, arithmetic functions implementing standard rules are implemented as

  top-level functions that take and return `Rational`s, e.g.:

  ````scala
  def addRational(r: Rational, s: Rational): Rational = // ...
  ````

+ Then, the functions operating on the data abstraction are packaged in the abstraction itself -- as methods:

  ````scala
  class Rational(x: Int, y: Int) {
    def numer = x
    def denom = y
    
    def add(that: Rational) =
      new Rational(
        numer * that.denom +  that.numer * denom,
        denom * that.denom)
    
    def neg =
      new Rational(-numer, denom)
    
    def sub(that: Rational) =
      add(that.neg)
    
    override def toString =
      numer + "/" + denom
  }

  val x = new Rational(1, 3)
  val y = new Rational(5, 7)
  val z = new Rational(3, 2)
  x.numer // 1
  x.denom // 3
  x.add(y) // 22/1
  println(x.sub(y).sub(z)) // -79/42
  ````

### Lecture 2.6 -- More Fun with Rationals

+ In the previous lecture, rationals have not always been represented in their simplest form. They can be reduced to their smallest numerator and denominator by dividing both with their greatest common divisor.

+ In order to adhere to the DIY ("Don't repeat yourself!") principle, we do not implement this in each operation but in the constructor. Three ways of doing so are shown. However, clients of the `Rational` class always experience the same behavior.

  ````scala
  // Approach 1
  // `gcd` is calculated immediately, its value can be re-used by `numer` and `denom`.
  class Rational(x: Int, y: Int) {
    // `private` members can only be accessed from inside the class
    private def gcd(a: Int, b: Int): Int =
      if (b == 0) a else gcd(b, a % b)
    private val g = gcd(x, y)
    def numer = x / g
    def denom = y / g
    // ...
  }

  // Approach 2
  // This can be advantageous if `numer` and `denom` are called infrequently.
  class Rational(x: Int, y: Int) {
    private def gcd(a: Int, b: Int): Int =
      if (b == 0) a else gcd(b, a % b)
    def numer = x / gcd(x, y)
    def denom = y / gcd(x, y)
    // ...
  }

  // Appraoch 3
  // This can be advantageous if `numer` and `denom` are called often.
  class Rational(x: Int, y: Int) {
    private def gcd(a: Int, b: Int): Int =
      if (b == 0) a else gcd(b, a % b)
    // As `val`s, `numer` and `denom` are computed only once.
    val numer = x / gcd(x, y)
    val denom = y / gcd(x, y)
    // ...
  }
  ````

+ The ability to choose from different implementations without affecting clients is called data abstraction.

+ Inside a class, there is the self reference `this`, which represents the object on which a method is executed. A simple name `x`, referring to a class member, is an abbreviation of `x`.  `this` is used to implement the member functions `less` and `max` on the function `Rational`.

  ````scala
  class Rational(x: Int, y: Int) {
    // ...
    def less(that: Rational) =
      numer * that.denom < that.numer * denom
    
    def max(that: Rational) =
      if (this.less(that)) that else this
  }
  ````

+ The `Rational` class requires `denom` to be positive. This can be enforced like so:

  ````scala
  class Rational(x: Int, y: Int) {
    require(y > 0, ”denominator must be positive”)
    // ...
  }
  ````

+ The predefined function `require` takes a condition and an optional message string. If the condition is `false`, an `IllegalArgumentException` is thrown. There is also `assert`, which takes the same arguments as `require`, but throws an `AssertionError` whenever the condition is `false`. The two differ in intent:

  + `require` is used to enforce a requirement on the function caller.
  + `assert` is used to check the code of the function itself.

+ Scala introduces the primary constructor of the class implicitly. It takes the parameters of the class (`class Rational(x: Int, y: Int)`) and executes the statements in the class body, e.g., `require(y > 0, ”denominator must be positive”)`.

+ Auxiliary constructors may be declared. They are called `this`.

  ````scala
  class Rational(x: Int, y: Int) {
    def this(x: Int) = this(x, 1)
    // ...
  }
  ````

+ It would be a bad idea if we altered `Rational` so that numbers are kept unsimplified until they are printed, i.e., if we moved the simplification inside the `toString` method. Why? We are dealing with integers here, and we might exceed the maximal number.

### Lecture 2.7 -- Evaluation and Operators

+  The previously introduced model of evaluation based on the term of rewriting, the so-called substitution model can be extended to model classes and objects. Consider the following class definition:

  ````scala
  // The list of function parameters is optional.
  // Also, parameter types are left out for brevity.
  class C(x1, ..., xm) { ... def f(y1, ..., yn) = b ... }
  ````

  How is the following expression evaluated?

  ````scala
  new C(v1, ..., vm).f(w1, ..., wn)
  ````

  Inside the function's body,

  + `f`'s parameters `y1, ..., yn` are substituted by the arguments `w1, ..., wn`,
  + `f`'s parameters `v1, ..., vn ` are substituted by the arguments `w1, ..., wn`, and
  + the self reference `this` is substituted by the value of the object `new C(v1, ..., vn)`:

  ````
     new C(v1, ..., vm).f(w1, ..., wn)
  -> [w1/y1, ..., wn/yn][v1/x1, ..., vm/xm][new C(v1, ..., vm)/this]
     b
  =  b
  ````

+ Two more object rewriting examples are given:

  `````
     new Rational(1, 2).numer
  -> [1/x, 2/y][][new Rational(1, 2)/this]
     x
  =  1

     new Rational(1, 2).less(new Rational(2, 3))
  -> [1/x, 2/y][new Rational(2, 3)/that][new Rational(1, 2)/this]
     this.numer * that.denom < that.numer * this.denom
  =  new Rational(1, 2).numer * new Rational(2, 3).denom <
     new Rational(2, 3).numer * new Rational(1, 2).denom
  // ...
  =  1 * 3 < 2 * 2
  =  true
  `````

+ With integers, we can write `x + y`, however, with `Rational`s we must do `r.add(s)`. In Scala, we can eliminate using

  1. Infix notation
  2. Relaxed identifiers

+ Using infix notation, we can write `r add s` instead of `r.add(s)`. Any method with one parameter can be used like an infix operator.

+ In Scala, an identifier are not confined to being alphanumeric. Instead, they can be

  + Alphanumeric: Starting with a letter, followed by a sequence of letters or numbers. (`_` counts as a letter in this definition.)
  + Symbolic: Starting with an operator symbol, e.g., `+` or `-`, followed by other operator symbols.
  + Alphanumeric identifiers can also end in an underscore, followed by some operator symbols.

+ Using infix notation and relaxed identifiers, we can define the class `Rational` more naturally:

  `````scala
  class Rational(x: Int, y: Int) {
    // ...
    
    def <(that: Rational) =
      numer * that.denom < that.numer * denom
    
    def max(that: Rational) =
      if (this < that) that else this
    
    def +(r: Rational) =
      new Rational(
        numer * r.denom + r.numer * denom,
        denom * r.denom)
    
    def unary_- : Rational = 
      new Rational(-numer, denom)
    
    def -(that: Rational) = 
      this + -that
  }

  val x = new Rational(1, 2)
  val y = new Rational(1, 3)
  x * x + y * y // (x * x) + (y * y)
  `````

+ The precedence of an operator is determined by its first character. Here is a list of the characters in increasing order of precedence:

  ````
  (all letters)
  |
  ^
  &
  < >
  = !
  :
  + -
  * / %
  (all other special characters)
  ````

+ A parenthesized version of `a + b ^? c ?^ d less a ==> b | c` would be `((a + b) ^? (c ?^ d)) less ((a ==> b) | c)`.
