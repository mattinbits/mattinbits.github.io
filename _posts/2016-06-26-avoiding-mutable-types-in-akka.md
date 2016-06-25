---
layout: post
title: Avoiding Mutable types in Akka, by Example
---

Early on in my experience programming with [Akka](http://akka.io/),
I came across the post
["Attention: Seq Is Not Immutable"](http://hseeberger.github.io/blog/2013/10/25/attention-seq-is-not-immutable/).
From that point on, I've always been careful to use `scala.collection.immutable.Seq`
rather than `scala.collection.Seq` when designing messages to pass between actors,
and in general to ensure messages are immutable.

Why is this important in Akka? Let's have a look at an example.

{% highlight scala %}
case class Message(parts: Seq[String])

class Receiver extends Actor {

  import context.dispatcher

  case object DoProcess

  var parts: Option[Seq[String]] = None

  override def receive: Receive = {

    case Message(p)  =>
      parts = Some(p)
      context.system.scheduler.scheduleOnce(5 seconds, self, DoProcess)

    case DoProcess =>
      parts.foreach(_.foreach(println))
      context.system.terminate()
  }
}
{% endhighlight %}

We have a simple message that contains a sequence of `String`. Since we have not
imported `scala.collection.immutable.Seq` this is a mutable sequence. The Receiver
actor, on receiving a message, assigns the sequence to a `var` and schedules the
output of the sequence for 5 seconds later. The schedule is present to give us
a chance to mutate the sequence from outside of the actor before the output occurs.

We have a `Sender` actor which sends a sequence to the receiver, then after 1 second
mutates the sequence by adding another element to it:

{% highlight scala %}
case class Start(receiver: ActorRef)

class Sender extends Actor {

  import context.dispatcher

  case object Mutate

  val buffer = ArrayBuffer("A", "B", "C")

  override def receive: Receive = {

    case Start(rec) =>
      rec ! Message(buffer)
      context.system.scheduler.scheduleOnce(1 seconds, self, Mutate)

    case Mutate => buffer += "D"

  }
}
{% endhighlight %}

Then all we need is some simple bootstrapping code to run the program:

{% highlight scala %}
object Example extends App {

  val system = ActorSystem("Example")

  val receiver = system.actorOf(Props[Receiver])
  val sender = system.actorOf(Props[Sender])

  sender ! Start(receiver)
}
{% endhighlight %}

The full project is available [on Github](https://github.com/mattinbits/blog_examples/tree/master/akka-mutable-seq).
Running it with `sbt run` yields the following output:

    A
    B
    C
    D

The `Sender` actor has been able to mutate the contents of the message after sending it.
In this case, because of the 5 second delay in the Receiver, the output is predictable. It is
almost certain (except for any very inaccurate behaviour from the scheduler) that the mutation
occurs before the output. Without the scheduling delays the behaviour would be highly unpredictable
and would depend on how the actors are scheduled by the actor system. This goes against the actor model
and is usually highly undesirable.

Even if you have a use-case where you think it might be desirable to mutate a message after sending
it, deliberately violating the concept of immutable messages between Actors, there is another reason
to avoid doing so. One of the advantages of the Actor module is location transparency; the application
behaves the same way whether it is running in a single JVM, multiple JVMs on a single machine,
or across different machines.

To run the example between two different JVMs, run the following two commands in two terminals:


    sbt 'runMain com.mjlivesey.mutableakka.RemoteExample receiver 2552 2553'
    sbt 'runMain com.mjlivesey.mutableakka.RemoteExample sender 2552 2553'


The receiver will yield:

    A
    B
    C

When remoting, the message is serialized in the sender, transmitted to the receiver,
and deserialized. The sequence on the receiver is therefore not the same object
as the sequence on the sender and so mutating the sequence in the sender has no effect.
This means we have an application which operates differently when we try and scale it using
Akka remoting.

If you develop an application which relies on mutating messages after they are sent, that
application will not be able to scale when required, losing one of the advantages of using
Akka and the Actor model. Use immutable messages in all cases, and make sure you're using
`scala.collection.immutable.Seq`.

PS if you're using Akka with Java, you might be interested in my post about [achieving immutability
in Java](2016-04-27-java-for-scala-immutable.md).
