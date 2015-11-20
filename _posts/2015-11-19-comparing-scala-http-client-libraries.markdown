---
layout: blog
---

# Comparing Scala's HTTP client libraries

#### Or why should probably use Play-WS.

Today we're looking at the various Scala libraries available to do some HTTP requests. It's a very easy problem, but there's a bunch of projects on Github all hoping to do better than the previous one. Let's see how they really compare to each other.

This comparison is not about low-level performance or cleverness of implementation. We will be only talking about already long-lived and popular libraries : I assume they are all correct and perform decently. Instead, I'm focusing on the quality of the documentation, the ease-of-use of the API, and what the resulting code looks like.

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



How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How `do we` do that
How do we do that
How do we do that
How do we do that

How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that
How do we do that

{% highlight scala %}
private def callWithPlayWs(): Future[Unit] = {
  import play.api.libs.ws.ning.NingWSClient
  import scala.concurrent.ExecutionContext.Implicits.global
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
      wsClient.close()
    }
}
{% endhighlight %}


How do we do that
How do we do that
How do we do that
