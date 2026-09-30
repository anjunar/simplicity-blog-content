# Completing the Public Pages

The blog can now serve readable HTML and hydrate it without replacing the page.
There is one remaining boundary to complete: what a URL tells search engines,
link previews and feed readers about its content.

An English article displayed inside the German interface is still an English
article. A search result is not the same page as the unfiltered journal.
A missing article must return 404 even when its error screen looks polished.

This final chapter makes those distinctions explicit. It adds page metadata,
canonical and language links, a sitemap, language-specific Atom feeds and
permanent redirects for the supported URL aliases.

## Give each public page a clear identity

We use the following policy:

| Request | Canonical and discovery behavior |
| --- | --- |
| English article | English canonical; link to German only when its translation is published. |
| Published German article | German canonical; reciprocal links to both published languages. |
| German URL with English fallback | English canonical; no German article entry in sitemap or feed. |
| Default journal and ordinary pagination | Self-canonical; pagination keeps its offset. |
| Search, custom sorting or page size | Normalized self-canonical URL with `noindex, follow`. |
| Missing article or route | HTTP 404 and `noindex`; no canonical article URL. |

A canonical identifies the preferred URL for duplicate content. Language
alternatives identify published variants. They solve different problems.
Our language links include the current variant and return links to the others;
English is also the `x-default` destination. This follows the
[canonical guidance](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)
and [language-link guidance](https://developers.google.com/search/docs/specialty/international/localized-versions).

All absolute addresses start with `BLOG_PUBLIC_ORIGIN`. The backend validates
that setting and supplies it to both SSR and the browser shell. We do not derive
the canonical host from forwarded request headers. Locally, its default is
`http://127.0.0.1:<BLOG_PORT>`; a hosted installation sets its HTTPS origin.

The detail API already provides `contentLocale`, `availableLocales` and the
selected `translation`. These methods from the [PageHead companion](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/application/frontend/src/main/scala/com/anjunar/blog/frontend/PageHead.scala)
turn that contract into head entries. They belong inside the existing companion
object; the two small helpers are included so every entry is visible:

```scala
package com.anjunar.blog.frontend

import ui.core.document.HeadEntry

// These methods belong to the PageHead companion object.
  private def feed(origin: String, locale: String): HeadEntry =
    HeadEntry("feed", "link", Seq("rel" -> "alternate", "type" -> "application/atom+xml",
      "title" -> s"Anjunar Journal ($locale)", "href" -> s"$origin/$locale/feed.xml"))

  private def social(title: String, description: String, url: String, kind: String): Seq[HeadEntry] = Seq(
    HeadEntry.title(s"$title — Anjunar Journal"),
    HeadEntry.meta("description", description),
    HeadEntry.property("og:title", title),
    HeadEntry.property("og:description", description),
    HeadEntry.property("og:type", kind),
    HeadEntry.property("og:url", url),
    HeadEntry.property("og:site_name", "Anjunar Journal"))

  def article(origin: String, post: BlogPost): Seq[HeadEntry] = {
    val translation = Option(post.translation.get)
    val title = translation.map(_.title.get).getOrElse(post.title.get)
    val summary = translation.map(_.summary.get).getOrElse(post.summary.get)
    val description = Option(summary).filter(_.nonEmpty).getOrElse(title)
    val locale = post.contentLocale.get
    val canonical = s"$origin/$locale/posts/${post.slug.get}"
    val languages = (Seq("en") ++ post.availableLocales.toSeq.filter(_ == "de")).distinct
    val alternates = languages.map(language =>
      HeadEntry.alternate(language, s"$origin/$language/posts/${post.slug.get}")) :+
      HeadEntry.alternate("x-default", s"$origin/en/posts/${post.slug.get}")
    val image = Option(post.coverImage.get).toSeq.flatMap(value => Seq(
      HeadEntry.property("og:image", origin + value.source),
      HeadEntry.property("og:image:alt", Option(post.coverAlt.get).getOrElse(""))))
    social(title, description, canonical, "article") ++ Seq(
      HeadEntry.link("canonical", canonical), HeadEntry.meta("robots", "index, follow"),
      HeadEntry.property("og:locale", if (locale == "de") "de_DE" else "en_US"), feed(origin, locale)) ++
      post.publishedAt.get.toSeq.map(value => HeadEntry.property("article:published_time", value)) ++ alternates ++ image
  }
```

Notice the summary selection. A German translation without a summary falls back
to its German title. It does not borrow the English summary. The same whole-article
selection rule introduced in chapter 21 also governs the description.

`HeadEntry` handles attribute and text escaping. The cover URL refers to the
existing media endpoint; its authorization rules still apply. These are document
links, while REST action links continue to come from `LinkBuilder`.

## Let metadata follow the component lifecycle

Writing the correct initial head is only half the job. After navigating from an
article to the list, the article title, image and publication metadata must leave
the head. Otherwise the URL and the preview describe different pages.

The existing `DocumentHead` registry gives entries stable keys. Registrations
can override the same key, and disposing a registration reveals the previous
value. On the server, the document's head sink writes that registry into the HTML.

Our browser hydrates only `#app`, so its head remains outside the hydration
cursor. The following complete `PageHead` class uses the library's
`BrowserHeadSink` through its `HeadSink` interface. It synchronizes the registry
when a component adds or removes its entries:

```scala
import ui.core.component.AbstractComponent
import ui.core.document.{DocumentHead, HeadEntry, HeadSink}

final class PageHead(val origin: String, registry: DocumentHead, sink: Option[HeadSink]) {
  def bind(entries: HeadEntry*)(using owner: AbstractComponent): Unit = {
    val handle = registry.handle()
    handle.set(entries*)
    synchronize()
    owner.addDisposable(() => {
      handle.dispose()
      synchronize()
    })
  }

  private def synchronize(): Unit =
    sink.foreach(_.update(registry.entries, registry.htmlAttributes))
}
```

`BlogPage.compose` provides one instance through the component context.
Its default is the journal title with `noindex, follow`. A loaded public page
overrides that default. `PostPage.compose` calls `head.bind(PageHead.article(...)*)`
the list and error components bind their own entries. The full
[root setup](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala) and
[article component](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostPage.scala) show those calls in context.

When the router disposes an article component, the callback disposes its handle
and updates the browser head. Returning to an account page therefore restores
the default and removes the article canonical. The component's UI tree stays
together in `compose`; no rendering fragments or manual DOM attribute writes
are needed.

The translated list description uses the existing `i18n` macro. Head entries
accept strings, so this boundary resolves the message once for the route's
locale. Language navigation creates the new route component and its new entries.

## Publish discovery documents from public data

The sitemap and feeds are database views of published content. They cannot
depend on whichever articles happened to be present on a rendered list page.

[PublishedPages](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/application/backend/src/main/scala/com/anjunar/blog/PublishedPages.scala) selects a bounded
projection through Criteria and the entity schemas. The parent must be
`PUBLISHED`; the German join additionally requires a published translation.
A German feed excludes English fallback entries. Neither a draft translation
nor a published translation beneath a draft parent becomes public.

The endpoints are:

| URL | Contents |
| --- | --- |
| `/sitemap.xml` | The two journal roots and all published canonical article variants. |
| `/en/feed.xml` | The latest 20 English articles. |
| `/de/feed.xml` | The latest 20 published German translations, ordered by parent publication time. |
| `/robots.txt` | The sitemap address. |

Language alternatives stay in the HTML head; we do not maintain another
hreflang implementation inside the sitemap. This tutorial bounds a sitemap to
5,000 source posts and returns 503 above that limit rather than silently omitting
articles. A sitemap index for larger installations is outside the series.

Modification times need actual stored data. We add this nullable field and
callback to `BlogPost`; `BlogPostTranslation` receives the same structure with
its own schema ID:

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
import jakarta.json.bind.annotation.JsonbTransient
import jakarta.persistence.{Column, PrePersist, PreUpdate}
import java.time.Instant

// Fields and callback in the body of BlogPost.
  @Column(name = "updated_at") @SchemaId("ab240001") @JsonbTransient
  var updatedAt: Instant = null

  @PrePersist @PreUpdate
  def recordChange(): Unit = updatedAt = Instant.now()
```

Hibernate records the time when a row is inserted or updated. A failed transaction
rolls it back with the other changes. Existing rows remain null because we cannot
reconstruct their editing history. The sitemap omits unknown `lastmod` values.
For a translated page, a known source or translation change can affect its
metadata. These timestamps describe persisted row changes, not a complete audit
of separately managed media, tags or account names.

Atom requires an update time. Legacy entries use publication time until a new
row change is recorded; new changes advance `updated`. Entry IDs use the source
or translation UUID, so editing a slug does not create a different feed identity.
The [sitemap protocol](https://www.sitemaps.org/protocol.html) and
[Atom specification](https://www.rfc-editor.org/rfc/rfc4287) define the formats.

Here is the complete resource. Fixed aliases in `FrontendHandler` send those
public URLs into the ordinary JAX-RS request boundary:

```scala
package com.anjunar.blog

import jakarta.annotation.security.PermitAll
import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.servlet.ServletContext
import jakarta.ws.rs.{GET, Path, PathParam, Produces}
import jakarta.ws.rs.core.{Context, Response}

import java.nio.charset.StandardCharsets.UTF_8
import scala.compiletime.uninitialized

@PermitAll
@RequestScoped
@Path("/discovery")
class DiscoveryResource {
  @Inject var pages: PublishedPages = uninitialized
  @Context var servlet: ServletContext = uninitialized

  private def origin: String = servlet.getAttribute(PublicSite.OriginAttribute).asInstanceOf[String]

  private def response(body: String, mediaType: String): Response =
    Response.ok(body.getBytes(UTF_8), mediaType).header("Cache-Control", "no-store")
      .header("X-Content-Type-Options", "nosniff").build()

  @GET @Path("/sitemap") @Produces(Array("application/xml"))
  def sitemap(): Response = response(DiscoveryXml.sitemap(origin, pages.sitemap()), "application/xml; charset=UTF-8")

  @GET @Path("/feed/{locale}") @Produces(Array("application/atom+xml"))
  def feed(@PathParam("locale") raw: String): Response = {
    val locale = PostLocale.parse(raw)
    response(DiscoveryXml.feed(origin, locale, pages.feed(locale)), "application/atom+xml; charset=UTF-8")
  }

  @GET @Path("/robots") @Produces(Array("text/plain"))
  def robots(): Response = response(s"User-agent: *\nAllow: /\nSitemap: $origin/sitemap.xml\n", "text/plain; charset=UTF-8")
}
```

`DiscoveryXml` uses the JDK XML writer to escape text and attributes. Summaries
are plain text, never injected HTML; invalid XML control characters are replaced.
The response is already a UTF-8 byte array, so the existing transaction boundary
closes the read transaction before sending it. No additional `flush` belongs
in these methods.

The application uses `no-store` here. Retraction is therefore reflected on the
next request. Feed readers may retain entries they downloaded earlier.

## Make HTTP agree with the page

The root and `/index.html` now redirect directly to `/en` with HTTP 308.
Supported article aliases and trailing slashes also lead to one preferred path;
queries are preserved. `/feed.xml` redirects to the English feed. Redirect
destinations are fixed local paths.

Unknown localized routes render the shared error component with HTTP 404.
Draft and missing articles also remain 404. GET and HEAD agree on status and
metadata, but HEAD has no response body. Account and editorial shells send
`X-Robots-Tag: noindex`; their actual protection remains authentication and
authorization.

Before running this revision, stop the backend and migrate:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server frontendAssets "application-backend/run"
```

The migration adds two columns. A second migration applies zero statements.
The read-only preview can report `INCOMPLETE` for the existing PostgreSQL CHECK
predicate; migration verifies it under its locks. The
[chapter guide](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/docs/completing-the-public-pages.md) documents that distinction
and the test environment.

The browser tests inspect the head before JavaScript runs and after client
navigation. They parse the XML, verify draft exclusion and retraction, check
redirects and HEAD, and retain chapter 23's DOM-identity checks. Open a translated
article, switch language, return to the list and enter the account page: each
step should leave exactly the metadata belonging to that page.

This completes the 24-chapter tutorial. Starting from an empty project, we now
have a persistent blog with accounts, editorial workflows, media, translations,
server rendering, hydration and coherent public URLs. The implementation and
its tests remain available at this chapter's checkpoint.
