---
layout: blog
---

# Comparing Scala's HTTP client libraries

#### Or why should probably use Play-WS.

When writing back-end code in any language, some tasks come repetitively. You will probably have to create a HTTP server, connect to a database, and maybe interrogate another API via HTTP. Today we're looking at the various Scala libraries available to do this last task. It's a very easy problem, but there's a bunch of projects on Github all hoping to do better than the previous one. Let's see how they really compare to each other.

This comparison is not about low-level performance or cleverness of implementation. We will be only talking about already long-lived and popular libraries, so let's assume they are all correct and perform decently for an everyday usage. Instead, I'm focusing deliberately on the quality of the documentation, the ease-of-use of the API, and what the resulting code looks like.

What happens when one try to get a glimpse of the available options and starts at Google ? Let's see.








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
