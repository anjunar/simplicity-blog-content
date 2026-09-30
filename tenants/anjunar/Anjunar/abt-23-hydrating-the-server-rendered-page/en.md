# Hydrating the Server-Rendered Page

Chapter 22 made the blog readable before JavaScript started. But `boot` then
cleared `#app`, fetched the current route again and rebuilt the page.

Now we want the browser to **keep the existing nodes** and attach the component
tree to them. The article, its heading and the search input should remain the
same DOM objects. The initial public data request should disappear from the
browser's network log.

That requires two things to agree: the component tree and the data used to build
it. Replacing `DomCursor` with `HydratingCursor` solves only the first part.

## Carry the response that produced the HTML

Suppose the server renders a German article. If the browser fetches it again,
the translation could have changed in the meantime. Even without a change, the
router has to wait for that request.

Each [BlogDocument](https://github.com/anjunar/anjunar-blog-example/blob/8de5903024000706abf604833a32ee18738d37aa/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogDocument.scala)
therefore creates `InitialPageData.capture(url)` and passes it to its
`BlogService`. This object records the public response used by that request:

| Snapshot field | Purpose |
| --- | --- |
| `format`, `url` | Identify the state format and the document's path/query. |
| `request` | Match the exact API path and query, including locale and filters. |
| `status`, `contentType`, `body` | Replay the same successful response or failure. |

We keep the original JSON body. Serializing the frontend entity again would be
wrong here: its write rules intentionally omit response-only fields such as
translation and publication metadata. Replaying the received body preserves
those fields, omitted values, nulls and version zero through the existing mapper.

This is one public response per rendered page. Account sessions, credentials
and private editorial responses do not enter the snapshot.

In `BlogDocument.compose`, the following block comes after the existing
`DocumentHead` setup. `documentHead` is the document's registry and `initial`
is its per-request recorder. Its encoded property changes when the response arrives:

```scala
import ui.core.document.HeadEntry

val stateHead = documentHead.handle(this)
addDisposable(initial.encoded.observe(value => stateHead.set(
  HeadEntry("application-state", "script",
    Seq("id" -> "application-state", "type" -> "application/json", "data-state" -> value))
)))
```

`HeadEntry` writes an inert `application/json` element. Its `data-state`
attribute contains URI-encoded JSON; the head writer also escapes attribute
values. Article text containing quotes or `</script>` remains data, without
becoming executable script content.

The recorder lives on the document instance. It is not a shared cache between
server requests.

## Keep the initial route synchronously available

There is a subtle second requirement. In this framework version, a router
loader that is still pending during hydration temporarily adopts the visible
range, then replaces its content when loading finishes. That avoids an empty
page, but it does not preserve the article's DOM identity.

Our replay must therefore return an **already-completed Future**, and the pure
transformations leading to the component must preserve that property.

Here is the complete public service:

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
has three modes. On the server it captures the received response. During browser
startup it immediately decodes the matching saved response through
`HttpJson.decode`. After startup it forwards requests to the normal HTTP path.

The replayed response is consumed once. `ExecutionContext.parasitic` runs the
small validation maps inline when the input Future is already complete.
It does not make browser network requests synchronous: live requests still wait
for `fetch`.

The public route's final map needs the same treatment. This entry replaces the
detail entry in the existing `BlogRoutes.routes` sequence; `service` is its
constructor dependency:

```scala
import ui.router.Route
import scala.concurrent.ExecutionContext

Route.view("/posts/:slug") { context =>
  val locale = context.locale.map(_.code).getOrElse("en")
  service.detail(context.pathParams("slug"), context.signal, locale)
    .map(new PostPage(_, locale))(using ExecutionContext.parasitic)
}
```

The list route follows the same pattern. Using the usual queued execution
context for that final map would make the router see a pending Future again,
even though the JSON is already available.

Failures also matter. A saved 404 produces an already-failed Future and selects
the existing error route without another API request. An invalid search URL can
have an empty snapshot because parsing rejected it before any request was sent.

## Claim the page and finish hydration

The document wrappers from chapter 22 stay in place. We hydrate `BlogPage`
inside `#app`, where the server mounted that same component.

Here is the complete updated `Main`. Its server `render` function remains
unchanged; the browser path now owns hydration and its recovery:

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

The steps inside `start` are significant:

1. Read the current path/query and validate the embedded snapshot against it.
   A snapshot for another language or search URL cannot initialize this page.
   URL fragments remain browser-local.
2. Mount with `HydratingCursor` so components claim existing elements and
   virtual-range anchors. The ready first route lets it claim the article too.
3. Drain the registered asynchronous work, then call `completeHydration`.
   This checks that the expected tree has been consumed and activates deferred
   callbacks.
4. Call `finish` to release replay state. Future navigation goes back to the API.
   Remove the serialized bootstrap element once startup settles.

`boot` retains its promise, so repeated calls cannot mount another application
or register another set of listeners. `ClientMain` still invokes it once when
the browser bundle starts.

## Preserve input and make recovery explicit

Readers can type into the server-rendered search form while the module is
downloading. Ordinary input bindings would otherwise write their initial model
values over that typing.

[PostSearchForm](https://github.com/anjunar/anjunar-blog-example/blob/8de5903024000706abf604833a32ee18738d37aa/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostSearchForm.scala)
captures the native control values in `beforeHostBinding`, before those
bindings run. Its `afterHydration` callback restores the values through the
bound properties. The initial tree stays unchanged while it is being claimed;
afterward, the normal validation and language-switch guard see the edited model.
The DSL tree still stays together in `compose`.

Missing state uses a fresh mount, as do the account/editorial shells.
A corrupt snapshot, wrong page URL or structural mismatch cancels the hydration
attempt and unmounts its partial tree before **one** fresh mount. That recovery
logs a warning and can fetch again. It does not count as successful hydration,
and it does not start a reload loop. A complete rebuild can discard input typed
before startup; successful hydration preserves it.

The recovery test checks an actual working toggle after rebuilding. Seeing one
page on screen would not prove that abandoned event listeners were removed.

## Verify identity, not just identical text

Run the application with the existing database; no migration or new dependency
is needed:

```text
sbt --server frontendAssets "application-backend/run"
npm run test:browser:ssr
```

Stop the running server before starting the automated suite, which owns port
18080. The [chapter guide](https://github.com/anjunar/anjunar-blog-example/blob/8de5903024000706abf604833a32ee18738d37aa/docs/hydrating-the-server-rendered-page.md) documents
the environment and full regression commands.

The browser tests pause the module download, retain references to server-created
nodes and compare those references after `boot`. They also assert zero initial
public API requests, preserve early typing and call `boot` twice. Another test
changes the database after SSR: startup keeps the rendered version, while a
later language round trip fetches the newer title.

The page now moves from readable HTML to an interactive component tree without
replacing its content on the normal startup path. Chapter 24 is the final
installment: metadata, canonical and alternate-language links, sitemap and feed.
