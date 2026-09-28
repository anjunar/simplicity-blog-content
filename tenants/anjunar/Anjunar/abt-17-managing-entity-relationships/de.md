# Entitätsbeziehungen verwalten

Unser Editor kann den Text eines Artikels ändern. Nun soll er auch die Auswahl eines Autors und Tags wie „Scala“ oder „Persistence“ ermöglichen.

Diese Auswahl führt zu einer anderen Art von Änderung. Ein Titel gehört zu einem Artikel, ein Tag kann jedoch in vielen Artikeln vorkommen. Beim Ändern der Auswahl eines Artikels darf der gemeinsam genutzte Tag nicht versehentlich umbenannt oder gelöscht werden.

Wir verfolgen diese Unterscheidung durch Entity-Mapping, API und Formular. Das Kapitel baut auf dem Bearbeitungsablauf aus Kapitel 14–15 und der Suchinfrastruktur aus Kapitel 16 auf. Der [fertige Checkpoint](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/docs/entity-relationships.md) enthält die vollständige Implementierung und Einrichtung.

## Eigentümerschaft modellieren

Ein Artikel verweist auf ein optionales Account-Objekt und eine Menge von BlogTag-Entitäten. Das Konto ist bereits vorhanden; wir ergänzen einen öffentlichen Anzeigenamen. Ein Tag erhält eine eigene ID, Version, eindeutigen Slug und Namen.

Der Artikel speichert seinen Autor in author_id. Tags verwenden die Verknüpfungstabelle blog_post_tag, da mehrere Artikel denselben Tag nutzen können. Diese Ergänzungen kommen in [BlogPost.scala](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala):

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.{FetchType, ForeignKey, JoinColumn, JoinTable, ManyToMany, ManyToOne}
import jakarta.validation.constraints.{NotNull, Size}
import java.util

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "author_id", foreignKey = new ForeignKey(name = "fk_blog_post_author"))
@SchemaId("6a1eab40") @JsonbProperty
var author: Account = null

@ManyToMany(fetch = FetchType.LAZY)
@JoinTable(name = "blog_post_tag", schema = "public",
  joinColumns = Array(new JoinColumn(name = "post_id")),
  inverseJoinColumns = Array(new JoinColumn(name = "tag_id")),
  foreignKey = new ForeignKey(name = "fk_blog_post_tag_post"),
  inverseForeignKey = new ForeignKey(name = "fk_blog_post_tag_tag"))
@NotNull @Size(max = 20)
@SchemaId("42b9d1ef") @JsonbProperty
var tags: util.Set[BlogTag] = new util.LinkedHashSet[BlogTag]()
```

Wichtig ist, dass kein kaskadierendes Löschen eingerichtet wird. Wird „Persistence“ aus einem Artikel entfernt, wird lediglich ein Datensatz aus der Verknüpfungstabelle entfernt. Der Tag bleibt für andere Artikel verfügbar.

Der Autor ist nullable, damit bestehende Artikel ohne erfundene Zuordnung durch die Migration kommen. Neue Artikel erhalten standardmäßig den angemeldeten Administrator als Autor; Redakteure können diese Auswahl aufheben. In diesem Tutorial stehen nur nicht gesperrte Administratoren als Autoren zur Verfügung.

Die vorhandene CDI-Erweiterung erkennt [BlogTag](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/BlogTag.scala). Hibernate DDL Manager ergänzt mithilfe stabiler SchemaIds die neuen Spalten und Tabellen. Wir registrieren die Entität nicht manuell und erstellen die Datenbank nicht neu.

## Die öffentliche Beziehung beschreiben

Das Mapping teilt Hibernate mit, was gespeichert wird. Entity-Schema und Graph legen fest, wie die API damit umgeht.

In BlogPost.Schema ergänzen wir die beiden Eigenschaften:

```scala
import com.anjunar.json.mapper.schema.property.{SetProperty, SingularProperty}
import java.util

val author: SingularProperty[BlogPost, Account] =
  reference(_.author, classOf[PostEditRule])

val tags: SetProperty[BlogPost, util.Set[BlogTag]] =
  set(_.tags, classOf[PostEditRule])
```

Diese Eigenschaften erhalten die Beziehungstypen und wenden die vorhandene Schreibregel für Administratoren an. Der Mapper verwendet dieses Schema zusammen mit dem Detailgraphen.

Ein Konto enthält weit mehr als eine Autorenzeile. Deshalb muss der Graph die verschachtelten Autorendaten bewusst einschränken:

| Detailfeld | Veröffentlichte Eigenschaften |
| --- | --- |
| author | id, version, displayName |
| tags | id, version, slug, name |

E-Mail-Adresse, Rolle und Authentifizierungsdaten gehören nicht in eine öffentliche Artikelantwort. [Account.author](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/Account.scala) und der Autor-Untergraph des Artikels wählen nur die öffentlichen Felder aus.

Die Listenprojektion aus Kapitel 16 bleibt unverändert. Die Beziehungen laden wir in der Detailansicht; ein Sammlungs-Join in einer paginierten Liste würde ein eigenes Abfrageproblem verursachen.

## Eine Auswahl als ID entgegennehmen

Wählt der Redakteur einen bestehenden Autor und Tag aus, sendet die Anfrage deren IDs zusammen mit der Artikelversion:

```json
{
  "version": 4,
  "author": { "id": "df99c76a-c98d-41f4-b391-3d2631cfeea8" },
  "tags": [{ "id": "b891ec6d-11dc-43e4-94ef-bc65406b5262" }]
}
```

Die IDs sind Beispiele; echte Optionen stammen aus dem Autoren- und Tag-Katalog. Wird neben einer Tag-ID auch dessen name gesendet, wird die Anfrage abgewiesen. Die Aktualisierung eines Artikels berechtigt nicht dazu, die referenzierte Entität zu bearbeiten.

Weglassen und Löschen müssen unterschiedliche Bedeutungen haben:

| Eingabe für die Aktualisierung | Ergebnis |
| --- | --- |
| author oder tags ausgelassen | Bestehenden Wert beibehalten |
| author: null | Autor entfernen |
| tags: [] | Alle Tag-Zuordnungen entfernen |
| tags: null | Abweisen; die Tag-Sammlung muss ein Array sein |

[ReferenceAccess](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/ReferenceAccess.scala) prüft, dass jede Referenz genau eine ID enthält, validiert deren UUID und weist doppelte Tags zurück. Anschließend ruft der Mapper seinen Loader auf, um die tatsächliche Entität aufzulösen.

Dies ist die Loader-Methode in der request-scoped Klasse; manager und caller werden über CDI injiziert:

```scala
import jakarta.ws.rs.ForbiddenException
import java.util.UUID

override def load(id: UUID, clazz: Class[?]): Any = {
  if (!caller.administrator) throw new ForbiddenException()
  if (clazz == classOf[Account]) {
    val account = manager.find(classOf[Account], id)
    if (account == null || account.locked || account.role != "ADMIN")
      Problem.invalidField("author", "Choose an available author.")
    account
  } else if (clazz == classOf[BlogTag]) {
    val tag = manager.find(classOf[BlogTag], id)
    if (tag == null) Problem.invalidField("tags", "Choose an available tag.")
    tag
  } else {
    throw new ApiProblem(400, "This entity type cannot be referenced.")
  }
}
```

Die Verfügbarkeit hier zu prüfen ist wichtig. Ein Konto könnte gesperrt worden sein, nachdem der Redakteur die Auswahl geladen hat. Eine zuvor angezeigte Option ist keine Erlaubnis, sie jetzt zuzuweisen.

Der Loader gibt eine verwaltete Entität zurück. Er erstellt niemals aus einem übermittelten Objekt ein neues Konto oder einen neuen Tag.

## Die vorhandene Schreibgrenze beibehalten

Der restliche Ablauf folgt dem PreparedChange-Verfahren, das wir bereits aufgebaut haben:

1. Artikel laden und die übermittelte Version prüfen.
2. Eingabe mit Schema, Graph, Referenz-Loader und Validator vorbereiten.
3. Zugriffsrechte prüfen und danach applyChanges() genau einmal aufrufen.
4. Innerhalb der Request-Transaktion flushen und serialisieren.

Der JSON Mapper validiert die übermittelte Sammlung einschließlich der Grenze von 20 Tags. Ein Fehler bei einer Referenz oder Validierung rollt die gesamte Anfrage zurück, auch Änderungen an anderen Feldern desselben Vorgangs.

[PreparedChanges](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/backend/src/main/scala/com/anjunar/blog/PreparedChanges.scala) unterstützt nun BlogPost, BlogTag und Account. Das ist eine explizite Positivliste für Schreibzugriffe, getrennt von der CDI-Erkennung persistenter Entitäten. Eine erkannte Entität kann dadurch nicht automatisch über REST bearbeitet werden.

Gemeinsam genutzte Metadaten erhalten eigene Operationen. Administratoren können den öffentlichen Namen eines Autors ändern sowie Tags anlegen und umbenennen. Bei jeder Änderung wird die eigene Version der Entität geprüft. Der Konto-Endpunkt akzeptiert nur displayName sowie Identitäts- und Versionsmetadaten; Rollen und Zugangsdaten bleiben unveränderbar.

Die Kataloge verwenden erneut HibernateSearch, typisierte Schema-Attribute und CDI-Provider. Sie liefern begrenzte Seiten mit Links zur nächsten Seite. Für sie gelten dieselben Endpunkt-Richtlinien und derselbe CSRF-Schutz wie für die Artikelbearbeitung.

## Die Auswahl an das Artikelmodell binden

Das Scala.js-Modell bildet die veröffentlichte Struktur nach. In der Frontend-Datei [BlogPost.scala](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPost.scala) kommen folgende Felder hinzu:

```scala
import ui.core.state.{ListProperty, Property}
import ui.forms.validators.Size
import scala.annotation.meta.field

val author: Property[Account] = Property(null)

@(Size @field)(max = 20)
val tags: ListProperty[BlogTag] = ListProperty()
```

Das Frontend behält vollständige Referenzmodelle, um Namen anzuzeigen. Der Schreibinhalt darf dennoch nur IDs enthalten. In writeBody() verkleinern wir die Referenzobjekte, nachdem der JSON-Mapper die geänderten Felder in body serialisiert hat:

```scala
import scala.scalajs.js

def onlyId(reference: js.Dynamic): Unit =
  if (reference != null && !js.isUndefined(reference))
    js.Object.keys(reference.asInstanceOf[js.Object]).filter(_ != "id")
      .foreach(key => js.special.delete(reference, key))

onlyId(body.author)
if (!js.isUndefined(body.tags))
  body.tags.asInstanceOf[js.Array[js.Dynamic]].foreach(onlyId)
```

So bleibt das partielle Schreibverhalten des Mappers erhalten: Unveränderte Felder werden ausgelassen, während null und ein leeres Array weiterhin ausdrückliche Löschvorgänge darstellen.

Die Autoren-ComboBox bindet direkt an Property[Account]. Für Tags brauchen wir einen kleinen Adapter: Die festgelegte ComboBox-API stellt selbst im Mehrfachauswahlmodus nur einen einzelnen Formularwert bereit. [TagSelection](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/frontend/src/main/scala/com/anjunar/blog/frontend/TagSelection.scala) bindet die Auswahlliste an das tags-Feld des übergeordneten Formulars. So bleiben Sammlungsvalidierung und Serverfehler dem richtigen Steuerelement zugeordnet.

Beide Steuerelemente sind Teil des vorhandenen [Artikelformulars](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostEditorPage.scala). Auf einer separaten Metadatenseite können Administratoren die gemeinsam verwendeten Namen vor der Auswahl pflegen.

Das Speicherverhalten aus Kapitel 15 gilt auch für Beziehungen. Sendet eine Anfrage Autor A ab und wählt der Nutzer währenddessen B aus, darf die Antwort für A die neuere Auswahl nicht überschreiben. Wir vergleichen Autoren-IDs und Mengen von Tag-IDs, erhalten neuere Auswahlen und aktualisieren trotzdem die gespeicherte Version.

## Den vollständigen Ablauf ausprobieren

Der Checkpoint verwendet JSON Mapper 1.1.6 von Maven Central. Diese Version enthält die hier benötigte Korrektur zur Auflösung von Referenzen. Bei eingerichteter Entwicklungsdatenbank und gestopptem Server:

```text
git switch --detach a16b0c53d65b6ef5e9351e42f00637948dd884c8
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server frontendAssets "application-backend/run"
```

Das Upgrade ergänzt öffentliche Namen, Autorenreferenzen, Tags und die Verknüpfungstabelle. Bestehende Artikel behalten ihre Daten und erhalten keinen zugewiesenen Autor. Der [Migrationsleitfaden](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/docs/entity-relationships.md#start-and-migrate) erläutert das bestehende Vorschauverhalten bei CHECK-Constraints.

Melde dich an, öffne **Autoren und Tags verwalten**, lege einen öffentlichen Namen fest und erstelle einen Tag. Erstelle danach einen Artikel, wähle den Tag, speichere und veröffentliche ihn. In der öffentlichen Detailansicht erscheinen Name und Tag. Entferne den Tag im Editor und speichere erneut: Er verschwindet aus dem Artikel, bleibt aber im Katalog erhalten.

Das [ausführbare API-Beispiel](https://github.com/anjunar/anjunar-blog-example/blob/a16b0c53d65b6ef5e9351e42f00637948dd884c8/docs/examples/entity-relationships.js) prüft auch weniger offensichtliche Fälle: Eine verschachtelte Umbenennung eines Tags weist die gesamte Änderung zurück; ausgelassene Referenzen bleiben erhalten; beim Entfernen bleibt der gemeinsam genutzte Tag bestehen. Es läuft unverändert im echten HTTP-/PostgreSQL-Browsertest.

Damit haben wir Beziehungen mit klarer Eigentümerschaft, einer begrenzten öffentlichen Darstellung und bewusst festgelegter Schreibsemantik. Als Nächstes wenden wir diese Ideen auf Medien-Uploads an, bei denen Eigentümerschaft und Bereinigung anderen Regeln folgen.
