Am Ende dieses Kapitels liefert eine Anfrage an `http://127.0.0.1:8080/service/hello` folgende Antwort:

```text
Welcome to Anjunar Blog Tutorial!
```

Wir beginnen mit einem leeren Verzeichnis und erstellen alle Dateien, die wir zum Bauen, Starten und Testen dieser Antwort brauchen. So entsteht eine kleine lauffähige Anwendung, die wir in späteren Kapiteln erweitern.

## Vorbereitung der Werkzeuge

Dieses Projekt verwendet diese Versionen:

|Werkzeug|Version|Wo es ausgewählt ist|
| --- | --- | --- |
|JDK|25|Ihre lokale Java-Installation|
|sbt|2.0.9|`project/build.properties`|
|Scala|3.9.0|`build.sbt`|

Installieren Sie eine JDK-25-Distribution. Für diese Reihe verwenden wir [GraalVM](https://www.graalvm.org/downloads/), weil wir später JavaScript innerhalb der JVM ausführen. Installieren Sie sbt mithilfe der [offiziellen Installationsanleitung](https://www.scala-sbt.org/download/).

Setzen Sie `JAVA_HOME` auf das JDK-Verzeichnis und nehmen Sie dessen `bin`-Verzeichnis in `PATH` auf. Prüfen Sie in einem Terminal:

```text
java -version
javac -version
```

Beide Befehle sollten Version 25 melden. Wenn sbt startet, überprüfen Sie auch die Java-Version im Startbanner: Es muss das gleiche JDK verwenden.

sbt lädt den Scala-Compiler und die vom Projekt deklarierten Abhängigkeiten herunter. Der erste Build benötigt Zugang zu Maven Central. PostgreSQL und Node.js kommen in späteren Kapiteln hinzu.

## Erstellen des Verzeichnislayouts

Erstellen Sie ein leeres Verzeichnis namens `anjunar-blog-example`. Alle Dateipfade und Befehle unten sind relativ zu diesem Verzeichnis.

Das fertige Layout für dieses Kapitel ist:

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

Es gibt ein Backend-Modul unter `application/backend`. Die Struktur lässt Platz für das Frontend und die Feature-Module, die wir später ergänzen.

## Den Build definieren

Zuerst legen wir die sbt-Version fest:

**Datei:** `project/build.properties`

```properties
sbt.version=2.0.9
```

Geben Sie dem lokalen sbt-Prozess ein explizites Speicherbudget und verwenden Sie UTF-8:

**Datei:** `.jvmopts`

```text
-Xms128m
-Xmx1536m
-Xss2m
-Dfile.encoding=UTF-8
```

Diese Optionen konfigurieren die Build JVM. Sie sind keine Produktionsserverkonfiguration.

Definieren Sie nun das Modul und seine Abhängigkeiten:

**Datei:** `build.sbt`

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

`ThisBuild` wendet die Organisation, Version und Scala-Version über den Build an. Die Projekt-ID des Backends lautet `application-backend`; das ist der Name, den wir in Befehlen verwenden werden.

Das Root-Projekt aggregiert Backend-Aufgaben. Mit der Aggregation kann eine Aufgabe über Projekte hinweg ausgeführt werden; `dependsOn`, das wir beim Hinzufügen von Modulen verwenden, stellt eine Codeabhängigkeit zwischen ihnen her.

Die Jakarta-Abhängigkeiten stellen die API-Typen bereit. RESTEasy verarbeitet REST-Anfragen, Undertow betreibt den Server und Weld übernimmt die CDI-Abhängigkeitsinjektion. ScalaTest ist nur in der Testkonfiguration verfügbar.

Bei der Servlet-Abhängigkeit ist Folgendes wichtig: Diese Kombination verwendet `io.undertow.ee:undertow-servlet`. Der Ausschluss entfernt das ältere Artefakt `io.undertow:undertow-servlet`, das transitiv eingebunden werden könnte. Ausschluss und explizit gewählter Ersatz gehören zusammen.

`%%` wählt ein Bibliotheksartefakt für unsere Scala Binärversion aus. Die Java-Bibliotheken verwenden `%`.

Anwendung und Tests laufen in separaten JVM-Prozessen. Dadurch bleiben Klassenladen und Lebenszyklus des eingebetteten Servers von sbt getrennt.

## Den Einstiegspunkt hinzufügen

Die Anwendung benötigt ein Objekt mit einer `main`-Methode:

**Datei:** `application/backend/src/main/scala/com/anjunar/blog/ApplicationMain.scala`

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

`start` erstellt den eingebetteten Server, registriert unsere REST-Anwendung und bindet sie an Loopback. `main` wählt den Port aus, installiert den Shutdown-Hook und hält den Prozess am Laufen.

Die Trennung von Startlogik und `main` erlaubt dem Integrationstest, denselben Server zu starten und zu stoppen, ohne unbegrenzt zu warten. Die atomare Variable stellt sicher, dass das Herunterfahren höchstens einmal ausgeführt wird, selbst wenn Shutdown-Hook und `finally`-Block es beide versuchen.

Der Standardport ist 8080. Wir können es mit `BLOG_PORT` überschreiben.

## Registrieren einer REST-Ressource

Die REST-Anwendung definiert das allgemeine URL-Präfix und die Ressourcenklassen:

**Datei:** `application/backend/src/main/scala/com/anjunar/blog/ServerApplication.scala`

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

Für dieses erste Modul ist die Ressourcenliste explizit. Das Präfix `/service` gilt für die hier registrierten Endpunkte.

Die Ressource erhält ihre Antwort von einem kleinen Dienst:

**Datei:** `application/backend/src/main/scala/com/anjunar/blog/GreetingService.scala`

```scala
package com.anjunar.blog

import jakarta.enterprise.context.ApplicationScoped

@ApplicationScoped
class GreetingService {

  def message: String = "Welcome to Anjunar Blog Tutorial!\n"

}
```

`@ApplicationScoped` macht dies zu einer CDI-Bean, deren kontextbezogene Instanz in der gesamten Anwendung geteilt wird.

Fügen Sie nun den Endpunkt hinzu:

**Datei:** `application/backend/src/main/scala/com/anjunar/blog/HelloResource.scala`

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

Die Annotationen beschreiben einen HTTP GET-Endpunkt, der UTF-8-Klartext zurückgibt. Der vollständige Pfad kombiniert `/service` aus der Anwendung mit `/hello` aus der Ressource.

Weld injiziert `GreetingService`, bevor die Request-Methode den Dienst verwendet. Die Begrüßung ist bewusst einfach: Eine erfolgreiche HTTP-Antwort zeigt, dass Routing und Abhängigkeitsinjektion funktionieren.

Schließlich aktivieren wir die CDI-Bean-Erkennung:

**Datei:** `application/backend/src/main/resources/META-INF/beans.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       version="4.0"
       bean-discovery-mode="annotated">
</beans>
```

Mit `bean-discovery-mode="annotated"` entdeckt CDI Beans mit einer Bean-definierenden Annotation, wie die Bereiche, die für unseren Service und unsere Ressource verwendet werden.

## Generierte Dateien aus Git heraushalten

Das Repository sollte die Quellen und die Builddefinition enthalten. Fügen Sie diese Ignorierregeln hinzu:

**Datei:** `.gitignore`

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

Die `target/`-Regel deckt auch die generierte Ausgabe in Unterverzeichnissen ab. Lokale Anwendungskonfigurations- und Umgebungsdateien sind ausgeschlossen, so dass zukünftige Anmeldeinformationen außerhalb von Commits bleiben.

Verwenden Sie konsistente Zeilenenden:

**Datei:** `.gitattributes`

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
```

Sie können nun ein Git-Repository mit `git init` initialisieren, wenn Sie das Beispiel von Hand erstellen.

## Testen Sie die komplette Anfrage

Beim Kompilieren fallen Typfehler auf. Um das Zusammenspiel von Undertow, RESTEasy und Weld zu prüfen, senden wir außerdem eine echte Anfrage.

**Datei:** `application/backend/src/test/scala/com/anjunar/blog/ServerIntegrationSpec.scala`

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

Der Test fragt das Betriebssystem nach einem verfügbaren Loopback-Port und schließt diesen temporären Socket, bevor der Server gestartet wird. Es gibt ein kurzes Intervall, in dem ein anderer Prozess den Port beanspruchen könnte; wenn das passiert, führen Sie den Test erneut aus.

Die Assertions prüfen Status, Medientyp und Antwortinhalt. Die Begrüßung zeigt zugleich, dass die CDI-Injektion funktioniert. Die zweite Anfrage prüft, ob eine unbekannte Ressource HTTP 404 liefert. Der `finally`-Block schließt HTTP-Client und Server.

Führen Sie den Befehl im Projektverzeichnis aus:

```text
sbt --server "application-backend/testFull"
```

Hier läuft `--server` sbt im Vordergrund. Der erste Aufruf kann länger dauern, während Abhängigkeiten heruntergeladen werden.

Das Ergebnis sollte einen erfolgreichen Test melden. Wir verwenden `testFull`, da die `test`-Aufgabe von sbt 2 inkrementell ist und Tests überspringen kann, die bereits bestanden wurden. Der [sbt Testdokumentation](https://www.scala-sbt.org/2.x/docs/en/reference/sbt-test.html) beschreibt die Unterscheidung. Wenn `testFull` null Tests meldet, überprüfen Sie das Testverzeichnis und die Projekt-ID im Befehl.

## Starten Sie die Anwendung

Im selben Projektverzeichnis ausführen:

```text
sbt --server "application-backend/run"
```

Nach dem Start druckt die Anwendung:

```text
Blog server: http://127.0.0.1:8080/service/hello
```

Halten Sie das Terminal offen. Senden Sie in einem anderen Terminal die Anfrage:

```text
curl -i http://127.0.0.1:8080/service/hello
```

Verwenden Sie in Windows PowerShell `curl.exe`, wenn `curl` ein Alias für einen PowerShell-Befehl ist.

Erwarten Sie HTTP 200, einen `text/plain`-Inhaltstyp und die am Anfang dieses Artikels gezeigte Begrüßung. Das Anfordern von `/service/missing` sollte HTTP 404 zurückgeben.

Wenn Port 8080 bereits besetzt ist, wählen Sie vor dem Start einen anderen Port aus. In PowerShell:

```powershell
$env:BLOG_PORT = "8081"
sbt --server "application-backend/run"
```

In Bash:

```bash
BLOG_PORT=8081 sbt --server "application-backend/run"
```

Verwenden Sie den ausgewählten Port in der Anfrage-URL. Drücken Sie Strg + C im Server-Terminal, wenn Sie fertig sind. Unter Windows fordert der Batch-Launcher Sie möglicherweise auch auf, zu bestätigen, dass Sie den Batch-Auftrag beenden möchten.

## Der Stand dieses Kapitels

Der vollständige Quellcode ist im [Kapitelstand von Artikel 2](https://github.com/anjunar/anjunar-blog-example/tree/0b9ea6f9447069bbe291e486fcdc8fb633d10656) festgehalten. Um genau diese Version zu verwenden:

```text
git clone https://github.com/anjunar/anjunar-blog-example.git
cd anjunar-blog-example
git switch --detach 0b9ea6f9447069bbe291e486fcdc8fb633d10656
sbt --server "application-backend/testFull"
```

Verwenden Sie ein separates Verzeichnis für diesen Klon, wenn Sie die Dateien bereits von Hand erstellt haben. Der Checkout ist getrennt, so dass spätere Änderungen an `main` das Beispiel nicht stillschweigend ändern können. Erstellen Sie einen Branch mit `git switch -c my-blog`, wenn Sie darauf aufbauen möchten.

Wir haben jetzt einen reproduzierbaren Build und einen HTTP-Endpunkt, der von einer CDI-Bean unterstützt wird. Das nächste Kapitel folgt dieser Anfrage über den Server im Detail: wie die Komponenten entdeckt werden, wie sich ihre Lebensdauer verhält und was beim Start und Herunterfahren passiert.
