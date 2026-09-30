# Blog-Inhalte übersetzen

Kapitel 20 hat die Oberfläche übersetzt. Jetzt soll ein Administrator einen
deutschen Artikel schreiben, zunächst privat speichern und anschließend
veröffentlichen können. Bis dahin sehen Leser unter `/de/posts/shared-slug`
weiterhin den veröffentlichten englischen Artikel. Beide Sprachen teilen
denselben Slug.

Wir verfolgen diese Änderung vom Datenmodell über das Speichern bis zur
öffentlichen Seite. Oberflächentexte verwenden weiterhin den `i18n`-Katalog;
Artikeltexte kommen aus der Datenbank. Die
[Checkpoint-Anleitung](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/docs/translating-blog-content.md)
enthält die Befehle zum Einrichten und Prüfen.

## Gemeinsame Identität und übersetzte Texte trennen

Englisch liegt bereits in `BlogPost`. Diese Felder bleiben bestehen; für Deutsch
ergänzen wir `BlogPostTranslation`. Dadurch erhält die Übersetzung eine eigene
Version und einen eigenen Veröffentlichungsstatus, ohne die vorhandenen
englischen Inhalte zu verschieben.

| Gespeichert in BlogPost | Gespeichert in BlogPostTranslation |
| --- | --- |
| Slug, Autor, Tags, Titelbild | Verpflichtende Beitragsreferenz und Sprache |
| Englischer Titel, Zusammenfassung und Inhalt | Deutscher Titel, optionale Zusammenfassung und Markdown-Inhalt |
| Version und Veröffentlichungsstatus der Quelle | Version und Veröffentlichungsstatus der Übersetzung |

Die [Translation-Entity und ihr Schema](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/BlogPostTranslation.scala)
definieren diesen Vertrag gemeinsam. Ein verpflichtendes `@ManyToOne` bildet
`post_id` ab; der eindeutige Schlüssel `(post_id, locale)` erlaubt nur eine
deutsche Zeile pro Beitrag. Ein eigenes `@Version` schützt deutsche Änderungen:
Wird der englische Titel bearbeitet, steigt die deutsche Version nicht mit.
Die Übersetzung beginnt mit `published = false`.

Das Mapper-Schema gibt `title`, `summary` und `content` über
`TranslationEditRule` zum Bearbeiten frei. Beitrag, Sprache, Version und
Veröffentlichung lassen sich nicht beliebig aus JSON zuweisen. Persistente Felder
verwenden typisierte `reference`-Properties; dadurch liefert dasselbe Schema die
Criteria-Attribute für die spätere Abfrage. Der Detail-Graph enthält ausdrücklich
`version`, und das Frontend-Modell erhält sie für den nächsten Request.

Bei gestoppter Anwendung führen wir `SchemaMain preview` aus, prüfen das Ergebnis
und starten `SchemaMain migrate`. Hinzu kommen die Übersetzungs- und
Bildreferenztabellen, Eindeutigkeitsbedingung und Fremdschlüssel. Die vorhandenen
englischen Zeilen bleiben bestehen.

## Eine Änderung vom Request bis zur Antwort verfolgen

Angenommen, ein gespeicherter deutscher Entwurf hat Version 0. Eine Titeländerung
sendet diesen Body an den `update`-Link aus der Antwort:

```json
{
  "version": 0,
  "title": "Eine gemeinsame Anwendung"
}
```

Die Adresse lautet
`PATCH /service/editorial/posts/{postId}/translations/{id}`.
Dabei bezeichnet `postId` den englischen Beitrag und `id` die Übersetzung.
Nicht mitgesendete Zusammenfassung und Inhalt bleiben unverändert. Die UUID bleibt
in der URL, die Methode erhält aber `post: BlogPost`: Unser
[EntityParamConverterProvider](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/EntityParamConverterProvider.scala)
lädt den verwalteten Beitrag im Persistence Context des Requests. Ungültige oder
fehlende Beiträge ergeben 404, bevor die Methode ausgeführt wird.

Hier stehen die Klassendeklaration, die vollständige Update-Methode und der
Antwortaufbau aus
[EditorialTranslationsResource](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/EditorialTranslationsResource.scala).
Die Methoden zum Lesen, Anlegen und Veröffentlichen stehen weiterhin in der
vollständigen Resource; die LinkBuilder-Ausdrücke unten beziehen sich auf sie.
`Data`, `Schema`, `EntityGraph` und die fachlichen Klassen der
Anwendung liegen im selben Package; externe Typen werden importiert.

```scala
package com.anjunar.blog

import com.anjunar.json.mapper.PreparedChange
import jakarta.annotation.security.RolesAllowed
import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager
import jakarta.ws.rs.{Consumes, NotFoundException, PATCH, Path, PathParam, Produces}
import jakarta.ws.rs.core.MediaType

import scala.compiletime.uninitialized
import scala.jdk.CollectionConverters.*

@Path("/editorial/posts/{postId}/translations")
@RolesAllowed(Array("ADMIN"))
@RequestScoped
@Produces(Array(MediaType.APPLICATION_JSON))
class EditorialTranslationsResource {
  @Inject var manager: EntityManager = uninitialized
  @Inject var media: PostMedia = uninitialized

  // Other endpoints remain in this class; see the complete resource.

  @PATCH @Path("/{id}") @Consumes(Array(MediaType.APPLICATION_JSON))
  @EntityGraph("BlogPostTranslation.detail")
  def update(@PathParam("postId") post: BlogPost,
      @PathParam("id") change: PreparedChange[BlogPostTranslation]): Data[BlogPostTranslation] = {
    val translation = change.getEntity()
    if (translation.post.id != post.id) throw new NotFoundException()
    change.applyChanges()
    media.synchronize(translation)
    result(translation)
  }

  private def result(value: BlogPostTranslation): Data[BlogPostTranslation] = {
    val self = LinkBuilder.create[EditorialTranslationsResource](_.read(value.post)).withRel("self").build()
    val actions = if (value.id == null) {
      Seq(LinkBuilder.create[EditorialTranslationsResource](_.create(value.post, null)).build())
    } else {
      val update = LinkBuilder.create[EditorialTranslationsResource](_.update(value.post, null))
        .withVariable("id", value.id).build()
      val publication = if (value.published) {
        Some(LinkBuilder.create[EditorialTranslationsResource](_.retract(value.post, null))
          .withVariable("id", value.id).build())
      } else if (PostDocument.hasContent(value.content, "MARKDOWN")) {
        Some(LinkBuilder.create[EditorialTranslationsResource](_.publish(value.post, null))
          .withVariable("id", value.id).build())
      } else None
      Seq(update) ++ publication
    }
    val links = (Seq(self) ++ actions).filter(_ != null).asJava
    new Data(value, Schema.forGraph(BlogPostTranslation.schema, manager.getEntityGraph("BlogPostTranslation.detail")), links)
  }
}
```

Hier treffen drei getrennte Aufgaben aufeinander:

1. **Berechtigen und vorbereiten.** Der bestehende Sicherheitsfilter erzwingt
   ADMIN und CSRF. Anschließend liest unser
   [PreparedChangeParamConverter](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/PreparedChangeProviders.scala)
   die ID aus dem Pfad und den JSON-Body. Er lädt und sperrt die Übersetzung,
   prüft die mitgesendete Version und bereitet die Mapper-Operation vor.
   Dies leistet unser registrierter Converter, nicht die eingebaute
   JAX-RS-Verarbeitung eines Request-Bodys.
2. **Die URL prüfen, dann Änderungen anwenden.** `getEntity()` liefert die
   verwaltete Entity, bevor die vorbereiteten Änderungen angewendet werden.
   Gehört die Übersetzung zu einem anderen Beitrag, folgt 404, auch wenn ihre
   ID ansonsten gültig ist. Der Vergleich ihrer Beitragsidentität mit `post.id`
   prüft diese Zuordnung, nicht die Eigentümerschaft eines Benutzers. `applyChanges()` wird einmal
   aufgerufen und wendet Feldregeln und Validierung des Mappers an.
3. **Abgeleiteten Zustand pflegen und die Antwort beschreiben.**
   `media.synchronize` liest das übersetzte Markdown und aktualisiert dessen
   interne Bildreferenzen einschließlich der Zugriffsprüfung.
   `result` verpackt dieselbe Entity mit Graph-Metadaten und aktuellen
   Aktionslinks. Keiner dieser Aufrufe führt einen Commit aus.

[LinkBuilder](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/LinkBuilder.scala) ist aus dem Referenz-Stack
übernommen und an das Begleitprojekt angepasst.
`create[EditorialTranslationsResource](_.read(value.post))` beschreibt für das
Makro einen Methodenaufruf; der Endpunkt wird dabei nicht ausgeführt. Der Builder
leitet Pfad und HTTP-Methode aus dessen Annotationen ab und bindet die ID der
Beitrags-Entity. Bei `update` steht `null` für die vorbereitete Änderung;
`withVariable("id", value.id)` liefert die ID der gespeicherten Übersetzung.
Beim Erzeugen der Links wird kein JSON-Body gelesen.

Der Builder prüft die Endpoint-Berechtigung des aktuellen Aufrufers. Die Resource
entscheidet anhand des Zustands, ob Anlegen, Veröffentlichen oder Zurückziehen
möglich ist. Die Endpunkte erzwingen diese Prüfungen weiterhin beim tatsächlichen
Request. Pfade und HTTP-Methoden werden dadurch einmal an den Resource-Methoden
definiert.

Im Controller steht kein `flush()`. Die vorhandene
[TransactionBoundary](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/TransactionBoundary.scala) flusht erfolgreiche
Schreibzugriffe **vor der JSON-Serialisierung**. Dabei werden ausstehende
SQL-Anweisungen ausgeführt, die gesamte Entity durch Hibernate validiert und
ihre Version erhöht. `Data` hält weiterhin die verwaltete Entity. Der Writer
sieht deshalb den aktualisierten Stand und keine zuvor angelegte Kopie.

Der Writer serialisiert in einen Puffer; die Transaktion wird abgeschlossen,
bevor dieser erfolgreiche Antwort-Body gesendet wird. Schlagen Validierung,
Serialisierung oder Commit fehl, greift der bestehende Fehler-/Rollback-Pfad.
Flush und Commit sind unterschiedliche Vorgänge. Ein zusätzlicher Flush in
jedem Endpunkt würde diesen gemeinsamen Ablauf verschleiern.

Für die geänderte Überschrift aus unserem Beispiel enthält die erfolgreiche
Antwort Version 1. Ein erneuter Request mit Version 0 liefert 409 und lässt den
gespeicherten Titel unverändert. Auch ein anschließendes Laden liefert dieselbe
Version wie die erfolgreiche Antwort.

Das Anlegen verwendet denselben Mapper-Pfad für eine neue Entity, setzt deren
Beitrag serverseitig und persistiert sie. Vor der Prüfung auf eine vorhandene
Übersetzung wird der Beitrag gesperrt: Zwei gleichzeitige Anlegeversuche dürfen
nicht beide erfolgreich sein. Veröffentlichen und Zurückziehen sind getrennte
Befehle mit ausschließlich der gespeicherten Version. Sie speichern keine noch
offenen Formulareingaben.

## Für Leser eine vollständige Sprache auswählen

Das Speichern eines deutschen Entwurfs darf ihn nicht öffentlich machen.
Der öffentliche Detail-Endpunkt lädt zunächst einen veröffentlichten Beitrag.
Dann ruft er `select(post, locale)` auf; die gewünschte Sprache wurde bereits
auf `en` oder `de` geprüft. Dies ist die vollständige Klasse
[PostLocalization](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/PostLocalization.scala):

```scala
package com.anjunar.blog

import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.persistence.EntityManager

import scala.compiletime.uninitialized

@RequestScoped
class PostLocalization {
  @Inject var manager: EntityManager = uninitialized

  // Only response-only properties change. The managed English fields remain untouched.
  def select(post: BlogPost, locale: String): Unit = {
    val builder = manager.getCriteriaBuilder
    val query = builder.createQuery(classOf[BlogPostTranslation])
    val translation = query.from(classOf[BlogPostTranslation])
    query.select(translation).where(
      builder.equal(translation.get(BlogPostTranslation.schema.post), post),
      builder.equal(translation.get(BlogPostTranslation.schema.locale), "de"),
      builder.equal(translation.get(BlogPostTranslation.schema.published), true))
    val german = Option(manager.createQuery(query).getSingleResultOrNull)
    BlogPost.schema.translation // Register the nested response field after both schemas exist.
    post.translation = if (locale == "de") german.orNull else null
    post.contentLocale = if (post.translation == null) "en" else "de"
    post.availableLocales.clear()
    post.availableLocales.add("en")
    if (german.nonEmpty) post.availableLocales.add("de")
  }
}
```

Die Abfrage berücksichtigt nur die veröffentlichte deutsche Zeile. Bei einer
deutschen Anfrage wird sie zur ausgewählten `translation`. Andernfalls bleibt
die Auswahl null und `contentLocale` ist Englisch. Dass auch englische Requests
nach Deutsch suchen, ermöglicht die Angabe aller veröffentlichten Sprachen.

Alle Zuweisungen betreffen transiente Antwortfelder. Wir überschreiben niemals
den verwalteten englischen Titel oder Inhalt mit deutschem Text. Das Frontend
liest die verschachtelte Übersetzung, falls vorhanden, und sonst die
ursprünglichen Felder.

`BlogPost.schema.translation` registriert die verzögert initialisierte,
verschachtelte Antwort-Property, nachdem beide Persistenzschemas existieren.
Andernfalls würde JSON Mapper 1.1.6 die Schemas von Beitrag und Übersetzung
zyklisch auflösen. Die [Anleitung](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/docs/translating-blog-content.md)
erläutert dieses Initialisierungsdetail; die Auswahlregel ändert sich dadurch nicht.

| Beitrag | Deutsche Übersetzung | Deutsche öffentliche Seite |
| --- | --- | --- |
| Veröffentlicht | Veröffentlicht | Vollständiger deutscher Artikel |
| Veröffentlicht | Fehlend oder Entwurf | Vollständiger englischer Artikel mit Hinweis |
| Entwurf | Beliebiger Zustand | 404 |

Der Fallback wählt einen ganzen Artikel. Eine bewusst fehlende deutsche
Zusammenfassung bleibt leer; eine englische Zusammenfassung würde hier die
Sprachen mischen. Der redaktionelle Endpunkt verwendet keinen Fallback:
Er liefert den tatsächlichen deutschen Entwurf oder ein leeres neues Formular.

## Das Formular binden und seine Inhaltssprache angeben

[TranslationEditorPage](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/frontend/src/main/scala/com/anjunar/blog/frontend/TranslationEditorPage.scala) erhält
den englischen `post`, die geladenen `initial: TranslationData` und ihre
HTTP-/Mediendienste. Sie erzeugt
`actions = new TranslationActions(initial, service.send)` und bindet
`translation = actions.translation`. Die Actions verwalten Speicherzustand,
Serverfehler und die Übernahme bestätigter Werte.

Dieser zusammenhängende Ausschnitt beginnt bei `form(translation)` innerhalb
von `compose`/`render`. Er enthält das Absenden, die Bindung von Serverfehlern,
das Titelfeld und dessen Fehlerausgabe. Zusammenfassung, Markdown und Buttons
folgen im selben Formular der verlinkten Komponente.

```scala
import ui.core.dsl.AttributeDsl.{setAttribute as attr}
import ui.core.dsl.AttributeDsl.*
import ui.core.dsl.ClassDsl.classes
import ui.core.dsl.EventDsl.on
import ui.core.i18n.i18n
import ui.core.layout.Div.div
import ui.core.layout.Label.label
import ui.core.layout.Paragraph.paragraph
import ui.core.layout.TextComponent.text
import ui.forms.ErrorResponse
import ui.forms.Form.form
import ui.forms.Input.input

import scala.scalajs.js

// Inside TranslationEditorPage.compose, within render(this, cursor):
form(translation) { mountedForm ?=>
  classes = "post-form"
  attr("novalidate", "")
  mountedForm.addDisposable(actions.errors.observe(values =>
    mountedForm.setErrorResponses(values.map(value => ErrorResponse(value.message, value.path)))))
  on("submit") { event =>
    event.preventDefault()
    if (!actions.busy.get && !actions.blocked.get && upload.get.pending == 0) {
      mountedForm.clearErrors()
      actions.generalError.set("")
      if (mountedForm.validateBindings().nonEmpty) actions.notice.set(SaveNotice.BindingFailed)
      else if (mountedForm.validate().nonEmpty) actions.notice.set(SaveNotice.Invalid)
      else actions.save()
    }
  }
  div {
    classes = "post-field"
    label {
      attr("for", "translation-title")
      text(i18n"Title") {}
    }
    val control = input("title") { fieldInput ?=>
      attr("aria-invalid", fieldInput.invalid.map(_.toString))
      id = "translation-title"
      lang = translation.locale.get
      attr("aria-describedby", "translation-title-errors")
    }
    paragraph {
      id = "translation-title-errors"
      classes = "field-error"
      text(control.errors.map((values: js.Array[String]) => values.mkString(", "))) {}
    }
  }
  // Summary, Markdown editor and submit/publication controls continue here.
}
```

`input("title")` bindet direkt an die Model-Property. `i18n"Title"` im Label
folgt der Oberflächensprache. Dagegen verwendet
`lang = translation.locale.get` die benannte Attribut-DSL, um die Sprache des
eingegebenen Artikeltexts anzugeben. Diese Sprache steht für die geladene
Übersetzung fest; daher ist das einmalige Lesen hier passend. Ein Wechsel der
Oberflächensprache baut die Seite neu auf, ändert aber nicht die bearbeitete
Übersetzung.

Eine englische Oberfläche zeigt somit **Title** an einem deutschen Eingabefeld.
Eine deutsche Oberfläche zeigt **Titel**, weiterhin am selben deutschen Feld.
Der englische Quellbereich verwendet `lang = "en"`. Öffentliche Überschriften
und Inhalte verwenden die ausgewählte `contentLocale`, auch beim englischen
Fallback in einer deutschen Oberfläche. Das HTML-Attribut beschreibt die Sprache;
es übersetzt keinen Text.

Der Import benennt `setAttribute` aus der DSL in `attr` um und vermeidet damit
die gleichnamige geerbte Komponenten-Methode. So gehört
`attr("novalidate", "")` zum Formularknoten und `attr("for", ...)` zum Label.
Das ist ein Importalias, keine selbst gebaute Attribut-Hilfsmethode.
Die Bindung von `aria-invalid` übergibt die reaktive Property direkt an die DSL;
diese verwaltet Beobachtung und Aufräumen.

Vor dem Speichern prüft das Formular Binding- und Modellfehler. Die Actions
halten geänderte Werte und Version vor der asynchronen Arbeit fest. Eine verspätete
Antwort aktualisiert Version, Links und gespeicherte Ausgangswerte, erhält aber
zwischenzeitlich eingegebenen Text. Bei einem Konflikt bleibt die Eingabe sichtbar;
ein ausdrückliches Neuladen ersetzt blinde Wiederholungsversuche. Ungespeicherte
Eingaben oder laufende Uploads sperren außerdem Veröffentlichung und Sprachwechsel.

## Suche und Medien konsistent halten

Eine deutsche Suche muss den Text finden, den Leser anschließend sehen.
Erst nach der Seiteneinteilung auf Deutsch umzuschalten, würde Sortierung und
Gesamtzahl verfälschen.
[LocalizedPostFields](https://github.com/anjunar/anjunar-blog-example/blob/2accb8e1b7e0cb2044a0d57c13782e56fda3a563/application/backend/src/main/scala/com/anjunar/blog/LocalizedPostFields.scala) versorgt die
bestehende `HibernateSearch` mit einem Left Join auf die veröffentlichte
Übersetzung und gemeinsamen CASE-Ausdrücken für Titel und Zusammenfassung.
Filter, Sortierung und Projektion wählen Deutsch, sobald diese Zeile existiert;
die Zählabfrage baut dasselbe Prädikat auf einem eigenen Root auf.
Seitenlinks behalten die Sprache. Eine deutsche Null-Zusammenfassung bleibt null.

Auch Bilder folgen der Veröffentlichung: Ein nur im deutschen Entwurf
referenziertes Bild bleibt privat, selbst wenn sein englischer Beitrag öffentlich
ist. Beide Veröffentlichungszustände müssen aktiv sein, damit diese Referenz das
Bild öffentlich zugänglich macht. Gespeicherte Entwürfe schützen ihre Bilder
weiterhin vor der Bereinigung.

## Den vollständigen Ablauf prüfen

Speichere einen deutschen Entwurf für einen veröffentlichten englischen Beitrag
und lade den Editor neu. Die deutsche öffentliche URL sollte weiterhin Englisch
zeigen. Veröffentliche die Übersetzung, suche nach ihrem deutschen Titel und
ziehe sie wieder zurück. Anschließend muss der englische Fallback erscheinen.
Wiederhole den Editor-Ablauf in beiden Oberflächensprachen.

Die automatisierten Prüfungen decken zusätzlich veraltete Versionen, fremde
Beitrags-IDs in der URL, Validierungs-Rollback, Berechtigungen, gleichzeitiges
Anlegen und Bildbereinigung ab. Die Checkpoint-Anleitung enthält die Befehle.

Das Bearbeiten einer bereits veröffentlichten Übersetzung ändert den öffentlichen
Text sofort; für private Überarbeitungen muss sie zuvor zurückgezogen werden.
Übersetzungen bleiben manuell, englische Änderungen machen Deutsch nicht
automatisch ungültig, und Slug sowie gemeinsame Metadaten bleiben geteilt.

Kapitel 22 rendert diese lokalisierten Seiten anschließend auf dem Server.
