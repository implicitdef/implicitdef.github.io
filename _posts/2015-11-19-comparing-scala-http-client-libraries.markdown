---
layout: blog-post
title: Comparing Scala's HTTP client libraries
---



# Comparing Scala's HTTP client libraries

#### Or why should probably use Play! WS


Today we're looking at the various Scala libraries available to perform HTTP requests. It's a very easy problem, yet there's a bunch of projects on Github all hoping to do better than the previous one. Let's see how they really compare to each other.

This comparison is not about low-level performance or cleverness of implementation. We will be only talking about already long-lived and popular libraries : I assume they are all correct and perform decently for an everyday usage. Instead, I'm focusing on the quality of the documentation, the ease-of-use of the API, and what the resulting code looks like.

Let's start where a Scala newcomer would start : googling. If one types "__scala http client__" into Google, he'll get in order :

1. [Dispatch](https://dispatch.databinder.net)
2. [Newman](https://github.com/stackmob/newman)
2. [scalaj-http](https://github.com/scalaj/scalaj-http)
2. [spray-client](http://spray.io/documentation/1.2.3/spray-client/)
2. [Play! WS API](https://www.playframework.com/documentation/2.4.x/ScalaWS)

Most of them give a one-line example of how to perform a very simple GET request in their docs. That's not sufficient.
In real-life, you always need something a bit more complex than what's in the doc.

So we're gonna do a little experiment. We will take a moderately complex request and see how each of these libraries can help us to do it. Here's our arbitrary requirements :

- We want to perform a GET request on `http://jsonplaceholder.typicode.com/comments/1`
- We want to add two query string parameters, i.e. `?some_parameter=some_value&some_other_parameter=some_other_value`
- We want to add some additional HTTP header. I randomly picked `Cache-Control : no-cache`
- We want to throw an exception if the API doesn't respond with an status code between 200 and 299
- We want to print both the response body as a string, and an arbitrary response header (I randomly picked `Content-Length`)

Nothing fancy, that should be easy.

### Dispatch

Dispatch looks good. First to pop-up in the results, and the doc [http://dispatch.databinder.net/](http://dispatch.databinder.net/) looks good, that's a good sign right ? Guess again.

{% highlight scala %}
libraryDependencies += "net.databinder.dispatch" %% "dispatch-core" % "0.11.3"
{% endhighlight %}

{% highlight scala %}
import dispatch._
import dispatch.Defaults._

// Instantiation of the client
// In a real-life application, you would instantiate it once, share it everywhere,
// and call h.shutdown() when you're done
val h = new Http
val requestWithHandler =
  // Defining the request
  url("http://jsonplaceholder.typicode.com/comments/1")
    .<<?(Seq("some_parameter" -> "some_value", "some_other_parameter" -> "some_other_value"))
    .<:<(Seq("Cache-Control" -> "no-cache"))
    // Requires a 2xx status code
    .OK { response =>
      // Defines a handler
      println(s"OK, received ${response.getResponseBody}")
      println(s"The response header Content-Length was ${response.getHeader("Content-Length")}")
}
// Executes it
h(requestWithHandler)
{% endhighlight %}

Sure it works, if you like hieroglyphs. Noticed the funky method names ? Here's a peek on a few other methods defined by the API : `/`, `/?`, `<<`, `<<<`, `as_!`. Good luck.

Oh, and actually this library is not maintained. The last meaningful commits seems to be from early 2013, and there's a bunch of unanswered issues on Github.

Diagnostic : __abandonware with a cryptic DSL__. Don't use it.

### Newman

From the github documentation [https://github.com/stackmob/newman](https://github.com/stackmob/newman) : "_Newman comes with a DSL which is inspired by Dispatch, but uses mostly english instead of symbols_". Should be good, right ? Well, maybe, but here too the project is abandoned. Last commits in 2014, and Scala 2.11 isn't supported.


Somebody made a quick fork ([https://github.com/megamsys/newman](https://github.com/megamsys/newman)) to support Scala 2.11, but without much info available. It's unlikely the original author wakes up and merges the fork, or that the fork gains enough traction to replace the original.

I still call it an abandonware. No point in going further.

Diagnostic : __abandonware__. Don't use it.

### scalaj-http

Scala-http ([https://github.com/scalaj/scalaj-http](https://github.com/scalaj/scalaj-http)) looks clean and maintained. However it is synchronous ! No Futures there. Each HTTP request _will_ block the thread.

In Scala it is quite idiomatic to do everything asynchronously, using synchronous code feels like a regression. It's probably a decent library but I can't see a use case for it. No point in going further.

Diagnostic : __synchronous__. Good to know this kind of stuff still exist somewhere. But if your app is anything more than a prototype, there's really no value in using it.

### spray-client

Now we are attacking the big players. [Spray](http://spray.io/) is a solid HTTP framework, split in multiple modules. One of them is [spray-client](http://spray.io/documentation/1.2.3/spray-client/).

{% highlight scala %}
libraryDependencies += "io.spray" %% "spray-client" % "1.3.1"
{% endhighlight %}

{% highlight scala %}
import akka.actor._
import spray.http._
import spray.client.pipelining._

// Start an Akka Actor System
// In a real-life webapp, you would use only one, share it everywhere,
// and call actorSystem.shutdown() when you're done
implicit val actorSystem = ActorSystem()
import actorSystem.dispatcher

val pipeline = sendReceive
pipeline(
  // Building the request
  Get(
    Uri(
      "http://jsonplaceholder.typicode.com/comments/1"
    ).withQuery("some_parameter" -> "some_value", "some_other_parameter" -> "some_other_value")
  )
    .withHeaders(HttpHeaders.`Cache-Control`(CacheDirectives.`no-cache`))
)
  .map { response =>
    // Treating the response
    if (response.status.isFailure) {
      sys.error(s"Received unexpected status ${response.status} : ${response.entity.asString(HttpCharsets.`UTF-8`)}")
    }
    println(s"OK, received ${response.entity.asString(HttpCharsets.`UTF-8`)}")
    println(s"The response header Content-Length was ${response.header[HttpHeaders.`Content-Length`]}")
  }
{% endhighlight %}

A few observations there.

First, spray-client, like everything in Spray, is heavily tied to Akka. You need an actor system. If your app is not already built on Akka, it's not ideal. Sure, you can declare an actor system just for your HTTP requests, but it feels a bit cumbersome.

Second, the API is convoluted. Take a look at the first line :
{% highlight scala %}
val pipeline = sendReceive
{% endhighlight %}
You probably thought that `sendReceive` is a variable being assigned to `pipeline`. Nope. The `pipeline` variable is of type `SendReceive`, and `sendReceive` is actually a _method_, that builds a `SendReceive` object. It would be better to add parentheses to clarify that :
{% highlight scala %}
val pipeline = sendReceive()
{% endhighlight %}
But you can't, because the method takes implicit parameters (the actor system and the execution context). Adding the parentheses would require to pass those explicitly.

OK, nevermind. What about this type, `SendReceive`, what is it ? Well it's actually a type alias for a function type :
{% highlight scala %}
type SendReceive = HttpRequest ⇒ Future[HttpResponse]
{% endhighlight %}
So the `sendReceive` that you start with is actually a function that produces a function. Feeling confused ? It's OK, it's because it is confusing. Now I'm not saying functions that produce functions are bad inherently, it can very useful to express complex logic. What I'm saying is that if your API uses that as the building block for _even just a simple HTTP call_, your API is convoluted.

To be thorough, I should mention that Spray actually advocates a different writing style, based on a DSL and method composition. From the doc :
{% highlight scala %}
val pipeline: HttpRequest => Future[OrderConfirmation] = (
  addHeader("X-My-Special-Header", "fancy-value")
  ~> addCredentials(BasicHttpCredentials("bob", "secret"))
  ~> encode(Gzip)
  ~> sendReceive
  ~> decode(Deflate)
  ~> unmarshal[OrderConfirmation]
)
{% endhighlight %}
We're back to hieroglyphs. I'm sure once you've mastered this DSL (which is barely documented), it will probably look very clean and functional. In the meantime, if you just want to do _a simple HTTP call_, and don't want to spend a few hours deciphering spray-client's code source to grasp the API, use something else.

Diagnostic : __solid but over-engineered__. Use it if you want to feel clever.

### Play! WS

I saved the best for last. [Play!](https://www.playframework.com/) is a massive Scala web framework. It provides a bunch of useful stuff, and while less modular than Spray, some stuff can be used independently. One of them is the [WS API](https://www.playframework.com/documentation/2.4.x/ScalaWS). It is oddly named (who calls HTTP APIs _webservices_ these days ?) and not really advertised, but it's pretty decent.

{% highlight scala %}
libraryDependencies += "com.typesafe.play" %% "play-ws" % "2.4.3"
{% endhighlight %}
{% highlight scala %}
import play.api.libs.ws.ning.NingWSClient
import scala.concurrent.ExecutionContext.Implicits.global

// Instantiation of the client
// In a real-life application, you would instantiate one, share it everywhere,
// and call wsClient.close() when you're done
val wsClient = NingWSClient()
wsClient
  .url("http://jsonplaceholder.typicode.com/comments/1")
  .withQueryString("some_parameter" -> "some_value", "some_other_parameter" -> "some_other_value")
  .withHeaders("Cache-Control" -> "no-cache")
  .get()
  .map { wsResponse =>
    if (! (200 to 299).contains(wsResponse.status)) {
      sys.error(s"Received unexpected status ${wsResponse.status} : ${wsResponse.body}")
    }
    println(s"OK, received ${wsResponse.body}")
    println(s"The response header Content-Length was ${wsResponse.header("Content-Length")}")
  }
{% endhighlight %}

Overall, the API is easy to understand, everything is self-explanatory.

Cons : the documentation is awful for our use-case. It is exclusively targeted at people using it inside the Play! framework, even if it's perfectly usable outside of it.

Pro : you don't need the doc. The methods are well named, just use the auto-completion in your favorite IDE and you will find what you're looking for quickly.

Diagnostic : __good__. Could really use a bit more visibility outside of the Play! world.

### Conclusion

Today we learned a few things.

First, the Google ranking is not an accurate metric to evaluate the quality of a library :)

Second, Github should really display a big red flag over a repo when it has not been updated in over a year.

Third, that there is actually only two serious HTTP client libraries in Scala : __spray-client__ and __Play! WS__.

And finally : that you probably want the latter.
