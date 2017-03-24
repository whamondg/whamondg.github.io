---
layout: post
title: Dozing with Ratpack
---

Recently I've been playing around with performance monitoring tools and wanted to put together a web service with poor performance to act as a test end-point that could be monitored.  It seems like a good opportunity to play with Ratpack as well, something I've been meaning to do for a while.

The web service (I'm calling it Dozy) doesn't need to do much.  Simply sleep for a random amount of time between a configured minimum and maximum and then return a response.  It seems reasonable for the response to include how long the service has slept for.

## Ratpack

Ratpack is a set of Java libraries that can be used to build fast, scalable HTTP applications.  It's built on top of Netty, making use of its efficient non-blocking IO.  Ratpack is a runtime dependency and no installation is required. It doesn't bundle any build tooling, but support for Gradle is provided through plugins and the Ratpack documentation explains how to use it with Java or Groovy to create a simple application.

### Getting started

For a simple "Hello World" application using Gradle and Groovy we need just 2 files. 

The first is build.gradle:

{% gist whamondg/c80315f41103325ab50d build.gradle %}

The second is src/ratpack/ratpack.groovy:

{% gist whamondg/c80315f41103325ab50d ratpack.groovy %}

With these in place the application will be available at http://localhost:5050/ after executing

```$ gradle run```

### Next steps

So getting up and running with Ratpack and Gradle is pretty simple, but we dont want to run up using gradle

package as a JAR

then talk about sleeping


