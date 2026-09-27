In Kapitel 5 entstanden die Tabelle `blog_post` und eine funktionierende Entität. Stellen Sie sich nun vor, dass die Tabelle Posts enthält, die wir behalten möchten, aber die Anwendung benötigt ein anderes Feld.

Dieses Kapitel fügt eine optionale Zusammenfassung hinzu, ohne die Tabelle neu zu erstellen. Zuerst machen wir Hibernate DDL Manager mit dem bestehenden Schema vertraut, ändern dann die Entität und wenden die resultierende Migration an. Die Veröffentlichungseinschränkung aus Kapitel 5 bleibt durchweg aktiv.

Beginnen Sie mit dem [Kapitel 5 Projekt](https://github.com/anjunar/anjunar-blog-example/tree/7259db379231b095c503069f6a5950f385302e01) und seiner PostgreSQL-Datenbank. Stoppen Sie den HTTP-Server, während Sie diese Schritte ausführen. Alle Dateipfade unten sind relativ zum Projektverzeichnis.

## Die Geschichte des Schemas festhalten

Hibernate läuft derzeit mit `hibernate.hbm2ddl.auto=validate`. Es kann eine fehlende kartierte Spalte erkennen, fügt sie jedoch nicht hinzu. Wir halten an diesem Setting fest.

Hibernate DDL Manager nimmt das abgebildete Schema, vergleicht es mit seinem aufgezeichneten Modell, überprüft die tatsächliche Datenbank und wendet eine Migration in einer Datenbanktransaktion an. Es zeichnet das Ergebnis in `__hibernate_ddl.schema_history` auf.

Wir unterscheiden drei Zustände:

|Zustand|Bedeutung|
| --- | --- |
|Entitätszuordnung|Das Schema, das diese Anwendungsversion möchte.|
|Aufgezeichnetes Modell|Das Schema der letzten erfolgreichen Migration.|
|PostgreSQL Katalog|Das Schema, das jetzt tatsächlich existiert.|

Die Katalogprüfung ist wichtig: Das manuelle Ändern einer Tabelle darf die aufgezeichnete Historie nicht stillschweigend ungültig machen.

Fügen Sie diese Abhängigkeit zur vorhandenen Sequenz in `build.sbt` hinzu:

```scala
"com.anjunar.hibernateddl" %% "schema-integration" % "1.1.0",
```

Version 1.1.0 unterstützt die benannte, mehrspaltige CHECK-Einschränkung, die von unserer Veröffentlichungsregel verwendet wird. Die Abhängigkeit kommt von Maven Central; Leser brauchen keinen lokalen Framework-Checkout.

## Geben Sie Tabellen und Spalten stabile Identitäten

Ein Spaltenname kann sich ändern. Seine Identität sollte diese Veränderung überleben. Importieren Sie die Annotation in `BlogPost.scala`:

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
```

Fügen Sie `@SchemaId("d4f39c20")` zur Klasse und eine ID zu jedem persistenten Feld hinzu:

|Element|Schema ID|
| --- | --- |
|`BlogPost`|`d4f39c20`|
|`id`|`a2473e8b`|
|`version`|`dcb0681e`|
|`slug`|`682d9ace`|
|`title`|`46fdb02a`|
|`content`|`7b20efc1`|
|`status`|`cf271a06`|
|`publishedAt`|`398bfd50`|

Zum Beispiel bleibt der Titel:

```scala
@NotBlank
@Size(min = 3, max = 180)
@Column(nullable = false, length = 180)
@SchemaId("46fdb02a")
var title: String = ""
```

Dies sind Felddeklarationen innerhalb des Klassenkörpers, so dass die Annotationen kein explizites `@field`-Ziel benötigen.

Der Manager kombiniert die Tabellen- und Feld-IDs: Die Identität des Titels lautet `d4f39c20/46fdb02a`. Wählen Sie IDs einmal aus, committen Sie sie und behalten Sie sie beim Bearbeiten des gleichen Schemaelements. Regenerieren Sie sie nicht bei jedem Build.

Diese IDs identifizieren Schemaelemente. Die UUID eines Beitrags identifiziert immer noch eine Zeile, und seine `version` erkennt immer noch veraltete Bearbeitungen. Keine dieser Kennungen ersetzt `@SchemaId`. Diese Annotation hat außerdem einen anderen Zweck als `EntitySchema`, die wir als nächstes vorstellen werden.

## Fügen Sie die Datenbankregel in das Mapping ein

Kapitel 5 erstellte zwei Datenbankprüfungen: erlaubte Statuswerte und Konsistenz zwischen Status und Veröffentlichungszeit. Hibernate leitet die erlaubten Werte aus dem Enum-Mapping ab. Wir müssen die zweite Regel explizit beschreiben.

Fügen Sie `CheckConstraint` zum bestehenden `jakarta.persistence`-Import hinzu und ersetzen Sie die `@Table`-Annotation der Klasse:

```scala
@Table(name = "blog_post", schema = "public",
  uniqueConstraints = Array(new UniqueConstraint(name = "uq_blog_post_slug", columnNames = Array("slug"))),
  check = Array(new CheckConstraint(name = "ck_blog_post_publication",
    constraint = "(status = 'DRAFT' AND published_at IS NULL) OR (status = 'PUBLISHED' AND published_at IS NOT NULL)")))
```

Der Ausdruck entspricht der vorhandenen SQL-Regel. Behalten Sie auch die Bean Validation Annotationen und Publikationsmethoden. Sie geben nützliche Fehler in der Anwendung; Die Datenbankregel schützt auch außerhalb von Hibernate erstellte Schreibvorgänge.

Der [Ausgangsstand](https://github.com/anjunar/anjunar-blog-example/tree/b693dba5c8e3402a1249bdebf38bc9503d45e99a) enthält die vollständige annotierte Entität. Zu diesem Zeitpunkt hat es noch genau die sieben Felder aus Kapitel 5.

## Führen Sie Migrationen als separaten Befehl aus

Wir entdecken bereits Entitätsklassen durch CDI. Der Migrationsbefehl verwendet das gleiche `EntityRegistry`, so dass das Hinzufügen einer anderen Entität keine Aktualisierung einer zweiten Klassenliste erfordert.

Erstellen Sie `application/backend/src/main/scala/com/anjunar/blog/SchemaMain.scala`:

```scala
package com.anjunar.blog

import com.anjunar.hibernateddl.executor.{ExecutionOptions, PreviewOutcome, PreviewRendering}
import com.anjunar.hibernateddl.integration.HibernateSchemaMigration
import jakarta.enterprise.inject.se.SeContainerInitializer
import org.hibernate.boot.{Metadata, MetadataSources}
import org.hibernate.boot.registry.StandardServiceRegistryBuilder
import org.postgresql.ds.PGSimpleDataSource

import javax.sql.DataSource
import scala.util.Using

object SchemaMain {
  def main(args: Array[String]): Unit = {
    val (command, adoptExisting) = args.toList match {
      case List("preview") => ("preview", false)
      case List("preview", "--adopt-existing") => ("preview", true)
      case List("migrate") => ("migrate", false)
      case List("migrate", "--adopt-existing") => ("migrate", true)
      case _ => throw new IllegalArgumentException(
        "Usage: SchemaMain preview|migrate [--adopt-existing]")
    }

    val config = DatabaseConfig.load()
    val exitCode = Using.resource(SeContainerInitializer.newInstance().initialize()) { container =>
      val entities = container.select(classOf[EntityRegistry]).get()
      withMetadata(config, entities) { (metadata, dataSource) =>
        val options = ExecutionOptions(adoptExistingSchema = adoptExisting)
        command match {
          case "preview" =>
            val report = HibernateSchemaMigration.preview(metadata, dataSource, options)
            println(PreviewRendering.text(report))
            report.outcome match {
              case PreviewOutcome.Ready => 0
              case PreviewOutcome.Blocked => 2
              case PreviewOutcome.Incomplete => 3
            }
          case "migrate" =>
            val result = HibernateSchemaMigration.migrate(metadata, dataSource, options)
            println(s"${result.status}: revision ${result.revision}, ${result.statementCount} SQL statements")
            0
        }
      }
    }
    if (exitCode != 0) sys.exit(exitCode)
  }

  private[blog] def withMetadata[A](config: DatabaseConfig, entities: EntityRegistry)(
      work: (Metadata, DataSource) => A): A = {
    val dataSource = new PGSimpleDataSource()
    dataSource.setUrl(config.url)
    dataSource.setUser(config.user)
    dataSource.setPassword(config.password)
    dataSource.setLoginTimeout(5)

    // The migration executor owns a JDBC transaction, outside the request's JTA boundary.
    val registry = new StandardServiceRegistryBuilder()
      .applySetting("hibernate.connection.datasource", dataSource)
      .applySetting("hibernate.hbm2ddl.auto", "validate")
      .build()
    try {
      val sources = new MetadataSources(registry)
      entities.entityClasses.foreach(sources.addAnnotatedClass)
      work(sources.buildMetadata(), dataSource)
    } finally StandardServiceRegistryBuilder.destroy(registry)
  }
}
```

Der Befehl startet CDI, erstellt Hibernate-Metadaten aus den entdeckten Klassen, führt seine Arbeit aus und schließt CDI und die Serviceregistrierung von Hibernate. Es erstellt keine SessionFactory oder startet Undertow.

Der `PGSimpleDataSource` befindet sich bewusst außerhalb der JTA-Transaktion der Anfrage. Der Migrationsexecutor verwaltet seine JDBC-Transaktion, einschließlich Schemaänderungen und Geschichte. Die Anfragebearbeitung verwendet weiterhin Agroal und Narayana wie zuvor.

Beide Befehle lesen die vorhandenen Variablen `BLOG_DB_URL`, `BLOG_DB_USER` und `BLOG_DB_PASSWORD`. Die Anwendungsrolle muss die verwalteten Tabellen besitzen und in der Lage sein, das Verlaufsschema zu erstellen.

## Die Datenbank aus Kapitel 5 übernehmen

Mit dem begleitenden Repository können Sie diesen genauen Zwischenstand auschecken:

```text
git switch --detach b693dba5c8e3402a1249bdebf38bc9503d45e99a
```

Verwenden Sie diesen Checkout, bevor Sie die Adoptionsbefehle ausführen. Übernehmen Sie das endgültige Mapping nicht gegen die Tabelle aus Kapitel 5, bevor die neue Spalte angelegt ist.

Tun Sie dies, bevor Sie das Zusammenfassungsfeld hinzufügen. Die Übernahme prüft, ob die vorhandene Datenbank mit der aktuellen Zuordnung übereinstimmt, und zeichnet dann die Revision 1 auf, ohne die Anwendungstabelle zu ändern.

Wenn Sie möchten, dass eine konkrete Zeile folgt, führen Sie diese einmal in der Kapitel 5-Datenbank mit `psql` aus:

```sql
INSERT INTO public.blog_post
    (id, version, slug, title, content, status)
VALUES
    ('a090f23a-42bb-40c6-a21a-0ee8df71e7c3', 0, 'migration-example',
     'Keep this title', 'Keep this content', 'DRAFT');
```

Zuerst normalisieren Sie den Namen des enum constraint. Der Manager leitet diesen Namen von der stabilen ID der Statusspalte und den zulässigen Werten ab. Unser ursprüngliches Skript verwendete `ck_blog_post_status`.

Erstellen Sie `database/002-adopt-check-name.sql`:

```sql
ALTER TABLE public.blog_post
    RENAME CONSTRAINT ck_blog_post_status TO ck_6576a412cbd2bfbd503a3350;
```

Dies ist der erwartete Name für die IDs und Enum-Werte oben. Die Umbenennung der Einschränkung ändert weder ihr Prädikat noch die gespeicherten Zeilen. Wenden Sie das Skript einmal an. Mit Compose:

```text
docker compose cp database/002-adopt-check-name.sql postgres:/tmp/002-adopt-check-name.sql
docker compose exec -T postgres psql -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --single-transaction --file /tmp/002-adopt-check-name.sql
```

Mit nativem PostgreSQL:

```text
psql -h 127.0.0.1 -p 5433 -U blog -d anjunar_blog --set ON_ERROR_STOP=1 --single-transaction --file database/002-adopt-check-name.sql
```

Verwenden Sie Ihren konfigurierten Port und Ihre Datenbank, falls abweichend. Führen Sie die SQL-Einrichtung eines Kapitels nicht erneut gegen eine bereits vorbereitete Datenbank aus.

Prüfen Sie nun die Übernahme:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview --adopt-existing"
```

Erwarten Sie `INCOMPLETE`, mit einem Befund über die Normalisierung der benannten Publikationsprüfung. Die Vorschau ist read-only. Der Vergleich eines beliebigen SQL-Prädikats erfordert eine temporäre Probetabelle, die diese Vorschau nicht erstellen kann. Der Befehl endet absichtlich mit Exit-Code 3; sbt meldet diesen Fehlerstatus.

Prüfen Sie die Ergebnisse. Ein unbekanntes Prädikat unterscheidet sich von einer fehlenden Spalte oder einer unerwarteten Einschränkung. Fahren Sie nicht fort, wenn der Bericht einen Blocker enthält.

Führen Sie die Übernahme aus:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate --adopt-existing"
```

Erwarten:

```text
Adopted: revision 1, 0 SQL statements
```

Die Null ist die Anzahl der Migrationsanweisungen für das Anwendungsschema. Die Übernahme schreibt den Verlauf. Während der Ausführung vergleicht der Manager die normalisierten Prüfdefinitionen von PostgreSQL unter seiner Migrationssperre. Eine Einschränkung mit dem richtigen Namen, aber einem schwächeren Ausdruck wird abgelehnt.

## Hinzufügen der Zusammenfassung

Wenn Revision 1 aufgezeichnet ist, fügen Sie dieses Feld zu `BlogPost` hinzu:

```scala
@Size(max = 300)
@Column(length = 300)
@SchemaId("0ca6e520")
var summary: String = null
```

Eine Zusammenfassung ist optional. Bestehende Posts haben keine Zusammenfassung, daher ist SQL NULL der richtige Anfangswert. `@Size` erlaubt Null und begrenzt einen bereitgestellten Wert auf 300 Zeichen. Ein JVM-Initializer würde vorhandene Datenbankzeilen nicht füllen.

Überprüfen Sie die Änderung:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
```

Der Plan fügt eine nullbare Spalte hinzu:

```sql
ALTER TABLE "public"."blog_post" ADD COLUMN "summary" varchar(300);
```

Die Veröffentlichungsprüfung macht immer noch diese schreibgeschützte Vorschau `INCOMPLETE`. Die Ausführung wird dieses Prädikat überprüfen, bevor die Änderung angewendet wird.

Führen Sie die Migration ohne Adoptionsflag aus:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

Erwarten Sie `Applied: revision 2, 1 SQL statements`. Führen Sie den gleichen Befehl erneut aus und erwarten Sie `AlreadyApplied: revision 2, 0 SQL statements`. Ein wiederholter Durchlauf überprüft die Datenbank, legt jedoch keine weitere Revision an.

## Überprüfen Sie das Ergebnis

Prüfen Sie die beibehaltene Zeile:

```sql
SELECT slug, title, content, summary, status, published_at
FROM public.blog_post
WHERE id = 'a090f23a-42bb-40c6-a21a-0ee8df71e7c3';
```

Der ursprüngliche Slug sowie Titel, Inhalt und Status bleiben erhalten. Sowohl `summary` als auch `published_at` sind NULL.

Überprüfen Sie die Regel mit einem separaten `psql`-Befehl:

```sql
UPDATE public.blog_post
SET status = 'PUBLISHED'
WHERE id = 'a090f23a-42bb-40c6-a21a-0ee8df71e7c3';
```

PostgreSQL lehnt dies mit SQLSTATE `23514` ab: Für die Veröffentlichung ist noch ein Zeitstempel erforderlich. Die fehlgeschlagene Anweisung lässt den Entwurf unverändert.

Führen Sie dann die Anwendungsprüfungen durch:

```text
sbt --server "application-backend/testFull"
sbt --server "application-backend/run"
```

Erwarten Sie **28 erfolgreiche Tests**, einschließlich optionaler Zusammenfassungen, ihrer Längenbegrenzung, Persistenz, CDI-Entdeckung und der vorhandenen Transaktionsprüfungen. Bereitschaft bei `/service/health/ready` sollte HTTP 200 mit `UP` zurückgeben.

## Mit einer leeren Datenbank beginnen

Ein Leser, der direkt mit dem endgültigen Code dieses Kapitels beginnt, muss die alte Tabelle nicht erstellen und übernehmen.

Mit einer leeren Tutorial-Datenbank und den gleichen Umgebungsvariablen führen Sie aus:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
```

Der Manager erstellt die aktuelle Tabelle, einschließlich Zusammenfassung und beide Constraints, und zeichnet Revision 1 auf. Überspringen Sie `001-blog-post.sql`, `002-adopt-check-name.sql` und `--adopt-existing` auf diesem Weg.

## Wissen, wann eine Migration mehr Arbeit braucht

Stabile IDs ermöglichen es dem Planer, ein vorhandenes Schemaelement zu erkennen, auch wenn sich sein Name ändert. Dadurch wird jedoch nicht jede Änderung automatisch möglich.

Version 1.1.0 behandelt benannte Prüfausdrücke als SQL, ohne zu versuchen, sie neu zu schreiben. Das Ändern einer solchen Überprüfung oder das Umbenennen von Spalten in der Tabelle erfordert eine explizite manuelle Migration und den `acceptManualMigration`-Workflow des Managers. Wir legen diesen Workflow in diesem kleinen CLI nicht offen.

Ein erforderliches neues Feld benötigt auch eine Entscheidung über bestehende Zeilen: Welcher Wert korrekt ist, wie er nachgetragen wird und wann er benötigt wird. Das Hinzufügen eines JVM-Standards ist kein Datenbank-Backfill.

Für dieses Kapitel haben wir eine vollständige, kleine Änderung: das bestehende Modell übernehmen, eine optionale Zusammenfassung hinzufügen, die Posts beibehalten und die Datenbankregel beibehalten.

Der [vollständige Quellcode zu Kapitel 6](https://github.com/anjunar/anjunar-blog-example/tree/185a0fd7634f1da3e7f7b420a806a022033cd242) enthält das lauffähige Ergebnis. Als nächstes beschreiben wir die Entität mit `EntitySchema` und geben den JSON-Mapper- und Kriterienabfragen ein gemeinsames Feldmodell.
