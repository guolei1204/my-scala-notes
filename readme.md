# my-scala-notes
My scala notes, because I need to store them somewhere...

## Case Class Tricks
A case class is a [scala.Product](http://www.scala-lang.org/files/archive/api/current/scala/Product.html), which
is the base trait for all case classes and [tuples](http://www.scala-lang.org/files/archive/api/current/scala/Tuple1.html). Because both case classes and tuples are a Product type, they have access to the methods:

| method | description |
| ------ | ----------- |
| productArity: Int | The size of this product ie. the number of types it contains eg. Person(name: String, age: Int) has an arity of 2 |
| productElement(n: Int): Any | Returns the n-th element of the product, and is zero-based |
| productIterator: Iterator[Any] | An iterator over the elements of the product |
| productPrefix: String | A String used in the toString() methods of derived classes | 

```scala
scala> case class Person(name: String, age: Int)
defined class Person

scala> val person = Person("foo", 42)
person: Person = Person(foo,42)

scala> person.isInstanceOf[Product]
res0: Boolean = true

scala> val values = person.productIterator.toList
values: List[Any] = List(foo, 42)

scala> person.productElement(0)
res1: Any = foo

scala> person.productElement(1)
res2: Any = 42

scala> person.productArity
res3: Int = 2

scala> person.productPrefix
res4: String = Person
```

Getting the fields of the case class:

```scala
import scala.reflect.runtime.universe._

scala> def getMethods[T <: Product : TypeTag] : List[MethodSymbol] = typeOf[T].members.collect {
     |     case m: MethodSymbol if m.isCaseAccessor => m
     |   }.toList
getMethods: [T <: Product](implicit evidence$1: reflect.runtime.universe.TypeTag[T])List[reflect.runtime.universe.MethodSymbol]

scala> getMethods[Person]
res5: List[reflect.runtime.universe.MethodSymbol] = List(value age, value name)

scala> getMethods[Person].map(_.name.toString)
res6: List[String] = List(age, name)
```

Getting the names and values of the case class:

```scala
scala> :paste
// Entering paste mode (ctrl-D to finish)

getMethods[Person]
  .map(_.name.toString)
  .iterator.zip(Person("foo", 42).productIterator)
  .toMap

// Exiting paste mode, now interpreting.

res7: scala.collection.immutable.Map[String,Any] = Map(age -> foo, name -> 42)
```

## Scalaz ReaderT usage
A [monad transformer](http://eed3si9n.com/learning-scalaz/Monad+transformers.html) for the Reader Monad

```scala
import scalaz._
import Scalaz._
import scala.language.higherKinds
import scala.language.implicitConversions

case class StringStats(length: Int, palindrome: Boolean)

def stringStats(calcLength: String => Int, isPalindrome: String => Boolean): String => StringStats = for {
  length <- calcLength
  palindrome <- isPalindrome
} yield StringStats(length, palindrome)

def stringStatsCtx[F[_]: Monad]
  (calcLength: ReaderT[F, String, Int],
   isPalindrome: ReaderT[F, String, Boolean]): ReaderT[F, String, StringStats] = for {
  length <- calcLength
  palindrome <- isPalindrome
} yield StringStats(length, palindrome)

def compose[F[_]: Monad, A: Monoid](fx: F[A], fy: F[A]): F[A] = for {
  x <- fx
  y <- fy
} yield x |+| y

compose[Id, Int](1, 2) == 3

val statsFunction: (String) => StringStats = 
  stringStats(_ => 4, _ => true)

statsFunction("abba") == StringStats(4, true)

// actually use the function and define the effect
val statsFunctionCtx: ReaderT[Option, String, StringStats] =
  stringStatsCtx[Option](
    ReaderT[Option, String, Int](_ => Option(4)),
    ReaderT[Option, String, Boolean](_ => Option(true))
  )

statsFunctionCtx("abba") == Some(StringStats(4, true))

// there must be an implicit execution context
// available at the call site
import scala.concurrent.duration._
import scala.concurrent.{Await, Future}
def callStrLengthService(str: String): Future[Int] =
  Future.successful(4)

def callStrPalindromeService(str: String): Future[Boolean] =
  Future.successful(str.reverse == str)

import scala.concurrent.ExecutionContext.Implicits.global
val statsFunctionAsync: ReaderT[Future, String, StringStats] =
  stringStatsCtx[Future](
    ReaderT[Future, String, Int](callStrLengthService),
    ReaderT[Future, String, Boolean](callStrPalindromeService)
  )

Await.result(statsFunctionAsync("abba"), 1.second) == 
  StringStats(4, true)
```