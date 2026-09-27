The blog already serves published posts. Now it needs to recognize the person using it.

By the end of this chapter, an operator can create the first administrator, sign in at `/en/account`, reload the page without losing the session, and sign out. Readers can still browse the public blog anonymously.

Start from the [chapter 10 source](https://github.com/anjunar/anjunar-blog-example/tree/db992619d48e8938308216641856849e8be9bd97). The [complete chapter 11 checkpoint](https://github.com/anjunar/anjunar-blog-example/tree/8c7f7a93fa164a3a3f64395e529a358605621f22) contains the runnable implementation and tests. File paths below are relative to the repository root.

Public registration, email confirmation, and password recovery belong to chapter 12. This chapter establishes the identity and session that those workflows will use.

## Wire Soteria into the embedded server

We use [Jakarta Security](https://jakarta.ee/specifications/security/4.0/jakarta-security-spec-4.0) for authentication, with Soteria as its implementation. Add these dependencies to the backend's `libraryDependencies` in `build.sbt`:

```scala
"jakarta.security.enterprise" % "jakarta.security.enterprise-api" % "4.0.0",
"jakarta.authentication" % "jakarta.authentication-api" % "3.1.0",
"jakarta.authorization" % "jakarta.authorization-api" % "3.0.0",
"jakarta.enterprise" % "jakarta.enterprise.cdi-el-api" % "4.1.0",
"jakarta.json" % "jakarta.json-api" % "2.1.3",
"org.glassfish.soteria" % "soteria" % "4.0.2",
"org.glassfish.soteria" % "soteria.spi.bean.decorator.weld" % "4.0.2",
"com.nimbusds" % "nimbus-jose-jwt" % "10.10",
"org.wildfly.security.elytron-web" % "undertow-server-servlet" % "4.2.1.Final",
"org.wildfly.security" % "wildfly-elytron-auth-server-http" % "2.8.4.Final",
"org.wildfly.security" % "wildfly-elytron-http-util" % "2.8.4.Final",
"org.wildfly.security.jakarta" % "jakarta-authentication" % "4.0.0.Final",
```

Then run:

```text
sbt --server "application-backend/update"
```

An embedded Servlet container needs the integration normally supplied by an application server. Elytron connects Undertow to Jakarta Authentication; Soteria's bridge connects that layer to our CDI authentication mechanism.

[SoteriaIntegration.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SoteriaIntegration.scala) configures the Elytron security domain and authentication factory. `ApplicationMain` calls it before starting the deployment. [SoteriaInitializer](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SoteriaInitializer.scala) runs Soteria's servlet initializer and checks that its authentication module was registered. A missing provider stops startup.

The official Soteria Weld adapter gives the framework's generated identity-store and mechanism-handler beans distinct identities in Weld. The CDI EL, JSON, and Nimbus dependencies supply types used by Soteria's CDI bootstrap; adding them does not enable an OpenID Connect login in this chapter.

Two small files complete the embedded setup. [CdiNamingFactory](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/CdiNamingFactory.scala) exposes Weld's current BeanManager under `java:comp/BeanManager`, where Soteria looks for it. [SoteriaCallerDetails](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SoteriaCallerDetails.scala) lets Jakarta Security read the principal and roles already established in Undertow; it is registered through `META-INF/services`.

The setup guide documents this wiring in one place. We continue to run a single embedded application.

## Define the account and its public contract

Create [Account.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/Account.scala) in the backend. It is a JPA entity mapped to `public.blog_account`, with these fields:

| Field | Purpose | Returned to the signed-in user? |
| --- | --- | --- |
| `id` | Generated UUID | Yes |
| `version` | Optimistic locking | Yes |
| `email` | Unique sign-in address | Yes |
| `role` | `ADMIN` or `READER` | Yes |
| `passwordHash` | Salted password verifier | No |
| `authenticationVersion` | Revoke existing sessions | No |
| `locked` | Disable sign-in and existing sessions | No |

The email has `@Email`, `@NotBlank`, and `@Size(max = 254)` validation, plus a named database uniqueness constraint. Our application treats email addresses as case-insensitive: it trims surrounding whitespace and lowercases with `Locale.ROOT` before creation or lookup.

As in chapter 5, annotations on fields declared in the class body need no `@field` target. Each table and column receives a stable `@SchemaId`. CDI discovers the entity through the existing extension; there is no class list to update in `Persistence`.

The account remains the REST model. Its `Account.self` graph selects the four public fields. The companion's schema declares those same fields:


```scala
import com.anjunar.json.mapper.schema.EntitySchema
import com.anjunar.json.mapper.schema.property.SingularProperty

import java.util.UUID

// Inside object Account, which extends SchemaProvider[Account.Schema].
  class Schema extends EntitySchema[Account](RuntimeContext.entityManager()) {
    val id: SingularProperty[Account, UUID] = reference(_.id)
    val version: SingularProperty[Account, Long] = reference(_.version)
    val email: SingularProperty[Account, String] = reference(_.email)
    val role: SingularProperty[Account, String] = reference(_.role)
  }
```

The persistent fields use `reference`, so email lookup can use `account.get(schema.email)` in a typed Criteria query. Default rules still deny incoming entity writes.

The three internal fields carry `@JsonbTransient` and are absent from the mapper schema, graph, and browser model. A password hash is sensitive server data even though it is not the original password.

`version` and `authenticationVersion` are separate. JPA advances the first when an entity changes. We will deliberately advance the second when a security operation needs to revoke all existing sessions.

## Add the table without replacing the blog

Keep the database settings from the previous chapter. Stop the application and run:


```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

A chapter 10 database advances to revision 2 and gains `blog_account`. Existing post IDs, content, and the publication constraint remain intact. A fresh database starts at revision 1 with both tables; repeating `migrate` reports `AlreadyApplied`.

As explained in chapter 6, a preview involving an existing named CHECK constraint can report `INCOMPLETE` and exit with code 3. The migration executor verifies that constraint under its lock. Do not drop the post table to make a preview pass.

## Store a password verifier

[PasswordHash.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/PasswordHash.scala) uses PBKDF2-HMAC-SHA256 from the JDK. Each password gets a fresh 16-byte salt and a 256-bit derived key, with 600,000 iterations.

This keeps the example within the JDK's cryptographic APIs. PBKDF2 is CPU-hard; it does not provide Argon2id's memory-hard design. OWASP generally prefers Argon2id and lists 600,000 iterations for PBKDF2-HMAC-SHA256. See its [password storage guidance](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html).

Here is the complete implementation:


```scala
package com.anjunar.blog

import java.security.{MessageDigest, SecureRandom}
import java.util.{Arrays, Base64}
import javax.crypto.SecretKeyFactory
import javax.crypto.spec.PBEKeySpec

object PasswordHash {
  private val iterations = 600000
  private val random = new SecureRandom()
  private val encoder = Base64.getUrlEncoder.withoutPadding()
  private val decoder = Base64.getUrlDecoder

  def acceptable(password: String): Boolean =
    password != null && password.length >= 15 && password.length <= 128

  def create(password: String): String = {
    require(acceptable(password), "Use a password with 15 to 128 characters")
    val salt = new Array[Byte](16)
    random.nextBytes(salt)
    s"pbkdf2-sha256$$$iterations$$${encoder.encodeToString(salt)}$$${encoder.encodeToString(derive(password, salt))}"
  }

  def verify(password: String, stored: String): Boolean = {
    if (password == null || password.length > 128 || stored == null || stored.length > 200) return false
    val parts = stored.split("\\$", -1)
    if (parts.length != 4 || parts(0) != "pbkdf2-sha256" || parts(1) != iterations.toString) return false
    try {
      val salt = decoder.decode(parts(2))
      val expected = decoder.decode(parts(3))
      salt.length == 16 && expected.length == 32 &&
        MessageDigest.isEqual(expected, derive(password, salt))
    } catch {
      case _: IllegalArgumentException => false
    }
  }

  // Unknown accounts still perform one password derivation.
  lazy val decoy: String = create("an-unusable-random-account-" + encoder.encodeToString(random.generateSeed(24)))

  private def derive(password: String, salt: Array[Byte]): Array[Byte] = {
    val chars = password.toCharArray
    val spec = new PBEKeySpec(chars, salt, iterations, 256)
    try SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256").generateSecret(spec).getEncoded
    finally {
      spec.clearPassword()
      Arrays.fill(chars, '\u0000')
    }
  }
}
```

The stored string contains the algorithm, work factor, salt, and derived key. Verification accepts only the format this chapter creates; it does not trust a stored value to choose an arbitrarily expensive work factor. Supporting older formats and upgrading them would require an explicit migration policy.

New passwords must contain 15–128 characters as measured by `String.length`. We preserve spaces and do not impose rules such as “one symbol and one uppercase letter.”

Unknown accounts still perform password derivation against a decoy. Incorrect passwords, unknown addresses, and locked accounts return the same 401 response. This reduces obvious account-discovery signals; it does not make the entire request execute in constant time.

## Create the first administrator explicitly

The application does not promote the first visitor. An operator runs [BootstrapAdminMain](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/BootstrapAdminMain.scala) once.

With the database variables already set, use PowerShell:


```powershell
$env:BLOG_ADMIN_EMAIL = "admin@example.com"
$adminSecret = Read-Host "Administrator password" -AsSecureString
try {
  $env:BLOG_ADMIN_PASSWORD = [Net.NetworkCredential]::new("", $adminSecret).Password
  sbt --server "application-backend/runMain com.anjunar.blog.BootstrapAdminMain"
} finally {
  Remove-Item Env:BLOG_ADMIN_PASSWORD -ErrorAction SilentlyContinue
  Remove-Item Env:BLOG_ADMIN_EMAIL -ErrorAction SilentlyContinue
  $adminSecret.Dispose()
}
```

Or Bash:


```bash
export BLOG_ADMIN_EMAIL=admin@example.com
read -r -s -p "Administrator password: " BLOG_ADMIN_PASSWORD
echo
export BLOG_ADMIN_PASSWORD
sbt --server "application-backend/runMain com.anjunar.blog.BootstrapAdminMain"
unset BLOG_ADMIN_PASSWORD BLOG_ADMIN_EMAIL
```

The command opens a CDI request context and a write transaction. It takes a PostgreSQL transaction advisory lock before checking whether an administrator exists. That lock prevents two cooperating bootstrap processes from both passing the check.

If any administrator exists, bootstrap fails. It also refuses an email already assigned to another account. It never resets credentials or elevates an existing user. On success, it hashes the password, persists the account, commits, and prints the new UUID.

The password travels through the short-lived process environment, not a command argument or committed properties file. Remove that environment variable afterward. Keep this command outside the HTTP API.

## Validate credentials through an IdentityStore

Create [PasswordIdentityStore.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/PasswordIdentityStore.scala):

```scala
package com.anjunar.blog

import jakarta.enterprise.context.ApplicationScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.security.enterprise.credential.{Credential, UsernamePasswordCredential}
import jakarta.security.enterprise.identitystore.{CredentialValidationResult, IdentityStore}

import java.time.Instant
import java.util.Set
import scala.compiletime.uninitialized

@ApplicationScoped
class PasswordIdentityStore extends IdentityStore {
  @Inject var manager: EntityManager = uninitialized
  @Inject var limiter: LoginLimiter = uninitialized

  override def validate(credential: Credential): CredentialValidationResult = credential match {
    case password: UsernamePasswordCredential =>
      val found = Account.byEmail(Account.canonicalEmail(password.getCaller))(using manager)
      val valid = limiter.withHashing {
        PasswordHash.verify(password.getPasswordAsString, found.map(_.passwordHash).getOrElse(PasswordHash.decoy))
      }
      found.filter(account => valid && !account.locked).map { account =>
        new CredentialValidationResult(
          SessionPrincipal(account.id, account.authenticationVersion, Instant.now()), Set.of(account.role))
      }.getOrElse(CredentialValidationResult.INVALID_RESULT)
    case _ => CredentialValidationResult.NOT_VALIDATED_RESULT
  }
}
```

Its job is credentials in, validated principal and groups out. It does not read HTTP headers, create cookies, or change a session. Unsupported credential types return `NOT_VALIDATED_RESULT`; rejected passwords return `INVALID_RESULT`.

The injected `IdentityStoreHandler` comes from Soteria. It finds our CDI store and handles its validation result. Our [SoteriaAuthenticationMechanism](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SoteriaAuthenticationMechanism.scala) connects that result to HTTP:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.ApplicationScoped
import jakarta.inject.Inject
import jakarta.security.enterprise.AuthenticationStatus
import jakarta.security.enterprise.authentication.mechanism.http.{HttpAuthenticationMechanism, HttpMessageContext}
import jakarta.security.enterprise.credential.UsernamePasswordCredential
import jakarta.security.enterprise.identitystore.{CredentialValidationResult, IdentityStoreHandler}
import jakarta.servlet.http.{HttpServletRequest, HttpServletResponse}

import java.lang
import java.util.Set
import scala.compiletime.uninitialized

@ApplicationScoped
class SoteriaAuthenticationMechanism extends HttpAuthenticationMechanism {
  @Inject var stores: IdentityStoreHandler = uninitialized
  @Inject var identity: SessionIdentity = uninitialized

  override def validateRequest(request: HttpServletRequest, response: HttpServletResponse,
      context: HttpMessageContext): AuthenticationStatus = {
    // Undertow can call before JAX-RS/CDI request work. Database authentication
    // starts explicitly after TransactionBoundary has opened the persistence context.
    if (request.getAttribute(SoteriaIntegration.requestKey) != lang.Boolean.TRUE) return context.doNothing()
    context.getAuthParameters.getCredential match {
      case credential: UsernamePasswordCredential =>
        val result = stores.validate(credential)
        if (result.getStatus != CredentialValidationResult.Status.VALID) AuthenticationStatus.SEND_FAILURE
        else {
          identity.accept(request, result.getCallerPrincipal.asInstanceOf[SessionPrincipal])
          context.notifyContainerAboutLogin(result)
        }
      case null =>
        identity.resolve(request) match {
          case Some(principal) => context.notifyContainerAboutLogin(principal, Set.of(identity.requireAccount().role))
          case None => context.doNothing()
        }
      case _ => AuthenticationStatus.SEND_FAILURE
    }
  }

  override def cleanSubject(request: HttpServletRequest, response: HttpServletResponse,
      context: HttpMessageContext): Unit = {
    identity.clear(request)
    context.cleanClientSubject()
  }
}
```

`SessionPrincipal` extends Jakarta Security's `CallerPrincipal`. On successful verification, `notifyContainerAboutLogin` hands it and its groups to the container. For an existing session, the mechanism checks its current database account and supplies the current role.

Undertow can start authentication before the JAX-RS request transaction exists. The request attribute in this mechanism is an internal lifecycle marker: `AuthenticationFilter` sets it after `TransactionBoundary` begins, then asks Undertow to authenticate again. It is not a client header or a permission flag. This keeps database work inside our existing persistence context.

`setIntegratedJaspi(false)` in the Elytron configuration means Soteria's verified identity does not undergo a second account lookup in an Elytron realm. The password is still checked by our IdentityStore.

## Let the server own the session

After sign-in, the browser sends an opaque `BLOGSESSION` cookie. The server stores a `SessionPrincipal` containing the account UUID, its authentication version, and the sign-in time.

[SessionIdentity.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SessionIdentity.scala) is called by the Soteria mechanism to reload the account on authenticated application REST requests; liveness is excluded. The identity is valid only while the account exists, is unlocked, has the recorded authentication version, and is within the eight-hour absolute lifetime. Role checks use the current database role.

A deleted or locked account therefore loses its session on its next such request. Incrementing `authenticationVersion` invalidates every session issued with the old value. There is no account-management endpoint yet; the tests exercise these changes directly in their own fixtures.

[ApplicationMain.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/ApplicationMain.scala) configures Servlet sessions:

- Cookie-only tracking, so a session ID cannot be supplied through the URL.
- `HttpOnly`, `SameSite=Lax`, `Path=/`, and no `Domain`.
- `Secure` by default, with an explicit local HTTP override.
- A 15-minute idle timeout and a maximum of 2,048 active sessions.

Sessions live in this process. Restarting it signs everyone out. Successful sign-in changes the session ID; signing out invalidates the old session. These choices follow the [OWASP session-management guidance](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html).

The small [SessionServer](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SessionServer.scala) subclass also makes shutdown explicit. With our RESTEasy version, session listeners must finish while Weld is still available, before the remaining server shutdown closes CDI.

## Protect sign-in and sign-out with CSRF

Cookies are sent automatically. That makes a cookie insufficient evidence that the user intended a state-changing request.

Our four endpoints are defined in [AuthenticationResource.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/AuthenticationResource.scala):

| Request | Result |
| --- | --- |
| `GET /service/auth/session` | A CSRF token and the optional safe account |
| `GET /service/auth/me` | `Data[Account]`, or 401 when anonymous |
| `POST /service/auth/login` | Validate credentials and establish the identity |
| `POST /service/auth/logout` | Invalidate the old session and return anonymous state |

`GET session` creates an anonymous server session if necessary. It returns a random synchronizer token belonging to that session. Login and logout must send that token in `X-CSRF-Token`; a missing token or a token from another session gets 403. Requests marked `Sec-Fetch-Site: cross-site` are also rejected.

Sign-in needs CSRF protection too: a forged login can put the victim into an account controlled by someone else. `SameSite` supplements the synchronizer-token check. See [OWASP's CSRF guidance](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html).

[AuthenticationFilter.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/AuthenticationFilter.scala) resolves identity after the transaction filter has begun the request. It enters Undertow's authentication path, applies CSRF checks to unsafe methods on `AuthenticationResource`, and marks auth responses `Cache-Control: no-store`. RESTEasy reads the principal and roles from the authenticated Servlet request. New credential resources must join that protection when we add them.

## Commit before persisting sign-in

A correct password is necessary, but the request still has to finish successfully. We should not leave a browser signed in if response serialization or transaction commit fails.

This is the login method inside `AuthenticationResource`. The class injects Jakarta Security's `SecurityContext` along with the entity/session services. It receives the Servlet request and response through `@Context`:


```scala
import jakarta.security.enterprise.AuthenticationStatus
import jakarta.security.enterprise.authentication.mechanism.http.AuthenticationParameters
import jakarta.security.enterprise.credential.UsernamePasswordCredential
import jakarta.ws.rs.{Consumes, NotAuthorizedException, POST, Path, WebApplicationException}
import jakarta.ws.rs.core.{MediaType, Response}

  @POST
  @Path("/login")
  @Consumes(Array(MediaType.APPLICATION_JSON))
  @EntityGraph("Account.self")
  def login(input: LoginRequest): SessionState = {
    if (identity.account.nonEmpty) throw new WebApplicationException(Response.status(409).build())
    limiter.check(input.email, request.getRemoteAddr)
    val credential = new UsernamePasswordCredential(input.email, input.password)
    val status = try security.authenticate(request, response,
      AuthenticationParameters.withParams().credential(credential))
    finally credential.clear()
    if (status != AuthenticationStatus.SUCCESS) throw new NotAuthorizedException("Session")
    val account = identity.requireAccount()
    val token = SessionIdentity.newToken()
    val principal = identity.principal
    transaction.afterCommit(() => identity.establish(principal, token))
    new SessionState(token, account)
  }
```

The [LoginRequest reader](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/LoginRequest.scala) accepts only two string fields, `email` and `password`, with a total body limit of 4 KiB. Extra fields such as `role` are rejected. Credentials are a specific command, not a general Account update.

[LoginLimiter](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/LoginLimiter.scala) permits five attempts per canonical email and thirty per direct client address in a 60-second window. Successful attempts count too. It bounds its key map and allows four concurrent password verifications; excess work receives 429 with `Retry-After: 60`. Its state is local to this process and resets on restart. Forwarded-address headers are not trusted; a reverse proxy would currently share one address bucket across its clients.

The [RequestTransaction.afterCommit](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/RequestTransaction.scala) callback runs only after successful serialization and commit. Soteria has authenticated the current request by this point. Only then does `identity.establish` persist the principal for future requests and rotate the session ID and CSRF token. Logout uses the same callback mechanism: `request.logout()` invokes Soteria's cleanup through Elytron, followed by invalidation of the session.

We deliberately do not request automatic session registration from Soteria. That would persist authentication before our response and database transaction have completed.

The existing writer already buffers JSON while the EntityManager is open. We preserve that boundary and perform the session mutation before sending the buffered response. A rejected request, writer failure, or failed commit discards the callback.

This cannot guarantee delivery to the browser. The network may fail after the server has completed the operation; the account page reads the current session again when reopened.

## Mirror the safe account in Scala.js

Add the forms library beside core, JSON, and router in the frontend dependencies:


```scala
"com.anjunar" %% "scalajs-ui-forms" % "1.0.9",
```

It comes from Maven Central. Refresh resolution with:


```text
sbt --server "application-frontend/update"
```

Create the frontend [Account.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/Account.scala):


```scala
package com.anjunar.blog.frontend

import ui.core.state.Property
import ui.json.JsonId

final class Account {
  @JsonId val id: Property[String] = Property("")
  val version: Property[Long] = Property(-1L)
  val email: Property[String] = Property("")
  val role: Property[String] = Property("")
}

final class SessionState(var csrfToken: String = "", var account: Option[Account] = None)

final class LoginCredentials {
  val email: Property[String] = Property("")
  val password: Property[String] = Property("")
}

final class LoginCommand(var email: String = "", var password: String = "")
```

The Account model mirrors the four published entity fields, including version zero. `LoginCredentials` contains the properties bound to the form. `LoginCommand` takes a snapshot for transport. Neither credential model is a second version of the persistent Account entity.

[AccountService.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/AccountService.scala) obtains the current CSRF token before each mutation:


```scala
package com.anjunar.blog.frontend

import org.scalajs.dom
import ui.json.JsonMapper

import scala.concurrent.{ExecutionContext, Future}
import scala.scalajs.js

final class AccountService(using ExecutionContext) {
  def session(signal: Option[dom.AbortSignal] = None): Future[SessionState] =
    HttpJson.get[SessionState]("/service/auth/session", signal).map { state =>
      require(state.csrfToken.nonEmpty, "Missing session token")
      state
    }

  def login(email: String, password: String): Future[SessionState] = {
    val command = JsonMapper.serialize(new LoginCommand(email, password))
    session().flatMap(current =>
      HttpJson.post[SessionState]("/service/auth/login", command, current.csrfToken))
  }

  def logout(): Future[SessionState] =
    session().flatMap(current =>
      HttpJson.post[SessionState]("/service/auth/logout", js.Dynamic.literal(), current.csrfToken))
}
```

The shared [HttpJson](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/HttpJson.scala) now supports JSON POST requests, same-origin credentials, and the CSRF header. The session cookie never goes into localStorage or sessionStorage.

## Bind the form and handle its lifecycle

[AccountPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/AccountPage.scala) keeps its entire UI tree in `compose`. The page shows either the sign-in form or the current account and a sign-out button. Every new interface message uses the `i18n` macro.

This form block belongs inside that page's `render(this, cursor)` tree:


```scala
import ui.core.dsl.AttributeDsl.*
import ui.core.dsl.ClassDsl.classes
import ui.core.dsl.EventDsl.on
import ui.core.i18n.i18n
import ui.core.layout.Button.{button, buttonType, disabled, disabled_=}
import ui.core.layout.Label.label
import ui.core.layout.TextComponent.text
import ui.forms.Form.{form, editable, editable_=}
import ui.forms.Input.{input, inputType, inputType_=}

form(actions.credentials) {
  classes = "sign-in-form"
  editable = actions.busy.map(!_)
  on("submit") { event => event.preventDefault(); actions.signIn() }
  label {
    text(i18n"Email") {}
    input("email") {
      inputType = "email"
      autoComplete = "username"
      setAttribute("required", "")
      setAttribute("maxlength", "254")
    }
  }
  label {
    text(i18n"Password") {}
    input("password") {
      inputType = "password"
      autoComplete = "current-password"
      setAttribute("required", "")
      setAttribute("maxlength", "128")
    }
  }
  button(i18n"Sign in") {
    buttonType("submit")
    disabled = actions.busy
  }
}
```

The field names match the model properties. Native labels and a submit button make keyboard submission work. Browser email/required checks improve feedback; the server still validates its own input.

[AccountActions.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/AccountActions.scala) guards duplicate submission and owns busy/error state. It clears the visible password after taking the request snapshot. Invalid credentials receive a generic message; throttling asks the reader to wait. Clearing a field does not erase every immutable JavaScript string from memory.

Disposing the page prevents a late completion from updating its old UI. We do not treat navigation as cancellation of a server-side authentication mutation. The next visit asks the server for its current state.

The route loads `/service/auth/session` with its AbortSignal. A failed initial load renders a retry state; it must not look like a successful anonymous response.

## Run the complete workflow

For local HTTP, explicitly disable the cookie's Secure flag in the terminal that starts the app.

PowerShell:


```powershell
$env:BLOG_COOKIE_SECURE = "false"
sbt --server frontendAssets "application-backend/run"
```

Bash:


```bash
export BLOG_COOKIE_SECURE=false
sbt --server frontendAssets "application-backend/run"
```

Open `http://127.0.0.1:8080/en/account`. Sign in with the bootstrapped administrator, reload, and sign out. The public blog should continue to work in another anonymous browser context.

Outside this local HTTP setup, keep Secure enabled and serve HTTPS. The cookie flag does not add TLS to Undertow; deployment comes later in the series.

With a separate migrated test database configured:


```text
sbt --server "application-backend/testFull"
sbt --server "application-frontend/testFull"
npm ci
npx playwright install chromium
npm run test:browser
```

This checkpoint passes 64 backend tests, 9 Scala.js model tests, and 16 browser contract tests. The latter start the real server but intercept data responses.

For real database-to-browser checks, load the sample posts and bootstrap an administrator in a dedicated test database. Set `BLOG_TEST_ADMIN_EMAIL` and `BLOG_TEST_ADMIN_PASSWORD` for that test account, then run:


```text
npm run test:browser:database
npm run test:browser:auth
```

Those projects add two blog tests and one complete authentication test. They verify sign-in, cookie flags, reload, CSRF rejection, and sign-out against the actual server. The backend tests cover token/session rotation, stale-cookie replay, revocation, throttling, private fields, bounded input, the same principal and roles through Servlet/JAX-RS/Jakarta Security, role changes in the database, and failed login/logout serialization or commit without a persistent session change. The [setup guide](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/docs/user-accounts.md) contains the detailed commands.

We now have a persisted account and a session the server can validate and revoke. Chapter 12 adds registration, email confirmation, and password recovery. Endpoint and field permissions follow in chapter 13, before we expose an editorial write API.
