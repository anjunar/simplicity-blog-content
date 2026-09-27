Der Blog bedient bereits veröffentlichte Posts. Jetzt muss er die Person erkennen, die ihn verwendet.

Am Ende dieses Kapitels kann ein Operator den ersten Administrator erstellen, sich bei `/en/account` anmelden, die Seite neu laden, ohne die Sitzung zu verlieren, und sich abmelden. Leser können den öffentlichen Blog immer noch anonym durchsuchen.

Beginnen Sie mit dem [Kapitel 10 Quelle](https://github.com/anjunar/anjunar-blog-example/tree/db992619d48e8938308216641856849e8be9bd97). Der [Checkpoint für Kapitel 11](https://github.com/anjunar/anjunar-blog-example/tree/8c7f7a93fa164a3a3f64395e529a358605621f22) enthält die lauffähige Implementierung und Tests. Dateipfade unten sind relativ zum Repository-Verzeichnis.

Öffentliche Registrierung, E-Mail-Bestätigung und Passwortwiederherstellung gehören zu Kapitel 12. Dieses Kapitel legt die Identität und Sitzung fest, die diese Workflows verwenden werden.

## Soteria in den eingebetteten Server integrieren

Wir verwenden [Jakarta Security](https://jakarta.ee/specifications/security/4.0/jakarta-security-spec-4.0) für die Authentifizierung, mit Soteria als Implementierung. Fügen Sie diese Abhängigkeiten dem `libraryDependencies` des Backends in `build.sbt` hinzu:

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

Führen Sie danach aus:

```text
sbt --server "application-backend/update"
```

Ein eingebetteter Servlet-Container benötigt die Integration, die normalerweise von einem Anwendungsserver bereitgestellt wird. Elytron verbindet Undertow mit Jakarta Authentication; Die Brücke von Soteria verbindet diese Schicht mit unserem CDI-Authentifizierungsmechanismus.

[SoteriaIntegration.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SoteriaIntegration.scala) konfiguriert die Elytron-Sicherheitsdomäne und die Authentifizierungsfabrik. `ApplicationMain` ruft es vor dem Start der Bereitstellung auf. [SoteriaInitializer](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SoteriaInitializer.scala) führt den Servlet-Initializer von Soteria aus und überprüft, ob sein Authentifizierungsmodul registriert wurde. Ein fehlender Provider verhindert den Start.

Der offizielle Soteria Weld-Adapter gibt den generierten Identitätsspeicher und Mechanismushandler-Beans in Weld unterschiedliche Identitäten. Die Abhängigkeiten für CDI EL, JSON und Nimbus liefern Typen, die vom CDI-Bootstrap von Soteria verwendet werden; das Hinzufügen von ihnen ermöglicht in diesem Kapitel keine OpenID Connect-Anmeldung.

Zwei kleine Dateien vervollständigen das Embedded Setup. [CdiNamingFactory](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/CdiNamingFactory.scala) zeigt Welds aktuellen BeanManager unter `java:comp/BeanManager`, wo Soteria danach sucht. Mit [SoteriaCallerDetails](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SoteriaCallerDetails.scala) kann Jakarta Security die in Undertow bereits etablierten Principal und Rollen auslesen; es ist über `META-INF/services` registriert.

Die Einrichtungsanleitung dokumentiert diese Integration an einer Stelle. Wir betreiben weiterhin eine einzige Embedded-Anwendung.

## Das Konto und seinen öffentlichen Vertrag definieren

Erstellen Sie [Account.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/Account.scala) im Backend. Es ist eine JPA-Entität, die `public.blog_account` zugeordnet ist, mit diesen Feldern:

|Feld|Zweck|An angemeldete Benutzer zurückgegeben?|
| --- | --- | --- |
|`id`|Generierte UUID|Ja|
|`version`|Optimistisches Locking|Ja|
|`email`|Eindeutige Anmeldeadresse|Ja|
|`role`|`ADMIN` oder `READER`|Ja|
|`passwordHash`|Prüfwert für ein Passwort mit Salt|Nein|
|`authenticationVersion`|Bestehende Sitzungen widerrufen|Nein|
|`locked`|Anmeldung und bestehende Sitzungen sperren|Nein|

Die E-Mail verfügt über `@Email`, `@NotBlank` und `@Size(max = 254)`-Validierung sowie eine benannte Datenbankeinschränkung. Unsere Anwendung behandelt E-Mail-Adressen als fallunempfindlich: Sie entfernt umgebende Leerzeichen und wandelt die Adresse mit `Locale.ROOT` in Kleinbuchstaben um vor der Erstellung oder dem Nachschlagen.

Wie in Kapitel 5 brauchen Annotationen an Feldern des Klassenkörpers kein ausdrückliches `@field`-Ziel. Jede Tabelle und Spalte erhält ein stabiles `@SchemaId`. CDI entdeckt die Entität durch die bestehende Erweiterung; es gibt keine Klassenliste zum Aktualisieren in `Persistence`.

Das Konto bleibt das REST-Modell. Sein `Account.self`-Graph wählt die vier öffentlichen Felder aus. Das Begleitschema deklariert dieselben Felder:


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

Die persistenten Felder verwenden `reference`, so dass E-Mail-Lookup `account.get(schema.email)` in einer typisierten Criteria-Abfrage verwenden kann. Die Standardregeln verbieten weiterhin eingehende Änderungen an der Entität.

Die drei internen Felder tragen `@JsonbTransient` und fehlen im Mapper-Schema, Graphen und Browsermodell. Ein Passwort-Hash gehört zu den sensiblen Serverdaten, obwohl es nicht das ursprüngliche Passwort ist.

`version` und `authenticationVersion` sind getrennt. Die JPA-Version erhöht sich bei Entity-Änderungen. Die Authentifizierungsversion erhöhen wir gezielt, wenn eine Sicherheitsoperation alle bestehenden Sitzungen widerrufen muss.

## Fügen Sie die Tabelle hinzu, ohne den Blog zu ersetzen

Behalten Sie die Datenbankeinstellungen aus dem vorherigen Kapitel bei. Beenden Sie die Anwendung und führen Sie aus:


```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

Eine Datenbank aus Kapitel 10 erreicht Revision 2 und erhält die Tabelle `blog_account`. Bestehende Artikel-IDs, Inhalte und die Veröffentlichungsregel bleiben erhalten. Eine neue Datenbank beginnt mit beiden Tabellen bei Revision 1; ein erneutes `migrate` meldet `AlreadyApplied`.

Wie in Kapitel 6 erläutert, kann eine Vorschau mit einer bestehenden namens CHECK-Einschränkung `INCOMPLETE` melden und mit Code 3 aussteigen. Der Migrationsausführer überprüft diese Einschränkung unter seiner Sperre. Löschen Sie die Artikeltabelle nicht, nur um eine erfolgreiche Vorschau zu erhalten.

## Einen Passwort-Verifier speichern

[PasswordHash.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/PasswordHash.scala) verwendet PBKDF2-HMAC-SHA256 aus dem JDK. Jedes Passwort erhält ein frisches 16-Byte-Salz und einen 256-Bit-abgeleiteten Schlüssel mit 600.000 Iterationen.

Dies hält das Beispiel innerhalb der kryptographischen APIs des JDK. PBKDF2 ist rechenaufwendig, bietet aber nicht die Speicherhärte von Argon2id. OWASP bevorzugt im Allgemeinen Argon2id und listet 600.000 Iterationen für PBKDF2-HMAC-SHA256 auf. Siehe die [OWASP-Empfehlungen zur Passwortspeicherung](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html).

Hier die komplette Implementierung:


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

Der gespeicherte String enthält den Algorithmus, den Arbeitsfaktor, das Salz und den abgeleiteten Schlüssel. Die Überprüfung akzeptiert nur das Format, das dieses Kapitel erstellt; es vertraut nicht darauf, dass ein gespeicherter Wert einen willkürlich teuren Arbeitsfaktor auswählt. Ältere Formate zu unterstützen und zu aktualisieren, würde eine explizite Migrationspolitik erfordern.

Neue Passwörter müssen 15-128 Zeichen enthalten, gemessen mit `String.length`. Leerzeichen bleiben erhalten und legen keine Regeln wie "ein Symbol und ein Großbuchstabe" fest.

Unbekannte Konten führen weiterhin eine Passwortableitung gegen einen Dummy-Hash durch. Falsche Passwörter, unbekannte Adressen und gesperrte Konten geben die gleiche 401-Antwort zurück. Dies reduziert offensichtliche Account-Discovery-Signale; es führt nicht dazu, dass die gesamte Anforderung in konstanter Zeit ausgeführt wird.

## Erstellen Sie den ersten Administrator explizit

Die Anwendung fördert nicht den ersten Besucher. Eine betreibende Person führt [BootstrapAdminMain](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/BootstrapAdminMain.scala) einmal aus.

Wenn die Datenbankvariablen bereits festgelegt sind, verwenden Sie PowerShell:


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

Oder Bash:


```bash
export BLOG_ADMIN_EMAIL=admin@example.com
read -r -s -p "Administrator password: " BLOG_ADMIN_PASSWORD
echo
export BLOG_ADMIN_PASSWORD
sbt --server "application-backend/runMain com.anjunar.blog.BootstrapAdminMain"
unset BLOG_ADMIN_PASSWORD BLOG_ADMIN_EMAIL
```

Der Befehl öffnet einen CDI-Anforderungskontext und eine Schreibtransaktion. Es erfordert eine PostgreSQL-Transaktionsberatungssperre, bevor überprüft wird, ob ein Administrator existiert. Die Sperre verhindert, dass zwei gleichzeitig gestartete Bootstrap-Prozesse die Prüfung beide bestehen.

Wenn ein Administrator existiert, schlägt Bootstrap fehl. Es lehnt auch eine E-Mail ab, die bereits einem anderen Konto zugewiesen wurde. Der Befehl setzt keine Zugangsdaten zurück und befördert keinen bestehenden Benutzer. Bei Erfolg hasht er das Passwort, speichert das Konto, committet die Transaktion und gibt die neue UUID aus.

Das Passwort reist durch die kurzlebige Prozessumgebung, nicht als Befehlsargument oder in einer eingecheckten Konfigurationsdatei. Entfernen Sie diese Umgebungsvariable danach. Behalten Sie diesen Befehl außerhalb der HTTP-API.

## Validierung von Anmeldeinformationen über einen IdentityStore

Erstellen Sie [PasswordIdentityStore.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/PasswordIdentityStore.scala):

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

Seine Aufgabe: Zugangsdaten entgegennehmen und einen validierten Principal mit Gruppen zurückgeben. Es liest keine HTTP-Header, erstellt keine Cookies oder ändert eine Sitzung. Nicht unterstützte Anmeldeinformationen geben `NOT_VALIDATED_RESULT` zurück; abgelehnte Passwörter geben `INVALID_RESULT` zurück.

Das injizierte `IdentityStoreHandler` stammt von Soteria. Er findet unseren CDI-IdentityStore und verarbeitet sein Validierungsergebnis. Unser [SoteriaAuthenticationMechanism](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SoteriaAuthenticationMechanism.scala) verbindet dieses Ergebnis mit HTTP:

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

`SessionPrincipal` erweitert Jakarta Security `CallerPrincipal`. Nach erfolgreicher Überprüfung übergibt `notifyContainerAboutLogin` es und seine Gruppen an den Container. Bei einer bestehenden Sitzung prüft der Mechanismus das aktuelle Datenbankkonto und gibt die aktuelle Rolle an.

Undertow kann die Authentifizierung starten, bevor die JAX-RS-Anforderungstransaktion existiert. Das Request-Attribut in diesem Mechanismus ist ein interner Lifecycle-Marker: `AuthenticationFilter` setzt ihn nach dem Beginn von `TransactionBoundary` ein und fordert Undertow auf, sich erneut zu authentifizieren. Es ist kein Client-Header oder ein Berechtigungsflag. Dies hält die Datenbankarbeit in unserem bestehenden Persistenzkontext.

`setIntegratedJaspi(false)` in der Elytron-Konfiguration bedeutet, dass die verifizierte Identität von Soteria keine zweite Kontosuche in einem Elytron-Reich durchläuft. Das Passwort wird weiterhin von unserem IdentityStore überprüft.

## Lassen Sie den Server die Sitzung besitzen

Nach der Anmeldung sendet der Browser ein undurchsichtiges `BLOGSESSION`-Cookie. Der Server speichert eine `SessionPrincipal`, die die Konto-UUID, ihre Authentifizierungsversion und die Anmeldezeit enthält.

[SessionIdentity.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SessionIdentity.scala) wird vom Soteria-Mechanismus aufgerufen, um das Konto auf authentifizierte REST-Anfragen der Anwendung neu zu laden; Liveness ist ausgeschlossen. Die Identität ist nur gültig, während das Konto existiert, freigeschaltet ist, die aufgezeichnete Authentifizierungsversion hat und sich innerhalb der absoluten Lebensdauer von acht Stunden befindet. Rollenüberprüfungen verwenden die aktuelle Datenbankrolle.

Ein gelöschtes oder gesperrtes Konto verliert deshalb bei der nächsten geschützten Anfrage seine Sitzung. Das Erhöhen von `authenticationVersion` macht alle Sitzungen mit dem alten Wert ungültig. Es gibt noch keinen Account-Management-Endpunkt; die Tests üben diese Änderungen direkt in ihren eigenen Einrichtungen aus.

[ApplicationMain.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/ApplicationMain.scala) konfiguriert Servlet-Sitzungen:

- Nur Cookie-Tracking, so dass eine Sitzungs-ID nicht über die URL bereitgestellt werden kann.
- `HttpOnly`, `SameSite=Lax`, `Path=/` und kein `Domain`.
- `Secure` standardmäßig mit einem expliziten lokalen HTTP-Override.
- Eine 15-minütige Leerlaufzeit und maximal 2.048 aktive Sitzungen.

Sitzungen liegen in diesem Prozess. Ein Neustart meldet alle Benutzer ab. Erfolgreiches Anmelden ändert die Sitzungs-ID; das Abmelden macht die alte Sitzung ungültig. Diese Entscheidungen folgen dem [OWASP Session Management Anleitung](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html).

Die kleine [SessionServer](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/SessionServer.scala)-Unterklasse macht auch das Herunterfahren explizit. Mit unserer RESTEasy-Version müssen Session-Hörer beenden, während Weld noch verfügbar ist, bevor die verbleibende Server-Abschaltung CDI schließt.

## An- und Abmeldung mit CSRF schützen

Cookies werden automatisch gesendet. Ein Cookie allein beweist damit nicht dafür, dass der Benutzer eine Zustandsänderungsanfrage beabsichtigt hat.

Unsere vier Endpunkte sind in [AuthenticationResource.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/AuthenticationResource.scala) definiert:

|Anfrage|Ergebnis|
| --- | --- |
|`GET /service/auth/session`|Ein CSRF-Token und das optionale Safe-Konto|
|`GET /service/auth/me`|`Data[Account]` oder 401, wenn anonym|
|`POST /service/auth/login`|Validierung von Anmeldeinformationen und Feststellung der Identität|
|`POST /service/auth/logout`|Invalidieren Sie die alte Sitzung und geben Sie den anonymen Zustand zurück|

`GET session` erstellt bei Bedarf eine anonyme Serversitzung. Es gibt ein zufälliges Synchronisierungs-Token zurück, das zu dieser Sitzung gehört. Login und Logout müssen dieses Token in `X-CSRF-Token` senden; ein fehlendes Token oder ein Token aus einer anderen Sitzung erhält 403. Anfragen mit der Bezeichnung `Sec-Fetch-Site: cross-site` werden ebenfalls abgelehnt.

Die Anmeldung benötigt auch CSRF-Schutz: Eine gefälschte Anmeldung kann das Opfer bei einem Konto anmelden, das von jemand anderem kontrolliert wird. `SameSite` ergänzt den Synchronizer-Token-Check. Siehe [CSRF-Leitlinien der OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html).

[AuthenticationFilter.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/AuthenticationFilter.scala) löst die Identität auf, nachdem der Transaktionsfilter die Anforderung gestartet hat. Er aktiviert den Authentifizierungspfad von Undertow, wendet CSRF-Prüfungen auf unsichere Methoden auf `AuthenticationResource` an und markiert Auth-Antworten `Cache-Control: no-store`. RESTEasy liest Principal und Rollen aus der authentifizierten Servlet-Anfrage. Neue Credential-Ressourcen müssen sich diesem Schutz anschließen, wenn wir sie hinzufügen.

## Vor der dauerhaften Anmeldung committen

Ein korrektes Passwort ist notwendig, aber die Anfrage muss noch erfolgreich abgeschlossen werden. Wir sollten keinen Browser einloggen lassen, wenn die Antwort-Serialisierung oder der Transaktions-Commit fehlschlägt.

Dies ist die Anmeldemethode in `AuthenticationResource`. Die Klasse injiziert Jakarta Security `SecurityContext` zusammen mit den Entity / Session Services. Es erhält die Servlet-Anfrage und Antwort über `@Context`:


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

Der [LoginRequest-Reader](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/LoginRequest.scala) akzeptiert nur die beiden Stringfelder `email` und `password`, mit einer Gesamtkörpergrenze von 4 KiB. Zusätzliche Felder wie `role` werden abgelehnt. Anmeldeinformationen sind ein spezifischer Befehl, keine allgemeine Kontoaktualisierung.

[LoginLimiter](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/LoginLimiter.scala) ermöglicht fünf Versuche pro kanonischer E-Mail und dreißig pro direkter Client-Adresse in einem 60-Sekunden-Fenster. Auch erfolgreiche Versuche zählen. Es begrenzt seine Schlüsselzuordnung und ermöglicht vier gleichzeitige Passwortprüfungen; weitere Anfragen erhalten 429 mit `Retry-After: 60`. Sein Zustand ist lokal für diesen Prozess und wird beim Neustart zurückgesetzt. Forwarded-Adress-Header sind nicht vertrauenswürdig; ein Reverse-Proxy würde derzeit einen Adress-Bucket für seine Clients teilen.

Der [RequestTransaction.afterCommit](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/backend/src/main/scala/com/anjunar/blog/RequestTransaction.scala) Callback läuft erst nach erfolgreicher Serialisierung und Commit. Soteria hat die aktuelle Anfrage zu diesem Zeitpunkt authentifiziert. Nur dann macht `identity.establish` den Principal für spätere Anfragen dauerhaft und erneuert Sitzungs-ID sowie CSRF-Token. Der Logout verwendet den gleichen Rückrufmechanismus: `request.logout()` ruft die Bereinigung von Soteria durch Elytron auf, gefolgt von der Ungültigerklärung der Sitzung.

Wir verlangen bewusst keine automatische Sitzungsregistrierung von Soteria. Dies würde die Authentifizierung fortsetzen, bevor unsere Antwort- und Datenbanktransaktion abgeschlossen ist.

Der vorhandene Writer puffert JSON bereits, während der EntityManager geöffnet ist. Wir bewahren diese Grenze und führen die Sitzungsmutation durch, bevor wir die gepufferte Antwort senden. Eine abgelehnte Anfrage, ein Schreibfehler oder ein fehlgeschlagenes Commit verwerfen den Rückruf.

Dies kann die Lieferung an den Browser nicht garantieren. Das Netzwerk kann fehlschlagen, nachdem der Server den Vorgang abgeschlossen hat; die Kontoseite liest die aktuelle Sitzung erneut, wenn sie wieder geöffnet wird.

## Spiegeln Sie das sichere Konto in Scala.js

Fügen Sie die Formularbibliothek neben Core, JSON und Router in den Frontend-Abhängigkeiten hinzu:


```scala
"com.anjunar" %% "scalajs-ui-forms" % "1.0.9",
```

Es kommt von Maven Central. Aktualisieren Sie die Abhängigkeitsauflösung mit:


```text
sbt --server "application-frontend/update"
```

Erstellen Sie das Frontend [Account.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/Account.scala):


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

Das Frontend-Kontomodell spiegelt die vier veröffentlichten Entity-Felder wider, einschließlich Version Null. `LoginCredentials` enthält die an das Formular gebundenen Eigenschaften. `LoginCommand` macht eine Momentaufnahme für den Transport. Kein Anmeldemodell ist eine zweite Version der persistenten Account-Entität.

[AccountService.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/AccountService.scala) erhält das aktuelle CSRF-Token vor jeder Mutation:


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

Das freigegebene [HttpJson](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/HttpJson.scala) unterstützt jetzt JSON POST-Anfragen, Anmeldeinformationen gleichen Ursprungs und den CSRF-Header. Das Session-Cookie wird weder in `localStorage` noch in `sessionStorage` kopiert.

## Binden Sie das Formular und behandeln Sie seinen Lebenszyklus

[AccountPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/AccountPage.scala) behält seinen gesamten UI-Baum in `compose`. Die Seite zeigt entweder das Anmeldeformular oder das aktuelle Konto und eine Abmeldeschaltfläche an. Jede neue Schnittstellennachricht verwendet das `i18n`-Makro.

Dieser Formularblock gehört in den `render(this, cursor)`-Baum dieser Seite:


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

Die Feldnamen stimmen mit den Modelleigenschaften überein. Native Labels und ein Submit-Button lassen die Eingabe von Tastaturen funktionieren. Browser-E-Mail / erforderliche Überprüfungen verbessern das Feedback; der Server validiert weiterhin seine eigenen Eingaben.

[AccountActions.scala](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/application/frontend/src/main/scala/com/anjunar/blog/frontend/AccountActions.scala) schützt die doppelte Formularübermittlung und besitzt den Lade- und Fehlerzustand. Es löscht das sichtbare Passwort, nachdem es den Snapshot für die Anfrage gemacht hat. Ungültige Anmeldeinformationen erhalten eine generische Nachricht; Drosselung fordert den Leser auf zu warten. Durch das Löschen eines Feldes wird nicht jeder unveränderliche JavaScript-String aus dem Speicher gelöscht.

Das Entsorgen der Seite verhindert, dass eine verspätete Fertigstellung die alte Benutzeroberfläche aktualisiert. Wir behandeln die Navigation nicht als Löschung einer serverseitigen Authentifizierungsmutation. Der nächste Besuch fragt den Server nach seinem aktuellen Zustand.

Die Route lädt `/service/auth/session` mit seinem AbortSignal. Ein fehlgeschlagenes anfängliches Laden führt in einen Zustand mit Wiederholungsoption; es darf nicht wie eine erfolgreich geladene anonyme Sitzung aussehen.

## Führen Sie den vollständigen Workflow aus

Deaktivieren Sie bei lokalem HTTP ausdrücklich das Secure-Flag des Cookies im Terminal, das die App startet.

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

Öffnen Sie `http://127.0.0.1:8080/en/account`. Melden Sie sich beim zuvor angelegten Administrator an, laden Sie neu und melden Sie sich ab. Der öffentliche Blog sollte weiterhin in einem anderen anonymen Browserkontext arbeiten.

Halten Sie außerhalb dieses lokalen HTTP-Setups Secure aktiviert und bedienen Sie HTTPS. Das Cookie-Flag fügt TLS nicht zu Undertow hinzu; die Bereitstellung erfolgt später in der Serie.

Mit einer separaten migrierten Testdatenbank konfiguriert:


```text
sbt --server "application-backend/testFull"
sbt --server "application-frontend/testFull"
npm ci
npx playwright install chromium
npm run test:browser
```

Dieser Checkpoint besteht 64 Backend-Tests, 9 Scala.js-Modelltests und 16 Browser-Vertragstests. Letztere starten den echten Server, fangen aber Datenantworten ab.

Für echte Datenbank-zu-Browser-Checks laden Sie die Beispielbeiträge und booten Sie einen Administrator in einer dedizierten Testdatenbank. Setzen Sie `BLOG_TEST_ADMIN_EMAIL` und `BLOG_TEST_ADMIN_PASSWORD` für dieses Testkonto und führen Sie dann aus:


```text
npm run test:browser:database
npm run test:browser:auth
```

Diese Projekte fügen zwei Blog-Tests und einen vollständigen Authentifizierungstest hinzu. Sie überprüfen die Anmeldung, Cookie-Flags, das Neuladen, die CSRF-Ablehnung und die Anmeldung gegen den eigentlichen Server. Die Backend-Tests umfassen Token / Session-Rotation, abgestandene Cookie-Wiedergabe, Widerruf, Drosselung, private Felder, begrenzte Eingaben, die gleichen Haupt- und Rollen durch Servlet / JAX-RS / Jakarta Security, Rollenänderungen in der Datenbank und fehlgeschlagene Login / Logout-Serialisierung oder Commit ohne dauerhafte Sitzungsänderung. Die [Einrichtungsanleitung](https://github.com/anjunar/anjunar-blog-example/blob/8c7f7a93fa164a3a3f64395e529a358605621f22/docs/user-accounts.md) enthält die detaillierten Befehle.

Wir haben jetzt ein persistentes Konto und eine Sitzung, die der Server validieren und widerrufen kann. Kapitel 12 fügt Registrierung, E-Mail-Bestätigung und Passwortwiederherstellung hinzu. Endpunkt- und Feldberechtigungen folgen in Kapitel 13, bevor wir eine redaktionelle Schreib-API freigeben.
