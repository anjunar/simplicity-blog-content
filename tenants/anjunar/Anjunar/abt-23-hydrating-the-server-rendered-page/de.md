# Die serverseitig gerenderte Seite hydrieren

Kapitel 22 machte den Blog bereits vor dem JavaScript-Start lesbar. Anschließend
leerte `boot` aber noch `#app`, lud die aktuelle Route erneut und baute die
Seite neu auf.

Jetzt soll der Browser **die vorhandenen Knoten behalten** und den
Komponentenbaum mit ihnen verbinden. Artikel, Überschrift und Suchfeld sollen
dieselben DOM-Objekte bleiben. Der erste öffentliche Datenabruf soll aus dem
Netzwerkprotokoll des Browsers verschwinden.

Dafür müssen zwei Dinge übereinstimmen: der Komponentenbaum und die Daten, aus
denen er entsteht. `DomCursor` durch `HydratingCursor` zu ersetzen, löst nur
den ersten Teil.

## Die Antwort mitgeben, aus der das HTML entstand

Angenommen, der Server rendert einen deutschen Artikel. Ruft der Browser ihn
erneut ab, könnte die Übersetzung inzwischen geändert worden sein. Auch ohne
Änderung muss der Router auf diesen Abruf warten.

Deshalb erzeugt jedes
[BlogDocument](https://github.com/anjunar/anjunar-blog-example/blob/8de5903024000706abf604833a32ee18738d37aa/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogDocument.scala)
ein `InitialPageData.capture(url)` und reicht es an seinen `BlogService`
weiter. Dieses Objekt merkt sich die öffentliche Antwort der aktuellen Anfrage:

| Feld im Snapshot | Aufgabe |
| --- | --- |
| `format`, `url` | Zustandsformat und Pfad/Query des Dokuments identifizieren. |
| `request` | Den exakten API-Pfad samt Query, Sprache und Filtern abgleichen. |
| `status`, `contentType`, `body` | Dieselbe erfolgreiche Antwort oder denselben Fehler wiedergeben. |

Wir behalten den ursprünglichen JSON-Body. Das Frontend-Modell erneut zu
serialisieren wäre hier falsch: Seine Schreibregeln lassen reine Ausgabefelder
wie Übersetzung und Veröffentlichungsdaten bewusst aus. Der ursprüngliche Body
erhält diese Felder, ausgelassene Werte, null und Version 0 über den
vorhandenen Mapper.

Es geht um eine öffentliche Antwort pro gerenderter Seite. Kontosessions,
Zugangsdaten und private Redaktionsantworten gehören nicht in diesen Snapshot.

In `BlogDocument.compose` steht der folgende Block nach der vorhandenen
`DocumentHead`-Einrichtung. `documentHead` ist die Registry des Dokuments,
`initial` dessen Recorder für diese Anfrage. Seine kodierte Property ändert
sich, sobald die Antwort vorliegt:

```scala
import ui.core.document.HeadEntry

val stateHead = documentHead.handle(this)
addDisposable(initial.encoded.observe(value => stateHead.set(
  HeadEntry("application-state", "script",
    Seq("id" -> "application-state", "type" -> "application/json", "data-state" -> value))
)))
```

`HeadEntry` schreibt ein inertes `application/json`-Element. Das Attribut
`data-state` enthält URI-kodiertes JSON; zusätzlich maskiert der Head-Writer
die Attributwerte. Artikeltext mit Anführungszeichen oder `</script>` bleibt
dadurch Daten und wird nicht zu ausführbarem Script-Inhalt.

Der Recorder gehört zur Dokumentinstanz. Er ist kein gemeinsamer Cache für
verschiedene Server-Anfragen.

## Die erste Route sofort bereitstellen

Es gibt eine zweite, leicht übersehene Voraussetzung. Ist ein Router-Loader
während der Hydration noch nicht fertig, übernimmt diese Framework-Version
den sichtbaren Bereich zunächst vorläufig und ersetzt seinen Inhalt nach dem
Laden. Das verhindert eine leere Seite, erhält aber nicht die DOM-Identität
des Artikels.

Der Replay-Pfad muss deshalb ein **bereits abgeschlossenes Future** liefern.
Auch die reinen Transformationen bis zur fertigen Komponente müssen diese
Eigenschaft erhalten.

Hier ist der vollständige öffentliche Service:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom

import scala.concurrent.{ExecutionContext, Future}
import scala.scalajs.js.URIUtils.encodeURIComponent

final class BlogService(initial: InitialPageData = InitialPageData.live())(using ExecutionContext) {
  def list(search: PostSearch, signal: Option[dom.AbortSignal], locale: String = "en"): Future[BlogPostTable] =
    initial.get[BlogPostTable](s"/service/blog/posts?${search.queryString(includeDefaults = true)}&locale=$locale", signal)
      .map { table =>
        require(table.size >= 0 && table.rows != null, "Invalid post table")
        table.rows.foreach(row => require(row != null && row.data != null, "Missing post data"))
        table
      }(using ExecutionContext.parasitic)

  def detail(slug: String, signal: Option[dom.AbortSignal], locale: String = "en"): Future[BlogPost] =
    initial.get[BlogPostData](s"/service/blog/posts/${encodeURIComponent(slug)}?locale=$locale", signal)
      .map { result =>
        require(result.data != null && result.data.content.get != null, "Missing post detail")
        result.data
      }(using ExecutionContext.parasitic)
}
```

[InitialPageData](https://github.com/anjunar/anjunar-blog-example/blob/8de5903024000706abf604833a32ee18738d37aa/application/frontend/src/main/scala/com/anjunar/blog/frontend/InitialPageData.scala)
hat drei Betriebsarten. Auf dem Server zeichnet es die empfangene Antwort auf.
Beim Browser-Start dekodiert es die passende gespeicherte Antwort unmittelbar
über `HttpJson.decode`. Danach reicht es Anfragen an den normalen HTTP-Pfad
weiter.

Die gespeicherte Antwort wird einmal verbraucht. `ExecutionContext.parasitic`
führt die kleinen Validierungsschritte unmittelbar aus, wenn das eingehende
Future bereits fertig ist. Browser-Netzwerkzugriffe werden dadurch nicht
synchron: Echte Anfragen warten weiterhin auf `fetch`.

Auch der letzte Map-Schritt der öffentlichen Route braucht dieses Verhalten.
Der folgende Eintrag ersetzt die Detailroute in der vorhandenen Sequenz
`BlogRoutes.routes`; `service` ist deren Konstruktorabhängigkeit:

```scala
import ui.router.Route
import scala.concurrent.ExecutionContext

Route.view("/posts/:slug") { context =>
  val locale = context.locale.map(_.code).getOrElse("en")
  service.detail(context.pathParams("slug"), context.signal, locale)
    .map(new PostPage(_, locale))(using ExecutionContext.parasitic)
}
```

Die Listenroute folgt demselben Muster. Mit dem üblichen verzögert ausgeführten
ExecutionContext sähe der Router beim letzten Map-Schritt wieder ein wartendes
Future, obwohl das JSON längst vorhanden ist.

Fehler gehören ebenfalls dazu. Eine gespeicherte 404-Antwort liefert ein bereits
fehlgeschlagenes Future und wählt die vorhandene Fehlerroute ohne erneuten
API-Aufruf. Eine ungültige Such-URL kann einen leeren Snapshot haben, weil die
Parameterprüfung sie bereits vor dem ersten Abruf ablehnt.

## Die Seite übernehmen und die Hydration abschließen

Die Dokumenthülle aus Kapitel 22 bleibt bestehen. Wir hydrieren `BlogPage`
innerhalb von `#app`, wo der Server dieselbe Komponente gemountet hat.

Hier ist das vollständig aktualisierte `Main`. Die Server-Funktion `render`
bleibt unverändert; der Browser-Pfad übernimmt jetzt Hydration und Fehlerbehandlung:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom
import ui.core.async.AsyncRenderContext
import ui.core.component.Runtime
import ui.core.render.{DomCursor, HydratingCursor}

import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.Future
import scala.util.control.NonFatal
import scala.scalajs.js
import scala.scalajs.js.annotation.JSExportTopLevel
import scala.scalajs.js.JSConverters.*

object Main {
  @JSExportTopLevel("render")
  def render(url: String): js.Promise[js.Object] = {
    val document = new BlogDocument(url)
    Runtime.renderToStringAsync(cursor => Runtime.mount(document, cursor), timeoutMs = 10000).map { html =>
      js.Dynamic.literal(html = ("<!doctype html>" + html), status = document.responseStatus)
    }.toJSPromise
  }

  private var booted: Option[js.Promise[Unit]] = None

  @JSExportTopLevel("boot")
  def boot(): js.Promise[Unit] = booted.getOrElse {
    val result = try start().toJSPromise catch { case NonFatal(error) => Future.failed[Unit](error).toJSPromise }
    booted = Some(result)
    result
  }

  private def start(): Future[Unit] = {
    val root = dom.document.getElementById("app")
    require(root != null, "The page must contain an element with id='app'")
    val url = dom.window.location.pathname + dom.window.location.search
    val embedded = Option(dom.document.getElementById("application-state"))
    val actions = new BlogActions(() => dom.window.location.reload())

    def mountFresh(): Future[Unit] = {
      root.textContent = ""
      val async = new AsyncRenderContext()
      Runtime.mount(new BlogPage(new BlogService, actions), DomCursor.root(root, async))
      async.drain()
    }

    val result = embedded match {
      case None => mountFresh() // Account/editorial pages and the shell-only test configuration.
      case Some(node) =>
        val async = new AsyncRenderContext()
        var page: Option[BlogPage] = None
        val hydrated = try {
          val initial = InitialPageData.replay(node.getAttribute("data-state"), url)
          val cursor = HydratingCursor.root(root, async)
          val component = new BlogPage(new BlogService(initial), actions)
          page = Some(component)
          Runtime.mount(component, cursor)
          async.drain().map { _ =>
            cursor.completeHydration()
            initial.finish()
          }
        } catch { case NonFatal(error) => Future.failed(error) }
        hydrated.recoverWith { case NonFatal(error) =>
          async.cancel()
          page.filter(_.isBound).foreach(Runtime.unmount)
          dom.console.warn("Hydration failed; rebuilding the page.", error.getMessage)
          mountFresh()
        }
    }
    // The snapshot is bootstrap data. Removing it also releases the serialized response.
    result.andThen { case _ => embedded.foreach(_.remove()) }
  }
}
```

Die Reihenfolge in `start` ist entscheidend:

1. Aktuellen Pfad und Query lesen und den eingebetteten Snapshot dagegen prüfen.
   Ein Snapshot für eine andere Sprache oder Such-URL darf diese Seite nicht
   initialisieren. URL-Fragmente bleiben Sache des Browsers.
2. Mit `HydratingCursor` mounten, damit Komponenten vorhandene Elemente und
   Anker virtueller Bereiche übernehmen. Die sofort fertige erste Route erlaubt
   das auch für den Artikel.
3. Die registrierte asynchrone Arbeit abwarten und dann
   `completeHydration` aufrufen. Das prüft, ob der erwartete Baum vollständig
   übernommen wurde, und aktiviert zurückgestellte Callbacks.
4. Mit `finish` den Replay-Zustand freigeben. Spätere Navigation verwendet wieder
   die API. Das serialisierte Bootstrap-Element wird nach dem Start entfernt.

`boot` behält sein Promise. Wiederholte Aufrufe können deshalb keine zweite
Anwendung mounten oder zusätzliche Listener registrieren. `ClientMain` ruft
die Funktion beim Start des Browser-Bundles weiterhin einmal auf.

## Eingaben erhalten und Wiederherstellung sichtbar machen

Leser können bereits ins serverseitig gerenderte Suchformular tippen, während
das Modul noch lädt. Normale Input-Bindings würden ihre Modell-Anfangswerte
sonst über diese Eingaben schreiben.

[PostSearchForm](https://github.com/anjunar/anjunar-blog-example/blob/8de5903024000706abf604833a32ee18738d37aa/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostSearchForm.scala)
erfasst deshalb die nativen Feldwerte in `beforeHostBinding`, bevor die Bindings
laufen. Sein `afterHydration`-Callback überträgt sie anschließend in die
gebundenen Properties. Der Anfangsbaum bleibt während der Übernahme unverändert;
danach sehen die normale Validierung und die Sprachwechsel-Sperre das bearbeitete
Modell. Der DSL-Baum bleibt weiterhin zusammenhängend in `compose`.

Fehlt der Zustand, wird frisch gemountet. Dasselbe gilt für die Shell von Konto-
und Redaktionsseiten. Ein beschädigter Snapshot, eine falsche Seiten-URL oder
eine abweichende DOM-Struktur brechen den Hydrationsversuch ab. Sein teilweiser
Komponentenbaum wird ausgehängt, bevor **ein** frisches Mounten beginnt.

Diese Wiederherstellung protokolliert eine Warnung und kann erneut Daten laden.
Sie zählt nicht als erfolgreiche Hydration und startet keine Reload-Schleife.
Ein kompletter Neuaufbau kann Eingaben verwerfen, die vor dem Start entstanden
sind; erfolgreiche Hydration erhält sie.

Der Wiederherstellungstest prüft nach dem Neuaufbau einen tatsächlich
funktionierenden Schalter. Nur eine sichtbare Seite würde nicht beweisen, dass
Listener des abgebrochenen Versuchs entfernt wurden.

## Identität prüfen, nicht nur gleichen Text

Starte die Anwendung mit der vorhandenen Datenbank. Weder Migration noch neue
Abhängigkeit sind nötig:

```text
sbt --server frontendAssets "application-backend/run"
npm run test:browser:ssr
```

Beende den laufenden Server vor der automatischen Suite; sie verwendet Port
18080. Die [Kapitelanleitung](https://github.com/anjunar/anjunar-blog-example/blob/8de5903024000706abf604833a32ee18738d37aa/docs/hydrating-the-server-rendered-page.md) beschreibt
Umgebung und vollständige Regressionstests.

Die Browser-Tests pausieren den Modul-Download, behalten Referenzen auf
serverseitig erzeugte Knoten und vergleichen sie nach `boot`. Sie prüfen
außerdem, dass kein erster öffentlicher API-Abruf entsteht, frühe Eingaben
erhalten bleiben und zwei `boot`-Aufrufe keine zweite Instanz erzeugen.
Ein weiterer Test ändert die Datenbank nach dem SSR: Beim Start bleibt die
gerenderte Fassung erhalten, ein späterer Sprachwechsel hin und zurück lädt
den neueren Titel.

Aus dem lesbaren HTML wird jetzt ein interaktiver Komponentenbaum, ohne seinen
Inhalt beim normalen Start zu ersetzen. Kapitel 24 ist der Abschluss:
Metadaten, Canonical- und Sprachlinks, Sitemap und Feed.
