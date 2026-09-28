By the end of this chapter, a request to `http://127.0.0.1:8080/service/hello` will return:

```text
Welcome to Anjunar Blog Tutorial!
```

We will start with an empty directory and create every file needed to build, run, and test that response. This gives us a small working application to extend in later chapters.

## Prepare the tools

This project uses these versions:

| Tool | Version | Where it is selected |
| --- | --- | --- |
| JDK | 25 | Your local Java installation |
| sbt | 2.0.9 | `project/build.properties` |
| Scala | 3.9.0 | `build.sbt` |

Install a JDK 25 distribution. We use [GraalVM](https://www.graalvm.org/downloads/) for the series because we will later run JavaScript inside the JVM. Install sbt using the [official installation instructions](https://www.scala-sbt.org/download/).

Set `JAVA_HOME` to the JDK directory and make its `bin` directory available on `PATH`. Open a terminal and check:

```text
java -version
javac -version
```

Both commands should report version 25. When sbt starts, also check the Java version in its startup banner: it needs to use the same JDK.

sbt downloads the Scala compiler and the dependencies declared by the project. The initial build needs access to Maven Central. PostgreSQL and Node.js enter the project in later chapters.

## Create the directory layout

Create an empty directory called `anjunar-blog-example`. All file paths and commands below are relative to that directory.

The finished layout for this chapter is:

```text
anjunar-blog-example/
  .gitattributes
  .gitignore
  .jvmopts
  build.sbt
  project/
    build.properties
  application/
    backend/
      src/
        main/
          resources/
            META-INF/
              beans.xml
          scala/
            com/anjunar/blog/
              ApplicationMain.scala
              ServerApplication.scala
              GreetingService.scala
              HelloResource.scala
        test/
          scala/
            com/anjunar/blog/
              ServerIntegrationSpec.scala
```

There is one backend module. It lives under `application/backend`, leaving room for the frontend and feature modules we will introduce later.

## Define the build

First, pin sbt itself:

**File:** `project/build.properties`

```properties
sbt.version=2.0.9
```

Give the local sbt process an explicit memory budget and use UTF-8:

**File:** `.jvmopts`

```text
-Xms128m
-Xmx1536m
-Xss2m
-Dfile.encoding=UTF-8
```

These options configure the build JVM. They are not a production server configuration.

Now define the module and its dependencies:

**File:** `build.sbt`

```scala
ThisBuild / organization := "com.anjunar"
ThisBuild / version := "0.1.0-SNAPSHOT"
ThisBuild / scalaVersion := "3.9.0"

lazy val backend = Project("application-backend", file("application/backend"))
  .settings(
    libraryDependencies ++= Seq(
      "jakarta.ws.rs" % "jakarta.ws.rs-api" % "4.0.0",
      "jakarta.enterprise" % "jakarta.enterprise.cdi-api" % "4.1.0",
      "org.jboss.resteasy" % "resteasy-undertow-cdi" % "7.0.5.Final",
      "io.undertow" % "undertow-core" % "2.4.3.Final",
      "io.undertow.ee" % "undertow-servlet" % "2.0.2.Final",
      "org.jboss.weld.servlet" % "weld-servlet-core" % "6.0.4.Final",
      "org.scalatest" %% "scalatest" % "3.2.20" % Test
    ),
    // Undertow 2.4 uses the separately published Servlet 6.1 integration.
    excludeDependencies += ExclusionRule("io.undertow", "undertow-servlet"),
    Compile / run / mainClass := Some("com.anjunar.blog.ApplicationMain"),
    Compile / run / fork := true,
    Test / fork := true,
    Test / parallelExecution := false
  )

lazy val root = Project("anjunar-blog-tutorial", file("."))
  .aggregate(backend)
  .settings(publish / skip := true)
```

`ThisBuild` applies the organization, version, and Scala version across the build. The backend's project ID is `application-backend`; that is the name we will use in commands.

The root project aggregates backend tasks. Aggregation lets a task run across projects; `dependsOn`, which we will use when adding modules, establishes a code dependency between them.

The Jakarta dependencies provide the API types. RESTEasy handles REST requests, Undertow hosts the server, and Weld supplies CDI dependency injection. ScalaTest is available only in the test configuration.

The Servlet dependency needs attention: this combination uses `io.undertow.ee:undertow-servlet`. The exclusion removes the older `io.undertow:undertow-servlet` artifact that can arrive transitively. Keep the exclusion and the explicitly selected replacement together.

`%%` selects a library artifact for our Scala binary version. The Java libraries use `%`.

The application and tests run in separate JVM processes. This keeps the embedded server's class loading and lifecycle separate from sbt.

## Add the entry point

The application needs an object with a `main` method:

**File:** `application/backend/src/main/scala/com/anjunar/blog/ApplicationMain.scala`

```scala
package com.anjunar.blog

import dev.resteasy.embedded.server.UndertowCdiEmbeddedServer
import jakarta.ws.rs.SeBootstrap

import java.util.concurrent.CountDownLatch
import java.util.concurrent.atomic.AtomicBoolean
import scala.util.control.NonFatal

object ApplicationMain {

  def main(args: Array[String]): Unit = {
    val port = sys.env.getOrElse("BLOG_PORT", "8080").toInt
    val server = start(port)
    val stopped = new AtomicBoolean(false)

    def stop(): Unit =
      if (stopped.compareAndSet(false, true)) server.stop()

    Runtime.getRuntime.addShutdownHook(new Thread(() => stop(), "blog-shutdown"))
    println(s"Blog server: http://127.0.0.1:$port/service/hello")

    try new CountDownLatch(1).await()
    finally stop()
  }

  def start(port: Int): UndertowCdiEmbeddedServer = {
    require(port >= 1 && port <= 65535, "BLOG_PORT must be between 1 and 65535")
    val server = new UndertowCdiEmbeddedServer()
    server.getDeployment.setApplication(new ServerApplication())
    val configuration = SeBootstrap.Configuration.builder()
      .host("127.0.0.1")
      .port(port)
      .rootPath("/")
      .build()

    try {
      server.start(configuration)
      server
    } catch {
      case NonFatal(error) =>
        server.stop()
        throw error
    }
  }
}
```

`start` creates the embedded server, registers our REST application, and binds it to loopback. `main` selects the port, installs the shutdown hook, and keeps the process running.

Separating startup from `main` lets the integration test start and stop the same server without waiting indefinitely. The atomic flag makes the shutdown action run at most once when the hook and the `finally` block both attempt it.

The default port is 8080. We can override it with `BLOG_PORT`.

## Register a REST resource

The REST application defines the common URL prefix and the resource classes:

**File:** `application/backend/src/main/scala/com/anjunar/blog/ServerApplication.scala`

```scala
package com.anjunar.blog

import jakarta.ws.rs.ApplicationPath
import jakarta.ws.rs.core.Application

import java.util

@ApplicationPath("/service")
class ServerApplication extends Application {

  override def getClasses: util.Set[Class[?]] =
    util.Set.of[Class[?]](classOf[HelloResource])

}
```

For this first module, the resource list is explicit. The prefix `/service` applies to the endpoints registered here.

The resource will obtain its response from a small service:

**File:** `application/backend/src/main/scala/com/anjunar/blog/GreetingService.scala`

```scala
package com.anjunar.blog

import jakarta.enterprise.context.ApplicationScoped

@ApplicationScoped
class GreetingService {

  def message: String = "Welcome to Anjunar Blog Tutorial!\n"

}
```

`@ApplicationScoped` makes this a CDI bean whose contextual instance is shared across the application.

Now add the endpoint:

**File:** `application/backend/src/main/scala/com/anjunar/blog/HelloResource.scala`

```scala
package com.anjunar.blog

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.ws.rs.{GET, Path, Produces}

import scala.compiletime.uninitialized

@Path("/hello")
@RequestScoped
class HelloResource {

  @Inject
  var greeting: GreetingService = uninitialized

  @GET
  @Produces(Array("text/plain;charset=UTF-8"))
  def hello(): String = greeting.message

}
```

The annotations describe an HTTP GET endpoint returning UTF-8 plain text. The full path combines `/service` from the application with `/hello` from the resource.

Weld injects `GreetingService` before the request method uses it. The greeting is deliberately simple: a successful HTTP response will establish that request routing and dependency injection are both working.

Finally, enable CDI bean discovery:

**File:** `application/backend/src/main/resources/META-INF/beans.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       version="4.0"
       bean-discovery-mode="annotated">
</beans>
```

With `bean-discovery-mode="annotated"`, CDI discovers beans with a bean-defining annotation, such as the scopes used on our service and resource.

## Keep generated files out of Git

The repository should contain the sources and build definition. Add these ignore rules:

**File:** `.gitignore`

```gitignore
target/
.bsp/
.metals/
.idea/
.vscode/
*.iml
.env
.env.*
!.env.example
application.properties
*.log
```

The `target/` rule also covers generated output inside subdirectories. Local application configuration and environment files are excluded so that future credentials stay out of commits.

Use consistent line endings:

**File:** `.gitattributes`

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
```

You can now initialize a Git repository with `git init` if you are building the example by hand.

## Test the complete request

Compiling the classes will catch type errors. To verify the connection between Undertow, RESTEasy, and Weld, we also need to make a real request.

**File:** `application/backend/src/test/scala/com/anjunar/blog/ServerIntegrationSpec.scala`

```scala
package com.anjunar.blog

import org.scalatest.funsuite.AnyFunSuite

import java.net.{InetAddress, ServerSocket, URI}
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import java.time.Duration

class ServerIntegrationSpec extends AnyFunSuite {

  test("Undertow serves the REST resource with its CDI dependency and returns 404 for unknown resources") {
    // Let the OS choose a currently available loopback port for this test.
    val reservation = new ServerSocket(0, 1, InetAddress.getByName("127.0.0.1"))
    val port = try reservation.getLocalPort finally reservation.close()

    val server = ApplicationMain.start(port)
    val client = HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(5)).build()
    try {
      def get(path: String): HttpResponse[String] = {
        val request = HttpRequest.newBuilder(URI.create(s"http://127.0.0.1:$port$path"))
          .timeout(Duration.ofSeconds(10))
          .GET()
          .build()
        client.send(request, HttpResponse.BodyHandlers.ofString())
      }

      val response = get("/service/hello")
      assert(response.statusCode() == 200)
      assert(response.headers().firstValue("Content-Type").orElse("").startsWith("text/plain"))
      assert(response.body() == "Welcome to Anjunar Blog Tutorial!\n")
      assert(get("/service/missing").statusCode() == 404)
    } finally {
      client.close()
      server.stop()
    }
  }

}
```

The test asks the operating system for an available loopback port and closes that temporary socket before starting the server. There is a brief interval in which another process could claim the port; if that happens, rerun the test.

The assertions check the response status, media type, and body. Returning the greeting also exercises CDI injection. The second request checks that an unknown resource produces HTTP 404. The `finally` block closes the HTTP client and server.

Run this from the project root:

```text
sbt --server "application-backend/testFull"
```

Here, `--server` runs sbt in the foreground. The first invocation may take longer while dependencies are downloaded.

The result should report one successful test. We use `testFull` because sbt 2's `test` task is incremental and may skip tests that already passed. The [sbt testing documentation](https://www.scala-sbt.org/2.x/docs/en/reference/sbt-test.html) describes the distinction. If `testFull` reports zero tests, check the test directory and the project ID in the command.

## Start the application

In the same project directory, run:

```text
sbt --server "application-backend/run"
```

After startup, the application prints:

```text
Blog server: http://127.0.0.1:8080/service/hello
```

Keep that terminal open. In another terminal, send the request:

```text
curl -i http://127.0.0.1:8080/service/hello
```

In Windows PowerShell, use `curl.exe` if `curl` is an alias for a PowerShell command.

Expect HTTP 200, a `text/plain` content type, and the greeting shown at the beginning of this article. Requesting `/service/missing` should return HTTP 404.

If port 8080 is already occupied, choose another port before starting. In PowerShell:

```powershell
$env:BLOG_PORT = "8081"
sbt --server "application-backend/run"
```

In Bash:

```bash
BLOG_PORT=8081 sbt --server "application-backend/run"
```

Use the selected port in the request URL. Press Ctrl+C in the server terminal when you are finished. On Windows, the sbt batch launcher may also ask you to confirm that you want to terminate the batch job.

## The chapter checkpoint

The complete source is recorded in the [article 2 checkpoint](https://github.com/anjunar/anjunar-blog-example/tree/0b9ea6f9447069bbe291e486fcdc8fb633d10656). To use that exact version:

```text
git clone https://github.com/anjunar/anjunar-blog-example.git
cd anjunar-blog-example
git switch --detach 0b9ea6f9447069bbe291e486fcdc8fb633d10656
sbt --server "application-backend/testFull"
```

Use a separate directory for this clone if you have already created the files by hand. The checkout is detached so later changes on `main` cannot silently change the example. Create a branch with `git switch -c my-blog` when you want to build on it.

We now have a reproducible build and an HTTP endpoint backed by a CDI bean. The next chapter follows that request through the server in more detail: how the components are discovered, how their lifetimes relate, and what happens during startup and shutdown.
