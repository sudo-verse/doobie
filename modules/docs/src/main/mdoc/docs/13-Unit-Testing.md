## Unit Testing

The YOLO-mode query checking feature demonstrated in an earlier chapter is also available as a trait you can mix into your [Specs2](http://etorreborre.github.io/specs2/), [ScalaTest](http://www.scalatest.org/), [MUnit](https://scalameta.org/munit) or [Weaver](https://disneystreaming.github.io/weaver-test/) unit tests.

### Setting Up

As with earlier chapters we set up a `Transactor` and YOLO mode. We will also use the `doobie-specs2` and `doobie-scalatest` add-ons.

```scala mdoc:silent
import doobie._
import doobie.implicits._
import cats._
import cats.data._
import cats.effect._
import cats.implicits._

// This is just for testing. Consider using cats.effect.IOApp instead of calling
// unsafe methods directly.
import cats.effect.unsafe.implicits.global

// A transactor that gets connections from java.sql.DriverManager and executes blocking operations
// on an our synchronous EC. See the chapter on connection handling for more info.
val xa = Transactor.fromDriverManager[IO](
  driver = "org.postgresql.Driver",  // JDBC driver classname
  url = "jdbc:postgresql:world",     // Connect URL - Driver specific
  user = "postgres",                 // Database user name
  password = "password",             // Database password
  logHandler = None                  // Don't setup logging for now. See Logging page for how to log events in detail
)
```

```scala mdoc:invisible
implicit val mdocColors: doobie.util.Colors = doobie.util.Colors.None
```

And again we are playing with the `country` table, given here for reference.

```sql
CREATE TABLE country (
  code        character(3)  NOT NULL,
  name        text          NOT NULL,
  population  integer       NOT NULL,
  gnp         numeric(10,2),
  indepyear   smallint
  -- more columns, but we won't use them here
)
```

So here are a few queries we would like to check. Note that we can only check values of type `Query0` and `Update0`; we can't check `Process` or `ConnectionIO` values, so a good practice is to define your queries in a DAO module and apply further operations at a higher level.

```scala mdoc:silent
case class Country(code: Int, name: String, pop: Int, gnp: Double)

val trivial =
  sql"""
    select 42, 'foo'::varchar
  """.query[(Int, String)]

def biggerThan(minPop: Short) =
  sql"""
    select code, name, population, gnp, indepyear
    from country
    where population > $minPop
  """.query[Country]

val update: Update0 =
  sql"""
    update country set name = "new" where name = "old"
  """.update

```

### The Specs2 Package

The `doobie-specs2` add-on provides a mix-in trait that we can add to a `Specification` to allow for typechecking of queries, interpreted as a set of specifications.

Our unit test needs to extend `AnalysisSpec` and must define a `Transactor[IO]`. To construct a testcase for a query, pass it to the `check` method. Note that query arguments are never used, so they can be any values that typecheck.

```scala mdoc:silent
import org.specs2.mutable.Specification

class AnalysisTestSpec extends Specification with doobie.specs2.IOChecker {

  val transactor = Transactor.fromDriverManager[IO](
    driver = "org.postgresql.Driver", url = "jdbc:postgresql:world", user = "postgres", password = "password", logHandler = None
  )

  check(trivial)
  checkOutput(biggerThan(0))
  check(update)

}
```

When we run the test we get output similar to what we saw in the previous chapter on checking queries, but each item is now a test. Note that doing this in the REPL is a little awkward; in real source you would get the source file and line number associated with each query.

```scala mdoc
import _root_.specs2.{ run => runTest }
import _root_.org.specs2.main.{ Arguments, Report }

// Run a test programmatically. Usually you would do this from sbt, bloop, etc.
runTest(new AnalysisTestSpec)(Arguments(report = Report(_color = Some(false))))
```

### The ScalaTest Package

The `doobie-scalatest` add-on provides a mix-in trait that we can add to any `Assertions` implementation (like `AnyFunSuite`) much like the Specs2 package above.

```scala mdoc:silent
import org.scalatest._

class AnalysisTestScalaCheck extends funsuite.AnyFunSuite with matchers.must.Matchers with doobie.scalatest.IOChecker {

  override val colors = doobie.util.Colors.None // just for docs

  val transactor = Transactor.fromDriverManager[IO](
    driver = "org.postgresql.Driver", url = "jdbc:postgresql:world", user = "postgres", password = "password", logHandler = None
  )

  test("trivial")    { check(trivial)        }
  test("biggerThan") { checkOutput(biggerThan(0))  }
  test("update")     { check(update) }

}
```

Details are shown for failing tests.

```scala mdoc
// Run a test programmatically. Usually you would do this from sbt, bloop, etc.
(new AnalysisTestScalaCheck).execute(color = false)
```

### The MUnit Package

The `doobie-munit` add-on provides a mix-in trait that we can add to any `Assertions` implementation (like `FunSuite`) much like the ScalaTest package above.

```scala mdoc:silent
import _root_.munit._

class AnalysisTestSuite extends FunSuite with doobie.munit.IOChecker {

  override val colors = doobie.util.Colors.None // just for docs

  val transactor = Transactor.fromDriverManager[IO](
    driver = "org.postgresql.Driver", url = "jdbc:postgresql:world", user = "postgres", password = "password", logHandler = None
  )

  test("trivial")    { check(trivial)        }
  test("biggerThan") { checkOutput(biggerThan(0))  }
  test("update")     { check(update) }

}
```

### The Weaver Package

The `doobie-weaver` add-on provides a mix-in trait what we can add to any effectful test Suite. 
The `check` function takes an implicit `Transactor[F]` parameter. Since Weaver has its own way 
to manage shared resources, it is convenient to use that to allocate the transactor.

```scala mdoc:silent
import _root_.weaver._
import doobie.weaver._

object AnalysisTestSuite extends IOSuite with IOChecker {

  override type Res = Transactor[IO]
  override def sharedResource: Resource[IO,Res] = 
    Resource.pure(Transactor.fromDriverManager[IO](
      driver = "org.postgresql.Driver", url = "jdbc:postgresql:world", user = "postgres", password = "password", logHandler = None
    ))

  test("trivial")    { implicit transactor => check(trivial)        }
  test("biggerThan") { implicit transactor => checkOutput(biggerThan(0))  }
  test("update")     { implicit transactor => check(update) }

}
```
