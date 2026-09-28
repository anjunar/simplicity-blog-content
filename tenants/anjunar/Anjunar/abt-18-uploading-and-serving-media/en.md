# Uploading and Serving Media

Our post editor can save text, authors and tags. Now we will give a post a cover
image: choose a file, see a preview, add a description and publish it.

That small feature crosses three boundaries. A file must become a valid image,
the image must become a post reference, and each download must respect the
post's visibility. Keeping those steps explicit also gives us a way to clean
up uploads that never reach a saved post.

The [chapter checkpoint](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/docs/uploading-and-serving-media.md)
contains the complete implementation and setup instructions. We build on chapter
17's reference handling; this chapter concentrates on what changes for media.

## Give the image its own lifetime

Uploading creates a `Media` entity with an ID, owner, creation time, dimensions,
content type and bytes. PostgreSQL stores the bytes in a `bytea` column alongside
the metadata, so both commit or roll back together. There is no second file store
to coordinate.

A post references that entity. Its alternative text belongs to the post because
the same image can serve different purposes in different articles. Inside
[BlogPost](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala),
the new fields are:

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.{Column, FetchType, ForeignKey, JoinColumn, ManyToOne}
import jakarta.validation.constraints.Size

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "cover_image_id",
  foreignKey = new ForeignKey(name = "fk_blog_post_cover"))
@SchemaId("ad182001") @JsonbProperty
var coverImage: Media = null

@Size(max = 300) @Column(name = "cover_alt", length = 300)
@SchemaId("ad182002") @JsonbProperty
var coverAlt: String = null
```

Both fields are nullable so existing posts remain valid. A complete-entity
constraint requires a nonblank description whenever an image is attached.

There is no cascading deletion: removing the cover from one post must not delete
an image used elsewhere. The existing CDI extension discovers `Media`, and
Hibernate DDL Manager adds its table, the two post columns and the foreign key.

## Turn the uploaded file into pixels

The browser sends a `File` directly as the body of
`POST /service/editorial/media`, with `image/jpeg` or `image/png` as its content
type. The endpoint requires an administrator and the session's CSRF token.
An optional filename is only a display label.

The content-type header is a claim from the client. We verify the actual format,
inspect dimensions, decode the image and encode its pixels into a new JPEG or
PNG. This removes source metadata and trailing content before storage.

The first boundary is a bounded read. These lines are inside
[ImageContent.read](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/ImageContent.scala),
after checking the declared type and acquiring a decoder slot:

```scala
import java.io.InputStream

// input: InputStream; MaxBytes = 5 * 1024 * 1024
val bytes = input.readNBytes(MaxBytes + 1)
if (bytes.length > MaxBytes)
  throw new ApiProblem(413, "Images may be at most 5 MiB.")
if (bytes.isEmpty)
  Problem.invalidField("coverImage", "Choose a nonempty image.")
decode(bytes, contentType)
```

Reading one extra byte distinguishes an allowed file from an oversized one
without reading the entire request. The decoder also checks a maximum of
12 million pixels and 8000 pixels per side **before** allocating the decoded
image. Compressed file size alone does not bound decoded memory.

Only two decoders run concurrently, and normalized output is capped at 8 MiB.
Undertow's listener allows enough room for the upload while our reader enforces
the smaller application limit. The
[guide](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/docs/uploading-and-serving-media.md)
records those limits and their tests.

This first version stores still images without resizing or thumbnails. It also
does not correct EXIF orientation; export photographs with the intended pixel
orientation before uploading.

## Save a reference, not another upload

A successful upload returns `Data[Media]`: ID, version, name, content type,
dimensions and byte count. Its entity graph excludes ownership and binary data.
The frontend mapper turns that metadata into a `Media` object held by
`BlogPost.coverImage`.

Uploading does not save the post. When the editor presses **Save post**, the
existing `PreparedChange[BlogPost]` path receives an ID-only reference:

```json
{
  "version": 0,
  "coverImage": { "id": "896d0270-b13b-4735-bac6-3606a0407cc0" },
  "coverAlt": "A blue notebook beside a laptop"
}
```

Use the ID returned by the upload. Omitting `coverImage` preserves the current
selection; sending `null` removes it. Including nested fields such as `name`
does not rename the media—it rejects the request.

[ReferenceAccess](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/ReferenceAccess.scala)
loads and authorizes the image before the mapper applies the change. An unused
upload belongs to its uploader. Once a post references it, other administrators
may use it too, matching our shared editorial permissions. Guessing an unused
upload's ID therefore gives another editor no access.

The form keeps the upload and save states separate. Save waits for the upload,
then sends the selected ID and description. Removing a pending image aborts its
request and ignores a late response. If a post save finishes after another image
was selected, the merge preserves that newer selection, just as chapter 15
preserves newer typing.

## Authorize every image request

An image URL must not bypass draft visibility. Our policy is:

| Caller | Images they can read |
| --- | --- |
| Anonymous visitor or reader | Images referenced by at least one published post. |
| Administrator | Their own uploads and images referenced by any post, including drafts. |

Inside
[MediaResource.image](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/MediaResource.scala),
after parsing the UUID, the read path is:

```scala
import jakarta.ws.rs.NotFoundException
import jakarta.ws.rs.core.Response

val media = manager.find(classOf[Media], id)
if (media == null ||
    (!lifecycle.referenced(media, publishedOnly = true) && !lifecycle.canUse(media)))
  throw new NotFoundException()

Response.ok(media.data, media.contentType)
  .header("Content-Length", media.byteSize)
  .header("Content-Disposition", "inline")
  .header("Cache-Control", "no-store")
  .header("X-Content-Type-Options", "nosniff")
  .build()
```

A hidden image and a missing image both return 404. Retracting the last published
post using an image closes public access on subsequent requests. `no-store`
prevents the browser from treating those bytes as a permanently public asset;
it cannot erase a copy already downloaded.

There is also a transaction change. JSON serialization may still need Hibernate
relationships, so its transaction stays open through the writer. Here the
controller already has a bounded `Array[Byte]`. `TransactionBoundary` finishes
that transaction before writing the image, freeing the database connection
before a slow client downloads it.

The public page uses the same component DSL as the rest of the application.
This excerpt belongs inside `PostPage.compose`'s existing `render` block:

```scala
import ui.core.dsl.ClassDsl.classes
import ui.core.layout.Condition.when
import ui.core.layout.Image

when(post.coverImage.map(_ != null)) {
  Image.image {
    classes = "post-cover"
    Image.src = post.coverImage.map(image => if (image == null) "" else image.source)
    Image.alt = post.coverAlt.map(value => Option(value).getOrElse(""))
  }
}
```

`Media.source` derives the same-origin image URL from a validated UUID. The
[complete component](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostPage.scala)
also binds dimensions through the attribute DSL to reserve the image's space.

## Collect uploads that were left behind

An editor can upload an image and close the tab before saving. Deleting every
unreferenced image immediately would race with an editor still writing.

Our cleanup only considers images created more than 24 hours ago and referenced
by no post. Draft references count too. The clock starts at upload, not when a
post later removes its cover.

Each successful upload checks a small batch belonging to that administrator.
The operator command handles a bounded batch across all owners:

```text
sbt --server "application-backend/runMain com.anjunar.blog.MediaCleanupMain"
```

Selection alone is insufficient: another request might attach an image while
cleanup waits. Both attachment and cleanup lock the same media row. After
obtaining that lock, cleanup checks references again before deleting.
[MediaLifecycle](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/MediaLifecycle.scala)
implements this protocol; a PostgreSQL concurrency test verifies the race.

## Try the complete path

Apply the additive migration and build the frontend using the
[checkpoint guide](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/docs/uploading-and-serving-media.md).
Open a draft, upload a JPEG or PNG, describe it, save and reload. Publish it and
open its public page in a signed-out browser. Retract the post and request its
image again: access should now be denied.

Finally, remove the cover and save. The post loses its reference; cleanup can
collect the image later if nothing else uses it. We now have a complete media
lifecycle to build on when the next chapter introduces images inside structured
post content.
