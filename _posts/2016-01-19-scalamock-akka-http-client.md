---
layout: post
title: Mocking Akka Http on the Client-side
---

This is my first meaningful blog post and is inspired by
[This Stack Overflow](http://stackoverflow.com/questions/34714931/how-to-test-client-side-akka-http/34717677) question. I answered the question but wasn't entirely happy with it (and apparently
neither was the community!). So I decided to look at the problem in more depth
as I expect to use Akka Http in the future, and mocking is an important part
of testing components which connect to external services.

In this case, the client is to connect to the
[Police UK Data service for street level crimes](https://data.police.uk/docs/method/crime-street/),
chosen because it is a freely available data API, and fairly interesting,
so it's a reasonable target for a programmatic client.

The API accepts URLs such as:

    /api/crimes-street/all-crime?lat=52.62&lng=-1.13&date=2013-01

And returns JSON such as

{% highlight js %}
[  
  {  
    "category":"anti-social-behaviour",
    "location_type":"Force",
    "location":{  
      "latitude":"52.624424",
      "street":{  
        "id":882275,
        "name":"On or near Stuart Street"
      },
      "longitude":"-1.150485"
    },
    "context":"",
    "outcome_status":null,
    "persistent_id":"",
    "id":20597953,
    "location_subtype":"",
    "month":"2013-01"
  },
  {  
    "category":"burglary",
    "location_type":"Force",
    "location":{  
      "latitude":"52.632832",
      "street":{  
        "id":883513,
        "name":"On or near Dane Street"
      },
      "longitude":"-1.148312"
    },
    "context":"",
    "outcome_status":{  
      "category":"Investigation complete; no suspect identified",
      "date":"2013-01"
    },
    "persistent_id":"80e81a544ebfb96de7527e94d1e8498feca479607d3e2ab303a96a4694fd2ce0",
    "id":20600200,
    "location_subtype":"",
    "month":"2013-01"
  }
]
{% endhighlight %}

The output is a list of 'Crime' objects. For simplicity the client does not attempt to model
the entire set of fields per Crime, but aims to populate the following case class:

{% highlight Scala %}
case class Crime(category: String,
                 street: String,
                 outcome: Option[String])
{% endhighlight %}

The full deserialization code (which uses Spray JSON) is not shown here
since it is not the focus of this post, but the full source is available
on [Github](https://github.com/mattinbits/blog_examples).

{% highlight Scala %}
trait CrimeJsonProtocol extends DefaultJsonProtocol {

  implicit object crimeFormat extends JsonFormat[Crime] {
   ...
  }
}
{% endhighlight %}

In order to loosely couple the Http logic from the business logic, the
[Cake Pattern](http://www.cakesolutions.net/teamblogs/2011/12/19/cake-pattern-in-depth)
is used. First we have an Http Service trait:

{% highlight Scala %}
trait HttpRequestService {

  def makeRequest(uri: String): Future[String]
}
{% endhighlight %}

Since this is just a simple example, the `makeRequest` method takes a `String` URI
and returns a `Future` containing the response text from the server, also as a `String`.
Since a future can complete with `Success` or `Failure`,
it can represent any invalid HTTP responses
as failures. In a more complex scenario, a richer method signature would probably
be required, for example allowing the HTTP Method to be set, and allowing content to
be provided e.g. for PUT and POST requests. And it would make sense to return some
model object representing the Http response rather than just a `String`. However,
it is advisable not to design the signature to return the `HttpResponse` object from
a particular HTTP library, such as `akka.http.scaladsl.model.HttpResponse`, since
this makes it harder to switch out one `HttpRequestService` for another based
on a different HTTP Client library, and some of the benefit of the Cake Pattern would
be lost.

Next, we create a Trait representing the service implementing the API specific
logic:

{% highlight Scala %}

trait PoliceUKDataService extends CrimeJsonProtocol {

  this: HttpRequestService =>

  implicit def ec: ExecutionContext

  def getData(latitude: Double, longitude: Double, month: String): Future[Seq[Crime]] = {
    val uri = s"https://data.police.uk/api/crimes-street/all-crime?lat=${latitude}&lng=${longitude}&date=${month}"
    makeRequest(uri).map(_.parseJson.convertTo[List[Crime]])
  }
}
{% endhighlight %}

The following line is the essence of the Cake Pattern:

    this: HttpRequestService =>

It is called a "Self-type annotation" and it means that any class or object
which extends the `PoliceUKDataService` trait must also extend the
`HttpRequestService` trait (or a sub-trait of it). Essentially it dictates that
the `PoliceUKDataService` must always have access to the behaviour of an
`HttpRequestService` and an attempt to use `PoliceUKDataService` without mixing
in an `HttpRequestService` would cause a compile time error.

The implicit `ExecutionContext` is necessary since we are mapping the result of
the `Future` returned from `makeRequest`. Any operation with a `Future`
requires one.

It is desirable to unit test the `getData` method of `PoliceUKDataService`
without calling out to the Police Service API. Being able to do so is the
reason for this post. It is possible to go ahead and implement tests for it even before
having any working implementation of an `HttpRequestService`.

Here, [ScalaMock](http://scalamock.org/) is used to create a mock of the `makeRequest`
method.

{% highlight Scala %}
import org.scalamock.scalatest.MockFactory
import org.scalatest.{Matchers, WordSpec}
import scala.concurrent._
import scala.concurrent.duration._

class PoliceUKDataClientSpec extends WordSpec
  with Matchers with MockFactory {

  trait MockHttpRequestService
    extends HttpRequestService {

    val mock =
      mockFunction[String, Future[String]]

    override def makeRequest(uri: String): Future[String] =
      mock(uri)
  }

  "PoliceUKDataClient" should {

    "Return a list of crimes from a uri" in {

      val client = new PoliceUKDataService
        with MockHttpRequestService {

        override def ec =
          scala.concurrent.ExecutionContext.Implicits.global
      }
      client.mock
        .expects("https://data.police.uk/api/crimes-street/all-crime?lat=51.5&lng=0.13&date=2013-01")
        .returning(Future.successful(
          """
            |[
            |  {
            |    "category":"anti-social-behaviour",
            |    "location_type":"Force",
            |    "location":{
            |      "latitude":"52.624424",
            |      "street":{"id":882275,"name":"On or near Stuart Street"},
            |      "longitude":"-1.150485"
            |    },"context":"",
            |    "outcome_status":null,
            |    "persistent_id":"",
            |    "id":20597953,
            |    "location_subtype":"",
            |    "month":"2013-01"
            |  },
            |  {
            |    "category":"burglary",
            |    "location_type":"Force",
            |    "location":{
            |      "latitude":"52.627058",
            |      "street":{"id":882207,"name":"On or near Beaconsfield Road"},
            |      "longitude":"-1.154260"
            |    },
            |    "context":"",
            |    "outcome_status":{"category":"Investigation complete; no suspect identified","date":"2013-03"},
            |    "persistent_id":"",
            |    "id":20600818,
            |    "location_subtype":"",
            |    "month":"2013-01"
            |  }
            |]
          """.stripMargin))
      val crimesF = client.getData(51.5, 0.13, "2013-01")
      val crimes = Await.result(crimesF, 1 second)
      crimes should be (
        List(
          Crime("anti-social-behaviour",
            "On or near Stuart Street", None ),
          Crime("burglary",
            "On or near Beaconsfield Road",
            Some("Investigation complete; no suspect identified"))))
    }
  }
}
{% endhighlight %}

By implementing `makeRequest` as a mock, we can dictate what it returns for a given
input. Simple! Further tests e.g. for failure cases could be implemented in the same
way.

Now we have confidence in the correctness of our API Service, we can implement
our client application using Akka Http. First an implementation of the Http Service:

{% highlight Scala %}
trait AkkaHttpRequestService extends HttpRequestService {

  implicit def system: ActorSystem
  implicit def materializer: ActorMaterializer
  implicit def ec: ExecutionContext

  lazy val http = Http()

  def makeRequest(uri: String): Future[String] = {
    println(uri)
    val req = HttpRequest(HttpMethods.GET, uri)
    val resp = http.singleRequest(req)
    resp.flatMap {r => println(r); Unmarshal(r.entity).to[String] }
  }

  def shutdown() = {
    Http().shutdownAllConnectionPools().onComplete{ _ =>
      system.shutdown()
      system.awaitTermination()
    }
  }
}
{% endhighlight %}

Then we mix the API service and Akka Http Service together to build a client:

{% highlight Scala %}
object PoliceUKDataClient extends App {

  val client = new PoliceUKDataService
    with AkkaHttpRequestService {

    override implicit val system =
      ActorSystem("PoliceUKData")

    override implicit val materializer =
      ActorMaterializer.create(system)

    override implicit val ec: ExecutionContext =
      system.dispatcher
  }
  implicit val ec = client.ec
  val (lat, lon, date) = (args(0).toDouble, args(1).toDouble, args(2))
  client.getData(lat, lon, date).onComplete { res =>
    res match {
      case Success(list) => list.foreach(println)
      case Failure(ex) => println(ex.getMessage)
    }
    client.shutdown()
  }
}
{% endhighlight %}

The full source code for this post is available on [Github](https://github.com/mattinbits/blog_examples).

Comments and corrections always welcome.
