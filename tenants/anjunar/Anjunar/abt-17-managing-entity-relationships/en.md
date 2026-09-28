# Managing Entity Relationships

Our editor can change a post's text. Now it should also let us choose an author
and attach tags such as “Scala” and “Persistence”.

Those choices introduce a different kind of write. A title belongs to one post,
but a tag can appear on many posts. Changing a post's selection must not
accidentally rename or delete that shared tag.

We will follow that distinction through the entity mapping, the API and the
form. This chapter builds on the editing workflow from chapters 14–15 and the
search infrastructure from chapter 16. The
[completed checkpoint](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/docs/entity-relationships.md)
contains the full implementation and setup instructions.

## Model who owns what

A post references one optional `Account` and a set of `BlogTag` entities. The
account already exists; we add a public `displayName`. A tag gets its own ID,
version, unique slug and name.

The post stores its author in `author_id`. Tags use the `blog_post_tag` join
table because many posts can share the same tag. These are the additions inside
[BlogPost.scala](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala):

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.{FetchType, ForeignKey, JoinColumn, JoinTable, ManyToMany, ManyToOne}
import jakarta.validation.constraints.{NotNull, Size}
import java.util

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "author_id", foreignKey = new ForeignKey(name = "fk_blog_post_author"))
@SchemaId("6a1eab40") @JsonbProperty
var author: Account = null

@ManyToMany(fetch = FetchType.LAZY)
@JoinTable(name = "blog_post_tag", schema = "public",
  joinColumns = Array(new JoinColumn(name = "post_id")),
  inverseJoinColumns = Array(new JoinColumn(name = "tag_id")),
  foreignKey = new ForeignKey(name = "fk_blog_post_tag_post"),
  inverseForeignKey = new ForeignKey(name = "fk_blog_post_tag_tag"))
@NotNull @Size(max = 20)
@SchemaId("42b9d1ef") @JsonbProperty
var tags: util.Set[BlogTag] = new util.LinkedHashSet[BlogTag]()
```

The important choice is the absence of cascading deletion. Removing
“Persistence” from a post removes a join-table row. The tag remains available
to other posts.

The author is nullable so existing posts survive the migration without invented
attribution. New posts default to the signed-in administrator, and editors can
clear that choice. In this tutorial, only unlocked administrators are available
as authors.

The existing CDI extension discovers
[BlogTag](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/BlogTag.scala).
Hibernate DDL Manager adds the new columns and tables using stable SchemaIds.
We do not register the entity manually or rebuild the database.

## Describe the public relationship

The mapping tells Hibernate what to store. The entity schema and graph tell our
API how to handle it.

Inside `BlogPost.Schema`, we add the same two properties:

```scala
import com.anjunar.json.mapper.schema.property.{SetProperty, SingularProperty}
import java.util

val author: SingularProperty[BlogPost, Account] =
  reference(_.author, classOf[PostEditRule])

val tags: SetProperty[BlogPost, util.Set[BlogTag]] =
  set(_.tags, classOf[PostEditRule])
```

These properties preserve the relationship types and apply the existing
administrator write rule. The mapper uses this schema alongside the detail
graph.

An account contains much more than a byline, so the graph must deliberately
limit the nested author:

| Detail field | Published properties |
| --- | --- |
| author | id, version, displayName |
| tags | id, version, slug, name |

Email, role and authentication data do not belong in a public post response.
[Account.author](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/Account.scala)
and the post's author subgraph select only the public fields.

The list projection from chapter 16 stays unchanged. We load these relationships
on detail; adding a collection join to a paginated list would introduce a
separate query problem.

## Accept a selection as an ID

Suppose the editor chooses an existing author and tag. The request sends their
IDs, together with the post version:

```json
{
  "version": 4,
  "author": { "id": "df99c76a-c98d-41f4-b391-3d2631cfeea8" },
  "tags": [{ "id": "b891ec6d-11dc-43e4-94ef-bc65406b5262" }]
}
```

The IDs above are illustrative; real choices come from the author and tag
catalogs. Sending a tag's `name` alongside its ID is rejected. A post update
does not grant permission to edit the referenced entity.

Omission and clearing also need distinct meanings:

| Update input | Result |
| --- | --- |
| author or tags omitted | Keep the existing value |
| author: null | Clear the author |
| tags: [] | Remove all tag associations |
| tags: null | Reject; a tag collection must be an array |

[ReferenceAccess](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/ReferenceAccess.scala)
checks that each reference contains exactly an ID, validates its UUID and rejects
duplicate tags. The mapper then calls its loader to resolve the actual entity.

This is the loader method inside that request-scoped class; `manager` and
`caller` are injected through CDI:

```scala
import jakarta.ws.rs.ForbiddenException
import java.util.UUID

override def load(id: UUID, clazz: Class[?]): Any = {
  if (!caller.administrator) throw new ForbiddenException()
  if (clazz == classOf[Account]) {
    val account = manager.find(classOf[Account], id)
    if (account == null || account.locked || account.role != "ADMIN")
      Problem.invalidField("author", "Choose an available author.")
    account
  } else if (clazz == classOf[BlogTag]) {
    val tag = manager.find(classOf[BlogTag], id)
    if (tag == null) Problem.invalidField("tags", "Choose an available tag.")
    tag
  } else {
    throw new ApiProblem(400, "This entity type cannot be referenced.")
  }
}
```

Checking availability here matters. An account might have been locked after the
editor loaded the choices. A previously displayed option is not permission to
assign it now.

The loader returns a managed entity. It never constructs a new account or tag
from a submitted object.

## Keep the existing write boundary

The rest follows the `PreparedChange` workflow we already built:

1. Load the post and check its submitted version.
2. Prepare the input with the schema, graph, reference loader and Validator.
3. Check access, then call `applyChanges()` once.
4. Flush and serialize within the request transaction.

The JSON Mapper validates the submitted collection, including its 20-tag limit.
A failed reference or validation error rolls back the request, including other
fields changed in the same update.

[PreparedChanges](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/PreparedChanges.scala)
now supports BlogPost, BlogTag and Account. That is an explicit write allowlist,
separate from CDI's discovery of persistent entities. Discovering an entity
does not automatically make it editable through REST.

Shared metadata gets its own operations. Administrators can change an author's
public name or create and rename tags. Each update checks that entity's own
version. The account endpoint accepts only `displayName` plus identity/version
metadata; it cannot change roles or credentials.

Their catalogs reuse `HibernateSearch`, typed schema attributes and CDI
providers. They return bounded pages with next links. The same endpoint policies
and CSRF protection used for post editing apply here.

## Bind the choices to the post model

The Scala.js model mirrors the published structure. In frontend
[BlogPost.scala](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPost.scala),
the new fields are:

```scala
import ui.core.state.{ListProperty, Property}
import ui.forms.validators.Size
import scala.annotation.meta.field

val author: Property[Account] = Property(null)

@(Size @field)(max = 20)
val tags: ListProperty[BlogTag] = ListProperty()
```

The frontend keeps complete referenced models so it can display names. The
write payload must still contain only IDs. Inside `writeBody()`, after the JSON
mapper has serialized the dirty fields into `body`, we narrow the reference
objects:

```scala
import scala.scalajs.js

def onlyId(reference: js.Dynamic): Unit =
  if (reference != null && !js.isUndefined(reference))
    js.Object.keys(reference.asInstanceOf[js.Object]).filter(_ != "id")
      .foreach(key => js.special.delete(reference, key))

onlyId(body.author)
if (!js.isUndefined(body.tags))
  body.tags.asInstanceOf[js.Array[js.Dynamic]].foreach(onlyId)
```

This preserves the mapper's partial-write behavior: unchanged fields remain
omitted, while explicit null and an empty array remain clear operations.

The author ComboBox binds directly to `Property[Account]`. Tags need a small
adapter: the pinned ComboBox API exposes a singular form value even in
multi-select mode.
[TagSelection](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/frontend/src/main/scala/com/anjunar/blog/frontend/TagSelection.scala)
binds its selection list to the parent form's `tags` field. This keeps collection
validation and server errors attached to the correct control.

Both controls stay inside the existing
[post form](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostEditorPage.scala).
The separate metadata page lets administrators maintain the shared names before
selecting them.

The save behavior from chapter 15 also extends to relationships. If a request
submits author A and the user chooses B while waiting, A's response must not
overwrite B. We compare author IDs and sets of tag IDs, preserve newer choices,
and still advance the saved version.

## Try the complete workflow

The checkpoint uses JSON Mapper 1.1.6 from Maven Central, which contains the
reference-resolution correction needed here. With the existing development
database configured and the server stopped:

```text
git switch --detach a16b0c53d65b6ef5e9351e42f00637948dd884c8
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server frontendAssets "application-backend/run"
```

The upgrade adds public names, author references, tags and the join table.
Existing posts keep their data and an unassigned author. The
[migration guide](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/docs/entity-relationships.md#start-and-migrate)
explains the existing CHECK-constraint preview behavior.

Sign in, open **Manage authors and tags**, set a public name and create a tag.
Then create a post, select that tag, save and publish it. The public detail
shows the name and tag. Clear the tag in the editor and save again: it disappears
from the post but remains in the catalog.

The [runnable API example](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/docs/examples/entity-relationships.js)
checks the less visible cases: a nested tag rename rejects the entire update,
omitting references preserves them, and clearing them leaves the shared tag
intact. It runs unchanged in the real HTTP/PostgreSQL browser test.

We now have relationships with explicit ownership, a narrow public representation
and deliberate write semantics. Next, we apply those ideas to media uploads,
where ownership and cleanup need different rules.
