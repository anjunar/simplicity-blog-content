Wir haben eine öffentliche API. Jetzt brauchen wir eine Seite, die jemand lesen kann.

Dieses Kapitel fügt die erste Scala.js-Oberfläche hinzu: ein Journal mit drei Post-Vorschauen, einem Zusammenfassungsschalter und einer Schaltfläche, die ihre Reihenfolge ändert. Die ganze Seite entsteht mit der Komponenten-DSL von Scala.js UI, und der bestehende Undertow-Server liefert sie unter `/`.

Wir beginnen mit lokalen Beispieldaten, damit das Komponentenmodell, der Zustand und das Rendern sichtbar bleiben. Kapitel 10 verbindet die Seite mit der API und fügt Navigation, Laden und Fehlerbehandlung hinzu.

Beginnen Sie mit dem [Kapitel 8 Quelle](https://github.com/anjunar/anjunar-blog-example/tree/54fac6042b34be4c8d7ab9b3f4ed054272220b81). Alle Pfade unten sind relativ zum Repository-Verzeichnis.

## Hinzufügen eines Browsermoduls

Das Backend läuft auf der JVM. Das neue Frontend kompiliert in JavaScript und läuft im Browser. Sie gehören zum gleichen Build, aber das Frontend hängt nicht von den Hibernate-Entitätsklassen ab.

Erstellen Sie `project/plugins.sbt`:

```scala
addSbtPlugin("org.scala-js" % "sbt-scalajs" % "1.22.0")
```

Fügen Sie diese Importe oben auf `build.sbt` hinzu:

```scala
import org.scalajs.linker.interface.ModuleKind
import org.scalajs.sbtplugin.ScalaJSPlugin
```

Fügen Sie das Frontend-Projekt und die Asset-Aufgabe hinzu:

```scala
lazy val frontend = Project("application-frontend", file("application/frontend"))
  .enablePlugins(ScalaJSPlugin)
  .settings(
    libraryDependencies += "com.anjunar" %% "scalajs-ui-core" % "1.0.9",
    scalaJSUseMainModuleInitializer := true,
    scalaJSLinkerConfig := scalaJSLinkerConfig.value.withModuleKind(ModuleKind.ESModule)
  )

lazy val frontendAssets = taskKey[File]("Build and copy the browser application")
frontendAssets / aggregate := false

frontendAssets := Def.uncached {
  val _ = (frontend / Compile / fastLinkJS).value
  val linked = (frontend / Compile / fastLinkJS / scalaJSLinkerOutputDirectory).value
  val destination = (LocalRootProject / baseDirectory).value / "target" / "frontend"
  IO.createDirectory(destination)
  IO.copyDirectory(linked, destination)
  IO.copyDirectory((frontend / Compile / resourceDirectory).value, destination)
  destination
}
```

Fügen Sie das Frontend in die Aggregation des bestehenden Root-Projekts ein:

```scala
lazy val root = Project("anjunar-blog-tutorial", file("."))
  .aggregate(backend, frontend)
  .settings(publish / skip := true)
```

Die bestehende Scala 3.9.0-Einstellung und der Maven Central Resolver bleiben bestehen. In diesem sbt 2-Build wählt `%%` das Scala.js-Artefakt `scalajs-ui-core_sjs1_3` aus. Version 1.0.9 kommt von Maven Central; Leser brauchen weder einen Checkout noch eine lokale Veröffentlichung des UI-Frameworks.

`scalaJSUseMainModuleInitializer` lässt das verknüpfte Modul unsere `main`-Methode aufrufen, wenn der Browser es lädt. `ModuleKind.ESModule` entspricht dem `type="module"`-Skript in der HTML-Shell. Der [Scala.js-Modulführer](https://www.scala-js.org/doc/project/module.html) beschreibt dieses Modulformat.

`frontendAssets` führt den Entwicklungslinker aus und kopiert die verknüpften Dateien und Ressourcen in `target/frontend`. Die Aufgabe fragt sbt nach dem Ausgabeverzeichnis des Linkers, anstatt einen versionsspezifischen Zielpfad fest zu codieren. `Def.uncached` lässt den Kopierschritt jedes Mal laufen; sbt kann unveränderte Kompilierungs- und Verknüpfungsergebnisse immer noch wiederverwenden.

Dateien unter `application/frontend` bearbeiten. Dateien unter `target/frontend` sind generierte Ausgaben. Wir werden optimierte Deployment-Pakete in einem eigenen Kapitel vorstellen.

## Bereitstellen eines kleinen HTML-Hosts

Erstellen Sie `application/frontend/src/main/resources/index.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="description" content="Notes on Scala, the web, and the decisions in between.">
  <title>Anjunar Journal</title>
  <link rel="icon" href="data:,">
  <link rel="stylesheet" href="/style.css">
  <script type="module" src="/main.js"></script>
</head>
<body>
  <div id="app"></div>
  <noscript><p class="no-script">Enable JavaScript to read this preview.</p></noscript>
</body>
</html>
```

Das `app`-Element beginnt leer. Der Browser lädt das Modul und montiert unseren Komponentenbaum in dieses Element. Dies ist Browser-Rendering; Server-Rendering und Hydratation kommen später.

Das Dokument erklärt seine englische Sprache, Zeichenkodierung und Viewport. Sein statischer Titel und seine Beschreibung reichen für diese erste Seite aus. Die `noscript`-Nachricht erklärt, warum die Vorschau nicht verfügbar ist, wenn JavaScript deaktiviert ist.

## Beginnen Sie mit expliziten Beispieldaten

Erstellen Sie `application/frontend/src/main/scala/com/anjunar/blog/frontend/PostPreview.scala`:

```scala
package com.anjunar.blog.frontend

// A read-only preview for the first UI; the API model follows in chapter 10.
final case class PostPreview(
    id: String,
    slug: String,
    title: String,
    summary: String,
    publishedAt: String
)

object ExamplePosts {
  val all: Seq[PostPreview] = Seq(
    PostPreview(
      "b62db12a-61a7-46a5-b8d2-c487f80e825a",
      "serving-posts-through-rest",
      "Serving posts through REST",
      "Give readers a small, predictable API. Start with published posts, a useful list, and the full story behind each title.",
      "2026-09-27T10:15:42Z"
    ),
    PostPreview(
      "437e408e-c027-4edf-a5da-0d9156c47965",
      "one-model-two-jobs",
      "One model, two jobs",
      "Use the same field description for JSON mapping and typed database queries. Less duplication, with a contract you can test.",
      "2026-09-26T09:00:00Z"
    ),
    PostPreview(
      "35d433b8-7b67-437b-bfe3-389aa38e18f8",
      "room-for-the-model-to-grow",
      "Room for the model to grow",
      "A schema changes as an application takes shape. Add a summary, keep existing posts, and make the next migration repeatable.",
      "2026-09-25T08:00:00Z"
    )
  )
}
```

`PostPreview` ist eine kleine schreibgeschützte Projektion mit Feldnamen, die mit den entsprechenden Postfeldern übereinstimmen. Seine Strings sind lokale redaktionelle Beispiele. Es ist kein Deserialisierungsmodell für die REST-Antwort.

Die Trennung der Beispiele von der Komponente erleichtert den nächsten Schritt: In Kapitel 10 werden das Frontend-JSON-Modell und der HTTP-Dienst vorgestellt, während der Kompositionsansatz der Seite beibehalten wird. Das Ändern einer Datenbankzeile ändert diese Vorschau noch nicht.

## Erstellen Sie einen zusammenhängenden Komponentenbaum

Erstellen Sie `application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala`:

```scala
package com.anjunar.blog.frontend

import ui.core.component.AbstractComponent
import ui.core.dsl.AttributeDsl.*
import ui.core.dsl.ClassDsl.classes
import ui.core.dsl.DslLayer.{child, render}
import ui.core.dsl.EventDsl.onClick
import ui.core.i18n.{I18nConfig, I18nLocale, I18nResolver, I18nRuntime, MessageCatalog, i18n}
import ui.core.layout.Anchor.{anchor, href}
import ui.core.layout.Article
import ui.core.layout.Button.{button, buttonType}
import ui.core.layout.Condition.when
import ui.core.layout.Div.div
import ui.core.layout.Footer.footer
import ui.core.layout.Header.header
import ui.core.layout.Heading.heading
import ui.core.layout.Li.li
import ui.core.layout.Main.main
import ui.core.layout.Nav.nav
import ui.core.layout.Paragraph.paragraph
import ui.core.layout.Section.section
import ui.core.layout.Span.span
import ui.core.layout.TextComponent.text
import ui.core.layout.Ul.ul
import ui.core.render.Cursor
import ui.core.state.{ListProperty, Property}
import ui.core.statement.Foreach.foreach

import scala.scalajs.js

final class BlogPage extends AbstractComponent {
  val tagName = "div"

  private val showSummaries = Property(true)
  private val newestFirst = Property(true)
  private val posts = ListProperty(js.Array[PostPreview]())
  private val translations = I18nRuntime.managed(I18nConfig(
    resolver = new I18nResolver(MessageCatalog.empty),
    supportedLocales = Seq(I18nLocale.En),
    defaultLocale = I18nLocale.En
  ))

  override def compose(cursor: Cursor): Unit = {
    I18nRuntime.provide(translations)(using this)
    addDisposable(newestFirst.observe { newest =>
      val ordered = ExamplePosts.all.sortBy { post =>
        val timestamp = js.Date.parse(post.publishedAt)
        (if (newest) -timestamp else timestamp, post.id)
      }
      posts.setAll(ordered)
    })

    render(this, cursor) {
      classes = "blog"
      anchor() {
        classes = "skip-link"
        href = "#main-content"
        text(i18n"Skip to content") {}
      }
      header {
        classes = "site-header"
        anchor() {
          classes = "brand"
          href = "/"
          text("Anjunar") {}
          span { classes = "brand-note"; text(i18n"Journal") {} }
        }
        nav {
          ariaLabel = translations.text(i18n"Main navigation")
          anchor() { href = "#posts"; text(i18n"Latest posts") {} }
          anchor() {
            href = "https://github.com/anjunar/anjunar-blog-example"
            text(i18n"Source code") {}
          }
        }
      }
      main {
        id = "main-content"
        tabIndex = -1
        section {
          classes = "introduction"
          ariaLabelledBy = "page-title"
          paragraph { classes = "eyebrow"; text(i18n"The Anjunar journal") {} }
          heading(1) {
            id = "page-title"
            text(i18n"From idea to working software.") {}
          }
          paragraph {
            classes = "introduction-copy"
            text(i18n"Notes on Scala, the web, and the decisions in between.") {}
          }
        }
        section {
          id = "posts"
          ariaLabelledBy = "posts-title"
          div {
            classes = "list-heading"
            div {
              heading(2) { id = "posts-title"; text(i18n"Latest posts") {} }
              paragraph { classes = "list-note"; text(i18n"Three notes from the build.") {} }
            }
            div {
              classes = "list-controls"
              button(i18n"Show summaries") {
                buttonType("button")
                ariaPressed = showSummaries
                ariaControls = "post-list"
                onClick(_ => showSummaries.set(!showSummaries.get))
              }
              button(newestFirst.flatMap(newest =>
                translations.text(if (newest) i18n"Newest first" else i18n"Oldest first"))) {
                buttonType("button")
                ariaLabel = newestFirst.flatMap(newest =>
                  translations.text(if (newest) i18n"Sort oldest first" else i18n"Sort newest first"))
                ariaControls = "post-list"
                onClick(_ => newestFirst.set(!newestFirst.get))
              }
            }
          }
          ul {
            id = "post-list"
            classes = "post-list"
            role = "list"
            foreach(posts) { post =>
              li {
                child(new Article) {
                  classes = "post"
                  ariaLabelledBy = s"post-${post.slug}"
                  paragraph {
                    classes = "post-date"
                    text(post.publishedAt.take(10)) {}
                  }
                  div {
                    classes = "post-copy"
                    heading(3) {
                      id = s"post-${post.slug}"
                      text(post.title) {}
                    }
                    when(showSummaries) {
                      paragraph { classes = "post-summary"; text(post.summary) {} }
                    }
                  }
                }
              }
            }
          }
          paragraph {
            classes = "example-note"
            text(i18n"Example posts from the Anjunar Blog Tutorial.") {}
          }
        }
      }
      footer {
        classes = "site-footer"
        text(i18n"Anjunar Blog. Built in the open.") {}
        anchor() { href = "#main-content"; text(i18n"Back to top") {} }
      }
    }
  }
}
```

Der Code ist absichtlich ein Baum. Header, Hauptinhalt, Steuerelemente, Listenelemente und Fußzeile bleiben in `compose` zusammen. Es gibt keine separaten Rendering-Methoden für einzelne Markup-Teile.

`render(this, cursor)` liefert die aktuelle übergeordnete Komponente und den Render-Cursor an die geschachtelte DSL. Ein Aufruf wie `section { ... }` fügt an dieser Stelle eine Komponente in den Baum ein. In dieser Framework-Version hat `Article` keine Komfort-Factory, so dass `child(new Article) { ... }` diese vorhandene Komponente direkt montiert.

Der Titel in der Zeile ist vorerst Klartext. Wir haben keinen Link erfunden, dessen Route nicht existiert. Die eigentliche Postnavigation kommt mit dem Router in Kapitel 10 an.

## Folgen Sie einer Zustandsänderung

`showSummaries` ist ein `Property[Boolean]`. Drei Teile des Baumes verwenden denselben Wert:

```scala
ariaPressed = showSummaries
onClick(_ => showSummaries.set(!showSummaries.get))
```

und in jedem Artikel:

```scala
when(showSummaries) {
  paragraph { classes = "post-summary"; text(post.summary) {} }
}
```

Das Ereignis liest den aktuellen Wert und ändert ihn. Die Attributbindung aktualisiert den gedrückten Zustand der Schaltfläche, und `when` fügt oder entfernt jeden zusammenfassenden Absatz.

Ein gewöhnliches `if (showSummaries.get)` würde nur den Wert zur Kompositionszeit inspizieren. Es würde kein Abonnement für zukünftige Änderungen erstellen. Verwenden Sie `when`, wenn der Baum reagieren muss.

`newestFirst` folgt dem gleichen Prinzip. Sein Beobachter sortiert die Beispielbeiträge nach analysiertem Zeitstempel, verwendet die ID als Entscheidung bei gleichen Zeitstempeln, und ruft dann `posts.setAll` auf. Die Liste ist ein `ListProperty`, so dass das `foreach(posts)` des DSL den Reset beobachtet und die neue Reihenfolge rendert.

Dies ist das `Foreach.foreach` des Frameworks, nicht die gewöhnliche Sammlungstraversal von Scala. Bei einem Reset werden die Zeilenkomponenten entfernt und neu aufgebaut. Das Framework gibt ihre Bindungen mit ihnen frei; die Steuerelemente außerhalb der Liste bleiben bestehen.

Wir registrieren den Sortierbeobachter über `addDisposable`, so dass er zum Lebenszyklus der Seite gehört. Die erste Beobachtung füllt die Liste, bevor der Baum gebaut wird. Spätere Änderungen aktualisieren es synchron.

## Halten Sie UI-Nachrichten bereit für die Übersetzung

Die Seite bietet ein `I18nRuntime` mit Englisch als einzigem Gebietsschema und einem leeren Katalog. Ohne Katalogeintrag verwendet der Resolver den englischen Quelltext der Nachricht.

Labels wie `i18n"Show summaries"` durchlaufen bereits das Makro. Die Posttitel und Zusammenfassungen bleiben redaktionelle Inhalte, die von der Übersetzung von Buttons und Navigation getrennt sind.

Die Sortierschaltfläche benötigt sowohl seinen Status als auch die Übersetzungslaufzeit:

```scala
newestFirst.flatMap(newest =>
  translations.text(if (newest) i18n"Newest first" else i18n"Oldest first"))
```

Das Ergebnis bleibt eine reaktive String-Property. Wenn Sie `.get` beim Konstruieren der Schaltfläche aufrufen, wird stattdessen eine Zeichenfolge erfasst. Wir verwenden `.get` im Click-Handler, weil dieser Handler bewusst den aktuellen booleschen Wert benötigt.

Es gibt noch keinen Sprachwechsel. Mit der englischen Laufzeit können wir von Anfang an die beabsichtigte Nachrichten-API verwenden; deutsche Kataloge und Locale-Auswahl gehören zu Kapitel 18.

## Mounten Sie die Seite im Browser

Erstellen Sie `application/frontend/src/main/scala/com/anjunar/blog/frontend/Main.scala`:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom
import ui.core.component.Runtime
import ui.core.render.DomCursor

object Main {
  def main(args: Array[String]): Unit = {
    val root = dom.document.getElementById("app")
    require(root != null, "The page must contain an element with id='app'")
    Runtime.mount(new BlogPage, DomCursor.root(root))
  }
}
```

`DomCursor.root` zeigt auf den leeren Host. `Runtime.mount` erstellt den BlogPage-Host und führt seinen Komponentenlebenszyklus aus.

Der DOM-Zugang ist auf diesen Browser-Einstiegspunkt beschränkt. Die Seite beschreibt ihren Baum durch DSL und Cursor. Später kann ein Server-Cursor Komponenten ohne Browserdokument rendern.

Wir hydratisieren nichts in diesem Kapitel. Die Hydrierung erfordert ein vom Server gerendertes Markup und das Abgleichen der Anfangsdaten; das Passieren eines anderen Cursors allein würde diese Infrastruktur nicht bereitstellen.

## Geben Sie der Seite lesbare Stile

Erstellen Sie `application/frontend/src/main/resources/style.css` mit dem [Chapter Stylesheet](https://github.com/anjunar/anjunar-blog-example/blob/b7c8f7674c759a27432925353a7b2090038b8c5b/application/frontend/src/main/resources/style.css).

Das Design verwendet eine zurückhaltende Farbpalette, Überschriften in Serifenschrift, lesbare Textbreiten und horizontale Regeln zwischen den Beiträgen. Es benötigt keine heruntergeladene Schriftart, Icon-Paket oder Image-Asset.

Die wichtige Layoutregel ist die Postzeile:

```css
.post {
  display: grid;
  grid-template-columns: 150px minmax(0, 1fr);
  gap: 30px;
  padding: 32px 0;
  border-bottom: 1px solid var(--line);
}
```

Das Datum steht neben dem Artikel auf einem breiten Bildschirm. Unterhalb von 680 Pixeln schaltet das Stylesheet die Zeile in eine einzelne Spalte und lässt die Steuerelemente umschließen. `minmax(0, 1fr)` lässt die Textspalte schrumpfen, anstatt einen horizontalen Überlauf zu erzwingen.

Tastatur und semantisches Verhalten sind Teil dieser ersten Implementierung:

- Native Buttons bieten Enter und Space Aktivierung.
- Der Schalter stellt `aria-pressed` frei; beide Steuerelemente identifizieren die betroffene Liste mit `aria-controls`.
- Die Seite besitzt einen Main-Landmark, eine Überschrift der Ebene 1, eine gekennzeichnete Navigation und eine echte Liste von Artikeln.
- Der Skip-Link zeigt auf ein fokussierbares Hauptelement mit `tabIndex = -1`.
- Sichtbare Fokusrahmen und Schaltflächen, die mindestens 44 Pixel hoch sind, halten die Bedienelemente nutzbar.

Das Stylesheet folgt direkt dem State Attribut:

```css
button[aria-pressed="true"] {
  color: white;
  background: var(--accent);
  border-color: var(--accent);
}
```

Es ist kein zusätzlicher CSS-Zustand nötig, der mit dem Komponentenstatus synchronisiert werden müsste.

## Dateien über Undertow ausliefern

Wir verwenden den gleichen Ursprung für die Seite und die API. `application/backend/src/main/scala/com/anjunar/blog/FrontendHandler.scala` hinzufügen:

```scala
package com.anjunar.blog

import io.undertow.server.{HttpHandler, HttpServerExchange}
import io.undertow.server.handlers.resource.{PathResourceManager, ResourceHandler}
import io.undertow.util.{Headers, Methods, StatusCodes}

import java.nio.file.{Files, Path}

final class FrontendHandler(api: HttpHandler, assets: Path) extends HttpHandler {
  private lazy val files = new ResourceHandler(new PathResourceManager(assets.toAbsolutePath.normalize(), 1024L))
    .setDirectoryListingEnabled(false)
    .setWelcomeFiles("index.html")
  private val publicPaths = Set("/", "/index.html", "/main.js", "/main.js.map", "/style.css")

  override def handleRequest(exchange: HttpServerExchange): Unit = {
    val path = exchange.getRequestPath
    if (path == "/service" || path.startsWith("/service/")) api.handleRequest(exchange)
    else if (!publicPaths.contains(path)) {
      exchange.setStatusCode(StatusCodes.NOT_FOUND)
      exchange.endExchange()
    } else if (exchange.getRequestMethod != Methods.GET && exchange.getRequestMethod != Methods.HEAD) {
      exchange.setStatusCode(StatusCodes.METHOD_NOT_ALLOWED)
      exchange.getResponseHeaders.put(Headers.ALLOW, "GET, HEAD")
      exchange.endExchange()
    } else if (!Files.isDirectory(assets)) {
      exchange.setStatusCode(StatusCodes.SERVICE_UNAVAILABLE)
      exchange.getResponseHeaders.put(Headers.CONTENT_TYPE, "text/plain; charset=UTF-8")
      exchange.getResponseSender.send("Frontend assets are unavailable.\n")
    } else {
      // Development assets have stable names; a rebuild must be visible after reload.
      exchange.getResponseHeaders.put(Headers.CACHE_CONTROL, "no-cache")
      files.handleRequest(exchange)
    }
  }
}
```

Der Wrapper übergibt nur `/service` und `/service/...` an den vorhandenen REST-Handler. Es bedient die bekannten Frontend-Dateien für GET und HEAD. Eine unbekannte URL bleibt eine 404, anstatt die Homepage stillschweigend zurückzugeben.

Der Resource-Handler initialisiert sich erst bei Bedarf. Ohne ein gebautes Asset-Verzeichnis gibt die Webseite 503 zurück, während API-Routen verfügbar bleiben. Dadurch bleiben API-Tests und Backend-only-Arbeit unabhängig von einem Frontend-Build.

`Cache-Control: no-cache` ermöglicht eine Browser-Revalidierung, so dass das Neuaufbauen einer Datei unter demselben Entwicklungsnamen nach dem Neuladen sichtbar wird. Assets mit Fingerprint und Produktions-Cache-Richtlinien gehören zum Verpackungskapitel.

In `ApplicationMain.scala` ersetzen Sie den Import der einzelnen Serverklasse durch:

```scala
import dev.resteasy.embedded.server.{UndertowCdiEmbeddedServer, UndertowConfigurationOptions}
import io.undertow.servlet.api.DeploymentInfo

import java.nio.file.Path
```

Behalten Sie seine anderen Importe und ersetzen Sie `start` durch:

```scala
def start(port: Int, assets: Path = Path.of("target", "frontend")): UndertowCdiEmbeddedServer = {
    require(port >= 1 && port <= 65535, "BLOG_PORT must be between 1 and 65535")
    val server = new UndertowCdiEmbeddedServer()
    server.getDeployment.setApplication(new ServerApplication())
    val deployment = new DeploymentInfo()
      .addInitialHandlerChainWrapper(api => new FrontendHandler(api, assets))
    val configuration = SeBootstrap.Configuration.builder()
      .property(UndertowConfigurationOptions.DEPLOYMENT_INFO, deployment)
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
```

`DeploymentInfo` wickelt den vorhandenen Servlet-Handler ein; RESTEasy und CDI behalten ihren etablierten Anforderungspfad bei. Der Standard-Asset-Pfad ist relativ zum Repository-Verzeichnis, wo wir sbt ausführen.

Der Begleiter druckt auch die Seiten-URL und die öffentliche API-URL beim Start:

```scala
println(s"Blog: http://127.0.0.1:$port/")
println(s"API: http://127.0.0.1:$port/service/blog/posts")
```

## Die Oberfläche starten

Im Repository-Verzeichnis:

```text
sbt --server frontendAssets "application-backend/run"
```

Öffnen Sie [http://127.0.0.1:8080/](http://127.0.0.1:8080/). Setzen Sie `BLOG_PORT`, wenn dieser Port besetzt ist.

Der lokale Preview- und Liveness-Endpunkt funktioniert ohne PostgreSQL. Die REST-Post-Endpunkte und die Bereitschaftsprüfung erfordern weiterhin die Datenbankkonfiguration und Migration aus den früheren Kapiteln.

Verwenden Sie **Zusammenfassungen anzeigen**, um die Beschreibungen zu entfernen und wiederherzustellen. Wählen Sie **Neueste zuerst**, um zur Sortierung mit den ältesten Artikeln zuerst zu wechseln, und wechseln Sie dann zurück. Wiederholen Sie diese Aktionen mit der Tastatur und verengen Sie das Browserfenster.

Nachdem Sie Scala, HTML oder CSS geändert haben, führen Sie dies in einem anderen Terminal aus:

```text
sbt --server frontendAssets
```

Laden Sie die Seite neu, um die neuen Dateien zu sehen. Es gibt keinen Hot-Reload-Server in diesem Kapitel. Beenden Sie die Anwendung mit Ctrl + C.

## Überprüfen Sie das laufende Ergebnis

Der Begleiter beinhaltet Playwright 1.63.0, gepinnt in `package.json` und `package-lock.json`. Node/npm wird für diese Browsertests benötigt, nicht um die Schnittstelle zu bauen oder zu bedienen.

```text
npm ci
npx playwright install chromium
npm run test:browser
```

Der Befehl erstellt neue Assets und startet dann das eigentliche Backend auf `127.0.0.1:18080`. Dieser Port muss frei sein. Playwright stoppt danach seinen Server und verwendet niemals eine fremde laufende Instanz wieder.

Erwarten Sie **vier erfolgreiche Browsertests**. Sie prüfen gerenderte Inhalte und reaktive Bedienelemente, Tastaturfokus, mobilen Überlauf, JavaScript-Fehler, statische Inhaltstypen, Liveness und die Trennung zwischen Seiten- und API-Pfaden. Sie erfassen auch Desktop- und mobile Screenshots zur visuellen Inspektion. Diese Prüfungen decken die durchgeführten Abläufe ab und ersetzen kein vollständiges Accessibility-Audit.

Der Interaktionstest verbirgt beispielsweise Zusammenfassungen mit Space, ändert die Listenreihenfolge, ändert sie zurück und stellt Zusammenfassungen mit Enter wieder her. Es prüft dann, ob genau drei Artikel und drei Zusammenfassungen verbleiben. Dies übt die echte verknüpfte Anwendung und den Lebenszyklus von umgebauten Zeilen aus.

Führen Sie mit der separaten PostgreSQL-Testdatenbank, die konfiguriert und migriert wurde, die bestehende Backend-Suite aus:

```text
sbt --server "application-backend/testFull"
```

Erwarten Sie **42 erfolgreiche Backend-Tests**. Der neue Regressionstest beginnt mit einem fehlenden Frontend-Verzeichnis und überprüft, ob die API-Liveness immer noch 200, die Seite 503 und ein unbekannter API-Pfad 404 zurückgibt. Für dieses Kapitel ist keine Änderung des Datenbankschemas erforderlich.

Der [vollständige Quellcode zu Kapitel 9](https://github.com/anjunar/anjunar-blog-example/tree/b7c8f7674c759a27432925353a7b2090038b8c5b) enthält die vollständige Seite, das Stylesheet, den Asset Handler und die Tests.

Wir haben jetzt eine Browser-Schnittstelle, die auf Zustandsänderungen reagiert und neben der API läuft. Kapitel 10 ersetzt die lokalen Beispiele mit dem veröffentlichten Post REST-Vertrag.
