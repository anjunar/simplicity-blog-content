# Rendering Pages on the Server

Open an article with JavaScript disabled. Until now, the response contains an
empty `#app`: the browser must start Scala.js and fetch the post before anyone
can read it. In this chapter, the response already contains the article.

We will run our existing Scala.js interface inside GraalJS on the JVM. The same
routes load the same public API and mount the same components. English, German,
Markdown and the translation fallback therefore follow the contract from earlier
chapters. A missing article also needs a **404 document response**, not merely an
error message inside a successful page.

This chapter delivers that HTML. Chapter 23 will teach the browser to adopt it
through hydration. For now, the browser starts a fresh component tree.

## Give the frontend two entry points

The old `Main.main` immediately accesses `document` and `window`. That is a
browser entry point; evaluating it on the server would fail before rendering.

We split the build into two Scala.js projects:

| Project | Purpose | Module initializer | Copied output |
| --- | --- | --- | --- |
| `application-frontend` | Shared interface and exported server render function | Disabled | `target/frontend/ssr/main.js` |
| `application-client` | Browser startup; depends on frontend | Enabled | `target/frontend/main.js` |

Both produce ES modules. The existing `frontendAssets` task builds and copies
both; UI dependencies remain in the shared frontend project. The
[build definition](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/build.sbt) shows the complete settings.

The entire browser entry point is:

```scala
package com.anjunar.blog.client

import com.anjunar.blog.frontend.Main

object ClientMain {
  def main(args: Array[String]): Unit = Main.boot()
}
```

Here is the updated shared `Main`, including what happens when the browser starts:

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

The export makes `render` callable from the JavaScript module. It constructs a
**new document for each request**. `Runtime.mount(document, cursor)` is essential:
the callback receives the server cursor, but constructing a component alone does
not mount its tree.

`renderToStringAsync` waits for work registered by the router's asynchronous
loaders before serializing. A synchronous render could finish before the post
arrives. The promise resolves to HTML and a separate HTTP status. The runtime
also disposes the mounted tree after rendering.

`boot` still uses `DomCursor`. Clearing `#app` prevents a second page from being
appended below the server's page. It also means another data request and a
possible loading transition. That is the deliberate boundary before hydration.

## Compose the document around the existing page

SSR must return a whole document, including the styles and browser module.
`BlogDocument` owns that outer structure; `BlogPage` still owns navigation,
routing and the page content. This is the complete document component:

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

The DSL tree stays together in `compose`. The existing `BlogPage` is mounted as
a child inside the same `#app` used by `boot`; there is no second JVM template
for article content.

`DocumentHead` collects head entries and writes them into `head`. Its keys
identify entries: the two stylesheets need distinct keys, otherwise one would
replace the other. Article titles, descriptions and canonical URLs come in
chapter 24; this chapter supplies the document needed to display the page.

The third `BlogPage` argument is the request URL. On the server there is no
browser location, so the component uses this explicit URL to initialize both
`I18nRuntime` and `Router`. A request for `/de/posts/example` therefore selects
German before loading or rendering anything. The existing post component still
marks English fallback content with `lang="en"`.

In [BlogPage](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala),
browser listeners and updates to the actual browser document remain guarded by
`cursor.isBrowser`. Rendering HTML must not attempt to install a browser history
listener or read a DOM element.

## Let GraalJS execute the shared module

The JVM needs the Polyglot API and GraalJS, both pinned to 25.3.4.1 in
`build.sbt` and obtained from Maven Central. Our JDK 25 setup can run them
without a separate GraalVM installation.

[SsrRenderer](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/backend/src/main/scala/com/anjunar/blog/SsrRenderer.scala)
owns one worker and an eight-entry waiting queue. Each request gets a fresh
JavaScript `Context`; a shared `Engine` and module `Source` allow code reuse
without retaining one request's component state for the next.

GraalJS is not a browser. Our existing `HttpJson` needs `Headers` and `fetch`,
and the Scala.js runtime needs timers. A small
[host adapter](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/backend/src/main/resources/ssr/host.js) supplies those
interfaces. Its fetch bridge only accepts GET requests to the public posts API
at this server's fixed loopback origin. It sends no session cookies, follows no
redirects and cannot call the editorial API.

The bridge uses Java's HTTP client on the renderer worker and presents a promise
to JavaScript. REST still selects published rows, applies localization and owns
its normal transaction. There is no extra database reader inside the renderer.

Inside `SsrRenderer.evaluate`, after the host adapter has been installed and
`exports = context.eval(module)` has evaluated the ES module, this block receives
the result. These imports belong to that class; `RenderedPage` is its local
package's case class with `html: String` and `status: Int`:

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

The callbacks run on the context's owning worker. The following loop processes
registered timers until a result or failure arrives; the complete implementation
is linked above. It closes the context on every exit.

The caller's 15-second deadline includes queueing. API requests have a
three-second response-header timeout and a 2-MiB body limit; reading the body
remains inside the overall render deadline. Cancellation also closes the
context, so a stuck JavaScript computation does not permanently occupy the
worker. This deliberately small runtime processes one render at a time; it is
not a general browser emulator or a tuned production renderer.

## Make the document's status match its content

A route can fail after the first asynchronous API request. We therefore set
`renderErrorsOnServer = true` in the existing
[RouterConfig](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogRoutes.scala).
Its existing failure mapping selects the 400, 404 or 503 error route, and
`Router.responseStatus` supplies the corresponding status through
`BlogPage.responseStatus` and `BlogDocument.responseStatus`.

[FrontendHandler](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/application/backend/src/main/scala/com/anjunar/blog/FrontendHandler.scala)
dispatches public page requests off Undertow's I/O thread, waits for the render
result and sends its HTML with that status. HEAD performs the same rendering but
sends no body. Render failures, missing bundles and queue/deadline failures return
a safe 503. The API stays available.

The resulting public documents use `Cache-Control: no-store`. Account and
editorial pages still use the browser shell and their existing authentication;
they are outside this anonymous SSR path. The server bundle is not exposed as a
downloadable asset.

## Verify the actual response

No migration is added here. With the chapter 21 database configured, build and run:

```text
npm ci
sbt --server frontendAssets "application-backend/run"
curl -i "http://127.0.0.1:8080/en?limit=1"
curl -I "http://127.0.0.1:8080/en/posts/no-such-public-post"
```

The first response should contain the list; the second should be 404. Disable
JavaScript and open a published article in both languages. Check its Markdown,
follow list and paging links, and verify English fallback for an unpublished
German translation. Search submission, language buttons and editorial tools
still require JavaScript; direct filtered/localized URLs do not.

The [checkpoint guide](https://github.com/anjunar/anjunar-blog-example/blob/ca8a382ef8e1afb447d7edb266715dfe24171d26/docs/rendering-pages-on-the-server.md) gives the exact
test setup. `npm run test:browser:ssr` uses real PostgreSQL data and a browser
with JavaScript disabled for its reading checks. It also verifies escaped
titles, error statuses, HEAD and normal browser remounting. The existing browser
contract suite explicitly disables SSR because it intercepts browser data
requests; that suite alone cannot verify server rendering.

After rebuilding frontend code, restart the backend: its module source is cached
for the renderer's lifetime. Server and browser bundles must match.

Readers can now receive the complete public page in the first response.
Chapter 23 will preserve that DOM and its initial data when interactivity starts.
