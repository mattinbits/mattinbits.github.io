---
layout: post
title: Doing Evil with Scala Macros
---

[Scala Macros](http://scalamacros.org/) have been part of the language since
Scala 2.10. Let's have some fun with them...

The full project accompanying this blog post is
[on github](https://github.com/mattinbits/blog_examples/tree/master/evil_macros).

Consider this simple group of functions:

{% highlight scala %}
@evil
object Operations {

  def addTwoNumbers(a: Int, b: Int): Int = a + b

  def maxOfThreeNumbers(a:Int, b: Int, c: Int): Int = math.max(math.max(a, b), c)

  def lengthOfString(s: String): Int = s.length
}
{% endhighlight %}

And associated simple tests:

{% highlight scala %}
import org.scalatest.{Matchers, WordSpec}

class OperationsSpec extends WordSpec with Matchers {

  import Operations._

  "Operations" should {

    "Add Two Numbers" in {
      addTwoNumbers(3, 4) should be (7)
    }

    "Find the max of three numbers" in {
      maxOfThreeNumbers(12, 23, 20) should be (23)
    }

    "Find the length of a string" in {
      lengthOfString("abc") should be (3)
    }
  }
}
{% endhighlight %}

Running these tests (with `sbt core/test`) gives:

```
[info] OperationsSpec:
[info] Operations
[info] - should Add Two Numbers *** FAILED ***
[info]   8 was not equal to 7 (OperationsSpec.scala:12)
[info] - should Find the max of three numbers *** FAILED ***
[info]   24 was not equal to 23 (OperationsSpec.scala:16)
[info] - should Find the length of a string *** FAILED ***
[info]   4 was not equal to 3 (OperationsSpec.scala:20)
```

This does not bode well for the future of Scala! However, observant readers may have
noticed the `@evil` annotation. An obvious culprit, let's see what it does.

In the 'macros' sub project in the git repo, the `@evil` annotation is defined as:

{% highlight scala %}
@compileTimeOnly("enable macro paradise to expand macro annotations")
class evil extends StaticAnnotation {
  def macroTransform(annottees: Any*) = macro EvilMacro.impl
}
{% endhighlight %}

`scala.annotation.StaticAnnotation` is a trait with no members. Extending `StaticAnnotation`
in this way defines an annotation with the same name as the class. The inclusion
of the method `def macroTransform(annottees: Any*)` informs the Scala macro engine
that this annotation is a Macro annotation. The `compileTimeOnly` annotation
on our custom annotation class ensures that the code will only work if we have
the [macro paradise plugin](https://github.com/scalamacros/paradise) enabled in our project.
While macros in general are supported natively in Scala, macro annotations require this plugin
in order to operate.

The result is that the annotation `@evil` is made available, and wherever it is used,
the macro defined by `EvilMacro.impl` will be executed during compilation.
The input to the Macro is the source tree of the artifact on which the annotation
was used. The Macro can alter the tree, therefore altering the code which results
from the compilation of the source.

Let's take a look at the implementation of `EvilMacro`:

{% highlight scala %}
import scala.reflect.macros.whitebox.Context

object EvilMacro {

  def impl(c: Context)(annottees: c.Expr[Any]*): c.Expr[Any] = {
    import c.universe._
    annottees match {
      case List(annottee @ Expr(
                  mod@ModuleDef(m, n, Template(p, s, b)))) =>
        //annottee.tree.children.foreach(println)
        val newTemplateChildren = b.map{
          case d @ DefDef(mods, name, types, params,
                      t @ Ident(TypeName("Int")), body) =>
            val amendedBody = q"($body) + 1"
            DefDef(mods, name, types, params, t, amendedBody)

          case other => other
        }
        val amendedModule =
          ModuleDef(m, n, Template(p, s, newTemplateChildren))
        c.Expr(q"{$amendedModule}")

      case other =>
        Expr
        println(s"Expecting single module")
        c.Expr(Block(annottees.map(_.tree).toList, Literal(Constant(()))))
    }
  }
}
{% endhighlight %}

Implementing a macro requires a context, either blackbox or whitebox. The difference
is explained [on this page](http://docs.scala-lang.org/overviews/macros/blackbox-whitebox.html)
but macro annotations are required to be whitebox so we import `scala.reflect.macros.whitebox.Context`.
An implementation of a macro is a curried function which accepts a context and a list
of expressions, and returns a new expression.

In this case we are implementing a macro specifically for annotating objects, so
we pattern match on the input being a list with a single member of type `ModuleDef`.

It can take some trial and error to find the types representing the intended code tree
when first implementing macros. The unapply methods for types such as `ModuleDef` are
contained in classes with names such as `ModuleDefExtractor`. Their scaladoc can be found
[here](http://www.scala-lang.org/files/archive/nightly/docs/library/index.html#scala.reflect.api.Trees).
The third parameter of the `ModuleDef` extractor
is a instance of `Template` and its third parameter is the list of expressions
within the object definition. We then iterate over each expression, matching specifically
on instances of `DefDef` (representing function definitions) with a return type of `Int`.

We redefine the definition of the the body of each such function, using the expression
`q"($body) + 1"`. The use of 'q' defines this as a [quasiquote](http://docs.scala-lang.org/overviews/quasiquotes/intro.html).
Quasiquote notation allows us to combine expressions with scala code strings
to form new expressions. In this case, we take whatever the existing body of the
function was, and add 1 to it. Then we reassemble the Template and ModuleDef.

Anything tagged with our `@evil` annotation is subject to this transformation at
compile time, so our functions change their behaviour and our tests fail. Perhaps a good
way to make sure your colleagues are paying attention during code review?
