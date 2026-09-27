Wir können einen Beitrag speichern und seine Datenbanktabelle weiterentwickeln. Die nächste Frage ist, wie der Rest der Anwendung diesen Beitrag beschreiben sollte.

Der JSON-Mapper muss wissen, welche Felder verfügbar sind und welche Regeln für sie gelten. Eine Kriterienabfrage benötigt typisierte Attribute wie `slug` und `status`. Wir werden diese Felder einmal in `BlogPost.Schema` beschreiben und für beide Aufgaben verwenden.

Am Ende des Kapitels findet eine echte PostgreSQL-Abfrage einen veröffentlichten Artikel anhand seines Slugs. Tests prüfen außerdem den JSON-Feldvertrag und bestätigen, dass eingehendes JSON keine Felder ändern kann, deren Schema-Regeln Schreibzugriff verweigern.

Beginnen Sie mit dem [Kapitel 6 Quelle](https://github.com/anjunar/anjunar-blog-example/tree/185a0fd7634f1da3e7f7b420a806a022033cd242) und einer Datenbank, die bereits auf diese Version migriert wurde. Alle Pfade unten sind relativ zum Projektverzeichnis.

## Halten Sie die verschiedenen Schemata getrennt

Wir verwenden bereits `@SchemaId`, aber es beantwortet eine andere Frage:

|Mechanismus|Verantwortung|
| --- | --- |
|JPA-Annotationen|Ordnen die Entität Tabellen, Spalten und Beziehungen zu.|
|Bean Validation|Prüfen Sie, ob Werte und Publikationszustand gültig sind.|
|`@SchemaId`|Halten Sie die Identität eines Schemaelements über Datenbankmigrationen hinweg stabil.|
|`EntitySchema`|Beschreiben Sie Felder, Mapperregeln und typisierte JPA-Attribute.|

Durch Hinzufügen von `EntitySchema` wird keine Datenbankspalte erstellt. Das Hinzufügen einer JPA-Spalte fügt nicht automatisch den Eintrag zum Mapper-Schema hinzu. Wir pflegen diese Beschreibungen zusammen.

## Hinzufügen der Mapper-Abhängigkeit

Fügen Sie diesen Eintrag zur bestehenden `libraryDependencies`-Sequenz in `build.sbt` hinzu:

```scala
"com.anjunar" %% "json-mapper" % "1.1.5",
```

Fügen Sie auch diese Build-Einstellung neben `ThisBuild / scalaVersion` hinzu:

```scala
ThisBuild / externalResolvers := Seq(Resolver.mavenCentral)
```

sbt enthält normalerweise das lokale Ivy-Repository, wenn Anwendungsabhängigkeiten gelöst werden. Die explizite Auswahl von Central verhindert, dass ein lokal veröffentlichter Framework-Build die von den Lesern verwendete Version überschattet. Das [sbt Abhängigkeit Dokumentation](https://www.scala-sbt.org/2.x/docs/en/reference/sbt-update.html) erklärt diese Resolver-Einstellungen.

Aktualisieren Sie die Auflösung eines bestehenden Checkouts einmal:

```text
sbt --server "application-backend/update"
```

Kein benachbartes Repository ist eine Build Dependency.

## Erhalten Sie den aktuellen EntityManager

Eine einfache Schema-Eigenschaft kann ein Feld ohne Datenbankverbindung beschreiben. Eine typisierte JPA-Eigenschaft benötigt auch das initialisierte Metamodell von Hibernate.

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/RuntimeContext.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.inject.spi.CDI
import jakarta.persistence.EntityManager

object RuntimeContext {
  def entityManager(): EntityManager =
    CDI.current().select(classOf[EntityManager]).get()
}
```

Dies löst den angeforderten EntityManager-Produzenten aus Kapitel 4 auf. Sein zugrunde liegender EntityManager gehört zum aktiven `RequestTransaction`.

Der erste Zugriff auf `BlogPost.schema` muss daher erfolgen, nachdem CDI und Hibernate bereit sind, innerhalb einer aktiven Anfragetransaktion. Initialisieren Sie es nicht in einem eifrigen Top-Level-Wert während des Bootstraps.

## Beschreiben Sie jedes Feld

Fügen Sie in `BlogPost.scala` diese Importe hinzu:

```scala
import com.anjunar.json.mapper.schema.{EntitySchema, SchemaProvider}
import com.anjunar.json.mapper.schema.property.SingularProperty
```

Die Datei importiert bereits `java.lang`, `Instant` und `UUID`. Fügen Sie dieses Begleitobjekt nach der Entity-Klasse hinzu:

```scala
object BlogPost extends SchemaProvider[BlogPost.Schema] {
  class Schema extends EntitySchema[BlogPost](RuntimeContext.entityManager()) {
    val id: SingularProperty[BlogPost, UUID] = reference(_.id)
    val version: SingularProperty[BlogPost, lang.Long] = reference(_.version)
    val slug: SingularProperty[BlogPost, String] = reference(_.slug)
    val title: SingularProperty[BlogPost, String] = reference(_.title)
    val content: SingularProperty[BlogPost, String] = reference(_.content)
    val status: SingularProperty[BlogPost, BlogPostStatus] = reference(_.status)
    val publishedAt: SingularProperty[BlogPost, Instant] = reference(_.publishedAt)
    val summary: SingularProperty[BlogPost, String] = reference(_.summary)
  }

}
```

Der Begleiter implementiert `SchemaProvider`. Der Mapper findet dieses Companion-Objekt und fragt nach seinem Schema. In Version 1.1.5 entdeckt der Anbieter die verschachtelte Klasse `Schema` und erzeugt sie erst bei Bedarf. Unser Schema besitzt einen öffentlichen parameterlosen Konstruktor, wie es diese Konvention erwartet.

Es gibt einen Eintrag für jedes persistente Feld, einschließlich `id`, `version` und dem optionalen `summary`. Das Auslassen eines Feldes aus diesem Schema kann es aus der Mapper-Ausgabe entfernen, auch wenn das Feld noch `@JsonbProperty` trägt.

Der Selektor `_.slug` wird vom Scala-Compiler überprüft. Wenn wir das Feld umbenennen, wird der alte Selektor nicht mehr kompiliert. Die Eigenschaft behält auch ihren Namen, den Werttyp und die Zugriffsregel bei.

## Warum diese Felder reference verwenden

Die Factory `reference` umfasst auch persistente skalare Attribute. Es ist nicht auf eine Beziehung mit einer anderen Entität beschränkt.

Für `slug` erhält die Factory das tatsächliche singuläre Attribut von Hibernate und gibt ein `SingularProperty[BlogPost, String]` zurück. Dieser Wert implementiert auch die `SingularAttribute`-Schnittstelle von JPA, sodass die Criteria API es direkt verwenden kann.

|Factory|Verwendung|
| --- | --- |
|`property`|Mapper-Metadaten für einen Wert, einschließlich berechneter oder `@Transient`-Ausgabe. Es handelt sich nicht um ein Attribut der JPA-Kriterien.|
|`reference`|Ein persistentes Singularattribut: ein Skalarfeld oder eine Einzelwertassoziation.|
|`set`|Eine persistente Sammlung, die als JPA-Set abgebildet ist.|
|`list`|Eine persistente Sammlung, die als JPA-Liste abgebildet ist.|

Behalten Sie den konkreten Rückgabetyp. Wenn Sie `slug` als einfaches `Property[BlogPost, String]` deklarieren, wird die JPA-Attributschnittstelle vor dem Compiler ausgeblendet.

Die Factory muss dem realen Mapping entsprechen. Ein berechnetes Feld wird nicht zu einer Datenbankspalte, weil wir `reference` aufrufen oder seinen Namen in einer Abfrage verwenden. Wir werden die Sammlungsfabriken verwenden, wenn das Modell tatsächlich Beziehungen gewinnt.

## Verwenden Sie das Schema in einer Kriterienabfrage

Hinzufügen von `EntityManager` zu den bestehenden JPA-Importen in `BlogPost.scala`. Fügen Sie diese Methode dann im Begleiter nach `class Schema` hinzu:

```scala
def findPublishedBySlug(slug: String)(using entityManager: EntityManager): Option[BlogPost] = {
    val builder = entityManager.getCriteriaBuilder
    val query = builder.createQuery(classOf[BlogPost])
    val post = query.from(classOf[BlogPost])
    query.select(post).where(
      builder.equal(post.get(schema.slug), builder.parameter(classOf[String], "slug")),
      builder.equal(post.get(schema.status), BlogPostStatus.PUBLISHED)
    )
    Option(entityManager.createQuery(query).setParameter("slug", slug).getSingleResultOrNull)
  }
```

Die Abfrage hat zwei Prädikate: den angeforderten Slug und den `PUBLISHED`-Status. Der Slug wird als Abfrageparameter übergeben. Die Unique Constraint für den Slug bedeutet, dass das Ergebnis entweder ein Post oder kein Post ist, dargestellt durch `Option`.

Die Schlüsselausdrücke lauten:

```scala
post.get(schema.slug)
post.get(schema.status)
```

Beide verwenden Attribute aus dem gleichen Schema, das der Mapper inspizieren wird. Ein separates handgeschriebenes JPA-Metamodell und doppelte Feldnamen als Strings sind nicht nötig.

Das Statusprädikat bleibt unerlässlich. Feldsichtbarkeitsregeln wählen nicht aus, welche Zeilen eine Abfrage zurückgeben darf. Ein Schema, dessen Felder lesbar sind, darf nicht mit der Erlaubnis verwechselt werden, jeden gespeicherten Entwurf öffentlich anzuzeigen.

## JSON-Felder deklarieren

Der Mapper entdeckt annotierte Mitglieder und konsultiert dann das Schema. Beide Teile müssen vorhanden sein.

Fügen Sie diesen Import hinzu:

```scala
import jakarta.json.bind.annotation.JsonbProperty
```

Fügen Sie `@JsonbProperty` zu jedem der acht persistenten Felder hinzu: `id`, `version`, `slug`, `title`, `content`, `status`, `publishedAt` und `summary`.

Zum Beispiel lautet der Titel jetzt:

```scala
@NotBlank
@Size(min = 3, max = 180)
@Column(nullable = false, length = 180)
@SchemaId("46fdb02a")
@JsonbProperty
var title: String = ""
```

Diese Annotationen befinden sich auf Klassenkörperfeldern, so dass sie kein explizites `@field`-Ziel benötigen. Behalten Sie die vorhandenen Persistenz- und Validierungsanmerkungen. Die Publikationskonsistenzmethode ist eine interne Validierungsprüfung und erhält kein `@JsonbProperty`.

## Aufbewahrung des Publikationszeitstempels

Unsere Entität verwendet `Instant`. Der in Version 1.1.5 integrierte temporale Serialisierer des Mappers behandelt diesen Typ nicht, daher liefern wir seine String-Darstellung explizit.

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/InstantConverter.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.converter.JacksonJsonConverter
import com.anjunar.scala.universe.ResolvedClass

import java.time.Instant

class InstantConverter extends JacksonJsonConverter {
  override def toJson(input: Any, resolvedClass: ResolvedClass): String =
    input.asInstanceOf[Instant].toString

  override def toJava(json: String, resolvedClass: ResolvedClass): Any =
    Instant.parse(json)
}
```

Die Converter-Annotation des Mappers erwartet eine `JacksonJsonConverter`-Unterklasse. Wir überschreiben seine beiden Konvertierungsmethoden, um einen ISO-8601-Zeitstempel zu verwenden. Der Mapper verpackt den zurückgegebenen Text als JSON-String; der Converter darf selbst keine JSON-Anführungszeichen hinzufügen.

Fügen Sie diesen Import zu `BlogPost.scala` hinzu:

```scala
import com.anjunar.json.mapper.annotations.UseConverter
```

Das vollständige Publikationszeitfeld wird:

```scala
@Column(name = "published_at")
@SchemaId("398bfd50")
@JsonbProperty
@UseConverter(classOf[InstantConverter])
var publishedAt: Instant = null
```

Ein Wert wie `2026-09-27T10:15:42.123456Z` behält seine Sekunden, Nachkommastellen und UTC-Bezeichnung bei. Die Datenbank verwendet weiterhin das Zeitstempel-Mapping aus Kapitel 5.

## Lesbar bedeutet nicht schreibbar

Wir haben keine benutzerdefinierte Regel an irgendeine Schemafabrik übergeben. Das wählt `DefaultRule` aus:

|Regelmethode|Ergebnis|
| --- | --- |
|`isVisible`|`true`|
|`isWriteable`|`false`|

Der Mapper kann diese Felder serialisieren. Während der Deserialisierung überspringt es eingehende Änderungen an ihnen. Dies ist unabhängig davon, ob Scala ein Feld als veränderliches `var` deklariert.

Zum Beispiel versucht der Test zu liefern:

```json
{
  "title": "Unauthorized title",
  "summary": "Unauthorized summary",
  "version": 999,
  "status": "PUBLISHED",
  "publishedAt": "2026-09-27T10:00:00Z"
}
```

Der gespeicherte Artikel bleibt unverändert. Dieses Ergebnis bedeutet, dass der Mapper geschützte Felder ignoriert; es bedeutet nicht, dass ein HTTP-Endpunkt einen Autorisierungsfehler zurückgegeben hat. Endpunkt-Autorisierung und absichtliche Mutationsbehandlung werden in ihren eigenen Kapiteln hinzugefügt.

Später kann ein benutzerdefiniertes `VisibilityRule` mit der aktuellen Entität und dem Aufrufer entscheiden. Halten Sie diese Entscheidungen in der Regel. `SchemaProvider.schema` wird zwischengespeichert, so dass das Speichern der aktuellen Berechtigungen eines Benutzers im Schema selbst den falschen Zustand für spätere Anforderungen beibehalten würde.

## Überprüfen Sie den Vertrag mit der realen Laufzeit

Das Companion-Objekt erweitert `BlogPostPersistenceSpec` mit fünf Prüfungen:

- Das Schema deckt jedes persistente Attribut ab; ID und Version behalten ihre JPA-Metadaten.
- Typed Criteria findet einen veröffentlichten Slug und gibt kein Ergebnis für Entwürfe oder unbekannte Slugs zurück.
- Ein vollständig ausgefüllter Artikel serialisiert alle acht Felder, einschließlich seiner Version und der genauen Veröffentlichungszeit.
- Optionale `null`-Felder werden ausgelassen, die Versionsnummer 0 bleibt erhalten.
- Eingehendes JSON kann die durch die Standardregeln geschützten Felder nicht ändern.

Die Tests aktivieren den Anforderungskontext von CDI und lösen `RequestTransaction` über den Container auf. Dies übt den gleichen EntityManager-Produzenten aus, der von der Anwendung verwendet wird. Eine spätere Transaktion verwendet das Schema, nachdem die Anforderung, die zuerst initialisiert wurde, beendet wurde.

Der Serialisierer-Test ruft den realen Mapper direkt auf:

```scala
val constructRule = [T] => (clazz: Class[T]) =>
  clazz.getDeclaredConstructor().newInstance()

val output = JsonMapper.serialize(
  post,
  TypeResolver.resolve(classOf[BlogPost]),
  null,
  constructRule
)
```

Hier wird `JsonMapper` aus `com.anjunar.json.mapper` und `TypeResolver` aus `com.anjunar.scala.universe` importiert. Das `null`-Argument bedeutet, dass dieser Test keinen Entitätsgraphen liefert. Sein Resolver konstruiert unsere zustandslosen Standardregeln; spätere Regeln mit injizierten Diensten werden über CDI aufgelöst.

Optionale Nullwerte fehlen in dieser Mapper-Version. Sie werden nicht als explizite JSON null ausgegeben. Diese Unterscheidung werden wir bei der Umsetzung des Frontend-Modells beibehalten.

Die neue Abhängigkeit bietet auch eine Logback-Implementierung. Der Begleiter enthält einen kleinen `logback.xml` mit einem INFO-Konsolenlogger, so dass ein normales Startup kein Framework-DEBUG-Tracing ausgibt. Es ändert nicht das Verhalten des Mappers.

## Dieses Kapitel ausführen

Verwenden Sie die vorhandene separate Entwicklungsdatenbank und die Umgebungsvariablen in Kapitel 6:

```text
sbt --server "application-backend/update"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server "application-backend/testFull"
```

In einer Datenbank bereits in Kapitel 6 meldet die Migration `AlreadyApplied` mit null SQL-Anweisungen. Das Schema und die JSON-Annotationen fügen keine persistenten Felder hinzu. Erwarten Sie **33 erfolgreiche Tests**.

Für eine neue Datenbank erstellt `SchemaMain migrate` die aktuelle Tabelle. Bei einer älteren Datenbank aus Kapitel 5 führen Sie zuerst die Übernahmeschritte aus Kapitel 6 aus.

Der [vollständige Quellcode zu Kapitel 7](https://github.com/anjunar/anjunar-blog-example/tree/ee01b68ac30a1a2f93f1d8153b7f08114637bef2) ist die genaue Version, die von diesem Artikel verwendet wird.

Wir haben jetzt ein Feldmodell, das vom Mapper und von einer echten Kriterienabfrage verwendet wird. Kapitel 8 verbindet diesen Vertrag mit öffentlichen REST-Endpunkten, Antwortumschlägen und Entitätsgraphen.
