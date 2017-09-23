# Notes on "Functional Programming Principles in Scala"

## Week 4 -- Types and Pattern Matching

### Lecture 4.1 -- Objects Everywhere

+ In a pure object-oriented programming language, each value is an object. If this language is based on classes, the type of each value, i.e., object, would be a class.

  + In Scala, there are not just reference types but also primitive types and functions. Does this mean Scala is not purely object-oriented?
    + Conceptually, primitive types, e.g., `Int`, do not receive any special treatment and are used like other classes. The fact that, under the hood, `Int`s are represented as 32-bit integers, can be regarded as a mere optimization. The rest of this section elaborates on this point.
    + Functions are actually represented by classes, too, just like I have demonstrated at the end of lecture 3.2. Lecture 4.2 elaborates on classes representing functions. 

+ Scala's `Boolean` type maps to the JVM's primitive type `boolean`, however, one could easily implement a purely object-oriented `Boolean`  type from scratch.

  ````scala
  abstract class Boolean {
    def ifThenElse[T](thenExpr: => T, elseExpr: T): T

    def && (x: => Boolean): Boolean = ifThenElse(x, False)
    def || (x: => Boolean): Boolean = ifThenElse(True, x)
    def unary_! : Boolean           = ifThenElse(False, True)

    def == (x: Boolean): Boolean    = ifThenElse(x, !x)
    def != (x: Boolean): Boolean    = ifThenElse(!x, x)

    // In the following, we assume `False` < `True`:
    def < (x: Boolean): Boolean     = ifThenElse(False, x)
  }

  object True extends Boolean {
    def ifThenElse[T](thenExpr: => T, elseExpr: T): T =
      thenExpr
  }

  object False extends Boolean {
    def ifThenElse[T](thenExpr: => T, elseExpr: T): T =
      elseExpr
  }

  val x: Boolean = True || False
  val y: Boolean = !x == x
  val z: AnyVal = y.ifThenElse(84 / 2, false)
  ````

+ Analogously, a partial specification of such a class `Int` is given (but omitted here for brevity). While the operations on `Int`s are expressed as methods, the question is asked whether we can actually implement this class solely based on objects, i.e., not resorting back to primitive `int`s.

+ Regarding the previous question, a class `Nat` representing non-negative integers is implemented:

  ````scala
  abstract class Nat {
    def isZero: Boolean
    def predecessor: Nat
    def successor: Nat = new Succ(this)
    def +(that: Nat): Nat
    def -(that: Nat): Nat
  }

  object Zero extends Nat {
    def isZero = true
    def predecessor = throw new Error("Zero.predecessor")
    def +(that: Nat) = that
    def -(that: Nat) = if (that.isZero) this else throw new Error("Negative number")
  }

  class Succ(n: Nat) extends Nat {
    def isZero = false
    def predecessor = n
    def +(that: Nat) = new Succ(n + that)
    def -(that: Nat) = if (that.isZero) this else n - that.predecessor
  }
  ````

### Lecture 4.2 -- Functions as Objects

+ In Scala, function values are objects. `A => B` is just a shorthand for `scala.Function1[A, B]`, which is defined similar to what I defined at the end of lecture 3.2. To represent functions with two parameters, there is `Function2`, to represent those with three parameters, there is `Function2`, and so on.

+ This is what anonymous function definitions are expanded to:

  ````scala
  (x: Int) = x * x
  // ...is expanded to...
  {
    class AnonFun extends Function1[Int, Int] {
      def apply(x: Int) = x * x
    }
    new AnonFun
  }
  // ...or, for short, using anonymous class syntax
  new Function1[Int, Int] {
    def apply(x: Int) = x * x
  }
  ````

+ A function call such as `f(a, b)`, is expanded to `f.apply(a, b)`. Remember, `f` is the value of a class type!

+ Methods are no objects, if they were, then `apply` would be an instance of some `Function` class, and a call to it would be a call to its `apply` method, so we'd end up in an infinite loop. However, if some function `f` is used in a place where a `Function` type is expected, one is automatically provided. E.g., for a method `def f(x: Int): Boolean  `, `(x: Int) => f(x)` would be provided. In lambda calculus, this conversion is known as "eta expansion"

+ Given our `List` type, we could define an object of the same name, which provides the means to obtain a list of two elements, e.g., `1` and `2`, by writing `List(1, 2)`.

  ````scala
  object List {
    // List(1, 2) == List.apply(1, 2)
    def apply[T](x1: T, x2: T): List[T] =
      new Cons(x1, new Cons(x2, new Nil))
  }
  ````

### Lecture 4.3 -- Subtyping and Generics

+ We have already covered two forms of polymorphism, i.e., subtyping, which originated in object-oriented programming, and generics, which originated in functional programming.
+ As a reminder, subtyping allows us to pass a type where a base type was required, while generics enable types to be parameterized with other types.
+ In this lecture, we will look at how subtyping and generics interact. In this context, we will look at two main areas:
  + Bounds (Subjecting type parameters to subtype constraints)
  + Variance (How do parameterized types behave under subtyping?)

+ Type bounds

  + Consider a method `assertAllPos`, which takes  an `IntSet` and returns it if all elements are positive, otherwise it will throw an error.

  + What would its signature look like if we wanted it to reflect the following two equations:

    ````
    assertAllPos(Empty)    = Empy
    assertAllPos(NonEmpty) = Either NonEmpty or Exception
    ````

    In other words, we want the signature to reflect that if we provide an instance of `IntSet`'s subtype `Empty`, we will get an `Empty` back, and if we provide an instance of `IntSet`'s subtype `NonEmpty`, we get a `NonEmpty` back (if no exception has been thrown). In no case do we provide, say, a `Empty` and receive a `NonEmpty`. One way to express this is:

    ```` scala
    def assertAllPos[S <: IntSet](r: S): S
    ````

  + `<: IntSet` is an upper bound of the type parameter `S`. This notation is also used outside of type bounds. Generally, it means this:

    + `S <: T` means `S` is a subtype of `T`
    + `S >: T` means `S` is a supertype of `T`, or `T` is a subtype of `S`

  + Lower bounds can also be used: `[S >: NonEmpty]` introduces a type parameter `S` that can range only over supertypes of `NonEmpty`. Consequently, `S` could be one of `NonEmpty`, `IntSet`, `AnyRef`, or `Any`.

  + `S` can also be bound from below (`NonEmpty`) as well as from above (`IntSet`): `[S >: NonEmpty <: IntSet]` restricts `S` to the interval between `NonEmpty` and `IntSet`, which only contains `NonEmpty` and `IntSet`.

+ Covariance

  + Should `NonEmpty <: IntSet` imply `List[NonEmpty] <: List[IntSet]`?

  + Types for which this relationship holds are called covariant, because their subtyping relationship varies with the type parameter. For example, if `List` were covariant, `List[S]` would only be a subtype of `List[T]` if `S` was a subtype of `T`. Thus, the relationship of the two `List` types would depended on the relationship of the two type parameters.

  + Covariance does not always make sense. Consider this Java code snippet:

    ````java
    NonEmpty[] a = new NonEmpty[]{new NonEmpty(1, Empty, Empty)}
    // In Scala, this line wouldn't compile, as Scala arrays aren't covariant.
    IntSet[] b = a
    b[0] = Empty
    NonEmpty s = a[0]
    ````

    In this (compiling!) example, we assign an `Empty` to variable of type `NonEmpty`! At run time, assigning `Empty` to `b[0]` will cause an `ArrayStoreException` to be thrown. (Java stores a type tag for every array, so it knows that this assignment is malicious.) Thus, making arrays covariant produced a hole in the type system (assignment compiles) that had to be patched with run time checks (comparing against type tag).

  + The following principle ("The Liskov Substitution Principle") tells us when a type can be subtype of another:

    > If `A <: B`, then everything one can do with a value of type `B`, one should also be able to do with value of type `A`.

    How would this translate to the problem described above? Well, `NonEmpty[]` should not be a subtype of `IntSet[]`, because one can put `Empty` instances into `IntSet[]` but not into `NonEmpty[]`

### Lecture 4.4 -- Variance

+ In this lecture, we are going to cover the concept of variance, i.e., how subtyping relates to genericity.

+ In the previous lecture, it has been demonstrated how some types should be covariant whereas others shouldn't.

  + Generally, a type that permits its elements to be mutated should not be covariant, e.g. arrays.
  + Analogously, immutable types can be covariant, if certain restrictions on methods are met, e.g. lists.

+ Given a type `C[T]` as well as two types `A` and `B` with `A <: B`, there are three possible relationships between `C[A]` and `C[B]`:

  + `C` is covariant: `C[A] <: C[B]`
  + `C` is contravariant: `C[A] <: C[B]`
  + `C` is nonvariant: Neither `C[A]` nor `C[B]` are subtype of the other.

+ By annotating the type parameter, a type's variance can be declared:

  + `C` is covariant: `class C[+A]`
  + `C` is contravariant: `class C[-A]`
  + `C` is nonvariant: `class C[A]`

+ Consider the following two types `A` and `B`. According to the Liskov Subsitution Principle, should `A` be a subtype of `B` (`A <: B`), `B` be a subtype of `A` (`B <: A`), or should they be unrelated?

  ````scala
  type A = IntSet => NonEmpty
  type B = NonEmpty => IntSet
  ````

  Since `NonEmpty <: IntSet` is true, `A <: B` should hold.

  + Whenever we expect a function that takes a `NonEmpty` as argument, a function that takes an `IntSet` will satisfy this requirement.
  + Whenever we expect a function that returns an `IntSet`, a function that returns a `NonEmpty` does just that.
  + To put it in terms of the aforementioned principle, you can do with `A` everything you can do with `B`.

+ Generally, there is the following subtyping rule between function types: If `A2 <: A1` and `B1 <: B2`, then ` A1 => B1 <: A2 => B2 `. Thus, you can always do the following:

  ````scala
  val f: Function1[A2, B2] = new Function[A1, B1] { /* ... */ }
  ````

+ Since we now know that functions are contravariant in their argument type(s) and covariant in their return type(s), we can revise our `Function1` trait from before:

  ````scala
  trait Function1[-T, +U] {
    def apply(x: T): U
  }
  ````

+ We have, however, seen an example, i.e., `Array[+T]`, in which covariance in combination a certain method, i.e., `update(T)`, is unsound. You may want to refer back to the Java code snippet of lecture 4.3, in which updating the element of an array -- which are covariant in Java -- causes a run time error to be thrown. The problem of the code below is that the covariant type parameter `T` is the type of a method parameter.

  ````scala
  class Array[+T] {
    def update(x: T) // ...
  }
  ````

+ Thus, the Scala compiler enforces certain rules regarding variance annotations. Roughly, these variance checks are:

  + Covariant type parameters may only appear in method results.
  + Contravariant type parameters can only appear in method parameters.
  + Invariant type parameters can appear everywhere.

+ In the following, we are going to improve our `List` implementation. The first goal is to make `Nil` a singleton object, as there really is only one empty list. We can easily achieve this by changing its signature to `object Nil extends List[Nothing]`. However, a `Cons[T]` expects a `List[T]` as its second argument (`tail`). Thus, we cannot pass `Nil`, i.e., a `List[Nothing]`, as the second argument of, say, a `List[String]`. Sure, `Nothing <: String`, but `List[Nothing]` and `List[String]` have no subtyping relationship. Obviously, we have to make `List` covariant. Here's the revised code:

  ````scala
  trait List[+T] {
    def isEmpty: Boolean
    def head: T
    def tail: List[T]
  }

  class Cons[T](val head: T, val tail: List[T]) extends List[T] {
    def isEmpty: Boolean = false
  }

  object Nil extends List[Nothing] {
    def isEmpty: Boolean = true
    def head: Nothing = throw new NoSuchElementException("Nil.head")
    def tail: Nothing = throw new NoSuchElementException("Nil.tail")
  }

  // Now, we can do this:
  val l1: List[String] = Nil
  val l2: List[String] = new Cons("Hello, world.", Nil)
  ````

+ Lastly, we want add a `prepend` method to our `List` class. However, the following does not compile, as it violates the one of the variance rules enforced by the compiler, i.e., covariant type parameters may only appear in method results.

  ````scala
  // Caution, this does not compile!
  trait List[+T] {
    def prepend(elem: T): List[T] = new Cons(elem, this)
  }
  ````

  The compiler is actually right to reject the previous definition, as it violates the Liskov Substitution Principle: Not everything you can do with a `List[IntSet]` you can do with a `List[NonEmpty]`, e.g.,  `prepend` an `Empty`. Thus, `List[NonEmpty]` cannot be a subtype of `List[IntSet]`.

+ How can we make `prepend`, which is a natural list on immutable lists, variance-correct? By the use of a lower bound:

  `````scala
  def prepend[U >: T](elem: U): List[U] = new Cons(elem, this)
  `````

  This compiles, because:

  + Covariant type parameters may appear in lower bounds of method type parameters.
  + Contravariant type parameters may appear in upper bounds of method type parameters.

+ What is the result type of `def f(xs: List[NonEmpty], x: Empty) = xs prepend x`? This function definition compiles, so we know the argument for the `prepend` method's type parameter `U` must satisfy two conditions:

  + `U >: T` must hold, and since `T` is `NonEmpty`, `U ` has to be either `NonEmpty`, `IntSet`, `AnyRef`, or `Any`.
  + `elem`, which is of type `Empty`, must conform to `U`.

  The type inferencer chooses `U` to be `IntSet`.

   + `IntSet` is a superclass of `NonEmpty`.
   + By being of type `Empty`, `elem` is also a `IntSet`.

  Thus, the return type of the function is `List[IntSet]`.

### Lecture 4.5 -- Decomposition

+ Decomposition is an important problem in programming. Suppose we have a hierarchy of classes and want to build tree-like data structures from the instances of these classes. How would we find out what kinds of elements are in this tree?

+ To find out more about this, let's start with a simple example: A small interpreter for arithmetic expressions. Expressions are represented as a class hierarchy, with a base trait `Expr` and two subclasses, `Number` and `Sum`.

  ````scala
  trait Expr {
    def isNumber: Boolean
    def isSum: Boolean
    def numValue: Int
    def leftOp: Expr
    def rightOp: Expr
  }

  class Number(n: Int) extends Expr {
    def isNumber: Boolean = true
    def isSum: Boolean = false
    def numValue: Int = n
    def leftOp: Expr = throw new Error("Number.leftOp")
    def rightOp: Expr = throw new Error("Number.rightOp")
  }

  class Sum(e1: Expr, e2: Expr) extends Expr {
    def isNumber: Boolean = false
    def isSum: Boolean = true
    def numValue: Int = throw new Error("Sum.numValue")
    def leftOp: Expr = e1
    def rightOp: Expr = e2
  }
  ````

  Now, we can easily write an evaluation function.

  ````scala
  def eval(e: Expr): Int = {
    if (e.isNumber) e.numValue
    else if (e.isSum) eval(e.leftOp) + eval(e.rightOp)
    else throw new Error("Unknown expression " + e)
  }
  ````

  However, it is tedious to write out all these classification methods, e.g., `isNumber`, as well as accessor functions, e.g., `numValue`. Furthermore,  if we want to add new expressions, e.g., the ones below, we need to add classification and accessor methods to all classes defined above.

  ````scala
  class Prod(e1: Expr, e2: Expr) extends Expr
  class Var(x: String) extends Expr
  ````

  Actually, in order to integrate `Prod` and `Var` into the hierarchy, 25 new methods would need to be defined. In fact, if we continued to add new expressions, there would be a quadratic increase in methods we would need to define.

+ A "non-solution" to this problem would be to use type tests and type casts. These can be done using the following methods defined in class `Any`:

  ````scala
  def isInstanceOf[T]: Boolean // Checks whether this object's type conforms to `T`
  def asInstanceOf[T]: T       // Treats this object as an instance of type `T`,
                               // throws a `ClassCastException` if it isn't. 
  ````

  Here is a definition of the `eval` method using type tests and type casts:

  `````scala
  def eval(e: Expr): Int =
    if (e.isInstanceOf[Number])
      e.asInstanceOf[Number].numValue
    else if (e.isInstanceOf[Sum])
      eval(e.asInstanceOf[Sum].leftOp) +
      eval(e.asInstanceOf[Sum].rightOp)
    else throw new
      Error("Unknown expression " + e)
  `````

  On the plus side, with this approach, there is no need for classification methods at all, and the accessor methods are only for the classes where the value in question is defined. However, the usage of type tests and type casts is very low-level. Generally, relying on them is error-prone, since, at run time, we never know whether a type cast will succeed or not. (Above, we can statically assure that the type casts will not fail, since we guarded them with type checks, but still.)

+ Another -- better -- approach would be object-oriented decomposition. Suppose all we wanted to do is evaluate expressions evaluate expressions.

  ````scala
  trait Expr {
    def eval: Int
  }

  class Number(n: Int) extends Expr {
    def eval: Int = n
  }

  class Sum(e1: Expr, e2: Expr) extends Expr {
    def eval: Int = e1.eval + e2.eval
  }
  ````

  However, this approach is limited. Say we want to simplify expressions using this rule:

  ````
  a * b + a * c -> a * (b + c)
  ````

  This is a non-local simplification. It cannot be encapsulated into a method of one object. In order to inspect our tree-like data structure of `Expr` instances, we need to go back to square one, using classification and accessor methods.

### Lecture 4.6 -- Pattern Matching

+ Recap

  + We are still trying to find a general and convenient way to access objects in a extensible class hierarchy.

  + Here is what we already tried:

    + Classification and accessor methods: quadratic "explosion" of number of methods to be defined for each new subtype of `Expr`
    + Type tests and type casts: unsafe, low-level
    + Object-oriented decomposition: does not work for every problem to be solved, all classes need to be touched to add a new method.

  + Observation: The purpose of the classification and accessor methods can be viewed as reversing the construction process:

    + What class was used?
    + What were the arguments to the constructor when it was invoked?

    This situation is so common in functional programming languages that many offer a construct to automate it. In Scala and other languages, e.g., Haskell, this construct is called pattern matching. It is introduced in the rest of this lecture.

+ A case class definition is like a normal class definition, except that it is preceded by the modifier `case`.

  `````scala
  trait Expr
  case class Number(n: Int) extends Expr
  case class Sum(e1: Expr, e2: Expr) extends Expr
  `````

  Other than defining a trait and two concrete subclasses of it, so-called companion objects with `apply` methods are defined implicitly.

  ````scala
  object Number {
    def apply(n: Int) =
      new Number(n)
  }

  object Sum {
    def apply(e1: Expr, e2: Expr) =
      new Sum(e1, e2)
  }
  ````

  As we saw before, terms like `Number(2)` expand into `Number.apply(2)`, so we can just write `Number(2)`

   instead of `new Number(2)`.

+ In Scala, pattern matching is expressed using the `match ` keyword.

  ````scala
  def eval(e:Expr): Int = e match {
    case Number(n) => n
    case Sum(e1, e2) => eval(e1) + eval(e2)
  }
  ````

  + The syntax can be described as follows: The `match` keyword is followed by a sequence of `case`s, each of which associate a pattern (``pat``) with an expression (`expr`): `pat => expr`.

  + Patterns are constructed from:

    + Constructors, e.g, ``Number``, ``Sum``
    + Variables, e.g., `n`, `e1`, `e2`
    + Constants, e.g. `1`, `true`
      + Besides literal constants, there can also be named constants.
    + Wildcard pattern, i.e., `_`
      + This indicates that we do not care about the value, we cannot reference it inside the expression.

    There is some "fine print":

    + Variables begin with a lowercase letter.
    + The same variable name may appear only once in a pattern, thus, ``Sum(x, x)`` is not a legal pattern.
    + Names of constants begin with a capital letter, with the exception of reserved words `null`, ``true``, ``false``. What if we some named constant, say, `val hello = "Hello, world."` is in scope and we want to use it in our pattern. Then we must either surround it with backticks (`` `hello` ``) or make it uppercase (`Hello`). See the following StackOverflow post for more information: https://stackoverflow.com/questions/7078022/why-does-pattern-matching-in-scala-not-work-with-variables

  + Patterns are evaluated as follows: An expression `e match { case p1 => e1 ... case pn => en }` matches the value of the selector `e` with the patterns ``p1, ..., pn`` in the order in which they are written. The whole match expression is rewritten to the right-hand side of the first case where the pattern matches matches `e`. Variables in the pattern are replaced by the corresponding parts in the selector.

    + When do patterns match?

      + A constructor pattern `C(p1, ..., pn)` matches all instances of type `C` (or subtypes) that were constructed with arguments matching the patterns `p1, ..., pn`.
      + A variable pattern `x` matches any value, and binds the name of the variable to it.
      + A constant pattern `c` matches values that are equal to `c` (in the sense of `==`)

    + This is what the evaluation of a pattern matching expression looks like:

      ````scala
         eval(Sum(Number(1), Number(2)))

      -> Sum(Number(1), Number(2)) match {
           case Number(n) => n
           case Sum(e1, e2) => eval(e1) + eval(e2)
         }

      -> eval(Number(1)) + eval(Number(2))

      -> Number(1) match {
           case Number(n) => n
           case Sum(e1, e2) => eval(e1) + eval(e2)
         } + eval(Number(2))

      -> 1 + eval(Number(2))

      // ...

      -> 3
      ````

  + Obviously, we could also define the evaluation function as a member function of our base trait `Expr`:

    ````scala
    trait Expr {
      def eval: Int = this match {
        case Number(n) => n
        case Sum(e1, e2) => e1.eval + e2.eval
      }
    }
    ````

    This is equivalent to `eval` as we previously defined it, except that we now pattern match over `this` and write `e1.eval` instead of `eval(e1)`.

  + Now, what is the difference between the pattern-matching-based approach of this lecture and the object-oriented decomposition approach from the lecture before? It comes down to whether we more often create new subclasses, e.g., `Number`, or do we more often create new methods, e.g., `eval`? In the former class, the object-oriented approach is favorable, as we can create a new subclass of `Expr`, e.g., `Var`, and simply implement `eval` in it without having to touch any existing code.  On the other hand, if we mostly add new functions, e.g., `show`, it's easiest to just use pattern matching and add the new function to the base trait or even put it outside of it -- no need to touch any existing classes. This two-dimensional problem (subclasses vs. functions) is called the "expression problem".

+ Lastly, an exercise is given, asking us to define two subclasses of `Expr`, `Var` for variables `x` and `Prod` for products `x * y`. Then, we are asked to implement a method ``show``, that gets operator precedence right, i.e., that it uses as few parentheses as necessary. Here's my shot at it:

  ````scala
  def show(e: Expr): String = e match {
    case Number(x) => x.toString
    case Sum(e1, e2) => show(e1) + " + " + show(e2)
    case Prod(e1, e2) =>
      val lhs = e1 match {
        case Sum(se1, se2) => "(" + show(se1) + " + " + show(se2) + ")"
        case _ => show(e1)
      }
      val rhs = e2 match {
        case Sum(se1, se2) => "(" + show(se1) + " + " + show(se2) + ")"
        case _ => show(e2)
      }
      lhs + " * " + rhs
    case Var(x) => x
  }

  show(Sum(Prod(Number(2), Var("x")), Var("y"))) // 2 * x + y
  show(Prod(Sum(Number(2), Var("x")), Var("y"))) // (2 + x) * y
  ````

### Lecture 4.7 -- Lists

+ As mentioned before, lists are a fundamental data structure in functional programming.

+ A list having `x1`, ..., `xn` as elements is written `List(x1, ..., xn)`.

  ````scala
  val fruit = List("apples", "oranges", "pears")
  val nums  = List(1, 2, 3, 4)
  ````

+ There are two important differences between lists and arrays:

  + Lists are immutable, while arrays are mutable, i.e., you can update its elements.
  + Lists are recursive, while arrays are flat.

+ Like arrays, lists are homogenous. `scala.List` (or, for short, `List`) takes one type parameter `T`, which denotes the type of all the elements of the list.

  ````scala
  val fruit = List[String]("apples", "oranges", "pears")
  val nums  = List[Int](1, 2, 3, 4)
  ````

+ All lists are constructed from the empty list `Nil` and the construction operation `::` (pronounced `cons`), so that ``x :: xs `` results in a new list with `x` being the first element of the list (head), and `xs` being the rest of the list (tail). (In our own list implementation, this would be `new Cons(x, xs)`.)

  ````scala
  val fruit = "apples" :: ("oranges" :: ("pears" :: Nil))
  val nums  = 1 :: (2 :: (3 :: (4 :: Nil)))
  val empty = Nil
  ````

+ By convention, operators ending in `:` associate to the right, thus, `A :: B :: C` can be interpreted as `A :: (B :: C)`: Consequently, we can omit the parentheses and just write:

  ````scala
  val fruit = "apples" :: "oranges" :: "pears" :: Nil
  val nums  = 1 :: 2 :: 3 :: 4 :: Nil
  ````

  Furthermore, operators ending in `:` are also different in that they are seen as method calls of the right-hand operand. So the expression `1 :: 2 :: 3 :: 4 :: Nil` is equivalent to `Nil.::(4).::(3).::(2).::(1)`. (So it really corresponds to the `prepend` method of our own list implementation.)

+ All operations on lists can be defined in terms of the three operations `head`, `tail`, and `isEmpty`, which are defined as methods of objects of the type `List`.

+ Lists can be decomposed using pattern matching:

  + A pattern matching the `Nil` constant: `Nil`
  + A pattern matching a list with a head matching `p` and a tail matching `ps`: `p :: ps` (`List(p1, ..., pn)` is equivalent to `p1 :: ... :: pn :: Nil`.)

  ````scala
  List(1, 2, xs) // Matches lists starting with the elements `1` and `2`,
                 // equivalent to `1 :: 2 :: xs`
  List(x)        // Matches lists with one element, equivalent to `x :: Nil`
  List()         // Matches the empty list, equivalent to `Nil`
  ````

+ The length of a list that is matched by the pattern `x :: y :: List(xs, ys) :: zs` is >= 3, as it matches a list with a first element `x`, a second element `y`, a third element `List(xs, ys)`, which is itself a list, and a tail `zs`.

+ Finally, the insertion sort algorithm for sorting a list of numbers in ascending order is given.

  1. Given a list of numbers to be sorted, we first recursively call the algorithm to sort the tail of the list.
  2. Then, after having obtained a sorted list containing the elements of the tail of the original list in an ascending order,  we insert the head element of the original list into the sorted list at the right position.

  ````scala
  def isort(xs: List[Int]): List[Int] = xs match {
    case List() => List()
    case y :: ys => insert(y, isort(ys))
  }

  def insert(x: Int, xs: List[Int]): List[Int] = xs match {
    case List() => List(x)
    case y :: ys => if (x < y) x :: xs else y :: insert(x, ys)
  }
  ````

  The worst-case complexity of of insertion sort relative to the length of the input list `n` is proportional to `n^2`

  + Looking at `insert` first, the worst case is that `x` is greater than all the elements of the list `xs`, so its steps would be proportional to `n`, where `n` is the length of the list.
  + `isort` calls insert `n` times, where `n` is the length of the original list.
