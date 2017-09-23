# Notes on "Functional Programming Principles in Scala"

## Week 3 -- Data and Abstraction

### Lecture 3.1 -- Class Hierarchies

+ Abstract classes contain members that are yet to be implemented. Accordingly, they cannot be instantiated. `IntSet` is an example of an abstract class:

  ````scala
  abstract class IntSet {
    def contains(x: Int): Boolean
    def incl(x: Int): IntSet
  }
  ````

+ Here is an implementation of `IntSet` in terms of binary trees:

  ````scala
  class Empty extends IntSet {
    def contains(x: Int): Boolean = false
    def incl(x: Int): IntSet = new NonEmpty(x, new Empty, new Empty)
  }

  class NonEmpty(elem: Int, left: IntSet, right: IntSet) extends IntSet {
    def contains(x: Int): Boolean =
      if (x < elem) left contains x
      else if (x > elem) right contains x
      else true

    def incl(x: Int): IntSet =
      if (x < elem) new NonEmpty(elem, left incl x, right)
      else if (x > elem) new NonEmpty(elem, left, right incl x)
      else this
  }
  ````

   + This is still a purely functional approach, there is no mutation going on. We never "change" the data structure, we always return a new one.
   + Data structures like this are called persistent data structures, since when we do "changes", e.g., by calling `incl`, the old version of the data structure is still maintained, i.e., it is not discarded.
      + Example: Let `x` be a `NonEmpty` node. If we call `incl` on it with an element that is smaller than the element `x` is carrying, the new `NonEmpty` node resulting from this call will reference  `x`'s right-hand side, i.e., `x`'s `right`. Parts of `x`'s left-hand side may also stay referenced. This can easily be reproduced using pen and paper.

+ Terminology

  + `Empty` and `NonEmpty` extend `IntSet`. As such, they conform to the `IntSet` type and can be used wherever an `IntSet` is required.
  + `IntSet` is the superclass of `Empty` and `NonEmpty`.
  + `Empty` and `NonEmpty` are subclasses of `IntSet`.
  + In Scala, any user-defined class extends another class, if no superclass is provided, the class in question simply extends `java.lang.Object`.
  + The base classes of a class are all direct or indirect superclasses of that class. E.g., `Empty`'s base classes are `IntSet` and `java.lang.Object`.
  + `incl` and `contains` in `Empty` and `NonEmpty` implement the respective abstract functions in `InSet`.
  + A class may redefine a non-abstract function of its superclass by prefixing its definition with the keyword `override`, e.g., `override def foo = 2`. Using `override` here is not optional, it is, however, optional and mostly omitted when implementing an abstract function. 

+ As it can be argued that there really is one single `Empty` node, it can be replaced by a singleton object. Singleton objects cannot be created. Behind the scenes, one instance is being created once it is first referenced. Singletons are values and evaluate to themselves.

  ````scala
  object Empty extends IntSet {
    def contains(x: Int): Boolean = false
    def incl(x: Int): IntSet =
      new NonEmpty(x, Empty, Empty) // It does not say `new Empty` anymore but `Empty`.
  }
  ````

+ Standalone Scala programs (as opposed to worksheets or the REPL) need to have a `main` method inside an object.

  ````scala
  object Hello {
    def main(args: Array[String]) =
      println("Hello, world.")
  }
  ````

+ What would a method `def union(other: IntSet): IntSet` look like?

  + For `Empty`, it would be `other`, as the union of an empty set with another set always results in the other set.
  + For `NonEmpty`, it would be `((left union right) union other) incl elem`? How can we know this actually terminates? Well, the first recursive call, `left union right`, is on `left`. Thus, the first recursive call is on something smaller than what we started with, `this`, and has to eventually reach the base case defined in `Empty`. The same goes for the second recursive call, `(left union right) union other`, which is on the union of `left` and `right`, which, in turn, is still smaller than `this`, as it doesn't contain `elem`. Therefore, the base case will eventually be reached, too.

+ Scala implements dynamic binding/dynamic method dispatch. Thus, whenever an overridden method is called through a superclass reference, e.g., `val s: IntSet = Empty; s.incl(42)`, which version of the method to execute is made based on the runtime type of the object that contains it. Thus, at run time, `Empty`'s implementation is chosen -- not `IntSet`'s (if it had one).

+ Here two examples of how dynamic binding evaluates:

  ````
     Emtpy contains 1
  -> [1/x][Empty/this]
     false
  =  false

     (new NonEmpty(7, Empty, Empty)) contains 7
  -> [7/elem][7/x][new NonEmpty(7, Empty, Empty)/this]
     if (x < elem) this.left contains x
     else if (x > elem) this.right contains x
     else true
  =  if (7 < 7) new NonEmpty(7, Empty, Empty).left contains 7
     else if (7 > 7) new NonEmpty(7, Empty, Empty).right contains 7
     else true
  =  true
  ````

+ Can we implement higher-order functions in terms of objects? Obviously. Here is my shot at it:

  ````scala
  trait Function[-Param, +Return] {
    def apply(param: Param): Return
  }

  // `f1` demonstrates why covariance and contravariance are necessary.
  val f1: Function[String, Any] = new Function[Any, String] {
    override def apply(param: Any) =
      param.toString
  }

  assert(f1.apply("Hello, world.") == "Hello, world.")

  // `f2` demonstrates how `Function` objects can represent higher-order functions.
  val f2: Function[Int, Function[Int, Int]] = new Function[Int, Function[Int, Int]] {
    override def apply(outer: Int) =
      new Function[Int, Int] {
        override def apply(inner: Int) =
          inner + outer
      }
  }

  assert(f2.apply(2).apply(40) == 42)
  ````

+ Can We implement objects is terms of higher-order functions. I'll delegate to the following blog post, which argues that objects and closures are equivalent: http://c2.com/cgi/wiki?ClosuresAndObjectsAreEquivalent

### Lecture 3.2 -- How Classes are Organized

+  Classes and objects are organized in packages. To do so, a package clause can be placed at the top of a source file.

  ````scala
  package progfun.examples
  object Hello {}
  ````

  `Hello`'s fully qualified name is `progfun.examples.Hello`.

+ A class `Rational` in package `week3` can be used via its fully qualified name, e.g., `new week3.Rational(1, 2)`. Alternatively, imports can be used.

  ````scala
  // Named imports
  import week3.Rational          // Imports just `Rational`
  import week3.{Rational, Hello} // Imports both `Rational` and `Hello`
  // Wildcard import
  import week3._                 // Imports everything in package `week3`
  ````

  One can import from packages and (singleton) objects.

+ Some entities are imported automatically into any Scala program.

  + All members of package `scala`, e.g., `Int`.
  + All members of package  `java.lang`, e.g., `Object`.
  + All members of the singleton object `scala.Predef`, e.g. `assert`.

+ Scala's standard library can be explored using the scaladoc pages: http://www.scala-lang.org/files/archive/api/current/

+ Just like in Java, classes may only inherit from one superclass. To inherit from arbitrarily many superclasses ("single inheritance language"), one can use traits.

  ````scala
  trait Planar {
    def height: Int
    def width: Int
    def surface = height * width
  }

  class Square extends Shape with Planar with Movable // ...
  ````

  Traits resemble Java's interfaces. They are, however, more powerful, as they can also contain fields and concrete methods. Unlike classes, traits cannot have (value) parameters.

+ The remainder of this lecture is devoted to Scala' class hierarchy.![Scala Class Hierarchy](scala_class_hierarchy.png)
+ Top types
  + `Any` is the base type of all types, and conceptually defines certain universal methods, e.g., `==`, `!=`, `equals`, `hashCode`, `toString`. (`==` is essentially just a forwarder that calls the Java `equals` method.)
  + `AnyVal` is the base type of all primitive types.
  + `AnyRef` is the base type of all reference types. It is an alias of `java.lang.Object`.
+ The `Nothing` type is at the bottom of the type hierarchy. It subclasses every other type. There is no value of type `Nothing`. What is it useful for?
  + It can be used to signal abnormal termination. (An example follows when exceptions are covered.)
  + It can be used as an element type of empty collections, e.g., `Set[Nothing]`.
+ The `Null` type subclasses every class that inherits from `Object`. The type of `null` is `Null`. Every reference class type has a value `null`.
+ Exceptions is Scala are similar to exceptions in Java. The expression `throw Exc` aborts the evaluation with the exception `Exc`. The type of this expression is `Nothing`.
+ The type of  `if (true) 1 else false` is `AnyVal`, as `1` is an `Int`, `false` is a `Boolean` and `AnyVal` happens to be the closest `AnyVal` of the two.

### Lecture 3.3 -- Polymorphism

+ The immutable linked list is a fundamental data structure in many functional programming languages. Such a list is either a `Nil` or a `Cons`. `Nil` represents an empty list, whereas `Cons` holds an element `head` as well as a pointer to the rest of the list `tail`.

+ At first, an implementation of an `IntList`, i.e., a definition of lists with integers, is given. By this example, the concept of value parameters is explained, which is about defining parameters and fields of a class at the same time.

  ````scala
  // Given...
  trait IntList
  // ...then the following...
  class Cons(val head: Int, val tail: IntList) extends IntList
  // ...is equalvalent to...
  class Cons(_head: Int, _tail: IntList) extends IntList {
    val head = _head
    val tail = _tail
  }
  // ...where `_head` and `_tail` are otherwise unused names. 
  ````

+ `IntList` does not scale, as we'd need `DoubleList`, and so on. Therefore, a generalized definition that makes use of type parameters is given:

  ````scala
  trait List[T] {
    def isEmpty: Boolean
    def head: T
    def tail: List[T]
  }

  class Cons[T](val head: T, val tail: List[T]) extends List[T] {
    def isEmpty = false
  }

  class Nil[T] extends List[T] {
    def isEmpty: Boolean = true
    def head: Nothing = throw new NoSuchElementException("Nil.head")
    def tail: List[T] = throw new NoSuchElementException("Nil.tail")
  }
  ````

  Type parameters are written in square brackets, e.g., `[T]`.

+ In order to demonstrate that functions can have type parameters, too, here is a function that creates `List`s containing one element of type `T`:

  ````scala
  def singleton[T](elem: T) = new Cons[T](elem, new Nil[T])

  singleton[Int](1) // new Cons[Int](1, new Nil[Int])
  singleton[Boolean](true) // new Cons[Boolean](true, new Nil[Boolean])
  ````

+ The Scala compiler can oftentimes deduce the correct type parameters from the provided value arguments. Thus, they can be omitted in a lot of cases. E.g., `singleton(1)` can be written instead of `singleton[Int](1)`.

+ Type parameters (and arguments) do not affect evaluation in Scala. In terms of our substitution model, we can assume that all types are removed before evaluating a program. Types are only important for the compiler. Languages that also do type erasure include Java, Scala, and Haskell, whereas C++, C# and F# keep them at run time.

+ Polymorphism is a Greek word with the original meaning of "having many forms". Applied to a function, it means that this function can be applied to arguments of many types. Applies to a type, it means that the type can have instances of many types. In this course, polymorhism has been seen in two forms:

  + Subtyping: Wherever there is a parameter that accepts a base class instance, a subclass instance can be passed.
  + Generics: Many instances of a function or class can be created using type parameterization, e.g., `List[Int]` or `List[String]`.

+ A function `nth` selecting the `n`th integer of a given list might look like this:

  ````scala
  def nth[T](n: Int, xs: List[T]): T =
    if (xs.isEmpty) throw new IndexOutOfBoundsException
    else if (n == 0) xs.head
    else nth(n - 1, xs.tail)
  ````

  The first list element has the number 0. If `n` is outside the range from 0 until length of the list minus one, an `IndexOutOfBoundsException` is thrown.
