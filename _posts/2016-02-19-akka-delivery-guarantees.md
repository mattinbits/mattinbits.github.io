---
layout: post
title: Akka Delivery Guarantees by example
---

In this post, the delivery guarantee provided by Akka is explored by example,
demonstrating techniques for handling unreliable message delivery.

This post assumes you a familiar with basic Akka programming in Scala. If not,
consider [reading this first](http://alvinalexander.com/scala/simple-scala-akka-actor-examples-hello-world-actors).

The [documentation for Akka](http://doc.akka.io/docs/akka/2.4.1/general/message-delivery-reliability.html)
describes three types of message delivery guarantee:

> - at-most-once delivery means that for each message handed to the mechanism, that message is delivered zero or one times; in more casual terms it means that messages may be lost.
> - at-least-once delivery means that for each message handed to the mechanism potentially multiple attempts are made at delivering it, such that at least one succeeds; again, in more casual terms this means that messages may be duplicated but not lost.
> - exactly-once delivery means that for each message handed to the mechanism exactly one delivery is made to the recipient; the message can neither be lost nor duplicated.

The same page explains that Akka offers the first level, at-most-once delivery. The
justification for this is explained well by the classic post,
["Nobody Needs Reliable Messaging"](http://www.infoq.com/articles/no-reliable-messaging).

Essentially, the argument is that what it means to have at-least-once or exactly-once
delivery is specific to the business logic of an application. So it is up to each
application to build on the underlying at-most-once provision in a way which is
specific to its own needs.

## Why are messages lost?

If you develop an application and run it on a single machine, or even clustered
across multiple machines on a reliable network, you could get the impression
that Akka provides exactly-once semantics. This is simply because a situation where a message
gets lost requires some form of failure that has never occurred. However relying
on this as you scale up is dangerous. As an application increases in traffic or is
distributed across more machines, the chances of failure increase significantly.

Let's look at some code. The examples here are available on
 [Github](https://github.com/mattinbits/blog_examples/tree/master/akka-delivery-semantics).
The Sender Actor sends 100 messages, with a small delay between each:

{% highlight Scala %}
case object CountMe
case object Finished

class Sender(receiver: ActorRef) extends Actor with ActorLogging {

  import context.dispatcher

  case object Send

  var numSent = 0

  override def preStart(): Unit = {
    scheduleNext()
  }

  def receive = {
    case Send =>
      receiver ! CountMe
      numSent = numSent + 1
      log.info(s"Sent message ${numSent}")
      if(numSent == 100) {
        log.info("Sender sent 100 messages")
        receiver ! Finished
      } else
        scheduleNext()
  }

  def scheduleNext() = context.system.scheduler.scheduleOnce(10 millis, self, Send)
}
{% endhighlight %}

The Receiver Actor receives these and increments a counter:

{% highlight Scala %}
class Receiver extends Actor with ActorLogging {

  def countingReceiver(current: Int): Receive = {

    case CountMe =>
      log.info(s"Received message number ${current+1}")
      context.become(countingReceiver(current+1))

    case Finished =>
      log.info(s"Receiver received ${current} messages.")
      context.system.terminate()
  }

  def receive = countingReceiver(0)
}
{% endhighlight %}

When this is run (using the following command if you have the project
cloned from
[Github](https://github.com/mattinbits/blog_examples/tree/master/akka-delivery-semantics)):

    sbt 'run basic akka.actor.default-mailbox akka.actor.default-mailbox'

The output is:

     [akka://SendReceive/user/$a] Received message number 1
     [akka://SendReceive/user/$b] Sent message 1
     [akka://SendReceive/user/$b] Sent message 2
     [akka://SendReceive/user/$a] Received message number 2
     ...
     [akka://SendReceive/user/$b] Sent message 99
     [akka://SendReceive/user/$a] Received message number 99
     [akka://SendReceive/user/$b] Sent message 100
     [akka://SendReceive/user/$a] Received message number 100
     [akka://SendReceive/user/$b] Sender sent 100 messages
     [akka://SendReceive/user/$a] Receiver received 100 messages.

Everything seems fine, all messages received. So under what conditions can
this apparently reliable delivery fail?

On a distributed system, some causes are immediately obvious. A machine could
fail, someone could trip over an Ethernet cable. Since sending messages is an
asynchronous action, the sending actor gets no indication in these cases that
their message will not be delivered. However failures can occur within a single
JVM on a single machine as well. The Akka documentation describes some possible
causes:

 - Errors such as `StackOverflowError` or `OutOfMemoryError`
 - The receiving actor is using a bounded mailbox and it is full
 - The receiving actor fails during processing, or is already terminated

## Demonstrating message failure

We could try and alter our example to make any of these causes a possibility.
However a straightforward and reliable way to introduce failure is to introduce
a new mailbox type, which randomly drops a fraction of messages. The core of the
logic is shown below, the complete implementation is available in the
[Github](https://github.com/mattinbits/blog_examples/tree/master/akka-delivery-semantics)
repo.

{% highlight Scala %}
object UnreliableMailbox {

  class MyMessageQueue extends MessageQueue
  with UnreliableMailboxSemantics {
    private final val queue = new ConcurrentLinkedQueue[Envelope]()

    // 5% of messages are lost. Unless we're sending them to ourself
    def enqueue(receiver: ActorRef, handle: Envelope): Unit = {
      if(Random.nextDouble > 0.05 || receiver == handle.sender)
      queue.offer(handle)
    }

    def dequeue(): Envelope = queue.poll()

    def numberOfMessages: Int = queue.size

    def hasMessages: Boolean = !queue.isEmpty

    def cleanUp(owner: ActorRef, deadLetters: MessageQueue) {
      while (hasMessages) {
        deadLetters.enqueue(owner, dequeue())
      }
    }
  }
}
{% endhighlight %}

The rule that messages are not dropped at random when an actor sends to itself
is just to make the examples below simpler. In general, such messages also
have at-most-once delivery.

Let's run our example again, using this unreliable mailbox for the Receiver:

    sbt 'run basic akka.actor.default-mailbox unreliable'

    ...
    [akka://SendReceive/user/$a] Received message number 90
    [akka://SendReceive/user/$b] Sent message 99
    [akka://SendReceive/user/$a] Received message number 91
    [akka://SendReceive/user/$a] Received message number 92
    [akka://SendReceive/user/$b] Sent message 100
    [akka://SendReceive/user/$b] Sender sent 100 messages
    [akka://SendReceive/user/$a] Receiver received 92 messages.

## At-least-once with Ack-Retry

So the unreliable mailbox has failed to deliver some of the messages to the
Receiver, and there is no built-in remedy for this. We need to alter our
application in order to take account of this. The typical way to do this, is
with an Acknowledge and Retry approach. The Receiver must acknowledge each
message from the Sender. If the Sender does not receive the acknowledgement
within the time limit, it retries by sending the message again.

{% highlight Scala %}
class Sender(receiver: ActorRef) extends Actor with ActorLogging {

    import context.dispatcher

    case object Send
    case class InternalAck(id: Int)
    case object Timeout


    var numSent = 0
    var retry: Option[Cancellable] = None


    override def preStart(): Unit = {
      scheduleNext()
    }

    def sending(next: Int): Receive = {
      case Send =>
        numSent = numSent + 1
        log.info(s"Sending message ${next}")
        receiver ! CountMe(next)
        val timer = context.system.scheduler.scheduleOnce(50 millis, self, Timeout)
        context.become(waiting(next, timer))
    }

    def waiting(next: Int, timer: Cancellable): Receive = {

      case Ack(i) =>
        timer.cancel()
        log.info(s"Received Ack for message ${next}")
        if (i == 100) {
          log.info(s"Sender sent ${numSent} messages")
          receiver ! Finished
        } else {
          scheduleNext()
          context.become(sending(next+1))
        }

      case Timeout =>
        log.info(s"Did not receive Ack for message ${next}")
        scheduleNext()
        context.become(sending(next))
    }

    def receive = sending(1)

    def scheduleNext() = context.system.scheduler.scheduleOnce(10 millis, self, Send)
  }
{% endhighlight %}

This time the Sender alternates between two states, `sending` and `waiting`. After
sending each message, it waits for the acknowledgement. If received in time, the actor
moves on to sending the next message, otherwise it retries the current message.

The Receiver is also changed so that it sends the acknowledgements.

{% highlight Scala %}
class Receiver extends Actor with ActorLogging {

  def countingReceiver(counter: Int): Receive = {

    case CountMe(i) =>
      log.info(s"Received message number ${i}")
      sender ! Ack(i)
      context.become(countingReceiver(counter+1))

    case Finished =>
      log.info(s"Receiver received ${counter} messages.")
      context.system.terminate()

    case other => log.error(s"Received ${other}")
  }

  def receive = countingReceiver(0)
}
{% endhighlight %}

Let's try this implementation, giving the Receiver an unreliable mailbox as before:

    sbt 'run ack akka.actor.default-mailbox unreliable'

    ...
    [akka://SendReceive/user/$b] Sending message 61
    [akka://SendReceive/user/$b] Did not receive Ack for message 61
    [akka://SendReceive/user/$b] Sending message 61
    [akka://SendReceive/user/$a] Received message number 61
    ...
    [akka://SendReceive/user/$b] Sender sent 106 messages
    [akka://SendReceive/user/$a] Receiver received 100 messages.

This looks good, the sender notices when messages were not received and resends them.
Although 106 messages were sent, exactly 100 were received. However,
it is not only messages from the Sender to Receiver which can be lost, the
acknowledgements themselves can be lost. Let's run this example with unreliable
mailboxes at each end:

    sbt 'run ack unreliable unreliable'

    ...
    [akka://SendReceive/user/$b] Sender sent 111 messages
    [akka://SendReceive/user/$a] Receiver received 106 messages.

Now the Receiver has received more messages that the Sender intended. We've
achieved at-least-once delivery, which is what an Ack-Retry solution typically
 delivers.

## Improving on At-least-once

In some cases, at-least-once is sufficient. However in other cases, receiving and
processing messages more than once could go against the required business logic
of the application. Imagine if each of our messages from Sender to Receiver
represents a person with a ticket entering a venue, and we need an accurate
count of the number of people in attendance. The implementation above would
over-count. Whereas at-least-once delivery is primarily the responsibility of
the Sender (other than the act of sending acknowledgements), achieving some sort
of exactly-once semantics is the responsibility of the Receiver. The solution
will always be very much application specific. In this case, let's set the
requirement that we must record exactly which of 100 ticket holders have arrived.

Exactly the same Sender as before will be used. The Receiver is changed so that
it holds a Map of `Int` to `Boolean`,  containing the numbers 1 to 100
all initialized to `false`. When receiving a message with an `Int` ID, we set
the corresponding `Boolean` to `true`. If a message is received multiple times,
the act of setting its flag to `true` multiple times is benign. This is an example
of [Idempotence](https://en.wikipedia.org/wiki/Idempotence), the property that
an action can be performed multiple times, without changing the outcome beyond the
initial instance. Ensuring that actions are Idempotent is a common way of
achieving exactly-once behaviour when messages will arrive with at-least-once
guarantees. Our Idempotent Receiver looks like this:

{% highlight Scala %}
class Receiver extends Actor with ActorLogging {

  var receipts: Map[Int, Boolean] = (1 to 100).map(i => i -> false).toMap

  def countingReceiver(counter: Int): Receive = {

    case CountMe(i) =>
      log.info(s"Received message number ${i}")
      receipts = receipts + (i -> true)
      sender ! Ack(i)
      context.become(countingReceiver(counter+1))

    case Finished =>
      log.info(s"Receiver received ${receipts.values.count(_ == true)} distinct messages.")
      context.system.terminate()

    case other => log.error(s"Received ${other}")
  }

  def receive = countingReceiver(0)
}
{% endhighlight %}

Let's run this with unreliable mailboxes at both ends:

    sbt 'run idempotent unreliable unreliable'

    ...
    [akka://SendReceive/user/$b] Sender sent 108 messages
    [akka://SendReceive/user/$a] Receiver received 100 distinct messages.

The Receiver has correctly kept track of which messages were received, even though
8 additional messages were sent.

## Conclusion

Although Akka is highly robust and often appears to offer reliable messaging in some deployment
situations, developers need to be aware that it offers at-most-once semantics.
Applications which require at-least-once or exactly-once delivery
must implement logic depending on their requirements.

The Ack-Retry solution presented here illustrates how application logic can
extend the inbuilt guarantees to handle certain failure scenarios, but
is not a complete solution for achieving at-least-once or exactly-once delivery.
For example if the Idempotent receiver were to restart due to an error
(which could be at the object level, JVM level, hardware level etc.) then its
state would be reset, losing the memory of which messages had been received
thus far. This can be remedied using
[Akka persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html)
which I will cover in a future post.

Here is a challenge for any readers with existing Akka code. If you configure the
Actors in your system to use a lossy mailbox like the one decribed here, does
it continue to operate as required?

Feedback and comments are welcome, either below or tweet me @mattinbits.
