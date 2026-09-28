# Die Benutzeroberfläche übersetzen

Unser Blog kann Artikel mit formatiertem Text und Bildern veröffentlichen. Nun sollen Leser die Navigation auf Deutsch verwenden können, und Administratoren sollen denselben Artikel mit deutschen Bedienelementen bearbeiten können.

Hier geht es um Übersetzungen der Benutzeroberfläche. Wenn aus **Save post** **Beitrag speichern** wird, bleiben Titel und Dokument unverändert. Die Übersetzung dieser redaktionellen Inhalte braucht ein eigenes Modell, das wir in Kapitel 21 aufbauen.

Dieses Kapitel ergänzt englische und deutsche URLs, einen Nachrichtenkatalog und einen Sprachumschalter. Der [Leitfaden zum Checkpoint](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/docs/translating-the-interface.md) enthält die ausführbare Version und die Prüfbefehle. Wir verwenden scalajs-ui 1.0.13 von Maven Central; eine Datenbankmigration ist nicht erforderlich.

## Die Sprache aus der URL bestimmen

Rufe /de/posts/same-post direkt auf. Die Seite soll sofort Deutsch verwenden, ohne einen vorherigen Besuch oder eine gespeicherte Browsereinstellung vorauszusetzen. Beim Neuladen und Teilen der Adresse soll die Sprachauswahl erhalten bleiben.

Wir initialisieren die Laufzeit anhand derselben URL, mit der auch der Router gestartet wird. In [BlogPage.compose](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogPage.scala), noch bevor Navigation oder geroutete Komponenten zusammengesetzt werden:

```scala
import ui.core.i18n.I18nRuntime

val initialUrl = cursor.browserUrl.getOrElse("/")
val translations = I18nRuntime.managed(BlogI18n.config, initialUrl)
I18nRuntime.provide(translations)(using this)
```

BlogI18n.config deklariert Englisch und Deutsch als unterstützte Sprachen und Englisch als Standardsprache. Durch die Bereitstellung der Laufzeit ist sie im gesamten Komponentenbaum verfügbar, auch im Editor. Wir speichern keinen zweiten Sprachwert im lokalen Speicher des Browsers.

Der HTTP-Server muss diese Adressen ebenfalls erkennen. FrontendHandler liefert nun für bekannte Routen unter /en oder /de das vorhandene Browsergerüst aus. Konto- und Redaktionsseiten behalten ihre private Cache-Richtlinie. REST- und Bildanfragen verwenden weiterhin /service/; das Sprachpräfix gehört zur Seitenadresse, nicht zur API.

Unbekannte Sprachen und unbekannte Routen liefern 404. Bei einer bekannten Route mit einem nicht vorhandenen Artikel wird zunächst das Browsergerüst geladen; erst die REST-Anfrage meldet den fehlenden Inhalt. HTTP-Statuscodes für serverseitig gerenderte Seiten kommen in einem späteren Kapitel.

## Für jede Nachricht einen Katalogeintrag anlegen

In den früheren Kapiteln wurde das i18n-Makro bereits für Beschriftungen verwendet. Diese englischen Quellnachrichten dienen jetzt als Schlüssel für die deutschen Übersetzungen.

[BlogI18n](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/application/frontend/src/main/scala/com/anjunar/blog/frontend/BlogI18n.scala) verwendet diesen Helfer. Der Ausschnitt zeigt zwei Einträge aus dem umfangreicheren Katalog:

```scala
import ui.core.i18n.{CatalogEntry, I18n, I18nLocale, MessageCatalog, RuntimeMessage, i18n}
import ui.editor.EditorMessages

val German: I18nLocale = I18nLocale("de")

private def de(message: RuntimeMessage, translation: String): CatalogEntry =
  I18n.entry(message.key).translations(German -> translation)

val catalog: MessageCatalog = MessageCatalog(
  de(i18n"Save post", "Beitrag speichern"),
  de(EditorMessages.editImage, "Bild bearbeiten")
)
```

Das Makro liefert die Nachrichtenidentität und die Quellinformationen. Wir greifen auf message.key zurück, statt eigene Hashes oder Kennungen zu pflegen. Fehlt ein deutscher Eintrag, verwendet der Resolver den englischen Quelltext.

Der zweite Eintrag zeigt eine sinnvolle Grenze: Der Editor stellt EditorMessages bereit. Seine Werkzeugleiste und Dialoge verwenden dieselbe Laufzeit wie unsere Anwendung. Wir übersetzen diese öffentlichen Nachrichten in unserem Katalog; das vorhandene DSL editor("content") und die Plugin-Konfiguration bleiben unverändert.

Ein Katalog sollte vollständige Nachrichten enthalten. „Save“, „post“ und Satzzeichen als getrennte Stücke zu übersetzen, würde jeder Sprache die englische Satzstruktur aufzwingen.

## Werte und Bindungen beibehalten

Manche Nachrichten enthalten Daten. Unsere Liste kennt bereits die Anzahl der geladenen Zeilen und die Gesamtzahl der Treffer. In der Zusammensetzung von [PostListPage](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/application/frontend/src/main/scala/com/anjunar/blog/frontend/PostListPage.scala):

```scala
import ui.core.i18n.{I18n, i18n}
import ui.core.layout.TextComponent.text

text(i18n"Showing ${I18n.named("count", table.rows.size)} of ${I18n.named("total", table.size)} posts") {}
```

Das deutsche Muster lautet „{count} von {total} Beiträgen“. Übersetzer können die benannten Werte dort platzieren, wo sie im Satz gebraucht werden. Die Beispielwerte im Katalog kennzeichnen das Muster; die angezeigte Nachricht setzt die tatsächlichen Zahlen ein. Tests prüfen, dass jede Übersetzung die Platzhalternamen der Quelle beibehält.

Diese Muster setzen benannte Werte ein. Sie sind keine ICU-Pluralisierungssprache. Deshalb wählen wir eine Formulierung, die für die Zähleranzeige passt.

Das DSL akzeptiert das Makro direkt: text(i18n"Save post"), button(i18n"Save post") oder innerhalb eines Formularelements placeholder = i18n"Choose an author". Diese APIs verwenden TextValue, um den i18n-Kontext der Komponente aufzulösen und die reaktive Bindung zu erhalten. Gleiches gilt für ariaLabel = i18n"Publication status" und SelectOption("DRAFT", i18n"Draft"). Ein expliziter Übersetzungsaufruf ist nicht nötig.

Bei Nachrichten, die sich durch eine Zustandsänderung ändern, übergib state.map(...) an das DSL und gib in der Abbildung die passende i18n-Nachricht zurück. TextValue beobachtet sowohl die Nachrichteneigenschaft als auch die Sprache. Das Makro allein beobachtet keine beliebigen interpolierten Werte: Die Listendaten bleiben für die jeweilige geroutete Seite unverändert. Wenn sich die Daten ändern, muss auch eine aktualisierte Nachricht erzeugt werden.

## Den Sprachwechsel als Navigation behandeln

Der Schalter im Kopfbereich behält die aktuelle Route bei und ändert nur deren Sprachpräfix. Wer sich auf /de?q=Scala#posts befindet, wechselt zu /en?q=Scala#posts; Suchparameter und Anker bleiben erhalten. Mit der Zurück-Schaltfläche des Browsers geht es wieder auf Deutsch.

Der Handler in BlogPage.compose verwendet die öffentliche Pfad-API des Routers:

```scala
import org.scalajs.dom
import ui.core.i18n.I18nLocale

def changeLanguage(next: I18nLocale): Unit =
  if (!languageNavigation.blocked.get && translations.locale.get != next) {
    val state = router.state.get
    val suffix = if (cursor.isBrowser)
      dom.window.location.search + dom.window.location.hash
    else state.search + state.hash
    router.navigate(router.localizedPath(state.path, next) + suffix)
  }
```

Die aktuelle Browser-Fragmentkennung ist bei Kontoseiten wichtig. Bestätigungs- und Passwort-zurücksetzen-Seiten entfernen ihr geheimes Token aus der sichtbaren URL, nachdem sie es gelesen haben. Würde ein früherer Router-Hash wiederhergestellt, könnte dieses Token erneut in der URL erscheinen.

Die Navigation ersetzt außerdem die geroutete Komponente. Ein ungespeicherter Artikel, eine unfertige Suche oder ein teilweise eingegebenes Passwort darf beim Sprachwechsel nicht verloren gehen.

[LanguageNavigation](https://github.com/anjunar/anjunar-blog-example/blob/645aceabb859696ae05bed7d09b4e079e5a75519/application/frontend/src/main/scala/com/anjunar/blog/frontend/LanguageNavigation.scala) kombiniert deshalb die Schutzfunktionen der gerade geöffneten Formulare. Ein Formular beobachtet geänderte Werte und laufende Vorgänge; beim Entfernen werden Schutzfunktion und Abonnements wieder abgemeldet. Solange ein Wechsel blockiert ist, sind die Sprachschaltflächen deaktiviert und erklären, dass das Formular zuerst abgeschlossen oder zurückgesetzt werden muss.

Nach dem Speichern kann der Artikel in der anderen Sprache mit den gespeicherten Werten erneut geöffnet werden. Seiten mit Wiederherstellungs-Token bleiben geschützt, bis der Nutzer sie über ihren regulären Abschlusslink verlässt. Dieser Schutz gilt für die Sprachschaltflächen; er verhindert weder ein Neuladen noch jede andere Art der Navigation.

## Den gesamten Ablauf prüfen

Rufe /de auf, öffne einen Artikel, wechsle zu Englisch, gehe zurück und lade die Seite neu. Navigation und Bedienelemente sollen der URL folgen, während der Artikel unverändert bleibt. Für Hilfstechnologien wird außerdem html.lang aktualisiert.

Bearbeite danach einen Titel. Der Sprachwechsel muss gesperrt sein, bis die Änderung gespeichert oder verworfen wurde. Nach dem Speichern und dem Sprachwechsel öffne den Bilddialog erneut: Seine Beschriftungen sollen deutsch sein, und der Dokumentinhalt soll erhalten bleiben.

Die sechs neuen Browsertests decken diese Übergänge, Kontofehler, Wiederherstellungs-Token und Fehler-Routen ab. Katalogtests prüfen Rückfallwerte, reaktive Nachrichten und Platzhalternamen; Servertests prüfen lokalisierte Routen und Cache-Richtlinien.

Dieser Checkpoint hat ausdrücklich festgelegte Grenzen. Detaillierte Backend-Fehler und eingebaute Validierungsmeldungen bleiben vorerst englisch, ebenso transaktionale E-Mails. Das anfängliche statische HTML ist bis zum Browserstart weiterhin englisch. Ein späteres Kapitel behandelt serverseitiges Rendering.

Damit steht dieselbe Benutzeroberfläche in zwei Sprachen zur Verfügung. Im nächsten Kapitel modellieren wir Übersetzungen der Artikel selbst – einschließlich des Umgangs mit fehlenden Übersetzungen.
