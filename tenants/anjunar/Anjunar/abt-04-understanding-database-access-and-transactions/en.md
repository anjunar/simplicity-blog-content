A working HTTP server does not yet tell us whether database changes survive a request. This chapter adds PostgreSQL and makes that behavior explicit: successful writes commit, failed writes roll back, and reads leave no changes behind.

Our visible result is `GET /service/health/ready`. It executes a query through Hibernate and returns `UP`. The integration tests go further: they insert rows, force failures, and check the result through a separate database connection.

Start with the [chapter 3 project](https://github.com/anjunar/anjunar-blog-example/tree/3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6). All paths below are relative to its root. We will introduce the `BlogPost` entity in chapter 5; here we establish the infrastructure it needs.

## Understand the path to PostgreSQL

| Part | Responsibility |
| --- | --- |
| EntityManager | Holds the persistence context used by one request and executes its queries. |
| Hibernate | Implements Jakarta Persistence and translates entity operations into SQL. |
| Agroal | Supplies pooled JDBC connections. |
| PostgreSQL JDBC driver | Communicates with PostgreSQL. |
| Narayana | Coordinates commit and rollback through Jakarta Transactions, also called JTA. |
| PostgreSQL | Stores the data and enforces database constraints. |

The pool and EntityManagerFactory live for the application. Each database-backed request gets its own EntityManager and transaction. Sharing one EntityManager across concurrent requests would mix their persistence contexts.

We will use the same libraries as the reference stack, with one database and no tenant context.

## Start a local database

Create `compose.yaml`:

```yaml
services:
  postgres:
    image: postgres:18.6
    environment:
      POSTGRES_DB: anjunar_blog
      POSTGRES_USER: blog
      POSTGRES_PASSWORD: ${BLOG_DB_PASSWORD:?Set BLOG_DB_PASSWORD}
    ports:
      - "127.0.0.1:5433:5432"
    volumes:
      - blog-data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U blog -d anjunar_blog"]
      interval: 2s
      timeout: 5s
      retries: 20

volumes:
  blog-data:
```

With Docker and Compose installed, set a development password and start PostgreSQL. In PowerShell:

```powershell
$env:BLOG_DB_PASSWORD = "local-blog-password"
docker compose up -d --wait
```

In Bash:

```bash
export BLOG_DB_PASSWORD=local-blog-password
docker compose up -d --wait
```

Keep using that terminal for sbt so the application inherits the password. The database listens on port 5433 of the loopback interface.

PostgreSQL 18 images use `/var/lib/postgresql` as the volume mount. The image's initialization variables apply when the data directory is empty; changing the password variable later does not change an existing database user's password. See the [official image documentation](https://hub.docker.com/_/postgres).

If PostgreSQL is already installed locally, you can instead create a dedicated tutorial role and database from an administrator's `psql` session:

```sql
CREATE ROLE blog LOGIN PASSWORD 'local-blog-password';
CREATE DATABASE anjunar_blog OWNER blog;
```

Set `BLOG_DB_URL` to that installation's port, for example `jdbc:postgresql://127.0.0.1:5432/anjunar_blog`. Use this separate development database for the tests below.

## Add the database libraries

Add these entries to the existing `libraryDependencies` sequence in `build.sbt`:

```scala
"org.hibernate.orm" % "hibernate-core" % "7.4.10.Final",
"org.postgresql" % "postgresql" % "42.7.13",
"org.jboss.narayana.jta" % "narayana-jta" % "7.3.4.Final",
"io.agroal" % "agroal-pool" % "3.2.1",
"io.agroal" % "agroal-narayana" % "3.2.1",
```

Hibernate supplies the Jakarta Persistence API transitively. Agroal's Narayana module connects the pool to the transaction manager.

Keep the existing HTTP, CDI, and test dependencies. Add `ObjectStore/` and `PutObjectStoreDirHere/` to `.gitignore` for Narayana's local runtime files.

## Read connection settings from the environment

Create `application/backend/src/main/scala/com/anjunar/blog/DatabaseConfig.scala`:

```scala
package com.anjunar.blog

final class DatabaseConfig(val url: String, val user: String, val password: String)

object DatabaseConfig {
  def load(environment: Map[String, String] = sys.env): DatabaseConfig = {
    val url = environment.getOrElse("BLOG_DB_URL", "jdbc:postgresql://127.0.0.1:5433/anjunar_blog")
    val user = environment.getOrElse("BLOG_DB_USER", "blog")
    val password = environment.getOrElse("BLOG_DB_PASSWORD",
      throw new IllegalArgumentException("Set BLOG_DB_PASSWORD before accessing the database"))
    require(url.startsWith("jdbc:postgresql:"), "BLOG_DB_URL must be a PostgreSQL JDBC URL")
    require(user.nonEmpty && password.nonEmpty, "Database user and password must not be empty")
    new DatabaseConfig(url, user, password)
  }
}
```

The defaults match our Compose service. The password has no default and is supplied through `BLOG_DB_PASSWORD`. The application reads these variables directly; it does not load a `.env` file.

## Create the pool and EntityManagerFactory

Create `application/backend/src/main/scala/com/anjunar/blog/Persistence.scala`:

```scala
package com.anjunar.blog

import com.arjuna.ats.internal.jta.transaction.arjunacore.TransactionSynchronizationRegistryImple
import io.agroal.api.AgroalDataSource
import io.agroal.api.configuration.supplier.{AgroalConnectionFactoryConfigurationSupplier, AgroalConnectionPoolConfigurationSupplier, AgroalDataSourceConfigurationSupplier}
import io.agroal.api.security.{NamePrincipal, SimplePassword}
import io.agroal.narayana.NarayanaTransactionIntegration
import jakarta.annotation.{PostConstruct, PreDestroy}
import jakarta.enterprise.context.{ApplicationScoped, RequestScoped}
import jakarta.enterprise.inject.Produces
import jakarta.persistence.{EntityManager, EntityManagerFactory}
import org.hibernate.boot.MetadataSources
import org.hibernate.boot.registry.{StandardServiceRegistry, StandardServiceRegistryBuilder}
import org.postgresql.xa.PGXADataSource

import java.time.Duration
import scala.util.control.NonFatal

@ApplicationScoped
class Persistence {
  private var pool: AgroalDataSource = null
  private var factory: EntityManagerFactory = null

  @PostConstruct
  def initialize(): Unit = {
    val config = DatabaseConfig.load()
    val connection = new AgroalConnectionFactoryConfigurationSupplier()
      .connectionProviderClass(classOf[PGXADataSource])
      .jdbcUrl(config.url)
      .principal(new NamePrincipal(config.user))
      .credential(new SimplePassword(config.password))
      .loginTimeout(Duration.ofSeconds(5))
    val pooling = new AgroalConnectionPoolConfigurationSupplier()
      .maxSize(8)
      .acquisitionTimeout(Duration.ofSeconds(5))
      .connectionFactoryConfiguration(connection)
      .transactionIntegration(new NarayanaTransactionIntegration(
        com.arjuna.ats.jta.TransactionManager.transactionManager(),
        new TransactionSynchronizationRegistryImple()))
    pool = AgroalDataSource.from(new AgroalDataSourceConfigurationSupplier()
      .connectionPoolConfiguration(pooling))

    var registry: StandardServiceRegistry = null
    try {
      registry = new StandardServiceRegistryBuilder()
        .applySetting("jakarta.persistence.jtaDataSource", pool)
        .applySetting("hibernate.transaction.coordinator_class", "jta")
        .applySetting("hibernate.transaction.jta.platform",
          "org.hibernate.engine.transaction.jta.platform.internal.JBossStandAloneJtaPlatform")
        .applySetting("hibernate.hbm2ddl.auto", "none")
        .build()
      // The first domain entity arrives in chapter 5.
      factory = new MetadataSources(registry).buildMetadata().buildSessionFactory()
    } catch {
      case NonFatal(error) =>
        try {
          if (registry != null) StandardServiceRegistryBuilder.destroy(registry)
        } catch { case NonFatal(cleanup) => error.addSuppressed(cleanup) }
        try pool.close()
        catch { case NonFatal(cleanup) => error.addSuppressed(cleanup) }
        throw error
    }
  }

  def openEntityManager(): EntityManager = factory.createEntityManager()

  @Produces
  @RequestScoped
  def entityManager(transaction: RequestTransaction): EntityManager =
    transaction.entityManager

  @PreDestroy
  def close(): Unit =
    try {
      if (factory != null) factory.close()
    } finally {
      if (pool != null) pool.close()
    }
}
```

Three connections between the libraries matter here:

- `PGXADataSource` provides transaction-aware database connections to Agroal.
- `NarayanaTransactionIntegration` enlists borrowed connections in the active JTA transaction.
- Hibernate receives that same pool, selects the `jta` coordinator, and uses the standalone Narayana platform.

The [Hibernate transaction documentation](https://docs.hibernate.org/orm/7.4/userguide/html_single/#transactions) describes the coordinator and JTA platform settings. Agroal's [configuration documentation](https://agroal.github.io/docs.html) covers pool size, connection acquisition, and transaction integration.

We cap this development pool at eight connections and give connection acquisition a five-second timeout. Hibernate schema generation is disabled. The metadata currently contains no entities; schema creation and migration arrive with the domain model.

The producer method exposes the current request's EntityManager for `@Inject`. It does not create a second one. `RequestTransaction`, defined next, owns its creation and closure.

`Persistence` initializes when it is first used. At shutdown it closes the factory before the pool. If factory initialization fails, it releases the registry and pool while preserving the original failure.

## Own one transaction per request

Create `application/backend/src/main/scala/com/anjunar/blog/RequestTransaction.scala`:

```scala
package com.anjunar.blog

import jakarta.annotation.PreDestroy
import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.{EntityManager, FlushModeType}
import jakarta.transaction.Status

import scala.compiletime.uninitialized
import scala.util.control.NonFatal

@RequestScoped
class RequestTransaction {
  @Inject
  var persistence: Persistence = uninitialized

  private var manager: EntityManager = null
  private var started = false
  private var readOnly = false
  private def transaction = com.arjuna.ats.jta.UserTransaction.userTransaction()

  def active: Boolean = started

  def entityManager: EntityManager = {
    require(started && manager != null, "No database transaction is active for this request")
    manager
  }

  def begin(readRequest: Boolean): Unit = {
    require(transaction.getStatus == Status.STATUS_NO_TRANSACTION, "A transaction is already active")
    transaction.setTransactionTimeout(30)
    transaction.begin()
    started = true
    readOnly = readRequest
    try {
      manager = persistence.openEntityManager()
      manager.joinTransaction()
      if (readOnly) manager.setFlushMode(FlushModeType.COMMIT)
    } catch {
      case NonFatal(error) =>
        try finish(false)
        catch { case NonFatal(cleanup) => error.addSuppressed(cleanup) }
        throw error
    }
  }

  def flush(successful: Boolean): Unit =
    if (started && successful && !readOnly) entityManager.flush()

  def finish(successful: Boolean): Unit =
    if (started) {
      started = false
      try {
        if (successful && !readOnly) transaction.commit()
        else if (transaction.getStatus != Status.STATUS_NO_TRANSACTION) transaction.rollback()
      } catch {
        case NonFatal(error) =>
          try {
            if (transaction.getStatus != Status.STATUS_NO_TRANSACTION) transaction.rollback()
          } catch { case NonFatal(cleanup) => error.addSuppressed(cleanup) }
          throw error
      } finally {
        if (manager != null) {
          try manager.close()
          finally manager = null
        }
      }
    }

  @PreDestroy
  def abortUnfinishedRequest(): Unit = finish(false)
}
```

`begin` starts a Narayana transaction, creates the EntityManager, and joins it to that transaction. For GET and HEAD, `FlushModeType.COMMIT` prevents ordinary automatic flushing before queries; the final rollback discards any writes, including native SQL executed by a test.

Our completion policy is:

| Request result | Completion |
| --- | --- |
| GET or HEAD | Rollback |
| Other method with status below 400 | Commit |
| Error status or failed serialization | Rollback |

`flush` sends pending entity changes to the database. It does **not** commit them. A later failure can still roll back those statements. Some constraints are checked only at commit, which is why a successful flush cannot establish that a write succeeded.

The EntityManager closes when the transaction finishes. The `@PreDestroy` callback is a fallback: if an earlier failure leaves a request unfinished, destroying its CDI context rolls it back.

This chapter uses Narayana's programmatic transaction API explicitly. It does not enable automatic `@Transactional` interception or Weld's transactional observers. Weld may still log that its own transactional services are unavailable during startup.

## Keep the transaction open through serialization

Returning from a resource method does not finish writing an HTTP response. A message-body writer still needs to turn its result into bytes. Later in the series, that writer may traverse entity relationships and need the open persistence context.

Create `application/backend/src/main/scala/com/anjunar/blog/TransactionBoundary.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.ws.rs.container.{ContainerRequestContext, ContainerRequestFilter, ContainerResponseContext, ContainerResponseFilter, ResourceInfo}
import jakarta.ws.rs.core.Context
import jakarta.ws.rs.ext.{Provider, WriterInterceptor, WriterInterceptorContext}

import java.io.ByteArrayOutputStream
import scala.compiletime.uninitialized
import scala.util.control.NonFatal

@Provider
@RequestScoped
class TransactionBoundary
    extends ContainerRequestFilter
    with ContainerResponseFilter
    with WriterInterceptor {
  @Inject
  var transaction: RequestTransaction = uninitialized

  @Context
  var resource: ResourceInfo = uninitialized

  private var successful = false

  override def filter(request: ContainerRequestContext): Unit =
    if (resource.getResourceClass != classOf[HealthResource]) {
      transaction.begin(request.getMethod == "GET" || request.getMethod == "HEAD")
    }

  override def filter(request: ContainerRequestContext, response: ContainerResponseContext): Unit = {
    successful = response.getStatus < 400
    try {
      transaction.flush(successful)
      if (!response.hasEntity || request.getMethod == "HEAD") transaction.finish(successful)
    } catch {
      case NonFatal(error) =>
        abort(error)
        throw error
    }
  }

  override def aroundWriteTo(context: WriterInterceptorContext): Unit =
    if (!transaction.active) context.proceed()
    else {
      // Finish serialization and the transaction before sending a success body.
      val output = context.getOutputStream
      val buffer = new ByteArrayOutputStream()
      context.setOutputStream(buffer)
      try {
        context.proceed()
        transaction.finish(successful)
      } catch {
        case NonFatal(error) =>
          abort(error)
          throw error
      } finally {
        context.setOutputStream(output)
      }
      buffer.writeTo(output)
    }

  private def abort(error: Throwable): Unit =
    try transaction.finish(false)
    catch { case NonFatal(cleanup) => error.addSuppressed(cleanup) }
}
```

Chapter 3's extension discovers this `@Provider` automatically.

The request filter starts the transaction. It exempts `HealthResource`, our liveness endpoint, so checking whether the HTTP server is alive does not require PostgreSQL.

The response filter knows the status and flushes successful writes. For HEAD and responses without an entity, it also completes the transaction: there will be no response body to intercept.

For other responses, the [writer interceptor](https://jakarta.ee/specifications/restful-ws/4.0/apidocs/jakarta.ws.rs/jakarta/ws/rs/ext/writerinterceptor) wraps serialization. Its order is deliberate:

1. Serialize into a buffer while the EntityManager remains open.
2. Commit a successful write, or roll back a read or error response.
3. Close the EntityManager.
4. Copy the buffered bytes to the HTTP output.

If serialization or commit fails, the success body has not been sent. RESTEasy can produce an error response.

The current endpoints return small, synchronous responses, so an in-memory buffer is sufficient. Streaming downloads and asynchronous endpoints will need a different boundary. Also, committing a transaction cannot guarantee network delivery afterward; a disconnected client may miss a response for a write that committed.

## Query the database from a resource

Create `application/backend/src/main/scala/com/anjunar/blog/DatabaseHealthResource.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.ws.rs.{GET, Path, Produces}

import scala.compiletime.uninitialized

@Path("/health/ready")
@RequestScoped
class DatabaseHealthResource {
  @Inject
  var entityManager: EntityManager = uninitialized

  @GET
  @Produces(Array("text/plain;charset=UTF-8"))
  def ready(): String = {
    entityManager.createNativeQuery("select 1", classOf[java.lang.Integer]).getSingleResult
    "UP\n"
  }
}
```

The resource receives its EntityManager through CDI and executes `select 1`. The request boundary handles transaction completion and cleanup. There is no manual commit in the resource.

Start the application:

```text
sbt --server "application-backend/run"
```

In another terminal:

```text
curl -i http://127.0.0.1:8080/service/health/live
curl -i http://127.0.0.1:8080/service/health/ready
```

Both should return HTTP 200 and `UP`. Use `curl.exe` in Windows PowerShell if needed, and adjust the HTTP port if you set `BLOG_PORT`.

The checks answer different questions. Liveness reaches an HTTP resource. Readiness also reaches PostgreSQL through Hibernate and the transaction-aware pool. A database failure currently produces HTTP 500 for readiness, while liveness continues to return 200.

Database initialization happens on the first database-backed request. The server's startup message confirms the HTTP deployment; the readiness request checks the database path.

## Verify what actually commits

The checkpoint adds [DatabaseIntegrationSpec](https://github.com/anjunar/anjunar-blog-example/blob/e96365e906bb14b212fe2b0b9e11664f6b5afd93/application/backend/src/test/scala/com/anjunar/blog/DatabaseIntegrationSpec.scala) and [TransactionProbeResource](https://github.com/anjunar/anjunar-blog-example/blob/e96365e906bb14b212fe2b0b9e11664f6b5afd93/application/backend/src/test/scala/com/anjunar/blog/TransactionProbeResource.scala) under `application/backend/src/test/scala/com/anjunar/blog`. Add both when reproducing the chapter by hand.

The suite creates a table with a unique generated name and drops it afterward. Test resources execute real inserts. Assertions use separate JDBC connections, so they observe committed database state rather than the request's uncommitted changes.

The checks cover:

- A successful write and a successful response with no body.
- Rollback for an error status and for an SQL exception after an insert.
- Database access during body serialization.
- Rollback when the writer fails after producing some buffered bytes.
- A foreign-key constraint that fails only at commit.
- A transaction explicitly marked rollback-only.
- Writes attempted during GET and HEAD.
- The readiness endpoint.

The deferred foreign key is useful precisely because PostgreSQL can postpone that check until transaction completion. See [PostgreSQL's constraint timing rules](https://www.postgresql.org/docs/18/sql-set-constraints.html). That test checks both outcomes: no inserted row and no success response.

With PostgreSQL running and the environment variables set, execute:

```text
sbt --server "application-backend/testFull"
```

Expect **12 successful tests**, including the two tests from the previous chapters. The intentional failure cases produce error logs; the final test summary establishes whether their expected behavior passed.

The probe resources are test-only. A normal application run does not expose their write endpoints.

## Use the chapter checkpoint

The [chapter 4 source](https://github.com/anjunar/anjunar-blog-example/tree/e96365e906bb14b212fe2b0b9e11664f6b5afd93) contains the complete implementation and tests. [Pull request #4](https://github.com/anjunar/anjunar-blog-example/pull/4) shows the changes from chapter 3. To use it in a separate directory:

```text
git clone https://github.com/anjunar/anjunar-blog-example.git
cd anjunar-blog-example
git switch --detach e96365e906bb14b212fe2b0b9e11664f6b5afd93
```

Then configure and start PostgreSQL as described above, and run `testFull`. Create a branch with `git switch -c my-blog` to continue your own implementation.

Stop the application with Ctrl+C. For the Compose database, `docker compose down` stops the service while preserving its data volume.

The examples and all 12 tests were verified against PostgreSQL 18.6 using an isolated native installation. The Compose file supplies the equivalent local database setup; Docker was unavailable on the authoring machine.

Next we will add `BlogPost`: its identity, slug, publication status, version, and validation rules. The request and transaction infrastructure is now ready to persist it.
