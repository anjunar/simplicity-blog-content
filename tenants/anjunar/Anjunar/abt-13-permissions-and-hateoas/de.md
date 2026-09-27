# Berechtigungen und HATEOAS

Unser Blog weiß, wer angemeldet ist. Er muss noch entscheiden, was diese Person tun kann.

Dieses Kapitel gibt dem Administrator einen kleinen redaktionellen Arbeitsbereich. Administratoren können einen Entwurf öffnen, veröffentlichen und die Veröffentlichung wieder zurückziehen. Ein Leser kann weiterhin öffentliche Beiträge lesen, kann diesen Arbeitsbereich jedoch nicht betreten.

Wir werden eine Entscheidung den ganzen Weg durch die Anwendung tragen: **Darf dieser Aufrufer diesen Artikel jetzt veröffentlichen?** Das Backend beantwortet es, erzwingt es und beinhaltet die entsprechende Aktion in der Antwort. Der Browser verwendet diese Antwort, um einen Publish-Button anzuzeigen.

## Beginnen Sie mit Kapitel 12

Ausgangspunkt ist das abgeschlossene Registrierungs- und Wiederherstellungskapitel:

```text
git switch --detach f1599a3904737ba63ce413e2f5db14fd2f38569f
```

Die fertige Quelle für diesen Artikel ist:

```text
git switch --detach 37f2a3e6a7d440be4bbc99730bc430915f1bdaa4
```

Die [begleitende Anleitung](https://github.com/anjunar/anjunar-blog-example/blob/37f2a3e6a7d440be4bbc99730bc430915f1bdaa4/docs/permissions-and-hateoas.md) enthält die Setup- und Verifizierungsschritte. Verwenden Sie die vorhandene Entwicklungsdatenbank und den zuvor angelegten Administrator. Alle Abhängigkeiten kommen immer noch von Maven Central.

Es gibt keine neuen Tabellen oder Spalten. Gegen die Datenbank aus Kapitel 12 meldet `SchemaMain migrate` `AlreadyApplied`, Revision 3 und null SQL-Anweisungen.

Das optionale Skript `database/examples/public-posts.sql` liefert einen öffentlichen Artikel und den Entwurf „Our private draft“. Laden Sie diese Beispieldaten in die Entwicklungsdatenbank, bauen Sie `frontendAssets` und starten Sie `application-backend/run`. Bei lokalem HTTP behalten Sie `BLOG_COOKIE_SECURE=false` aus Kapitel 11 bei.

Melden Sie sich unter `/en/account` an. Der neue Link „Open editorial“ führt zur Artikelliste.

## Entscheiden Sie, was jede Schicht schützt

Eine Publikationsanfrage wirft verschiedene Fragen auf:

|Fragestellung|Wo wir es beantworten|
| --- | --- |
|Wer stellt die Anfrage?|Soteria und der Container-Sicherheitskontext.|
|Kann dieser Anrufer den redaktionellen Endpunkt aufrufen?|EndpointPolicy und AuthorizationFilter.|
|Kann dieser spezielle Beitrag veröffentlicht werden?|PostAccess und die BlogPost-Domänenmethode.|
|Welche Felder darf der Mapper lesen oder schreiben?|Regeln in `BlogPost.Schema`.|
|Welche Aktionen sollte der Browser jetzt anbieten?|`PostLinks` für diese Antwort.|

Diese Entscheidungen kooperieren, sind aber nicht austauschbar. Ein Graph, der Inhalt auswählt, ist keine Berechtigung zum Lesen eines Entwurfs. Eine versteckte Schaltfläche stoppt keine direkte HTTP-Anfrage. Eine Rolle allein macht einen bereits veröffentlichten Post nicht wieder veröffentlichbar.

## Soteria als Quelle der Identität

Kapitel 11 verband Soteria bereits mit Undertow. Wir nutzen diese Integration, anstatt einen anderen Principal oder ein zweites Rollensystem einzuführen.

Die vollständige Klasse `application/backend/src/main/scala/com/anjunar/blog/CallerAccess.scala` ist klein:

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

Die Anfrage löst die Sitzung weiterhin mit dem aktuellen Konto in der Datenbank auf. Ein gesperrtes Konto oder eine geänderte Authentifizierungsversion macht die Sitzung ungültig; bei der nächsten Anforderung gilt eine geänderte Rolle. `CallerAccess` liest die Identität des Containers für diese Anfrage.

Sein Request-Scope ist wichtig. Wir dürfen das Berechtigungsergebnis eines Administrators nicht für eine spätere anonyme Anfrage wiederverwenden.

## Setzen Sie eine explizite Richtlinie für jeden Endpunkt

Öffentliche Ressourcen erhalten `@PermitAll`. Die neue Klasse `EditorialPostsResource` verwendet `@RolesAllowed(Array("ADMIN"))` aus `jakarta.annotation.security`.

Die Klasse bedient diese Operationen:

|Methode|API-Pfad|Zweck|
| --- | --- | --- |
|GET|`/service/editorial/posts`|Begrenzte Liste aus Entwürfen und veröffentlichten Artikeln.|
|GET|`/service/editorial/posts/{id}`|Vollständige redaktionelle Vorschau.|
|POST|`/service/editorial/posts/{id}/publish`|Einen nicht leeren Entwurf veröffentlichen.|
|POST|`/service/editorial/posts/{id}/retract`|Einen veröffentlichten Artikel wieder zum Entwurf machen.|

In `EndpointPolicy.scala` prüft die Policy-Suche die Methode vor der Klasse:

```scala
  def of(method: Method, resource: Class[?]): EndpointPolicy =
    declared(method).orElse(declared(resource)).getOrElse(EndpointPolicy.Denied)
```

Diese Methode gehört zum EndpointPolicy-Begleiter und importiert `Method` aus `java.lang.reflect`. Der gezeigte Helfer liest `PermitAll`, `DenyAll` und `RolesAllowed`. Sie lehnt widersprüchliche Erklärungen zum gleichen Element ab.

Eine Annotation an der Methode hat Vorrang vor der Annotation an der Klasse, gemäß [Jakarta Annotationen](https://jakarta.ee/specifications/annotations/3.0/annotations-spec-3.0). Das Sperren eines Endpunkts ohne Annotation ist der explizite Standard unserer Anwendung.

Das Hinzufügen einer Annotation verbindet unsere eingebettete REST-Bereitstellung nicht mit ihrer Durchsetzung. Der Autorisierungsfilter liest die Richtlinie und ruft requireAccess auf, bevor die Ressourcenmethode ausgeführt wird. Eine anonyme Anfrage an einen rollengeschützten Endpunkt erhält 401; ein authentifizierter Leser erhält 403.

Die Anforderungsfilter laufen in dieser Reihenfolge:

1. TransactionBoundary öffnet die Anfragetransaktion mit Priorität 2000.
2. Der Authentifizierungsfilter löst Soterias Aufrufer bei 2100 auf.
3. Autorisierungsfilter überprüft den Endpunkt bei 2200.

Autorisierungsfilter prüft auch CSRF auf unsichere Anforderungen an die Authentifizierungs-, Wiederherstellungs- und redaktionellen Ressourcen. Wenn ein späteres Kapitel eine weitere Schreibressource hinzufügt, muss sie diesem Schutz beitreten. Eine gültige Rolle ist kein Ersatz für ein gültiges CSRF-Token.

## Den konkreten Artikel prüfen

Die Endpunktberechtigung lässt den Aufrufer bis zur Ressource gelangen. `PostAccess.scala` prüft anschließend die konkrete Entität und ihren Zustand:

```scala
  def canPublish(post: BlogPost): Boolean =
    canEdit(post) && post.status == BlogPostStatus.DRAFT &&
      post.content != null && !post.content.isBlank

  def canRetract(post: BlogPost): Boolean =
    canEdit(post) && post.status == BlogPostStatus.PUBLISHED
```

Hier erfordert canEdit einen nicht nullwertigen Artikel und einen Administrator. canRead ermöglicht veröffentlichte Posts oder einen Administrator. Wir haben noch keine Autorenbeziehung, also gibt es keine Eigentumsregel zu erfinden.

Die öffentliche API verwendet weiterhin nur veröffentlichte Abfragen. Selbst ein Administrator kann einen Entwurf nicht über `/service/blog/posts/{slug}` abrufen; Preview ist eine separate redaktionelle Operation. Das hält die Bedeutung einer öffentlichen URL stabil.

Die Veröffentlichungsmethode in EditorialPostsResource.scala ist:

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

Die Ressource importiert POST, Path und PathParam von jakarta.ws.rs und Instant von java.time. EntityGraph und die anderen Anwendungstypen teilen sich ihr Paket.

Der Load-Helfer analysiert die UUID, findet die Entität und prüft den Lesezugriff. Für einen Befehl erwirbt er eine `PESSIMISTIC_WRITE`-Sperre. Wir bewerten den Übergang nach dieser Sperre gegen die aktuelle Zeile. Zwei Administratoren, die den gleichen Entwurf veröffentlichen, erhalten gleichzeitig eine erfolgreiche Veröffentlichung und einen 409-Konflikt.

Die BlogPost-Methode besitzt weiterhin die Zustandsänderung: Bei `publish` setzt sie `status` und `publishedAt` gemeinsam. `retract` entfernt den Zeitstempel. Die bestehende Validierung und Datenbankprüfung bleibt bestehen.

Diese Befehle akzeptieren keine bearbeitbaren Entity-Felder. Sie speichern keine vom Client übermittelte Kopie des Artikels. Allgemeine Updates und eingereichte Versionskonflikte gehören zu PreparedChange im nächsten Kapitel.

Die Request-Transaktion umfasst immer noch die Antwort-Serialisierung. Ein Schreiberfehler oder ein fehlgeschlagener Commit rollt den Übergang zurück. Bei Erfolg enthält die zurückgegebene Entität die inkrementierte Version und ihren neuen Veröffentlichungszustand.

## Geben Sie die Mapper-Feldregeln an

Ein Feld kann gelesen werden, ohne beschreibbar zu sein. Ein Administrator kann einen Titel bearbeiten, aber der Veröffentlichungsstatus sollte sich durch den Veröffentlichungsbefehl ändern.

`BlogPost.Schema` verwendet daher zwei explizite Regeln. Die Definitionen bleiben in der bestehenden Schema-Klasse:

```scala
    val title: SingularProperty[BlogPost, String] = reference(_.title, classOf[PostEditRule])
    val content: SingularProperty[BlogPost, String] = reference(_.content, classOf[PostEditRule])
    val status: SingularProperty[BlogPost, BlogPostStatus] = reference(_.status, classOf[PostReadRule])
    val publishedAt: SingularProperty[BlogPost, Instant] = reference(_.publishedAt, classOf[PostReadRule])
```

SingularProperty wird von com.anjunar.json.mapper.schema.property importiert; Instant kommt von java.time. Slug und Summary verwenden auch PostEditRule. ID und Version behalten die Standard-Read-only-Regel bei.

Beide Regeln erlauben das Lesen, wenn PostAccess.canRead wahr ist. PostEditRule erlaubt Schreiben nur, wenn canEdit wahr ist. PostReadRule lehnt Mapper-Schreiben immer ab.

Wir verwenden `reference`, weil es persistente Singularattribute sind. Ihre Regeln machen sie nicht zu einem separaten, handgeschriebenen Kriterien-Metamodell.

Das zwischengespeicherte Schema enthält Regelklassen. Die vorhandene Rule-Factory des Mappers ruft `RuntimeContext` auf. Die Bean löst ihre injizierte `PostAccess`-Instanz über CDI auf. Es speichert nicht den aktuellen Benutzer oder ein erlaubtes Flag im Schema.

Es gibt noch keinen generischen Edit-Endpunkt. Eine Test-Only-Ressource führt den Mapper für einen unveröffentlichten Artikel unter echten anonymen Leser- und Administratoranforderungen aus. Nur der Administrator kann seinen Titel ändern. Keiner kann Version, Status oder publishedAt über den Mapper schreiben. Dies überprüft die Regel, bevor wir uns für die allgemeine Bearbeitung darauf verlassen.

Schema.forGraph bleibt strukturelle Metadaten: welche Felder zur ausgewählten Antwort gehören und deren Typen. Wir haben es nicht in einen Schreibfeldkatalog verwandelt.

## Beschreiben Sie die nächsten Aktionen mit Links

Ein Administrator, der einen nicht leeren Entwurf betrachtet, kann ihn veröffentlichen. Nach der Veröffentlichung wird die nützliche Aktion zurückgezogen. Wir drücken diesen Unterschied in der Antwort aus.

`Data`, `Table` und `SessionState` besitzen nun eine `$links`-Sammlung. Die Links gehören zu den Antwortumschlägen; es wird kein neues JPA-Feld oder eine neue Datenbankspalte benötigt. Der vorhandene Link-Typ des JSON-Mappers liefert rel, url, method und seinen @type-Wert.

Ein verkürzter Entwurf der Antwort sieht so aus:

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

Die echte Detailantwort enthält auch die ausgewählten Entity-Felder und Schema-Metadaten. Dieser Auszug behält nur die Teile, die benötigt werden, um der Aktion zu folgen.

Die Relation sagt, was der Link bedeutet; die URL identifiziert ihr Ziel. Das ist das grundlegende Linkmodell, das von [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288.html) beschrieben wird. Unsere $links JSON-Darstellung und Aktionsnamen sind ein Anwendungsvertrag, kein von diesem RFC standardisiertes JSON-Format.

`PostLinks.scala` fügt die Befehle abhängig vom Zustand hinzu:

```scala
    if (endpoint("publish") && access.canPublish(post))
      links.add(new Link("publish", s"$path/publish", "POST", "BlogPost"))
    if (endpoint("retract") && access.canRetract(post))
      links.add(new Link("retract", s"$path/retract", "POST", "BlogPost"))
```

Dieser Auszug gehört in editorialPost. Der Link wird von com.anjunar.json.mapper.schema importiert. Der Endpoint-Helfer liest die gleiche EndpointPolicy, die von AuthorizationFilter verwendet wird, und Access ist derselbe PostAccess, der vom Befehl verwendet wird. Der Code behält keine weitere Kopie der Rollenregeln bei.

Ein veröffentlichter Beitrag erhält auch einen öffentlichen GET-Link. Die redaktionelle Sammlung enthält selbst und gegebenenfalls vorherige und nächste Seitenlinks. Die Sitzungsantwort eines Administrators enthält den Einstieg in den Redaktionsbereich; anonyme Sitzungen und Lesersitzungen nicht.

Jeder Aufruf baut neue Links auf. Da sogar ein öffentlicher Postumschlag einen nur für Administratoren bestimmten Vorschau-Link enthalten kann, verwenden API-Antworten Cache-Control: no-store. Andernfalls könnte ein Antwort-Cache die Aktionen eines Anrufers an einen anderen weiterleiten.

## Lassen Sie den Browser der Antwort folgen

Die Scala.js-Umschläge weisen den JSON-Namen $links einem Link-Mitglied zu. Eine ausgelassene Sammlung behält ihren leeren Standard. BlogPost selbst spiegelt weiterhin die Entitätsfelder wider.

Die Kontoseite zeigt „Open editorial“, wenn die Sitzung die redaktionelle Beziehung enthält. Die Detailseite zeigt „Publish“ oder „Retract“ an, wenn die aktuelle Antwort diese Aktion enthält. Sie leitet diese Schaltflächen nicht aus einem `ADMIN`-String ab.

In EditorialService.scala folgt die Ausführung dem bereitgestellten Link:

```scala
  def execute(link: ApiLink): Future[BlogPostData] = {
    // Validate before fetching CSRF or sending a command.
    val path = link.path("POST")
    accounts.session().flatMap(state =>
      HttpJson.post[BlogPostData](path, js.Dynamic.literal(), state.csrfToken)).map(validate)
  }
```

`Future` wird aus `scala.concurrent` importiert, `js` aus `scala.scalajs`. Der Dienst verfügt bereits über einen AccountService und einen ExecutionContext.

ApiLink.path prüft die erwartete HTTP-Methode und löst die URL mit dem aktuellen Ursprung auf. Es akzeptiert nur ein Ziel unter `/service/` mit gleichem Ursprung ohne Anmeldeinformationen oder ein Fragment. Ein externer Aktionslink darf niemals das CSRF-Token unserer Sitzung erhalten.

EditorialActions ersetzt die aktuelle Detailantwort nach Erfolg. Das aktualisiert die Entity-Werte, Version und verfügbare Aktionen zusammen. Während ein Befehl aussteht, sind die Buttons deaktiviert. Wenn die Komponente entsorgt wurde, kann ihre späte Antwort die Seite nicht neu zeichnen.

Der UI-Baum bleibt in EditorialPostPage.compose zusammen. Sein Publish-Steuerelement verwendet die reaktive Linkliste:

```scala
        when(actions.current.map(_.links.exists(_.rel == "publish"))) {
          button(i18n"Publish") { buttonType("button"); disabled = actions.busy; onClick(_ => actions.run("publish")) }
        }
```

Die Imports der Komponente stellen `when`, die Button-DSL, `EventDsl.onClick` und das `i18n`-Makro bereit. Dies ist Teil seines bestehenden Renderblocks, kein separater Rendering-Helfer.

Die API entscheidet, für welche Operationen sie wirbt. Die Anwendung definiert weiterhin ihre Seitenrouten und wie eine Publikationsaktion zu präsentieren ist; HATEOAS generiert nicht automatisch eine ganze Oberfläche.

## Behandeln Sie einen Link als aktuelles Angebot

Ein Link kann zwischen dem Laden der Seite und dem Klicken auf sie veraltet sein.

Ein anderer Administrator kann den Artikel veröffentlichen. Eine betreibende Person kann diesem Konto die Administratorrolle entziehen. Die Sitzung kann enden. Jemand kann auch eine bekannte URL anrufen, ohne jemals einen Link zu erhalten.

Jede Anfrage wiederholt daher Endpunkt-, CSRF- und Objektzustandsprüfungen. Es gibt kein Link-Token, das diese Prüfungen umgeht.

Der Browser behandelt Fehler absichtlich:

- 401 fordert den Benutzer auf, sich erneut anzumelden.
- 403 erklärt, dass die Aktion nicht mehr erlaubt ist.
- 409 fordert den Benutzer auf, den geänderten Beitrag neu zu laden.
- Ein Netzwerk- oder Serverfehler besagt, dass das Ergebnis nicht bestätigt werden konnte.

Nachdem eine Aktion fehlgeschlagen ist, werden die alten Aktionslinks verworfen und ein Reload angeboten. Es wiederholt nicht automatisch eine POST: Die Antwort ist möglicherweise nach einem erfolgreichen Commit verloren gegangen.

## Überprüfen Sie den vollständigen Pfad

Erstellen Sie die Anwendung und führen Sie die vorhandenen Suiten mit den isolierten Datenbank- und SMTP-Erfassungseinstellungen aus Kapitel 12 aus:

```text
sbt --server "application-backend/testFull" "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
```

Dieser Checkpoint besteht 89 Backend-Tests, 9 Scala.js-Modelltests und 28 Browserverträge. Die Berechtigungstests umfassen echte Anruferrollen, widerrufenen Zugriff, Feldregeln, CSRF, ungültigen Zustand, gleichzeitige Veröffentlichung und Rollback. Die Browserverträge prüfen Aktions-URLs und -Methoden, versteckte Aktionen, Konfliktwiederherstellung, Klartextinhalte und Ablehnung externer Links.

Es gibt auch einen echten Publikationsworkflow:

```text
npx playwright test --project=editorial
```

Es erfordert die dedizierten Testadministratorvariablen aus Kapitel 11 und psql auf PATH oder `BLOG_PSQL`, das auf seine ausführbare Datei zeigt. Es fügt seinen eigenen einzigartigen Entwurf in die konfigurierte Testdatenbank ein, meldet sich über Soteria an, veröffentlicht über die Benutzeroberfläche, lädt neu, zieht zurück, überprüft die öffentliche Sichtbarkeit und entfernt den angelegten Entwurf.

Für eine manuelle Überprüfung folgen Sie den gleichen Schritten mit unserem privaten Entwurf. Überprüfen Sie die Detailantwort im Netzwerkpanel des Browsers vor und nach der Veröffentlichung. Die wechselnden $links sollten mit den verfügbaren Buttons übereinstimmen, während ein direkter Anruf von einer nicht autorisierten Sitzung immer noch fehlschlägt.

Wir können nun eine Operation vom Endpunkt bis zum Post erzwingen und dem Browser den verfügbaren nächsten Schritt erklären. Das nächste Kapitel nutzt diese Grundlage, um allgemeine Entity-Änderungen mit `PreparedChange` anzuwenden.