---
layout: default
title: Slick vs JOOQ compile time type safety
permalink: /slick-vs-jooq-compile-time-safety/
---

# Slick vs JOOQ compile time type safety

<br>

[Slick](http://slick.lightbend.com/) for Scala and [JOOQ](http://www.jooq.org/) for Java are database access / query building libraries with DSL that can map 
almost 1:1 to standard SQL. They are not ORMs, so they provide maximum flexibility when working with SQL databases while at the same time maintaning 
performance of plain SQL (Slick relies on database internal query optimization, just to note).

I came accross a blog article [A Typesafety Comparison of SQL Access APIs](https://blog.jooq.org/2013/02/25/a-typesafety-comparison-for-sql-access-apis/) where
both Slick and JOOQ evaluated as having "much typesafety". So the purpose of this article is to evaluate this claim from my practical point of view. 

For me the main difference between using something like Slick or JOOQ vs plain SQL is compile time type safety. So if say you have a plain SQL query string and
you fatfinger something your application will compile, but then most likely throw a runtime exception:

```sql
SELECTT id, address FROM user
```

So the advantage of tools like Slick and JOOQ is that they should protect us from such mistakes (or so I thought). Lets test this assumption with a 
[simple app](https://github.com/vrcod/slick-jooq).

Both Slick and JOOQ have database table class generators, so I am using them in the app out of the box. This is a simple table with some data that both 
libraries will query:

```sql
CREATE TABLE IF NOT EXISTS user (
	id INTEGER AUTO_INCREMENT PRIMARY KEY,
	name VARCHAR NOT NULL,
	address VARCHAR NOT NULL
);

INSERT INTO user (name, address) VALUES ('John Smith', '66 Palm Street');
```

The goal is to do a simple select and store the result in a case class `UserData` (in case of Slick/Scala) or a pojo `User` (in case of JOOQ/Java). This is how 
JOOQ code looks like:

```java
String url = "jdbc:h2:mem:test1;DB_CLOSE_DELAY=-1;INIT=runscript from 'src/main/resources/create.sql'";
try {
	Connection conn = DriverManager.getConnection(url);
	DSLContext create = DSL.using(conn, SQLDialect.H2);
	Optional<User> user = create.select(USER.ID, USER.NAME, USER.ADDRESS).from(USER).fetchOptionalInto(User.class);
	user.map(u -> {
		System.out.println(u);
		return null;
	});
} catch (SQLException e) {
	e.printStackTrace();
}
```

Equivalent of the above with Slick:

```scala
val db = slick.jdbc.JdbcBackend.Database.forURL(
	"jdbc:h2:mem:test1;DB_CLOSE_DELAY=-1;INIT=runscript from 'src/main/resources/create.sql'",
	driver="org.h2.Driver"
)

val user: Future[Option[UserData]] = db.run(Tables.User.map(u => (u.id, u.name, u.address)).result.map(_.headOption.map(UserData.tupled)))
val result: Future[Unit] = user.map(u => u.map(println))
Await.ready(result, Duration.Inf)
```

The two examples compile and run.

JOOQ:

```
sbt compile
``` 
```
sbt "runMain Main"
```
result output: 

```
User(1,John Smith,66 Palm Street)
```

Slick:

```
sbt compile
``` 
```
sbt "runMain App"
```
result output: 

```
UserData(1,John Smith,66 Palm Street)
```

So now it's time to test the compile time type safety by trying to break things. I'm commenting out one of the selected fields in Slick query which means the 
result dataset will no longer map to case class `UserData`:

```scala
db.run(Tables.User.map(u => (u.id, /*u.name,*/ u.address)).result.map(_.headOption.map(UserData.tupled)))
```

```
sbt compile
``` 

produces an obvious compile time error:

```
[error] /home/vic/code/slick-jooq/src/main/scala/App.scala:15:135: type mismatch;
[error]  found   : ((Int, String, String)) => UserData
[error]  required: ((Int, String)) => UserData
[error] 	val user: Future[Option[UserData]] = db.run(Tables.User.map(u => (u.id, /*u.name,*/ u.address)).result.map(_.headOption.map(UserData.tupled)))
[error] 	                                                                                                                                     ^

```

This is very useful, because you can see a mistake in your query before you have a chance to deploy your code live and get a runtime exception. 

Now let's try the same with JOOQ:

```java
create.select(USER.ID, /*USER.NAME,*/ USER.ADDRESS).from(USER).fetchOptionalInto(User.class);
```

```
sbt compile
``` 

compiles just fine. Now that's a problem, this means than when we actually run it

```
sbt "runMain Main"
```

it produces a runtime exception

```
[error] (run-main-0) org.jooq.exception.MappingException: No matching constructor found on type class User for record org.jooq.impl.DefaultRecordMapper@298d99a3
```

In a real world scenario you might be satisfied that your code compiles and deploy it. Conclusion in this case is that while JOOQ is an excellent library for 
Java, it's not nearly as compile time type safe as Slick with Scala. This means that in JOOQ you need to have a unit test to cover this particular situation 
and you don't need it in Slick, because it's compile time type safety got you covered.

The full demo application can be found [here](https://github.com/vrcod/slick-jooq). At the time of this writing version of JOOQ is 3.9.5 and Slick 3.2.1.