Chapter 5 gave us a `blog_post` table and a working entity. Now imagine that the table contains posts we want to keep, but the application needs another field.

This chapter adds an optional summary without recreating the table. We first teach Hibernate DDL Manager about the existing schema, then change the entity and apply the resulting migration. The publication constraint from chapter 5 stays active throughout.

Start with the [chapter 5 project](https://github.com/anjunar/anjunar-blog-example/tree/7259db379231b095c503069f6a5950f385302e01) and its PostgreSQL database. Stop the HTTP server while following these steps. All file paths below are relative to the project root.

## Give the schema a history

Hibernate currently runs with `hibernate.hbm2ddl.auto=validate`. It can detect a missing mapped column, but it will not add it. We keep that setting.

Hibernate DDL Manager takes the mapped schema, compares it with its recorded model, checks the actual database, and applies a migration in a database transaction. It records the result in `__hibernate_ddl.schema_history`.

There are three states to distinguish:

| State | Meaning |
| --- | --- |
| Entity mapping | The schema this application version wants. |
| Recorded model | The schema the last successful migration established. |
| PostgreSQL catalog | The schema that actually exists now. |

The catalog check matters: changing a table manually must not silently invalidate the recorded history.

Add this dependency to the existing sequence in `build.sbt`:

```scala
"com.anjunar.hibernateddl" %% "schema-integration" % "1.1.0",
```

Version 1.1.0 supports the named, multi-column CHECK constraint used by our publication rule. The dependency comes from Maven Central; readers do not need a local framework checkout.

## Give tables and columns stable identities

A column name can change. Its identity should survive that change. Import the annotation into `BlogPost.scala`:

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
```

Add `@SchemaId("d4f39c20")` to the class and an ID to each persistent field:

| Element | Schema ID |
| --- | --- |
| `BlogPost` | `d4f39c20` |
| `id` | `a2473e8b` |
| `version` | `dcb0681e` |
| `slug` | `682d9ace` |
| `title` | `46fdb02a` |
| `content` | `7b20efc1` |
| `status` | `cf271a06` |
| `publishedAt` | `398bfd50` |

For example, the title remains:

```scala
@NotBlank
@Size(min = 3, max = 180)
@Column(nullable = false, length = 180)
@SchemaId("46fdb02a")
var title: String = ""
```

These are field declarations inside the class body, so the annotations need no explicit `@field` target.

The manager combines the table and field IDs: the title's identity is `d4f39c20/46fdb02a`. Choose IDs once, commit them, and retain them when editing the same schema element. Do not regenerate them on each build.

These IDs identify schema elements. A post's UUID still identifies a row, and its `version` still detects stale edits. Neither replaces `@SchemaId`. This annotation also has a different purpose from the `EntitySchema` we will introduce next.

## Include the database rule in the mapping

Chapter 5 created two database checks: allowed status values, and consistency between status and publication time. Hibernate derives the allowed values from the enum mapping. We must explicitly describe the second rule.

Add `CheckConstraint` to the existing `jakarta.persistence` import and replace the class's `@Table` annotation:

```scala
@Table(name = "blog_post", schema = "public",
  uniqueConstraints = Array(new UniqueConstraint(name = "uq_blog_post_slug", columnNames = Array("slug"))),
  check = Array(new CheckConstraint(name = "ck_blog_post_publication",
    constraint = "(status = 'DRAFT' AND published_at IS NULL) OR (status = 'PUBLISHED' AND published_at IS NOT NULL)")))
```

The expression matches the existing SQL rule. Keep the Bean Validation annotations and publication methods as well. They give useful errors in the application; the database rule also protects writes made outside Hibernate.

The [baseline checkpoint](https://github.com/anjunar/anjunar-blog-example/tree/b693dba5c8e3402a1249bdebf38bc9503d45e99a) contains the complete annotated entity. At this point, it still has exactly the seven fields from chapter 5.

## Run migrations as a separate command

We already discover entity classes through CDI. The migration command uses that same `EntityRegistry`, so adding another entity does not require updating a second class list.

Create `application/backend/src/main/scala/com/anjunar/blog/SchemaMain.scala`:

```scala
package com.anjunar.blog

import com.anjunar.hibernateddl.executor.{ExecutionOptions, PreviewOutcome, PreviewRendering}
import com.anjunar.hibernateddl.integration.HibernateSchemaMigration
import jakarta.enterprise.inject.se.SeContainerInitializer
import org.hibernate.boot.{Metadata, MetadataSources}
import org.hibernate.boot.registry.StandardServiceRegistryBuilder
import org.postgresql.ds.PGSimpleDataSource

import javax.sql.DataSource
import scala.util.Using

object SchemaMain {
  def main(args: Array[String]): Unit = {
    val (command, adoptExisting) = args.toList match {
      case List("preview") => ("preview", false)
      case List("preview", "--adopt-existing") => ("preview", true)
      case List("migrate") => ("migrate", false)
      case List("migrate", "--adopt-existing") => ("migrate", true)
      case _ => throw new IllegalArgumentException(
        "Usage: SchemaMain preview|migrate [--adopt-existing]")
    }

    val config = DatabaseConfig.load()
    val exitCode = Using.resource(SeContainerInitializer.newInstance().initialize()) { container =>
      val entities = container.select(classOf[EntityRegistry]).get()
      withMetadata(config, entities) { (metadata, dataSource) =>
        val options = ExecutionOptions(adoptExistingSchema = adoptExisting)
        command match {
          case "preview" =>
            val report = HibernateSchemaMigration.preview(metadata, dataSource, options)
            println(PreviewRendering.text(report))
            report.outcome match {
              case PreviewOutcome.Ready => 0
              case PreviewOutcome.Blocked => 2
              case PreviewOutcome.Incomplete => 3
            }
          case "migrate" =>
            val result = HibernateSchemaMigration.migrate(metadata, dataSource, options)
            println(s"${result.status}: revision ${result.revision}, ${result.statementCount} SQL statements")
            0
        }
      }
    }
    if (exitCode != 0) sys.exit(exitCode)
  }

  private[blog] def withMetadata[A](config: DatabaseConfig, entities: EntityRegistry)(
      work: (Metadata, DataSource) => A): A = {
    val dataSource = new PGSimpleDataSource()
    dataSource.setUrl(config.url)
    dataSource.setUser(config.user)
    dataSource.setPassword(config.password)
    dataSource.setLoginTimeout(5)

    // The migration executor owns a JDBC transaction, outside the request's JTA boundary.
    val registry = new StandardServiceRegistryBuilder()
      .applySetting("hibernate.connection.datasource", dataSource)
      .applySetting("hibernate.hbm2ddl.auto", "validate")
      .build()
    try {
      val sources = new MetadataSources(registry)
      entities.entityClasses.foreach(sources.addAnnotatedClass)
      work(sources.buildMetadata(), dataSource)
    } finally StandardServiceRegistryBuilder.destroy(registry)
  }
}
```

The command boots CDI, builds Hibernate metadata from the discovered classes, performs its work, and closes CDI and Hibernate's service registry. It does not build a SessionFactory or start Undertow.

The `PGSimpleDataSource` is deliberately outside the request's JTA transaction. The migration executor owns its JDBC transaction, including schema changes and history. Request handling continues to use Agroal and Narayana as before.

Both commands read the existing `BLOG_DB_URL`, `BLOG_DB_USER`, and `BLOG_DB_PASSWORD` variables. The application role must own the managed tables and be able to create the history schema.

## Adopt the chapter 5 database

Readers using the companion repository can reproduce this exact intermediate state with:

```text
git switch --detach b693dba5c8e3402a1249bdebf38bc9503d45e99a
```

Use this checkout before running the adoption commands. Do not adopt the final mapping against a chapter 5 table that still lacks its new column.

Do this before adding the summary field. Adoption verifies that the existing database matches the current mapping, then records revision 1 without altering the application table.

If you want a concrete row to follow, run this once in the chapter 5 database using `psql`:

```sql
INSERT INTO public.blog_post
    (id, version, slug, title, content, status)
VALUES
    ('a090f23a-42bb-40c6-a21a-0ee8df71e7c3', 0, 'migration-example',
     'Keep this title', 'Keep this content', 'DRAFT');
```

First normalize the enum constraint's name. The manager derives that name from the status column's stable ID and allowed values. Our original script used `ck_blog_post_status`.

Create `database/002-adopt-check-name.sql`:

```sql
ALTER TABLE public.blog_post
    RENAME CONSTRAINT ck_blog_post_status TO ck_6576a412cbd2bfbd503a3350;
```

This is the expected name for the IDs and enum values above. Renaming the constraint changes neither its predicate nor the stored rows. Apply the script once. With Compose:

```text
docker compose cp database/002-adopt-check-name.sql postgres:/tmp/002-adopt-check-name.sql
docker compose exec -T postgres psql -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --single-transaction --file /tmp/002-adopt-check-name.sql
```

With native PostgreSQL:

```text
psql -h 127.0.0.1 -p 5433 -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --single-transaction --file database/002-adopt-check-name.sql
```

Use your configured port and database if different. Do not rerun either chapter's setup SQL against an already prepared database.

Now inspect adoption:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview --adopt-existing"
```

Expect `INCOMPLETE`, with a finding about normalizing the named publication check. The preview is read-only. Comparing an arbitrary SQL predicate requires a temporary probe table, which this preview cannot create. The command deliberately exits with code 3; sbt reports that nonzero exit.

Inspect the findings. An unknown predicate is different from a missing column or an unexpected constraint. Do not proceed if the report contains a blocker.

Apply adoption:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate --adopt-existing"
```

Expect:

```text
Adopted: revision 1, 0 SQL statements
```

The zero counts application-schema migration statements. Adoption does write the history. During execution, the manager compares PostgreSQL's normalized check definitions under its migration lock. A constraint with the right name but a weaker expression is rejected.

## Add the summary

With revision 1 recorded, add this field to `BlogPost`:

```scala
@Size(max = 300)
@Column(length = 300)
@SchemaId("0ca6e520")
var summary: String = null
```

A summary is optional. Existing posts have no summary, so SQL NULL is the correct initial value. `@Size` permits null and limits a provided value to 300 characters. A JVM initializer would not fill existing database rows.

Inspect the change:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
```

The plan adds one nullable column:

```sql
ALTER TABLE "public"."blog_post" ADD COLUMN "summary" varchar(300);
```

The publication check still makes this read-only preview `INCOMPLETE`. Execution will verify that predicate before applying the change.

Run the migration without the adoption flag:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

Expect `Applied: revision 2, 1 SQL statements`. Run the same command again and expect `AlreadyApplied: revision 2, 0 SQL statements`. A repeated run checks the database but does not add another revision.

## Verify the result

Inspect the retained row:

```sql
SELECT slug, title, content, summary, status, published_at
FROM public.blog_post
WHERE id = 'a090f23a-42bb-40c6-a21a-0ee8df71e7c3';
```

The original slug, title, content, and status remain. Both `summary` and `published_at` are NULL.

Check the rule using a separate `psql` command:

```sql
UPDATE public.blog_post
SET status = 'PUBLISHED'
WHERE id = 'a090f23a-42bb-40c6-a21a-0ee8df71e7c3';
```

PostgreSQL rejects this with SQLSTATE `23514`: publication still requires a timestamp. The failed statement leaves the draft intact.

Then run the application checks:

```text
sbt --server "application-backend/testFull"
sbt --server "application-backend/run"
```

Expect **28 successful tests**, including optional summaries, their length limit, persistence, CDI discovery, and the existing transaction checks. Readiness at `/service/health/ready` should return HTTP 200 with `UP`.

## Start from an empty database

A reader starting directly from this chapter's final code does not need to create and adopt the old table.

With an empty tutorial database and the same environment variables, run:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

The manager creates the current table, including summary and both checks, and records revision 1. Skip `001-blog-post.sql`, `002-adopt-check-name.sql`, and `--adopt-existing` on this path.

## Know when a migration needs more work

Stable IDs allow the planner to recognize an existing schema element even when its name changes. That does not make every change automatic.

Version 1.1.0 treats named check expressions as SQL, without attempting to rewrite them. Changing such a check, or renaming or dropping columns in its table, requires an explicit manual migration and the manager's `acceptManualMigration` workflow. We do not expose that workflow in this small CLI.

A required new field also needs a decision about existing rows: which value is correct, how it is populated, and when it becomes required. Adding a JVM default is not a database backfill.

For this chapter we have a complete, small change: adopt the existing model, add an optional summary, preserve the posts, and keep the database rule.

The [completed chapter 6 source](https://github.com/anjunar/anjunar-blog-example/tree/185a0fd7634f1da3e7f7b420a806a022033cd242) contains the runnable result. Next we describe the entity with `EntitySchema`, giving the JSON mapper and Criteria queries a shared field model.
