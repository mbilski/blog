---
title: Customizing Circe's auto generic derivation
date: 2017-02-25 14:42:11
highlightjslanguages: ["scala"]
---

While modeling the domain I often use sealed trait hierarchies, value classes, and case classes. They are essential for idiomatic Scala code. However, the encoding and decoding them to JSON can be problematic. Even though Circe provides boilerplate-free generic derivation, the JSON encoded using the default configuration may be confusing. In this post, I describe how to make it more friendly.

<!--more-->

## Intro

[Circe] is a JSON library for Scala, is a fork of [Argonaut] and depends on [Cats]. It provides generic codec derivation using [Shapeless] with support for algebraic data types (ADT, sealed trait hierarchies).

In this post, I build a simple model with sealed traits, serialize it to JSON using [Circe], and show how to customize encoding using [Shapeless]. You can see the full example [here](https://github.com/mbilski/circe-configuration).

## Model

First, let's build the ADT. In this example I use a `Movie` case class which consists of `Name`, `Genre`, `Reviews` and `ReleaseDate`. The `Genre` is a sealed trait with case object values only.

```
sealed trait Genre
case object Action extends Genre
case object Comedy extends Genre
case object Drama extends Genre
```

The `Review` is a hierarchy of case classes. It can be a review written by a user/critic or posted as a link. User and critic are identified using value classes `CriticId` and `UserId`.

```
case class CriticId(value: String) extends AnyVal
case class UserId(value: String) extends AnyVal

sealed trait Review

case class RemoteReview(
  url: Option[String], score: Int
) extends Review

case class CritictReview(
  cricitId: CriticId, review: String, score: Int
) extends Review

case class UserReview(
  userId: UserId, review: String, score: Int
) extends Review
```

The `Movie` has a list of reviews which are either `RemoteReview`, `CriticReview` or `UserReview`.

```
import java.time._

case class Movie(
  name: String, genre: Genre,
  reviews: List[Review], releaseDate: LocalDate
)

val movie = Movie("Coherence", Drama, List(
  RemoteReview(Some("http://www.imdb.com/title/tt2866360/reviews"), 9),
  UserReview(UserId("ab38d19dj"), "Absolutely brilliant", 10)
), LocalDate.of(2014, 8, 6))

```

## Standard configuration

Before encoding the movie to JSON we need [Circe] dependencies. I use [Typelevel Scala 2.12.1](https://github.com/typelevel/scala/blob/typelevel-readme/notes/2.12.1.md) (but it is not required).


```
scalaVersion := "2.12.1"

scalaOrganization in ThisBuild := "org.typelevel"

val circeVersion = "0.7.0"

libraryDependencies ++= Seq(
  "io.circe" %% "circe-core",
  "io.circe" %% "circe-java8",
  "io.circe" %% "circe-parser",
  "io.circe" %% "circe-generic-extras"
).map(_ % circeVersion)

```

To encode the movie into JSON you only need some imports as Circe is boilerplate-free.

```
import io.circe.generic.extras.auto._, io.circe.syntax._
import io.circe.java8.time._
movie.asJson.spaces2
```

Below you can see the result of encoding.

```JSON
{
  "name" : "Coherence",
  "genre" : {
    "Drama" : {}
  },
  "reviews" : [
    {
      "RemoteReview" : {
        "url" : "http://www.imdb.com/title/tt2866360/reviews",
        "score" : 9
      }
    },
    {
      "UserReview" : {
        "userId" : {
          "value" : "ab38d19dj"
        },
        "review" : "Absolutely brilliant",
        "score" : 10
      }
    }
  ],
  "releaseDate" : "2014-08-06"
}
```

There are three things that I do not like:
 - the type of sealed trait is a JSON `key`
 - value class `UserId` is encoded as JSON object with a `value`,
 - `Genre` is a JSON object with type as a `key` and empty object as a value.

Let's see how to customize it.

## Custom configuration

Module `circe-generic-extras` provides a way to set the discriminator which is used for encoding hierarchies. Now the type will be encoded as a field of name `type` without nesting.

```
implicit val configuration: Configuration = Configuration.default
  .withDiscriminator("type")
```

To simplify encoding of value classes we need following generic codecs built using [Shapeless] ([source](https://gist.github.com/Taig/03386fa584bd48f392f9d812b1c86072)).

```
import shapeless._

implicit def encoderValueClass[T <: AnyVal, V](implicit
  g: Lazy[Generic.Aux[T, V :: HNil]],
  e: Encoder[V]
): Encoder[T] = Encoder.instance { value =>
  e(g.value.to(value).head)
}

implicit def decoderValueClass[T <: AnyVal, V](implicit
  g: Lazy[Generic.Aux[T, V :: HNil]],
  d: Decoder[V]
): Decoder[T] = Decoder.instance { cursor =>
  d(cursor).map { value
    g.value.from(value :: HNil)
  }
}
```

For sealed trait hierarchies, which consist only of `case objects`, first, we need to define what `enum` is.


```
import shapeless.labelled._

trait IsEnum[C <: Coproduct] {
  def to(c: C): String
  def from(s: String): Option[C]
}

object IsEnum {
  implicit val cnilIsEnum: IsEnum[CNil] = new IsEnum[CNil] {
    def to(c: CNil): String = sys.error("Impossible")
    def from(s: String): Option[CNil] = None
  }

  implicit def cconsIsEnum[K <: Symbol, H <: Product, T <: Coproduct](implicit
    witK: Witness.Aux[K],
    witH: Witness.Aux[H],
    gen: Generic.Aux[H, HNil],
    tie: IsEnum[T]
  ): IsEnum[FieldType[K, H] :+: T] = new IsEnum[FieldType[K, H] :+: T] {
    def to(c: FieldType[K, H] :+: T): String = c match {
      case Inl(h) => witK.value.name
      case Inr(t) => tie.to(t)
    }

    def from(s: String): Option[FieldType[K, H] :+: T] =
      if (s == witK.value.name) Some(Inl(field[K](witH.value)))
      else tie.from(s).map(Inr(_))
  }
}
```

And then it can be used to make the `type` a JSON `value`. For this, I used example posted by Travis Brown [here](http://stackoverflow.com/a/37015184).

```
implicit def encodeEnum[A, C <: Coproduct](implicit
  gen: LabelledGeneric.Aux[A, C],
  rie: IsEnum[C]
): Encoder[A] = Encoder[String].contramap[A](a => rie.to(gen.to(a)))

implicit def decodeEnum[A, C <: Coproduct](implicit
  gen: LabelledGeneric.Aux[A, C],
  rie: IsEnum[C]
): Decoder[A] = Decoder[String].emap { s =>
  rie.from(s).map(gen.from).toRight("enum")
}
```

Having all encoders and decoders defined, now we can build the JSON and make sure that after decoding the `Movie` still looks the same.

```
val encoded = movie.asJson.spaces2
val decoded = decode[Movie](encoded)
assert(movie == decoded.right.get)
```

As you can see, this JSON looks much more friendly.

```json
{
  "name" : "Coherence",
  "genre" : "Drama",
  "reviews" : [
    {
      "url" : "http://www.imdb.com/title/tt2866360/reviews",
      "score" : 9,
      "type" : "RemoteReview"
    },
    {
      "userId" : "ab38d19dj",
      "review" : "Absolutely brilliant",
      "score" : 10,
      "type" : "UserReview"
    }
  ],
  "releaseDate" : "2014-08-06"
}
```

## Integration

To integrate custom configuration with auto generic derivation you can create an object that extends `AutoDerivation`, and then put all the codecs there. In addition, I prefer to use semi-automatic derivation and have the encoders and decoders defined in code.

```
import io.circe._
import io.circe.generic.extras._
import io.circe.generic.semiauto._
import io.circe.java8.time.TimeInstances

object json {
  object codecs {
    import auto._, pl.immutables.circe._

    implicit val movieEncoder: Encoder[Movie] = deriveEncoder[Movie]
    implicit val movieDecoder: Decoder[Movie] = deriveDecoder[Movie]
  }

  private object auto extends AutoDerivation with TimeInstances {
    import shapeless._

    // all the custom codecs ...

    implicit val configuration: Configuration = Configuration.default
      .withDiscriminator("type")
  }
}
```

Having that, whenever you need `implicit Encoder[Movie]` just put `import json.codecs._` there.


## Resources

You can see full example [here](https://github.com/mbilski/circe-configuration). To learn [Shapeless] you can read [The Type Astronaut's Guide to Shapeless Book](http://underscore.io/books/shapeless-guide/).

[Cats]: http://typelevel.org/cats/
[Circe]: http://circe.github.io/circe/
[Argonaut]: http://argonaut.io/
[Shapeless]: https://github.com/milessabin/shapeless
