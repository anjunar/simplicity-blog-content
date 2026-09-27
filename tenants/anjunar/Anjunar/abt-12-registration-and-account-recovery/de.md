# Registrierung und Kontowiederherstellung

Ein Administrator kann sich in unserem Blog anmelden. Ein Leser kann noch kein Konto erstellen, und wer sein Passwort vergisst, kann den Zugang noch nicht wiederherstellen.

Dieses Kapitel fügt beide Pfade hinzu. Ein Besucher fordert eine Bestätigungs-E-Mail an, öffnet seinen Link und wählt ein Passwort. Ein vorhandener Benutzer fordert eine Reset-E-Mail an und wählt einen Ersatz. Beide Links laufen ab, funktionieren einmal und lassen kein verwendbares Token in der Datenbank.

Die Anmeldung läuft weiterhin über die Soteria-Integration aus Kapitel 11. Registrierung und Wiederherstellung ändern die Anmeldeinformationen, die der IdentityStore überprüft. Sie führen keinen weiteren Authentifizierungsmechanismus ein.

Am Ende können wir den gesamten Ablauf im Browser verfolgen und die ausgehenden Nachrichten in einem lokalen Posteingang überprüfen.

## Beginnen Sie mit Kapitel 11

Ausgangspunkt ist das Soteria-Follow-up:

```text
git switch --detach 8c7f7a93fa164a3a3f64395e529a358605621f22
```

Der vollständige Quellcode für diesen Artikel ist:

```text
git switch --detach f1599a3904737ba63ce413e2f5db14fd2f38569f
```

Verwenden Sie eine separate lokale PostgreSQL-Datenbank und die Konfiguration aus Kapitel 11. Die [Einrichtungsanleitung](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/docs/registration-and-recovery.md) enthält die vollständigen PowerShell- und Bash-Befehle. Alle unten stehenden Dateilinks weisen auf die getestete Quellrevision hin.

## Bestätigen Sie die Adresse, bevor Sie ein Passwort auswählen

Wir könnten gemeinsam nach E-Mail und Passwort fragen, ein inaktives Konto erstellen und es aktivieren, wenn jemand einen Link öffnet. Das hinterlässt ein Sicherheitsproblem: jemand kann die Adresse einer anderen Person mit einem Passwort registrieren, das der Angreifer kennt, und die Person mit Zugriff auf das Postfach könnte diese Zugangsdaten später versehentlich aktivieren.

Unsere Registrierung beginnt mit der Adresse allein. Wir speichern ein ausstehendes Bestätigungstoken. Die Person, die die E-Mail öffnen kann, wählt das erste Passwort. Erst dann erstellen wir einen Account.

Die beiden Workflows teilen sich einen Token-Mechanismus, aber ihre Auswirkungen bleiben unterschiedlich:

|Vorgang|Erforderlicher Nachweis|Ergebnis|
| --- | --- | --- |
|Registrierung anfordern|Gültige E-Mail-Adresse und CSRF|Generische Bestätigung; eine berechtigte Adresse erhält einen Link.|
|Registrierung bestätigen|Registrierungstoken, neues Passwort und CSRF|Ein `READER`-Konto anlegen.|
|Zurücksetzung anfordern|Gültige E-Mail-Adresse und CSRF|Generische Bestätigung; ein berechtigtes bestehendes Konto erhält einen Link.|
|Zurücksetzung abschließen|Reset-Token, neues Passwort und CSRF|Das Passwort ersetzen und ältere Sitzungen widerrufen.|

Das Öffnen eines der beiden Links bewirkt nichts für das Konto. Der Benutzer muss ein Formular einreichen. Dies ist wichtig, da E-Mail-Software Links prüfen kann, bevor der Empfänger sie öffnet.

Wir halten den öffentlichen Registrierungsbefehl bewusst klein. Es gibt kein Rollenfeld, keine Konto-ID oder keine allgemeine Entitätsaktualisierung. Der Server weist READER zu; der explizite Administrator-Bootstrap bleibt getrennt.

## Den Nachweis statt des Links speichern

Unsere neue Entität ist [AccountToken.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AccountToken.scala):

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

Das Token selbst stammt von SessionIdentity.newToken(), das bereits 32 kryptographisch zufällige Bytes generiert und als URL-sichere Base64 codiert. Die E-Mail enthält dieses Token. Die Tabelle speichert seinen SHA-256 Digest.

Dies unterscheidet sich vom Passwort-Hashing. Ein Passwort benötigt die in Kapitel 11 eingeführte bewusst teure Ableitung. Unser Token beginnt mit 256 Bits zufälliger Eingabe; ein Digest gibt uns einen indexierten Lookup, ohne das Trägergeheimnis zu behalten.

Der Zweck trennt die Registrierung vom Reset. Ein für eine Operation ausgegebenes Token kann die andere nicht autorisieren. Das eindeutige E-Mail / Zweckpaar gibt jeder Operation einen aktuellen Link. Ein neu angeforderter Link ersetzt den vorherigen. Beide verfallen nach 30 Minuten.

`authenticationVersion` ist für Passwortzurücksetzungen relevant: Eine spätere Credential-Änderung muss einen zuvor ausgestellten Reset-Link ungültig machen. Die Registrierung hat noch kein Konto und speichert -1.

AccountToken ist ein interner Persistenzzustand. Es wird durch die bestehende CDI-Entitätserweiterung entdeckt und erhält stabile SchemaId-Werte. Es besitzt weder ein öffentliches `EntitySchema` noch einen REST-Entity-Graphen oder ein spiegelndes Frontend-Entity-Modell. Das Frontend benötigt nur den Input und das Ergebnis der Operation.

## Eine Schemaänderung anwenden

Aktualisieren Sie die Mail-Abhängigkeiten und prüfen Sie das Schema:

```text
sbt --server "application-backend/update"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
```

Die Vorschau ergänzt `public.blog_account_token`. `Account` und `BlogPost` ändern sich nicht. Wie in Kapitel 6 führt die bestehende benannte Veröffentlichungsregel dazu, dass die schreibgeschützte Vorschau `INCOMPLETE` mit Exit-Code 3 meldet. Überprüfen Sie diesen Befund und die vorgeschlagene Tabelle; führen Sie dann die Migration separat aus:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

Ein Upgrade der Testdatenbank aus Kapitel 11 ergibt:

```text
Applied: revision 3, 1 SQL statements
AlreadyApplied: revision 3, 0 SQL statements
```

Eine neue Datenbank beginnt bei Revision 1 mit den drei aktuellen Tabellen. Wir erstellen keine bestehenden Konten, setzen Passwörter zurück oder ändern veröffentlichte Beiträge.

## Geben Sie einen Link innerhalb der Transaktion aus

[AccountRecovery.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AccountRecovery.scala) besitzt die Datenbankarbeit. Dies ist die vollständige RequestLink-Methode:

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

Der Anrufer hat die Adresse bereits kanonisiert und validiert. Ein Registrierungslink ist nur zulässig, wenn kein Konto existiert. Ein Reset-Link ist nur für ein bestehendes, freigeschaltetes Konto berechtigt.

Wir überprüfen die Mail-Konfiguration vor dieser Entscheidung. Eine nicht konfigurierte Anwendung gibt die gleiche nicht verfügbare Antwort zurück, unabhängig davon, ob die Adresse zu einem Konto gehört.

Bei berechtigten Anfragen entfernen wir abgelaufene Einträge für diese Adresse und ersetzen zu diesem Zweck das vorherige Token. Der explizite Flush entfernt die alte Zeile, bevor der Ersatz auf die einzigartige Einschränkung trifft.

Die Datenbanktransaktion registriert auch eine After-Commit-Aktion. Sie legt unveränderliche Nachrichtendaten erst nach dem Commit des Tokens in die Mail-Warteschlange. Eine fehlgeschlagene Antwort-Serialisierung oder ein fehlgeschlagener Commit hinterlässt kein Token und sendet keine E-Mail. Die bestehende RequestTransaction besitzt diese Grenze.

Der Endpunkt bestätigt immer eine gültige, ungedrosselte Anfrage mit HTTP 202 und dem gleichen Ergebniskörper. Sie verrät nicht, ob ein Konto existiert, unbekannt oder gesperrt ist. SMTP läuft außerhalb des Antwortpfads, so dass das Warten auf einen Mailserver kein Kontoexistenzsignal wird. Dies ist keine zeitlich konstante Garantie: Datenbankpfade unterscheiden sich immer noch.

## Verbrauchen Sie das Token und ändern Sie das Konto zusammen

Der gleiche Dienst übernimmt die letzte Operation:

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

Der erste Lookup gibt nur die Adresse zurück, die mit dem Token und dem Zweck verbunden ist. Wir sperren dann diese Adresse und laden das Token erneut. Diese zweite Suche ist wichtig: Eine andere Anfrage hat sie möglicherweise verbraucht oder ersetzt, während wir gewartet haben.

Für die Registrierung überprüfen wir, ob die Adresse noch unbenutzt ist, leiten den Passwort-Hash ab, erstellen einen READER und verbrauchen das Token in der gleichen Transaktion. Die Antwort meldet den neuen Benutzer nicht automatisch an. Er durchläuft den normalen Soteria-Anmeldeablauf.

Zum Zurücksetzen sperren und aktualisieren wir die Kontozeile. Wir lehnen ein gesperrtes Konto oder eine geänderte Authentifizierungsversion ab. Die Aktualisierung des Passworts erhöht diese Version und verbraucht das Token atomar.

Kapitel 11 vergleicht bereits die Version in jedem Sitzungsprinzip mit dem aktuellen Datenbankwert. Ein Reset widerruft daher jede ältere Sitzung bei der nächsten Anforderung, einschließlich eines anderen Browsers oder Geräts. Das neue Passwort etabliert auch nicht stillschweigend eine neue Sitzung.

Wenn Serialisierung oder Commit fehlschlagen, werden Änderungen an Hash, Version und Token zurückgerollt. Ein noch gültiger Link bleibt für eine Wiederholung verfügbar.

## Einmalige Nutzung auch bei parallelen Anfragen sichern

Eine Prüfung mit anschließender Löschung reicht nicht, wenn zwei Anfragen zusammen ankommen. Beide könnten den Scheck bestehen, bevor eine der Anfragen die Zeile löscht.

Alle Token-Ausgabe und -Verbrauch für eine Adresse erhalten die gleiche transaktionsgebundene PostgreSQL-Advisory-Lock:

```scala
  private def lockEmail(email: String): Unit =
    manager.createNativeQuery(
      "select 1 from pg_advisory_xact_lock(hashtextextended(:email, 12012))", classOf[lang.Integer])
      .setParameter("email", email).getSingleResult
```

Dies serialisiert auch die Registrierung, bevor eine Kontozeile existiert. Die Sperre wird per Commit oder Rollback freigegeben; keine prozesslokale Sperre muss unterschiedliche Datenbankverbindungen koordinieren. Die E-Mail-Adresse wird als Parameter gebunden. Ein eigener Seed trennt diese Sperre von der für den Administrator-Bootstrap.

Das Nachschlagen einer Adresse anhand des Digests vor der Sperre autorisiert noch nichts. Die Autorisierung erfolgt nach dem Erhalten der Sperre, wenn wir das Token erneut lesen und überprüfen. Die Unique Constraint für E-Mail-Adressen der Kontotabelle ist immer noch der letzte Schutz vor einem anderen Kontoerstellungspfad mit Registrierung.

Beim Zurücksetzen schützt die Kontozeilensperre zusätzlich die Versions- und Passwortaktualisierung vor gleichzeitigen Kontoänderungen. Diese Regeln werden durch tatsächliche parallele HTTP-Anforderungen in den Integrationstests abgedeckt.

## Halten Sie die REST-Grenze schmal

[AccountRecoveryResource.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AccountRecoveryResource.scala) stellt diese Endpunkte frei:

|POST-Endpunkt|JSON-Felder|Erfolgreiche Antwort|
| --- | --- | --- |
|/service/auth/register|E-Mail|202, Ergebnis akzeptiert|
|/service/auth/confirm|Token, Passwort|200, Ergebnis abgeschlossen|
|/service/auth/forgot-password|E-Mail|202, Ergebnis akzeptiert|
|/service/auth/reset-password|Token, Passwort|200, Ergebnis abgeschlossen|

[RecoveryRequests.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/RecoveryRequests.scala) enthält die begrenzten Leser und den geteilten AuthJson-Parser. Es liest höchstens 4 KiB, akzeptiert nur die erwarteten Feldnamen und überprüft deren Typen und Längen. E-Mail-Befehle akzeptieren eine kanonische Adresse; Abschlussbefehle erfordern ein korrekt geformtes Token und ein 15-128-Zeichen-Passwort.

Der Login-Reader verwendet diesen Parser nun auch wieder. Passwörter bleiben separate Befehle, niemals Felder, die über den Account.self-Graphen angezeigt werden.

[AuthenticationFilter.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AuthenticationFilter.scala) wendet den bestehenden CSRF-Check auch auf die neue Ressource an. Die Anfrage durchläuft weiterhin Undertow, Elytron und Soteria, nachdem ihre Transaktion begonnen hat. Alle Authentifizierungsantworten bleiben No-Store.

[RecoveryLimiter.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/RecoveryLimiter.scala) hält E-Mail-/Wiederherstellungsversuche vom Anmeldebudget getrennt: fünf Versuche pro Schlüssel und dreißig pro Client-Adresse in einer Minute bei begrenztem Speicherverbrauch. Versuche mit ungültigen Tokens zählen ebenfalls. Diese Grenzen sind lokal für diesen Serverprozess; eine Multi-Instanz-Bereitstellung würde eine gemeinsame Richtlinie erfordern.

## Echte E-Mails versenden und den Zustellstatus definieren

Die [Build-Datei](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/build.sbt) fügt Jakarta Mail 2.1.3, Angus Mail 2.0.4 und Angus Activation 2.0.2 von Maven Central hinzu.

[MailConfig.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/MailConfig.scala) erstellt Kontolinks aus `BLOG_PUBLIC_ORIGIN`. Es leitet seinen Host niemals von der eingehenden Anfrage ab. Außerhalb von localhost muss dieser Ursprung HTTPS verwenden. Es kann keine Anmeldeinformationen, einen Pfad, eine Abfrage oder ein Fragment enthalten.

Die SMTP-Verbindung erfordert standardmäßig STARTTLS und überprüft den Zertifikats-Hostnamen des Servers. Authentifizierungsdaten werden zusammen konfiguriert; Verbindungs-, Lese- und Schreibvorgänge haben endliche Timeouts. Plain SMTP ist ein expliziter Localhost-Entwicklungsmodus.

[AccountMail.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/main/scala/com/anjunar/blog/AccountMail.scala) verwendet einen Arbeiter und eine Warteschlange mit 64 Slots. Der Worker erhält nur unveränderliche Werte, nicht die aktuellen EntityManager, Session oder Request-Scoped Beans. In einer Nachricht ist kein Passwort enthalten.

Hier gibt es eine absichtliche Liefergrenze. Die Warteschlange liegt nur im Arbeitsspeicher. Ein Prozessabsturz, eine volle Warteschlange oder ein SMTP-Ausfall können eine Nachricht nach dem Commit der Datenbank verlieren. HTTP 202 bestätigt die Anfrage; es verspricht nicht, dass eine E-Mail angekommen ist. Wir protokollieren einen generischen Fehler ohne Adressen, Anmeldeinformationen oder Token und lassen den Benutzer einen neuen Link anfordern.

Zuverlässige Retries würden eine dauerhafte Warteschlange oder eine transaktionale Outbox erfordern. Das bedeutet auch, zu entscheiden, wie man Tokens in einer ruhenden Warteschlange schützt. Wir geben nicht vor, dass ein AfterCommit Callback diese Garantie bietet.

Abgelaufene Zeilen können periodisch entfernt werden mit:

```sql
DELETE FROM public.blog_account_token WHERE expires_at <= CURRENT_TIMESTAMP;
```

Die Ablaufprüfung läuft bei jedem Abschlussversuch, unabhängig davon, ob dieses Housekeeping stattgefunden hat oder nicht.

## Verwenden Sie einen lokalen Posteingang

Die [Compose-Datei](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/compose.yaml) beinhaltet einen optionalen Mailpit-Service:

```text
docker compose --profile mail up -d mailpit
```

Halten Sie die Datenbankumgebung aus dem vorherigen Kapitel konfiguriert. Compose validiert die Einstellung des Datenbankpassworts auch beim Starten von Mailpit.

Mailpit erfasst Nachrichten lokal, mit SMTP an Port 1025 und einem Posteingang an http://127.0.0.1:8025. Beide Ports sind an localhost gebunden. Das angeheftete Bild ist v1.31.3; eine native Mailpit-Binärdatei ist eine Alternative, wenn Docker nicht verfügbar ist.

Für PowerShell:

```powershell
$env:BLOG_SMTP_HOST = "127.0.0.1"
$env:BLOG_SMTP_PORT = "1025"
$env:BLOG_SMTP_MODE = "local"
$env:BLOG_MAIL_FROM = "blog@example.test"
$env:BLOG_PUBLIC_ORIGIN = "http://127.0.0.1:8080"
$env:BLOG_COOKIE_SECURE = "false"
sbt --server frontendAssets "application-backend/run"
```

Bash verwendet die gleichen Variablennamen mit Export; die Setup-Anleitung enthält die Vollversion.

Öffnen Sie `/en/register` und geben Sie reader@example.test ein. Lesen Sie die E-Mail in Mailpit, öffnen Sie den Bestätigungslink und wählen Sie ein Passwort. Melden Sie sich danach unter /en/account an. Wiederholen Sie den Ablauf über /en/forgot-password und bestätigen Sie, dass die alte Sitzung und das Passwort nicht mehr funktionieren.

## Lassen Sie den Browser das Token nur so lange wie nötig tragen

Eine Bestätigungs-URL sieht so aus:

```text
http://127.0.0.1:8080/en/confirm#token=<random-token>
```

Das Fragment ist nicht Teil der HTTP-Anfrage. Der Browser übernimmt es in den Seitenzustand und entfernt es aus der Adressleiste. Der Helfer lebt in [RecoveryActions.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/frontend/src/main/scala/com/anjunar/blog/frontend/RecoveryActions.scala):

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

Das Token wird nur im endgültigen POST-Body zusammen mit dem Passwort und dem CSRF-Header der aktuellen Sitzung eingereicht. Kontoseiten erhalten auch No-Store- und No-Referrer-Antwort-Header. Das Token wird nicht in localStorage gespeichert.

Das Neuladen der gereinigten URL verliert das In-Memory-Token. Die Seite teilt dem Benutzer dann mit, den E-Mail-Link wieder zu öffnen oder einen anderen anzufordern. Das erneute Öffnen eines Links, während er sich bereits auf dieser Route befindet, wird über einen Lifecycle-verwalteten Navigations-Listener in BlogPage abgewickelt, der vor dem Router installiert ist. Es lädt das Dokument neu, so dass das neue Token gelesen wird.

[RecoveryPage.scala](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/frontend/src/main/scala/com/anjunar/blog/frontend/RecoveryPage.scala) stellt alle vier Bildschirme mit einem zusammenhängenden Kompositionsbaum dar. Seiten zum Anfordern eines Links binden nur das E-Mail-Feld; Seiten zum Abschließen des Vorgangs binden nur ein New-Password-Feld. Jede neue UI-Nachricht verwendet das i18n-Makro. RecoveryActions behandelt den Ladezustand, löscht Passwörter nach dem Einreichen und ignoriert die Antworten, nachdem die Seite entsorgt wurde.

Die Anfragebestätigung ist bewusst generisch. Der Abschlusserfolg ist spezifisch: Ein Konto ist bereit oder ein Passwort hat sich geändert. Fehlende und abgelaufene Links weisen auf die entsprechende Anfrageseite zurück. Eine Schaltfläche zum erneuten Senden setzt das E-Mail-Formular zurück, ohne ein Dokument neu laden zu müssen.

## Überprüfen Sie auch die Fehlerpfade

Die [Backend-Integrationssuite](https://github.com/anjunar/anjunar-blog-example/blob/f1599a3904737ba63ce413e2f5db14fd2f38569f/application/backend/src/test/scala/com/anjunar/blog/AccountRecoverySpec.scala) startet den echten Server gegen PostgreSQL und erfasst tatsächliche SMTP-Nachrichten. Es überprüft den in der Tabelle gespeicherten Digest, das Verfallsdatum und den Ersatz, die Zwecktrennung, CSRF, fehlerhafte Befehle, Drosselung und gleichzeitige Verwendung.

Es erzwingt auch Serialisierung und Transaktionsfehler. Diese Tests beweisen, dass die fehlgeschlagene Ausgabe keine E-Mails sendet und die fehlgeschlagene Bestätigung / Zurücksetzung den Link nicht verbraucht oder die Anmeldeinformationen teilweise aktualisiert. Bestehende Session-Tests üben weiterhin den echten Soteria-Pfad aus.

Legen Sie die lokale Testumgebung wie in der Einrichtungsanleitung beschrieben fest, einschließlich eines freien SMTP-Ports wie 27125 und `BLOG_PUBLIC_ORIGIN=http://127.0.0.1:18080`. Die automatisierte Suite startet einen eigenen SMTP-Testserver; verbinden Sie sie nicht mit einer laufenden Mailpit-Instanz.

```text
sbt --server "application-backend/testFull" "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
npx playwright test --project=recovery
```

Die abgeschlossene Revision umfasst 79 bestandene Backend-Tests, neun Scala.js-Modelltests und zwanzig Browser-Vertragstests. Der zusätzliche echte Wiederherstellungsbrowsertest registriert einen Leser, folgt der erfassten E-Mail, meldet sich an, setzt das Passwort zurück, überprüft den Widerruf der Sitzung und lehnt die Wiederverwendung des alten Links ab.

Mit den früheren Sample-Posts und Administratoren laufen alle vier Playwright-Projekte 24 Tests. Browser-Screenshots umfassen Desktop-Registrierung und einen mobilen Wiederherstellungsfehler. Automatisierte E-Mails bleiben auf localhost; kein externer Posteingang ist erforderlich.

Der Browser-Workflow lässt einen eindeutig benannten Leser in seiner dedizierten Testdatenbank. Backend-Tests entfernen ihre eigenen Zeilen. Behandeln Sie lokale Posteingänge und fehlgeschlagene Testartefakte als privat, da sie Live-Links enthalten können.

Unsere Leser können jetzt ein Konto erstellen und den Zugriff wiederherstellen. Das nächste Kapitel verwendet diese Identität, um zu entscheiden, wer lesen, bearbeiten und veröffentlichen darf, und um die erlaubten Aktionen durch HATEOAS aufzudecken.

Weitere Informationen: [OWASP Passwortwiederherstellung](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html), [Angus SMTP-Konfiguration](https://eclipse-ee4j.github.io/angus-mail/docs/api/org.eclipse.angus.mail/org/eclipse/angus/mail/smtp/package-summary.html) und [Installation von Mailpit](https://mailpit.axllent.org/docs/install/).
