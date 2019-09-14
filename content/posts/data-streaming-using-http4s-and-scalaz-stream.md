---
title: Data streaming using http4s and scalaz-stream
date: 2016-01-16 19:38:59
aliases:
- /2016/01/16/Streaming-data-using-http4s-and-scalaz-stream/
---

[Http4s](http://http4s.org/) and [scalaz-stream](https://github.com/functional-streams-for-scala/fs2) are good alternatives for akka-http and akka-stream. In this post, I show how easy it is to stream data over HTTP using them. Streaming allows to operate on heavy data without the necessity to store it in the memory. With http4s, it is just a few lines of code.

<!--more-->

## Reactive programming

Reactive programming is all about programming with asynchronous data streams. There are several libraries which do that in scala: akka-stream, scalaz-stream, play's iteratees or RxScala. It allows developers to create, consume and manipulate streams of data. Streams are non-blocking and provide a control over thread and memory consumption.

## http4s

According to the http4s docs it is *a minimal, idiomatic Scala interface for HTTP and Scala's answer to Ruby's Rack, Python's WSGI, Haskell's WAI, and Java's Servlets.* An Http4s service transforms a **Request** into scalaz **Task[Response]**. It integrates with scalaz-streams and bodies of requests and responses are represented as streams of bytes. Http4s also provides an asynchronous HTTP client which works similar like the server.

## Example

I implemented a simple server and a client. The server allows to download and upload a file. The client downloads a file and immediately uploads it back to the server using streaming. The whole file is never stored in the memory.

```
import scalaz.stream._
import scalaz.concurrent.Task
import scodec.bits.ByteVector

trait Streaming {
  val bufferSize = 128

  def read(path: String): Process[Task, ByteVector] = Process
    .constant(bufferSize).toSource
    .through(io.fileChunkR(path, bufferSize))

  def write(path: String, data: EntityBody) =
    (data to io.fileChunkW(path)).run
}
```

The *Streaming* trait has two functions. The *read* function is for reading a file and it creates a stream of bytes. The *write* function takes a body stream as argument and pipes it to a stream which writes bytes to a file.

```
import org.http4s._
import org.http4s.dsl._
import org.http4s.server._

object StreamingServer extends App with Streaming {
  val service = HttpService {
    case GET -> Root / "download" =>
      Ok(read("avatar.png"))
    case req @ POST -> Root / "upload" =>
      write("uploaded", req.body).flatMap(_ => Created())
  }

  BlazeBuilder.bindHttp(8080)
    .mountService(service, "/")
    .run
    .awaitShutdown()
}
```

*StreamingServer* exposes two HTTP services: GET /download and POST /upload. The first one just reads the content of a file. The second one streams the body into a file. Both services use functions defined in the *Streaming* trait for that.

```
import org.http4s.dsl._
import org.http4s.client._

object StreamingClient extends App {
  client
    .prepare(GET(uri("http://localhost:8080/download")))
    .flatMap(response =>
      client.prepare(POST(uri("http://localhost:8080/upload"), response.body))
    ).run
}
```

The client executes the *download* request and then streams the content into *upload* endpoint. Thanks to http4s this is trivial.

See [data-streaming-example](https://github.com/mbilski/data-streaming-example) for source code and instructions how to build and run.

## Summary

*Http4s* provides a nice DSL which integrates seamlessly with scalaz-concurrent and scalaz-stream. If you are looking for alternative http libraries, you should take a look at http4s.

## Resources

[Http4s examples](https://github.com/http4s/http4s/tree/master/examples)
[Introduction to scalaz-stream](https://gist.github.com/djspiewak/d93a9c4983f63721c41c)
[Introduction to Reactive Programming](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)
