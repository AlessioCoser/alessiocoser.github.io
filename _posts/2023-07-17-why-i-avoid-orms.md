---
layout: post
title:  "Why I avoid ORMs"
author: alessio
categories: [ orm ]
image: assets/images/avoid-orm.jpg
---
<script src="https://platform.linkedin.com/in.js" type="text/javascript">lang: en_US</script>

I will not bother you by expressing the advantages and disadvantages of the ORMs. There are plenty of articles out there. I just want to share my experience to clarify why I do not feel the need to use an ORM, why it could be harmful and, hopefully, to spark some constructive conversation and thoughts about it in the context of good software design.

Please <script type="IN/Share" data-url="{{site.url}}{{page.url}}"></script> **this article on LinkedIn and tag me** if you want to start a public conversation on this topic or [**contact me**](/contact) if you prefer to talk about it privately.

Over the past few years I have been dealing with several ORMs, mainly on JVM-based platforms, especially Hibernate, but not only. I think they could be useful when you need to do simple CRUD operations with no business logic. However, I have rarely seen a simple CRUD service in the real world.

<center>Converting tables and relations in objects and dependencies sounds very helpful at first, but I think it could be easily the wrong abstraction.</center>

## Separation of concerns
I usually separate the domain logic from the outside world adopting [hexagonal architecture](https://medium.com/ssense-tech/hexagonal-architecture-there-are-always-two-sides-to-every-story-bc0780ed7d9c) or [clean architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html).

I will go into the details of these approaches in another post. For now, the important thing to note is that **my domain's code should not depend on the implementation detail of the external world** such as databases, APIs, external libraries, whatever is not my code. **It should depend on abstractions that express business behaviors**.

To do so, of course, I have to map objects that belong to the external world into domain objects and vice-versa. It's a tiny cost that I'm usually more than happy to pay.

Having the domain logic independent helps me to easily test all the core domain logic using [mocks of the adapters](https://medium.com/@xpmatteo/how-i-learned-to-love-mocks-1-fb341b71328) that abstract away the details of the external world.

Moreover, it is possible to communicate with these adapters using the **domain's language and ignoring all the technical details**.

Last but not least, I can also test these adapters with really specific integration tests that run against a real local database.

### A bit of code
I will take the [birthday greeting kata](http://matteo.vaccari.name/blog/archives/154.html) as an example.

The purpose of this kata is to learn about the hexagonal architecture, and how to **shield your domain model from external APIs and systems**.

The given task is to write a program that loads a set of employee records from a flat file and then sends a greetings email to all employees whose birthday is today.

In particular I want to focus on the **database communication** part.

As suggested by the article linked above, for that part I would use a repository pattern (or Facade-Adapter combo) so I can abstract away the implementation details from my domain logic.

#### An ORM implementation
Implementing the Repository pattern seems easy with SpringBoot JPA. I only need to implement the `JpaRepository` interface:
```kotlin
@Entity
data class Employee (
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) val id: Long = 0,
    val first_name: String = "",
    val last_name: String = "",
    val email: String = "",
    @Temporal(TemporalType.DATE) val dateOfBirth: java.util.Date
)
```
```kotlin
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository

@Repository
interface EmployeeRepository: JpaRepository<Employee, Long> {
  @Query("<JPQ statement here>")
  findEmployeesBornOn(month: Int, day: Int): List<Employee>
}
```

#### What's wrong here?
I see two main problems:
1. **The Employee class should belong to the domain**. But it has the details of the database implementation and we shouldn't know where it comes from.
   - it has an id (also if I don't need it)
   - it is annotated as `@Entity`
   - it has other database-specific annotations on the fields
   - the field names cannot be changed otherwise the query will not work since the entity reflects the db columns (you can add another annotation though)
3. **The EmployeeRepository interface is not talking about the domain's language**:
   - it doesn't return a domain's object (Employee it's an ORM entity)
   - it extends a specific JPARepository interface

#### A plain implementation without ORM
I skipped the repository implementation here because it doesn't matter for the example:
```kotlin
data class Employee(
    val firstName: String,
    val lastName: String,
    val dateOfBirth: Date,
    val email: String
)
```
```kotlin
interface EmployeeRepository {
    fun findEmployeesBornOn(month: Int, day: Int): List<Employee>
}
```

The implementation of EmployeeRepository will have all the things needed to satisfy the `findEmployeesBornOn` method call. It could be a database connection or an API request. I just don't care from the domain's perspective.

#### Considerations

**In the best-case scenario, you use an ORM only inside the repository's implementation and map the ORM Entities into the domain objects.** This lets you preserve the domain-infrastructure separation.

In this case, you are doing a double mapping: one handled _auto-magically_ by the ORM that converts the retrieved SQL records into the Database Entity, and another, that converts the Database Entity into your domain class.

Anyway, this can be the way to go if you feel the need to keep using ORMs. I used this approach sometimes, but I felt as if I were adding an abstraction of another abstraction. So, probably, an unneeded complexity.

**In the worst-case scenario, you end up leaking all the database entities within the core domain, letting them become part of the domain logic and entangled with each other due to the database relations.**

Unfortunately, I experienced the worst-case scenario much more often than the best one.

This can lead to an _obscure code mass_ that will make it difficult to know the real side effects because the real dependencies between entities are obfuscated by the ORM's mechanics.

Eventually, you will have to test the code. This entanglement of domain and technical details makes it really difficult to test it in isolation. Your tests will start to feel weird and slow, and you will need an h2 or similar tools just to be able to verify the core business logic.

## What are ORMs abstracting?

The ORMs force you to map tables into objects, to use their custom language to interact with the database, and to define an object-oriented representation of the database tables. **Is this necessary? Is this useful? Is this really worth it?**

I think it's better to have **domain-aligned abstractions** and put the databse logic in a Repository along with an _"intelligent"_ mapping to translate database entities to a domain object. This is due to the way we structure our domain objects that should not reflect the way we want to store the data, but the way we want the system behave.

<center>I prefer to keep the mapping logic very specific, clear, and as straightforward as possible.</center>

The mapping logic could be very specific and different from case to case, so this logic should not be generalized by an ORM, but should be handled by us.

So, for a library that aims to simplify the database communication, I wouldn't do what ORMs do, I would rather focus on other parts:
- managing the database **connection**, **transactions**, and **configuration**
- building **SQL queries** with parameters and keeping them easy to read and maintain.

I would then leave it up to the user to compose the parts as needed.

## There aren't only ORMs

There are also query builders or database connection abstractions that are simpler and less _"magic"_, you don't have to make objects and relations to fit in tables and relations: you only build a query, execute it, and then you can map the result as you wish.

I played a bit on this concept by creating a **Kotlin library** that I called [JAKO (Just Another Kotlin ORM)](https://github.com/AlessioCoser/jako). It isn't really an ORM, since it doesn't deal with objects and relations, but it helps me to configure the database connection, to execute SQL instructions, and to map them into our domain objects with a fluent syntax.

It started when I was working as a consultant and since then has grown iteratively and incrementally driven only by my usage needs.

```kotlin
val db = Database.connect("jdbc:mysql://host:3306/db?user=usr&password=pwd", Dialect.All.MYSQL)

val tableIds: List<Int> = db
  .select("""SELECT `id` FROM `users` WHERE `city` = ?""", listOf("Milan"))
  .all { int("id") }
```

If you see the code above there is a question mark on the query to apply safely the value that is put in the second parameter. Easy peasy.

I wanted to go further, so on top of that, I played a bit with the **Kotlin DSL** to build a **SQL builder** in order to keep together the SQL instructions and their corresponding values.

The same query as above could have been written in this way:
```kotlin
val db = Database.connect("jdbc:mysql://host:3306/db?user=usr&password=pwd", Dialect.All.MYSQL)

val query = Query()
  .from("users")
  .fields("id")
  .where("city" EQ "Milano")

val tableIds: List<Int> = db
  .select(query)
  .all { int("id") }
```

You can also print the statement as SQL with params to see how exactly is traduced.

**Note** that the same methods (`toSQL()` and `params()`) are also used by the library to run the statement.
```kotlin
println(query.toSQL(Dialect.All.MYSQL))
// SELECT `id` FROM `users` WHERE `city` = ?
println(query.params())
// ["Milano"]
```

The good thing is that there is no _"magic"_ around objects involved: you are only building an SQL statement. The library only wraps the tables and the fields with the dialect-specific separator.

**So you will continue to use SQL.**

Another important thing to note is the way you map the result of the query:
`.all { int("id") }` or `.first { int("id") }`. In this way I get all results (or only the first one) I queried mapped with the id column as integer.

What about a more complex object like an `Employee`? Here you are a simple implementation of the `EmployeeRepository` of the example above:
```kotlin
class MysqlEmployeeRepository(private val db: Database) : EmployeeRepository {
    override fun findEmployeesBornOn(month: Int, day: Int): List<Employee> {
        val query = Query()
            .from("employees")
            .where(("DAY(date_of_birth)" EQ day) AND ("MONTH(date_of_birth)" EQ month))
        return db.select(query).all {
            Employee(
                firstName = str("first_name"),
                lastName = str("last_name"),
                dateOfBirth = date("date_of_birth"),
                email = str("email")
            )
        }
    }
}
```

## Conclusions
I think there are more effective ways to handle the database communication than the ORMs way. Thanks to the great Kotlin DSL you can do a lot of things keeping the sintax as similar as possible to SQL and still have a lot of help building SQL instructions.

However, I think, and hope, that there are similar libraries out there for other languages as well. If they do not exist, well, they should.

Please let me know what do you think :) <br /><script type="IN/Share" data-url="{{site.url}}{{page.url}}"></script> this article on LinkedIn and tag me if you find it interesting.