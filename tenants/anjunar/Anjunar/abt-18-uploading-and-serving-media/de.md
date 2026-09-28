# Medien hochladen und ausliefern

Unser Artikelformular kann Text, Autoren und Tags speichern. Jetzt ergänzen wir ein Titelbild: eine Datei auswählen, eine Vorschau ansehen, eine Beschreibung hinzufügen und den Artikel veröffentlichen.

Diese kleine Funktion überschreitet drei Grenzen. Eine Datei muss zu einem gültigen Bild werden, das Bild muss als Artikelreferenz gespeichert werden, und jeder Abruf muss die Sichtbarkeit des Artikels berücksichtigen. Wenn wir diese Schritte ausdrücklich behandeln, können wir auch Uploads bereinigen, die nie einem gespeicherten Artikel zugeordnet wurden.

Der [Checkpoint dieses Kapitels](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/docs/uploading-and-serving-media.md) enthält die vollständige Implementierung und Einrichtung. Wir bauen auf der Referenzbehandlung in Kapitel 17 auf; hier geht es um die Besonderheiten von Medien.

## Dem Bild einen eigenen Lebenszyklus geben

Ein Upload erzeugt eine Media-Entität mit ID, Eigentümer, Erstellungszeitpunkt, Abmessungen, Inhaltstyp und Bytes. PostgreSQL speichert die Bytes zusammen mit den Metadaten in einer bytea-Spalte. So werden beides gemeinsam übernommen oder zurückgerollt. Ein zweiter Dateispeicher muss nicht koordiniert werden.

Ein Artikel verweist auf diese Entität. Der Alternativtext gehört zum Artikel, denn dasselbe Bild kann in verschiedenen Artikeln unterschiedliche Zwecke erfüllen. In [BlogPost](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala) kommen folgende Felder hinzu:

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.{Column, FetchType, ForeignKey, JoinColumn, ManyToOne}
import jakarta.validation.constraints.Size

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "cover_image_id",
  foreignKey = new ForeignKey(name = "fk_blog_post_cover"))
@SchemaId("ad182001") @JsonbProperty
var coverImage: Media = null

@Size(max = 300) @Column(name = "cover_alt", length = 300)
@SchemaId("ad182002") @JsonbProperty
var coverAlt: String = null
```

Beide Felder sind nullable, damit bestehende Artikel gültig bleiben. Eine Ganzobjekt-Constraint verlangt eine nichtleere Beschreibung, sobald ein Bild zugeordnet ist.

Es gibt kein kaskadierendes Löschen: Wird das Titelbild aus einem Artikel entfernt, darf ein Bild, das an anderer Stelle verwendet wird, nicht gelöscht werden. Die vorhandene CDI-Erweiterung erkennt Media; Hibernate DDL Manager ergänzt die Tabelle, die zwei Artikelspalten und den Fremdschlüssel.

## Die hochgeladene Datei in Bildpunkte umwandeln

Der Browser sendet eine File-Datei direkt als Body von POST /service/editorial/media. Der Inhaltstyp ist image/jpeg oder image/png. Der Endpunkt verlangt einen Administrator und das CSRF-Token der Sitzung. Ein optionaler Dateiname dient nur als Anzeigename.

Der Content-Type-Header ist eine Behauptung des Clients. Wir prüfen das tatsächliche Format, lesen die Abmessungen aus, dekodieren das Bild und kodieren seine Bildpunkte in ein neues JPEG oder PNG. Dadurch werden vor dem Speichern Quellmetadaten und angehängte Zusatzdaten entfernt.

Die erste Grenze ist ein beschränktes Einlesen. Diese Zeilen aus [ImageContent.read](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/ImageContent.scala) folgen auf die Prüfung des angegebenen Typs und die Reservierung eines Decoder-Slots:

```scala
import java.io.InputStream

// input: InputStream; MaxBytes = 5 * 1024 * 1024
val bytes = input.readNBytes(MaxBytes + 1)
if (bytes.length > MaxBytes)
  throw new ApiProblem(413, "Images may be at most 5 MiB.")
if (bytes.isEmpty)
  Problem.invalidField("coverImage", "Choose a nonempty image.")
decode(bytes, contentType)
```

Ein zusätzliches Byte unterscheidet eine zulässige Datei von einer zu großen, ohne die ganze Anfrage einzulesen. Vor der Zuweisung des dekodierten Bildes prüft der Decoder außerdem höchstens 12 Millionen Bildpunkte und maximal 8000 Pixel pro Seite. Die Größe der komprimierten Datei allein begrenzt den Speicherbedarf nach dem Dekodieren nicht.

Gleichzeitig laufen höchstens zwei Decoder, und die normalisierte Ausgabe ist auf 8 MiB begrenzt. Der Undertow-Listener lässt für den Upload genügend Platz, während unser Leser das kleinere Anwendungslimit durchsetzt. Der [Leitfaden](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/docs/uploading-and-serving-media.md) dokumentiert diese Grenzen und ihre Tests.

Diese erste Version speichert Standbilder ohne Größenänderung oder Vorschaubilder. Außerdem korrigiert sie die EXIF-Ausrichtung nicht. Exportiere Fotos daher vor dem Upload mit der gewünschten Pixelausrichtung.

## Eine Referenz speichern, keinen zweiten Upload

Ein erfolgreicher Upload liefert Data[Media]: ID, Version, Name, Inhaltstyp, Abmessungen und Byteanzahl. Der Entitätsgraph enthält weder Eigentümer noch Binärdaten. Der Frontend-Mapper macht aus diesen Metadaten ein Media-Objekt in BlogPost.coverImage.

Der Upload speichert den Artikel noch nicht. Drückt der Redakteur **Artikel speichern**, erhält der vorhandene PreparedChange[BlogPost]-Ablauf eine Referenz, die nur die ID enthält:

```json
{
  "version": 0,
  "coverImage": { "id": "896d0270-b13b-4735-bac6-3606a0407cc0" },
  "coverAlt": "A blue notebook beside a laptop"
}
```

Verwende die beim Upload zurückgegebene ID. Wird coverImage ausgelassen, bleibt die bisherige Auswahl bestehen; null entfernt sie. Verschachtelte Felder wie name benennen das Medium nicht um, sondern führen zur Ablehnung der Anfrage.

[ReferenceAccess](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/ReferenceAccess.scala) lädt das Bild und prüft seine Berechtigung, bevor der Mapper die Änderung anwendet. Ein noch ungenutzter Upload gehört seinem Uploader. Sobald ein Artikel darauf verweist, können ihn entsprechend den gemeinsamen Redaktionsberechtigungen auch andere Administratoren verwenden. Wer die ID eines ungenutzten Uploads errät, erhält dadurch keinen Zugriff.

Das Formular behandelt Upload- und Speicherzustand getrennt. Beim Speichern wird zunächst das Ende des Uploads abgewartet; danach werden die gewählte ID und die Beschreibung gesendet. Wird ein noch laufender Upload entfernt, bricht das Formular die Anfrage ab und ignoriert eine verspätete Antwort. Wird ein Artikel gespeichert, nachdem inzwischen ein anderes Bild gewählt wurde, erhält die Zusammenführung die neuere Auswahl – so wie Kapitel 15 neuere Texteingaben erhält.

## Jeden Bildabruf autorisieren

Eine Bild-URL darf die Sichtbarkeit von Entwürfen nicht umgehen. Unsere Richtlinie lautet:

| Aufrufer | Lesbare Bilder |
| --- | --- |
| Anonymer Besucher oder Leser | Bilder, auf die mindestens ein veröffentlichter Artikel verweist. |
| Administrator | Eigene Uploads und Bilder, auf die beliebige Artikel einschließlich Entwürfen verweisen. |

Im Leseweg von [MediaResource.image](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/MediaResource.scala), nach dem Parsen der UUID:

```scala
import jakarta.ws.rs.NotFoundException
import jakarta.ws.rs.core.Response

val media = manager.find(classOf[Media], id)
if (media == null ||
    (!lifecycle.referenced(media, publishedOnly = true) && !lifecycle.canUse(media)))
  throw new NotFoundException()

Response.ok(media.data, media.contentType)
  .header("Content-Length", media.byteSize)
  .header("Content-Disposition", "inline")
  .header("Cache-Control", "no-store")
  .header("X-Content-Type-Options", "nosniff")
  .build()
```

Ein verborgenes und ein nicht vorhandenes Bild liefern beide 404. Wird der letzte veröffentlichte Artikel zurückgezogen, der ein Bild verwendet, endet der öffentliche Zugriff bei nachfolgenden Anfragen. no-store verhindert, dass der Browser die Bytes als dauerhaft öffentliches Asset behandelt; eine bereits heruntergeladene Kopie kann es nicht löschen.

Auch die Transaktionsgrenze ändert sich. Bei JSON kann die Serialisierung noch Hibernate-Beziehungen benötigen, daher bleibt die Transaktion bis zum Writer geöffnet. Hier hat der Controller bereits ein begrenztes Array[Byte]. TransactionBoundary schließt die Transaktion vor dem Schreiben des Bildes ab und gibt die Datenbankverbindung frei, bevor ein langsamer Client den Download beendet.

Die öffentliche Seite verwendet dasselbe Komponenten-DSL wie der übrige Teil der Anwendung. Dieser Ausschnitt gehört in den vorhandenen render-Block von PostPage.compose:

```scala
import ui.core.dsl.ClassDsl.classes
import ui.core.layout.Condition.when
import ui.core.layout.Image

when(post.coverImage.map(_ != null)) {
  Image.image {
    classes = "post-cover"
    Image.src = post.coverImage.map(image => if (image == null) "" else image.source)
    Image.alt = post.coverAlt.map(value => Option(value).getOrElse(""))
  }
}
```

Media.source leitet die gleichursprüngliche Bild-URL aus einer validierten UUID ab. Die [vollständige Komponente](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostPage.scala) bindet auch die Abmessungen über das Attribut-DSL ein, damit Platz für das Bild reserviert wird.

## Zurückgelassene Uploads bereinigen

Ein Redakteur kann ein Bild hochladen und den Tab schließen, bevor er speichert. Alle unreferenzierten Bilder sofort zu löschen, würde mit noch aktiven Bearbeitungen kollidieren.

Unsere Bereinigung berücksichtigt nur Bilder, die vor mehr als 24 Stunden erstellt wurden und auf die kein Artikel verweist. Auch Referenzen aus Entwürfen zählen. Die Frist beginnt beim Upload, nicht erst dann, wenn ein Artikel später sein Titelbild entfernt.

Nach jedem erfolgreichen Upload wird ein kleiner Stapel von Bildern dieses Administrators geprüft. Der Betreiberbefehl verarbeitet einen begrenzten Stapel über alle Eigentümer hinweg:

```text
sbt --server "application-backend/runMain com.anjunar.blog.MediaCleanupMain"
```

Die Auswahl allein reicht nicht aus: Während die Bereinigung wartet, könnte eine andere Anfrage das Bild anhängen. Sowohl das Anhängen als auch die Bereinigung sperren dieselbe Medienzeile. Nach Erhalt der Sperre prüft die Bereinigung die Referenzen erneut, bevor sie löscht. [MediaLifecycle](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/application/backend/src/main/scala/com/anjunar/blog/MediaLifecycle.scala) implementiert dieses Verfahren; ein PostgreSQL-Konkurrenztest prüft den Wettlauf.

## Den vollständigen Ablauf ausprobieren

Wende die additive Migration an und baue das Frontend anhand des [Checkpoint-Leitfadens](https://github.com/anjunar/anjunar-blog-example/blob/c31d860175df373be9c4a82af806596a4ac7ba2c/docs/uploading-and-serving-media.md). Öffne einen Entwurf, lade ein JPEG oder PNG hoch, beschreibe es, speichere und lade die Seite neu. Veröffentliche den Artikel und rufe seine öffentliche Seite in einem abgemeldeten Browser auf. Ziehe den Artikel zurück und rufe das Bild erneut ab: Der Zugriff sollte jetzt verweigert werden.

Entferne schließlich das Titelbild und speichere. Der Artikel verliert seine Referenz; die Bereinigung kann das Bild später einsammeln, falls es sonst niemand verwendet. Damit steht ein vollständiger Medienlebenszyklus bereit, auf dem das nächste Kapitel mit Bildern in strukturierten Artikelinhalten aufbaut.
