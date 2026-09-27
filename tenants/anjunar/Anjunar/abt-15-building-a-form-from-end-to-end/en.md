# Building a Form from End to End

The API can save a post. Now an administrator should be able to do that from
the editorial workspace.

A useful form needs more than inputs and a button. It must send the version
the editor actually loaded, display the server's field errors, and preserve
text typed while a previous save is still in flight.

This chapter builds that complete path. We will create a draft, edit it,
clear an optional field, and handle a version conflict without losing the
local text.

## Start from the write contract

The starting point is chapter 14:

```text
git switch --detach bb212a133dc20f9647ead13d5347be42e5829592
```

The completed checkpoint for this chapter is:

```text
git switch --detach 9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b
sbt --server frontendAssets
sbt --server "application-backend/run"
```

Keep the existing development database and administrator. No schema migration
or dependency upgrade is needed. For local HTTP, keep BLOG_COOKIE_SECURE=false.
Sign in at /en/account and open the editorial workspace.

The [companion guide](https://github.com/anjunar/anjunar-blog-example/blob/9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b/docs/building-a-form-from-end-to-end.md)
contains the configuration and full walkthrough.

The workspace now offers **New post** when the collection provides a create
link. A post preview offers **Edit post** when the detail provides an update
link. Direct navigation checks those capabilities too. The backend remains
responsible for authorization on every request.

## Bind the entity mirror directly

We already have a frontend BlogPost with the same published fields as the
JPA entity. The form binds to that object.

The title property now also carries the client constraint metadata:

```scala
  @(NotBlank @field)()
  @(Size @field)(min = 3, max = 180)
  val title: Property[String] = Property("")
```

This excerpt belongs to the frontend BlogPost.scala. Property is imported
from ui.core.state; NotBlank and Size come from ui.forms.validators, and
field from scala.annotation.meta. These are frontend descriptor annotations,
not JPA annotations.

The slug carries the same length and lowercase-hyphen pattern as the entity.
Summary is limited to 300 characters, and content to 100000. The form library
reads these annotations from its model descriptor.

Client checks give immediate feedback. Server-side field validation still
happens inside the JSON mapper's applyChanges. Hibernate retains its existing
complete-entity validation before persistence. We add no controller validation
pass.

The two string fields that were optional in the read model need some care.
Native text controls bind Property[String]; they do not convert an Option
object into input text. Content and summary therefore use nullable String
properties. Their names and JSON value types remain the same as on the entity.

Null still matters. A public list omits content, while a draft detail may have
an empty string. The editorial detail loader restores empty draft content;
writeBody refuses a model whose content is still missing. We never turn a
partially loaded list row into a full editing form.

## Keep the form tree together

PostEditorPage contains one form(post) inside its compose method. Each control
names the property it binds to: input("title"), input("slug"),
textAreaInput("summary") and textAreaInput("content").

Here is the complete title field from inside that form:

```scala
        div {
          classes = "post-field"
          label { fieldLabel ?=> fieldLabel.setAttribute("for", "post-title"); text(i18n"Title") {} }
          val control = input("title") { fieldInput ?=>
            id = "post-title"
            fieldInput.setAttribute("aria-describedby", "post-title-errors")
          }
          control.addDisposable(control.invalid.observe(value => control.setAttribute("aria-invalid", value.toString)))
          paragraph { id = "post-title-errors"; classes = "field-error"; text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {} }
        }
```

PostEditorPage imports the layout DSL, Input.input, Form.form, the i18n macro
and scala.scalajs.js. The complete file shows those imports and the enclosing
compose/render/form blocks.

The label points to the real input. The error text has a stable ID, and
aria-invalid follows the control's validation state. A screen reader can
associate the field, its label and its error; the browser test locates it by
that label.

The explicit receiver on setAttribute is intentional. A page component also
has that method. Calling it without a receiver inside a nested block can
change the outer component instead of the intended label or input.

The field's markup, bindings and event behavior stay together in compose.
There are no rendering helpers that scatter pieces of the form across methods.

## Check the binding before sending

The form handles the native submit event, so the button and keyboard submission
use the same path. Before saving, it clears old server errors and performs two
different checks:

```scala
            val bindings = mountedForm.validateBindings()
            val validation = mountedForm.validate()
            if (bindings.nonEmpty) actions.notice.set(BindingFailed)
            else if (validation.nonEmpty) actions.notice.set(Invalid)
```

This is an excerpt from PostEditorPage's submit handler. BindingFailed and
Invalid are SaveNotice values imported into the component.

validateBindings catches a control that is not connected to the named model
property. validate checks the bound values and makes their errors visible.
A misspelled field name must not produce an input that looks editable but
never updates the model.

Only after both checks succeed does the handler call the save action.
The save button is disabled during a request. The text controls remain editable.

## Let the mapper build the payload

The Scala.js JSON mapper serializes changed properties. That gives us the
partial update format from chapter 14 without manually copying every field
into a separate request object.

BlogPost.writeBody handles the few transport rules around that mapping:

```scala
  def writeBody(): js.Dynamic = {
    require(content.get != null, "Load a detail before editing")
    val body = JsonMapper.serialize(this)
    if (id.get.isEmpty) js.special.delete(body, "id")
    // The serializer emits dirty properties. A PATCH precondition is required even when clean.
    if (id.get.nonEmpty) body.updateDynamic("version")(version.get.toDouble)
    if (!js.isUndefined(body.summary) && body.summary.asInstanceOf[String] == "") body.updateDynamic("summary")(null)
    body
  }
```

JsonMapper comes from ui.json; js is imported from scala.scalajs.

An existing post must always send its saved version, even when that property
has not changed. Version zero is valid. A new post omits its empty ID, so the
server generates one.

A cleared summary becomes JSON null. Empty content stays an empty string:
an empty draft is allowed. An omitted summary or content property remains
omitted. These cases must not collapse into one generic "empty value" rule.

Status and publishedAt are marked JsonIgnore with deserializable=true. We
read them from responses, but publication still uses the dedicated commands.

For example, changing only the title of version zero produces a partial body
with title, identity/type metadata and version. It does not resend the whole
post or overwrite its untouched summary.

## Freeze the submission before awaiting anything

Saving requires a CSRF token from the current session. That lookup is
asynchronous, and the editor may keep typing while it runs.

PostEditorActions therefore captures both the comparison values and the JSON
payload immediately:

```scala
      val submitted = post.snapshot
      // Serialize now, before session lookup; later keystrokes belong to the next save.
      val body = post.writeBody()
```

PostSnapshot is a small, immutable record of the four editable text values.
It is internal comparison state. The REST payload still comes from BlogPost
through JsonMapper.

EditorialService then checks the supplied link's relation, method and
destination before looking up the session. It sends POST for create and
PATCH for update, with the CSRF header. The existing ApiLink guard restricts
the destination to this application's origin and /service/ path.

The server's refreshed detail envelope supplies the saved values, version
and next set of links.

## Merge the acknowledgement without losing new text

Consider this sequence:

1. The editor submits the title "First revision".
2. Before the response arrives, they type "Second revision".
3. The server returns the saved "First revision" with version 1.

Replacing the form model with that response would erase "Second revision".
Keeping the entire old model would lose the new version and any other
acknowledged server values.

BlogPost.mergeSaved compares each current field with its submitted value:

```scala
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
```

editableFields and PostSnapshot.values use the same order: slug, title,
content and summary. A field that still equals its submitted value can accept
the server value. A changed field keeps the editor's newer input.

Every field's default advances to the acknowledged value. That matters because
the JSON mapper uses the default to decide what is dirty. The next request sends
the remaining changes against version 1.

The action also replaces the response links. If update is no longer offered,
it stops offering another save.

Creation uses the same merge. The returned ID is adopted even if typing continued
during the POST. The next save uses update, so it cannot accidentally create
another post. Once no newer edits remain, the router replaces the new-post URL
with the saved post's edit URL.

## Put errors where the editor can act on them

Chapter 14 already supplies typed ProblemDetails. The form converts matching
field errors to the form library's ErrorResponse:

```scala
        mountedForm.addDisposable(actions.errors.observe(values =>
          mountedForm.setErrorResponses(values.map(value => ErrorResponse(value.message, value.path)))))
```

ErrorResponse is imported from ui.forms. A path such as ["slug"] resolves to
the slug control. An entity-level error such as publicationConsistent has no
input of its own, so its message appears in the form-level alert.

Errors also belong to a particular submission. If the title was corrected
while the request was pending, an error about the old title must not mark the
new value invalid. PostEditorActions compares the submitted and current values
before attaching a field error.

A slug conflict is correctable: show the slug error and allow another save.
A stale version needs a different response. Keep all local inputs, disable
saving and offer **Discard my edits and reload**. That action explicitly replaces
the local form with the current server state.

Authentication failures and uncertain network/server results also keep the text.
They stop blind retries. In particular, an unconfirmed create might already have
committed; repeating it automatically could create another post.

## Respect the component's lifetime

A response can arrive after the editor has navigated away.

The page registers actions.dispose through addDisposable. Disposal removes
model observers and prevents later responses from changing state or navigating
back to the abandoned form.

It does not undo a request that already reached the server. This chapter has
no autosave, offline draft storage or automatic conflict merge. Navigating away
or reloading discards unsaved input; copy text you need before explicitly
discarding a conflict.

## Try the complete workflow

Open editorial, choose New post and enter a title, slug, summary and content.
Save, then change the title and clear the summary. Save again and reload.
The title should persist and the summary should remain empty.

For a conflict, open the same post's edit route in two tabs. Save in the first.
In the second, type another title and save its older version. The form should
keep your text, explain the conflict and disable saving until you choose
how to proceed.

The automated database workflow performs these operations against the real API.
The controlled browser tests also delay responses, change permissions, return
field errors, and inspect the mobile layout.

```text
sbt --server "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
npx playwright test --project=forms
```

The forms project uses the dedicated test administrator, migrated database and
psql configuration described in the companion guide. It removes its own created
post afterwards.

Verification for this chapter passes 21 Scala.js tests, 18 affected backend
integration tests and all 49 browser tests. The browser total includes 42
controlled contracts and seven real workflows.

Next, we will make the growing post list easier to navigate with search,
filtering and pagination.
