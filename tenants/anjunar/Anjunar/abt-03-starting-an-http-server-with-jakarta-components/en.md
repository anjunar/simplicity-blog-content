The server from chapter 2 can answer a request, but its resource list is hard-coded. Every new endpoint would require another entry in `ServerApplication`.

In this chapter, we will connect that list to CDI discovery. Adding a resource with `@Path` and a CDI scope will make it available through RESTEasy. A new `GET /service/health/live` endpoint gives us a concrete result to check.

We will also follow startup, request handling, and shutdown through Undertow, RESTEasy, and Weld. This explains who creates our objects and when they are released.

## Start from the previous chapter

Use the working project from [chapter 2](https://github.com/anjunar/anjunar-blog-example/tree/0b9ea6f9447069bbe291e486fcdc8fb633d10656). Keep its JDK, sbt, Scala, and library versions. This chapter adds no dependencies.

All paths below are relative to the project root. The complete chapter checkpoint, including the new tests, is linked at the end.

## Give each component a clear responsibility

| Component | Responsibility in our application |
| --- | --- |
| Undertow | Opens the HTTP listener and runs the servlet deployment. |
| RESTEasy | Matches requests to Jakarta REST resources and produces HTTP responses. |
| Weld | Discovers CDI beans, resolves dependencies, and manages their lifetimes. |

Jakarta REST and CDI define the APIs used in our code. RESTEasy and Weld implement those APIs. The `resteasy-undertow-cdi` dependency provides the integration that starts Weld alongside the embedded server. See the [RESTEasy bootstrap documentation](https://docs.resteasy.dev/7.0/userguide/).

Our `ApplicationMain.start` already creates that integrated server:

```scala
val server = new UndertowCdiEmbeddedServer()
server.getDeployment.setApplication(new ServerApplication())
val configuration = SeBootstrap.Configuration.builder()
  .host("127.0.0.1")
  .port(port)
  .rootPath("/")
  .build()
```

This is an excerpt from `application/backend/src/main/scala/com/anjunar/blog/ApplicationMain.scala`; keep the existing method and its error handling.

With our pinned RESTEasy version, startup proceeds through these stages:

1. Undertow opens the listener.
2. The CDI deployment starts Weld and processes the discovered beans.
3. RESTEasy reads the application's resource and provider classes and configures its CDI integration.
4. The servlet deployment connects the dispatcher and the Weld listener to incoming requests.

The early Undertow log message means the listener has started. Our own `Blog server: ...` message appears after `server.start(configuration)` returns successfully. A real request verifies the complete path.

## Collect REST components during CDI discovery

In chapter 2, `ServerApplication.getClasses` returned a set containing only `HelloResource`. We will replace that manually maintained set with classes collected during CDI bootstrap.

CDI provides a *portable extension*: a class that observes container lifecycle events. `ProcessManagedBean` supplies the annotated class of a discovered managed bean. Our extension keeps those marked with `@Path` or `@Provider`. These hooks are defined by [CDI's portable extension API](https://jakarta.ee/specifications/cdi/4.1/jakarta-cdi-spec-4.1.html#process_bean).

Create `application/backend/src/main/scala/com/anjunar/blog/RestComponentsExtension.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.event.Observes
import jakarta.enterprise.inject.spi.{Extension, ProcessManagedBean}
import jakarta.ws.rs.Path
import jakarta.ws.rs.ext.Provider

import java.util
import java.util.concurrent.ConcurrentHashMap

class RestComponentsExtension extends Extension {

  private val components = ConcurrentHashMap.newKeySet[Class[?]]()

  def collect(@Observes event: ProcessManagedBean[?]): Unit = {
    val beanType = event.getAnnotatedBeanClass
    if (beanType.isAnnotationPresent(classOf[Path]) ||
        beanType.isAnnotationPresent(classOf[Provider])) {
      components.add(beanType.getJavaClass)
    }
  }

  def classes: util.Set[Class[?]] = util.Set.copyOf(components)

}
```

`@Observes` marks the event parameter. `@Path` identifies resource classes; `@Provider` identifies REST providers, such as response filters. We collect class metadata here. RESTEasy and Weld will handle instances when they are needed.

The set belongs to this extension instance, so a server created by the next test starts with its own collection. `classes` returns an immutable copy.

Register the extension in this exact file:

`application/backend/src/main/resources/META-INF/services/jakarta.enterprise.inject.spi.Extension`

```text
com.anjunar.blog.RestComponentsExtension
```

The filename names the service interface; its content names our implementation. Keep the existing `META-INF/beans.xml` as well. It selects annotated bean discovery for the application.

Our convention is that every resource and provider declares its scope explicitly. The extension handles managed bean classes; it does not register arbitrary objects returned by producer methods.

## Let RESTEasy use the discovered classes

Replace `application/backend/src/main/scala/com/anjunar/blog/ServerApplication.scala` with:

```scala
package com.anjunar.blog

import jakarta.enterprise.inject.spi.CDI
import jakarta.ws.rs.ApplicationPath
import jakarta.ws.rs.core.Application

import java.util

@ApplicationPath("/service")
class ServerApplication extends Application {

  override def getClasses: util.Set[Class[?]] =
    CDI.current().getBeanManager
      .getExtension(classOf[RestComponentsExtension])
      .classes

}
```

`BeanManager.getExtension` retrieves the extension instance created by the running container. Its collection has already been populated when RESTEasy asks for these classes.

The explicit lookup matters here: `ApplicationMain` constructs this application with `new ServerApplication()`. We therefore obtain the container at the integration boundary. Our resources continue to use ordinary `@Inject` dependencies.

Restart the application after adding components; discovery happens during startup.

## Add the liveness endpoint

Create `application/backend/src/main/scala/com/anjunar/blog/HealthResource.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.RequestScoped
import jakarta.ws.rs.{GET, Path, Produces}

@Path("/health/live")
@RequestScoped
class HealthResource {

  @GET
  @Produces(Array("text/plain;charset=UTF-8"))
  def live(): String = "UP\n"

}
```

There is no class-list entry to add. During startup, Weld discovers this scoped bean and our extension exposes its class to RESTEasy.

The URL comes from three pieces:

| Setting | Value |
| --- | --- |
| Server root path | `/` |
| `ServerApplication` application path | `/service` |
| `HealthResource` resource path | `/health/live` |
| Result | `http://127.0.0.1:8080/service/health/live` |

Start the application from the project root:

```text
sbt --server "application-backend/run"
```

In another terminal:

```text
curl -i http://127.0.0.1:8080/service/health/live
```

Use `curl.exe` in Windows PowerShell if `curl` is an alias. Expect HTTP 200, a `text/plain` content type, and:

```text
UP
```

The existing greeting at `/service/hello` should still work, and `/service/missing` should return 404. If you set `BLOG_PORT`, use that port in each URL.

This liveness response shows that the server can dispatch a request to a resource. We have no database yet, so it makes no claim about database connectivity or readiness to serve stored posts.

## Follow a request through the scopes

Consider `GET /service/hello`. Undertow delivers the request to the RESTEasy dispatcher. The servlet integration activates the CDI request context. RESTEasy selects `HelloResource`, and its CDI integration obtains the resource through Weld.

Our resource uses `@RequestScoped`. The `GreetingService` injected into it uses `@ApplicationScoped`:

| Scope | What we expect in this application |
| --- | --- |
| Request | One contextual instance of a bean within a request; another request gets another instance. |
| Application | One contextual instance of a bean shared across requests in this container. |

CDI can inject a proxy that resolves the instance for the active context. The injected reference therefore does not necessarily identify the underlying object directly. The [CDI client proxy rules](https://jakarta.ee/specifications/cdi/4.1/jakarta-cdi-spec-4.1.html#client_proxies) describe this indirection.

For our code, the practical consequence is simple: keep request-specific mutable state out of an application-scoped service. Several requests can call it concurrently. `GreetingService` currently returns a fixed string and has no such state.

Constructing `new HelloResource()` ourselves would bypass the container lifecycle that supplies its dependency. Let the integration obtain resource instances.

## Verify discovery and lifetimes over HTTP

Extend the existing `ServerIntegrationSpec` with these assertions inside its `try` block, immediately before the unknown-resource check:

```scala
val health = get("/service/health/live")
assert(health.statusCode() == 200)
assert(health.body() == "UP\n")
```

The checkpoint also contains a second test, [ServerLifecycleSpec](https://github.com/anjunar/anjunar-blog-example/blob/3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6/application/backend/src/test/scala/com/anjunar/blog/ServerLifecycleSpec.scala), and its [ScopeProbe fixtures](https://github.com/anjunar/anjunar-blog-example/blob/3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6/application/backend/src/test/scala/com/anjunar/blog/ScopeProbe.scala). Add both files under `application/backend/src/test/scala/com/anjunar/blog` to reproduce that test in your own project. Copy the existing `beans.xml` to `application/backend/src/test/resources/META-INF/beans.xml` so the test beans are discoverable too.

The fixtures give each request-scoped and application-scoped probe a UUID. A test resource returns both IDs; a test response filter adds the request ID to a header. Their `@PreDestroy` methods record which instances have been released.

Two HTTP requests then check that:

- The resource's request ID matches the filter's header for each response.
- The request ID changes between requests.
- The application ID stays the same.
- Both request instances are destroyed after their requests complete.
- The application instance is destroyed when the test stops the server.

Because the resource and filter are registered only through our extension, these checks also exercise discovery of both `@Path` and `@Provider` classes. The test waits briefly for request destruction: receiving a response and completing server-side cleanup can happen at slightly different times.

Run:

```text
sbt --server "application-backend/testFull"
```

With the complete checkpoint, the result should report **two successful tests**. The probe classes live under `src/test`. A normal application run excludes them: `/service/_test/scopes` returns 404.

## Stop the server through its owner

`ApplicationMain` already owns shutdown. Its hook calls the integrated server's `stop()`, guarded by an `AtomicBoolean` so the hook and the `finally` block cannot both perform that call.

The embedded server tears down Weld, the servlet deployment, and Undertow. Our lifecycle test verifies the application bean's destruction callback. We also retain the startup error handler, which calls `stop()` if starting the server fails.

Press Ctrl+C in the server terminal. On Windows, the sbt launcher may ask you to confirm termination of the batch job. The listener should then close.

This gives our development process explicit startup and cleanup. Coordinating in-flight requests during a production deployment belongs to the deployment chapter.

## The chapter checkpoint

The [complete source for chapter 3](https://github.com/anjunar/anjunar-blog-example/tree/3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6) includes every file used above. [Pull request #3](https://github.com/anjunar/anjunar-blog-example/pull/3) shows the changes from the previous chapter. To run it in a separate directory:

```text
git clone https://github.com/anjunar/anjunar-blog-example.git
cd anjunar-blog-example
git switch --detach 3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6
sbt --server "application-backend/testFull"
```

Choose a parent directory where `anjunar-blog-example` does not already exist. The detached revision keeps the examples stable as the series grows. Use `git switch -c my-blog` to continue your own implementation from this point.

The next chapter adds PostgreSQL, Hibernate, a connection pool, and transactions. We will extend the request lifecycle we established here with database work, commit, and rollback.
