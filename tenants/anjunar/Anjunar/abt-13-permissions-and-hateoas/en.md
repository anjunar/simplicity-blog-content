# Permissions and HATEOAS

Our blog knows who is signed in. It still needs to decide what that person
can do.

This chapter gives the administrator a small editorial workspace. Open a draft,
publish it, and retract it again. A reader can keep reading public posts but
cannot enter that workspace.

We will carry one decision all the way through the application: **may this
caller publish this post now?** The backend answers it, enforces it, and includes
the corresponding action in the response. The browser uses that response to
show a Publish button.

## Start from chapter 12

The starting point is the completed registration and recovery chapter:

```text
git switch --detach f1599a3904737ba63ce413e2f5db14fd2f38569f
```

The finished source for this article is:

```text
git switch --detach 37f2a3e6a7d440be4bbc99730bc430915f1bdaa4
```

The [companion guide](https://github.com/anjunar/anjunar-blog-example/blob/37f2a3e6a7d440be4bbc99730bc430915f1bdaa4/docs/permissions-and-hateoas.md)
contains the setup and verification steps. Use the existing development database
and bootstrapped administrator. All dependencies still come from Maven Central.

There are no new tables or columns. Against the chapter 12 database, running
SchemaMain migrate reports AlreadyApplied, revision 3, and zero SQL statements.

The optional database/examples/public-posts.sql script supplies a public post
and Our private draft. Load those fixtures into your development database,
build frontendAssets, and start application-backend/run. For local HTTP, keep
BLOG_COOKIE_SECURE=false as in chapter 11.

Sign in at /en/account. The new Open editorial link leads to the list of posts.

## Decide what each layer protects

A publication request raises several different questions:

| Question | Where we answer it |
| --- | --- |
| Who is making the request? | Soteria and the container security context. |
| May this caller invoke the editorial endpoint? | EndpointPolicy and AuthorizationFilter. |
| Can this particular post be published? | PostAccess and the BlogPost domain method. |
| Which fields may the mapper read or write? | Rules in BlogPost.Schema. |
| Which actions should the browser offer now? | PostLinks on this response. |

These decisions cooperate, but they are not interchangeable. A graph selecting
content is not permission to read a draft. A hidden button does not stop a
direct HTTP request. A role alone does not make an already published post
publishable again.

## Keep Soteria as the source of identity

Chapter 11 already connected Soteria to Undertow. We use that integration rather
than introducing another principal or a second role system.

The complete application/backend/src/main/scala/com/anjunar/blog/CallerAccess.scala
class is small:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.security.enterprise.SecurityContext

import scala.compiletime.uninitialized

@RequestScoped
class CallerAccess {
  @Inject var security: SecurityContext = uninitialized

  def authenticated: Boolean = security.getCallerPrincipal != null
  def hasRole(role: String): Boolean = authenticated && security.isCallerInRole(role)
  def administrator: Boolean = hasRole("ADMIN")
}
```

The request still resolves the session against the current account in the
database. A locked account or changed authentication version invalidates the
session; a changed role applies on the next request. CallerAccess reads the
container's answer for that request.

Its request scope matters. We must not reuse one administrator's permission
result for a later anonymous request.

## Put an explicit policy on every endpoint

The public resources receive @PermitAll. The new EditorialPostsResource class
uses @RolesAllowed(Array("ADMIN")), imported from jakarta.annotation.security.

The class serves these operations:

| Method | API path | Purpose |
| --- | --- | --- |
| GET | /service/editorial/posts | Bounded list of drafts and published posts. |
| GET | /service/editorial/posts/{id} | Complete editorial preview. |
| POST | /service/editorial/posts/{id}/publish | Publish a nonempty draft. |
| POST | /service/editorial/posts/{id}/retract | Return a published post to draft. |

In EndpointPolicy.scala, policy lookup checks the method before the class:

```scala
  def of(method: Method, resource: Class[?]): EndpointPolicy =
    declared(method).orElse(declared(resource)).getOrElse(EndpointPolicy.Denied)
```

This method belongs to the EndpointPolicy companion and imports Method from
java.lang.reflect. The declared helper reads PermitAll, DenyAll and RolesAllowed.
It rejects conflicting declarations on the same element.

A method declaration overrides the class declaration, following
[Jakarta Annotations](https://jakarta.ee/specifications/annotations/3.0/annotations-spec-3.0).
Closing an endpoint without an annotation is our application's explicit default.

Adding an annotation does not, by itself, connect our embedded REST deployment
to its enforcement. AuthorizationFilter reads the policy and calls requireAccess
before the resource method runs. An anonymous request to a role-protected
endpoint receives 401; an authenticated reader receives 403.

The request filters run in this order:

1. TransactionBoundary opens the request transaction at priority 2000.
2. AuthenticationFilter resolves Soteria's caller at 2100.
3. AuthorizationFilter checks the endpoint at 2200.

AuthorizationFilter also checks CSRF for unsafe requests to the authentication,
recovery and editorial resources. If a later chapter adds another write resource,
it must join that protection. A valid role is not a substitute for a valid CSRF
token.

## Check the actual post

Endpoint access gets the caller into the resource. PostAccess.scala then checks
the entity and its current state:

```scala
  def canPublish(post: BlogPost): Boolean =
    canEdit(post) && post.status == BlogPostStatus.DRAFT &&
      post.content != null && !post.content.isBlank

  def canRetract(post: BlogPost): Boolean =
    canEdit(post) && post.status == BlogPostStatus.PUBLISHED
```

Here, canEdit requires a non-null post and an administrator. canRead allows
published posts or an administrator. We have no author relationship yet, so
there is no ownership rule to invent.

The public API continues to use published-only queries. Even an administrator
cannot fetch a draft through /service/blog/posts/{slug}; preview is a separate
editorial operation. That keeps the meaning of a public URL stable.

The publish method in EditorialPostsResource.scala is:

```scala
  @POST @Path("/{id}/publish")
  @EntityGraph("BlogPost.detail")
  def publish(@PathParam("id") id: String): Data[BlogPost] = {
    val post = load(id, true)
    access.requireTransition(access.canPublish(post))
    post.publish(Instant.now())
    result(post)
  }
```

The resource imports POST, Path and PathParam from jakarta.ws.rs, and Instant
from java.time. EntityGraph and the other application types share its package.

The load helper parses the UUID, finds the entity and checks read access.
For a command it acquires a PESSIMISTIC_WRITE lock. We evaluate the transition
after that lock, against the current row. Two administrators publishing the
same draft concurrently get one successful publication and one 409 conflict.

The BlogPost method still owns the state change: publication sets status and
publishedAt together. Retracting clears the timestamp. The existing validation
and database check remain in place.

These commands accept no editable entity fields. They do not save a submitted
copy of the post. General updates and submitted-version conflicts belong to
PreparedChange in the next chapter.

The request transaction still spans response serialization. A writer failure
or failed commit rolls the transition back. On success, the returned entity
contains the incremented version and its new publication state.

## Give the mapper field rules

A field can be readable without being writable. An administrator may edit
a title, but publication status should change through the publication command.

BlogPost.Schema therefore uses two explicit rules. These declarations remain
inside its existing Schema class:

```scala
    val title: SingularProperty[BlogPost, String] = reference(_.title, classOf[PostEditRule])
    val content: SingularProperty[BlogPost, String] = reference(_.content, classOf[PostEditRule])
    val status: SingularProperty[BlogPost, BlogPostStatus] = reference(_.status, classOf[PostReadRule])
    val publishedAt: SingularProperty[BlogPost, Instant] = reference(_.publishedAt, classOf[PostReadRule])
```

SingularProperty is imported from com.anjunar.json.mapper.schema.property;
Instant comes from java.time. Slug and summary also use PostEditRule.
ID and version retain the default read-only rule.

Both rules allow reading when PostAccess.canRead is true. PostEditRule allows
writes only when canEdit is true. PostReadRule always rejects mapper writes.

We keep reference because these are persistent singular attributes. Their rules
do not turn them into a separate, hand-written Criteria metamodel.

The cached schema holds rule classes. The mapper's existing rule factory calls
RuntimeContext.bean, which resolves their injected PostAccess through CDI.
It does not store the current user or an allowed flag in the schema.

There is no generic edit endpoint yet. A test-only resource runs the mapper
against an unpersisted post under real anonymous, reader and administrator
requests. Only the administrator can change its title. None can write version,
status or publishedAt through the mapper. This checks the rule before we rely
on it for general editing.

Schema.forGraph remains structural metadata: which fields belong to the
selected response and their types. We have not turned it into a writable-field
catalogue.

## Describe the next actions with links

An administrator looking at a nonempty draft can publish it. After publication,
the useful action is retract. We express that difference in the response.

Data, Table and SessionState now have a $links collection. The links belong to
the response envelopes; no new JPA field or database column is needed.
The existing Link type from the JSON mapper supplies rel, url, method and
its @type value.

A shortened draft response looks like this:

```json
{
  "data": {
    "id": "9f524fe5-649b-4d91-aeb6-77e6dc034cf1",
    "version": 0,
    "title": "Our private draft",
    "status": "DRAFT"
  },
  "$links": [
    {
      "rel": "self",
      "url": "/service/editorial/posts/9f524fe5-649b-4d91-aeb6-77e6dc034cf1",
      "method": "GET",
      "@type": "BlogPost"
    },
    {
      "rel": "publish",
      "url": "/service/editorial/posts/9f524fe5-649b-4d91-aeb6-77e6dc034cf1/publish",
      "method": "POST",
      "@type": "BlogPost"
    }
  ]
}
```

The real detail response also contains the selected entity fields and schema
metadata. This excerpt keeps only the parts needed to follow the action.

The relation says what the link means; the URL identifies its target.
That is the basic link model described by
[RFC 8288](https://www.rfc-editor.org/rfc/rfc8288.html).
Our $links JSON representation and action names are an application contract,
not a JSON format standardized by that RFC.

PostLinks.scala adds the commands conditionally:

```scala
    if (endpoint("publish") && access.canPublish(post))
      links.add(new Link("publish", s"$path/publish", "POST", "BlogPost"))
    if (endpoint("retract") && access.canRetract(post))
      links.add(new Link("retract", s"$path/retract", "POST", "BlogPost"))
```

This excerpt belongs inside editorialPost. Link is imported from
com.anjunar.json.mapper.schema. The endpoint helper reads the same
EndpointPolicy used by AuthorizationFilter, and access is the same PostAccess
used by the command. The code does not maintain another copy of the role rules.

A published post also receives a public GET link. The editorial collection
includes self and, when applicable, previous and next page links. An administrator's
session response includes the editorial entry link; an anonymous or reader
session does not.

Each invocation builds fresh links. Because even a public post envelope can
include an administrator-only preview link, API responses use Cache-Control:
no-store. Otherwise a response cache could serve one caller's actions to another.

## Let the browser follow the response

The Scala.js envelopes map the JSON name $links to a links member. An omitted
collection keeps its empty default. BlogPost itself continues to mirror the
entity fields.

The account page shows Open editorial when the session includes the editorial
relation. The detail page shows Publish or Retract when the current response
includes that action. It does not derive those buttons from an ADMIN string.

In EditorialService.scala, execution follows the supplied link:

```scala
  def execute(link: ApiLink): Future[BlogPostData] = {
    // Validate before fetching CSRF or sending a command.
    val path = link.path("POST")
    accounts.session().flatMap(state =>
      HttpJson.post[BlogPostData](path, js.Dynamic.literal(), state.csrfToken)).map(validate)
  }
```

Future is imported from scala.concurrent and js from scala.scalajs.
The service already has an AccountService and an ExecutionContext.

ApiLink.path checks the expected HTTP method and resolves the URL against the
current origin. It accepts only a same-origin /service/ destination without
credentials or a fragment. An external action link must never receive our
session's CSRF token.

EditorialActions replaces the current detail response after success. That
updates the entity values, version and available actions together. While a
command is pending, the buttons are disabled. If the component has been
disposed, its late response cannot redraw the page.

The UI tree stays together in EditorialPostPage.compose. Its Publish control
uses the reactive link list:

```scala
        when(actions.current.map(_.links.exists(_.rel == "publish"))) {
          button(i18n"Publish") { buttonType("button"); disabled = actions.busy; onClick(_ => actions.run("publish")) }
        }
```

The imports in that component provide Condition.when, the Button DSL,
EventDsl.onClick and the i18n macro. This is part of its existing render block,
not a separate rendering helper.

The API decides which operations it advertises. The application still defines
its page routes and how to present a publication action; HATEOAS does not
automatically generate an entire interface.

## Treat a link as a current offer

A link can become stale between loading the page and clicking it.

Another administrator may publish the post. The operator may remove this
account's administrator role. The session may expire. Someone may also call
a known URL without ever receiving a link.

Every request therefore repeats endpoint, CSRF and object-state checks.
There is no link token that bypasses those checks.

The browser handles failures deliberately:

- 401 asks the user to sign in again.
- 403 explains that the action is no longer allowed.
- 409 asks the user to reload the changed post.
- A network or server failure says the result could not be confirmed.

After an action fails, it discards the old action links and offers reload.
It does not automatically repeat a POST: the response may have been lost
after a successful commit.

## Verify the complete path

Build the application and run the existing suites with the isolated database
and SMTP capture settings from chapter 12:

```text
sbt --server "application-backend/testFull" "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
```

This checkpoint passes 89 backend tests, 9 Scala.js model tests and 28 browser
contracts. The permissions tests cover real caller roles, revoked access,
field rules, CSRF, invalid state, concurrent publication and rollback. The
browser contracts check action URLs and methods, hidden actions, conflict
recovery, plain-text content and external-link rejection.

There is also a real publication workflow:

```text
npx playwright test --project=editorial
```

It requires the dedicated test administrator variables from chapter 11 and
psql on PATH, or BLOG_PSQL pointing to its executable. It inserts its own
unique draft into the configured test database, signs in through Soteria,
publishes through the UI, reloads, retracts, checks public visibility and
deletes its fixture.

For a manual check, follow the same steps with Our private draft. Inspect the
detail response in the browser's network panel before and after publication.
The changing $links should match the available buttons, while a direct call
from an unauthorized session still fails.

We can now enforce an operation from the endpoint down to the post and explain
the available next step to the browser. The next chapter uses that foundation
to apply general entity changes with PreparedChange.
