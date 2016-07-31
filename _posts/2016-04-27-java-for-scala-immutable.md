---
layout: post
title: Java 8 for Scala developers part 1 - Immutability
---

Like many Scala developers I was developing object oriented, imperative code
in Java long before switching to Scala and learning the techniques of functional
programming. In the meantime, Java 8 has come along with some enhancements
to make functional programming in Java more natural. So as a Scala developer,
if required to switch back to Java, what can be done to maintain a functional style?

This is the first in a series of posts to address this question. Today, immutability.
The second part is on
[replacing Option and Try]({% post_url 2016-07-31-java-for-scala-option-try %}).

## Immutability

As an example, let's consider a simple scenario consisting of two classes,
`Publisher` and `Book`. Let's assume we want to make these classes immutable. For why
we might want to do so, consider
[this post](http://alvinalexander.com/scala/scala-idiom-immutable-code-functional-programming-immutability).
For a real world scenario, consider that may want to use the classes as
[messages in Akka](http://doc.akka.io/docs/akka/snapshot/java/untyped-actors.html#Messages_and_immutability).

### Scala representation

In Scala our model is:

{% highlight scala %}
case class Publisher(name: String, catalogue: List[Book]) {

  def withNewBook(b: Book): Publisher =
    this.copy(catalogue = this.catalogue :+ b)
}

case class Book(isbn: Long, title: String, author: String,
                datePublished: String, hardbacksSold: Int,
                paperbacksSold: Int) {

  def withAdditionalHardbackSale: Book =
    this.copy(hardbacksSold = this.hardbacksSold + 1)

  def withAdditionalPaperbackSale: Book =
    this.copy(paperbacksSold = this.paperbacksSold + 1)

}
{% endhighlight %}

What does using `case class` give us? Some of the things relevant here are:

- A default constructor consisting of all the parameters of the class.
- Read-only access to each parameter
- A `copy` method which we can use to build additional methods which produce
new instances with a transformation applied.

Case classes are designed for immutability. If we limit our model to compositions of the basic
built in types (`String`, `Long` etc) and other immutable classes,
we can construct an immutable model.

We also use `List`, which is an immutable collection. When building an immutable model
we must use immutable collections. The scala `List` type maps to `scala.collection.immutable.List`.
A full list of Scala immutable collections is available [here](http://docs.scala-lang.org/overviews/collections/concrete-immutable-collection-classes.html).

### Java representation

In Java there is no equivalent to `case class` to help us build immutable classes.
We do not need to implement all the features provided by Scala's `case class` syntactic
sugar in order to build an immutable class. The important things we need are:

- A constructor allowing us to set all the parameters at creation.
- Read-only accessors
- parameters themselves are immutable
- methods for returning new instances with transformations applied
- An absence of mutating methods

An initial attempt looks like:

{% highlight java %}
import java.util.ArrayList;
import java.util.List;

class Publisher {

    final private List<Book> catalogue;
    final private String name;

    public Publisher(String name, List<Book> catalogue) {
        this.name = name;
        this.catalogue = catalogue;
    }

    public List<Book> getCatalogue() {
        return catalogue;
    }

    public String getName() {
        return name;
    }

    public Publisher withNewBook(Book b) {
        List<Book> newCatalogue = new ArrayList<Book>(catalogue.size());
        Collections.copy(newCatalogue, catalogue);
        newCatalogue.add(b);
        return new Publisher(name, newCatalogue);
    }
}

class Book {

  public Long getIsbn() {
      return isbn;
  }

  public String getTitle() {
      return title;
  }

  public String getAuthor() {
      return author;
  }

  public String getDatePublished() {
      return datePublished;
  }

  public Integer getHardbacksSold() {
      return hardbacksSold;
  }

  public Integer getPaperbacksSold() {
      return paperbacksSold;
  }

  final private Long isbn;
  final private String title;
  final private String author;
  final private String datePublished;
  final private Integer hardbacksSold;
  final private Integer paperbacksSold;

  public Book(long isbn, String title, String author,
              String datePublished, Integer hardbacksSold,
              Integer paperbacksSold) {
      this.isbn = isbn;
      this.title = title;
      this.author = author;
      this.datePublished = datePublished;
      this.hardbacksSold = hardbacksSold;
      this.paperbacksSold = paperbacksSold;
  }

  public Book withAdditionalHardbackSale() {
      return new Book(isbn, title, author, datePublished,
        hardbacksSold+1, paperbacksSold);
  }

  public Book withAdditionalPaperbackSale() {
      return new Book(isbn, title, author, datePublished,
        hardbacksSold+1, paperbacksSold);
  }
}
{% endhighlight %}

Although much more verbose than the Scala version
this looks reasonable, there is no way to reassign the members of the classes after creation,
and we can create transformed instances. However there is a flaw using `java.util.List`.
This a mutable collection, so the code using these classes could pass in an instance of
`List`, then subsequently mutate it. Java does not have immutable collections built in
to its standard library, but we can use [
Google's Guava library](https://github.com/google/guava/wiki/ImmutableCollectionsExplained).

We can keep the interface to `Publisher` the same but use an immutable list internally:

{% highlight java %}
class Publisher {

  final private ImmutableList<Book> catalogue;
  final private String name;

  public Publisher(String name, List<Book> catalogue) {
      this.name = name;
      this.catalogue = ImmutableList.copyOf(catalogue);
  }

  public List<Book> getCatalogue() {
      return catalogue;
  }

  public String getName() {
      return name;
  }

  public Publisher withNewBook(Book b) {
      ImmutableList<Book> newCatalogue =
              ImmutableList.<Book>builder()
                      .addAll(catalogue)
                      .add(b)
                      .build();
      return new Publisher(name, newCatalogue);
  }
}
{% endhighlight %}

## Summary

For the same reasons that using Immutable representations of data is desirable
in Scala, so it is desirable in Java. Although we do not have the convenience of
`case class` in Java we can look to its behaviour as inspiration for good Immutable
implementations, in particular:

- Contructors allowing all parameters to be set.
- Read only accessors and no methods which mutate the class.
- Methods to produce new instances of objects with transformations.
- All parameters should themselves be immutable.
- Use a third party library of immutable collections such as Guava.
