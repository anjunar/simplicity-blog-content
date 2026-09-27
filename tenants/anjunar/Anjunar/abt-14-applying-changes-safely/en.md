# Applying Changes Safely

The editorial workspace can publish a post. Now it needs to save the post itself.

The tempting implementation is short: read JSON, assign the fields, and flush.
It also leaves several questions unanswered. Did we check access before changing
the managed entity? Did the user edit the version we still have? What happens
when the title is valid but the summary is not?

This chapter adds draft creation and partial edits through PreparedChange.
We will follow the whole write: prepare the input, inspect the original entity,
authorize the operation, apply once, validate, and return the committed result.

## Start from chapter 13

Use the completed permissions chapter:

```text
git switch --detach 37f2a3e6a7d440be4bbc99730bc430915f1bdaa4
```

The completed source for this chapter is:

```text
git switch --detach CHAPTER_14_SOURCE
```

The [companion guide](https://github.com/anjunar/anjunar-blog-example/blob/CHAPTER_14_SOURCE/docs/applying-changes-safely.md)
covers setup and the precise request contract. Keep the development database,
administrator and local cookie settings from the previous chapter. There are
no new database objects; migrating the existing database reports AlreadyApplied
with zero statements.

Build frontendAssets and start application-backend/run. Sign in at /en/account.
The editing form comes next chapter. For this chapter, the API and a runnable
browser-console example make the write path visible.

## Keep the entity as the model

The input uses the same field names as BlogPost:

```json
{
  "slug": "our-next-post",
  "title": "Our next post",
  "content": "A small draft with a complete write path.",
  "summary": "From a prepared change to a committed response."
}
```

We do not introduce another object with copies of these fields and a chain of
assignments back to the entity. BlogPost defines the data, its EntitySchema
defines mapper rules, and Bean Validation defines the field constraints.

Two operations join the editorial resource:

| Method | Path | Result |
| --- | --- | --- |
| POST | /service/editorial/posts | Create a draft; return 201, Location and the detail envelope. |
| PATCH | /service/editorial/posts/{id} | Apply supplied fields; return the refreshed detail envelope. |

Both require ADMIN, CSRF and application/json. A PATCH body includes the version
the caller loaded. For example:

```json
{
  "version": 0,
  "title": "A clearer title"
}
```

Omitted fields stay unchanged. A supplied null clears optional summary; it cannot
clear required title or content. This is our API's partial entity format, not
JSON Patch or JSON Merge Patch. PATCH provides the HTTP operation for partial
changes, and the request must succeed or fail atomically.
[RFC 5789](https://www.rfc-editor.org/rfc/rfc5789.html)

## Prepare before applying

The published json-mapper library already provides PreparedChange. Calling
JsonMapper.prepare retains the input and target entity without assigning the
submitted values.

Its two central methods have different jobs:

- getEntity returns the original target for permission and business checks.
- applyChanges runs binding, schema rules and field validation against that target.

In PreparedChanges.scala, the actual preparation call is:

```scala
    JsonMapper.prepare(json, post, TypeResolver.resolve(classOf[BlogPost]),
      manager.getEntityGraph("BlogPost.detail"), noReferences,
      [T] => (clazz: Class[T]) => RuntimeContext.bean(clazz), validation.validator)
```

This belongs inside the request-scoped PreparedChanges bean. JsonMapper comes
from com.anjunar.json.mapper; TypeResolver comes from com.anjunar.scala.universe.
The bean injects EntityManager and PostValidation. RuntimeContext resolves the
field rules through CDI, just as it did during reads in chapter 13.

The graph selects the post contract. The rules decide whether a field is writable.
Selecting status in a graph does not grant permission to assign it.

## Connect it to REST

Creation and editing start with different entities.

For POST, PreparedChangeReader reads the JSON body and prepares a new BlogPost.
Its defaults are meaningful: a new post is a draft, has no publication timestamp
and may have empty content.

For PATCH, a ParamConverter resolves the URL parameter to a prepared change
against the existing, managed post. The controller can therefore declare:

```scala
  @PATCH @Path("/{id}")
  @Consumes(Array(MediaType.APPLICATION_JSON))
  @EntityGraph("BlogPost.detail")
  def update(@PathParam("id") change: PreparedChange[BlogPost]): Data[BlogPost] = {
    if (!access.canEdit(change.getEntity())) throw new ForbiddenException()
    val post = change.applyChanges()
    validation.requireValid(post)
    requireFreeSlug(post)
    manager.flush()
    result(post)
  }
```

This method is in EditorialPostsResource.scala. PreparedChange is imported from
com.anjunar.json.mapper. PATCH, Path, Consumes, PathParam and ForbiddenException
come from jakarta.ws.rs; MediaType comes from jakarta.ws.rs.core.

The first line of the method checks the original entity. Only the next line
applies incoming values. The reader and converter do not make that authorization
decision for the controller.

The providers deliberately support PreparedChange[BlogPost] at this checkpoint.
There is no client-controlled entity class name. An optional @type value must
be BlogPost, and an optional ID on an update must match the URL. Unknown fields
are reported instead of silently swallowing spelling mistakes.

## Make the version mandatory

A database transaction does not tell us which version the user saw.

Suppose two editors load version 0. The first saves a new title and produces
version 1. The second must not quietly replace it using an old copy.

EntityVersions.scala validates the submitted version:

```scala
  def requireCurrent(post: BlogPost, json: JsonObject): Unit = {
    val version = json.value.get("version") match {
      case number: JsonNumber => number.value.toLongOption.filter(_ >= 0)
      case _ => None
    }
    if (version.isEmpty) Problem.invalidField("version", "Send the nonnegative integer version you loaded.")
    if (version.get != post.version) throw new ApiProblem(409, conflictDetail, Problem.conflict)
  }
```

JsonNumber and JsonObject are imported from the mapper's intermediate.model
package. This method belongs to the EntityVersions object; conflictDetail is its
user-facing explanation to reload before saving again.

Zero is a valid initial version. Missing, null, quoted, fractional and overflowing
versions receive a 400 field error. A valid version that differs from the stored
version receives 409. We never assign the submitted number to the entity's
@Version field.

There is also a concurrency window to close. Comparing versions and then writing
without coordinating the requests can let two requests pass the same comparison.

PreparedChanges.update loads the row with a write lock before checking:

```scala
    val post = manager.find(classOf[BlogPost], uuid, LockModeType.PESSIMISTIC_WRITE)
    if (post == null || !access.canRead(post)) throw new NotFoundException()
    EntityVersions.requireCurrent(post, json)
    prepare(json, post)
```

LockModeType is imported from jakarta.persistence, and NotFoundException from
jakarta.ws.rs. The preceding code parses the URL's UUID.

The second writer waits, then compares its submitted version with the current
row. Our concurrent test gets one 200 and one 409. Hibernate's @Version remains
part of the entity contract; we have added an explicit required version at the
HTTP boundary. Pessimistic locking holds the database lock for the transaction.
[Jakarta Persistence](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2)

A no-op does not force a new version. An actual edit returns Hibernate's next
version. Publication also advances it, so an edit based on the pre-publication
version is stale.

## Validate fields and the complete result

The mapper checks submitted field values while applying the change. That is
necessary, but it is not the whole validation story.

A creation request can omit title entirely. There is then no submitted title
value for property binding to validate. A partial edit can also leave every
individual value valid while breaking a rule that spans several fields.

PostValidation therefore validates the completed entity:

```scala
  def requireValid(post: BlogPost): Unit = {
    val violations = validator.validate(post)
    if (!violations.isEmpty) throw new ConstraintViolationException(violations)
  }
```

This method is in PostValidation.scala. The application-scoped bean owns and
closes a ValidatorFactory; ConstraintViolationException is imported from
jakarta.validation.

For our model, a draft may have empty content. A published post may not. If an
administrator clears the content of a published post, the full entity check
rejects it even though an empty string passes the content field's size constraint.

The field rules from chapter 13 still apply. Title, slug, content and summary
are writable for an administrator. Status and publishedAt remain read-only to
the mapper; callers use publish and retract for those transitions. Read-only
values sent back in an entity representation are ignored. ID and version have
the additional identity and precondition checks described above.

## Check uniqueness without flushing too early

A useful slug error says which field needs attention. The controller first
checks whether another post already uses the requested slug.

That query uses the schema's typed Criteria attributes. It also has an explicit
flush mode:

```scala
    if (manager.createQuery(query).setFlushMode(FlushModeType.COMMIT).getSingleResult.longValue() > 0)
      throw new ApiProblem(409, "This slug is already used by another post.", Problem.conflict,
        Seq(new ErrorRequest(util.List.of[Any]("slug"), "Choose an unused slug.")))
```

This is the final part of requireFreeSlug in EditorialPostsResource.
FlushModeType comes from jakarta.persistence, ErrorRequest from
com.anjunar.json.mapper, and util from java.util.

Without that setting, Hibernate could flush the pending edit before running the
query. We want to finish this check first, then flush deliberately.

The precheck cannot replace the database constraint. Two new posts can race for
the same unused slug. The unique constraint decides the winner; the error mapper
turns that failure into 409 with a slug error. It recognizes both the adopted
blog_post_slug_key name and uq_blog_post_slug from the mapping.

## Keep failure atomic

PreparedChange is deferred binding. It is not an immutable diff or an independent
transaction.

During applyChanges, a valid title may already be assigned when an invalid
summary is discovered. The managed object has changed in memory. Letting that
exception escape is essential: the request must roll back.

Our existing TransactionBoundary still controls the lifetime. The controller
validates and flushes, the writer serializes the response into a buffer, and
the transaction commits before that successful body is sent. If validation,
serialization or commit fails, the changes are rolled back.

A flush is not a commit. Explicitly flushing in the controller gives us the
updated version and surfaces database constraints, while leaving the final
commit under TransactionBoundary.

The tests intentionally fail after a valid edit has been applied. Both writer
and commit failures leave the original title and version in the database.
Another test rejects a prepared request before applying it and verifies that
the controller still sees the original title.

After a successful application, calling applyChanges again is an error. Do not
retry a failed application on the same object either; roll back that request
and start with a fresh one.

## Be strict at the HTTP boundary

The incoming body is untrusted input even for an authenticated caller.

RequestJson reads at most 1 MiB, accepts valid UTF-8, rejects duplicate object
keys and limits nesting to 32 levels. Jackson Core supplies streaming token
parsing; it was already a dependency of json-mapper and is now declared directly
because the application uses it.

The parser creates the JSON mapper's intermediate nodes. It does not bind a
second domain model or assign entity fields.

PreparedChanges also checks JSON shapes for persistent String attributes, using
the JPA metamodel. An object where title should be a string is a field error.
It must not be treated as an empty bean or silently leave the title unchanged.
Null remains distinct from a missing property.

The input helpers are specific to the contract we expose now. BlogPost does
not yet have author or media relationships. The EntityLoader passed to the
mapper rejects reference loading. When chapter 17 introduces relationships,
we must load and authorize each referenced target; unrestricted find-by-ID
would not be sufficient.

## Return errors a form can use

A bare 400 tells the browser that something failed. It does not identify the
field to fix.

The API now returns application/problem+json with a stable problem type, HTTP
status, readable detail and request path. Field errors use an errors extension:

```json
{
  "type": "/problems/validation",
  "title": "Bad Request",
  "status": 400,
  "detail": "The request contains invalid values.",
  "instance": "/service/editorial/posts",
  "errors": [
    {
      "path": ["title"],
      "message": "must not be blank"
    }
  ]
}
```

This follows the problem-details format in
[RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html).
The errors member and its path/message structure are our application extension.
Validation messages can vary by constraint and validator locale.

Expected input errors never return driver text or a stack trace. Unexpected
failures return an errorId for correlation with the server log. Authentication
challenges and rate-limit headers are preserved when converting existing HTTP
errors to problem responses.

On Scala.js, HttpFailure now retains an optional ProblemDetails object with
typed field errors. The real HTTP status remains authoritative if the body is
missing, malformed or inconsistent. Existing pages keep their status-based
messages; chapter 15 can bind the field errors to inputs.

## Return the state the client should use next

A successful create returns 201 with Location and Data[BlogPost]. The ID and
initial version come from persistence. A successful update returns the current
detail envelope, including its version, structural schema and fresh $links.

The collection now offers create, and an editable post offers update with method
PATCH. The browser's future save action can follow those links rather than
reconstructing command URLs.

There is one existing mapper behavior to account for: empty strings are omitted
from responses. A newly created draft with empty content is legitimate.
EditorialService restores that omitted draft content to an empty string in its
detail model. A published detail still requires content. This preserves the
difference between a compact public list and an editable draft detail.

## Run the exact example

The repository contains
[docs/examples/post-changes.js](https://github.com/anjunar/anjunar-blog-example/blob/CHAPTER_14_SOURCE/docs/examples/post-changes.js).

After signing in as the development administrator, paste that file into the
browser console. It obtains the session, follows the editorial entry, creates
a draft through its create link and updates it through its update link.

The essential update uses the returned version:

```javascript
  const savedResponse = await send(update, {
    version: created.data.version,
    title: "A safely updated post",
    summary: "Prepared, authorized, applied and validated."
  });
```

The file defines send, supplies the CSRF token and validates the destination.
It then repeats an edit with the old version and expects 409. The returned object
contains the generated ID, versions 0 and 1, the conflict status and a preview
address. One draft remains for you to inspect.

This is the same file executed by the real browser test. The test cleans up
its created post afterwards.

## Verify the write contract

Use the isolated test database and local SMTP capture settings from chapter 12:

```text
sbt --server "application-backend/testFull" "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
npx playwright test --project=changes
```

The changes browser project also needs the dedicated test administrator and
psql on PATH, or BLOG_PSQL pointing to it. The companion guide lists the variables.

The checkpoint passes 112 backend tests, 12 Scala.js model tests and 29 browser
contracts. The real changes workflow checks the article example against Soteria,
REST and PostgreSQL. All six browser projects together contain 35 tests.

We now have a write contract that creates a draft, applies only permitted fields,
rejects stale edits and rolls back a failed request. Next we will give it a
proper form with direct model binding, field errors and save-state handling.
