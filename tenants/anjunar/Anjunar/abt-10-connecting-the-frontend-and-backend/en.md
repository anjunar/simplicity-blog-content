The journal can render a list, and the backend can return one. This chapter connects them.

A visitor opens the blog, sees published posts from PostgreSQL, follows a title to the complete article, and returns with the browser's Back button. The same page also has to work when the database is empty, a request fails, or the visitor leaves before a response arrives.

Start from the [chapter 9 source](https://github.com/anjunar/anjunar-blog-example/tree/b7c8f7674c759a27432925353a7b2090038b8c5b). The [complete chapter 10 checkpoint](https://github.com/anjunar/anjunar-blog-example/tree/db992619d48e8938308216641856849e8be9bd97) contains every file discussed below. Paths are relative to the repository root.

## Add JSON mapping and routing

The frontend already uses Scala.js UI Core. In the frontend project's settings in `build.sbt`, replace its single library dependency with:

```scala
libraryDependencies ++= Seq(
  "com.anjunar" %% "scalajs-ui-core" % "1.0.9",
  "com.anjunar" %% "scalajs-ui-json" % "1.0.9",
  "com.anjunar" %% "scalajs-ui-router" % "1.0.9",
  "org.scalatest" %% "scalatest" % "3.2.20" % Test
),
```

This project uses sbt 2, where `%%` selects the platform-specific Scala.js artifacts for the frontend project. All four dependencies come from Maven Central.

Refresh the frontend's dependency resolution:

```text
sbt --server "application-frontend/update"
```

Remove `PostPreview.scala`, including its fixed example posts. The application will now show the API's result, including an empty list when nothing has been published.

## Mirror the published entity

The frontend needs a Scala.js model. It cannot use the JVM class containing Hibernate and JPA annotations, but it should preserve its field names and meaning.

Create `application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPost.scala`:

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

There are eight entity fields, just as on the backend. UUIDs and ISO-8601 timestamps arrive as strings; the publication status is the enum's name. The optimistic-lock version remains a `Long`. A persisted entity can have version zero, so `0` must never mean “missing.”

The list graph leaves out `content`. We represent that absence with `None`; the detail request supplies `Some(text)`. Optional summaries and publication timestamps use the same representation. A list row is therefore not a complete model for an update. This chapter only reads data.

Each row keeps the REST `data` wrapper. The table keeps `rows` and `size`, where size is the total number of published posts before pagination. Our backend omits empty collections, so an empty response can be:

```json
{"size":0,"@type":"Table"}
```

The empty default for `rows` is deliberate. If the visitor requests an offset beyond the last post, rows can also be absent while size remains positive.

The `-1` default for size lets the service reject a response with no total. The data wrapper starts with `null` so a missing `data` member cannot silently become a plausible empty post.

### Two schemas, two jobs

The backend's `EntitySchema` describes entity fields, mapper rules, and Criteria attributes. The browser's `ui.json.JsonSchema` is derived from these local Scala.js classes and supplies constructors and property accessors to its mapper.

The structural `schema` object in a REST response does not generate our frontend classes or grant permission to write them. This reader does not consume that metadata. The concrete frontend models also tolerate the backend's `@type` markers.

The models are maintained together with the REST contract. Adding a backend field does not automatically add a browser property.

## Put HTTP in one place

Create `HttpJson.scala` beside the model:

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

The method accepts a target model type and an optional cancellation signal. Scala derives the required `JsonSchema[M]` at the concrete call site. After reading the body, `JsonMapper` constructs the model and populates its properties; components never walk `js.Dynamic` objects themselves.

Check the status before mapping. Fetch also resolves its promise for HTTP errors such as 404, so a successful promise alone does not mean a successful request. Passing an abort signal lets navigation cancel a request that is no longer needed. [MDN explains both behaviors](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch).

We use relative URLs on the same origin as the page. Undertow already serves both the frontend and `/service`, so this step needs no second server or CORS configuration. The helper preserves the status of unsuccessful responses without displaying their bodies to visitors. Structured validation problems will become relevant when we introduce writes.

## Give the API operations names

Create `BlogService.scala`:

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

The service owns endpoint paths, response types, and the page size. The detail method returns the entity inside its envelope. It also checks that the detail graph actually supplied content.

A slug is encoded as a URL segment. No component needs to remember the REST prefix or how to unwrap a response.

The server already orders posts by publication time descending, then by ID for ties. We keep that order. Chapter 9's local order button is removed: reversing twenty loaded rows would only reverse one page, not the entire result.

## Let the route own the request

Create `BlogRoutes.scala`:

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

A route loader returns a future component. The list route reads a nonnegative integer offset from the URL, requests one page, and constructs a `PostListPage`. The detail route requests a full post by slug.

The loading component is visible while the future is pending. A 404 selects the missing-post page; an invalid offset selects the invalid-page response. HTTP failures, lost connections, and invalid JSON reach the unavailable page. An error route is an internal forward: the visitor's requested address stays in the URL bar.

Notice the path of `context.signal`: route → service → HTTP helper → fetch. When another navigation begins, the router aborts the previous request. Router 1.0.9 also checks a render token before installing an asynchronous result. That second check prevents an old completion from replacing the new page even if completion races with cancellation.

Keep the request in the loader. Starting another request in `compose` would create a second owner for the same operation.

### Keep UI actions outside the entity

Create `BlogActions.scala`:

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

The summary preference is application UI state. It does not belong in `BlogPost`, where it could be mistaken for part of the entity contract.

One shared actions instance survives list/detail navigation. The summary button binds `ariaPressed` to `actions.showSummaries`, and its click handler calls `toggleSummaries()`. The existing `when` block observes the same property.

Retry intentionally reloads the current document. Router 1.0.9 has no public reload operation, and navigating to an equal route state does not start another load. Reloading preserves the path and query, retries the request, and resets the in-memory summary preference.

## Keep the shell and page trees together

`BlogPage` now owns the persistent site header, main landmark, footer, English translation runtime, and router. Each route owns its page component. These are meaningful component boundaries: opening an article replaces the list while leaving the site shell mounted.

The setup at the start of `BlogPage.compose` uses these imports:

```scala
import ui.core.i18n.I18nRuntime
import ui.router.Router
```

Inside that method, before its existing `render(this, cursor)` block:

```scala
I18nRuntime.provide(translations)(using this)
val router = new Router(pages.routes, cursor.browserUrl.getOrElse("/"), pages.config)
Router.provide(router)(using this)
```

Here `pages` is the component's `BlogRoutes` instance. Providing the router on the shell makes it available to both header links and the page subtree. Inside the main landmark, `child(router) {}` mounts the route outlet. See the [complete BlogPage](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala) for the surrounding tree and imports.

The existing English runtime makes router links locale-aware. The list lives at `/en`, and an article at `/en/posts/:slug`. The root `/` remains an entry point. Route definitions stay locale-independent, while API paths remain under `/service`. German messages and language selection enter in chapter 18.

### Render list rows and detail content

[PostListPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostListPage.scala) keeps the introduction and list from chapter 9. It initializes its `ListProperty` from `table.rows.map(_.data)`. The DSL's `foreach` renders the rows, and each title becomes a `routerLink` to the post's slug.

It also adds three behaviors:

- A zero total displays “No posts have been published yet.” An empty page with a positive total displays “There are no posts on this page.”
- A missing summary produces no summary paragraph. Toggling summaries remains reactive.
- Newer/Older links carry offset in the URL and request twenty posts per page. Browser history restores the query; the server still chooses row order.

The page links expose the API's existing pagination. Chapter 15 will expand the search and sorting model.

The detail is a separate component. Create `PostPage.scala`:

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

The content goes through a text binding. Text that resembles a script or HTML element is displayed as text. The `post-content` CSS rule uses `white-space: pre-wrap` to preserve line breaks and `overflow-wrap: anywhere` for long content. Rich content rendering comes with the editor in chapter 17.

The complete [LoadingPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/frontend/src/main/scala/com/anjunar/blog/frontend/LoadingPage.scala) uses a status region. [ErrorPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/frontend/src/main/scala/com/anjunar/blog/frontend/ErrorPage.scala) uses an alert region, a readable message, and a recovery action. Each component keeps its entire UI tree inside `compose`; none starts a data request.

UI labels use the `i18n` macro. Post titles, summaries, and content are editorial data and stay outside the message catalog.

Finally, update `Main.scala` to create the shared service and actions:

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

## Make direct article URLs work

Client navigation is only half the job. Reloading `/en/posts/our-first-public-post` sends that path to Undertow before any Scala.js code runs.

Update [FrontendHandler.scala](https://github.com/anjunar/anjunar-blog-example/blob/db992619d48e8938308216641856849e8be9bd97/application/backend/src/main/scala/com/anjunar/blog/FrontendHandler.scala) to recognize `/en`, `/en/`, and `/en/posts/{slug}`, using the same lowercase, hyphen-separated slug shape as the entity. For these GET/HEAD requests it serves `index.html`. The `/index.html` entry point redirects to `/` while preserving its query. Existing static assets keep their paths, and `/service` stays with RESTEasy.

This is a limited page fallback. Unknown assets and unrelated paths still return 404. POST to a page path returns 405. API startup still works before frontend assets have been built.

There is an important boundary here: the HTML document for a well-formed but unknown slug returns 200. The browser then calls the API, receives 404, and renders “Post not found.” A client error route cannot change a document response that has already been sent. Chapter 19 will load the route on the server and give the document its proper status.

## Run the complete path

Use the database setup from chapter 6, with `BLOG_DB_URL`, `BLOG_DB_USER`, and `BLOG_DB_PASSWORD` set for your local database. Start and migrate it before running the application:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

This chapter changes no database fields or schema IDs.

With the tutorial's Compose database, load the optional chapter 8 examples:

```text
docker compose cp database/examples/public-posts.sql postgres:/tmp/public-posts.sql
docker compose exec -T postgres psql -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --file /tmp/public-posts.sql
```

For native PostgreSQL, use the connection details for your tutorial database:

```text
psql -h 127.0.0.1 -p 5433 -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --file database/examples/public-posts.sql
```

The script inserts one published post and one draft. Repeating it preserves those rows. It is optional example data, not a migration or an automatic startup task. The native psql command prompts for its password separately.

Build and start from the repository root:

```text
sbt --server frontendAssets "application-backend/run"
```

Open `http://127.0.0.1:8080/`. Select **Our first public post**, reload its detail URL, and use Back and Forward. The full content appears only on the detail page. Opening `/en/posts/our-private-draft` shows the same missing-post page as an unknown slug.

An empty migrated database shows the empty state. A database connection failure shows the unavailable state. The application no longer substitutes local preview posts.

## Verify the boundary

The JSON tests run in Scala.js:

```text
sbt --server "application-frontend/testFull"
```

The six checks cover the entity fields, version zero, missing content, optional nulls, omitted rows, and totals on empty pages. They also verify that server metadata does not interfere with the concrete browser models.

Install the browser test dependencies and run the controlled response scenarios:

```text
npm ci
npx playwright install chromium
npm run test:browser
```

These eleven checks run the compiled frontend against the actual HTTP server, but intercept data requests. That lets them hold a response pending, produce a network error, return malformed JSON, and verify cancellation without depending on timing in a database. They also cover history, paging, text escaping, keyboard controls, mobile layout, and static route status codes.

Then use a dedicated test database migrated to the current schema and seeded with the unchanged SQL examples:

```text
sbt --server "application-backend/testFull"
npm run test:browser:database
```

The 42 backend tests remain green. The two additional browser tests use real PostgreSQL data without interception: public list → detail → reload, followed by draft and unknown-post handling. Keep the sample public post on the first page of this test database. The database browser tests do not add or delete rows.

Both browser projects start their own server on port 18080 and stop it afterward. Keep that port free. Screenshots are stored in `test-results`.

We now have a complete public read path, from a database row to a navigable article. Next we will add user accounts and sign-in, which gives us an identity to use when we introduce permissions and editing.
