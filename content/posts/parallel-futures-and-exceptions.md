---
title: Parallel futures and exceptions
date: 2016-10-08 20:06:24
---

Asynchronous applications are much harder to debug. You do not get the whole stack trace so easily. So it is important to react to all the exceptions that might unexpectedly occur. I found a case when following a simple pattern to run futures in parallel leads to loss of exceptions. In this post I describe a solution for that.

<!--more-->

## Future[A]

`Future[A]` represents an asynchronous computation. It may result in a value of type `A` or an exception. It is possible to apply a function `A => B` to the potential value using `map[B](f: A => B)` or apply a function `A => Future[B]` using `flatMap[B](f: A => Future[B])`. Both results in a new future of type `Future[B]`. The latter can be used to sequence computations that depend on the value of previous ones.

```
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

Future(1).flatMap(x => Future("a").map(y => (a, b)))

> Success((1, "a"))
```

In practice it is more convenient to use `for-comprehensions` as a syntactic sugar for a sequence of `maps` and `flatMaps`.

```
val ret: Future[(Int, String)] = for {
  x <- Future(1)
  y <- Future("a")
} yield (x, y)

> Success((1, "a"))
```

What if we want to execute multiple futures in parallel? I often see the following pattern as a solution. It works because scala `Future` starts the asynchronous computation immediately.

```
val fx = Future(1)
val fy = Future("a")

val ret: Future[(Int, String)] = for {
  x <- fx
  y <- fy
} yield (x, y)

> Success((1, "a"))
```

What is the problem with this approach? If both futures fail, only the first exception is returned. The second one is lost.

```
val fx = Future(1 / 0)
val fy = Future(List[String]().head)

val ret: Future[(Int, String)] = for {
  x <- fx
  y <- fy
} yield (x, y)

> Failure(java.lang.ArithmeticException: / by zero)
```

How to collect all the exceptions? For instance, the same way that parallel validation in cats does - using the *applicative* type class. See [validated] as a reference.

## Applicative[Future]

Add `"org.typelevel" %% "cats" % "0.7.0"` as a dependency to run the snippets. First, let's obtain the default applicative instance for scala futures and see how it works.

```
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

import cats._
import cats.implicits._

Applicative[Future].map2(Future(1), Future("a"))((x, y) => (x, y))

> Success((1, "a"))

Applicative[Future].map2(
  Future(1 / 0),
  Future(List[String]().head)
)((x, y) => (x, y))

> Failure(java.lang.ArithmeticException: / by zero)
```

It still behaves the same way as before. Apparently, the default implementation of *applicative* for *future* just uses `flatMap`. We need to create our own implementation.

```
import scala.concurrent.Promise
import scala.util._

implicit def futureApplicative(implicit st: Semigroup[Throwable]) =
  new Applicative[Future] {

  override def pure[A](x: A): Future[A] = Future.successful(x)

  override def ap[A, B](ff: Future[(A) => B])(fa: Future[A]): Future[B] = {
    val p = Promise[B]()
    ff.onComplete { f =>
      fa.onComplete { a =>
        (f, a) match {
          case (Success(fs), Success(as)) =>
            p.complete(Try(fs(as)))
          case (Failure(fx), Failure(ax)) =>
            p.failure(st.combine(fx, ax))
          case (Failure(fx), _) => p.failure(fx)
          case (_, Failure(ax)) => p.failure(ax)
        }
      }
    }
    p.future
  }
}
```

Here I used `Semigroup` type class to combine exceptions. An example implementation of `Semigroup[Throwable]` may look like this:

```
implicit val throwableSemigroup = new Semigroup[Throwable] {
  override def combine(x: Throwable, y: Throwable): Throwable = (x, y) match {
    case (Exceptions(_, xs), Exceptions(_, ys)) => Exceptions(xs ++ ys)
    case (Exceptions(_, xs), e) => Exceptions(e :: xs)
    case (e, Exceptions(_, ys)) => Exceptions(e :: ys)
    case (a, b) => Exceptions(List(a, b))
  }
}
```

Where `Exceptions` is a class that represents multiple exceptions merged together. The problem with this solution is that some `recover` block may fail to catch the previously excepted exception. However, it does it job. It preserves the stack traces and messages.

```
case class Exceptions(message: String, es: List[Throwable])
  extends RuntimeException(message)

object Exceptions {
  def apply(es: List[Throwable]): Exceptions = {
    val messages = es.map(_.getMessage).mkString(", ")
    val exceptions = Exceptions(messages, es)
    exceptions.setStackTrace(es
      .map(_.getStackTrace)
      .foldLeft(Array[StackTraceElement]())(_ ++ _)
    )
    exceptions
  }
}
```
Let's see what happens if we use this implementation.

```
futureApplicative.map2(Future(1/0), Future(List[String]().head))((x, y) => (x, y))

> Failure(Exceptions: / by zero, head of empty list)
```

Now it is possible to see that both futures failed.

[validated]: http://typelevel.org/cats/tut/validated.html
