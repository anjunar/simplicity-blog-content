# Die öffentlichen Seiten vervollständigen

Der Blog liefert inzwischen lesbares HTML und hydriert es, ohne die Seite zu
ersetzen. Eine letzte Schnittstelle fehlt noch: Was sagt eine URL Suchmaschinen,
Linkvorschauen und Feedreadern über ihren Inhalt?

Ein englischer Artikel innerhalb der deutschen Oberfläche bleibt ein englischer
Artikel. Ein Suchergebnis ist nicht dieselbe Seite wie die ungefilterte Übersicht.
Ein fehlender Beitrag muss 404 zurückgeben, auch wenn seine Fehlerseite gut aussieht.

Dieses letzte Kapitel macht diese Unterschiede ausdrücklich. Wir ergänzen
Metadaten, Canonical- und Sprachlinks, eine Sitemap, sprachabhängige Atom-Feeds
und dauerhafte Weiterleitungen für die unterstützten URL-Aliase.

## Jede öffentliche Seite erhält eine eindeutige Adresse

Dafür verwenden wir folgende Regeln:

| Aufruf | Canonical und Auffindbarkeit |
| --- | --- |
| Englischer Artikel | Englische Canonical-URL; deutscher Sprachlink nur bei veröffentlichter Übersetzung. |
| Veröffentlichter deutscher Artikel | Deutsche Canonical-URL; gegenseitige Links zwischen beiden veröffentlichten Sprachen. |
| Deutsche URL mit englischem Ersatzinhalt | Englische Canonical-URL; kein deutscher Artikeleintrag in Sitemap oder Feed. |
| Normale Übersicht und reguläre Folgeseiten | Jeweils eigene Canonical-URL; Folgeseiten behalten ihren Offset. |
| Suche, andere Sortierung oder Seitengröße | Normalisierte eigene Canonical-URL mit `noindex, follow`. |
| Fehlender Artikel oder unbekannte Route | HTTP 404 und `noindex`; keine Canonical-URL eines Artikels. |

Canonical benennt die bevorzugte Adresse für doppelte Inhalte. Sprachlinks
benennen veröffentlichte Varianten. Beides erfüllt unterschiedliche Aufgaben.
Unsere Sprachlinks enthalten die aktuelle Variante und Rückverweise auf die
anderen; Englisch ist außerdem das Ziel für `x-default`. Die Grundlagen stehen
in den Hinweisen zu [Canonical-URLs](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)
und [Sprachvarianten](https://developers.google.com/search/docs/specialty/international/localized-versions).

Alle absoluten Adressen beginnen mit `BLOG_PUBLIC_ORIGIN`. Das Backend prüft
diese Einstellung und übergibt sie an SSR und die Browser-Hülle. Der Canonical-Host
wird nicht aus weitergereichten Request-Headern abgeleitet. Lokal gilt standardmäßig
`http://127.0.0.1:<BLOG_PORT>`; eine gehostete Installation setzt ihre HTTPS-Adresse.

Die Detail-API liefert bereits `contentLocale`, `availableLocales` und die
ausgewählte `translation`. Die folgenden Methoden aus dem
[PageHead-Companion](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/application/frontend/src/main/scala/com/anjunar/blog/frontend/PageHead.scala) machen daraus Head-Einträge.
Sie gehören in das bestehende Companion-Objekt. Die beiden kleinen Hilfsmethoden
sind mit abgebildet, damit jeder erzeugte Eintrag nachvollziehbar bleibt:

```scala
package com.anjunar.blog.frontend

import ui.core.document.HeadEntry

// These methods belong to the PageHead companion object.
  private def feed(origin: String, locale: String): HeadEntry =
    HeadEntry("feed", "link", Seq("rel" -> "alternate", "type" -> "application/atom+xml",
      "title" -> s"Anjunar Journal ($locale)", "href" -> s"$origin/$locale/feed.xml"))

  private def social(title: String, description: String, url: String, kind: String): Seq[HeadEntry] = Seq(
    HeadEntry.title(s"$title — Anjunar Journal"),
    HeadEntry.meta("description", description),
    HeadEntry.property("og:title", title),
    HeadEntry.property("og:description", description),
    HeadEntry.property("og:type", kind),
    HeadEntry.property("og:url", url),
    HeadEntry.property("og:site_name", "Anjunar Journal"))

  def article(origin: String, post: BlogPost): Seq[HeadEntry] = {
    val translation = Option(post.translation.get)
    val title = translation.map(_.title.get).getOrElse(post.title.get)
    val summary = translation.map(_.summary.get).getOrElse(post.summary.get)
    val description = Option(summary).filter(_.nonEmpty).getOrElse(title)
    val locale = post.contentLocale.get
    val canonical = s"$origin/$locale/posts/${post.slug.get}"
    val languages = (Seq("en") ++ post.availableLocales.toSeq.filter(_ == "de")).distinct
    val alternates = languages.map(language =>
      HeadEntry.alternate(language, s"$origin/$language/posts/${post.slug.get}")) :+
      HeadEntry.alternate("x-default", s"$origin/en/posts/${post.slug.get}")
    val image = Option(post.coverImage.get).toSeq.flatMap(value => Seq(
      HeadEntry.property("og:image", origin + value.source),
      HeadEntry.property("og:image:alt", Option(post.coverAlt.get).getOrElse(""))))
    social(title, description, canonical, "article") ++ Seq(
      HeadEntry.link("canonical", canonical), HeadEntry.meta("robots", "index, follow"),
      HeadEntry.property("og:locale", if (locale == "de") "de_DE" else "en_US"), feed(origin, locale)) ++
      post.publishedAt.get.toSeq.map(value => HeadEntry.property("article:published_time", value)) ++ alternates ++ image
  }
```

Die Auswahl der Kurzbeschreibung ist entscheidend. Hat die deutsche Übersetzung
keine Zusammenfassung, verwenden wir ihren deutschen Titel. Die englische
Zusammenfassung wird nicht übernommen. Die Auswahl einer vollständigen
Sprachfassung aus Kapitel 21 gilt damit auch für die Beschreibung.

`HeadEntry` übernimmt das Escaping von Attributen und Text. Die Bildadresse
verweist auf den vorhandenen Medienendpunkt; dessen Zugriffsregeln gelten weiter.
Das sind Dokumentlinks. REST-Aktionslinks entstehen weiterhin über `LinkBuilder`.

## Metadaten folgen dem Lebenszyklus der Komponente

Ein korrekter anfänglicher Head reicht nicht. Nach dem Wechsel vom Artikel zur
Übersicht müssen Artikeltitel, Bild und Veröffentlichungsmetadaten verschwinden.
Sonst beschreiben URL und Vorschau unterschiedliche Seiten.

Die vorhandene `DocumentHead`-Registry gibt Einträgen stabile Schlüssel.
Registrierungen können denselben Schlüssel überschreiben; beim Entfernen wird
der vorherige Wert wieder sichtbar. Auf dem Server schreibt der Head-Sink des
Dokuments diese Registry in das HTML.

Unser Browser hydriert nur `#app`. Der Dokumentkopf liegt deshalb außerhalb
dieses Cursors. Die folgende vollständige Klasse `PageHead` verwendet den
`BrowserHeadSink` der Bibliothek über dessen `HeadSink`-Schnittstelle. Sie
synchronisiert die Registry, wenn eine Komponente Einträge hinzufügt oder entfernt:

```scala
import ui.core.component.AbstractComponent
import ui.core.document.{DocumentHead, HeadEntry, HeadSink}

final class PageHead(val origin: String, registry: DocumentHead, sink: Option[HeadSink]) {
  def bind(entries: HeadEntry*)(using owner: AbstractComponent): Unit = {
    val handle = registry.handle()
    handle.set(entries*)
    synchronize()
    owner.addDisposable(() => {
      handle.dispose()
      synchronize()
    })
  }

  private def synchronize(): Unit =
    sink.foreach(_.update(registry.entries, registry.htmlAttributes))
}
```

`BlogPage.compose` stellt eine Instanz über den Komponentenkontext bereit.
Ihr Standard besteht aus dem Journal-Titel und `noindex, follow`. Eine geladene
öffentliche Seite überschreibt diesen Standard. `PostPage.compose` ruft
`head.bind(PageHead.article(...)*)` auf; Listen- und Fehlerkomponente binden
ihre eigenen Einträge. Die vollständige
[Einrichtung in der Wurzelkomponente](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala) und die
[Artikelkomponente](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostPage.scala) zeigen diese Aufrufe im Zusammenhang.

Entsorgt der Router eine Artikelkomponente, entfernt der Callback deren
Registrierungen und aktualisiert den Browser-Head. Beim Wechsel zur Kontoseite
kehrt dadurch der Standard zurück, und der Artikel-Canonical verschwindet.
Der UI-Baum bleibt zusammenhängend in `compose`; weder Rendering-Fragmente
noch manuelle DOM-Attributzuweisungen sind dafür nötig.

Die übersetzte Listenbeschreibung verwendet das bestehende `i18n`-Makro.
Head-Einträge erwarten Strings; an dieser Schnittstelle wird die Nachricht
einmal für die Sprache der Route aufgelöst. Ein Sprachwechsel erzeugt die neue
Routenkomponente mit ihren neuen Einträgen.

## Veröffentlichte Daten für Sitemap und Feed verwenden

Sitemap und Feeds sind Datenbanksichten auf veröffentlichte Inhalte. Ihr Inhalt
darf nicht davon abhängen, welche Artikel zufällig auf einer gerenderten
Übersichtsseite stehen.

[PublishedPages](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/application/backend/src/main/scala/com/anjunar/blog/PublishedPages.scala) liest über Criteria und
die Entity-Schemas eine begrenzte Projektion. Der Elternbeitrag muss
`PUBLISHED` sein; der deutsche Join verlangt zusätzlich eine veröffentlichte
Übersetzung. Der deutsche Feed enthält keine englischen Ersatzartikel.
Weder ein Übersetzungsentwurf noch eine veröffentlichte Übersetzung unter
einem Entwurfsbeitrag wird dadurch öffentlich.

Die Endpunkte sind:

| URL | Inhalt |
| --- | --- |
| `/sitemap.xml` | Beide Übersichten und alle veröffentlichten kanonischen Artikelvarianten. |
| `/en/feed.xml` | Die neuesten 20 englischen Artikel. |
| `/de/feed.xml` | Die neuesten 20 veröffentlichten deutschen Übersetzungen, nach Veröffentlichungszeit des Elternbeitrags sortiert. |
| `/robots.txt` | Die Adresse der Sitemap. |

Sprachalternativen stehen im HTML-Head. Wir pflegen keine zweite
hreflang-Implementierung in der Sitemap. Das Tutorial begrenzt eine Sitemap auf
5.000 Ausgangsbeiträge. Darüber antwortet sie mit 503, statt Artikel still
wegzulassen. Ein Sitemap-Index für größere Installationen liegt außerhalb der Reihe.

Änderungszeiten brauchen gespeicherte Daten. Wir ergänzen dieses nullable Feld
samt Callback in `BlogPost`. `BlogPostTranslation` erhält denselben Aufbau mit
einer eigenen Schema-ID:

```scala
import com.anjunar.hibernateddl.hibernate.annotation.SchemaId
import jakarta.json.bind.annotation.JsonbTransient
import jakarta.persistence.{Column, PrePersist, PreUpdate}
import java.time.Instant

// Fields and callback in the body of BlogPost.
  @Column(name = "updated_at") @SchemaId("ab240001") @JsonbTransient
  var updatedAt: Instant = null

  @PrePersist @PreUpdate
  def recordChange(): Unit = updatedAt = Instant.now()
```

Hibernate setzt den Zeitpunkt beim Einfügen oder Ändern einer Zeile. Schlägt die
Transaktion fehl, wird er mit den übrigen Änderungen zurückgerollt. Bestehende
Zeilen behalten null, weil ihre Bearbeitungshistorie nicht rekonstruierbar ist.
Bei unbekanntem Zeitpunkt lässt die Sitemap `lastmod` weg. Für eine übersetzte
Seite können bekannte Änderungen am Ausgangsbeitrag oder an der Übersetzung
maßgeblich sein. Die Zeitstempel beschreiben Änderungen dieser Datenbankzeilen,
keine vollständige Historie separat verwalteter Medien, Tags oder Kontonamen.

Atom verlangt eine Änderungszeit. Alte Einträge verwenden zunächst die
Veröffentlichungszeit, bis eine neue Zeilenänderung erfasst wurde. Neue Änderungen
aktualisieren `updated`. Als Eintrags-ID dient die UUID des Beitrags beziehungsweise
der Übersetzung; ein geänderter Slug erzeugt daher keine neue Feed-Identität.
Das [Sitemap-Protokoll](https://www.sitemaps.org/protocol.html) und die
[Atom-Spezifikation](https://www.rfc-editor.org/rfc/rfc4287) definieren die Formate.

Hier ist die vollständige Resource. Feste Aliase in `FrontendHandler` leiten
die öffentlichen URLs in die normale JAX-RS-Request-Verarbeitung:

```scala
package com.anjunar.blog

import jakarta.annotation.security.PermitAll
import jakarta.enterprise.context.RequestScoped
import jakarta.inject.Inject
import jakarta.servlet.ServletContext
import jakarta.ws.rs.{GET, Path, PathParam, Produces}
import jakarta.ws.rs.core.{Context, Response}

import java.nio.charset.StandardCharsets.UTF_8
import scala.compiletime.uninitialized

@PermitAll
@RequestScoped
@Path("/discovery")
class DiscoveryResource {
  @Inject var pages: PublishedPages = uninitialized
  @Context var servlet: ServletContext = uninitialized

  private def origin: String = servlet.getAttribute(PublicSite.OriginAttribute).asInstanceOf[String]

  private def response(body: String, mediaType: String): Response =
    Response.ok(body.getBytes(UTF_8), mediaType).header("Cache-Control", "no-store")
      .header("X-Content-Type-Options", "nosniff").build()

  @GET @Path("/sitemap") @Produces(Array("application/xml"))
  def sitemap(): Response = response(DiscoveryXml.sitemap(origin, pages.sitemap()), "application/xml; charset=UTF-8")

  @GET @Path("/feed/{locale}") @Produces(Array("application/atom+xml"))
  def feed(@PathParam("locale") raw: String): Response = {
    val locale = PostLocale.parse(raw)
    response(DiscoveryXml.feed(origin, locale, pages.feed(locale)), "application/atom+xml; charset=UTF-8")
  }

  @GET @Path("/robots") @Produces(Array("text/plain"))
  def robots(): Response = response(s"User-agent: *\nAllow: /\nSitemap: $origin/sitemap.xml\n", "text/plain; charset=UTF-8")
}
```

`DiscoveryXml` verwendet den XML-Writer des JDK für korrektes Escaping.
Zusammenfassungen sind reiner Text, kein eingesetztes HTML. Unzulässige
XML-Steuerzeichen werden ersetzt. Die Antwort liegt bereits als UTF-8-Bytearray
vor; die bestehende Transaktionsgrenze schließt die Lesetransaktion deshalb
vor dem Senden. Ein zusätzlicher `flush` gehört nicht in diese Methoden.

Die Anwendung liefert diese Antworten mit `no-store`. Eine zurückgezogene
Veröffentlichung wirkt damit beim nächsten Request. Feedreader können früher
heruntergeladene Einträge weiterhin aufbewahren.

## HTTP und Seiteninhalt stimmen überein

Die Wurzeladresse und `/index.html` leiten mit HTTP 308 direkt nach `/en`
weiter. Unterstützte Artikelaliase und abschließende Schrägstriche führen
ebenfalls zu einer bevorzugten Adresse; Query-Parameter bleiben erhalten.
`/feed.xml` verweist auf den englischen Feed. Alle Weiterleitungsziele sind
festgelegte lokale Pfade.

Unbekannte lokalisierte Routen rendern die gemeinsame Fehlerkomponente mit
HTTP 404. Fehlende Artikel und Entwürfe bleiben ebenfalls 404. GET und HEAD
liefern denselben Status und dieselben Metadaten, HEAD jedoch keinen Body.
Konto- und Redaktionshüllen senden `X-Robots-Tag: noindex`; ihre tatsächliche
Absicherung bleibt Aufgabe von Anmeldung und Berechtigungsprüfung.

Vor dem Start dieser Revision stoppen wir das Backend und migrieren:

```text
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain preview"
sbt --server "application-backend/runMain com.anjunar.blog.SchemaMain migrate"
sbt --server frontendAssets "application-backend/run"
```

Die Migration ergänzt zwei Spalten. Ein zweiter Lauf führt keine Anweisung aus.
Die nur lesende Vorschau kann wegen des bestehenden PostgreSQL-CHECK-Prädikats
`INCOMPLETE` melden; die Migration prüft es unter ihren Sperren. Der
[Kapitel-Leitfaden](https://github.com/anjunar/anjunar-blog-example/blob/ab293295d9489d3bb868061d3ccd2969f316938f/docs/completing-the-public-pages.md) erklärt diesen
Unterschied und die Testumgebung.

Die Browsertests prüfen den Head vor dem JavaScript-Start und nach der
Clientnavigation. Sie parsen das XML, prüfen Entwürfe und zurückgezogene
Übersetzungen, testen Weiterleitungen und HEAD und behalten die Prüfungen der
DOM-Identität aus Kapitel 23 bei. Ein übersetzter Artikel, ein Sprachwechsel,
der Rückweg zur Übersicht und anschließend die Kontoseite: Bei jedem Schritt
bleiben genau die Metadaten der jeweiligen Seite übrig.

Damit ist das Tutorial nach 24 Kapiteln abgeschlossen. Aus einem leeren Projekt
ist ein persistenter Blog mit Konten, Redaktion, Medien, Übersetzungen,
Server-Rendering, Hydration und konsistenten öffentlichen Adressen geworden.
Implementierung und Tests bleiben über den Checkpoint dieses Kapitels erreichbar.
