# Building a Form from End to End

The write API from chapter 14 works. Now we will connect it to an editorial
form: enter a title and slug, save a draft, edit it again, and show errors
beside the fields that caused them.

We also need to handle a less obvious case. Someone clicks **Save post** and
continues writing before the response arrives. The server's saved version
must become the basis for the next update, while the newer text stays in the
form. The same rule must work during creation, when the post does not have
an ID yet.

We will build that path through four files: the frontend model, the HTTP
service, the save actions and the page. The complete implementations below
include their imports. Then we will connect the routes and test a delayed
response.

## Start from chapter 14

The starting revision is:

```text
git switch --detach bb212a133dc20f9647ead13d5347be42e5829592
```

To run the completed chapter instead:

```text
git switch --detach 9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b
git restore --source 214eeb6d5a2c6062013d95abf7f990a1cb032cce -- application/frontend/src/main/scala/com/anjunar/blog/frontend/PostEditorPage.scala
sbt --server frontendAssets
sbt --server "application-backend/run"
```

The restore command applies the corrected editor's DSL attribute bindings to
this chapter's snapshot. The editor listing below matches that corrected file;
it does not require the search feature from the following chapter.

Keep the existing development database and bootstrapped administrator. No
schema migration or dependency upgrade is needed. For local HTTP, use
`BLOG_COOKIE_SECURE=false`. Sign in at `/en/account`, then open editorial.
The [companion guide](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/docs/building-a-form-from-end-to-end.md)
contains the database and local configuration.

All four frontend files belong in
`application/frontend/src/main/scala/com/anjunar/blog/frontend/`.
The model and service already exist; we extend them. The actions and page
are new.

| File | What we put there |
| --- | --- |
| [BlogPost.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPost.scala) | Bound properties, client constraints, partial payloads and response merging. |
| [EditorialService.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/EditorialService.scala) | New-post initialization and authenticated POST/PATCH transport. |
| [PostEditorActions.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostEditorActions.scala) | The pending submission, save state and error handling. |
| [PostEditorPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostEditorPage.scala) | The complete form tree, field binding and user feedback. |

The request path will be:

```text
form(post)
  -> PostEditorActions.save
  -> BlogPost.writeBody
  -> EditorialService.save
  -> HttpJson.write
  -> chapter 14's PreparedChange endpoint
  -> BlogPost.mergeSaved
  -> the same bound form
```

## 1. Make the existing model editable

We keep using the frontend `BlogPost` that mirrors the published JPA entity.
Its properties become the form's values; there is no second set of title,
slug, content and summary variables in the page.

Replace the existing contents of [BlogPost.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPost.scala) with this
complete version:

```scala
package com.anjunar.blog.frontend

import ui.core.state.Property
import ui.forms.validators.{NotBlank, Pattern, Size}
import ui.json.{JsonId, JsonIgnore, JsonMapper, JsonProperty}

import scala.annotation.meta.field
import scala.scalajs.js

final class BlogPost {
  @JsonId
  val id: Property[String] = Property("")
  val version: Property[Long] = Property(-1L)
  @(NotBlank @field)()
  @(Size @field)(min = 3, max = 220)
  @(Pattern @field)("^[a-z0-9]+(?:-[a-z0-9]+)*$")
  val slug: Property[String] = Property("")
  @(NotBlank @field)()
  @(Size @field)(min = 3, max = 180)
  val title: Property[String] = Property("")
  // Null preserves the distinction between an omitted list field and an empty draft detail.
  @(Size @field)(max = 100000)
  val content: Property[String] = Property(null)
  @(Size @field)(max = 300)
  val summary: Property[String] = Property(null)
  @JsonIgnore(deserializable = true)
  val status: Property[String] = Property("DRAFT")
  @JsonIgnore(deserializable = true)
  val publishedAt: Property[Option[String]] = Property(None)

  @JsonIgnore()
  def editableFields: Seq[Property[String]] = Seq(slug, title, content, summary)

  @JsonIgnore()
  def snapshot: PostSnapshot = PostSnapshot(slug.get, title.get, content.get, summary.get)

  @JsonIgnore()
  def isDirty: Boolean = editableFields.exists(_.isDirty)

  def writeBody(): js.Dynamic = {
    require(content.get != null, "Load a detail before editing")
    val body = JsonMapper.serialize(this)
    if (id.get.isEmpty) js.special.delete(body, "id")
    // The serializer emits dirty properties. A PATCH precondition is required even when clean.
    if (id.get.nonEmpty) body.updateDynamic("version")(version.get.toDouble)
    if (!js.isUndefined(body.summary) && body.summary.asInstanceOf[String] == "") body.updateDynamic("summary")(null)
    body
  }

  def mergeSaved(saved: BlogPost, submitted: PostSnapshot): Unit = {
    val before = submitted.values
    editableFields.zip(saved.editableFields).zipWithIndex.foreach { case ((current, fresh), index) =>
      val unchangedSinceSubmit = current.get == before(index)
      current.setDefault(fresh.get)
      if (unchangedSinceSubmit) current.set(fresh.get)
    }
    id.set(saved.id.get)
    id.setDefault(saved.id.get)
    version.set(saved.version.get)
    version.setDefault(saved.version.get)
    status.set(saved.status.get)
    publishedAt.set(saved.publishedAt.get)
  }
}

final case class PostSnapshot(slug: String, title: String, content: String, summary: String) {
  def values: Seq[String] = Seq(slug, title, content, summary)
  def value(name: String): Option[String] = name match {
    case "slug" => Some(slug)
    case "title" => Some(title)
    case "content" => Some(content)
    case "summary" => Some(summary)
    case _ => None
  }
}

final class BlogPostData(var data: BlogPost = null,
    @(JsonProperty @field)("$links") var links: Seq[ApiLink] = Seq.empty)

final class BlogPostTable(
    var rows: Seq[BlogPostData] = Seq.empty,
    var size: Long = -1L,
    @(JsonProperty @field)("$links") var links: Seq[ApiLink] = Seq.empty
)
```

The four editable fields carry the same basic limits as the entity:

| Property | Client constraint |
| --- | --- |
| `slug` | Required, 3–220 characters, lowercase words separated by hyphens. |
| `title` | Required, 3–180 characters. |
| `content` | At most 100000 characters; an empty draft is allowed. |
| `summary` | Optional, at most 300 characters. |

These `NotBlank`, `Size` and `Pattern` imports belong to the frontend
form library. The explicit `@field` targets put their metadata on the fields
read by its descriptor. This is separate from the JPA annotations on our
backend entity.

The server still validates supplied values inside the JSON mapper's
`applyChanges()`. Hibernate's existing callbacks validate the complete entity
before persistence. The client constraints give immediate feedback; they do
not add or replace a backend validation pass.

### Why content and summary are nullable strings

The native text controls bind to `Property[String]`. They do not convert
`Option[String]` to and from text, so content and summary now use nullable
string properties. Their names and JSON scalar types still match the entity.

The default `null` on content has a useful meaning: this object might have
come from a list response that omitted the body. We must load its detail
before editing it. A new draft and a loaded empty draft will explicitly use
`""`; the service below performs that initialization.

`publishedAt` remains an `Option[String]` because it is read-only here.
Together with `status`, it can be deserialized but cannot be written through
the frontend mapper. Publishing and retracting remain the dedicated actions
from chapter 13.

### What writeBody sends

`JsonMapper.serialize(this)` uses each property's default to determine
whether it changed. A response loaded through the mapper establishes those
defaults. That gives us the partial update body from chapter 14.

The extra lines in `writeBody` implement three specific transport rules:

- A new object drops its empty ID so the server can generate one.
- An existing object always sends its current version, including zero.
- A summary explicitly changed to an empty string is sent as JSON null.

An unchanged field stays omitted. Empty content stays an empty string because
a draft is allowed to have no body. We do not treat all empty values alike.

For an existing post loaded at version zero, these are the relevant payload
effects; identity/type metadata emitted by the mapper is separate:

| Change in the form | Changed field in the body | Version |
| --- | --- | --- |
| Change title to `Revised title` | `"title": "Revised title"` | `0` |
| Clear a previously nonempty summary | `"summary": null` | `0` |
| Clear previously nonempty content | `"content": ""` | `0` |
| Leave summary untouched | No summary entry | `0` |

### Why mergeSaved updates defaults as well as values

Suppose the title starts as **Original**, and the editor submits **First
revision**. While the request is pending, they type **Second revision**.

`mergeSaved` compares the current field with the submitted snapshot. If
they still match, it accepts the server value. Otherwise, it keeps the newer
input. In both cases, it advances the default to the server's acknowledged
value:

| Moment | Current title | Title default | Version |
| --- | --- | --- | --- |
| Detail loaded | Original | Original | 0 |
| First save sent | First revision | Original | 0 |
| More text typed | Second revision | Original | 0 |
| First response merged | Second revision | First revision | 1 |
| Second save acknowledged | Second revision | Second revision | 2 |

After the first response, the title is still dirty. The next save therefore
sends **Second revision**, now with version 1.

`editableFields` and `PostSnapshot.values` deliberately use the same order.
The snapshot is only immutable comparison state for a submission; the JSON
payload still comes from the bound model.

## 2. Load an editable detail and send the save

The page needs either a newly initialized draft or the full detail of an
existing post. Both also need the operation links supplied by the server.

Here is the complete [EditorialService.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/EditorialService.scala), including its existing
list, detail and publication operations:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom

import scala.concurrent.{ExecutionContext, Future}
import scala.scalajs.js
import scala.scalajs.js.URIUtils.encodeURIComponent

final class EditorialService(accounts: AccountService)(using ExecutionContext) {
  def list(offset: Int, limit: Int, signal: Option[dom.AbortSignal]): Future[BlogPostTable] =
    accounts.session(signal).flatMap { state =>
      state.links.find(_.rel == "editorial") match {
        case None => Future.failed(new HttpFailure(if (state.account.isEmpty) 401 else 403))
        case Some(link) =>
          val target = new dom.URL(link.path("GET"), dom.window.location.origin)
          target.searchParams.set("offset", offset.toString)
          target.searchParams.set("limit", limit.toString)
          HttpJson.get[BlogPostTable](target.pathname + target.search, signal).map { table =>
            require(table.size >= 0 && table.rows.forall(row => row.data != null), "Invalid editorial table")
            table
          }
      }
    }

  def detail(id: String, signal: Option[dom.AbortSignal]): Future[BlogPostData] =
    HttpJson.get[BlogPostData](s"/service/editorial/posts/${encodeURIComponent(id)}", signal).map(validate)

  def newPost(signal: Option[dom.AbortSignal]): Future[BlogPostData] =
    list(0, 1, signal).map { table =>
      val link = table.links.find(_.rel == "create").getOrElse(throw new HttpFailure(403))
      link.path("POST")
      val post = new BlogPost()
      post.content.set("")
      post.content.setDefault("")
      new BlogPostData(post, Seq(link))
    }

  def save(link: ApiLink, body: js.Dynamic): Future[BlogPostData] = {
    val expectedMethod = if (link.rel == "create") "POST" else {
      require(link.rel == "update", "Unexpected save relation")
      "PATCH"
    }
    val path = link.path(expectedMethod)
    accounts.session().flatMap(state =>
      HttpJson.write[BlogPostData](path, expectedMethod, body, state.csrfToken)).map { result =>
        val saved = validate(result)
        require(saved.data.id.get.nonEmpty && saved.data.version.get >= 0, "Missing saved identity or version")
        saved
      }
  }

  def execute(link: ApiLink): Future[BlogPostData] = {
    // Validate before fetching CSRF or sending a command.
    val path = link.path("POST")
    accounts.session().flatMap(state =>
      HttpJson.post[BlogPostData](path, js.Dynamic.literal(), state.csrfToken)).map(validate)
  }

  private def validate(result: BlogPostData): BlogPostData = {
    require(result.data != null, "Missing editorial detail")
    // The backend mapper omits empty strings. A draft detail may legitimately have an empty body.
    if (result.data.status.get == "DRAFT" && result.data.content.get == null)
      { result.data.content.set(""); result.data.content.setDefault("") }
    require(result.data.content.get != null, "Missing editorial content")
    result
  }
}
```

`newPost` obtains the collection's create link before constructing the
model. It initializes content to `""` and sets that as the default. An
untouched empty draft body therefore does not appear as an accidental change.

An existing post comes through `detail`. The backend mapper omits empty
strings, so `validate` restores an omitted body on a draft to `""` and
sets its default too. A published detail must contain content.

The save method chooses POST or PATCH from the link relation and verifies
that the supplied method and destination agree. The existing
`ApiLink.path` guard limits destinations to this application's origin and
`/service/` path. Only then do we obtain the session's CSRF token and send
the already prepared body.

### Add PATCH support to the existing transport

In [HttpJson.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/HttpJson.scala), add this method inside the existing
`HttpJson` object, alongside `get` and `post`:

```scala
  def write[M](path: String, method: String, body: js.Dynamic, csrf: String)(using
      ExecutionContext, JsonSchema[M]): Future[M] = {
    val httpMethod = method match {
      case "POST" => dom.HttpMethod.POST
      case "PATCH" => dom.HttpMethod.PATCH
      case _ => throw new IllegalArgumentException("Unexpected write method")
    }
    request(path, httpMethod, None, Some(body), Some(csrf))
  }
```

The file already imports `org.scalajs.dom`,
`ui.json.{JsonMapper, JsonSchema}`, `scala.concurrent.{ExecutionContext, Future}`
and `scala.scalajs.js`. The method reuses its existing `request` function,
which sends same-origin credentials, the CSRF header and JSON, and maps a
successful response back into the requested model.

That transport also converts unsuccessful responses into `HttpFailure`
with typed problem details. The form will consume those existing errors.

The important ordering is outside this method: the body must be created
*before* `accounts.session()` runs. Otherwise, text entered during the
session lookup could unexpectedly become part of an earlier submission.

## 3. Give each save its own snapshot

Create [PostEditorActions.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostEditorActions.scala) with the following contents:

```scala
package com.anjunar.blog.frontend

import ui.core.state.Property

import scala.concurrent.{ExecutionContext, Future}
import scala.scalajs.js
import scala.util.{Failure, Success, Try}

enum SaveNotice {
  case Idle, Saved, NewerEdits, Invalid, Conflict, SignedOut, Forbidden, Unconfirmed, BindingFailed

  def isError: Boolean = this match {
    case Idle | Saved | NewerEdits => false
    case _ => true
  }
}

final class PostEditorActions(initial: BlogPostData,
    send: (ApiLink, js.Dynamic) => Future[BlogPostData])(using ExecutionContext) {
  val post: BlogPost = initial.data
  val links: Property[Seq[ApiLink]] = Property(initial.links)
  val busy: Property[Boolean] = Property(false)
  val blocked: Property[Boolean] = Property(false)
  val dirty: Property[Boolean] = Property(post.isDirty)
  import SaveNotice.*

  val notice: Property[SaveNotice] = Property(Idle)
  val errors: Property[Seq[FieldError]] = Property(Seq.empty)
  val generalError: Property[String] = Property("")
  private var disposed = false
  private val subscriptions = post.editableFields.map(_.observeWithoutInitial { _ =>
    dirty.set(post.isDirty)
    if (notice.get == Saved || notice.get == NewerEdits) notice.set(Idle)
  })

  def save(onSaved: () => Unit): Unit = {
    if (disposed || busy.get || blocked.get) return
    val relation = if (post.id.get.isEmpty) "create" else "update"
    links.get.find(_.rel == relation).foreach { link =>
      val submitted = post.snapshot
      // Serialize now, before session lookup; later keystrokes belong to the next save.
      val body = post.writeBody()
      busy.set(true)
      notice.set(Idle)
      errors.set(Seq.empty)
      generalError.set("")
      Future.fromTry(Try(send(link, body))).flatten.onComplete { result =>
        if (!disposed) {
          result match {
            case Success(saved) =>
              post.mergeSaved(saved.data, submitted)
              links.set(saved.links)
              dirty.set(post.isDirty)
              blocked.set(!saved.links.exists(_.rel == "update"))
              notice.set(if (dirty.get) NewerEdits else Saved)
              busy.set(false)
              onSaved()
            case Failure(error) =>
              val http = error match { case value: HttpFailure => Some(value); case _ => None }
              val fields = http.flatMap(_.problem).toSeq.flatMap(value => Option(value.errors).getOrElse(Seq.empty))
                .filter(value => value != null && value.path != null && value.message != null)
              val applicable = fields.filter(value => value.path.size == 1 &&
                submitted.value(value.path.head).nonEmpty &&
                submitted.value(value.path.head) == post.snapshot.value(value.path.head))
              errors.set(applicable)
              generalError.set(fields.filter(value => value.path.size != 1 ||
                submitted.value(value.path.head).isEmpty).map(_.message).mkString(" "))
              val status = http.map(_.status).getOrElse(0)
              val fieldConflict = status == 409 && fields.exists(_.path == Seq("slug"))
              val retryable = status == 400 || fieldConflict
              blocked.set(!retryable)
              notice.set(if (retryable) Invalid else status match {
                case 409 => Conflict
                case 401 => SignedOut
                case 403 => Forbidden
                case _ => Unconfirmed
              })
              busy.set(false)
          }
        }
      }
    }
  }

  def dispose(): Unit = {
    disposed = true
    subscriptions.foreach(_.dispose())
  }
}
```

The constructor receives a function that sends a link and a JSON body.
The application supplies `service.save`; the test can supply a delayed
future. Both use exactly the same save logic.

There are three distinct pieces of state:

| State | Meaning |
| --- | --- |
| `busy` | A save is pending; do not submit a second one. |
| `dirty` | Some current text differs from the last acknowledged values. |
| `blocked` | Saving cannot continue safely after the latest result. |

At the beginning of `save`, the action captures `post.snapshot` and calls
`post.writeBody()` synchronously. Keystrokes after this point belong to the
next save. The method also guards against duplicate submission and calls
after disposal.

On success, the action merges values, replaces the links, recalculates
dirty state and ends the pending operation. The page receives either
`Saved` or `NewerEdits`, so it can distinguish “everything is saved” from
“that request succeeded, but you have written more since then.”

### A delayed create must become an update

Creation follows the same path. Before the first POST, the ID is empty and
the action selects the create link. When the response arrives,
`mergeSaved` adopts the server-generated ID even if some current text
must be preserved.

The action also replaces the old links with the response's update link.
Consequently, the next save selects update and sends PATCH. It does not
create a second post just because the editor kept typing during creation.

### Apply errors to the values that caused them

The failure branch first extracts usable field errors. It then distinguishes
three situations:

1. A known field still contains the submitted value. Attach its error.
2. A known field was changed after submission. Do not attach that stale error.
3. The path does not identify a form field. Display the message at form level.

For example, `["slug"]` identifies a control. A class-level constraint such
as `publicationConsistent` has no corresponding input; its message goes to
`generalError`.

HTTP 400 and a slug conflict allow correction and another save. A version
conflict must not silently retry against a newer version, because that would
bypass the user's review of the intervening change. Authentication failures
and unconfirmed network/server outcomes also block another save while
preserving the current inputs.

An unconfirmed POST may already have committed. Stopping automatic retries
prevents a second creation request from being sent without knowing the
outcome of the first.

Finally, `dispose` removes the property observers and prevents a late
response from updating a page that has been left. It does not undo a request
that has already reached the server.

## 4. Build the complete form in compose

Create [PostEditorPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostEditorPage.scala) below. This is the complete component:
all four controls, the submit handler, field errors, save feedback and
navigation stay in the same `compose` tree.

Read the `form(post)` block in three parts: first the error subscription
and submission handler, then the four bound controls, and finally the
feedback and save button.

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom
import ui.core.component.AbstractComponent
import ui.core.dsl.AttributeDsl
import ui.core.dsl.AttributeDsl.*
import ui.core.dsl.ClassDsl.classes
import ui.core.dsl.DslLayer.render
import ui.core.dsl.EventDsl.{on, onClick}
import ui.core.i18n.{I18nRuntime, i18n}
import ui.core.layout.Button.{button, buttonType, disabled, disabled_=}
import ui.core.layout.Condition.when
import ui.core.layout.Div.div
import ui.core.layout.Heading.heading
import ui.core.layout.Label.label
import ui.core.layout.Paragraph.paragraph
import ui.core.layout.TextComponent.text
import ui.core.render.Cursor
import ui.forms.ErrorResponse
import ui.forms.Form.form
import ui.forms.Input.input
import ui.forms.TextAreaInput.textAreaInput
import ui.router.Router
import ui.router.RouterLink.routerLink

import scala.concurrent.ExecutionContext
import scala.scalajs.js

final class PostEditorPage(initial: BlogPostData, service: EditorialService)
    (using ExecutionContext) extends AbstractComponent {
  import SaveNotice.*

  val tagName = "section"
  private val wasNew = initial.data.id.get.isEmpty
  private val actions = new PostEditorActions(initial, service.save)
  private val post = actions.post

  override def compose(cursor: Cursor): Unit = {
    val translations = I18nRuntime.current(using this).get
    addDisposable(() => actions.dispose())
    render(this, cursor) {
      classes = "post-editor"
      paragraph { classes = "eyebrow"; text(i18n"Editorial") {} }
      heading(1) { text(post.id.flatMap(value => translations.text(
        if (value.isEmpty) i18n"New post" else i18n"Edit post"))) {} }
      paragraph { text(i18n"Save your text here. Publication is managed from the preview.") {} }
      form(post) { mountedForm ?=>
        classes = "post-form"
        AttributeDsl.setAttribute("novalidate", "")
        mountedForm.addDisposable(actions.errors.observe(values =>
          mountedForm.setErrorResponses(values.map(value => ErrorResponse(value.message, value.path)))))
        on("submit") { event =>
          event.preventDefault()
          if (!actions.busy.get && !actions.blocked.get) {
            mountedForm.clearErrors()
            actions.generalError.set("")
            val bindings = mountedForm.validateBindings()
            val validation = mountedForm.validate()
            if (bindings.nonEmpty) actions.notice.set(BindingFailed)
            else if (validation.nonEmpty) actions.notice.set(Invalid)
            else actions.save(() => {
              if (wasNew && !actions.dirty.get) Router.replace(s"/editorial/posts/${post.id.get}/edit")
            })
          }
        }
        div {
          classes = "post-field"
          label { AttributeDsl.setAttribute("for", "post-title"); text(i18n"Title") {} }
          val control = input("title") { fieldInput ?=>
            id = "post-title"
            AttributeDsl.setAttribute("aria-describedby", "post-title-errors")
            fieldInput.addDisposable(fieldInput.invalid.observe(value =>
              AttributeDsl.setAttribute("aria-invalid", value.toString)))
          }
          paragraph { id = "post-title-errors"; classes = "field-error"; text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {} }
        }
        div {
          classes = "post-field"
          label { AttributeDsl.setAttribute("for", "post-slug"); text(i18n"Slug") {} }
          val control = input("slug") { fieldInput ?=>
            id = "post-slug"
            spellCheck = false
            AttributeDsl.setAttribute("aria-describedby", "post-slug-help post-slug-errors")
            fieldInput.addDisposable(fieldInput.invalid.observe(value =>
              AttributeDsl.setAttribute("aria-invalid", value.toString)))
          }
          paragraph { id = "post-slug-help"; classes = "field-help"; text(i18n"Use lowercase words separated by hyphens. Changing this changes the public URL.") {} }
          paragraph { id = "post-slug-errors"; classes = "field-error"; text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {} }
        }
        div {
          classes = "post-field"
          label { AttributeDsl.setAttribute("for", "post-summary"); text(i18n"Summary (optional)") {} }
          val control = textAreaInput("summary") { fieldInput ?=>
            id = "post-summary"
            AttributeDsl.setAttribute("rows", "3")
            AttributeDsl.setAttribute("aria-describedby", "post-summary-help post-summary-errors")
            fieldInput.addDisposable(fieldInput.invalid.observe(value =>
              AttributeDsl.setAttribute("aria-invalid", value.toString)))
          }
          paragraph { id = "post-summary-help"; classes = "field-help"; text(i18n"Up to 300 characters. Leave it empty to clear the summary.") {} }
          paragraph { id = "post-summary-errors"; classes = "field-error"; text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {} }
        }
        div {
          classes = "post-field"
          label { AttributeDsl.setAttribute("for", "post-content"); text(i18n"Content") {} }
          val control = textAreaInput("content") { fieldInput ?=>
            id = "post-content"
            classes = "post-content-input"
            AttributeDsl.setAttribute("rows", "12")
            AttributeDsl.setAttribute("aria-describedby", "post-content-help post-content-errors")
            fieldInput.addDisposable(fieldInput.invalid.observe(value =>
              AttributeDsl.setAttribute("aria-invalid", value.toString)))
          }
          paragraph { id = "post-content-help"; classes = "field-help"; text(i18n"Plain text for now. A draft may be empty; a published post needs content.") {} }
          paragraph { id = "post-content-errors"; classes = "field-error"; text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {} }
        }
        when(actions.notice.map(_.isError)) {
          div {
            role = "alert"
            classes = "form-error"
            paragraph {
              text(actions.notice.flatMap(value => translations.text(value match {
                case Invalid => i18n"Review the fields and save again."
                case Conflict => i18n"This post changed elsewhere. Your edits are still here. Copy what you need before discarding and reloading."
                case SignedOut => i18n"Your session ended. Your edits are still here. Sign in again before reloading."
                case Forbidden => i18n"You can no longer save this post. Your edits are still here."
                case BindingFailed => i18n"The form could not bind its fields. Saving is unavailable."
                case _ => i18n"The save could not be confirmed. Check the editorial list before trying again."
              }))) {}
            }
            when(actions.generalError.map(_.nonEmpty)) { paragraph { text(actions.generalError) {} } }
          }
        }
        div {
          classes = "editorial-controls"
          button(i18n"Save post") {
            buttonType("submit")
            disabled = actions.busy.flatMap(busy => actions.blocked.map(blocked => busy || blocked))
          }
          when(actions.busy) { paragraph { role = "status"; text(i18n"Saving… You can keep writing.") {} } }
          when(actions.notice.map(value => value == Saved || value == NewerEdits)) {
            paragraph {
              role = "status"
              text(actions.notice.flatMap(value => translations.text(
                if (value == Saved) i18n"Saved." else i18n"Saved. Your newer edits still need saving."))) {}
            }
          }
        }
      }
      div {
        classes = "editorial-controls"
        when(post.id.map(_.nonEmpty)) {
          routerLink(s"/editorial/posts/${post.id.get}") { text(i18n"Open preview") {} }
          when(actions.blocked) {
            button(i18n"Discard my edits and reload") {
              buttonType("button")
              onClick { _ =>
                if (cursor.isBrowser) dom.window.location.assign(Router.requireCurrent.hrefFor(s"/editorial/posts/${post.id.get}/edit"))
              }
            }
          }
        }
        routerLink("/editorial") { text(i18n"Back to editorial") {} }
        routerLink("/account") { text(i18n"Your account") {} }
      }
    }
  }
}
```

### Follow one edit through the form

When someone types into `input("title")`, the form binding updates
`post.title`. No event handler copies the input into a separate DTO.

The native submit event handles both the save button and keyboard submission.
The handler clears earlier errors, then checks:

- `validateBindings()`: does each control bind to the named model property?
- `validate()`: do the current bound values satisfy their constraints?

A misspelled name must not produce an apparently editable control that never
updates the model. Binding errors stop the save, just like invalid values.
The form uses `novalidate` so its own validation and error presentation
handle submission consistently.

Only after both checks succeed does `actions.save` run. The save button
becomes disabled while `busy` is true; the inputs remain editable.

### Connect field errors to the actual controls

The subscription near the top of the form calls
`mountedForm.setErrorResponses`. It converts the already filtered server
errors to the form library's `ErrorResponse(message, path)`. A path such
as `Seq("title")` reaches the same title control used for client validation.

Each field has a label, a matching input ID and an error paragraph referenced
by `aria-describedby`. The input's `aria-invalid` value follows its
validation state. The visible error text comes from that control's
`errors` property.

Attributes belong to the DSL. We import `ui.core.dsl.AttributeDsl` and call
`AttributeDsl.setAttribute(...)` inside each element's block. The outer
page also inherits a `setAttribute` method; naming the imported DSL object
makes the choice explicit and uses the component supplied by the current
block. This applies to labels, controls and the form's `novalidate`.

The `aria-invalid` observer stays inside the input or textarea block, so
its DSL call captures that control. Registering it with `fieldInput.addDisposable`
removes the observer with the control. Submit events use the DSL's `on("submit")`.

All observers use the component's disposal mechanism. The DSL tree remains
together in `compose`; validation and transport logic live in the model,
actions and service.

### Keep a newly created form open if it still has edits

`wasNew` records which route created the component. After a successful
save, the page replaces the new-post URL with the post's edit URL only when
`actions.dirty` is false.

If someone typed during creation, the current form stays mounted. Its model
already has the saved ID and version, so the next save is an update. Once
those newer edits are acknowledged, the page can change the route without
discarding them.

For an existing post with a conflict, **Discard my edits and reload** is an
explicit choice to load fresh server state. It uses a real browser
navigation even if the edit URL is unchanged. The `cursor.isBrowser`
guard keeps this browser API on the browser path.

There is no autosave or offline draft storage in this chapter. Leaving the
page or reloading discards unsaved text. The conflict message tells the
editor to copy anything they want to retain before discarding.

## 5. Connect the routes and editorial links

In [BlogRoutes.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogRoutes.scala), add these entries to the existing route
sequence before the post preview route:

```scala
    Route.view("/editorial/new") { context =>
      editorial.newPost(context.signal).map(value => new PostEditorPage(value, editorial))
    },
    Route.view("/editorial/posts/:id/edit") { context =>
      editorial.detail(context.pathParams("id"), context.signal).map { value =>
        val link = value.links.find(_.rel == "update").getOrElse(throw new HttpFailure(403))
        link.path("PATCH")
        new PostEditorPage(value, editorial)
      }
    },
```

`BlogRoutes` already imports `Route` from `ui.router` and owns
`editorial = new EditorialService(accounts)`. Its existing error routes
handle 401 and 403 failures. The router supplies the locale prefix, so these
paths are relative to `/en` in the browser.

The new-post loader checks the collection's create link. The edit route
requires and validates an update link before showing the form. Those checks
control what the UI offers; the server still authorizes every request.

There are three small integration changes around these routes:

| File | Change |
| --- | --- |
| [EditorialListPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/EditorialListPage.scala) | Show **New post**, pointing to `/editorial/new`, when the collection offers create. |
| [EditorialPostPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/scala/com/anjunar/blog/frontend/EditorialPostPage.scala) | Show **Edit post**, pointing to the post's edit route, when its detail offers update. |
| [FrontendHandler.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/backend/src/main/scala/com/anjunar/blog/FrontendHandler.scala) | Serve the browser shell for `/en/editorial/new` and the existing editorial post path with an optional `/edit` suffix, retaining `no-store`. |

The last change is necessary for reloads and direct links. A client-side route
can work after navigation inside the app while a direct browser request
still receives 404 if the HTTP handler does not recognize it.

Because content and summary now use nullable strings, the existing read views
also render their optional text through `Option(value).getOrElse("")`.
The public detail loader still requires content. The checkpoint includes
these adaptations and the [form styles](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/main/resources/style.css).

## 6. Test typing while a response is pending

A test that only enters text and receives an immediate response cannot prove
that newer input survives. We need to control when the response arrives.

The following excerpt from [PostEditorSpec.scala](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/application/frontend/src/test/scala/com/anjunar/blog/frontend/PostEditorSpec.scala) includes the imports,
fixtures and one complete test. The repository file contains the other
cases as well.

```scala
package com.anjunar.blog.frontend

import org.scalatest.funsuite.AnyFunSuite
import ui.json.JsonMapper

import scala.concurrent.{ExecutionContext, Promise}
import scala.scalajs.js

class PostEditorSpec extends AnyFunSuite {
  private given ExecutionContext = ExecutionContext.parasitic
  private val id = "8a1c1582-e841-4a27-a506-1a630337df48"
  private def link(rel: String = "update"): ApiLink =
    new ApiLink(rel, "/service/editorial/posts/" + id, if (rel == "create") "POST" else "PATCH")

  private def post(version: Int = 0): BlogPost =
    JsonMapper.deserialize[BlogPost](js.JSON.parse(
      s"""{"id":"$id","version":$version,"slug":"first-post","title":"First title",
         |"content":"Original content","summary":"Original summary","status":"DRAFT"}""".stripMargin))

  test("pending saves suppress duplicates and retain text typed after submission") {
    val pending = Promise[BlogPostData]()
    var calls = 0
    var body: js.Dynamic = null
    val value = post()
    val actions = new PostEditorActions(new BlogPostData(value, Seq(link())), (_, request) => {
      calls += 1; body = request; pending.future
    })
    value.title.set("Sent title")
    actions.save(() => ())
    value.title.set("Newer title")
    actions.save(() => ())
    assert(calls == 1 && body.title.asInstanceOf[String] == "Sent title" && actions.busy.get)
    val saved = post(1)
    saved.title.set("Sent title")
    pending.success(new BlogPostData(saved, Seq(link())))
    assert(value.title.get == "Newer title" && value.version.get == 1)
    assert(!actions.busy.get && actions.notice.get == SaveNotice.NewerEdits && actions.dirty.get)
    actions.dispose()
  }

}
```

The test deliberately leaves the promise incomplete while it changes the
title and attempts another save. Its assertions check three different things:

- Only one request was sent, with the title captured at submission time.
- The delayed response advanced the version without replacing the newer title.
- The form left its busy state and still reports unsaved changes.

`ExecutionContext.parasitic` makes the small completion callbacks run
synchronously here. We can complete the promise and inspect the result
without a sleep or timing assumption.

Other cases in the same suite check cleared fields, version zero, read-only
publication fields, late field errors, version conflicts and disposal.
The creation case also checks that the sequence of operations is
`create`, then `update`.

## Try the complete workflow

Build the assets and start the backend with the earlier local configuration.
Sign in at `/en/account` and open editorial.

1. Choose **New post** and enter a valid title and slug. Add summary and content.
2. Save. The response creates version zero and the browser opens its edit route.
3. Change the title and clear the summary. Save and reload: the new title
   persists and the summary remains empty.
4. Open the same edit route in two tabs. Save a change in the first, then
   save different text from the second.
5. The second tab keeps its text, explains the conflict and disables saving.
   **Discard my edits and reload** adopts the current server values.
6. Open preview to use the existing publication controls.

The [real browser workflow](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/tests/browser/post-editor-database.spec.mjs) performs creation,
editing, summary clearing and a stale-version conflict against PostgreSQL.
The controlled browser tests also delay responses, verify field errors,
check operation links and exercise the mobile layout.

```text
sbt --server "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
npx playwright test --project=forms
```

The forms project requires the dedicated test administrator, migrated
database and psql configuration from the companion guide. It deletes its own
created post afterwards. Run it only against the tutorial's test database.

At this checkpoint, verification passed 21 Scala.js tests, 18 affected
backend integration tests and all 49 browser tests. That browser total
includes 42 controlled contracts and seven real workflows; the commands
above select the contracts and form workflow.

We now have the full editing path: the same model feeds the controls and
mapper, each submission has a stable snapshot, and its response updates the
form without erasing later input. Chapter 16 adds search, filtering and
pagination to make the growing post list easier to navigate.
