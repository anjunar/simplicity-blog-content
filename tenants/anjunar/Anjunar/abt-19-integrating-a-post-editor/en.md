# Integrating a Post Editor

Our form already saves a post safely. Now we want to write an actual article:
headings, emphasized text, code examples and images between paragraphs.

We will replace the content textarea with the editor from scalajs-ui. The
interesting part is how that editor joins the system we have already built.
Formatting must survive a save and reload, newer typing must survive a slow
response, and an embedded image must follow the post's visibility.

The [chapter checkpoint](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/docs/integrating-a-post-editor.md)
contains the runnable implementation and upgrade commands. It uses scalajs-ui
1.0.12 from Maven Central, including its English editor controls. The Scala.js
build targets ES2021 for the Markdown parser's regular-expression support.

## Choose the document boundary

The editor keeps a structured document while we work. Its public value is
Markdown: a string that our existing form, JSON mapper and database can carry.
We keep `BlogPost.content` and add a field that tells readers how to interpret it.

Inside [BlogPost](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala):

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.Column
import jakarta.validation.constraints.Pattern

@Pattern(regexp = "PLAIN_TEXT|MARKDOWN")
@Column(name = "content_format", length = 16)
@SchemaId("ad191001") @JsonbProperty
var contentFormat: String = "PLAIN_TEXT"
```

The nullable column is intentional. Historical rows have no format and keep
their plain-text interpretation. A paragraph containing `*asterisks*` must not
suddenly acquire emphasis after a deployment. The migration adds the column
without rewriting existing content.

New posts explicitly submit `MARKDOWN`. Existing posts initially keep their
textarea and offer **Enable rich text**. That action escapes Markdown
punctuation and preserves line breaks before switching formats. Conversion is
an editorial choice.

As with earlier fields, the change crosses the whole contract:
`EntitySchema`, the detail graph, the frontend property and save snapshots.
Lists still use their compact projection; they do not need the full document.

## Bind the editor to the post

We already have `form(post)`, versioned saves and a path for field errors.
The editor is another form control. This excerpt belongs inside
[PostEditorPage.compose](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostEditorPage.scala),
in that existing form:

```scala
import ui.editor.Editor.*
import ui.editor.plugins.*

editor("content") {
  showModeActions = false
  markdownMode = sourceMode
  mediaUrlPolicy = PostMarkdown.mediaPolicy
  mediaUploader = PostMarkdown.uploader(mediaService)
  onMediaStatus = status => embeddedUpload.set(status)
  basePlugin()
  headingPlugin()
  listPlugin()
  linkPlugin()
  imagePlugin()
  codePlugin()
}
```

The name `"content"` binds directly to `post.content`. No second editor model
or HTTP copying step is needed. The complete component also supplies the
accessible label, help text and bound errors.

The plugin selection defines our toolbar: ordinary formatting, headings,
lists, links, images and code blocks. A local **Edit Markdown** button changes
the `sourceMode` property; **Visual editor** switches back without leaving
the form. This is useful for pasting a fenced Scala example or inspecting the
document that will be saved.

Chapter 15's save behavior still applies. We freeze the submitted Markdown
before the request starts. If the author keeps typing, the response advances
the version but only replaces fields that have not changed since submission.
The new format property participates in that comparison too.

Code blocks store code as text. This chapter adds no syntax highlighter; angle
brackets in an example remain part of the example.

## Reuse the upload service

An image in Markdown needs a durable address. A browser object URL would stop
working after the page closes, and embedding the file in the document would
bypass our upload limits and ownership checks.

The editor accepts a `MediaUploader`. Our adapter delegates to chapter 18's
service, which sends the actual file with the session's CSRF token:

```scala
import org.scalajs.dom
import ui.editor.{MediaUploader, UploadedMediaReference}
import scala.concurrent.{ExecutionContext, Future}

// Inside PostMarkdown, in the frontend package.
def uploader(service: MediaService)(using ExecutionContext): MediaUploader =
  new MediaUploader {
    def upload(file: dom.File, signal: dom.AbortSignal)
        : Future[UploadedMediaReference] =
      service.upload(file, signal).map(media =>
        UploadedMediaReference(media.source, media.id.get))
  }
```

The result identifies the stored image and its `/service/media/{UUID}` address.
A matching `MediaUrlPolicy` permits only that route. The same policy applies
when viewing a document, so externally hosted images cannot quietly become
tracking requests.

While an upload is pending, saving and switching to source mode are disabled.
After insertion, the author selects the image and uses **Edit image** to add
alternative text. The server requires a meaningful description even if a
different client bypasses the form.

Uploading alone still creates private, unreferenced media. Saving the post is
what attaches it.

## Derive references from the document

Markdown image syntax is not a trusted entity reference. The server must
determine which images the document actually uses and whether the caller may
attach them.

[PostDocument](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/backend/src/main/scala/com/anjunar/blog/PostDocument.scala)
parses the source with CommonMark. It collects UUIDs from image nodes, checks
alternative text, rejects raw HTML and restricts link destinations. Parsing
matters: image-looking text inside a code fence is an example, not an attachment.

After `change.applyChanges()`, the resource calls
[PostMedia.synchronize](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/backend/src/main/scala/com/anjunar/blog/PostMedia.scala).
These lines resolve the parsed IDs before replacing the stored relation:

```scala
import jakarta.persistence.LockModeType

// ids comes from PostDocument; manager and lifecycle are injected.
val resolved = ids.toSeq.sortBy(_.toString).map { id =>
  val media = manager.find(classOf[Media], id, LockModeType.PESSIMISTIC_WRITE)
  if (media == null || !lifecycle.canUse(media))
    Problem.invalidField("content",
      "An embedded image is unavailable or belongs to another account.")
  media
}

post.inlineMedia.clear()
resolved.foreach(post.inlineMedia.add)
```

The relation lives in `blog_post_media`. It is derived server state, excluded
from JSON input and output. A client sends the document, not a second list
that could contradict it. Invalid or inaccessible references reject the whole
change; text, format and relations commit together.

The media-row lock is the same one used by cleanup. Cleanup rechecks references
after obtaining it. Both cover images and embedded images now protect an upload,
including images in drafts. There is no cascading deletion of shared media.

The parser bounds document size and complexity: 100,000 characters, 10,000
nodes, depth 32 and 20 distinct images. The JSON mapper continues to validate
submitted field constraints, and Hibernate's callbacks check complete-entity
invariants. Publication also requires meaningful content; a document containing
only a horizontal rule does not qualify.

## Share the reading view

The editorial preview and public post both use
[PostContent](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostContent.scala).
For Markdown, it mounts the same editor in read-only mode with the same image
policy. For legacy content, it renders literal text.

There is no public toolbar or writable surface. More importantly, preview and
publication do not have separate rendering implementations that can gradually
disagree about code blocks or images.

Image delivery retains chapter 18's access rules. Signed-out readers receive
bytes only while a published post references the image. Retraction closes
subsequent anonymous requests, unless another published post still uses it.
The Markdown URL itself grants no access.

The browser editor and the server parser have different jobs. We support the
headings, lists, links, code and images produced by this configured editor.
Arbitrary Markdown extensions and raw HTML are outside this contract.

## Follow one article all the way through

After the additive migration, create a post and format a paragraph. Switch to
Markdown, add a fenced code example, then return to the visual editor. Upload
an image, describe it, save and reload.

Open the preview, publish, and read the public page in a signed-out browser.
The formatting, code and image should match. Retract the post and request the
image again: the response should now be 404.

The checkpoint's browser workflow performs those steps against PostgreSQL.
Additional tests cover explicit legacy conversion, newer typing during a slow
save, document errors, private image references and cleanup. The editor now
participates in the same persistence, permissions and lifecycle as the rest
of the application.
