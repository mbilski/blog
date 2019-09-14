---
title: For-comprehensions with monad transformers
date: 2016-05-14 14:33:13
---

The code should be easy to read. One of the programming principles to make the code more readable is to avoid nested operations. The so-called *pyramid of doom* does not only refers to *callback hell* in javascript but ofter appear in scala as well. Here, I describe how to make the for-comprehensions more elegant when using monad transformers in scala.

<!--more-->

## Theory

For-comprehensions in scala allows writing easy to read and maintain code. In practice, it is just a syntactic sugar for composing `map`, `flatMap` and `filter` operations. Any class providing them can be used in for-comprehension statement.

```
case class User(name: String, email: Option[String])

val users = List(
  User("andrzej", email: Some("andrzej@email.com")),
  User("grzegorz", None)
)
```

For such list of users to get an email by name the following code

```
def getEmail(name: String): Option[String] =
  users.find(_.name == name).flatMap(_.email)
```

can be rewritten using for-comprehension like this

```
def getEmailFor(name: String): Option[String] = for {
  user <- users.find(_.name == name)
} yield user.email
```
The return type is always the same as the first expression's type.

### Monads

In scala we often use context containers (monads) like

 - `Future[A]` - for asynchronous computations,
 - `Option[A]` - for optional values,
 - `Try[A]` - for exception handling,
 - `Either[A, B]` - usually to express an error or a value.

It is possible to use all of them in for-comprehensions.

However, each statement must result in the same type as the first one. It is not possible to mix `Either` with `Option`, you need to convert `Option` to `Either` in order to use it next to `Either` resulting statement. Things get even more complicated when you combine monads with each other.

You may have functions that return an async optional value `Future[Option[A]]` or an async error or value `Future[Either[Error, A]]`. For this, you need to use nested for-comprehensions.

```
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

case class Error(msg: String)
case class User(username: String, email: String)

def authenticate(token: String): Future[Either[Error, String]] = ???
def getUser(username: String): Future[Option[User]] = ???

val result: Future[Either[Error, User]] = for {
  user <- authenticate("secret1234").flatMap {
    case Left(error) => Future.successful(Left(error))
    case Right(username) => for {
      userEither <- getUser(username).map {
        case Some(user) => Right(user)
        case _ => Left(Error("not found"))
      }
    } yield userEither
  }
} yield user
```

In this case, for comprehensions are not easy to read as when using single monads..

### Monad transformers

To avoid nested for-comprehensions you can use a common context container (monad) and transform all expressions into it. In the example above, the combined monad is `Future[Either[Error, A]]`. If you have a way to transform to result of `authenticate` and `getUser` methods into this monad, you can make this for-comprehension look simple.

## Example

I prepared an example that shows how to use scalaz monad transformers to build neat for-comprehensions for a http4s service with monad transformers. The example app provides a REST endpoint that authenticates a user using token and gets the user and his devices. The model looks like this:

```
case class User(email: String, token: String, devices: List[String])
case class Device(id: String, name: String)
case class UserWithDevices(user: User, devices: List[Device])

sealed trait Error
object InvalidToken extends Error
```

There are two methods: authenticate a user using token and get a device by id.

```
object UserService {
  val users = Seq(User("test@immutables.pl", "abcde123", List("1")))

  def authenticate(token: String): Task[Error \/ User] = {
    Task.now(users.find(_.token == token)).map {
      case Some(user) => user.right[Error]
      case _ => InvalidToken.left[User]
    }
  }
}

object DeviceService {
  val devices = Seq(Device("1", "nexus"))

  def getById(id: String): Task[Option[Device]] = {
    Task.now(devices.find(_.id == id))
  }
}
```

> `Task` is a `scalaz.concurrent.Future[Throwable \/ A]` and is used in **http4s** a lot. Read more about task [here](http://timperrett.com/2014/07/20/scalaz-task-the-missing-documentation/). `A \/ B` is a scalaz way to express `Either[A, B]`

UserService's `authenticate` function returns asynchronous computation of either error or user. DeviceService's `getByid` function return asynchronous computation of optional device.

In order to use both functions in for-comprehension without nesting, we need to use a combined monad.

```
type Failure = Task[Response]
type Result[A] = EitherT[Task, Failure, A]
```

`EitherT[Task, Failure, A]` represents something like `Future[Either[Failure, A]]`. It is a monad transformer. It allows wrapping `Either` into another monad like `Task` easily. `Result[A]` shares the behaviour of `Task` and `Either`. In simple terms, it provides *flatMap* and *map* methods that do the *nesting* for you.

It is easy to create a `Result[A]` of a single value. You just need to do `"email".point[Result]`. However, to build it from a `Task[Option[A]]` in a convenient way you need helper functions.

```
object Result {
  def ofTask[A](task: Task[A]): Result[A] = EitherT(task.map(_.right))

  def ofOption[A](failure: => Failure)(oa: Option[A]): Result[A] =
    EitherT(Task.now(oa.toRightDisjunction(failure)))

  def ofTOption[A](failure: => Failure)(toa: Task[Option[A]]): Result[A] =
    EitherT(toa.map(_.toRightDisjunction(failure)))

  def ofTEither[A, E](failure: E => Failure)(tea: Task[E \/ A]): Result[A] =
    EitherT(tea.map(_.leftMap(failure)))
}
```

Here, are some functions that simplify creation of a `Result[A]` and provide a way to express a failure.

```
lazy val service = HttpService {
  case req @ GET -> Root / "profile" => for {
    token <- req.headers.get(tokenHeader) |>
      Result.ofOption(BadRequest("missing token"))

    user <- UserService.authenticate(token.value) |>
      Result.ofTEither(e => Forbidden("invalid token"))

    devices <- Task.gatherUnordered(
        user.devices.map(id => DeviceService.getById(id))
      ) |> Result.ofTask
  } yield Ok(UserWithDevices(user, devices.flatten).asJson)
}
```

> `|>` is the Thrush combinator. In this context it applies the value on the left to a function on the right.

Although, the for-comprehension operates on monad transformers the code is not nested. In addition, it handles the cases when `Either == Left` or `Option == None`. Complete example can be found at [github]. Run `sbt test` to see the spec.

```
ProfileEndpointSpec:
- should return Ok for valid token
- should return Forbidden when for invalid token
- should return BadRequest when token is missing
```

## Summary

There are many resources that cover the monad transformers topic in more detail. Here, I wanted to show a particular problem that can be solved using them and the benefits. I used scalaz and http4s just as an example. Similar functionality can be achieved in vanilla scala as well, but requires more code.

[github]: https://github.com/mbilski/http4s-monad-transformers-example
