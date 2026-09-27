A reader needs two things from our backend: a list of published posts and the complete text of one post. The database and field schema are ready. This chapter makes them available over HTTP.

We will return the actual `BlogPost` entities through the JSON mapper. Named entity graphs choose the fields for each response; small `Data` and `Table` envelopes describe how those values reach the client.

Start with the [chapter 7 source](https://github.com/anjunar/anjunar-blog-example/tree/ee01b68ac30a1a2f93f1d8153b7f08114637bef2) and its migrated PostgreSQL database. All paths below are relative to the repository root. We keep the published dependencies and versions from that chapter.

## Define the public contract

| Request | Response |
| --- | --- |
| `GET /service/blog/posts` | A page of published posts without their long content. |
| `GET /service/blog/posts/{slug}` | The complete published post. |
| Detail request for a draft or unknown slug | HTTP 404. |

The list accepts `offset` and `limit`, defaulting to 0 and 20. Limit must be between 1 and 100; offset must be nonnegative. Invalid parameters return HTTP 400.

These endpoints are public reads. They do not provide a write method. Signing in, permissions, and safe editing will enter the series after the first frontend.

Three separate decisions shape each response:

| Decision | Mechanism |
| --- | --- |
| Which posts may this visitor read? | The query's `PUBLISHED` predicate. |
| Which fields does this endpoint include? | Its named entity graph. |
| Which mapped fields are visible under the current rules? | The `EntitySchema` and mapper rules from chapter 7. |

A graph does not hide a draft row. A readable schema does not grant permission to return every stored post. The query must select the public rows first.

## Make BlogPost a mapper entity

The mapper uses `EntityProvider` to recognize entities whose fields should be selected by a graph. Add this import in `application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala`:

```scala
import com.anjunar.json.mapper.provider.EntityProvider
```

Change the class declaration to `class BlogPost extends EntityProvider`. The interface requires a UUID ID and a Scala `Long` version.

Our ID already matches. Replace the version field with:

```scala
@Version
@Column(nullable = false)
@SchemaId("dcb0681e")
@JsonbProperty
var version: Long = -1L
```

Update its entry in `BlogPost.Schema` as well:

```scala
val version: SingularProperty[BlogPost, Long] = reference(_.version)
```

The previous chapter used nullable `lang.Long`. The new interface uses primitive `Long`, with -1 as the value before persistence. Hibernate assigns version 0 on insertion and increments it on changes. The persistence tests verify both behaviors and still reject stale edits.

This changes the Scala contract, not the database column. The existing `bigint` column and its stable schema ID remain. Running `SchemaMain migrate` against a chapter 6 or 7 database reports `AlreadyApplied` with zero SQL statements.

The annotations remain ordinary annotations on class-body fields; they need no `@field` target.

## Select list and detail fields

Add `NamedAttributeNode`, `NamedEntityGraph`, and `NamedEntityGraphs` to the JPA imports in `BlogPost.scala`. Add these annotations above the entity class, keeping its existing annotations:

```scala
@NamedEntityGraphs(Array(
  new NamedEntityGraph(name = "BlogPost.list", attributeNodes = Array(
    new NamedAttributeNode("id"),
    new NamedAttributeNode("version"),
    new NamedAttributeNode("slug"),
    new NamedAttributeNode("title"),
    new NamedAttributeNode("summary"),
    new NamedAttributeNode("status"),
    new NamedAttributeNode("publishedAt")
  )),
  new NamedEntityGraph(name = "BlogPost.detail", attributeNodes = Array(
    new NamedAttributeNode("id"),
    new NamedAttributeNode("version"),
    new NamedAttributeNode("slug"),
    new NamedAttributeNode("title"),
    new NamedAttributeNode("content"),
    new NamedAttributeNode("summary"),
    new NamedAttributeNode("status"),
    new NamedAttributeNode("publishedAt")
  ))
))
```

Both graphs include `id` and `version`. The list leaves out `content`; the detail contains all eight fields. The graph names will connect queries, REST methods, and response metadata.

We use the graphs in two places. Hibernate receives them as fetch hints when querying. The mapper receives them as the JSON field selection when writing the response.

Those jobs are related, but they are not identical. Leaving `content` out of the JSON graph guarantees that it is absent from the list response. A JPA fetch graph does not guarantee that Hibernate leaves every unselected basic column out of its SQL. We will examine dedicated list projections and query performance in the search chapter.

## Query public rows in a predictable order

In `BlogPost.scala`, add `import java.util`; retain the existing `java.lang` import for the count query's Java wrapper. Replace `findPublishedBySlug` and add these two methods inside the companion object, after its nested `Schema` class:

```scala
def findPublishedBySlug(slug: String)(using entityManager: EntityManager): Option[BlogPost] = {
    val builder = entityManager.getCriteriaBuilder
    val query = builder.createQuery(classOf[BlogPost])
    val post = query.from(classOf[BlogPost])
    query.select(post).where(
      builder.equal(post.get(schema.slug), builder.parameter(classOf[String], "slug")),
      builder.equal(post.get(schema.status), BlogPostStatus.PUBLISHED)
    )
    Option(entityManager.createQuery(query)
      .setHint("jakarta.persistence.fetchgraph", entityManager.getEntityGraph("BlogPost.detail"))
      .setParameter("slug", slug).getSingleResultOrNull)
  }

  def listPublished(offset: Int, limit: Int)(using entityManager: EntityManager): util.List[BlogPost] = {
    val builder = entityManager.getCriteriaBuilder
    val query = builder.createQuery(classOf[BlogPost])
    val post = query.from(classOf[BlogPost])
    query.select(post)
      .where(Seq(builder.equal(post.get(schema.status), BlogPostStatus.PUBLISHED))*)
      .orderBy(builder.desc(post.get(schema.publishedAt)), builder.asc(post.get(schema.id)))
    entityManager.createQuery(query)
      .setHint("jakarta.persistence.fetchgraph", entityManager.getEntityGraph("BlogPost.list"))
      .setFirstResult(offset)
      .setMaxResults(limit)
      .getResultList
  }

  def countPublished()(using entityManager: EntityManager): Long = {
    val builder = entityManager.getCriteriaBuilder
    val query = builder.createQuery(classOf[lang.Long])
    val post = query.from(classOf[BlogPost])
    query.select(builder.count(post))
      .where(Seq(builder.equal(post.get(schema.status), BlogPostStatus.PUBLISHED))*)
    entityManager.createQuery(query).getSingleResult.longValue()
  }
```

The list orders by publication time descending. UUID ascending breaks ties, so equal timestamps do not produce an arbitrary order. The same `PUBLISHED` predicate applies to the page, its total count, and the detail lookup.

The `Seq(...)*` form explicitly selects Java's varargs `where` overload. Without it, a single predicate is ambiguous between the Criteria overloads in this Scala version.

`size` will mean the total published count before pagination. It is not the number of rows in the current page. Page and count are separate queries, so concurrent publishing can change the total between them. We are building basic offset pagination here, not a stable snapshot across multiple requests.

There is no scheduling rule yet: `PUBLISHED` is the publication decision. The timestamp records publication and determines ordering.

## Give responses a consistent shape

Create `application/backend/src/main/scala/com/anjunar/blog/Data.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.provider.DTO
import jakarta.json.bind.annotation.JsonbProperty

import scala.annotation.meta.field

class Data[E](
    @(JsonbProperty @field) val data: E,
    @(JsonbProperty @field) val schema: Schema
) extends DTO
```

A detail response wraps one entity in `data` and its field description in `schema`. It does not copy each BlogPost field into a second backend model.

Create `application/backend/src/main/scala/com/anjunar/blog/Table.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.provider.DTO
import jakarta.json.bind.annotation.JsonbProperty

import java.util
import scala.annotation.meta.field

class Table[C](
    @(JsonbProperty @field) val rows: util.List[C],
    @(JsonbProperty @field) val size: Long
) extends DTO
```

A list is a `Table[Data[BlogPost]]`: `rows` holds those same data/schema wrappers, and `size` holds the total count.

`DTO` marks these envelopes for our REST writer. These fields are declared as constructor parameters, so their JSON annotations explicitly target the backing field with `@field`. That is different from the entity's class-body fields.

## Describe the selected fields

Create `application/backend/src/main/scala/com/anjunar/blog/Schema.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.provider.DTO
import com.anjunar.json.mapper.schema.EntitySchema
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.{EntityGraph as JpaEntityGraph}

import java.util
import scala.annotation.meta.field
import scala.jdk.CollectionConverters.*

class Schema(@(JsonbProperty @field) val entries: util.List[SchemaProperty]) extends DTO

class SchemaProperty(
    @(JsonbProperty @field) val name: String,
    @(JsonbProperty @field)("type") val typeName: String
) extends DTO

object Schema {
  // Describes the selected fields, not the caller's permissions.
  def forGraph(source: EntitySchema[?], graph: JpaEntityGraph[?]): Schema = {
    val selected = graph.getAttributeNodes.asScala.map(_.getAttributeName).toSet
    val entries = source.properties.valuesIterator
      .filter(property => selected.contains(property.name))
      .map(property => new SchemaProperty(property.name, property.typeName))
      .toList.asJava
    new Schema(entries)
  }
}
```

This `Schema` is a response model. It is separate from the cached `BlogPost.Schema` used by the mapper and Criteria.

For each response, `forGraph` takes the selected attribute names and obtains their names and types from the existing EntitySchema. We do not maintain another hand-written field catalog. All current fields are scalar; nested relationship metadata can grow with the relationship chapter.

The result describes structure. It does not say whether a caller may edit a field, and the frontend must not treat an entry as a write permission. Permission rules remain on the server. Contextual links will be added in the HATEOAS chapter.

A nullable summary still has a schema entry when selected, even if the particular post has no summary value. Field metadata and field values answer different questions.

## Connect the resource methods

We need a small annotation to tell the response writer which named graph to use. Create `application/backend/src/main/java/com/anjunar/blog/EntityGraph.java`:

```java
package com.anjunar.blog;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface EntityGraph {
    String value();
}
```

This is our method annotation. The persistence API's graph type is imported as `JpaEntityGraph` where both names would collide.

Now create `application/backend/src/main/scala/com/anjunar/blog/BlogPostsResource.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.ws.rs.{BadRequestException, DefaultValue, GET, NotFoundException, Path, PathParam, Produces, QueryParam}
import jakarta.ws.rs.core.MediaType

import scala.compiletime.uninitialized
import scala.jdk.CollectionConverters.*

@Path("/blog/posts")
@Produces(Array(MediaType.APPLICATION_JSON))
@RequestScoped
class BlogPostsResource {
  @Inject
  var entityManager: EntityManager = uninitialized

  @GET
  @EntityGraph("BlogPost.list")
  def list(@QueryParam("offset") @DefaultValue("0") rawOffset: String,
      @QueryParam("limit") @DefaultValue("20") rawLimit: String): Table[Data[BlogPost]] = {
    val offset = rawOffset.toIntOption.getOrElse(throw new BadRequestException("offset must be an integer"))
    val limit = rawLimit.toIntOption.getOrElse(throw new BadRequestException("limit must be an integer"))
    if (offset < 0 || limit < 1 || limit > 100)
      throw new BadRequestException("offset must be nonnegative and limit must be between 1 and 100")

    given EntityManager = entityManager
    val schema = Schema.forGraph(BlogPost.schema, entityManager.getEntityGraph("BlogPost.list"))
    val rows = BlogPost.listPublished(offset, limit).asScala
      .map(post => new Data(post, schema)).toList.asJava
    new Table(rows, BlogPost.countPublished())
  }

  @GET
  @Path("/{slug}")
  @EntityGraph("BlogPost.detail")
  def read(@PathParam("slug") slug: String): Data[BlogPost] = {
    given EntityManager = entityManager
    val post = BlogPost.findPublishedBySlug(slug).getOrElse(throw new NotFoundException())
    val schema = Schema.forGraph(BlogPost.schema, entityManager.getEntityGraph("BlogPost.detail"))
    new Data(post, schema)
  }
}
```

RESTEasy reads the paging values as strings. We parse them explicitly so malformed values, integer overflow, and invalid bounds all produce HTTP 400.

The detail lookup returns 404 whenever the published-slug query returns no row. That covers both missing posts and drafts without revealing a draft's content.

The existing CDI extension discovers `@Path` resources. There is no registration list to update. The injected EntityManager still belongs to the request transaction from chapter 4.

Keep the method's generic return types intact. The writer needs `Table[Data[BlogPost]]` to follow both envelopes to their entity type.

## Let the mapper write JSON

The mapper needs a resolver for field rules. Replace `RuntimeContext.scala` with:

```scala
package com.anjunar.blog

import jakarta.enterprise.inject.spi.CDI
import jakarta.persistence.EntityManager

object RuntimeContext {
  def bean[T](clazz: Class[T]): T = {
    val instance = CDI.current().select(clazz)
    if (instance.isUnsatisfied) clazz.getDeclaredConstructor().newInstance()
    else instance.get()
  }

  def entityManager(): EntityManager =
    CDI.current().select(classOf[EntityManager]).get()
}
```

A rule managed by CDI is resolved through CDI. Our stateless default rule has no bean registration, so the fallback uses its no-argument constructor. A rule that needs injection must be a CDI bean; constructing it through the fallback would not inject its dependencies.

Now create `application/backend/src/main/scala/com/anjunar/blog/MapperMessageBodyWriter.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.JsonMapper
import com.anjunar.json.mapper.provider.DTO
import com.anjunar.scala.universe.TypeResolver
import jakarta.annotation.Priority
import jakarta.ws.rs.Produces
import jakarta.ws.rs.container.ResourceInfo
import jakarta.ws.rs.core.{Context, MediaType, MultivaluedMap}
import jakarta.ws.rs.ext.{MessageBodyWriter, Provider}

import java.io.OutputStream
import java.lang.annotation.Annotation
import java.lang.reflect.Type
import java.nio.charset.StandardCharsets
import scala.compiletime.uninitialized

@Provider
@Priority(3900)
@Produces(Array(MediaType.APPLICATION_JSON))
class MapperMessageBodyWriter extends MessageBodyWriter[Any] {
  @Context
  var resource: ResourceInfo = uninitialized

  override def isWriteable(clazz: Class[?], genericType: Type,
      annotations: Array[Annotation], mediaType: MediaType): Boolean =
    classOf[DTO].isAssignableFrom(clazz)

  override def writeTo(body: Any, clazz: Class[?], genericType: Type,
      annotations: Array[Annotation], mediaType: MediaType,
      headers: MultivaluedMap[String, Object], stream: OutputStream): Unit = {
    val annotation = resource.getResourceMethod.getAnnotation(classOf[EntityGraph])
    val graph =
      if (annotation == null) null
      else RuntimeContext.entityManager().getEntityGraph(annotation.value())
    val targetType = if (genericType == null) clazz else genericType
    val json = JsonMapper.serialize(body, TypeResolver.resolve(targetType), graph,
      [T] => (ruleClass: Class[T]) => RuntimeContext.bean(ruleClass))
    stream.write(json.getBytes(StandardCharsets.UTF_8))
  }
}
```

`@Provider` lets the existing extension discover the writer. `isWriteable` selects our DTO envelopes; normal text responses continue through RESTEasy's text writer.

The resource method's `@EntityGraph` annotation supplies the graph name. The mapper receives the actual graph, the response's declared generic type, and the rule resolver. It follows `Table` and `Data` to the contained BlogPost and applies the graph there. The envelopes themselves are not BlogPost graph attributes.

Using only `body.getClass` would lose the nested type arguments. Keeping `genericType` is therefore part of the JSON contract, not just a reflection detail.

The writer encodes the result as UTF-8. Quotes, newlines, and non-ASCII text are handled by the mapper; we do not assemble JSON strings by hand.

## Keep the persistence context through HEAD serialization

Our chapter 4 boundary already keeps the EntityManager open while writing a normal response. The new writer exposes one adjustment we need for `HEAD`.

RESTEasy handles an implicit HEAD request by invoking the GET resource and its writer, then suppressing the response body. The mapper still needs the EntityManager to resolve its graph. Closing it in the response filter would turn an otherwise valid HEAD request into HTTP 500.

In `TransactionBoundary.scala`, replace:

```scala
if (!response.hasEntity || request.getMethod == "HEAD") transaction.finish(successful)
```

with:

```scala
// RESTEasy also serializes implicit HEAD responses; its writer still needs the EntityManager.
if (!response.hasEntity) transaction.finish(successful)
```

Responses with an entity now complete after the writer, including HEAD. Responses without an entity complete in the response filter. GET and HEAD remain read-only transactions that roll back. The existing buffer still prevents a serialization failure from sending a partial successful JSON response.

## Read the actual JSON contract

The mapper adds `@type` markers. Null optional fields and empty collections are omitted in version 1.1.5.

For an empty database, the complete list response is:

```json
{"size":0,"@type":"Table"}
```

There is no `rows: []` member. An offset beyond the final row also omits `rows`, while `size` retains the total published count. Our frontend model will initialize its collection accordingly.

For a published example, the detail response's `data` object is:

```json
{
  "id": "b62db12a-61a7-46a5-b8d2-c487f80e825a",
  "version": 0,
  "slug": "our-first-public-post",
  "title": "Our first public post",
  "content": "This is the complete article. The list response leaves this text out.",
  "status": "PUBLISHED",
  "publishedAt": "2026-09-27T10:15:42.123456Z",
  "summary": "A working REST response from PostgreSQL.",
  "@type": "BlogPost"
}
```

Alongside it, `schema.entries` contains the eight field descriptions. For example, the version entry is:

```json
{"name":"version","type":"Long","@type":"SchemaProperty"}
```

A list row uses the same envelope, with `content` absent from both its `data` object and its selected field descriptions. Version zero remains present.

## Run it with example posts

Set the database environment variables from the earlier chapters for a separate local database. Then run:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server "application-backend/testFull"
```

An existing chapter 6/7 database reports `AlreadyApplied` with zero SQL statements. An empty database is initialized by the migration. For an older chapter 5 database, follow chapter 6's adoption sequence first.

Expect **41 successful tests**. The eight new HTTP tests cover list and detail field selection, counts and paging, equal-time ordering, draft exclusion, invalid parameters, unsupported writes, empty pages, and HEAD. They use the real server, CDI, mapper, and PostgreSQL, then remove only their own rows. Error cases intentionally produce server logs.

For manual requests, the companion includes `database/examples/public-posts.sql`. It inserts the public example above and one private draft. Apply it after migration. With Compose:

```text
docker compose cp database/examples/public-posts.sql postgres:/tmp/public-posts.sql
docker compose exec -T postgres psql -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --file /tmp/public-posts.sql
```

With a native PostgreSQL installation, adjust the connection details:

```text
psql -h 127.0.0.1 -p 5433 -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --file database/examples/public-posts.sql
```

psql prompts for the database password; it does not read `BLOG_DB_PASSWORD`. Repeating the script leaves its two fixed example IDs unchanged. It is optional example data, not a schema migration or automatic startup seeding.

Start the application:

```text
sbt --server "application-backend/run"
```

In another terminal:

```text
curl -i "http://127.0.0.1:8080/service/blog/posts?offset=0&limit=20"
curl -i http://127.0.0.1:8080/service/blog/posts/our-first-public-post
curl -i http://127.0.0.1:8080/service/blog/posts/our-private-draft
curl -i "http://127.0.0.1:8080/service/blog/posts?limit=0"
curl -I http://127.0.0.1:8080/service/blog/posts/our-first-public-post
```

Use `curl.exe` in Windows PowerShell if needed. Expect 200, 200, 404, 400, and 200. On an otherwise empty database, the first response has one published row and size 1. The detail includes content; HEAD has no body. Stop the server with Ctrl+C.

The [completed chapter 8 source](https://github.com/anjunar/anjunar-blog-example/tree/54fac6042b34be4c8d7ab9b3f4ed054272220b81) is the exact revision used by this article.

Our first milestone is complete: the application starts, stores posts in PostgreSQL, and serves published posts through REST. Chapter 9 starts the Scala.js interface that will display them.
