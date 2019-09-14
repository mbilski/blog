---
title: Blocking and non-blocking frameworks
date: 2014-11-26 12:55:52
---

In my master thesis I evaluated performance of blocking and non-blocking web frameworks in the context of distributed RESTful micro services. Non-blocking frameworks tend to have higher scalability as the processor is not wasted on context switching between threads. This post summarizes my findings.

<!--more-->

## Theory

A web application can handle incoming and outgoing requests using blocking or non-blocking I/O operation. The way that a single server handles requests affects its ability to scale and depends on a web framework that was used to build the web application. It is especially important for [Microservices Architecture Pattern](http://microservices.io/) based applications which consist of distributed REST web services.

A request can be handled in two ways: using blocking or non-blocking I/O operation. In blocking approach when a thread performs an I/O operation, such as writing to a TCP port, it is blocked until the operation is finished. The processor switches its execution context to other threads while awaiting for the blocked thread to be ready again. Non-blocking approach offers more efficient and scalable alternative. It uses event notification facilities to register interest in certain events, such as data being ready to read on a particular port. Instead of waiting for an operation to complete, a thread can work on something else and handle the result of the operation when an event occurs.

There are two strategies to ensure the parallel requests handling: thread pool and event driven. The first one assigns a different thread for each request and uses blocking I/O operations. The size of the thread pool defines the number of requests that can be handled simultaneously.The latter usually uses one thread for request handling and perform I/O operations in non-blocking way. In this strategy one thread is capable of handling many requests simultaneously. In thread pool strategy, when the size of the pool is large, overhead due to thread scheduling and context-switching can degrade the performance

## Benchmark

To evaluate performance I performed an experiment which measured the performance of simple microservices based apps developed using six different frameworks (Spray, Grizzly, Jetty, Resteasy, CXF, Restlet). The first three are based on NIO java library and are non-blocking. The REST endpoints implemented using them proxied the request to the reference external endpoint. The performance of each app was measured using [wrk](https://github.com/wg/wrk). I executed the benchmark agains two environments: local *i7* and AWS *m2.large* and different number of simultaneous connections. The apps and scripts can be found at [github](https://github.com/mbilski/benchmark).

![](/img/results.png)

## Summary

On all figures which show requests per second the non-blocking web frameworks have higher performance. In addition, from the analysis of the figures that show latency it can be said that the non-blocking web frameworks have higher scalability. The latency grows slower for non-blocking web framework with increasing number of concurrent connections than for blocking web frameworks. On both environments the performance of non-blocking web frameworks was about 2.5 times higher then blocking web frameworks.
