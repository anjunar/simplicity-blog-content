Das Journal kann eine Liste darstellen, und das Backend kann eine zurückgeben. Dieses Kapitel verbindet sie.

Ein Besucher öffnet den Blog, sieht veröffentlichte PostgreSQL-Posts, folgt einem Titel zum vollständigen Artikel und kehrt mit dem Browser-Button Zurück zurück. Die gleiche Seite muss auch funktionieren, wenn die Datenbank leer ist, eine Anforderung fehlschlägt oder der Besucher geht, bevor eine Antwort eintrifft.

Beginnen Sie mit dem [Kapitel 9 Quelle](https://github.com/anjunar/anjunar-blog-example/tree/b7c8f7674c759a27432925353a7b2090038b8c5b). Der [vollständige Stand von Kapitel 10](https://github.com/anjunar/anjunar-blog-example/tree/db992619d48e8938308216641856849e8be9bd97) enthält jede unten diskutierte Datei. Pfade sind relativ zum Repository-Verzeichnis.

## Hinzufügen von JSON Mapping und Routing

Das Frontend verwendet bereits Scala.js UI Core. Ersetzen Sie in den Einstellungen des Frontend-Projekts in `build.sbt` die bisherige einzelne Bibliotheksabhängigkeit durch:

```scala
libraryDependencies ++= Seq(
  "com.anjunar" %% "scalajs-ui-core" % "1.0.9",
  "com.anjunar" %% "scalajs-ui-json" % "1.0.9",
  "com.anjunar" %% "scalajs-ui-router" % "1.0.9",
  "org.scalatest" %% "scalatest" % "3.2.20" % Test
),
```

Dieses Projekt verwendet sbt 2, wobei `%%` die plattformspezifischen Scala.js-Artefakte für das Frontend-Projekt auswählt. Alle vier Abhängigkeiten kommen von Maven Central.

Aktualisieren Sie die Abhängigkeitsauflösung des Frontends:

```text
sbt --server "application-frontend/update"
```

Entfernen Sie `PostPreview.scala`, einschließlich der festen Beispielbeiträge. Die Anwendung zeigt nun das Ergebnis der API an, einschließlich einer leeren Liste, wenn nichts veröffentlicht wurde.

## Spiegelung der veröffentlichten Entität

Das Frontend benötigt ein Scala.js-Modell. Es kann nicht die JVM-Klasse mit Hibernate- und JPA-Annotationen verwenden, aber es sollte seine Feldnamen und Bedeutung beibehalten.

Erstellen Sie `application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPost.scala`:

```scala
package com.anjunar.blog.frontend

import ui.core.state.Property
import ui.json.JsonId

final class BlogPost {
  @JsonId
  val id: Property[String] = Property("")
  val version: Property[Long] = Property(-1L)
  val slug: Property[String] = Property("")
  val title: Property[String] = Property("")
  // The list graph omits content. Only the detail request supplies it.
  val content: Property[Option[String]] = Property(None)
  val summary: Property[Option[String]] = Property(None)
  val status: Property[String] = Property("DRAFT")
  val publishedAt: Property[Option[String]] = Property(None)
}

final class BlogPostData(var data: BlogPost = null)

final class BlogPostTable(
    var rows: Seq[BlogPostData] = Seq.empty,
    var size: Long = -1L
)
```

Es gibt acht Entity-Felder, genau wie im Backend. UUIDs und ISO-8601-Zeitstempel werden als Zeichenfolgen verwendet; der Veröffentlichungsstatus ist der Name des Enums. Die optimistische Version bleibt ein `Long`. Eine persistierte Entität kann Versionsnummer 0 haben, daher darf `0` niemals "fehlen" bedeuten.

Der Listengraph lässt `content` aus. Wir stellen dieses fehlende Feld mit `None` dar; die Detailanforderung liefert `Some(text)`. Optionale Zusammenfassungen und Publikationszeitstempel verwenden die gleiche Darstellung. Eine Listenzeile ist daher kein vollständiges Modell für ein Update. Dieses Kapitel liest nur Daten.

Jede Zeile behält die REST-Hülle `data`. Die Tabelle hält `rows` und `size`, wobei die Größe die Gesamtzahl der veröffentlichten Beiträge vor der Paginierung ist. Unser Backend lässt leere Sammlungen aus, so dass eine leere Antwort sein kann:

```json
{"size":0,"@type":"Table"}
```

Der leere Standard für `rows` ist absichtlich. Wenn der Besucher einen Offset über den letzten Beitrag hinaus anfordert, können Zeilen auch fehlen, während die Größe positiv bleibt.

Der `-1`-Standard für Größe ermöglicht es dem Dienst, eine Antwort ohne Gesamtmenge abzulehnen. Die Datenhülle beginnt mit `null`, so dass ein fehlendes `data`-Mitglied nicht stillschweigend zu einem plausiblen leeren Beitrag werden kann.

### Zwei Schemata, zwei Jobs

`EntitySchema` beschreibt Entity-Felder, Mapper-Regeln und Kriterien-Attribute. Das `ui.json.JsonSchema` des Browsers wird aus diesen lokalen Scala.js-Klassen abgeleitet und liefert Konstruktoren und Property-Accessors für seinen Mapper.

Das strukturelle `schema`-Objekt in einer REST-Antwort generiert keine Frontend-Klassen oder erteilt keine Berechtigung zum Schreiben. Die Leseransicht verwendet diese Metadaten nicht. Die konkreten Frontend-Modelle tolerieren auch die `@type`-Marker des Backends.

Die Modelle werden zusammen mit dem REST-Vertrag gepflegt. Das Hinzufügen eines Backend-Feldes fügt nicht automatisch eine Browsereigenschaft hinzu.

## HTTP-Zugriffe bündeln

Erstellen Sie `HttpJson.scala` neben dem Modell:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom
import ui.json.{JsonMapper, JsonSchema}

import scala.concurrent.{ExecutionContext, Future}
import scala.scalajs.js
import scala.scalajs.js.JSConverters.*

final class HttpFailure(val status: Int) extends RuntimeException(s"HTTP $status")

object HttpJson {
  def get[M](path: String, signal: Option[dom.AbortSignal])(using
      ExecutionContext, JsonSchema[M]
  ): Future[M] = {
    val headers = new dom.Headers()
    headers.set("Accept", "application/json")
    val options = new dom.RequestInit {
      method = dom.HttpMethod.GET
      credentials = dom.RequestCredentials.`same-origin`
    }
    options.headers = headers
    signal.foreach(value => options.signal = value)

    dom.fetch(path, options).toFuture.flatMap { response =>
      // fetch resolves for HTTP errors too. Do not map an error body as a post.
      if (!response.ok) Future.failed(new HttpFailure(response.status))
      else response.text().toFuture.map { body =>
        JsonMapper.deserialize[M](js.JSON.parse(body))
      }
    }
  }
}
```

Die Methode akzeptiert einen Zielmodelltyp und ein optionales Abbruchsignal. Scala leitet das benötigte `JsonSchema[M]` am konkreten Einsatzort ab. Nach dem Lesen des Bodys konstruiert `JsonMapper` das Modell und füllt seine Eigenschaften aus; Komponenten gehen niemals selbst mit `js.Dynamic`-Objekten.

Überprüfen Sie den Status vor dem Mapping. Fetch löst auch sein Versprechen für HTTP-Fehler wie 404, so dass ein erfolgreiches Versprechen allein keine erfolgreiche Anfrage bedeutet. Durch das Übergeben eines Abbruchsignals kann die Navigation eine Anfrage abbrechen, die nicht mehr benötigt wird. [MDN erklärt beide Verhaltensweisen](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch).

Wir verwenden relative URLs mit dem gleichen Ursprung wie die Seite. Undertow bedient bereits sowohl das Frontend als auch `/service`, so dass dieser Schritt keine zweite Server- oder CORS-Konfiguration benötigt. Der Helfer behält den Status erfolgloser Antworten, ohne deren Body Besuchern anzuzeigen. Strukturierte Validierungsprobleme werden relevant, wenn wir Schreibvorgänge einführen.

## Die API-Operationen benennen

Erstellen Sie `BlogService.scala`:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom

import scala.concurrent.{ExecutionContext, Future}
import scala.scalajs.js.URIUtils.encodeURIComponent

final class BlogService(using ExecutionContext) {
  val pageSize = 20

  def list(offset: Int, signal: Option[dom.AbortSignal]): Future[BlogPostTable] =
    HttpJson.get[BlogPostTable](s"/service/blog/posts?offset=$offset&limit=$pageSize", signal)
      .map { table =>
        require(table.size >= 0 && table.rows != null, "Invalid post table")
        table.rows.foreach(row => require(row != null && row.data != null, "Missing post data"))
        table
      }

  def detail(slug: String, signal: Option[dom.AbortSignal]): Future[BlogPost] =
    HttpJson.get[BlogPostData](s"/service/blog/posts/${encodeURIComponent(slug)}", signal)
      .map { result =>
        require(result.data != null && result.data.content.get.nonEmpty, "Missing post detail")
        result.data
      }
}
```

Der Dienst besitzt Endpunktpfade, Antworttypen und die Seitengröße. Die Detailmethode gibt die Entität innerhalb ihres Umschlags zurück. Er prüft außerdem, ob der Detailgraph tatsächlich Inhalt geliefert hat.

Ein Slug wird als URL-Segment codiert. Keine Komponente muss sich an das REST-Präfix erinnern oder wie die Antwort entpackt wird.

Der Server sortiert Artikel bereits absteigend nach Veröffentlichungszeit und bei Gleichstand nach ID. Diese Reihenfolge behalten wir bei. Die lokale Sortierschaltfläche des Kapitels 9 wird entfernt: Zwanzig geladene Zeilen umzukehren würde nur eine Seite umkehren, nicht das gesamte Ergebnis.

## Die Route verwaltet ihre Anfrage

Erstellen Sie `BlogRoutes.scala`:

```scala
package com.anjunar.blog.frontend

import ui.router.{Route, RouteFailure, RouterConfig}

import scala.concurrent.{ExecutionContext, Future}

final class BlogRoutes(service: BlogService, actions: BlogActions)(using ExecutionContext) {
  val routes: Seq[Route] = Seq(
    Route.view("/") { context =>
      val offset = context.queryParams.get("offset").getOrElse("0").toIntOption
        .filter(_ >= 0).getOrElse(throw new HttpFailure(400))
      service.list(offset, context.signal)
        .map(table => new PostListPage(table, offset, service.pageSize, actions))
    },
    Route.view("/posts/:slug") { context =>
      service.detail(context.pathParams("slug"), context.signal).map(new PostPage(_))
    },
    Route.error("/bad-request", status = 400) { _ =>
      Future.successful(new ErrorPage(400, actions))
    },
    Route.error("/not-found", status = 404) { _ =>
      Future.successful(new ErrorPage(404, actions))
    },
    Route.error("/unavailable", status = 503) { _ =>
      Future.successful(new ErrorPage(503, actions))
    }
  )

  val config: RouterConfig = RouterConfig(
    loading = _ => new LoadingPage,
    onFailure = {
      case _: RouteFailure.NotMatched => Some("/not-found")
      case RouteFailure.LoadFailed(error: HttpFailure, _) if error.status == 404 =>
        Some("/not-found")
      case RouteFailure.LoadFailed(error: HttpFailure, _) if error.status == 400 =>
        Some("/bad-request")
      case _ => Some("/unavailable")
    }
  )
}
```

Ein Routenlader gibt eine zukünftige Komponente zurück. Die Listenroute liest einen nicht negativen Ganzzahl-Offset von der URL, fordert eine Seite an und konstruiert ein `PostListPage`. Die Detailroute fordert den vollständigen Artikel anhand seines Slugs an.

Die Ladekomponente ist sichtbar, solange das Future noch offen ist. Ein 404 wählt die Seite mit fehlendem Beitrag aus; ein ungültiger Offset wählt die Antwort mit ungültiger Seite aus. HTTP-Ausfälle, verlorene Verbindungen und ungültiges JSON erreichen die nicht verfügbare Seite. Eine Fehlerroute ist eine interne Weiterleitung: Die vom Besucher angeforderte Adresse bleibt in der URL-Leiste.

Beachten Sie den Pfad von `context.signal`: route → service → HTTP helper → fetch. Wenn eine andere Navigation beginnt, bricht der Router die vorherige Anforderung ab. Router 1.0.9 überprüft auch ein Render-Token, bevor ein asynchrones Ergebnis installiert wird. Diese zweite Überprüfung verhindert, dass eine alte Fertigstellung die neue Seite ersetzt, selbst wenn der Abschluss zeitgleich mit dem Abbruch erfolgt.

Halten Sie die Anforderung im Loader. Das Starten einer weiteren Anforderung in `compose` würde einen zweiten Eigentümer für denselben Vorgang erstellen.

### UI-Aktionen außerhalb der Entität behalten

Erstellen Sie `BlogActions.scala`:

```scala
package com.anjunar.blog.frontend

import ui.core.state.Property

final class BlogActions(reloadPage: () => Unit) {
  val showSummaries: Property[Boolean] = Property(true)

  def toggleSummaries(): Unit = showSummaries.set(!showSummaries.get)

  // A retry starts a fresh document request at the same URL.
  def retry(): Unit = reloadPage()
}
```

Die Einstellung für Zusammenfassungen ist UI-Zustand der Anwendung. Sie gehört nicht in `BlogPost`, wo sie mit einem Teil des Entitätsvertrags verwechselt werden könnte.

Eine freigegebene Aktionsinstanz überlebt die Listen-/Detailnavigation. Die Zusammenfassungsschaltfläche bindet `ariaPressed` an `actions.showSummaries`, und der Klick-Handler ruft `toggleSummaries()` auf. Der bestehende `when`-Block beobachtet die gleiche Eigenschaft.

Retry lädt das aktuelle Dokument absichtlich neu. Router 1.0.9 hat keinen öffentlichen Nachladevorgang, und das Navigieren in denselben Routenzustand startet keine weitere Ladung. Das Neuladen behält den Pfad und die Abfrage bei, wiederholt die Anforderung und setzt die In-Memory-Zusammenfassungseinstellung zurück.

## Site-Shell und Seitenbäume zusammenhalten

`BlogPage` besitzt jetzt den persistenten Site-Header, den Main-Landmark, die Fußzeile, die englische Übersetzungslaufzeit und den Router. Jede Route besitzt ihre Seitenkomponente. Dies sind sinnvolle Komponentengrenzen: Das Öffnen eines Artikels ersetzt die Liste, während die Site-Shell montiert bleibt.

Das Setup zu Beginn von `BlogPage.compose` verwendet diese Importe:

```scala
import ui.core.i18n.I18nRuntime
import ui.router.Router
```

Innerhalb dieser Methode, vor dem bestehenden `render(this, cursor)`-Block:

```scala
I18nRuntime.provide(translations)(using this)
val router = new Router(pages.routes, cursor.browserUrl.getOrElse("/"), pages.config)
Router.provide(router)(using this)
```

Hier ist `pages` die `BlogRoutes`-Instanz der Komponente. Die Bereitstellung des Routers auf der Shell stellt ihn sowohl Header-Links als auch dem Seitenunterbaum zur Verfügung. Innerhalb des Main-Landmarks montiert `child(router) {}` den Routenausgang. Siehe [komplette BlogPage](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala) für den umliegenden Baum und Importe.

Die bestehende englische Laufzeit macht Router-Links sprachabhängig. Die Liste lebt bei `/en` und ein Artikel bei `/en/posts/:slug`. Die Wurzel `/` bleibt ein Einstiegspunkt. Routendefinitionen bleiben lokal unabhängig, während API-Pfade unter `/service` bleiben. Deutsche Nachrichten und Sprachauswahl sind in Kapitel 18 enthalten.

### Listenzeilen und Details rendern

[PostListPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostListPage.scala) behält die Einleitung und Liste aus Kapitel 9 bei. Es initialisiert seine `ListProperty` von `table.rows.map(_.data)`. Das `foreach` des DSL macht die Zeilen, und jeder Titel wird zu einem `routerLink` für den Slug des Posts.

Es fügt auch drei Verhaltensweisen hinzu:

- Bei Gesamtzahl 0 erscheint "No posts have been published yet". Eine leere Seite mit einer positiven Gesamtanzeige zeigt "Es gibt keine Beiträge auf dieser Seite."
- Eine fehlende Zusammenfassung erzeugt keinen zusammenfassenden Absatz. Das Ein- und Ausblenden von Zusammenfassungen bleibt reaktiv.
- Neuere / ältere Links tragen Offset in der URL und fordern zwanzig Beiträge pro Seite an. Der Browserverlauf stellt die Abfrage wieder her; der Server wählt weiterhin die Zeilenreihenfolge.

Die Seitenlinks zeigen die bestehende Paginierung der API. Kapitel 15 wird das Such- und Sortiermodell erweitern.

Das Detail ist eine separate Komponente. Erstellen Sie `PostPage.scala`:

```scala
package com.anjunar.blog.frontend

import ui.core.component.AbstractComponent
import ui.core.dsl.AttributeDsl.*
import ui.core.dsl.ClassDsl.classes
import ui.core.dsl.DslLayer.render
import ui.core.i18n.i18n
import ui.core.layout.Div.div
import ui.core.layout.Heading.heading
import ui.core.layout.Paragraph.paragraph
import ui.core.layout.TextComponent.text
import ui.core.render.Cursor
import ui.router.RouterLink.routerLink

final class PostPage(post: BlogPost) extends AbstractComponent {
  val tagName = "article"

  override def compose(cursor: Cursor): Unit =
    render(this, cursor) {
      classes = "post-detail"
      ariaLabelledBy = "post-title"
      routerLink("/") { text(i18n"Back to latest posts") {} }
      paragraph {
        classes = "post-date"
        text(post.publishedAt.map(_.fold("")(_.take(10)))) {}
      }
      heading(1) { id = "post-title"; text(post.title) {} }
      if (post.summary.get.nonEmpty) {
        paragraph { classes = "detail-summary"; text(post.summary.map(_.getOrElse(""))) {} }
      }
      div {
        classes = "post-content"
        // Chapter 17 introduces structured content. Today content is plain text.
        text(post.content.map(_.getOrElse(""))) {}
      }
    }
}
```

Der Inhalt durchläuft eine Textbindung. Text, der wie ein Skript oder HTML-Element aussieht, wird als Text angezeigt. Die `post-content` CSS-Regel verwendet `white-space: pre-wrap`, um Zeilenumbrüche und `overflow-wrap: anywhere` für lange Inhalte zu erhalten. Rich Content Rendering kommt mit dem Editor in Kapitel 17.

Das komplette [LoadingPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/frontend/src/main/scala/com/anjunar/blog/frontend/LoadingPage.scala) verwendet eine Statusregion. [ErrorPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/frontend/src/main/scala/com/anjunar/blog/frontend/ErrorPage.scala) verwendet eine Alarmregion, eine lesbare Nachricht und eine Wiederherstellungsaktion. Jede Komponente behält ihren gesamten UI-Baum in `compose`; keine startet eine Datenanforderung.

UI-Labels verwenden das `i18n`-Makro. Posttitel, Zusammenfassungen und Inhalte sind redaktionelle Daten und bleiben außerhalb des Nachrichtenkatalogs.

Aktualisieren Sie schließlich `Main.scala`, um den freigegebenen Dienst und die Aktionen zu erstellen:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom
import ui.core.component.Runtime
import ui.core.render.DomCursor

import scala.concurrent.ExecutionContext.Implicits.global

object Main {
  def main(args: Array[String]): Unit = {
    val root = dom.document.getElementById("app")
    require(root != null, "The page must contain an element with id='app'")
    val service = new BlogService
    val actions = new BlogActions(() => dom.window.location.reload())
    Runtime.mount(new BlogPage(service, actions), DomCursor.root(root))
  }
}
```

## Direkte Artikel-URLs ermöglichen

Client-Navigation ist nur die Hälfte des Jobs. Das erneute Laden von `/en/posts/our-first-public-post` sendet diesen Pfad an Undertow, bevor ein Scala.js-Code ausgeführt wird.

Aktualisieren Sie [FrontendHandler.scala](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/backend/src/main/scala/com/anjunar/blog/FrontendHandler.scala), um `/en`, `/en/` und `/en/posts/{slug}` zu erkennen, wobei Sie die gleiche Kleinbuchstaben-, Bindestrich-getrennte Slug-Form wie die Entität verwenden. Für diese GET/HEAD-Anfragen dient es `index.html`. Der `/index.html`-Einstiegspunkt wird zu `/` weitergeleitet, während die Abfrage beibehalten wird. Bestehende statische Assets behalten ihren Weg und `/service` bleibt bei RESTEasy.

Dies ist eine begrenzte Seite Fallback. Unbekannte Assets und nicht verwandte Pfade geben immer noch 404 zurück. POST zu einem Seitenpfad gibt 405 zurück. API-Startup funktioniert noch, bevor Frontend-Assets erstellt wurden.

Hier gibt es eine wichtige Grenze: Das HTML-Dokument für eine wohlgeformte, aber unbekannten Slug gibt 200 zurück. Der Browser ruft dann die API auf, erhält 404 und rendert "Post not found". Eine Client-Fehlerroute kann eine bereits gesendete Dokumentenantwort nicht ändern. Kapitel 19 lädt die Route auf den Server und gibt dem Dokument seinen korrekten Status.

## Den vollständigen Ablauf ausführen

Verwenden Sie die Datenbankeinstellung aus Kapitel 6, wobei `BLOG_DB_URL`, `BLOG_DB_USER` und `BLOG_DB_PASSWORD` für Ihre lokale Datenbank festgelegt sind. Starten und migrieren Sie es, bevor Sie die Anwendung ausführen:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

Dieses Kapitel ändert keine Datenbankfelder oder Schema-IDs.

Laden Sie mit der Compose-Datenbank des Tutorials die optionalen Beispiele aus Kapitel 8:

```text
docker compose cp database/examples/public-posts.sql postgres:/tmp/public-posts.sql
docker compose exec -T postgres psql -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --file /tmp/public-posts.sql
```

Verwenden Sie für native PostgreSQL die Verbindungsdetails für Ihre Tutorial-Datenbank:

```text
psql -h 127.0.0.1 -p 5433 -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --file database/examples/public-posts.sql
```

Das Skript fügt einen veröffentlichten Beitrag und einen Entwurf ein. Wiederholtes Ausführen erhält diese Zeilen. Es sind optionale Beispieldaten, keine Migration oder eine automatische Startaufgabe. Der native Befehl psql fordert sein Passwort separat auf.

Bauen und starten Sie im Repository-Verzeichnis:

```text
sbt --server frontendAssets "application-backend/run"
```

Öffnen Sie `http://127.0.0.1:8080/`. Wählen Sie **Unser erster öffentlicher Beitrag**, laden Sie die Detail-URL neu und verwenden Sie Zurück und Vorwärts. Der vollständige Inhalt erscheint nur auf der Detailseite. Das Öffnen von `/en/posts/our-private-draft` zeigt die gleiche fehlende Postseite wie eine unbekannten Slug.

Eine leere migrierte Datenbank zeigt den leeren Zustand an. Ein Datenbankverbindungsfehler zeigt den nicht verfügbaren Zustand an. Die Anwendung ersetzt keine lokalen Preview-Posts mehr.

## Überprüfen Sie die Grenze

Die JSON-Tests laufen in Scala.js:

```text
sbt --server "application-frontend/testFull"
```

Die sechs Prüfungen umfassen die Entitätsfelder, Versionsnummer 0, fehlende Inhalte, optionale Nullen, ausgelassene Zeilen und Gesamtwerte auf leeren Seiten. Sie verifizieren auch, dass Server-Metadaten die konkreten Browsermodelle nicht beeinträchtigen.

Installieren Sie die Browsertestabhängigkeiten und führen Sie die kontrollierten Antwortszenarien aus:

```text
npm ci
npx playwright install chromium
npm run test:browser
```

Diese elf Prüfungen führen das kompilierte Frontend gegen den eigentlichen HTTP-Server aus, fangen jedoch Datenanforderungen ab. Damit können sie eine Antwort aussetzen, einen Netzwerkfehler erzeugen, fehlerhaftes JSON zurückgeben und die Stornierung überprüfen, ohne vom Timing in einer Datenbank abhängig zu sein. Sie umfassen auch Geschichte, Paging, Text-Escaping, Tastatursteuerung, mobiles Layout und statische Routenstatuscodes.

Verwenden Sie dann eine dedizierte Testdatenbank, die zum aktuellen Schema migriert und mit den unveränderten SQL-Beispielen befüllt wurde:

```text
sbt --server "application-backend/testFull"
npm run test:browser:database
```

Die 42 Backend-Tests bestehen weiterhin. Die beiden zusätzlichen Browsertests verwenden echte PostgreSQL-Daten ohne Mocking: öffentliche Liste → Details → Neuladen, gefolgt von Behandlung von Entwürfen und unbekannten Artikeln. Halten Sie den öffentlichen Beispielbeitrag auf der ersten Seite dieser Testdatenbank. Die Datenbankbrowsertests fügen keine Zeilen hinzu oder löschen sie.

Beide Browserprojekte starten ihren eigenen Server am Port 18080 und stoppen ihn danach. Halten Sie diesen Port frei. Screenshots werden in `test-results` gespeichert.

Wir haben jetzt einen vollständigen öffentlichen Lesepfad, von einer Datenbankzeile bis zu einem navigierbaren Artikel. Als nächstes fügen wir Benutzerkonten und die Anmeldung hinzu, was uns eine Identität gibt, die wir verwenden können, wenn wir Berechtigungen und Bearbeitungen einführen.
