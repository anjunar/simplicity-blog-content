We can store a post and evolve its database table. The next question is how the rest of the application should describe that post.

The JSON mapper needs to know which fields are available and which rules apply to them. A Criteria query needs typed attributes such as `slug` and `status`. We will describe those fields once in `BlogPost.Schema` and use them for both jobs.

By the end of this chapter, a real PostgreSQL query will find a published post by slug. Tests will also verify the JSON field contract and confirm that incoming JSON cannot change fields whose schema rules deny writing.

Start with the [chapter 6 source](https://github.com/anjunar/anjunar-blog-example/tree/185a0fd7634f1da3e7f7b420a806a022033cd242) and a database already migrated to that version. All paths below are relative to the project root.

## Keep the different schemas separate

We already use `@SchemaId`, but it answers a different question:

| Mechanism | Responsibility |
| --- | --- |
| JPA annotations | Map the entity to tables, columns, and relationships. |
| Bean Validation | Check whether values and publication state are valid. |
| `@SchemaId` | Keep a schema element's identity stable across database migrations. |
| `EntitySchema` | Describe fields, mapper rules, and typed JPA attributes. |

Adding `EntitySchema` does not create a database column. Adding a JPA column does not automatically add its mapper schema entry. We maintain these descriptions together.

## Add the mapper dependency

Add this entry to the existing `libraryDependencies` sequence in `build.sbt`:

```scala
"com.anjunar" %% "json-mapper" % "1.1.5",
```

Also add this build setting next to `ThisBuild / scalaVersion`:

```scala
ThisBuild / externalResolvers := Seq(Resolver.mavenCentral)
```

sbt normally includes the local Ivy repository when resolving application dependencies. Explicitly selecting Central prevents a locally published framework build from shadowing the version used by readers. The [sbt dependency documentation](https://www.scala-sbt.org/2.x/docs/en/reference/sbt-update.html) explains these resolver settings.

Refresh an existing checkout's resolution once:

```text
sbt --server "application-backend/update"
```

No neighboring repository is a build dependency.

## Obtain the current EntityManager

A plain schema property can describe a field without a database connection. A typed JPA property also needs Hibernate's initialized metamodel.

Create `application/backend/src/main/scala/com/anjunar/blog/RuntimeContext.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.inject.spi.CDI
import jakarta.persistence.EntityManager

object RuntimeContext {
  def entityManager(): EntityManager =
    CDI.current().select(classOf[EntityManager]).get()
}
```

This resolves the request-scoped EntityManager producer from chapter 4. Its underlying EntityManager belongs to the active `RequestTransaction`.

The first access to `BlogPost.schema` must therefore happen after CDI and Hibernate are ready, inside an active request transaction. Do not initialize it in an eager top-level value during bootstrap.

## Describe every field

In `BlogPost.scala`, add these imports:

```scala
import com.anjunar.json.mapper.schema.{EntitySchema, SchemaProvider}
import com.anjunar.json.mapper.schema.property.SingularProperty
```

The file already imports `java.lang`, `Instant`, and `UUID`. Add this companion object after the entity class:

```scala
object BlogPost extends SchemaProvider[BlogPost.Schema] {
  class Schema extends EntitySchema[BlogPost](RuntimeContext.entityManager()) {
    val id: SingularProperty[BlogPost, UUID] = reference(_.id)
    val version: SingularProperty[BlogPost, lang.Long] = reference(_.version)
    val slug: SingularProperty[BlogPost, String] = reference(_.slug)
    val title: SingularProperty[BlogPost, String] = reference(_.title)
    val content: SingularProperty[BlogPost, String] = reference(_.content)
    val status: SingularProperty[BlogPost, BlogPostStatus] = reference(_.status)
    val publishedAt: SingularProperty[BlogPost, Instant] = reference(_.publishedAt)
    val summary: SingularProperty[BlogPost, String] = reference(_.summary)
  }

}
```

The companion implements `SchemaProvider`. The mapper finds that companion and asks for its schema. In version 1.1.5, the provider discovers the nested class named `Schema` and constructs it lazily. Our schema has a public no-argument constructor, as that convention expects.

There is one entry for every persistent field, including `id`, `version`, and the optional `summary`. Omitting a field from this schema can remove it from mapper output even when the field still carries `@JsonbProperty`.

The selector `_.slug` is checked by the Scala compiler. If we rename the field, the old selector no longer compiles. The property also retains its name, value type, and access rule.

## Why these fields use reference

The name `reference` includes scalar persistent attributes. It is not limited to a relationship with another entity.

For `slug`, the factory obtains Hibernate's actual singular attribute and returns a `SingularProperty[BlogPost, String]`. That value also implements JPA's `SingularAttribute` interface, so Criteria can consume it directly.

| Factory | Use |
| --- | --- |
| `property` | Mapper metadata for a value, including calculated or `@Transient` output. It is not a JPA Criteria attribute. |
| `reference` | A persistent singular attribute: a scalar field or a single-valued association. |
| `set` | A persistent collection mapped as a JPA set. |
| `list` | A persistent collection mapped as a JPA list. |

Keep the concrete return type. Declaring `slug` as a plain `Property[BlogPost, String]` would hide the JPA attribute interface from the compiler.

The factory must match the real mapping. A calculated field does not become a database column because we call `reference` or use its name in a query. We will use the collection factories when the model actually gains relationships.

## Use the schema in a Criteria query

Add `EntityManager` to the existing JPA imports in `BlogPost.scala`. Then add this method inside the companion, after `class Schema`:

```scala
def findPublishedBySlug(slug: String)(using entityManager: EntityManager): Option[BlogPost] = {
    val builder = entityManager.getCriteriaBuilder
    val query = builder.createQuery(classOf[BlogPost])
    val post = query.from(classOf[BlogPost])
    query.select(post).where(
      builder.equal(post.get(schema.slug), builder.parameter(classOf[String], "slug")),
      builder.equal(post.get(schema.status), BlogPostStatus.PUBLISHED)
    )
    Option(entityManager.createQuery(query).setParameter("slug", slug).getSingleResultOrNull)
  }
```

The query has two predicates: the requested slug, and `PUBLISHED` status. The slug is passed as a query parameter. The table's unique slug constraint means the result is either one post or no post, represented by `Option`.

The key expressions are:

```scala
post.get(schema.slug)
post.get(schema.status)
```

Both use attributes from the same schema the mapper will inspect. No separate hand-written JPA metamodel or duplicated field-name strings are needed.

The status predicate remains essential. Field visibility rules do not select which rows a query may return. A schema whose fields are readable must not be mistaken for permission to publish every stored draft.

## Declare the JSON fields

The mapper discovers annotated members and then consults the schema. Both parts must be present.

Add this import:

```scala
import jakarta.json.bind.annotation.JsonbProperty
```

Add `@JsonbProperty` to each of the eight persistent fields: `id`, `version`, `slug`, `title`, `content`, `status`, `publishedAt`, and `summary`.

For example, title now reads:

```scala
@NotBlank
@Size(min = 3, max = 180)
@Column(nullable = false, length = 180)
@SchemaId("46fdb02a")
@JsonbProperty
var title: String = ""
```

These annotations are on class-body fields, so they need no explicit `@field` target. Keep the existing persistence and validation annotations. The publication-consistency method is an internal validation check and receives no `@JsonbProperty`.

## Preserve the publication timestamp

Our entity uses `Instant`. The mapper's built-in temporal serializer in version 1.1.5 does not handle that type, so we supply its string representation explicitly.

Create `application/backend/src/main/scala/com/anjunar/blog/InstantConverter.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.converter.JacksonJsonConverter
import com.anjunar.scala.universe.ResolvedClass

import java.time.Instant

class InstantConverter extends JacksonJsonConverter {
  override def toJson(input: Any, resolvedClass: ResolvedClass): String =
    input.asInstanceOf[Instant].toString

  override def toJava(json: String, resolvedClass: ResolvedClass): Any =
    Instant.parse(json)
}
```

The mapper's converter annotation expects a `JacksonJsonConverter` subclass. We override its two conversion methods to use an ISO-8601 timestamp. The mapper wraps the returned text as a JSON string; the converter must not add JSON quotes itself.

Add this import to `BlogPost.scala`:

```scala
import com.anjunar.json.mapper.annotations.UseConverter
```

The complete publication-time field becomes:

```scala
@Column(name = "published_at")
@SchemaId("398bfd50")
@JsonbProperty
@UseConverter(classOf[InstantConverter])
var publishedAt: Instant = null
```

A value such as `2026-09-27T10:15:42.123456Z` retains its seconds, fractional precision, and UTC designation. The database continues to use the timestamp mapping from chapter 5.

## Readable does not mean writable

We did not pass a custom rule to any schema factory. That selects `DefaultRule`:

| Rule method | Result |
| --- | --- |
| `isVisible` | `true` |
| `isWriteable` | `false` |

The mapper can serialize these fields. During deserialization it skips incoming changes to them. This is separate from whether Scala declares a field as a mutable `var`.

For example, the test attempts to supply:

```json
{
  "title": "Unauthorized title",
  "summary": "Unauthorized summary",
  "version": 999,
  "status": "PUBLISHED",
  "publishedAt": "2026-09-27T10:00:00Z"
}
```

The stored post remains unchanged. That result means the mapper ignored protected fields; it does not mean an HTTP endpoint returned an authorization error. Endpoint authorization and deliberate mutation handling will be added in their own chapters.

Later, a custom `VisibilityRule` can decide using the current entity and caller. Keep those decisions in the rule. `SchemaProvider.schema` is cached, so storing a user's current permissions in the schema itself would retain the wrong state for later requests.

## Check the contract against the real runtime

The companion extends `BlogPostPersistenceSpec` with five checks:

- The schema covers every persistent attribute; ID and version retain their JPA metadata.
- Typed Criteria finds a published slug and returns no result for drafts or unknown slugs.
- A populated post serializes all eight fields, including its version and precise publication time.
- Null optional fields are omitted, while version zero is retained.
- Incoming JSON cannot change the fields protected by the default rules.

The tests activate CDI's request context and resolve `RequestTransaction` through the container. This exercises the same EntityManager producer used by the application. A later transaction uses the schema after the request that first initialized it has ended.

The serializer test calls the real mapper directly:

```scala
val constructRule = [T] => (clazz: Class[T]) =>
  clazz.getDeclaredConstructor().newInstance()

val output = JsonMapper.serialize(
  post,
  TypeResolver.resolve(classOf[BlogPost]),
  null,
  constructRule
)
```

Here `JsonMapper` is imported from `com.anjunar.json.mapper` and `TypeResolver` from `com.anjunar.scala.universe`. The `null` argument means that this test supplies no entity graph. Its resolver constructs our stateless default rules; later rules with injected services will use CDI resolution.

Null optional values are absent in this mapper version. They are not emitted as explicit JSON null. We will preserve that distinction when implementing the frontend model.

The new dependency also supplies a Logback implementation. The companion includes a small `logback.xml` with an INFO console logger so normal startup does not emit framework DEBUG tracing. It does not change mapper behavior.

## Run this chapter

Use the existing separate development database and the chapter 6 environment variables:

```text
sbt --server "application-backend/update"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server "application-backend/testFull"
```

On a database already at chapter 6, the migration reports `AlreadyApplied` with zero SQL statements. The schema and JSON annotations add no persistent fields. Expect **33 successful tests**.

For a new database, `SchemaMain migrate` creates the current table. For an older chapter 5 database, follow chapter 6's adoption sequence first.

The [completed chapter 7 source](https://github.com/anjunar/anjunar-blog-example/tree/ee01b68ac30a1a2f93f1d8153b7f08114637bef2) is the exact version used by this article.

We now have one field model used by the mapper and by a real Criteria query. Chapter 8 will connect that contract to public REST endpoints, response envelopes, and entity graphs.
