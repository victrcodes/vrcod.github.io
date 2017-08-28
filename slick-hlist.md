---
layout: default
title: Slick and queries with more than 22 fields
permalink: /slick-and-queries-with-more-than-22-fields/
---

# Slick and queries with more than 22 fields

<br>

[Slick](http://slick.lightbend.com/) is a de facto database access library for Scala. It relies heavily on tuples which becomes a problem if you need to do
queries with more than 22 fields, because in Scala tuples have 22 element limit. So your standard query would look like this:

```scala
Company.map(c => (c.id, c.name, c.address)).result
```

Identical approach would not work if you have to select more than 22 fields. If like me you use Slick's code generator to generate database classes, you noticed
that when there are more than 22 columns in a table the generator falls back to a structure called `HCons`. HCons is Slick's version of `HList`
(Heterogeneous List) from [Shapeless](https://github.com/milessabin/shapeless) library which does not have element limit.

The standard approach is to have result of a select query converted into a case class, which looks like this:

```scala
case class CompanyData(id: Int, name: String, address: String)
Company.map(c => (c.id, c.name, c.address)).result.map(_.map(CompanyData.tupled))
```

In this instance you would get a result of `Seq[CompanyData]`. This would obviously not work with more than 22 fields. Shapeless `Hlist` can be easily converted
into a case class, however in our case we have Slick's `HCons` which doesn't have this functionality and is overall somewhat limited in comparison to `HList`.

Fortunatelly there is a solution - good folks from [Underscore](https://underscore.io) wrote a library that patches the gap between `HCons` and `HList` and
called it [Slickless](https://github.com/underscoreio/slickless). In their examples they demonstrate usage of Shapeless `HList` when defining projections in
database classes. That is not very convenient if you're using Slick's code generator and don't feel like fiddling around with customizing it. Instead, there's
another way - using Slickless directly in your query code.

Let's do a simple but realistic example which looks a little long, boring and crude, however proves a working solution. The final working application can be
found [here](https://github.com/vrcod/slick-hlist).

This will be our table:

```sql
CREATE TABLE IF NOT EXISTS user (
    id INTEGER AUTO_INCREMENT PRIMARY KEY,
    c2 VARCHAR,
    c3 VARCHAR,
    c4 VARCHAR,
    c5 VARCHAR,
    c6 VARCHAR,
    c7 VARCHAR,
    c8 VARCHAR,
    c9 VARCHAR,
    c10 VARCHAR,
    c11 VARCHAR,
    c12 VARCHAR,
    c13 VARCHAR,
    c14 VARCHAR,
    c15 VARCHAR,
    c16 VARCHAR,
    c17 VARCHAR,
    c18 VARCHAR,
    c19 VARCHAR,
    c20 VARCHAR,
    c21 VARCHAR,
    c22 VARCHAR,
    c23 VARCHAR,
    c24 VARCHAR,
    c25 VARCHAR
);
```

Case class to fetch data into:

```scala
case class User(
    id: Int,
    c2: Option[String],
    c3: Option[String],
    c4: Option[String],
    c5: Option[String],
    c6: Option[String],
    c7: Option[String],
    c8: Option[String],
    c9: Option[String],
    c10: Option[String],
    c11: Option[String],
    c12: Option[String],
    c13: Option[String],
    c14: Option[String],
    c15: Option[String],
    c16: Option[String],
    c17: Option[String],
    c18: Option[String],
    c19: Option[String],
    c20: Option[String],
    c21: Option[String],
    c22: Option[String],
    c23: Option[String],
    c24: Option[String],
    c25: Option[String]
)
```

In order to save space and cleaner code presentation I'm defining query field mapping projection as a separate function:

```scala
def columns = { user: User =>
    user.id ::
    user.c2 ::
    user.c3 ::
    user.c4 ::
    user.c5 ::
    user.c6 ::
    user.c7 ::
    user.c8 ::
    user.c9 ::
    user.c10 ::
    user.c11 ::
    user.c12 ::
    user.c13 ::
    user.c14 ::
    user.c15 ::
    user.c16 ::
    user.c17 ::
    user.c18 ::
    user.c19 ::
    user.c20 ::
    user.c21 ::
    user.c22 ::
    user.c23 ::
    user.c24 ::
    user.c25 ::
    HNil
}
```

Also values that will be used in insert/update queries:

```scala
val values =
    1 ::
    Some("two") ::
    Some("three") ::
    Some("four") ::
    Some("five") ::
    Some("six") ::
    Some("seven") ::
    Some("eight") ::
    Some("nine") ::
    Some("ten") ::
    Some("eleven") ::
    Some("twelve") ::
    Some("thirteen") ::
    Some("fourteen") ::
    Some("fifteen") ::
    Some("sixteen") ::
    Some("seventeen") ::
    Some("eighteen") ::
    Some("nineteen") ::
    Some("twenty") ::
    Some("twenty one") ::
    Some("twenty two") ::
    Some("twenty three") ::
    Some("twenty four") ::
    Some("twenty five") ::
    HNil
```

CRUD methods:

```scala
def getUser(id: Int): Future[Option[UserData]] = {
    database.run(User.filter(_.id === id).map(columns).result).map(_.headOption.map(row => Generic[UserData].from(row)))
}

def insertUser(user: UserData): Future[Int] = {
    database.run(User.map(columns).returning(User.map(_.id)) += Generic[UserData].to(user))
}

def updateUser(user: UserData): Future[Int] = {
    database.run(User.filter(_.id === user.id).map(columns).update(Generic[UserData].to(user)))
}
```

As you can see the main difference from a standard approach is that we're mapping `columns` HList instead of a tuple and in the end using Shapeless
`Generic[User].from(row)` and `Generic[User].to(row)` methods instead of `User.tupled`.

Finally application code that performs the main CRUD actions - insert, select and update:

```scala
//Connection to database
val database = slick.jdbc.JdbcBackend.Database.forURL(
    "jdbc:h2:mem:test1;DB_CLOSE_DELAY=-1;INIT=runscript from 'src/main/resources/create.sql'",
    driver="org.h2.Driver"
)

//Insert user and retrieve it's ID
val userID = insertUser(Generic[User].from(values))
//Select the inserted user
val selectedUser = userID.flatMap(id => getUser(id))

//Update selected user
val updatedUserID = selectedUser.flatMap {
    case Some(u) =>
        //Change value of the second field (c2)
        val newValues = u.id :: Some("two plus two") :: values.tail.tail
        updateUser(Generic[User].from(newValues))
    case None => Future(0)
}
val updatedUser = updatedUserID.flatMap(id => getUser(id))

//Print out the results
val result = (for {
    sUser <- selectedUser
    uUser <- updatedUser
} yield (sUser, uUser)).map {
    case (Some(u1), Some(u2)) =>
        println(u1)
        println(u2)
    case _ => println("not found")
}

Await.ready(result, Duration.Inf)
```

So the output should look like this:

```
User(1,Some(two),Some(three),Some(four),Some(five),Some(six),Some(seven),Some(eight),Some(nine),Some(ten),Some(eleven),Some(twelve),Some(thirteen),Some(fourteen),Some(fifteen),Some(sixteen),Some(seventeen),Some(eighteen),Some(nineteen),Some(twenty),Some(twenty one),Some(twenty two),Some(twenty three),Some(twenty four),Some(twenty five))
User(1,Some(two plus two),Some(three),Some(four),Some(five),Some(six),Some(seven),Some(eight),Some(nine),Some(ten),Some(eleven),Some(twelve),Some(thirteen),Some(fourteen),Some(fifteen),Some(sixteen),Some(seventeen),Some(eighteen),Some(nineteen),Some(twenty),Some(twenty one),Some(twenty two),Some(twenty three),Some(twenty four),Some(twenty five))
```

Thus prooving that our concept application works. As mentioned before you can find the full application [here](https://github.com/vrcod/slick-hlist). At the time of this writing
the version of Slick is 3.2.1.

NOTE: Some IDEs do not play nice with Shapeless (I'm looking at you, IntelliJ) and may show `Generic[User].from(values)` as an error, but it's a valid code that
compiles nonetheless.