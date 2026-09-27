Ein Leser braucht zwei Dinge aus unserem Backend: eine Liste der veröffentlichten Beiträge und den vollständigen Text eines Beitrags. Die Datenbank und das Feldschema sind bereit. Dieses Kapitel stellt sie über HTTP zur Verfügung.

Wir geben die tatsächlichen `BlogPost`-Entitäten über den JSON-Mapper zurück. Benannte Entitätsgraphen wählen die Felder für jede Antwort aus; kleine `Data`- und `Table`-Umschläge beschreiben, wie diese Werte den Client erreichen.

Beginnen Sie mit dem [Kapitel 7 Quelle](https://github.com/anjunar/anjunar-blog-example/tree/ee01b68ac30a1a2f93f1d8153b7f08114637bef2) und seiner migrierten PostgreSQL-Datenbank. Alle Pfade unten sind relativ zum Repository-Verzeichnis. Wir behalten die veröffentlichten Abhängigkeiten und Versionen aus diesem Kapitel.

## Den öffentlichen API-Vertrag festlegen

|Anfrage|Antwort|
| --- | --- |
|`GET /service/blog/posts`|Eine Seite mit veröffentlichten Posts ohne ihren langen Inhalt.|
|`GET /service/blog/posts/{slug}`|Der vollständige veröffentlichte Post.|
|Detailanfrage für einen Entwurf oder einen unbekannten Slug|HTTP 404.|

Die Liste akzeptiert `offset` und `limit`, standardmäßig auf 0 und 20. Limit muss zwischen 1 und 100 liegen; Der Offset darf nicht negativ sein. Ungültige Parameter geben HTTP 400 zurück.

Diese Endpunkte sind Public Reads. Sie bieten keine Schreibmethode. Anmeldung, Berechtigungen und sichere Bearbeitung folgen nach dem ersten Frontend in dieser Reihe.

Drei separate Entscheidungen formen jede Antwort:

|Entscheidung|Mechanismus|
| --- | --- |
|Welche Posts darf dieser Besucher lesen?|Das `PUBLISHED`-Prädikat der Abfrage.|
|Welche Felder umfasst dieser Endpunkt?|Sein benannter Entitätsgraph.|
|Welche kartierten Felder sind nach den aktuellen Regeln sichtbar?|Die `EntitySchema`- und Mapper-Regeln aus Kapitel 7.|

Ein Graph verbirgt keine Entwurfszeile. Ein lesbares Schema gewährt nicht die Erlaubnis, jeden gespeicherten Beitrag zurückzugeben. Die Abfrage muss zuerst die öffentlichen Zeilen auswählen.

## Machen Sie BlogPost zu einer Mapper-Entität

Der Mapper verwendet `EntityProvider`, um Entitäten zu erkennen, deren Felder durch einen Graphen ausgewählt werden sollen. Fügen Sie diesen Import in `application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala` hinzu:

```scala
import com.anjunar.json.mapper.provider.EntityProvider
```

Ändern Sie die Klassenerklärung in `class BlogPost extends EntityProvider`. Das Interface verlangt eine UUID als ID und eine Scala-`Long`-Version.

Unsere ID stimmt bereits überein. Ersetzen Sie das Versionsfeld durch:

```scala
@Version
@Column(nullable = false)
@SchemaId("dcb0681e")
@JsonbProperty
var version: Long = -1L
```

Aktualisieren Sie auch den Eintrag in `BlogPost.Schema`:

```scala
val version: SingularProperty[BlogPost, Long] = reference(_.version)
```

Im vorherigen Kapitel wurde nullable `lang.Long` verwendet. Die neue Schnittstelle verwendet primitive `Long`, mit -1 als Wert vor der Persistenz. Hibernate weist Version 0 beim Einfügen zu und erhöht sie bei Änderungen. Die Persistenztests überprüfen beide Verhaltensweisen und lehnen immer noch veraltete Bearbeitungen ab.

Dies ändert den Scala-Vertrag, nicht die Datenbankspalte. Die bestehende `bigint`-Spalte und ihre stabile Schema-ID bleiben bestehen. Das Ausführen von `SchemaMain migrate` gegen eine Datenbank aus Kapitel 6 oder 7 meldet `AlreadyApplied` mit null SQL-Anweisungen.

Die Annotationen bleiben gewöhnliche Annotationen auf Klassenkörperfeldern; sie brauchen kein `@field`-Ziel.

## Liste und Detailfelder auswählen

Hinzufügen von `NamedAttributeNode`, `NamedEntityGraph` und `NamedEntityGraphs` zu den JPA-Importen in `BlogPost.scala`. Fügen Sie diese Annotationen über der Entitätsklasse hinzu und behalten Sie die vorhandenen Annotationen bei:

```scala
@NamedEntityGraphs(Array(
  new NamedEntityGraph(name = "BlogPost.list", attributeNodes = Array(
    new NamedAttributeNode("id"),
    new NamedAttributeNode("version"),
    new NamedAttributeNode("slug"),
    new NamedAttributeNode("title"),
    new NamedAttributeNode("summary"),
    new NamedAttributeNode("status"),
    new NamedAttributeNode("publishedAt")
  )),
  new NamedEntityGraph(name = "BlogPost.detail", attributeNodes = Array(
    new NamedAttributeNode("id"),
    new NamedAttributeNode("version"),
    new NamedAttributeNode("slug"),
    new NamedAttributeNode("title"),
    new NamedAttributeNode("content"),
    new NamedAttributeNode("summary"),
    new NamedAttributeNode("status"),
    new NamedAttributeNode("publishedAt")
  ))
))
```

Beide Graphen enthalten `id` und `version`. Die Liste lässt `content` aus; das Detail enthält alle acht Felder. Die Graphennamen verbinden Abfragen, REST-Methoden und Antwortmetadaten.

Wir verwenden die Graphen an zwei Stellen. Hibernate erhält sie als Fetch-Hinweise bei der Abfrage. Der Mapper erhält sie als JSON-Feldauswahl beim Schreiben der Antwort.

Diese Jobs sind verwandt, aber sie sind nicht identisch. Das Auslassen von `content` im JSON-Graphen garantiert, dass es in der Listenantwort fehlt. Ein JPA-Abrufgraph garantiert nicht, dass Hibernate jede nicht ausgewählte Basisspalte in seiner SQL-Abfrage auslässt. Wir werden dedizierte Listenprojektionen und Abfrageleistung im Suchkapitel untersuchen.

## Öffentliche Zeilen in einer vorhersehbaren Reihenfolge abfragen

Fügen Sie in `BlogPost.scala` `import java.util` hinzu; behalten Sie den vorhandenen `java.lang`-Import für den Java-Wrapper der Zählabfrage bei. Ersetzen Sie `findPublishedBySlug` und fügen Sie diese beiden Methoden im Begleitobjekt nach seiner verschachtelten `Schema`-Klasse hinzu:

```scala
def findPublishedBySlug(slug: String)(using entityManager: EntityManager): Option[BlogPost] = {
    val builder = entityManager.getCriteriaBuilder
    val query = builder.createQuery(classOf[BlogPost])
    val post = query.from(classOf[BlogPost])
    query.select(post).where(
      builder.equal(post.get(schema.slug), builder.parameter(classOf[String], "slug")),
      builder.equal(post.get(schema.status), BlogPostStatus.PUBLISHED)
    )
    Option(entityManager.createQuery(query)
      .setHint("jakarta.persistence.fetchgraph", entityManager.getEntityGraph("BlogPost.detail"))
      .setParameter("slug", slug).getSingleResultOrNull)
  }

  def listPublished(offset: Int, limit: Int)(using entityManager: EntityManager): util.List[BlogPost] = {
    val builder = entityManager.getCriteriaBuilder
    val query = builder.createQuery(classOf[BlogPost])
    val post = query.from(classOf[BlogPost])
    query.select(post)
      .where(Seq(builder.equal(post.get(schema.status), BlogPostStatus.PUBLISHED))*)
      .orderBy(builder.desc(post.get(schema.publishedAt)), builder.asc(post.get(schema.id)))
    entityManager.createQuery(query)
      .setHint("jakarta.persistence.fetchgraph", entityManager.getEntityGraph("BlogPost.list"))
      .setFirstResult(offset)
      .setMaxResults(limit)
      .getResultList
  }

  def countPublished()(using entityManager: EntityManager): Long = {
    val builder = entityManager.getCriteriaBuilder
    val query = builder.createQuery(classOf[lang.Long])
    val post = query.from(classOf[BlogPost])
    query.select(builder.count(post))
      .where(Seq(builder.equal(post.get(schema.status), BlogPostStatus.PUBLISHED))*)
    entityManager.createQuery(query).getSingleResult.longValue()
  }
```

Die Liste ordnet nach Veröffentlichungszeit absteigend an. Bei gleichen Zeitstempeln entscheidet die UUID in aufsteigender Reihenfolge, so dass gleiche Zeitstempel keine willkürliche Reihenfolge erzeugen. Das gleiche `PUBLISHED`-Prädikat gilt für die Seite, ihre Gesamtanzahl und die Detailsuche.

Die Schreibweise `Seq(...)*` wählt explizit Javas Varargs `where`-Überlastung aus. Ohne sie ist ein einziges Prädikat mehrdeutig zwischen den Criteria-Überladungen in dieser Scala Version.

`size` bedeutet die gesamte veröffentlichte Anzahl vor der Paginierung. Es ist nicht die Anzahl der Zeilen auf der aktuellen Seite. Page und Count sind separate Abfragen, so dass Gleichzeitiges Veröffentlichen die Gesamtsumme zwischen ihnen ändern kann. Wir bauen hier eine grundlegende Offset-Paginierung, keine stabile Momentaufnahme über mehrere Anfragen hinweg.

Es gibt noch keine Regel für zeitgesteuerte Veröffentlichung: `PUBLISHED` ist die Veröffentlichungsentscheidung. Der Zeitstempel zeichnet die Veröffentlichung auf und bestimmt die Sortierung.

## Geben Sie Antworten eine konsistente Form

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/Data.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.provider.DTO
import jakarta.json.bind.annotation.JsonbProperty

import scala.annotation.meta.field

class Data[E](
    @(JsonbProperty @field) val data: E,
    @(JsonbProperty @field) val schema: Schema
) extends DTO
```

Eine Detailantwort verpackt eine Entität in `data` und ihre Feldbeschreibung in `schema`. Es kopiert nicht jedes BlogPost-Feld in ein zweites Backend-Modell.

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/Table.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.provider.DTO
import jakarta.json.bind.annotation.JsonbProperty

import java.util
import scala.annotation.meta.field

class Table[C](
    @(JsonbProperty @field) val rows: util.List[C],
    @(JsonbProperty @field) val size: Long
) extends DTO
```

Eine Liste ist ein `Table[Data[BlogPost]]`: `rows` enthält dieselben Daten- und Schema-Hüllen, `size` die Gesamtzahl.

`DTO` markiert diese Umschläge für unseren REST-Writer. Diese Felder werden als Konstruktorparameter deklariert, so dass ihre JSON-Annotationen explizit auf das Backing-Feld mit `@field` abzielen. Das unterscheidet sich von den Klassen-Body-Feldern der Entität.

## Beschreiben Sie die ausgewählten Felder

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/Schema.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.provider.DTO
import com.anjunar.json.mapper.schema.EntitySchema
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.{EntityGraph as JpaEntityGraph}

import java.util
import scala.annotation.meta.field
import scala.jdk.CollectionConverters.*

class Schema(@(JsonbProperty @field) val entries: util.List[SchemaProperty]) extends DTO

class SchemaProperty(
    @(JsonbProperty @field) val name: String,
    @(JsonbProperty @field)("type") val typeName: String
) extends DTO

object Schema {
  // Describes the selected fields, not the caller's permissions.
  def forGraph(source: EntitySchema[?], graph: JpaEntityGraph[?]): Schema = {
    val selected = graph.getAttributeNodes.asScala.map(_.getAttributeName).toSet
    val entries = source.properties.valuesIterator
      .filter(property => selected.contains(property.name))
      .map(property => new SchemaProperty(property.name, property.typeName))
      .toList.asJava
    new Schema(entries)
  }
}
```

Dieses `Schema` ist ein Antwortmodell. Es ist getrennt von dem zwischengespeicherten `BlogPost.Schema`, das vom Mapper und den Kriterien verwendet wird.

Für jede Antwort nimmt `forGraph` die ausgewählten Attributnamen und erhält deren Namen und Typen aus dem vorhandenen EntitySchema. Wir pflegen keinen weiteren handgeschriebenen Feldkatalog. Alle aktuellen Felder sind skalar; verschachtelte Beziehungsmetadaten können mit dem Beziehungskapitel wachsen.

Das Ergebnis beschreibt die Struktur. Es sagt nicht, ob ein Anrufer ein Feld bearbeiten darf, und das Frontend darf einen Eintrag nicht als Schreibberechtigung behandeln. Permission-Regeln bleiben auf dem Server. Kontextuelle Links werden im Kapitel HATEOAS hinzugefügt.

Eine nullbare Zusammenfassung hat bei der Auswahl immer noch einen Schemaeintrag, auch wenn der jeweilige Beitrag keinen Zusammenfassungswert hat. Feldmetadaten und Feldwerte beantworten unterschiedliche Fragen.

## Verbinden Sie die Ressourcenmethoden

Wir brauchen eine kleine Annotation, um dem Antwortschreiber mitzuteilen, welchen Graphen er verwenden soll. Erstellen Sie `application/backend/src/main/java/com/anjunar/blog/EntityGraph.java`:

```java
package com.anjunar.blog;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface EntityGraph {
    String value();
}
```

Das ist unsere Annotation für Methoden. Der Graphtyp der Persistenz-API wird als `JpaEntityGraph` importiert, wo beide Namen kollidieren würden.

Erstellen Sie nun `application/backend/src/main/scala/com/anjunar/blog/BlogPostsResource.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.ws.rs.{BadRequestException, DefaultValue, GET, NotFoundException, Path, PathParam, Produces, QueryParam}
import jakarta.ws.rs.core.MediaType

import scala.compiletime.uninitialized
import scala.jdk.CollectionConverters.*

@Path("/blog/posts")
@Produces(Array(MediaType.APPLICATION_JSON))
@RequestScoped
class BlogPostsResource {
  @Inject
  var entityManager: EntityManager = uninitialized

  @GET
  @EntityGraph("BlogPost.list")
  def list(@QueryParam("offset") @DefaultValue("0") rawOffset: String,
      @QueryParam("limit") @DefaultValue("20") rawLimit: String): Table[Data[BlogPost]] = {
    val offset = rawOffset.toIntOption.getOrElse(throw new BadRequestException("offset must be an integer"))
    val limit = rawLimit.toIntOption.getOrElse(throw new BadRequestException("limit must be an integer"))
    if (offset < 0 || limit < 1 || limit > 100)
      throw new BadRequestException("offset must be nonnegative and limit must be between 1 and 100")

    given EntityManager = entityManager
    val schema = Schema.forGraph(BlogPost.schema, entityManager.getEntityGraph("BlogPost.list"))
    val rows = BlogPost.listPublished(offset, limit).asScala
      .map(post => new Data(post, schema)).toList.asJava
    new Table(rows, BlogPost.countPublished())
  }

  @GET
  @Path("/{slug}")
  @EntityGraph("BlogPost.detail")
  def read(@PathParam("slug") slug: String): Data[BlogPost] = {
    given EntityManager = entityManager
    val post = BlogPost.findPublishedBySlug(slug).getOrElse(throw new NotFoundException())
    val schema = Schema.forGraph(BlogPost.schema, entityManager.getEntityGraph("BlogPost.detail"))
    new Data(post, schema)
  }
}
```

RESTEasy liest die Paging-Werte als Strings. Wir parsen sie ausdrücklich, so dass fehlerhafte Werte, Ganzzahlüberlauf und ungültige Grenzen HTTP 400 erzeugen.

Die Detailsuche liefert 404, wenn die Abfrage nach einem veröffentlichten Slug keine Zeile findet. Das deckt sowohl fehlende Posts als auch Entwürfe ab, ohne den Inhalt eines Entwurfs preiszugeben.

Die bestehende CDI-Erweiterung entdeckt `@Path`-Ressourcen. Es gibt keine Registrierungsliste zum Aktualisieren. Der injizierte EntityManager gehört weiterhin zur Anforderungstransaktion aus Kapitel 4.

Halten Sie die generischen Rückgabetypen der Methode intakt. Der Writer benötigt `Table[Data[BlogPost]]`, um beiden Umschlägen zu ihrem Entity-Typ zu folgen.

## Lassen Sie den Mapper JSON schreiben

Der Mapper benötigt einen Resolver für Feldregeln. Ersetzen Sie `RuntimeContext.scala` durch:

```scala
package com.anjunar.blog

import jakarta.enterprise.inject.spi.CDI
import jakarta.persistence.EntityManager

object RuntimeContext {
  def bean[T](clazz: Class[T]): T = {
    val instance = CDI.current().select(clazz)
    if (instance.isUnsatisfied) clazz.getDeclaredConstructor().newInstance()
    else instance.get()
  }

  def entityManager(): EntityManager =
    CDI.current().select(classOf[EntityManager]).get()
}
```

Eine von CDI verwaltete Regel wird über CDI aufgelöst. Unsere zustandslose Standardregel hat keine Bean-Registrierung, daher verwendet der Fallback seinen No-Argument-Konstruktor. Eine Regel, die Injektion benötigt, muss eine CDI-Bean sein; die Konstruktion durch den Fallback würde ihre Abhängigkeiten nicht injizieren.

Erstellen Sie nun `application/backend/src/main/scala/com/anjunar/blog/MapperMessageBodyWriter.scala`:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.JsonMapper
import com.anjunar.json.mapper.provider.DTO
import com.anjunar.scala.universe.TypeResolver
import jakarta.annotation.Priority
import jakarta.ws.rs.Produces
import jakarta.ws.rs.container.ResourceInfo
import jakarta.ws.rs.core.{Context, MediaType, MultivaluedMap}
import jakarta.ws.rs.ext.{MessageBodyWriter, Provider}

import java.io.OutputStream
import java.lang.annotation.Annotation
import java.lang.reflect.Type
import java.nio.charset.StandardCharsets
import scala.compiletime.uninitialized

@Provider
@Priority(3900)
@Produces(Array(MediaType.APPLICATION_JSON))
class MapperMessageBodyWriter extends MessageBodyWriter[Any] {
  @Context
  var resource: ResourceInfo = uninitialized

  override def isWriteable(clazz: Class[?], genericType: Type,
      annotations: Array[Annotation], mediaType: MediaType): Boolean =
    classOf[DTO].isAssignableFrom(clazz)

  override def writeTo(body: Any, clazz: Class[?], genericType: Type,
      annotations: Array[Annotation], mediaType: MediaType,
      headers: MultivaluedMap[String, Object], stream: OutputStream): Unit = {
    val annotation = resource.getResourceMethod.getAnnotation(classOf[EntityGraph])
    val graph =
      if (annotation == null) null
      else RuntimeContext.entityManager().getEntityGraph(annotation.value())
    val targetType = if (genericType == null) clazz else genericType
    val json = JsonMapper.serialize(body, TypeResolver.resolve(targetType), graph,
      [T] => (ruleClass: Class[T]) => RuntimeContext.bean(ruleClass))
    stream.write(json.getBytes(StandardCharsets.UTF_8))
  }
}
```

`@Provider` lässt die vorhandene Erweiterung den Autor entdecken. `isWriteable` wählt unsere DTO-Umschläge aus; normale Textantworten werden über den Textschreiber von RESTEasy fortgesetzt.

Die `@EntityGraph`-Annotation der Ressourcenmethode liefert den Graphennamen. Der Mapper erhält den tatsächlichen Graphen, den deklarierten generischen Typ der Antwort und den Regelauflöser. Es folgt `Table` und `Data` auf den enthaltenen BlogPost und wendet dort den Graphen an. Die Umschläge selbst sind keine BlogPost-Graphenattribute.

Die Verwendung von nur `body.getClass` würde die verschachtelten Argumente verlieren. Das Halten von `genericType` ist daher Teil des JSON-Vertrags, nicht nur ein Reflexionsdetail.

Der Writer kodiert das Ergebnis als UTF-8. Zitate, Newlines und Nicht-ASCII-Text werden vom Mapper bearbeitet; wir setzen JSON-Strings nicht von Hand zusammen.

## Den Persistence Context bei HEAD bis zur Serialisierung halten

Unsere Grenze von Kapitel 4 hält den EntityManager bereits offen, während er eine normale Antwort schreibt. Der neue Autor zeigt eine Anpassung, die wir für `HEAD` benötigen.

RESTEasy verarbeitet eine implizite HEAD-Anfrage, indem es die GET-Ressource und ihren Schreiber aufruft und dann den Antwortkörper unterdrückt. Der Mapper benötigt weiterhin den EntityManager, um seinen Graphen aufzulösen. Das Schließen im Antwortfilter würde eine ansonsten gültige HEAD-Anfrage in HTTP 500 umwandeln.

In `TransactionBoundary.scala` ersetzen:

```scala
if (!response.hasEntity || request.getMethod == "HEAD") transaction.finish(successful)
```

mit:

```scala
// RESTEasy also serializes implicit HEAD responses; its writer still needs the EntityManager.
if (!response.hasEntity) transaction.finish(successful)
```

Antworten mit Entität werden nun erst nach dem Writer abgeschlossen, auch bei HEAD. Antworten ohne Entität werden weiterhin im Antwortfilter abgeschlossen. GET und HEAD bleiben schreibgeschützte Transaktionen mit Rollback. Der vorhandene Puffer verhindert weiterhin, dass ein Serialisierungsfehler eine teilweise erfolgreiche JSON-Antwort sendet.

## Lesen Sie den aktuellen JSON-Vertrag

Der Mapper fügt `@type`-Marker hinzu. Optionale Nullfelder und leere Sammlungen werden in Version 1.1.5 weggelassen.

Bei einer leeren Datenbank sieht die vollständige Listenantwort so aus:

```json
{"size":0,"@type":"Table"}
```

Es gibt kein `rows: []` Mitglied. Ein Offset über die letzte Zeile hinaus lässt auch `rows` aus, während `size` die gesamte veröffentlichte Anzahl behält. Unser Frontend-Modell wird seine Kollektion entsprechend initialisieren.

Für ein veröffentlichtes Beispiel ist das `data`-Objekt der Detailantwort:

```json
{
  "id": "b62db12a-61a7-46a5-b8d2-c487f80e825a",
  "version": 0,
  "slug": "our-first-public-post",
  "title": "Our first public post",
  "content": "This is the complete article. The list response leaves this text out.",
  "status": "PUBLISHED",
  "publishedAt": "2026-09-27T10:15:42.123456Z",
  "summary": "A working REST response from PostgreSQL.",
  "@type": "BlogPost"
}
```

Daneben enthält `schema.entries` die acht Feldbeschreibungen. Zum Beispiel ist der Versionseintrag:

```json
{"name":"version","type":"Long","@type":"SchemaProperty"}
```

Eine Listenzeile verwendet den gleichen Umschlag, wobei `content` sowohl im `data`-Objekt als auch in den ausgewählten Feldbeschreibungen fehlt. Version Null bleibt vorhanden.

## Mit Beispielartikeln ausführen

Legen Sie die Datenbankumgebungsvariablen aus den früheren Kapiteln für eine separate lokale Datenbank fest. Führen Sie dann aus:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server "application-backend/testFull"
```

Eine vorhandene Kapitel 6/7 Datenbank meldet `AlreadyApplied` mit null SQL-Anweisungen. Eine leere Datenbank wird durch die Migration initialisiert. Bei einer älteren Datenbank aus Kapitel 5 führen Sie zuerst die Übernahmeschritte aus Kapitel 6 aus.

Erwarten Sie **41 erfolgreiche Tests**. Die acht neuen HTTP-Tests umfassen Liste und Detailfeldauswahl, Zählungen und Paging, Sortierung bei gleichen Zeitstempeln, Entwurfsausschluss, ungültige Parameter, nicht unterstützte Schreibvorgänge, leere Seiten und HEAD. Sie verwenden den echten Server, CDI, Mapper und PostgreSQL und entfernen dann nur ihre eigenen Zeilen. Fehlerfälle erzeugen absichtlich Serverprotokolle.

Für manuelle Anfragen enthält der Begleiter `database/examples/public-posts.sql`. Es fügt das öffentliche Beispiel oben und einen privaten Entwurf ein. Wenden Sie es nach der Migration an. Mit Compose:

```text
docker compose cp database/examples/public-posts.sql postgres:/tmp/public-posts.sql
docker compose exec -T postgres psql -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --file /tmp/public-posts.sql
```

Mit einer nativen PostgreSQL-Installation passen Sie die Verbindungsdetails an:

```text
psql -h 127.0.0.1 -p 5433 -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --file database/examples/public-posts.sql
```

`psql` fragt nach dem Datenbankpasswort; es liest `BLOG_DB_PASSWORD` nicht. Die Wiederholung des Skripts lässt seine beiden festen Beispiel-IDs unverändert. Es sind optionale Beispieldaten, keine Schemamigration oder automatisches Start-Seeding.

Starten Sie die Anwendung:

```text
sbt --server "application-backend/run"
```

In einem anderen Terminal:

```text
curl -i "http://127.0.0.1:8080/service/blog/posts?offset=0&limit=20"
curl -i http://127.0.0.1:8080/service/blog/posts/our-first-public-post
curl -i http://127.0.0.1:8080/service/blog/posts/our-private-draft
curl -i "http://127.0.0.1:8080/service/blog/posts?limit=0"
curl -I http://127.0.0.1:8080/service/blog/posts/our-first-public-post
```

Verwenden Sie `curl.exe` in Windows PowerShell, falls erforderlich. Erwarten Sie 200, 200, 404, 400 und 200. In einer ansonsten leeren Datenbank hat die erste Antwort eine veröffentlichte Zeile und Größe 1. Das Detail beinhaltet den Inhalt; HEAD hat keinen Body. Stoppen Sie den Server mit Ctrl + C.

Der [vollständige Quellcode zu Kapitel 8](https://github.com/anjunar/anjunar-blog-example/tree/54fac6042b34be4c8d7ab9b3f4ed054272220b81) ist die genaue Revision, die in diesem Artikel verwendet wird.

Unser erster Meilenstein ist abgeschlossen: Die Anwendung startet, speichert Beiträge in PostgreSQL und bedient veröffentlichte Beiträge über REST. Kapitel 9 startet die Scala.js-Schnittstelle, die sie anzeigt.
