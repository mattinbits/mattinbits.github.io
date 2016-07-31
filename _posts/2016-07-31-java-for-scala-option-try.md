---
layout: post
title: Java 8 for Scala developers part 2 - Functional Option and Try
---

This is the second part of a series of posts looking at how features in
Java 8 can help Scala developers stick to a functional style if they
find themselves needing to develop Java code. The first part was about
[handling immutability]({% post_url 2016-04-27-java-for-scala-immutable %}).

## Option/Optional

In Java it is common for methods which return a result which may not exist,
to return null in the case where the result does not exist. Consider an
example from
[java.util.Map](https://docs.oracle.com/javase/8/docs/api/java/util/Map.html#get-java.lang.Object-)
in the standard library.

The `get` method:

> Returns the value to which the specified key is mapped, or null if this map contains no mapping for the key.

This means that a developer using the the `get` method should, in order to have
robust code, protect against the possibility of the value being null. This is fine
if the developer reads the documentation to understand the implicit contract, and
knows how to appropriately handle the `null` case,
but it would be better if the signature of the `get` method enforced that the
developer must handle the possibility of no result.

Scala developers will be familiar with the
[`scala.collection.Map` interface](http://www.scala-lang.org/api/2.11.8/#scala.collection.Map).
A map of type `T` has a `get` method which returns `Option[T]`. The `Option` type
is either an instance of Some[T] from which the value of type `T` can be extracted,
or it is of type `None`. The developer using a `Map` has no choice but to handle the
`Option` type, and most importantly the signature of the `get` method makes it unambiguously
clear that this method may fail to return a value.

Java 8 introduces the `Optional` type, which serves a similar purpose to `Option`
in Scala. The javadoc is
[here](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html).
It doesn't have the subtypes of `Some` and `None` but since Java 8 doesn't have
pattern matching this is not too much of a loss. `Optional` has familiar functional
operators such as `flatMap`, `map`, `filter` etc.

We can't change the fact that some Java library classes use `null` to indicate the
lack of a value to return, but we can choose to use it in Java code we write for
ourselves.

Consider the following code:

{% highlight java %}
class Person {

    private String firstName;
    private String middleName;
    private String lastName;

    public Person(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }

    public Person(String firstName, String middleName, String lastName) {
        this.firstName = firstName;
        this.middleName = middleName;
        this.lastName = lastName;
    }

    public String getFirstName() {
        return firstName;
    }

    public String getMiddleName() {
        return middleName;
    }

    public String getLastName() {
        return lastName;
    }
}
{% endhighlight %}

If the second constructor is used, the `Person` object will have a `null` value for
`middleName`. Nothing about the `getMiddleName` method indicates this however. Consider the
following code:

{% highlight java %}
public static String renderPersonUpperCase(Person p) {
    return p.getFirstName().toUpperCase() +
            " " + p.getMiddleName().toUpperCase() +
            " " + p.getLastName().toUpperCase();
}

Person person1 = new Person("Billy", "Joe", "Armstrong");
Person person2 = new Person("Trent", "Reznor");
System.out.println(renderPersonUpperCase(person1));
System.out.println(renderPersonUpperCase(person2));
{% endhighlight %}

This causes the familiar and dreaded `java.lang.NullPointerException`. Of course we
could code around this but we get no help from `getMiddleName` to know that it may
return `null`.

With `Optional` we can rearrange our code as follows:

{% highlight java %}
class Person {

    private String firstName;
    private Optional<String> middleName = Optional.empty();
    private String lastName;

    public Person(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }

    public Person(String firstName, String middleName, String lastName) {
        this.firstName = firstName;
        this.middleName = Optional.of(middleName);
        this.lastName = lastName;
    }

    public String getFirstName() {
        return firstName;
    }

    public Optional<String> getMiddleName() {
        return middleName;
    }

    public String getLastName() {
        return lastName;
    }
}
{% endhighlight %}

Then, we are forced to handle the fact that `getMiddleName` returns
`Optional<String>`:

{% highlight java %}
public static String renderPersonUpperCase(Person p) {
    return p.getMiddleName().<String>map((String m) ->
                    p.getFirstName().toUpperCase() +
                            " " + m.toUpperCase() +
                            " " + p.getLastName().toUpperCase()).
                    orElse(p.getFirstName().toUpperCase() +
                            " " + p.getLastName().toUpperCase());
}
{% endhighlight %}

If using `map` and `orElse` in Java seems a bit clunky, the same thing could
be achieved in a more imperative style using and if statement with `isPresent()`
but the important thing is that by using `Optional` in our method signature, a developer
using that code is forced to handle the possibility of the value being empty,
failure to do so can be caught at compile time (i.e. if we tried to use `Optional<String>` as
  a `String`) rather than receiving a `NullPointerException` at run-time.

## Try

In `Scala`, all exceptions are unchecked, meaning that there is no need
to use a `throws` block to to declare that a method may throw a checked exception.
In fact where declaring a method throws an exception is required in order to provide
compatibility with Java code it is necessary to
[use an annotation](http://alvinalexander.com/scala/how-to-declare-scala-methods-throws-exceptions).

For similar reasons as those stated above for why it is beneficial to return an `Option` wrapping
the return type rather than returning either the return type or `null`, in Scala we can
use the `Try` class to make it explicit that a method may fail.

Consider the following example, where we use a `Try` to represent the fact that
a `divide` operation may fail if we try to divide by zero. We can match on that result
to handle it functionally.

{% highlight scala %}
import scala.util.{Failure, Success, Try}

object TryExample extends App {

  def divide(a: Int, b: Int): Try[Int] = Try(a/b)

  def divideAndPrint(a: Int, b: Int) = {
    divide(a, b) match {
      case Success(result) => println(s"$a divided by $b is $result")
      case Failure(ex: ArithmeticException) => println("Arithmetic Exception")
      case Failure(ex: Throwable) => println(s"unexpected Exception ${ex.getMessage}")
    }
  }

  divideAndPrint(4, 2)
  divideAndPrint(1, 0)
}
{% endhighlight %}

In Java, since `ArithmeticException` is unchecked we're not obliged to use a `throws`
declaration on a similar method, but without a `Try` class a similar method fails
to communicate the possibility of an Exception:

{% highlight java %}
public static Integer divide(Integer a, Integer b) {
    return a / b;
}
{% endhighlight %}

There is no built-in equivalent of `scala.util.Try` but if we want similar functional
exception handling, one option is to use `com.aol.cyclops.control.Try` from the
[Cyclops React Library](https://github.com/aol/cyclops-react). We can wrap the
unsafe divide method defined above:

{% highlight java %}
public static Try<Integer, ArithmeticException> safeDivide(Integer a, Integer b) {
  return Try.catchExceptions(ArithmeticException.class).tryThis(
      new Try.CheckedSupplier<Integer, ArithmeticException>() {
          @Override
          public Integer get() throws ArithmeticException {
              return divide(a, b);
          }
      });
}
{% endhighlight %}

It's much more verbose than the `Scala` equivalent, but `Java` usually is!
Verbosity aside, this has allowed us to explicitly declare in the return type
that the method may fail. The Cyclops React `Try` class contains functional operators
for mapping and flatmapping over `Try` instances, but one simple way to handle instances
is with a simple `if-else` block.

{% highlight java %}
public static void printAndDivide(Integer a, Integer b) {
    Try<Integer, ArithmeticException> result = safeDivide(a, b);
    if(result.isSuccess()) {
        System.out.println(a + "divided by " + b + " equals " + result.get());
    } else {
        System.out.println("Arithmetic Exception thrown");
    }
}
{% endhighlight %}

## Conclusion

In Scala, where a method may fail to return a result we use `Option` rather than
returning `null`, and where a method may fail we use `Try` rather than allowing
expections to occur unchecked. This is a good approach, especially when designing
an API, since it means it is explicit to users of methods that they need handle
the possibility of no result, or of an Exception.

In Java 8, we can use the built-in `Optional` type in place of Scala's `Option`
and we can use the third-party `com.aol.cyclops.control.Try` in place of
`scala.util.Try`.
