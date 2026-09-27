# Änderungen sicher anwenden

Im Redaktionsbereich lässt sich ein Artikel inzwischen veröffentlichen. Jetzt soll dort auch der Artikel selbst gespeichert werden können.

Dazu reicht es nicht, JSON-Felder einer Entität zuzuweisen. Wir müssen den Zugriff prüfen, bevor sich die verwaltete Entität ändert, Bearbeitungen auf Basis einer alten Version ablehnen und die ganze Anfrage zurückrollen, wenn ein übermittelter Wert ungültig ist.

In diesem Kapitel ergänzen wir das Anlegen von Entwürfen und teilweise Änderungen mit `PreparedChange`. Wir verfolgen den gesamten Schreibvorgang: Eingabe vorbereiten, ursprüngliche Entität prüfen, Vorgang autorisieren, Änderungen über den Mapper anwenden und validieren sowie das festgeschriebene Ergebnis zurückgeben.

## Ausgangspunkt: Kapitel 13

Verwenden Sie den abgeschlossenen Stand des Berechtigungskapitels:

```text
git switch --detach 37f2a3e6a7d440be4bbc99730bc430915f1bdaa4
```

Der vollständige Quellcode für dieses Kapitel liegt hier:

```text
git switch --detach bb212a133dc20f9647ead13d5347be42e5829592
```

Die [begleitende Anleitung](https://github.com/anjunar/anjunar-blog-example/blob/bb212a133dc20f9647ead13d5347be42e5829592/docs/applying-changes-safely.md) beschreibt Einrichtung und Anfragevertrag im Detail. Verwenden Sie weiterhin die Entwicklungsdatenbank, den Administrator und die lokalen Cookie-Einstellungen aus dem vorherigen Kapitel. Es gibt keine neuen Datenbankobjekte; bei einer bestehenden Datenbank meldet die Migration `AlreadyApplied` und führt keine SQL-Anweisungen aus.

Bauen Sie `frontendAssets` und starten Sie `application-backend/run`. Melden Sie sich unter `/en/account` an. Das Bearbeitungsformular folgt im nächsten Kapitel. Hier machen die API und ein ausführbares Beispiel für die Browserkonsole den Schreibpfad sichtbar.

## Die Entität bleibt das Modell

Die Eingabe verwendet dieselben Feldnamen wie `BlogPost`:

```json
{
  "slug": "our-next-post",
  "title": "Our next post",
  "content": "A small draft with a complete write path.",
  "summary": "From a prepared change to a committed response."
}
```

Wir führen kein weiteres Objekt mit Kopien dieser Felder und einer Kette manueller Zuweisungen an die Entität ein. `BlogPost` definiert die Daten, sein `EntitySchema` die Mapper-Regeln und Bean Validation die Feldbeschränkungen.

Die Redaktionsressource erhält zwei neue Operationen:

| Methode | Pfad | Ergebnis |
| --- | --- | --- |
| POST | `/service/editorial/posts` | Einen Entwurf anlegen; 201, `Location` und die Detailhülle zurückgeben. |
| PATCH | `/service/editorial/posts/{id}` | Die übermittelten Felder anwenden und die aktualisierte Detailhülle zurückgeben. |

Beide verlangen `ADMIN`, CSRF-Schutz und `application/json`. Der Body einer PATCH-Anfrage enthält die Version, die der Aufrufer geladen hat. Zum Beispiel:

```json
{
  "version": 0,
  "title": "A clearer title"
}
```

Ausgelassene Felder bleiben unverändert. Ein übermitteltes `null` entfernt die optionale Zusammenfassung, kann aber weder den erforderlichen Titel noch den Inhalt entfernen. Das ist das Format unserer API für teilweise Entity-Änderungen, kein JSON Patch und kein JSON Merge Patch. PATCH ist die HTTP-Methode für teilweise Änderungen; die Anfrage muss als Ganzes erfolgreich sein oder fehlschlagen. Siehe [RFC 5789](https://www.rfc-editor.org/rfc/rfc5789.html).

## Vor dem Anwenden vorbereiten

Die veröffentlichte Bibliothek `json-mapper` stellt `PreparedChange` bereits bereit. `JsonMapper.prepare` hält die Eingabe und die Zielentität fest, ohne die übermittelten Werte schon zuzuweisen.

Die beiden zentralen Methoden haben unterschiedliche Aufgaben:

- `getEntity` liefert das ursprüngliche Ziel für Berechtigungs- und fachliche Prüfungen.
- `applyChanges` führt Binding, Schema-Regeln und Feldvalidierung für dieses Ziel aus.

In `PreparedChanges.scala` sieht der eigentliche Vorbereitungsaufruf so aus:

```scala
    JsonMapper.prepare(json, post, TypeResolver.resolve(classOf[BlogPost]),
      manager.getEntityGraph("BlogPost.detail"), noReferences,
      [T] => (clazz: Class[T]) => RuntimeContext.bean(clazz), validator)
```

Dieser Aufruf gehört in die Request-Scoped Bean `PreparedChanges`. `JsonMapper` stammt aus `com.anjunar.json.mapper`, `TypeResolver` aus `com.anjunar.scala.universe`. Die Bean injiziert `EntityManager` und `Validator`. `RuntimeContext` löst Feldregeln über CDI auf, wie schon beim Lesen in Kapitel 13.

Der Graph wählt den Vertrag für Artikel aus. Die Regeln bestimmen, ob ein Feld schreibbar ist. Dass `status` im Graphen ausgewählt ist, erlaubt noch keine Zuweisung.

## Mit REST verbinden

Anlegen und Bearbeiten beginnen mit unterschiedlichen Entitäten.

Bei POST liest `PreparedChangeReader` den JSON-Body und bereitet einen neuen `BlogPost` vor. Dessen Standardwerte haben eine Bedeutung: Ein neuer Artikel ist ein Entwurf, hat noch keinen Veröffentlichungszeitpunkt und darf einen leeren Inhalt besitzen.

Bei PATCH löst ein `ParamConverter` den URL-Parameter in eine vorbereitete Änderung der vorhandenen, verwalteten Entität auf. Der Controller kann deshalb Folgendes deklarieren:

```scala
  @PATCH @Path("/{id}")
  @Consumes(Array(MediaType.APPLICATION_JSON))
  @EntityGraph("BlogPost.detail")
  def update(@PathParam("id") change: PreparedChange[BlogPost]): Data[BlogPost] = {
    if (!access.canEdit(change.getEntity())) throw new ForbiddenException()
    val post = change.applyChanges()
    requireFreeSlug(post)
    manager.flush()
    result(post)
  }
```

Diese Methode steht in `EditorialPostsResource.scala`. `PreparedChange` wird aus `com.anjunar.json.mapper` importiert. `PATCH`, `Path`, `Consumes`, `PathParam` und `ForbiddenException` stammen aus `jakarta.ws.rs`; `MediaType` aus `jakarta.ws.rs.core`.

Die erste Zeile der Methode prüft die ursprüngliche Entität. Erst die nächste Zeile wendet eingehende Werte über den Mapper an und validiert sie. Reader und Converter treffen diese Autorisierungsentscheidung nicht für den Controller.

Die Provider unterstützen an diesem Stand bewusst `PreparedChange[BlogPost]`. Der Client kann keinen Entity-Klassennamen bestimmen. Ein optionaler `@type`-Wert muss `BlogPost` sein; eine optionale ID bei einem Update muss zur URL passen. Unbekannte Felder werden gemeldet, statt Tippfehler stillschweigend zu übergehen.

## Die Version verpflichtend machen

Eine Datenbanktransaktion verrät uns nicht, welche Version der Benutzer gesehen hat.

Angenommen, zwei Personen laden Version 0. Die erste speichert einen neuen Titel und erzeugt Version 1. Die zweite darf diese Änderung nicht mit ihrer veralteten Kopie überschreiben.

`EntityVersions.scala` prüft die übermittelte Version:

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

`JsonNumber` und `JsonObject` werden aus `intermediate.model` des Mappers importiert. Die Methode gehört zum Objekt `EntityVersions`; `conflictDetail` erklärt dem Benutzer, dass er vor einem neuen Speicherversuch die Daten neu laden muss.

Die Zahl 0 ist eine gültige Anfangsversion. Fehlende, `null`-, als String übermittelte, gebrochene oder überlaufende Versionswerte führen zu einem Feldfehler mit Status 400. Eine gültige Version, die von der gespeicherten abweicht, führt zu 409. Die übermittelte Zahl weisen wir niemals dem `@Version`-Feld der Entität zu.

Es gibt außerdem ein Konkurrenzfenster: Wenn zwei Anfragen ihre Version vergleichen und anschließend ohne Koordination schreiben, können beide denselben Vergleich bestehen.

`PreparedChanges.update` lädt die Zeile deshalb vor der Prüfung mit einer Schreibsperre:

```scala
    val post = manager.find(classOf[BlogPost], uuid, LockModeType.PESSIMISTIC_WRITE)
    if (post == null || !access.canRead(post)) throw new NotFoundException()
    EntityVersions.requireCurrent(post, json)
    prepare(json, post)
```

`LockModeType` stammt aus `jakarta.persistence`, `NotFoundException` aus `jakarta.ws.rs`. Der vorangehende Code liest die UUID aus der URL.

Der zweite Schreibvorgang wartet und vergleicht seine Version danach mit der aktuellen Zeile. Unser Test mit parallelen Anfragen erhält einmal 200 und einmal 409. Hibernates `@Version` bleibt Teil des Entity-Vertrags; zusätzlich verlangen wir die Version ausdrücklich an der HTTP-Grenze. Die pessimistische Sperre hält die Datenbanksperre für die Dauer der Transaktion, wie in [Jakarta Persistence](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2) beschrieben.

Eine Änderung ohne Wirkung erzwingt keine neue Version. Eine tatsächliche Bearbeitung gibt Hibernates nächste Version zurück. Auch eine Veröffentlichung erhöht sie; eine Bearbeitung auf Basis der Version vor der Veröffentlichung ist deshalb veraltet.

## Eingaben durch den JSON-Mapper validieren

Die Validierung ist bereits Teil von `applyChanges`. Der Mapper ruft für übermittelte Eigenschaften `Validator.validateValue` auf, bevor er gültige Werte zuweist. Verstöße sammelt er mit Feldpfaden in `ErrorRequestException`. Der Controller ruft `validate` nicht noch einmal auf.

`PreparedChanges` injiziert einen `Validator` und übergibt ihn an `JsonMapper.prepare`. Die folgende `ValidationProducer.scala` verwaltet die Factory und stellt den Validator über CDI bereit:

```scala
package com.anjunar.blog

import jakarta.annotation.{PostConstruct, PreDestroy}
import jakarta.enterprise.context.ApplicationScoped
import jakarta.enterprise.inject.Produces
import jakarta.validation.{Validation, Validator, ValidatorFactory}

@ApplicationScoped
class ValidationProducer {
  private var factory: ValidatorFactory = null
  @PostConstruct def initialize(): Unit = factory = Validation.buildDefaultValidatorFactory()

  @Produces def validator: Validator = factory.getValidator

  @PreDestroy def close(): Unit = if (factory != null) factory.close()
}
```

Diese Bean liefert die Abhängigkeit des Mappers und schließt die Factory beim Herunterfahren. Sie enthält keine artikelspezifische Validierungslogik.

Der Mapper prüft übermittelte Werte. Hibernate behält außerdem die im Persistenzkapitel konfigurierte `CALLBACK`-Validierung bei: Vor einem Insert oder Update prüft es die vollständige Entität. So fallen ein beim Anlegen ausgelassener Pflicht-Titel und ein leerer Inhalt bei einem veröffentlichten Artikel auf. Die daraus entstehenden `ConstraintViolationException`-Fehler erreichen denselben Mapper für Problemantworten. Die bestehende Request-Transaktion rollt bei beiden Arten von Validierungsfehlern zurück.

Die Feldregeln aus Kapitel 13 gelten weiterhin. Administratoren dürfen Titel, Slug, Inhalt und Zusammenfassung schreiben. `status` und `publishedAt` bleiben für den Mapper schreibgeschützt; für diese Übergänge verwenden Aufrufer `publish` und `retract`. Schreibgeschützte Werte, die in einer Entity-Darstellung zurückgesendet werden, werden ignoriert. ID und Version unterliegen zusätzlich den oben beschriebenen Identitäts- und Vorbedingungsprüfungen.

## Eindeutigkeit prüfen, ohne zu früh zu flushen

Ein hilfreicher Slug-Fehler nennt das betroffene Feld. Der Controller prüft zuerst, ob bereits ein anderer Artikel den gewünschten Slug verwendet.

Die Abfrage nutzt die typisierten Criteria-Attribute des Schemas. Außerdem setzt sie ausdrücklich den Flush-Modus:

```scala
    if (manager.createQuery(query).setFlushMode(FlushModeType.COMMIT).getSingleResult.longValue() > 0)
      throw new ApiProblem(409, "This slug is already used by another post.", Problem.conflict,
        Seq(new ErrorRequest(util.List.of[Any]("slug"), "Choose an unused slug.")))
```

Das ist der letzte Teil von `requireFreeSlug` in `EditorialPostsResource`. `FlushModeType` stammt aus `jakarta.persistence`, `ErrorRequest` aus `com.anjunar.json.mapper` und `util` aus `java.util`.

Ohne diese Einstellung könnte Hibernate die ausstehende Änderung vor der Abfrage flushen. Wir wollen erst die Prüfung abschließen und danach gezielt flushen.

Die Vorprüfung ersetzt die Datenbankbeschränkung nicht. Zwei neue Artikel können gleichzeitig denselben noch freien Slug beanspruchen. Die Unique Constraint entscheidet, welcher Schreibvorgang gewinnt; der Fehlermapper macht aus dem anderen einen 409-Fehler für das Slug-Feld. Er erkennt sowohl den übernommenen Namen `blog_post_slug_key` als auch `uq_blog_post_slug` aus dem Mapping.

## Fehler atomar behandeln

`PreparedChange` bindet Werte zeitversetzt. Es ist weder ein unveränderliches Diff noch eine eigenständige Transaktion.

Während `applyChanges` läuft, kann ein gültiger Titel bereits zugewiesen sein, bevor eine ungültige Zusammenfassung auffällt. Die verwaltete Entität hat sich dann im Speicher geändert. Die Ausnahme muss nach außen gelangen, damit die ganze Anfrage zurückgerollt wird.

Die vorhandene `TransactionBoundary` steuert weiterhin die Lebensdauer. Der Controller ruft den Mapper auf und flusht. Der Writer serialisiert die Antwort in einen Puffer. Erst nach dem Commit wird der erfolgreiche Antwortinhalt gesendet. Wenn Validierung, Serialisierung oder Commit fehlschlagen, werden die Änderungen zurückgerollt.

Ein Flush ist kein Commit. Der explizite Flush im Controller liefert die aktualisierte Version und macht Datenbankbeschränkungen sichtbar; der endgültige Commit bleibt Aufgabe der `TransactionBoundary`.

Die Tests erzeugen absichtlich Fehler, nachdem eine gültige Änderung schon angewendet wurde. Sowohl Writer- als auch Commit-Fehler lassen den ursprünglichen Titel und die ursprüngliche Version in der Datenbank stehen. Ein weiterer Test weist eine vorbereitete Anfrage vor dem Anwenden zurück und prüft, dass der Controller noch den ursprünglichen Titel sieht.

Nach erfolgreichem Anwenden ist ein zweiter Aufruf von `applyChanges` ein Fehler. Auch einen fehlgeschlagenen Aufruf dürfen wir nicht mit demselben Objekt wiederholen: Rollen Sie die Anfrage zurück und beginnen Sie mit einer neuen.

## An der HTTP-Grenze streng prüfen

Ein eingehender Body ist auch bei einem authentifizierten Aufrufer nicht vertrauenswürdig.

`RequestJson` akzeptiert höchstens 1 MiB und gültiges UTF-8, weist doppelte Objektschlüssel zurück und begrenzt die Verschachtelung auf 32 Ebenen. Jackson Core parst Tokens als Stream. Es war bereits eine Abhängigkeit von `json-mapper` und wird nun direkt deklariert, weil die Anwendung es selbst verwendet.

Der Parser erstellt die Zwischenknoten des JSON-Mappers. Er bindet kein zweites Domänenmodell und weist keine Entity-Felder zu.

`PreparedChanges` prüft mithilfe des JPA-Metamodells auch die JSON-Typen persistenter String-Attribute. Ein Objekt anstelle eines String-Titels ist ein Feldfehler. Es darf weder als leere Bean behandelt werden noch den Titel stillschweigend unverändert lassen. JSON-`null` bleibt von einer fehlenden Eigenschaft unterscheidbar.

Die Eingabe-Helfer sind auf den derzeit veröffentlichten Vertrag zugeschnitten. `BlogPost` besitzt noch keine Beziehungen zu Autoren oder Medien. Der an den Mapper übergebene `EntityLoader` weist das Laden von Referenzen zurück. Wenn Kapitel 17 Beziehungen einführt, müssen wir jedes referenzierte Ziel laden und autorisieren; uneingeschränktes Laden nach ID genügt dann nicht.

## Fehler zurückgeben, die ein Formular nutzen kann

Ein bloßer Status 400 teilt dem Browser mit, dass etwas fehlgeschlagen ist. Er sagt nicht, welches Feld korrigiert werden muss.

Die API liefert nun `application/problem+json` mit einem stabilen Problemtyp, HTTP-Status, lesbaren Details und Anfragepfad. Feldfehler stehen in einer `errors`-Erweiterung:

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

Das folgt dem Problem-Details-Format aus [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html). `errors` und seine Struktur aus Pfad und Nachricht sind eine Erweiterung unserer Anwendung. Validierungsmeldungen können je nach Constraint und Locale des Validators variieren.

Erwartete Eingabefehler geben weder Datenbanktreibertext noch Stack Traces zurück. Unerwartete Fehler liefern eine `errorId`, mit der sich der Serverlogeintrag finden lässt. Authentication Challenges und Header für Ratenbegrenzungen bleiben erhalten, wenn vorhandene HTTP-Fehler in Problemantworten umgewandelt werden.

Auf Scala.js behält `HttpFailure` nun ein optionales `ProblemDetails`-Objekt mit typisierten Feldfehlern. Wenn der Body fehlt, fehlerhaft oder widersprüchlich ist, bleibt der tatsächliche HTTP-Status maßgeblich. Bestehende Seiten verwenden weiter statusbasierte Meldungen; Kapitel 15 kann die Feldfehler an Eingaben binden.

## Den Zustand für den nächsten Client-Schritt zurückgeben

Ein erfolgreiches Anlegen liefert Status 201, `Location` und `Data[BlogPost]`. ID und Anfangsversion stammen aus der Persistenz. Ein erfolgreiches Update liefert die aktuelle Detailhülle mit Version, strukturellem Schema und frischen `$links`.

Die Collection bietet nun eine Create-Aktion an, ein bearbeitbarer Artikel eine Update-Aktion mit PATCH. Die spätere Speicheraktion im Browser kann diesen Links folgen, statt Befehls-URLs selbst zusammenzusetzen.

Ein bestehendes Mapper-Verhalten müssen wir berücksichtigen: Leere Strings werden in Antworten ausgelassen. Ein neu angelegter Entwurf mit leerem Inhalt ist dennoch gültig. `EditorialService` ergänzt den ausgelassenen Inhalt in seinem Detailmodell wieder als leeren String. Ein veröffentlichter Artikel benötigt weiterhin Inhalt. So bleibt der Unterschied zwischen einer kompakten öffentlichen Liste und einem bearbeitbaren Entwurf erhalten.

## Das konkrete Beispiel ausführen

Im Repository liegt [docs/examples/post-changes.js](https://github.com/anjunar/anjunar-blog-example/blob/bb212a133dc20f9647ead13d5347be42e5829592/docs/examples/post-changes.js).

Melden Sie sich als Entwicklungsadministrator an und fügen Sie die Datei in die Browserkonsole ein. Sie lädt die Sitzung, folgt dem Einstieg in den Redaktionsbereich, legt über dessen Create-Link einen Entwurf an und aktualisiert ihn über dessen Update-Link.

Das wesentliche Update verwendet die zurückgegebene Version:

```javascript
  const savedResponse = await send(update, {
    version: created.data.version,
    title: "A safely updated post",
    summary: "Prepared, authorized, applied and validated."
  });
```

Die Datei definiert `send`, übermittelt das CSRF-Token und prüft das Ziel. Danach wiederholt sie eine Änderung mit der alten Version und erwartet 409. Das zurückgegebene Objekt enthält die generierte ID, die Versionen 0 und 1, den Konfliktstatus und eine Vorschauadresse. Ein Entwurf bleibt zur eigenen Prüfung erhalten.

Dieselbe Datei läuft auch im echten Browsertest. Der Test entfernt den von ihm angelegten Artikel anschließend.

## Den Schreibvertrag prüfen

Verwenden Sie die isolierte Testdatenbank und die lokalen SMTP-Capture-Einstellungen aus Kapitel 12:

```text
sbt --server "application-backend/testFull" "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
npx playwright test --project=changes
```

Das Browserprojekt für Änderungen benötigt außerdem den eigenen Testadministrator und `psql` in `PATH` oder einen darauf zeigenden Wert in `BLOG_PSQL`. Die begleitende Anleitung nennt die Variablen.

Der Kapitelstand besteht 112 Backend-Tests, 12 Scala.js-Modelltests und 29 Browser-Contract-Tests. Der echte Änderungsablauf prüft das Artikelbeispiel mit Soteria, REST und PostgreSQL. Alle sechs Browserprojekte enthalten zusammen 35 Tests.

Wir haben nun einen Schreibvertrag, der Entwürfe anlegt, nur erlaubte Felder ändert, veraltete Bearbeitungen zurückweist und fehlgeschlagene Anfragen zurückrollt. Im nächsten Kapitel erhält er ein richtiges Formular mit direkter Modellbindung, Feldfehlern und Zustandsverwaltung beim Speichern.
