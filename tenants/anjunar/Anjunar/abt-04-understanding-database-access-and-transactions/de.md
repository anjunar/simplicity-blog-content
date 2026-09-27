Ein funktionierender HTTP-Server sagt noch nichts darüber aus, ob Datenbankänderungen eine Anfrage überdauern. In diesem Kapitel ergänzen wir PostgreSQL und klare Transaktionsregeln: Erfolgreiche Schreibvorgänge werden committet, fehlgeschlagene zurückgerollt und Leseanfragen hinterlassen keine Änderungen.

Unser sichtbares Ergebnis ist `GET /service/health/ready`. Es führt eine Abfrage über Hibernate aus und gibt `UP` zurück. Die Integrationstests gehen noch weiter: Sie fügen Zeilen ein, erzwingen Ausfälle und überprüfen das Ergebnis über eine separate Datenbankverbindung.

Beginnen Sie mit dem [Kapitel 3 Projekt](https://github.com/anjunar/anjunar-blog-example/tree/3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6). Alle Pfade unten sind relativ zu seiner Wurzel. Die Entität `BlogPost` führen wir in Kapitel 5 ein; hier legen wir die Infrastruktur fest, die sie benötigt.

## Den Weg zu PostgreSQL verstehen

|Teil|Verantwortung|
| --- | --- |
|EntityManager|Hält den Persistenzkontext, der von einer Anforderung verwendet wird, und führt ihre Abfragen aus.|
|Hibernate|Implementiert Jakarta Persistence und übersetzt Entity-Operationen in SQL.|
|Agroal|Stellt gepoolte JDBC-Verbindungen bereit.|
|PostgreSQL JDBC Treiber|Kommuniziert mit PostgreSQL.|
|Narayana|Koordiniert Commit und Rollback über Jakarta Transactions (JTA).|
|PostgreSQL|Speichert die Daten und erzwingt Datenbankbeschränkungen.|

Pool und EntityManagerFactory bestehen während der gesamten Laufzeit der Anwendung. Jede datenbankgestützte Anfrage erhält ihren eigenen EntityManager und ihre eigene Transaktion. Die gemeinsame Nutzung eines EntityManagers für gleichzeitige Anfragen würde deren Persistenzkontexte mischen.

Wir werden die gleichen Bibliotheken wie der Referenzstapel verwenden, mit einer Datenbank und keinem Mandantenkontext.

## Starten Sie eine lokale Datenbank

Erstellen Sie `compose.yaml`:

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

Wenn Docker und Compose installiert sind, legen Sie ein Entwicklungskennwort fest und starten Sie PostgreSQL. In PowerShell:

```powershell
$env:BLOG_DB_PASSWORD = "local-blog-password"
docker compose up -d --wait
```

In Bash:

```bash
export BLOG_DB_PASSWORD=local-blog-password
docker compose up -d --wait
```

Verwenden Sie dieses Terminal weiterhin für sbt, damit die Anwendung das Passwort erbt. Die Datenbank hört auf Port 5433 der Loopback-Schnittstelle.

PostgreSQL 18-Bilder verwenden `/var/lib/postgresql` als Volume-Mount. Die Initialisierungsvariablen des Bildes gelten, wenn das Datenverzeichnis leer ist; das spätere Ändern der Passwortvariable ändert nicht das Passwort eines vorhandenen Datenbankbenutzers. Siehe [Offizielle Bilddokumentation](https://hub.docker.com/_/postgres).

Wenn PostgreSQL bereits lokal installiert ist, können Sie stattdessen eine dedizierte Tutorialrolle und Datenbank aus der `psql`-Sitzung eines Administrators erstellen:

```sql
CREATE ROLE blog LOGIN PASSWORD 'local-blog-password';
CREATE DATABASE anjunar_blog OWNER blog;
```

Setzen Sie `BLOG_DB_URL` auf den Port dieser Installation, zum Beispiel `jdbc:postgresql://127.0.0.1:5432/anjunar_blog`. Verwenden Sie diese separate Entwicklungsdatenbank für die Tests unten.

## Hinzufügen von Datenbankbibliotheken

Fügen Sie diese Einträge zur bestehenden `libraryDependencies`-Sequenz in `build.sbt` hinzu:

```scala
"org.hibernate.orm" % "hibernate-core" % "7.4.10.Final",
"org.postgresql" % "postgresql" % "42.7.13",
"org.jboss.narayana.jta" % "narayana-jta" % "7.3.4.Final",
"io.agroal" % "agroal-pool" % "3.2.1",
"io.agroal" % "agroal-narayana" % "3.2.1",
```

Hibernate liefert die Jakarta Persistence API transitiv. Das Narayana-Modul von Agroal verbindet den Pool mit dem Transaktionsmanager.

Behalten Sie die vorhandenen HTTP-, CDI- und Testabhängigkeiten. Fügen Sie `ObjectStore/` und `PutObjectStoreDirHere/` zu `.gitignore` für die lokalen Laufzeitdateien von Narayana hinzu.

## Lesen Sie die Verbindungseinstellungen aus der Umgebung

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/DatabaseConfig.scala`:

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

Die Standardwerte passen zu unserem Compose-Service. Das Passwort hat keinen Standard und wird über `BLOG_DB_PASSWORD` bereitgestellt. Die Anwendung liest diese Variablen direkt; sie lädt keine `.env`-Datei.

## Erstellen Sie den Pool und die EntityManagerFactory

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/Persistence.scala`:

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

Drei Verbindungen zwischen den Bibliotheken sind hier wichtig:

- `PGXADataSource` bietet transaktionsbewusste Datenbankverbindungen zu Agroal.
- `NarayanaTransactionIntegration` nimmt geliehene Verbindungen in die aktive JTA-Transaktion auf.
- Hibernate erhält den gleichen Pool, wählt den `jta`-Koordinator aus und verwendet die eigenständige Narayana-Plattform.

Die [Hibernate-Dokumentation zu Transaktionen](https://docs.hibernate.org/orm/7.4/userguide/html_single/#transactions) beschreibt die Koordinator- und JTA-Plattformeinstellungen. Die [Konfigurationsdokumentation](https://agroal.github.io/docs.html) von Agroal deckt die Poolgröße, die Verbindungserfassung und die Transaktionsintegration ab.

Wir begrenzen diesen Entwicklungspool auf acht Verbindungen und geben dem Verbindungserwerb ein 5-Sekunden-Timeout. Hibernate Schema Generation ist deaktiviert. Die Metadaten enthalten derzeit keine Entitäten; Schema-Erstellung und Migration kommen mit dem Domänenmodell.

Die Producer-Methode stellt den EntityManager der aktuellen Anforderung für `@Inject` zur Verfügung. Sie erzeugt keinen zweiten. `RequestTransaction`, definiert als nächstes, besitzt seine Erstellung und Schließung.

`Persistence` initialisiert, wenn es zum ersten Mal verwendet wird. Beim Herunterfahren schließt es die Factory vor dem Pool. Wenn die Factoryinitialisierung fehlschlägt, werden die Registrierung und der Pool freigegeben, während der ursprüngliche Fehler beibehalten wird.

## Eine Transaktion pro Anfrage verwalten

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/RequestTransaction.scala`:

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

`begin` startet eine Narayana-Transaktion, erstellt den EntityManager und verbindet ihn mit dieser Transaktion. Für GET und HEAD verhindert `FlushModeType.COMMIT` das normale automatische Spülen vor Abfragen; das endgültige Rollback verwirft alle Schreibvorgänge, einschließlich nativer SQL, die durch einen Test ausgeführt werden.

Unsere Abschlussrichtlinie lautet:

|Anfrageergebnis|Abschluss|
| --- | --- |
|GET oder HEAD|Rollback|
|Andere Methode mit einem Status unter 400|Commit|
|Fehlerstatus oder fehlgeschlagene Serialisierung|Rollback|

`flush` sendet ausstehende Entity-Änderungen an die Datenbank, **committet sie aber nicht**. Ein späterer Fehler kann diese Änderungen weiterhin zurückrollen. Manche Constraints werden erst beim Commit geprüft. Ein erfolgreicher Flush beweist deshalb noch nicht, dass ein Schreibvorgang erfolgreich abgeschlossen wird.

Der EntityManager wird geschlossen, wenn die Transaktion abgeschlossen ist. Der `@PreDestroy`-Callback ist ein Fallback: Wenn ein früherer Fehler eine Anforderung unvollendet lässt, wird sie durch die Zerstörung des CDI-Kontexts zurückgesetzt.

In diesem Kapitel wird die programmatische Transaktions-API von Narayana explizit verwendet. Sie aktiviert weder automatische `@Transactional`-Interceptors oder die Transaktionsbeobachter von Weld. Weld kann immer noch protokollieren, dass seine eigenen Transaktionsdienste während des Starts nicht verfügbar sind.

## Halten Sie die Transaktion durch Serialisierung offen

Die Rückkehr von einer Ressourcenmethode beendet das Schreiben einer HTTP-Antwort nicht. Ein MessageBodyWriter muss das Ergebnis noch in Bytes umwandeln. Später in der Reihe kann dieser Writer Entity-Beziehungen traversieren und den offenen Persistenzkontext benötigen.

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/TransactionBoundary.scala`:

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

Die Erweiterung von Kapitel 3 erkennt dieses `@Provider` automatisch.

Der Anforderungsfilter startet die Transaktion. Er nimmt `HealthResource`, unseren Liveness-Endpunkt, aus, so dass die Überprüfung, ob der HTTP-Server am Leben ist, PostgreSQL nicht erfordert.

Der Antwortfilter kennt den Status und flusht erfolgreiche Schreibvorgänge. Bei HEAD und Responses ohne Entität schließt es auch die Transaktion ab: Es gibt dann keinen Response-Body, den der Interceptor abfangen könnte.

Für andere Antworten wickelt der [WriterInterceptor](https://jakarta.ee/specifications/restful-ws/4.0/apidocs/jakarta.ws.rs/jakarta/ws/rs/ext/writerinterceptor) die Serialisierung ab. Die Reihenfolge ist bewusst gewählt:

1. Serialisieren Sie in einen Puffer, während der EntityManager geöffnet bleibt.
2. Committen Sie einen erfolgreichen Schreibvorgang oder rollen Sie eine Lese- oder Fehlerantwort zurück.
3. Schließen Sie EntityManager.
4. Kopieren Sie die gepufferten Bytes in den HTTP-Ausgang.

Wenn Serialisierung oder Commit fehlschlagen, ist noch kein Erfolgsinhalt gesendet worden. RESTEasy kann eine Fehlerantwort erzeugen.

Die aktuellen Endpunkte geben kleine, synchrone Antworten zurück, so dass ein In-Memory-Puffer ausreicht. Streaming-Downloads und asynchrone Endpunkte benötigen eine andere Grenze. Auch das Begehen einer Transaktion kann keine Netzwerklieferung danach garantieren; ein getrennter Client kann die Antwort auf einen bereits committeten Schreibvorgang verpassen.

## Abfrage der Datenbank aus einer Ressource

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/DatabaseHealthResource.scala`:

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

Die Ressource erhält ihren EntityManager über CDI und führt `select 1` aus. Die Anforderungsgrenze übernimmt den Abschluss und die Bereinigung der Transaktion. Es gibt keinen manuellen Commit in der Ressource.

Starten Sie die Anwendung:

```text
sbt --server "application-backend/run"
```

In einem anderen Terminal:

```text
curl -i http://127.0.0.1:8080/service/health/live
curl -i http://127.0.0.1:8080/service/health/ready
```

Beide sollten HTTP 200 und `UP` zurückgeben. Verwenden Sie `curl.exe` in Windows PowerShell, falls erforderlich, und passen Sie den HTTP-Port an, wenn Sie `BLOG_PORT` einstellen.

Die Kontrollen beantworten verschiedene Fragen. Liveness erreicht eine HTTP-Ressource. Readiness erreicht auch PostgreSQL über Hibernate und den transaktionsbewussten Pool. Ein Datenbankfehler erzeugt derzeit HTTP 500 für die Bereitschaft, während Liveness weiterhin 200 zurückgibt.

Die Datenbank wird bei der ersten datenbankgestützten Anfrage initialisiert. Die Startnachricht des Servers bestätigt die HTTP-Bereitstellung; die Bereitschaftsanforderung überprüft den Datenbankpfad.

## Prüfen, was tatsächlich committet wird

Der Checkpoint fügt [DatenbankIntegrationSpec](https://github.com/anjunar/anjunar-blog-example/blob/e96365e906bb14b212fe2b0b9e11664f6b5afd93/application/backend/src/test/scala/com/anjunar/blog/DatabaseIntegrationSpec.scala) und [TransactionProbeResource](https://github.com/anjunar/anjunar-blog-example/blob/e96365e906bb14b212fe2b0b9e11664f6b5afd93/application/backend/src/test/scala/com/anjunar/blog/TransactionProbeResource.scala) unter `application/backend/src/test/scala/com/anjunar/blog` hinzu. Fügen Sie beide hinzu, wenn Sie das Kapitel von Hand reproduzieren.

Die Suite erstellt eine Tabelle mit einem eindeutigen Namen und entfernt sie anschließend. Testressourcen führen echte Inserts aus. Assertions verwenden separate JDBC-Verbindungen, so dass sie den committeten Datenbankzustand statt der noch offenen Änderungen der Anfrage sehen.

Die Kontrollen umfassen:

- Ein erfolgreiches Schreiben und eine erfolgreiche Antwort ohne Body.
- Rollback für einen Fehlerstatus und für eine SQL-Ausnahme nach einem Einfügen.
- Datenbankzugriff während Body-Serialisierung.
- Rollback, wenn der Schreiber nach dem Erstellen einiger gepufferter Bytes fehlschlägt.
- Eine Fremdschlüsseleinschränkung, die nur beim Commit fehlschlägt.
- Eine Transaktion, die ausdrücklich als Rollback-only gekennzeichnet ist.
- Schreibversuche während GET und HEAD.
- Der Bereitschaftsendpunkt.

Der verschobene Fremdschlüssel ist gerade deshalb nützlich, weil PostgreSQL diese Prüfung bis zum Abschluss der Transaktion verschieben kann. Siehe [Die Constraint Timing Regeln von PostgreSQL](https://www.postgresql.org/docs/18/sql-set-constraints.html). Dieser Test überprüft beide Ergebnisse: keine eingefügte Zeile und keine Erfolgsantwort.

Mit PostgreSQL ausgeführt und die Umgebungsvariablen gesetzt, ausführen:

```text
sbt --server "application-backend/testFull"
```

Erwarten Sie **12 erfolgreiche Tests**, einschließlich der beiden Tests aus den vorherigen Kapiteln. Die absichtlichen Fehlerfälle erzeugen Fehlerprotokolle; die abschließende Testzusammenfassung stellt fest, ob ihr erwartetes Verhalten bestanden hat.

Die Sondenressourcen sind Test-only. Ein normaler Anwendungslauf stellt ihre Schreibendpunkte nicht frei.

## Den Stand des Kapitels verwenden

Der [Quellcode zu Kapitel 4](https://github.com/anjunar/anjunar-blog-example/tree/e96365e906bb14b212fe2b0b9e11664f6b5afd93) enthält die komplette Implementierung und Tests. [Pull Request #4](https://github.com/anjunar/anjunar-blog-example/pull/4) zeigt die Änderungen aus Kapitel 3. Um es in einem separaten Verzeichnis zu verwenden:

```text
git clone https://github.com/anjunar/anjunar-blog-example.git
cd anjunar-blog-example
git switch --detach e96365e906bb14b212fe2b0b9e11664f6b5afd93
```

Konfigurieren und starten Sie PostgreSQL wie oben beschrieben und führen Sie `testFull` aus. Erstellen Sie einen Branch mit `git switch -c my-blog`, um Ihre eigene Implementierung fortzusetzen.

Beenden Sie die Anwendung mit Ctrl + C. Für die Compose-Datenbank stoppt `docker compose down` den Dienst unter Beibehaltung seines Datenvolumens.

Die Beispiele und alle 12 Tests wurden mit einer isolierten nativen Installation gegen PostgreSQL 18.6 verifiziert. Die Compose-Datei liefert die entsprechende lokale Datenbankeinrichtung; Docker war auf dem Rechner, auf dem dieser Artikel entstand, nicht verfügbar.

Als nächstes werden wir `BlogPost` hinzufügen: Identität, Slug, Veröffentlichungsstatus, Version und Validierungsregeln. Die Anfrage- und Transaktionsinfrastruktur ist nun bereit, sie fortzusetzen.
