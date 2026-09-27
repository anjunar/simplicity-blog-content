# Registration and Account Recovery

An administrator can sign in to our blog. A reader cannot create an account
yet, and anyone who forgets a password has no way back in.

This chapter adds both paths. A visitor requests a confirmation email, opens
its link and chooses a password. An existing user requests a reset email and
chooses a replacement. Both links expire, work once, and leave no usable
token in the database.

Sign-in continues through the Soteria integration from chapter 11. Registration
and recovery change the credentials that its IdentityStore checks. They do not
introduce another authentication mechanism.

By the end, we can follow the complete journey in the browser and inspect
the outgoing messages in a local inbox.

## Start from chapter 11

The starting point includes the Soteria follow-up:

```text
git switch --detach 8c7f7a93fa164a3a3f64395e529a358605621f22
```

The completed source for this article is:

```text
git switch --detach f1599a3904737ba63ce413e2f5db14fd2f38569f
```

Use a separate local PostgreSQL database and the configuration from chapter 11.
The [setup guide](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/docs/registration-and-recovery.md) contains the
complete PowerShell and Bash commands. All file links below point to the tested
source revision.

## Confirm the address before choosing a password

We could ask for email and password together, create an inactive account, and
activate it when someone opens a link. That leaves an awkward situation:
someone can register another person's address with a password they know, and
the mailbox owner might later activate those credentials.

Our registration starts with the address alone. We store a pending confirmation
token. The person who can open the email chooses the first password. Only then
do we create an Account.

The two workflows share a token mechanism, but their effects stay distinct:

| Operation | Required proof | Result |
| --- | --- | --- |
| Request registration | Valid email-shaped input and CSRF | Generic acknowledgement; an eligible address receives a link. |
| Confirm registration | Registration token, new password and CSRF | Create a READER account. |
| Request reset | Valid email-shaped input and CSRF | Generic acknowledgement; an eligible existing account receives a link. |
| Complete reset | Reset token, new password and CSRF | Replace the password and revoke older sessions. |

Opening either link does nothing to the account. The user must submit a form.
This matters because mail software can inspect links before the recipient
opens them.

We deliberately keep the public registration command small. There is no
role field, account ID or general entity update. The server assigns READER;
the explicit administrator bootstrap remains separate.

## Store the proof, not the link

Our new entity is [AccountToken.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AccountToken.scala):

```scala
package com.anjunar.blog

import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
import jakarta.persistence.{Access, AccessType, Column, Entity, Id, Table, UniqueConstraint}
import jakarta.validation.constraints.{Email, NotBlank, Pattern}

import java.nio.charset.StandardCharsets.UTF_8
import java.security.MessageDigest
import java.time.Instant
import java.util.HexFormat

@Entity
@Access(AccessType.FIELD)
@SchemaId("bc12e601")
@Table(name = "blog_account_token", schema = "public",
  uniqueConstraints = Array(new UniqueConstraint(name = "uq_account_token_email_purpose",
    columnNames = Array("email", "purpose"))))
class AccountToken {
  @Id
  @Column(length = 64, nullable = false)
  @SchemaId("bc12e602")
  var digest: String = ""

  @Email
  @NotBlank
  @Column(length = 254, nullable = false)
  @SchemaId("bc12e603")
  var email: String = ""

  @Pattern(regexp = "REGISTER|RESET")
  @Column(length = 16, nullable = false)
  @SchemaId("bc12e604")
  var purpose: String = ""

  @Column(name = "expires_at", nullable = false)
  @SchemaId("bc12e605")
  var expiresAt: Instant = null

  @Column(name = "authentication_version", nullable = false)
  @SchemaId("bc12e606")
  var authenticationVersion: Long = -1L
}

object AccountToken {
  val register = "REGISTER"
  val reset = "RESET"
  val lifetimeSeconds = 30 * 60L

  def digest(token: String): String =
    HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(token.getBytes(UTF_8)))

  def wellFormed(token: String): Boolean =
    token != null && token.matches("[A-Za-z0-9_-]{43}")
}
```

The token itself comes from SessionIdentity.newToken(), which already generates
32 cryptographically random bytes and encodes them as URL-safe Base64. The email
contains that token. The table stores its SHA-256 digest.

This is different from password hashing. A password needs the deliberately
expensive derivation introduced in chapter 11. Our token starts with 256 bits
of random input; a digest gives us an indexed lookup without retaining the
bearer secret.

The purpose separates registration from reset. A token issued for one operation
cannot authorize the other. The unique email/purpose pair gives each operation
one current link. Requesting another replaces its predecessor. Both expire
after 30 minutes.

The authenticationVersion is relevant to resets: a later credential change
must invalidate a previously issued reset link. Registration has no account
yet and stores -1.

AccountToken is internal persistence state. It is discovered by the existing
CDI entity extension and receives stable SchemaId values. It has no public
EntitySchema, REST entity graph or frontend entity mirror. The frontend only
needs the operation's input and outcome.

## Apply one schema change

Refresh the mail dependencies and preview the schema:

```text
sbt --server "application-backend/update"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
```

The preview adds public.blog_account_token. Account and BlogPost do not change.
As in chapter 6, the existing named publication check makes the read-only
preview report INCOMPLETE with exit code 3. Inspect that finding and the
proposed table; then run migration separately:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

Upgrading the chapter 11 test database produces:

```text
Applied: revision 3, 1 SQL statements
AlreadyApplied: revision 3, 0 SQL statements
```

A fresh database starts at revision 1 with the three current tables. We do not
recreate existing accounts, reset passwords or alter published posts.

## Issue a link inside the transaction

[AccountRecovery.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AccountRecovery.scala) owns the database
work. This is its complete requestLink method:

```scala
  def requestLink(email: String, purpose: String): Unit = {
    val config = mail.configuration()
    lockEmail(email)
    val account = Account.byEmail(email)(using manager)
    val eligible = if (purpose == AccountToken.register) account.isEmpty else account.exists(!_.locked)
    if (eligible) {
      // Expired entries are housekeeping only; validity is always checked when consuming a link.
      manager.createQuery("delete from AccountToken t where t.email = :email and t.expiresAt <= :now")
        .setParameter("email", email).setParameter("now", Instant.now()).executeUpdate()
      removeTokens(email, purpose)
      manager.flush()
      val raw = SessionIdentity.newToken()
      val token = new AccountToken()
      token.digest = AccountToken.digest(raw)
      token.email = email
      token.purpose = purpose
      token.expiresAt = Instant.now().plusSeconds(AccountToken.lifetimeSeconds)
      token.authenticationVersion = account.map(_.authenticationVersion).getOrElse(-1L)
      manager.persist(token)
      // Only immutable values cross into the mail worker; no CDI request or EntityManager.
      transaction.afterCommit(() => mail.enqueue(config, email, purpose, raw))
    }
  }
```

The caller has already canonicalized and validated the address. A registration
link is eligible only when no account exists. A reset link is eligible only
for an existing, unlocked account.

We check mail configuration before this decision. An unconfigured application
returns the same unavailable response regardless of whether the address belongs
to an account.

For eligible requests, we remove expired entries for this address and replace
the previous token for this purpose. The explicit flush removes the old row
before the replacement encounters the unique constraint.

The database transaction also registers an afterCommit action. It sends
immutable message values to the mail queue after the token is committed.
A failed response serialization or failed commit leaves no token and sends
no email. The existing RequestTransaction owns that boundary.

The endpoint always acknowledges a valid, unthrottled request with HTTP 202
and the same outcome body. It does not reveal whether an account already
exists, is unknown, or is locked. SMTP runs outside the response path, so
waiting for a mail server does not become an account-existence signal.
This is not a constant-time guarantee: database paths still differ.

## Consume the token and change the account together

The same service handles the final operation:

```scala
  def complete(input: TokenPasswordRequest, purpose: String): Unit = {
    val digest = AccountToken.digest(input.token)
    val email = Option(manager.createQuery(
      "select t.email from AccountToken t where t.digest = :digest and t.purpose = :purpose", classOf[String])
      .setParameter("digest", digest).setParameter("purpose", purpose).getSingleResultOrNull)
      .getOrElse(invalid())
    lockEmail(email)
    val token = manager.find(classOf[AccountToken], digest)
    if (token == null || token.purpose != purpose || !Instant.now().isBefore(token.expiresAt)) invalid()
    val found = Account.byEmail(email)(using manager)
    if (purpose == AccountToken.register) {
      if (found.nonEmpty) invalid()
      val account = new Account()
      account.email = email
      account.role = "READER"
      account.passwordHash = hashing.withHashing(PasswordHash.create(input.password))
      manager.persist(account)
    } else {
      val account = found.getOrElse(invalid())
      manager.lock(account, LockModeType.PESSIMISTIC_WRITE)
      manager.refresh(account)
      if (account.locked || account.authenticationVersion != token.authenticationVersion) invalid()
      account.passwordHash = hashing.withHashing(PasswordHash.create(input.password))
      account.authenticationVersion = Math.addExact(account.authenticationVersion, 1L)
    }
    removeTokens(email, purpose)
  }
```

The first lookup returns only the address associated with the token and purpose.
We then lock that address and load the token again. This second lookup matters:
another request may have consumed or replaced it while we waited.

For registration, we check that the address is still unused, derive the password
hash, create a READER and consume the token in the same transaction. The response
does not sign the new user in. They go through the normal Soteria login flow.

For reset, we lock and refresh the account row. We reject a locked account or
a changed authenticationVersion. Updating the password increments that version
and consumes the token atomically.

Chapter 11 already compares the version in every session principal with the
current database value. A reset therefore revokes every older session on its
next request, including a different browser or device. The new password does
not silently establish a new session either.

If serialization or commit fails, the hash, version and token deletion all roll
back. A still-valid link remains available for a retry.

## Make single-use hold under concurrency

A check followed by deletion is not enough when two requests arrive together.
Both could pass the check before either deletes the row.

All token issuance and consumption for an address acquire the same
transaction-scoped PostgreSQL advisory lock:

```scala
  private def lockEmail(email: String): Unit =
    manager.createNativeQuery(
      "select 1 from pg_advisory_xact_lock(hashtextextended(:email, 12012))", classOf[lang.Integer])
      .setParameter("email", email).getSingleResult
```

This also serializes registration before an Account row exists. The lock is
released by commit or rollback; no process-local lock has to coordinate
different database connections. The email is a bound parameter, and the seed
namespaces this lock separately from administrator bootstrap.

A digest-to-address lookup before the lock does not authorize anything.
Authorization happens after the lock, when we read and check the token again.
The account table's unique email constraint is still the final protection
against another account-creation path racing with registration.

For reset, the account row lock additionally protects the version and password
update from concurrent account changes. These rules are covered by actual
parallel HTTP requests in the integration tests.

## Keep the REST boundary narrow

[AccountRecoveryResource.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AccountRecoveryResource.scala)
exposes these endpoints:

| POST endpoint | JSON fields | Successful response |
| --- | --- | --- |
| /service/auth/register | email | 202, outcome accepted |
| /service/auth/confirm | token, password | 200, outcome completed |
| /service/auth/forgot-password | email | 202, outcome accepted |
| /service/auth/reset-password | token, password | 200, outcome completed |

[RecoveryRequests.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/RecoveryRequests.scala) contains the bounded
readers and shared AuthJson parser. It reads at most 4 KiB, accepts only the
expected field names and checks their types and lengths. Email commands accept
one canonical address; completion commands require a correctly shaped token
and a 15–128 character password.

The login reader now reuses that parser too. Passwords remain separate commands,
never fields exposed through the Account.self graph.

[AuthenticationFilter.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AuthenticationFilter.scala) applies the
existing CSRF check to the new resource as well. The request still enters
Undertow, Elytron and Soteria after its transaction begins. All authentication
responses remain no-store.

[RecoveryLimiter.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/RecoveryLimiter.scala) keeps mail/recovery
attempts separate from the sign-in budget: five attempts per key and thirty
per client address in a minute, with bounded storage. Invalid-token attempts
also count. These limits are local to this server process; a multi-instance
deployment would need a shared policy.

## Send real mail, and state what delivery means

The [build](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/build.sbt) adds Jakarta Mail 2.1.3, Angus Mail 2.0.4 and
Angus Activation 2.0.2 from Maven Central.

[MailConfig.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/MailConfig.scala) builds account links from
BLOG_PUBLIC_ORIGIN. It never derives their host from the incoming request.
Outside localhost that origin must use HTTPS. It cannot contain credentials,
a path, a query or a fragment.

The SMTP connection requires STARTTLS by default and checks the server's
certificate hostname. Authentication credentials are configured together;
connection, read and write operations have finite timeouts. Plain SMTP is
an explicit localhost development mode.

[AccountMail.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AccountMail.scala) uses one worker and a queue
with 64 slots. The worker receives only immutable values, not the current
EntityManager, session or request-scoped beans. No password is included in a
message.

There is an intentional delivery limit here. The queue is in memory. A process
crash, full queue or SMTP failure can lose a message after the database commits.
HTTP 202 acknowledges the request; it does not promise an email has arrived.
We log a generic failure without addresses, credentials or tokens, and let the
user request a new link.

Reliable retries would require a durable queue or transactional outbox. That
also means deciding how to protect queued bearer tokens at rest. We do not
pretend that an afterCommit callback provides that guarantee.

Expired rows can be removed periodically with:

```sql
DELETE FROM public.blog_account_token WHERE expires_at <= CURRENT_TIMESTAMP;
```

The expiry check runs on every completion attempt, whether or not this
housekeeping has happened.

## Use a local inbox

The [Compose file](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/compose.yaml) includes an optional Mailpit service:

```text
docker compose --profile mail up -d mailpit
```

Keep the database environment from the previous chapter configured. Compose
validates its database password setting even when starting only Mailpit.

Mailpit captures messages locally, with SMTP on port 1025 and an inbox at
http://127.0.0.1:8025. Both ports are bound to localhost. The pinned image is
v1.31.3; a native Mailpit binary is an alternative if Docker is unavailable.

For PowerShell:

```powershell
$env:BLOG_SMTP_HOST = "127.0.0.1"
$env:BLOG_SMTP_PORT = "1025"
$env:BLOG_SMTP_MODE = "local"
$env:BLOG_MAIL_FROM = "blog@example.test"
$env:BLOG_PUBLIC_ORIGIN = "http://127.0.0.1:8080"
$env:BLOG_COOKIE_SECURE = "false"
sbt --server frontendAssets "application-backend/run"
```

Bash uses the same variable names with export; the setup guide includes the
full version.

Open /en/register and enter reader@example.test. Read the email in Mailpit,
open its confirmation link and choose a password. Then sign in at /en/account.
Repeat the journey through /en/forgot-password and confirm that the old session
and password stop working.

## Let the browser carry the token only as long as needed

A confirmation URL looks like this:

```text
http://127.0.0.1:8080/en/confirm#token=<random-token>
```

The fragment is not part of the HTTP request. The browser takes it into the
page's state and removes it from the address bar. The helper lives in
[RecoveryActions.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/frontend/src/main/scala/com/anjunar/blog/frontend/RecoveryActions.scala):

```scala
import org.scalajs.dom
import scala.scalajs.js

object AccountLink {
  // A fragment-only navigation may reuse the document. Handle it before the router.
  def listen(): () => Unit = {
    val reopen: js.Function1[dom.Event, Unit] = event =>
      if (Set("/en/confirm", "/en/reset-password").contains(dom.window.location.pathname) &&
          dom.window.location.hash.startsWith("#token=")) {
        event.stopImmediatePropagation()
        dom.window.location.reload()
      }
    dom.window.addEventListener("popstate", reopen, true)
    dom.window.addEventListener("hashchange", reopen, true)
    () => {
      dom.window.removeEventListener("popstate", reopen, true)
      dom.window.removeEventListener("hashchange", reopen, true)
    }
  }

  // Fragments are not sent in HTTP requests. Remove the secret from the visible URL too.
  def takeToken(): Option[String] = {
    val fragment = dom.window.location.hash.stripPrefix("#token=")
    val token = Option(fragment).filter(_.matches("[A-Za-z0-9_-]{43}"))
    if (dom.window.location.hash.nonEmpty)
      dom.window.history.replaceState(null, "", dom.window.location.pathname)
    token
  }
}
```

The token is submitted only in the final POST body, together with the password
and the current session's CSRF header. Account pages also receive no-store and
no-referrer response headers. The token is not stored in localStorage.

Reloading the cleaned URL loses the in-memory token. The page then tells the
user to reopen the email link or request another one. Reopening a link while
already on that route is handled through a lifecycle-managed navigation listener in BlogPage, installed
before the router. It reloads the document so the new token is read.

[RecoveryPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/frontend/src/main/scala/com/anjunar/blog/frontend/RecoveryPage.scala) renders all four screens
with one cohesive compose tree. Request pages bind only the email field;
completion pages bind only a new-password field. Every new UI message uses
the i18n macro. RecoveryActions handles busy state, clears passwords after
submission and ignores responses after the page is disposed.

The request acknowledgement is deliberately generic. Completion success is
specific: an account is ready, or a password has changed. Missing and expired
links point back to the appropriate request page. A resend button resets the
email form without forcing a document reload.

## Verify the failure paths too

The [backend integration suite](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/test/scala/com/anjunar/blog/AccountRecoverySpec.scala)
starts the real server against PostgreSQL and captures actual SMTP messages.
It verifies the digest stored in the table, token expiry and replacement,
purpose separation, CSRF, malformed commands, throttling and concurrent use.

It also forces serialization and transaction failures. Those tests prove that
failed issuance does not send email, and failed confirmation/reset does not
consume the link or partially update credentials. Existing session tests
continue to exercise the real Soteria path.

Set the local capture environment from the setup guide, including a free SMTP
port such as 27125 and BLOG_PUBLIC_ORIGIN=http://127.0.0.1:18080. The automated
suite starts its own capture server; do not point it at a running Mailpit.

```text
sbt --server "application-backend/testFull" "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
npx playwright test --project=recovery
```

The completed revision has 79 passing backend tests, nine Scala.js model tests
and twenty browser contract tests. The additional real recovery browser test
registers a reader, follows the captured email, signs in, resets the password,
checks session revocation and rejects reuse of the old link.

With the earlier sample-post and administrator fixtures, all four Playwright
projects run 24 tests. Browser screenshots cover desktop registration and a
mobile recovery error. Automated mail stays on localhost; no external inbox
is required.

The browser workflow leaves one uniquely named reader in its dedicated test
database. Backend tests remove their own rows. Treat local inboxes and failed
test traces as private because they can contain live links.

Our readers can now create an account and recover access. The next chapter
uses that identity to decide who may read, edit and publish, and to expose the
allowed actions through HATEOAS.

Further reading: [OWASP password recovery](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html),
[Angus SMTP configuration](https://eclipse-ee4j.github.io/angus-mail/docs/api/org.eclipse.angus.mail/org/eclipse/angus/mail/smtp/package-summary.html),
and [Mailpit installation](https://mailpit.axllent.org/docs/install/).
