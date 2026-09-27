Wir haben jetzt einen Server, eine Datenbankverbindung und eine Transaktionsgrenze. Dieses Kapitel gibt ihnen etwas Nützliches zu speichern: `BlogPost`.

Am Ende können wir einen Entwurf speichern, ihn in einen neuen Persistenzkontext laden, ihn veröffentlichen und eine Bearbeitung basierend auf einer alten Version ablehnen. Diese Verhaltensweisen werden gegen PostgreSQL verifiziert. Öffentliche Postendpunkte kommen in Kapitel 8.

Beginnen Sie mit dem [Kapitel 4 Projekt](https://github.com/anjunar/anjunar-blog-example/tree/e96365e906bb14b212fe2b0b9e11664f6b5afd93) und seinem lokalen PostgreSQL-Setup. Alle Pfade unten sind relativ zum Projektverzeichnis.

## Definieren Sie den ersten Post

Unser erstes Modell hat sieben Felder:

|Feld|Zweck|
| --- | --- |
|`id`|Generierte UUID, die den Post während seiner gesamten Lebensdauer identifiziert.|
|`version`|Wert, mit dem Hibernate Änderungen auf Basis veralteter Daten erkennt.|
|`slug`|Einzigartige, lesbare Kennung für die zukünftige öffentliche URL.|
|`title`|Erforderlicher Titel, zwischen 3 und 180 Zeichen.|
|`content`|Textinhalt, bis zu 100.000 Zeichen; ein Entwurf kann es leer lassen.|
|`status`|`DRAFT` oder `PUBLISHED`.|
|`publishedAt`|Die aktuelle Veröffentlichungszeit, abwesend für einen Entwurf.|

Ein Titel kann sich ändern, während die Slug gleich bleibt. Dies verhindert, dass eine redaktionelle Korrektur automatisch die zukünftige URL des Beitrags ändert. Die UUID bleibt die Identität, die zum Laden und Referenzieren der Entität verwendet wird.

Wir beginnen mit englischem Text in der Entität. Autorenbeziehungen, strukturierte redaktionelle Inhalte und Übersetzungen erweitern das Modell in ihren jeweiligen Kapiteln.

## Bean Validation hinzufügen

Fügen Sie diese Abhängigkeiten zur vorhandenen Sequenz in `build.sbt` hinzu:

```scala
"org.hibernate.validator" % "hibernate-validator" % "9.1.4.Final",
"org.glassfish.expressly" % "expressly" % "6.0.0",
```

Hibernate Validator implementiert Jakarta Validation. Die Implementierung der Expression Language wird ausdrücklich eingebunden, weil sie für die Interpolation der Standardmeldungen benötigt wird. Die [Dokumentation zu Hibernate Validator](https://hibernate.org/validator/documentation/) deckt diese Abhängigkeiten und die Einschränkungsvalidierung ab.

Die Annotationen beschreiben, welche Feldwerte gültig sind. Wir werden auch die Validierung während des Persistenzlebenszyklus von Hibernate ermöglichen, so dass die Regeln vor Inserts und Updates ausgeführt werden.

## Den Veröffentlichungsstatus abbilden

Erstellen Sie `application/backend/src/main/java/com/anjunar/blog/BlogPostStatus.java`:

```java
package com.anjunar.blog;

public enum BlogPostStatus {
    DRAFT,
    PUBLISHED
}
```

Dieses kleine Java-Enum arbeitet direkt mit dem Enum-Mapping von JPA. sbt kompiliert es zusammen mit den Scala-Quellen.

## Die Entität abbilden

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala`:

```scala
package com.anjunar.blog

import jakarta.persistence.{Access, AccessType, Column, Entity, Enumerated, EnumType, GeneratedValue, GenerationType, Id, Table, Transient, UniqueConstraint, Version}
import jakarta.validation.constraints.{AssertTrue, NotBlank, NotNull, Pattern, Size}

import java.lang
import java.time.Instant
import java.util.UUID

@Entity
@Access(AccessType.FIELD)
@Table(name = "blog_post", schema = "public",
  uniqueConstraints = Array(new UniqueConstraint(name = "uq_blog_post_slug", columnNames = Array("slug"))))
class BlogPost {
  @Id
  @GeneratedValue(strategy = GenerationType.UUID)
  @Column(nullable = false, updatable = false)
  var id: UUID = null

  @Version
  @Column(nullable = false)
  var version: lang.Long = null

  @NotBlank
  @Size(min = 3, max = 220)
  @Pattern(regexp = "^[a-z0-9]+(?:-[a-z0-9]+)*$")
  @Column(nullable = false, length = 220)
  var slug: String = ""

  @NotBlank
  @Size(min = 3, max = 180)
  @Column(nullable = false, length = 180)
  var title: String = ""

  @NotNull
  @Size(max = 100000)
  @Column(nullable = false, columnDefinition = "text")
  var content: String = ""

  @NotNull
  @Enumerated(EnumType.STRING)
  @Column(nullable = false, length = 24)
  var status: BlogPostStatus = BlogPostStatus.DRAFT

  @Column(name = "published_at")
  var publishedAt: Instant = null

  def publish(at: Instant): Unit = {
    require(status == BlogPostStatus.DRAFT, "Only a draft can be published")
    require(at != null, "Publication time is required")
    require(content != null && !content.isBlank, "A published post needs content")
    status = BlogPostStatus.PUBLISHED
    publishedAt = at
  }

  def retract(): Unit = {
    require(status == BlogPostStatus.PUBLISHED, "Only a published post can be retracted")
    status = BlogPostStatus.DRAFT
    publishedAt = null
  }

  @Transient
  @AssertTrue(message = "Publication status, time, and content must be consistent")
  def isPublicationConsistent: Boolean =
    status match {
      case BlogPostStatus.DRAFT => publishedAt == null
      case BlogPostStatus.PUBLISHED => publishedAt != null && content != null && !content.isBlank
      case null => false
    }
}
```

Eine reguläre Klasse stellt den parameterlosen Konstruktor bereit, den Hibernate benötigt. `@Access(AccessType.FIELD)` sagt Hibernate, Felder direkt zu lesen und zu schreiben. Unsere `var`-Deklarationen befinden sich im Klassenkörper, so dass die Annotationen zu JPA und Bean Validation bereits auf ihre Felder abzielen; es wird kein `@field` benötigt.

Konstruktorparameter sind eine andere Annotationsstelle: Verwenden Sie `@field`, wenn eine Annotation zu einem Konstruktorparameter auf dem Backing-Feld platziert werden muss. Die [Scala-Dokumentation zu Annotationszielen](https://www.scala-lang.org/api/3.x/scala/annotation/meta.html) erklärt diese Standardwerte. Dieses Compiler-Targeting ist von der Feldzugriffsstrategie der JPA getrennt.

Die generierte UUID und Version starten als `null`. Hibernate vergibt sie beim Persistieren der Entität. Wir verwenden `import java.lang` und `lang.Long` für den nullbaren Java-Wrapper; Hibernate besitzt den Versionswert und seine Inkremente. Die [Jakarta-Persistence-Spezifikation](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2) definiert generierte UUID-Identifikatoren und Versionsattribute.

`EnumType.STRING` speichert Statusnamen wie `DRAFT` und `PUBLISHED`. `Instant` stellt die Veröffentlichungszeit unabhängig von der Zeitzone eines Lesers dar.

Der Titel und der Slug müssen sogar für einen Entwurf vorhanden sein. Die Slug akzeptiert Kleinbuchstaben, Ziffern und einzelne Trennstriche mit einer Länge von 3 bis 220 Zeichen. Ein neu konstruiertes Objekt benötigt daher einen Titel und einen Slug, bevor es gültig ist.

## Publikationsänderungen konsistent halten

Die Veröffentlichung ändert zwei Felder zusammen: Status wird zu `PUBLISHED` und `publishedAt` erhält den bereitgestellten Moment. Dafür ist ein nicht leerer Inhalt erforderlich. Das Zurückziehen stellt `DRAFT` wieder her und löscht die aktuelle Veröffentlichungszeit unter Beibehaltung des Textes.

Die Methoden überprüfen ihre Voraussetzungen, bevor sie ein Feld ändern. Der erneute Aufruf von `publish` in einem veröffentlichten Beitrag lässt daher die Veröffentlichungszeit unverändert und meldet einen ungültigen Übergang.

Die Felder bleiben veränderlich, weil sie unser Persistenzmodell bilden. `isPublicationConsistent` überprüft ihre Beziehung auch dann, wenn ein anderer Anrufer die Felder direkt zuweist. Sein Name im JavaBean-Stil macht es zu einer Validierungseigenschaft; `@AssertTrue` erfordert, dass sein Ergebnis wahr ist. Es wird nicht als Spalte gespeichert.

Diese Kontrollen dienen verschiedenen Zwecken:

|Mechanismus|Was sie festlegt|
| --- | --- |
|`publish` und `retract`|Ein expliziter Zustandsübergang mit Voraussetzungen.|
|Feldbeschränkungen|Erforderliche Werte, Längen und Slug-Format.|
|`isPublicationConsistent`|Der resultierende Publikationszustand ist kohärent.|
|Einschränkungen der Datenbank|Slugs und gültige gespeicherte Status/Zeit-Kombinationen.|

Der Aufruf einer Übergangsmethode führt nicht jede Feldeinschränkung sofort aus. Die Validierungs-Callbacks von Hibernate überprüfen die gesamte Entität, bevor sie sie schreiben.

## Erstellen Sie die erste Tabelle explizit

Erstellen Sie `database/001-blog-post.sql`:

```sql
CREATE TABLE public.blog_post (
    id uuid PRIMARY KEY,
    version bigint NOT NULL,
    slug varchar(220) NOT NULL,
    title varchar(180) NOT NULL,
    content text NOT NULL,
    status varchar(24) NOT NULL,
    published_at timestamp(6) with time zone,
    CONSTRAINT uq_blog_post_slug UNIQUE (slug),
    CONSTRAINT ck_blog_post_status CHECK (status IN ('DRAFT', 'PUBLISHED')),
    CONSTRAINT ck_blog_post_publication CHECK (
        (status = 'DRAFT' AND published_at IS NULL)
        OR (status = 'PUBLISHED' AND published_at IS NOT NULL)
    )
);
```

Die Unique Constraint erzwingt eindeutige Slugs, auch bei gleichzeitigen Inserts. Die Status- und Publikationsprüfungen gelten auch für SQL, das außerhalb von Hibernate ausgeführt wird. Inhalts- und Titelvalidierung bleiben im Modell; diese Mechanismen kopieren nicht automatisch alle ihre Regeln ineinander.

Wenn die Compose-Datenbank aus Kapitel 4 ausgeführt wird, wenden Sie das Skript einmal an:

```text
docker compose cp database/001-blog-post.sql postgres:/tmp/001-blog-post.sql
docker compose exec -T postgres psql -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --single-transaction --file /tmp/001-blog-post.sql
```

Diese Befehle funktionieren sowohl in PowerShell als auch in Bash. Führen Sie für native PostgreSQL den `psql`-Client mit Ihren Verbindungsdetails aus:

```text
psql -h 127.0.0.1 -p 5433 -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --single-transaction --file database/001-blog-post.sql
```

Ändern Sie den Port, wenn nötig. Der native Client fordert bei Bedarf das Datenbankpasswort auf.

Das Skript erwartet eine Datenbank ohne diese Tabelle und meldet einen Fehler, wenn sie bereits vorhanden ist. Bewahren Sie vorhandene Daten auf und wenden Sie das Skript nur einmal an. Wir werden verfolgte Schemamigrationen im nächsten Kapitel vorstellen.

## Entitätsklassen durch CDI entdecken

Die Persistenzschicht sollte eine andere Entität akzeptieren, ohne dass eine Bearbeitung des Bootstrap-Codes erforderlich ist. Wir können den CDI-Erweiterungsmechanismus verwenden, der in Kapitel 3 eingeführt wurde, um die Klassen für Hibernate zu sammeln.

Ändern Sie zunächst `application/backend/src/main/resources/META-INF/beans.xml` in:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       version="4.0"
       bean-discovery-mode="all">
</beans>
```

`@Entity` ist keine CDI-Bean-definierende Annotation. Der vorherige `annotated`-Modus entdeckt daher keine Klasse, nur weil er `@Entity` hat. Mit `all` entdeckt CDI alle Typen in diesem Bean-Archiv und liefert ihre `ProcessAnnotatedType`-Events. Jedes zukünftige Modul, das Entitäten enthält, benötigt das gleiche Discovery-Setup.

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/EntityRegistry.scala`:

```scala
package com.anjunar.blog

class EntityRegistry(val entityClasses: List[Class[?]])
```

Diese Registry enthält die Entity-Klassen, die beim Start gefunden wurden. Es enthält Klassenmetadaten, keine Instanzen.

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/EntityExtension.scala`:

```scala
package com.anjunar.blog

import jakarta.enterprise.context.spi.CreationalContext
import jakarta.enterprise.event.Observes
import jakarta.enterprise.inject.{Any as AnyQualifier, Default}
import jakarta.enterprise.inject.spi.{AfterBeanDiscovery, Extension, ProcessAnnotatedType, WithAnnotations}
import jakarta.inject.Singleton
import jakarta.persistence.Entity

import java.util.concurrent.ConcurrentHashMap
import scala.jdk.CollectionConverters.*

class EntityExtension extends Extension {
  private val entityClasses = ConcurrentHashMap.newKeySet[Class[?]]()

  def collect(@Observes @WithAnnotations(Array(classOf[Entity])) event: ProcessAnnotatedType[?]): Unit = {
    val annotatedType = event.getAnnotatedType
    if (annotatedType.isAnnotationPresent(classOf[Entity])) {
      entityClasses.add(annotatedType.getJavaClass)
      // Hibernate manages entity instances; CDI only discovers their classes.
      event.veto()
    }
  }

  def registerRegistry(@Observes event: AfterBeanDiscovery): Unit = {
    val discovered = entityClasses.asScala.toList.sortBy(_.getName)
    event.addBean[EntityRegistry]()
      .beanClass(classOf[EntityRegistry])
      .types(classOf[EntityRegistry], classOf[Object])
      .scope(classOf[Singleton])
      .qualifiers(Default.Literal.INSTANCE, AnyQualifier.Literal.INSTANCE)
      .createWith((_: CreationalContext[EntityRegistry]) => new EntityRegistry(discovered))
  }
}
```

`@WithAnnotations` filtert Entdeckungsereignisse. Für jeden mit `@Entity` annotierten Typ speichert der Beobachter seine Klasse und ruft `veto()` auf, um ihn von der CDI-Bean-Registrierung auszuschließen. Hibernate verwaltet die Instanzen der Entität.

Nach der Entdeckung nimmt die Erweiterung eine unveränderliche, sortierte Momentaufnahme und registriert eine `EntityRegistry`-Bean mit Singleton-Scope. `createWith` legt seine Erzeugung fest; `@Default` stellt es einem gewöhnlichen Injektionspunkt zur Verfügung. Die Registry gehört zu diesem CDI-Container. Die [CDI-Spezifikation](https://jakarta.ee/specifications/cdi/4.1/jakarta-cdi-spec-4.1) beschreibt diese Entdeckungsereignisse und synthetische Beans.

Fügen Sie die Erweiterung an `application/backend/src/main/resources/META-INF/services/jakarta.enterprise.inject.spi.Extension` an, wobei Sie den vorhandenen Eintrag beibehalten:

```text
com.anjunar.blog.RestComponentsExtension
com.anjunar.blog.EntityExtension
```

## Registrieren Sie die entdeckten Klassen mit Hibernate

Fügen Sie in `application/backend/src/main/scala/com/anjunar/blog/Persistence.scala` diese Importe hinzu:

```scala
import jakarta.inject.Inject
import org.hibernate.engine.transaction.jta.platform.internal.NarayanaJtaPlatform
import scala.compiletime.uninitialized
```

Fügen Sie dieses Feld innerhalb der vorhandenen `Persistence`-Klasse hinzu:

```scala
@Inject
var entityRegistry: EntityRegistry = uninitialized
```

CDI injiziert die Registry vor dem Aufruf von `@PostConstruct initialize()`. Ersetzen Sie die Registrierungs- und Metadatenkonstruktion innerhalb des bestehenden `try`-Blocks dieser Methode durch:

```scala
registry = new StandardServiceRegistryBuilder()
  .applySetting("jakarta.persistence.jtaDataSource", pool)
  .applySetting("hibernate.transaction.coordinator_class", "jta")
  .applySetting("hibernate.transaction.jta.platform",
    new NarayanaJtaPlatform())
  .applySetting("hibernate.hbm2ddl.auto", "validate")
  .applySetting("jakarta.persistence.validation.mode", "CALLBACK")
  .build()
val sources = new MetadataSources(registry)
entityRegistry.entityClasses.foreach(sources.addAnnotatedClass)
factory = sources.buildMetadata().buildSessionFactory()
```

Behalten Sie die bestehende Poolkonstruktion, Fehlerbereinigung, Produzent und Abschaltungsmethode bei. Das [vollständige Persistence.scala](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/main/scala/com/anjunar/blog/Persistence.scala) zeigt die resultierende Datei an.

Es gibt keine fest codierte Entitätsliste in `Persistence`. Eine weitere gemappte Entität in einem erkannten Bean-Archiv wird dadurch für Hibernate verfügbar. Das zugehörige Datenbankschema muss weiterhin existieren.

Zwei Einstellungen definieren die Persistenzprüfungen:

- `validate` überprüft die abgebildete Tabellen- und Spaltenstruktur während der Factoryinitialisierung.
- `CALLBACK` erfordert die Bean-Validierung, bevor Entitäten eingefügt und aktualisiert werden.

Der Schemamodus führt Überprüfungen durch, ohne die Tabelle zu erstellen oder zu ändern. Einschränkungen im SQL-Skript benötigen weiterhin eigene Tests; die Schemavalidierung ersetzt diese Prüfungen nicht.

Der Checkpoint wendet auch die Importkonvention des Projekts auf den vorhandenen Code an: importierte Typen und APIs, Importaliase für Narayana-Zugriff und `lang.Long` oder `lang.Integer` für Java-Wrapper.

## Veraltete Bearbeitungen erkennen

Das `@Version`-Feld verbindet einen Entity-Snapshot mit einer bestimmten gespeicherten Version. Wenn sich ein verwalteter Beitrag ändert, erhöht Hibernate diesen Wert, während es das Update schreibt.

Betrachten Sie zwei Editoren, die denselben Beitrag laden. Der erste Editor speichert einen neuen Titel. Die Kopie des zweiten Editors trägt immer noch die alte Version. Das Speichern dieser Kopie muss einen Konflikt melden und gleichzeitig die erste Änderung beibehalten.

Dies ist der eigentliche Test von `BlogPostPersistenceSpec`:

```scala
test("an old detached version cannot overwrite a newer committed edit") {
  val saved = savedDraft()
  val firstEditor = load(saved.id)
  val secondEditor = load(saved.id)
  firstEditor.title = "The committed title"
  inTransaction()(_.merge(firstEditor))
  secondEditor.title = "The stale title"
  intercept[OptimisticLockException] {
    inTransaction()(_.merge(secondEditor))
  }
  val current = load(saved.id)
  assert(current.title == "The committed title")
  assert(current.version.longValue() == saved.version.longValue() + 1)
}
```

Der `load`-Helfer des Tests liest sich jedes Mal in einem separaten Persistenzkontext. `inTransaction` verwendet die Klassen `Persistence` und `RequestTransaction` aus Kapitel 4. Die beiden Kopien stellen somit getrennte Lesevorgänge derselben gespeicherten Version dar.

Der erste Merge-Vorgang wird committet. Der zweite wirft `OptimisticLockException` aus. Der letzte Lesevorgang bestätigt, dass der neuere Titel in der Datenbank verbleibt. Die kompletten Helfer sind im [Persistenztest](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/test/scala/com/anjunar/blog/BlogPostPersistenceSpec.scala). Sobald REST und Frontend hinzukommen, führen wir diese Version durch deren Verträge weiter.

## Durchführung der Prüfungen

Fügen Sie [BlogPostValidationSpec](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/test/scala/com/anjunar/blog/BlogPostValidationSpec.scala) und [BlogPostPersistenceSpec](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/test/scala/com/anjunar/blog/BlogPostPersistenceSpec.scala) zusammen mit [EntityDiscoveryProbe](https://github.com/anjunar/anjunar-blog-example/blob/7259db379231b095c503069f6a5950f385302e01/application/backend/src/test/scala/com/anjunar/blog/EntityDiscoveryProbe.scala) unter `application/backend/src/test/scala/com/anjunar/blog` hinzu, wenn Sie das Kapitel von Hand reproduzieren.

Setzen Sie `bean-discovery-mode="all"` auch in `application/backend/src/test/resources/META-INF/beans.xml`. Die Persistenz-Suite startet nun einen echten CDI-Container und erhält `Persistence` von ihm, so dass Registry-Injection und `@PostConstruct` wie in der Anwendung laufen. Das Schließen des Containers schließt auch die Persistenzressourcen.

`EntityDiscoveryProbe` ist eine zweite Entität unter `src/test`, ohne CDI-Bereich und ohne Eintrag in `Persistence`. Sie bildet nur ID und Titel der vorhandenen Tabelle ab. Ein Test überprüft, ob beide Entitäten die Registry erreichen und von der CDI-Bean-Auflösung ausgeschlossen sind. Ein anderer speichert eine `BlogPost` und liest diese Zeile durch die Sonde, was beweist, dass Hibernate die zusätzliche Zuordnung erhalten hat. Die Sonde benötigt keine zusätzliche Tabelle und fehlt bei einem normalen Anwendungslauf.

Mit PostgreSQL ausgeführt, die Tabelle erstellt und die `BLOG_DB_*` Umgebungsvariablen wie in Kapitel 4 konfiguriert:

```text
sbt --server "application-backend/testFull"
```

Erwarten Sie **26 erfolgreiche Tests**. Die Prüfungen umfassen CDI-Entity-Erkennung, Feldannotationen, Publikationsübergänge, Persistenz über Kontexte hinweg, Versionsinkremente, Validierung von Inserts und Updates, Duplicate Slugs, veraltete Edits und eine Datenbankprüfungsbeschränkung.

Die Modelltests entfernen nur die Zeilen, die sie erstellt haben. Die Transaktionstests aus Kapitel 4 erstellen und entfernen weiterhin ihre eigene Testtabelle. Erwartete Fehlerfälle können Fehlerprotokolle erzeugen; Prüfen Sie die abschließende Testzusammenfassung.

Sie können auch die normale Anwendung starten und die Bereitschaft überprüfen:

```text
sbt --server "application-backend/run"
```

In einem anderen Terminal:

```text
curl -i http://127.0.0.1:8080/service/health/ready
```

Verwenden Sie `curl.exe` in Windows PowerShell, falls erforderlich. Erwarten Sie HTTP 200 und `UP`. Die Factory-Initialisierung validiert nun `blog_post`, so dass eine fehlende Tabelle die Bereitschaft mit HTTP 500 fehlschlägt. Liveness funktioniert weiterhin unabhängig von der Datenbank.

## Den Stand des Kapitels verwenden

Der [vollständige Quellcode zu Kapitel 5](https://github.com/anjunar/anjunar-blog-example/tree/7259db379231b095c503069f6a5950f385302e01) enthält das Modell, das SQL-Skript und die Tests. [Pull Request #5](https://github.com/anjunar/anjunar-blog-example/pull/5) führt das Modell ein, [Pull Request #6](https://github.com/anjunar/anjunar-blog-example/pull/6) vereinfacht seine Annotationen und [Pull Request #7](https://github.com/anjunar/anjunar-blog-example/pull/7) fügt die automatische Entitätserkennung über CDI hinzu. Der Checkpoint beinhaltet alle drei Änderungen. Um es in einem separaten Verzeichnis zu verwenden:

```text
git clone https://github.com/anjunar/anjunar-blog-example.git
cd anjunar-blog-example
git switch --detach 7259db379231b095c503069f6a5950f385302e01
```

Konfigurieren Sie PostgreSQL, wenden Sie das ursprüngliche SQL-Skript einmal an und führen Sie `testFull` aus. Verwenden Sie `git switch -c my-blog`, um Ihre eigene Implementierung fortzusetzen.

Die Quelle und das SQL-Skript wurden mit PostgreSQL 18.6 verifiziert. Die Tests bestätigen, dass unsere erste Entität gespeichert und geändert werden kann, während ihre Validierung, Einzigartigkeit und Versionsregeln durchgesetzt werden.

Als nächstes werden wir das Schema ändern, während wir bestehende Posts beibehalten. Damit werden Hibernate DDL Manager, stabile Schema-Identifikatoren und aufgezeichnete Migrationen eingeführt.
