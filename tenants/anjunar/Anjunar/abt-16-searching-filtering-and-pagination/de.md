# Suchen, filtern und paginieren

Wir können jetzt Artikel erstellen und bearbeiten. Wenn die Liste wächst, wird es wichtig, einen bestimmten Artikel wiederzufinden. Leser sollen veröffentlichte Artikel durchsuchen können. Administratoren müssen außerdem Entwürfe finden, die Sortierung ändern und durch passende Ergebnisse blättern können.

Eine Suche ist mehr als ein Textfeld, das an eine Abfrage angehängt ist. Die Trefferzahl muss dieselben Filter verwenden wie die Zeilen. Bei gleichen Titeln braucht es eine vorhersehbare Reihenfolge. Eine Änderung des Filters soll zur ersten Seite zurückkehren; beim Wechsel auf eine andere Seite muss der Filter erhalten bleiben. Wird die URL neu geladen oder geteilt, sollen dieselben Steuerelemente wiederhergestellt werden.

Dieses Kapitel baut den vollständigen Ablauf mit der wiederverwendbaren HibernateSearch-Architektur des Stacks auf: ein annotiertes Suchmodell, CDI-Provider, typisierte Criteria-Abfragen und ein gebundenes Suchformular. Außerdem lassen wir die Datenbank nur die Spalten auswählen, die eine Liste tatsächlich benötigt.

## Den Checkpoint starten

Ausgangspunkt ist Kapitel 15:

```text
git switch --detach 9e3edb2af6d498cf299ffbb2c898bf37aeeedf1b
```

Die fertige Version dieses Kapitels:

```text
git switch --detach 214eeb6d5a2c6062013d95abf7f990a1cb032cce
sbt --server frontendAssets
sbt --server "application-backend/run"
```

Verwende die vorhandene Entwicklungsdatenbank und den Administrator. In diesem Kapitel ändern wir weder das Datenbank-Mapping noch Bibliotheksversionen. Für lokales HTTP muss BLOG_COOKIE_SECURE=false gesetzt bleiben. Der [Begleit-Leitfaden](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/docs/searching-filtering-and-pagination.md) enthält den Ablauf und die Testkonfiguration.

Die öffentliche Liste unter /en erhält Textsuche, Sortierung und Seitengröße. Die redaktionelle Liste unter /en/editorial erhält dieselben Steuerelemente sowie den Veröffentlichungsstatus. Eine Suche wird ausdrücklich über **Suchen** oder die Eingabetaste abgeschickt.

## 1. Den Suchvertrag festlegen

Beide Endpunkte akzeptieren folgende Abfrageparameter:

| Parameter | Zulässige Werte |
| --- | --- |
| q | Ein getrimmter Teilstring mit höchstens 100 Zeichen; leer bedeutet kein Textfilter. |
| status | Redaktionell: nicht angegeben, DRAFT oder PUBLISHED. Öffentlich: nicht angegeben oder PUBLISHED. |
| sort | newest, oldest, title oder title-desc. |
| offset | Nichtnegative Ganzzahl, Standardwert 0. |
| limit | Ganzzahl von 1 bis 100, Standardwert 20. |

Die öffentliche Standardreihenfolge bleibt „neueste zuerst“. Die redaktionelle Liste behält ihre bisherige Sortierung nach Titel. Unbekannte Sortiernamen, ungültige Grenzen und Steuerzeichen führen zu HTTP 400.

Zum Beispiel:

```text
/service/blog/posts?q=scala&sort=title&offset=0&limit=10
/service/editorial/posts?q=release&status=DRAFT&sort=title&offset=0&limit=10
```

Lege PostSearchParams.scala im Backend-Paket unter [application/backend/src/main/scala/com/anjunar/blog/](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/PostSearchParams.scala) an:

```scala
package com.anjunar.blog

import jakarta.ws.rs.{BadRequestException, DefaultValue, QueryParam}

final class PostSearchParams {
  @QueryParam("q") var query: String = ""
  @QueryParam("status") var status: String = ""
  @QueryParam("sort") var sort: String = ""
  @QueryParam("offset") @DefaultValue("0") var offset: String = "0"
  @QueryParam("limit") @DefaultValue("20") var limit: String = "20"

  def search(editorial: Boolean): BlogPostSearch = {
    val text = Option(query).getOrElse("").trim
    if (text.length > 100 || text.exists(Character.isISOControl))
      throw new BadRequestException("q must contain at most 100 characters and no control characters")
    val selectedStatus = Option(status).getOrElse("") match {
      case "" => None
      case "DRAFT" if editorial => Some(BlogPostStatus.DRAFT)
      case "PUBLISHED" => Some(BlogPostStatus.PUBLISHED)
      case _ => throw new BadRequestException("Invalid status filter")
    }
    val selectedSort = Option(sort).filter(_.nonEmpty).getOrElse(if (editorial) "title" else "newest")
    if (!Set("newest", "oldest", "title", "title-desc").contains(selectedSort))
      throw new BadRequestException("Invalid sort order")
    val start = Option(offset).flatMap(_.toIntOption).filter(_ >= 0)
      .getOrElse(throw new BadRequestException("offset must be a nonnegative integer"))
    val size = Option(limit).flatMap(_.toIntOption).filter(value => value >= 1 && value <= 100)
      .getOrElse(throw new BadRequestException("limit must be between 1 and 100"))
    BlogPostSearch(text, if (editorial) selectedStatus else Some(BlogPostStatus.PUBLISHED), selectedSort, start, size)
  }
}
```

PostSearchParams ist das JAX-RS-Eingabeobjekt. Es sammelt Zeichenfolgen, damit unsere eigene Eingabegrenze einheitliche Validierungsfehler erzeugen kann. Das unveränderliche BlogPostSearch erweitert AbstractSearch und übergibt die geprüften Werte an HibernateSearch. Dieses Modell und seine Provider definieren wir in Abschnitt 3.

Der öffentliche Endpunkt ruft search(editorial = false) auf. Damit wird immer PUBLISHED ausgewählt, auch wenn der Aufrufer zufällig Administrator ist. status=DRAFT am öffentlichen Endpunkt wird zurückgewiesen. Die Berechtigung für den redaktionellen Endpunkt prüft unabhängig davon dessen vorhandene ADMIN-Richtlinie.

Diese Abfrageparameter beschreiben einen Lesevorgang. Sie werden an der Anfragegrenze eingelesen; der JSON-Mapper bleibt für die Validierung von Entitätsänderungen aus den vorigen Kapiteln zuständig.

## 2. Eine Listenprojektion auswählen

Die Liste muss nicht den gesamten Artikelinhalt laden. Der bestehende BlogPost.list-Graph wählt zwar die Felder aus, die der Mapper sendet, garantiert allein aber nicht, dass die SQL-Abfrage die Inhaltsspalte auslässt.

Lege BlogPostSummary.scala im selben Backend-Paket an:

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.annotations.UseConverter
import com.anjunar.json.mapper.provider.DTO
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.criteria.Expression
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.time.Instant
import java.util
import java.util.UUID
import scala.annotation.meta.field

// A read-only list projection. Load BlogPost detail before editing.
final class BlogPostSummary(
    @(JsonbProperty @field) val id: UUID,
    @(JsonbProperty @field) val version: Long,
    @(JsonbProperty @field) val slug: String,
    @(JsonbProperty @field) val title: String,
    @(JsonbProperty @field) val summary: String,
    @(JsonbProperty @field) val status: BlogPostStatus,
    @(JsonbProperty @field) @(UseConverter @field)(classOf[InstantConverter]) val publishedAt: Instant
) extends DTO

object BlogPostSummary {
  def select(
      query: JpaCriteriaQuery[BlogPostSummary], post: JpaRoot[BlogPost],
      selection: util.List[Expression[?]], builder: HibernateCriteriaBuilder
  ): JpaCriteriaQuery[BlogPostSummary] = {
    val schema = BlogPost.schema
    query.select(builder.construct(classOf[BlogPostSummary],
      post.get(schema.id), post.get(schema.version), post.get(schema.slug),
      post.get(schema.title), post.get(schema.summary), post.get(schema.status),
      post.get(schema.publishedAt)))
  }
}
```

Das ist bewusst eine schreibgeschützte Listenform: sieben Felder aus BlogPost mit denselben Namen und Werttypen. Ein Inhaltsfeld und ein Persistenzlebenszyklus fehlen. Hibernate erstellt das Objekt direkt aus den ausgewählten Spalten.

Die Annotationen beziehen sich auf die Konstruktorfelder, denn dort liest der Mapper die Metadaten dieses DTOs. Der Zeitstempel verwendet weiterhin den vorhandenen InstantConverter.

Diese Projektion ersetzt die Entität weder im Detail- noch im Aktualisierungsendpunkt. Vorschau und Bearbeitung laden weiterhin das echte BlogPost-Objekt; Schreibvorgänge laufen weiterhin über PreparedChange. Die Projektion erzeugt auch kein zweites bearbeitbares Domänenmodell, sondern eine bewusst eingeschränkte Listenantwort.

Projektionsfelder führen die Eigenschaftsregeln der Entität nicht automatisch aus. Hier sind sie die ausdrücklich erlaubten Metadaten für die Sichtbarkeit der jeweiligen Listenzeile. Ein künftig eingeschränktes Feld müsste auch in der Projektion eine Berechtigungsentscheidung erhalten.

## 3. Die HibernateSearch-Architektur des Stacks wiederverwenden

Eine Suche im Blog ist ein geeigneter erster Anwendungsfall für die wiederverwendbare Suchinfrastruktur. Der Stack trennt die Aufgaben bereits in ein AbstractSearch-Modell, die Metadaten @RestPredicate und @RestSort, CDI-Provider und HibernateSearch zur Ausführung von Treffer- und Zählabfragen. Diese Struktur übernehmen wir in das Begleitprojekt.

Hier bezeichnet HibernateSearch unseren Criteria-Helfer. Es ist nicht das separate Hibernate-Search-Produkt zur Volltextindizierung. Wir fügen weder eine Bibliothek noch eine lokale Stack-Abhängigkeit hinzu und verwenden keinen Mandantenkontext.

Der Ablauf sieht so aus:

```text
PostSearchParams → BlogPostSearch → HibernateSearch.searchContext
                                      ↓
                                SearchBeanReader
                                      ↓
                         CDI predicate and sort providers
                                      ↓
                         entities(...) / count(...)
                                      ↓
                           BlogPostSummary.select
```

Der letzte Auswahl-Callback gilt für die Trefferabfrage. Die Zählabfrage verwendet dieselben Prädikate, wählt aber stattdessen einen Zähler aus.

### Wiederverwendbare Verträge definieren

Lege die folgenden Scala-Dateien unter application/backend/src/main/scala/com/anjunar/blog/hibernate/search/ an.

[AbstractSearch.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/AbstractSearch.scala) hält die Paginierung unabhängig von JAX-RS. Der Stack nennt den Zeilenoffset index; unser öffentlicher HTTP-Parameter heißt weiterhin offset. Die Eingabe und HTTP-400-Antworten bleiben in PostSearchParams.

```scala
package com.anjunar.blog.hibernate.search

// HTTP parsing belongs to the resource's input bean; index is a row offset.
abstract class AbstractSearch {
  def index: Int
  def limit: Int
}
```

[Context.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/Context.scala) wird für einen Provider-Aufruf erzeugt. Es enthält den Root dieser Abfrage, Prädikate, Parameterbindungen und zusätzliche Ausdrücke, die ein Provider beitragen muss:

```scala
package com.anjunar.blog.hibernate.search

import jakarta.persistence.criteria.{Expression, Predicate}
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.util

final case class Context[V, E](
    value: V,
    builder: HibernateCriteriaBuilder,
    predicates: util.List[Predicate],
    root: JpaRoot[E],
    query: JpaCriteriaQuery[?],
    selection: util.List[Expression[?]],
    name: String,
    parameters: util.Map[String, Any]
)
```

Ein Prädikat-Provider erhält den Wert des annotierten Feldes. Ein Sortier-Provider erhält das gesamte Suchobjekt und kann daher für eine feste Sortierung mehrere Eingaben berücksichtigen.

[PredicateProvider.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/PredicateProvider.scala):

```scala
package com.anjunar.blog.hibernate.search

trait PredicateProvider[V, E] {
  def build(context: Context[V, E]): Unit
}
```

[SortProvider.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/SortProvider.scala):

```scala
package com.anjunar.blog.hibernate.search

import jakarta.persistence.criteria.Order

import java.util

trait SortProvider[V, E] {
  def sort(context: Context[V, E]): util.List[Order]
}
```

Der Reader liefert Prädikate und Parameter für eine einzelne, neue Abfrage in [HibernateSearchContextResult.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/HibernateSearchContextResult.scala):

```scala
package com.anjunar.blog.hibernate.search

import jakarta.persistence.criteria.{Expression, Predicate}

import java.util

final case class HibernateSearchContextResult(
    selection: util.List[Expression[?]],
    predicates: util.List[Predicate],
    parameters: util.Map[String, Any]
)
```

[HibernateSearchContext.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/HibernateSearchContext.scala) verbindet die Engine mit diesen Providern, ohne von einer bestimmten Entität oder einem Suchmodell abhängig zu sein:

```scala
package com.anjunar.blog.hibernate.search

import jakarta.persistence.criteria.{Expression, Order, Predicate}
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.util

trait HibernateSearchContext {
  def apply[E](
      builder: HibernateCriteriaBuilder, query: JpaCriteriaQuery[?], root: JpaRoot[E]
  ): HibernateSearchContextResult

  def sort[E](
      builder: HibernateCriteriaBuilder, query: JpaCriteriaQuery[?], root: JpaRoot[E],
      predicates: util.List[Predicate], selection: util.List[Expression[?]]
  ): util.List[Order]
}
```

Die HTTP-Grenze prüft bereits die Seitengrenzen. Die generische Engine schützt über [QuerySurface.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/QuerySurface.scala) zusätzlich ihre internen Aufrufer:

```scala
package com.anjunar.blog.hibernate.search

object QuerySurface {
  def firstResult(index: Int): Int = {
    require(index >= 0, "Search index must be nonnegative")
    index
  }

  def maxResults(limit: Int): Int = {
    require(limit >= 1 && limit <= 100, "Search limit must be between 1 and 100")
    limit
  }
}
```

### Die Zuordnung von Annotationen zu Providern deklarieren

Lege die Java-Annotationen unter application/backend/src/main/java/com/anjunar/blog/hibernate/search/annotations/ an. Sie verweisen auf die Scala-Provider-Schnittstellen; der vorhandene gemischte Scala-/Java-Build kompiliert sie gemeinsam.

[RestPredicate.java](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/java/com/anjunar/blog/hibernate/search/annotations/RestPredicate.java) verknüpft ein Feld mit seinem Provider. Ein optionaler Name verleiht einem Parameter einen stabilen Namen, der sich vom Feldnamen unterscheiden kann:

```java
package com.anjunar.blog.hibernate.search.annotations;

import com.anjunar.blog.hibernate.search.PredicateProvider;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD})
public @interface RestPredicate {
    Class<? extends PredicateProvider<?, ?>> value();
    String name() default "";
}
```

[RestSort.java](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/java/com/anjunar/blog/hibernate/search/annotations/RestSort.java) verlangt für die Sortierregel der Suche einen ausdrücklich angegebenen Provider:

```java
package com.anjunar.blog.hibernate.search.annotations;

import com.anjunar.blog.hibernate.search.SortProvider;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD})
public @interface RestSort {
    Class<? extends SortProvider<?, ?>> value();
}
```

Der Stack kann außerdem generische Sortierung über Feldpfade. Dieses Kapitel stellt vier benannte Sortierungen bereit. Daher definiert das Suchmodell seinen eigenen Provider, statt beliebige Eigenschaftspfade zu akzeptieren.

### Metadaten lesen und CDI-Provider auflösen

[SearchBeanReader.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/SearchBeanReader.scala) verwendet den AnnotationIntrospector, der bereits von unseren veröffentlichten Abhängigkeiten bereitgestellt wird. @JsonbProperty macht ein Suchfeld für diesen Introspector sichtbar; @RestPredicate oder @RestSort legt sein Suchverhalten fest.

Der Reader löst die angegebene Provider-Klasse über CDIs Instance auf. Für neue Suchvorgänge muss keine manuelle Provider-Registry gepflegt werden. Provider verwenden @ApplicationScoped; ihr Aufrufzustand liegt im Context, niemals in Bean-Feldern.

```scala
package com.anjunar.blog.hibernate.search

import com.anjunar.blog.hibernate.search.annotations.{RestPredicate, RestSort}
import com.anjunar.scala.universe.introspector.AnnotationIntrospector
import jakarta.enterprise.inject.Instance
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.criteria.{Expression, Order, Predicate}
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.util
import scala.jdk.CollectionConverters.*

object SearchBeanReader {
  def read[E](
      search: AbstractSearch, builder: HibernateCriteriaBuilder,
      root: JpaRoot[E], query: JpaCriteriaQuery[?],
      instances: Instance[PredicateProvider[Any, E]]
  ): HibernateSearchContextResult = {
    val model = AnnotationIntrospector.createWithType(search.getClass, classOf[JsonbProperty])
    val predicates = new util.ArrayList[Predicate]()
    val selection = new util.ArrayList[Expression[?]]()
    val parameters = new util.HashMap[String, Any]()
    model.properties.foreach { property =>
      val annotation = property.findAnnotation(classOf[RestPredicate])
      if (annotation != null) {
        // A missing provider or unreadable field is a configuration error.
        // Never silently drop a predicate that could restrict public visibility.
        val provider = findProvider(instances, annotation.value())
        val value = property.get(search)
        if (value != null) {
          val name = if (annotation.name().isBlank) property.name else annotation.name()
          provider.build(Context(value, builder, predicates, root, query, selection, name, parameters))
        }
      }
    }
    HibernateSearchContextResult(selection, predicates, parameters)
  }

  def order[E](
      search: AbstractSearch, builder: HibernateCriteriaBuilder,
      root: JpaRoot[E], query: JpaCriteriaQuery[?],
      predicates: util.List[Predicate], selection: util.List[Expression[?]],
      instances: Instance[SortProvider[Any, E]]
  ): util.List[Order] = {
    val model = AnnotationIntrospector.createWithType(search.getClass, classOf[JsonbProperty])
    val properties = model.properties.filter(_.findAnnotation(classOf[RestSort]) != null).toList
    require(properties.size <= 1, "A search must declare at most one sort provider")
    properties.headOption match {
      case Some(property) =>
        val provider = findProvider(instances, property.findAnnotation(classOf[RestSort]).value())
        if (property.get(search) == null) new util.ArrayList[Order]()
        else {
          // As in the stack, sorting receives the entire search, not just the sort field.
          provider.sort(Context(search, builder, predicates, root, query, selection,
            property.name, new util.HashMap[String, Any]()))
        }
      case None => new util.ArrayList[Order]()
    }
  }

  private def findProvider[T](instances: Instance[T], providerClass: Class[?]): T = {
    val matches = instances.iterator().asScala.filter(providerClass.isInstance).toList
    if (matches.size != 1)
      throw new IllegalStateException(
        s"Expected one CDI search provider for ${providerClass.getName}, found ${matches.size}")
    matches.head
  }
}
```

Ein fehlender Provider ist auch bei einer optionalen Eingabe ein Fehler. Ebenso wird ein Fehler beim Lesen eines annotierten Feldes weitergegeben. Ein stilles Fortfahren könnte das Prädikat für den Veröffentlichungsstatus weglassen und dadurch Zeilen offenlegen, die der Endpunkt eigentlich ausschließen sollte. None ist etwas anderes: Es ist ein gültiger Wert, den der redaktionelle Status-Provider bewusst als „alle Statuswerte“ interpretiert.

Es ist nur eine Sortierdeklaration zulässig. Ihr Provider erhält das Suchobjekt, nicht die Zeichenfolge im annotierten Sortierfeld. Diese Unterscheidung ist bei SortProvider[V, E] zu beachten.

### Treffer und Zähler in einer wiederverwendbaren Engine ausführen

Lege [HibernateSearch.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/hibernate/search/HibernateSearch.scala) an:

```scala
package com.anjunar.blog.hibernate.search

import jakarta.enterprise.context.ApplicationScoped
import jakarta.enterprise.inject.Instance
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.persistence.criteria.{Expression, Order, Predicate}
import org.hibernate.query.criteria.{HibernateCriteriaBuilder, JpaCriteriaQuery, JpaRoot}

import java.lang
import java.util
import scala.compiletime.uninitialized

@ApplicationScoped
class HibernateSearch {
  @Inject var entityManager: EntityManager = uninitialized
  @Inject var predicateProviders: Instance[PredicateProvider[?, ?]] = uninitialized
  @Inject var sortProviders: Instance[SortProvider[?, ?]] = uninitialized

  def searchContext[S <: AbstractSearch](search: S): HibernateSearchContext =
    new HibernateSearchContext {
      override def apply[E](
          builder: HibernateCriteriaBuilder, query: JpaCriteriaQuery[?], root: JpaRoot[E]
      ): HibernateSearchContextResult =
        SearchBeanReader.read(search, builder, root, query,
          predicateProviders.asInstanceOf[Instance[PredicateProvider[Any, E]]])

      override def sort[E](
          builder: HibernateCriteriaBuilder, query: JpaCriteriaQuery[?], root: JpaRoot[E],
          predicates: util.List[Predicate], selection: util.List[Expression[?]]
      ): util.List[Order] =
        SearchBeanReader.order(search, builder, root, query, predicates, selection,
          sortProviders.asInstanceOf[Instance[SortProvider[Any, E]]])
    }

  def entities[E, P](
      index: Int, limit: Int, entityClass: Class[E], projection: Class[P],
      context: HibernateSearchContext,
      select: (JpaCriteriaQuery[P], JpaRoot[E], util.List[Expression[?]], HibernateCriteriaBuilder) => JpaCriteriaQuery[P]
  ): util.List[P] = {
    val start = QuerySurface.firstResult(index)
    val size = QuerySurface.maxResults(limit)
    val builder = entityManager.getCriteriaBuilder.asInstanceOf[HibernateCriteriaBuilder]
    val query = builder.createQuery(projection)
    val root = query.from(entityClass)
    val result = context(builder, query, root)
    val order = context.sort(builder, query, root, result.predicates, result.selection)
    select(query, root, result.selection, builder).where(result.predicates).orderBy(order)
    val typedQuery = entityManager.createQuery(query).setFirstResult(start).setMaxResults(size)
    result.parameters.forEach((name, value) => typedQuery.setParameter(name, value))
    typedQuery.getResultList
  }

  def count[E](entityClass: Class[E], context: HibernateSearchContext): Long = {
    val builder = entityManager.getCriteriaBuilder.asInstanceOf[HibernateCriteriaBuilder]
    val query = builder.createQuery(classOf[lang.Long])
    val root = query.from(entityClass)
    // Providers rebuild Criteria nodes against this count query's own root.
    val result = context(builder, query, root)
    query.select(builder.count()).where(result.predicates)
    val typedQuery = entityManager.createQuery(query)
    result.parameters.forEach((name, value) => typedQuery.setParameter(name, value))
    typedQuery.getSingleResult.longValue()
  }
}
```

Die Engine hängt nicht von BlogPost ab. Sie erhält die Entitätsklasse, die Ergebnisklasse und einen Projektions-Callback. Der vorhandene request-scoped EntityManager-Provider stellt über CDI den aktiven Persistenzkontext bereit. Die Erzeugung der application-scoped Engine legt weder ein zwischengespeichertes Entity-Schema an noch bindet sie eine Anfrage.

searchContext behält die unveränderliche Suche. Die Criteria-Objekte werden erst beim Aufruf von entities oder count erstellt. Deshalb kann die Zählabfrage nicht versehentlich ein an den Root der Trefferabfrage gebundenes Prädikat wiederverwenden.

Diese Übertragung behält die Provider-, Kontext- und Projektionsstruktur des Stacks bei. Abfrage-Cache-Hinweise fehlen, weil dieses Kapitel keinen Abfrage-Cache konfiguriert. Außerdem bleibt die HTTP-Auswertung im Eingabeobjekt und – wie oben beschrieben – wird ein ausdrücklicher Sortier-Provider verwendet.

### Blog-Suche und ihre Provider definieren

Lege [BlogPostSearch.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/BlogPostSearch.scala) neben dem Eingabeobjekt und der Projektion an:

```scala
package com.anjunar.blog

import com.anjunar.blog.hibernate.search.{AbstractSearch, Context, PredicateProvider, SortProvider}
import com.anjunar.blog.hibernate.search.annotations.{RestPredicate, RestSort}
import jakarta.enterprise.context.ApplicationScoped
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.criteria.Order
import jakarta.ws.rs.core.UriBuilder

import java.lang
import java.util
import java.util.Locale
import scala.annotation.meta.field
import scala.jdk.CollectionConverters.*

final case class BlogPostSearch(
    @(JsonbProperty @field) @(RestPredicate @field)(classOf[BlogPostSearch.QueryPredicate])
    query: String,
    @(JsonbProperty @field) @(RestPredicate @field)(classOf[BlogPostSearch.StatusPredicate])
    status: Option[BlogPostStatus],
    @(JsonbProperty @field) @(RestSort @field)(classOf[BlogPostSearch.PostSort])
    sort: String,
    offset: Int,
    override val limit: Int
) extends AbstractSearch {
  override def index: Int = offset

  def pageUrl(path: String, start: Int): String = {
    val uri = UriBuilder.fromPath(path).queryParam("offset", start).queryParam("limit", limit)
    // Insert raw text as a template value: literal %20 and braces must be encoded as data.
    if (query.nonEmpty) uri.queryParam("q", "{search}")
    status.foreach(value => uri.queryParam("status", value.name()))
    uri.queryParam("sort", sort)
    (if (query.nonEmpty) uri.build(query) else uri.build()).toASCIIString
  }
}

object BlogPostSearch {
  @ApplicationScoped
  class QueryPredicate extends PredicateProvider[String, BlogPost] {
    override def build(context: Context[String, BlogPost]): Unit = {
      if (context.value.nonEmpty) {
        val builder = context.builder
        val post = context.root
        val pattern = builder.parameter(classOf[String], context.name)
        context.predicates.add(builder.or(
          builder.like(builder.lower(post.get(BlogPost.schema.title)), pattern, '!'),
          builder.like(builder.lower(post.get(BlogPost.schema.slug)), pattern, '!'),
          builder.like(builder.lower(post.get(BlogPost.schema.summary)), pattern, '!')
        ))
        val literal = context.value.toLowerCase(Locale.ROOT)
          .replace("!", "!!").replace("%", "!%").replace("_", "!_")
        context.parameters.put(context.name, s"%$literal%")
      }
    }
  }

  @ApplicationScoped
  class StatusPredicate extends PredicateProvider[Option[BlogPostStatus], BlogPost] {
    override def build(context: Context[Option[BlogPostStatus], BlogPost]): Unit =
      context.value.foreach { status =>
        val parameter = context.builder.parameter(classOf[BlogPostStatus], context.name)
        context.predicates.add(context.builder.equal(context.root.get(BlogPost.schema.status), parameter))
        context.parameters.put(context.name, status)
      }
  }

  @ApplicationScoped
  class PostSort extends SortProvider[BlogPostSearch, BlogPost] {
    override def sort(context: Context[BlogPostSearch, BlogPost]): util.List[Order] = {
      val builder = context.builder
      val post = context.root
      val title = builder.lower(post.get(BlogPost.schema.title))
      val publication = post.get(BlogPost.schema.publishedAt)
      val primary = context.value.sort match {
        case "title" => Seq(builder.asc(title))
        case "title-desc" => Seq(builder.desc(title))
        case direction @ ("oldest" | "newest") =>
          // Keep drafts after dated posts in both directions.
          val undated = builder.selectCase[lang.Integer]()
            .when(builder.isNull(publication), 1).otherwise(0)
          Seq(builder.asc(undated),
            if (direction == "oldest") builder.asc(publication) else builder.desc(publication))
        case _ => throw new IllegalArgumentException("Unsupported post sort")
      }
      (primary :+ builder.asc(post.get(BlogPost.schema.id))).asJava
    }
  }
}
```

Das Modell enthält validierte Werte und legt fest, welcher Provider die jeweilige Sucheigenschaft verarbeitet. Es handelt sich um Konstruktorparameter; @field richtet die Annotationen auf ihre Hintergrundfelder. Gewöhnliche JPA-Felder im Klassenrumpf benötigen dieses Ziel nicht.

QueryPredicate, StatusPredicate und PostSort enthalten die blogspezifischen Entscheidungen. Sie öffnen weder EntityManager noch führen sie Abfragen aus. Alle Criteria-Pfade stammen aus BlogPost.schema, dessen reference-Eigenschaften die JPA-Attributschnittstellen implementieren. Es gibt weder ein zweites Metamodell noch einen beliebigen Sortierpfad aus der URL.

BlogPostSummary.select, das wir zuvor eingeführt haben, bestimmt die sieben ausgewählten Spalten. Über das Argument selection können Provider zusätzliche Ausdrücke bereitstellen; diese einfache Listenprojektion benötigt sie nicht.

### Wörtliche Textsuche über drei Felder

Das Textprädikat verbindet Titel, Slug und Zusammenfassung mit ODER. Ist ein Status angegeben, verbindet die Abfrage dieses Statusprädikat mit der ODER-Gruppe. Die Suche macht einen ODER-Zweig nicht zu einer Alternative zur öffentlichen Sichtbarkeitsbeschränkung.

QueryPredicate wandelt die Eingabe mit Locale.ROOT in Kleinbuchstaben um, maskiert sie und ergänzt außen Platzhalter für Teilstrings. Der tatsächliche Wert wird über den benannten Parameter query übermittelt.

Das Escape-Zeichen ist !:

| Eingabezeichen | Gebundener LIKE-Musterteil |
| --- | --- |
| % | !% |
| _ | !_ |
| ! | !! |

Das ! zuerst zu maskieren ist wichtig. Sonst könnten wir Zeichen maskieren, die wir selbst gerade eingefügt haben. Die Suche nach 100% soll diesen Text finden und das Prozentzeichen nicht als „beliebig viele folgende Zeichen“ behandeln.

Das Inhaltsfeld gehört nicht zu dieser Suche. So stimmt das Verhalten mit der Liste überein: Titel, Slug und Zusammenfassung sind durchsuchbar; der Artikeltext wird erst beim Öffnen der Detailansicht geladen.

### Prädikate für jeden Criteria-Root neu aufbauen

HibernateSearch.entities(...) und count(...) erzeugen separate Abfragen, jede mit einem eigenen Root. Beide rufen context.apply auf. Dadurch laufen dieselben annotierten Prädikat-Provider und binden ihre Werte. Der Kontext behält die unveränderliche Eingabe; bei jedem Aufruf werden neue Criteria-Knoten erstellt.

Der Zähler hat weder Offset noch Limit. Passen 31 Artikel und enthält eine Seite 10 davon, bleibt size gleich 31. Ein Offset hinter dem letzten Treffer liefert keine Zeilen, behält aber die gefilterte Gesamtzahl.

### Bei gleichen Werten eine stabile Reihenfolge festlegen

Jede Sortierung endet mit der Artikel-ID in aufsteigender Reihenfolge. Haben zwei Artikel denselben Titel oder Veröffentlichungszeitpunkt, erhalten sie von der Datenbank dennoch eine vollständige Reihenfolge. Aufeinanderfolgende Seiten hängen nicht von einer unbestimmten Reihenfolge gleicher Werte ab.

Bei der Datumssortierung muss außerdem festgelegt werden, was mit Entwürfen ohne Veröffentlichungszeitpunkt geschieht. Der CASE-Ausdruck ordnet undatierte Zeilen hinter datierte ein, bevor nach neuestem oder ältestem Eintrag sortiert wird. Entwürfe bleiben daher in beiden Richtungen am Ende.

Für einen gegebenen Datenbestand ist die Reihenfolge stabil. Über mehrere Anfragen hinweg wird die Liste dadurch nicht eingefroren: Neue, bearbeitete oder zurückgezogene Artikel können weiterhin zwischen Offset-Seiten wechseln.

### Rohtext in Seitenlinks erhalten

Beachte, wie BlogPostSearch.pageUrl den Suchtext als Wert einer URI-Vorlage übergibt. Diese Funktion hat zwei Ebenen der Maskierung mit unterschiedlichen Zwecken.

Auf URL-Ebene könnte jemand wörtlich nach %20 oder {query} suchen. Wird Rohtext direkt an einen URI-Vorlagen-Builder übergeben, kann er als vorhandene URL-Kodierung oder als weiterer Platzhalter interpretiert werden. Der feste Platzhalter {search} und build(query) stellen sicher, dass der Wert als Daten kodiert wird. Ein Seitenlink darf die wörtlichen Zeichen %20 nicht in ein Leerzeichen verwandeln.

Auf SQL-Ebene maskieren wir getrennt davon Zeichen, die in LIKE-Mustern eine besondere Bedeutung haben. URL-Kodierung löst dieses Problem nicht.

## 4. Die gefilterte Liste über REST zurückgeben

Ersetze die öffentliche [BlogPostsResource.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/BlogPostsResource.scala) durch diese vollständige Version:

```scala
package com.anjunar.blog

import com.anjunar.blog.hibernate.search.HibernateSearch

import jakarta.annotation.security.PermitAll

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.ws.rs.{BeanParam, GET, NotFoundException, Path, PathParam, Produces}
import jakarta.ws.rs.core.MediaType

import scala.compiletime.uninitialized
import scala.jdk.CollectionConverters.*

@PermitAll
@Path("/blog/posts")
@Produces(Array(MediaType.APPLICATION_JSON))
@RequestScoped
class BlogPostsResource {
  @Inject var links: PostLinks = uninitialized
  @Inject var queries: HibernateSearch = uninitialized
  @Inject
  var entityManager: EntityManager = uninitialized

  @GET
  @EntityGraph("BlogPost.list")
  def list(@BeanParam parameters: PostSearchParams): Table[Data[BlogPostSummary]] = {
    given EntityManager = entityManager
    val search = parameters.search(editorial = false)
    val context = queries.searchContext(search)
    val schema = Schema.forGraph(BlogPost.schema, entityManager.getEntityGraph("BlogPost.list"))
    val rows = queries.entities(search.index, search.limit, classOf[BlogPost],
      classOf[BlogPostSummary], context, BlogPostSummary.select).asScala
      .map(post => new Data(post, schema, links.summary(post, editorial = false))).toList.asJava
    val total = queries.count(classOf[BlogPost], context)
    new Table(rows, total, links.page(search, total, editorial = false))
  }

  @GET
  @Path("/{slug}")
  @EntityGraph("BlogPost.detail")
  def read(@PathParam("slug") slug: String): Data[BlogPost] = {
    given EntityManager = entityManager
    val post = BlogPost.findPublishedBySlug(slug).getOrElse(throw new NotFoundException())
    val schema = Schema.forGraph(BlogPost.schema, entityManager.getEntityGraph("BlogPost.detail"))
    new Data(post, schema, links.publicPost(post))
  }
}
```

Der Listenendpunkt akzeptiert nun das Parameterobjekt und liefert Table[Data[BlogPostSummary]] zurück. Die Detailmethode arbeitet weiterhin mit der Entität.

Der Graph beschreibt nach wie vor die Antwortfelder und liefert die passenden Schema-Metadaten. Die Criteria-Konstruktorprojektion steuert, welche Spalten die Datenbank auswählt. Das sind zusammenhängende Entscheidungen mit verschiedenen Aufgaben.

Ersetze in der bestehenden, mit ADMIN geschützten [EditorialPostsResource.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/EditorialPostsResource.scala) die Listenmethode durch Folgendes. Ergänze BeanParam in den jakarta.ws.rs-Imports und importiere com.anjunar.blog.hibernate.search.HibernateSearch. Füge neben den vorhandenen injizierten Feldern @Inject var queries: HibernateSearch = uninitialized ein. Detail- und Befehlsmethoden bleiben erhalten.

```scala
  @GET
  @EntityGraph("BlogPost.list")
  def list(@BeanParam parameters: PostSearchParams): Table[Data[BlogPostSummary]] = {
    given EntityManager = manager
    val search = parameters.search(editorial = true)
    val context = queries.searchContext(search)
    val schema = Schema.forGraph(BlogPost.schema, manager.getEntityGraph("BlogPost.list"))
    val rows = queries.entities(search.index, search.limit, classOf[BlogPost],
      classOf[BlogPostSummary], context, BlogPostSummary.select).asScala
      .map(post => new Data(post, schema, links.summary(post, editorial = true))).asJava
    val total = queries.count(classOf[BlogPost], context)
    new Table(rows, total, links.page(search, total, editorial = true))
  }
```

Der entscheidende Unterschied ist editorial = true. Der Aufrufer hat die @RolesAllowed(Array("ADMIN"))-Richtlinie der Ressource bereits bestanden, und die validierte Suche darf Entwürfe umfassen.

Die früher getrennten Methoden BlogPost.listPublished und countPublished entfallen. Beide Listenendpunkte verwenden nun denselben HibernateSearch-Kontext. Wird ein Prädikat ergänzt, aktualisiert das somit beide Abfragewege.

### Filter in Navigationslinks beibehalten

In [PostLinks.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/main/scala/com/anjunar/blog/PostLinks.scala) verwendet die Suche des Sammlungsendpunkts für den Parametertyp der Listenmethode jetzt classOf[PostSearchParams]. So bleibt die bestehende, richtlinienbasierte Ermittlung von Sitzungslinks mit der neuen Signatur synchron.

Die folgenden Methoden in dieser Klasse erstellen Zusammenfassungs- und Seitenlinks:

```scala
  def summary(post: BlogPostSummary, editorial: Boolean): util.List[Link] = {
    val path = if (editorial) s"/service/editorial/posts/${post.id}"
      else s"/service/blog/posts/${post.slug}"
    val values = new util.ArrayList[Link]()
    values.add(new Link("self", path, "GET", "BlogPost"))
    if (editorial && endpoint("update"))
      values.add(new Link("update", path, "PATCH", "BlogPost"))
    if (!editorial && endpoint("read"))
      values.add(new Link("preview", s"/service/editorial/posts/${post.id}", "GET", "BlogPost"))
    // Publication capabilities need the loaded body and are advertised by detail responses.
    values
  }

  def page(search: BlogPostSearch, total: Long, editorial: Boolean): util.List[Link] = {
    val path = if (editorial) "/service/editorial/posts" else "/service/blog/posts"
    def link(rel: String, start: Int) =
      new Link(rel, search.pageUrl(path, start), "GET", "BlogPost")
    val values = Seq(Some(link("self", search.offset)),
      Option.when(editorial && endpoint("create"))(new Link("create", path, "POST", "BlogPost")),
      Option.when(search.offset > 0)(link("first", 0)),
      Option.when(search.offset > 0)(link("previous", math.max(0, search.offset - search.limit))),
      Option.when(search.offset.toLong + search.limit < total && search.offset <= Int.MaxValue - search.limit)(
        link("next", search.offset + search.limit)))
    values.flatten.asJava
  }
```

Die vorhandenen Imports liefern bereits Link, java.util und die Scala-Collection-Konverter. Alle Seitenlinks verwenden dieselbe validierte Suche und ändern nur den Offset. Ein Link zur ersten Seite ist besonders hilfreich, wenn Löschungen oder ein gespeicherter tiefer Offset zu einer leeren Seite führen.

Eine Zeile der redaktionellen Liste kann Lese- und Bearbeitungsmöglichkeiten angeben, ohne den Inhalt zu laden. Veröffentlichung ist anders: canPublish muss den Inhalt prüfen. Diese Fähigkeiten bleiben auf der Detailantwort, die beim Öffnen der Vorschau geladen wird.

## 5. Die geladene Suche durch die Browser-URL bestimmen

Das Backend versteht die Anfrage. Als Nächstes braucht der Browser ein kleines Modell für die Parameter, die seine aktuelle Liste ergeben haben.

Lege das Frontendmodell [PostSearch.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostSearch.scala) unter application/frontend/src/main/scala/com/anjunar/blog/frontend/ an:

```scala
package com.anjunar.blog.frontend

import scala.scalajs.js.URIUtils.encodeURIComponent

final case class PostSearch(query: String = "", status: String = "", sort: String = "newest",
    offset: Int = 0, limit: Int = 20, editorial: Boolean = false) {
  def path: String = if (editorial) "/editorial" else "/"
  def filtered: Boolean = query.nonEmpty || status.nonEmpty

  def queryString(includeDefaults: Boolean = false): String = {
    val defaultSort = if (editorial) "title" else "newest"
    val values = Seq(
      Option.when(includeDefaults || offset > 0)("offset" -> offset.toString),
      Option.when(includeDefaults || limit != 20)("limit" -> limit.toString),
      Option.when(query.nonEmpty)("q" -> query),
      Option.when(status.nonEmpty)("status" -> status),
      Option.when(sort != defaultSort)("sort" -> sort)
    ).flatten
    values.map((name, value) => s"$name=${encodeURIComponent(value)}").mkString("&")
  }

  def url: String = {
    val query = queryString()
    if (query.isEmpty) path else s"$path?$query"
  }
}

object PostSearch {
  def parse(read: String => Option[String], editorial: Boolean): PostSearch = {
    val query = read("q").getOrElse("").trim
    val status = read("status").getOrElse("")
    val sort = read("sort").filter(_.nonEmpty).getOrElse(if (editorial) "title" else "newest")
    val offset = read("offset").getOrElse("0").toIntOption.filter(_ >= 0)
    val limit = read("limit").getOrElse("20").toIntOption.filter(value => value >= 1 && value <= 100)
    val invalidText = query.length > 100 || query.exists(c => c < ' ' || (c >= 127 && c <= 159))
    val statuses = if (editorial) Set("", "DRAFT", "PUBLISHED") else Set("", "PUBLISHED")
    if (invalidText || !statuses.contains(status) ||
        !Set("newest", "oldest", "title", "title-desc").contains(sort) || offset.isEmpty || limit.isEmpty)
      throw new HttpFailure(400)
    PostSearch(query, status, sort, offset.get, limit.get, editorial)
  }
}
```

Diese Klasse bildet den Routenstatus ab und ist vom Entitätsmodell getrennt. Sie prüft dieselben öffentlich geltenden Grenzen und Sortiernamen, bevor eine Listenanfrage gesendet wird. Das Backend führt seine eigenen maßgeblichen Prüfungen weiterhin durch.

Browser-URLs lassen Standardwerte nach Möglichkeit aus. API-Anfragen übermitteln Offset und Limit ausdrücklich. encodeURIComponent kodiert jeden Abfragewert einzeln; wir kodieren nicht die gesamte URL als eine Zeichenfolge.

Beim Seitenwechsel wird eine Kopie der aktuellen Suche mit einem anderen Offset erstellt. Abfrage, Status, Sortierung und Seitengröße bleiben darin erhalten.

Der öffentliche [BlogService.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogService.scala) nimmt jetzt diesen Wert entgegen:

```scala
package com.anjunar.blog.frontend

import org.scalajs.dom

import scala.concurrent.{ExecutionContext, Future}
import scala.scalajs.js.URIUtils.encodeURIComponent

final class BlogService(using ExecutionContext) {
  def list(search: PostSearch, signal: Option[dom.AbortSignal]): Future[BlogPostTable] =
    HttpJson.get[BlogPostTable](s"/service/blog/posts?${search.queryString(includeDefaults = true)}", signal)
      .map { table =>
        require(table.size >= 0 && table.rows != null, "Invalid post table")
        table.rows.foreach(row => require(row != null && row.data != null, "Missing post data"))
        table
      }

  def detail(slug: String, signal: Option[dom.AbortSignal]): Future[BlogPost] =
    HttpJson.get[BlogPostData](s"/service/blog/posts/${encodeURIComponent(slug)}", signal)
      .map { result =>
        require(result.data != null && result.data.content.get != null, "Missing post detail")
        result.data
      }
}
```

Der redaktionelle Dienst folgt weiterhin dem redaktionellen Sitzungslink, setzt die Abfrage des validierten Ziels auf search.queryString(includeDefaults = true) und sendet das AbortSignal der Route. Sein Loader für neue Artikel verwendet eine standardmäßige redaktionelle Suche mit einer Zeile, um die Erstellungsberechtigung für die Sammlung abzurufen.

In [BlogRoutes.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogRoutes.scala) werden die beiden Listen-Loader zu:

```scala
    Route.view("/editorial") { context =>
      val search = PostSearch.parse(context.queryParams.get, editorial = true)
      editorial.list(search, context.signal).map(new EditorialListPage(_, search))
    },
```

```scala
    Route.view("/") { context =>
      val search = PostSearch.parse(context.queryParams.get, editorial = false)
      service.list(search, context.signal)
        .map(table => new PostListPage(table, search, actions))
    },
```

Diese Einträge gehören in die bestehende Routensequenz. Dienste, Fehlerrouten und Imports bleiben erhalten. Eine ungültige Suche führt zu einem Routenfehler 400 und verwendet die vorhandene Fehlerseite. Eine neue Navigation bricht den vorherigen Ladevorgang ab, sodass ein verspätetes Ergebnis die neue Route nicht ersetzen kann.

## 6. Ein Suchformular für beide Listen erstellen

Die gerade angezeigten Ergebnisse gehören zu einem unveränderlichen PostSearch. Nutzer können vor dem Absenden die Steuerelemente ändern; diese ausstehenden Werte brauchen deshalb einen eigenen Formularzustand.

Lege [PostSearchForm.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostSearchForm.scala) mit der vollständigen Implementierung an:

```scala
package com.anjunar.blog.frontend

import ui.core.component.AbstractComponent
import ui.core.dsl.AttributeDsl
import ui.core.dsl.AttributeDsl.*
import ui.core.dsl.ClassDsl.classes
import ui.core.dsl.DslLayer.render
import ui.core.dsl.EventDsl.on
import ui.core.i18n.{I18nRuntime, i18n}
import ui.core.layout.Button.{button, buttonType}
import ui.core.layout.Condition.when
import ui.core.layout.Div.div
import ui.core.layout.Label.label
import ui.core.layout.Paragraph.paragraph
import ui.core.layout.TextComponent.text
import ui.core.render.Cursor
import ui.core.state.Property
import ui.forms.Form.form
import ui.forms.Input.{input, inputType, inputType_=}
import ui.forms.SelectInput.selectInput
import ui.forms.SelectOption
import ui.forms.validators.Size
import ui.router.Router
import ui.router.RouterLink.routerLink

import scala.annotation.meta.field
import scala.scalajs.js

// Search controls are route state, separate from the editable post model.
final class PostSearchFields(search: PostSearch) {
  @(Size @field)(max = 100)
  val query: Property[String] = Property(search.query)
  val status: Property[String] = Property(search.status)
  val sort: Property[String] = Property(search.sort)
  val limit: Property[String] = Property(search.limit.toString)

  def submitted(editorial: Boolean): PostSearch = {
    val values = Map("q" -> query.get, "status" -> status.get, "sort" -> sort.get, "limit" -> limit.get)
    // A new search always starts at offset zero.
    PostSearch.parse(values.get, editorial)
  }
}

final class PostSearchForm(search: PostSearch) extends AbstractComponent {
  val tagName = "div"
  private val fields = new PostSearchFields(search)
  private val invalid = Property(false)

  override def compose(cursor: Cursor): Unit = {
    val translations = I18nRuntime.current(using this).get
    render(this, cursor) {
      classes = "post-search"
      form(fields) { mountedForm ?=>
        classes = "search-form"
        role = "search"
        AttributeDsl.setAttribute("novalidate", "")
        on("submit") { event =>
          event.preventDefault()
          invalid.set(false)
          val bindings = mountedForm.validateBindings()
          val errors = mountedForm.validate()
          if (bindings.nonEmpty || errors.nonEmpty) invalid.set(true)
          else {
            try Router.navigate(fields.submitted(search.editorial).url)
            catch { case _: HttpFailure => invalid.set(true) }
          }
        }
        div {
          classes = "search-query"
          label {
            AttributeDsl.setAttribute("for", "post-query")
            text(i18n"Search posts") {}
          }
          val control = input("query") { fieldInput ?=>
            id = "post-query"
            inputType = "search"
            AttributeDsl.setAttribute("maxlength", "100")
            AttributeDsl.setAttribute("aria-describedby", "search-help search-errors")
            fieldInput.addDisposable(fieldInput.invalid.observe(value =>
              AttributeDsl.setAttribute("aria-invalid", value.toString)))
          }
          paragraph {
            id = "search-help"
            classes = "field-help"
            text(i18n"Search title, slug or summary.") {}
          }
          paragraph {
            id = "search-errors"
            classes = "field-error"
            text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {}
          }
        }
        if (search.editorial) {
          div {
            label {
              AttributeDsl.setAttribute("for", "post-status")
              text(i18n"Publication status") {}
            }
            selectInput("status", Seq(
              SelectOption("", translations.text(i18n"All statuses")),
              SelectOption("DRAFT", translations.text(i18n"Draft")),
              SelectOption("PUBLISHED", translations.text(i18n"Published"))
            )) { id = "post-status" }
          }
        }
        div {
          label {
            AttributeDsl.setAttribute("for", "post-sort")
            text(i18n"Sort by") {}
          }
          selectInput("sort", Seq(
            SelectOption("newest", translations.text(i18n"Newest first")),
            SelectOption("oldest", translations.text(i18n"Oldest first")),
            SelectOption("title", translations.text(i18n"Title A–Z")),
            SelectOption("title-desc", translations.text(i18n"Title Z–A"))
          )) { id = "post-sort" }
        }
        div {
          label {
            AttributeDsl.setAttribute("for", "post-limit")
            text(i18n"Posts per page") {}
          }
          selectInput("limit", (Seq(10, 20, 50, 100) :+ search.limit).distinct.sorted.map(value =>
            SelectOption(value.toString, Property(value.toString)))) { id = "post-limit" }
        }
        div {
          classes = "search-buttons"
          button(i18n"Search") { buttonType("submit") }
          routerLink(search.path) { text(i18n"Reset search") {} }
        }
        when(invalid) {
          paragraph { role = "alert"; text(i18n"Check the search fields and try again.") {} }
        }
      }
    }
  }
}
```

PostSearchFields enthält nur ausstehende Suchwerte und keine Kopien von Entitätsfeldern. Die Methode submitted lässt den Offset absichtlich aus; dadurch beginnt die nächste Suche wieder bei null.

Attributänderungen verwenden AttributeDsl.setAttribute. AttributeDsl wird aus ui.core.dsl importiert. So wird der geerbte Komponenten-Setter mit demselben Namen vermieden und das vom umgebenden DSL-Block bereitgestellte Element angesprochen. Der reaktive aria-invalid-Beobachter bleibt im Eingabeblock und wird zusammen mit diesem Eingabefeld entfernt. Label-Verknüpfungen, Maximallänge und novalidate verwenden denselben DSL-Weg.

Der native Submit-Handler wird über on("submit") registriert. Vor der Navigation validiert er die Bindung und die Textbegrenzung. Tippen allein löst keine Anfrage aus und benennt das gerade angezeigte Ergebnis nicht um. **Search** oder Enter übernimmt die neuen Werte gemeinsam.

Die Auswahlfelder binden wie das Suchfeld Zeichenfolgen. Der Status wird nur in der redaktionellen Ansicht gezeigt. Die Seitengrößen enthalten normalerweise 10, 20, 50 und 100; eine gültige Größe aus einer direkten URL wird ebenfalls ergänzt. Dadurch kann ein Test mit einer Zeile oder ein geteilter Link sein Steuerelement korrekt wiederherstellen.

Der gesamte Formularbaum bleibt in compose: Beschriftungen, Absenden, Optionen und Validierungshinweise. Es ist eine gemeinsame Komponente, weil das Formular auf beiden Listenseiten eine vollständige Funktion hat. Neue sichtbare Nachrichten verwenden das i18n-Makro; die numerischen Seitengrößen brauchen keine Übersetzung.

### Das Formular einbinden und den Zustand beim Blättern erhalten

[PostListPage](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostListPage.scala) und [EditorialListPage](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/scala/com/anjunar/blog/frontend/EditorialListPage.scala) erhalten zusammen mit der Tabelle die geladene Suche. Vor den Ergebnissen binden sie child(new PostSearchForm(search)) {} ein.

Die Links „Vorherige Seite“ und „Nächste Seite“ der öffentlichen Liste verwenden eine Kopie dieser Suche mit neuem Offset. Die redaktionellen Links verwenden die vom Server angegebenen vorherigen und nächsten URLs, nachdem die vorhandene API-Link-Prüfung sie validiert hat. Die Beschriftungen lauten **Vorherige Seite** und **Nächste Seite**, denn bei einer Titelsortierung gibt es keine sinnvolle Richtung „älter“.

Gibt es keine Treffer, meldet die Seite das. Gibt es Treffer, aber der Offset liegt hinter ihnen, bleiben Gesamtzahl und Filter erhalten; zusätzlich wird **Erste Seite mit Treffern** angeboten. **Suche zurücksetzen** führt dagegen zur unveränderten Listenroute zurück und stellt deren Standardwerte wieder her.

Neuladen und Browserverlauf laden den Zustand erneut aus der URL. Dadurch werden Abfrage und Steuerelemente wiederhergestellt. Wir führen keinen zweiten, verborgenen Filterzustand, der von den angezeigten Ergebnissen abweichen könnte.

Der Checkpoint enthält außerdem [responsive Formularstile](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/main/resources/style.css).

## 7. Routenverhalten und echte Abfrageergebnisse prüfen

Hier ist die vollständige Frontend-Datei [PostSearchSpec.scala](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/frontend/src/test/scala/com/anjunar/blog/frontend/PostSearchSpec.scala) unter application/frontend/src/test/scala/com/anjunar/blog/frontend/:

```scala
package com.anjunar.blog.frontend

import org.scalatest.funsuite.AnyFunSuite

class PostSearchSpec extends AnyFunSuite {
  private def parse(values: (String, String)*)(editorial: Boolean = false): PostSearch =
    PostSearch.parse(values.toMap.get, editorial)

  test("public and editorial defaults keep their existing ordering") {
    assert(parse()() == PostSearch())
    assert(parse()(editorial = true).sort == "title")
    assert(parse()().url == "/")
    assert(parse()(editorial = true).url == "/editorial")
  }

  test("filter values are encoded once and survive a page change") {
    val value = parse("q" -> "  100%_! & + café  ", "sort" -> "title-desc", "limit" -> "10")()
    assert(value.query == "100%_! & + café")
    val next = value.copy(offset = 10)
    assert(next.url.startsWith("/?offset=10&limit=10&q=100%25_!%20%26%20%2B%20caf%C3%A9"))
    assert(next.url.endsWith("&sort=title-desc"))
    assert(next.copy(offset = 0).query == value.query)
  }

  test("submitting changed filters resets the page without changing the current route state") {
    val current = PostSearch("old", "DRAFT", "title", 40, 10, editorial = true)
    val form = new PostSearchFields(current)
    form.query.set("new")
    form.status.set("PUBLISHED")
    val submitted = form.submitted(editorial = true)
    assert(submitted.offset == 0 && submitted.limit == 10 && submitted.query == "new")
    assert(submitted.status == "PUBLISHED" && submitted.sort == "title")
    assert(current.offset == 40 && current.query == "old")
  }

  test("malformed routes fail before issuing an HTTP request") {
    for ((key, value) <- Seq("offset" -> "-1", "offset" -> "2147483648", "limit" -> "0",
      "limit" -> "101", "sort" -> "content", "status" -> "DRAFT", "q" -> ("a" * 101), "q" -> "a\u0000b")) {
      assert(intercept[HttpFailure](parse(key -> value)()).status == 400)
    }
    assert(parse("status" -> "DRAFT")(editorial = true).status == "DRAFT")
  }

  test("API requests include bounds while default browser URLs stay short") {
    assert(PostSearch().queryString(includeDefaults = true) == "offset=0&limit=20")
    assert(PostSearch(offset = 20).url == "/?offset=20")
    assert(parse("limit" -> "1")().limit == 1)
  }
}
```

Die Tests prüfen verschiedene Teile des Vertrags: Standardsortierung, kodierte Filterwerte, erhaltene Seitengröße, Zurücksetzen des Offsets und Ablehnung vor einer HTTP-Anfrage. Der Test der übermittelten Filter bestätigt auch, dass Änderungen an den ausstehenden Steuerelementen die Suche für die aktuellen Ergebnisse nicht verändern.

Das Datenbankverhalten braucht Datenbanktests. Das Backend [PostSearchRestSpec](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/test/scala/com/anjunar/blog/PostSearchRestSpec.scala) fügt eigene Artikel ein und prüft die echten Endpunkte. Es deckt Folgendes ab:

- Treffer in Titel, Slug und Zusammenfassung mit derselben gefilterten Trefferzahl.
- Wörtliche Prozentzeichen, Unterstriche und Escape-Zeichen.
- Hin- und Rückwege von Seitenlinks mit Unicode, %20 und geschweiften Klammern.
- Reihenfolge gleicher Schlüssel, undatierte Entwürfe, leere Seiten und Ganzzahlgrenzen.
- Öffentliche Suchen nur für veröffentlichte Artikel, auch bei Administratoren.
- Ablehnung von nicht angemeldeten Lesern beim Zugriff auf redaktionelle Suchen.

Der [echte Browserablauf](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/tests/browser/post-search-database.spec.mjs) verwendet anschließend die tatsächlichen Steuerelemente mit PostgreSQL. Er sucht, wechselt Seiten, lädt neu, findet wörtliche Satzzeichen, meldet sich an und zeigt in der Redaktion nur Entwürfe. Die Testzeilen werden in einem finally-Block entfernt.

Führe die Prüfungen mit der isolierten Datenbank, dem Testadministrator und den SMTP-Aufzeichnungseinstellungen der vorherigen Kapitel aus:

```text
sbt --server "application-backend/testFull" "application-frontend/testFull" frontendAssets
npx playwright test --project=contracts
npx playwright test --project=search --project=forms --project=changes --project=editorial --project=database --project=authentication --project=recovery
```

Für das Such-Browserprojekt muss psql im PATH liegen oder BLOG_PSQL gesetzt sein. Außerdem werden BLOG_TEST_ADMIN_EMAIL und BLOG_TEST_ADMIN_PASSWORD benötigt. Halte Port 18080 frei und führe Backend- und Browsersuiten nacheinander aus.

Der zusätzliche [HibernateSearchSpec](https://github.com/anjunar/anjunar-blog-example/blob/214eeb6d5a2c6062013d95abf7f990a1cb032cce/application/backend/src/test/scala/com/anjunar/blog/HibernateSearchSpec.scala) läuft mit echtem CDI und PostgreSQL. Er deklariert eine unabhängige ById-Suche samt Provider, ohne die Engine zu ändern, wählt Entitäten und skalare Titel aus und prüft passende Trefferzahlen. Außerdem wird geprüft, dass ein fehlender Prädikat- oder Sortier-Provider einen Fehler auslöst und interne Aufrufer die Seitengrenzen nicht umgehen können.

Der fertige Checkpoint besteht 125 Backend-Tests, 26 Scala.js-Tests und 56 Browsertests: 48 kontrollierte Verträge und acht reale Abläufe. Die Browserprüfungen decken außerdem Zurück/Vor, Zurücksetzen, leere Ergebnisse, mobile Darstellung und eine langsame Suche ab, deren verspätetes Ergebnis keine spätere Navigation überschreiben darf.

## Was diese Suche leistet

Es handelt sich um eine wörtliche Teilstringsuche über drei Metadatenfelder. Sie bietet weder Volltext-Ranking noch eine Suche im Artikeltext oder eine akzentunabhängige Suche. Ein LIKE-Ausdruck mit vorangestelltem Platzhalter kann viele Zeilen durchsuchen. Bei einem größeren Blog sollte man daher die Abfragen messen, bevor man einen Index oder eine andere Suchstrategie auswählt.

Treffer und Gesamtzahl sind getrennte Anweisungen unter der vorhandenen Transaktionsisolation READ COMMITTED. Gleichzeitige Änderungen können sie unterschiedlich beeinflussen. Eine deterministische Sortierung macht Offset-Seiten ebenfalls nicht zu einem Snapshot. Sehr große Offsets können trotz begrenzter Seitengröße weiterhin teuer sein.

Innerhalb dieser Grenzen steht nun ein vollständiger Suchablauf bereit: validierte Parameter, typisierte Prädikate, eine bewusst kleine Projektion, gefilterte Trefferzahlen und eine URL-gesteuerte Oberfläche. Im nächsten Kapitel kommen Autoren, Tags und sichere Entitätsreferenzen hinzu. Medien-Uploads folgen separat in Kapitel 18.

