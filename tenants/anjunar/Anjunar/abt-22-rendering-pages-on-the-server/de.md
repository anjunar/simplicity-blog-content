# Seiten auf dem Server rendern

Öffne einen Artikel mit deaktiviertem JavaScript. Bisher enthält die Antwort
ein leeres `#app`: Erst muss der Browser Scala.js starten und den Beitrag laden,
bevor jemand ihn lesen kann. In diesem Kapitel steht der Artikel bereits in
der Antwort.

Dazu führen wir unsere vorhandene Scala.js-Oberfläche mit GraalJS auf der JVM
aus. Dieselben Routen laden dieselbe öffentliche API und mounten dieselben
Komponenten. Englisch, Deutsch, Markdown und der Übersetzungs-Fallback folgen
damit dem Vertrag aus den vorherigen Kapiteln. Ein fehlender Artikel braucht
außerdem eine **Dokumentantwort mit HTTP 404**, nicht lediglich eine
Fehlermeldung auf einer erfolgreich ausgelieferten Seite.

Dieses Kapitel liefert das HTML. In Kapitel 23 übernimmt der Browser es durch
Hydration. Vorerst startet er einen neuen Komponentenbaum.

## Zwei Einstiegspunkte für das Frontend

Das bisherige `Main.main` greift sofort auf `document` und `window` zu.
Das ist ein Browser-Einstieg; auf dem Server würde er bereits vor dem Rendern
scheitern.

Wir teilen den Build in zwei Scala.js-Projekte:

| Projekt | Aufgabe | Modul-Initializer | Kopierte Ausgabe |
| --- | --- | --- | --- |
| `application-frontend` | Gemeinsame Oberfläche und exportierte Render-Funktion | Deaktiviert | `target/frontend/ssr/main.js` |
| `application-client` | Browser-Start; hängt vom Frontend ab | Aktiviert | `target/frontend/main.js` |

Beide erzeugen ES-Module. Der vorhandene Task `frontendAssets` baut und kopiert
beide Ausgaben; die UI-Abhängigkeiten bleiben im gemeinsamen Frontend-Projekt.
Die [Build-Definition](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/build.sbt) enthält sämtliche Einstellungen.

Der vollständige Browser-Einstieg lautet:

```scala
package com.anjunar.blog.client

import com.anjunar.blog.frontend.Main

object ClientMain {
  def main(args: Array[String]): Unit = Main.boot()
}
```

Das gemeinsame `Main` sieht jetzt so aus, einschließlich des Browser-Starts:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom
import ui.core.component.Runtime
import ui.core.render.DomCursor

import scala.concurrent.ExecutionContext.Implicits.global
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

  def boot(): Unit = {
    val root = dom.document.getElementById("app")
    require(root != null, "The page must contain an element with id='app'")
    // Chapter 22 remounts the page. Chapter 23 will reuse this tree through hydration.
    root.textContent = ""
    val service = new BlogService
    val actions = new BlogActions(() => dom.window.location.reload())
    Runtime.mount(new BlogPage(service, actions), DomCursor.root(root))
  }
}
```

Der Export macht `render` im JavaScript-Modul aufrufbar. Für **jede Anfrage**
entsteht ein neues Dokument. `Runtime.mount(document, cursor)` ist entscheidend:
Der Callback erhält zwar den Server-Cursor, aber das bloße Erzeugen einer
Komponente mountet ihren Baum noch nicht.

`renderToStringAsync` wartet vor der Serialisierung auf die Arbeit, die die
asynchronen Router-Loader registrieren. Ein synchrones Rendern könnte enden,
bevor der Beitrag geladen ist. Das Promise liefert HTML und einen separaten
HTTP-Status. Die Runtime räumt den gemounteten Baum nach dem Rendern wieder auf.

`boot` verwendet weiterhin `DomCursor`. Das Leeren von `#app` verhindert, dass
unter der Server-Seite eine zweite Seite angehängt wird. Es bedeutet aber auch
einen weiteren Datenabruf und möglicherweise einen sichtbaren Ladezustand.
Das ist die bewusst gesetzte Grenze vor der Hydration.

## Das Dokument um die vorhandene Seite aufbauen

SSR muss ein vollständiges Dokument samt Styles und Browser-Modul zurückgeben.
`BlogDocument` übernimmt diesen äußeren Aufbau; Navigation, Routing und Inhalt
bleiben bei `BlogPage`. Hier ist die vollständige Dokument-Komponente:

```scala
package com.anjunar.blog.frontend

import ui.core.document.{DocumentHead, HeadEntry}
import ui.core.dsl.AttributeDsl.*
import ui.core.dsl.DslLayer.{child, render}
import ui.core.layout.Body.body
import ui.core.layout.Div.div
import ui.core.layout.Head.head
import ui.core.layout.Html
import ui.core.render.Cursor

import scala.concurrent.ExecutionContext

final class BlogDocument(url: String)(using ExecutionContext) extends Html {
  private val documentHead = new DocumentHead
  private val page = new BlogPage(new BlogService, new BlogActions(() => ()), Some(url))

  def responseStatus: Int = page.responseStatus

  override def compose(cursor: Cursor): Unit = {
    DocumentHead.provide(documentHead)(using this)
    documentHead.bind(
      HeadEntry.charset(),
      HeadEntry.meta("viewport", "width=device-width, initial-scale=1"),
      HeadEntry.title("Anjunar Journal"),
      HeadEntry.link("icon", "data:,"),
      HeadEntry("style:main", "link", Seq("rel" -> "stylesheet", "href" -> "/style.css")),
      HeadEntry("style:editor", "link", Seq("rel" -> "stylesheet", "href" -> "/editor.css")),
      HeadEntry.script("client", "/main.js", "type" -> "module")
    )(using this)
    render(this, cursor) {
      lang = if (url.takeWhile(_ != '?').split("/").lift(1).contains("de")) "de" else "en"
      head {}
      body {
        div {
          id = "app"
          child(page) {}
        }
      }
    }
  }
}
```

Der DSL-Baum bleibt zusammenhängend in `compose`. Die vorhandene `BlogPage`
wird als Kind in dasselbe `#app` gemountet, das auch `boot` verwendet. Für den
Artikelinhalt entsteht kein zweites Template auf der JVM.

`DocumentHead` sammelt die Head-Einträge und schreibt sie in `head`. Seine
Schlüssel identifizieren die Einträge: Die beiden Stylesheets brauchen
verschiedene Schlüssel, sonst ersetzt eines das andere. Artikeltitel,
Beschreibungen und Canonical-URLs folgen in Kapitel 24; hier entsteht zunächst
das Dokument, das die Seite darstellen kann.

Das dritte Argument von `BlogPage` ist die Request-URL. Auf dem Server gibt es
keine Browser-Adresse. Die Komponente initialisiert deshalb `I18nRuntime` und
`Router` mit dieser expliziten URL. Eine Anfrage für `/de/posts/example` wählt
Deutsch, bevor etwas geladen oder gerendert wird. Die vorhandene
Beitragskomponente kennzeichnet englischen Fallback-Inhalt weiterhin mit
`lang="en"`.

In [BlogPage](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala)
bleiben Browser-Listener und Änderungen am tatsächlichen Browser-Dokument
durch `cursor.isBrowser` geschützt. Das Rendern von HTML darf keinen
Browser-History-Listener installieren oder auf ein DOM-Element zugreifen.

## Das gemeinsame Modul mit GraalJS ausführen

Die JVM benötigt die Polyglot-API und GraalJS. Beide sind in `build.sbt` auf
25.3.4.1 festgelegt und kommen von Maven Central. Unsere JDK-25-Umgebung kann
sie ohne separate GraalVM-Installation ausführen.

[SsrRenderer](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/backend/src/main/scala/com/anjunar/blog/SsrRenderer.scala)
besitzt einen Worker und eine Warteschlange mit acht Plätzen. Jede Anfrage
bekommt einen frischen JavaScript-`Context`; eine gemeinsame `Engine` und
Modul-`Source` erlauben die Wiederverwendung von Code, ohne den
Komponentenzustand einer Anfrage in die nächste zu übernehmen.

GraalJS ist kein Browser. Unser vorhandenes `HttpJson` braucht `Headers` und
`fetch`, die Scala.js-Runtime benötigt Timer. Ein kleiner
[Host-Adapter](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/backend/src/main/resources/ssr/host.js) stellt diese
Schnittstellen bereit. Seine Fetch-Anbindung akzeptiert ausschließlich
GET-Anfragen an die öffentliche Beitrags-API unter der festgelegten lokalen
Server-Adresse. Sie sendet keine Session-Cookies, folgt keinen Redirects und
kann die Redaktions-API nicht aufrufen.

Die Anbindung verwendet Javas HTTP-Client auf dem Renderer-Worker und liefert
JavaScript ein Promise. REST wählt weiterhin veröffentlichte Datensätze aus,
wendet die Lokalisierung an und verwaltet seine normale Transaktion. Im
Renderer entsteht kein zusätzlicher Datenbankzugriff.

In `SsrRenderer.evaluate` nimmt der folgende Block das Ergebnis entgegen,
nachdem der Host-Adapter eingerichtet und das ES-Modul mit
`exports = context.eval(module)` ausgewertet wurde. Die Imports gehören in
diese Klasse; `RenderedPage` ist die Case Class im selben Package mit
`html: String` und `status: Int`:

```scala
import org.graalvm.polyglot.Value
import org.graalvm.polyglot.proxy.ProxyExecutable

var result: RenderedPage = null
var failure: String = null
val resolved = new ProxyExecutable {
  override def execute(args: Value*): AnyRef = {
    val value = args.head
    val status = value.getMember("status").asInt()
    require(Set(200, 400, 401, 403, 404, 503).contains(status), "Unexpected SSR status")
    result = RenderedPage(value.getMember("html").asString(), status)
    null
  }
}
val rejected = new ProxyExecutable {
  override def execute(args: Value*): AnyRef = { failure = args.head.toString; null }
}
exports.getMember("render").execute(url).invokeMember("then", resolved, rejected)
```

Die Callbacks laufen auf dem Worker, dem der Context gehört. Die anschließende
Schleife verarbeitet registrierte Timer, bis ein Ergebnis oder Fehler vorliegt;
die vollständige Implementierung ist oben verlinkt. Bei jedem Ausgang wird
der Context geschlossen.

Das Zeitlimit des Aufrufers von 15 Sekunden schließt die Wartezeit in der Queue
ein. Für die Antwort-Header einer API-Anfrage gilt ein Timeout von drei Sekunden,
für den Body ein Limit von 2 MiB. Das Lesen des Bodys liegt weiterhin innerhalb
des gesamten Render-Zeitlimits. Beim Abbruch wird auch der Context geschlossen, damit eine festhängende
JavaScript-Berechnung den Worker nicht dauerhaft belegt. Diese bewusst kleine
Runtime verarbeitet jeweils ein Rendering; sie ist kein allgemeiner
Browser-Emulator und kein für hohe Last optimierter Produktionsrenderer.

## HTTP-Status und Seiteninhalt zusammenhalten

Eine Route kann erst nach dem ersten asynchronen API-Abruf scheitern. Deshalb
setzen wir im vorhandenen
[RouterConfig](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogRoutes.scala)
`renderErrorsOnServer = true`. Seine bestehende Fehlerzuordnung wählt die
400-, 404- oder 503-Fehlerroute. `Router.responseStatus` reicht den passenden
Status über `BlogPage.responseStatus` und `BlogDocument.responseStatus` weiter.

[FrontendHandler](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/backend/src/main/scala/com/anjunar/blog/FrontendHandler.scala)
verschiebt öffentliche Seitenanfragen vom Undertow-I/O-Thread auf einen Worker,
wartet auf das Render-Ergebnis und sendet das HTML mit dessen Status. HEAD
rendert ebenso, sendet aber keinen Body. Render-Fehler, fehlende Bundles sowie
Queue- und Zeitlimitfehler ergeben eine sichere 503-Antwort. Die API bleibt
erreichbar.

Die öffentlichen Dokumentantworten verwenden `Cache-Control: no-store`.
Konto- und Redaktionsseiten behalten ihre Browser-Shell samt bestehender
Anmeldung; sie liegen außerhalb dieses anonymen SSR-Pfads. Das Server-Bundle
ist nicht als öffentliches Asset abrufbar.

## Die tatsächliche Antwort prüfen

Dieses Kapitel ergänzt keine Migration. Mit der konfigurierten Datenbank aus
Kapitel 21 bauen und starten wir:

```text
npm ci
sbt --server frontendAssets "application-backend/run"
curl -i "http://127.0.0.1:8080/en?limit=1"
curl -I "http://127.0.0.1:8080/en/posts/no-such-public-post"
```

Die erste Antwort soll die Liste enthalten, die zweite den Status 404 liefern.
Deaktiviere JavaScript und öffne einen veröffentlichten Artikel in beiden
Sprachen. Prüfe Markdown, folge Beitrags- und Seitenlinks und kontrolliere den
englischen Fallback bei einer unveröffentlichten deutschen Übersetzung.
Suchformular, Sprachbuttons und Redaktion brauchen weiterhin JavaScript;
direkte gefilterte oder lokalisierte URLs benötigen es nicht.

Die [Anleitung zum Checkpoint](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/docs/rendering-pages-on-the-server.md) beschreibt
den genauen Testaufbau. `npm run test:browser:ssr` verwendet echte
PostgreSQL-Daten und für die Leseprüfungen einen Browser mit deaktiviertem
JavaScript. Außerdem prüft es maskierte Titel, Fehlerstatus, HEAD und das
erneute Mounten im Browser. Die vorhandenen Browser-Vertragstests deaktivieren
SSR ausdrücklich, weil sie Datenanfragen im Browser abfangen. Allein können
diese Tests das Server-Rendering daher nicht nachweisen.

Starte nach einem Frontend-Neubau den Backend-Prozess neu: Seine Modulquelle
bleibt für die Lebensdauer des Renderers gecacht. Server- und Browser-Bundle
müssen zusammenpassen.

Leser erhalten jetzt die vollständige öffentliche Seite mit der ersten Antwort.
Kapitel 23 erhält diesen DOM und die Ausgangsdaten beim Start der Interaktivität.
