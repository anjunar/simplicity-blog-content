# Translating Blog Content

Chapter 20 translated the interface. Now an administrator should be able to write
a German article, save it privately and publish it when ready. Until then, readers
at `/de/posts/shared-slug` continue to see the published English article. Both
languages share the same slug.

We will follow that change from its data model through saving to the public page.
UI messages still use the `i18n` catalog; article text comes from the database.
The [checkpoint guide](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/docs/translating-blog-content.md)
contains setup and verification commands.

## Separate shared identity from translated text

English already lives in `BlogPost`. We keep those fields and add
`BlogPostTranslation` for German. This gives a translation its own version and
publication state without migrating existing English content.

| Stored on BlogPost | Stored on BlogPostTranslation |
| --- | --- |
| Slug, author, tags, cover | Required parent reference and locale |
| English title, summary and content | German title, optional summary and Markdown content |
| Source version and publication state | Translation version and publication flag |

The [translation entity and schema](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/BlogPostTranslation.scala)
define this contract together. A required `@ManyToOne` maps `post_id`; the unique
key `(post_id, locale)` allows only one German row per post. Its own `@Version`
protects German edits: changing the English title does not advance the German
version. The translation starts with `published = false`.

The mapper schema exposes `title`, `summary` and `content` through
`TranslationEditRule`. Parent, locale, version and publication cannot be assigned
freely from JSON. Persistent fields use typed `reference` properties so the same
schema supplies the Criteria attributes used below. The detail graph explicitly
includes `version`, and the frontend model preserves it for the next request.

With the application stopped, run `SchemaMain preview`, inspect it, then run
`SchemaMain migrate`. This adds the translation and image-reference tables,
uniqueness and foreign keys. Existing English rows stay in place.

## Follow one edit from request to response

Suppose a saved German draft has version 0. Editing its title sends this body to
the response's `update` link:

```json
{
  "version": 0,
  "title": "Eine gemeinsame Anwendung"
}
```

The URL is
`PATCH /service/editorial/posts/{postId}/translations/{id}`.
Here `postId` identifies the English parent and `id` identifies the translation.
Omitted summary and content stay unchanged. The UUID remains in the URL, but the
method receives `post: BlogPost`: our
[EntityParamConverterProvider](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/EntityParamConverterProvider.scala)
loads the managed parent in the request's persistence context. Invalid or missing
parents produce 404 before the method runs.

Below are the class declaration, complete update method and response builder from
[EditorialTranslationsResource](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/EditorialTranslationsResource.scala).
The read, create and publication methods remain in the complete resource; the
LinkBuilder expressions below refer to those methods. The application's `Data`, `Schema`, `EntityGraph` and domain classes live in the same
package; external types are imported.

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.PreparedChange
import jakarta.annotation.security.RolesAllowed
import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.ws.rs.{Consumes, NotFoundException, PATCH, Path, PathParam, Produces}
import jakarta.ws.rs.core.MediaType

import scala.compiletime.uninitialized
import scala.jdk.CollectionConverters.*

@Path("/editorial/posts/{postId}/translations")
@RolesAllowed(Array("ADMIN"))
@RequestScoped
@Produces(Array(MediaType.APPLICATION_JSON))
class EditorialTranslationsResource {
  @Inject var manager: EntityManager = uninitialized
  @Inject var media: PostMedia = uninitialized

  // Other endpoints remain in this class; see the complete resource.

  @PATCH @Path("/{id}") @Consumes(Array(MediaType.APPLICATION_JSON))
  @EntityGraph("BlogPostTranslation.detail")
  def update(@PathParam("postId") post: BlogPost,
      @PathParam("id") change: PreparedChange[BlogPostTranslation]): Data[BlogPostTranslation] = {
    val translation = change.getEntity()
    if (translation.post.id != post.id) throw new NotFoundException()
    change.applyChanges()
    media.synchronize(translation)
    result(translation)
  }

  private def result(value: BlogPostTranslation): Data[BlogPostTranslation] = {
    val self = LinkBuilder.create[EditorialTranslationsResource](_.read(value.post)).withRel("self").build()
    val actions = if (value.id == null) {
      Seq(LinkBuilder.create[EditorialTranslationsResource](_.create(value.post, null)).build())
    } else {
      val update = LinkBuilder.create[EditorialTranslationsResource](_.update(value.post, null))
        .withVariable("id", value.id).build()
      val publication = if (value.published) {
        Some(LinkBuilder.create[EditorialTranslationsResource](_.retract(value.post, null))
          .withVariable("id", value.id).build())
      } else if (PostDocument.hasContent(value.content, "MARKDOWN")) {
        Some(LinkBuilder.create[EditorialTranslationsResource](_.publish(value.post, null))
          .withVariable("id", value.id).build())
      } else None
      Seq(update) ++ publication
    }
    val links = (Seq(self) ++ actions).filter(_ != null).asJava
    new Data(value, Schema.forGraph(BlogPostTranslation.schema, manager.getEntityGraph("BlogPostTranslation.detail")), links)
  }
}
```

There are three separate responsibilities here:

1. **Authorize and prepare.** The existing security filter enforces ADMIN and
   CSRF. Our [PreparedChangeParamConverter](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/PreparedChangeProviders.scala)
   then reads the path ID and JSON body. It loads and locks the translation,
   checks the supplied version and prepares the mapper operation. This is our
   registered converter, not built-in JAX-RS body handling.
2. **Check the URL, then apply.** `getEntity()` returns the managed entity before
   the prepared changes are applied. A translation belonging to another post
   produces 404, even when its ID is otherwise valid. Comparing its parent
   identity with `post.id` checks that relationship, not user ownership.
   `applyChanges()` runs once and applies the mapper's field rules and validation.
3. **Maintain derived state and describe the response.**
   `media.synchronize` parses the translated Markdown and rebuilds its internal
   image references, checking image access. `result` wraps the same entity with
   graph metadata and current action links. Neither call commits the transaction.

[LinkBuilder](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/LinkBuilder.scala) is adapted from the reference
stack. `create[EditorialTranslationsResource](_.read(value.post))` describes a
method call for the macro; it does not execute that endpoint. The builder derives
the path and HTTP method from its annotations and binds the parent entity's ID.
For `update`, `null` stands in for the prepared change, while
`withVariable("id", value.id)` supplies the saved translation ID. No JSON body is
read while building links.

The builder checks the endpoint's access policy for the current caller. The
resource decides whether the current state permits create, publish or retract.
Endpoints still enforce those checks when a request follows a link. Routes and
verbs are therefore defined on the resource methods once.

There is no controller `flush()`. The existing
[TransactionBoundary](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/TransactionBoundary.scala) flushes successful
writes **before JSON serialization**. That sends pending SQL, runs Hibernate's
whole-entity validation and advances the version. `Data` still holds the managed
entity, so the writer sees the updated version rather than an earlier copy.

The writer serializes into a buffer; the transaction commits before that success
body is sent. A validation, serialization or commit failure takes the existing
error/rollback path. Flush and commit are different operations, and repeating
flush inside every endpoint would obscure this shared lifecycle.

For the changed title above, a successful response contains version 1. Sending
version 0 again yields 409 and leaves the saved title intact. Reloading the
translation returns the same version as the successful response.

Creation uses the same mapper path for a new entity, sets its parent on the server
and persists it. It locks the parent before checking for an existing translation:
two simultaneous creates must not both succeed. Publish and retract are separate
commands carrying only the saved version; they do not save unfinished form input.

## Select a complete language for readers

Saving a German draft must not expose it. The public detail endpoint first loads
a published parent. It then calls `select(post, locale)`, with the requested
locale already checked as `en` or `de`. This is the complete
[PostLocalization](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/PostLocalization.scala) class:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager

import scala.compiletime.uninitialized

@RequestScoped
class PostLocalization {
  @Inject var manager: EntityManager = uninitialized

  // Only response-only properties change. The managed English fields remain untouched.
  def select(post: BlogPost, locale: String): Unit = {
    val builder = manager.getCriteriaBuilder
    val query = builder.createQuery(classOf[BlogPostTranslation])
    val translation = query.from(classOf[BlogPostTranslation])
    query.select(translation).where(
      builder.equal(translation.get(BlogPostTranslation.schema.post), post),
      builder.equal(translation.get(BlogPostTranslation.schema.locale), "de"),
      builder.equal(translation.get(BlogPostTranslation.schema.published), true))
    val german = Option(manager.createQuery(query).getSingleResultOrNull)
    BlogPost.schema.translation // Register the nested response field after both schemas exist.
    post.translation = if (locale == "de") german.orNull else null
    post.contentLocale = if (post.translation == null) "en" else "de"
    post.availableLocales.clear()
    post.availableLocales.add("en")
    if (german.nonEmpty) post.availableLocales.add("de")
  }
}
```

The query admits only the published German row. On a German request, that row
becomes the selected `translation`; otherwise the selection is null and
`contentLocale` is English. Querying German on English requests also lets us
report which published languages are available.

All assignments target transient response fields. We never overwrite the
managed English title or content with German text. The frontend reads the nested
translation when present and otherwise reads the original fields.

`BlogPost.schema.translation` registers the lazy nested response property after
both persistent schemas exist. JSON Mapper 1.1.6 would otherwise follow the
parent/translation schemas in a cycle. The [guide](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/docs/translating-blog-content.md)
explains this initialization detail; it does not change the selection rule.

| Parent | German translation | German public page |
| --- | --- | --- |
| Published | Published | Complete German article |
| Published | Missing or draft | Complete English article with a notice |
| Draft | Any state | 404 |

Fallback chooses a whole article. A deliberately absent German summary stays
absent; borrowing the English summary would mix languages. The editorial endpoint
has no fallback: it returns the actual German draft or an empty new form.

## Bind the form and declare its content language

[TranslationEditorPage](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/frontend/src/main/scala/com/anjunar/blog/frontend/TranslationEditorPage.scala) receives
the English `post`, the loaded `initial: TranslationData`, and its HTTP/media
services. It creates `actions = new TranslationActions(initial, service.send)`
and binds `translation = actions.translation`. The actions own save state,
server errors and the merge of acknowledged values.

This contiguous excerpt begins at `form(translation)` inside the component's
`compose`/`render` block. It includes submission, server-error binding, the title
control and its error output. Summary, Markdown and buttons continue in that same
form in the linked component.

```scala
import ui.core.dsl.AttributeDsl.{setAttribute as attr}
import ui.core.dsl.AttributeDsl.*
import ui.core.dsl.ClassDsl.classes
import ui.core.dsl.EventDsl.on
import ui.core.i18n.i18n
import ui.core.layout.Div.div
import ui.core.layout.Label.label
import ui.core.layout.Paragraph.paragraph
import ui.core.layout.TextComponent.text
import ui.forms.ErrorResponse
import ui.forms.Form.form
import ui.forms.Input.input

import scala.scalajs.js

// Inside TranslationEditorPage.compose, within render(this, cursor):
form(translation) { mountedForm ?=>
  classes = "post-form"
  attr("novalidate", "")
  mountedForm.addDisposable(actions.errors.observe(values =>
    mountedForm.setErrorResponses(values.map(value => ErrorResponse(value.message, value.path)))))
  on("submit") { event =>
    event.preventDefault()
    if (!actions.busy.get && !actions.blocked.get && upload.get.pending == 0) {
      mountedForm.clearErrors()
      actions.generalError.set("")
      if (mountedForm.validateBindings().nonEmpty) actions.notice.set(SaveNotice.BindingFailed)
      else if (mountedForm.validate().nonEmpty) actions.notice.set(SaveNotice.Invalid)
      else actions.save()
    }
  }
  div {
    classes = "post-field"
    label {
      attr("for", "translation-title")
      text(i18n"Title") {}
    }
    val control = input("title") { fieldInput ?=>
      attr("aria-invalid", fieldInput.invalid.map(_.toString))
      id = "translation-title"
      lang = translation.locale.get
      attr("aria-describedby", "translation-title-errors")
    }
    paragraph {
      id = "translation-title-errors"
      classes = "field-error"
      text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {}
    }
  }
  // Summary, Markdown editor and submit/publication controls continue here.
}
```

`input("title")` binds the model property directly. The label's
`i18n"Title"` follows the interface language. In contrast,
`lang = translation.locale.get` uses the named attribute DSL to declare the
language of the entered article text. This locale is fixed for the loaded
translation, so reading it once is appropriate. Switching the UI language
remounts the page; it does not change the translation being edited.

An English interface therefore shows **Title** around a German input. A German
interface shows **Titel**, still around the same German input. The English source
panel uses `lang = "en"`. Public headings and content use the selected
`contentLocale`, including English fallback inside a German interface.
The HTML attribute describes language; it does not perform translation.

The import renames the DSL's `setAttribute` to `attr`, avoiding the inherited
component method with the same name. Thus `attr("novalidate", "")` belongs to the
form node and `attr("for", ...)` belongs to the label. This is an import alias,
not a custom attribute helper. The `aria-invalid` binding passes the reactive
property directly to the DSL, which owns its subscription and cleanup.

Before saving, the form checks binding and model errors. Actions snapshot the
changed values and version before asynchronous work. A delayed reply updates
version, links and saved baselines while retaining anything typed more recently.
A conflict keeps the text visible and requires an explicit reload; it does not
blindly retry. Dirty input or pending uploads also block publication and the
language-switch buttons.

## Keep discovery and media consistent

German search must match the text readers will see. Translating results only
after pagination would give incorrect ordering and totals.
[LocalizedPostFields](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/LocalizedPostFields.scala) supplies the
existing `HibernateSearch` with one left join to the published translation and
shared CASE expressions for title/summary. Filtering, sorting and projection
select German when that row exists; count builds the equivalent predicate on its
own root. Page links retain locale. A null German summary still stays null.

Image visibility also follows publication: an image referenced only by a German
draft remains private, even when its English parent is public. Both publications
must be active to expose that reference. Saved drafts still protect their images
from cleanup.

## Verify the complete workflow

Save a German draft for a published English post and reload the editor. Its German
public URL should still show English. Publish the translation, search for its
German title, then retract it and check the English fallback again. Repeat the
editor workflow in both interface languages.

The automated checks additionally exercise stale versions, mismatched parent
URLs, validation rollback, permissions, concurrent creation and image cleanup.
The checkpoint guide gives the commands.

Editing an already published translation changes the public text immediately;
retract it first for private revisions. Translation remains manual, source edits
do not invalidate German automatically, and slug/shared metadata stay common.

Chapter 22 will render these localized pages on the server.
