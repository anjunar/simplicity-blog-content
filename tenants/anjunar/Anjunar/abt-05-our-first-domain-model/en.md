We now have a server, a database connection, and a transaction boundary. This chapter gives them something useful to store: `BlogPost`.

By the end, we can persist a draft, load it in a new persistence context, publish it, and reject an edit based on an old version. These behaviors are verified against PostgreSQL. Public post endpoints arrive in chapter 8.

Start with the [chapter 4 project](https://github.com/anjunar/anjunar-blog-example/tree/e96365e906bb14b212fe2b0b9e11664f6b5afd93) and its local PostgreSQL setup. All paths below are relative to the project root.

## Define the first post

Our initial model has seven fields:

| Field | Purpose |
| --- | --- |
| `id` | Generated UUID identifying the post throughout its lifetime. |
| `version` | Value Hibernate uses to detect an update based on stale data. |
| `slug` | Unique, readable identifier for the future public URL. |
| `title` | Required title, between 3 and 180 characters. |
| `content` | Text content, up to 100,000 characters; a draft may leave it empty. |
| `status` | `DRAFT` or `PUBLISHED`. |
| `publishedAt` | The current publication time, absent for a draft. |

A title can change while the slug stays the same. This keeps a wording correction from automatically changing the post's future URL. The UUID remains the identity used to load and reference the entity.

We start with English text in the entity. Author relationships, structured editor content, and translations will extend the model in their respective chapters.

## Add Bean Validation

Add these dependencies to the existing sequence in `build.sbt`:

```scala
"org.hibernate.validator" % "hibernate-validator" % "9.1.4.Final",
"org.glassfish.expressly" % "expressly" % "6.0.0",
```

Hibernate Validator implements Jakarta Validation. Expressly supplies the expression-language implementation used by its default message interpolation. The [Hibernate Validator documentation](https://hibernate.org/validator/documentation/) covers these dependencies and constraint validation.

The annotations will describe which field values are valid. We will also enable validation during Hibernate's persistence lifecycle so the rules run before inserts and updates.

## Represent publication status

Create `application/backend/src/main/java/com/anjunar/blog/BlogPostStatus.java`:

```java
package com.anjunar.blog;

public enum BlogPostStatus {
    DRAFT,
    PUBLISHED
}
```

This small Java enum works directly with JPA's enum mapping. sbt compiles it together with the Scala sources.

## Map the entity

Create `application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala`:

```scala
package com.anjunar.blog

import jakarta.persistence.{Access, AccessType, Column, Entity, Enumerated, EnumType, GeneratedValue, GenerationType, Id, Table, Transient, UniqueConstraint, Version}
import jakarta.validation.constraints.{AssertTrue, NotBlank, NotNull, Pattern, Size}

import java.lang
import java.time.Instant
import java.util.UUID

@Entity
@Access(AccessType.FIELD)
@Table(name = "blog_post", schema = "public",
  uniqueConstraints = Array(new UniqueConstraint(name = "uq_blog_post_slug", columnNames = Array("slug"))))
class BlogPost {
  @Id
  @GeneratedValue(strategy = GenerationType.UUID)
  @Column(nullable = false, updatable = false)
  var id: UUID = null

  @Version
  @Column(nullable = false)
  var version: lang.Long = null

  @NotBlank
  @Size(min = 3, max = 220)
  @Pattern(regexp = "^[a-z0-9]+(?:-[a-z0-9]+)*$")
  @Column(nullable = false, length = 220)
  var slug: String = ""

  @NotBlank
  @Size(min = 3, max = 180)
  @Column(nullable = false, length = 180)
  var title: String = ""

  @NotNull
  @Size(max = 100000)
  @Column(nullable = false, columnDefinition = "text")
  var content: String = ""

  @NotNull
  @Enumerated(EnumType.STRING)
  @Column(nullable = false, length = 24)
  var status: BlogPostStatus = BlogPostStatus.DRAFT

  @Column(name = "published_at")
  var publishedAt: Instant = null

  def publish(at: Instant): Unit = {
    require(status == BlogPostStatus.DRAFT, "Only a draft can be published")
    require(at != null, "Publication time is required")
    require(content != null && !content.isBlank, "A published post needs content")
    status = BlogPostStatus.PUBLISHED
    publishedAt = at
  }

  def retract(): Unit = {
    require(status == BlogPostStatus.PUBLISHED, "Only a published post can be retracted")
    status = BlogPostStatus.DRAFT
    publishedAt = null
  }

  @Transient
  @AssertTrue(message = "Publication status, time, and content must be consistent")
  def isPublicationConsistent: Boolean =
    status match {
      case BlogPostStatus.DRAFT => publishedAt == null
      case BlogPostStatus.PUBLISHED => publishedAt != null && content != null && !content.isBlank
      case null => false
    }
}
```

A regular class supplies the no-argument constructor Hibernate needs. `@Access(AccessType.FIELD)` tells Hibernate to read and write fields directly. Our `var` declarations are in the class body, so plain JPA and Bean Validation annotations already target their fields; no `@field` is needed.

Constructor parameters are a different annotation site: use `@field` when an annotation on a constructor parameter must be placed on its backing field. The [Scala annotation-target documentation](https://www.scala-lang.org/api/3.x/scala/annotation/meta.html) explains these defaults. This compiler targeting is separate from JPA's field-access strategy.

The generated UUID and version start as `null`. Hibernate supplies them when the entity is persisted. We use `import java.lang` and `lang.Long` for the nullable Java wrapper; Hibernate owns the version value and its increments. The [Jakarta Persistence specification](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2) defines generated UUID identifiers and version attributes.

`EnumType.STRING` stores status names such as `DRAFT` and `PUBLISHED`. `Instant` represents the publication time independently of a reader's time zone.

The title and slug must be present even for a draft. The slug accepts lowercase ASCII letters, digits, and single separating hyphens, with a length of 3 to 220 characters. A newly constructed object therefore needs a title and slug before it is valid.

## Keep publication changes consistent

Publishing changes two fields together: status becomes `PUBLISHED` and `publishedAt` receives the supplied instant. It requires nonblank content. Retraction restores `DRAFT` and clears the current publication time while retaining the text.

The methods check their prerequisites before changing either field. Calling `publish` again on a published post therefore leaves its publication time unchanged and reports an invalid transition.

The fields remain mutable because they form our persistence model. `isPublicationConsistent` checks their relationship even when another caller assigns the fields directly. Its JavaBean-style name makes it a validation property; `@AssertTrue` requires its result to be true. It is not stored as a column.

These checks serve different purposes:

| Mechanism | What it establishes |
| --- | --- |
| `publish` and `retract` | An explicit state transition with prerequisites. |
| Field constraints | Required values, lengths, and slug format. |
| `isPublicationConsistent` | The resulting publication state is coherent. |
| Database constraints | Unique slugs and valid stored status/time combinations. |

Calling a transition method does not run every field constraint immediately. Hibernate's validation callbacks check the whole entity before writing it.

## Create the first table explicitly

Create `database/001-blog-post.sql`:

```sql
CREATE TABLE public.blog_post (
    id uuid PRIMARY KEY,
    version bigint NOT NULL,
    slug varchar(220) NOT NULL,
    title varchar(180) NOT NULL,
    content text NOT NULL,
    status varchar(24) NOT NULL,
    published_at timestamp(6) with time zone,
    CONSTRAINT uq_blog_post_slug UNIQUE (slug),
    CONSTRAINT ck_blog_post_status CHECK (status IN ('DRAFT', 'PUBLISHED')),
    CONSTRAINT ck_blog_post_publication CHECK (
        (status = 'DRAFT' AND published_at IS NULL)
        OR (status = 'PUBLISHED' AND published_at IS NOT NULL)
    )
);
```

The unique constraint enforces slug uniqueness across this application, including simultaneous inserts. The status and publication checks also apply to SQL executed outside Hibernate. Content and title validation remain in the model; these mechanisms do not automatically copy all their rules into each other.

With the Compose database from chapter 4 running, apply the script once:

```text
docker compose cp database/001-blog-post.sql postgres:/tmp/001-blog-post.sql
docker compose exec -T postgres psql -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --single-transaction --file /tmp/001-blog-post.sql
```

These commands work in both PowerShell and Bash. For native PostgreSQL, run its `psql` client with your connection details:

```text
psql -h 127.0.0.1 -p 5433 -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --single-transaction --file database/001-blog-post.sql
```

Adjust the port if necessary. The native client prompts for the database password when needed.

The script expects a database without this table and reports an error if it already exists. Keep existing data and apply the script only once. We will introduce tracked schema migrations in the next chapter.

## Discover entity classes through CDI

The persistence layer should accept another entity without requiring an edit to its bootstrap code. We can use the CDI extension mechanism introduced in chapter 3 to collect the classes for Hibernate.

First, change `application/backend/src/main/resources/META-INF/beans.xml` to:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       version="4.0"
       bean-discovery-mode="all">
</beans>
```

`@Entity` is not a CDI bean-defining annotation. The previous `annotated` mode therefore does not discover a class merely because it has `@Entity`. With `all`, CDI Full discovers the types in this bean archive and delivers their `ProcessAnnotatedType` events. Each future module containing entities needs the same discovery setup.

Create `application/backend/src/main/scala/com/anjunar/blog/EntityRegistry.scala`:

```scala
package com.anjunar.blog

class EntityRegistry(val entityClasses: List[Class[?]])
```

This registry holds the entity classes found during startup. It contains class metadata, not entity instances.

Create `application/backend/src/main/scala/com/anjunar/blog/EntityExtension.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.spi.CreationalContext
import jakarta.enterprise.event.Observes
import jakarta.enterprise.inject.{Any as AnyQualifier, Default}
import jakarta.enterprise.inject.spi.{AfterBeanDiscovery, Extension, ProcessAnnotatedType, WithAnnotations}
import jakarta.inject.Singleton
import jakarta.persistence.Entity

import java.util.concurrent.ConcurrentHashMap
import scala.jdk.CollectionConverters.*

class EntityExtension extends Extension {
  private val entityClasses = ConcurrentHashMap.newKeySet[Class[?]]()

  def collect(@Observes @WithAnnotations(Array(classOf[Entity])) event: ProcessAnnotatedType[?]): Unit = {
    val annotatedType = event.getAnnotatedType
    if (annotatedType.isAnnotationPresent(classOf[Entity])) {
      entityClasses.add(annotatedType.getJavaClass)
      // Hibernate manages entity instances; CDI only discovers their classes.
      event.veto()
    }
  }

  def registerRegistry(@Observes event: AfterBeanDiscovery): Unit = {
    val discovered = entityClasses.asScala.toList.sortBy(_.getName)
    event.addBean[EntityRegistry]()
      .beanClass(classOf[EntityRegistry])
      .types(classOf[EntityRegistry], classOf[Object])
      .scope(classOf[Singleton])
      .qualifiers(Default.Literal.INSTANCE, AnyQualifier.Literal.INSTANCE)
      .createWith((_: CreationalContext[EntityRegistry]) => new EntityRegistry(discovered))
  }
}
```

`@WithAnnotations` filters discovery events. For each type annotated with `@Entity`, the observer saves its class and calls `veto()` to exclude it from CDI bean registration. Hibernate manages the entity instances.

After discovery, the extension takes an immutable, sorted snapshot and registers an `EntityRegistry` bean with singleton scope. `createWith` supplies its construction; `@Default` makes it available to an ordinary injection point. The registry belongs to this CDI container. The [CDI specification](https://jakarta.ee/specifications/cdi/4.1/jakarta-cdi-spec-4.1) describes these discovery events and synthetic beans.

Append the extension to `application/backend/src/main/resources/META-INF/services/jakarta.enterprise.inject.spi.Extension`, keeping the existing entry:

```text
com.anjunar.blog.RestComponentsExtension
com.anjunar.blog.EntityExtension
```

## Register the discovered classes with Hibernate

In `application/backend/src/main/scala/com/anjunar/blog/Persistence.scala`, add these imports:

```scala
import jakarta.inject.Inject
import org.hibernate.engine.transaction.jta.platform.internal.NarayanaJtaPlatform
import scala.compiletime.uninitialized
```

Add this field inside the existing `Persistence` class:

```scala
@Inject
var entityRegistry: EntityRegistry = uninitialized
```

CDI injects the registry before calling `@PostConstruct initialize()`. Replace the registry and metadata construction inside that method's existing `try` block with:

```scala
registry = new StandardServiceRegistryBuilder()
  .applySetting("jakarta.persistence.jtaDataSource", pool)
  .applySetting("hibernate.transaction.coordinator_class", "jta")
  .applySetting("hibernate.transaction.jta.platform",
    new NarayanaJtaPlatform())
  .applySetting("hibernate.hbm2ddl.auto", "validate")
  .applySetting("jakarta.persistence.validation.mode", "CALLBACK")
  .build()
val sources = new MetadataSources(registry)
entityRegistry.entityClasses.foreach(sources.addAnnotatedClass)
factory = sources.buildMetadata().buildSessionFactory()
```

Keep the existing pool construction, error cleanup, producer, and shutdown method. The [complete Persistence.scala](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/main/scala/com/anjunar/blog/Persistence.scala) shows the resulting file.

There is no hard-coded entity list in `Persistence`. Adding a mapped entity to a discovered bean archive makes it available to Hibernate. Its database schema still has to exist.

Two settings define the persistence checks:

- `validate` checks the mapped table and column structure during factory initialization.
- `CALLBACK` requires Bean Validation before entity inserts and updates.

The schema mode performs checks without creating or altering the table. Constraints in the SQL script still need their own tests; schema validation does not replace those checks.

The checkpoint also applies the project's import convention to the existing code: imported types and APIs, import aliases for Narayana access, and `lang.Long` or `lang.Integer` for Java wrappers.

## Detect an outdated edit

The `@Version` field connects an entity snapshot to a particular stored version. When a managed post changes, Hibernate increments that value as it writes the update.

Consider two editors loading the same post. The first editor saves a new title. The second editor's copy still carries the old version. Saving that copy must report a conflict while preserving the first change.

This is the actual test from `BlogPostPersistenceSpec`:

```scala
test("an old detached version cannot overwrite a newer committed edit") {
  val saved = savedDraft()
  val firstEditor = load(saved.id)
  val secondEditor = load(saved.id)
  firstEditor.title = "The committed title"
  inTransaction()(_.merge(firstEditor))
  secondEditor.title = "The stale title"
  intercept[OptimisticLockException] {
    inTransaction()(_.merge(secondEditor))
  }
  val current = load(saved.id)
  assert(current.title == "The committed title")
  assert(current.version.longValue() == saved.version.longValue() + 1)
}
```

The test's `load` helper reads in a separate persistence context each time. `inTransaction` uses the `Persistence` and `RequestTransaction` classes from chapter 4. The two copies therefore represent separate reads of the same stored version.

The first merge commits. The second merge raises `OptimisticLockException`, and the final read confirms that the newer title remains in the database. The complete helpers are in the [persistence test](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/test/scala/com/anjunar/blog/BlogPostPersistenceSpec.scala). We will carry this version through the REST and form contracts when those layers arrive.

## Run the checks

Add [BlogPostValidationSpec](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/test/scala/com/anjunar/blog/BlogPostValidationSpec.scala) and [BlogPostPersistenceSpec](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/test/scala/com/anjunar/blog/BlogPostPersistenceSpec.scala) together with [EntityDiscoveryProbe](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/test/scala/com/anjunar/blog/EntityDiscoveryProbe.scala) under `application/backend/src/test/scala/com/anjunar/blog` if you are reproducing the chapter by hand.

Set `bean-discovery-mode="all"` in `application/backend/src/test/resources/META-INF/beans.xml` as well. The persistence suite now starts a real CDI container and obtains `Persistence` from it, so registry injection and `@PostConstruct` run as they do in the application. Closing the container also closes the persistence resources.

`EntityDiscoveryProbe` is a second entity under `src/test`, with no CDI scope and no entry in `Persistence`. It maps just the ID and title of the existing table. One test checks that both entities reach the registry and are excluded from CDI bean resolution. Another saves a `BlogPost` and reads that row through the probe, proving that Hibernate received the additional mapping. The probe requires no extra table and is absent from a normal application run.

With PostgreSQL running, the table created, and the `BLOG_DB_*` environment variables configured as in chapter 4:

```text
sbt --server "application-backend/testFull"
```

Expect **26 successful tests**. The checks cover CDI entity discovery, field annotations, publication transitions, persistence across contexts, version increments, validation of inserts and updates, duplicate slugs, stale edits, and a database check constraint.

The model tests remove only the rows they created. The transaction tests from chapter 4 still create and drop their own probe table. Expected failure cases can produce error logs; check the final test summary.

You can also start the normal application and check readiness:

```text
sbt --server "application-backend/run"
```

In another terminal:

```text
curl -i http://127.0.0.1:8080/service/health/ready
```

Use `curl.exe` in Windows PowerShell if needed. Expect HTTP 200 and `UP`. Factory initialization now validates `blog_post`, so an absent table makes readiness fail with HTTP 500. Liveness continues to work independently of the database.

## Use the chapter checkpoint

The [complete chapter 5 source](https://github.com/anjunar/anjunar-blog-example/tree/7259db379231b095c503069f6a5950f385302e01) includes the model, SQL script, and tests. [Pull request #5](https://github.com/anjunar/anjunar-blog-example/pull/5) introduces the model, [pull request #6](https://github.com/anjunar/anjunar-blog-example/pull/6) simplifies its annotations, and [pull request #7](https://github.com/anjunar/anjunar-blog-example/pull/7) adds automatic entity discovery through CDI. The checkpoint includes all three changes. To use it in a separate directory:

```text
git clone https://github.com/anjunar/anjunar-blog-example.git
cd anjunar-blog-example
git switch --detach 7259db379231b095c503069f6a5950f385302e01
```

Configure PostgreSQL, apply the initial SQL script once, and run `testFull`. Use `git switch -c my-blog` to continue your own implementation.

The source and SQL script were verified against PostgreSQL 18.6. The tests confirm that our first entity can be stored and changed while enforcing its validation, uniqueness, and version rules.

Next we will change the schema while keeping existing posts. That introduces Hibernate DDL Manager, stable schema identifiers, and recorded migrations.
