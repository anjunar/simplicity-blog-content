# Einen Artikel-Editor integrieren

Unser Formular speichert einen Artikel bereits sicher. Jetzt möchten wir einen richtigen Beitrag schreiben: mit Überschriften, hervorgehobenem Text, Codebeispielen und Bildern zwischen den Absätzen.

Wir ersetzen das Inhalts-Textfeld durch den Editor aus scalajs-ui. Interessant ist, wie sich dieser Editor in das bereits aufgebaute System einfügt. Formatierungen müssen Speichern und Neuladen überstehen, neuere Eingaben müssen eine langsame Antwort überdauern, und eingebettete Bilder müssen der Sichtbarkeit des Artikels folgen.

Der [Checkpoint zu diesem Kapitel](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/docs/integrating-a-post-editor.md) enthält die lauffähige Implementierung und die Upgrade-Befehle. Er verwendet scalajs-ui 1.0.12 von Maven Central mit den englischen Bedienelementen des Editors. Der Scala.js-Build zielt auf ES2021, damit der Markdown-Parser die benötigten regulären Ausdrücke verwenden kann.

## Die Dokumentgrenze festlegen

Während der Bearbeitung verwaltet der Editor ein strukturiertes Dokument. Sein öffentlicher Wert ist Markdown: eine Zeichenfolge, die unser bestehendes Formular, der JSON-Mapper und die Datenbank speichern können. BlogPost.content bleibt erhalten; wir ergänzen ein Feld, das Lesern mitteilt, wie der Inhalt interpretiert werden soll.

In [BlogPost](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/backend/src/main/scala/com/anjunar/blog/BlogPost.scala):

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
import jakarta.json.bind.annotation.JsonbProperty
import jakarta.persistence.Column
import jakarta.validation.constraints.Pattern

@Pattern(regexp = "PLAIN_TEXT|MARKDOWN")
@Column(name = "content_format", length = 16)
@SchemaId("ad191001") @JsonbProperty
var contentFormat: String = "PLAIN_TEXT"
```

Die Spalte darf null sein. Historische Datensätze haben kein Format und werden weiterhin als einfacher Text interpretiert. Ein Absatz mit *Sternchen* soll nach einer Bereitstellung nicht plötzlich Hervorhebungen enthalten. Die Migration ergänzt die Spalte, ohne vorhandene Inhalte umzuschreiben.

Neue Artikel übermitteln ausdrücklich MARKDOWN. Bestehende Artikel behalten zunächst ihr Textfeld und bieten **Rich-Text aktivieren** an. Dabei werden Markdown-Satzzeichen maskiert und Zeilenumbrüche beibehalten, bevor das Format gewechselt wird. Die Umwandlung ist eine redaktionelle Entscheidung.

Wie bei früheren Feldern betrifft die Änderung den gesamten Vertrag: EntitySchema, Detailgraph, Frontend-Eigenschaft und Speicher-Snapshots. Listen verwenden weiterhin ihre kompakte Projektion; sie benötigen nicht das ganze Dokument.

## Den Editor an den Artikel binden

Wir haben bereits form(post), versionierte Speicherungen und einen Weg für Feldfehler. Der Editor ist ein weiteres Formularelement. Dieser Ausschnitt gehört in das bestehende Formular von [PostEditorPage.compose](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostEditorPage.scala):

```scala
import ui.editor.Editor.*
import ui.editor.plugins.*

editor("content") {
  showModeActions = false
  markdownMode = sourceMode
  mediaUrlPolicy = PostMarkdown.mediaPolicy
  mediaUploader = PostMarkdown.uploader(mediaService)
  onMediaStatus = status => embeddedUpload.set(status)
  basePlugin()
  headingPlugin()
  listPlugin()
  linkPlugin()
  imagePlugin()
  codePlugin()
}
```

Der Name content bindet den Editor direkt an post.content. Ein zweites Editormodell oder ein zusätzlicher HTTP-Kopierschritt ist nicht nötig. Die vollständige Komponente stellt außerdem die zugängliche Beschriftung, den Hilfetext und die gebundenen Fehler bereit.

Die ausgewählten Plugins bestimmen die Werkzeugleiste: einfache Formatierungen, Überschriften, Listen, Links, Bilder und Codeblöcke. Eine lokale Schaltfläche **Markdown bearbeiten** ändert die sourceMode-Eigenschaft; **Visueller Editor** schaltet zurück, ohne das Formular zu verlassen. Das ist praktisch, um ein eingerahmtes Scala-Codebeispiel einzufügen oder das Dokument zu prüfen, das gespeichert wird.

Das Speicherverhalten aus Kapitel 15 gilt weiterhin. Vor Beginn der Anfrage halten wir den eingereichten Markdown-Text fest. Tippt der Autor währenddessen weiter, aktualisiert die Antwort die Version, ersetzt aber nur Felder, die seit dem Absenden unverändert geblieben sind. Die neue Formateigenschaft wird dabei ebenfalls verglichen.

Codeblöcke speichern Code als Text. Dieses Kapitel ergänzt keinen Syntax-Highlighter; spitze Klammern in einem Beispiel bleiben Teil des Beispiels.

## Den Upload-Dienst wiederverwenden

Ein Bild in Markdown braucht eine dauerhafte Adresse. Eine Browser-Objekt-URL funktioniert nicht mehr, sobald die Seite geschlossen ist. Würde die Datei in das Dokument eingebettet, könnten unsere Upload-Grenzen und Besitzprüfungen umgangen werden.

Der Editor akzeptiert einen MediaUploader. Unser Adapter delegiert an den Dienst aus Kapitel 18, der die eigentliche Datei mit dem CSRF-Token der Sitzung übermittelt:

```scala
import org.scalajs.dom
import ui.editor.{MediaUploader, UploadedMediaReference}
import scala.concurrent.{ExecutionContext, Future}

// Inside PostMarkdown, in the frontend package.
def uploader(service: MediaService)(using ExecutionContext): MediaUploader =
  new MediaUploader {
    def upload(file: dom.File, signal: dom.AbortSignal)
        : Future[UploadedMediaReference] =
      service.upload(file, signal).map(media =>
        UploadedMediaReference(media.source, media.id.get))
  }
```

Das Ergebnis verweist auf das gespeicherte Bild und seine Adresse /service/media/{UUID}. Eine passende MediaUrlPolicy erlaubt nur diese Route. Dieselbe Richtlinie gilt beim Anzeigen eines Dokuments, sodass extern gehostete Bilder nicht unbemerkt Tracking-Anfragen auslösen können.

Solange ein Upload läuft, sind Speichern und der Wechsel in den Quelltextmodus deaktiviert. Nach dem Einfügen kann der Autor das Bild auswählen und über **Bild bearbeiten** einen Alternativtext ergänzen. Der Server verlangt eine aussagekräftige Beschreibung, selbst wenn ein anderer Client das Formular umgeht.

Ein Upload allein erzeugt weiterhin private, noch nicht referenzierte Medien. Erst beim Speichern des Artikels wird das Medium angehängt.

## Referenzen aus dem Dokument ableiten

Die Markdown-Syntax für Bilder ist keine vertrauenswürdige Entitätsreferenz. Der Server muss ermitteln, welche Bilder das Dokument tatsächlich verwendet und ob der Aufrufer sie anhängen darf.

[PostDocument](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/backend/src/main/scala/com/anjunar/blog/PostDocument.scala) analysiert die Quelle mit CommonMark. Es sammelt UUIDs aus Bildknoten, prüft Alternativtexte, weist rohes HTML zurück und begrenzt Linkziele. Die Analyse ist wichtig: Text, der innerhalb eines Codeblocks wie eine Bildreferenz aussieht, ist ein Beispiel und kein Anhang.

Nach change.applyChanges() ruft die Ressource [PostMedia.synchronize](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/backend/src/main/scala/com/anjunar/blog/PostMedia.scala) auf. Dieser Ausschnitt löst die erkannten IDs auf, bevor die gespeicherte Beziehung ersetzt wird:

```scala
import jakarta.persistence.LockModeType

// ids comes from PostDocument; manager and lifecycle are injected.
val resolved = ids.toSeq.sortBy(_.toString).map { id =>
  val media = manager.find(classOf[Media], id, LockModeType.PESSIMISTIC_WRITE)
  if (media == null || !lifecycle.canUse(media))
    Problem.invalidField("content",
      "An embedded image is unavailable or belongs to another account.")
  media
}

post.inlineMedia.clear()
resolved.foreach(post.inlineMedia.add)
```

Die Beziehung liegt in blog_post_media. Sie ist abgeleiteter Serverzustand und wird weder in JSON ein- noch ausgegeben. Der Client sendet das Dokument, nicht eine zweite Liste, die dazu im Widerspruch stehen könnte. Ungültige oder unzugängliche Referenzen weisen die gesamte Änderung zurück; Text, Format und Beziehungen werden gemeinsam übernommen.

Die Sperre der Medienzeile ist dieselbe, die auch die Bereinigung verwendet. Nach dem Sperren prüft die Bereinigung Referenzen erneut. Sowohl Titelbilder als auch eingebettete Bilder schützen nun einen Upload, auch in Entwürfen. Gemeinsam verwendete Medien werden nicht kaskadierend gelöscht.

Der Parser begrenzt Größe und Komplexität des Dokuments: 100.000 Zeichen, 10.000 Knoten, Tiefe 32 und höchstens 20 verschiedene Bilder. Der JSON-Mapper validiert weiterhin die Feldbeschränkungen der Eingabe, und Hibernate-Callbacks prüfen Invarianten des vollständigen Objekts. Zur Veröffentlichung braucht es außerdem aussagekräftigen Inhalt; ein Dokument, das nur aus einer horizontalen Linie besteht, genügt nicht.

## Dieselbe Leseansicht verwenden

Die redaktionelle Vorschau und der öffentliche Artikel verwenden beide [PostContent](https://github.com/anjunar/anjunar-blog-example/blob/618af6fb722bdc2d9ccf07c9d612b1a40afa5fad/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostContent.scala). Bei Markdown wird derselbe Editor im Nur-Lesen-Modus und mit derselben Bildrichtlinie eingebunden. Alte Inhalte werden als wörtlicher Text dargestellt.

Es gibt weder eine öffentliche Werkzeugleiste noch eine bearbeitbare Fläche. Vor allem verwenden Vorschau und Veröffentlichung keine getrennten Renderer, die Codeblöcke oder Bilder mit der Zeit unterschiedlich darstellen könnten.

Die Bildauslieferung behält die Zugriffsregeln aus Kapitel 18 bei. Nicht angemeldete Leser erhalten Bilddaten nur, solange ein veröffentlichter Artikel das Bild referenziert. Wird der Artikel zurückgezogen, werden nachfolgende anonyme Anfragen abgewiesen – außer ein anderer veröffentlichter Artikel verwendet dasselbe Bild. Die Markdown-URL selbst gewährt keinen Zugriff.

Der Browsereditor und der Serverparser haben unterschiedliche Aufgaben. Unterstützt werden Überschriften, Listen, Links, Code und Bilder, die der konfigurierte Editor erzeugt. Beliebige Markdown-Erweiterungen und rohes HTML gehören nicht zu diesem Vertrag.

## Einen Artikel durch den ganzen Ablauf verfolgen

Führe nach der additiven Migration einen Artikel an und formatiere einen Absatz. Wechsle zu Markdown, füge ein eingerahmtes Codebeispiel ein und kehre dann zum visuellen Editor zurück. Lade ein Bild hoch, beschreibe es, speichere und lade die Seite neu.

Öffne die Vorschau, veröffentliche den Artikel und lies die öffentliche Seite in einem abgemeldeten Browser. Formatierung, Code und Bild sollten übereinstimmen. Ziehe den Artikel zurück und rufe das Bild erneut ab: Die Antwort sollte nun 404 lauten.

Der Browserablauf des Checkpoints führt diese Schritte mit PostgreSQL aus. Weitere Tests decken die ausdrückliche Umwandlung alter Inhalte, neue Eingaben während eines langsamen Speichervorgangs, Dokumentfehler, private Bildreferenzen und die Bereinigung ab. Der Editor verwendet nun dieselbe Persistenz, dieselben Berechtigungen und denselben Lebenszyklus wie der Rest der Anwendung.
