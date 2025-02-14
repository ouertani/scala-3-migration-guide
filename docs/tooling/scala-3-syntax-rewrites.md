---
id: scala-3-syntax-rewrites
title: Scala 3 Syntax Rewriting
---
## Introduction

Scala 3 gives Scala developers the option to adopt the new (and optional)
_significant indentation based syntax_ (_SIB_ syntax). The existing syntax which
uses curly braces to group expressions remains fully supported, and we will
refer to this syntax as _Classic syntax_.

Even though we would advise against it, a single code base might mix the two
syntax versions.
A better approach is to choose one and to consistently apply it to all
source code in a code base.

Scala 3 also introduces a new syntax for control structures. The changes
introduced with this new syntax apply to `if`-expressions, `while`-loops, and
`for`-expressions.

## Syntax rewriting with the Scala 3 compiler

Converting existing code to use the new syntax by hand would be a tedious
and error-prone task. The good news is that the Scala 3 compiler can do the
hard work for us!

There are 4 possible combinations of the syntax options we covered so far,
and they are all valid.


|  | Classic Syntax | Significant Indentation Based Syntax |
| :--- | :-------: | :-----: |
| **Classic control structures**| ✅ | ✅ |
| **New control structures**| ✅ | ✅ |

Let's start with showing the compiler options we have available to achieve our goal.
If we simply type `scalac` on the command line it shows the usage, including all the options.
With the Scala 3 compiler you will find the following five among those options:

```bash
$ scalac
Usage: scalac <options> <source files>
where possible standard options include:
...
-indent</b>            Allow significant indentation
...
-new-syntax</b>        Require `then` and `do` in control expressions.
-noindent</b>          Require classical {...} syntax, indentation is not significant.
...
-old-syntax</b>        Require `(...)` around conditions.
...
-rewrite</b>           When used in conjunction with a `...-migration` source version,
                   rewrites sources to migrate to new version.
...

```

We can combine one of four syntax related options:`-indent`, `-noindent`, `-new-syntax`,
and `-old-syntax`, with the `-rewrite` option to rewrite source code.

To apply these options in an sbt project, set `scalacOptions`, for example by adding the line:<br>
`scalacOptions ++= Seq("-indent","-rewrite")`

Let's have a look at how this works by using a small example.

## Rewriting to _SIB_ syntax & new Control Structure syntax

Given the following source code that uses the classic syntax and classic
control structure syntax:

```scala
object Counter {
  enum Protocol {
    case Reset
    case MoveBy(step: Int)
  }
}

case class Animal(name: String)

trait Incrementer {
  def increment(n: Int): Int
}

case class State(n: Int, minValue: Int, maxValue: Int) {
  def inc: State =
    if (n == maxValue)
      this
    else
      this.copy(n = n + 1)
  def printAll: Unit = {
    println("Printing all")
    for {
      i <- minValue to maxValue
      j <- 0 to n
    } println(i + j)
  }
}
```

Assume that we want to convert this piece of code to _SIB_ syntax & the new control syntax.
Note that we cannot do this in a single step as the compiler disallows simultaneous rewrites of control syntax and _SIB_/Classic syntax. In other words, the following
combinations of compiler options are not allowed and each results in an error like the one shown:
- <del>-indent -old-syntax</del> -rewrite
- <del>-no-indent -old-syntax</del> -rewrite
- <del>-indent -new-syntax</del> -rewrite
- <del>-noindent -new-syntax</del> -rewrite

``` bash
$ scalac -indent -old-syntax -rewrite hello.scala
-- Error: hello.scala:1:0 ------------------------------------------------------
1 |
  |^
  |illegal combination of -rewrite targets: -old-syntax and -indent
1 error found
```

Instead we have to rewrite syntax one step at a time.

Let's start by moving to _SIB_ syntax. We compile the code with options `-indent -rewrite` and the result looks as follows:

```scala
object Counter:
  enum Protocol:
    case Reset
    case MoveBy(step: Int)

case class Animal(name: String)

trait Incrementer:
  def increment(n: Int): Int

case class State(n: Int, minValue: Int, maxValue: Int):
  def inc: State =
    if (n == maxValue)
      this
    else
      this.copy(n = n + 1)
  def printAll: Unit =
    println("Printing all")
    for {
      i <- minValue to maxValue
      j <- 0 to n
    } println(i + j)
```

A few things to observe after the switch to _SIB_ syntax:

- the number of lines was reduced by 4 because of the elimination of
  a series of closing curly braces
- the classic control structure syntax is unchanged

After this first rewrite, we can have the compiler rewrite the control
structures to the new syntax by specifying the `-new-syntax -rewrite`.
This results in the following version:

```scala
object Counter:
  enum Protocol:
    case Reset
    case MoveBy(step: Int)

case class Animal(name: String)

trait Incrementer:
  def increment(n: Int): Int

case class State(n: Int, minValue: Int, maxValue: Int):
  def inc: State =
    if n == maxValue then
      this
    else
      this.copy(n = n + 1)
  def printAll: Unit =
    println("Printing all")
    for
      i <- minValue to maxValue
      j <- 0 to n
    do println(i + j)
```

> ### Converting code using a project build
>
> Converting source files one by one using the Scala 3 compiler, is time consuming
> (and requires you to specify the class path which can be a hassle). A more
> efficient way to do this in bulk is to recompile your code as part of your
> build. For example, if you have an sbt based build, simply add the required
> options to `Compile / scalacOptions` and do a clean compilation of your project.

## Moving back to Classic syntax

Starting from the final result in the previous section, we can move "back"
towards the initial state of our sample code.

Let's rewrite to Classic syntax and retain the new Control Structures syntax.
We can do that by a single rewrite and the `-no-indent -rewrite` compiler
options with the following result:

```scala
object Counter {
  enum Protocol {
    case Reset
    case MoveBy(step: Int)
  }
}

case class Animal(name: String)

trait Incrementer {
  def increment(n: Int): Int
}

case class State(n: Int, minValue: Int, maxValue: Int) {
  def inc: State =
    if n == maxValue then
      this
    else
      this.copy(n = n + 1)
  def printAll: Unit = {
    println("Printing all")
    for {
      i <- minValue to maxValue
      j <- 0 to n
    }
    do println(i + j)
  }
}
```

Applying one more rewrite, with compiler options `-old-syntax -rewrite`, will
take us back to our original code (with the possible exception of some
differences in formatting):

```scala
object Counter {
  enum Protocol {
    case Reset
    case MoveBy(step: Int)
  }
}

case class Animal(name: String)

trait Incrementer {
  def increment(n: Int): Int
}

case class State(n: Int, minValue: Int, maxValue: Int) {
  def inc: State =
    if (n == maxValue)
      this
    else
      this.copy(n = n + 1)
  def printAll: Unit = {
    println("Printing all")
    for {
      i <- minValue to maxValue
      j <- 0 to n
    }
    println(i + j)
  }
}
```

And with this last rewrite, we have come full circle.

> ### Loss of formatting when cycling through syntax versions
>
> When formatting tools such as [scalafmt](https://scalameta.org/scalafmt) are
> used to apply customised formatting to your code, cycling back and forth between
> different Scala 3 syntax variants may result in differences when going full circle.
