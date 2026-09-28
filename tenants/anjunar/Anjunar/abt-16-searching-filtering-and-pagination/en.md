# Searching, Filtering and Pagination

We can now create and edit posts. As the list grows, finding a particular
post becomes the next problem. Readers should be able to search published
posts. Administrators also need to find drafts, change the sort order and
move through the matching results.

A search is more than a text box attached to a query. The count must use
the same filters as the rows. Equal titles need a predictable order.
Changing a filter should return to the first page, while following a page
link must preserve the filter. Reloading or sharing that URL should restore
the same controls.

This chapter builds that complete path using the stack's reusable
`HibernateSearch` architecture: an annotated search model, CDI providers,
typed Criteria queries and a bound search form. We will also make the
database select only the columns that a list actually needs.

## Run the checkpoint

Start from chapter 15:

```text
git switch --detach 9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b
```

The completed version of this chapter is:

```text
git switch --detach 214eeb6d5a2c6062013d95abf7f990a1cb032cce
sbt --server frontendAssets
sbt --server "application-backend/run"
```

Use the existing development database and administrator. No database mapping
or library version changes in this chapter. Keep `BLOG_COOKIE_SECURE=false`
for local HTTP. The [companion guide](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/docs/searching-filtering-and-pagination.md)
contains the walkthrough and test configuration.

The public list at `/en` gets text search, sorting and page size. The
editorial list at `/en/editorial` gets the same controls plus publication
status. We submit a search explicitly with **Search** or Enter.

## 1. Define the search contract

Both endpoints accept the following query parameters:

| Parameter | Accepted values |
| --- | --- |
| `q` | A trimmed substring of up to 100 characters; empty means no text filter. |
| `status` | Editorial: omitted, `DRAFT` or `PUBLISHED`. Public: omitted or `PUBLISHED`. |
| `sort` | `newest`, `oldest`, `title` or `title-desc`. |
| `offset` | A nonnegative integer, default 0. |
| `limit` | An integer from 1 to 100, default 20. |

The public default order remains newest first. Editorial keeps its existing
title order. Unknown sort names, invalid bounds and control characters
receive HTTP 400.

For example:

```text
/service/blog/posts?q=scala&sort=title&offset=0&limit=10
/service/editorial/posts?q=release&status=DRAFT&sort=title&offset=0&limit=10
```

Create [PostSearchParams.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/PostSearchParams.scala) in
`application/backend/src/main/scala/com/anjunar/blog/`:

```scala
package com.anjunar.blog

import jakarta.ws.rs.{BadRequestException, DefaultValue, QueryParam}

final class PostSearchParams {
  @QueryParam("q") var query: String = ""
  @QueryParam("status") var status: String = ""
  @QueryParam("sort") var sort: String = ""
  @QueryParam("offset") @DefaultValue("0") var offset: String = "0"
  @QueryParam("limit") @DefaultValue("20") var limit: String = "20"

  def search(editorial: Boolean): BlogPostSearch = {
    val text = Option(query).getOrElse("").trim
    if (text.length > 100 || text.exists(Character.isISOControl))
      throw new BadRequestException("q must contain at most 100 characters and no control characters")
    val selectedStatus = Option(status).getOrElse("") match {
      case "" => None
      case "DRAFT" if editorial => Some(BlogPostStatus.DRAFT)
      case "PUBLISHED" => Some(BlogPostStatus.PUBLISHED)
      case _ => throw new BadRequestException("Invalid status filter")
    }
    val selectedSort = Option(sort).filter(_.nonEmpty).getOrElse(if (editorial) "title" else "newest")
    if (!Set("newest", "oldest", "title", "title-desc").contains(selectedSort))
      throw new BadRequestException("Invalid sort order")
    val start = Option(offset).flatMap(_.toIntOption).filter(_ >= 0)
      .getOrElse(throw new BadRequestException("offset must be a nonnegative integer"))
    val size = Option(limit).flatMap(_.toIntOption).filter(value => value >= 1 && value <= 100)
      .getOrElse(throw new BadRequestException("limit must be between 1 and 100"))
    BlogPostSearch(text, if (editorial) selectedStatus else Some(BlogPostStatus.PUBLISHED), selectedSort, start, size)
  }
}
```

`PostSearchParams` is the JAX-RS input bean. It collects strings so our own
boundary can produce consistent validation errors. The immutable
`BlogPostSearch` extends `AbstractSearch` and carries the validated values to
`HibernateSearch`. We will define that model and its providers in section 3.

The public endpoint calls `search(editorial = false)`. That always selects
`PUBLISHED`, even when the caller happens to be an administrator. Sending
`status=DRAFT` to the public endpoint is rejected. Permission to use the
editorial endpoint is checked separately by its existing ADMIN policy.

These query parameters describe a read operation. Their parsing belongs
at the request boundary; the JSON mapper remains responsible for validating
entity changes from the preceding chapters.

## 2. Select a list projection

The list has no reason to load a post's entire body. Our existing
`BlogPost.list` graph selects the fields sent by the mapper, but that
alone does not guarantee that the SQL query excludes the content column.

Create [BlogPostSummary.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/BlogPostSummary.scala) in the same backend package:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.annotations.UseConverter
import com.anjunar.json.mapper.provider.DTO
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.criteria.Expression
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.time.Instant
import java.util
import java.util.UUID
import scala.annotation.meta.field

// A read-only list projection. Load BlogPost detail before editing.
final class BlogPostSummary(
    @(JsonbProperty @field) val id: UUID,
    @(JsonbProperty @field) val version: Long,
    @(JsonbProperty @field) val slug: String,
    @(JsonbProperty @field) val title: String,
    @(JsonbProperty @field) val summary: String,
    @(JsonbProperty @field) val status: BlogPostStatus,
    @(JsonbProperty @field) @(UseConverter @field)(classOf[InstantConverter]) val publishedAt: Instant
) extends DTO

object BlogPostSummary {
  def select(
      query: JpaCriteriaQuery[BlogPostSummary], post: JpaRoot[BlogPost],
      selection: util.List[Expression[?]], builder: HibernateCriteriaBuilder
  ): JpaCriteriaQuery[BlogPostSummary] = {
    val schema = BlogPost.schema
    query.select(builder.construct(classOf[BlogPostSummary],
      post.get(schema.id), post.get(schema.version), post.get(schema.slug),
      post.get(schema.title), post.get(schema.summary), post.get(schema.status),
      post.get(schema.publishedAt)))
  }
}
```

This is an intentional read-only list shape: seven fields from `BlogPost`,
with the same names and value types. It has no content field and no
persistence lifecycle. Hibernate will construct it directly from the selected
columns.

The annotations target constructor fields because that is where the mapper
reads this DTO's metadata. The timestamp retains the existing
`InstantConverter`.

This projection does not replace the entity in the detail or update endpoints.
Previewing or editing still loads the real `BlogPost`, and writes still go
through `PreparedChange`. Nor does the projection create a second editable
domain model: it is a deliberately limited list response.

Projection fields do not automatically run the entity's property rules.
Here they are the explicit metadata allowed by the endpoint's row visibility.
A future restricted field would need an authorization decision in the
projection as well.

## 3. Reuse the stack's HibernateSearch architecture

A blog search is a useful first client of reusable search infrastructure.
The stack already separates the work into an `AbstractSearch` model,
`@RestPredicate` and `@RestSort` metadata, CDI providers, and
`HibernateSearch` for executing rows and count queries. We bring that
structure into the companion project.

Here, `HibernateSearch` names our Criteria helper. It is not the separate
Hibernate Search product for full-text indexing. We add no library or local
dependency on the stack, and no tenant context.

The resulting path is:

```text
PostSearchParams → BlogPostSearch → HibernateSearch.searchContext
                                      ↓
                                SearchBeanReader
                                      ↓
                         CDI predicate and sort providers
                                      ↓
                         entities(...) / count(...)
                                      ↓
                           BlogPostSummary.select
```

The last selection callback applies to the row query. Count uses the same
predicates but selects a count instead.

### Define the reusable contracts

Create the following Scala files under
`application/backend/src/main/scala/com/anjunar/blog/hibernate/search/`.

[AbstractSearch.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/AbstractSearch.scala) keeps paging independent of JAX-RS.
The stack calls the row offset `index`; our public HTTP parameter remains
`offset`. Parsing and HTTP 400 responses stay in `PostSearchParams`.

```scala
package com.anjunar.blog.hibernate.search

// HTTP parsing belongs to the resource's input bean; index is a row offset.
abstract class AbstractSearch {
  def index: Int
  def limit: Int
}
```

[Context.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/Context.scala) is created for one provider invocation.
It carries that query's root, predicates, parameter bindings and any
additional expressions a provider needs to contribute:

```scala
package com.anjunar.blog.hibernate.search

import jakarta.persistence.criteria.{Expression, Predicate}
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.util

final case class Context[V, E](
    value: V,
    builder: HibernateCriteriaBuilder,
    predicates: util.List[Predicate],
    root: JpaRoot[E],
    query: JpaCriteriaQuery[?],
    selection: util.List[Expression[?]],
    name: String,
    parameters: util.Map[String, Any]
)
```

A predicate provider receives its annotated field's value. A sort provider
receives the entire search object, so it can consider more than one input
when selecting a fixed order.

[PredicateProvider.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/PredicateProvider.scala):

```scala
package com.anjunar.blog.hibernate.search

trait PredicateProvider[V, E] {
  def build(context: Context[V, E]): Unit
}
```

[SortProvider.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/SortProvider.scala):

```scala
package com.anjunar.blog.hibernate.search

import jakarta.persistence.criteria.Order

import java.util

trait SortProvider[V, E] {
  def sort(context: Context[V, E]): util.List[Order]
}
```

The reader returns the predicates and parameters for one fresh query in
[HibernateSearchContextResult.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/HibernateSearchContextResult.scala):

```scala
package com.anjunar.blog.hibernate.search

import jakarta.persistence.criteria.{Expression, Predicate}

import java.util

final case class HibernateSearchContextResult(
    selection: util.List[Expression[?]],
    predicates: util.List[Predicate],
    parameters: util.Map[String, Any]
)
```

[HibernateSearchContext.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/HibernateSearchContext.scala) connects the engine to those
providers without depending on a particular entity or search model:

```scala
package com.anjunar.blog.hibernate.search

import jakarta.persistence.criteria.{Expression, Order, Predicate}
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.util

trait HibernateSearchContext {
  def apply[E](
      builder: HibernateCriteriaBuilder, query: JpaCriteriaQuery[?], root: JpaRoot[E]
  ): HibernateSearchContextResult

  def sort[E](
      builder: HibernateCriteriaBuilder, query: JpaCriteriaQuery[?], root: JpaRoot[E],
      predicates: util.List[Predicate], selection: util.List[Expression[?]]
  ): util.List[Order]
}
```

The HTTP boundary already checks page bounds. The generic engine also
protects its internal callers through [QuerySurface.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/QuerySurface.scala):

```scala
package com.anjunar.blog.hibernate.search

object QuerySurface {
  def firstResult(index: Int): Int = {
    require(index >= 0, "Search index must be nonnegative")
    index
  }

  def maxResults(limit: Int): Int = {
    require(limit >= 1 && limit <= 100, "Search limit must be between 1 and 100")
    limit
  }
}
```

### Declare the annotation-to-provider mapping

Create the Java annotations under
`application/backend/src/main/java/com/anjunar/blog/hibernate/search/annotations/`.
They refer to the Scala provider interfaces; the existing mixed Scala/Java
build compiles them together.

[RestPredicate.java](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/java/com/anjunar/blog/hibernate/search/annotations/RestPredicate.java) associates a field with its provider.
An optional name gives a parameter a stable name different from its field:

```java
package com.anjunar.blog.hibernate.search.annotations;

import com.anjunar.blog.hibernate.search.PredicateProvider;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD})
public @interface RestPredicate {
    Class<? extends PredicateProvider<?, ?>> value();
    String name() default "";
}
```

[RestSort.java](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/java/com/anjunar/blog/hibernate/search/annotations/RestSort.java) requires an explicit provider for the search's
sort policy:

```java
package com.anjunar.blog.hibernate.search.annotations;

import com.anjunar.blog.hibernate.search.SortProvider;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD})
public @interface RestSort {
    Class<? extends SortProvider<?, ?>> value();
}
```

The stack also has generic field-path sorting. This chapter exposes four
named orders, so its search names its own provider rather than accepting
arbitrary property paths.

### Read metadata and resolve CDI providers

[SearchBeanReader.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/SearchBeanReader.scala) uses the `AnnotationIntrospector`
already supplied by our published dependencies. `@JsonbProperty` makes a
search field visible to this introspector; `@RestPredicate` or
`@RestSort` supplies its search behavior.

The reader resolves the declared provider class from CDI's `Instance`.
There is no manual provider registry to maintain when adding another
search. Providers use `@ApplicationScoped`; their invocation state lives
in `Context`, never in bean fields.

```scala
package com.anjunar.blog.hibernate.search

import com.anjunar.blog.hibernate.search.annotations.{RestPredicate, RestSort}
import com.anjunar.scala.universe.introspector.AnnotationIntrospector
import jakarta.enterprise.inject.Instance
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.criteria.{Expression, Order, Predicate}
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.util
import scala.jdk.CollectionConverters.*

object SearchBeanReader {
  def read[E](
      search: AbstractSearch, builder: HibernateCriteriaBuilder,
      root: JpaRoot[E], query: JpaCriteriaQuery[?],
      instances: Instance[PredicateProvider[Any, E]]
  ): HibernateSearchContextResult = {
    val model = AnnotationIntrospector.createWithType(search.getClass, classOf[JsonbProperty])
    val predicates = new util.ArrayList[Predicate]()
    val selection = new util.ArrayList[Expression[?]]()
    val parameters = new util.HashMap[String, Any]()
    model.properties.foreach { property =>
      val annotation = property.findAnnotation(classOf[RestPredicate])
      if (annotation != null) {
        // A missing provider or unreadable field is a configuration error.
        // Never silently drop a predicate that could restrict public visibility.
        val provider = findProvider(instances, annotation.value())
        val value = property.get(search)
        if (value != null) {
          val name = if (annotation.name().isBlank) property.name else annotation.name()
          provider.build(Context(value, builder, predicates, root, query, selection, name, parameters))
        }
      }
    }
    HibernateSearchContextResult(selection, predicates, parameters)
  }

  def order[E](
      search: AbstractSearch, builder: HibernateCriteriaBuilder,
      root: JpaRoot[E], query: JpaCriteriaQuery[?],
      predicates: util.List[Predicate], selection: util.List[Expression[?]],
      instances: Instance[SortProvider[Any, E]]
  ): util.List[Order] = {
    val model = AnnotationIntrospector.createWithType(search.getClass, classOf[JsonbProperty])
    val properties = model.properties.filter(_.findAnnotation(classOf[RestSort]) != null).toList
    require(properties.size <= 1, "A search must declare at most one sort provider")
    properties.headOption match {
      case Some(property) =>
        val provider = findProvider(instances, property.findAnnotation(classOf[RestSort]).value())
        if (property.get(search) == null) new util.ArrayList[Order]()
        else {
          // As in the stack, sorting receives the entire search, not just the sort field.
          provider.sort(Context(search, builder, predicates, root, query, selection,
            property.name, new util.HashMap[String, Any]()))
        }
      case None => new util.ArrayList[Order]()
    }
  }

  private def findProvider[T](instances: Instance[T], providerClass: Class[?]): T = {
    val matches = instances.iterator().asScala.filter(providerClass.isInstance).toList
    if (matches.size != 1)
      throw new IllegalStateException(
        s"Expected one CDI search provider for ${providerClass.getName}, found ${matches.size}")
    matches.head
  }
}
```

A missing provider is an error, even for an optional input. An unreadable
annotated field also propagates its failure. Silently continuing could
omit the publication-status predicate and expose rows that the endpoint
was supposed to exclude. `None` is different: it is a valid value that
the editorial status provider deliberately interprets as all statuses.

Only one sort declaration is allowed. Its provider receives the search
object, not the string stored in the annotated sort field. Keep that
distinction when implementing `SortProvider[V, E]`.

### Execute rows and count in one reusable engine

Create [HibernateSearch.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/HibernateSearch.scala):

```scala
package com.anjunar.blog.hibernate.search

import jakarta.enterprise.context.ApplicationScoped
import jakarta.enterprise.inject.Instance
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.persistence.criteria.{Expression, Order, Predicate}
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.lang
import java.util
import scala.compiletime.uninitialized

@ApplicationScoped
class HibernateSearch {
  @Inject var entityManager: EntityManager = uninitialized
  @Inject var predicateProviders: Instance[PredicateProvider[?, ?]] = uninitialized
  @Inject var sortProviders: Instance[SortProvider[?, ?]] = uninitialized

  def searchContext[S <: AbstractSearch](search: S): HibernateSearchContext =
    new HibernateSearchContext {
      override def apply[E](
          builder: HibernateCriteriaBuilder, query: JpaCriteriaQuery[?], root: JpaRoot[E]
      ): HibernateSearchContextResult =
        SearchBeanReader.read(search, builder, root, query,
          predicateProviders.asInstanceOf[Instance[PredicateProvider[Any, E]]])

      override def sort[E](
          builder: HibernateCriteriaBuilder, query: JpaCriteriaQuery[?], root: JpaRoot[E],
          predicates: util.List[Predicate], selection: util.List[Expression[?]]
      ): util.List[Order] =
        SearchBeanReader.order(search, builder, root, query, predicates, selection,
          sortProviders.asInstanceOf[Instance[SortProvider[Any, E]]])
    }

  def entities[E, P](
      index: Int, limit: Int, entityClass: Class[E], projection: Class[P],
      context: HibernateSearchContext,
      select: (JpaCriteriaQuery[P], JpaRoot[E], util.List[Expression[?]], HibernateCriteriaBuilder) => JpaCriteriaQuery[P]
  ): util.List[P] = {
    val start = QuerySurface.firstResult(index)
    val size = QuerySurface.maxResults(limit)
    val builder = entityManager.getCriteriaBuilder.asInstanceOf[HibernateCriteriaBuilder]
    val query = builder.createQuery(projection)
    val root = query.from(entityClass)
    val result = context(builder, query, root)
    val order = context.sort(builder, query, root, result.predicates, result.selection)
    select(query, root, result.selection, builder).where(result.predicates).orderBy(order)
    val typedQuery = entityManager.createQuery(query).setFirstResult(start).setMaxResults(size)
    result.parameters.forEach((name, value) => typedQuery.setParameter(name, value))
    typedQuery.getResultList
  }

  def count[E](entityClass: Class[E], context: HibernateSearchContext): Long = {
    val builder = entityManager.getCriteriaBuilder.asInstanceOf[HibernateCriteriaBuilder]
    val query = builder.createQuery(classOf[lang.Long])
    val root = query.from(entityClass)
    // Providers rebuild Criteria nodes against this count query's own root.
    val result = context(builder, query, root)
    query.select(builder.count()).where(result.predicates)
    val typedQuery = entityManager.createQuery(query)
    result.parameters.forEach((name, value) => typedQuery.setParameter(name, value))
    typedQuery.getSingleResult.longValue()
  }
}
```

The engine has no dependency on `BlogPost`. It accepts the entity class,
result class and a projection callback. The existing request-scoped
EntityManager producer supplies the active persistence context through CDI.
Creating the application-scoped engine does not create a cached entity
schema or capture a request.

`searchContext` retains the immutable search. The actual Criteria
objects are built when `entities` or `count` runs. Consequently the
count cannot accidentally reuse a predicate attached to the row query's
root.

This port keeps the stack's provider/context/projection structure.
It leaves out query-cache hints because this chapter has not configured
a query cache. It also keeps HTTP parsing in the input bean and uses an
explicit sort provider, as described above.

### Define the post search and its providers

Create [BlogPostSearch.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/BlogPostSearch.scala) beside the input bean and
projection:

```scala
package com.anjunar.blog

import com.anjunar.blog.hibernate.search.{AbstractSearch, Context, PredicateProvider, SortProvider}
import com.anjunar.blog.hibernate.search.annotations.{RestPredicate, RestSort}
import jakarta.enterprise.context.ApplicationScoped
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.criteria.Order
import jakarta.ws.rs.core.UriBuilder

import java.lang
import java.util
import java.util.Locale
import scala.annotation.meta.field
import scala.jdk.CollectionConverters.*

final case class BlogPostSearch(
    @(JsonbProperty @field) @(RestPredicate @field)(classOf[BlogPostSearch.QueryPredicate])
    query: String,
    @(JsonbProperty @field) @(RestPredicate @field)(classOf[BlogPostSearch.StatusPredicate])
    status: Option[BlogPostStatus],
    @(JsonbProperty @field) @(RestSort @field)(classOf[BlogPostSearch.PostSort])
    sort: String,
    offset: Int,
    override val limit: Int
) extends AbstractSearch {
  override def index: Int = offset

  def pageUrl(path: String, start: Int): String = {
    val uri = UriBuilder.fromPath(path).queryParam("offset", start).queryParam("limit", limit)
    // Insert raw text as a template value: literal %20 and braces must be encoded as data.
    if (query.nonEmpty) uri.queryParam("q", "{search}")
    status.foreach(value => uri.queryParam("status", value.name()))
    uri.queryParam("sort", sort)
    (if (query.nonEmpty) uri.build(query) else uri.build()).toASCIIString
  }
}

object BlogPostSearch {
  @ApplicationScoped
  class QueryPredicate extends PredicateProvider[String, BlogPost] {
    override def build(context: Context[String, BlogPost]): Unit = {
      if (context.value.nonEmpty) {
        val builder = context.builder
        val post = context.root
        val pattern = builder.parameter(classOf[String], context.name)
        context.predicates.add(builder.or(
          builder.like(builder.lower(post.get(BlogPost.schema.title)), pattern, '!'),
          builder.like(builder.lower(post.get(BlogPost.schema.slug)), pattern, '!'),
          builder.like(builder.lower(post.get(BlogPost.schema.summary)), pattern, '!')
        ))
        val literal = context.value.toLowerCase(Locale.ROOT)
          .replace("!", "!!").replace("%", "!%").replace("_", "!_")
        context.parameters.put(context.name, s"%$literal%")
      }
    }
  }

  @ApplicationScoped
  class StatusPredicate extends PredicateProvider[Option[BlogPostStatus], BlogPost] {
    override def build(context: Context[Option[BlogPostStatus], BlogPost]): Unit =
      context.value.foreach { status =>
        val parameter = context.builder.parameter(classOf[BlogPostStatus], context.name)
        context.predicates.add(context.builder.equal(context.root.get(BlogPost.schema.status), parameter))
        context.parameters.put(context.name, status)
      }
  }

  @ApplicationScoped
  class PostSort extends SortProvider[BlogPostSearch, BlogPost] {
    override def sort(context: Context[BlogPostSearch, BlogPost]): util.List[Order] = {
      val builder = context.builder
      val post = context.root
      val title = builder.lower(post.get(BlogPost.schema.title))
      val publication = post.get(BlogPost.schema.publishedAt)
      val primary = context.value.sort match {
        case "title" => Seq(builder.asc(title))
        case "title-desc" => Seq(builder.desc(title))
        case direction @ ("oldest" | "newest") =>
          // Keep drafts after dated posts in both directions.
          val undated = builder.selectCase[lang.Integer]()
            .when(builder.isNull(publication), 1).otherwise(0)
          Seq(builder.asc(undated),
            if (direction == "oldest") builder.asc(publication) else builder.desc(publication))
        case _ => throw new IllegalArgumentException("Unsupported post sort")
      }
      (primary :+ builder.asc(post.get(BlogPost.schema.id))).asJava
    }
  }
}
```

The model holds validated values and declares which provider handles each
search field. These are constructor parameters, so `@field` directs the
annotations to their backing fields. Ordinary class-body JPA fields do
not need that target.

`QueryPredicate`, `StatusPredicate` and `PostSort` contain the
blog-specific decisions. They neither open EntityManagers nor execute
queries. All Criteria paths come from `BlogPost.schema`, whose
`reference` properties implement the JPA attribute interfaces. There is
no second metamodel or arbitrary sort path from the URL.

`BlogPostSummary.select`, introduced above, owns the seven selected
columns. Additional provider expressions are available through its
`selection` argument; this simple list projection does not need them.

### Match literal text across three fields

The text predicate is an OR across title, slug and summary. If a status is
present, the query combines that status predicate with the OR. Searching
does not turn an OR branch into an alternative to the public visibility
restriction.

`QueryPredicate` converts the input to lower case with `Locale.ROOT`, escapes it,
and adds the surrounding substring wildcards. The actual value is sent
through the named `query` parameter.

The escape character is `!`. Therefore:

| Input character | Bound LIKE pattern fragment |
| --- | --- |
| `%` | `!%` |
| `_` | `!_` |
| `!` | `!!` |

Escaping `!` first matters. Otherwise, we could escape the characters that
we just introduced ourselves. A search for `100%` should find that text,
rather than treat the percent sign as “any following characters.”

The content field is not part of this search. That keeps its behavior aligned
with the list: title, slug and summary are searchable; the article body is
loaded when someone opens a detail.

### Rebuild predicates for each Criteria root

`HibernateSearch.entities(...)` and `count(...)` create separate queries,
each with its own root. Both call `context.apply`, which runs the same
annotated predicate providers and binds their values. The context retains
the immutable input; every invocation builds new Criteria nodes.

The count has no offset or limit. If 31 posts match and a page contains 10,
`size` remains 31. An offset beyond the last match returns no rows while
retaining that filtered total.

### Give ties a stable order

Each sort ends with the post ID in ascending order. If two posts have the
same title or publication timestamp, the database still has a complete order
for them. Consecutive pages do not depend on an unspecified tie order.

Date sorting also needs a decision about drafts, which have no publication
timestamp. The CASE expression groups undated rows after dated rows before
applying newest or oldest order. Drafts therefore stay at the end in both
directions.

This is stable ordering for a given set of data. It does not freeze the list
across requests: inserting, editing or retracting posts can still move rows
between offset pages.

### Preserve raw text when building page links

Notice how `BlogPostSearch.pageUrl` passes the search text as a URI template value.
There are two levels of escaping in this feature, with different purposes.

At the URL level, a user may literally search for `%20` or `{query}`.
Passing raw text directly into a URI template builder can interpret it as
existing URL encoding or another template placeholder. The fixed
`{search}` placeholder and `build(query)` ensure the value is encoded
as data. A page link must not turn the literal characters `%20` into
a space.

At the SQL level, we will separately escape the characters that have special
meaning in LIKE patterns. URL encoding does not solve that problem.

## 4. Return the filtered list through REST

Replace the public [BlogPostsResource.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/BlogPostsResource.scala) with this complete version:

```scala
package com.anjunar.blog

import com.anjunar.blog.hibernate.search.HibernateSearch

import jakarta.annotation.security.PermitAll

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.ws.rs.{BeanParam, GET, NotFoundException, Path, PathParam, Produces}
import jakarta.ws.rs.core.MediaType

import scala.compiletime.uninitialized
import scala.jdk.CollectionConverters.*

@PermitAll
@Path("/blog/posts")
@Produces(Array(MediaType.APPLICATION_JSON))
@RequestScoped
class BlogPostsResource {
  @Inject var links: PostLinks = uninitialized
  @Inject var queries: HibernateSearch = uninitialized
  @Inject
  var entityManager: EntityManager = uninitialized

  @GET
  @EntityGraph("BlogPost.list")
  def list(@BeanParam parameters: PostSearchParams): Table[Data[BlogPostSummary]] = {
    given EntityManager = entityManager
    val search = parameters.search(editorial = false)
    val context = queries.searchContext(search)
    val schema = Schema.forGraph(BlogPost.schema, entityManager.getEntityGraph("BlogPost.list"))
    val rows = queries.entities(search.index, search.limit, classOf[BlogPost],
      classOf[BlogPostSummary], context, BlogPostSummary.select).asScala
      .map(post => new Data(post, schema, links.summary(post, editorial = false))).toList.asJava
    val total = queries.count(classOf[BlogPost], context)
    new Table(rows, total, links.page(search, total, editorial = false))
  }

  @GET
  @Path("/{slug}")
  @EntityGraph("BlogPost.detail")
  def read(@PathParam("slug") slug: String): Data[BlogPost] = {
    given EntityManager = entityManager
    val post = BlogPost.findPublishedBySlug(slug).getOrElse(throw new NotFoundException())
    val schema = Schema.forGraph(BlogPost.schema, entityManager.getEntityGraph("BlogPost.detail"))
    new Data(post, schema, links.publicPost(post))
  }
}
```

The list endpoint now accepts the parameter bean and returns
`Table[Data[BlogPostSummary]]`. The detail method stays entity-based.

The graph still describes the response fields and supplies matching schema
metadata. The Criteria constructor projection controls which columns are
selected from the database. Those are related decisions with different jobs.

In the existing ADMIN-protected [EditorialPostsResource.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/EditorialPostsResource.scala), replace
the list method with the following. Add `BeanParam` to its
`jakarta.ws.rs` imports and import
`com.anjunar.blog.hibernate.search.HibernateSearch`. Add
`@Inject var queries: HibernateSearch = uninitialized` beside its existing
injected fields. Its detail and command methods stay in place.

```scala
  @GET
  @EntityGraph("BlogPost.list")
  def list(@BeanParam parameters: PostSearchParams): Table[Data[BlogPostSummary]] = {
    given EntityManager = manager
    val search = parameters.search(editorial = true)
    val context = queries.searchContext(search)
    val schema = Schema.forGraph(BlogPost.schema, manager.getEntityGraph("BlogPost.list"))
    val rows = queries.entities(search.index, search.limit, classOf[BlogPost],
      classOf[BlogPostSummary], context, BlogPostSummary.select).asScala
      .map(post => new Data(post, schema, links.summary(post, editorial = true))).asJava
    val total = queries.count(classOf[BlogPost], context)
    new Table(rows, total, links.page(search, total, editorial = true))
  }
```

The essential difference is `editorial = true`. The caller has already
passed the resource's `@RolesAllowed(Array("ADMIN"))` policy, and the
validated search may include drafts.

The old separate `BlogPost.listPublished` and `countPublished` methods
are removed. Both list endpoints now go through the same search
`HibernateSearch` context, so adding a predicate updates both paths.

### Keep filters in navigation links

In [PostLinks.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/PostLinks.scala), the collection endpoint lookup now uses
`classOf[PostSearchParams]` for the list method's parameter type. This
keeps the existing policy-based session link discovery in sync with the
new signature.

The following methods build summary links and page links inside that class:

```scala
  def summary(post: BlogPostSummary, editorial: Boolean): util.List[Link] = {
    val path = if (editorial) s"/service/editorial/posts/${post.id}"
      else s"/service/blog/posts/${post.slug}"
    val values = new util.ArrayList[Link]()
    values.add(new Link("self", path, "GET", "BlogPost"))
    if (editorial && endpoint("update"))
      values.add(new Link("update", path, "PATCH", "BlogPost"))
    if (!editorial && endpoint("read"))
      values.add(new Link("preview", s"/service/editorial/posts/${post.id}", "GET", "BlogPost"))
    // Publication capabilities need the loaded body and are advertised by detail responses.
    values
  }

  def page(search: BlogPostSearch, total: Long, editorial: Boolean): util.List[Link] = {
    val path = if (editorial) "/service/editorial/posts" else "/service/blog/posts"
    def link(rel: String, start: Int) =
      new Link(rel, search.pageUrl(path, start), "GET", "BlogPost")
    val values = Seq(Some(link("self", search.offset)),
      Option.when(editorial && endpoint("create"))(new Link("create", path, "POST", "BlogPost")),
      Option.when(search.offset > 0)(link("first", 0)),
      Option.when(search.offset > 0)(link("previous", math.max(0, search.offset - search.limit))),
      Option.when(search.offset.toLong + search.limit < total && search.offset <= Int.MaxValue - search.limit)(
        link("next", search.offset + search.limit)))
    values.flatten.asJava
  }
```

The existing imports already provide `Link`, `java.util` and the Scala
collection converters. All page links use the same validated search and
change only the offset. The first-page link is particularly useful when
deletions or a bookmarked deep offset leave an empty page.

An editorial list row can advertise reading and editing without loading its
body. Publishing is different: `canPublish` needs to inspect content.
Those capabilities remain on the detail response, which is loaded when
someone opens the preview.

## 5. Make the browser URL the loaded search state

The backend understands the request. Next, the browser needs a small model
for the parameters that produced its current list.

Create the frontend [PostSearch.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostSearch.scala) in
`application/frontend/src/main/scala/com/anjunar/blog/frontend/`:

```scala
package com.anjunar.blog.frontend

import scala.scalajs.js.URIUtils.encodeURIComponent

final case class PostSearch(query: String = "", status: String = "", sort: String = "newest",
    offset: Int = 0, limit: Int = 20, editorial: Boolean = false) {
  def path: String = if (editorial) "/editorial" else "/"
  def filtered: Boolean = query.nonEmpty || status.nonEmpty

  def queryString(includeDefaults: Boolean = false): String = {
    val defaultSort = if (editorial) "title" else "newest"
    val values = Seq(
      Option.when(includeDefaults || offset > 0)("offset" -> offset.toString),
      Option.when(includeDefaults || limit != 20)("limit" -> limit.toString),
      Option.when(query.nonEmpty)("q" -> query),
      Option.when(status.nonEmpty)("status" -> status),
      Option.when(sort != defaultSort)("sort" -> sort)
    ).flatten
    values.map((name, value) => s"$name=${encodeURIComponent(value)}").mkString("&")
  }

  def url: String = {
    val query = queryString()
    if (query.isEmpty) path else s"$path?$query"
  }
}

object PostSearch {
  def parse(read: String => Option[String], editorial: Boolean): PostSearch = {
    val query = read("q").getOrElse("").trim
    val status = read("status").getOrElse("")
    val sort = read("sort").filter(_.nonEmpty).getOrElse(if (editorial) "title" else "newest")
    val offset = read("offset").getOrElse("0").toIntOption.filter(_ >= 0)
    val limit = read("limit").getOrElse("20").toIntOption.filter(value => value >= 1 && value <= 100)
    val invalidText = query.length > 100 || query.exists(c => c < ' ' || (c >= 127 && c <= 159))
    val statuses = if (editorial) Set("", "DRAFT", "PUBLISHED") else Set("", "PUBLISHED")
    if (invalidText || !statuses.contains(status) ||
        !Set("newest", "oldest", "title", "title-desc").contains(sort) || offset.isEmpty || limit.isEmpty)
      throw new HttpFailure(400)
    PostSearch(query, status, sort, offset.get, limit.get, editorial)
  }
}
```

This class represents route state. It is separate from the entity model.
It validates the same public bounds and sort names before a list request is
sent, while the backend keeps its own authoritative checks.

Browser URLs omit default values where possible. API requests explicitly
include offset and limit. `encodeURIComponent` handles each query value;
we do not encode the whole URL as one string.

Changing the page is a copy of the current search with a different offset.
The query, status, sort and page size remain attached to it.

The public [BlogService.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogService.scala) now takes this value:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom

import scala.concurrent.{ExecutionContext, Future}
import scala.scalajs.js.URIUtils.encodeURIComponent

final class BlogService(using ExecutionContext) {
  def list(search: PostSearch, signal: Option[dom.AbortSignal]): Future[BlogPostTable] =
    HttpJson.get[BlogPostTable](s"/service/blog/posts?${search.queryString(includeDefaults = true)}", signal)
      .map { table =>
        require(table.size >= 0 && table.rows != null, "Invalid post table")
        table.rows.foreach(row => require(row != null && row.data != null, "Missing post data"))
        table
      }

  def detail(slug: String, signal: Option[dom.AbortSignal]): Future[BlogPost] =
    HttpJson.get[BlogPostData](s"/service/blog/posts/${encodeURIComponent(slug)}", signal)
      .map { result =>
        require(result.data != null && result.data.content.get != null, "Missing post detail")
        result.data
      }
}
```

The editorial service follows the session's editorial link as before,
sets the validated destination's query to
`search.queryString(includeDefaults = true)`, and sends the route's
AbortSignal. Its new-post loader uses a one-row default editorial search
to obtain the collection's create capability.

In [BlogRoutes.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogRoutes.scala), the two list loaders become:

```scala
    Route.view("/editorial") { context =>
      val search = PostSearch.parse(context.queryParams.get, editorial = true)
      editorial.list(search, context.signal).map(new EditorialListPage(_, search))
    },
```

```scala
    Route.view("/") { context =>
      val search = PostSearch.parse(context.queryParams.get, editorial = false)
      service.list(search, context.signal)
        .map(table => new PostListPage(table, search, actions))
    },
```

These entries belong in the existing route sequence. Its services, error
routes and imports remain in place. An invalid search is a 400 route failure
and uses the existing error page. A new navigation aborts the previous load,
so a late result cannot replace the new route.

## 6. Build one search form for both lists

The currently displayed results belong to an immutable `PostSearch`.
The user can change the controls before submitting, so those pending values
need their own form state.

Create [PostSearchForm.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostSearchForm.scala) with the complete implementation below:

```scala
package com.anjunar.blog.frontend

import ui.core.component.AbstractComponent
import ui.core.dsl.AttributeDsl
import ui.core.dsl.AttributeDsl.*
import ui.core.dsl.ClassDsl.classes
import ui.core.dsl.DslLayer.render
import ui.core.dsl.EventDsl.on
import ui.core.i18n.{I18nRuntime, i18n}
import ui.core.layout.Button.{button, buttonType}
import ui.core.layout.Condition.when
import ui.core.layout.Div.div
import ui.core.layout.Label.label
import ui.core.layout.Paragraph.paragraph
import ui.core.layout.TextComponent.text
import ui.core.render.Cursor
import ui.core.state.Property
import ui.forms.Form.form
import ui.forms.Input.{input, inputType, inputType_=}
import ui.forms.SelectInput.selectInput
import ui.forms.SelectOption
import ui.forms.validators.Size
import ui.router.Router
import ui.router.RouterLink.routerLink

import scala.annotation.meta.field
import scala.scalajs.js

// Search controls are route state, separate from the editable post model.
final class PostSearchFields(search: PostSearch) {
  @(Size @field)(max = 100)
  val query: Property[String] = Property(search.query)
  val status: Property[String] = Property(search.status)
  val sort: Property[String] = Property(search.sort)
  val limit: Property[String] = Property(search.limit.toString)

  def submitted(editorial: Boolean): PostSearch = {
    val values = Map("q" -> query.get, "status" -> status.get, "sort" -> sort.get, "limit" -> limit.get)
    // A new search always starts at offset zero.
    PostSearch.parse(values.get, editorial)
  }
}

final class PostSearchForm(search: PostSearch) extends AbstractComponent {
  val tagName = "div"
  private val fields = new PostSearchFields(search)
  private val invalid = Property(false)

  override def compose(cursor: Cursor): Unit = {
    val translations = I18nRuntime.current(using this).get
    render(this, cursor) {
      classes = "post-search"
      form(fields) { mountedForm ?=>
        classes = "search-form"
        role = "search"
        AttributeDsl.setAttribute("novalidate", "")
        on("submit") { event =>
          event.preventDefault()
          invalid.set(false)
          val bindings = mountedForm.validateBindings()
          val errors = mountedForm.validate()
          if (bindings.nonEmpty || errors.nonEmpty) invalid.set(true)
          else {
            try Router.navigate(fields.submitted(search.editorial).url)
            catch { case _: HttpFailure => invalid.set(true) }
          }
        }
        div {
          classes = "search-query"
          label {
            AttributeDsl.setAttribute("for", "post-query")
            text(i18n"Search posts") {}
          }
          val control = input("query") { fieldInput ?=>
            id = "post-query"
            inputType = "search"
            AttributeDsl.setAttribute("maxlength", "100")
            AttributeDsl.setAttribute("aria-describedby", "search-help search-errors")
            fieldInput.addDisposable(fieldInput.invalid.observe(value =>
              AttributeDsl.setAttribute("aria-invalid", value.toString)))
          }
          paragraph {
            id = "search-help"
            classes = "field-help"
            text(i18n"Search title, slug or summary.") {}
          }
          paragraph {
            id = "search-errors"
            classes = "field-error"
            text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {}
          }
        }
        if (search.editorial) {
          div {
            label {
              AttributeDsl.setAttribute("for", "post-status")
              text(i18n"Publication status") {}
            }
            selectInput("status", Seq(
              SelectOption("", translations.text(i18n"All statuses")),
              SelectOption("DRAFT", translations.text(i18n"Draft")),
              SelectOption("PUBLISHED", translations.text(i18n"Published"))
            )) { id = "post-status" }
          }
        }
        div {
          label {
            AttributeDsl.setAttribute("for", "post-sort")
            text(i18n"Sort by") {}
          }
          selectInput("sort", Seq(
            SelectOption("newest", translations.text(i18n"Newest first")),
            SelectOption("oldest", translations.text(i18n"Oldest first")),
            SelectOption("title", translations.text(i18n"Title A–Z")),
            SelectOption("title-desc", translations.text(i18n"Title Z–A"))
          )) { id = "post-sort" }
        }
        div {
          label {
            AttributeDsl.setAttribute("for", "post-limit")
            text(i18n"Posts per page") {}
          }
          selectInput("limit", (Seq(10, 20, 50, 100) :+ search.limit).distinct.sorted.map(value =>
            SelectOption(value.toString, Property(value.toString)))) { id = "post-limit" }
        }
        div {
          classes = "search-buttons"
          button(i18n"Search") { buttonType("submit") }
          routerLink(search.path) { text(i18n"Reset search") {} }
        }
        when(invalid) {
          paragraph { role = "alert"; text(i18n"Check the search fields and try again.") {} }
        }
      }
    }
  }
}
```

`PostSearchFields` contains only pending search values, not copies of
entity fields. Its `submitted` method deliberately omits offset, which
returns the next search to zero.

Attribute writes use `AttributeDsl.setAttribute`, with `AttributeDsl` imported
from `ui.core.dsl`. This avoids the inherited component setter with the same
name and targets the component supplied by the enclosing DSL block. The
reactive `aria-invalid` observer stays inside the input block and is disposed
with that input. Label associations, maximum length and `novalidate` follow
the same DSL path.

The native submit handler, registered through `on("submit")`, validates the binding and the text constraint
before navigating. Typing by itself does not issue a request or relabel the
currently displayed result. Pressing Search or Enter applies the new values
together.

The select controls bind strings just like the search input. Status is shown
only for editorial. The page-size choices normally contain 10, 20, 50 and
100; a valid size from a direct URL is added as well. That lets a one-row
test or shared link restore its control correctly.

The entire form tree stays in `compose`, including labels, submission,
options and validation feedback. This is a shared component because the same
form has a complete role on both list pages. New visible messages use the
i18n macro; the numeric page-size options need no translation.

### Mount the form and preserve state while paging

[PostListPage](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostListPage.scala)
and [EditorialListPage](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/EditorialListPage.scala)
receive the loaded search alongside the table. They mount
`child(new PostSearchForm(search)) {}` before their results.

The public previous/next links use a copy of that search with the new offset.
The editorial links use the server's advertised previous/next URLs after
the existing API-link guard validates them. The labels are **Previous page**
and **Next page**, since a title sort has no meaningful “older” direction.

When there are no matches, the page says so. When matches exist but the offset
is beyond them, it retains the total and offers **First matching page**.
That link keeps the filters. **Reset search** instead goes to the bare list
route and restores its defaults.

Reload and browser history load from the URL again, which reconstructs both
the query and the controls. We do not keep a second hidden filter store that
can disagree with the displayed results.

The checkpoint also includes the [responsive form styles](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/resources/style.css).

## 7. Verify the route behavior and real query results

Here is the complete frontend [PostSearchSpec.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/test/scala/com/anjunar/blog/frontend/PostSearchSpec.scala), under
`application/frontend/src/test/scala/com/anjunar/blog/frontend/`:

```scala
package com.anjunar.blog.frontend

import org.scalatest.funsuite.AnyFunSuite

class PostSearchSpec extends AnyFunSuite {
  private def parse(values: (String, String)*)(editorial: Boolean = false): PostSearch =
    PostSearch.parse(values.toMap.get, editorial)

  test("public and editorial defaults keep their existing ordering") {
    assert(parse()() == PostSearch())
    assert(parse()(editorial = true).sort == "title")
    assert(parse()().url == "/")
    assert(parse()(editorial = true).url == "/editorial")
  }

  test("filter values are encoded once and survive a page change") {
    val value = parse("q" -> "  100%_! & + café  ", "sort" -> "title-desc", "limit" -> "10")()
    assert(value.query == "100%_! & + café")
    val next = value.copy(offset = 10)
    assert(next.url.startsWith("/?offset=10&limit=10&q=100%25_!%20%26%20%2B%20caf%C3%A9"))
    assert(next.url.endsWith("&sort=title-desc"))
    assert(next.copy(offset = 0).query == value.query)
  }

  test("submitting changed filters resets the page without changing the current route state") {
    val current = PostSearch("old", "DRAFT", "title", 40, 10, editorial = true)
    val form = new PostSearchFields(current)
    form.query.set("new")
    form.status.set("PUBLISHED")
    val submitted = form.submitted(editorial = true)
    assert(submitted.offset == 0 && submitted.limit == 10 && submitted.query == "new")
    assert(submitted.status == "PUBLISHED" && submitted.sort == "title")
    assert(current.offset == 40 && current.query == "old")
  }

  test("malformed routes fail before issuing an HTTP request") {
    for ((key, value) <- Seq("offset" -> "-1", "offset" -> "2147483648", "limit" -> "0",
      "limit" -> "101", "sort" -> "content", "status" -> "DRAFT", "q" -> ("a" * 101), "q" -> "a\u0000b")) {
      assert(intercept[HttpFailure](parse(key -> value)()).status == 400)
    }
    assert(parse("status" -> "DRAFT")(editorial = true).status == "DRAFT")
  }

  test("API requests include bounds while default browser URLs stay short") {
    assert(PostSearch().queryString(includeDefaults = true) == "offset=0&limit=20")
    assert(PostSearch(offset = 20).url == "/?offset=20")
    assert(parse("limit" -> "1")().limit == 1)
  }
}
```

The tests check different parts of the contract: default ordering, encoded
filter values, preserved page size, offset reset and rejection before HTTP.
The submitted-filter test also confirms that editing the pending controls
does not mutate the search that describes the current results.

Database behavior needs database tests. The backend
[PostSearchRestSpec](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/test/scala/com/anjunar/blog/PostSearchRestSpec.scala)
inserts its own posts and exercises the real endpoints. It checks:

- Title, slug and summary matches, with the same filtered row count.
- Literal percent signs, underscores and escape characters.
- Page-link round trips for Unicode, `%20` and braces.
- Equal-key ordering, undated drafts, empty pages and integer bounds.
- Published-only public searches, including for an administrator.
- Rejection of unauthenticated and reader access to editorial searches.

The [real browser workflow](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/tests/browser/post-search-database.spec.mjs)
then uses the actual controls with PostgreSQL. It searches, changes pages,
reloads, matches literal punctuation, signs in and finds only drafts in
editorial. Its rows are removed in a finally block.

Run the checks with the isolated database, test administrator and local SMTP
capture settings from the previous chapters:

```text
sbt --server "application-backend/testFull" "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
npx playwright test --project=search --project=forms --project=changes --project=editorial --project=database --project=authentication --project=recovery
```

The search browser project needs psql on PATH or `BLOG_PSQL`, and
`BLOG_TEST_ADMIN_EMAIL` / `BLOG_TEST_ADMIN_PASSWORD`. Keep port 18080
free and run backend and browser suites sequentially.

The additional [HibernateSearchSpec](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/test/scala/com/anjunar/blog/HibernateSearchSpec.scala)
runs against real CDI and PostgreSQL. It declares an independent `ById`
search and provider without changing the engine, selects both entities
and scalar titles, and verifies matching counts. It also verifies that a
missing predicate or sort provider fails, and that internal callers cannot
bypass the page bounds.

The completed checkpoint passes 125 backend tests, 26 Scala.js tests and
56 browser tests: 48 controlled contracts and eight real workflows.
The browser checks also cover back/forward, reset, empty results, mobile
layout and a slow search that must not overwrite later navigation.

## What this search promises

This is a literal substring search over three metadata fields. It is not
full-text ranking, body search or accent-insensitive matching. Lower-case
LIKE with a leading wildcard can scan many rows; a larger blog should measure
its queries before choosing an index or another search strategy.

Rows and total are separate statements under the existing READ COMMITTED
transaction isolation. Concurrent changes can affect them differently.
Likewise, deterministic ordering does not make offset pages a snapshot.
Very deep offsets can still be expensive despite bounded page sizes.

Within those boundaries, we now have a complete search path: validated
parameters, typed predicates, a deliberately small projection, filtered
counts and a URL-driven interface. The next chapter introduces authors, tags and safe entity references.
Media uploads follow separately in chapter 18.

