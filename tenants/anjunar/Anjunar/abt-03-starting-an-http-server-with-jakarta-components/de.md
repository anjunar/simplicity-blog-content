Der Server aus Kapitel 2 kann eine Anfrage beantworten, aber seine Ressourcenliste ist hart codiert. Jeder neue Endpunkt würde einen weiteren Eintrag in `ServerApplication` erfordern.

In diesem Kapitel werden wir diese Liste mit der CDI-Erkennung verbinden. Das Hinzufügen einer Ressource mit `@Path` und einem CDI-Bereich stellt sie über RESTEasy zur Verfügung. Ein neuer `GET /service/health/live`-Endpunkt gibt uns ein konkretes Ergebnis, das wir überprüfen müssen.

Wir verfolgen außerdem Start, Anfrageverarbeitung und Herunterfahren durch Undertow, RESTEasy und Weld. So sehen wir, wer unsere Objekte erzeugt und wann sie freigegeben werden.

## Beginnen Sie mit dem vorherigen Kapitel

Verwenden Sie das Arbeitsprojekt von [Kapitel 2](https://github.com/anjunar/anjunar-blog-example/tree/0b9ea6f9447069bbe291e486fcdc8fb633d10656). Behalten Sie die bisherigen Versionen von JDK, sbt, Scala und den Bibliotheken bei. Dieses Kapitel fügt keine Abhängigkeiten hinzu.

Alle Pfade unten sind relativ zum Projektverzeichnis. Der komplette Kapitel-Checkpoint inklusive der neuen Tests ist am Ende verlinkt.

## Geben Sie jeder Komponente eine klare Verantwortung

|Komponente|Verantwortung in unserer Anwendung|
| --- | --- |
|Undertow|Öffnet den HTTP-Listener und betreibt das Servlet-Deployment.|
|RESTEasy|Leitet Anfragen an Jakarta-REST-Ressourcen weiter und erzeugt HTTP-Antworten.|
|Weld|Entdeckt CDI-Beans, löst Abhängigkeiten und verwaltet deren Lebensdauer.|

Jakarta REST und CDI definieren die APIs, die in unserem Code verwendet werden. RESTEasy und Weld implementieren diese APIs. Die `resteasy-undertow-cdi`-Abhängigkeit bietet die Integration, die Weld neben dem eingebetteten Server startet. Siehe [RESTEasy Bootstrap Dokumentation](https://docs.resteasy.dev/7.0/userguide/).

Unser `ApplicationMain.start` erstellt bereits diesen integrierten Server:

```scala
val server = new UndertowCdiEmbeddedServer()
server.getDeployment.setApplication(new ServerApplication())
val configuration = SeBootstrap.Configuration.builder()
  .host("127.0.0.1")
  .port(port)
  .rootPath("/")
  .build()
```

Dies ist ein Auszug aus `application/backend/src/main/scala/com/anjunar/blog/ApplicationMain.scala`; behalten Sie die bestehende Methode und ihre Fehlerbehandlung bei.

Mit unserer gepinnten RESTEasy-Version durchläuft das Startup diese Phasen:

1. Undertow öffnet den Zuhörer.
2. Die CDI-Bereitstellung startet Weld und verarbeitet die entdeckten Beans.
3. RESTEasy liest die Ressourcen- und Providerklassen der Anwendung und konfiguriert die CDI-Integration.
4. Die Servlet-Bereitstellung verbindet den Dispatcher und den Weld-Listener mit eingehenden Anfragen.

Die frühe Undertow-Logmeldung bedeutet, dass der Listener gestartet ist. Unsere eigene `Blog server: ...`-Nachricht erscheint, nachdem `server.start(configuration)` erfolgreich zurückgekehrt ist. Erst eine echte Anfrage prüft den vollständigen Ablauf.

## REST-Komponenten während der CDI-Entdeckung sammeln

In Kapitel 2 gab `ServerApplication.getClasses` einen Satz zurück, der nur `HelloResource` enthielt. Wir werden das manuell gepflegte Set durch Klassen ersetzen, die während des CDI-Bootstraps gesammelt wurden.

CDI bietet eine *Portable Extension*: eine Klasse, die Lebenszyklusereignisse des Containers beobachtet. `ProcessManagedBean` liefert die annotierte Klasse einer entdeckten Managed Bean. Unsere Erweiterung hält diejenigen, die mit `@Path` oder `@Provider` gekennzeichnet sind. Diese Hooks sind in [CDIs Portable Extension API](https://jakarta.ee/specifications/cdi/4.1/jakarta-cdi-spec-4.1.html#process_bean) definiert.

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/RestComponentsExtension.scala`:

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

`@Observes` markiert den Ereignisparameter. `@Path` identifiziert Ressourcenklassen; `@Provider` identifiziert REST-Anbieter wie Antwortfilter. Wir sammeln hier Klassenmetadaten. RESTEasy und Weld kümmern sich bei Bedarf um die Instanzen.

Das Set gehört zu dieser Erweiterungsinstanz, so dass ein vom nächsten Test erstellter Server mit einer eigenen Sammlung beginnt. `classes` gibt eine unveränderliche Kopie zurück.

Registrieren Sie die Erweiterung in dieser exakten Datei:

`application/backend/src/main/resources/META-INF/services/jakarta.enterprise.inject.spi.Extension`

```text
com.anjunar.blog.RestComponentsExtension
```

Der Dateiname benennt die Service-Schnittstelle; sein Inhalt benennt unsere Implementierung. Behalten Sie auch das vorhandene `META-INF/beans.xml`. Sie aktiviert die Erkennung annotierter Beans für die Anwendung.

Unsere Konvention ist, dass jede Ressource und jeder Anbieter ihren Umfang explizit erklärt. Die Erweiterung behandelt verwaltete Beansklassen; sie registriert keine willkürlichen Objekte, die von Produzentenmethoden zurückgegeben werden.

## Lassen Sie RESTEasy die entdeckten Klassen verwenden

Ersetzen Sie `application/backend/src/main/scala/com/anjunar/blog/ServerApplication.scala` durch:

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

`BeanManager.getExtension` ruft die vom laufenden Container erstellte Erweiterungsinstanz ab. Seine Sammlung wurde bereits befüllt, wenn RESTEasy nach diesen Klassen fragt.

Der explizite Lookup ist hier wichtig: `ApplicationMain` konstruiert diese Anwendung mit `new ServerApplication()`. Daher erhalten wir den Container an der Integrationsgrenze. Unsere Ressourcen verwenden weiterhin gewöhnliche `@Inject`-Abhängigkeiten.

Starten Sie die Anwendung nach dem Hinzufügen von Komponenten neu: Die Erkennung findet beim Start statt.

## Hinzufügen des Liveness-Endpunkts

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/HealthResource.scala`:

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

Es gibt keinen Klassenlisteneintrag zum Hinzufügen. Beim Start erkennt Weld diese Bean. Unsere Erweiterung macht ihre Klasse für RESTEasy verfügbar.

Die URL besteht aus drei Teilen:

|Einstellung|Wert|
| --- | --- |
|Server-Root-Pfad|`/`|
|`ServerApplication`-Anwendungspfad|`/service`|
|`HealthResource` Ressourcenpfad|`/health/live`|
|Ergebnis|`http://127.0.0.1:8080/service/health/live`|

Starten Sie die Anwendung im Projektverzeichnis:

```text
sbt --server "application-backend/run"
```

In einem anderen Terminal:

```text
curl -i http://127.0.0.1:8080/service/health/live
```

Verwenden Sie `curl.exe` in Windows PowerShell, wenn `curl` ein Alias ist. Erwarten Sie HTTP 200, einen `text/plain`-Inhaltetyp, und:

```text
UP
```

Die bestehende Begrüßung bei `/service/hello` sollte noch funktionieren, und `/service/missing` sollte 404 zurückgeben. Wenn Sie `BLOG_PORT` festlegen, verwenden Sie diesen Port in jeder URL.

Die Liveness-Antwort zeigt, dass der Server eine Anfrage an die Ressource weiterleiten kann. Eine Datenbank gibt es noch nicht; über Datenbankverbindung oder Bereitschaft zum Ausliefern gespeicherter Artikel sagt sie daher nichts aus.

## Folgen Sie einer Anfrage durch die Scopes

Betrachten Sie `GET /service/hello`. Undertow liefert die Anfrage an den RESTEasy-Dispatcher. Die Servlet-Integration aktiviert den CDI-Anforderungskontext. RESTEasy wählt `HelloResource` und seine CDI-Integration erhält die Ressource über Weld.

Unsere Ressource verwendet `@RequestScoped`. Der in ihn injizierte `GreetingService` verwendet `@ApplicationScoped`:

|Anwendungsbereich|Was wir in dieser Anwendung erwarten|
| --- | --- |
|Anfrage|Eine Bean-Instanz pro Anfrage; eine andere Anfrage erhält eine andere Instanz.|
|Anwendung|Eine Bean-Instanz, die alle Anfragen in diesem Container gemeinsam verwenden.|

CDI kann einen Proxy einfügen, der die Instanz für den aktiven Kontext auflöst. Die injizierte Referenz identifiziert daher nicht notwendigerweise das zugrunde liegende Objekt direkt. Die [CDI-Client-Proxy-Regeln](https://jakarta.ee/specifications/cdi/4.1/jakarta-cdi-spec-4.1.html#client_proxies) beschreiben diese Indirektion.

Für unseren Code ist die praktische Konsequenz einfach: Halten Sie den anfragespezifischen veränderlichen Zustand aus einem anwendungsweiten Dienst heraus. Mehrere Anfragen können es gleichzeitig aufrufen. `GreetingService` gibt derzeit einen festen String zurück und hat keinen solchen Zustand.

Die Konstruktion von `new HelloResource()` selbst würde den Containerlebenszyklus umgehen, der seine Abhängigkeit liefert. Lassen Sie die Integration Ressourceninstanzen erhalten.

## Überprüfen Sie Discovery und Lebensdauern über HTTP

Erweitern Sie den bestehenden `ServerIntegrationSpec` mit diesen Aussagen innerhalb seines `try`-Blocks, unmittelbar vor der Überprüfung unbekannter Ressourcen:

```scala
val health = get("/service/health/live")
assert(health.statusCode() == 200)
assert(health.body() == "UP\n")
```

Der Checkpoint enthält auch einen zweiten Test, [ServerLifecycleSpec](https://github.com/anjunar/anjunar-blog-example/blob/3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6/application/backend/src/test/scala/com/anjunar/blog/ServerLifecycleSpec.scala), und seinen [ScopeProbe-Test-Fixture](https://github.com/anjunar/anjunar-blog-example/blob/3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6/application/backend/src/test/scala/com/anjunar/blog/ScopeProbe.scala). Fügen Sie beide Dateien unter `application/backend/src/test/scala/com/anjunar/blog` hinzu, um diesen Test in Ihrem eigenen Projekt zu reproduzieren. Kopieren Sie die vorhandenen `beans.xml` auf `application/backend/src/test/resources/META-INF/beans.xml`, damit auch die Test-Beans erkannt werden.

Die Test-Fixtures geben jeder Request-Scoped- und Application-Scoped-Sonde eine UUID. Eine Testressource gibt beide IDs zurück; ein Testantwortfilter fügt die Anforderungs-ID einem Header hinzu. Ihre `@PreDestroy`-Methoden erfassen, welche Instanzen freigegeben wurden.

Zwei HTTP-Anfragen prüfen dann:

- Die Anforderungs-ID der Ressource entspricht für jede Antwort dem Header des Filters.
- Die Request-ID wechselt zwischen den Requests.
- Die Anwendungs-ID bleibt gleich.
- Beide Request-Instanzen werden nach Abschluss ihrer Requests zerstört.
- Die Anwendungsinstanz wird zerstört, wenn der Test den Server stoppt.

Da die Ressource und der Filter nur über unsere Erweiterung registriert werden, üben diese Prüfungen auch die Erkennung sowohl der `@Path`- als auch der `@Provider`-Klassen aus. Der Test wartet kurz auf die Zerstörung der Anforderung: Der Erhalt einer Antwort und der Abschluss der serverseitigen Bereinigung können zu leicht unterschiedlichen Zeiten erfolgen.

Führen Sie aus:

```text
sbt --server "application-backend/testFull"
```

Mit dem kompletten Checkpoint sollte das Ergebnis **zwei erfolgreiche Tests** melden. Die Sondenklassen leben unter `src/test`. Ein normaler Anwendungslauf schließt sie aus: `/service/_test/scopes` gibt 404 zurück.

## Den Server über seinen Besitzer stoppen

`ApplicationMain` besitzt bereits Shutdown. Sein Hook ruft den integrierten Server `stop()` auf, der von einem `AtomicBoolean` geschützt wird, so dass der Hook und der `finally`-Block diesen Anruf nicht beide ausführen können.

Der eingebettete Server stoppt Weld, das Servlet-Deployment und Undertow. Unser Lifecycle-Test überprüft den Zerstörungs-Callback der Anwendungsbohne. Wir behalten auch den Startfehlerhandler bei, der `stop()` aufruft, wenn der Server ausfällt.

Drücken Sie Strg + C im Server-Terminal. Unter Windows kann der sbt-Launcher Sie auffordern, die Beendigung des Batch-Auftrags zu bestätigen. Der Listener sollte sich anschließend schließen.

Dies gibt unserem Entwicklungsprozess eine explizite Inbetriebnahme und Bereinigung. Wie laufende Anfragen während eines Produktionsdeployments behandelt werden, besprechen wir im Deploymentkapitel.

## Der Stand dieses Kapitels

Der [vollständige Quellcode für Kapitel 3](https://github.com/anjunar/anjunar-blog-example/tree/3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6) enthält alle oben verwendeten Dateien. [Pull Request #3](https://github.com/anjunar/anjunar-blog-example/pull/3) zeigt die Änderungen aus dem vorherigen Kapitel. Um es in einem separaten Verzeichnis auszuführen:

```text
git clone https://github.com/anjunar/anjunar-blog-example.git
cd anjunar-blog-example
git switch --detach 3a30646b4992dda24f30f1ebe1bdd560d6f3b8a6
sbt --server "application-backend/testFull"
```

Wählen Sie ein übergeordnetes Verzeichnis, in dem `anjunar-blog-example` nicht bereits vorhanden ist. Der Checkout auf einem festen Commit hält die Beispiele stabil, wenn die Serie wächst. Verwenden Sie `git switch -c my-blog`, um von diesem Punkt an Ihre eigene Implementierung fortzusetzen.

Im nächsten Kapitel werden PostgreSQL, Hibernate, ein Verbindungspool und Transaktionen hinzugefügt. Wir werden den hier beschriebenen Request-Lebenszyklus mit Datenbankarbeit, Commit und Rollback verlängern.
