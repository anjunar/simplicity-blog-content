We have a public API. Now we need a page someone can read.

This chapter adds the first Scala.js interface: a journal with three post previews, a summary toggle, and a button that changes their order. The whole page is built with the Scala.js UI component DSL, and the existing Undertow server delivers it at `/`.

We start with local example data so the component model, state, and rendering stay visible. Chapter 10 will connect the page to the API and add navigation, loading, and error handling.

Start with the [chapter 8 source](https://github.com/anjunar/anjunar-blog-example/tree/54fac6042b34be4c8d7ab9b3f4ed054272220b81). All paths below are relative to the repository root.

## Add a browser module

The backend runs on the JVM. The new frontend compiles to JavaScript and runs in the browser. They belong to the same build, but the frontend does not depend on the Hibernate entity classes.

Create `project/plugins.sbt`:

```scala
addSbtPlugin("org.scala-js" % "sbt-scalajs" % "1.22.0")
```

Add these imports at the top of `build.sbt`:

```scala
import org.scalajs.linker.interface.ModuleKind
import org.scalajs.sbtplugin.ScalaJSPlugin
```

Add the frontend project and asset task:

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

Include the frontend in the existing root project's aggregation:

```scala
lazy val root = Project("anjunar-blog-tutorial", file("."))
  .aggregate(backend, frontend)
  .settings(publish / skip := true)
```

The existing Scala 3.9.0 setting and Maven Central resolver remain in place. In this sbt 2 build, `%%` selects the Scala.js artifact `scalajs-ui-core_sjs1_3`. Version 1.0.9 comes from Maven Central; readers do not need a checkout or local publication of the UI framework.

`scalaJSUseMainModuleInitializer` makes the linked module call our `main` method when the browser loads it. `ModuleKind.ESModule` matches the `type="module"` script in the HTML shell. The [Scala.js module guide](https://www.scala-js.org/doc/project/module.html) describes that module format.

`frontendAssets` runs the development linker and copies the linked files and resources to `target/frontend`. The task asks sbt for the linker's output directory instead of hard-coding a version-specific target path. `Def.uncached` makes the copy step run each time; sbt can still reuse unchanged compilation and linking results.

Edit files under `application/frontend`. Files under `target/frontend` are generated output. We will introduce optimized deployment packaging in its own chapter.

## Provide a small HTML host

Create `application/frontend/src/main/resources/index.html`:

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

The `app` element starts empty. The browser loads the module and mounts our component tree into that element. This is browser rendering; server rendering and hydration come later.

The document declares its English language, character encoding, and viewport. Its static title and description are enough for this first page. The `noscript` message explains why the preview is unavailable when JavaScript is disabled.

## Start with explicit example data

Create `application/frontend/src/main/scala/com/anjunar/blog/frontend/PostPreview.scala`:

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

`PostPreview` is a small read-only display projection, with field names that match the corresponding post fields. Its strings are local editorial examples. It is not a deserialization model for the REST response.

Keeping these examples separate from the component makes the next step straightforward: chapter 10 will introduce the frontend JSON model and HTTP service while retaining the page's composition approach. Changing a database row does not change this preview yet.

## Build one cohesive component tree

Create `application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala`:

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

The code is intentionally one tree. Header, main content, controls, list items, and footer remain together in `compose`. There are no separate rendering methods for individual pieces of markup.

`render(this, cursor)` supplies the current parent component and rendering cursor to the nested DSL. A call such as `section { ... }` mounts a component at that position in the tree. In this framework version, `Article` has no convenience factory, so `child(new Article) { ... }` mounts that existing component directly.

The row title is plain text for now. We have not invented a link whose route does not exist. Actual post navigation arrives with the router in chapter 10.

## Follow a state change

`showSummaries` is a `Property[Boolean]`. Three parts of the tree use that same value:

```scala
ariaPressed = showSummaries
onClick(_ => showSummaries.set(!showSummaries.get))
```

and, within each post:

```scala
when(showSummaries) {
  paragraph { classes = "post-summary"; text(post.summary) {} }
}
```

The event reads the current value and changes it. The attribute binding updates the button's pressed state, and `when` adds or removes each summary paragraph.

An ordinary `if (showSummaries.get)` would inspect only the value at composition time. It would not create a subscription to future changes. Use `when` when the tree must react.

`newestFirst` follows the same principle. Its observer sorts the example posts by parsed timestamp, using ID to break ties, then calls `posts.setAll`. The list is a `ListProperty`, so the DSL's `foreach(posts)` observes the reset and renders the new order.

This is the framework's `Foreach.foreach`, not Scala's ordinary collection traversal. On a reset, these row components are unmounted and rebuilt. The framework releases their bindings with them; the controls outside the list stay mounted.

We register the sorting observer through `addDisposable` so it belongs to the page's lifecycle. The initial observation fills the list before the tree is built. Later changes update it synchronously.

## Keep UI messages ready for translation

The page provides an `I18nRuntime` with English as its only locale and an empty catalog. Without a catalog entry, the resolver uses the message's English source text.

Labels such as `i18n"Show summaries"` already go through the macro. The post titles and summaries remain editorial content, which is a separate concern from translating buttons and navigation.

The order button needs both its state and the translation runtime:

```scala
newestFirst.flatMap(newest =>
  translations.text(if (newest) i18n"Newest first" else i18n"Oldest first"))
```

The result remains a reactive string property. Calling `.get` while constructing the button would instead capture one string. We use `.get` in the click handler because that handler deliberately needs the current boolean value.

There is no language switch yet. The English runtime lets us use the intended message API from the start; German catalogs and locale selection belong to chapter 18.

## Mount the page in the browser

Create `application/frontend/src/main/scala/com/anjunar/blog/frontend/Main.scala`:

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

`DomCursor.root` points at the empty host. `Runtime.mount` creates the BlogPage host and runs its component lifecycle.

DOM access is confined to this browser entry point. The page describes its tree through the DSL and cursor. Later, a server cursor can render components without a browser document.

We are not hydrating anything in this chapter. Hydration will require server-rendered markup and matching initial data; passing a different cursor alone would not provide that infrastructure.

## Give the page readable styles

Create `application/frontend/src/main/resources/style.css` using the [chapter stylesheet](https://github.com/anjunar/anjunar-blog-example/blob/b7c8f7674c759a27432925353a7b2090038b8c5b/application/frontend/src/main/resources/style.css).

The design uses a restrained color palette, a serif heading, readable text widths, and horizontal rules between posts. It needs no downloaded font, icon package, or image asset.

The important layout rule is the post row:

```css
.post {
  display: grid;
  grid-template-columns: 150px minmax(0, 1fr);
  gap: 30px;
  padding: 32px 0;
  border-bottom: 1px solid var(--line);
}
```

The date sits beside the post on a wide screen. Below 680 pixels, the stylesheet switches the row to a single column and lets the controls wrap. `minmax(0, 1fr)` lets the text column shrink instead of forcing horizontal overflow.

Keyboard and semantic behavior are part of this first implementation:

- Native buttons provide Enter and Space activation.
- The toggle exposes `aria-pressed`; both controls identify the affected list with `aria-controls`.
- The page has one main landmark, one level-one heading, a labelled navigation, and a real list of articles.
- The skip link points to a focusable main element, with `tabIndex = -1`.
- Visible focus outlines and buttons at least 44 pixels high keep the controls usable.

The stylesheet follows the state attribute directly:

```css
button[aria-pressed="true"] {
  color: white;
  background: var(--accent);
  border-color: var(--accent);
}
```

No second CSS-only boolean needs to be synchronized with the component state.

## Serve the files through Undertow

We will use the same origin for the page and API. Add `application/backend/src/main/scala/com/anjunar/blog/FrontendHandler.scala`:

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

The wrapper passes only `/service` and `/service/...` to the existing REST handler. It serves the known frontend files for GET and HEAD. An unknown URL remains a 404 instead of silently returning the homepage.

The resource handler is lazy. Without a built asset directory, the webpage returns 503 while API routes remain available. This keeps API tests and backend-only work independent of a frontend build.

`Cache-Control: no-cache` permits browser revalidation, so rebuilding a file under the same development name becomes visible after reload. Fingerprinted assets and production cache policies belong to the packaging chapter.

In `ApplicationMain.scala`, replace the single server-class import with:

```scala
import dev.resteasy.embedded.server.{UndertowCdiEmbeddedServer, UndertowConfigurationOptions}
import io.undertow.servlet.api.DeploymentInfo

import java.nio.file.Path
```

Keep its other imports and replace `start` with:

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

`DeploymentInfo` wraps the existing servlet handler; RESTEasy and CDI keep their established request path. The default asset path is relative to the repository root, which is where we run sbt.

The companion also prints the page URL and public API URL at startup:

```scala
println(s"Blog: http://127.0.0.1:$port/")
println(s"API: http://127.0.0.1:$port/service/blog/posts")
```

## Run the interface

From the repository root:

```text
sbt --server frontendAssets "application-backend/run"
```

Open [http://127.0.0.1:8080/](http://127.0.0.1:8080/). Set `BLOG_PORT` if that port is occupied.

The local preview and liveness endpoint work without PostgreSQL. The REST post endpoints and readiness check still need the database configuration and migration from the earlier chapters.

Use **Show summaries** to remove and restore the descriptions. Select **Newest first** to switch to oldest-first order, then switch back. Repeat those actions with the keyboard and narrow the browser window.

After changing Scala, HTML, or CSS, run this in another terminal:

```text
sbt --server frontendAssets
```

Reload the page to see the new files. There is no hot-reload server in this chapter. Stop the application with Ctrl+C.

## Check the running result

The companion includes Playwright 1.63.0, pinned in `package.json` and `package-lock.json`. Node/npm is needed for these browser tests, not to build or serve the interface.

```text
npm ci
npx playwright install chromium
npm run test:browser
```

The command builds fresh assets, then starts the real backend on `127.0.0.1:18080`. That port must be free. Playwright stops its server afterward and never reuses an unrelated running instance.

Expect **four successful browser tests**. They verify rendered content, reactive controls, keyboard focus, mobile overflow, JavaScript errors, static content types, liveness, and the separation between page and API paths. They also capture desktop and mobile screenshots for visual inspection. These checks cover the implemented flows, rather than claiming a complete accessibility audit.

For example, the interaction test hides summaries with Space, changes the list order, changes it back, and restores summaries with Enter. It then checks that exactly three articles and three summaries remain. This exercises the real linked application and the lifecycle of rebuilt rows.

With the separate PostgreSQL test database configured and migrated, run the existing backend suite:

```text
sbt --server "application-backend/testFull"
```

Expect **42 successful backend tests**. The new regression test starts with a missing frontend directory and verifies that API liveness still returns 200, the page returns 503, and an unknown API path returns 404. No database schema change is needed for this chapter.

The [completed chapter 9 source](https://github.com/anjunar/anjunar-blog-example/tree/b7c8f7674c759a27432925353a7b2090038b8c5b) contains the full page, stylesheet, asset handler, and tests.

We now have a browser interface that reacts to state changes and runs beside the API. Chapter 10 replaces the local examples with the published-post REST contract.
